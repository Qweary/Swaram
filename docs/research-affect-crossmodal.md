# Research Brief — Image → Affect → Music: A Cited Bridge Model

**Author:** Pre-fabrication domain-research pass
**Date:** 2026-06-15
**Status:** Research only. This is the knowledge foundation for a new AudioHax specialist that will own the image-affect → musical-character mapping. It does NOT design AudioHax or author code.
**Scope:** Evidence base + buildable mapping tables for translating an image's emotional/energetic content into musical character, using AudioHax's existing pure-Rust features. Every claim carries a direction, a confidence level, and a real citation.

---

## 0. Why this brief exists (the failure being fixed)

AudioHax currently produces a slow-to-mid "ballad" for *every* image, including bright, chaotic, highly-saturated, high-energy abstract paintings that should feel fast and joyful. Reading the current `assets/mappings.json` shows the mechanism precisely:

- **Tempo is driven by brightness alone and capped at 120 BPM** (`brightness_to_tempo_bpm: {"71-100": 120}`). There is no path for saturation, edge activity, colorfulness, or complexity to push tempo higher. A maximally energetic image tops out at mid-tempo.
- **`character` defaults to `"ballad"` with an empty `rules` array** — so character is constant regardless of features.
- **There is no aggregated "energy"/arousal signal.** Saturation only touches harmonic complexity; edge density only touches rhythm and form. Nothing combines the arousal-bearing features into one quantity that co-drives tempo + loudness + density.

The literature below shows the fix is not "add more thresholds" — it is to insert a **principled affect bridge**: collapse the image features into **arousal** and **valence**, then drive the musical character knobs (tempo, dynamics, density, articulation, register, mode, dissonance) from those two axes. The single most load-bearing correction is that **arousal must be a composite of saturation + colorfulness + complexity + edge activity, and that composite must drive tempo (uncapped), loudness, and density together.**

---

## 1. Executive Summary — the recommended bridge model

**Use a two-stage bridge:**

1. **Image → Affect (valence/arousal), the primary path.** Map AudioHax's low-level features onto Russell's two-dimensional **valence × arousal circumplex** [Russell 1980]. Two relationships are robust enough to be load-bearing:
   - **Saturation (+ colorfulness + complexity + edge activity) → AROUSAL (positive).** [Valdez & Mehrabian 1994; Berlyne 1971]
   - **Brightness → VALENCE (positive).** [Valdez & Mehrabian 1994; Wilms & Oberfeld 2018]
   Then drive musical character from affect: **arousal → tempo, loudness, rhythmic density, register, articulation**; **valence → mode (major/minor) and consonance/dissonance**. The music-side directions are the best-established part of the whole chain [Eerola, Friberg & Bresin 2013; Juslin & Laukka 2003].

2. **Image → Music directly, via cross-modal correspondence, the secondary path** for features that have a perceptually grounded shortcut that does *not* need to route through emotion: **brightness ↔ pitch height**, **subject size ↔ pitch (inverse)**, **vertical position ↔ register**, **visual motion/energy ↔ tempo**, **angularity/edge ↔ timbral sharpness/dissonance** [Spence 2011; bouba/kiki — Ćwiek et al. 2022].

**Dimensional, not categorical.** Map to continuous valence/arousal, not to a handful of emotion labels. A continuous space lets graded image features produce graded musical parameters and handles ambiguous images gracefully; categorical models force a winner-take-all label and resolve ambiguous stimuli poorly [Eerola & Vuoskoski 2011].

**The three to four highest-confidence mappings that directly fix the "bright energetic image → no energy" bug:**
1. **Saturation → arousal → tempo + loudness + density** (positive; HIGH). Saturation is *the* dominant arousal driver in the color-emotion literature.
2. **Colorfulness/variety + visual complexity + edge activity → arousal** (positive; MEDIUM–HIGH), pooled into the same arousal composite so a chaotic multi-colored image reads as high-energy.
3. **Arousal → tempo, with the tempo ceiling removed** (positive; HIGH on the music side). Fast tempo is the strongest single arousal cue in music; the current 120 BPM cap is the proximate cause of the failure.
4. **Brightness → valence → major mode + consonance** (positive; HIGH), so a bright image also reads as *joyful*, not merely loud.

