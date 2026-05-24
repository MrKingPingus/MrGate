# spec.md — Mr. Gate Feature Specification

This document is the source of truth for what Mr. Gate does. It is locked down — changes require explicit user approval and an update to this file before implementation.

## Scope

Mr. Gate is a Reaper JSFX dynamics plugin with three modes (Gate, Downward Expander, Upward Expander), a single envelope algorithm called "General," and a custom UI focused on rich visual feedback. CPU efficiency is an explicit goal. Sound quality is judged by the user's ear on real material — there is no comparison target.

## In Scope (v1)

### Mode toggle (top-level)

Three-button row at the top of the plugin:

```
[ Gate ] [ Downward ] [ Upward ]
```

Selecting a mode:
- Reloads default values for all dynamics controls (Threshold, Ratio, Range, Knee, Attack, Release, Hold) — see defaults table below.
- Reconfigures slider display ranges where they differ between modes.
- Flips the gain computation direction (downward modes reduce below threshold; upward mode boosts above threshold).
- Updates the transfer curve display geometry.

The DSP code path is shared across modes. Only the direction sign and coefficient values change.

### Dynamics controls

| Control | Range | Notes |
|---|---|---|
| Threshold | -80 to 0 dB | Same range across all modes. |
| Ratio | 1:1 to 30:1 | UI displays full range in Gate mode; capped at ~10:1 visually in Expander modes. Internal value uncapped. |
| Range | 0 to 60 dB | "Maximum gain change" — caps how far gain can deviate from unity. Same internal range; defaults differ per mode. |
| Knee | 0 to 24 dB | Soft knee width centered on threshold. Symmetric in upward mode. |
| Attack | 0.05 to 100 ms | Time for gain to fully open (downward) or fully boost (upward). |
| Release | 5 to 5000 ms | Time for gain to fully close (downward) or fully return to unity (upward). |
| Hold | 0 to 250 ms | Minimum time gain stays at its active value before release begins. |
| Lookahead | 0 to 10 ms | When non-zero, plugin reports latency to Reaper via `pdc_delay`. Default 0 (zero-latency). |

### Defaults per mode

| Control | Gate | Downward | Upward |
|---|---|---|---|
| Threshold | -40 dB | -50 dB | -20 dB |
| Ratio | 10:1 | 2:1 | 2:1 |
| Range | 40 dB | 12 dB | 6 dB |
| Knee | 3 dB | 9 dB | 6 dB |
| Attack | 1 ms | 10 ms | 5 ms |
| Release | 100 ms | 250 ms | 200 ms |
| Hold | 10 ms | 0 ms | 0 ms |
| Lookahead | 0 ms | 0 ms | 0 ms |

### Style dropdown

A single dropdown labeled "Style" with one option: **General**.

The dropdown exists in v1 for architectural and visual reasons (so the UI doesn't change shape later when more styles are added). Styles like Vocal, Drums, etc., are deferred.

### "General" algorithm

The single algorithm used in v1. Specification:

- **Detection:** peak detection with light RMS-style smoothing (~5ms window) to avoid sample-level chatter. Stereo link via max(|L|, |R|).
- **Envelope smoothing:** asymmetric. Attack uses the user's attack time as the time-to-90%-target. Release uses the user's release time as the time-to-90%-target. Standard exponential one-pole filters with coefficients computed in `@slider`.
- **Knee:** soft-knee implementation that smoothly interpolates the gain curve through the threshold region. Symmetric in upward mode.
- **Hold:** counter-based, prevents release from beginning until hold time elapses after the last threshold crossing.

### Routing

- Stereo only for v1. Channel link uses max envelope of L/R.
- Mid-side and external sidechain are out of scope for v1.

### Visual feedback (the centerpiece)

- **Real-time scrolling level history.** Input level (dark blue) overlaid with output level (light blue), gain reduction visible as the gap. ~3 second window. 60 dB scale. Updates at UI refresh rate.
- **Static transfer curve overlay.** Drawn on top of (or alongside) the scrolling display. Shows the input→output mapping based on current Threshold, Ratio, Range, Knee. Curve geometry flips for Upward mode. Threshold reference lines shown dashed.
- **Live input dot.** A small marker that traces the current input level along the transfer curve in real time. Updates at UI refresh rate.
- **I/O level meters.** Vertical or horizontal meters on the sides of the display. Show peak input, peak output, and instantaneous gain reduction.
- **Display toggle.** Button to disable the scrolling history display (saves CPU and reduces visual distraction).

### Workflow controls

- Mode toggle: `[ Gate | Downward | Upward ]` (described above).
- Style dropdown: `[ General ]` (described above).
- Display toggle: enables/disables the scrolling level history.

## Out of Scope (v1)

These are explicitly NOT in v1. Do not implement them. They may be revisited later.

- Additional styles (Vocal, Drums, Guitar, Ducking, etc.).
- Mid-side processing.
- External sidechain input.
- Sidechain EQ / filtering.
- Wet/Dry mix control.
- Oversampling (any kind).
- MIDI trigger / MIDI Learn.
- A/B compare buttons. (User can use Reaper's built-in undo and snapshot system.)
- Custom preset system. (User can use Reaper's JSFX preset system.)
- Resizable UI / Full-screen mode.
- Tooltips / help system.
- Histogram of input levels overlaid on curve.
- "Learn threshold" auto-set feature.
- Freeze gain reduction button.

## UI Layout (structural)

A single fixed-aspect window. Approximate proportions:

```
+------------------------------------------------------------------+
|  [Gate]  [Downward]  [Upward]              Style: [General ▾]    |
|                                                                  |
|  Threshold                                       Attack          |
|  Ratio                                           Release         |
|  Range       |---- Scrolling Level Display ----| Hold            |
|  Knee        |     + Transfer Curve            | Lookahead       |
|              |     + Live Input Dot            |                 |
|              +----------------------------------+                |
|                                                                  |
|  [In Meter]                                          [Out Meter] |
|                                                                  |
|  [Display Toggle]                              [Gain Reduction]  |
+------------------------------------------------------------------+
```

Specific pixel values, colors, and visual treatment will be decided in the UI phases. The above is structural only.

## Sound Design Targets (subjective)

Mr. Gate is judged by ear on real material. No null-test targets, no reference renders. The user's targets:

1. **Gate mode** should cleanly silence noise and bleed without audible chatter or pumping. Should be usable on drums (fast attack, transient preservation) and vocals (gentle close on breaths).
2. **Downward Expander mode** should subtly reduce noise without obvious gating artifacts. Should sound transparent at modest settings (2:1, 12 dB range).
3. **Upward Expander mode** should restore punch to over-compressed material without sounding obviously "expanded." Should not pump on sustained material.
4. **CPU efficiency** is an explicit goal across all modes. Mr. Gate should be insertable on every track of a busy session without measurable CPU impact.
