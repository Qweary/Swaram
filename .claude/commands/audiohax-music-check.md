You are performing a musical code review of AudioHax. This is not a standard code review — you are evaluating whether the code produces musically coherent, expressive output. The project owner has a music performance degree (trombone) and professional experience. Use proper music theory terminology throughout.

Step 1: Read src/chord_engine.rs completely. Analyze:

VOICE LEADING QUALITY:
- Does the code track each voice's previous pitch across chord changes?
- Is there a closest-pitch-class or similar algorithm for smooth voice motion?
- Are parallel perfect fifths and octaves detected and avoided?
- Is voice crossing prevented (voice N stays above voice N-1)?
- Are common tones retained when possible?
- Do tendency tones resolve correctly (leading tone up to tonic, chordal seventh down by step)?
- What is the maximum melodic interval allowed per voice? Is it enforced?

HARMONIC LANGUAGE:
- What modes/scales are supported? Are they correctly implemented (check interval patterns)?
- How are chord progressions selected? Do they follow functional harmony or are they arbitrary?
- Is modal interchange implemented? Is it musically motivated (e.g., borrowing bVII from parallel minor)?
- Are secondary dominants handled? Do they resolve to their target?
- What chord extensions are available (7ths, 9ths, etc.)?

Step 2: Read the note decision logic in src/main.rs (worker_decide_action and surrounding orchestration). Analyze:

PHRASE STRUCTURE:
- Are steps grouped into phrases? What determines phrase length?
- Is there a sense of antecedent-consequent or tension-release within phrases?
- Do phrase boundaries feature cadential patterns (V-I, V-vi, IV-I)?
- Is there macro-level form across the full scan (exposition, development, recapitulation)?

DYNAMIC CONTOUR:
- Does velocity vary within phrases (crescendo/diminuendo)?
- Is there a messa di voce or similar dynamic shape?
- Are strong beats accented? Are phrase endings tapered?
- Does the macro form have an intensity arc (building to climax)?

RHYTHMIC VARIETY:
- How many distinct rhythm patterns exist?
- Is there variation beyond arpeggio vs. sustained?
- Does phrase position influence rhythm (simpler at boundaries, complex mid-phrase)?
- Is there ritardando at phrase endings? Rubato? Swing?

MELODIC INDEPENDENCE:
- Do voices have distinct roles (bass, melody, harmonic)?
- Does the melody voice have stepwise contour with occasional leaps?
- Are non-chord tones used (passing, neighbor, suspension, appoggiatura)?
- Is the bass voice in an appropriate register with idiomatic movement?

ARTICULATION:
- Does note duration vary (staccato, legato, portato)?
- Are MIDI CC messages used for expression, modulation, or sustain?
- Is articulation driven by visual features or arbitrary?

Step 3: Read assets/mappings.json. Assess whether the visual-to-musical mappings are musically meaningful or arbitrary.

Step 4: Produce a MUSICAL HEALTH REPORT with:

STRENGTHS: What the system does well musically.
WEAKNESSES: Specific musical problems, each with the code location and a concrete suggestion for improvement.
PRIORITY IMPROVEMENTS: The 3 highest-impact changes that would most improve musical output, ordered by impact.
OVERALL ASSESSMENT: A candid evaluation — does this sound like composed music, competent algorithmic composition, data sonification, or random noise? Where on that spectrum is it, and what would move it up?
