# WorkBuddy 每日自动签到（云端版）

每天自动领取 WorkBuddy 每日积分，**跑在 GitHub Actions 上，电脑关机也照跑**。

本项目基于 [88lin/workbuddy-auto-signin](https://github.com/88lin/workbuddy-auto-signin) 改造。原版依赖本机登录态、只能本机运行；这一版把你**自己的**登录凭据放进仓库的加密 Secret，交给 GitHub 云端定时执行，不必自备服务器、不必装 Python。

---

## 它做什么

| 时间（北京时间） | 动作 |
|---|---|
| 每天 00:05 | 签到 + 成长中心（领旅行礼物 / 派 Buddy / 领任务奖 / 连登兑换 / 开盲盒） |
| 每天 08:00 | 再跑一次，补上错过的那轮 |

每次运行写一行 JSON 日志。**失败时会在你的仓库里自动开一个告警 issue，下次成功运行时自动关闭**——不需要你天天盯。

---

## 前置条件

- 你自己的 WorkBuddy 账号，并且**在桌面端登录过至少一次**（登录后凭据文件才会生成）
- 一个 GitHub 账号（免费版够用，Actions 额度完全撑得住）
- 不需要 Python、不需要服务器、不需要开着电脑

---

## 部署（四步）

### 第 1 步 · 从你自己的电脑导出凭据

先确认 WorkBuddy 桌面端处于**已登录**状态，然后找到这个文件：

| 系统 | 路径 |
|---|---|
| Windows | `%LOCALAPPDATA%\CodeBuddyExtension\Data\Public\auth\workbuddy-desktop.info` |
| macOS | `~/Library/Application Support/CodeBuddyExtension/Data/Public/auth/workbuddy-desktop.info` |
| Linux | `~/.config/CodeBuddyExtension/Data/Public/auth/workbuddy-desktop.info` |

Windows 上通常就是：

```
C:\Users\<你的用户名>\AppData\Local\CodeBuddyExtension\Data\Public\auth\workbuddy-desktop.info
```

用记事本打开，**全选、复制里面的全部内容**（一整段 JSON），下一步要用。

> [!WARNING]
> 这个文件等同于你的账号密码——里面有 `accessToken`、`refreshToken`，还有你的手机号。**不要发给任何人、不要贴进聊天窗口、不要提交到 git。** 第 3 步粘到 GitHub Secret 里是加密存储，那才是它该待的地方。

### 第 2 步 · 建一个你自己的仓库

**方式 A：Import repository（推荐，可以设成私有）**

1. 打开 https://github.com/new/import
2. **Your old repository's clone URL** 填 `https://github.com/zqw666/workbuddy-auto-signin`
3. **Repository name** 填你喜欢的名字，例如 `wb-signin`
4. 可见性选 **Private**（推荐）或 Public
5. 点 **Begin import**，等它跑完

**方式 B：直接 Fork（更省事，但有两个差别）**

点仓库右上角的 **Fork**。注意：

- Fork 出来的仓库**必定是 public，而且不能单独改成 private**
- Fork 出来的仓库 **Actions 默认是关闭的**，必须去 Actions 页面手动启用一次，否则永远不会自动跑

### 第 3 步 · 把凭据存进 Secret

**网页操作**（不需要装任何工具）：

1. 进入你刚建好的仓库 → **Settings** → 左侧 **Secrets and variables** → **Actions**
2. 点 **New repository secret**
3. **Name** 填 `WORKBUDDY_AUTH_JSON`（一个字符都不能差）
4. **Secret** 粘贴第 1 步复制的**整段 JSON**
5. 点 **Add secret**

**命令行操作**（本机装了 `gh` 的话）：

```bash
gh secret set WORKBUDDY_AUTH_JSON --repo <你的用户名>/<你的仓库名> < "凭据文件的路径"
```

### 第 4 步 · 手动跑一次，确认通了

进入仓库 → **Actions** 标签 → 左侧选 **WorkBuddy 每日签到** → 右侧 **Run workflow** → 绿色按钮。

等一分钟左右，点进这次运行记录，看输出：

| 你看到 | 含义 |
|---|---|
| `账号 [default] 凭据就绪：uid=xxxx****xxxx 昵称=...` | 凭据读到了，uid 已自动打码 |
| `今日已签过` 或 `成功领取 N 积分` | **成功了**，之后会自动按点跑 |
| `NO_AUTH / 未找到 WorkBuddy 登录凭据` | Secret 没配对——回去检查名字是不是 `WORKBUDDY_AUTH_JSON`、内容是不是整段 JSON |
| `NO_SESSION / HTTP 401` | 凭据过期了——看下面一节 |

---

## 每 30 天要重新导出一次凭据

这是硬限制，先说在前面：

| 凭据 | 有效期 |
|---|---|
| `accessToken` | **30 天** |
| `refreshToken` | 60 天 |

签到脚本只读取 `accessToken` 直接调接口，**本身不会续期**。所以 Secret 里那份副本大约一个月后就会失效，日志会出现 `NO_SESSION`，签到随之停摆。

到期后把**第 1 步和第 3 步重做一遍**就行：登录桌面端 → 复制文件内容 → 更新 Secret，两三分钟的事。

> [!NOTE]
> 桌面端会用本地凭据文件自行刷新 token，但**刷新的只是你电脑上那份文件，Secret 里的副本不会跟着变**。两边是各自独立的，所以仍然需要你手动同步一次。

### 想管多个账号

`Secret` 里改填一个 JSON 对象即可：键是账号标签，值是各自的整段凭据。

```json
{
  "me":   { "account": { "...": "..." }, "auth": { "...": "..." } },
  "work": { "account": { "...": "..." }, "auth": { "...": "..." } }
}
```

工作流会自动拆开、**逐个串行**执行并合并日志。注意必须串行——脚本内部有全局的时间预算状态，并行会互相干扰。

---

## 本机手动运行（可选）

调试或想立刻领一次时，在本机跑：

```bash
python signin.py auto     # 签到 + 成长中心
python signin.py status   # 只查签到状态（不发写请求）
python signin.py growth   # 只跑成长中心，不签到
```

不需要任何第三方包，任意 Python 3 均可。本机跑会直接读取你本机的凭据文件，不用配 Secret。

---

## 工作原理

登录后，WorkBuddy 桌面端会写出明文会话文件 `workbuddy-desktop.info`。脚本流程：

1. **定位**凭据文件（自动探测四个平台路径，或用环境变量 `WORKBUDDY_AUTH_FILE` 指定）
2. **查询** `POST /v2/billing/meter/checkin-activity-status` —— 今天签过没
3. **领取** 若未签，`POST /v2/billing/meter/daily-checkin`
4. **成长中心** 领旅行礼物 → 派 Buddy → 领任务奖 → 连登兑换 → 开盲盒
5. **输出** 一行 JSON，`report` 字段是人话汇报

在 GitHub Actions 上跑的时候，工作流把 Secret 里的 JSON 拆成单独的凭据文件（权限 0600）放在临时目录，通过 `WORKBUDDY_AUTH_FILE` 指给脚本；跑完立刻删掉临时目录。脚本进程本身**拿不到 Secret**，只拿到一个文件路径。

> [!NOTE]
> 签到接口系从桌面端 `app.asar` 逆向得到，请求都打到官方客户端用的同一个 endpoint（`https://copilot.tencent.com`）。
>
> 查询类请求失败会自动重试一次；领取、抽奖这类写操作**不重试**，避免超时发生在服务端已经处理完之后造成重复提交。签到领取接口本身是幂等的，所以允许重试。

---

## 配置

| 环境变量 | 作用 |
|---|---|
| `WORKBUDDY_AUTH_FILE` | 手动指定凭据文件路径（自动探测失败时用） |
| `WORKBUDDY_SIGNIN_LOG` | `silent` 模式下日志文件路径（默认 `signin.log`） |
| `WORKBUDDY_BUDGET_SECONDS` | 单次运行的时间预算，默认 `420` 秒。必须小于定时任务的执行上限，超过会被自动夹到安全值 |
| `WORKBUDDY_GROWTH_LOG_EMPTY` | 设为 `1` 时，成长中心空跑也写日志（默认只在领到东西或出错时记录） |

---

## 排错

| 现象 | 处理 |
|---|---|
| `NO_AUTH / 未找到登录凭据` | Secret 名或内容不对；确认名字严格是 `WORKBUDDY_AUTH_JSON`，内容是整段 JSON |
| `NO_AUTH / WORKBUDDY_AUTH_FILE 指向的文件不存在` | 环境变量路径写错了，核对输出里 `looked_in` 列出的实际路径 |
| `NO_SESSION / HTTP 401\|403` | 凭据过期，重新登录桌面端并重新导出更新 Secret（见上文 30 天那一节） |
| Actions 页面一直没有运行记录 | 若是 Fork 来的仓库，Actions 默认关闭——去 Actions 标签手动启用一次 |
| `INACTIVE / 签到活动未开启` | 非签到季，正常，无需处理 |
| `NETWORK / 网络不可达` | 断网或服务端抖动，**不是**登录问题，下次运行会自动重试 |
| `TIMEOUT / 已达本次运行时间预算` | 网络严重超时导致主动收尾，已领到的部分照常记录 |
| `ERROR / 登录凭据文件不是合法 JSON` | Secret 里粘的内容不完整——重新复制一次完整文件内容 |
| 轮询日志里写着「今日旅行名额已用完」 | 服务端每日只放行一次 Buddy 派出，当天派过就是这样，属正常 |
| 想看原始返回 | 本机跑 `python signin.py status` |

---

## 安全与隐私

- 脚本**不含、不内嵌**任何凭据；运行时才去读凭据文件
- 凭据以 GitHub Secret 形式加密存储，网页上**无法读回明文**，只有 Actions 运行期间能取到
- 日志里的 uid 会自动打码（前 4 位 + 后 4 位），任何模式下都不会打印 token
- 工作流的触发方式只有**定时**和**手动**，没有 `pull_request`，因此不存在「别人开个 PR 偷走 Secret」这类攻击路径
- 跑完立即删除临时凭据目录

> [!WARNING]
> 但要清楚一点：**如果仓库是 Public，Actions 的运行日志和告警 issue 是所有人都能看的**，日志里会带账号标签和昵称。介意的话请把仓库设为 **Private**（Private 仓库的 Actions 额度是每月 2000 分钟，本项目每天两次、每次一两分钟，一个月约 120 分钟，绰绰有余）。

---

## 免责声明

> [!WARNING]
> 本项目为**非官方**工具，与腾讯及 WorkBuddy 无任何隶属关系。签到接口系从桌面端 `app.asar` 逆向得到。使用风险自负；接口可能随时变动且不另行通知。请遵守相关服务条款。

---

## 致谢

签到脚本主体来自 [88lin/workbuddy-auto-signin](https://github.com/88lin/workbuddy-auto-signin)。本仓库在其基础上增加了 GitHub Actions 云端定时执行、多账号支持与失败告警 issue 链路。
