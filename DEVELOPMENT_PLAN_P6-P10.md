# P6–P10 阶段规划 — 用户权限 / 测试管理 / 数据统计 / 站内助手 / 性能·插件·版本

> 版本: v1.3（2026-09-04 确认范围与边界；2026-09-05 P10 并入；2026-09-07 P6-1~P6-5
> 实现状态 + P6 暂记验收通过 + 13.6 后置增补「项目可见性两层 + 权限申请审批流」）
> 归属: 本文件是 `DEVELOPMENT_PLAN.md` 的阶段扩编。四个新阶段插在 P5（MCP）之后、
> 原 P6（性能/插件/版本，**已顺延为 P10**）之前；**P10 全章于 2026-09-05 自主计划
> 十四章并入本文件**。主文档只保留编号变更与指针，完整范围、边界、迁移、路由、
> 批次与验收门槛以本文件为准。
>
> 排序依据（2026-09-04 用户确认）：
> - **用户与权限必须第一**——现在全平台只有一个硬编码账号（`lib/auth.ts:20-25` 是唯一
>   的 `INSERT INTO users`）。「指派给谁」「谁做的归因」「助手用谁的身份」在一个用户的
>   系统里全是空话；主计划 9.8 也早已把用户注册列为站内助手的前置 1。
> - **测试管理在统计之前**——统计要画「用例数 / 覆盖率」趋势，而测试管理会引入第三类
>   用例（文本用例）与第三个覆盖口径（自动化覆盖率）。反过来做，P8 的口径当场过期。
> - **助手最后**——依赖 P5 的工具面、P6 的用户与通知、页面结构在 P7/P8 之后才定型
>   （教程的 selector 不能对着还在动的页面写）。
>
> 章节映射（本文件 → 主计划）：P6 = 十、P7 = 十一、P8 = 十二、P9 = 十三、
> P10 = 十四。主计划原「十、P6 性能/插件/版本」已顺延为 P10，并于 2026-09-05
> 连章并入本文件（主计划原位只留指针）。

---

## 十、P6 — 用户与权限（注册 / 成员 / 角色落地，约 3 周）

### 10.0 P6 范围与边界（2026-09-04 确认）

**问题**

平台只有一种身份：每次启动种子的 `admin@local.test`（`lib/auth.ts:20-25`，密码写死
`admin123`，`Login.tsx:14` 还替用户预填好了）。由此缺了一整层产品能力：

1. **没有第二个账号的创建路径**——无注册路由、无建号接口、无 CLI，加人只能手写 SQL。
2. **成员与角色是死的**——`user_project_roles` 的唯一写入是建项目时给创建者插一行
   （`projects.ts:39`）；「把某个人加进项目 / 改他的角色」没有 API、没有页面。
3. **三级角色只判了两级**——`rbac.ts:11` 唯一的区别是 `role !== "viewer"`，
   `project_admin` 与 `developer` 在代码里完全等价。
4. **前端不知道自己是谁**——`/auth/me` 只回 `isSystemAdmin`（`auth.ts:18`），前端因此
   无法按角色隐藏任何东西；viewer 看到与 developer 完全相同的界面，点按钮才被 403。
   401 也没有统一处理（`api.ts:2061` 只有 request 拦截器），token 失效只能靠刷新页面。
5. **`users.status='disabled'` 有判定端、没有写入端**——判定在 `auth.ts:9` 与
   `lib/auth.ts:29`（每请求查库，所以禁用天然即时生效），但没有开关能把它置上去。

**核心模型**

**身份在 `users`，角色在 `user_project_roles`，两级判据（平台 `is_system_admin` /
项目三级角色）不变、补第三级执行。进人走两条通道：邀请码注册 + 管理员建号；
不做开放注册、不引 SMTP。前端按角色隐藏入口而不是禁用。**

**边界决策（12 项）**

1. **注册两条通道，不做开放自助注册**。内部测试平台，开放注册意味着任何拿到 URL 的
   人都能造账号进看板。① 邀请码：系统管理员签发 `apiinv_` 前缀的一次性码（可预设
   「进哪个项目、什么角色」），受邀人开 `/register?invite=` 自设密码；**链接带外传递**
   （企微/钉钉），与 Runner 部署命令、上报 Token 同一种交付方式。② 管理员建号：当场
   显示初始密码**仅一次**（照 `RepoCredentials.tsx` 的明文只出现一次纪律），并置
   `must_change_password` 强制首登改密。
2. **不引 SMTP，不推翻「Email 不做」**（P3 已定，`notify.ts:10`）。邀请链接由签发人
   自己转达。注册不预填邮箱验证——内网平台，邮箱归属由管理员目视判断。
3. **邀请码是第五次照抄 Token 形状**（`users.password_hash` / `ingest_tokens` /
   `runner_tokens` / P5 的 `mcp_tokens`）：scrypt `salt:digest`、明文前 8 位做候选集
   收窄、部分索引排除已用码。**照旧不抽公共层**——五份 40 行的具体实现仍然好过一层
   参数化抽象；这不是懒惰，是「吊销/过期/单次使用」的差异会在抽象层漏出来。差异点：
   邀请码默认 7 天过期、单次使用（`consumed_at` + `consumed_user_id`）。
4. **`project_admin` 与 `developer` 真正分开**：新增 `requireProjectAdmin` 守卫（系统
   管理员短路），只挂三类路由——成员管理、项目设置写（默认环境 / `store_plaintext` /
   将来的 `mcp_enabled`）、项目删除。**不改 `canAccess` 的签名**——80+ 处调用点零
   回归；「项目管理员多管什么」从一开始就限定在「管人与管项目级配置」，资源 CRUD
   （接口/用例/流程/套件）developer 照旧可写。
5. **`GET /projects/:id` 的返回加 `myRole`**（admin / developer / viewer / system_admin），
   项目列表 `queryProjectMetrics` 同步带出。**不给 `/auth/me` 加全部项目角色映射**——
   用户在 50 个项目里时那是一坨没人整体消费的数据；前端消费的始终是「当前项目我的
   角色」，`projectStore` 已经预取项目详情，零额外请求。
6. **前端按角色隐藏入口，不做禁用态**（沿用 `GlobalApp.tsx:223-232`「一个你永远打不开
   的 tab 只是杂物」的既定策略）。viewer 隐藏：新建/编辑/删除/执行按钮、导入入口、
   批量操作条。developer 额外隐藏：成员管理入口。**只藏入口不藏路由**——后端守卫是
   权限的单一事实，URL 直达时后端照拒。
7. **JWT 补过期：`expiresIn` 默认 7 天**（`JWT_EXPIRES_IN` 环境变量），登录响应带
   `expiresAt`。**不做 refresh token 轮换**——`currentUser()` 每请求查库，改密/禁用
   天然即时生效；refresh 解决的是「无状态 token 撤销难」，这里不是那个问题。改密时
   在 payload 里带 `pwdEpoch`（`users.pwd_epoch` 自增），校验时不等即 401——旧 token
   全部失效不需要黑名单表。
8. **axios 补响应拦截器**：401 → 清 token 跳登录（带 `returnTo`）；403 → 顶部
   `message` 提示无权限（不跳转——用户可能只是在编辑一个刚被降权的页面，跳走会丢
   上下文）。
9. **只做禁用、不做删除用户**。`projects.owner_id` 是全库唯一没写 `ON DELETE` 的
   users 外键（`001:20`，硬删直接被 FK 挡）；且执行历史、审计、归因都要活过被引用方
   （与 `endpoint_name` 快照同一条纪律——历史要能回答「谁做的」）。禁用即时生效
   （既有判定端就位），角色行保留。
10. **成员变更全进审计**：`AUDIT_ACTIONS` 从 3 个动作扩到 12 个（user.create /
    user.disable / user.enable / user.reset_password / member.upsert / member.remove /
    invite.create / invite.revoke / invite.consume / project.role...），`detail` 只放
    email 与角色，**绝不含密码或邀请码明文**（`lib/audit.ts:6-9` 纪律）。
11. **审计从「只写不读」变「可读」**：系统管理新增「审计日志」只读 tab（系统管理员），
    项目设置页给「最近 50 条项目级操作」。这是本阶段最便宜的功能——表在写、写入点
    现成，缺一条 SELECT 和一张表；没有它，「谁把谁加进来的」永远答不出来。
12. **站内通知不在本阶段**——用户已定（2026-09-04）：通知中心（铃铛小红点 + 弹窗）
    归 **P9 站内助手**。因此 P6/P7 的成员变更、计划指派**不触发任何站内通知**，
    P9 落地通知生产端时回补。

### 10.1 数据库迁移：051_p6_users.sql

```sql
ALTER TABLE users ADD COLUMN IF NOT EXISTS must_change_password BOOLEAN NOT NULL DEFAULT false;
ALTER TABLE users ADD COLUMN IF NOT EXISTS pwd_epoch INTEGER NOT NULL DEFAULT 0;
ALTER TABLE users ADD COLUMN IF NOT EXISTS last_login_at TIMESTAMPTZ;
ALTER TABLE users ADD COLUMN IF NOT EXISTS updated_at TIMESTAMPTZ NOT NULL DEFAULT now();

CREATE TABLE IF NOT EXISTS invitations (
  id UUID PRIMARY KEY,
  token_hash TEXT NOT NULL,
  token_prefix TEXT NOT NULL,
  note TEXT NOT NULL DEFAULT '',           -- 给签发人自己看的备注（给谁/为什么）
  project_id UUID REFERENCES projects(id) ON DELETE CASCADE,   -- NULL = 不预分配项目
  role TEXT CHECK (role IN ('project_admin','developer','viewer')),
  created_by UUID REFERENCES users(id) ON DELETE SET NULL,
  expires_at TIMESTAMPTZ NOT NULL,
  consumed_at TIMESTAMPTZ,
  consumed_user_id UUID REFERENCES users(id) ON DELETE SET NULL,
  revoked_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX IF NOT EXISTS invitations_prefix_idx ON invitations (token_prefix) WHERE consumed_at IS NULL AND revoked_at IS NULL;

ALTER TABLE user_project_roles
  ADD COLUMN IF NOT EXISTS granted_by UUID REFERENCES users(id) ON DELETE SET NULL,
  ADD COLUMN IF NOT EXISTS granted_at TIMESTAMPTZ NOT NULL DEFAULT now();
```

`role` 允许 NULL（不预分配项目）；注册成功时仅当 `project_id` 非空才写
`user_project_roles`。`must_change_password` 同样覆盖管理员建号与重置密码两条路径。

### 10.2 后端新 API

