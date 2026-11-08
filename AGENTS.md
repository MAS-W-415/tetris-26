# AGENTS.md

Tetris 26 is a local, single-page 26-piece Tetris variant (polyominoes of 3–5 cells)
with a numpy AI. Python 3.11+ stdlib HTTP server, vanilla JS UI; no frontend build step.

## Commands

- `uv sync` — creates `.venv` and provisions Python 3.11 (the system Python may be older).
- `uv run python run.py` — serve <http://127.0.0.1:8000> and open a browser.
  Flags: `--port`, `--host`, `--no-browser`, `--public`.
- There is no test suite, linter, formatter, or typecheck config. Minimum check:
  `uv run python -m py_compile *.py`, then start the server and inspect `GET /state`.
- Single-file build: `uv sync --dev && uv run pyinstaller tetris26.spec --noconfirm`
  (artifact: `dist/tetris26`, or `dist/tetris26.exe` on Windows). Add new runtime
  data files to `tetris26.spec`'s `datas`;
  `paths.base_dir()` already handles source vs PyInstaller `_MEIPASS`.
  CI builds Linux/macOS/Windows and publishes on `v*` tags.
- Runtime deps are deliberately minimal (numpy only); the spec excludes torch etc.
  Prefer stdlib/pure numpy for code that ships in the executable.

## Development-mode gotcha

- The tracked `.env` sets `PUBLIC=1` (and is bundled into release builds), so a
  source run is player-facing: `/observer` is disabled and `/state` omits `scores`,
  `hardest`, `player`, `ai_aps`, sampler logs, and benchmark results. There is no
  CLI flag to turn this off; set `PUBLIC=0` in `.env` (revert before committing)
  or move `.env` aside.
- `server.py` references `ui/observer.html` / `ui/observer.js`, but those files are
  not in this repository; `/observer` 404s even in non-public mode.
- `run.py` parses `.env` itself (no python-dotenv); keys are `HOST`, `PORT`,
  `AUTO_OPEN`, `PUBLIC`.

## Code map and invariants

- `engine.py` is the rule source of truth; `placement.py` reimplements the same
  collision/rotation/line-clear rules for fast AI enumeration. Change both together.
- Board cells are `0` (empty) or `piece_index + 1`; that value doubles as the UI color index.
- AI path: `pieces.py` (26 pieces) -> `placement.py` (afterstates) ->
  `dt_features.py` (9 DT features, DT10/DT20) -> `search.py` (lookahead) ->
  `assist.py` (hint levels 1–3). `difficulty.py` samples among the hardest pieces.
- Weight wiring is easy to misread: the server's `hardest_fast` (difficulty and
  scoring panel) always uses DT20; assistant hints use the gitignored optional
  `tuned_weights.npz`, else DT20. `weights.npz` is only used by
  `agent.load_score()` / `panel_scores` / `panel_hardest`, which no runtime path calls here.
- `server.py` runs one shared game in a loop thread at 30 Hz while browsers poll
  `/state` every 33 ms and POST commands to `/control`; `Game.lock` guards shared
  state. Hints are computed in a worker thread, and gravity waits up to 1.5 s for
  the precise hint when assist is enabled. A new game starts paused; sending
  `{"cmd":"restart"}` starts it.
- `train/` is referenced in docstrings but is not part of this repo; don't look for it.

## UI conventions

- Static files live in `ui/` and are served through a hardcoded route table in
  `Handler.do_GET`; a new asset needs an entry there.
- Adding a theme requires both `ui/theme.js` (`THEMES`: palette/render params) and
  `ui/themes.css` (CSS variables under `body[data-theme=...]`). Add a matching
  `THEME_MUSIC` entry in `ui/music.js` or it silently uses the `dark` track.
- User-facing strings are Simplified Chinese; code and comments are English.
- `README.md` is English and `README.zh-CN.md` is Chinese. Update both when
  features, controls, downloads, or screenshots change; do not mix languages in
  one file.
- Browser settings use the `tetris26_*` localStorage prefix. `ui/theme.js`
  migrates old `polyfall_*` keys on load; keep that compatibility path.
- Supported URL params: `?theme=<key>`, `?assist=0..3`, `?play=1`.
