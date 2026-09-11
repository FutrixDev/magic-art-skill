---
name: magic-art
description: >
  Turn an idea into a finished visual with Magic Art (magic-design.art): AI posters,
  Xiaohongshu-style multi-page card decks, portrait shoots, image series — and download
  the results locally. Use when the user wants a poster, cover, social card deck, portrait
  or a set of on-theme images, or mentions Magic Art / magic-design.art. Everything runs
  through the `magic` CLI.
version: 0.6.0
metadata:
  openclaw:
    emoji: 🎨
    homepage: https://magic-design.art/connect
    requires:
      bins:
        - magic
    install:
      - node: "@magic-art/cli"
---

# Creating with Magic Art

Magic Art is itself an agent with human checkpoints: it asks clarifying questions,
recommends templates, confirms how many images to make, and can stop mid-generation to ask
for more. Your job is to **decide who answers at each checkpoint** — answer from the
conversation yourself where you can, and go back to the user only when the judgment is
genuinely theirs.

## Preflight

1. `magic --version`
   - Command not found, or lower than this skill's `version` (see frontmatter):
     `npm i -g @magic-art/cli@latest && magic skill install`. The skill files ship inside
     the CLI package, so a copy installed from a hub can be older than the binary that
     implements it; re-installing writes the matching copy to `~/.agents/skills/magic-art`
     and relinks every agent on the machine.
2. `magic auth status --json`
   - Exit code 20 → tell the user to run `magic login`, and relay the URL and pairing code
     the CLI prints **verbatim**. The user authorizes in their own browser.
     **Never enter a code, password or any credential on their behalf.**
3. Quote before spending anything:
   `magic billing quote --op image -n <count> --json`
   - Exit code 21 (`canAfford=false`, or `requiredPlanId` non-empty) → give the user the
     point price and the top-up link, then stop. **Never buy or upgrade for them.**

## Picking a mode (details in references/modes.md)

