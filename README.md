# 🎵 Music Recommender Simulation

## Project Summary

This project builds and evaluates a content-based music recommender system to understand how algorithms like Spotify's work and where they can fail. The system uses a point-based scoring algorithm (max 6.0 points) that combines categorical features (genre +2, mood +1) with numeric similarity (energy, valence, tempo, danceability, acousticness using Gaussian kernels).

**Core finding:** The system reveals a 52% fairness gap—mainstream users (pop, lofi, rock preferences) get scores of 5.49–5.88/6.0, while niche users (country, contradictory moods) get only 2.00–3.88/6.0. This demonstrates how dataset composition, not algorithmic complexity, creates filter bubbles in real-world recommenders.

---

## How The System Works

This project builds a simple, content-based music recommender that follows the two-step approach used by production systems: (1) generate candidates, and (2) rank and re-rank a slate of results for the user. The goal here is clarity and reproducibility: the system uses interpretable song attributes (genre, mood, energy, etc.) and a math-based scoring rule so you can see exactly why a song was recommended.

Core idea: score each song against a user's taste profile using a mix of categorical boosts (genre/mood) and numeric "vibe" similarity (energy, valence, tempo, danceability, acousticness). The numeric similarity is computed with smooth Gaussian kernels so the system rewards closeness to a user's target, not just larger or smaller values.

### Finalized Algorithm Recipe (point-based)
This is the exact, implementable recipe used by the recommender in this project.

1) Preprocess
   - Load `songs.csv` into memory and compute `tempo_norm` for each song using catalog min/max (clip to [0,1]).
   - Clip numeric features (energy, valence, danceability, acousticness) to [0,1].

2) Points and caps
   - Categorical caps: genre = 2.0 points (exact match), mood = 1.0 point (exact match). Partial genre match (e.g., "indie pop" vs "pop") can get 1.0 as a soft match.
   - Numeric block total = 3.0 points distributed across features: energy (1.0), valence (0.7), tempo_norm (0.5), danceability (0.5), acousticness (0.3).
   - Maximum raw points per song = 6.0 (2 + 1 + 3).

3) Per-feature numeric scoring (Gaussian closeness)
   - Use G(x, μ, σ) = exp(-0.5 * ((x - μ)/σ)^2), which returns (0,1].
   - Default σ values: sigma_energy = 0.15, sigma_valence = 0.15, sigma_tempo = 0.12 (applied to normalized tempo), sigma_dance = 0.10, sigma_acoustic = 0.20.
   - Feature points: feature_points_f = feature_max_f * G_f(song_value, user_target, σ_f).

4) Categorical points
   - genre_points: +2.0 if exact match, +1.0 for partial match (substring or token overlap), else 0.
   - mood_points: +1.0 if exact match, else 0.

5) Aggregate and normalize
   - raw_points = numeric_points + genre_points + mood_points
   - normalized_score = raw_points / 6.0

6) Ranking and slate construction
   - Score every candidate (loop over songs or pre-filtered candidates).
   - Sort by raw_points (descending).
   - Apply list-level rules: artist cap (e.g., max 2 songs per artist), exploration slots, and business filters (region/licensing).
   - Return Top-K with short explanations built from the top contributors (e.g., "Matches your favorite genre (pop) and target energy").

### Song fields
- id (int)
- title (str)
- artist (str)
- genre (str) — categorical (e.g., "pop", "lofi")
- mood (str) — categorical (e.g., "chill", "happy", "intense")
- energy (float, 0–1) — perceived intensity
- tempo_bpm (float) — tempo in beats per minute
- valence (float, 0–1) — musical positivity/happy vs sad
- danceability (float, 0–1) — suitability for dancing / groove
- acousticness (float, 0–1) — acoustic vs electronic/produced

### UserProfile fields
- favorite_genre (str)
- favorite_mood (str)
- target_energy (float, 0–1)
- likes_acoustic (bool)

### Limitations and Bias

**Key Weakness Discovered: Cold-Start Collapse with Missing Moods**