```
POST   /api/v1/auth/register                  公开；{invite, name, password}；消费邀请码
GET    /api/v1/system/users                   系统管理员；列表 + keyword
POST   /api/v1/system/users                   系统管理员；建号，初始密码只在本次响应出现
PATCH  /api/v1/system/users/:userId           系统管理员；name / status / is_system_admin
POST   /api/v1/system/users/:userId/reset-password   系统管理员；新密码只出现一次 + 强制改密
GET    /api/v1/system/invitations             系统管理员
POST   /api/v1/system/invitations             系统管理员；签发，明文码只出现一次
POST   /api/v1/system/invitations/:id/revoke  系统管理员
GET    /api/v1/projects/:id/members           项目读权限（viewer 可看，知道找谁要权限）
POST   /api/v1/projects/:id/members           requireProjectAdmin；{userId, role}
PATCH  /api/v1/projects/:id/members/:userId   requireProjectAdmin；改角色
DELETE /api/v1/projects/:id/members/:userId   requireProjectAdmin；移除
GET    /api/v1/system/audit-logs              系统管理员；分页 + action/project 筛选
GET    /api/v1/projects/:id/audit-logs        requireProjectAdmin；最近 50 条
```

`GET /projects/:id` 与项目列表响应新增 `myRole`。守卫放行但不挡只读：`myRole` 对
`is_system_admin` 回 `system_admin`。

### 10.3 前端

- **系统管理新增两个 tab**（`GlobalApp.tsx:198` 的 `SystemTab` 联合与 `:233` 的数组）：
  「用户」（列表 / 建号 / 禁用 / 重置密码 / 邀请码）与「审计日志」——非管理员不渲染
  tab（不是禁用）。
- **项目层新增导航项「成员与权限」**：`ProjectShell.tsx` 的 `PAGES` / `NAV_ICONS` /
  nav 分组（`:12` / `:24` / `:90`）三处同改，落「通用」组；`main.tsx` 加路由
  `/projects/:projectId/members`。页面 = 成员表（角色下拉就地改）+ 添加成员（搜索
  平台用户 → 选角色）+ 最近审计 50 条。
- **注册页** `/register`（`Protected` 之外，与分享页同层）；`Login.tsx` 删掉
  `admin@local.test` 预填、加「使用邀请码注册」链接。
- **`useMyRole()` hook**（读 `projectStore` 的 `myRole`）+ 全站 viewer 入口隐藏走查。
- **`api.ts` 响应拦截器**（401 / 403，见边界 8）。
- **强制改密页**：登录后发现 `mustChangePassword` 则只放行 `/change-password`。

### 10.4 实施顺序（P6-1 … P6-5）

| 步 | 内容 | 产出 |
| --- | --- | --- |
| P6-1 | 迁移 051 + `requireProjectAdmin` + `myRole` 下发 — **已实现**（2026-09-07） | `pnpm check` 过，既有路由零回归 |
| P6-2 | 用户 CRUD / 禁用 / 重置密码 / 审计动作与读接口 — **已实现**（2026-09-07） | 系统管理两 tab 可用 |
| P6-3 | 邀请码签发/消费 + `/register` + 强制改密 — **已实现**（2026-09-07） | 第二个账号能进来 |
| P6-4 | 成员 CRUD + 前端成员页 + `useMyRole` 隐藏走查 — **已实现**（2026-09-07） | viewer/developer/admin 三级界面分叉 |
| P6-5 | JWT 过期 + pwdEpoch + 401/403 拦截器 — **已实现**（2026-09-07） | 改密后旧 token 立即失效 |

> **P6-1 实现状态（2026-09-07）**：迁移 051 落地（users 三列 + `invitations` +
> `user_project_roles.granted_by/granted_at`）；`rbac.ts` 重构出 `myRoleInProject` 单点
> 角色查询，`requireProjectAccess` 返回值增 `myRole`（签名兼容，80+ 调用点零改动），
> 新增 `requireProjectAdmin` 并即挂两类既有路由（项目设置 PUT / 项目删除 DELETE，
> 边界 4 的第三类「成员管理」待 P6-4 建路由时挂）；`myRole` 随 `GET /projects/:id`
> 与项目列表（`queryProjectMetrics`，dashboard 聚合同步受益）下发。**一处词汇决策**：
> `myRole` 用 DB 值 `project_admin`（计划原文写的是「admin」），与成员 CRUD 的 role
> 词汇、DB CHECK 保持同一套，避免前端再长一层映射；`system_admin` 照计划短路。
>
> **P6-2 实现状态（2026-09-07）**：后端 `routes/systemUsers.ts`——`GET/POST
> /system/users`（列表 keyword/分页 + 建号：初始密码 18 字节 base64url 只在响应出现
> 一次、`must_change_password` 置位）、`PATCH /system/users/:userId`（name / status /
> isSystemAdmin，partial）、`POST …/reset-password`（同明文纪律 + 强制改密）、
> `GET /system/audit-logs`（分页 + action/project 筛选，LEFT JOIN email/name 快照）、
> `GET /projects/:id/audit-logs`（`requireProjectAdmin`，最近 50 条——P6-4 成员页
> 消费）。`AUDIT_ACTIONS` 扩到 17 动作（新增 user.create / disable / enable /
> update_role / reset_password，detail 只放 email 与前后角色）；`models/types.ts`
> 增 `ManagedUser` / `AuditLog` 与两个映射（哈希列不进管理查询）。**两处护栏**：
> ① 不能禁用自己 / 不能摘自己的管理员旗 / 最后一个启用的系统管理员不能被降
> （409，用户面的「最后一个 project_admin」纪律）；② 重置密码**不动 pwd_epoch**
> ——那是 P6-5 的 JWT 失效机制，提前动会引入本批未实现的第二个判定端。前端
> `GlobalApp.tsx` 的 `SystemTab` 增「用户」「审计日志」两 tab（非管理员不渲染），
> `UsersPanel`（建号弹窗 + 初始密码 code-block + 禁用/启用确认 + 管理员旗就地开关
> + 重置密码）与 `AuditLogsPanel`（action 筛选候选取自当前页出现过的动作、detail
> 键值行渲染）落地，i18n 中英双语补齐，`.audit-detail` 进 design-system.css。
>
> **P6-3 实现状态（2026-09-07）**：邀请码通道与强制改密全套落地。后端：新
> `lib/inviteAuth.ts`（第五次照抄 Token 形状：scrypt `salt:digest`、`apiinv_` 明文、
> 前缀收窄候选、**不抽公共层**；差异点 = 7 天过期 + 单次使用）、`routes/invitations.ts`
> （GET/POST `/system/invitations` + `POST …/:id/revoke`，全 `requireSystemAdmin`；
> 签发明文码只在响应出现一次，audit 三动作 invite.create/revoke/consume、明文码绝不
> 进 detail）；`routes/auth.ts` 增公开 `POST /auth/register`（**事务 + 带条件
> UPDATE 原子消费**，并发双注册只有一个赢；过期 410 / 吊销 403 / 无效 401 / 已用
> 409 分路报错；预分配角色行 `granted_by` 落签发人）与 `POST /auth/change-password`
> （旧密码必验、清 `must_change_password`；**不动 pwd_epoch**——P6-5 的机制，提前
> 动 = 无判定端的字段变更）；login 响应带 `mustChangePassword` + 写 `last_login_at`；
> **`/auth/me` 同步带旗**（登录响应里的旗活不过一次刷新，读时从库里来，否则强制改密
> 闸刷新即绕过）。前端：`Register.tsx`（Protected 之外，`?invite=` 预填、注册即登录、
> 服务端分路 message 直接透传）、`ChangePassword.tsx` + `main.tsx` 的 `PasswordGate`
> （`mustChangePassword` 为 true 时除改密页一律弹回；改密页自身在闸外，hydrate 从
> me() 恢复旗）、`Login.tsx` **删掉 admin@local.test / admin123 预填** + 「使用邀请码
> 注册」链接（`.login-card .form-note a` 用 accent）；`UsersPanel` 增邀请码区块
> （签发一行：备注 + 预分配项目/角色（role 必须配项目，后端 400 前端禁按钮）+ 明文
> 注册链接 code-block 只出现一次 + 列表（待使用/已使用/已吊销/已过期四态、被谁消费
> 展示 email）+ 吊销确认）；authStore 增 `register`/`clearMustChangePassword`。
> **一处词汇决策**：email 由注册人自报——计划 10.2 的 body 清单
> `{invite, name, password}` 没有 email，但 `users.email NOT NULL UNIQUE` 且登录按
> email 找行，没有 email 的账号登录不了；归属由签发人带外转交时目视判断（边界 2
> 的原话），冲突 409 而不是枚举既有账号。
>
> **P6-4 实现状态（2026-09-07）**：成员 CRUD + 成员页 + 全站角色分叉落地。**零迁移**
> ——051 的 `user_project_roles.granted_by/granted_at` 已就位。后端：新 `routes/members.ts`
> 四条路由——`GET /projects/:id/members`（`requireProjectAccess` 读，viewer 可看；
> JOIN users 读时快照 + 排序 project_admin→developer→viewer→email）、`POST`
> （requireProjectAdmin，**upsert 语义**：ON CONFLICT 改角色，同一张码加两次不 409
> ——审计动作叫 member.upsert 的原因）、`PATCH …/:userId`、`DELETE`；外加一条计划
> API 清单之外的**候选搜索** `GET …/members/candidates?keyword=`（requireProjectAdmin、
> 只回 id/email/name 三列、只 active、只非成员、上限 20——「搜索平台用户」选择器的
> 数据源；不放宽 `/system/users`，那是全量管理面）。「最后一个 project_admin」闸
> （门槛 8）：降级与移除先数其他 project_admin，数到 0 → 409/2003；POST 改角色同过
> 此闸。审计 `AUDIT_ACTIONS` 增 member.upsert / member.remove（detail 只放 email 与
> 前后角色）。前端：`api.ts` 增 ProjectMember/MemberCandidate 与五个成员 API +
> Project/ProjectMetric 带 `myRole`；`projectStore` 存 `myRole`（null = 未加载，
> 一律按无权限处理）；`hooks/useMyRole.ts` 三个判定（useMyRole / useCanWrite /
> useCanManageProject——**三档不是两档**：developer 能建接口但看不到成员管理）；
> `Members.tsx` 成员页（角色下拉就地改 + 添加成员防抖候选搜索 + 最近审计 50 条复用
> `AuditLogsPanel` 提出的 `AuditDetail`；管理面只对 project_admin 渲染，页面对
> viewer/developer 可读）；导航「成员与权限」落通用组 + `/members` 路由 + i18n 双语。
> **全站隐藏走查**（边界 6「只藏入口不做禁用态、只藏入口不藏路由」）：viewer 藏——
> 接口列表（新建/导入/勾选列/批量条/行内运行·复制·编辑·删除）、工作台（发送/保存/
> 另存用例/复制/删除/⌘S/用例框行内操作/重跑/同步 tab）、环境（新建/编辑/删除/复制
> + 抽屉保存）、公共脚本、流程（列表与画布加节点/运行/保存/节点抽屉调试与删除——
> 抽出共享 `NodeDrawerActions`）、套件（新建/运行/编辑/删除/保存/⌘S/取消他人运行）、
> 数据源（含 SQL 定义与测试连接）、Mock、CI 任务（新建/触发/编辑/删除/保存并触发/
> 凭据就地创建）、告警渠道与规则、MCP Token 签发吊销、仓库用例树勾选执行、凭据页、
> 报告分享创建/撤销、run 取消、调度面板全量入口。**developer 额外收敛到项目管理员**
> （P6-1 已改后端守卫、本批补 UI）：默认环境选择器、明文记录开关、MCP 开关（项目
> 设置写）。编辑器本体（表单/画布）对 viewer 保留——本地草稿存不回去，路由不藏。
>
> **P6-5 实现状态（2026-09-07）**：JWT 过期 + pwdEpoch + 401/403 拦截器全套落地。
> **零迁移**——051 的 `pwd_epoch` 列已就位，`JWT_EXPIRES_IN` 是 env-only。后端：
> `@fastify/jwt` 注册时挂全局 `sign.expiresIn`（`JWT_EXPIRES_IN` 默认 7d；只影响 auth
> 路由的 jwtSign，产物直传的一次性 token 是 objectStore 自己的 HMAC、不走这里）；
> `FastifyJWT` 类型加可选 `pwdEpoch`；新 `lib/auth.ts#signSessionToken` 单点（login /
> register / change-password 三处共用，`expiresAt` 直接从签出的 token 解 `exp`——
> 不自己再解析一遍时长字符串，两处来源迟早漂移）；`currentUser` 的 SELECT 带
> `pwd_epoch`、**不等即 401**——判定端单一（它是全库唯一的 jwtVerify 点，rbac /
> stream 全部经它），改一处全部路由生效。login 的旗查询并进主 SELECT（P6-5 起签
> token 要带 epoch，不再值得单独发一次查询）。**两处自增收口**：`change-password`
> 与管理员 `reset-password` 都 `pwd_epoch = pwd_epoch + 1`（P6-3 / P6-2「提前动 =
> 无判定端的字段变更」的临时取舍就此结束）；改密响应**换发带新 epoch 的 token**——
> 当前会话凭它无缝续命、其他设备上的旧会话全部下线，而不是把刚改完密的人踢去
> 再登一次。前端：`api.ts` 补响应拦截器——401 清 token 带 `returnTo` 硬跳登录
> （硬导航连内存 store 一起重置；不反向 import authStore——会与它 import api 成环）；
> 403 顶部 `message` 提示无权限、**不跳转**（3 秒一条，防一页并发 403 弹五条叠着的
> toast）。**公开页前缀对两个分支都豁免**：`/login`、`/register`（那里的 401/403 是
> 「凭据不对 / 邀请码无效或被吊销」的领域语义，页面有自己的分路报错——重定向会把
> 错误抹掉，全局 toast 又词不达意）与 `/share/*`（公开落地页，匿名访客不该被一枚
> 过期 token 从公开报告页踢去登录）——401 只清不跳，403 不弹全局 toast。
> `authStore` 的 `clearMustChangePassword` 被 `renewSession` 取代（换发
> token 落地 + 清旗一体）；i18n 补 `errors.forbidden` 双语。**一处兼容决策**：缺
> `pwdEpoch` claim 的存量 token 按 0 对齐（`pwd_epoch` 列缺省也是 0）——部署
> P6-5 不强制存量会话全员重登；失效语义只对「之后的改密 / 重置」生效。
>
> **P6 验收结论（2026-09-07，暂记通过）**：P6-1 ~ P6-5 五个批次（迁移与守卫 / 用户
> CRUD 与审计 / 邀请码与强制改密 / 成员管理与角色分叉 / JWT 过期与改密失效）用户
> 验收暂记通过。后续使用中暴露的问题按 `issue_fix/` 流程记录处理（缺陷不入本计划，
> 见主计划 Plan Tracking 纪律）。

