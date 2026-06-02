# plan.md — Mr. Gate Development Roadmap

This is the phased plan for building Mr. Gate. Reference this file in Claude Code prompts via `@plan.md`. Each phase ends with explicit "definition of done" criteria. Do not advance phases until criteria are met.

## Phase 0 — Project Setup

**Goal:** working scaffold that loads in Reaper.

- [x] Create `mrgate.jsfx` with `desc:`, `tags:`, basic stereo I/O.
- [x] Add placeholder sliders for the eight dynamics controls plus mode and style sliders.
- [x] Empty `@init`, `@slider`, `@sample` (passthrough).
- [ ] Verify plugin loads in Reaper and audio passes through unchanged.

**Done when:** plugin loads in Reaper, all sliders visible (default UI is fine), audio passes through without artifacts.

---

## Phase 1 — DSP Core, Downward Direction (default UI)

**Goal:** functional gate/downward-expander using default JSFX sliders. No upward mode yet, no mode toggle yet, no custom UI. Just the DSP engine working in the downward direction.

- [x] Implement peak envelope follower with light RMS smoothing in `@sample`.
- [x] Implement gain computation: threshold, ratio, range, knee.
- [x] Implement attack/release smoothing (asymmetric exponential).
- [x] Implement hold counter logic.
- [x] Apply final gain to audio.
- [x] Verify stereo channel link (max envelope across L/R).
- [ ] Test by ear on a noisy vocal track and a drum loop.

**Done when:** plugin gates audio audibly correctly in downward direction. Threshold/Ratio/Range/Knee/Attack/Release/Hold all behave as documented in `spec.md`. Sounds clean — no chatter, no obvious pumping. CPU usage is minimal.

---

## Phase 2 — Mode Toggle and Upward Direction

**Goal:** add the three-way mode toggle and the upward expansion path.

- [x] Add mode slider (0 = Gate, 1 = Downward, 2 = Upward).
- [x] Refactor gain computation to use a `direction` sign (+1 for downward, -1 for upward) — single code path.
- [x] Implement mode-dependent default loading (when mode changes, reload defaults per the table in `spec.md`).
- [ ] Verify all three modes audibly do the right thing — Gate kills, Downward gently reduces, Upward boosts loud parts.
- [ ] Confirm no DSP regressions in Gate or Downward modes after refactor.

**Done when:** user can switch between three modes via the mode slider. Defaults reload sensibly. Each mode sounds right by ear. DSP is one shared code path with mode affecting only direction sign and coefficients.

---

## Phase 3 — Lookahead

**Goal:** zero-to-10ms lookahead with correct latency reporting.

- [x] Add a delay buffer for the audio path.
- [x] Detector reads the present sample; audio output reads from the delayed buffer.
- [x] Report latency to Reaper via `pdc_delay`.
- [x] Handle the case where lookahead = 0 (no latency reported).
- [ ] Test PDC by inserting on a track alongside other plugins and confirming alignment.

**Done when:** lookahead non-zero, transients are gated cleanly without "clipped" attack. PDC works correctly.

---

## Phase 4a — Layout Skeleton + Static Knob Drawing

**Goal:** `@gfx` renders the complete layout as static art — no mouse interaction yet. All pixel constants, colors, angles, and positions are pre-decided in `layout.md`. Translate spec to code; make no design decisions.

- [ ] Add `gfx_w = 700; gfx_h = 420;` to `@init`. Define all `layout.md` constants: `KNOB_R`, `ANG_MIN`, `ANG_SWEEP`, `KNOB_L_X`, `KNOB_R_X`, 4 Y positions, slider min/max tables, and defaults table at address 6200.
- [ ] Set fonts 1–3 in `@init` via `gfx_setfont` (per `layout.md`).
- [ ] Implement `function ui_draw_knob(cx, cy, r, t)` — body circle, track arc, fill arc, indicator dot.
- [ ] Draw top bar (background, all three mode buttons, style dropdown). Active mode button (per `slider1`) uses COL_BTN_ACTIVE; others use COL_BTN_INACTIVE.
- [ ] Draw display placeholder rect (center area, `layout.md` dimensions).
- [ ] Draw all 8 knobs using each slider's current value normalized per `layout.md` normalization table.
- [ ] Draw knob labels (hard-coded strings, centered `KNOB_R + 5` below each knob center).
- [ ] Draw bottom bar (display toggle button, GR placeholder).

**Done when:** plugin renders the full layout in Reaper. Active mode button highlighted. All labels visible. No interaction. No DSP regressions.

---

## Phase 4b — Mouse Interaction

**Goal:** all knobs respond to mouse drag, scroll, ctrl+click, and double-click. Reference `layout.md` for all sensitivity values and state-variable names.

- [ ] Add to `@init`: `ui_drag_knob = -1`, `ui_drag_start_y`, `ui_drag_start_v`, `ui_prev_lmb`, `ui_last_click_t`, `ui_disp_enabled = 1`. Add `SLIDER_MIN_TABLE` and `SLIDER_MAX_TABLE` arrays (8 entries each) and `DEFAULT_TABLE` (24 entries) per `layout.md`.
- [ ] Knob hit test: circular check, 1.5× visual radius hit target.
- [ ] Drag-to-adjust: 200 px = full range. Shift held = 10× slower. Write result to correct slider, call `sliderchange()`.
- [ ] Ctrl+click or double-click (< 0.4 s between clicks): reset to `DEFAULT_TABLE[mode_id * 8 + knob_index]`, call `sliderchange()`.
- [ ] Scroll-wheel nudge: ±0.5% of range per tick.

**Done when:** all 8 knobs adjustable. Audible in real time. Shift slows, ctrl resets, scroll nudges. No DSP regressions.

