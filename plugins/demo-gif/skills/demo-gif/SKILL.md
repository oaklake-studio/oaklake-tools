---
name: demo-gif
description: Use when the user wants a demo GIF / screen recording of a web UI (for a PR, README, changelog, or showcase) in ANY project. Drives a browser through a short scripted flow you define with the user, records one continuous video (headless by default), and converts it to a calm, followable GIF saved under .demo-gifs/ at the project root (auto-added to .gitignore when the root is a git repo). Browser/UI demos only, not terminal/CLI recordings. Assumes a POSIX shell (macOS/Linux; on Windows use WSL or git-bash).
allowed-tools: Read, Write, Edit, Bash
---

# demo-gif

## Overview

Records a demo GIF of a running web UI in any project. Pipeline: prepare a reusable Playwright environment → scope the demo with the user → resolve the output folder + gitignore + a run-config file → write a capture script → record the run as one continuous video → verify each beat by screenshot → trim and convert to a palette-optimized GIF under `.demo-gifs/`.

Two principles carry the quality:
- **Calm and followable.** Hold a beat on the full view (~3s), then perform **one deliberate interaction at a human pace**, then hold again. Don't rapid-fire toggles or cram actions. A slow, clear story reads far better.
- **Artifacts, not repo files.** GIFs are large binaries. They live in `.demo-gifs/` at the project root, gitignored by default. The user attaches them where needed.

Recording is **headless by default** (deterministic, works over SSH/CI/no-display). The browser records video fine headless; you don't need a visible window.

## When to Use

- "Make/record a demo gif of <feature>", "capture this for the PR/README", "screen-record the flow"
- Re-taking a demo (bump the version: `-v1` → `-v2`)

**Do NOT use when:**
- The demo needs a **real third-party SSO login** on camera (Google/Okta/…). Hands-off automation is blocked by modern browsers, so use hybrid capture: run headed, drive to the provider's screen, then poll only the URL (never screenshot credentials) while the **user** signs in; delete the throwaway profile after.
- It's a **terminal/CLI** demo: use a terminal recorder (asciinema), not this skill.
- A still screenshot would do, just take one.

## ⚠️ Shell state does not persist between Bash calls

Each Bash tool call is a **fresh shell**: variables set in one call are gone in the next, and you MUST stop between capture and conversion to Read the screenshots. So: **Step 2 writes a run-config file; every later Bash block starts by sourcing it.** Never assume a variable set in an earlier step still exists.

```bash
RUNENV="$HOME/.cache/claude/demo-gif/run.env"   # fixed path used by every step
```

## Step 0: Reusable environment (idempotent; once per machine)

POSIX shell required (macOS/Linux; on Windows run this under WSL or git-bash). Keep Playwright in a **dedicated persistent venv** outside any project, because dependency managers (`uv sync`, `poetry install`) prune anything not in their lockfile, which is why an in-project install "disappears."

```bash
VENV="$HOME/.cache/claude/demo-gif/venv"; PW="$VENV/bin/python"
mkdir -p "$HOME/.cache/claude/demo-gif"
command -v python3 >/dev/null || { echo "python3 required"; exit 1; }
python3 -c 'import sys; raise SystemExit(0 if sys.version_info>=(3,9) else 1)' || { echo "need python3 >= 3.9"; exit 1; }

# capability-gated (not just "does bin/python exist"): a half-built venv is rebuilt, not reused
if ! "$PW" -c 'import playwright' 2>/dev/null; then
  rm -rf "$VENV"                       # a partial venv must be deleted, never reused
  python3 -m venv "$VENV" || { echo "venv failed (Debian: sudo apt-get install python3-venv)"; exit 1; }
  "$VENV/bin/pip" install -q --upgrade pip playwright || { echo "pip failed: see stderr; behind a proxy set HTTPS_PROXY"; exit 1; }
fi

# ensure a real browser exists: prefer system Chrome, else fetch bundled Chromium once (~150MB)
if ! "$PW" - <<'PY' 2>/dev/null
from playwright.sync_api import sync_playwright
with sync_playwright() as p:
    try: b = p.chromium.launch(headless=True, channel="chrome")
    except Exception: b = p.chromium.launch(headless=True)   # bundled chromium (raises if not installed)
    b.close()
PY
then
  echo "no usable browser: installing bundled Chromium…"
  "$PW" -m playwright install chromium || echo "chromium install failed (network/proxy?). On Linux use: $PW -m playwright install --with-deps chromium"
fi

command -v ffmpeg >/dev/null || echo "WARN: ffmpeg missing, install it (brew install ffmpeg | apt-get install ffmpeg | dnf install ffmpeg | choco install ffmpeg)"
echo "ready: $PW"
```