### 10.5 验收门槛

1. viewer 看不到任何新建/执行按钮；绕过 UI 直接 POST 得 403。
2. developer 能建接口但不能进成员管理；project_admin 能改角色、能删项目。
3. 被降级的用户下一个请求立即按新角色判定（无缓存窗口）。
4. 禁用用户的 token 下一次请求 401（判定端即时生效）。
5. 邀请码单次使用；过期/吊销的码注册得明确报错而不是 500。
6. 改密后旧 token 全部失效（pwdEpoch），新 token 7 天过期。
7. 管理员建号/重置的初始密码只在响应里出现一次；首登强制改密。
8. 最后一个 project_admin 不能被移除/降级（409 + 提示），防止项目无人能管。
9. 成员与角色的每次变更在审计日志可查到操作人、时间、前后角色。

---

## 十一、P7 — 测试管理（文本用例 + 导入 + 双视图 + 测试计划，约 5 周）

### 11.0 P7 范围与边界（2026-09-04 确认）

**问题**

平台有两种**自动化**用例资产：接口用例（`test_cases`，平台托管、可执行）与仓库用例
（`repo_test_cases`，代码在用户仓库、CI/Runner 执行）。但**测试设计这一层是空的**：
QA 用 XMind/Excel 写的功能用例（目录树 / 标题 / 前置 / 步骤 / 预期）只活在平台外面。
三个直接后果：

1. 「这个模块设计了哪些用例」在平台上无处安放，新人接手只能去翻群文件。
2. 覆盖率只有「接口有没有用例」（`dashboard.ts:139`），**没有「设计的用例里多少已
   自动化」**——这才是 QA 汇报时真正被问的那个数。
3. 一次回归的手工执行结果（通过/失败/阻塞 + 缺陷单）记在个人 Excel 里，平台看不见。

**核心模型**

**文本用例是第三类用例资产：一棵人工维护的模块树，叶子是「用例」（title / 前置 /
步骤 / 预期），它自己不含请求定义、平台不执行它；通过「绑定」（N:N）指向自动化资产
（接口用例 / 流程 / 仓库用例），从而读出自动化状态。测试计划 = 一批文本用例的一次
手工执行记录：结果只来自人工标记；绑定资产的最近自动化结果只读展示、不触发。**

用例的三层词汇（写进 i18n 与用户手册，不再混用）：

| 层 | 平台词 | 表 | 谁执行 |
| --- | --- | --- | --- |
| 设计 | 文本用例 | `spec_cases`（新） | 无人——只被绑定 |
| 自动化（平台托管） | 接口用例 | `test_cases` | 平台 worker |
| 自动化（仓库） | 仓库用例 | `repo_test_cases` | CI 任务 / Runner |

**边界决策（16 项）**

1. **文本用例是独立的第三类资产，不复用 `test_cases`**。后者 `endpoint_id NOT NULL` +
   `request_snapshot JSONB NOT NULL`（`007:9-24`）——文本用例两者皆无，硬塞要么放宽
   约束要么造空壳行，都会污染接口用例的全部既有查询与覆盖口径。
2. **目录树真存储（`parent_id` + 物化 `path`）**。这与 P4「目录不做树的一层」
   （`035:88-90`）**不冲突**：那说的是仓库用例的文件路径不该层级化（代码组织的副产
   品）；这里的树是用户手工组织的**测试设计结构本身**，XMind 导入的就是它。`path`
   物化（`/模块A/子模块/` 前缀匹配）使「模块及全部子级的用例」一条索引查询，不递归
   CTE。树深上限 6，模块名同层可重（身份是 path，不是名字）。
3. **XMind 只解析 `content.json`**（XMind 8+ / Zen 格式，zip 内单文件 JSON）。旧版
   `content.xml` 不支持——遇到时返回 1001 并提示「请用 XMind 8 及以上重新保存」。
   解析用服务端 `lib/zip.ts`（零依赖、三道 zip 炸弹闸，`zip.ts:16-22`）+ `JSON.parse`，
   **不引依赖**。
4. **层级映射默认「叶子即用例」**：叶子节点的祖先链 = 目录路径；可选按标记前缀识别
   （`tc:` / `用例:` 开头的节点视为用例，其子树按约定拆字段）。两种模式在预览页可选、
   有即时预览，**不做自动猜测**。XMind 的 notes → 描述、labels → tags、priority
   marker → priority。
5. **Excel 固定模板，不做列映射**（用户 2026-09-04 确认）。平台提供模板下载；列固定
   为：`目录路径 | 用例编号 | 用例标题 | 前置条件 | 步骤 | 预期结果 | 优先级 | 标签`。
   解析走后端 `exceljs`（**按需 `import()`，不进启动路径**——照 `lib/adapters/index.ts`
   与 `@aws-sdk` 的先例）。Excel 的复杂度（合并单元格 / sharedStrings / 内联串 / 日期
   序列号）远超 junit XML，手写最小读取器在模板是「平台发的契约」这个前提下仍会踩
   WPS/Numbers 另存出的边角，不值。
6. **`.xlsx` 与 `.xmind` 都走 `application/octet-stream` 原始流上传**（文件名与来源放
   query）。照 Runner 产物直传的既有写法（`runners.ts:140` 的 content-type parser +
   路由级 `bodyLimit` 8MB），**不引 `@fastify/multipart`**——两个文件类型、没有多文件
   场景，multipart 的解析边界/临时文件全是新增面。注意 parser 要在本路由插件内注册
   （Fastify 插件封装作用域，runners 那份不跨作用域）。
7. **导入两阶段：preview → commit**。preview 上传文件、解析、返回归一化树 + 逐条
   判定（新增 / 已存在未变 / 已存在有变）+ 冲突摘要；commit 提交用户确认后的归一化
   JSON（前端持有，不落暂存表）。判重键：优先「用例编号」（`item_key`，如
   `TC-001`）；无编号时回落 `(path, title)` 哈希——**改标题会变成新用例**，预览页明
   说这一条。单次上限 2000 条。
8. **绑定是 N:N、多态目标**：一条文本用例可绑多条自动化资产（主流程 + 异常各一条
   很常见），一条自动化资产也可被多条文本用例引用。目标四类：`endpoint`（接口）/
   `case`（接口用例）/ `flow`（流程）/ `repo_case`（仓库用例）。
9. **仓库用例按 `(project_id, case_key)` 绑定，不按行 id**。仓库用例行会被对账标
   `removed`、同 key 重新出现时是同一行还是新行取决于上报序列——`case_key` 才是
   身份（`035:76-77`）。绑定行 `ON DELETE CASCADE` 挂文本用例侧；目标侧读时校验，
   悬空显示「目标已删除」。
10. **绑定接口（`endpoint`）是合法目标**——「这个接口该被测什么但还没自动化」本身
    是有用信息；这产生「设计了但没自动化」与「自动化了」之间的中间态，正好喂
    覆盖口径。
11. **新增第三个覆盖口径：自动化覆盖率 = 有 ≥1 个绑定（`endpoint` 不算自动化）的
    active 文本用例 / 全部 active 文本用例**。与接口覆盖率（接口有无用例）、仓库
    覆盖率（形状去重）**并列展示、互不合并**——三个口径回答三个问题，页面上各配
    一句话说明，避免「覆盖率」一词裸奔。
