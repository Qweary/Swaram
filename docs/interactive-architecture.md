# Interactive Experience Architecture Plan

## The Central Problem: Batch to Real-Time

Everything about the interactive experience vision depends on solving one foundational problem. The current pipeline is batch-oriented: main() loads an image once, calls scan_image() to precompute all steps as a Vec<Vec<ScanBarFeatures>>, generates chords once from global features, then plays through the precomputed steps sequentially in play_scanned_steps_concurrent(). The image never changes after the initial load. Global features are computed once. Chords are fixed for the entire scan. The worker-coordinator barrier pattern assumes a known, fixed step count.

Every layer of the interactive vision requires the image to change while playback is happening, features to be re-extracted on the fly, and musical decisions to respond to those changes in real-time. This isn't a feature addition — it's a structural refactor of the core pipeline from "precompute everything, then play" to "continuously react to a changing image source."

The good news: the existing modules (image_analysis, chord_engine, mapping_loader, midi_output) are well-separated. The problem is entirely in main.rs's orchestration. The analysis functions, chord generation, and MIDI output are already stateless enough to call repeatedly in a loop. What needs to change is the control flow that connects them.


## Prerequisite Refactor: The Reactive Pipeline Core

Before any interactive feature can work, main.rs needs to be restructured around an event loop rather than a fixed iteration count. This refactor is the critical-path dependency for all three layers. It should be done as a standalone phase before the interactive features.

### Current Flow (Batch)

```
main():
  img = load_image()                          # once
  global = analyze_global(img)                # once
  chords = generate_chords(global)            # once
  steps = scan_image(img, N)                  # precompute ALL steps
  play_scanned_steps(steps, chords)           # iterate fixed N
    for step in 0..N:
      workers analyze precomputed steps[step]
      coordinator plays MIDI
      barrier sync
```

### Required Flow (Reactive)

```
main():
  image_buffer = SharedImageBuffer::new()     # mutable, shared
  event_loop:
    image_buffer.maybe_update(source)         # poll/receive new image data
    global = analyze_global(image_buffer)     # re-extract if image changed
    chords = update_chords_if_needed(global)  # recompute only on significant change
    step_features = analyze_current_bar(image_buffer, scan_position)  # one step
    actions = decide_actions(step_features, chords, phrase_state)
    schedule_and_play(actions)
    advance_scan_position()                   # or react to external control
    handle_input_events()                     # mouse, keyboard, game, network
```

The key differences are: the image can change between steps (or even mid-step); global analysis and chord generation only re-trigger when features change significantly (with hysteresis to avoid constant modulation); scan position can be driven by time, user input, or external events rather than a fixed counter; and input events from multiple sources feed into the loop.

### New Module: src/engine.rs (Pipeline Engine)

This is the core refactor. Extract the pipeline orchestration from main.rs into a new module that encapsulates the reactive loop. The engine should expose a trait-based interface:

```rust
/// The central pipeline engine. Owns the scan state, musical state,
/// and MIDI output. Receives image frames and input events,
/// produces musical output.
pub struct PipelineEngine {
    image_buffer: Arc<RwLock<Mat>>,
    global_features: GlobalFeatures,
    chord_state: ChordState,        // current chords + voice leading state
    phrase_state: PhraseState,       // current phrase position
    scan_position: ScanPosition,    // where the scan bar is
    midi: MidiOut,
    mappings: MappingTable,
    config: EngineConfig,
}

impl PipelineEngine {
    /// Process one tick of the pipeline. Call this at the step rate.
    /// Returns the actions taken (for visualization feedback).
    pub fn tick(&mut self) -> Result<TickResult>;

    /// Update the image buffer. The next tick() will use the new image.
    pub fn update_image(&mut self, image: Mat);

    /// Inject an external event (mouse, game, network).
    pub fn inject_event(&mut self, event: InteractionEvent);

    /// Query current state (for visualization overlay).
    pub fn current_state(&self) -> EngineSnapshot;
}
```

