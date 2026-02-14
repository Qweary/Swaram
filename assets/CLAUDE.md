# assets/ — Configuration and Input Data

## mappings.json — Visual-to-Musical Parameter Rules

This is the central configuration for how visual features map to musical parameters. Three sections:

"global": Whole-image feature mappings. hue_to_mode (hue ranges → mode names like "Ionian", "Dorian"), saturation_to_harmonic_complexity (→ TriadsOnly / TriadsAnd7ths / Triads7thsAndExtensions), brightness_to_tempo_bpm, progression_families (warm/cool/neutral → chord progression strings like "I-vi-IV-V"), dominant_substitution_trigger, modal_interchange_trigger, cadence_trigger.

"instrument_section": Per-instrument mappings. edge_density_to_rhythm (→ HalfNotes/QuarterNotes/EighthNotes), line_orientation_to_interval, contrast_to_articulation (→ Legato/Portato/Staccato), color_shift_to_chord_extension, texture_to_modal_color.

"fine_detail": Local-region mappings. pixel_y_position_to_pitch, pixel_brightness_to_velocity, local_jaggedness_to_chromaticism, shape_to_ostinato.

## How to Extend mappings.json

All new mapping fields must also be added to the MappingTable struct in src/mapping_loader.rs. Use #[serde(default)] on new fields so the existing mappings.json parses without error before the JSON is updated. Range map keys use "low-high" format (e.g., "0-30"). Test that both the old and new JSON parse correctly.

Never hardcode musical values in source files — all visual→musical translations go through this file.

## images/ — Input Images

Contains source images for the music pipeline. Any common image format works (JPEG, PNG). The default is images/example.jpg. For testing, use solid-color images to isolate individual feature mappings (e.g., pure red → Phrygian mode, pure blue → Aeolian mode). High-contrast images with strong edges test the arpeggio trigger. Gradients test smooth feature transitions.

## Image Guidelines for Testing

A good test suite includes: solid colors (one per mode), gradients (horizontal for temporal evolution, vertical for instrument differentiation), high-contrast patterns (sharp edges for arpeggio behavior), photographs (complex real-world features), and a blank white image (degenerate case). Keep test images small (640x480 or less) for fast CI.