If anything above fails, **surface the stderr and stop**. Don't proceed on a broken environment. `channel="chrome"` skips the browser download entirely when Chrome is present; on headless Linux you generally want the bundled Chromium with `--with-deps` (pulls libnss3/libatk/libgbm).

## Step 1: Scope the demo

Ask the user only what you can't infer:
1. **What we're demoing**: a one-line story + a short slug for the filename (e.g. `checkout-flow`).
2. **Base URL** + whether the app is **already running** (if not, how to start it, or have them start it).
3. **Login**: none / credentials / a described flow. Never guess credentials; for real SSO see "Do NOT use."
4. **Theme + data**: light or dark (`color_scheme`)? And confirm the target has **meaningful, stable demo data** (not an empty/half-seeded DB or randomized rows) so the GIF shows something real.
5. **The beats**: 3–4 max, each one deliberate action.

Discover selectors from the live page with a quick throwaway script using the Step-0 venv (navigate + `page.get_by_role(...)` / `page.content()` / a snapshot screenshot) rather than guessing. (If the Playwright MCP is available in your session, its `browser_snapshot` also works.)

## Step 2: Output dir + gitignore + run-config

Resolve the project root (git top-level if in a repo, else cwd), make `.demo-gifs/`, gitignore it (only when not already effectively ignored), pick the next version, and persist everything to `$RUNENV`:

```bash
RUNENV="$HOME/.cache/claude/demo-gif/run.env"
VENV="$HOME/.cache/claude/demo-gif/venv"
if ROOT=$(git rev-parse --show-toplevel 2>/dev/null); then IS_GIT=1; else ROOT=$(pwd); IS_GIT=0; fi
mkdir -p "$ROOT/.demo-gifs"
if [ "$IS_GIT" = 1 ] && ! git -C "$ROOT" check-ignore -q .demo-gifs/ 2>/dev/null; then
  printf '\n# demo/screencast artifacts (claude demo-gif skill)\n.demo-gifs/\n' >> "$ROOT/.gitignore"
fi

SLUG="<slug>"; SCHEME="light"        # set SLUG; SCHEME=dark if the app is dark-first
N=1; while [ -e "$ROOT/.demo-gifs/$SLUG-v$N.gif" ]; do N=$((N+1)); done
WORK=$(mktemp -d "${TMPDIR:-/tmp}/demo-gif-$SLUG.XXXXXX")

cat > "$RUNENV" <<EOF
PW="$VENV/bin/python"
ROOT="$ROOT"
WORK="$WORK"
SLUG="$SLUG"
N="$N"
OUT="$ROOT/.demo-gifs/$SLUG-v$N.gif"
BASE="<base-url>"
SCHEME="$SCHEME"
EOF
cat "$RUNENV"
```

(If a project deliberately wants demos committed, the user just removes the `.demo-gifs/` line; mention only if relevant.)

## Step 3: Write the capture script

Write the template below to `$WORK/capture.py` (source `$RUNENV` first to know `$WORK`). Set/delete the login block and replace `# ===== BEATS =====`. For each beat: perform one action → **wait on the resulting element** (`get_by_*().wait_for()`), never `networkidle` → `hold()` → `shot()`. Scroll to below-the-fold content with a short `page.mouse.wheel` loop so the motion reads.

## Step 4: Run + verify

```bash
source "$HOME/.cache/claude/demo-gif/run.env"
rm -f "$WORK"/video/*.webm 2>/dev/null; mkdir -p "$WORK/video"   # clean slate so re-runs don't reuse a stale take
DEMO_URL="$BASE" DEMO_WORK="$WORK" DEMO_SCHEME="$SCHEME" "$PW" "$WORK/capture.py"
```