12. **测试计划只记手工结果**（用户 2026-09-04 确认）：`passed / failed / blocked /
    skipped / 未执行` + 备注 + 缺陷链接（纯文本 URL，不做缺陷系统）。**绑定资产的
    最近一次自动化结果作为只读参考列展示**（状态 + 时间 + 深链），不参与计划通过率、
    不自动回写。
13. **「按绑定触发自动化」整体后置**（用户 2026-09-04 确认：「代价可能有点大，放到
    待办不着急实现，后面有人提需求再说」）。记录设计取向以防将来跑偏：**不发明第三
    条执行路径**——触发时复用既有两个扇出器（平台侧绑定 → 生成 manual 套件或临时
    入队；仓库侧绑定 → `triggerCiTask({caseKeys})`），计划只记 `generated_suite_id`
    供跳回。
14. **计划通过率口径对齐平台标准**：`passed/(passed+failed)`，`blocked / skipped /
    未执行` 不进分母（8.17 ② 同款）；进度 = 已标记 / 总数。
15. **不做 XMind 导出回写**——回写要处理「平台改了、源文件也改了」的合并冲突，那是
    另一个产品；分享用平台链接或计划导出（CSV 起步）。
16. **不做用例评审流 / 版本历史**。状态就 `draft / active / deprecated` 三值；「谁改
    的」走 `updated_at` + 审计，逐版本 diff 与回滚归 P10 14.3（与本节「接口变更追踪」
    是同一件事的两个面，分两处做会长出两套版本模型）。

### 11.1 数据库迁移：052_p7_spec_cases.sql（四张表）

```sql
CREATE TABLE IF NOT EXISTS spec_modules (
  id UUID PRIMARY KEY,
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  parent_id UUID REFERENCES spec_modules(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  path TEXT NOT NULL,                      -- 物化：'/模块A/子模块/'；根节点 path = '/'
  position INTEGER NOT NULL DEFAULT 0,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (project_id, path)
);
CREATE INDEX IF NOT EXISTS spec_modules_parent_idx ON spec_modules (parent_id);

CREATE TABLE IF NOT EXISTS spec_cases (
  id UUID PRIMARY KEY,
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  module_id UUID NOT NULL REFERENCES spec_modules(id) ON DELETE CASCADE,
  item_key TEXT,                           -- 用例编号（TC-001）；NULL = 无编号（回落 path+title 哈希）
  title TEXT NOT NULL,
  preconditions TEXT NOT NULL DEFAULT '',
  steps JSONB NOT NULL DEFAULT '[]',       -- [{ step, expected }]
  priority TEXT NOT NULL DEFAULT 'P2' CHECK (priority IN ('P0','P1','P2','P3')),
  tags TEXT[] NOT NULL DEFAULT '{}',
  status TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('draft','active','deprecated')),
  source TEXT,                             -- 'xmind' | 'excel' | 'manual'
  created_by UUID REFERENCES users(id) ON DELETE SET NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (project_id, item_key)
);
CREATE INDEX IF NOT EXISTS spec_cases_module_idx ON spec_cases (project_id, module_id);
CREATE INDEX IF NOT EXISTS spec_cases_status_idx ON spec_cases (project_id, status);

CREATE TABLE IF NOT EXISTS spec_case_links (
  id UUID PRIMARY KEY,
  spec_case_id UUID NOT NULL REFERENCES spec_cases(id) ON DELETE CASCADE,
  target_type TEXT NOT NULL CHECK (target_type IN ('endpoint','case','flow','repo_case')),
  target_id UUID NOT NULL,                 -- repo_case 目标额外存 case_key 快照（边界 9）
  target_key TEXT,                         -- 仅 repo_case：(project 内的 case_key)
  created_by UUID REFERENCES users(id) ON DELETE SET NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (spec_case_id, target_type, target_id)
);

CREATE TABLE IF NOT EXISTS test_plans (
  id UUID PRIMARY KEY,
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  description TEXT NOT NULL DEFAULT '',
  status TEXT NOT NULL DEFAULT 'open' CHECK (status IN ('open','closed')),
  owner UUID REFERENCES users(id) ON DELETE SET NULL,
  started_on DATE,
  due_on DATE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE IF NOT EXISTS test_plan_items (
  id UUID PRIMARY KEY,
  plan_id UUID NOT NULL REFERENCES test_plans(id) ON DELETE CASCADE,
  spec_case_id UUID REFERENCES spec_cases(id) ON DELETE CASCADE,
  title_snapshot TEXT NOT NULL,            -- 计划是历史：用例改名后旧计划仍读得通
  module_path_snapshot TEXT NOT NULL,
  assignee UUID REFERENCES users(id) ON DELETE SET NULL,
  result TEXT CHECK (result IN ('passed','failed','blocked','skipped')),  -- NULL = 未执行
  note TEXT NOT NULL DEFAULT '',
  defect_url TEXT,
  executed_at TIMESTAMPTZ,
  UNIQUE (plan_id, spec_case_id)
);
CREATE INDEX IF NOT EXISTS test_plan_items_assignee_idx ON test_plan_items (assignee) WHERE result IS NULL;
```

### 11.2 后端新 API

```
GET    /api/v1/projects/:id/spec-modules            树全量（模块 + 每模块用例计数）
POST   /api/v1/projects/:id/spec-modules            建模块
PATCH  /api/v1/projects/:id/spec-modules/:moduleId  改名/移动（子树 path 批量重算）
DELETE /api/v1/projects/:id/spec-modules/:moduleId  扫引用，有用例则 409（force 语义同环境）
GET    /api/v1/projects/:id/spec-cases              服务端分页 + path/keyword/priority/status/绑定态筛选
POST   /api/v1/projects/:id/spec-cases              手工建（写权限）
GET    /api/v1/projects/:id/spec-cases/:caseId      详情（含绑定 + 最近自动化结果读时关联）
PATCH  /api/v1/projects/:id/spec-cases/:caseId
DELETE /api/v1/projects/:id/spec-cases/:caseId
POST   /api/v1/projects/:id/spec-cases/:caseId/links     绑定 {targetType, targetId}
DELETE /api/v1/projects/:id/spec-cases/:caseId/links/:linkId
POST   /api/v1/projects/:id/spec-cases/import/preview   octet-stream 上传 → 解析 + 判定，不落库
POST   /api/v1/projects/:id/spec-cases/import           提交归一化 JSON（上限 2000 条）
GET    /api/v1/projects/:id/test-plans              列表 + 状态/owner 筛选 + 进度汇总
POST   /api/v1/projects/:id/test-plans              建（可带初始 specCaseIds，title 快照）
GET    /api/v1/projects/:id/test-plans/:planId      详情（项 + 只读自动化参考列）
PATCH  /api/v1/projects/:id/test-plans/:planId
POST   /api/v1/projects/:id/test-plans/:planId/items        加项
DELETE /api/v1/projects/:id/test-plans/:planId/items/:itemId
PATCH  /api/v1/projects/:id/test-plans/:planId/items/:itemId  标结果/指派/备注/缺陷链接
GET    /api/v1/projects/:id/test-plans/:planId/export       CSV（UTF-8 BOM，Excel 直开）
```

`GET /spec-cases/:caseId` 的「最近自动化结果」读时关联：按绑定目标分别查
`executions`（case/flow，取 `case_id`/`flow_execution_id` 最新一条）、`repo_test_cases.
last_result`（P4 已有列，`035:102`）。**绝不回写**（照迁移 047 树×任务报告的纪律）。

### 11.3 前端

- **导航新增「测试管理」分组**（编排与回归之前）：两个导航项——「用例库」
  （`/spec-cases`）与「测试计划」（`/test-plans`）。`ProjectShell.tsx` 三处同改 +
  `main.tsx` 路由 + `pageTitle` kebab 特例补两行。
- **用例库**：单一页面 = 筛选栏 + **树/列表视图切换**（`?view=tree|list` 进 URL）。
  - 树视图：目录树（复用/泛化 `StepTree`——`PipelineRunPage.tsx:956` 已是可复用递归
    树，`RepoCaseTree.tsx:11` 在用）+ 展开态 + 勾选（勾选集给批量条：加入计划 /
    移动模块 / 批量改优先级 / 删除）。**不是思维导图画布**——XMind 的价值在层级与
    字段拆解，可编辑脑图画布（自由布局、拖拽改父、就地改字）是另一个量级的产品，
    且 `@xyflow/react` 的自由画布与「树」的约束布局是两种东西。
  - 列表视图：`.grid` 扁平表 + 服务端分页（20/50/100）+ 目录路径列。
  - 两个视图**共享同一份筛选与勾选状态**（切换视图不丢勾选——「选中后建计划」的
    主路径跨视图）。
- **用例详情抽屉**：字段编辑 + 步骤表（行内两列：步骤/预期）+ **绑定区**（按
  target_type 分四小节，各配一个选择器：接口选择器复用 `SuiteMemberPicker` 的接口
  分组数据源、用例选择器复用其用例分组、流程平铺、仓库用例复用 `RepoCaseTree` 的
  行数据源）。
- **导入向导**（Modal 三步）：上传（`.xmind` / `.xlsx` 二选一 + 模板下载）→ 预览
  （树形勾选 + 逐条三态 + 冲突摘要 + XMind 层级映射模式选择）→ 提交（结果
  `message` 汇总新增/更新/跳过）。
- **测试计划**：列表（名称 / 状态 / 进度条 / owner / 截止）+ 详情页（按模块分组
  的项表：结果下拉就地标 + 指派 + 备注 + 缺陷链接；**自动化参考列**只读，含深链；
  顶部进度与通过率 `Readout`）。导出 CSV 走 `GET .../export`。

### 11.4 实施顺序（P7-1 … P7-7）

| 步 | 内容 | 产出 |
| --- | --- | --- |
| P7-1 | 迁移 052 + 模块树 CRUD（含移动的 path 重算） | 树能建能改 |
| P7-2 | 用例 CRUD + 列表分页筛选 + 详情抽屉 | 手工建用例可用 |
| P7-3 | 绑定（四类目标 + 读时关联最近结果）+ 覆盖口径 | 绑定与「自动化覆盖率」可见 |
| P7-4 | XMind 解析（`lib/zip.ts` + content.json）+ preview/commit | 一份真实 XMind 导入成功 |
| P7-5 | Excel 固定模板解析（exceljs 按需导入）+ 模板下载 | 模板往返（下载→填→导入）闭环 |
| P7-6 | 树视图 + 视图切换 + 勾选批量条 | 双视图 + 跨视图勾选 |
| P7-7 | 测试计划全套 + CSV 导出 + i18n 双语 | 计划建/标/看进度可用 |

### 11.5 验收门槛

1. 一份 300 条用例的真实 XMind：模块层级与源文件一致、`tc:`/`pc:` 字段拆解正确、
   标记与备注进对应字段。
