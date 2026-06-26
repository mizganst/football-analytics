# Football Analytics · From Raw Web Data to Pitch Visuals

A set of Python notebooks that build a full data pipeline, scraping live match event data from the web, parsing and cleaning the raw Opta event feed, engineering football-specific metrics from scratch, and rendering publication-quality pitch visualisations.
---

## Why This Exists

Most football analytics tutorials hand you a tidy CSV and skip to the charts. That's fine for learning plots, but it hides the real work — and the real understanding. The interesting questions happen earlier:

- How do you get the data in the first place when the site uses JavaScript rendering?
- How do you turn a stream of timestamped events into meaningful player-level actions?
- How do you engineer a metric like *carry speed* or *off-ball run detection* when those things aren't logged anywhere — only their preconditions are?
- How do you lay out a visual that actually communicates something to someone who watches football, not just someone who reads charts?

These notebooks try to answer all of that, end to end, for real players in real competitions.

---

## The Data Source

All event data is scraped from **WhoScored**, which serves Opta's event-level feed under the hood. Each match record is a JSON object containing every on-ball action in the game — passes, shots, take-ons, tackles, and more — tagged with coordinates (Opta's 0→100 × 0→100 pitch grid), timestamps down to the second, and a rich qualifier system that encodes things like *was this a cross?*, *which foot?*, *was this from a corner?*

Selenium + webdriver-manager handles the dynamic page rendering. The scraper collects data across a player's full season by iterating a list of match IDs, then consolidates everything into a single event pool before any analysis begins.

---

## The Pipeline

Every notebook follows the same four-stage architecture:

```
1. SCRAPE      →  Selenium renders the WhoScored match page; raw JSON is extracted from the page state
2. PARSE       →  Event type IDs, qualifier codes, and outcome flags are decoded into meaningful filters
3. ENGINEER    →  Player-specific metrics are derived from the event stream (see below)
4. VISUALISE   →  mplsoccer + matplotlib render pitch maps with a consistent visual identity
```

### Feature Engineering Highlights

**Carry reconstruction** — Opta doesn't log ball carries as explicit events. They're inferred by detecting consecutive possession events by the same player where the ball position jumps, the time gap is plausible (< 12 s), and the spatial gap falls within carry distance bounds (2.5–35 Opta units). Speed is computed from distance ÷ time, then mapped to a colour scale.

**Expected Threat (xT)** — Implements Karun Singh's 16 × 12 pitch grid, assigning a probability value to every zone based on how likely possession from there leads to a goal. xT gained per pass or carry is computed as `xT(end_zone) − xT(start_zone)`. Zone totals are accumulated across a season and rendered as a heatmap on the pitch.

**Open-play filtering** — Set-piece actions (corners, free kicks, throw-ins, goal kicks) are stripped from pass and chance-creation maps using Opta's qualifier codes, so the visuals reflect true open-play patterns rather than dead-ball volume.

**Off-ball run detection** — Identifies moments when a player receives the ball in the final third after a gap since their previous possession event, suggesting a run into space rather than a stationary receipt.

**Post-action chaining** — Determines whether a dribble or carry led to a shot within a fixed window (3 team actions / 15 seconds), flagging the most dangerous sequences on the pitch.

**Pre-action movement distance** — For midfielders, estimates how far a player travelled before receiving the ball or making a defensive action, giving a proxy for off-ball engine work.

---

## Notebooks

### `Single_forward_analysis_with_chance_creation.ipynb`
A full solo analysis of one forward across a season. The example subject is **Yan Diomandé (RB Leipzig, 2025/26)**. Produces 11 visuals covering every dimension of an attacking player's contribution:

| # | Visual | What it shows |
|---|--------|---------------|
| 01 | Touch map | KDE density heatmap of all touches |
| 02 | Off-ball runs | Runs into the final third with arrows |
| 03 | Chance creation | Key passes + goal assists, open play only |
| 04 | Take-on map | Successful vs failed dribbles |
| 05 | Post-dribble actions | Pass arrows after a take-on, coloured by xT gained |
| 06 | Progressive carries | Carries ≥10 units forward, shaded by speed |
| 07 | xT map | Net xT generated per zone via passes and crosses |
| 08 | Passing connections | Pass arrows ending in or beyond the final third |
| 09 | Foot split | Right vs left foot percentage breakdown |
| 10 | Crosses into box | Cross origins and destinations in the penalty area |
| 11 | Summary radar | Single-page stat block with percentile radar |

---

### `Two_Forward_Comparison.ipynb`
Side-by-side comparison of two forwards, laid out on mirrored pitches for direct visual contrast. The example pairing is **Victor Muñoz (Osasuna, 2025/26) vs Luis Díaz (Liverpool, 2024/25)** — a deliberate cross-league, cross-season comparison to stress-test the pipeline across different data prefixes. Covers the same range of metrics as the solo notebook minus the summary radar, with both players rendered on every plot.

---

### `Three-midfielder comparison.ipynb`
A three-way comparison built around **central midfielders** rather than forwards, so the metric set shifts accordingly. Subject trio: **Ayyoub Bouaddi (Lille)**, **Lamine Camaraet (Monaco)**, and **Mamadou Sangaré (Lens)** — all 2025/26 Ligue 1. The layout moves to a 2-above-1 pitch format to accommodate three panels per visual cleanly. Additional metrics not in the forward notebooks:

- Pass map with final-third / box-pass distinction (arrow thickness encodes destination zone)
- Fast pass map — passes played within 2 seconds of receiving the ball
- Ball recovery map — tackles, interceptions, and loose-ball recoveries with action-type encoding
- Post-recovery pass map — where the ball goes within 3 seconds of winning it back
- Pre-receive movement distance and pre-defensive action distance

---

## Tech Stack

| Tool | Role |
|------|------|
| `selenium` + `webdriver-manager` | Dynamic page scraping |
| `json`, `re`, `pathlib` | Raw data extraction and I/O |
| `numpy` | Array-level metric computation |
| `matplotlib` | Figure construction and custom styling |
| `mplsoccer` | Pitch drawing (Opta coordinate system) |
| `scipy` | KDE for touch and heatmap density |

---

## Running the Notebooks

1. Install dependencies: `pip install mplsoccer matplotlib pandas numpy selenium webdriver-manager scipy`
2. Open the notebook you want in Jupyter or VS Code
3. Edit the `PLAYER` / `PLAYERS` config block at the top — drop in your own player ID(s) and match ID list from WhoScored
4. Run all cells; outputs are written to a local folder named after the subject(s)

Match IDs live in WhoScored URLs: `https://www.whoscored.com/Matches/XXXXXXX/Live` — the number after `/Matches/` is the full ID. The notebooks split these into a shared prefix (same for all matches in a competition-season) and per-match suffixes to keep the config compact.

---

## Notes on the Data

WhoScored's terms of service restrict automated scraping. These notebooks are personal research tools shared for educational purposes. If you use them, be respectful — add delays between requests, don't hammer the server, and don't use this for commercial purposes.

The Opta coordinate system runs from 0 to 100 on both axes (left goal to right goal on x, bottom to top on y). `mplsoccer`'s `pitch_type='opta'` maps these natively, so no coordinate flipping is needed.

xT grid values are Karun Singh's published 16×12 grid, reproduced here for research use.

---

*Data: Opta via WhoScored · Visuals produced with mplsoccer and matplotlib*
