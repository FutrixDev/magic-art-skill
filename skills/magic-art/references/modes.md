# Creation mode reference

The values `magic project create --mode <mode>` accepts, and for each: what the count means,
its bounds, and what input it needs. The bounds match the site's `lib/creation-mode-contracts.ts`;
an out-of-range count is clamped server-side rather than rejected — so **check here first,
instead of letting the user believe they are getting 30 images**.

## Quick reference

| mode | What it makes | What the count means | Default | Range | Needs clarify |
|---|---|---|---|---|---|
| `poster` | Poster | Candidate images | 1 | 1–20 | No (generate directly) |
| `image-card` | Image card | Candidate images | 1 | 1–20 | No |
| `social-card` | Xiaohongshu multi-page note | **Pages** | 6 | 3–10 | Yes |
| `image-series` | Image series | Candidate images | 1 | 1–20 | Yes |
| `portrait` | Portrait shoot | Candidate images | 1 | 1–20 | Yes (needs a photo) |
| `portrait-series` | Portrait series | Candidate images | 1 | 1–20 | Yes (needs a photo) |
| `free` | Free creation | Candidate images | 1 | 1–20 | Depends |
| `brand-series` | Brand series | Candidate images | 1 | 1–20 | Yes |
| `ecommerce-visual` | E-commerce visual | Candidate images | 1 | 1–20 | Yes |
| `film` | Film Studio (recurring characters and places) | **Frames per camera** | 1 | 1–12 total (`cameras × count`) | No — `magic film plan` replaces clarify |
| `logo` | Logo 工坊 (brand marks) | **Candidates per direction** | 2 (× 3 directions) | 1–5 per direction, 1–5 directions (or one per `--mark-types` entry — up to six with `auto` alongside all five types) | No — `magic logo generate` carries the brief |

`slide` (decks) and `fashion` (apparel design) are marked coming soon on the site and
`project create` rejects them. Even once apparel opens up, its multi-delivery batches can only
be generated in the web workspace; CLI v1 does not support them (`magic generate` says so
explicitly).

## Routing calls: what the user said → which mode

| The user says | Pick | Why |
|---|---|---|
| "Make a poster for a coffee shop opening" | `poster` | Single hero visual with a clear posting/sharing context |
| "I need a cover image" | `poster` | Single image; composition matters more than narrative |
| "Write me a Xiaohongshu post, with the images" | `social-card` | Multi-page note; the count is pages, not candidates |
| "A recommendation post, the 6-page kind" | `social-card` | "Page" is the keyword |
| "Give me a set of images on one theme" | `image-series` | Emphasis on a coherent set with one style |
| "A few angles on the product" | `image-series` or `ecommerce-visual` | The latter when it's for e-commerce placement |
| "Turn my photo into a portrait shoot" | `portrait` | Requires the user's own photo up front |
| "A series shoot — flower field / seaside / Tang-dynasty style" | `portrait-series` | Official scene-template albums; multiple scenes can be selected |
| "Just run with it, give me a few directions" | `free` | Doesn't lock a template; blends the layout vocabulary of the library |
| "A full set of brand collateral" | `brand-series` | A matched set needing one consistent brand language |
| "Draw this scene — same character in each shot" | `film` | The same identity has to survive across images, which only cards can guarantee |
| "A short comic / storyboard / shot list" | `film` | Several shots sharing characters, places and a look |
| "The same product, five different scenes" | `image-series` or `ecommerce-visual` | A product is a reference photo, not a character card — no benches needed |
| "I need a logo / brand mark / app icon" | `logo` | A mark is drawn wordless and transparent, then packed — a different deliverable from a poster of a name |

When it isn't clear, don't guess: naming the two candidates with a one-line difference and
letting the user pick is cheaper than picking wrong and redoing it.

## Input requirements

- **`portrait` / `portrait-series`**: requires a clear photo of the user themselves, passed
  with `--ref <file>`, with `--ref-prompt` saying "this is an identity reference; the person's
  likeness must be preserved". With no photo, ask the user for one — never substitute someone
  else's.
- **`ecommerce-visual` and product work**: pass real product shots with `--ref`, and state in
  `--ref-prompt` that the product itself must be preserved and not redesigned.
- **`poster` / `image-card` / `free`**: reference images are optional style references.
- For long copy, use `--text-file <file>` or `--text -` (reads stdin) and skip shell quoting hell.

## Choosing the count

1. The user gave a number → use it (`--count N`).
2. They didn't → use the default from the table; `social-card` uses 6 pages.
3. They said "give me a few to choose from" → 3–4 is the usual sweet spot. **Quote first**
   (`billing quote -n <N>`); if it exceeds the balance, tell the user what they can afford and
   let them decide.

The count multiplies the cost directly. Re-quote whenever it changes; never reuse the previous
quote.

## Clarify and template selection

- `magic clarify open <id> --json` returns `status`, `reply`, `recommended_templates`,
  `selected_template_ids`, `image_count`, `min_image_count` and `brief`.
