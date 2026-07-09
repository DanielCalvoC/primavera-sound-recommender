# 🎵 Primavera Sound 2026 — Artist Recommender

A data-driven artist recommendation system for Primavera Sound 2026. Given an artist from the lineup, the system recommends similar artists playing the same festival, ranked by festival day and recommendation strength.

**[Live Demo](https://username.github.io/primavera-recommender/)** | Search any artist in the Primavera lineup to discover recommendations.

---

## How It Works

### The Pipeline

```
Phase 1: Spotify Enrichment
┌──────────────────────────────────────────┐
│ Input: primavera_sound_2026.csv          │
│ (artist, date, start_time, end_time)     │
│                                          │
│ → Query Spotify for each artist:         │
│   • Followers, popularity, genres        │
│   • Image URL                            │
│   • Top track + preview URL (iTunes)     │
│                                          │
│ Output: Intermediate CSV                 │
└──────────────────────────────────────────┘

Phase 2: Manual Corrections (Optional)
┌──────────────────────────────────────────┐
│ Review mismatched Spotify results        │
│ (e.g., "DJ XYZ" matched to wrong artist) │
│                                          │
│ → Correct URIs, re-link to right profile │
└──────────────────────────────────────────┘

Phase 3: Last.fm Enrichment
┌──────────────────────────────────────────┐
│ For each artist, query Last.fm:          │
│ • Similar artists (top 100 by score)     │
│ • Community tags (normalized)            │
│ • Listener stats & playcounts            │
│                                          │
│ Output: primavera_sound_2026_full.csv    │
└──────────────────────────────────────────┘

Static Frontend
┌──────────────────────────────────────────┐
│ Vanilla JS (no backend)                  │
│ • Upload CSV or fetch from repo          │
│ • Search artist → get recommendations    │
│ • Sort by day + strength                 │
│ • Play 30s preview from iTunes           │
└──────────────────────────────────────────┘
```

---

## Why This Approach

### Last.fm as the Similarity Engine

Spotify's `related-artists` and `recommendations` endpoints are **deprecated or unreliable** for non-mainstream artists (as of 2024–2025). 

Last.fm's `artist.getSimilar` is better because:
- Uses **real listening behavior** from 20+ years of user data
- Name-based matching (no Spotify ID dependency)
- Works well for electronic, underground, and regional acts
- Returns a **scored list** of similar artists (not just "related")

**Result:** Even obscure producers have rich similarity data. A niche techno artist will have Last.fm similars; Spotify might return nothing.

### iTunes URLs for Previews

Why not Spotify or Deezer?
- **Spotify:** Preview endpoint was deprecated in 2024
- **Deezer:** URLs have **expiring tokens** (die in hours) — can't host them statically
- **iTunes:** CDN URLs are **permanent** → works forever on GitHub Pages

Preview chain: iTunes → Deezer (backup) → skip

### No Backend

The recommendation logic runs entirely in the browser:
- CSV is uploaded or fetched once
- All queries happen client-side
- No server = no costs, no maintenance
- Perfect for portfolio: code is visible, demo is live

---

## Data Coverage

| Metric | Value | Notes |
|--------|-------|-------|
| Total artists | 150 | Unique lineup entries |
| Spotify found | 147 (98%) | Via artist name search |
| Last.fm similar | 130 (87%) | Artists with ≥1 similar match |
| iTunes previews | ~110 | Available + playable |

**Why gaps?**
- Spotify: New/unsigned artists, ambiguous names (corrected in Phase 2)
- Last.fm: Lower coverage for experimental/ambient (but still richer than Spotify)
- iTunes: Some artists have no recordings on iTunes (rare)

---

## The Recommendation Engine

### Layer 1: Direct Similars (High Precision)

Does the artist you search for appear in other artists' "similar" lists?

```
Search: "Viagra Boys"
↓
For each artist in lineup:
  If Viagra Boys is in their "similar artists" list:
    → Add to results

Score = Last.fm similarity score × (1 / position in their similar list)
        e.g., if Viagra Boys is 2nd in their list with score 0.85:
        → 0.85 × (1/2) = 0.425
```

**Why position matters:** If an artist lists you at position 1, that's stronger than position 50.

### Layer 2: Shared Tags (Wider Net)

Do two artists share Last.fm tags?

```
Searched artist: Viagra Boys
Tags: post-punk, experimental, swedish, art punk, indie rock

For each other artist in lineup:
  Count shared tags
  If ≥ 2 (configurable):
    → Add to results

Score = (shared tags / artist's total tags) × 0.5 × tag count
        The × 0.5 ensures Layer 2 never beats Layer 1
```

### Results Sorting

1. Group by festival day
2. Sort within each day by recommendation strength (descending)
3. Cap at N results per day (default: 10)

---

## Features

### Search + Autocomplete

Type an artist name → suggestions appear from the lineup.
- Arrow keys to navigate
- Enter or click to search
- Case-insensitive, partial matching

### Recommendation Cards

Each recommended artist shows:

```
┌─────────────────────────────┐
│ [Image]                     │
│                             │
│ Artist Name                 │
│ similares | shared tags     │  ← Source badge
│ [=====>     ] 42%           │  ← Strength bar
│                             │
│ Popularity: [====>  ] 68    │  ← Spotify metric
│                             │
│ [Hover to play preview]     │  ← 30s iTunes track
│ ↗ Open on Spotify           │  ← Direct link
└─────────────────────────────┘
```

**Color coding:**
- 🟢 `similares` (yellow): Layer 1 — direct match
- 🔴 `tags` (coral): Layer 2 — tag-based match

### Searched Artist Card

At the top of results, shows the artist you searched for in detail:

```
┌──────────────────────────────────────┐
│ [Image] SEARCHING FOR:               │
│         Artist Name                  │
│         Mié 24 may · 23:00 · Auditori│
│                                      │
│         Tags: post-punk | indie rock │
│         123k followers · Popularity  │
│                                      │
│         [Play] Top Track (30s)       │
│         ↗ Open on Spotify            │
└──────────────────────────────────────┘
```

---

## Setup

### Prerequisites

For enrichment (Google Colab):
- Google Drive access
- [Spotify Developer account](https://developer.spotify.com/dashboard) → Client ID + Secret
- [Last.fm account](https://www.last.fm/api/accounts) → API Key

### Data Enrichment

The three notebooks run in **Google Colab**. Each takes ~1–2 hours for 150 artists.

**Phase 1 — Spotify:**
```
Input:  primavera_sound_2026.csv
        (artist | date | start_time | end_time | stage)

Process:
  For each artist:
    Search Spotify
    Get profile (followers, genres, image)
    Get top track + preview URL
    Save checkpoint every 10 artists

Output: primavera_sound_2026_phase1.csv
        (+ spotify_id, spotify_url, popularity, image_url, preview_url, etc.)
```

**Phase 2 — Manual Corrections (optional):**
```
Review mismatch report:
  • Which artists had different Spotify names?
  • Any wrong matches?

Correct via CSV or Spotify URI if needed.
Usually quick: ~10–15 min for 150 artists.
```

**Phase 3 — Last.fm:**
```
Input:  Phase 2 output (or Phase 1 if skipping Phase 2)

Process:
  For each artist:
    Query artist.getSimilar (top 100)
    Query artist.getInfo (tags, listeners, playcount)
    Normalize tags (hip-hop → hip hop, etc.)
    Save checkpoint every 20 artists

Output: primavera_sound_2026_full.csv
        (+ lastfm_similar, lastfm_similar_scores, lastfm_tags, etc.)
```

### Local Testing

```bash
# Clone repo
git clone https://github.com/username/primavera-recommender.git
cd primavera-recommender

# Serve locally (required for fetch() to work)
python -m http.server 8000

# Open browser
# http://localhost:8000/index.html
```

### Deploy to GitHub Pages

1. Push `index.html` and `data/primavera_sound_2026_full.csv` to repo
2. Go to repo Settings → Pages → enable GitHub Pages
3. Choose branch (usually `main` or `gh-pages`)
4. Live at: `https://username.github.io/primavera-recommender/`

**Note:** `fetch()` fails with `file://` protocol (browser security). Must use HTTP server or GitHub Pages.

---

## Configuration

Two parameters in the app (top of results area):

| Parameter | Default | What it does |
|-----------|---------|--------------|
| Tags mínimos compartidos | 2 | Minimum tags for Layer 2 to match |
| Máx. por día | 10 | Max recommendations shown per festival day |

Raise "tags mínimos" for stricter results (fewer false positives).
Lower "máx. por día" for top-N only.

---

## Under the Hood

### Recommendation Strength Calculation

```javascript
// Layer 1: Direct Similars
if (artist appears in candidate's similar list):
  position = index in list (0-based)
  lastfm_score = similarity score (0–1)
  strength = lastfm_score * (1 / (position + 1)) + 1

// Layer 2: Shared Tags
if (shared tags ≥ min_shared_tags):
  tag_score = shared_tags / artist_total_tags
  strength = tag_score * 0.5 * shared_tag_count

// Result: Layer 1 always ranked above Layer 2
// (because Layer 1 starts at strength ≥ 1, Layer 2 max is 0.5×5 = 2.5)
```

### CSV Format

```
artist,date,start_time,end_time,stage,spotify_id,spotify_url,followers,popularity,genres,image_url,spotify_artist_name,top_track,preview_url,lastfm_similar,lastfm_similar_scores,lastfm_tags,lastfm_listeners,lastfm_playcount
```

Pipe-separated lists (e.g., tags):
```
lastfm_tags: "post-punk | art punk | indie rock | swedish"
lastfm_similar: "Squid | Shame | IDLES | Fontaines D.C."
lastfm_similar_scores: "0.85 | 0.79 | 0.76 | 0.72"
```

### Audio Preview Logic

Hover over image → 400ms delay → play 30s preview
Click anywhere else → stop playback
Wave animation shows while playing
Progress bar tracks position

---

## Example: Searching "Viagra Boys"

1. **Type "Viagra Boys"** → autocomplete suggests it
2. **Press Enter**
3. **Layer 1 finds:**
   - Squid (Viagra Boys is in their similar list, position 3, score 0.87)
   - Shame (position 7, score 0.75)
4. **Layer 2 finds:**
   - Fontaines D.C. (shared 4 tags: post-punk, art punk, swedish, indie rock)
   - Black Midi (shared 3 tags: experimental, art rock, uk)
5. **Results sorted by day, then by strength**
6. **Each card shows:** day/stage, recommendation source, strength bar, preview

---

## Project Status

- ✅ **Pipeline complete** (all 3 phases, Phase 2 corrections applied)
- ✅ **Frontend live** (search, preview, schedule display)
- ✅ **GitHub Pages deployed** (live demo)
- 🔄 **Future:** Similarity graph tab (D3.js force-directed)

---

## Tech Stack

| Layer | Tools |
|-------|-------|
| **Data** | Python 3.10+, pandas, spotipy, requests |
| **APIs** | Spotify Web API (Client Credentials), Last.fm (key-only), iTunes Search (public) |
| **Frontend** | HTML5, vanilla JavaScript (ES6), no frameworks |
| **Hosting** | GitHub Pages (static, free) |
| **Dev Environment** | Google Colab (notebooks), local testing (http.server) |

---

## Decision Log

### Why `SIMILAR_N=100` (not 5)?

Primavera has 150 artists. With `SIMILAR_N=5`, you only see the Top 5 similars per artist — many lineup intersections get missed.

With `SIMILAR_N=100`:
- More overlap in the lineup
- Better chance of finding indirect connections
- Still manageable API load (0.4s/request = ~1 min for 150 artists)

### Why Tag Normalization?

Last.fm tags are user-generated, so duplicates exist:
- `hip hop` vs `hip-hop`
- `post rock` vs `post-rock`

During Phase 3, we run a `TAG_SYNONYMS` dict to canonicalize:
```python
TAG_SYNONYMS = {
    'hip-hop': 'hip hop',
    'post-rock': 'post rock',
    'electronic': 'electronica',
    ...
}
```

Result: cleaner tags, better Layer 2 matching.

### Why is Layer 2 Multiplied by 0.5?

If Layer 2 (tags) could outrank Layer 1 (direct similars), you'd get weird results:
- "Both have tag 'electronic'" shouldn't beat "Last.fm says they're directly similar"

The `× 0.5` ensures:
- Layer 1 max: 1.0 + 0.87 (score × position) = ~1.9
- Layer 2 max: 0.5 × 5 (share all 5 tags) = 2.5

So they can be close, but Layer 1 has priority in practice (usually higher scores).

### Why Not Include Location Tags in Similarity?

Last.fm tags include locations: `spain`, `spanish`, `uk`, `french`.

These create false positives: Two acts from Spain sound nothing alike, but they'd "share tags."

**Solution:** Keep location tags in CSV (useful for regional discovery) but filter them during Layer 2 matching.

---

## File Structure

```
.
├── README.md                        (this file)
├── CHANGELOG.md                     (version history)
├── .gitignore
├── index.html                       (app frontend)
├── data/
│   └── primavera_sound_2026_full.csv
└── notebooks/
    ├── enrichment_phase1_spotify.ipynb
    ├── enrichment_phase2_corrections.ipynb
    └── enrichment_phase3_lastfm_itunes.ipynb
```

---

## Common Issues

### "CSV loads but search returns no results"

- **Cause:** Column names don't match (`artist` vs `Artist`?)
- **Fix:** Check CSV headers match what the frontend expects

### "Preview URLs don't play"

- **Cause:** iTunes URLs expired or blocked by CORS
- **Fix:** Re-run Phase 3 to fetch fresh URLs

### "Autocomplete empty"

- **Cause:** CSV not loaded or artist names are blank
- **Fix:** Upload CSV again, check file format

### "Fetch error on localhost"

- **Cause:** Running as `file://` protocol
- **Fix:** Use `python -m http.server 8000`, visit `http://localhost:8000`

---

## Portfolio Value

This project demonstrates:

- **Data Pipeline:** Multi-stage enrichment, API integration (Spotify + Last.fm), error handling, checkpoints/resume
- **API Strategy:** Choosing the right data source (why Last.fm > Spotify for this use case), fallback chains (iTunes → Deezer)
- **Frontend:** Vanilla JS, CSV parsing, state management, audio playback, UX polish
- **Product Thinking:** Trade-offs documented, decisions justified, user-facing polish

**Relevant for roles:** Analytics Engineer, Data Scientist, Full-stack Engineer, ML Engineer

---

## Next Steps

1. Replace `username` with your GitHub handle in live demo link
2. Push to GitHub + enable Pages
3. Optional: add screenshot of the UI to README
4. Optional: create CHANGELOG.md with version history

---

**Last updated:** July 2026  
**Status:** Live & ready for Primavera Sound 2026 (May 24–26)
