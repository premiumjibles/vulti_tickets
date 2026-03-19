---
id: tic-1ebc
status: open
type: task
priority: 3
assignee: Jibles
created: 2026-03-19T09:14:57.600471622Z
---
# Migrate LUNA/LUNC balance support to chat app

## Context

Station wallet (Terra Station fork) is being gutted to become the new chat app. The previous owner requested we maintain native LUNA/LUNC balance display for existing users who still hold bags from the 2022 Terra collapse.

## Approach

- Terra is **Cosmos-based** (not EVM) — uses LCD RPC endpoints, bech32 addresses
- Native balance query is simple: `lcd.bank.balance()` / `lcd.bank.spendableBalances()`
- Shapeshift already has a working Terra implementation in their hdwallet packages at `../shapeshift/web/packages/hdwallet-core/src/terra.ts` and `hdwallet-native/src/terra.ts` — use as reference
- Station's current implementation is in `station/src/data/queries/bank.ts` and `station/src/data/token.tsx`

## Chain details

- **phoenix-1** (mainnet) → LUNA
- **columbus-5** (classic) → LUNC
- Denom: `uluna` — branch on chainID to distinguish LUNA vs LUNC

## Scope

- Only need native coin balance display (LUNA and LUNC)
- Do NOT need CW20, IBC, factory token support
- Do NOT need staking, governance, swaps, or other Station features
- This is a courtesy feature for existing bag holders, not a growth feature