Then **Read the beat screenshots** in `$WORK`: confirm framing and that each beat landed on real content (not a loader/empty state/overlay). Fix the script and re-run before converting. To pick trim points, build a duration-spanning labeled montage and Read it:

```bash
source "$HOME/.cache/claude/demo-gif/run.env"
WEBM=$(ls -t "$WORK"/video/*.webm | head -1)
DUR=$(ffprobe -v error -show_entries format=duration -of csv=p=0 "$WEBM")
STEP=$(awk "BEGIN{s=$DUR/20; print (s<0.5?0.5:s)}")   # 20 tiles always span the whole clip
ffmpeg -y -i "$WEBM" -vf "fps=1/$STEP,scale=320:-1,drawtext=text='%{pts\:hms}':x=6:y=6:fontsize=20:fontcolor=yellow:box=1:boxcolor=black@0.6,tile=5x4" -frames:v 1 "$WORK/montage.png" 2>/dev/null \
  || ffmpeg -y -i "$WEBM" -vf "fps=1/$STEP,scale=320:-1,tile=5x4" -frames:v 1 "$WORK/montage.png"   # fallback if drawtext/font unavailable
echo "montage tiles are ${STEP}s apart (tile k ≈ k*${STEP}s), clip is ${DUR}s"
```

## Step 5: Convert to GIF

```bash
source "$HOME/.cache/claude/demo-gif/run.env"
WEBM=$(ls -t "$WORK"/video/*.webm | head -1)
DUR=$(ffprobe -v error -show_entries format=duration -of csv=p=0 "$WEBM")
A=0; B=$DUR; FPS=14; SCALE=800        # set A/B from the montage (drop login head + dead tail)
awk "BEGIN{exit !($A<$DUR && $B>$A)}" || { echo "bad trim window: A=$A B=$B DUR=$DUR"; exit 1; }
PAL="$WORK/pal.png"
ffmpeg -y -i "$WEBM" -vf "trim=$A:$B,setpts=PTS-STARTPTS,fps=$FPS,scale=$SCALE:-1:lanczos,palettegen=stats_mode=diff:max_colors=200" "$PAL"
[ -s "$PAL" ] || { echo "palette empty: trim window likely off"; exit 1; }
ffmpeg -y -i "$WEBM" -i "$PAL" -lavfi "[0:v]trim=$A:$B,setpts=PTS-STARTPTS,fps=$FPS,scale=$SCALE:-1:lanczos[x];[x][1:v]paletteuse=dither=bayer:bayer_scale=4" -loop 0 "$OUT"
ffmpeg -y -sseof -1 -i "$OUT" -frames:v 1 "$WORK/check.png"   # Read this to confirm legibility at scale
ls -lh "$OUT"
```

## Step 6: Report + clean up

State the path (`.demo-gifs/<slug>-v<N>.gif`), size, dimensions, duration, and the beats; note whether `.demo-gifs/` was added to `.gitignore`. Then `rm -rf "$WORK"`. Offer a re-take (`-v<N+1>`) for different framing/pacing.

## Capture script template

