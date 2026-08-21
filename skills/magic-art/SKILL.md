---
name: magic-art
description: >
  用 Magic Art (magic-design.art) 把一个想法做成成品视觉作品：AI 海报、小红书图文
  卡片、写真、系列图，并把成图下载到本地。当用户想生成海报/配图/写真/系列视觉、
  提到 Magic Art 或 magic-design.art 时使用。全部操作通过 `magic` CLI 完成。
---

# 用 Magic Art 创作

Magic Art 自己也是一个带人机断点的 agent：它会提问（clarify）、推荐模板、确认数量，
生成途中还可能停下来要补充信息。你的工作是**在这些断点上决定「谁来答」**——能从对话
上下文答的你直接答，只有真正需要人的判断才回头问用户。

## 前置

1. `magic auth status --json`
   - 退出码 20 → 让用户运行 `magic login`，把 CLI 打印的 URL 和配对码**原样转述**给用户，
     由用户在浏览器完成授权。**不要代填验证码、密码或任何凭据。**
   - 首次使用可先 `magic skill install` 更新本技能到与 CLI 同版本。
2. 任何生成前先报价：
   `magic billing quote --op image -n <数量> --json`
   - 退出码 21（`canAfford=false` 或 `requiredPlanId` 非空）→ 把点数价格和充值链接
     告诉用户并停止。**绝不代买、代升级。**

## 选择 mode（详见 references/modes.md）

| 用户在说 | mode |
|---|---|
| 海报、宣传图、封面、单张主视觉 | `poster` |
| 小红书图文、多页笔记、种草帖 | `social-card` |
| 一个主题要多张成一组、系列图 | `image-series` |
| 人像写真、把我的照片变成⋯（需用户照片） | `portrait` / `portrait-series` |
| 说不清、想自由发挥 | `free` |

拿不准就先 `magic styles list --kind <mode> --json` 看看这个模式下有哪些风格，
或 `magic search --json` 一次拿到全部可选面。

## 标准流程

```bash
magic project create --mode poster --text "<用户想法原文>" [--ref 图片]...   # → 打印作品 id
magic clarify open <id> --json                                              # 需要厘清的模式
magic clarify send <id> -m "<你的回答>" --json                              # 循环到 ready_for_plan
magic generate <id> --count 4 --wait --json                                 # 提交并跟到结束
magic assets download <id> --out ./out                                      # 成图落到本地
```

1. **创建**：`magic project create --mode <mode> --text "<用户想法原文>"`。
   用户给了参考图就 `--ref <文件>`（可重复），并用 `--ref-prompt` 说明这些图的用途
   （是风格参考还是要保留的产品/人物）。分辨率只能在这一步定（`--resolution 1k|2k|4k`），
   之后不可更改。
2. **厘清**（`social-card` / `image-series` 等需要 clarify 的模式）：
   `magic clarify open <id> --json` 拿到开场问题，然后循环 `clarify send` / `clarify select`
   直到返回 `ready_for_plan: true`。
   - 选模板/定数量用 `magic clarify select <id> --template <id> --template <id> --count 4`
     （不走模型，快且确定）。
   - **代答准则**见下面「谁来答」。
3. **生成**：`magic generate <id> --count <N> --wait --json`。
   - 不加 `--wait` 只提交并打印 jobId，之后用 `magic job wait <id> --json` 跟。
   - 退出码 30（`awaiting_human`）：读 stdout 里的问题，能代答就
     `magic job answer <id> --job <jobId> -m "<回答>"`（默认自动续等）；否则转述给用户。
   - 退出码 40：`magic job list <id> --json` 读 error 与可重放 params，
     `magic job resume <id>` 重试，或调整参数重新生成。
4. **交付**：`magic assets download <id> --out <目录>`，把本地文件路径交给用户。
5. **改图**：`magic edit <id> --parent <assetId> --prompt "<修改指令>"`（同步返回新图），
   再 download。整图讲不清、只想动某一块时改用**局部标注**（下一节）。

## 局部标注修改

用户说「就这块不对」时，不要把「改左上角那个图标」写进 `--prompt` 让模型自己找——
直接把位置标出来：

```bash
magic assets download "$ID" --asset <assetId> --out ./out   # 先把图拿到本地并看过
magic edit "$ID" --parent <assetId> \
      --mark "0.62,0.08,0.3,0.12=这行标题换成「限时 8 折」" \
      --mark "0.1,0.75=这个角落太空，加一个小图标" --json
```

- 坐标是**相对比例 0–1**，原点在左上：`x,y,w,h=说明` 框一块，`x,y=说明` 点一个位置。
  写像素（`1200,800,...`）会被直接拒绝，不会被当成比例悄悄改错地方。
