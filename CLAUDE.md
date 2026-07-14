# 03_tools · 公司 dev 根 CLAUDE.md

> 所有子项目工具（gsa01-*、siamhan_crm、gsa02/03、email_assistant …）都**叠加于本文件**。
> 子项目的 CLAUDE.md 只写自己的差异，通用架构标准以本文件为准。

## §1 总体设计逻辑：中心化处理 → 分发到子系统 (Centralize → Fan-out)

这是贯穿所有工具的**核心架构原则**，后续每个项目工具的设计都要遵循它：

**事件和信息（email / LINE / 微信 / 文件 / 表单 …）先集中到一个中心层做一次处理，
再按规则分发到各子系统；子系统不各自去源头重复抓取和处理。**

一次进入、一次去重、一次识别，之后扇出（fan-out）。绝不让 N 个子系统各自扫全量数据源——
那会造成 N×重复抓取、N 份不一致的判断、和无法收敛的成本。

### 三层职责（每层只做一件事）

1. **接入 / 去重（Ingest, 一次）** — 只有中心层连数据源（如共享信箱 lifang@siamhan.com
   的 gmail-inbound），把每个事件抓一次、按稳定键去重（email 用 message_id），写入**中心账本**
   （append-only ledger）。这一层是唯一的"贵动作"，永远只发生一次。
2. **路由 / 分发（Route & Fan-out）** — 中心层查一张**显式路由表**（route_rules），
   决定这条事件该推给哪些子系统，然后主动推送（push / webhook）。加新子系统 = 往路由表插一行，
   **不改代码、不重扫历史**。
3. **各子系统本地处理（Spoke-local）** — 每个子系统只对收到的那一份，按**自己的维度**贴轻量标签
   （CRM 问"归哪个客户"，GSA 问"归哪个进口票"）。这是给同一事件加不同视角，不是重复处理。

### 星形，不是网状 (Star, not mesh)

- 中心是一颗**星**：唯一看全量的地方。子系统之间**不互相拉数据**。
- **身份/主数据单一真源**：客户/联系人/项目身份以 CRM(`siamhan_crm`) 为准，其他项目**只读**
  引用（`sharing.*` 视图 + postgres_fdw），一个方向流动。
- 出件同样入中心账本（自铸 Message-ID），保证收发一条线程可追溯。

### 判断一个新设计是否合规（checklist）

- [ ] 这个新工具是**从中心层收推送**，还是自己又去连了数据源？——必须是前者。
- [ ] 新增分发目标，是**插一行 route_rules**，还是改中心层代码？——必须是插行。
- [ ] 它需要的客户/联系人身份，是**只读引用 CRM**，还是自建一份？——必须只读引用。
- [ ] 它对事件的处理，是**只贴自己维度的标签**，还是重跑一遍中心已做的识别？——必须只贴自己的。

### 反面模式（禁止）

- 每个项目工具各自去连信箱扫全量邮件。
- 分发逻辑散落在各 spoke 的匹配代码里（隐式路由）——要收敛到中心的 route_rules。
- 子系统各存一份客户名单并各自维护。

详见 `docs/ARCHITECTURE_hub_and_spoke.md`（含 route_rules 表设计与接入步骤）。

## §2 参考实现 (reference implementations)

- **Email hub**：`gsa01-ops` 的 `gmail-inbound`（v4 共享路由器）= 中心接入+账本+扇出；
  `siamhan_crm` 的 `crm-hub-intake` = CRM 这个 spoke 的入口。契约见
  `siamhan_crm/docs/email_hub_handoff.md`。
- **身份真源 / 数据共享**：`siamhan_crm/docs/data_sharing.md`（`sharing.*` 只读视图 + FDW）。

## §3 平台与约定 (platform conventions)

- 后端统一 Supabase（Postgres + RLS + edge functions Deno + pg_cron/pg_net + Vault）。
- 前端多为单文件 web dashboard（supabase-js from esm.sh），部署 Vercel。
- 迁移文件 `supabase/migrations/NNNN_*.sql` 顺序编号，只增不改历史。
- 跨系统认证用自定义 header 密钥（x-hub-key 等）+ verify_jwt=false，密钥存双侧 secrets。
