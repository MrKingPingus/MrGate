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

## Phase 4a — Layout Skeleton (no knobs)

**Goal:** Replace knob columns with the new display-centric layout. Static drawing only — no mouse interaction yet.

- [x] Top bar: mode buttons + style dropdown (already working).
- [x] Display area: scrolling history + transfer curve (already working, Phases 5/6 done).
- [ ] Remove `ui_draw_knob` function and all knob drawing code from `@gfx`.
- [ ] Remove knob layout constants from `@init` (`KNOB_R`, `ANG_MIN`, `ANG_SWEEP`, `KNOB_L_X`, `KNOB_R_X`, `KNOB_Y0..Y3`).
- [ ] Expand display: `disp_x = 5`, `disp_w = 620` (was 440, reclaiming both former knob columns).
- [ ] Add right panel (x=630..695, y=48..360): draw 5 stacked value strips for Attack, Release, Hold, Lookahead, SC HP. Each strip: label centered at top, current value + unit centered below. Strip height = (disp_h / 5).
- [ ] Relocate SC HP knob drawing to the right panel strip (was bottom-center).
- [ ] Bottom bar: Display toggle + GR readout (already working).

**Done when:** plugin renders without knobs. Display is 620px wide. Right panel shows all 5 time controls as labeled value strips. No DSP regressions.

---

## Phase 4b — On-Display Parameter Interaction

**Goal:** Threshold, Ratio, Range, and Knee are draggable directly on the transfer curve display.

- [ ] Add mouse state variables to `@init`: `ui_mouse_cap_prev`, `ui_drag_param` (-1 = none), `ui_drag_start_x`, `ui_drag_start_y`, `ui_drag_start_v`.
- [ ] Each frame in `@gfx`: read `mouse_x`, `mouse_y`, `mouse_cap`. On LMB down, hit-test each draggable element (see below). On drag, compute delta and update slider + call `sliderchange()`. On LMB up, clear `ui_drag_param`.
- [ ] **Threshold line** (vertical at `_thr_x`): hit if `abs(mouse_x - _thr_x) < 8*ui_sx`. Drag maps x delta to threshold_db: `threshold_db = clamp(-60 * (mouse_x - disp_x) / disp_w, -80, 0)`.
- [ ] **Range cap line** (dashed diagonal): hit if mouse is within 8px of the line. Drag up/down: `range_db = clamp(range_db - dy * 60 / disp_h, 0, 60)`.
- [ ] **Knee region** (band around threshold): hit if `abs(mouse_x - _thr_x) < (knee_db/60)*disp_w/2 + 8`. Drag left/right: `knee_db = clamp(knee_db + dx * 48 / disp_w, 0, 24)`. Priority below threshold.
- [ ] **Ratio slope** (curve segment beyond knee): hit if mouse is on the active-slope portion and not already matched by threshold/knee. Drag up/down: `ratio = clamp(ratio - dy * 29 / disp_h, 1, 30)`.
- [ ] Hover highlight: when mouse is within hit distance of an element (and no drag is active), draw that element brighter.
- [ ] Shift key held: all delta computations scaled by 0.1× for fine adjustment.

**Done when:** all four curve parameters adjustable by dragging on the display. Changes are audible in real time. Hover highlights show before drag starts. No DSP regressions.

---

## Phase 4c — Right Panel Interaction

**Goal:** the five time-control strips respond to vertical drag.

- [ ] Add `ui_drag_strip` (-1 = none), `ui_drag_strip_start_y`, `ui_drag_strip_start_v` to `@init`.
- [ ] Hit test: LMB down inside a strip rect → record which strip and start value.
- [ ] Drag: map y delta to value change proportional to the slider's range. 200px = full range. Shift = 10× slower.
- [ ] Write result to correct slider, call `sliderchange()`.
- [ ] Scroll wheel: when mouse is over a strip, nudge ±0.5% of range per tick.

**Done when:** all five time controls draggable. Shift slows. Scroll nudges. No DSP regressions.

---

## Phase 4d — Mode Buttons + Display Toggle Interaction

**Goal:** clicking the mode buttons and display toggle updates plugin state.

- [ ] LMB click on a mode button: `slider1 = mode_index`, call `sliderchange()`.
- [ ] LMB click on style dropdown: no-op (single option).
- [ ] LMB click on display toggle: `ui_disp_enabled = 1 - ui_disp_enabled`.

**Done when:** mode switching works via the custom buttons. Display toggle fires on click. No DSP regressions.

---

## Phase 5 — Transfer Curve Display

**(Complete)** Transfer curve, dashed range line, threshold line, live input dot, dB grid, PK/RMS readouts all implemented and working.

---

## Phase 6 — Scrolling Level History

**(Complete)** Ring-buffer scrolling display with three-color gain band (dark blue / warm red / teal) implemented. Peak-bucketed anti-aliasing applied for smooth scrolling.

---

## Phase 7 — Polish

**Goal:** the difference between "works" and "feels good."

- [ ] Hover cursor feedback (brightness change already in 4b; consider distinct colors per parameter type).
- [ ] Consistent color palette: threshold = one color family, ratio = another, range = another, knee = another.
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
