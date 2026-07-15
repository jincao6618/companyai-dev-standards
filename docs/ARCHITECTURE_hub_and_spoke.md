# 中心化处理 → 分发到子系统 (Hub-and-Spoke)

公司 dev 的核心架构标准。总述见根 `CLAUDE.md §1`；本文给出完整设计、`route_rules`
表、以及"接入一个新子系统"的标准步骤。

## 1. 为什么

事件和信息从多个渠道进来（email / LINE / 微信 / 文件 / 表单），会被多个子系统关心
（CRM、GSA 进口作业台、未来的其他项目工具）。若每个子系统各自去源头抓取和识别：

- N 个系统 × 全量数据源 = 重复抓取，成本随项目数线性膨胀；
- 各系统对同一条信息各判一次，结论不一致、无法对账；
- 加一个新系统要动源头接入，牵一发动全身。

因此：**一次进入、中心去重、按规则扇出、各子系统只对自己那份贴标签。**

## 2. 三层

```
渠道(email/LINE/微信/文件)
        │  只有中心层连源头
        ▼
┌──────────────────────────────┐
│ 中心层 HUB                     │
│ 1) 接入+去重 → 中心账本 ledger  │   append-only，稳定键去重(message_id)
│ 2) 路由 route_rules → 扇出      │   查表决定推给谁，push/webhook
└──────────────────────────────┘
     │            │            │
     ▼            ▼            ▼
   CRM spoke   GSA01 spoke   未来项目 spoke
   (归客户)     (归进口票)     (自己的维度)
        ▲
        └─ 身份单一真源：客户/联系人/项目 只读引用 CRM (sharing.* + FDW)
```

- **接入/去重**：唯一"贵动作"，只发生一次。当前实现 = `gsa01-ops` 的 `gmail-inbound` v4，
  扫共享信箱 → 写 `hub_email_ledger`，按 message_id 去重，回信按 in_reply_to/references
  继承线程与归属。
- **路由/扇出**：查 `route_rules`（见 §3），命中则推给对应 spoke 的入口函数
  （CRM = `crm-hub-intake`，带 x-hub-key）。
- **spoke 本地**：只对收到的那份按自己维度贴标签，不重跑中心已做的抓取/去重/识别。

## 3. `route_rules` — 显式路由表（中心层持有）

目标：把"这条事件推给谁"从各 spoke 的隐式匹配代码，收敛成 hub 侧一张表。
**加新子系统 = 插一行，不改代码、不重扫历史。**

```sql
-- 位置：HUB 侧库（当前 = gsa01 库）。CRM 侧不持有此表，只作为被推送的 spoke。
create table if not exists route_rules (
  id           bigint generated always as identity primary key,
  spoke        text    not null,              -- 目标子系统: 'crm' | 'gsa01' | ...
  endpoint     text    not null,              -- 该 spoke 的入口 URL (edge function)
  match_kind   text    not null,              -- 'all' | 'sender_domain' | 'sender_addr'
                                              --  | 'to_addr' | 'keyword' | 'thread_owner'
  match_value  text,                          -- 与 match_kind 配对的值; 'all' 时为 null
  priority     int     not null default 100,  -- 小者先评估
  fanout       boolean not null default true, -- true=命中也继续评估其他规则(可多投)
                                              -- false=命中即停(独占路由)
  enabled      boolean not null default true,
  note         text,
  created_at   timestamptz not null default now()
);
create index if not exists idx_route_rules_active
  on route_rules(enabled, priority) where enabled;

-- 种子：CRM 收全部(身份真源+triage 兜未知发件人)；GSA01 只收其相关。
insert into route_rules(spoke,endpoint,match_kind,match_value,priority,fanout,note) values
  ('crm','https://rujlokezqpkttetasugr.supabase.co/functions/v1/crm-hub-intake',
   'all', null, 10, true, 'CRM 是身份真源，看全量，兜未知发件人'),
  ('gsa01','<gsa01 intake endpoint>',
   'sender_domain','huataili.cc', 50, true, '华士/GSA 相关来件'),
  ('gsa01','<gsa01 intake endpoint>',
   'thread_owner','gsa01', 50, true, '已归属 GSA 线程的回信');
```

**评估逻辑（hub 扇出时）**：按 `priority` 升序遍历 `enabled` 规则，事件命中 `match_kind`
即向该 `endpoint` 投递；`fanout=false` 的规则命中后停止继续评估（独占）。同一 spoke 多次命中
只投一次（按 spoke 去重）。CRM 的 `all` 规则保证它始终收到。

**回信/线程**：`thread_owner` 规则让已归属某 spoke 的线程，其后续回信自动继续投给该 spoke，
无需重判——归属信息来自 ledger 的线程投影。

## 4. 接入一个新子系统（标准步骤）

1. 新子系统建一个入口 edge function（如 `xxx-hub-intake`），`verify_jwt=false` +
   校验 `x-hub-key`；收到后写入自己库、按**自己维度**贴标签；用 message_id 幂等去重。
2. 需要客户/联系人/项目身份时，**只读引用 CRM**：`sharing.*` 视图 + postgres_fdw，
   单向流动，不自建主数据（见 `siamhan_crm/docs/data_sharing.md`）。
3. 在 hub 的 `route_rules` **插行**：指定 `spoke/endpoint/match_kind/match_value`。上线即生效，
   不动源头、不重扫历史。
4. 出件（该系统往外发邮件/消息）也回写中心账本（自铸 Message-ID），保持收发同线程可追溯。

## 5. 合规红线（对照根 CLAUDE.md §1 checklist）

- 子系统**从 hub 收推送**，不自己连源头扫全量。
- 分发规则集中在 `route_rules`，不散落在各 spoke 的匹配代码。
- 身份主数据只读引用 CRM，不各存一份。
- spoke 只贴自己维度的标签，不重跑中心已做的识别。

## 6. 现状与待办

- [x] Email 中心接入+账本+扇出：`gsa01-ops/gmail-inbound` v4（隐式匹配版已上线）。
- [x] CRM spoke 入口：`crm-hub-intake` v3。
- [x] 身份共享：`sharing.*` + `crm_share_reader` FDW。
- [ ] **route_rules 显式化**：把 gmail-inbound v4 里的隐式匹配，迁到本表驱动（hub 侧改动，
      需在 gsa01 库建表 + gmail-inbound 读表扇出）。本文 §3 为落地设计，待 hub 侧排期实施。

## 7. 决策记录 (decisions)

- **2026-07-15 · hub 暂留 GSA01，不迁不新建。** 现状：email 扫描/路由 hub 在 `gsa01-ops`，
  CRM 作为 spoke 收扇出。评估过三条路：(A) 维持现状；(B) 把 hub 并入 CRM，CRM 成真正中心、
  GSA 降为 spoke；(C) 独立中立 hub 项目，CRM/GSA 平级。
  **结论：现阶段维持 (A)。** 两名合伙人、约两套系统的规模下，(C) 是最重的、只是"换个盒子"
  并不消除"双中心"，(B) 才真正收敛但属较大有风险的联动迁移。现状唯一成本是"handoff 税"（改 hub
  要跨会话协调 body_text / received_at / 分类器 / reply-send），成本可控且已用 handoff 文档承接。
  **触发再评估的条件**：出现第 3 个 spoke，或跨会话协调开始明显拖累。届时优先选 (B) 并入 CRM，
  而非 (C)。当前专注：用实盘数据修 CRM bug，系统稳定后再谈 hub 归并。
