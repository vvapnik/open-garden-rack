# AGENTS.md

Guidance for AI agents working in this repo.

Project map: `ARCHITECTURE.takesix.json` — read it first. It shows how the
spec is split into layers (physical, enumeration, frames, registers/IDs),
which `docs/*.md` file holds each part, and which libraries implement it.

## What this is

The OpenGardenRack (OGR) protocol specification: documents only, no code.
Each part of the spec lives in its own `docs/NN-*.md` file; the index is in
[README.md](README.md). Reference implementations live in separate
repositories (see `docs/08-libraries.md`).

## Rules

- Keep changes scoped to the `docs/*.md` file(s) they affect, and note
  them in `CHANGELOG.md`.
- When a change moves or adds a spec part, update
  `ARCHITECTURE.takesix.json` so its block paths still match `docs/`.
- Everything in this repo is written in English.

## AI assistance

This project is developed with the help of Claude Code (Anthropic).
