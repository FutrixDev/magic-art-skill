# 配方（端到端命令序列）

每个配方都可以照抄。`$ID` 是 `project create` 打印在 stdout 的作品 id。
所有命令都带 `--json` 时，stdout 是可解析的；人类说明一律在 stderr。

---

## 1. 单张海报（最短路径）

用户：「帮我做一张咖啡店开业海报，主色调米白，周六上午十点开门。」

```bash
magic billing quote --op image -n 1 --json          # 退出码 21 → 停下来问人
ID=$(magic project create --mode poster \
      --text "咖啡店开业海报，主色米白，周六上午十点开门，写「新店开业」" )
magic generate "$ID" --count 1 --wait --json         # JSONL 事件流，结束打印产物
magic assets download "$ID" --out ./out
```

期望：`generate --wait --json` 最后一行是 `{"event":"done","jobId":…,"assets":[{"id":…,"url":…}]}`，
退出码 0；`assets download` 打印本地文件路径。

海报模式不需要 clarify，可以直接生成。用户想先看看风格选项时：

```bash
magic styles list --kind poster --json
magic generate "$ID" --template <风格 id> --count 3 --wait --json
```

---

## 2. 小红书多页笔记（走 clarify）

用户：「写个小红书，推荐我家楼下这家面包店，6 页。」

```bash
magic billing quote --op image -n 6 --json
ID=$(magic project create --mode social-card --text "推荐楼下的面包店，6 页笔记，真实探店口吻")
magic clarify open "$ID" --json                      # 读 reply 里的问题
magic clarify send "$ID" -m "受众是附近上班族；重点是可颂和酸种；有实拍图" --json
magic clarify select "$ID" --template <推荐里选一个> --pages 6 --json
magic clarify status "$ID" --json                    # ready_for_plan: true 才继续
magic generate "$ID" --wait --json
magic assets download "$ID" --out ./out
```

要点：
- `clarify open` 的 `recommended_templates[].reason` 就是选模板的依据，能选就代选。
- `clarify select` 不跑模型，是把「选了哪几个模板 / 多少页」定下来的快通道。
- 页数是 3–10，越界会被夹住。

---

## 3. 用参考图做系列图

用户：「这是我们的产品照，给我一组 4 张不同场景的图。」

```bash
magic billing quote --op image -n 4 --json
ID=$(magic project create --mode image-series \
      --text "同一款保温杯的 4 个使用场景：办公桌、露营、健身房、通勤" \
      --ref ./cup.jpg --ref-prompt "这是产品实拍，必须保留产品本身外观，不要重新设计")
magic clarify open "$ID" --json
magic clarify send "$ID" -m "场景如上，风格干净自然光，不要浓重滤镜" --json
magic generate "$ID" --count 4 --wait --json
magic assets download "$ID" --out ./out
```

要点：`--ref` 可以重复给多张；`--ref-prompt` 是告诉模型这些图**扮演什么角色**
（风格参考？要保留的主体？），漏了它模型会当成随便的灵感图。

---

## 4. 生成中被问了问题（`awaiting_human`）

```bash
magic generate "$ID" --count 4 --wait --json
# → 最后一行 {"event":"awaiting_human","jobId":"job_x","question":{"text":"标题想用中文还是英文？"}}
# → 退出码 30
```

能代答就代答，然后自动继续等：

```bash
magic job answer "$ID" --job job_x -m "用中文，正文可以有少量英文点缀" --json
```

答不了（纯主观偏好，上下文里没依据）就把问题原样转述给用户，拿到回答再 `job answer`。

注意 `awaiting_human` 会**中止整批等待**，但其余 slot 仍在服务端跑。答完之后
`magic job wait "$ID" --json` 会把剩下的一起跟完。

---

## 5. 改图迭代

用户：「第二张不错，但把标题换成「限时 8 折」，颜色再暖一点。」

```bash
magic assets list "$ID" --json                       # 拿 asset id
magic edit "$ID" --parent <assetId> \
      --prompt "标题改成「限时 8 折」，整体色温再暖一点，其他保持不变" --json
magic assets download "$ID" --asset <新 assetId> --out ./out
```

