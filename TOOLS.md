# PlaylistGen — API Reference

Base URL: `http://localhost:5678` (or whatever `PORT` and `MUSIC_SERVER_URL` are set to).

**Never construct player URLs manually. Always use the `url` field returned by the server.**

---

## Playlist generation

### Fully automated (recommended)

One call — interpret → fetch → curate → save:

```
POST /api/generate
{"prompt": "obscure 80s synth for late night driving", "max_count": 30}

→ {"title": "🎵 ...", "url": "http://<host>:5678/player?saved=XXXX", "count": 28, "tags": {...}}
```

Share only the returned `url`. Do not modify it.

---

### Manual workflow (agent-participates in interpret and/or curate)

Four steps. You can hook into Step 1 and/or Step 3.

#### Step 1 — Interpret prompt → structured tags

**Option A — let the server interpret (MiniMax M2.7, guided by MUSIC_RULES.md):**
```
GET /api/interpret?q=<prompt>
→ {
    "genres": [...],        "subgenres": [...],
    "moods": [...],         "usage_contexts": [...],
    "year": [start, end],   "energy": [...],
    "language": "...",      "regions": [...],
    "popularity_hint": "..."
  }
```

**Option B — construct tags yourself:**

First fetch the catalog vocabulary to know what tags exist:
```
GET /api/catalog/vocab
→ {"genres": {"Rock": 9123, ...}, "subgenres": {...}, "moods": {...}, ...}
```

Tags with count ≤ 2 are long-tail (niche/rare). Selection rules (from MUSIC_RULES.md):
- `genres` — 2 values (regular only)
- `subgenres` — 3 regular + 1 long-tail
- `moods` — 2 regular + 1 long-tail
- `usage_contexts` — 2 regular + 1 long-tail
- `year` — `[start_year, end_year]` if era implied, else `[]`
- `energy` — list of Energy values if implied, else `[]`
- `language` — single Languages value if language-specific, else `""`
- `regions` — list of Regions values if geography implied, else `[]`
- `popularity_hint` — `"mainstream"`, `"popular"`, `"indie"`, `"niche"`, `"obscure"`, or `"any"`

#### Step 2 — Fetch candidates

Language is a **hard filter**. Region and popularity are NOT filtered here — pass them to Step 3.

```
GET /api/songs?genre=Rock,Indie&subgenre=Shoegaze,Dream+Pop&mood=Melancholic,Dreamy&usage_context=night&language=&limit=300
→ {"songs": [...], "count": N}
```

All params are substring-matched. Multiple values are comma-separated. Returns up to `limit` songs (default 300).

Additional params: `mode=or|and|weighted` (default `or`), `artist_limit=5` (max songs per artist in pool).

#### Step 3 — Curate: select and order final playlist

**Option A — let the server curate (LLM, guided by MUSIC_RULES.md):**
```
POST /api/curate
{
  "songs": [...],           // from Step 2
  "prompt": "...",          // original user prompt
  "max_count": 30,
  "language": "...",        // hard requirement if non-English
  "popularity_hint": "...", // soft hint
  "regions": [...]          // soft hint
}
→ {"curated": ["path1", "path2", ...], "count": N}
```

**Option B — curate yourself:**
Each song has: `path`, `artist`, `title`, `genre`, `subgenre`, `mood`, `energy`, `language`, `region`, `year`, `popularity`. Select and order them, then pass paths to Step 4.

#### Step 4 — Save and get player URL

```
POST /api/playlist/save
{"title": "🎵 My Playlist", "songs": ["path1", "path2", ...]}
→ {"url": "http://<host>:5678/player?saved=XXXX", "title": "...", "count": N}
```

---

## Other endpoints

| Endpoint | Description |
|---|---|
| `GET /api/stats` | Library stats: total songs, top artists/albums |
| `GET /api/search?q=<text>` | Keyword search on title/artist/album (50 results) |
| `GET /api/interpret?q=<prompt>&provider=minimax\|claude` | LLM prompt → structured tags |
| `GET /api/catalog/vocab` | Full tag→count dict for all fields |
| `GET /api/catalog/vocab/refresh` | Rebuild vocab from DB, rewrite catalog JSON file |
| `GET /` | Web search UI |
| `GET /player?saved=<key>` | Playlist player page |

---

## Song fields

| Field | Type | Notes |
|---|---|---|
| `path` | string | Absolute filesystem path — use as song ID in all API calls |
| `url` | string | Relative path — use as audio `src` in browser |
| `title` | string | |
| `artist` | string | |
| `album` | string | |
| `year` | string | |
| `genre` | string | e.g. `"Rock"`, `"Electronic"` |
| `subgenre` | string | e.g. `"Shoegaze"`, `"Synthpop"` |
| `mood` | string | Comma-separated, e.g. `"Melancholic, Dreamy"` |
| `usage_context` | string | e.g. `"night"`, `"driving"` |
| `energy` | string | `"low"`, `"medium"`, or `"high"` |
| `language` | string | e.g. `"English"`, `"Mandarin"` |
| `region` | string | e.g. `"UK"`, `"USA"`, `"Taiwan"` |
| `popularity` | string | `"mainstream"`, `"indie"`, `"niche"`, `"obscure"` |
| `duration` | float | Seconds |