Through experimental evaluation of six user profiles, we discovered a critical weakness: **when a user requests a mood not present in the catalog, their recommendation score collapses by 55%**. Specifically, a user preferring "country" music with a "nostalgic" mood (neither present in our 10-song catalog) receives only 2.0/6.0 points instead of the baseline 4.5/6.0, forcing them into default recommendations based on pure numeric similarity. The system has no fallback mechanism to substitute related moods (e.g., treating "nostalgic" as similar to "chill" or "relaxed"), making it completely unable to serve users with niche preferences. This reveals a hidden assumption in the design: the 10-song catalog must represent all user preferences, which catastrophically fails for the 5–10% of users with non-mainstream tastes.

### Expected biases and limitations
- Genre dominance: Because genre match gives a large, interpretable boost (+2 points), the system may over-prioritize genre and under-represent songs that fit a user's numeric vibe but are in adjacent genres. Partial-genre matching mitigates this but does not remove it.
- Numeric dead zones: Very tight σ values (small σ) can create "dead zones" where songs that are slightly off-target score near zero; use σ ≈ 0.15 to be forgiving while still discriminating.
- Catalog bias: Small or imbalanced catalogs will skew recommendations toward over-represented genres or production styles in `songs.csv`. Our catalog is 30% lofi music with zero country, classical, or electronic music.
- Lack of behavioral signals: This design ignores collaborative signals (co-listens, skips, saves). It cannot learn the subtle patterns that come from other users' behavior and may miss serendipitous recommendations.

**For comprehensive analysis of five major filter bubbles and a complete evaluation framework, see `model_card.md` Section 6 (Limitations and Bias) and `FILTER_BUBBLES_ANALYSIS.md` (8,000+ word deep-dive).**

These biases are intentional trade-offs for interpretability in this educational simulation. The model card (`model_card.md`) documents these limitations alongside experimental evidence from a six-profile evaluation.
![alt text](image.png)


### Evaluation for Different User Preferences

**Profile 1: High-Energy Pop Enthusiast**
![Profile 1 - Pop](image-copy.png)

**Profile 2: Chill Lofi Listener**
![Profile 2 - Lofi](image-copy2.png)

**Profile 3: Edge Case - Contradictory Preferences**
![Profile 3 - Edge Case](image-copy3.png)
---

## Getting Started

### Setup

1. Create a virtual environment (optional but recommended):

   ```bash
   python -m venv .venv
   source .venv/bin/activate      # Mac or Linux
   .venv\Scripts\activate         # Windows

2. Install dependencies

```bash
pip install -r requirements.txt
```

3. Run the app:

```bash
python -m src.main
```

### Running Tests

Run the starter tests with:

```bash
pytest
```

You can add more tests in `tests/test_recommender.py`.

---

## Experiments You Ran

### Sensitivity Weight Shift Experiment
To test how the algorithm prioritizes features, we shifted feature weights:
- **Original weights:** Genre 2.0, Energy 1.0
- **Test weights:** Genre 1.0, Energy 2.0

**Results:** Energy-prioritized version changed rankings significantly for high-energy and low-energy users, confirming that genre weight dominates the baseline. The shift revealed that genre acts as a "filter" (accept/reject) while energy acts as a "ranker" (fine-tune within accepted set).

### Six-Profile Evaluation
Tested with diverse user profiles to identify edge cases:
1. **Pop Enthusiast** (mainstream) → 5.49/6.0 ✓
2. **Lofi Listener** (mainstream) → 5.88/6.0 ✓
3. **Rock Fan** (mainstream) → 5.73/6.0 ✓
4. **Contradictory Prefs** (ambient + intense mood) → 2.32/6.0 ⚠
5. **Extreme Calm/Cheerful** (low energy + high valence) → 3.88/6.0 ⚠
6. **Country/Nostalgic** (cold-start, no genre match) → 2.00/6.0 ⚠

**Key finding:** 52% fairness gap between mainstream (avg 5.70) and niche users (avg 2.73).

---

## Reflection

See [`model_card.md`](model_card.md) for the complete Model Card with full sections on:
- Model Name: **MoodBeam 1.0**
- How It Works (plain language explanation)
- Data (10 songs, imbalanced genres/moods)
- Strengths & Limitations (5 major biases identified)
- Evaluation (6 profiles, 52% fairness gap)
- Future improvements & personal learning moments