**The single biggest honest limitation:** "energy" and "arousal" are reachable from pure-Rust low-level features (saturation, colorfulness, complexity, edge density) with good support. **"Joy" in the everyday sense is only partially reachable** — pure features can deliver *bright-major-consonant-fast*, which is the musical signature of happiness, but they cannot recognize *why* an image is joyful (a smiling face, a celebration, a sunny beach). True semantic affect — and the warm/cool hue→valence intuition, which is weak and culturally contingent — requires scene/object recognition (ML), which is an opt-in later tier, not the pure-Rust default.

---

## 2. Affect Models (Research Q1)

**Russell's circumplex** [Russell 1980] places affect in a 2-D space of two orthogonal axes:
- **Valence** (pleasure–displeasure): the positive↔negative quality of the state — how good or bad it feels.
- **Arousal** (activation): the energy/intensity of the state, from calm/sleepy (low) to excited/activated (high).

A third axis, **dominance** (control vs. submission), completes the **PAD** model [Mehrabian & Russell 1974; Russell & Mehrabian 1977], but valence and arousal carry most of the explanatory weight and most affective-computing systems drop dominance. **Recommendation: use valence + arousal; treat dominance/tension as an optional later refinement.**

**Why dimensional beats categorical for image→music:** a continuous V/A space maps *graded* inputs to *graded* outputs and places ambiguous stimuli between poles; categorical models (happy/sad/angry) force a single label and have poorer resolution for emotionally ambiguous material [Eerola & Vuoskoski 2011, who showed a 3-D music-emotion model reduces cleanly to 2-D valence×arousal without major loss of fit]. For a feature-driven engine this is decisive: the image features *are* continuous, so the bridge should be too.

| Source feature | Affect role | Direction | Confidence | Citation |
|---|---|---|---|---|
| (model choice) | use valence × arousal as the 2-D bridge | — | HIGH | Russell 1980; Eerola & Vuoskoski 2011 |
| (model choice) | dominance/tension = optional 3rd axis | — | MEDIUM | Russell & Mehrabian 1977 |

---

## 3. Image → Affect (Research Q2)

Anchor equations from **Valdez & Mehrabian 1994** (standardized regressions; B = brightness/value, S = saturation, both as the dominant terms):

> **Pleasure = .69·B + .22·S  |  Arousal = −.31·B + .60·S  |  Dominance = −.76·B + .32·S**

Read directly: **brightness dominates pleasure (valence); saturation dominates arousal.** Replicated by **Wilms & Oberfeld 2018** ("saturated and bright colors were associated with higher arousal"; valence highest for bright, saturated colors).

**Buildable mapping table (AudioHax feature names from `ImageUnderstanding`):**

