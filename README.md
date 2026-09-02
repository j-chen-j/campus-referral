# 内推码生成器（HR 自助版）

校招内推二维码工具。HR 自己填、自己生成，每张码按内推人**姓名**编码；候选人微信扫码后填写简历，直接发到 HR 邮箱，且**邮件自动带推荐人**。不拉群、方便微信分享、整套文件夹拷到任意电脑双击即用（生成器离线可用）。

## 已部署的服务
- 投递页（候选人扫二维码打开）：https://j-chen-j.github.io/campus-referral/apply.html
- 收件方式：候选人浏览器**直连 formsubmit.co 的「隐形邮箱别名」**提交，由 formsubmit 转发到 HR 邮箱。**投递页源码与 GitHub 仓库里只有别名、没有任何邮箱**，候选人查看网页源代码或爬仓库都拿不到 HR 真实邮箱。
- 岗位列表独立成 `jobs.json` 放在仓库根目录，投递页每次打开实时拉取，所有二维码共享，**改一次全局实时生效**；HR 用 `hr-jobs.html`（Edge 双击打开）自助改，**GitHub 令牌仅在 HR 本机浏览器保存，不写入源码、不上传 GitHub**（详见下文「HR 改岗位」）。

## 文件怎么用
| 文件 | 用途 | 谁用 |
|---|---|---|
| `generator.html` | 生成内推二维码（离线单文件，自带台账记录） | HR |
| `hr-jobs.html` | **HR 改岗位**（Edge 双击即用；令牌仅存本机浏览器，不进源码/不传 GitHub） | HR ★ |
| `apply.html` | 候选人投递页（已部署到 GitHub Pages，实时拉取 `jobs.json`） | 候选人（扫码自动打开） |
| `jobs.json` | 岗位数据源（唯一权威，改岗位只动它；投递页实时拉取） | 系统自动读取 |
| `admin.html` | 备用改岗位页（直接调 GitHub API，需 token） | 管理员 |
| `cloudflare-worker.js` | 备用方案遗留（当前岗位管理走 `hr-jobs.html` 直连 GitHub，已不依赖 Cloudflare） | 可保留可删 |

## HR 生成二维码（generator.html）
1. 双击 `generator.html`（断网也能用，已内联二维码/打包库，无需额外文件夹）。
2. 二维码固定为「投递网页」模式（候选人微信扫码直接打开投递页，无需邮件 App）。
3. 填「公司名称」+ 在「内推人名单」每行填一个姓名。
4. 点「生成二维码」→ 每张码按姓名编码，内容 `apply.html?ref=姓名&co=公司`。
5. 单张存图发微信，或一键打包。

> 旧二维码指向同一投递页地址，长期有效，不用重做。

## 已生成台账（generator.html 本机记录）
每次点「生成二维码」，系统会用浏览器 IndexedDB **自动记下这张码**（内推人姓名、公司、生成时间、二维码图片），并在生成器第 4 步「已生成台账」面板里展示：
- **查看 / 下载 / 删除**：单条记录可单独重新查看大图、下载 PNG、或删除。
- **导出台账**：点「⬇ 导出台账」把全部记录导成一个 JSON 文件，存到 HR 电脑随时双击可查看/再导入。
- **导入台账**：换电脑或清缓存后，点「⬆ 导入台账」选回之前的 JSON 即可恢复全部历史。
- **清空**：仅清本机记录，已导出的文件不受影响。

> 记录存在 HR 所用浏览器的本地数据库（IndexedDB），**断网可用、不上传任何服务器**。注意：不同浏览器/电脑之间不共享，需靠「导出/导入」迁移；清浏览器数据会丢失（导出可防丢）。

## HR 管理岗位（hr-jobs.html，★ 推荐给 HR 用）
**HR 电脑上没有 GitHub 账号、也没有本地代码文件时，用这个。** 双击 `hr-jobs.html` 即可改岗位，
不需要 GitHub 账号、不需要 token、不需要本地服务——只需能上网。

1. 双击 `hr-jobs.html`（服务地址一般已预填好，不用改）。
2. 点「读取当前岗位」→ 列出当前所有岗位。
3. 改文字 / ↑↓ 调整顺序 / ✕ 删除 / ＋ 添加。
4. 点「保存并生效」→ 约 30 秒后所有二维码（含已打印的旧码）自动显示新岗位。

**原理**：页面把改动发到 Cloudflare Worker，由 Worker 读写背后的 **Cloudflare KV 存储**（岗位列表就存在这里）。
整套链路**完全不需要 GitHub Token、不用填任何密钥**——你只需要在 Cloudflare 后台建一个 KV 命名空间并绑给 Worker 即可。

