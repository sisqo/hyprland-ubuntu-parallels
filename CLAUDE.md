# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Documentation only: no code, no build, no tests, no lint. It records the
*why* behind the Hyprland setup on this Ubuntu 24.04 arm64 Parallels VM, so
the setup could be rebuilt from scratch by reading it.

The real config lives on the VM itself (`~/.config/hypr/`, `~/.config/waybar/`,
`~/.config/rofi/`, `~/.config/foot/`, `~/.local/bin/`, ...). **Never copy
config files into this repo.** A second copy would go stale. Quote only the
few lines that matter and point to the real file.

## Writing conventions

- Tie every claim to something you can check: a file path, a config line, a
  command and its output, or a package version. Pin versions in the note,
  e.g. "waybar 0.9.24", "Hyprland 0.56.2". `docs/setup.md` holds the version
  baseline. It is dated, so update the date when you re-check it.
- Before you write a note, check the live system: `cat` the real config, run
  `dpkg -l`, `hyprctl`, and so on. Don't document from memory.
- Explain the reason for each choice and what breaks without it. Include the
  alternatives you rejected when that isn't obvious (see "`mainMod` is ALT,
  not SUPER" in `docs/shortcuts.md`).
- Each topic has one file under `docs/`, listed in the README's **Index**. If
  you add a new doc file, add a one-line entry to that index.
- When a version change fails silently or gives a confusing symptom, it goes
  in `docs/config-gotchas.md`. Give it its own `##` section with the version
  pinned. Also cross-link it from the topic doc it affects.
- New or changed keybinds go in the table in `docs/shortcuts.md`. That table
  can drift from the real config. The source of truth is
  `~/.config/hypr/hyprland.conf`, shown live by
  `~/.config/hypr/scripts/shortcuts.sh`.
- Link between docs with relative links and heading anchors, e.g.
  `[setup.md](setup.md#installing-hyprland)`. If you rename a heading, fix
  the anchors that point to it.
- Docs are written in English.

## Commits

Use one commit per documented change. The subject is short and imperative,
usually starting with "Document ...", e.g. `Document mainMod+È for cla2`.

## What belongs here

Record lasting setup changes: config edits, new gotchas, new keybinds, and
non-obvious dependencies. Skip one-off tweaks with no lasting effect. If
unsure, ask the user rather than skipping it or over-documenting.