| Source feature (AudioHax field) | Affect dim | Direction | Confidence | Citation |
|---|---|---|---|---|
| `avg_saturation` | **arousal** | **positive (dominant driver)** | **HIGH** | Valdez & Mehrabian 1994; Wilms & Oberfeld 2018 |
| `avg_brightness` | **valence** | **positive (dominant driver)** | **HIGH** | Valdez & Mehrabian 1994; Wilms & Oberfeld 2018 |
| `avg_brightness` | arousal | mildly negative, AND only raises arousal under high saturation (interaction) | MEDIUM | Valdez & Mehrabian 1994; Wilms & Oberfeld 2018 |
| `colorfulness` (`hue_spread`) | arousal | positive (color variety = collative/arousal-potential variable) | MEDIUM | Machajdik & Hanbury 2010; Berlyne 1971 |
| `complexity`, `texture` | arousal | positive (monotone); valence follows an inverted-U (Wundt curve) — peaks at *moderate* complexity | MEDIUM | Berlyne 1971; Lu et al. 2012 |
| `edge_activity` (edge density / spatial frequency) | arousal | positive (complexity/arousal proxy) | LOW–MEDIUM | Machajdik & Hanbury 2010 (by analogy to Berlyne); no isolated landmark study |
| `quadrant_contrast` (value contrast) | arousal | positive | LOW–MEDIUM | Machajdik & Hanbury 2010; no isolated landmark study |
| shape roundness vs angularity (proxy: low vs high `complexity`/`edge_activity`) | valence (round→+), arousal (angular→+) | round → pleasant/calm; angular → higher arousal/threat | MEDIUM | Lu et al. 2012 |
| composition organization / figure-ground (`fg_bg_contrast`, balance) | valence | positive (processing fluency → pleasure) | LOW–MEDIUM | Reber, Schwarz & Winkielman 2004 (principle mainstream; exact ref unverified this session) |
| `dominant_hue` warmth (warm reds-oranges vs cool blues) | valence (and some arousal) | **unstable/contested**: warm→arousal is the more reliable part; warm→positive-valence reverses with saturation and is culturally contingent | **LOW** | Wilms & Oberfeld 2018; Valdez & Mehrabian 1994 (their own hue ranking puts blue/green among *most* pleasant) |

**Honesty note:** treat `avg_saturation → arousal` and `avg_brightness → valence` as load-bearing. Treat `dominant_hue → valence` (the warm=happy intuition) as a weak garnish, not a control axis — it is the most culturally contingent link in the literature and its sign is not stable.

**Recommended arousal composite (design guidance, not a verified formula):** arousal ≈ weighted sum of normalized `avg_saturation` (highest weight), `colorfulness`, `complexity`, `edge_activity`. This is the piece missing from `mappings.json` today and the direct cause of the energetic-image failure.

---

## 4. Affect → Music (Research Q3)

The best-established link in the whole chain. Two structuring facts:
- **Most cues load on arousal; MODE is the dominant valence carrier.** Eerola, Friberg & Bresin 2013 effect-size ranking: **mode 0.29 > tempo 0.14 > register 0.08 > dynamics 0.04 > articulation 0.02 > timbre 0.01**, and cues combine roughly **linearly and additively** (~77–89% variance explained). This additivity is a gift for a rule-based engine: you can sum cue contributions.
- Directions below are the Gabrielsson & Lindström / Juslin & Laukka 2003 consensus, reproduced in Cespedes-Guevara & Eerola 2018.

**Buildable mapping table (affect → musical parameter):**

| Affect dim | Musical parameter | Direction | Confidence | Citation |
|---|---|---|---|---|
| **arousal ↑** | **tempo** | faster (REMOVE the current 120 BPM cap) | **HIGH** | Hevner 1937; Eerola et al. 2013 (tempo strongest arousal cue) |
| **arousal ↑** | **dynamics / loudness** | louder | **HIGH** | Juslin & Laukka 2003; Eerola et al. 2013 |
| arousal ↑ | rhythmic density / note rate | more notes, faster onsets | HIGH | Juslin & Laukka 2003; Gabrielsson & Lindström 2010 |
| arousal ↑ | articulation | toward staccato (detached); legato = calmer | MEDIUM–HIGH | Juslin & Laukka 2003; Eerola et al. 2013 |
| arousal ↑ | register / pitch height | higher | HIGH (arousal); MEDIUM (also reads happier) | Hevner 1937; Eerola et al. 2013 |
| arousal ↑ | texture density / # voices | denser, more layers | MEDIUM | Webster & Weir 2005; Gabrielsson & Lindström 2010 |
| arousal ↑ | timbre | brighter/sharper | MEDIUM | Eerola et al. 2013; Juslin & Laukka 2003 |
| **valence ↑** | **mode** | **major; valence ↓ → minor** | **HIGH (Western listeners; learned/cultural)** | Hevner 1936; Eerola et al. 2013 (top effect size) |
| valence ↑ | harmonic consonance/dissonance | more consonant; valence ↓ → more dissonant/tension | HIGH | Gabrielsson & Lindström 2010; Juslin & Laukka 2003 |
| valence ↑ | tempo (secondary) | slightly faster (but reverses inside sad expressions) | MEDIUM | Eerola et al. 2013 |