The engine's tick() method replaces the inner loop body of play_scanned_steps_concurrent(). Each tick: (1) reads the current image from the shared buffer, (2) extracts features for the current scan position, (3) checks whether global features have changed enough to warrant re-deriving mode/chords (using a configurable delta threshold — this is the hysteresis), (4) runs worker_decide_action for each instrument, (5) schedules and sends MIDI events, (6) advances the scan position.

The image_buffer is behind an Arc<RwLock<Mat>> so that input sources (mouse handler, camera thread, game bridge) can write new image data concurrently with the engine reading it. RwLock is the right primitive here: the engine reads frequently but input sources write relatively infrequently, so reader-writer separation avoids contention.

### Change Detection and Musical Hysteresis

A naive implementation that re-analyzes global features and regenerates chords every tick will sound terrible — the mode and key will thrash on every pixel change, producing constant jarring modulations. The engine needs a change detection layer:

Global feature deltas: compute the Euclidean distance between the current and previous GlobalFeatures. Only re-derive mode/key/chords when the delta exceeds a threshold (configurable, probably ~15% of the feature range). When it does trigger, the chord engine should modulate smoothly rather than jump — use a pivot chord that's diatonic in both the old and new key, then cadence into the new key over 2-4 steps. This is basically a musical key-change protocol.

Scan-bar feature deltas: these can change every tick because they're position-dependent. The existing feature extraction is already designed for this. No hysteresis needed at this level.

Rate limiting on heavy operations: global analysis involves whole-image statistics (histogram, Laplacian, contour detection). At 4 steps/second (250ms per step), this is fine. At higher tick rates (needed for responsive interaction), you'll want to throttle global analysis to run at most every N ticks or when the image-changed flag is set.

### Concurrency Model Change

The current barrier-synchronized worker pool assumes all workers process the same precomputed step in lockstep. In the reactive model, this pattern still works but with one change: workers no longer index into a precomputed Vec<Vec<ScanBarFeatures>>. Instead, each tick, the engine extracts features for the current scan position on-the-fly and distributes them to workers. The workers still compute InstrumentActions in parallel, the barrier still synchronizes them, and the coordinator still schedules MIDI events. The barrier pattern doesn't need to change — only the data source feeding it changes from precomputed to live-extracted.

