# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This repository stores sprite assets for **Cataclysm: The Last Generation** (CTLG), a fork of Cataclysm: Dark Days Ahead. Assets are PNG sprites (plus layered source files in PSD/XCF/KRA). The CI pipeline compiles these PNGs into game-ready tilesets using `compose.py` from the main CDDA repo.

## Building Tilesets

**Prerequisites:** Python 3, `pyvips`, and `libvips`.

```sh
# Via Nix (recommended — enters a devshell with all deps)
nix develop .
python3 tools/compose.py --use-all gfx/<TilesetName> out/

# Build a specific tileset via Nix
nix build .#UltimateCataclysm   # output in ./result/

# Manual (requires pyvips installed)
pip3 install pyvips
python3 /path/to/compose.py --use-all gfx/<TilesetName> out/
```

`compose.py` is maintained in the upstream CDDA repo (`tools/gfx_tools/compose.py`) and is downloaded fresh by CI at build time. It is not vendored here.

Common `compose.py` flags used in CI:
- `--use-all` — include all sprites (most tilesets)
- `--obsolete-fillers` — also include obsolete filler sprites (Altica, BrownLikeBears, HollowMoon, UltimateCataclysm)
- `--feedback CONCISE --loglevel INFO` — used in CI for readable output

## Autotile Slicing Tools

`tools/slice_multitile.py` — slices a multitile template PNG into individual tile images + JSON connection data:

```sh
tools/slice_multitile.py mud_autotile.png 32 --out mud_tiles
tools/slice_multitile.py --iso iso_autotile.png 64  # isometric
```

`tools/slice_variants.py` — slices randomly-weighted sprite variant sheets:

```sh
tools/slice_variants.py t_floor_multitile.png 32 32
```

## Repository Architecture

```
gfx/
  <TilesetName>/          # One directory per tileset
    tile_info.json        # Sprite sheet dimensions and offsets per sheet type
    tileset.txt           # Tileset metadata (name, description)
    layering.json         # (optional) Layering order for overlapping sprites
    fallback.png          # (optional) Fallback sprite
    pngs_normal_32x32/    # Regular 32×32 sprites
    pngs_large_64x64/     # Large 64×64 sprites
    pngs_character_32x36/ # Character/overlay sprites
    pngs_tall_32x42/      # Tall sprites
    ... (other size variants)

templates/                # Autotile and multitile grid templates (PNGs)
tools/                    # Python helper scripts (slice_multitile.py, etc.)
doc/                      # mdBook documentation (built via .github/workflows/book.yml)
scratch/                  # Working/draft files, not compiled into releases
```

Each `pngs_*` folder name encodes the sprite sheet type and dimensions. The `tile_info.json` in the tileset root maps sheet filenames to sprite dimensions/offsets.

## Sprite JSON Format

Each sprite (or group of sprites) has a sidecar `.json` file with the game IDs it covers:

```json
{ "id": "overlay_male_worn_longshirt", "fg": "longshirt_m", "bg": "" }
```

Arrays are used when a single file covers multiple IDs. The `fg` value is the PNG filename (without extension) relative to its directory.

## CI / Workflows

Each tileset has its own CI workflow (e.g., `.github/workflows/msx_ci_build.yml`) that triggers on pushes/PRs touching `gfx/<TilesetName>/**`. All delegate to `.github/workflows/composer_template.yml`.

Weekly automated releases run every Sunday at 20:00 UTC via `make_release.yml`, compiling all tilesets and attaching them as GitHub release artifacts.

## EditorConfig

File formatting is enforced via `.editorconfig`. A conformance check runs in CI (`.github/workflows/editorconfig_conformance.yml`). Respect existing indentation rules when editing JSON files.

---

# context-mode — MANDATORY routing rules

You have context-mode MCP tools available. These rules are NOT optional — they protect your context window from flooding. A single unrouted command can dump 56 KB into context and waste the entire session.

## BLOCKED commands — do NOT attempt these

### curl / wget — BLOCKED
Any Bash command containing `curl` or `wget` is intercepted and replaced with an error message. Do NOT retry.
Instead use:
- `ctx_fetch_and_index(url, source)` to fetch and index web pages
- `ctx_execute(language: "javascript", code: "const r = await fetch(...)")` to run HTTP calls in sandbox

### Inline HTTP — BLOCKED
Any Bash command containing `fetch('http`, `requests.get(`, `requests.post(`, `http.get(`, or `http.request(` is intercepted and replaced with an error message. Do NOT retry with Bash.
Instead use:
- `ctx_execute(language, code)` to run HTTP calls in sandbox — only stdout enters context

### WebFetch — BLOCKED
WebFetch calls are denied entirely. The URL is extracted and you are told to use `ctx_fetch_and_index` instead.
Instead use:
- `ctx_fetch_and_index(url, source)` then `ctx_search(queries)` to query the indexed content

## REDIRECTED tools — use sandbox equivalents

### Bash (>20 lines output)
Bash is ONLY for: `git`, `mkdir`, `rm`, `mv`, `cd`, `ls`, `npm install`, `pip install`, and other short-output commands.
For everything else, use:
- `ctx_batch_execute(commands, queries)` — run multiple commands + search in ONE call
- `ctx_execute(language: "shell", code: "...")` — run in sandbox, only stdout enters context

### Read (for analysis)
If you are reading a file to **Edit** it → Read is correct (Edit needs content in context).
If you are reading to **analyze, explore, or summarize** → use `ctx_execute_file(path, language, code)` instead. Only your printed summary enters context. The raw file content stays in the sandbox.

### Grep (large results)
Grep results can flood context. Use `ctx_execute(language: "shell", code: "grep ...")` to run searches in sandbox. Only your printed summary enters context.

## Tool selection hierarchy

1. **GATHER**: `ctx_batch_execute(commands, queries)` — Primary tool. Runs all commands, auto-indexes output, returns search results. ONE call replaces 30+ individual calls.
2. **FOLLOW-UP**: `ctx_search(queries: ["q1", "q2", ...])` — Query indexed content. Pass ALL questions as array in ONE call.
3. **PROCESSING**: `ctx_execute(language, code)` | `ctx_execute_file(path, language, code)` — Sandbox execution. Only stdout enters context.
4. **WEB**: `ctx_fetch_and_index(url, source)` then `ctx_search(queries)` — Fetch, chunk, index, query. Raw HTML never enters context.
5. **INDEX**: `ctx_index(content, source)` — Store content in FTS5 knowledge base for later search.

## Subagent routing

When spawning subagents (Agent/Task tool), the routing block is automatically injected into their prompt. Bash-type subagents are upgraded to general-purpose so they have access to MCP tools. You do NOT need to manually instruct subagents about context-mode.

## Output constraints

- Keep responses under 500 words.
- Write artifacts (code, configs, PRDs) to FILES — never return them as inline text. Return only: file path + 1-line description.
- When indexing content, use descriptive source labels so others can `ctx_search(source: "label")` later.

## ctx commands

| Command | Action |
|---------|--------|
| `ctx stats` | Call the `ctx_stats` MCP tool and display the full output verbatim |
| `ctx doctor` | Call the `ctx_doctor` MCP tool, run the returned shell command, display as checklist |
| `ctx upgrade` | Call the `ctx_upgrade` MCP tool, run the returned shell command, display as checklist |
