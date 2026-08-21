# 排错手册

## 退出码（稳定契约，按它分支）

| code | 含义 | 你该做什么 |
|---|---|---|
| 0 | 成功 | 继续 |
| 10 | 参数/用法错误 | 读 stderr 的提示改参数；`magic <命令> --help` 查全 |
| 20 | 未登录 / token 过期 | 让用户 `magic login`，转述 URL + 配对码，**不代填** |
| 21 | 余额不足 / 需升级 | 把点数价格和充值 URL 给用户，**停止**，绝不代买 |
| 30 | `awaiting_human` | 读 stdout 的问题；能代答 → `magic job answer`；否则问人 |
| 40 | job 失败 | `magic job list --json` 看 error 与 params → `job resume` 或改参 |
| 50 | 瞬时错误（网络/5xx/429，CLI 已重试） | 等一会儿重跑**同一条**命令 |
| 70 | CLI 内部错误 | 不是契约码；把 stderr 原文报给用户 |

`--json` 模式下失败也是一个 JSON 对象：`{"error":{"code":21,"message":"…"}}`。

## 退出码 20：登录过期

session token 有效期约 7 天，服务器端也可能被用户在
「个人资料 → 已连接设备」里手动断开。表现都一样：任何命令退出码 20。

处理：

```bash
magic auth status --json      # 确认确实是登录问题
magic login                   # 打印授权 URL + 配对码，用户在浏览器确认
```

把 URL 和配对码**原样**转述给用户。不要替用户打开登录页填写任何东西，
不要向用户索要密码或验证码。

`auth status --json` 里的 `expires_at` / `expires_in_days` 是这台设备的会话还剩多久。
快到期可以顺口提醒用户重新 `magic login`，但**不要**因此提前替他跑——授权必须由
用户在浏览器里确认。

## 退出码 21：余额不足

`magic billing quote --op image -n <N> --json` 返回：

```json
{"points":40,"balancePoints":12,"canAfford":false,"requiredPlanId":null}
```

告诉用户：这次要 40 点，账上 12 点，差 28 点，充值在 `https://magic-design.art/pricing`。
然后**停下来**。可以提议减少张数并重新 quote——那是用户能马上决定的事。

每个账号有一张免费图额度；单张生成会自动用掉它（批量生成不会）。

## 退出码 50 与限流

CLI 已经做了指数退避重试（上限 4 次，尊重 `Retry-After`）。看到 50 说明重试完还是失败。

- **等一会儿再重跑同一条命令**。`generate` 是幂等的，重跑不会重复扣费。
- 不要缩短间隔连打，不要并发多开 `magic` 进程刷同一个作品——那只会把限流打得更死。
- 设备登录轮询里的 `slow_down` 由 CLI 自己处理，你不需要管。

## 绝对不要自己写轮询

```
# 反面教材，不要这样做
while true; do magic job list "$ID" --json; sleep 2; done
```

用 `magic job wait <id> --json`。它按站点的节奏轮询（2.5s 起、线性升到 5s、总时限 15 分钟、
容忍连续 4 次瞬时失败），并把状态变化输出成 JSONL 事件：

```jsonl
{"event":"phase","jobId":"…","phase":"running","label":"生成中"}
{"event":"progress","jobId":"…","donePages":1,"totalPages":4}
{"event":"preview","jobId":"…","text":"…"}
{"event":"retry","jobId":"…","attempt":1,"total":3}
{"event":"awaiting_human","jobId":"…","question":{"text":"…"}}
{"event":"done","jobId":"…","assets":[{"id":"…","url":"…","thumb":"…"}]}
{"event":"failed","jobId":"…","error":"…"}
{"event":"timeout","pending":["…"]}
```

`timeout` 事件退出码 50，但**任务仍在服务端跑**——再 `job wait` 一次即可，不要重新 generate。

## `awaiting_human` 代答模板

`{"event":"awaiting_human","question":{"text":"…"}}` 之后退出码 30。常见问题类型：

| 问题类型 | 能不能代答 | 怎么答 |
|---|---|---|
| 文案细节（标题写什么、要不要加电话） | **能**，如果用户提过 | 从对话上下文原样回填；没提过就问人 |
| 语言（中文/英文/中英混排） | **能** | 跟用户输入的语言一致 |
| 事实信息（时间、地址、价格、联系方式） | **只能**用户给过的 | 上下文里没有就问人，**不要编** |
| 构图取舍（横版还是竖版、留白多少） | 一般能 | 按用途推断：海报竖版、社媒方图、封面横版 |
| 审美偏好（喜欢哪种色调、要不要更活泼） | **看情况** | 用户表达过倾向就照办；纯偏好且无依据 → 问人 |
| 人物/产品的身份细节 | **不能** | 一律问人；猜错等于生成了错的东西 |

代答后：

```bash
magic job answer "$ID" --job <jobId> -m "<回答>" --json
```