### Worker 侧部署（一次性，管理员做）
1. Cloudflare → **Workers & Pages** → 你的 Worker（如 `old-brook-e3b5`）→ **Edit code**。
2. 把 `cloudflare-worker.js` 内容**全部替换**进去 → **Deploy**（先把代码部署上）。
3. 创建并绑定 KV 命名空间（**不需要任何密钥/Token**）：
   - 左侧菜单 **KV** → **Create a namespace** → 随便起名（如 `referral-jobs`）→ Create。
   - 回到 Worker → **Settings → Variables** → **KV namespace bindings** → Add binding：
     - Variable name 填 **`JOBS_KV`**（必须一模一样，代码里就读这个名）
     - KV namespace 选刚才建的 `referral-jobs`
   - **Save**。
4. 回到 **Deployments** 再点一次 **Deploy**（绑定 KV 后必须重新部署才生效，这是 Cloudflare 的坑）。

> 部署后第一次点「读取当前岗位」会看到内置的 4 个默认岗位；HR 改完点「保存」即写入 KV，之后所有二维码实时生效。
> 想换岗位数据不丢，KV 命名空间一直在，Worker 重新部署也不影响已有数据。

## 备用方案：admin.html（需 GitHub Token，管理员自己用）
岗位统一存放在仓库的 **`jobs.json`**，改一处所有二维码实时更新：
1. 双击 `admin.html`。
2. 粘贴 GitHub Token（详见下方「凭证管理」）。
3. 点「加载岗位」→ 可改文字 / ↑↓ 排序 / ✕ 删除 / + 添加。
4. 点「保存并更新到 GitHub」→ 自动写回 `jobs.json`，约 30 秒后旧二维码自动显示新岗位。

> `admin.html` 只在你本地运行，不会上传任何第三方；它直接调 GitHub API，所以**用的人必须有 token**。
> 给 HR 用请优先用上面的 `hr-jobs.html`。

## 为什么旧二维码改完也能实时更新（架构说明）
岗位列表**不再写死在 `apply.html` 里**，而是存在 Cloudflare KV（经 Worker 暴露 `GET /jobs`）；`apply.html` 打开时用 `fetch(WORKER_URL + '/jobs')` 实时拉取，并加了 `no-cache` 等防缓存头。
- 因此候选人（含微信内置浏览器）扫**任意一张已生成的旧二维码**，打开的永远是线上最新岗位。
- 之前“改了岗位但旧码不变”的根因正是：岗位写死在 HTML 内、被微信整页缓存。改成数据分离后该问题消除。
- 若微信仍显示旧岗位，多半是 HTML 本身被缓存：长按页面「刷新」一次即可，之后即实时。

## 凭证管理（重要，避免大号风险）
**核心策略：小号常开 Token，大号不常开。**
- 仓库 `campus-referral` 的 owner 是**大号 `j-chen-j`**，但小号 `allison0107623-maker` 已被加为该仓库的 **Write collaborator**，因此小号推送的修改同样会触发 GitHub Pages 重建，二维码实时更新。
- 日常改岗位用**小号**生成的 Token 即可，大号平时不用生成 Token。

**小号 Token 生成方式（一次性，长期有效）：**

> ⚠️ 关键坑：fine-grained token 的 **All repositories** 只覆盖你**自己拥有**的仓库，不包含作为 collaborator 的大号仓库。若误选 All repositories，admin.html「加载岗位」能成功（public 仓库可匿名读），但**保存会报 `Resource not accessible by personal access token`**。因此请优先用下面更稳的 **classic token** 方案。

**方案 A（推荐，最稳）：小号 classic token**
1. 登录小号 `allison0107623-maker` → 右上角头像 → Settings。
2. Developer settings → Personal access tokens → **Tokens (classic)** → Generate new token (classic)。
3. Note 任意（如 `内推码`）；Expiration 选 **No expiration**。
4. 勾选 **`repo`**（这一个即可，会覆盖你小号能访问的所有仓库包括协作仓）。
5. Generate，复制 `ghp_...` 那串，填进 `admin.html`。

**方案 B（更细粒度但易踩坑）：小号 fine-grained token**
1. 登录小号 → Settings → Developer settings → **Fine-grained tokens** → Generate new token。
2. Expiration 选 **No expiration**。
3. Repository access → **Only select repositories** → 明确勾选 `j-chen-j/campus-referral`（不要选 All repositories）。
4. Permissions → Repository permissions → **Contents: Read and write**。
5. Generate，复制 `github_pat_...` 那串，填进 `admin.html`。

> 小号 Token 长期有效，请勿截图外传、勿明文存文件。若小号 Token 疑似泄露，到小号 PAT 页面 Revoke 即可，不影响大号。

## 把文件部署 / 更新到 GitHub（首次或改完代码后）
本地改了 `apply.html` / `admin.html` / `jobs.json` 后，要推到 `j-chen-j/campus-referral` 仓库才生效。两种方式任选：

**方式 A（最省事，不用 Token）：GitHub 网页编辑器逐个提交**
1. 浏览器打开 `https://github.com/j-chen-j/campus-referral`。
2. 点对应文件名（如 `jobs.json`）→ 右上角铅笔图标「Edit」。
3. 把本地文件**全部内容**复制粘贴覆盖进去 → 底部「Commit changes」。
4. 对 `apply.html`、`admin.html`（若改过）重复同样操作。
5. 等约 30 秒，GitHub Pages 自动重建。

