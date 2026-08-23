---
name: magic-art
description: >
  Turn an idea into a finished visual with Magic Art (magic-design.art): AI posters,
  Xiaohongshu-style multi-page card decks, portrait shoots, image series — and download
  the results locally. Use when the user wants a poster, cover, social card deck, portrait
  or a set of on-theme images, or mentions Magic Art / magic-design.art. Everything runs
  through the `magic` CLI.
version: 0.2.2
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
| Can't articulate it, wants free rein | `free` |

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

## Who answers (checkpoint policy)

| Cloud checkpoint | Your strategy |
|---|---|
| clarify opening / follow-ups | **Answer yourself first**: distil it from the outer conversation; ask only when the information genuinely isn't there |
| Template recommendation (`selecting_templates`) | Choose for them, using `recommended_templates` reasons plus the taste the user has already expressed; relay the options only if they asked to pick from a few directions |
| Count confirmation (`selecting_count`) | Answer yourself: the number they said, or the mode default if they said none |
| `awaiting_human` during generation | Answer yourself first; purely subjective preference with nothing in context → ask |
| Whether a regional box is accurate | **Hand it over when it isn't**: any doubt about position → `--mark-ui` so the user confirms or fixes the box on the image. Don't spend points on guessed coordinates |
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
- **Always get images down with `magic assets download`** (the CLI resolves CDN or signed
  private-bucket URLs). Don't try to pull bytes out of the API.
- Look up each command's full set of flags with `magic <command> --help` rather than
  assembling arguments from memory.

## Reference

- `references/modes.md` — what each mode is for, count bounds, input requirements, routing calls
- `references/recipes.md` — end-to-end recipes (command sequences plus expected output)
- `references/troubleshooting.md` — exit-code handbook, rate limits, expired tokens,
  `awaiting_human` reply templates, the `job resume` decision tree
