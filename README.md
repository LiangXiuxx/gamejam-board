# GameJam Production Board

一个单文件的看板（Kanban）+ Firestore 实时同步 + GitHub Pages 部署。

```
index.html                        ← 整个应用（UI + Firestore 读写，无需构建）
firestore.rules                   ← Firestore 安全规则
.github/workflows/deploy.yml      ← 可选：Actions 部署（默认走分支部署，用不到它）
README.md                         ← 本文档
```

- 线上地址：https://liangxiuxx.github.io/gamejam-board/
- 仓库：https://github.com/LiangXiuxx/gamejam-board

打开网页 → 所有人看到同一份数据 → 拖动卡片直接写回数据库。

---

## ⚠️ 首次克隆/推送前先对齐历史

远端最初的提交是走 GitHub API 创建的（当时沙箱代理只放行 `api.github.com`，
`github.com` 被拦，git push 走不通），所以远端和你本地的 SHA 不一样、内容相同。
在你自己的机器上先对齐一次，之后正常 `git push` 即可：

```bash
cd D:/Program_Files/Game/gamejam_board
git fetch origin && git reset --hard origin/main
```

---

## 你现在要做的事（按顺序）

### ① 关于你贴的那份 Firebase config

`apiKey` 是**公开信息**，写在网页里是正常做法，不是泄露。真正决定安全的是 `firestore.rules`。

我已经把它填进 `index.html` 第 193 行左右。顺手删掉了 `getAnalytics`：
Analytics 在 `file://` 本地打开时会直接抛错，而看板用不到它。

```js
// index.html 顶部，需要改的只有这几行
const firebaseConfig = { ... };     // 已按你贴的填好，若有出入以控制台为准
const COLLECTION = "tasks";         // ← 你 Firestore 里放任务的那个集合名
const JAM_START  = "2026-09-16";    // ← 开赛日期，用来算 DAY n；不想算就填 ""
const MEMBERS    = ["程序 A","程序 B","美术","PM A","PM B"];  // 新任务下拉的候选人
```

**最需要确认的是 `COLLECTION`**：控制台左侧「Firestore Database」里，你那条 `测试任务` 的**上一级集合名**是什么？我默认写的是 `tasks`，不一样就改这一个词。

### ② Firestore 建库 + 放规则

Firebase 控制台 → **Firestore Database** → 创建数据库
- 位置随便（asia-east1 即可）
- 模式选 **Production mode**（不要用 test mode，30 天就过期）

然后 → **Rules** 标签页 → 把 `firestore.rules` 整个内容粘进去 → 发布。

规则的含义：任何人都能读，写入必须是一条"形状正确"的任务（标题非空、字段长度有上限），其它集合一律拒绝。5 人小队够用。

> ⚠️ 代价：知道网址的人都能改数据。给队友的链接别外发。
> 想收紧就打开 `firestore.rules` 底部注释，启用匿名登录 + `if request.auth != null`。

### ③ 本地先跑一遍

浏览器的 file:// 直开能显示界面，但连不上 Firestore（会走"离线演示模式"）。要真连库，起个本地服务：

```bash
cd D:/Program_Files/Game/gamejam_board
python -m http.server 8000
# 打开 http://localhost:8000
```

右上角同步徽标会显示：
- `● 已同步` —— 连上了，卡片是数据库里的真实数据
- `○ 离线演示` —— 没连上，用内置的 24 条示例数据兜底（改动不会保存）

### ④ 推到 GitHub（✓ 已完成）

仓库 `LiangXiuxx/gamejam-board` 已建好，文件也已提交到 `main`。
以后改完东西照常推送即可：

```bash
cd D:/Program_Files/Game/gamejam_board
git add -A && git commit -m "描述你的改动"
git push
```

如果是从零开始在别的机器上重建，命令是：

```bash
git init -b main && git add . && git commit -m "init"
gh repo create gamejam-board --public --source=. --remote=origin --push
```

### ⑤ 开启 GitHub Pages

仓库 → **Settings** → 左侧 **Pages** → Source 选 **Deploy from a branch** →
Branch 选 **`main`** / 目录选 **`/ (root)`** → Save。

等一分钟，访问：

```
https://liangxiuxx.github.io/gamejam-board/
```

> **为什么用分支部署而不是 Actions？** 这个项目是零构建的纯静态站，`index.html`
> 直接改直接生效，不需要 CI 编译。`.github/workflows/deploy.yml` 我留在仓库里作为
> **可选升级路径**（以后真要加构建步骤时再用）。注意：往 GitHub 推
> `.github/workflows/` 下的文件需要额外的 `workflow` 授权，本地跑一次
> `gh auth refresh -s workflow` 即可。

### ⑥ 灌入示例数据（可选）

打开线上地址 → 点右上角 **「写入示例任务」** → 确认。24 条任务一次性写进 Firestore。
`seedBtn` 在集合非空时会自动禁用，不会重复灌。

---

## 日常使用

| 操作 | 结果 |
| --- | --- |
| 拖动卡片 | 改 `status` 字段，实时同步给所有人 |
| 卡片右上角 × | 删除任务（有确认弹窗） |
| ＋ New Task | 新任务，可指定状态/负责人/优先级/验收标准 |
| 搜索 / 优先级 / 成员筛选 | 纯前端过滤，不写库 |

## 字段约定（Firestore 里长这样）

```
tasks/{自动ID}
  title      "测试任务"
  status     "doing"        ← doing / backlog / ready / review / testing / blocked / done
  priority   "P0"
  owner      "程序员A"
  type       "Programming"  ← Programming / Art / Design / Production
  desc       "验收标准"
  createdAt / updatedAt      timestamp
```

页面对 `status` 做了**别名兼容**：`进行中`、`Doing`、`in progress`、`wip` 都会被识别成 `doing`。
所以你现在手敲的 `测试任务 / 进行中 / P0 / 程序员A` 不用改，页面能直接读出来。

`owner` 想显示成头像缩写（`程序员A` → `PA`），用 `程序 A` 更整齐，但两种都认。

## 常见问题

**页面显示「离线演示」** → 按 F12 看 Console。
- `permission-denied` → 规则没发布，或者你没走第 ② 步
- `invalid-api-key` → config 复制不完整
- `Failed to fetch` → 用 file:// 打开的，换 localhost 或线上地址

**推送后 Pages 还是 404** → Settings → Pages 的 Source 必须是 *GitHub Actions*；或者第一次构建还在跑，看 Actions 标签。

**队友改了数据我这边没变** → 正常是实时的。如果卡住，刷新一次；再不行看 Console 报错。