2. 同一份 XMind 导入两次：第二次预览显示「已存在未变」为主，无重复行。
3. 无编号用例改标题再导入：预览明确提示「将创建新用例」（判重键回落语义可预期）。
4. Excel 模板下载→填写→导入闭环；目录路径用 `/` 分隔多级。
5. 上传超过 8MB / 非白名单扩展名 / zip 炸弹：明确报错，不是 500 或静默。
6. 绑定一条接口用例后，用例列表「已自动化」筛选与项目自动化覆盖率读数一致。
7. 仓库用例被对账标 removed 后，绑定显示「目标已失效」而不是消失或报错。
8. 勾选（跨树/列表视图）→ 创建计划 → 计划项带 title/module 快照；此后改用例标题，
   旧计划显示快照。
9. 计划通过率 = passed/(passed+failed)；blocked/skipped/未执行不进分母。
10. 计划详情的「自动化参考列」与被绑定资产的实际最近结果一致，且点击可跳转。
11. 树视图 2000 条用例不卡（模块计数聚合在服务端，不前端全量拉用例）。
12. CSV 导出用 Excel 打开中文不乱码（UTF-8 BOM）。

### 11.6 后置（记录取向，不排期）

- **按绑定触发自动化**（用户 2026-09-04：待办，有人提需求再做）：取向见边界 13——
  复用套件 / CI 两条既有扇出器，计划只记 `generated_suite_id`。
- 用例评审流（草稿→评审→生效的状态机 + 通知）。
- 逐版本 diff 与回滚 → P10 14.3 一并做。
- XMind 导出回写。

---

## 十二、P8 — 数据统计（全局/项目趋势 + 失败归因 + 下钻，约 4 周）

### 12.0 P8 范围与边界（2026-09-04 确认）

**问题**

平台有三处统计、各说各话，且只有「当下」没有「变化」：dashboard 是项目卡片 + 四个
比率（读时聚合）；trend 是单项目 24h/7d/30d（`reports.ts:153`，样本 `execution_index`）；
summary 是 24h 失败分布（`reports.ts:204`）。具体缺口：

1. **没有全局（跨项目）趋势**——trend 挂在 `requireProjectAccess` 下（`reports.ts:154`），
   单项目作用域是 RBAC 最小单位（既有明确决策）；总报告列表的 UNION SQL 两侧写死
   `project_id = $1`（`reports.ts:288,307`）。
2. **失败归因为零**——现在只有读时四分类（assertion/script/timeout/request，
   `reports.ts:230-243`），只覆盖 `executions`；CI 用例失败（`pipeline_run_cases.message`）、
   上报失败、套件父级失败完全不参与。「这周失败里多少是环境问题」答不出来。
3. **无新增趋势**——资产（接口/用例/流程）只有当前总量，没有「本周新增多少」。
4. **无下钻**——趋势图无 hover、无点击（交互文档 6.2d 早已预留 canonical URL 表但从未
   接线；`executions/by-index/:indexId` 统一入口也未实现）。

**既有缺陷（本阶段 P8-1 一并收口，缺陷本身记 `issue_fix/`）**

| # | 缺陷 | 锚点 |
| --- | --- | --- |
| ① | 通过率三处口径分叉：`success/total`（skipped 进分母）与 `success/(s+f+skipped)` 两种旧算法仍在跑 | `lib/reportPayload.ts:22-23`（报告详情 + 分享页）、`lib/alerts.ts:334`（通知模版）；`issue_fix/问题记录-P4.5验收20260903第二批.md:53-60` 列了三个落点、漏了这两处 |
| ② | 单步调试自指索引行污染趋势样本——列表排除了（`executionIndex.ts:23` 的 `detail_id <> id`），趋势与汇总没排除 | 写入在 `flows.ts:84-88`；漏过滤在 `reports.ts:176,216` |
| ③ | `includeRepo` 只作用于 summary 第一段（总量读 `execution_index`），失败分布两段读 `executions`（无 ingest/runner 数据）——打开开关后总量变、分布纹丝不动 | `reports.ts:59-66` vs `:221-243` |

**核心模型**

**先把口径收口到一个模块（`lib/metrics.ts`），再叠加全局维度、下钻能力与人工归因。
归因是用户填的一等数据（不是从 error 文本推定的读时分类）；自动归因经 MCP 由外部
agent 写回。图表继续手写 SVG，提成可复用组件并补 hover/刻度/下钻。**

**边界决策（18 项）**

1. **P8-1 是纯收口步，先于一切新图表**（照 P5-2「重构独立成步」的先例）：三处口径
   全改 `passed/(passed+failed)`；趋势/汇总补 `detail_id <> id`；`includeRepo` 的一致性
   在新归因统计上直接做对（旧 summary 的②③段随新页替换退役）。独立成步的理由：
   口径修复与新增代码混在一批，数字变了分不清是「修对了」还是「新 bug」。
2. **口径三件事收敛到 `lib/metrics.ts`**：通过率公式、失败样本集（status 过滤 +
   自指排除 + `includeRepo` 语义）、资产新增口径。导出 **SQL 片段常量**（不是函数），
   嵌进各处 SQL——`PASS_RATE_SQL = "(CASE WHEN (passed + failed) > 0 THEN ROUND(passed::numeric*100/(passed+failed),1) END)"` 这类字符串常量比参数化抽象好读，这是
   SQL 拼接纪律（只有代码片段可拼接）允许的形态。
3. **失败归因是人工标注的落库数据**，与读时四分类并存、分层展示：读时分类是
   「失败长什么样」（客观、粗筛），归因是「为什么失败」（判断、可统计）。归因对象
   是失败的最小可读单元，三类目标（PK 都是 UUID，多态一表）：
   - `executions`（接口用例/流程步骤的 HTTP 失败）
   - `pipeline_run_cases`（CI 用例失败，status IN ('failed','error')）
   - `execution_steps`（套件/流程的非 HTTP 步骤失败）
4. **不在父级（run/suite execution）上归因**——一次 run 挂 20 条用例可能三个原因，
   父级打一个标签会把统计做假。但**批量归因是一等交互**（一次 CI 挂一片同因失败太
   常见）：报告页支持勾选多条失败一次打标。
5. **归因分类法平台级一份、7 类固定、项目不可改**（用户 2026-09-04 确认）：
   产品缺陷 / 用例缺陷 / 环境问题 / 数据问题 / 不稳定（flaky）/ 外部依赖 / 其他。
   归因统计的价值在跨项目可比（「全平台 40% 失败是环境问题」），项目各自定义会让
   全局饼图无法聚合。项目特有原因写进 `note` 自由文本。**升级判据**：某项目 60%
   归因落在「其他」且 note 内容重复时，再议项目级扩展。
6. **一条失败只有一个归因**：`UNIQUE (target_type, target_id)`。重打标 = UPDATE。
   保留 `created_at`/`updated_at`/`created_by`，不做归因历史版本。
7. **未归因不入库（没有行 = 未归因）**，因此「归因覆盖率」是可推动的指标。**任何
   归因占比图必须同屏显示归因覆盖率**——一张「40% 环境问题」的饼图在只归因了 10%
   失败时没有意义。这是本阶段最重要的展示纪律。
8. **自动归因走 MCP，平台不写一行 LLM 代码**（用户 2026-09-04 确认；P5 边界 16 的
   自然延伸——平台不调模型，模型调平台）：新增两个 MCP 工具
   `list_unattributed_failures`（读，带证据摘要）与 `set_failure_attribution`（写，
   复用 REST 同一套校验）。外部 agent 读证据 → 自行判断 → 写回，`created_by` 落
   `mcp_token.created_by`（**真人判据照旧**，P5 边界 2）。来源标 `source: 'mcp' | 'human'`，
   统计页可按来源筛——回答「这个分类是 AI 打的还是人打的」。**不建审核队列**：agent
   归因视为事实、可被人覆盖；审核流是没人会清理的债务。
9. **归因入口两处**：报告详情页失败行（CI 用例行、套件成员行）与执行记录列表的
   失败行——点击开小抽屉：分类单选 + note + 证据摘要（状态码/断言失败项/error 原文
   只读展示）。成功/取消的行不出现归因入口。
10. **全局统计走新接口 `/api/v1/stats/*`**，不改旧项目级接口的形状——旧的挂在
    `requireProjectAccess` 下（权限是它的形状的一部分）；全局接口自己 `currentUser()`
    + 可见项目集。**旧接口保留**（趋势页在用），新页面上线后前端切换，旧路由退役
    列入收尾批次。
11. **可见项目集在 SQL 里过滤**：非系统管理员追加
    `EXISTS (SELECT 1 FROM user_project_roles upr WHERE upr.project_id = ei.project_id
    AND upr.user_id = $n)`——照 `dashboard.ts:29-32` 的写法，**不做**「Node 里查项目
    列表再 `IN ($1…$50)`」（项目多时占位符爆炸）。
12. **时间粒度扩到 `1h/24h/7d/30d/90d` + 自定义 from/to**（上限 180 天）。桶粒度由
    服务端按跨度定：≤48h 小时桶、≤60d 天桶、更长周桶；跨度与桶仍是**代码常量**插
    SQL（请求值绝不进 interval 字符串）。
13. **下钻用 URL query，不新增路由**（交互 6.2d 的约定表）：
    - 趋势折线某桶 → `/projects/:pid/reports?status=failed&from=<bucket>&to=<bucket+1>`
    - 归因饼图某片 → 报告列表 + `attribution=<category>` 筛选
    - 榜单行 → 资产/报告详情深链
    - **列表接口补 `from`/`to` 参数**是下钻的前置（现在执行记录与报告列表都没有
      时间窗参数）——本阶段交付。
    - 交互 6.2d 表里预留的 `/executions/by-index/:indexId` 统一执行入口**一并实现**
      （按来源重定向到三个既有详情路由），统计侧从此不感知来源路由。
14. **图表继续手写 SVG**（用户 2026-09-04 确认）：把 `Trends.tsx` 的三个私有函数提
    成 `components/charts/` 可复用组件 + 从 `TimelineGantt` 抽共享的
    `niceStep`/网格/hover 交互。**hover 的标准做法**（参考项目见 12.6）：在图上盖一层
    透明 `<rect>`（`pointer-events: all`）捕 `onMouseMove`，用 `offsetX / plotWidth`
    反查最近数据桶索引——**不要逐 path 命中**（用户得精确指到 1.5px 宽的线上）；
    读数条**常驻高度**（照 `PipelineRunPage.tsx:682-689` 的 timeline-hover 模式，条件
    渲染会把下面的表格顶一次）；tooltip 用 HTML 绝对定位而不是 SVG `<text>`
    （非整数缩放下 SVG 文本发虚，`Trends.tsx:30-32` 的既有结论）。
15. **不做图表缩放/刷选/图例交互**——时间窗由顶部时间选择器控制，两套控制会打架；
    图例只读（色块 + 名称）。
16. **统计页不实时**：进入拉一次 + 手动刷新按钮。不接 SSE——统计是聚合读，接事件
    流意味着每个终态重算聚合，把读页变订阅端。
