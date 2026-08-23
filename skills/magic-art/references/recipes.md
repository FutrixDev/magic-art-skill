# Recipes (end-to-end command sequences)

Every recipe here can be copied as-is. `$ID` is the work id `project create` prints on stdout.
With `--json` on every command, stdout is parseable; human-facing notes always go to stderr.

---

## 1. A single poster (shortest path)

User: "Make me a poster for a coffee shop opening — off-white palette, doors open Saturday at 10am."

```bash
magic billing quote --op image -n 1 --json          # exit code 21 → stop and ask
ID=$(magic project create --mode poster \
      --text "Coffee shop opening poster, off-white palette, opens Saturday 10am, reads 'Now Open'")
magic generate "$ID" --count 1 --wait --json         # JSONL event stream, prints the results at the end
magic assets download "$ID" --out ./out
```

Expected: the last line of `generate --wait --json` is
`{"event":"done","jobId":…,"assets":[{"id":…,"url":…}]}` with exit code 0; `assets download`
prints local file paths.

Poster mode needs no clarify and can generate directly. When the user wants to see style
options first:

```bash
magic styles list --kind poster --json
magic generate "$ID" --template <style id> --count 3 --wait --json
```

---

## 2. A Xiaohongshu multi-page note (goes through clarify)

User: "Write me a Xiaohongshu post about the bakery downstairs, 6 pages."

```bash
magic billing quote --op image -n 6 --json
ID=$(magic project create --mode social-card --text "Recommend the bakery downstairs, 6-page note, genuine first-visit voice")
magic clarify open "$ID" --json                      # read the question in `reply`
magic clarify send "$ID" -m "Audience is office workers nearby; focus on the croissants and sourdough; I have real photos" --json
magic clarify select "$ID" --template <pick one from the recommendations> --pages 6 --json
magic clarify status "$ID" --json                    # continue only once ready_for_plan: true
magic generate "$ID" --wait --json
magic assets download "$ID" --out ./out
```

Key points:
- `recommended_templates[].reason` from `clarify open` is the basis for picking a template —
  choose for the user whenever you can.
- `clarify select` runs no model; it is the fast path for fixing which templates and how many
  pages.
- Pages are 3–10; out-of-range values get clamped.

---

## 3. An image series from a reference photo

User: "Here's our product shot — give me a set of 4 images in different settings."

```bash
magic billing quote --op image -n 4 --json
ID=$(magic project create --mode image-series \
      --text "One insulated bottle in 4 use settings: desk, camping, gym, commute" \
      --ref ./cup.jpg --ref-prompt "This is a real product shot; the product's appearance must be preserved, not redesigned")
magic clarify open "$ID" --json
magic clarify send "$ID" -m "Settings as above, clean natural light, no heavy filters" --json
magic generate "$ID" --count 4 --wait --json
magic assets download "$ID" --out ./out
```

Key point: `--ref` can be repeated for several images; `--ref-prompt` tells the model **what
role** those images play (a style reference? a subject that must be preserved?). Leave it out
and the model treats them as loose inspiration.

---

## 4. Being asked a question mid-generation (`awaiting_human`)

```bash
magic generate "$ID" --count 4 --wait --json
# → last line {"event":"awaiting_human","jobId":"job_x","question":{"text":"Should the headline be in English or Chinese?"}}
# → exit code 30
```

Answer it yourself when you can, and it keeps waiting automatically:

```bash
magic job answer "$ID" --job job_x -m "English, with a little Chinese accent text in the body" --json
```

When you can't (a pure preference with nothing in context), relay the question verbatim and
`job answer` once the user replies.

Note that `awaiting_human` **aborts the wait for the whole batch**, but the remaining slots
keep running server-side. After answering, `magic job wait "$ID" --json` follows the rest to
completion.

---

## 5. Iterating on an image

User: "The second one is good, but change the headline to 'Flash sale - 20% off' and warm it up."

```bash
magic assets list "$ID" --json                       # get the asset id
magic edit "$ID" --parent <assetId> \
      --prompt "Change the headline to 'Flash sale - 20% off', warm the overall colour temperature, leave everything else as is" --json
magic assets download "$ID" --asset <new assetId> --out ./out
```

