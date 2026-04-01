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

## Safety
Changes to key material, signing flows, or address derivation require extra scrutiny. Flag in PR descriptions.