- 一次最多 10 处，每处说明 ≤300 字。多处标注按 ①②③ 的顺序交给模型。
- 有 `--mark` 时 `--prompt` 可以省；两个一起给就是「整体要求 + 逐处要求」。
- 标注多时写进文件：`--marks-file marks.json`（JSON 数组，字段 `x,y,w,h,note`）。

**铁规则：没看过图就不要给坐标。** 你手里只有 asset id 时坐标全是猜的——先
`magic assets download` 落到本地、把图读进来看清元素在哪，再标。用户自己描述了位置
（「右下角的二维码」）也要先看图确认那里确实是二维码。

改一处坐标或一个字，都算一次新的渲染、一次新的扣费；原样重跑同一条命令才复用上次的
`operation_id`。

### 让用户自己框（`--mark-ui`）

坐标写在命令行里，用户是看不懂对不对的——等他发现框错了，钱已经花掉了。
**位置有任何不确定，就把框交给用户确认**，加 `--mark-ui`：

```bash
magic edit "$ID" --parent <assetId> \
      --mark "0.62,0.08,0.3,0.12=这行标题换成「限时 8 折」" \
      --mark-ui --json
```

CLI 会在本地起一个只监听 127.0.0.1 的页面并打开浏览器，把原图和你上面给的框**画好**
摆在用户面前。用户可以拖动/新增/删除框、逐个写说明，点「提交并生成」之后 CLI 才继续；
提交时一并把浏览器合成的**带 ①②③ 标记的示意图**回传给模型，和网页端标注改图完全一样。

- **`--mark` 可以不给**：什么都不确定时直接 `--mark-ui`，让用户从零框。
- **点「取消」或关掉页面 = 退出码 10，不生成、不扣点数。** 这时不要自作主张改用
  盲标坐标重跑，问用户想怎么改。
- 服务器上跑、开不了浏览器：加 `--no-open`，CLI 把 URL 打印出来给用户自己开
  （需要用户能访问这台机器的 127.0.0.1；不能就退回 `--mark` 给坐标）。
- 15 分钟没提交自动超时，同样按取消处理。

## 谁来答（断点策略）

| 云端断点 | 你的策略 |
|---|---|
| clarify 开场 / 追问 | **代答优先**：从外层对话里提炼答案回填；信息确实缺失才问人 |
| 模板推荐（`selecting_templates`） | 代选：按 `recommended_templates` 的 reason + 用户已表达的审美倾向；用户说过「给我几个方向挑」才转述选项 |
| 数量确认（`selecting_count`） | 代答：用户说了几张就几张，没说用 mode 默认值 |
| 生成中 `awaiting_human` | 先代答；纯主观偏好且上下文里没依据 → 问人 |
| 局部标注框得准不准 | **框不准就交给人**：位置有不确定就 `--mark-ui` 让用户在图上确认/改框，别拿猜的坐标去花钱 |
| 余额不足 / 需升级 | **必须问人**：报价 + 充值 URL，绝不代买 |
| 公开发布 / 永久删除 | **必须问人**（外发、不可逆） |
| 登录 | 只转述 URL + 配对码，**不代填任何凭据** |

## 硬规则

- **只用 `magic` CLI**，禁止直接 curl 站点 API。认证头、幂等 `operation_id`、轮询节奏、
  错误码语义全在 CLI 里，绕过必坏，且会踩限流。
- **花钱先报价**：生成消耗魔法点，`billing quote` 先行；退出码 21 一律停下来问人。
- **公开发布、永久删除**（`magic project delete --yes`）：先问用户。
- **不要自己写 sleep 轮询循环**：等待一律用 `magic job wait`，它按站点的节奏轮询。
- **中断恢复**：先 `magic job list <id> --json`——有 active job 就 `job wait`，
  有 failed 才 `job resume`；**不要直接重新 `generate`**（那是再买一次）。
- **给坐标前必须先看图**：`--mark` 的坐标只能来自你实际看过的那张图，不能靠 asset id 猜。
- **图片一律通过 `magic assets download` 落地**（CLI 拿到的是 CDN / 私有桶签名直连地址），
  不要尝试从 API 抓字节。
- 每条命令的完整参数用 `magic <命令> --help` 现查，不要凭记忆拼参数。

## 参考

- `references/modes.md` — 各 mode 的适用场景、数量边界、输入要求、话术判例
- `references/recipes.md` — 端到端配方（命令序列 + 期望输出）
- `references/troubleshooting.md` — 退出码手册、限流、token 过期、`awaiting_human` 代答模板、
  `job resume` 决策树
