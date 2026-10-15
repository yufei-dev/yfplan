# 于飞计划 · YUFEI PLAN

> 个人示例：数学竞赛 × CET6 九周备考计划 · 交互式单页应用
>
> **以计划为翼，以坚持为路。**

**当前版本：v1.2.1**

🌐 线上地址：<https://yfplan.weiyf.ccwu.cc/>

> `https://yufei-dev.github.io/yfplan/` 会自动跳转到上面的自定义域名。
> 绑定自定义域名后，**网址永久稳定**——以后仓库再改名，书签也不会失效。

---

## 这是什么

一个纯静态的单页应用（单个 `index.html`），用来管理九周备考计划：

- 📋 **今日任务** — 按时间段拆解，勾选打卡
- 🗓 **本周课表** — 一周七个时段 × 七天，一眼看清安排
- 🧭 **备考时间线** — 竞赛线与六级线的阶段划分
- ⏳ **倒计时** — 距数学竞赛 / CET6 的天数
- 💾 **存档** — 本地文件夹绑定 / 云端同步 / 导出备份 / 从文件恢复

部署在 **GitHub Pages**（免费），无后端、无构建步骤。

---

## 版本管理

本项目每次改动都会发一个新版本，**所有历史版本都可以找回**。

### 版本号规范

| 位数 | 何时递增 | 例子 |
|---|---|---|
| **主版本** | 架构级变动 | v2.0.0 接入用户登录系统 |
| **次版本** | 新增功能，向下兼容 | v1.1.0 新增版本管理 |
| **修订号** | 修 bug、调样式、改文案 | v1.0.1 修复课表行分隔线 |

### 三种找回历史版本的方式

1. **看改动记录** → [`CHANGELOG.md`](CHANGELOG.md)，每个版本改了什么写得清清楚楚
2. **看某个版本的完整代码** → 点文件页右上角的 **History**，再点某次提交的 **Browse files**
3. **下载某个版本存档** → [Releases 页面](../../releases)，每个版本都有 zip 可下载

### 发布新版本的流程

1. 改代码，把 `index.html` 里的 `var VER` 改成新版本号
2. 在 `CHANGELOG.md` 顶部加一条新版本记录
3. 提交（commit message 写清楚改了什么）
4. GitHub → **Releases** → **Draft a new release** → 填 tag（如 `v1.2.0`）→ 发布

---

## 文档

| 文档 | 内容 |
|---|---|
| [`CHANGELOG.md`](CHANGELOG.md) | 每个版本改了什么 |
| [`docs/BRAND.md`](docs/BRAND.md) | **品牌规范**（名称、标语、视觉）——后续版本必须遵循 |
| [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) | 长期架构、**免费云服务选型对比**、数据模型、权限设计 |
| [`docs/ROADMAP.md`](docs/ROADMAP.md) | 分阶段路线图与验收标准 |

---

## 长期目标

从单人静态页升级为**多用户系统**：

- 用户登录，每人有自己的计划
- 自定义界面模式与板块布局
- 导入自己的课程 / 竞赛安排
- **存档面板仅管理员可见**（数据库层面的权限隔离，不只是前端隐藏）

全部使用**免费**方案实现（Supabase 免费版），详见
[`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md)。

---

## 本地开发

没有构建步骤，直接打开即可：

```bash
# 方式一：直接用浏览器打开 index.html
# 方式二：起个本地服务器（推荐，File System Access API 需要 https 或 localhost）
python3 -m http.server 8000
# 然后访问 http://localhost:8000
```

> ⚠️ 「绑定存档文件夹」用到了 File System Access API，
> 需要 `https://` 或 `localhost` 环境。直接用 `file://` 打开时该功能不可用。
