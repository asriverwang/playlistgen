# PlaylistGen — API Reference

Base URL: `http://localhost:5678` (or whatever `PORT` and `MUSIC_SERVER_URL` are set to).

**Never construct player URLs manually. Always use the `url` field returned by the server.**

---

## Generating a playlist — try automated first, fall back to manual

**Always try the automated endpoint first:**

```
POST /api/generate
{"prompt": "obscure 80s synth for late night driving", "max_count": 30}

→ {"title": "🎵 ...", "url": "http://<host>:5678/player?saved=XXXX", "count": 28, "tags": {...}}
```

Share only the returned `url`. Do not modify it. One call handles everything (interpret → fetch → curate → save).

**If `/api/generate` returns `{"error": ...}`, switch to the manual workflow below.**

---

## Manual workflow (fallback)

Use this only when `/api/generate` fails. You participate in the pipeline directly.

**Step 1 — Interpret prompt → structured tags**

Fetch valid tags first: `GET /api/catalog/vocab`

Pick tags following these field rules:
- `genres` — 2 values
- `subgenres` — 3 regular + 1 long-tail (count ≤ 2)
- `moods` — 2 regular + 1 long-tail
- `usage_contexts` — 2 regular + 1 long-tail
- `year` — `[start_year, end_year]` if era implied, else `[]`
- `energy` — `["high"]`, `["low", "medium"]`, etc., or `[]`
- `language` — single value if language-specific, else `""`
- `regions` — list if geography implied, else `[]`
- `popularity_hint` — `"mainstream"`, `"indie"`, `"niche"`, `"obscure"`, or `"any"`

**Step 2 — Fetch candidates**

Language is a hard filter. Region and popularity are NOT filtered here — pass them to Step 3.
```
GET /api/songs?genre=Rock,Indie&subgenre=Shoegaze,Dream+Pop&mood=Melancholic,Dreamy&usage_context=night&language=&limit=300
→ {"songs": [...], "count": N}
```
All params are substring-matched, comma-separated for multiple values. Returns up to 300 songs with full metadata.

**Step 3 — Curate: select and order final playlist**

Each song has: `path` (use as ID), `artist`, `title`, `genre`, `subgenre`, `mood`, `energy`, `language`, `region`, `year`, `popularity`. Select and order them yourself, then pass the paths to Step 4.

**Step 4 — Save and get player URL**
```
POST /api/playlist/save
{"title": "🎵 My Playlist", "songs": ["path1", "path2", ...]}
→ {"url": "http://<host>:5678/player?saved=XXXX", "title": "...", "count": N}
```

Share only the returned `url`. Do not modify it.

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
