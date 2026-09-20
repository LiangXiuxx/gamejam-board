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
- `○ 未连接` —— 没连上，顶部会出现红色横幅，用内置的 24 条示例数据兜底（改动不会保存）

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

### ⑥ 开始建任务

打开线上地址 → 点右上角 **「＋ 新建任务」**。创建后会自动落库并同步给所有人，
刷新不会丢（前提是右上角徽标显示 `● 已同步`）。

---

## 看板流程：为什么是这 4 列

```
待办  ──▶  进行中  ──▶  待验收  ──▶  已完成
              ▲            │
              └──── 驳回 ───┘
```

**4 列就是工作真实的推进顺序**：还没动手 → 在做 → 做完等人点头 → 点头了。

之前那版有 7 列（待规划/待开始/进行中/待验收/测试中/阻塞/已完成），有三个毛病：

1. **「待验收」排在「测试中」前面**，等于要求先验收再测试，顺序是反的。
   现在「测试」不是独立的列——谁做的谁自己测，属于「进行中」的一部分；
   做完才进「待验收」。
2. **「待规划 / 待开始」在两周的项目里是同一件事**（还没开始就是待办）。
3. **「阻塞」根本不该是一列。** 阻塞不是一个流程阶段，是一个**标记**：
   任务被卡住时它还在"进行中"，拖到单独的列会丢失"它本来在做什么"，而且
   看板上会出现"进行中 0 / 阻塞 0"的假象——实际是有人卡住了。
   现在阻塞是卡片上的红色标记，任务**留在原地**，还能用右上角「只看阻塞」一键筛出来。

### 验收不合格怎么办

**把卡片从「待验收」往回拖**（拖到「进行中」= 小改，拖到「待办」= 要重做），
会弹出输入框让你写**哪里不行**。确认后：

- 卡片上留下黄色的「返工 ×N」标记和驳回原因，返工的人一眼看到要改什么
- PM 从此能看出哪个功能反复不合格——这是最容易丢的坏消息
- 以后再次验收通过、拖进「已完成」时，驳回原因自动清掉，返工次数留作历史

## 日常使用

| 操作 | 结果 |
| --- | --- |
| 拖动卡片 | 改 `status`，实时同步给所有人 |
| 从「待验收」往回拖 | 触发**驳回**，要填原因，记返工次数 |
| 卡片右上角 `!` | 标记 / 解除**阻塞**（卡片左边出现红条） |
| 卡片右上角 `×` | 删除任务（有确认弹窗） |
| 「只看阻塞」按钮 | 一键筛出所有被卡住的任务 |
| ＋ 新建任务 | 可指定状态 / 负责人 / 优先级 / 验收标准 |
| 搜索、优先级、成员筛选 | 纯前端过滤，不写库 |

> 卡片右上角的两个按钮只在**鼠标悬停**时出现，平时不干扰阅读。

## 界面语言

界面**中文为主，英文作为灰色小字副标题**（如「待办 / TO DO」），方便对照。

注意：**写进数据库的字段值仍然是英文标识符**（`doing`、`Programming`…），
只有显示层做中英映射。这样做的好处是老数据和新数据格式统一，不会一半中文一半英文；
换语言也只需要改 `TYPE_DEFS` / `STATUS_DEFS` 两个表，不用动数据库。

`owner` 保持原样显示——中文名、英文名、`程序 A`、`程序员A` 都能识别，
头像缩写会自动适配。

## 字段约定（Firestore 里长这样）

```
tasks/{自动ID}
  title       "测试任务"
  status      "doing"        ← todo / doing / review / done
  priority    "P0"           ← P0 / P1 / P2
  owner       "程序员A"
  type        "Programming"  ← Programming / Art / Design / Production
  desc        "验收标准"
  blocked     false          ← 阻塞标记（true 时卡片左边有红条）
  rework      0              ← 返工次数，验收不合格时 +1
  reviewNote  ""             ← 最近一次驳回原因，验收通过后自动清空
  createdAt / updatedAt       timestamp
```

`status` 做了**别名兼容**，旧词汇和中文都能读：

| 你写的 | 实际落到 |
| --- | --- |
| `backlog` / `ready` / `待规划` / `待开始` | `todo` |
| `doing` / `wip` / `进行中` / `在做` | `doing` |
| `review` / `testing` / `测试中` / `待验收` | `review` |
| `done` / `已完成` | `done` |
| `blocked` / `阻塞` / `卡住` | `doing` + `blocked: true` |

所以在 Firebase 控制台里手敲旧词也不会出错。**建议新数据直接用 `todo/doing/review/done`**。

`owner` 想显示成头像缩写（`程序员A` → `PA`），用 `程序 A` 更整齐，但两种都认。

## 常见问题

**顶部出现红色横幅「未连接 Firestore」** → 此时页面处于**离线演示模式**，
显示的是内置的 24 条示例任务，**任何改动刷新后都会丢失**（这是刻意设计：
宁可明确报错，也不假装保存成功）。横幅上写了原因，去发布规则后点「重试连接」。

按 F12 看 Console 里的具体错误码：
- `permission-denied` → 规则没发布，或者你没走第 ② 步
- `invalid-api-key` → config 复制不完整
- `Failed to fetch` → 用 file:// 打开的，换 localhost 或线上地址

**创建任务后刷新就没了** → 看右上角徽标。如果是 `○ 未连接`，说明当时在离线演示模式，
任务只存在于内存里，从来没写进数据库。发布规则 + 点「重试连接」，徽标变成
`● 已同步` 之后再创建就会真正落库。

**队友改了数据我这边没变** → 正常是实时的。如果卡住，刷新一次；再不行看 Console 报错。