`edit` 是**同步**的：命令返回时新图已经生成好了（最长等 5 分钟），没有 job 要跟。
它同样花点数，改之前也该 quote。

### 5b. 只改画面里的某一块（局部标注）

用户：「整体挺好，就右上角那行标题和左下角空着的地方要动一下。」

整图 prompt 会让模型顺手把别处也重画。把位置标出来：

```bash
magic assets download "$ID" --asset <assetId> --out ./out   # 必须：先把图看了
# 看清那行标题在右上、大约占宽 30% / 高 12%，左下角空白在 y≈0.75
magic edit "$ID" --parent <assetId> \
      --mark "0.62,0.08,0.3,0.12=这行标题换成「限时 8 折」" \
      --mark "0.1,0.75=这里太空，加一个咖啡杯小图标" \
      --json
magic assets download "$ID" --asset <新 assetId> --out ./out
```

标注多的时候写文件，`--marks-file marks.json`：

```json
[
  { "x": 0.62, "y": 0.08, "w": 0.3, "h": 0.12, "note": "标题换成「限时 8 折」" },
  { "x": 0.1, "y": 0.75, "note": "加一个咖啡杯小图标" }
]
```

要点：
- 坐标是 0–1 的比例，左上角是原点；`x,y,w,h` 框一块，只给 `x,y` 是点一个位置。
  给像素值会报退出码 10，不会被当成比例悄悄改错地方。
- **坐标必须来自你看过的图**。只有 asset id 时坐标是猜的，猜错就是花钱改错地方。
- 一次最多 10 处、每处 ≤300 字；有 `--mark` 时 `--prompt` 可省，也可以两个一起给
  （`--prompt` 是整体要求，`--mark` 是逐处要求）。
- 幂等键含坐标：原样重跑复用上次的 `operation_id`，动一个数字就是一次新的扣费。

---

### 5c. 让用户自己在图上框（`--mark-ui`）

用户：「这里、还有这里，不太对。」——位置说不清，或者你对自己框的位置没把握。

不要猜坐标然后花钱试。把图摆到用户面前让他自己框：

```bash
magic edit "$ID" --parent <assetId> \
      --mark "0.62,0.08,0.3,0.12=这行标题好像要改" \
      --mark-ui --json
```

发生了什么：

1. CLI 在 127.0.0.1 上起一个临时页面并打开浏览器，图直接从 CDN / 签名地址加载。
2. 你给的 `--mark` 已经画在图上了（没给也行，用户从零框）。用户拖框、写说明。
3. 用户点「提交并生成」→ CLI 拿到最终的框 **和浏览器合成的 ①②③ 标记图**，
   一起发给模型；这时才开始渲染、才扣点数。
4. 用户点「取消」/ 关掉页面 / 15 分钟没动 → 退出码 10，**没生成、没扣费**。

```
$ magic edit prj_x --parent ast_9 --mark-ui --json
等待你在页面上标注并提交⋯（关掉页面 = 取消）
{"asset":{"id":"ast_10","parent_id":"ast_9","url":"https://…"}}
```

取消之后不要拿盲标坐标重跑一遍——回头问用户想改什么。

服务器 / 无头环境：加 `--no-open`，CLI 只打印 URL；用户能访问这台机器的 127.0.0.1
才可用，否则退回 5b 的 `--mark` 给坐标。

---

## 6. 中断后恢复（agent 崩了 / 用户关了终端）

**不要重新 `generate`**——那是再买一次。

```bash
magic job list "$ID" --json
```

- 有 `active` → `magic job wait "$ID" --json`
- 有 `awaiting_human` → 先 `magic job answer`
- 只有 `failed` → `magic job resume "$ID" --json`
- 都没有、`assets` 已经有货 → 直接 `magic assets download`

`generate` 的幂等只覆盖「上一次没提交成功」：请求没拿到响应（超时、断网、退出码 50）时
`operation_id` 还开着，原样重跑会复用它，不会重复扣费。**服务端一旦接下了这次提交，
再跑一次 `generate` 就是一次新的购买**（和在网页上再点一次生成一样）——所以恢复只走上面的
`job list` / `job wait`。改了参数（数量/模板/版式）当然更是新的购买，因为那本来就是不同的图。
