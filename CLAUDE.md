# CLAUDE.md — Mr. Gate JSFX Plugin

This file governs how Claude Code works on this project. Read it at the start of every session. Do not deviate from these rules without explicit user approval.

## Project Summary

Mr. Gate is a Reaper JSFX dynamics plugin offering three modes (Gate, Downward Expander, Upward Expander), a single well-tuned envelope algorithm called "General," and a custom `@gfx` UI inspired by FabFilter Pro-G's visual feedback. The user is a Reaper-only user who wants a CPU-efficient expander with great visual feedback. There is no goal of imitating Pro-G's sound — the DSP is judged by ear, not by null testing against any commercial product.

## Architectural Rules

JSFX has a fixed section structure. Code MUST be organized into these sections in this order, and nowhere else:

1. **`desc:` / `tags:` / metadata** — plugin name, tags, channel config.
2. **`slider:` declarations** — every parameter the user can control.
3. **`@init`** — runs once when the plugin loads or sample rate changes. All memory allocation, lookup table generation, and one-time setup goes here.
4. **`@slider`** — runs when any slider changes. All parameter conversion (dB→linear, ms→coefficients, mode-dependent rescaling) goes here. Never do this conversion in `@sample`.
5. **`@block`** — runs once per audio block.
6. **`@sample`** — runs once per audio sample. The DSP hot path. Keep this LEAN. No allocations, no expensive math, no branches that can be hoisted to `@slider`.
7. **`@gfx`** — runs at UI refresh rate (typically 30fps). All custom drawing.

**Why this matters:** JSFX runs `@sample` per sample per channel. At 48kHz stereo that's ~96,000 calls per second. CPU efficiency is an explicit user goal — pre-compute everything possible in `@slider` or `@init`.

## Memory Layout

JSFX uses a flat memory model with index-based access. Memory regions MUST be declared at the top of `@init` with named index constants. Convention:

```
// Memory layout:
//   0..1023            : lookahead delay buffer (left)
//   1024..2047         : lookahead delay buffer (right)
//   2048..4095         : level history ring buffer (input)
//   4096..6143         : level history ring buffer (output)
//   6144..6175         : transfer curve cached points
//   ...
```

Always document memory layout in a comment block at the top of `@init`. When adding new memory regions, update the comment block.

## Naming Conventions

- Slider variables: `slider1`, `slider2`, etc. (JSFX requirement). Aliased to readable names in `@slider`: `threshold_db = slider1;`.
- DSP state variables: `lowercase_with_underscores` — `env_follower`, `gain_smoothed`, `hold_counter`.
- Constants from `@init`: `SCREAMING_CASE` — `BUFFER_SIZE`, `LOG2DB`.
- `@gfx` helpers: prefix with `ui_` — `ui_draw_knob`, `ui_hit_test_curve`.
- Mode-related: `MODE_GATE = 0;`, `MODE_DOWN = 1;`, `MODE_UP = 2;`.

## The Three Modes

Mr. Gate has three operational modes selected by a top-level toggle: Gate, Downward Expander, and Upward Expander. They share **one** DSP code path. The mode affects:

1. **Gain computation direction.** In Up mode, the sign of the deviation from threshold flips: gain is *added* when input is *above* threshold. In Gate and Down modes, gain is *reduced* when input is *below* threshold.
2. **Default values** when the user changes mode or first loads the plugin (see `spec.md`).
3. **Slider display ranges** in the UI (e.g., Range knob shows 0-60 dB in Gate mode but 0-30 dB in Expander modes).
4. **Transfer curve display geometry** — the curve flips orientation in Up mode.

It does NOT cause separate code paths for envelope detection, smoothing, or hold logic. Those are mode-agnostic.

**Implementation guideline:** the gain computation function takes a parameter `direction` (+1 for downward, -1 for upward) and the rest of the logic is shared.

## Don't Do This

These are explicit prohibitions based on lessons from prior projects. Do NOT do any of these without first stopping and asking the user.

1. **Do not put parameter conversion in `@sample`.** All dB→linear, ms→coefficient, ratio math goes in `@slider`. The hot path only reads pre-computed values.
2. **Do not allocate memory inside `@sample` or `@gfx`.** All memory regions are reserved in `@init`.
3. **Do not add new sliders without updating `spec.md`.** The slider list is the public API of the plugin and must stay aligned with the spec.
4. **Do not write `@gfx` code that depends on specific window sizes.** Use `gfx_w` and `gfx_h` and lay out proportionally.
5. **Do not implement modes by branching on `mode_id` inside `@sample`.** Branches in the hot path are a code smell. Pre-compute mode-dependent coefficients and direction sign in `@slider`.
6. **Do not change DSP behavior to "fix" a UI issue, or vice versa.** They are separate concerns. If the meter looks wrong, the meter code is wrong, not the gain reduction calculation.
7. **Do not refactor proactively.** Only refactor when the user asks or when explicitly required to complete a feature. Drift from "let me just clean this up" is the #1 failure mode.
8. **Do not introduce frameworks, abstractions, or "engines" beyond what JSFX natively provides.** No custom event systems, no observer patterns. JSFX is small; keep the code small.
9. **Do not write code that requires features outside standard JSFX.** No external libraries, no file I/O during processing, no network calls. Image assets loaded via `gfx_loadimg` in `@init` are fine.
10. **Do not add features beyond what's currently in `plan.md`'s active phase.** "While I'm in here..." is how scope creep starts.

## Workflow

1. Before any code change, identify which phase of `plan.md` it belongs to. If it doesn't fit a current phase, stop and ask.
2. Reference `spec.md` for what the feature should do. If the spec doesn't cover it, stop and ask.
3. After completing a feature, update `plan.md` to mark it done and note any deviations from the original plan.

## Communication

- When proposing a change, describe what it does in terms of audible or visual behavior, not just code structure.
- When fixing a bug, identify the root cause before applying a fix. The user has stated a strong preference for surgical, root-cause fixes over symptom patches.
- When unsure, ask. The user prefers being asked over assumptions.
