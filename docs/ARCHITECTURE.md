# 架构设计：从单页应用到多用户系统

> 本文说明长期目标的技术方案、**为什么全部免费**、以及数据结构怎么设计。
> 分阶段的落地顺序见 [ROADMAP.md](ROADMAP.md)。

---

## 一、目标

把现在这个「单人静态页」升级成：

| 能力 | 说明 |
|---|---|
| **用户登录** | 每个人注册后登录，看到自己的计划 |
| **个性化** | 每人可以有自己的界面模式（简洁/丰富）与板块布局 |
| **导入计划** | 用户导入自己的课程表 / 竞赛安排 |
| **权限隔离** | 普通用户**看不到**右下角「存档」；管理员登录后可见并可修改 |
| **数据归属** | 所有云端数据的所有权归管理员，可随时完整导出 |

---

## 二、现状（v1.0.x）

```
GitHub Pages（纯静态，免费）
   └── index.html  ← 全部逻辑 + 硬编码的九周计划数据
         ├── 数据存 localStorage（key: aaa3:data）
         ├── 可选：绑定本地文件夹（File System Access API）
         ├── 可选：云端同步（jsonbin.io，需自填 URL + Key）
         └── 可选：导出/导入 JSON 备份
```

**数据结构**（这是迁移的关键资产，设计得已经很好）：

```js
S = { _updated: 时间戳, 任务ID: 时间戳, 任务ID: 0, ... }
//                                              ↑ 0 表示"取消勾选"的墓碑
```

- `filterItems(S)` 剥掉 `_` 开头的元字段，得到纯数据快照
- `mergeRecords(a,b)` 逐键按时间戳取新 —— **天然支持多端冲突合并**
- 这套 `{updated, items}` 格式**可以直接原样存进后端**，不需要重新设计

> ⚠️ 但注意：现在的数据里有大量**硬编码内容**（九周计划、任务模板 `TMPL`、时间线等）
> 是写死在 `index.html` 里的。要让用户导入自己的计划，必须先把这部分**外部化**。

---

## 三、免费方案选型

### 3.1 结论先行

> **推荐：Supabase（免费版，东京 / 新加坡节点）**
>
> 理由：唯一一个把「用户认证 + 数据库 + 权限控制」三件事都做好、且真正免费的方案。
> 你自己那套「离线优先 + 后台同步」的架构，恰好把它的高延迟问题抵消掉了（详见 3.4）。

### 3.2 三家免费额度对比（2026-10 核实）

| | **Supabase** | Firebase（Google） | Cloudflare Workers |
|---|---|---|---|
| 数据库 | Postgres **500 MB** | Firestore **1 GB** | D1 **5 GB** |
| 读额度 | **不限**（API 请求不限量） | 50,000 次/天 | 5,000,000 行/天 |
| 写额度 | **不限** | 20,000 次/天 | 100,000 行/天 |
| 用户认证 | **50,000 月活**（内置） | 50,000 月活（内置） | **无内置，要自己写** |
| 文件存储 | 1 GB | 1 GB | R2 10 GB |
| 出网流量 | 5 GB/月 | 10 GB/月 | 充足 |
| 权限控制 | **行级安全 RLS**（内置） | 安全规则（可用） | 自己写 |
| 费用 | **$0**（信用卡都不用绑） | **$0**（Spark 不可产生账单） | **$0** |
| 国内访问 | ⚠️ 慢（700ms+），无国内节点 | ❌ **被墙** | ⚠️ 部分线路可达 |

**全部 $0，且都无需绑定信用卡** —— 这一点很重要：不绑卡就不会产生意外账单。

### 3.3 排除理由

- **Firebase**：数据库和认证很好用，但 **Google 服务在中国大陆无法直接访问**，
  你的同学不可能都挂梯子。直接排除。
- **Cloudflare Workers**：额度最慷慨，数据控制力最强，但**没有开箱即用的用户认证**，
  得自己实现注册/登录/密码重置/会话管理——工作量大好几倍，且容易写出安全漏洞。
- **jsonbin.io**（你现在用的）：只是「一个能存 JSON 的箱子」，**没有用户概念**
  ——所有数据挤在同一个 bin 里，无法区分用户，也无法做权限。适合单人，不适合多用户。

### 3.4 关于 Supabase 在国内慢的问题

**实测数据**：国内直连 Supabase 云服务，普通 REST 接口延迟 **700ms+**，
Edge Function **1.5s+**；部分运营商还会出现 DNS 污染。这是「跨境访问」的物理现实，
升级到付费版（$25/月）也改变不了。

**但对你这个应用影响很小**，原因是你现在已经采用了**离线优先（offline-first）**架构：

```
用户打开页面
   ↓
立刻从 localStorage 渲染  ←── 0ms，页面秒开，不卡
   ↓
后台静默向云端同步（去抖 1.2s）
   ↓
700ms 后才返回         ←── 用户根本感知不到
```

你代码里的 `debounceCloud()` 已经就是这个模式了。所以**延迟被完全藏在后台**。

**要注意的另外两点**：
1. 免费项目 **7 天无活动会被自动暂停**，需要去后台手动恢复（数据不丢）。
   对你这种每天都用的场景，基本不会触发。
