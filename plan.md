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

- [ ] Add a delay buffer for the audio path.
- [ ] Detector reads the present sample; audio output reads from the delayed buffer.
- [ ] Report latency to Reaper via `pdc_delay`.
- [ ] Handle the case where lookahead = 0 (no latency reported).
- [ ] Test PDC by inserting on a track alongside other plugins and confirming alignment.

**Done when:** lookahead non-zero, transients are gated cleanly without "clipped" attack. PDC works correctly.

---

## Phase 4 — Custom UI Shell

**Goal:** replace default sliders with custom `@gfx` UI. No fancy displays yet — just laid-out custom knobs, mode buttons, and a style dropdown.

- [ ] Design layout proportions based on `gfx_w` / `gfx_h`.
- [ ] Implement a reusable `ui_draw_knob` function.
- [ ] Implement a reusable `ui_hit_test_knob` function for mouse interaction.
- [ ] Implement mouse drag to adjust knob values (shift = fine, ctrl = reset, scroll = nudge, double-click = reset).
- [ ] Implement the three-button mode toggle row at the top.
- [ ] Implement the Style dropdown (single item for now).
- [ ] Lay out all eight dynamics knobs in their target positions.
- [ ] Add labels and value readouts.
- [ ] Implement the display toggle button.

**Done when:** plugin is fully usable from custom UI. No regressions in DSP. Mouse interaction feels responsive. Mode toggle works visually and reloads defaults correctly.

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

## Deferred

Captured here so they don't get lost. Move to a phase when ready.

- Additional styles (Vocal, Drums, Guitar, Ducking, etc.) — populate the Style dropdown.
- Mid-side processing.
- External sidechain.
- Sidechain EQ.
- Wet/Dry mix.
- Oversampling.
- MIDI trigger.
- Tooltips / help system.
- Workflow features unique to Mr. Gate: histogram overlay, learn-threshold, freeze-GR, etc.
