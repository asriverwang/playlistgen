# PlaylistGen — Agent Guide

This document is for AI agents (OpenClaw and others) installing or operating PlaylistGen.

PlaylistGen is a local music library server with LLM-powered playlist generation. It runs entirely on the user's machine and serves playlists via a web UI and REST API.

---

## Installing PlaylistGen (fresh setup)

Work through these steps with the user. Each step requires user input or confirmation before proceeding.

### Step 1 — Gather requirements

Ask the user:
1. Where is their music directory? (e.g. `/home/user/Music`, `/media/data/Music`)
2. Do they have an Anthropic API key, MiniMax API key, or both? (Anthropic Haiku preferred)
3. What IP/port should playlist links use? (default `http://localhost:5678` works for local use; set `MUSIC_SERVER_URL` to a LAN/Tailscale IP if they want links to work on other devices)

### Step 2 — Create the virtual environment

```bash
cd /path/to/playlistgen
python3 -m venv venv
venv/bin/pip install requests openai anthropic mutagen
```

Also check that `ffprobe` is available:
```bash
ffprobe -version
# If missing: sudo apt install ffmpeg   (Linux) or brew install ffmpeg (macOS)
```

### Step 3 — Create .env

Copy `.env.example` to `.env` and fill in the values from Step 1:

```bash
cp .env.example .env
```

Required keys:
- `MUSIC_DIR` — the user's music directory
- `ANTHROPIC_API_KEY` (recommended) and/or `MINIMAX_API_KEY`

Optional:
- `MUSIC_SERVER_URL` — public IP for playlist links (e.g. `http://192.168.1.100:5678`)
- `PORT` — default is 5678

### Step 4 — Present MUSIC_RULES.md for review

**Read `MUSIC_RULES.md` and summarize the two sections to the user:**

- **Prompt Interpretation** — how the LLM converts a natural language request into music tags (genre counts, long-tail rules, language/energy/era detection)
- **Final Playlist Curation** — how the LLM selects and orders the final tracks (diversity, transitions, artist limits)

**Ask the user:** "Do you want to use the default rules, or customize them before we start?"

If they want to customize:
- Walk through each rule and ask if they want to change it
- Common customizations: artist limit (currently 3), whether to bias toward niche/mainstream, energy transition strictness
- Write the changes directly into `MUSIC_RULES.md` — no code changes needed

If they use defaults: proceed.

### Step 5 — Index the music library

This enriches every song with LLM-generated tags (genre, subgenre, mood, energy, language, region, usage context). It runs once and is safe to resume if interrupted.

```bash
source .env
venv/bin/python3 smart_indexer.py \
  --path "$MUSIC_DIR" \
  --llm haiku \
  --key "$ANTHROPIC_API_KEY" \
  --db "${DB_PATH:-music.db}" \
  --batch 40
```

Use `--llm minimax --key "$MINIMAX_API_KEY"` if using MiniMax instead.

**Timing heads-up to give the user:** indexing takes roughly 1–3 hours per 5,000 songs depending on the response time and quality of the LLM model. Progress is saved after every batch so they can stop and resume without losing work. By default the indexer prints a live progress line with rate, ETA, and error counts:
```
Phase 2 (LLM): 2057 songs in 103 batches | haiku / claude-haiku | workers=1 | timeout=120s
Phase 2 (LLM): 45/103 (43.7%) [2.3 songs/s] | avg 8.7s/batch | ETA 4m12s
```
Errors, retries, and dropped batches are always printed (even without `--verbose`) so the agent can diagnose issues. Add `--verbose` for full per-batch detail including raw LLM response snippets.

> **Warning:** Playlist generation quality depends directly on how many songs have been enriched. Advise the user to wait until at least 500 songs have been indexed by the LLM before starting the server and using the service. They can monitor progress via the live `Phase 2 (LLM): N/M (X%)` output and resume at any time if they need to pause.

When done, confirm: "Indexed N songs."

### Step 6 — Start the server

```bash
bash start.sh
```

Verify it's up:
```bash
curl -s http://localhost:5678/api/stats | python3 -m json.tool | head -5
```

Tell the user: "PlaylistGen is running at http://localhost:5678 — open it in a browser to search your library."

---

## Day-to-day operations

### Generate a playlist (API)

```
POST http://localhost:5678/api/generate
{"prompt": "obscure 80s synth for late night driving", "max_count": 30}
→ {"title": "...", "url": "http://.../player?saved=XXXX", "count": 28}
```

Share only the returned `url`. Do not construct player URLs manually.

### Generate a playlist (step by step, if you want to participate)

You can hook into Step 1 (interpret) and/or Step 3 (curate) yourself. See `TOOLS.md` for the full manual workflow.

### Search the library

```
GET http://localhost:5678/api/search?q=radiohead
```

### Re-index after adding new music

```bash
source .env
venv/bin/python3 smart_indexer.py --path "$MUSIC_DIR" --llm haiku --key "$ANTHROPIC_API_KEY" --db "${DB_PATH:-music.db}"
```

Skips already-indexed songs automatically.

### Refresh catalog vocabulary (after re-indexing)

```
GET http://localhost:5678/api/catalog/vocab/refresh
```

Or just restart the server — it rebuilds on startup.

### Update playlist rules

Edit `MUSIC_RULES.md` directly. Changes take effect on next server start. Restart:

```bash
bash start.sh
```

### Check server status

```bash
curl -s http://localhost:5678/api/stats
tail -f playlist_server.log
```

---

## Architecture summary

| Component | Role |
|---|---|
| `playlist_server.py` | HTTP server: all API endpoints, web UI, player |
| `smart_indexer.py` | One-time LLM enrichment — run standalone |
| `MUSIC_RULES.md` | Editable rules for interpret and curate LLM prompts |
| `music.db` | SQLite: songs table + playlists table |
| `music_catalog_vocab.json` | Auto-generated tag→count snapshot (written at server start) |

**Playlist generation pipeline (inside `/api/generate`):**
1. Interpret prompt → structured tags (LLM, guided by `MUSIC_RULES.md`)
2. Fetch candidates from DB — language is a hard filter; region/popularity are not
3. Curate: LLM selects and orders final playlist (guided by `MUSIC_RULES.md`)
4. Save playlist to DB → return player URL

**Key design principles:**
- All LLM rules live in `MUSIC_RULES.md`, not in code
- The catalog vocabulary is pre-built at startup (zero DB I/O per request)
- Language is enforced at fetch time; region/popularity are soft hints at curation time
- Player URLs come from the server — never construct them manually
