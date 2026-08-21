# 创作模式（mode）参考

`magic project create --mode <mode>` 接受的值，以及每种模式的数量含义、边界和输入要求。
数量边界与站点 `lib/creation-mode-contracts.ts` 一致；越界的数量会被服务端夹到区间内，
不会报错——所以**先在这里对一遍，别让用户以为自己拿到了 30 张**。

## 速查表

| mode | 中文名 | 数量代表什么 | 默认 | 范围 | 需要 clarify |
|---|---|---|---|---|---|
| `poster` | 海报 | 候选图张数 | 1 | 1–20 | 否（可直接生成） |
| `image-card` | 图文卡片 | 候选图张数 | 1 | 1–20 | 否 |
| `social-card` | 小红书多页笔记 | **页数** | 6 | 3–10 | 是 |
| `image-series` | 系列图 | 候选图张数 | 1 | 1–20 | 是 |
| `portrait` | 写真 | 候选图张数 | 1 | 1–20 | 是（需照片） |
| `portrait-series` | 写真系列 | 候选图张数 | 1 | 1–20 | 是（需照片） |
| `free` | 自由创作 | 候选图张数 | 1 | 1–20 | 视情况 |
| `brand-series` | 品牌系列 | 候选图张数 | 1 | 1–20 | 是 |
| `ecommerce-visual` | 电商视觉 | 候选图张数 | 1 | 1–20 | 是 |

`slide`（PPT）与 `fashion`（服饰设计）在站点上标记为 coming soon，`project create` 会拒绝；
服饰的多交付批次即使开放也只能在网页工作台生成，CLI v1 不支持（`magic generate` 会明确报错）。

## 「用户话术 → mode」判例

| 用户说 | 选 | 理由 |
|---|---|---|
| 「做张咖啡店开业海报」 | `poster` | 单张主视觉，有明确张贴/传播场景 |
| 「帮我出个封面图」 | `poster` | 单张，构图重于叙事 |
| 「写篇小红书，配图也一起」 | `social-card` | 多页笔记，数量是页数不是候选图 |
| 「种草帖，6 页那种」 | `social-card` | 「页」是关键词 |
| 「同一个主题给我一组图」 | `image-series` | 强调成组、风格统一 |
| 「产品的几个角度」 | `image-series` 或 `ecommerce-visual` | 电商投放场景选后者 |
| 「把我的照片做成写真」 | `portrait` | 必须先拿到用户本人照片 |
| 「花田/海边/唐风那种系列写真」 | `portrait-series` | 官方场景模版相册，可多选场景 |
| 「随便发挥，给我几个方向」 | `free` | 不锁模板，融合模板库版式语言 |
| 「品牌全套物料」 | `brand-series` | 成套、需要一致的品牌语言 |

判不准时不要猜：把 2 个候选连同区别一句话说给用户选，比选错后重做便宜。

## 输入要求

- **`portrait` / `portrait-series`**：必须有用户本人的清晰照片，用 `--ref <文件>` 传，
  并用 `--ref-prompt` 说明「这是身份参考，需要保留人物长相」。没有照片就先问用户要，
  不要用别人的照片替代。
- **`ecommerce-visual` / 产品类**：产品实拍图用 `--ref` 传，`--ref-prompt` 里写明
  「保留产品本身，不要重新设计」。
- **`poster` / `image-card` / `free`**：参考图是可选的风格参考。
- 文案很长时用 `--text-file <文件>` 或 `--text -`（读 stdin），避免 shell 引号地狱。

## 数量怎么定

1. 用户说了数字 → 就用那个数字（`--count N`）。
2. 用户没说 → 用表里的默认值；`social-card` 用 6 页。
3. 用户说「多来几个挑挑」→ 3–4 张是常见的性价比点；**先 `billing quote -n <N>` 报价**，
   超出余额就把可负担的张数告诉用户，让用户决定。

数量直接乘算花费。改数量前重新 quote，不要用上一次的报价。

## clarify 与模板选择

- `magic clarify open <id> --json` 返回 `status`、`reply`、`recommended_templates`、
  `selected_template_ids`、`image_count`、`min_image_count`、`brief`。
- `status` 走到 `ready`（`ready_for_plan: true`）才可以 `generate`。
- 选模板 + 定数量用 **`magic clarify select`**：它不跑模型，一次就把选择固化下来。

  ```bash
  magic clarify select <id> --template tpl_a --template tpl_b --count 4 --json
  ```

  `social-card` 用 `--pages N` 表示页数。
- `recommended_templates[].reason` 是站点给的推荐理由，直接拿它跟用户已表达的偏好对齐，
  能选就选，不要每次都把整张模板表甩给用户。
- 海报这类有版式槽位的作品，生成前可以 `magic structure get <id> --json` 看结构，
  改完再 `magic structure set <id> --file slots.json` 写回。这是**点数花出去之前**
  最后一个可编辑的地方。