2. 免费版**不提供自动备份**。所以你现在那个「导出备份」功能一定要保留
   ——它是你数据安全的最后一道保险。

### 3.5 想更快的话（备选方案）

如果将来觉得慢得无法忍受，有两条路，但都**不是免费的**或**不划算**，暂不建议：

- 自部署 Supabase（Docker）：延迟可从 700ms 降到 50–100ms，但需要一台服务器，
  且要自己运维——已失去「免费 + 免运维」的意义。
- 国内云厂商（腾讯云开发 / 阿里云）：国内节点快，但免费额度有限、需实名认证，
  长期大概率产生费用。

---

## 四、数据模型设计

### 4.1 三张表（Supabase / Postgres）

```sql
-- ① 用户资料（扩展 Supabase 自带的 auth.users）
create table profiles (
  id          uuid primary key references auth.users(id) on delete cascade,
  email       text,
  display_name text,
  role        text not null default 'user',   -- 'user' | 'admin'  ← 权限的关键
  created_at  timestamptz default now()
);

-- ② 用户的计划（课程 / 竞赛安排）
create table plans (
  id          uuid primary key default gen_random_uuid(),
  owner_id    uuid not null references profiles(id) on delete cascade,
  title       text not null,                  -- 例如 "AAA 冲刺计划"
  kind        text default 'mixed',           -- 'course' | 'contest' | 'mixed'
  start_date  date,
  end_date    date,
  data        jsonb not null default '{}',    -- ★ 直接存现有的 {updated, items} 快照
  updated_at  timestamptz default now()
);

-- ③ 界面偏好（模式 + 板块布局）
create table prefs (
  user_id     uuid primary key references profiles(id) on delete cascade,
  mode        text default '丰富',             -- '简洁' | '丰富'
  layout      jsonb default '[]',             -- 板块顺序与显隐，例如 ["today","table","extra"]
  updated_at  timestamptz default now()
);
```

**关键设计**：`plans.data` 用 `jsonb`，**原样存你现在导出的那个 JSON 快照**。
这意味着老用户的备份文件可以直接导入新系统，零转换成本。

### 4.2 「存档只有我能看见」怎么实现

这就是 **行级安全（Row Level Security, RLS）**——把权限交给数据库，而不是前端 JS。

```sql
-- 开启行级安全
alter table plans enable row level security;

-- 规则一：每个人只能读写自己的行
create policy "own_plans" on plans
  for all
  using (owner_id = auth.uid());

-- 规则二：管理员可以读写所有行
create policy "admin_all_plans" on plans
  for all
  using (exists (
    select 1 from profiles
    where profiles.id = auth.uid() and profiles.role = 'admin'
  ));
```

**为什么这比前端隐藏更可靠**：

- 前端隐藏「存档」框，只是**不给普通用户看**——懂技术的人打开 F12 改一行 JS 就能看到。
- RLS 是在**数据库层面**拦截。即使有人绕过前端直接调 API，
  数据库也只会返回属于他自己的数据。**这才是真正的权限。**

两者要**同时做**：RLS 保证安全，前端隐藏保证体验。

### 4.3 认证方式选择

| 方式 | 免费 | 国内可用 | 备注 |
|---|---|---|---|
| 邮箱 + 密码 | ✅ | ✅ | **推荐主力**。注意：免费版内置邮件服务有**每小时发信上限**，用户量大时要配自己的 SMTP |
| GitHub 登录 | ✅ | ⚠️ 需能访问 GitHub | 省邮件，但你的同学不一定都有 GitHub 账号 |
| 手机号验证码 | ❌ 需付费短信服务 | ✅ | 排除 |

> 建议：**先只做邮箱+密码**，够用且简单。以后需要再加 GitHub 登录。

---

## 五、安全提醒

1. **Supabase 的 `anon` key 是公开的，这没问题**——它是设计成放在前端代码里的。
   安全性由 RLS 保证。但 **`service_role` key 绝对不能放进前端**，
   它绕过所有 RLS 规则。这个 key 只在你自己的管理脚本里用。
2. 管理员的 `role='admin'` **不能允许前端自行设置**，
   否则任何人都能把自己改成管理员。要手动在 Supabase 后台的 SQL Editor 里改。
3. 免费版没有自动备份，**务必保留现有的「导出备份」功能**。

---

## 六、迁移路径

```
v1.0.x  静态单页（现状）
   ↓
v1.1.x  ① 版本管理 + 文档体系          ← 已完成
        ② 数据外部化（把硬编码的计划抽成 JSON）
   ↓
v1.2.x  ③ 接入 Supabase 认证（能注册/登录，但功能不变）
        ④ 存档面板改为「登录后才显示」
   ↓
v1.3.x  ⑤ 数据上云，多设备同步
        ⑥ 管理员角色 + RLS 权限隔离
   ↓
v2.0.0  ⑦ 用户导入自己的课程/竞赛计划
        ⑧ 界面模式与板块布局个性化
```

**每一步都能独立跑通、独立发布版本**，不会出现「做到一半整个网页用不了」的情况。
这也是为什么建议分阶段——你随时都有一个能用的版本。