| What the user is describing | mode |
|---|---|
| A poster, promo image, cover, single hero visual | `poster` |
| A Xiaohongshu note, multi-page deck, recommendation post | `social-card` |
| One theme, several images that belong together | `image-series` |
| A portrait shoot, "turn my photo into…" (needs the user's own photo) | `portrait` / `portrait-series` |
| A scene, a story, recurring characters across several shots | `film` (see "Film Studio" below) |
| A logo, brand mark, symbol, app icon | `logo` (see "Logo Studio" below) |
| Can't articulate it, wants free rein | `free` |

**None of the above**: the user has already written the exact prompt they want rendered and
is asking for it to go to the model *as written* — skip modes entirely and use
`magic image` ("Raw prompt, straight through" below).

Unsure? Run `magic styles list --kind <mode> --json` to see what styles exist in that mode,
or `magic search --json` to pull every available facet in one call.

## The standard flow

```bash
magic project create --mode poster --text "<the user's idea, verbatim>" [--ref image]...  # → prints a work id
magic clarify open <id> --json                                              # modes that need clarifying
magic clarify send <id> -m "<your answer>" --json                           # loop until ready_for_plan
magic generate <id> --count 4 --wait --json                                 # submit and follow to the end
magic assets download <id> --out ./out                                      # results land locally
```

1. **Create**: `magic project create --mode <mode> --text "<the user's idea, verbatim>"`.
   If the user supplied reference images, pass `--ref <file>` (repeatable) and use
   `--ref-prompt` to say what those images are for (a style reference, or a product/person
   that must be preserved). Resolution can only be set here (`--resolution 1k|2k|4k`) and
   cannot be changed afterwards.
2. **Clarify** (`social-card`, `image-series` and other modes that require it):
   `magic clarify open <id> --json` returns the opening question; then loop
   `clarify send` / `clarify select` until it returns `ready_for_plan: true`.
   - To pick templates and set the count, use
     `magic clarify select <id> --template <id> --template <id> --count 4`
     (no model call — fast and deterministic).
   - For when to answer yourself, see "Who answers" below.
3. **Generate**: `magic generate <id> --count <N> --wait --json`.
   - Without `--wait` it only submits and prints a jobId; follow it later with
     `magic job wait <id> --json`.
   - Exit code 30 (`awaiting_human`): read the question on stdout. If you can answer it,
     `magic job answer <id> --job <jobId> -m "<answer>"` (keeps waiting by default);
     otherwise relay it to the user.
   - Exit code 40: `magic job list <id> --json` for the error and the replayable params,
     then `magic job resume <id>`, or adjust parameters and generate again.
4. **Deliver**: `magic assets download <id> --out <dir>`, then hand the user the local paths.
5. **Edit**: `magic edit <id> --parent <assetId> --prompt "<what to change>"` (returns the new
   image synchronously), then download. When the change is hard to describe for the whole
   image and only one area is wrong, switch to **regional annotation** (next section).

## Raw prompt, straight through (`magic image`)

Every other path in this skill *composes* a prompt: the mode's template, the chosen style,
the film cards, the director pass. That is the point of them — the user brings an idea and
the service turns it into a prompt that renders well.

`magic image` is the opposite, and the only command that does this:

```bash
magic image "<the exact prompt>" -n 2 --ratio 3:2 --resolution 2k --out ./out --json
```

The prompt string is sent to the image gateway **byte for byte**. Nothing is prepended,
appended, rewritten, translated, summarised or safety-wrapped, and no style template,
style bible, director or aspect-ratio sentence is mixed in. Aspect ratio, resolution,
quality and transparent background are structured parameters — they never become words
inside the prompt.

Use it when:

- the user hands you a prompt and says "render this, don't touch it";
- they are iterating on prompt wording themselves and need the model's honest response to
  exactly what they wrote;
- a prompt was authored elsewhere (another tool, a paper, their own library) and must be
  reproduced unchanged.

Do **not** use it as a shortcut for "the user described something they want": composing a
good prompt out of a rough idea is what `project create` + `generate` is for, and going raw
there just makes a worse picture.

Notes:

- `--prompt-file <path>` reads the prompt from a file, verbatim including newlines — use it
  for anything long enough that shell quoting could mangle it. It is mutually exclusive with
  the positional prompt.
- `--json` echoes back the `prompt` the server actually sent to the gateway, so the
  "unchanged" claim is checkable, not just asserted.
- `-n 1..4`. Each image is billed and fails on its own: one failure does not lose the others,
  and the failed one is refunded automatically (it shows up under `failures`).
- The results land in an ordinary work, so everything downstream is unchanged:
  `magic assets list|download <work id>`, the work page, the account library. `--out <dir>`
  downloads them in the same call.
- `magic billing quote --op image -n <N> --json` first, as with any other spend.

## Regional (annotated) edits

When the user says "just this bit is off", don't write "change the icon in the top-left" into
`--prompt` and let the model hunt for it — point at the place directly:

```bash
magic assets download "$ID" --asset <assetId> --out ./out   # get the image locally and look at it first
magic edit "$ID" --parent <assetId> \
      --mark "0.62,0.08,0.3,0.12=replace this headline with 'Flash sale - 20% off'" \
      --mark "0.1,0.75=this corner is empty, add a small icon" --json
```

- Coordinates are **fractions of 0–1** with the origin at the top-left: `x,y,w,h=note` boxes
  an area, `x,y=note` points at a spot. Pixels (`1200,800,...`) are rejected outright, never
  silently reinterpreted as fractions and applied to the wrong place.
- At most 10 marks per call, each note ≤300 characters. Multiple marks reach the model
  numbered ①②③ in the order you give them.
- With `--mark` present, `--prompt` is optional; giving both means "overall requirement plus
  per-area requirements".
- For many marks, put them in a file: `--marks-file marks.json` (a JSON array of objects with
  `x,y,w,h,note`).

**Hard rule: never give coordinates for an image you haven't looked at.** With only an asset
id in hand, every coordinate is a guess — `magic assets download` it first, read the image,
see where things actually are, then mark. Even when the user describes the location ("the QR
code in the bottom-right"), look at the image and confirm the QR code is really there.

Changing one coordinate or one word is a new render and a new charge; only re-running the
exact same command reuses the previous `operation_id`.

### Let the user draw the boxes (`--mark-ui`)

The user cannot see whether coordinates on a command line are right — by the time they find
out the box was wrong, the points are spent. **If there is any doubt about a position, hand
the boxing to the user** by adding `--mark-ui`:

```bash
magic edit "$ID" --parent <assetId> \
      --mark "0.62,0.08,0.3,0.12=replace this headline with 'Flash sale - 20% off'" \
      --mark-ui --json
```

The CLI serves a page bound to 127.0.0.1 only and opens a browser, showing the user the
original image with the boxes you gave **already drawn** on it. They can drag, add and delete
boxes and write a note on each; the CLI continues only after they click "submit and
generate". On submit, the browser-composed **①②③-annotated preview** goes to the model along
with the boxes — exactly like annotated editing on the website.

- **`--mark` is optional**: when you are unsure of everything, pass `--mark-ui` alone and let
  the user box from scratch.
- **"Cancel" or closing the page = exit code 10, nothing generated, nothing charged.** Do not
  fall back to blind coordinates on your own initiative — ask the user what they want changed.
- On a server with no browser: add `--no-open` and the CLI prints the URL for the user to open
  themselves (they must be able to reach 127.0.0.1 on that machine; if they can't, fall back
  to `--mark` with coordinates).
- No submission within 15 minutes times out, handled the same as a cancel.

## Film Studio (`--mode film`)

For work where **the same character or place has to look the same in more than one image** —
a scene, a short story, a comic, a shot list. The other modes render each image on its own;
film gives you cards that other shots reference, so consistency is structural instead of
something you re-describe in every prompt.

```bash
magic project create --mode film --text "<the user's idea>"          # → a work id
magic film plan "$ID" --seed "深夜天台，老陈和年轻警察对峙" --apply --json  # seed → benches (free)
magic film cand "$ID" bn_chen --count 3 --wait --json                # candidate bases (charged)
magic film sheet create "$ID" --from bn_chen --views front,profile --wait --json  # → fs_… (charged)
magic film anchor "$ID" fs_chen --name 老陈 --handle laochen          # exit 30 → ask the user
magic film anchor "$ID" fs_chen --name 老陈 --handle laochen --yes    # → aa_… in the library
magic film shot create "$ID" --title 天台对峙 \
      --card "aa_laochen:front:subject@0.3,0.6" --cam "A:lens=35" --action "老陈抬手" --json
magic film generate "$ID" fsh_1 --count 2 --dry-run --json            # price + composed prompt
magic film generate "$ID" fsh_1 --count 2 --wait --json               # charged
magic film out "$ID" <assetId> --show-prompt --json                   # what this frame was made of
magic film save "$ID" <assetId>                                       # keep it in the strip
```

`magic film list <id> --json` returns the whole film — benches, view cards, shots and any
running jobs — and is how you re-orient after an interruption instead of re-planning.

- **Pass ids and structured fields, never prompt prose.** The sentence the image model reads is
  composed server-side from the card ids, the camera and the action. To see it, read it back
  with `--dry-run` / `--show-prompt`; to change it, change the structure. `--prompt-edit "A=…"`
  exists for the rare full override, and `--auto-prompt` puts composition back.
- **A view is a free label**, not an enum: `--views front,profile,俯拍=俯拍` is fine. Omitting
  `--views` uses the type's default set.
- **Only five things cost points**: `film cand`, `film sheet create`, `film sheet update` (when it
  adds or re-shoots a view), `film generate`, `film swap`. Everything else — planning, listing,
  editing shots, reordering, saving, the whole library — is free.
- **Never compute a film price yourself.** `magic film generate --dry-run --json` returns the
  server's `quote` and the composed prompt in one call; quote from that.
- One click renders `cameras × --count` images, capped at 12.

### The user handed you photos

When the user gives you image files — a location photo, a costume shot, a mood board page — pass
them with `--ref`. They are **inspiration and composition reference, not something to copy**: the
render is a new image informed by them.

```bash
magic film cand "$ID" bn_roof --ref ~/photos/roof-dusk.jpg --count 3 --wait --json  # 挂在台面上
magic film shot create "$ID" --title 天台对峙 \
      --card "aa_laochen:front:subject" \
      --ref ~/photos/roof-dusk.jpg:background --json                                # 挂在这一镜上
```

- `--ref <file>[:<role>]` — the role is the same set as `--card` (`subject` / `background` /
  `style` / `element`), default `subject`. `--ref` on `film cand` takes no role: a bench reference
  is a look reference for the base image.
- **`--card` vs `--ref`**: a card is *identity consistency* — the same character, reused across
  shots and rebound when it is re-anchored. A `--ref` photo is a *one-off reference* attached to
  this bench or this shot, with no identity behind it. Reach for `--card` when the answer to "is
  this the same person/place?" must be yes.
- **Six reference slots**, cards and photos together, per bench and per shot. Over the limit the
  CLI refuses and says how many fit — it never quietly uses some of the user's photos.
- Photos are uploaded before the render and **stay attached even if the render fails**, so a retry
  does not re-upload them. `magic film list` shows them as `refs=2(image:<assetId>)`;
  `magic film out` prints `image:<assetId>` for a photo the frame actually used.
- `magic film shot update` replaces the refs list whole. Passing only `--card` drops the shot's
  photos; pass `--ref` again alongside it to keep them (or omit both to leave refs untouched).

### Anchoring is the user's decision

`magic film anchor` promotes a view card into the **account-level library**: it outlives this
work, and every shot referencing that identity is rebound onto it. Which take *is* the
character is taste, so without `--yes` the CLI prints the candidate views and their image URLs
and exits **30**. Show the user the images, get an answer, then re-run with `--yes`.

`magic library list|show|add|set|rm` is that library's own surface — the cards are pointers, so
deleting a work does not delete its cards, and `magic library rm --hard` on a card another film
still references is refused rather than quietly breaking that film.

## Logo Studio (`magic logo`)

For a **brand mark**: a logo, a symbol, an app icon. One brief goes in, a board of distinct
directions comes back as transparent PNGs, and the user picks the one that is theirs.

```bash
magic logo generate --name "Nimbus" --about "weather app for sailors" \
      --palette "#0f4c81,#e8b04b" --directions 3 --count 2 --dry-run --json   # price + the plan, free
magic logo generate --name "Nimbus" --about "weather app for sailors" \
      --directions 3 --count 2 --out ./logos --yes --json                     # charged; waits; downloads
magic logo generate --name "Nimbus" --about "weather app for sailors" \
      --mark-types mascot,emblem --count 2 --out ./logos --yes --json         # exactly these two directions
magic logo generate --name "Nimbus" --about "weather app for sailors" \
      --mark-types mascot,auto --count 2 --out ./logos --yes --json           # mascot + one the planner surprises you with
ls -la ./logos            # d1-c1.png d1-c2.png d2-c1.png … — show the user the FILES, not the ids
magic logo list <id> --out ./logos --json                                     # the board again, free
magic logo pack <id> --cand d1-c2 --out ./logos                               # anchor + delivery zip, free
```

`magic logo generate` creates the work itself when you don't pass `--project <id>` — **including
with `--dry-run`**, because the planner and the price both live on the work. That dry-run work is
empty and costs nothing, but it is real: reuse it with `--project <id>` instead of letting the next
call mint a second one. The id is the first thing on stdout — `project_id\t<id>` (with `--json`, a
`{"event":"project_created",…}` line) printed **before** anything is charged, so a run that dies
mid-flight is recoverable with `--project <id>`. `magic logo list <id>` is how you re-orient after
an interruption instead of generating a second board.

`--mark-types` names **which** kinds of mark to explore, out of `abstract` / `lettermark` /
`mascot` / `emblem` / `pictorial` (comma-separated, at most five). One type = one direction, in the
order given, so it sets the direction count and `--directions` is ignored (with a notice) when both
are passed. A sixth value, `auto`, is "you pick": that ONE slot's type is chosen server-side by the
planner, which is asked for a surprising angle the brief does not spell out — and it may pick a type
already named in the same list. At most one `auto`, but it combines with all five concrete types
(`abstract,lettermark,mascot,emblem,pictorial,auto` = six directions). `--mark-types mascot,auto` is
therefore two directions — a mascot, plus whatever the planner wants to surprise you with.
Leave it off and the planner picks the types from the brief, which is the default. It is
the same choice the 简报 card offers on the canvas, and it rides on the brief — so a board planned
with it keeps the selection.

The table both commands print is `id / direction / mark type / status / transparent ratio / local
file`. `status` is `ready` (downloaded), `opaque` (the model returned a background instead of
transparency — **already refunded**, nothing to pay or fix, offer a re-roll) or `failed`.

### The user handed you images

`magic logo generate --ref <file>` attaches up to **4** images to the brief — a sketch, a
competitor's feel, a photo of the thing the brand is about. They are **reference for direction and
mood, not artwork to reproduce**, and the output is still a wordless, transparent-background mark.

```bash
magic logo generate --name "Nimbus" --about "weather app for sailors" \
      --ref ~/photos/sketch.png --ref ~/photos/mood.jpg \
      --directions 3 --count 2 --out ./logos --yes --json
```

Resuming with `--project <id>` appends to what the brief already holds; asking for more than four
in total is refused rather than truncated. Handing over the same file twice uploads it twice and
costs a second slot — pass each photo once. `magic logo list <id>`
prints them as a `refs\t<assetId>,…` line.

- **The brand name never enters the image prompt.** Models cannot spell reliably, so the mark is
  drawn wordless and the wordmark is built from real font outlines later. Don't try to route the
  name in through `--about`; describe what the brand *is* instead.
- **Which mark is the brand is the user's decision, made by looking.** After `generate`, give the
  user the local file paths so they can open the PNGs. Never pick for them.
- **How big the batch is**: `--directions` defaults to **3** (1–5) and `--count` to **2** (1–4),
  so a bare `generate` renders 3 × 2 = 6 marks. The server clamps both, and the canvas's 简报 card
  stores the same two numbers — passing neither on a work that already has a brief reuses what the
  user chose there.
- **Only `generate` costs points** — directions × count images, priced per 2K image; `list` and
  `pack` are free. Quote from `--dry-run --json`'s `quote`, never by arithmetic. Over 300 points
  without `--yes`, the CLI prints the quote and exits **30**: relay it, then re-run with
  `--project <id> --yes` so it uses the work it already created. Exit **30** with
  `"reason":"no_quote"` means the server priced nothing and the CLI refused to dispatch blind —
  retry. Not enough points is exit **21** (only the user can fix it); nothing was dispatched in
  either case.
- **Exit 50 after a paid `generate` means "still running", not "broken"** — the points were spent
  and the jobs are alive server-side, so the move is `magic logo list <id> --out ./logos`, never a
  second `generate`. A candidate that actually failed shows as exit **40**, once nothing is in
  flight.
- **`pack` always succeeds, and always tries for the SVG.** The zip holds transparent PNGs in six
  sizes, two mono versions, the wordmark lockups, a README — and `vector/mark.svg`, which the export
  job traces on the spot when the mark has none. Tracing needs the deployed logo-lab service, so
  where it is missing the archive ships without that one file and says so: `--json` carries
  `"vector": false` and the plain output prints a line. Never tell the user the SVG is in there
  without checking that field.

## Who answers (checkpoint policy)

| Cloud checkpoint | Your strategy |
|---|---|
| clarify opening / follow-ups | **Answer yourself first**: distil it from the outer conversation; ask only when the information genuinely isn't there |
| Template recommendation (`selecting_templates`) | Choose for them, using `recommended_templates` reasons plus the taste the user has already expressed; relay the options only if they asked to pick from a few directions |
| Count confirmation (`selecting_count`) | Answer yourself: the number they said, or the mode default if they said none |
| `awaiting_human` during generation | Answer yourself first; purely subjective preference with nothing in context → ask |
| Whether a regional box is accurate | **Hand it over when it isn't**: any doubt about position → `--mark-ui` so the user confirms or fixes the box on the image. Don't spend points on guessed coordinates |
| `magic film anchor` (exit 30) | **Always ask**: the card outlives this work and rebinds every shot that uses it. Relay the candidate views and their image URLs, then re-run with `--yes` |
| Which bench candidate / which take to build on | **Ask when it is taste, decide when it is craft**: "which of these three faces is 老陈" is the user's; "this one failed, re-roll it" is yours |
| Insufficient balance / upgrade needed | **Always ask**: quote plus top-up URL, never purchase |
| Publishing / permanent deletion | **Always ask** (outward-facing, irreversible) |
| Login | Relay the URL and pairing code only, **never enter any credential** |

## Hard rules

- **Use the `magic` CLI only**; never curl the site API directly. Auth headers, idempotent
  `operation_id`s, poll cadence and exit-code semantics all live in the CLI. Going around it
  breaks, and it will trip rate limits.
- **Quote before spending**: generation burns magic points, so `billing quote` comes first.
  Exit code 21 always means stop and ask.
- **Publishing and permanent deletion** (`magic project delete --yes`): ask the user first.
- **Never hand-roll a sleep/poll loop**: wait with `magic job wait`, which polls at the
  cadence the site expects.
- **Resuming after an interruption**: `magic job list <id> --json` first — an active job means
  `job wait`, a failed one means `job resume`. **Do not just run `generate` again** (that is
  buying it a second time).
- **Look at the image before giving coordinates**: `--mark` coordinates may only come from an
  image you actually viewed, never inferred from an asset id.
- **In film, pass ids — never a hand-written prompt.** Prompts are composed server-side from
  the cards, camera and action; read one back with `--dry-run` / `--show-prompt` instead of
  writing one, and never total up points yourself when the server returns a `quote`.
- **Never `magic film anchor --yes` on your own judgment**: exit 30 is a question for the user.
- **Always get images down with `magic assets download`** (the CLI resolves CDN or signed
  private-bucket URLs). Don't try to pull bytes out of the API.
- **`magic image` is for a prompt the user authored, not for an idea you would compose**:
  it sends the string unchanged, so anything that needs shaping belongs in
  `project create` + `generate`.
- Look up each command's full set of flags with `magic <command> --help` rather than
  assembling arguments from memory.

## Reference

- `references/modes.md` — what each mode is for, count bounds, input requirements, routing calls,
  and the `film` chain (bench → view card → anchor → shot) with its charged commands
- `references/recipes.md` — end-to-end recipes (command sequences plus expected output),
  including a full film and the second film that reuses its cards
- `references/troubleshooting.md` — exit-code handbook, rate limits, expired tokens,
  `awaiting_human` reply templates, the `job resume` decision tree