---

## Phase 4c — Mode Buttons + Style Dropdown Interaction

**Goal:** clicking the mode row and style dropdown updates plugin state.

- [ ] On LMB click on a mode button: write `slider1 = mode_index`, call `sliderchange()`. Existing `@slider` logic fires the defaults reload.
- [ ] On LMB click on style dropdown: no-op (single option).

**Done when:** clicking a mode button switches mode, reloads defaults, and the correct button highlights. No DSP regressions.

---

## Phase 4d — Value Readouts + Display Toggle

**Goal:** numeric value labels under each knob, and the display toggle button works.

- [ ] Draw formatted value string `KNOB_R + 19` below each knob center. Format strings per `layout.md` value-readout table.
- [ ] On LMB click on display toggle button: `ui_disp_enabled = 1 - ui_disp_enabled`. Update button visual (COL_BTN_ACTIVE when on, COL_BTN_INACTIVE when off).
- [ ] When `ui_disp_enabled = 0`: skip drawing the center placeholder rect.

**Done when:** all knobs show current value. Display toggle button updates visually on click. No DSP regressions.

---

## Phase 5 — Transfer Curve Display

**Goal:** the static transfer curve overlay with live input dot.

- [ ] Compute curve points from threshold/ratio/range/knee/direction in `@slider` (cache them).
- [ ] Draw axes and dB labels.
- [ ] Draw the curve as a polyline. Geometry flips for upward mode.
- [ ] Draw threshold/range reference lines (dashed).
- [ ] Pass current input level from `@sample` to `@gfx` via a shared variable.
- [ ] Draw the live input dot at the correct position on the curve, updating at UI rate.
- [ ] Make threshold/range/ratio/knee draggable directly on the curve.

**Done when:** curve updates in real time when knobs change. Curve geometry is correct in all three modes. Dragging the curve adjusts the underlying parameters. Input dot tracks the audio level visibly.

---

## Phase 6 — Scrolling Level History

**Goal:** the real-time scrolling input/output level graph.

- [ ] Allocate ring buffers for input and output level history in `@init`.
- [ ] Push input and output peak values into the buffers in `@sample` (downsampled — once per ~10ms is plenty).
- [ ] Draw the history as filled regions in `@gfx` — input dark, output light.
- [ ] Verify the display toggle disables drawing (and ideally also pauses buffer updates to save CPU).

**Done when:** scrolling display updates smoothly at the UI refresh rate. Visual reads well at a glance. Disable toggle works and demonstrably reduces CPU load.

---

## Phase 7 — Polish

**Goal:** the difference between "works" and "feels good."

- [ ] Hover states on all interactive elements.
- [ ] Smooth value interpolation for displayed numeric values (avoid flickering).
- [ ] Consistent color palette and typography across all UI.
- [ ] Test at multiple Reaper UI scales and screen DPIs.
- [ ] Final by-ear pass on real mix material across all three modes.
- [ ] Write a short README for the plugin.

**Done when:** the user has been using the plugin in real mixes and is satisfied with both sound and feel.

---

## Phase 8 — Sidechain Highpass Filter

**Goal:** one-pole highpass on the detector signal, controlled by a single slider. Audio path untouched.

- [x] Add `slider11` for SC HP frequency (20–500 Hz, default 20).
- [x] Declare `sc_hp_coeff`, `sc_hp_state_l`, `sc_hp_state_r` variables in `@init`.
- [x] Pre-compute `sc_hp_coeff` in `@slider` (0 when slider ≤ 20.5 Hz = bypassed).
- [x] Apply first-order highpass to raw audio in `@sample` before rectification. Filter operates on spl0/spl1 directly; `det = max(abs(hp_l), abs(hp_r))`. Bypassed when coeff = 0. Note: spec plan said to filter `env_det` — corrected to filter raw audio before rectification, which is the proper sidechain HP approach.
- [x] Add SC HP knob to the UI (bottom center, radius 14, label + Hz readout).
- [x] DEFAULT_TABLE updated to 9 params per mode (SC HP = 20 for all modes).
- [ ] Test: set HP to 200 Hz, play a kick-heavy mix through a gate on a snare channel. Gate should ignore the kick energy in the detector without affecting the audio.

**Done when:** SC HP slider audibly prevents low-frequency energy from triggering the gate. Audio output at 20 Hz setting is bit-identical to having no filter. No DSP regressions.

---

## Phase 9 — External Sidechain

**Goal:** route Reaper channels 3/4 into the detector. Toggle between internal and external modes in the UI.

- [ ] Add `in_pin:sidechain left` and `in_pin:sidechain right` declarations.
- [ ] Add `slider12` for sidechain source (0 = internal, 1 = external).
- [ ] In `@sample`: when external mode, `det = max(abs(spl2), abs(spl3))` instead of main input.
- [ ] SC HP filter applies to whichever source is active (no change to the filter logic itself).
- [ ] Add sidechain source toggle button to the UI.
- [ ] When external sidechain is active, scrolling display shows the external signal level instead of main input, so the user can see the trigger signal vs. gain reduction relationship.
- [ ] Test: route a kick drum to the sidechain, put a sustained pad through the main input. Gate should duck on kick hits.

**Done when:** external sidechain routes correctly. Scrolling display switches to show external signal level. No DSP regressions in internal mode.

---

## Deferred

Captured here so they don't get lost. Move to a phase when ready.

- Additional styles (Vocal, Drums, Guitar, Ducking, etc.) — populate the Style dropdown.
- Mid-side processing.
- Wet/Dry mix.
- Oversampling.
- MIDI trigger.
- Tooltips / help system.
- Workflow features unique to Mr. Gate: histogram overlay, learn-threshold, freeze-GR, etc.
