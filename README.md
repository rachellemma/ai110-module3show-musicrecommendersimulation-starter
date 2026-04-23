# 🎵 Music Recommender Simulation

## Project Summary

  VibeMatch 1.0 is a scoring-based music recommendation system built as a classroom simulation. Given a user's stated preferences — favorite genre, current mood, target energy level, and acousticness — it scores every song in a 25-song catalog against those four signals, filters out weak matches, and returns the top 5 recommendations along with a plain-language explanation of why each song was chosen. The goal was to understand how real recommenders turn preference data into ranked lists, and to discover where that process breaks down.

---

## How The System Works

  Each Song stores genre (categorical), mood (categorical), energy (numeric, 0-1), and acousticness (numeric, 0-1). Energy measures how intense or driving a song feels. Acousticness measures how much the song relies on real instruments vs. electronic production.

  The UserProfile stores the user's preferred values for the same features: preferred genre, preferred mood, target energy (0-1), and target acousticness (0-1). For example: genre = "r&b", mood = "chill", target_energy = 0.45, target_acousticness = 0.65.

  The Recommender scores each song using four weighted components that add up to 1.0:

  - Genre match (22.5%): 1.0 if the song's genre matches the user's preferred genre, 0.0 if not
  - Mood match (25%): 1.0 if the song's mood matches the user's preferred mood, 0.0 if not
  - Energy similarity (30%): `1.0 - abs(song_energy - target_energy)` — closer to the user's target gets a higher score
  - Acousticness similarity (22.5%): `1.0 - abs(song_acousticness - target_acousticness)` — same idea

  Total score = (0.225 × genre_match) + (0.25 × mood_match) + (0.30 × energy_similarity) + (0.225 × acousticness_similarity)

  The original starter design weighted genre at 45%. The weights were shifted to test whether continuous features (energy, acousticness) could surface better-fitting songs even when the genre didn't match exactly.

  Songs are ranked by their total score in descending order. Only songs that score at least 40% (0.40) are included in the results. The top K songs from that filtered list are returned as recommendations.

---

## Getting Started

### Setup

1. Create a virtual environment (optional but recommended):

   ```bash
   python -m venv .venv
   source .venv/bin/activate      # Mac or Linux
   .venv\Scripts\activate         # Windows
   ```

2. Install dependencies:

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

---

## Experiments 

Four user profiles were tested to stress-test the scoring logic under different conditions.

**Profile 1 — Conflicting preferences: pop / sad mood / high energy (0.9) / high acousticness (0.65)**

![Conflicting energy and mood case](image-1.png)

No song in the dataset is both pop and sad, so the system split its attention between genre and mood. Sunrise City matched on genre (pop) but had the wrong mood (happy); Velvet Underground Blues matched on mood (sad) but had the wrong genre (blues). Neither result fully satisfied the user. High energy and high acousticness are also contradictory physical properties, so those two signals worked against each other and dragged all scores toward the middle.

---

**Profile 2 — Missing genre: bossa nova / happy mood / moderate energy (0.5) / acoustic (0.65)**

![Genre that doesn't exist case](image-2.png)

Because bossa nova does not exist in the dataset, genre match was always zero. The system fell back entirely on mood, energy, and acousticness — happy mood became the dominant signal, returning Rooftop Lights and Sunrise City. The system degraded gracefully but gave the user no indication that their stated genre was silently ignored.

---

**Profile 3 — Self-defeating preferences: rock / angry mood / high energy (0.8) / maximum acousticness (1.0)**

![Contradictory preferences case](image-3.png)

High energy and maximum acousticness directly contradict each other. Storm Runner matched on genre and came close on mood (intense vs. angry), but its acousticness of 0.10 versus a target of 1.0 created a penalty so severe that nearly every song was filtered out. Only one song cleared the 0.40 threshold — the system returned a single recommendation.

---

**Profile 4 — Low-energy user: jazz / chill mood / zero energy (0.0) / acoustic (0.65)**

![Energy preference 0.00 case](image-4.png)

Jazz was the preferred genre, but the only jazz song in the catalog (Coffee Shop Stories) has a relaxed mood rather than chill. That mood mismatch, combined with the reduced genre weight, meant it didn't crack the top 5. The list filled with chill-mood songs from lofi, ambient, and r&b instead — the jazz preference was quietly ignored.

---

**Key takeaway:** Profile 3 failed loudly with almost no output. Profile 4 failed silently with a full list that never acknowledged the stated genre. Both are failure modes, but only one is obvious to the user.

---

## Limitations and Risks

- **Genre bias:** Genre carries 22.5% of the score (45% in the original design), so a song that perfectly matches the user's mood, energy, and acousticness can still rank low if the genre label doesn't match exactly. A great R&B-adjacent song labeled "soul" or "jazz" would score as if it were completely wrong.

- **Dataset imbalance:** R&B makes up 32% of the catalog (8 of 25 songs), while jazz, blues, classical, and reggae each have only one song. An R&B fan has eight chances to match; a jazz fan has one — an unfair structural advantage built into the data before scoring even runs.

- **Exact string matching on mood:** Mood is binary — "chill" either matches or it doesn't. Similar moods like "relaxed" or "romantic" score zero, even though a real user might enjoy them equally.

- **Acousticness and energy may overlap:** Songs that are chill tend to also be acoustic, so these two features can reward the same songs twice rather than adding independent signal.

- **Energy floor problem:** Most songs in the catalog have energy values between 0.35 and 0.97. A user who sets `target_energy: 0.0` can never achieve a high energy score, even for the mellowest songs available.

- **No user history:** The profile is fixed. The system has no way to learn from what the user actually plays or skips.

---

## Reflection

Building this recommender made it clear that the dataset is just as important as the algorithm — the scoring math can be perfectly designed, but if the catalog doesn't have the songs a user needs, the system will silently return wrong answers with high confidence. The most striking example was Profile 4: a jazz-requesting user received zero jazz songs in their top five, and the output looked completely normal. There was no error, no warning, and no sign that the user's stated preference had been functionally ignored.

The weight experiments were equally revealing. Halving the genre weight from 0.45 to 0.225 seemed like a small tweak, but it was enough to erase an entire genre from someone's recommendations. That sensitivity made me think differently about real platforms like Spotify or Apple Music — when a recommendation feels slightly off, it could be a data gap, a weight imbalance, or a silent fallback that the user was never told about. The system looks intelligent on the surface, but it is really just arithmetic applied to whatever data happens to be there.

Read the full model card here: [**Model Card**](model_card.md)
