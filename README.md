# Mr. Gate

A Reaper JSFX dynamics plugin with three modes (Gate, Downward Expander, Upward Expander), a single well-tuned algorithm, and rich visual feedback inspired by FabFilter Pro-G.

## Project Files

- `mrgate.jsfx` — the plugin itself (single file, edited in Reaper).
- `CLAUDE.md` — rules for AI-assisted development. Read first.
- `spec.md` — feature specification. Source of truth for what the plugin does.
- `plan.md` — phased development roadmap.
- `reference/` — notes, screenshots, sketches.

## Working with Claude Code

Reference the planning files in your prompts:

```
@CLAUDE.md @plan.md @spec.md

I want to start Phase 1. Build the DSP core for downward direction...
```

Always include `@CLAUDE.md` first. The architectural rules there are non-negotiable.

When advancing between phases, mark items complete in `plan.md` and commit before starting the next phase.

## Installing the Plugin

Drop `mrgate.jsfx` into your Reaper Effects folder (Options → Show REAPER resource path → Effects). It will appear under JS in the FX browser.