17. **不建汇总表/物化视图**（第一版）：全平台读时实时 SQL 是既定姿态，`execution_index`
    上 `(project_id, created_at DESC)` 索引在（迁移 015/030）。90 天全局趋势先直查，
    **实测单次 > 1.5s 才立项汇总**——物化视图带来「何时刷新/刷新失败/口径变更重建」
    三个新问题，而冷数据该先走 P10 14.1 的归档分区。判据写死，不靠感觉。
18. **看板扩展为「当下 + 变化」**：既有四个比率旁加 7 天 delta（**用百分点 pp 而不是
    百分比**——「从 80% 到 82%」是 +2pp 不是 +2.5%）与 30 天 sparkline；**全局层不再
    多开一个统计页**——看板就是全局统计页（加 sparkline 与 delta 后），项目层「趋势
    分析」扩成「数据统计」（三段式，见 12.5）。

### 12.1 数据库迁移：053_p8_attribution.sql

```sql
CREATE TABLE IF NOT EXISTS failure_categories (
  code TEXT PRIMARY KEY,                   -- 'product_bug' / 'test_defect' / 'env' / 'data'
                                           -- / 'flaky' / 'dependency' / 'other'
  name TEXT NOT NULL,
  position INTEGER NOT NULL,
  active BOOLEAN NOT NULL DEFAULT true
);
INSERT INTO failure_categories (code, name, position) VALUES
  ('product_bug', '产品缺陷', 1),
  ('test_defect', '用例缺陷', 2),
  ('env', '环境问题', 3),
  ('data', '数据问题', 4),
  ('flaky', '不稳定', 5),
  ('dependency', '外部依赖', 6),
  ('other', '其他', 7)
ON CONFLICT (code) DO NOTHING;             -- 种子是字典数据，幂等

CREATE TABLE IF NOT EXISTS failure_attributions (
  id UUID PRIMARY KEY,
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  target_type TEXT NOT NULL CHECK (target_type IN ('execution','pipeline_run_case','execution_step')),
  target_id UUID NOT NULL,                 -- 多态、无 FK（三类目标删除语义各异，悬空=未归因，读时排除）
  category TEXT NOT NULL REFERENCES failure_categories(code),
  note TEXT NOT NULL DEFAULT '',
  source TEXT NOT NULL DEFAULT 'human' CHECK (source IN ('human','mcp')),
  created_by UUID REFERENCES users(id) ON DELETE SET NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (target_type, target_id)
);
CREATE INDEX IF NOT EXISTS failure_attributions_project_idx
  ON failure_attributions (project_id, category);
CREATE INDEX IF NOT EXISTS failure_attributions_created_idx
  ON failure_attributions (created_at DESC);
```

`created_at` 冗余记归因时间（非失败时间）——按时间看「归因行为」用；「按失败发生时间
的分布」join 目标表的 `created_at`。

### 12.2 后端新 API

```
# 归因（项目级，REST）
GET    /api/v1/projects/:id/failure-categories          登录即可读（字典）
POST   /api/v1/projects/:id/failure-attribution         {targetType, targetId, category, note}（写权限）
DELETE /api/v1/projects/:id/failure-attribution?targetType=&targetId=   清除归因
POST   /api/v1/projects/:id/failure-attribution/batch   批量：[{targetType, targetId}] + 同一 category/note

# 全局统计（新路由组，无项目前缀）
GET    /api/v1/stats/overview?days=30            全局读数（执行量/通过率/失败量/归因覆盖率/各资产总量）+ 7 天 delta
GET    /api/v1/stats/trend?metric=&days=&projectIds=   execution_trend（量/通过率/分位）| asset_trend（新增）
GET    /api/v1/stats/failures?days=&projectIds=  失败归因分布（含未归因）+ 失败 TopN（按接口/用例/CI 任务/套件四维）
GET    /api/v1/stats/flaky?days=&projectIds=     不稳定用例榜（见边界 19 的判定）
GET    /api/v1/stats/coverage?days=              三个覆盖率口径的 30 天序列
GET    /api/v1/stats/projects?days=              项目对比表（执行量/通过率/失败/归因覆盖率/最近执行）

# 既有接口的增量
GET    /api/v1/projects/:id/executions           补 from/to 时间窗参数（下钻前置）
GET    /api/v1/projects/:id/suite-reports        补 from/to + attribution 筛选
GET    /api/v1/projects/:id/executions/by-index/:indexId   统一执行入口（按来源重定向）

# MCP 工具（挂 P5 工具面，read + write scope）
list_unattributed_failures（读，分页，带证据摘要）
set_failure_attribution（写，复用上面 REST 的校验）
```

### 12.3 新增指标清单（按「能回答什么问题」筛）

| 指标 | 回答 | 来源 |
| --- | --- | --- |
| 失败归因分布 + 覆盖率 | 失败都是什么原因、还有多少没归因 | 新表 |
| 通过率/失败量趋势（全局 + 可按项目筛） | 整体质量在变好还是变坏 | `execution_index` |
| 资产新增趋势（接口/接口用例/流程/套件/文本用例/仓库用例） | 平台在长吗 | 各表 `created_at`（`removed` 的仓库用例仍计入——衡量的是「新增行为」不是存量） |
| Flaky 用例榜 | 哪些用例一会儿绿一会儿红 | `executions` 按 `case_id` 序列翻转计数 + `repo_test_cases` 的 SDK 上报序列 |
| 失败 TopN（接口/用例/CI 任务/套件四维） | 先修哪个 | 扩展既有 `reports.ts:221`（只有按 endpoint 一维） |
| 排队时长趋势 | 要不要加 worker/Runner | `started_at - created_at`（报告列表已在算 `queue_ms`） |
| 触发源分布（manual/scheduled/webhook/ci） | 自动化真的在自动跑吗 | `execution_index.trigger_source`（零成本） |
| 三个覆盖率口径序列 | 测试的底座在变大吗 | 接口覆盖（dashboard 口径）/ 仓库覆盖（形状去重）/ 自动化覆盖（P7 口径） |
| MTTR（失败→首次转绿） | 失败有人跟吗 | 同 Flaky 的序列查询 |
| 首次通过率 | 新用例质量 | 同上 |
| 计划进度 / 通过率 | 这轮回归到哪了 | P7 的 `test_plan_items` |
| **不做**：人员效能统计 | — | 容易沦为 KPI 工具、扭曲归因行为 |

Flaky 判定（写死在 `lib/metrics.ts`，与通过率同处口径库）：窗口 30 天内同一
`case_id`（或仓库 `case_key`）≥6 次执行、状态翻转 ≥3 次、passed 与 failed 各 ≥2 次。

### 12.4 前端

- **项目层**：「趋势分析」页（`trends`）改名「数据统计」（路由保留 `trends` 不换——
  深链不断，导航文案改）。三段式：`Readout` 行（执行量/通过率/失败量/归因覆盖率
  + delta）→ 趋势区（折线 + 堆叠柱）→ 归因与榜单区（环形 + TopN 表 + Flaky 榜）。
  顶部筛选：时间窗 / includeRepo / 触发源。
- **全局层**：数据看板（`/dashboard`）按边界 18 扩展——比率卡 + sparkline + delta，
  项目表加「归因覆盖率」列；不加第二个统计页。
- **`components/charts/`**：`LineChart`（多 series、null 断线）、`StackedBars`、
  `Donut`、`Sparkline`、共享 `niceStep` 与 hover 层（透明 rect + 常驻读数条 +
  HTML tooltip）。`Trends.tsx` 与 `TimelineGantt` 改为消费这套——三份画图逻辑收敛成
  一份。
- **下钻**：图表 `onClick` 按 12.2 的 URL 表跳转；时间窗/筛选进 URL（1.2.1 服务端
  检索铁律）。

### 12.5 手写 SVG 的参考项目（用户问的「悬浮折线统计」）

按可读源码程度排序，**只参考交互模式，不搬代码**：

1. **uPlot**（leeoniya/uPlot）——光标实现是「透明 overlay + 最近点吸附 + 十字线 +
   图例读数」的教科书，`cursor.ts` 单文件可读；它 ~50KB 的体积与零依赖证明这条路
   不需要库。
2. **react-sparklines**（borisyankov/react-sparklines）——sparkline 的最简范本
   （数百行）；我们的 `Sparkline` 组件照这个复杂度做。
3. **visx**（airbnb/visx）的 `@visx/tooltip` + `@visx/axis`——「SVG 定位 + HTML
   tooltip（portal 到 SVG 外，避免被 overflow 裁剪）」「ticks 的 nice 算法」两件事
   的标准解法；只看这两个包，不引整个 visx。
4. **Recharts** 的 `Tooltip` + `ActiveDot` 源码——「active 点放大 + 竖直参考线」的
   常规做法，React 实现与我们技术栈同构。
5. **Grafana timeseries 面板**（行为参考，源码太重）——十字线、最近点高亮、常驻
   读数框的交互行为即 12.4 hover 层的目标形态。
6. **站内已有范本**：`PipelineRunPage.tsx` 的 `TimelineGantt`（刻度 + 网格 + 常驻
   hover 读数）与 `Trends.tsx`（null 断线、HTML 轴标签）——组件就基于它们抽象。

### 12.6 实施顺序（P8-1 … P8-7）

| 步 | 内容 | 产出 |
| --- | --- | --- |
| P8-1 | 口径收口（三处通过率 + 自指排除 + includeRepo 一致性），纯重构 | 既有页面数字一致；缺陷记 issue_fix |
| P8-2 | 迁移 053 + 分类字典 + 归因 REST（单条/批量/清除） | 归因可打可查 |
| P8-3 | 归因前端入口（报告页失败行 + 执行记录失败行 + 批量抽屉） | 人工归因闭环 |
| P8-4 | `/stats/*` 六条接口 + 可见项目集过滤 + 列表接口 from/to | 全局数据可取 |
| P8-5 | `components/charts/` 抽象 + hover/刻度 + Trends/TimelineGantt 接入 | 一套图表底座 |
| P8-6 | 项目层「数据统计」三段式 + 看板 sparkline/delta + 下钻接线 | 页面可用 |
| P8-7 | MCP 两个归因工具 + `source` 筛选 + by-index 统一入口 + i18n | agent 归因闭环 |

### 12.7 验收门槛

1. 同一次套件执行：报告列表、报告详情、分享页、通知模版四处通过率一致
   （passed/(passed+failed)）。
2. 单步调试不再出现在趋势样本里（对比 P8-1 前后的失败率读数）。
3. 一条失败归因后，全局分布图立即变化；未归因占比始终可见。
4. 归因覆盖率低于 100% 时，分布图上有明确的「未归因 N 条」提示。
5. 批量归因 20 条失败 ≤ 2 次点击 + 1 次提交。
6. 非系统管理员的全局统计只含其可见项目；URL 直访别人的项目数据 403。
7. 趋势折线 hover：十字线 + 常驻读数（时间/通过率/量）；点击跳转列表且时间窗正确。
8. 90 天窗口出图 < 1.5s（超了就触发边界 17 的汇总表立项判据）。
9. Flaky 榜与人工观察一致（构造一条翻转用例验证判定）。
10. MCP `set_failure_attribution` 写回后 `source='mcp'`；人工覆盖后变 `human`。
11. 零新图表依赖（`package.json` 无 echarts/recharts/antv）。