**Church/diatonic modes** (Lydian/Ionian "bright" → Phrygian/Locrian "dark"): **LOW confidence / music-theory convention.** Only the **major (Ionian) vs minor (Aeolian)** valence contrast is empirically validated; no peer-reviewed validation of a graded 7-mode affect ordering was found. The current `hue_to_mode` table that spreads all six church modes across hue is therefore *expressive* but not *empirically grounded* — keep it only as a colorist choice, and let valence (not raw hue) decide the load-bearing major/minor split.

**Musical-emotion caveat worth flagging to the designer:** in *music* (unlike vocal startle), **fear = fast + minor + *low/soft* loudness**, which distinguishes it from anger (fast + minor + loud). A naïve "fear = loud" rule is wrong for the musical case [Cespedes-Guevara & Eerola 2018].

---

## 5. Cross-Modal Correspondence — the direct bridge (Research Q4)

Perceptually grounded visual↔auditory links that map a feature to a sound parameter **without routing through emotion**. Anchor review: **Spence 2011** (catalogue confirmed; per-row corroboration cited).

| Source feature (AudioHax) | Auditory target | Direction | Confidence | Citation |
|---|---|---|---|---|
| `avg_brightness` / per-bar `avg_brightness` | **pitch height** | brighter → higher pitch | **HIGH** | Marks 1987; McCormick et al. 2018 (low-level sensory effect, present from age ~4) |
| `subject_size` (`subject.area_frac`) | **pitch** | bigger subject → **lower** pitch (large objects vibrate lower) | **HIGH** | Gallace & Spence 2006; Evans & Treisman 2010; Spence 2011 |
| vertical position (`mass_centroid.y`, `vertical_emphasis`, per-bar y) | **register** | higher in frame → higher pitch | **HIGH** | Walker et al. 2010; Parise et al. 2014; McCormick et al. 2018 |
| visual energy/motion (arousal composite; `subject_energy`, `edge_activity`) | **tempo / event rate** | faster/busier → faster | MEDIUM–HIGH | Spence 2011 (temporal-rate); Eitan & Granot 2006 |
| angularity / jaggedness / `edge_activity` | **timbral sharpness / dissonance** | angular/jagged → sharp, harsh, dissonant; rounded → smooth, soft (bouba/kiki) | **HIGH** (~95–98% agreement, cross-cultural) | Köhler 1929; Ramachandran & Hubbard 2001; Ćwiek et al. 2022 |
| `edge_activity` / `texture` sharpness | **timbral brightness** | sharper visual texture → brighter/harsher timbre | MEDIUM–HIGH | Adeli, Rouat & Molotchnikoff 2014 |

**Design implication:** brightness→pitch and size→pitch are robust *direct* mappings — use them at the per-bar/per-voice level (e.g. bright pixels → higher melody notes) where they are cheaper and more defensible than an affect detour. Reserve the affect bridge for the *macro* character (tempo/mode/loudness/density).

---

## 6. Saliency → Musical Role (Research Q5)

The owner's intuition — **most-salient subject drives the MELODY; least-prominent regions drive the BACKGROUND/accompaniment** — rests on two robust principles and one heuristic inference.

