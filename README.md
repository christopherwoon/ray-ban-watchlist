# Ray-Ban Display Stock Watchlist

Two separate webapps sharing one backend:

- **`/` (this folder)** — phone/browser edit page. Add/remove tickers.
- **`/glasses/`** — the actual Meta Ray-Ban Display webapp. D-pad navigable,
  built against the real
  [meta-wearables-webapp](https://github.com/facebookincubator/meta-wearables-webapp)
  toolkit (cloned and inspected directly — see "Toolkit compliance" below).
  Full parity with the phone page: add, remove, mute, view catalyst headlines
  — all D-pad/focus navigable, using a standard `<input>` for the Add Ticker
  screen (see "Text entry on glasses" below) rather than a read-only view.

Both talk to the same `/api/watchlist`, `/api/quote`, `/api/news` backend, so
edits made on the phone page show up in the glasses app immediately.

There is no `window.storage` bridge or similar SDK-provided sync mechanism —
that was an assumption from early mockups made before the real toolkit was
available. The real toolkit gives each surface only plain `localStorage`,
which can't sync across two separate browser contexts (a phone browser and
the glasses' own WebView) on its own. `/api/watchlist`, backed by Upstash
Redis, is what actually makes the two surfaces share state.

## Run it locally

```
python devserver.py 8743
```

Then open `http://localhost:8743` (edit page) and `http://localhost:8743/glasses/`
(the D-pad app — use arrow keys + Enter/Escape to navigate, same as the
toolkit's own dev-testing instructions).

Do not use plain `python -m http.server` — it can't serve `/api/*`, so both
pages will fall back to defaults/mock data.

To test the real watchlist sync locally (optional): copy `.env.local.example`
to `.env.local` and fill in `KV_REST_API_URL`/`KV_REST_API_TOKEN` from your
Vercel project's Environment Variables page.

## Deployment note (2026-09-16)

`ray-ban-watchlist.vercel.app` (the Vercel project) is Git-connected to
**`christopherwoon-dev/ray-ban-watchlist`** (a private repo under a second
GitHub account), not `christopherwoon/ray-ban-watchlist`. Both repos exist
and both need pushes if you want history to match — but only pushes to the
`christopherwoon-dev` one actually trigger a deploy. This cost real time to
track down (multiple fixes were pushed to the wrong repo and never went
live), so worth knowing before pushing again.

`/api/watchlist` is now `api/watchlist.js` (Node), not `api/watchlist.py`
(Python) — the Python version consistently threw
`<urlopen error [Errno 16] Device or resource busy>` (and the same error
persisted through several rewrites: `http.client` instead of `urllib`, then
forcing IPv4). Root cause: the configured `KV_REST_API_URL` host doesn't
resolve in public DNS at all (checked against Google's and Cloudflare's
resolvers), even though `upstash.io` itself does — that's the signature of
a private, Vercel-internal storage hostname only reachable from inside
Vercel's own network fabric, and that path is far better supported from
Vercel's Node.js runtime than from Python. The Node rewrite needs no
dependencies (just built-in `fetch`). `api/_shared.py`'s `get_watchlist`/
`set_watchlist`/`_kv_request` are dead code for the deployed app now but
still used by `devserver.py` for local dev (plain local Python networking
doesn't hit this sandbox-specific issue), so left in place.

`glasses/index.html` inlines its CSS directly in a `<style>` tag rather than
linking a separate `glasses/styles.css` (deleted 2026-09-16) — on-device
testing showed the glasses WebView occasionally failing to load a linked
stylesheet while `index.html` and the JS files loaded fine, which leaves
every `.screen` visible at once with default browser styling (tiny fonts,
default `<input>`/`<button>` borders) since the `.hidden`/`.screen` rules
that hide and position screens never apply — this reads as "wrong screen
launches" and "can't navigate back," not obviously a missing-stylesheet
issue. Inlining removes the second network request entirely. If you edit
styles, edit the `<style>` block in `glasses/index.html` directly — there's
no longer a separate CSS file for glasses to keep in sync.

Same fix applied to the JS (2026-09-16): `glasses/index.html` now inlines
`storage.js`, `scoring.js`, `data.js`, and the glasses `app.js` into one
`<script>` block instead of four separate `<script src>` tags. Reported
symptom on-device: the ticker list stayed completely empty (not even the
"Watchlist is empty" message, which only renders from JS) and every button
press, including "+ Add Ticker", did nothing. That combination means no JS
ran at all — `app.js` is the last of the four script requests, and losing
it (or a file it depends on) leaves only the static HTML shell, since
`setupEvents()` (click handling) and `renderHome()` (list rendering) both
live inside it. Root cause presumed to be the same class of intermittent
script-load failure already confirmed for the CSS case above. If you edit
`storage.js`, `scoring.js`, or `data.js` (shared with the phone page), copy
the change into the `<script>` block in `glasses/index.html` too — there's
no longer a build step that would do this automatically.

## Toolkit compliance

`glasses/` was rebuilt against the actual toolkit template
(`plugins/meta-wearables-webapp/skills/create-webapp/templates/`), not the
original aesthetic mockups, after cloning the toolkit and finding real
mismatches:

| Assumption from early mockups | What the real toolkit requires |
|---|---|
| Touch: tap to expand, swipe to scroll, hold to mute | No touchscreen at all — D-pad only. Arrow keys move focus (wrap-around), Enter activates, Escape goes back. Mute is a normal focusable button, not a synthetic long-press. |
| Custom ~340×380px "lens" mockup frame | Fixed 600×600dp viewport, 8dp safe margin |
| Amber trading-terminal palette throughout | `#000000` transparent page background (mandatory — real world shows through on the additive display), visible UI surfaces in `#0a0a0f`–`#1C1E21`, **cyan focus ring is a hardware/legibility requirement, not a style choice**. Amber is kept for prices/CATALYST tags on top of that base. |
| `window.storage` bridge for phone↔glasses sync | Plain `localStorage`, no bridge — hence the `/api/watchlist` backend described above |
| Visible `←` back-button in screen headers | Toolkit removed the `back-btn` pattern from its templates/examples entirely (as of toolkit commit `a2714f8`/`ca5fb95`, re-audited 2026-09-15) — headers are just `<h1>`, and Escape (already wired to `navigateBack()`) is the sole back mechanism. `glasses/` updated to match. |
| `'IBM Plex Sans'`/`'IBM Plex Mono'` named throughout `glasses/styles.css` | Never actually loaded on this surface — no `@font-face`/`<link>`/`@import` in `glasses/`, only the phone page's `styles.css` has the Google Fonts `@import` (fine there, off-hardware). Every glasses element silently fell back to a generic browser `monospace`/`sans-serif`, producing an inconsistent mix. `performance-guidelines.md`'s Assets checklist is explicit: **"No external font downloads. Use the system font stack."** — so the fix isn't to add the missing `@import` (would violate that), it's to drop the dead font names and unify on the same system stack the official template uses (`-apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif`), everywhere. |

Re-audited against the toolkit repo again on 2026-09-15 (toolkit was at
`de9ebda` when this app was built, upstream had moved to `ca5fb95`). The
600×600dp viewport, 8dp safe margin, header/button dimensions, and color
system in `display-guidelines.md` are **unchanged** since this app was
built — the only functional doc changes were the back-button removal above
and the new "Pinch, drag, and text entry" section (see below).

`data.js`, `scoring.js`, and `storage.js` are shared unmodified between both
surfaces (`glasses/index.html` loads them via `../`) — only the UI/interaction
layer differs.

## Text entry on glasses (Add Ticker)

The Add Ticker screen (`glasses/index.html`) uses a plain
`<input type="text" class="text-input focusable">`, following the toolkit's
own "Form Screen" pattern (`skills/add-ui/references/vanilla-patterns.md`).
There is no SDK call needed for the Neural Band's handwriting/dictation
input, but per the toolkit's `add-text-input` skill and the "Pinch, drag,
and text entry" section added to `display-guidelines.md` (toolkit commits
`de9ebda..ca5fb95`), the on-glasses **composer opens on focus + tap, not on
focus alone**, and never via a programmatic `.focus()`. A pinch on an
already-focused text input is consumed by the composer instead of reaching
page JS at all — so this app's job is just to provide a standard,
correctly-focusable, `type="text"` input; the composer itself isn't testable
from a desktop browser preview, only confirmable on physical hardware.

While the input is focused, arrow keys are left alone (so typing/cursor
movement isn't hijacked by D-pad focus-navigation) — only Enter (submit,
via `data-submit-action`) and Escape (cancel) are intercepted, matching the
*current* official template's keydown handler exactly (re-verified against
toolkit commit `ca5fb95` on 2026-09-15 — this part hadn't drifted). On real
hardware this Enter path never actually fires while the field is focused: a
pinch on a focused text input is consumed by the on-glasses composer before
it reaches page JS at all, so it's a keyboard-testing/fallback affordance,
not the handwriting entry point itself. A live `input` listener clears the
"already on watchlist" hint as soon as the composer commits (or the wearer
types), which is the toolkit's recommended way to react to composer input.

## What's implemented

- **Catalyst scoring** (`scoring.js`): category weight (earnings/guidance/M&A/
  regulatory/analyst action) × recency decay × a real-move confirmation bonus
  (price change ≥ 1.5%), matching the framework style used by the
  `swing-trade-architect` / `daily-catalyst-scanner` skills. The `CATALYST`
  tag only appears above a threshold score.
- Default 19-ticker watchlist, plus ad-hoc ticker add/remove on both surfaces.
- **Real data, no API keys**: prices and headlines both from Yahoo Finance's
  public (unofficial) endpoints — see `api/_shared.py`. Google News RSS was
  tried first for headlines and works from a normal IP, but Google blocks
  Vercel's Lambda IP ranges; Yahoo's own news search endpoint doesn't have
  that problem.
- **Shared watchlist sync**: `/api/watchlist` (GET/POST), backed by Upstash
  Redis via its REST API (`KV_REST_API_URL`/`KV_REST_API_TOKEN`, the env vars
  Vercel injects once the database is connected to the project).
- If any live call fails, both pages fall back to mock/default data so the UI
  never breaks.

## Known limitation

`/api/watchlist`'s POST has no auth — anyone with the deployed URL could
overwrite the watchlist. Low stakes for a personal ticker list, but worth
knowing if you ever share the URL.

## Remaining step: load onto physical glasses

Everything above is finished, deployed, and tested in a browser (including
D-pad keyboard navigation). What's left needs the actual hardware, which I
don't have access to test against:

1. Enable Developer Mode in the Meta AI app
2. Use the toolkit's publish skill to generate a QR code from
   `https://ray-ban-watchlist.vercel.app/glasses/`
3. Scan it from the glasses

Happy to debug alongside you once you hit that stage — I can't verify
anything about the actual on-device rendering/gesture behavior from here.