---

## 十三、P9 — 站内助手（人物 + 聊天 + 新手教程 + 站内通知，约 4 周）

### 13.0 P9 范围与边界（2026-09-04 确认）

**定位**

主计划 9.8 的「后置：站内助手」落成正式阶段。方向不变：**平台不实现 agent、不管
session / token / function call**，链路是

```
浏览器 → 平台助手代理（只转发 SSE）→ 外部 agent 平台（Nuwax）→ 平台 /mcp → 库
```

平台在这条链里出现两次：哑管道 + 工具提供方。

**新增范围**（用户 2026-09-04 确认）：**站内通知并入本阶段**——通知中心（铃铛小红点
+ 弹窗提醒）是助手壳的一部分；P6/P7 落地的成员变更、计划指派在此回补通知生产端。

**后置增补之二**（用户 2026-09-07 提出）：**项目可见性两层 + 权限申请审批流**——非
项目成员在项目列表**看得到**项目但**进不去**（发现层开放、内容层不变）；站内信与人物
落地后，用审批流自助申请权限。取向与现状锚点见 13.6（记录取向，不排期）。

**上游能力核对**（`智能体平台-第三方接入接口文档.md`，2026-09-04 引入）：

| 9.8 的前置疑问 | 文档答案 |
| --- | --- |
| HTTP API 支持流式？ | ✅ `POST /api/v1/chat/{conversationId}` SSE，事件 `MESSAGE` / `PROCESSING` / `FINAL_RESULT` / `HEART_BEAT` / `ERROR` |
| 能接受外部传入会话 id？ | ⚠️ 不传 id，但会话由平台创建（§2.1）且历史可查（§2.5）——平台侧记 `conversationId ↔ (project, user)` 映射即可，无需自存 transcript |
| 能连任意外部 MCP Server？ | ✅ `STREAMABLE_HTTP` 安装方式（§3），P5 的无状态 `/mcp` 走兼容姿态可接 |
| 能按会话注入凭据？ | ❌ 未提及（API Key 平台级绑定智能体，§鉴权）——**L3 定案走 proposal-first**，P5-3 的该实测项降为「确认有无动态 Header 能力，有则可升级」 |
| 能转述 elicitation / MRTR？ | ❓ 文档无此概念；按「不能」处理——delete 在聊天里降级为「请到页面上操作」提示（P5 9.4.1 的降级分支） |

**边界决策（12 项）**

1. **助手代理归一事件形状，不透传上游**：`MESSAGE` 的 token 流原样拼；`PROCESSING`
   映射为「正在调用 list_endpoints…」的工具行（**用户能看见 AI 在调什么工具——信任
   的关键**）；`HEART_BEAT` 不转发，代理自己按 15s 发 SSE 注释行保活（长空闲连接
   防断，`routes/stream.ts:43` 的既有写法）；`ERROR` 映射为一条错误气泡 + 结束帧。
   上游加字段/改字段名时不让前端跟着改。
2. **对话正文不落库**（9.8 既定）：transcript 在 agent 平台侧（`GET …/messages` 可
   查），平台只存 `assistant_conversations` 映射行。理由照旧——再存一份等于把可能含
   业务数据的文本抄成两份。
3. **L3 身份绑定定案 proposal-first**（9.8 三选一的第 3 项）：聊天用的 MCP Token 只
   发 `read` scope；写操作由 agent 产出**草稿**（proposal），前端渲染成预览卡，用户
   点「确认应用」时**用自己的 JWT 调既有 REST**。零新增写库路径；`canAccess` 与审计
   全程落在真人。上游无按会话注入凭据能力（见上表），选项 1/2 的前提不成立。
   `ResponseScriptEditor.tsx:20` 的受控组件形态是草稿落点。
4. **上游配置全局一份**：`assistant` 配置块（baseUrl + agentId + apiKey，Key 走
   `lib/crypto.ts:19` AES-GCM——要拿明文调上游，所以加密而非哈希）放**系统管理**
   （不是 project_settings——上游 Key 与 agentId 是平台级运维资产，每个项目一份是
   错的抽象层级）；项目级只有 `project_settings.assistant_enabled`（默认 false，
   照 P5 边界 12 的 `mcp_enabled` 同款：服务端真不服务，不是藏入口）。
5. **人物：纯 Canvas 2D，种子 `users.id`**（9.8 既定）。搬 `hand-drawn-character-
   creator.html` 76-584 行（three.js 从 588 行起不搬），改画单张 2D canvas；动画只留
   眨眼、遵守 `prefers-reduced-motion`。头像不落库（确定性生成，前端每次画）；
   「换一个形象」用 `preferences.avatarSeed` 覆盖（不动 `users.id` 语义——改名≠换人，
   但允许本人主动换）。**`nightmare` 物种（12% 概率长角/锯齿嘴/空洞眼）默认移出**
   ——企业内网工具里它只会被当 bug 报；保留物种代码，皮肤开关藏进 preferences。
6. **聊天窗两层**：本地指令集（主题/语言/导航/打开某页/当前项目切换说明）走前端
   命令表（前缀匹配，命中即执行并回系统气泡）——**agent 平台挂了这些仍可用**；
   未命中走自然语言通道。
7. **新手教程数据驱动 + 一条主线**：骨架 `{ route, selector, i18nKey }[]` JSON +
   `preferences.onboarding` 记完成态。本阶段只做**一条 5 步主线**（登录 → 建项目 →
   建接口 → 跑一次 → 看报告）——P6/P7/P8 刚动过导航与页面，selector 只对定型页面
   写。交互用**侧边卡片 + 目标元素描边**（不用遮罩高亮——那要处理滚动跟随与定位
   计算，两倍复杂度一倍价值）。`data-tour` 属性标靶点（CSS 类重构不破坏教程）。
8. **站内通知是一张新表，不并入 `notification_deliveries`**——那张记的是外发投递
   证据（HTTP 状态/error），站内通知是给用户看的消息（已读态）。两者语义不同。
9. **通知生产端**复用既有三个终态订阅点（`scheduler.ts:39-75` 的 alert 派发 / 套件
   通知 / CI 通知）+ 两个新源（成员变更、计划指派——**P9 回补 P6/P7 的生产端**）。
   收件人 = 项目全体成员（含 viewer）；不做按人订阅偏好（第一版）。
10. **小红点 + 弹窗**（用户 2026-09-04 确认）：铃铛入口（顶栏）未读数；进入助手抽屉
    看列表（标题 + 时间 + 深链，点击即读 + 跳转）；**新通知到达时 antd `notification`
    弹窗**（右上角）——「有通知时弹窗提醒 or 小红点提醒」两者都做：弹窗给「正在页
    面上」的即时感，小红点给「回头再看」的存量感。
11. **未读数轮询 60s + 页面获焦即拉**，不做实时推送——通知不是聊天，60s 延迟无感；
    SSE 推送要把跨项目的用户级通道引进来（现有 `stream.ts` 是项目级），为一个计数
    不值。
12. **不做**：语音、多轮记忆管理、知识库（agent 平台侧能力）、助手主动推送周报
    （那是 agent 平台的定时任务，不是平台功能）。

### 13.1 数据库迁移：054_p9_assistant.sql

```sql
ALTER TABLE project_settings ADD COLUMN IF NOT EXISTS assistant_enabled BOOLEAN NOT NULL DEFAULT false;

CREATE TABLE IF NOT EXISTS assistant_conversations (
  id UUID PRIMARY KEY,
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  upstream_conversation_id TEXT NOT NULL,   -- Nuwax 的 conversationId
  topic TEXT NOT NULL DEFAULT '',
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  last_active_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX IF NOT EXISTS assistant_conversations_user_idx
  ON assistant_conversations (user_id, last_active_at DESC);

CREATE TABLE IF NOT EXISTS notifications (
  id UUID PRIMARY KEY,
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  project_id UUID REFERENCES projects(id) ON DELETE CASCADE,
  kind TEXT NOT NULL,                      -- 'member_change' | 'plan_assign' | 'alert' | 'suite_result' | 'ci_result'
  title TEXT NOT NULL,
  body TEXT NOT NULL DEFAULT '',
  link TEXT NOT NULL DEFAULT '',           -- 站内相对路径
  read_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX IF NOT EXISTS notifications_unread_idx
  ON notifications (user_id) WHERE read_at IS NULL;
CREATE INDEX IF NOT EXISTS notifications_user_created_idx
  ON notifications (user_id, created_at DESC);
```

上游连接配置（baseUrl/agentId/apiKey）不建表——**环境变量**
（`ASSISTANT_BASE_URL` / `ASSISTANT_AGENT_ID` / `ASSISTANT_API_KEY`，进
`.env.example`）。理由：它是部署期资产（与 `JWT_SECRET` 同类），换它等于换上游，
不是运行期配置；且避免「Key 在库里、库被 MCP 读」的递归暴露面。

### 13.2 后端新 API

```
# 助手（项目级；assistant_enabled=false 时整组 404，与 /mcp 同款闸门）
POST   /api/v1/projects/:id/assistant/conversations          创建（转发上游 conversation/add）
GET    /api/v1/projects/:id/assistant/conversations          我的会话列表
POST   /api/v1/projects/:id/assistant/conversations/:id/message   转发消息，SSE 流回（hijack）
POST   /api/v1/projects/:id/assistant/conversations/:id/stop      转发上游 stop
GET    /api/v1/projects/:id/assistant/conversations/:id/messages  历史代理（转发上游，带缓存头）

# 通知（跨项目，用户级）
GET    /api/v1/notifications?unread=1&page=&pageSize=    我的通知（分页）
GET    /api/v1/notifications/unread-count                未读数（轮询端点）
POST   /api/v1/notifications/:id/read                   标已读
POST   /api/v1/notifications/read-all                   全部已读
```

`lib/inbox.ts`：`deliverInbox(projectId, kind, title, body, link)`——拉项目成员
批量插入 + fire-and-forget（照 `writeAuditLogAsync` 的取舍：通知写失败不能拖垮业务
事务）。

### 13.3 前端

- **顶栏铃铛**（`ProjectShell` 与 `GlobalApp` 顶栏）：未读数 badge + 下拉最近 10 条；
  「查看全部」进通知列表页（全局层路由 `/notifications`）。
- **助手抽屉**：右下角常驻圆钮（人物头像 32px）→ 展开 420px 抽屉（头像 96px +
  会话切换 + 消息流 + 输入框）。消息流支持：markdown 纯文本（照 8.13 边界 19 的
  pre-wrap 纪律，不引渲染器）、工具调用行、**proposal 预览卡**（草稿 + 差异摘要 +
  「确认应用 / 放弃」）。
