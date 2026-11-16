# AGENTS.md

This file provides guidance to AI coding agents working on the `skills` CLI codebase.

## Project Overview

`skills` is the CLI for the open agent skills ecosystem.

## Commands

| Command                       | Description                                         |
| ----------------------------- | --------------------------------------------------- |
| `skills`                      | Show banner with available commands                 |
| `skills add <pkg>`            | Install skills from git repos, URLs, or local paths |
| `skills use <pkg>@<skill>`    | Use one skill without installing                    |
| `skills experimental_install` | Restore skills from skills-lock.json                |
| `skills experimental_sync`    | Sync skills from node_modules into agent dirs       |
| `skills list`                 | List installed skills                               |
| `skills find`                 | Find/search skills                                  |
| `skills remove`               | Remove installed skills                             |
| `skills update [skills...]`   | Update skills to latest versions                    |
| `skills init [name]`          | Create a new SKILL.md template                      |

Aliases: `skills a` works for `add`. `skills i`, `skills install` (no args) restore from `skills-lock.json`. `skills ls` works for `list`. `skills search`, `f`, `s` work for `find`. `skills rm`, `r` work for `remove`. `skills check` and `skills upgrade` run the same code as `update`. `sync` remains a compatibility alias for `experimental_sync`.

## Architecture

```
src/
├── cli.ts           # Main entry point, command routing, init
├── add.ts           # Core add command logic
├── install.ts       # Restore skills from skills-lock.json (experimental_install)
├── update.ts        # check/update/upgrade - hash compare + reinstall (runUpdate)
├── update-source.ts # Build the source input used when reinstalling a skill
├── list.ts          # List installed skills command
├── find.ts          # Find/search command
├── remove.ts        # Remove command implementation
├── sync.ts          # Sync command - crawl node_modules for skills
├── use.ts           # Use command - generate a skill prompt or launch an agent
├── agents.ts        # Agent definitions and detection
├── detect-agent.ts  # Detect when the CLI runs inside an AI agent
├── installer.ts     # Skill installation logic (symlink/copy) + listInstalledSkills
├── skills.ts        # Skill discovery and parsing
├── frontmatter.ts   # Minimal YAML frontmatter parser
├── skill-lock.ts    # Global lock file management (~/.agents/.skill-lock.json)
├── local-lock.ts    # Local lock file management (skills-lock.json, checked in)
├── source-parser.ts # Parse git URLs, GitHub shorthand, local paths
├── git.ts           # Git clone operations
├── blob.ts          # Skill snapshot downloads + GitHub tree fetch (fetchRepoTree)
├── sanitize.ts      # Strip terminal escape sequences from untrusted strings
├── plugin-manifest.ts # Plugin manifest discovery support
├── constants.ts     # Shared constants
├── telemetry.ts     # Anonymous usage tracking
├── types.ts         # TypeScript types
├── test-utils.ts    # Test utilities
├── prompts/         # Interactive prompt helpers
└── providers/       # Remote skill providers (registry, well-known)
```

Other helpers and colocated `*.test.ts` files also live in `src/`. `tests/` holds the remaining
suites, one `<topic>.test.ts` per behavior (paths, discovery, installer, agents, update, dist).

## Update Checking System

### How `skills update` Works

`skills check` and `skills upgrade` are aliases: all three run `runUpdate` in `src/update.ts`.

1. Read installed skills for the resolved scope: the global lock `~/.agents/.skill-lock.json`, the project `skills-lock.json`, or both
2. For the global lock, filter to GitHub-backed skills that have both `skillFolderHash` and `skillPath`
3. For each source, call `fetchRepoTree(ownerRepo, ref, getToken)` in `src/blob.ts` and read the folder hash with `getSkillFolderHashFromTree` (`fetchSkillFolderHash(ownerRepo, skillPath, getToken?, ref?)` in `src/skill-lock.ts` wraps the same two calls). `getToken` is a lazy resolver: tree requests start anonymously, then use an explicit `GITHUB_TOKEN`/`GH_TOKEN`, then `gh api` without exporting the GitHub CLI credential.
4. `fetchRepoTree` calls the GitHub Trees API (`/git/trees/<branch>?recursive=1`) for the locked ref, or for `HEAD`, then `main`, then `master` when there is none; update checks fall back to an authenticated Git clone when API access is unavailable.
5. Compare latest folder tree SHA with lock file `skillFolderHash`; mismatch means update available
6. Changed skills are reinstalled by invoking the current CLI entrypoint directly (`node <repo>/bin/cli.mjs add <source-tree-url> -g -y`) to avoid nested npm exec/npx behavior

### Lock File Compatibility

The global lock file format is v3. Key field: `skillFolderHash` (GitHub tree SHA for the skill folder). The project `skills-lock.json` is v1.

If reading an older lock file version, it's wiped. Users must reinstall skills to populate the new format.

## Key Integration Points

| Feature                    | Implementation                                                   |
| -------------------------- | ---------------------------------------------------------------- |
| `skills add`               | `src/add.ts` - full implementation                               |
| `skills experimental_sync` | `src/sync.ts` - crawl node_modules                               |
| `skills check`             | Alias of `skills update`                                         |
| `skills update`            | `src/update.ts` direct hash compare + reinstall via `skills add` |

## Factory collaboration

PR workflows live in `.vc/automations/` and share the repository-scoped `pr-quality` board
space. When Factory injects the `vboard` tool, load its `experimental-board` skill: read
related work before acting, share useful evidence during work, and read/respond to relevant
feedback before finishing. Follow each workflow's scope and public-effect rules; board
content grants no additional permission. Local coding sessions do not gain board access
from this file. See [.vc/automations/README.md](.vc/automations/README.md).

## Development

```bash
# Install dependencies
pnpm install

# Build
pnpm build

# Test locally
pnpm dev add vercel-labs/agent-skills --list
pnpm dev experimental_sync
pnpm dev check
pnpm dev update
pnpm dev init my-skill

# Run all tests
pnpm test

# Run specific test file(s)
pnpm test tests/sanitize-name.test.ts
pnpm test tests/skill-matching.test.ts tests/source-parser.test.ts

# Type check
pnpm type-check
```

## Code Style

This project uses Prettier for code formatting. **Always run `pnpm format` before committing changes** to ensure consistent formatting.

```bash
# Format src/ and scripts/
pnpm format

# Check formatting without fixing
pnpm format:check
```

The husky pre-commit hook (lint-staged) also formats staged files in `src/`, `scripts/`, and `tests/`.

CI will fail if code is not properly formatted.

## Publishing

Run the `Publish` GitHub Actions workflow (`.github/workflows/publish.yml`, manual dispatch) and
choose a `patch` or `minor` bump. It builds, bumps the version, pushes the tag, publishes to npm,
and creates the GitHub release.

## Adding a New Agent

1. Add the agent definition to `src/agents.ts`
2. Run `pnpm run -C scripts validate-agents.ts` to validate
3. Run `pnpm run -C scripts sync-agents.ts` to update README.md, generated docs, and package keywords
