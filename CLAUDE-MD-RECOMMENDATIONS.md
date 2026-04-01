# CLAUDE.md Recommendations

Analysis of the proposed hub-level CLAUDE.md from PROPOSAL.md, based on prompt engineering principles. Use this to build the actual CLAUDE.md.

---

## What to Keep (earns its context cost)

These fire on most turns and prevent real failure modes:

```markdown
## Repos
- vultiagent-app/ — React Native mobile app (Expo), chat-based wallet
- vultisig-sdk/ — Unified TypeScript SDK, all chain logic and signing
- agent-backend/ — Go AI chat service, Claude integration, MCP tool calls
- vasig/ — MCP server + CLI for 3rd party access
- station/ — Station wallet web app
```

**Why**: Claude needs to know what lives where to route changes correctly. Update to match actual repo layout (original missed station/, vultiagent-cli/, mobile repos).

```markdown
## Architecture Rule
Business logic goes in vultisig-sdk, not app/CLI/backend layers.
```

**Why**: This is the single most important architectural constraint. Without it, Claude will happily put chain logic in whatever repo it's editing. One line, high value.

```markdown
## Code Style
- Biome for TS/JS — run `pnpm fix` to auto-format
- gofmt + golangci-lint for Go
```

**Why**: Prevents Claude from using prettier, eslint, or wrong formatting.

---

## What to Cut

### Domains section
**Problem**: Domains are team coordination, not coding instructions. Claude doesn't need to know who owns what — it needs to know where code goes.
**Alternative**: The repos list + architecture rule covers this.

### Conventions that restate defaults
These lines add context cost without changing behavior:
- "Reuse existing code, check imports/utils before writing new ones" — already in the parent ~/.claude/CLAUDE.md
- "Prefer readable, procedural code over abstractions" — already in the parent
- "Comments only for non-obvious logic, one line max" — already in the parent
- "Commit prefixes: feat:, fix:, chore:" — Claude follows conventional commits by default
- "Self-test, self-merge. CI must pass." — team process, not a coding instruction

### Human process rules
These are for humans, not Claude:
- "Pre-merge review required for: key material, signing, address derivation"
- "Big cross-cutting changes: Discord summary..."
- "SDK interface changes: post the proposed change..."

**Where they belong**: TEAM-PROCESS.md (already there).

---

## What to Add

### Build/test commands per repo
Claude needs to know how to build and test. Without this, it guesses.

```markdown
## Build & Test
- vultisig-sdk/: `pnpm test`, `pnpm build`
- agent-backend/: `go test ./...`, `go build ./...`
- vultiagent-app/: `pnpm expo start`, `pnpm test`
```

### Sub-repo CLAUDE.md reminder
```markdown
Each sub-repo has its own CLAUDE.md with repo-specific context. This file is inherited by all.
```

### Critical safety constraint
The one human-process rule that Claude *should* know, reframed as a coding instruction:

```markdown
## Safety
Changes to key material, signing flows, or address derivation require extra scrutiny. Flag these in PR descriptions.
```

---

## Recommended Hub CLAUDE.md

Putting it together — ~20 lines, every line earns its keep:

```markdown
# Vultisig

## Repos
- vultisig-sdk/ — Unified TypeScript SDK, all chain logic and signing
- agent-backend/ — Go AI chat service, Claude integration, MCP tool calls
- vultiagent-app/ — React Native mobile app (Expo), chat-based wallet
- station/ — Station wallet web app
- vasig/ — MCP server + CLI for 3rd party access
- vultiagent-cli/ — CLI agent interface

Each sub-repo has its own CLAUDE.md. This file applies to all.

## Architecture
Business logic goes in vultisig-sdk, not app/CLI/backend layers.

## Code Style
- Biome for TS/JS — `pnpm fix` to auto-format
- gofmt + golangci-lint for Go

## Build & Test
- vultisig-sdk/: `pnpm test`, `pnpm build`
- agent-backend/: `go test ./...`, `go build ./...`
- vultiagent-app/: `pnpm expo start`, `pnpm test`

## Safety
Changes to key material, signing flows, or address derivation require extra scrutiny. Flag in PR descriptions.
```

---

## Design Notes

- **~20 lines** vs the original ~30. Cut ~40% of context cost.
- Every line prevents a specific Claude failure mode (wrong repo, wrong formatting, unsafe change shipped quietly).
- Domains, commit prefixes, and human coordination rules removed — they don't change Claude's coding behavior.
- Parent CLAUDE.md rules (reuse code, procedural style, minimal comments) aren't repeated.
- Build commands added — the biggest gap in the original. Claude can't run tests it doesn't know about.
- Verify build commands are accurate before using — I inferred them from common patterns.