- **人物生成器**：`lib/avatar.ts`（种子 → 物种/五官/配色确定性生成）+ 两个尺寸
  React 封装。眨眼动画仅抽屉打开时运行（Quiet Console「全应用只允许一个环境动画」
  ——与在飞请求呼吸点不同时出现）。
- **新手教程**：`lib/tour.ts` + `TourCard.tsx`（侧边卡片 + `data-tour` 描边）+ 一条
  主线的步骤 JSON（双 i18n）。首登自动触发（`preferences.onboarding.mainDone`）。
- **本地指令表**：`/主题 深色`、`/语言 en`、`/打开 接口管理`……前缀匹配 + 回车直发
  兜底。

### 13.4 实施顺序（P9-1 … P9-8）

| 步 | 内容 | 产出 |
| --- | --- | --- |
| P9-1 | 迁移 054 + 通知表/inbox + 铃铛 + 列表页 + 两个新源回补 | 通知闭环（小红点 + 弹窗） |
| P9-2 | 助手代理五条路由 + SSE 转发 + 事件归一 | curl 能对话 |
| P9-3 | 会话管理 + `assistant_enabled` 闸门 + 系统管理配置块 | 项目级开关可用 |
| P9-4 | 聊天抽屉 UI（消息流/工具行/输入） | 手工验收链路 |
| P9-5 | 本地指令表 + proposal 预览卡（草稿 → JWT 应用） | 闭环「AI 起草、人确认」 |
| P9-6 | 人物生成器（Canvas 2D 移植 + 头像挂点 + 换形象） | 形象可见 |
| P9-7 | 教程骨架 + 主线 5 步 + 双 i18n | 新用户引导可用 |
| P9-8 | 收尾：上游异常演练（断流/限流/ERROR 事件）+ 验收 | 门槛全过 |

### 13.5 验收门槛

1. `assistant_enabled=false` 的项目：助手路由组 404 且不泄露配置存在性。
2. 对话过程网络面板里**没有**上游 API Key（全链路服务端转发）。
3. `PROCESSING` 工具行如实显示 agent 正在调用的工具名。
4. agent 产出的草稿（如一段响应脚本）经预览卡「确认应用」后用**本人 JWT** 落库；
   审计里 created_by 是本人。
5. agent 平台停机：本地指令（主题/语言/导航）仍可用；聊天通道给出明确错误气泡。
6. 长空闲（>30s 无 token）：连接不断（保活注释行生效）。
7. 成员被加入项目后 ≤60s 顶栏出现小红点与弹窗；点击通知深链到正确页面。
8. 同名用户头像相同；改名后头像不变；「换一个形象」只影响本人。
9. `prefers-reduced-motion` 下无眨眼动画。
10. 主线教程 5 步全程无「目标元素找不到」的卡死（找不到自动跳步）。

### 13.6 后置增补：项目可见性两层 + 权限申请审批流（2026-09-07 提出，记录取向不排期）

**现状锚点**：`queryProjectMetrics`（`dashboard.ts:33-38`）对非系统管理员用
`EXISTS(user_project_roles)` 过滤——非成员在项目列表 / 数据看板 / 项目切换器**完全
看不到**非成员项目（不是「看得到进不去」），URL 直达 `GET /projects/:id` 由
`requireProjectAccess` 回 403。发现层的缺失正是动因：连项目存在都不知道，「找谁要
权限」无从谈起——成员页展示管理员清单的前提是先进得了项目（P6-4 的产品理由只覆盖
了已进项目的 viewer）。

**取向（防止将来跑偏）**：

1. **可见性拆两层，指标不随身份外泄**：项目身份（名称 / 描述）对全部登录用户可见；
   接口数 / 通过率等指标仍只对成员计算与下发，看板聚合的分母不含非成员项目——
   「看得到」买到的是发现，不是数据。
2. **无权限落地页**：非成员 URL 直达项目时不再是裸 403——身份级端点回 200（项目名 +
   管理员线索），前端渲染「无权限 + 申请入口」落地页。这是审批流的 UX 前置，也是
   `requireProjectAccess` 之外唯一要开的口子。
3. **审批流复用成员管理，不开第二条写路径**：新表 `access_requests`（project /
   user / 建议角色 / status: pending·approved·rejected / handled_by / handled_at /
   note）；**批准 = 现有成员 upsert**（`granted_by` 落审批人，`member.upsert` 审计
   照旧），拒绝只记状态。P6-4 成员页的人工添加与 P6-3 邀请码通道照旧——审批流是
   用户主动的自助通道，不是唯一通道。
4. **触达靠站内信**（排在本节的原因）：新申请 → 通知全体 project_admin；审批结果 →
   通知申请人。没有通知中心，审批流等于管理员靠刷页面发现申请，不如不做。
5. **幂等**：同一 (project, user) 只允许一条 pending；被拒后可再申请（第一版不限流）。
6. **实施时再定的开口**：申请是否带建议角色（管理员可改后批准）；非成员可见的管理员
   线索到什么粒度（email / 仅姓名）；项目切换器是否列出带锁的非成员项目（倾向不列
   ——切换器是「进入」的入口，目录页才是「发现」的入口）。

---

## 十四、P10 — 性能、插件、版本 (3 周)

> **原 P6，2026-09-04 顺延为 P10**（新增 P6 用户与权限 / P7 测试管理 / P8 数据统计 /
> P9 站内助手四个阶段插在它之前）。章节内编号随之由 `10.x` 改为 `14.x`；文档其余位置引用
> 本章时写 **P10 14.1** 而不是旧的「P6 10.1」。**顺延不是降级**：它排在最后的理由从头到尾
> 没变——归档与截断是口径变更，得先有稳定的读侧口径（P8-1 收口）才谈得上改存储形状。
>
> **2026-09-05 自主计划十四章整章并入本文件**，内容原样搬移，未改一字；主计划原位只留
> 一行指针。

### 14.1 性能优化

- 执行历史归档 (分区表)
- 大响应体截断 + 对象存储 (MinIO/S3)
- 查询缓存 (Redis)

> **对象存储抽象已于 P4.5 提前落地**（`lib/objectStore.ts`，`fs` + `s3` 两驱动，见 8.4）。
> 本节剩下的是另外两件：**执行历史归档（分区表）** 与 **大响应体截断**。后者不是「把
> `response_body` 搬进对象存储」这么一步——它是一次**口径变更**（多长算大？截断后详情页
> 显示什么？历史行怎么算？），必须连着归档策略一起想，因此 P4.5 一个字都没改
> `executions.response_body`（8.0 边界 14）。
>
> **P8 把这一节的入口条件写实了**（12.0 边界 17）：统计侧刻意不建汇总表与物化视图，因为
> 那会带来「什么时候刷新 / 刷新失败怎么办 / 口径改了怎么重建」三个新问题，而当时还没有
> 一个真实的慢查询。**归档才是这条路的正确入口**——先把冷数据分出去，再谈要不要预聚合。

### 14.2 插件机制

- 插件生命周期: 注册 → 启用 → 禁用 → 卸载
- 钩子点: 执行前/后, 断言, 报告生成

### 14.3 版本历史

- 接口变更追踪
- 环境配置快照
- 执行结果回放

> **文本用例的版本历史一并在这里**（P7 边界 18）：P7 只给 `updated_at` 与审计里的「谁改的」，
> 不做逐版本 diff 与回滚——那与本节的「接口变更追踪」是同一件事的两个面，分两处做会长出
> 两套版本模型。

---

## 十五、依赖与里程碑

### 15.1 新增依赖（五个阶段合计）

| 阶段 | 依赖 | 用途 | 备注 |
| --- | --- | --- | --- |
| P6 | — | — | 零（邀请码 scrypt 照抄第三次……第五次） |
| P7 | `exceljs` | `.xlsx` 固定模板解析 + 计划 CSV 导出的升级路径 | 后端按需 `import()`，不进启动路径 |
| P8 | — | — | 零（手写 SVG 是定案） |
| P9 | — | — | 零（Canvas 2D 人物、SSE 复用既有） |
| P10 | — | — | 零（查询缓存复用 P1 已装的 `ioredis`；对象存储抽象已于 P4.5 落地 `lib/objectStore.ts`） |

### 15.2 里程碑（并入主计划 12.1 表）

| 里程碑 | 阶段 | 交付物 | 验收标准 |
| --- | --- | --- | --- |
| M8 | P6 | 注册 + 成员与角色 + 三级权限 + 审计可读 | 邀请码注册即入项目；viewer 界面无写入口；成员变更可审计 |
| M9 | P7 | 文本用例库 + XMind/Excel 导入 + 绑定 + 测试计划 | 300 条 XMind 导入正确；自动化覆盖率可读；计划可标结果可导出 |
| M10 | P8 | 口径收口 + 失败归因 + 全局/项目统计 + 下钻 | 四处通过率一致；归因分布与覆盖率同屏；图表 hover/下钻可用；零图表库 |
| M11 | P9 | 站内助手 + 站内通知 + 教程 | 代理对话可用且 Key 不出服务端；proposal-first 闭环；通知小红点 + 弹窗 |
| M7 | P10 | 性能 + 插件（原 P6 顺延） | 支持 1000+ 并发执行 |

### 15.3 主计划与规格已同步的编辑（2026-09-04 全部完成）

- [x] 主计划原「十、P6 性能/插件/版本」→「十四、P10」（内部 10.x → 14.x）。
- [x] 主计划 9.8 加指针：P9 已立项，见本文件十三章。
- [x] 主计划「一、整体路线图」ASCII 图扩为 P0…P10。
- [x] 主计划头部加「P6–P9 已规划」说明与本文指针。
- [x] 主计划「十一、依赖安装清单」（P7 `exceljs` 行）与「十二、里程碑」（M8–M11 + M7 改挂 P10）。
- [x] `API_AUTOMATION_SPEC.md` 路线表补 P6–P10 行。
- [x] 全文旧「P6 10.1」类引用改为「P10 14.1（原 P6 10.1）」。

### 15.4 P10 并入本文件（2026-09-05 完成）

- [x] 主计划「十四、P10 — 性能、插件、版本」整章移入本文件（十四章，紧跟十三章），内容原样未改。
- [x] 文件更名 `DEVELOPMENT_PLAN_P6-P9.md` → `DEVELOPMENT_PLAN_P6-P10.md`，标题与头部归属/章节映射说明同步。
- [x] 15.1 补 P10 零新增依赖行（「四个阶段合计」→「五个阶段合计」）；15.2 补 M7 行（与主计划 12.1 表一致）。
- [x] 主计划十四章原位替换为一行指针；头部说明、路线图 ASCII、9.8、依赖清单、里程碑表中的文件名引用同步更名。
- [x] `API_AUTOMATION_SPEC.md` 路线表 P6 行的文件名引用同步更名。