**方式 B（批量，需小号 classic token）：本地用 admin.html 写 jobs.json**
- 仅适用于改岗位：双击 `admin.html` → 贴 token → 加载/保存即自动写回 `jobs.json`（详见「HR 管理岗位」）。注意首次需仓库里已存在 `jobs.json` 文件。

> 部署后微信里若仍显示旧岗位，长按投递页「刷新」一次让 HTML 重新拉取即可。

## 改岗位后需要重新点激活邮件吗？
**不需要。** formsubmit.co 的激活是按「收件邮箱」绑定的，与岗位列表无关：
- 只改 `JOBS` 岗位下拉 → 收件邮箱没变 → 之前对该邮箱的激活一直有效，直接生效。
- 只有把 **formsubmit 的别名换成绑定另一个新邮箱**时，新邮箱才会收到一封激活邮件，需点一次。

## HR 邮箱如何做到"候选人爬不到"（formsubmit 隐形别名）
投递页 `apply.html` 里**不写任何邮箱**，只提交到 formsubmit 发放的**随机别名地址** `FORM_ENDPOINT`
（形如 `https://formsubmit.co/xxxxxxxx`）。该别名由 formsubmit 在邮箱确认后发放，与其背后的 HR
真实邮箱一一对应，但**从别名无法反查出邮箱**。
因此：候选人查看 `apply.html` 源码、或用爬虫抓 GitHub 仓库，都**拿不到 HR 邮箱**。

### 拿到别名的步骤（一次性）
1. 先把 `apply.html` 里的 `FORM_ENDPOINT` 临时填成 `https://formsubmit.co/你的HR邮箱`，提交一次。
2. 去该邮箱（**务必翻垃圾箱/广告邮件**）找 formsubmit 的「请确认你的邮箱」邮件，**点链接激活**。
3. 激活后 formsubmit 会给你一个**随机别名字符串**。
4. 把 `FORM_ENDPOINT` 改成 `https://formsubmit.co/<别名>`，再按「把文件部署到 GitHub」重新上传 `apply.html`。
5. 之后候选人提交都走这个别名，邮箱永不出现在源码里。

> **为什么不再用 Cloudflare Worker 中继**：实测 formsubmit 会按**来源 IP** 做反滥用限流——同一时刻、同一邮箱，
> 从真实用户 IP 直连返回 `200`，而经 Cloudflare Worker 的共享机房 IP 转发返回 **`429`**。
> 这意味着真实候选人走 Worker 同样会被限流、投不进来。改为候选人浏览器直连即可规避。

## 候选人视角
微信扫二维码 → 打开投递页（**页面不显示内推人姓名**）→ 填自己信息、选意向岗位、传简历 → 提交 → HR 邮箱收到邮件（正文含 `ref: 推荐人姓名` 和 `job: 意向岗位`）。

## 收件邮箱设置
HR 邮箱**不写在 `apply.html` 里**，只通过 formsubmit 的别名间接绑定（详见上节）。
需要换收件邮箱时：按上节步骤重新走一遍激活流程，拿到新别名，改 `FORM_ENDPOINT` 并重新上传 `apply.html` 即可。

## 隐私与合规
- 投递页不显示内推人姓名，但提交时 `ref` 作为隐藏字段发出，HR 邮件中可见。
- **HR 邮箱不在任何前端/仓库源码中**，源码里只有 formsubmit 的随机别名，候选人无法反查、无法爬取。
- formsubmit.co 为境外服务，简历会经境外中转；若对数据出境有合规要求，可改用腾讯云 CloudBase 等国内云函数做转发（邮箱只存服务端环境变量），前端改为提交到云函数地址即可。

## 文件说明
- `generator.html` 主程序（离线单文件，二维码/打包库已内联）
- `hr-jobs.html` **HR 改岗位页**（Edge 双击即用；改动直连 GitHub 写回 `jobs.json`，**令牌仅存 HR 本机浏览器，不进源码/不上传**）
- `apply.html` 投递页源码（已部署，运行时实时拉取 `jobs.json`，直连 formsubmit 别名提交）
- `jobs.json` 岗位数据源（唯一权威，改岗位只动它）
- `admin.html` 备用改岗位页（本地 + GitHub API，需 token，管理员自用）
- `cloudflare-worker.js` 岗位管理 API（读写 Cloudflare KV，**不需要 GitHub Token**；**不再**用于简历转发，
  简历已改为候选人浏览器直连 formsubmit 别名，避开其机房 IP 限流）

## 两套链路（别混淆）
| 链路 | 走哪 | 说明 |
|---|---|---|
| **简历投递** | 候选人浏览器 → **formsubmit 别名** → HR 邮箱 | 不经 Worker（Worker 机房 IP 会被 formsubmit 限流 429） |
| **改岗位** | HR 浏览器 → `hr-jobs.html` → GitHub `jobs.json`（令牌仅存本机浏览器） | 不经 formsubmit；令牌不出现在源码或仓库 |
