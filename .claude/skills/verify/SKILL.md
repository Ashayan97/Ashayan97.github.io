---
name: verify
description: Drive this static site in a real browser to verify a change. Use when verifying edits to index.html or stylesheet.css.
---

# Verifying this site

Static site, no build step and no server needed — load `index.html` over `file://`
directly. The whole site is `index.html` + `stylesheet.css`; the `zipnerf/`,
`mipnerf/`, `mipnerf360/` dirs are unrelated project pages, leave them alone.

## Handle: Playwright driving the installed Chrome

There is no node/npm on this machine, but Google Chrome is installed. Playwright's
Python package drives it with `channel="chrome"`, so no browser download is needed:

```bash
pip3 install --quiet --user playwright     # already installed as of Jul 2026
```

```python
from playwright.sync_api import sync_playwright
URL = "file:///Users/shayan/Codes/personal_website/Ashayan97.github.io/index.html"
with sync_playwright() as p:
    b = p.chromium.launch(channel="chrome")          # no download; uses installed Chrome
    pg = b.new_page(viewport={"width": 1280, "height": 900})
    pg.goto(URL)
    pg.screenshot(path="out.png")
```

Useful page options: `reduced_motion="reduce"` to exercise the
`prefers-reduced-motion` block; `viewport={"width":390,"height":844}` for the
mobile breakpoint (the layout switches to a column at 640px).

Theme is set from `localStorage` before paint; to force one in a test:
`pg.evaluate("document.documentElement.setAttribute('data-theme','light')")`.
**Check both themes** — the palette is token-driven and contrast differs.

## Gotcha: do not verify animations with Chrome's `--virtual-time-budget`

`chrome --headless --screenshot --virtual-time-budget=N` seems attractive, but
**CSS animations do not tick on the virtual clock while `setTimeout` does**. The two
clocks decouple, so animation-driven logic reports false failures: progress bars read
`0px`, and `animationend` never fires, which makes anything gated on it look broken
when it is fine in a real browser. `--dump-dom` is worse — it never paints, so no
animation runs at all. Use real time via Playwright and `wait_for_timeout`.

## What to drive

Things worth exercising after a change to the hero or story viewer:

- Click `#story-trigger` → story opens; Esc / `.story-backdrop` / `.story-next` / the
  `.story-close` button all close it; `.story-prev` replays.
- The story self-closes when `.story-progress-fill` finishes (6s) — assert on the
  fill's measured width to check progress advances, restarts, and pauses on hold.
- After closing, confirm the page is still clickable (`document.elementFromPoint` over
  the hero heading) — a story left with `hidden=false` would trap clicks under an
  invisible overlay.
- Body scroll lock (`body.story-open`) is applied while open and released after.
