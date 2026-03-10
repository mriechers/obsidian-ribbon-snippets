# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Non-Claude Agents: Workspace Isolation Rules

**If you are NOT Claude Code** (e.g., Gemini CLI, Codex, Copilot, Cursor, or any other LLM agent), you MUST follow these isolation rules to avoid interfering with Claude agent work:

1. **Always work in a git worktree.** Never modify files on the current branch directly. Create a worktree before making any changes:
   ```bash
   git worktree add .worktrees/<your-agent-name>-<feature> -b <your-agent-name>/<feature-description>
   ```
2. **Always commit to a feature branch.** Your branch name must include your agent identifier (e.g., `gemini/fix-typo`, `codex/add-tests`, `copilot/refactor-utils`). Never commit to `main` or any branch you did not create.
3. **Never force-push, rebase, or modify branches you did not create.** Treat all existing branches as read-only references.
4. **Clean up after yourself.** When your work is complete, leave the worktree in place for human review. Do not merge your own branches.
5. **Follow all other rules in this file.** The conventions below (commit format, secrets management, attribution, etc.) apply to all agents equally.

> Claude Code agents are exempt from the worktree requirement because they are the primary agents for this workspace and coordinate directly with the user.


## Project Overview

Obsidian plugin that lets users define text snippets, each assigned to a ribbon button and command palette entry for one-click insertion into notes. Published as community plugin "Ribbon Snippets" (id: `ribbon-snippets`). Requires Obsidian ≥ 1.7.2 for `removeCommand()` API.

## Build Commands

- `npm run dev` — watch mode (esbuild rebuilds on file changes)
- `npm run build` — production build (outputs `main.js` at repo root)
- `./deploy.sh` — builds and copies `main.js`, `manifest.json`, `styles.css` to the local Obsidian vault plugin folder

No test framework is configured. No linter is configured.

## Architecture

Two-file plugin in `src/`:

**`src/main.ts`** — Plugin class (`RibbonSnippetsPlugin extends Plugin`)
- Manages the snippet lifecycle: ribbon icons, command palette entries, and text insertion
- Two save paths: `saveSettings()` (full save + rebuild ribbon/commands) for structural changes vs `debouncedSave()` (persist-only, 400ms debounce) for text field edits to avoid async race conditions in ribbon DOM management
- Insertion logic handles three positions: top (after YAML frontmatter), cursor, bottom
- Duplicate detection compares snippet's first line against full note content
- Dynamic command lifecycle using `addCommand()`/`removeCommand()` — commands are re-registered on every structural settings change

**`src/settings.ts`** — Settings tab UI (`RibbonSnippetsSettingTab extends PluginSettingTab`)
- Full `display()` re-render on structural changes (add/remove/reorder)
- Live Lucide icon preview with `setIcon()` wrapped in try/catch for invalid names
- Snippet actions: reorder (swap in array), delete (splice)

**Data model** — `RibbonSnippet` interface with `id`, `name`, `icon`, `content`, `position`, `checkDuplicate`. Settings stored as `{ snippets: RibbonSnippet[] }`.

## Build System

esbuild bundles `src/main.ts` → `main.js` (CJS format, ES2018 target). All Obsidian/CodeMirror/Electron modules are externalized. The built `main.js` is gitignored.

## Release Process

Push a semver tag (e.g., `1.0.1`) to trigger `.github/workflows/release.yml`, which builds and creates a GitHub release attaching `main.js`, `manifest.json`, and `styles.css`. Update `versions.json` to map plugin version → minimum Obsidian version.

## Key Conventions

- Snippet IDs are generated with `Date.now().toString(36) + random` — not UUIDs
- The plugin relies on mutating the `settings.snippets` array in place, then calling the appropriate save method
- CSS classes are prefixed with `ribbon-snippet-` / `ribbon-snippets-` and use Obsidian CSS custom properties for theming
