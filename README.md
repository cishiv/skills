# skills

Personal Claude skills.

## Skills

- **[interview-me](skills/interview-me/)** — Interviews you with batched, themed questions written to Obsidian before producing complex deliverables (specs, system prompts, architecture docs). Surfaces your own thinking rather than letting the AI fill in the blanks.

## Usage

Skills work in both Claude Code and Claude Desktop. They can be invoked via slash command (e.g. `/interview-me`) or triggered automatically when their description matches the request.

Skills are stored in markdown format, not `.skill` format.

## skill-sync

A small Bun CLI in [`skill-sync/`](skill-sync/) for managing Claude Code skills:

```
skill-sync add [--source <path|git-url>]   # discover SKILL.md dirs and install to ~/.claude/skills
skill-sync list                            # show installed skills
skill-sync remove                          # uninstall (symlinks are left alone)
skill-sync editor [--port 4173]            # local Monaco-based browser editor
```

Run with `bun run skill-sync/src/index.ts <cmd>` or link the `skill-sync/bin/skill-sync` shim onto your `PATH`.