- Only once `status` reaches `ready` (`ready_for_plan: true`) can you `generate`.
- To pick templates and fix the count, use **`magic clarify select`**: no model call, one
  round-trip to lock the choice in.

  ```bash
  magic clarify select <id> --template tpl_a --template tpl_b --count 4 --json
  ```

  For `social-card`, `--pages N` sets the page count.
- `recommended_templates[].reason` is the site's own rationale — line it up against the
  preferences the user already expressed and choose. Don't dump the whole template table on
  them every time.
- For layout-slot work like posters, inspect the structure before generating with
  `magic structure get <id> --json`, edit, and write it back with
  `magic structure set <id> --file slots.json`. This is the last editable point **before the
  points are spent**.

## `film` — the one mode that does not use clarify

Film Studio replaces the shared clarify conversation with its own director call, and replaces
"describe the image" with "reference these cards". Everything else in this file still applies —
quoting, downloading, exit codes — but the middle of the flow is different enough to be worth
reading before the first command.

### The chain

| Thing | Id | What it is | Charged |
|---|---|---|---|
| seed | — | One sentence describing the scene | No |
| bench | `bn_…` | A character or place the director proposed; holds candidate bases A/B/C | Candidates: **yes** |
| view card | `fs_…` | One identity, several views (front / profile / establishing …), rendered from a chosen base | Create + 补拍/重 roll: **yes** |
| library card | `aa_…` | An anchored view card, at the **account** level: survives this work, reusable in the next one | No |
| shot | `fsh_…` | Cards in roles + cameras + action + look | No |
| frame | asset id | One rendered image, carrying a receipt of exactly what made it | `generate` / `swap`: **yes** |

```bash
magic film plan <id> --seed "<one sentence>" --apply --json   # seed → benches + a first shot
magic film list <id> --json                                   # benches, cards, shots, running jobs
magic film cand <id> <bn_…> --count 3 --wait --json           # 1–5 candidate bases
magic film sheet create <id> --from <bn_…|aa_…> [--views …] --wait --json
magic film sheet update <id> <fs_…> --add-view profile --wait --json
magic film anchor <id> <fs_…> --name 老陈 --handle laochen [--yes]
magic film shot create <id> --card "<id>:<view>[:<role>][@x,y][=name]"... --cam "A:lens=35"...
magic film cand <id> <bn_…> --ref photo.jpg ...              # user photos on the bench
magic film shot create <id> --ref photo.jpg[:<role>] ...     # user photos on this shot
magic film generate <id> <fsh_…> --count 2 [--dry-run] [--wait] --json
magic film out|swap|save <id> <assetId>
```

### The four things that go wrong

1. **Writing a prompt.** There is no `--prompt` in film, by design: the sentence is composed from
   the ids. `--dry-run` / `--show-prompt` reads the composed prompt back; `--prompt-edit "A=…"`
   overrides one camera's prompt entirely, and `--auto-prompt` restores composition.
2. **Computing a price.** `magic film generate --dry-run --json` prices the exact click and
   returns the prompts with it. Points come from that response's `quote`, never from arithmetic.
3. **Anchoring without asking.** `magic film anchor` without `--yes` prints the candidate views
   with their image URLs and exits **30**. That is a question for the user, not a speed bump.
4. **Treating a view as an enum.** `--views` and `--add-view` take any label. `front`, `profile`,
   `establishing`, `俯拍` are all valid; the defaults (character: front / three_quarter / profile / back,
   scene: establishing / reverse / lateral / detail — four each) are a starting set, not the
   allowed set.

### Cameras, roles and blocking

- `--cam "A:x=0.3,y=0.85,aim=20,height=low,lens=35"` — tag plus optional geometry; `--cam A` alone
  takes the server's defaults. `tag=A,lens=35` is the same thing spelled out. Max 3 cameras.
  On `film generate`, `--cam A` **selects** an existing camera; change one with `film shot update`.
- `--card "<id>:<view>:<role>@x,y=name"` — role is `subject` / `background` / `style` / `element`
  (default `subject`); `@x,y` is the blocking position as 0–1 fractions of the frame.
- `--ref "<file>[:<role>]"` — a local image the user handed over, uploaded and attached to the
  bench (`film cand`, no role) or the shot (`film shot create|update`). Inspiration and
  composition reference, never something to copy. Cards and photos share **six** slots per bench
  and per shot; over the limit the CLI refuses instead of dropping photos. `film shot update`
  replaces the whole refs list, photos included.
- `--no-blocking` on a shot ignores the mini-map positions when composing.

### Cost shape

`cameras × --count` images per click, capped at 12; each image is priced like any other image at
the work's resolution. Film adds no pricing dimension of its own, so
`magic billing quote --op image -n <cameras × count>` still applies for planning ahead — but the
number you quote to the user should come from `film generate --dry-run`.
