# PlaylistGen

A local music library server with LLM-powered playlist generation. Point it at your music folder, run the indexer once, and get a natural language playlist generator — accessible via web browser or API.

---

## What it does

- Scans your local music directory and builds a SQLite database of all songs
- Uses an LLM (MiniMax M2.7 or Claude Haiku) to enrich each song with genre, subgenre, mood, energy, language, region, and usage context
- Serves a local HTTP server with a search UI and player
- Accepts natural language prompts ("obscure 80s synth for late night driving") and generates curated playlists using LLM reasoning

---

## Requirements

- Python 3.10+
- `ffprobe` (part of ffmpeg) — for audio metadata extraction
- A music directory of MP3/FLAC/M4A/OGG/WAV files
- One of:
  - **MiniMax API key** (`MINIMAX_API_KEY`) — recommended, used for both indexing and playlist generation
  - **Anthropic API key** (`ANTHROPIC_API_KEY`) — Claude Haiku fallback

---

## Setup

### 1. Clone / copy this folder

```bash
cp -r playlistgen/ ~/my-playlistgen
cd ~/my-playlistgen
```

### 2. Create a virtual environment and install dependencies

```bash
python3 -m venv venv
source venv/bin/activate
pip install requests openai anthropic mutagen
```

Also install ffmpeg if not already present:
```bash
# Ubuntu/Debian
sudo apt install ffmpeg

# macOS
brew install ffmpeg
```

### 3. Configure environment

```bash
cp .env.example .env
# Edit .env — set MUSIC_DIR and at least one API key
```

### 4. Index your music library

This scans your music directory and enriches each song with LLM-generated tags. Runs once; safe to re-run (skips already-indexed songs).

```bash
source .env
python3 smart_indexer.py --path "$MUSIC_DIR" --llm minimax --key "$MINIMAX_API_KEY" --db "$DB_PATH"
```

**Time estimate:** plan for roughly 1–3 hours per 10,000 songs depending on your LLM API speed. Progress is saved after every batch — you can stop and resume at any time without losing work. The indexer prints a live progress percentage so you can track how far along it is.

Add `--verbose` to see per-batch logs and a full stats breakdown at the end.

### 5. Start the server

```bash
bash start.sh
```

Server runs at `http://localhost:5678` (or the port in your `.env`).

---

## Customizing playlist rules

Open `MUSIC_RULES.md` and edit the two sections:

- **Prompt Interpretation** — how the LLM maps a free-form request to music tags (genre counts, long-tail rules, language/energy/era handling)
- **Final Playlist Curation** — how the LLM selects and orders the final tracks (diversity, energy transitions, artist limits)

Changes take effect on next server start (no code changes needed).

---

## Usage

**Web UI:** Open `http://localhost:5678` — search your library, click to play.

**Playlist from natural language:**
```
POST http://localhost:5678/api/generate
{"prompt": "melancholic shoegaze for a rainy afternoon", "max_count": 30}
```

**Full API reference:** See `TOOLS.md`.

---

## File overview

| File | Purpose |
|---|---|
| `playlist_server.py` | HTTP server — all API endpoints and web UI |
| `smart_indexer.py` | One-time LLM enrichment of your music library |
| `MUSIC_RULES.md` | Editable LLM rules for interpretation and curation |
| `TOOLS.md` | Full API reference |
| `.env.example` | Environment variable template |
| `start.sh` | Start the server |
