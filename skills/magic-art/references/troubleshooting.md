# Troubleshooting

## Exit codes (a stable contract — branch on it)

| code | meaning | what you do |
|---|---|---|
| 0 | success | continue |
| 10 | bad arguments / usage | read the hint on stderr and fix the flags; `magic <command> --help` for the full list |
| 20 | not logged in / token expired | have the user run `magic login`, relay the URL + pairing code, **never fill it in for them** |
| 21 | insufficient balance / upgrade required | give the user the point price and the top-up URL, **stop**, never buy on their behalf |
| 30 | `awaiting_human` | read the question on stdout; answer it yourself → `magic job answer`; otherwise ask the user |
| 40 | job failed | `magic job list --json` for the error and params → `job resume` or change the parameters |
| 50 | transient error (network / 5xx / 429, already retried by the CLI) | wait a moment and re-run **the same** command |
| 70 | internal CLI error | not a contract code; report the stderr text verbatim to the user |

Under `--json` a failure is also a JSON object: `{"error":{"code":21,"message":"…"}}`.

## Exit code 20: the session expired

Session tokens last about 7 days, and the user can also revoke a device from
"Profile → Connected devices". Both look the same: every command exits 20.

Handling:

```bash
magic auth status --json      # confirm it really is an auth problem
magic login                   # prints an authorization URL + pairing code for the browser
```

Relay the URL and the pairing code **verbatim**. Do not open the login page and type anything
on the user's behalf, and never ask the user for a password or a verification code.

`expires_at` / `expires_in_days` in `auth status --json` is how much of this device's session
is left. It's fine to mention that it is nearly up so the user can `magic login` again — but
**don't** pre-emptively run it for them; authorization has to be confirmed by the user in a
browser.

## Exit code 21: not enough points

`magic billing quote --op image -n <N> --json` returns:

```json
{"points":40,"balancePoints":12,"canAfford":false,"requiredPlanId":null}
```

Tell the user: this run costs 40 points, the account has 12, so it's 28 short, and top-ups are
at `https://magic-design.art/pricing`. Then **stop**. Offering to reduce the count and quote
again is fine — that's something the user can decide immediately.

Each account has one free image; single-image generation spends it automatically (batches
don't).

## Exit code 50 and rate limits

The CLI already retries with exponential backoff (4 attempts max, honouring `Retry-After`).
A 50 means it still failed after those retries.

- **Wait a bit and re-run the same command.** `generate` is idempotent; re-running it does not
  double-charge.
- Don't hammer with a shorter interval, and don't run several `magic` processes against the
  same work in parallel — that only makes the rate limiting worse.
- The `slow_down` signal in device-login polling is handled by the CLI; you don't have to.

## Never hand-roll a polling loop

```
# what NOT to do
while true; do magic job list "$ID" --json; sleep 2; done
```

Use `magic job wait <id> --json`. It polls at the site's cadence (starts at 2.5s, ramps
linearly to 5s, 15-minute overall limit, tolerates 4 consecutive transient failures) and
emits state changes as JSONL events:

```jsonl
{"event":"phase","jobId":"…","phase":"running","label":"Generating"}
{"event":"progress","jobId":"…","donePages":1,"totalPages":4}
{"event":"preview","jobId":"…","text":"…"}
{"event":"retry","jobId":"…","attempt":1,"total":3}
{"event":"awaiting_human","jobId":"…","question":{"text":"…"}}
{"event":"done","jobId":"…","assets":[{"id":"…","url":"…","thumb":"…"}]}
{"event":"failed","jobId":"…","error":"…"}
{"event":"timeout","pending":["…"]}
```

A `timeout` event exits 50, but **the job is still running server-side** — just `job wait`
again; do not re-run `generate`.

## Answering `awaiting_human` yourself

`{"event":"awaiting_human","question":{"text":"…"}}` comes with exit code 30. Common question
types:

| question type | can you answer it? | how |
|---|---|---|
| copy details (what the headline says, whether to include a phone number) | **yes**, if the user mentioned it | fill it back in from the conversation; if they never said, ask |
| language (English / Chinese / mixed) | **yes** | match the language the user is writing in |
| facts (dates, addresses, prices, contact details) | **only** what the user gave you | not in context → ask; **never invent** |
| composition trade-offs (portrait or landscape, how much white space) | usually yes | infer from the use: posters portrait, social square, covers landscape |
| aesthetic preference (which palette, should it be livelier) | **it depends** | follow any leaning the user expressed; pure preference with nothing to go on → ask |
| identity details of a person or product | **no** | always ask; guessing wrong means generating the wrong thing |

After deciding:

```bash
magic job answer "$ID" --job <jobId> -m "<answer>" --json
```

