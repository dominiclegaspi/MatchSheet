 https://matchsheet.vercel.app

# Matchsheet — Project Notes

An NBA player similarity finder: type any current player, get their 10
closest statistical matches, explore the whole league in an interactive 3D
stat-space map, and see exactly why any two players were flagged as similar.

---

## 1. What it does

- **Match Finder** — search any of 379 qualified NBA players (500+ minutes
  played, 2025–26 season), get a ranked top-10 list of statistical twins
  with a similarity score, percentile, and rarity tier ("Elite statistical
  match," "Strong comparison," etc.)
- **Vector Space** — a 3D point cloud of every qualified player, built from
  the same math that powers the matching (not a separate approximation).
  Search a player to fly the camera to their neighborhood and see labeled
  nearby comps.
- **Per-match detail** — expand any result to see a full stat comparison
  (all 15+ underlying stats, with real deltas) and a small vector diagram
  showing the two players' relative position in reduced stat-space.

---

## 2. Data sources

1. **User-provided CSV** (Kaggle: `nba-player-stats-2026`) — season box
   score totals: games played, minutes, FG/3P/FT makes & attempts,
   rebounds (split O/D), assists, steals, blocks, turnovers, fouls, points,
   and the NBA's built-in EFF rating.
2. **Basketball-Reference advanced stats** (fetched and merged in) — PER,
   BPM, USG%, AST%, TRB%, TOV%, 3PA rate, FTA rate, WS/48. These genuinely
   cannot be derived from box-score totals alone; they require team pace
   and lineup/opponent data the CSV doesn't contain. All 379 players were
   matched by name across both sources (with manual corrections for ~10
   naming mismatches — accented characters, suffixes, nicknames).

## 3. Data pipeline

```
Raw CSV + Basketball-Reference
        ↓
Filter to players with 500+ minutes played  (379 of ~580 total)
        ↓
Derive per-game rates (PPG, RPG, APG, SPG, BPG, MPG) from season totals
        ↓
Merge in real advanced metrics by player name
        ↓
Z-score normalize all 15 features across the qualified player pool
        ↓
PCA: reduce 15 → 8 components (retains 95.1% of real variance)
        ↓
Export player vectors + precomputed percentile lookup table as JSON,
embedded directly in the React component
```

Everything downstream (matching, scoring, the 3D map) runs client-side in
the browser — no backend, no API calls at runtime. The Python
preprocessing (pandas/numpy, PCA via SVD) is a one-time offline step; its
output is a static JSON blob baked into the component.

## 4. The 15 features

Deliberately chosen to avoid redundancy — e.g. not using `FG%` + `eFG%` +
`TS%` together, since `TS%` alone already captures shooting efficiency.

| Category | Stats |
|---|---|
| Volume | PPG, RPG, APG, SPG, BPG |
| Efficiency | TS% |
| Role / shot profile | USG%, 3PA Rate, FTA Rate, MIN |
| Advanced (real, from Basketball-Reference) | AST%, REB%, TOV%, PER, BPM, WS/48 |

## 5. The similarity algorithm

Full math derivation lives in a companion doc (`ALGORITHM.md` /
`matchsheet-algorithm-notes.md`); short version:

1. **Z-score** every stat: `z = (x - μ) / σ`, so no stat dominates by raw
   scale (PPG ≈ 20 vs. BPM ≈ 2 vs. TS% ≈ 0.58).
2. **PCA, truncated to 8 components** (retaining 95.1% of variance) —
   *not* whitened. This is the single most important design decision in
   the project: reducing dimensionality collapses correlated stats
   (PPG/USG%/PER all move together) into shared components instead of
   each independently inflating similarity. Whitening (rescaling every
   component to unit variance) was tried and empirically rejected — it
   amplifies low-variance noise directions and produced worse matches
   (see §7).
3. **Cosine similarity** on the reduced vectors — compares the *shape* of
   two statistical profiles independent of overall production level.
   `Score = cosine_similarity × 100`.
4. **Percentile**, computed against the real empirical distribution of all
   ~71,000 pairwise comparisons in the dataset — not a fixed 0–100 scale
   or a parametric assumption. This is also what drives the tier labels,
   which adapt automatically to whatever player pool is loaded.

## 6. UI / design

- **Visual identity**: a "box score / stat sheet" aesthetic — hardwood
  amber accent, ink-navy background, condensed athletic display type
  (Oswald) for names, monospace (JetBrains Mono) for all numeric data to
  keep digits tabular and aligned, Inter for body text.
- **Match Finder**: search-driven, autocomplete dropdown, anchor player
  card with headline stats, ranked result list with an animated gauge per
  match, expandable detail rows.
- **Vector Space**: three.js scene, manual orbit/zoom controls (three.js's
  OrbitControls isn't available in this environment, so drag-rotate and
  scroll-zoom are hand-implemented), camera "flies" to a searched player's
  neighborhood, nearest-neighbor labels are DOM elements projected onto
  the 3D scene each frame.
- **Reliability signaling**: stat deltas only get colored (green/red) when
  the difference clears 25% of that stat's real league-wide standard
  deviation — otherwise shown in neutral gray. Prevents implying a
  meaningful gap where the numbers are basically noise.

## 7. Iteration history — what was tried, and why it changed

This project went through a large number of algorithm revisions. Keeping
this history because the *reasoning* is more valuable than the current
snapshot alone.

| Version | Approach | Outcome |
|---|---|---|
| v1 | Per-36 stats, manually weighted Euclidean, exponential decay score | Reasonable start, but scores clustered near the top regardless of match quality |
| v2–v5 | Various score-recalibration attempts (percentile rescaling, neighborhood-relative z-scores) | Fixed display issues but introduced a real bug: any player's *rank-1* match got inflated to ~99% regardless of true similarity, because being "closest of 434 candidates" is mathematically always an extreme percentile |
| v6–v7 | Expanded to 19 then 14 features, added minutes-based shrinkage, added full covariance whitening + cosine/Euclidean blend | Whitening stripped out *real* correlation (the kind that defines an actual player archetype), producing technically-defensible but basketball-nonsensical matches (obscure bench players dominating "closest pairs," stars separated from their real comps) |
| v8 | Diagnosed the whitening problem; reverted to plain weighted Euclidean, no whitening, no cosine | Validated well — recognizable comps (LeBron→Harden, Curry→Ant/Mitchell) |
| v9 | User-specified 15-feature list, cosine + Euclidean hybrid (70/30 blend), min-max score rescaling, 500-min cutoff, real Basketball-Reference PER/BPM/WS-48/USG% merged in | Validated well; more rigorous but more complex |
| v10 | Replaced manual weights with PCA-derived ones (still blending cosine + Euclidean) | Validated well; components turned out interpretable (PC1 ≈ overall production, PC2 ≈ playmaking vs. rebounding role, PC3 ≈ ball security vs. shot volume) |
| v11 | Simplified to pure cosine similarity on the PCA-reduced space, dropped the Euclidean blend entirely | Validated well — simpler, one clean formula, same match quality |
| v12 | (Temporary revert, per request) plain weighted Euclidean, manual weights, no cosine, no PCA | Validated well too — confirms multiple reasonable algorithms converge on similar real-world answers |
| **Current** | **Back to v11: PCA (K=8, no whitening) + pure cosine similarity** | **Re-tested "proper" PCA whitening (reduce-then-whiten) on explicit request — it reintroduced the exact same failure mode (Powell/Edwards reunited, Jokić/Giannis separated). Confirmed non-whitened PCA + cosine as the best-performing version across every validation check run in this project.** |

**Validation method used throughout**: query recognizable stars (LeBron,
Curry, Giannis, Jokić, common role-player pairs) and check whether the top
matches pass a basketball fan's sniff test, before shipping any algorithm
change. This caught real regressions multiple times that looked fine on
paper but produced bad results in practice.

## 8. Known limitations

- **Single season only.** No historical cross-season comps (e.g. "who is
  this rookie's closest match from the last 15 years").
- **No shot-location or play-type data.** Two "high-volume 3-point
  shooters" can have very different shot diets (corner 3s vs. above-the-
  break, catch-and-shoot vs. pull-up) that this dataset can't distinguish.
- **No position or physical measurements** (height/weight) factored into
  matching — discussed but not yet implemented; position is available from
  the Basketball-Reference merge but not yet wired in, and height/weight
  would require scraping 30 individual team roster pages (no bulk source
  exists).
- **PCA components aren't hand-labeled in the UI** beyond the top 3 in
  documentation — a "why these two are similar" breakdown (which
  components/stats are driving a specific match) was proposed but not
  built into the app itself.
- **Basketball-Reference and the uploaded CSV were pulled at slightly
  different snapshot times** — minor game-count discrepancies possible for
  recently-traded players.

## 9. Possible future work

- Multi-season history for era-spanning comps
- Shot-location / play-type features if a data source becomes available
- Position filter or position-similarity weighting
- Height/weight integration (would need the 30-team roster scrape)
- In-app "why these two" component-contribution breakdown
- Move the embedded JSON out of the component into a fetched file, for
  easier updates without touching code
- Automate the data pipeline (currently a manual one-time Python step) so
  next season's stats can refresh the dataset without redoing the merge
  by hand

## 10. Tech stack

- **Frontend**: React (single-file component), three.js for the 3D view
- **Data processing**: Python (pandas, numpy) — z-scoring, PCA via SVD,
  percentile table generation, all done offline and exported as static JSON
- **Styling**: hand-written CSS (CSS custom properties for the box-score
  color palette), Google Fonts (Oswald / Inter / JetBrains Mono)
- **No backend** — fully static, all computation client-side after the
  one-time data export

## 11. Running it locally

```bash
npm create vite@latest matchsheet -- --template react
cd matchsheet
npm install three
# replace src/App.jsx with MatchSheet.jsx
npm run dev
```