- **Robust principle 1 — visual saliency is real and computable.** A topographic saliency map selects attended regions in order of decreasing saliency [Itti, Koch & Niebur 1998]. AudioHax's `pick_subject_region` (center-bias + local contrast + saturation pop) is a cheap, defensible saliency proxy in this spirit.
- **Robust principle 2 — figure-ground is a *shared* organization across vision and audition.** Bregman explicitly frames auditory streaming in visual figure/ground terms [Bregman 1990]. In polyphony the melody is the perceptual figure, and the **high voice is most salient** ("high-voice superiority," with cortical-evoked-potential support) [Trainor et al. 2014]. So "melody = auditory figure" is well-grounded.
- **Heuristic inference — "biggest/most-salient subject = melody, background = accompaniment."** This specific rule is a *reasonable design heuristic assembled from the two robust facts above*, **not an empirically validated perceptual law.** No study was found that directly tests and validates that exact mapping. Audio-visual saliency and image-sonification systems do modulate output by region prominence, which supports the family of ideas, but not this precise rule.

| Source feature (AudioHax) | Musical role | Direction | Confidence | Citation |
|---|---|---|---|---|
| `subject_*` (the `pick_subject_region` winner) | melody / foreground voice | most-salient region → carries the melody (place it in/toward the high voice) | MEDIUM (principled-but-heuristic) | Itti et al. 1998; Bregman 1990; Trainor et al. 2014 |
| `background_energy`, corner/border cells | accompaniment / pad / background | least-prominent regions → harmonic bed | MEDIUM (design heuristic) | Bregman 1990 (figure-ground); no direct validating study |
| melody voice placement | register | melody toward the **high** voice (high-voice superiority) | MEDIUM–HIGH | Trainor et al. 2014 |

**Verdict:** implement it — it is well-motivated and consistent with how listeners parse polyphony — but document it as a *design heuristic*, not as "proven."

---

## 7. Confidence & Honesty — robust vs. speculative, and the pure-Rust line (Research Q6)

**Robust enough to be load-bearing (build on these):**
- `avg_saturation` → arousal (positive). [Valdez & Mehrabian 1994]
- `avg_brightness` → valence (positive). [Valdez & Mehrabian 1994; Wilms & Oberfeld 2018]
- arousal → tempo, loudness, density, register (positive); valence → major/minor + consonance. [Eerola et al. 2013; Juslin & Laukka 2003]
- brightness↔pitch, size↔pitch, vertical↔register, angularity↔dissonance (direct cross-modal). [Spence 2011; Ćwiek et al. 2022; McCormick et al. 2018]
- additive cue combination — sum the contributions. [Eerola et al. 2013]

