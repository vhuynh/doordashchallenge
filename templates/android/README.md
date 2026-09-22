# Android project kit

Drop-in starting point for a new Android project worked on with Claude Code.

## Setup

```bash
# 1. Conventions + spec template
cp templates/android/CLAUDE.md             <new-project>/CLAUDE.md
mkdir -p <new-project>/docs
cp templates/android/docs/spec_template.md <new-project>/docs/spec_template.md

# 2. Google's official Android agent skills
cd <new-project>
android skills add --agent='claude-code' --all --project=.
```

Note the agent identifier is **`claude-code`**, not `claude` — the CLI rejects the latter. Run `android skills list` to see what's available, and `android skills add --skill=<name>` to install selectively instead of `--all`.

This installs into `.claude/skills/`. Skills load progressively — only the one-line descriptions sit in context, and a skill's full instructions load when its trigger matches — so `--all` is cheap but not free. For a focused project, the relevant subset is usually:

| Skill | Use |
|---|---|
| `testing-setup` | Test strategy and dependency setup |
| `styles` | Compose Styles API |
| `adaptive` | Adaptive layouts across window sizes |
| `edge-to-edge` | Edge-to-edge insets handling |
| `navigation-3` | Jetpack Navigation 3 |
| `navigation-event` | Predictive back |
| `android-cli` | Driving the `android` CLI itself |
| `agp-9-upgrade`, `r8-analyzer` | Build and shrinker work |
| `android-intent-security` | Intent-handling audit |

The rest target surfaces most projects never touch — Wear, TV, XR glasses, Cast, Play Billing, Engage SDK, ML Kit, CameraX, AppFunctions, credentials.

## Then

1. Fill in **Project overrides** at the top of `CLAUDE.md`.
2. Work through the per-project decision tables, record the choices in Project overrides, and delete the tables you've resolved.
3. Copy `docs/spec_template.md` to `docs/spec-<feature>.md` per feature and fill it out **before** writing code.

## Why two files

`CLAUDE.md` is loaded into context automatically every session — standing conventions belong there so they're in force whenever Claude writes code. `spec_template.md` is per-feature thinking that's only read when opened. Putting conventions in the template means they're silently ignored most of the time.

Keep them disjoint: if a rule holds for every feature, it goes in `CLAUDE.md`, and the spec should never restate it.

## Relationship to the skills

The skills are Google's official, API-specific guidance (how to use CameraX, how Navigation 3 works). `CLAUDE.md` is your architectural opinion (where the boundaries sit, what gets tested, what never happens). They don't overlap, and neither replaces the other — where a skill's sample code conflicts with a non-negotiable in `CLAUDE.md`, `CLAUDE.md` wins.