Alternatively, for simplicity in the first implementation, the engine can compute features and decide actions single-threaded (the worker parallelism is an optimization, not a correctness requirement). Profile first: if analyze_scan_bar for a single position completes in under 5ms (likely, since it's just computing statistics on a small image region), the parallelism overhead isn't worth it for 4-8 instruments.

### What main.rs Becomes

After the refactor, main.rs becomes a thin CLI layer: parse arguments, determine the operating mode (batch scan, interactive mouse, camera, game server), construct the appropriate input source and the PipelineEngine, then run the appropriate event loop. The mode-specific loop code can live in separate modules (one for each interaction layer) that all use the same PipelineEngine interface.


## Layer 1: Dynamic Image Manipulation

### 1A: Mouse/Touch Painting

The user opens an image in an OpenCV window, draws on it with the mouse, and hears the music change in response. This is the simplest interactive mode and the first milestone.

#### New Module: src/interaction.rs

Handles input event capture, image modification, and feedback display.

```rust
pub enum InteractionEvent {
    MouseDown { x: i32, y: i32, button: MouseButton },
    MouseMove { x: i32, y: i32 },
    MouseUp { x: i32, y: i32 },
    KeyPress { key: char },
    // Future: touch, game event, network
}

pub enum BrushMode {
    Smear { radius: i32, strength: f32 },     // directional pixel smearing
    Paint { radius: i32, color: (u8,u8,u8) }, // solid color painting
    Blur { radius: i32, sigma: f32 },          // Gaussian blur brush
    Erase { radius: i32 },                     // restore to original image
}

pub struct InteractionState {
    pub brush: BrushMode,
    pub is_drawing: bool,
    pub last_position: Option<(i32, i32)>,
    pub original_image: Mat,  // for undo/erase
}
```

The mouse callback (set via OpenCV's highgui::set_mouse_callback) captures mouse events and forwards them through a channel to the engine loop. The image modification happens synchronously when the engine processes the event: it applies the brush effect to the shared image buffer, which is then picked up by the next tick()'s feature extraction.

Smearing is the most musically interesting brush because it creates continuous feature gradients rather than sharp color changes. Implementation: for each pixel in the brush radius, blend it with its neighbor in the drag direction using a weighted average. OpenCV's remap() or warpAffine() on a small ROI can do this efficiently. The strength parameter controls how far pixels shift per drag frame.

#### OpenCV Integration

OpenCV already provides everything needed. highgui::imshow and highgui::wait_key are already used for the scan bar overlay. highgui::set_mouse_callback registers the callback. No new crates required — this is pure OpenCV.

The tricky part is the event loop timing. The current code calls wait_key(1) inside the coordinator loop for overlay display. In the reactive model, the event loop needs to:
1. Call wait_key(ms) where ms is the time until the next tick minus processing time.
2. Process any mouse events that arrived during the wait.
3. Run one engine tick.
4. Update the display.

wait_key serves triple duty as a sleep function, event pump, and key input reader. This is OpenCV's standard pattern for interactive applications and it works fine for this use case.

#### Musical Response to Painting

When the user paints, the image changes under and ahead of the scan bar. The musical response depends on where they paint relative to the scan position. Painting behind the scan bar (already scanned) has no immediate musical effect — it will matter on the next scan pass if the image loops. Painting on or near the current scan bar position changes the music immediately at the next tick. Painting ahead of the scan bar "pre-composes" future musical material.

This is actually a compelling interaction model: the user is painting the future of the music in real time, and they hear the results as the scan bar reaches what they've painted.

#### Complexity: MEDIUM

The core work is the engine refactor (which is shared infrastructure). The mouse interaction layer itself is straightforward OpenCV callback plumbing. Image modification brushes are well-understood image processing operations. Estimate 2-3 days of focused work once the engine refactor is done.


### 1B: Camera-Based Tracking

A camera feed replaces or augments the static image. Physical objects in the camera's field of view become part of the image that the music pipeline scans.

#### Approach

image_source.rs already has CameraIndex support that captures a single frame. The extension is straightforward: instead of capturing once, run a continuous capture loop in a dedicated thread that writes frames to the shared image buffer.

```rust
pub struct CameraSource {
    capture: VideoCapture,
    target_fps: f32,
}

impl CameraSource {
    /// Spawns a thread that continuously captures frames and writes them
    /// to the shared image buffer. Returns a handle to stop the capture.
    pub fn start_continuous(
        &mut self,
        image_buffer: Arc<RwLock<Mat>>,
    ) -> JoinHandle<()>;
}
```

The camera thread runs at its own frame rate (typically 30fps). The engine's tick rate (4 Hz at 250ms/step) is much slower, so most camera frames will be silently overwritten before the engine reads them. This is fine and desirable — the engine always gets the latest frame.

#### Object Tracking for Physical Interaction

For the art installation use case, you'll want to detect specific objects or colored markers rather than using the raw camera image directly. OpenCV provides the building blocks: background subtraction (createBackgroundSubtractorMOG2) to isolate moving objects from a static background, contour detection to identify distinct objects, color filtering (inRange in HSV space) to track specific colored objects, and optical flow (calcOpticalFlowFarneback) to detect motion direction and speed.

A practical approach for installation interaction: participants hold colored objects (bright colored balls, scarves, or lights). The system tracks their positions via color filtering and maps each tracked object to a "brush stroke" on a virtual canvas. The virtual canvas is what the music pipeline actually scans. This way participants' physical movements paint the musical source image in real-time.

#### New Module: src/tracking.rs

```rust
pub struct TrackedObject {
    pub id: usize,
    pub position: (f32, f32),   // normalized 0-1
    pub velocity: (f32, f32),   // pixels/frame
    pub color_hue: f32,         // dominant hue of the object
    pub area: f32,              // relative size
}

pub trait ObjectTracker {
    fn update(&mut self, frame: &Mat) -> Vec<TrackedObject>;
}

pub struct ColorTracker {
    pub hue_ranges: Vec<(f32, f32)>,  // HSV hue ranges to track
    pub min_area: f32,
    pub max_objects: usize,
}
```

The tracker produces TrackedObjects that the engine converts to InteractionEvents. Object position maps to brush position on the virtual canvas. Object velocity maps to brush stroke direction and intensity. Object color maps to paint color (which then maps to musical mode via hue-to-mode).

#### Complexity: HIGH

Camera integration is medium (OpenCV already supports it), but robust object tracking in variable lighting conditions is hard. Background subtraction requires calibration. Multi-object tracking requires assignment between frames (a simple nearest-neighbor approach works for a small number of objects). Installation environments have unpredictable lighting, reflections, and occlusion. Budget significant time for calibration tooling and testing in realistic environments. Estimate 1-2 weeks including calibration tools.

#### New Dependencies

No new Rust crates needed beyond what's already in Cargo.toml. OpenCV provides all the computer vision functionality. If more sophisticated tracking is needed later (deep learning-based), the opencv crate supports DNN module for running ONNX models, but that's a future optimization.


## Layer 2: Game Integration

### Architecture: Event Bridge, Not Embedding

The cleanest approach is NOT to embed the game engine into AudioHax or AudioHax into the game. Instead, AudioHax runs as a separate process and communicates with the game over a local protocol. This has several advantages: the game can be built with any engine (Godot, Bevy, Love2D, a custom Rust engine), the game doesn't need to link against OpenCV/FluidSynth/etc., AudioHax can be developed and tested independently, and the protocol can also serve the art installation layer.

#### New Module: src/game_bridge.rs

Defines the communication protocol and message types.

```rust
/// Messages FROM the game TO AudioHax
pub enum GameEvent {
    /// Game sends a complete image frame for AudioHax to scan
    ImageFrame { data: Vec<u8>, width: u32, height: u32, format: PixelFormat },

    /// Game modifies a region of the current image
    ImagePatch { x: u32, y: u32, width: u32, height: u32, data: Vec<u8> },

    /// Game directly modifies musical parameters (bypass image analysis)
    ParameterOverride {
        target: ParameterTarget,  // Mode, Tempo, Complexity, Velocity, etc.
        value: f32,
        duration_ms: u32,         // transition time
    },

    /// Game signals a structural event
    StructuralEvent {
        kind: StructuralEventKind, // LevelTransition, BossFight, Death, Victory, etc.
    },

    /// Game requests current musical state (for bidirectional integration)
    QueryState,
}

/// Messages FROM AudioHax TO the game
pub enum AudioHaxResponse {
    /// Current musical state snapshot
    StateSnapshot {
        current_mode: String,
        current_chord: String,
        beat_position: f32,
        intensity: f32,           // 0-1, derived from dynamic contour
        scan_position: f32,       // 0-1, position in scan
    },

    /// Beat/phrase event for game to sync to
    BeatEvent { beat_number: u32, phrase_position: f32 },
}
```

#### Transport: Local UDP or WebSocket

For game integration, latency matters more than reliability. A local UDP socket (127.0.0.1:port) gives the lowest latency and simplest implementation. Messages are serialized as simple binary or JSON. Lost messages are acceptable — the next frame's state update overwrites the previous one.

For the installation layer (where the controller might be on a different machine on a LAN), WebSocket over a local network provides reliable delivery with reasonable latency. The websocket crate (tungstenite) is lightweight and well-maintained.

Recommended approach: implement UDP first (simpler, faster), add WebSocket as an alternative transport later. Both deserialize into the same GameEvent/AudioHaxResponse types.

The game_bridge module runs a listener in a dedicated thread. Incoming GameEvents are forwarded to the engine via the same channel that mouse and camera events use. The engine doesn't care whether an image change came from a mouse brush, a camera frame, or a game sending an ImagePatch — it's all just image buffer updates.

#### Interaction Patterns

There are three levels of game-audio integration, from simple to deep:

Level 1 — Image-mediated: The game sends complete image frames or patches. AudioHax treats them like any other image source. The game controls the music indirectly by controlling what the image looks like. This is the simplest to implement and the most aligned with AudioHax's existing architecture. A 2D game could literally send its rendered framebuffer as the music source — the game's visual aesthetics directly become the music's tonal palette.

Level 2 — Parameter override: The game sends direct parameter changes (switch to Aeolian mode, increase harmonic complexity, slow the tempo). AudioHax applies these as overrides that take precedence over image-derived parameters. This gives the game designer precise musical control for key moments (boss fight = Phrygian + fast tempo + high complexity) while letting the image-based system handle the texture.

Level 3 — Bidirectional: AudioHax sends beat and phrase events back to the game, and the game synchronizes visual events to the musical pulse. Enemy spawns happen on downbeats. Level transitions happen at phrase boundaries. Player animations sync to the tempo. This creates the deep music-game integration that makes games feel rhythmically alive. Implementation requires AudioHax to expose a predictable beat clock, which means the phrase structure from Phase 1 becomes critical infrastructure — the game needs to know when the next strong beat is coming.

#### StructuralEvent Handling

When the game signals a major structural event (level transition, boss fight), AudioHax should respond with a coherent musical transition rather than an abrupt change. Implementation approach:

A StructuralEvent triggers a transition plan in the engine: over the next N steps (configurable per event type), smoothly shift parameters from current values to target values. A level transition might trigger a 4-step ritardando followed by a 2-step silence followed by the new level's musical character ramping in over 4 steps. A boss fight might trigger an immediate switch to a minor mode with high harmonic complexity and fast tempo, with a secondary dominant approach chord as the transition point. A player death could trigger a fermata (hold the current chord for an extended duration), then resolve to a somber cadence.

These transition plans are essentially short compositional algorithms that the chord engine executes. They're a natural extension of the Phase 1 phrase structure work — a StructuralEvent is just a forced phrase boundary with a specific cadential target.

#### New Dependencies

For UDP transport: the std::net module covers everything (UdpSocket). No new crate needed. For WebSocket (future): tungstenite (~lightweight, pure Rust) or tokio-tungstenite if async is adopted. For message serialization: serde_json (already in Cargo.toml) for JSON messages, or bincode for binary (more compact, faster).

#### Complexity: HIGH

The protocol design and UDP transport are medium complexity. The real difficulty is in the musical transition system for structural events — making mode changes, tempo shifts, and intensity ramps sound musical rather than mechanical requires careful compositional logic in the chord engine. The bidirectional beat-clock export requires the phrase manager to be predictive (the game needs to know where beat N+1 will be before it happens, which means the engine needs a stable tempo model). Estimate 2-3 weeks for the full game bridge with transition system.


## Layer 3: Art Installation

### Architecture: Configuration Layer Over Layers 1 + 2

The installation is not a new architectural layer — it's a deployment configuration that combines Layer 1B (camera tracking) with Layer 2's network protocol (for remote control and monitoring) and adds installation-specific concerns: reliability, multi-participant support, and operational tooling.

#### Multi-Participant Interaction

Multiple visitors interacting simultaneously is handled by the tracking system. Each TrackedObject from the camera tracker is an independent "brush" painting on the virtual canvas. The music responds to the aggregate state of all tracked objects: more participants create more visual activity, which maps to higher edge density, more color variety (potentially multiple simultaneous modes fighting for dominance), and more rhythmic complexity. When participants interact near each other, their visual contributions overlap, creating blended musical features.

The musical question is: how does the system handle multiple competing key centers? If one participant is painting warm reds (Phrygian) and another is painting cool blues (Aeolian), the global hue average might land on green (Ionian) — which is neither participant's "contribution." A more sophisticated approach: divide the scan bar into zones, each influenced by the nearest tracked object. Each zone could have its own modal color, creating a kind of spatial polymodality. This is musically adventurous but architecturally straightforward — the scan-bar feature extraction already divides the image into per-instrument sections, so different instruments could reflect different participants' contributions.

#### New Module: src/installation.rs

```rust
pub struct InstallationConfig {
    pub camera_index: i32,
    pub tracking_mode: TrackingMode,          // ColorTracker, BackgroundSubtraction, etc.
    pub virtual_canvas_size: (u32, u32),      // internal resolution
    pub projection_output: Option<DisplayConfig>, // for projector feedback
    pub network_control_port: u16,            // for remote monitoring/override
    pub auto_restart: bool,                   // restart on crash
    pub idle_timeout_secs: u32,               // return to attract mode after inactivity
    pub attract_mode_image: Option<String>,   // image to scan when no participants
    pub max_participants: usize,
    pub participant_fade_rate: f32,            // how quickly brush strokes fade
}
```

#### Attract Mode

When no participants are detected, the installation should still produce ambient music from a default image (or a slowly evolving generative image). This is the "attract mode" that draws visitors in. When the tracking system detects a new participant, the virtual canvas begins fading from the attract image toward a blank canvas that the participant's movements paint on. When all participants leave, the canvas slowly fades back to the attract image over 30-60 seconds.

The fade mechanic uses OpenCV's addWeighted() between the attract image and the participant-painted canvas, with the blend ratio controlled by a timer since the last tracked object was active.

#### Projection Feedback

For maximum installation impact, the virtual canvas (with scan bar overlay and tracked object positions) should be projected back into the physical space. This creates a feedback loop: participants see their visual contributions projected on a surface, see the scan bar moving across them, and hear the resulting music. The projection display is just another OpenCV imshow window on a display output configured for the projector. No new code needed beyond routing the overlay image to the correct display.

#### Reliability

An installation must run unattended for hours or days. The engine needs:

A watchdog that detects if the pipeline stalls (no tick has executed in >2x the expected interval). If stalled, log the error, attempt to restart the pipeline. If restart fails, restart the entire process. On Linux, systemd with Restart=always handles process-level recovery. On Windows, a wrapper script or Task Scheduler achieves the same.

Camera reconnection: if the camera feed drops (USB disconnect, driver crash), the camera source thread should detect this (VideoCapture::read returns false), back off with exponential retry, and reconnect when the camera reappears. During disconnection, the engine falls back to the last captured frame or the attract mode image.

Memory management: OpenCV Mat objects can accumulate if not properly released. The camera thread should overwrite the shared buffer in-place rather than allocating new Mats per frame. Profile memory usage over multi-hour runs.

#### Network Monitoring

The installation should expose a simple status endpoint so a curator or technician can monitor it remotely. The game_bridge's WebSocket transport serves double duty here: a monitoring client connects and receives periodic StateSnapshots plus alerts (camera disconnected, MIDI port lost, etc.). A web-based dashboard (a simple HTML page served by a tiny embedded HTTP server or a separate script) displays the status. The minreq crate or a small embedded server via tiny_http would handle this without pulling in a large web framework.

#### Complexity: MEDIUM (given Layers 1 and 2 already exist)

If the camera tracking and game bridge are already built, the installation layer is primarily configuration, operational tooling, and reliability hardening. The multi-participant musical challenge (spatial polymodality) is the most architecturally interesting piece. Estimate 1-2 weeks on top of completed Layers 1 and 2.


## Crate Summary

What's already in Cargo.toml and sufficient for all layers: opencv (camera, tracking, display, image manipulation), midir (MIDI output), serde/serde_json (serialization), rand (random number generation), anyhow (error handling).

Likely additions:

```
crossbeam-channel    # multi-producer channels for event routing (better than std mpsc for this use case)
bincode              # compact binary serialization for game bridge UDP messages
tiny_http            # embedded HTTP server for installation monitoring (optional)
tungstenite          # WebSocket for game bridge / installation remote control (optional, Layer 2+)
```

None of these are heavy dependencies. crossbeam-channel is the most important addition because the reactive engine needs a clean way to receive events from multiple sources (mouse callback thread, camera thread, game bridge listener thread) into a single engine loop. Rust's std::sync::mpsc works but crossbeam's select! macro makes multi-source receiving much cleaner.


## Refactoring Phases and Agent Team Decomposition

The work naturally decomposes into 4 phases with clear boundaries. Each phase depends on the previous one but has internally parallelizable tasks.

### Phase R (Reactive Refactor) — Prerequisite for everything

This is the most architecturally sensitive phase. It changes the core control flow without changing any musical behavior. The output should sound identical to the current batch mode when given a static image.

Task R.1: Extract PipelineEngine from main.rs (engine.rs). Move the pipeline loop, feature extraction calls, chord generation, and MIDI scheduling into a struct with a tick() method. main.rs becomes a thin CLI wrapper that constructs the engine and calls tick() in a loop.

Task R.2: Add SharedImageBuffer with Arc<RwLock<Mat>>. Replace the single img variable with the shared buffer. The engine reads from it; input sources write to it.

Task R.3: Add change detection / hysteresis for global features. Implement the delta-threshold system so the engine doesn't re-derive mode/chords on every tick.

Task R.4: Regression test — verify batch mode produces identical scan behavior. Run the current batch pipeline on a test image, capture the sequence of InstrumentActions. Run the new reactive pipeline on the same static image, capture the same sequence. They must match.

File ownership: engine.rs (new), main.rs (refactored). Does not touch chord_engine, image_analysis, mapping_loader, midi_output, or modem.

Agent team split: This is best done as a single focused agent (or by you personally) because the refactor touches main.rs's core control flow, which requires holistic understanding. A 2-agent split with one on engine.rs extraction and one on main.rs slim-down risks merge conflicts in the boundary between them. If using agents: one implementation agent for the refactor, one quality gate for regression testing.

### Phase I1 (Interactive Layer 1A — Mouse Painting)

Task I1.1: Create src/interaction.rs with InteractionEvent types, BrushMode, and brush application functions (smear, paint, blur, erase).

Task I1.2: Wire OpenCV mouse callback into the event loop. The callback sends InteractionEvents through a crossbeam channel to the engine.

Task I1.3: Add "interactive" mode to main.rs CLI that starts the reactive engine with mouse input enabled.

Task I1.4: Display overlay showing scan bar position, current mode/chord, and brush cursor on the image window.

File ownership: interaction.rs (new), main.rs (CLI addition), image_source.rs (minor — no new image loading, just window management).

Agent team split: Two implementation agents (one for interaction.rs brush logic, one for main.rs event loop / display wiring) plus quality gate. The brush logic agent can work in isolation since it operates on Mat inputs/outputs with no engine dependency.

### Phase I1B (Interactive Layer 1B — Camera Tracking)

Task I1B.1: Add continuous camera capture to image_source.rs (CameraSource::start_continuous).

Task I1B.2: Create src/tracking.rs with ColorTracker implementing the ObjectTracker trait.

Task I1B.3: Add "camera" mode to main.rs CLI. Camera frames flow into the shared image buffer, optionally through the tracker which converts objects to virtual canvas brush strokes.

Task I1B.4: Calibration tooling — a mode that displays the camera feed with tracked object overlays and lets the user adjust HSV ranges, minimum areas, and other tracker parameters interactively.

File ownership: image_source.rs (camera thread), tracking.rs (new), main.rs (CLI), and a calibration binary or mode.

Agent team split: Two implementation agents (camera source + tracking logic, calibration tooling) plus quality gate. The camera work is very OpenCV-specific and benefits from focused expertise.

### Phase I2 (Game Integration)

Task I2.1: Create src/game_bridge.rs with GameEvent/AudioHaxResponse types and UDP listener.

Task I2.2: Wire the listener into the engine's event channel. ImageFrame events update the shared image buffer. ParameterOverride events set engine override flags. StructuralEvent events trigger transition plans.

Task I2.3: Implement musical transition plans in chord_engine.rs — smooth modulation logic for mode changes, intensity ramps, and structural cadences.

Task I2.4: Implement the bidirectional beat clock. The engine publishes BeatEvent messages to connected game clients at musically meaningful moments (downbeats, phrase boundaries).

Task I2.5: Test harness — a simple Python or Rust script that sends GameEvents over UDP and verifies AudioHaxResponse messages, without needing an actual game.

File ownership: game_bridge.rs (new), chord_engine.rs (transition plans), engine.rs (event handling extension), and a test harness script.

Agent team split: Three implementation agents (protocol/transport, musical transition logic in chord_engine, beat clock + test harness) plus quality gate. The chord_engine transition work is a direct extension of Phase 1 music quality work — the agent needs music theory context.

### Phase I3 (Installation)

Task I3.1: Create src/installation.rs with InstallationConfig, attract mode logic, and participant fade system.

Task I3.2: Add camera reconnection with exponential backoff to image_source.rs.

Task I3.3: Add network monitoring endpoint (HTTP status page or WebSocket state broadcast).

Task I3.4: Create deployment documentation — systemd unit file, hardware requirements, calibration procedure, troubleshooting guide.

Task I3.5: Multi-hour soak test — run the system with simulated camera input and game events for 4+ hours, monitoring memory usage, MIDI port health, and event processing latency.

File ownership: installation.rs (new), image_source.rs (reconnection), game_bridge.rs (monitoring), docs/ (deployment docs).

Agent team split: Two implementation agents (installation logic + monitoring, reliability + deployment docs) plus quality gate focused on the soak test.


## Dependency Chain Summary

```
Phase 1 Music Quality (completed)
         │
         ▼
Phase R: Reactive Refactor ◄──── prerequisite for ALL interactive work
         │
    ┌────┴────┐
    ▼         ▼
Phase I1A   Phase I2 (can start game_bridge protocol
(Mouse)     design in parallel with I1A, but engine
    │        integration needs Phase R)
    ▼
Phase I1B
(Camera)
    │
    ▼
Phase I3 (Installation — needs I1B camera + I2 network)
```

Phase I1A and the protocol-design portion of Phase I2 can proceed in parallel after Phase R, since they add to different modules (interaction.rs vs game_bridge.rs). But the engine integration portions of both depend on the reactive refactor being stable.


## Key Risk: The Reactive Refactor

Phase R is the riskiest work in this entire plan. It changes the control flow that everything depends on. If the refactor introduces timing bugs (MIDI events arriving out of order, scan position advancing erratically) or concurrency issues (deadlocks between the image writer and reader, race conditions in change detection), every subsequent phase inherits those bugs.

Mitigation: the regression test (Task R.4) is the critical safety net. Before declaring Phase R complete, the batch mode must produce bit-identical musical output on a reference image. If it doesn't, the refactor has changed musical behavior, and that must be debugged before moving on. This is worth spending extra time on — a solid reactive core makes everything else straightforward, and a buggy one makes everything else a nightmare.

Second risk: performance. The batch pipeline precomputes everything, so playback is just scheduling pre-built events. The reactive pipeline computes features on every tick. At 250ms per step with 4-8 instruments, this is almost certainly fine (image analysis on a small region takes microseconds). But at higher tick rates (needed for responsive interaction — 60fps camera input implies potential 60Hz tick rate), you'll need to profile. The most likely bottleneck is global feature extraction (whole-image histogram and Laplacian), which is why the hysteresis / rate-limiting system in Task R.3 is essential.