**Speculative / supported-by-analogy (use, but tune by ear; do not over-claim):**
- `edge_activity`, `quadrant_contrast`, `colorfulness` → arousal (no isolated landmark psychophysics study; principled via Berlyne complexity and the Machajdik & Hanbury feature tradition).
- complexity → valence inverted-U (Berlyne's Wundt curve is classic but effect sizes vary; the *peak* location is image-dependent).
- figure-ground → valence via processing fluency (principle mainstream; exact citation unverified this session).
- "most-salient subject → melody" (design heuristic, not a law).

**Weak / contested (demote to garnish or drop):**
- `dominant_hue` warmth → valence. Sign is unstable, reverses with saturation, and is culturally contingent. Do **not** make hue a primary valence control. The existing `hue_to_mode` six-mode spread is expressive, not empirically grounded.
- Full church-mode brightness ordering (only major/minor is validated).

**The pure-Rust-reachable vs. ML-needed line (candid):**

| Desired effect | Pure-Rust reachable? | How |
|---|---|---|
| **Energy / high arousal** | **YES** | saturation + colorfulness + complexity + edge_activity → arousal composite → uncapped tempo + loudness + density. This is the direct fix for the current failure. |
| **Fast-paced** | **YES** | same arousal composite → tempo; remove the 120 BPM ceiling. |
| **Joy (musical signature)** | **PARTIAL** | brightness → valence → major + consonant + (with high arousal) fast/bright = the *acoustic signature* of happiness. Achievable from pure features. |
| **Joy (semantic — knowing the scene is happy)** | **NO (needs ML)** | recognizing a smiling face, a celebration, a sunny landscape requires object/scene recognition. Pure low-level features cannot read *subject matter*, only its statistical texture/color/structure. |
| **Reliable warm=happy / cool=sad** | **NO / unreliable** | hue→valence is weak and culturally contingent; pure hue cannot deliver this robustly. A semantic/learned model would be needed, and even then the link is contested. |
| **"This painting depicts a storm / a party / grief"** | **NO (needs ML)** | semantic affect of *content* is out of pure-Rust scope; opt-in later tier. |

**Net:** the owner's three desired effects — *energy, joy, fast-paced* — are **largely reachable from pure-Rust features**, because they correspond to **arousal (saturation/colorfulness/complexity-driven) and the acoustic signature of positive valence (brightness-driven major/consonant)**. What pure Rust cannot do is recognize *why* an image is joyful or read its subject matter; that is the ML opt-in tier. The current "always a ballad" failure is **not** a limit of pure Rust — it is a missing arousal composite and a tempo cap, both fixable within the pure-Rust default.

---

## 8. Bibliography (verified)

Sources below were located and/or fetched during this research. Items flagged "abstract/venue verified, full text not fetched" or "unverified this session" are marked lower-confidence in the tables.

1. Russell, J. A. (1980). A Circumplex Model of Affect. *Journal of Personality and Social Psychology*, 39(6), 1161–1178.
2. Mehrabian, A., & Russell, J. A. (1974). *An Approach to Environmental Psychology.* MIT Press. (origin of PAD)
3. Russell, J. A., & Mehrabian, A. (1977). Evidence for a three-factor theory of emotions. *Journal of Research in Personality*, 11(3), 273–294. doi:10.1016/0092-6566(77)90037-X
4. Eerola, T., & Vuoskoski, J. K. (2011). A comparison of the discrete and dimensional models of emotion in music. *Psychology of Music*, 39(1), 18–49. doi:10.1177/0305735610362821
5. Yang, Y.-H., & Chen, H. H. (2012). Machine Recognition of Music Emotion: A Review. *ACM TIST*, 3(3), Art. 40. doi:10.1145/2168752.2168754 (venue verified; full text not fetched)
6. Valdez, P., & Mehrabian, A. (1994). Effects of color on emotions. *Journal of Experimental Psychology: General*, 123(4), 394–409. doi:10.1037/0096-3445.123.4.394 (regression equations fetched, pubmed 7996122)
7. Wilms, L., & Oberfeld, D. (2018). Color and emotion: effects of hue, saturation, and brightness. *Psychological Research*, 82(5), 896–914. doi:10.1007/s00426-017-0880-8
8. Machajdik, J., & Hanbury, A. (2010). Affective image classification using features inspired by psychology and art theory. *Proc. 18th ACM Multimedia*, 83–92. doi:10.1145/1873951.1873965
9. Lu, X., Suryanarayan, P., Adams, R. B. Jr., Li, J., Newman, M. G., & Wang, J. Z. (2012). On shape and the computability of emotions. *Proc. 20th ACM Multimedia*, 229–238. doi:10.1145/2393347.2393384
10. Berlyne, D. E. (1971). *Aesthetics and Psychobiology.* Appleton-Century-Crofts. (arousal potential / collative variables / Wundt curve; verified via Marin & Leder, "Berlyne Revisited," Front. Hum. Neurosci. 2016)
11. Reber, R., Schwarz, N., & Winkielman, P. (2004). Processing fluency and aesthetic pleasure. *Personality and Social Psychology Review*, 8(4), 364–382. (principle mainstream; exact ref NOT fetched this session — lower-confidence)
12. Itti, L., Koch, C., & Niebur, E. (1998). A Model of Saliency-Based Visual Attention for Rapid Scene Analysis. *IEEE TPAMI*, 20(11), 1254–1259. (antecedent: Koch & Ullman 1985)
13. Bregman, A. S. (1990). *Auditory Scene Analysis: The Perceptual Organization of Sound.* MIT Press.
14. Trainor, L. J., Marie, C., Bruce, I. C., & Bidelman, G. M. (2014). Explaining the high voice superiority effect in polyphonic music. *Hearing Research*, 308, 60–70.
15. Hevner, K. (1936). Experimental studies of the elements of expression in music. *American Journal of Psychology*, 48, 246–268; Hevner, K. (1937). The affective value of pitch and tempo in music. *AJP*, 49, 621–630.
16. Juslin, P. N., & Laukka, P. (2003). Communication of emotions in vocal expression and music performance: Different channels, same code? *Psychological Bulletin*, 129(5), 770–814.
17. Gabrielsson, A., & Lindström, E. (2010). The role of structure in the musical expression of emotions. In *Handbook of Music and Emotion*, OUP. (earlier: Gabrielsson & Lindström 2001)
18. Eerola, T., Friberg, A., & Bresin, R. (2013). Emotional expression in music: contribution, linearity, and additivity of primary musical cues. *Frontiers in Psychology*, 4:487. (effect-size ranking + additivity)
19. Cespedes-Guevara, J., & Eerola, T. (2018). Music Communicates Affects, Not Basic Emotions. *Frontiers in Psychology*, 9:215. (cue→emotion table; musical fear = soft)
20. Webster, G. D., & Weir, C. G. (2005). Emotional Responses to Music: Interactive Effects of Mode, Texture, and Tempo. *Motivation and Emotion*, 29(1).
21. Spence, C. (2011). Crossmodal correspondences: A tutorial review. *Attention, Perception, & Psychophysics*, 73(4), 971–995. doi:10.3758/s13414-010-0073-7 (venue/DOI/scope verified; full text not loaded — per-row corroboration cited)
22. Marks, L. E. (1987 and *The Unity of the Senses*, 1978). Brightness↔pitch. (1978 monograph not directly fetched; 1987 work verified via secondary citation)
23. McCormick, K., et al. (2018/2019). Lightness/pitch and elevation/pitch crossmodal correspondences are low-level sensory effects. *Atten. Percept. Psychophys.* doi:10.3758/s13414-019-01668-w
24. Gallace, A., & Spence, C. (2006); Evans, K. K., & Treisman, A. (2010); Walker, P., et al. (2010); Parise, C. V., Knorre, K., & Ernst, M. O. (2014). size↔pitch and elevation↔pitch (verified via PMC11652408 review).
25. Köhler, W. (1929). *Gestalt Psychology*; Ramachandran, V. S., & Hubbard, E. M. (2001). Synaesthesia — A window into perception, thought and language. *Journal of Consciousness Studies*, 8(12), 3–34. (bouba/kiki origin)
26. Ćwiek, A., et al. (2022). The bouba/kiki effect is robust across cultures and writing systems. *Philosophical Transactions of the Royal Society B.* (PMC8591387)
27. Adeli, M., Rouat, J., & Molotchnikoff, S. (2014). Audiovisual correspondence between musical timbre and visual shapes. *Frontiers in Human Neuroscience*, 8:352. (PMC4038957)
28. Eitan, Z., & Granot, R. Y. (2006). How music moves: Musical parameters and listeners' images of motion. *Music Perception*, 23(3), 221–248.

**Lower-confidence / unverified flags carried into the tables:**
- Contrast→arousal, edge-density→arousal, colorfulness→arousal: feature-tradition + Berlyne analogy, no isolated landmark study.
- Full church-mode valence ordering: music-theory convention, not empirically validated (only major/minor is).
- Reber, Schwarz & Winkielman 2004: principle mainstream, exact reference not fetched this session.
- "Most-salient subject → melody": design heuristic assembled from verified principles, not a directly validated law.
- Spence 2011 full text not loaded; correspondences corroborated by the per-row secondary sources.