`edit` is **synchronous**: by the time the command returns, the new image exists (it waits up
to 5 minutes) and there is no job to follow. It costs points too, so quote before editing.

### 5b. Changing just one area (regional annotation)

User: "Overall it's fine — just that headline top-right and the empty space bottom-left."

A whole-image prompt invites the model to repaint other things on its way past. Point at the
positions instead:

```bash
magic assets download "$ID" --asset <assetId> --out ./out   # required: look at the image first
# Confirm the headline sits top-right at roughly 30% width / 12% height, and the empty space is at y≈0.75
magic edit "$ID" --parent <assetId> \
      --mark "0.62,0.08,0.3,0.12=replace this headline with 'Flash sale - 20% off'" \
      --mark "0.1,0.75=too empty here, add a small coffee cup icon" \
      --json
magic assets download "$ID" --asset <new assetId> --out ./out
```

With many marks, put them in a file and pass `--marks-file marks.json`:

```json
[
  { "x": 0.62, "y": 0.08, "w": 0.3, "h": 0.12, "note": "replace the headline with 'Flash sale - 20% off'" },
  { "x": 0.1, "y": 0.75, "note": "add a small coffee cup icon" }
]
```

Key points:
- Coordinates are fractions of 0–1 with the origin top-left; `x,y,w,h` boxes an area, `x,y`
  alone points at a spot. Pixel values raise exit code 10 rather than being silently
  reinterpreted as fractions and applied to the wrong place.
- **Coordinates must come from an image you have looked at.** With only an asset id they are
  guesses, and a wrong guess is paying to change the wrong thing.
- At most 10 marks, each note ≤300 characters. With `--mark` present, `--prompt` is optional;
  you can also give both (`--prompt` is the overall requirement, `--mark` the per-area ones).
- The idempotency key includes the coordinates: re-running the exact command reuses the
  previous `operation_id`, while changing a single number is a new charge.

---

### 5c. Letting the user box it themselves (`--mark-ui`)

User: "Here, and also here, aren't quite right." — the position is hard to pin down, or you
aren't confident in the boxes you would draw.

Don't guess coordinates and pay to find out. Put the image in front of them:

```bash
magic edit "$ID" --parent <assetId> \
      --mark "0.62,0.08,0.3,0.12=this headline probably needs to change" \
      --mark-ui --json
```

What happens:

1. The CLI serves a temporary page on 127.0.0.1 and opens a browser; the image loads straight
   from the CDN or a signed URL.
2. The `--mark` boxes you gave are already drawn on it (they're optional — without them the
   user boxes from scratch). The user drags boxes and writes notes.
3. The user clicks "submit and generate" → the CLI takes the final boxes **and the
   browser-composed ①②③ annotated image** and sends both to the model. Only now does
   rendering start and points get spent.
4. The user clicks cancel / closes the page / does nothing for 15 minutes → exit code 10,
   **nothing generated, nothing charged**.

```
$ magic edit prj_x --parent ast_9 --mark-ui --json
Waiting for you to annotate and submit in the page… (closing the page = cancel)
{"asset":{"id":"ast_10","parent_id":"ast_9","url":"https://…"}}
```

After a cancel, don't re-run with blind coordinates — go back and ask what they want changed.

Servers and headless environments: add `--no-open` and the CLI only prints the URL. That works
only if the user can reach 127.0.0.1 on that machine; otherwise fall back to the `--mark`
coordinates of 5b.

---

## 6. Resuming after an interruption (agent crashed / user closed the terminal)

**Do not run `generate` again** — that is buying it a second time.

```bash
magic job list "$ID" --json
```

- An `active` job → `magic job wait "$ID" --json`
- An `awaiting_human` job → `magic job answer` first
- Only `failed` jobs → `magic job resume "$ID" --json`
- None of the above and `assets` already has results → just `magic assets download`

`generate`'s idempotency only covers "the last submission never landed": when a request got no
response (timeout, dropped connection, exit code 50) the `operation_id` is still open, so
re-running the identical command reuses it and does not double-charge. **Once the server has
accepted the submission, running `generate` again is a new purchase** — the same as clicking
generate a second time on the website. So recovery goes through `job list` / `job wait` above.
Changed parameters (count, template, layout) are more obviously a new purchase, because they
are genuinely different images.
