# skill-memos

Read and write notes in the personal Memos instance at memos.ryanburnette.com with the `memo` CLI. Use when asked to save a note, jot something down, look something up in memos, or search past notes.

## Usage

This is an [Agent Skills](https://agentskills.io/) compatible skill. Load it with your agent harness and invoke via `skill:memos`.

The skill documents the `memo` CLI — a thin POSIX wrapper over the Memos REST API, shipped in this repo at `bin/memo` — plus a raw-curl fallback for everything the CLI lacks: comments, reactions, attachments, and pagination.

## Structure

- `SKILL.md` — Frontmatter, CLI commands, visibility, sharing, gotchas, auth, REST fallback, and safety notes.
- `bin/memo` — The `memo` CLI. Symlinked to `~/bin/memo` by the dotfiles `setup.sh`.