默认会自动继续等到结束；只想提交不想等就加 `--no-wait`。

## `job resume` 决策树

先看清楚状态：

```bash
magic job list "$ID" --json
```

```
有 active job?
├─ 是 → magic job wait          （别 resume，也别重新 generate）
└─ 否
   ├─ awaiting_human? → magic job answer（答完自动续等）
   └─ 有 failed job?
      ├─ error 是瞬时的（超时 / 上游不可用 / 网络）→ magic job resume
      ├─ error 是内容被拒（涉及敏感内容、参考图不合规）
      │   → 不要 resume。改 prompt / 换参考图，问清楚用户后重新 generate
      ├─ error 是参数问题（模板不存在、版式非法）
      │   → 修参数后重新 generate（这是新的一次购买）
      └─ 看不懂 → 把 error 原文转述给用户，让用户决定
```

`job resume` 会用失败任务记录下来的原参数重放，并**重新计费**——失败的那次已经退款了，
所以重放是一次诚实的新扣费，不是重复扣费。

## 图片拿不到 / 下载失败

- `assets download` 直接连 CDN 或私有桶的签名地址。403 通常是签名过期：
  重跑一次 `assets download`（会重新拿地址）即可。
- 报「没有直连地址」= 这个环境没配公共 CDN 也没配私有桶签名，是环境问题，
  **不要试图从 API 抓字节绕过**。把这个信息报给用户。

## `--mark` 被拒（退出码 10）

局部标注的坐标是 **0–1 的比例**，不是像素。`--mark "1200,800,300,200=改这里"` 会被
直接拒绝，而不是被截断成边角——这是故意的：一个悄悄改错位置的编辑比一次报错贵得多。

- 拿到像素坐标就自己除一下：`x/图宽`、`y/图高`、`w/图宽`、`h/图高`。
- 字段缺一不可：`x,y,w,h=说明` 或 `x,y=说明`，`=` 后面必须有说明文字。
  空字段（`0.2,,0.3,0.1=…`）会报错，不会被当成 0。
- `=` 只按**第一个**分割，说明里可以照常写等号。
- `--marks-file` 里 `note` 不能是空字符串——没说明的框等于花钱让模型猜，会直接报错。
- 超过 10 处、或某处说明超过 300 字会被拒——和网页端同一套上限，不是 CLI 自己加的。
- 报「坐标像是像素」时，先确认你真的看过那张图：`magic assets download` 落地后再标。

## `--mark-ui` 退出码 10：「已取消标注」

用户点了取消、关掉了页面，或者 15 分钟没提交。**没有生成、没有扣点数**，这不是错误，
是用户改主意了——回头问他想怎么改，不要换成盲标坐标再跑一次。

其他 `--mark-ui` 情况：

- **浏览器没自动打开**：CLI 已经把 URL 打印在 stderr 上了，转述给用户。服务器 /
  容器里跑就直接加 `--no-open`。
- **用户打不开这台机器的 127.0.0.1**（远程会话、容器）：`--mark-ui` 用不了，
  退回 `--mark` 给坐标，并且必须先 `magic assets download` 看过图。
- **「当前环境没有图片直连地址」**：这个环境没配 CDN / 签名地址，页面拿不到图。
  用 `--mark` 给坐标。
- **提交后页面提示不合法**：页面守的是和 `--mark` 同一套上限（≤10 处、说明 ≤300 字），
  页面不会关，用户删掉多余的框再提交即可。
- 页面合成的标记图只是给模型看的示意，**不进幂等键**；幂等键只认坐标和说明文字。

## 幂等与「我是不是重复扣费了」

- `generate` / `edit` / `job answer` / `job resume` 内部按「作品 + 意图指纹」维护
  `operation_id`。**这个 id 只在「上一次没提交成功」时被复用**：请求超时 / 断网 /
  退出码 50 时它还开着，原样重跑复用同一个 id，服务端去重，不会重复扣费。
- 服务端已经接下提交之后，同一条命令再跑一次就是一次新的购买——CLI 不会、也不应该
  拦下它，因为「再生成一版」本来就是合法请求。恢复请用 `job list` / `job wait`。
- 改了任何实质参数（数量、模板、版式、改图指令文字、`--mark` 的坐标或说明）同样是新的
  购买——因为那确实是不同的图。
- 你永远不需要、也不应该自己生成或传 `operation_id`。

## 其他

- `magic auth status --json` 看当前站点、账号、token 到期时间。
- 想连非生产站点：`--base-url https://…` 或环境变量 `MAGIC_ART_BASE_URL`。
  凭据是按站点存的，换站点等于未登录。
- 升级：`npm i -g @magic-art/cli@latest && magic skill install`
  （skill 文件随 CLI 发布，保持同版本）。技能正本写在 `~/.agents/skills/magic-art`，
  再软链到你机器上已安装的每个 agent，所以这一条命令就把所有 agent 一起升级了。