```python
"""<slug> demo: <story>. Records continuous video into DEMO_WORK; ffmpeg trims after."""
import os
from pathlib import Path
from playwright.sync_api import sync_playwright

BASE   = os.environ.get("DEMO_URL", "http://localhost:3000")
WORK   = Path(os.environ["DEMO_WORK"]); (WORK / "video").mkdir(parents=True, exist_ok=True)
SCHEME = os.environ.get("DEMO_SCHEME", "light")
HEADED = os.environ.get("DEMO_HEADED") == "1"   # default headless, works with no display
W, H = 1360, 900

def make_browser(p):
    try:    return p.chromium.launch(headless=not HEADED, channel="chrome")  # system Chrome
    except Exception:
        return p.chromium.launch(headless=not HEADED)                        # bundled chromium

with sync_playwright() as p:
    browser = make_browser(p)
    ctx = browser.new_context(
        viewport={"width": W, "height": H},
        record_video_dir=str(WORK / "video"), record_video_size={"width": W, "height": H},
        reduced_motion="reduce", color_scheme=SCHEME,   # deterministic frames; correct theme
    )
    page = ctx.new_page()
    hold = lambda ms=2500: page.wait_for_timeout(ms)
    def shot(n): page.screenshot(path=str(WORK / n), animations="disabled"); print("shot", n, flush=True)
    def hide_overlays():  # dev indicators / common banners that pollute frames (extend per app)
        page.add_style_tag(content="nextjs-portal,[data-nextjs-toast],#__next-build-watcher{display:none!important}")

    try:
        # ===== BEATS (customize) =====
        page.goto(BASE, wait_until="domcontentloaded")
        page.get_by_role("heading").first.wait_for(timeout=30000)   # wait on a REAL element, never networkidle
        hide_overlays(); hold(3000); shot("01.png")
        # BEAT 2, one deliberate action, wait on its result, then hold:
        #   page.get_by_role("button", name="<label>").click()
        #   page.get_by_text("<expected>").first.wait_for(timeout=15000); hold(); shot("02.png")
        # BEAT 3, below the fold? Smooth-scroll first:
        #   for _ in range(8): page.mouse.wheel(0, 90); page.wait_for_timeout(55)
        #   ...; hold(); shot("03.png")
        # ===== END BEATS =====
    finally:
        ctx.close(); browser.close()   # ALWAYS close: Playwright only finalizes the .webm here

vids = sorted((WORK / "video").glob("*.webm"), key=lambda f: f.stat().st_mtime)
print("VIDEO:", vids[-1] if vids else "NONE (capture failed before any frame)", flush=True)
```

A thrown beat still leaves a usable partial video (the `finally` finalizes it). Inspect it to see which beat failed.

## Design notes (calm & followable)

- Open holding on the full view so the viewer reads the starting state.
- One deliberate interaction per beat; waiting on the resulting element keeps timing honest across machines.
- A slow tooltip/hover sweep across a chart or list reads better than toggling things on/off (step `page.mouse.move(...)` across an element's bounding box with small waits).
- Pick a viewport that fits the important content; scroll to anything below the fold rather than shrinking everything.
- Continuous animation both breaks deterministic screenshot verification and defeats `palettegen`'s diff mode (bloating the GIF), hence `reduced_motion` + `animations="disabled"`.

## Sizing knobs (in priority order)

Detail-heavy UIs and full-page scrolls inflate GIF size. Turn these down if it's bigger than wanted:
1. **Duration**: tighter trim / shorter holds (biggest lever).
2. **fps**: 16 → 14 → 12 → 10.
3. **scale**: 900 → 800 → 720.
4. `paletteuse=dither=bayer:bayer_scale=4|5`; `palettegen ...:max_colors=200|128`.

Rule of thumb: a ~15s UI walkthrough at fps=14 / scale=800 lands ~2–3 MB. GitHub accepts GIFs ≤10 MB, so don't over-optimize unless asked. `gifsicle -O3 --lossy=80` squeezes further if it's installed.

## Gotchas

- **Never use `networkidle`.** It never settles on apps with a websocket/SSE/polling (Vite/Next HMR, live dashboards) and times out. Wait on a concrete element instead.
- **Headless by default.** Set `DEMO_HEADED=1` only when you need a visible window (e.g. SSO hybrid); on headless Linux, headed needs `xvfb-run`.
- **Never install Playwright into a project `.venv`**. Lockfile managers prune it. Use the `~/.cache/claude/demo-gif/venv` from Step 0; if `import playwright` fails, `rm -rf` that venv and re-run Step 0.
- **iframes:** `page.get_by_*` won't reach inside them. Use `page.frame_locator("iframe#…").get_by_role(...)` (Storybook canvas, embedded widgets).
- **Mobile demo:** build the context from a device preset (`**p.devices["iPhone 13"]`) and match `record_video_size`, instead of the 1360×900 desktop default.
- **Stale dev server:** if the UI shows old/wrong data, the running server may predate the code under test; restart it before filming.
- **SSO logins aren't automatable** (see "Do NOT use"): hybrid capture, user signs in, delete the throwaway profile after.
- **Keep GIFs out of git:** `.demo-gifs/` is gitignored by default; don't commit large binaries into history unless the user explicitly asks.
- **Proxy/offline:** if pip or the browser download fails, surface stderr and stop; levers are `HTTPS_PROXY`/`HTTP_PROXY`, `PLAYWRIGHT_DOWNLOAD_HOST`, or `channel="chrome"` to skip the download.