By default it keeps waiting until the job finishes; add `--no-wait` to submit without waiting.

## `job resume` decision tree

Get a clear picture of the state first:

```bash
magic job list "$ID" --json
```

```
Any active job?
├─ yes → magic job wait          (don't resume, and don't re-run generate)
└─ no
   ├─ awaiting_human? → magic job answer (keeps waiting once answered)
   └─ Any failed job?
      ├─ the error is transient (timeout / upstream unavailable / network) → magic job resume
      ├─ the error is a content rejection (sensitive content, non-compliant reference image)
      │   → do not resume. Change the prompt / reference image, check with the user, generate again
      ├─ the error is a parameter problem (template doesn't exist, illegal layout)
      │   → fix the parameters and generate again (this is a new purchase)
      └─ you can't tell → relay the error text to the user and let them decide
```

`job resume` replays the failed job with the parameters it recorded, and **charges again** —
the failed run was already refunded, so the replay is an honest new charge, not a double
charge.

## Images won't download

- `assets download` connects straight to the CDN or a signed private-bucket URL. A 403 is
  usually an expired signature: re-run `assets download` (it fetches a fresh URL).
- "no direct URL available" = this environment has neither a public CDN nor private-bucket
  signing configured. That is an environment problem; **do not try to pull the bytes through
  the API instead.** Report it to the user.

## `--mark` rejected (exit code 10)

Annotation coordinates are **fractions of 0–1**, not pixels. `--mark "1200,800,300,200=change
this"` is rejected outright rather than clamped into a corner — deliberately: an edit that
quietly lands in the wrong place costs far more than an error message.

- Given pixel coordinates, divide them yourself: `x/imageWidth`, `y/imageHeight`,
  `w/imageWidth`, `h/imageHeight`.
- Every field is required: `x,y,w,h=note` or `x,y=note`, and there must be note text after the
  `=`. An empty field (`0.2,,0.3,0.1=…`) is an error, not a 0.
- The split happens at the **first** `=` only, so notes can contain equals signs.
- `note` in `--marks-file` cannot be an empty string — an unexplained box is paying the model
  to guess, so it errors out.
- More than 10 marks, or a note over 300 characters, is rejected — the same limits as the web
  app, not something the CLI adds.
- If it says the coordinates look like pixels, first check that you actually looked at the
  image: `magic assets download` it, then annotate.

## `--mark-ui` exit code 10: "annotation cancelled"

The user hit cancel, closed the page, or didn't submit within 15 minutes. **Nothing was
generated and nothing was charged.** This isn't an error, it's the user changing their mind —
go back and ask what they want changed instead of falling back to blind coordinates.

Other `--mark-ui` situations:

- **The browser didn't open**: the CLI already printed the URL on stderr — relay it. On a
  server or in a container, just pass `--no-open`.
- **The user can't reach 127.0.0.1 on this machine** (remote session, container): `--mark-ui`
  is unusable; fall back to `--mark` coordinates, and you must `magic assets download` and
  look at the image first.
- **"no direct image URL in this environment"**: no CDN / signed URL is configured here, so
  the page can't load the image. Use `--mark` coordinates.
- **The page rejects a submission**: it enforces the same limits as `--mark` (≤10 marks, notes
  ≤300 chars). The page stays open; the user removes the extra boxes and submits again.
- The annotated image the page composes is only a visual aid for the model and **is not part
  of the idempotency key** — only coordinates and note text are.

## Idempotency, and "did I just get charged twice?"

- `generate` / `edit` / `job answer` / `job resume` internally keep an `operation_id` keyed by
  "work + intent fingerprint". **That id is only reused when the previous submission didn't
  land**: on a timeout, a dropped connection or exit code 50 it is still open, so re-running
  the identical command reuses it, the server deduplicates, and nothing is charged twice.
- Once the server has accepted the submission, running the same command again is a new
  purchase — the CLI won't block it and shouldn't, because "give me another version" is a
  legitimate request. For recovery use `job list` / `job wait`.
- Changing any material parameter (count, template, layout, the edit instruction text, a
  `--mark` coordinate or note) is likewise a new purchase — because those really are different
  images.
- You never need to, and never should, generate or pass an `operation_id` yourself.

## Miscellaneous

- `magic auth status --json` shows the current site, account and token expiry.
- To point at a non-production site: `--base-url https://…` or the `MAGIC_ART_BASE_URL`
  environment variable. Credentials are stored per site, so switching sites means being
  logged out.
- Upgrading: `npm i -g @magic-art/cli@latest && magic skill install` (the skill files ship with
  the CLI so the two stay on the same version). The canonical copy lives at
  `~/.agents/skills/magic-art` and is symlinked into every agent installed on the machine, so
  that one command upgrades all of them at once.
