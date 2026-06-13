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

- Stereo only. Channel link uses max envelope of L/R.
- Mid-side is out of scope.
- External sidechain is supported via extra input pins (see Phase 9).

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

### Sidechain highpass filter (Phase 8)

A one-pole highpass filter on the **detector signal only** — the audio path is unaffected. Controlled by a single "SC HP" slider.

| Control | Range | Notes |
|---|---|---|
| SC HP | 20 to 500 Hz | Highpass cutoff applied to the detection envelope before threshold comparison. 20 Hz = effectively bypassed. |

- Filter is a one-pole IIR highpass (first-order Butterworth) applied to `env_det` in `@sample` using a coefficient pre-computed in `@slider`.
- Only the detector signal is filtered; the audio output path is completely unaffected.
- Useful for preventing low-frequency energy (kick bleed, rumble) from triggering the gate on mid/high sources.
- Default: 20 Hz (bypassed) across all modes.
- Visual feedback: the scrolling display and live dot continue to show the unfiltered input level (what's going into the mix), but the threshold line shows where the *filtered* detector is being compared. A second visual trace for the filtered detector level is a future option, not required in Phase 8.

### External sidechain (Phase 9)

Routes an external signal (Reaper channels 3/4) into the detector instead of the main input.

- Declared via additional `in_pin:` declarations: `in_pin:sidechain left` / `in_pin:sidechain right`.
- A "Sidechain" toggle button in the UI switches between internal (default) and external modes.
- When external is active: `det = max(abs(spl2), abs(spl3))` instead of `max(abs(spl0), abs(spl1))`.
- The SC HP filter applies to whichever signal source is active.
- Visual feedback: when external sidechain is active, the scrolling display shows the sidechain signal level (instead of main input) alongside the output level, so the user can see the relationship between the trigger signal and the gain reduction.

## Out of Scope

These are explicitly not planned. Do not implement them without explicit user approval and a spec update.

- Additional styles (Vocal, Drums, Guitar, Ducking, etc.).
- Mid-side processing.
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
|                                                                   |
|  +------ Main Display (620 × 312) --------+  ATTACK             |
|  |  scrolling level history (background)   |  [  1.0 ms ]        |
|  |  transfer curve (overlay)               |  RELEASE            |
|  |  threshold line — drag left/right       |  [ 100 ms ]         |
|  |  range cap line — drag up/down          |  HOLD               |
|  |  knee region   — drag to widen/narrow   |  [  10 ms ]         |
|  |  ratio slope   — drag to steepen/ease   |  LOOKAHEAD          |
|  |  live input dot on curve                |  [   0 ms ]         |
|  |  PK / RMS readouts (top-right)          |  SC HP              |
|  |  dB grid lines                          |  [  off   ]         |
|  +------------------------------------------+                    |
|                                                                   |
|  [Display ON]                              [GR  0.0 dB]         |
+------------------------------------------------------------------+
```

There are no knobs. Curve parameters (Threshold, Ratio, Range, Knee) are adjusted directly on the display. Time parameters (Attack, Release, Hold, Lookahead, SC HP) are adjusted via compact draggable value strips in the right panel. The JSFX default sliders remain available as a precise fallback.

### Display interaction

| Element | Interaction |
|---|---|
| Threshold line (vertical) | Drag left/right — maps x pixel to threshold_db |
| Range cap line (dashed diagonal) | Drag up/down — maps y delta to range_db |
| Knee region | Drag left/right within knee band — maps x delta to knee_db |
| Ratio slope | Drag up/down on active curve segment — maps y delta to ratio |
| Hover | Within ~8px of a draggable line: highlight it and show cursor feedback |

### Right panel value strips

Five stacked controls in a ~75px column to the right of the display: Attack, Release, Hold, Lookahead, SC HP. Each shows a centered label and the current value with units, and responds to vertical drag (up = increase, down = decrease). Shift = 10× finer. Mode-change resets apply as before.

## Sound Design Targets (subjective)

Mr. Gate is judged by ear on real material. No null-test targets, no reference renders. The user's targets:

1. **Gate mode** should cleanly silence noise and bleed without audible chatter or pumping. Should be usable on drums (fast attack, transient preservation) and vocals (gentle close on breaths).
2. **Downward Expander mode** should subtly reduce noise without obvious gating artifacts. Should sound transparent at modest settings (2:1, 12 dB range).
3. **Upward Expander mode** should restore punch to over-compressed material without sounding obviously "expanded." Should not pump on sustained material.
4. **CPU efficiency** is an explicit goal across all modes. Mr. Gate should be insertable on every track of a busy session without measurable CPU impact.
