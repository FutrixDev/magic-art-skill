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
