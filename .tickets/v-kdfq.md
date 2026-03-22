---
id: v-kdfq
status: closed
deps: []
links: []
created: 2026-03-22T10:24:49Z
type: chore
priority: 2
assignee: Jibles
tags:
    - cli
    - validation
    - errors
---
# CLI: Input validation and error message cleanup

## Problem

Multiple vasig commands accept invalid inputs without local validation, producing confusing errors from the SDK or remote APIs. Error messages leak raw SDK/library internals (viem stack traces, LI.FI SDK version strings). Silent no-ops on remove/duplicate-add make agent automation unreliable.

## Research Findings

- `vasig balance --chain ethereum` fails — no case-insensitive chain matching, dumps full 36-chain list with no "did you mean?" suggestion
- `vasig chains --add NonexistentChain` → "Password required for encrypted vault" — validates chain name too late (after trying to derive address)
- SDK errors leak raw: viem `ContractFunctionExecutionError` with call arguments, LI.FI SDK version strings, raw NaN errors
- `vasig tokens --remove 0xnonexistent` / `vasig chains --add <already-active>` / `vasig chains --remove <inactive>` all report success (exit 0) — agents can't distinguish real operations from no-ops
- `vasig send --chain Arbitrum --memo "test"` → raw `eth_sendTransaction format error`
- The SDK uses exact `Chain` enum values (e.g., `"THORChain"`, `"BSC"`) — user input must be resolved to canonical values
- The SDK's `prepareSendTx` accepts a `memo` field but may not handle it correctly for EVM chains

## Notes

**2026-03-22T11:25:00Z**

# Design

## Assessment

vasig's existing architecture is already well-organized:
- `lib/errors.ts` — typed error hierarchy (VasigError → InvalidChainError, NoRouteError, etc.), classifyError(), toErrorJson()
- `lib/output.ts` — printResult()/printError() with json/table formatting
- `lib/signing.ts` — signWithRetry() for SDK signing reliability
- Commands are pure functions returning typed results, don't handle output formatting
- `index.ts` main() is the single top-level error boundary with classifyError() fallback

The pattern is right, it's just incomplete. No restructuring needed — fill the gaps.

## Architecture

One new file, targeted fixes to existing files:

**New: `lib/validation.ts`** — input validation helpers that throw errors from `lib/errors.ts`. Keeps error *types* separate from *validation logic*.

**Modified: `lib/errors.ts`** — enhance classifyError() to clean SDK message text, not just classify the type.

**Modified: 4-5 command files** — wire in validation, fix no-op semantics.

## Components

### 1. `lib/validation.ts` — `resolveChain(input: string): string`

Case-insensitive match against SUPPORTED_CHAINS, returns canonical Chain enum value. On failure, throws InvalidChainError with "Did you mean: X?" hint using basic prefix/Levenshtein matching.

Called in: chains.ts (add/remove), tokens.ts (--chain), send.ts (--chain), swap.ts (parseChainToken), balance.ts (--chain).

Also add `parseAmount(input: string): number` — validates numeric, > 0. Throws UsageError. Used in send.ts (currently missing validation) and already done inline in swap.ts.

### 2. `lib/errors.ts` — enhance `classifyError()` message cleaning

Current behavior: classifies error type correctly but passes through the raw SDK message. e.g., InvalidAddressError gets the full viem ContractFunctionExecutionError dump as its message.

Add a `cleanSdkMessage()` step that strips known noise patterns:
- `ContractFunctionExecutionError: ...` → "Invalid contract address" or "Contract call failed"
- Messages containing LI.FI SDK version strings → strip to useful part only
- `NaN` / `Cannot convert` → "Invalid amount"
- `eth_sendTransaction` format errors → "Transaction format error"

Cleaned message goes to the error class. Original preserved as `cause` for debugging.

### 3. `chains.ts` — fix no-op semantics

**Add (already-active):** Already checks `vault.chains.includes(chain)` — change output from `action: "added"` to `action: "already-active"` when chain was already there. Exit 0 (idempotent — agents can safely "ensure chain X is active").

**Remove (not-found):** Add `vault.chains.includes(chain)` check before removing. If not present, throw InvalidChainError with hint. Exit 1 (strict — removing something that doesn't exist is a logic error).

Both add and remove: run input through `resolveChain()` first.

### 4. `tokens.ts` — fix no-op semantics

**Remove:** Check if token exists in persisted tokens before removing. If not found, throw TokenNotFoundError. Exit 1.

**--chain option:** Run through `resolveChain()`.

### 5. `send.ts` — amount validation + EVM memo

**Amount:** Add `parseAmount()` check before BigInt parsing. Currently `--amount abc` → raw BigInt crash.

**EVM memo:** Reject with UsageError("Memos are not supported on EVM chains", "EVM transactions use the data field instead") when memo is provided for EVM chains. Hardcode EVM chain list for now — note for SDK to export chain metadata later.

### 6. `swap.ts` — chain validation

Run `parseChainToken()` results through `resolveChain()` so `--from ethereum:USDC` resolves to `Ethereum:USDC`.

## Data Flow

```
User input ("ethereum", "0.001", "0xinvalid")
  → Commander parses raw strings
  → Command calls resolveChain() / parseAmount() from lib/validation.ts
  → Validated, canonical values passed to SDK
  → SDK call
  → If SDK throws → command catch block or main() catch → classifyError() cleans type + message
  → VasigError formatted via printError() (json or table)
  → Appropriate exit code
```

Validation at two boundaries:
1. **Input boundary** (commands, via lib/validation.ts) — is this a real chain? is this a number?
2. **Output boundary** (main() catch, via classifyError()) — SDK errors cleaned and classified

Commands only handle **semantic validation** — things requiring vault state (is chain already active? does token exist?).

## Error Handling

| Tier | Where | Example | Exit |
|------|-------|---------|------|
| Input validation | Command via resolveChain() | `--chain ethereum` → "Unknown chain. Did you mean: Ethereum?" | 1 |
| Semantic (idempotent) | Command logic | `chains --add Avalanche` already active → `action: "already-active"` | 0 |
| Semantic (strict) | Command logic | `chains --remove Foo` not active → InvalidChainError | 1 |
| SDK wrapping | main() via classifyError() | viem dump → "Invalid contract address" | varies |

No-op semantics: **idempotent adds (exit 0), strict removes (exit 1).** Agent preparing a swap doesn't want "add Arbitrum" to fail because it's already there. But "remove a token that doesn't exist" is a logic error that should surface before the agent proceeds with wrong assumptions.

## Testing Approach

- Unit test `resolveChain()`: exact match, case-insensitive match, "did you mean?" suggestion, unknown chain error
- Unit test `cleanSdkMessage()`: viem errors, LI.FI errors, NaN errors, passthrough for unknown
- Unit test `parseAmount()`: valid numbers, zero, negative, non-numeric, "max"
- Integration: run each fixed command with bad input, verify error codes and clean messages in JSON mode
- Verify: `chains --add` on already-active returns `already-active`, `chains --remove` on inactive returns error

## Approved Approach

Fill gaps in vasig's existing well-organized architecture. One new file (lib/validation.ts), enhance classifyError() message cleaning, targeted fixes in 4-5 command files. No restructuring needed.

## SDK Opportunities (future — see ticket vc-mpnw)

These workarounds in vasig point to things the SDK should own:
- `resolveChain()` belongs in the SDK so all consumers get case-insensitive matching
- SUPPORTED_CHAINS should export chain metadata (isEVM, supportsMemo, etc.) so consumers don't hardcode EVM chain lists
- SDK error messages should be clean at source — classifyError() is a workaround

**2026-03-22T11:34:01Z**

# Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use /run tk:v-kdfq to implement this plan task-by-task via subagent-driven-development.

**Goal:** Add input validation, fix no-op semantics, and clean SDK error messages so vasig produces clear, actionable errors and reliable exit codes for both humans and agents.

**Architecture:** One new file (`lib/validation.ts`) for input validation helpers. Enhance `withExit` error handling to clean SDK messages. Targeted fixes in 5 command files to wire in validation and fix no-op semantics. The existing `findChainByName` in `interactive/completer.ts` does case-insensitive matching already — we extract and enhance it with "did you mean?" suggestions.

**Tech Stack:** TypeScript, Commander.js, @vultisig/sdk (Chain, SUPPORTED_CHAINS), existing CLIError/Errors from lib/errors.ts

---

### Task 1: Create lib/validation.ts — resolveChain()

**Files:**
- Create: \`src/lib/validation.ts\`
- Modify: \`src/lib/index.ts\` (add export)

**Step 1: Create lib/validation.ts with resolveChain()**

\`\`\`typescript
import type { Chain } from '@vultisig/sdk'
import { SUPPORTED_CHAINS } from '@vultisig/sdk'

import { Errors } from './errors'

export function resolveChain(input: string): Chain {
  const inputLower = input.toLowerCase()

  // Exact match
  const exact = SUPPORTED_CHAINS.find(c => c === input)
  if (exact) return exact

  // Case-insensitive match
  const caseMatch = SUPPORTED_CHAINS.find(c => c.toLowerCase() === inputLower)
  if (caseMatch) return caseMatch

  // "Did you mean?" — prefix match first, then Levenshtein
  const prefixMatches = SUPPORTED_CHAINS.filter(c => c.toLowerCase().startsWith(inputLower))
  if (prefixMatches.length === 1) {
    throw Errors.invalidUsage(
      \`Unknown chain: "\${input}". Did you mean: \${prefixMatches[0]}?\`,
      [\`Use --chain \${prefixMatches[0]}\`, 'Run "vultisig chains" to list all chains']
    )
  }

  const suggestions = prefixMatches.length > 0
    ? prefixMatches
    : findClosest(input, SUPPORTED_CHAINS, 3)

  const hint = suggestions.length > 0
    ? \`Did you mean: \${suggestions.join(', ')}?\`
    : 'Run "vultisig chains" to list all supported chains'

  throw Errors.invalidUsage(
    \`Unknown chain: "\${input}"\`,
    [hint, 'Run "vultisig chains" to list all chains']
  )
}

export function parseAmount(input: string): number {
  if (input === 'max') return -1 // sentinel, caller handles max

  const num = Number(input)
  if (isNaN(num) || num <= 0) {
    throw Errors.invalidUsage(
      \`Invalid amount: "\${input}"\`,
      ['Amount must be a positive number', 'Use --max to send the full balance']
    )
  }
  return num
}

// EVM chains — hardcoded since SDK only exports EvmChain as a type
const EVM_CHAINS: string[] = [
  'Arbitrum', 'Base', 'Blast', 'Optimism', 'Zksync', 'Mantle',
  'Avalanche', 'CronosChain', 'BSC', 'Ethereum', 'Polygon',
  'Hyperliquid', 'Sei',
]

export function isEvmChain(chain: string): boolean {
  return EVM_CHAINS.includes(chain)
}

function findClosest(input: string, candidates: readonly string[], maxResults: number): string[] {
  const inputLower = input.toLowerCase()
  const scored = candidates
    .map(c => ({ chain: c, dist: levenshtein(inputLower, c.toLowerCase()) }))
    .filter(s => s.dist <= Math.max(3, Math.floor(input.length / 2)))
    .sort((a, b) => a.dist - b.dist)
  return scored.slice(0, maxResults).map(s => s.chain)
}

function levenshtein(a: string, b: string): number {
  const m = a.length, n = b.length
  const dp: number[][] = Array.from({ length: m + 1 }, (_, i) =>
    Array.from({ length: n + 1 }, (_, j) => (i === 0 ? j : j === 0 ? i : 0))
  )
  for (let i = 1; i <= m; i++) {
    for (let j = 1; j <= n; j++) {
      dp[i][j] = a[i - 1] === b[j - 1]
        ? dp[i - 1][j - 1]
        : 1 + Math.min(dp[i - 1][j], dp[i][j - 1], dp[i - 1][j - 1])
    }
  }
  return dp[m][n]
}
\`\`\`

**Step 2: Add export to lib/index.ts**

Add this line to \`src/lib/index.ts\`:

\`\`\`typescript
export { isEvmChain, parseAmount, resolveChain } from './validation'
\`\`\`

**Step 3: Build to verify no compile errors**

Run: \`cd /home/sean/Repos/vultisig/vultisig-sdk && yarn build:fast\`
Expected: Build succeeds

**Step 4: Commit**

\`\`\`bash
git add clients/cli/src/lib/validation.ts clients/cli/src/lib/index.ts
git commit -m "feat(cli): add lib/validation.ts with resolveChain, parseAmount, isEvmChain"
\`\`\`

---

### Task 2: Enhance withExit error handling — cleanSdkMessage()

**Files:**
- Modify: \`src/adapters/cli-runner.ts\`

**Step 1: Add cleanSdkMessage() to cli-runner.ts and wire into withExit**

Replace the existing \`withExit\` function. Add a \`cleanSdkMessage()\` function that strips known SDK noise patterns. Wire it into the catch block so raw SDK errors get cleaned before display.

In \`src/adapters/cli-runner.ts\`, replace the entire catch block in \`withExit\`:

\`\`\`typescript
import { CLIError, Errors, isCLIError } from '../lib/errors'
import { isJsonOutput, outputJsonError, printError } from '../lib/output'

const SDK_NOISE_PATTERNS: [RegExp, string][] = [
  [/ContractFunctionExecutionError:[\s\S]*?(?=\n\n|\nContract|$)/i, 'Contract call failed'],
  [/eth_sendTransaction[\s\S]*?format[\s\S]*?error/i, 'Transaction format error'],
  [/Cannot convert .* to a BigInt/i, 'Invalid amount'],
  [/NaN/i, 'Invalid numeric value'],
]

const SDK_STRIP_PATTERNS: RegExp[] = [
  /Version: @lifi\/sdk@[\d.]+/gi,
  /Docs: https:\/\/docs\.li\.fi[\S]*/gi,
  /Details: [\s\S]*$/i,
  /Contract Call:[\s\S]*?(?=\n\n)/i,
]

function cleanSdkMessage(message: string): string {
  if (!message) return message

  for (const [pattern, replacement] of SDK_NOISE_PATTERNS) {
    if (pattern.test(message)) return replacement
  }

  let cleaned = message
  for (const pattern of SDK_STRIP_PATTERNS) {
    cleaned = cleaned.replace(pattern, '').trim()
  }

  // Collapse multiple newlines
  cleaned = cleaned.replace(/\n{3,}/g, '\n\n').trim()

  return cleaned || message
}

export function withExit<T extends any[]>(handler: (...args: T) => Promise<void>): (...args: T) => Promise<void> {
  return async (...args: T) => {
    try {
      await handler(...args)
      process.exit(0)
    } catch (err: any) {
      // CLIErrors are already clean — pass through
      if (isCLIError(err)) {
        if (isJsonOutput()) {
          outputJsonError(err.message, err.code?.toString() ?? 'GENERAL_ERROR')
          process.exit(err.code ?? 1)
        }
        printError(err.format())
        process.exit(err.code ?? 1)
      }

      // Raw SDK/library errors — clean the message
      const cleanedMessage = cleanSdkMessage(err.message ?? String(err))
      const exitCode = err.exitCode ?? 1

      if (isJsonOutput()) {
        outputJsonError(cleanedMessage, err.code ?? 'GENERAL_ERROR')
        process.exit(exitCode)
      }

      printError(\`\nx \${cleanedMessage}\`)
      process.exit(exitCode)
    }
  }
}
\`\`\`

Keep the existing \`runCommand\` function unchanged.

**Step 2: Build to verify**

Run: \`cd /home/sean/Repos/vultisig/vultisig-sdk && yarn build:fast\`
Expected: Build succeeds

**Step 3: Commit**

\`\`\`bash
git add clients/cli/src/adapters/cli-runner.ts
git commit -m "feat(cli): clean SDK error messages in withExit error handler"
\`\`\`

---

### Task 3: Wire resolveChain into index.ts — replace findChainByName

**Files:**
- Modify: \`src/index.ts\`

**Step 1: Replace findChainByName import with resolveChain**

In \`src/index.ts\`, change:

\`\`\`typescript
import { findChainByName } from './interactive'
\`\`\`

to:

\`\`\`typescript
import { resolveChain } from './lib'
\`\`\`

**Step 2: Replace all findChainByName usages**

Every occurrence of \`findChainByName(x) || (x as Chain)\` becomes \`resolveChain(x)\`. Every occurrence of \`findChainByName(x)\` where the result is checked for null becomes \`resolveChain(x)\` (it throws on failure, so no null check needed).

There are ~12 call sites in index.ts. Replace each one. Examples:

- \`chain: chainStr ? findChainByName(chainStr) || (chainStr as Chain) : undefined\`
  → \`chain: chainStr ? resolveChain(chainStr) : undefined\`

- \`chain: findChainByName(chainStr) || (chainStr as Chain)\`
  → \`chain: resolveChain(chainStr)\`

- \`fromChain: findChainByName(fromChainStr) || (fromChainStr as Chain)\`
  → \`fromChain: resolveChain(fromChainStr)\`

- The \`create-from-seedphrase\` sections that loop over chain names with \`findChainByName(name)\` and skip unknowns with \`console.warn\` — keep the same skip behavior but use resolveChain in a try/catch:
  \`\`\`typescript
  for (const name of chainNames) {
    try {
      chains.push(resolveChain(name))
    } catch {
      console.warn(\`Warning: Unknown chain "\${name}" — skipping\`)
    }
  }
  \`\`\`

**Step 3: Build to verify**

Run: \`cd /home/sean/Repos/vultisig/vultisig-sdk && yarn build:fast\`
Expected: Build succeeds

**Step 4: Commit**

\`\`\`bash
git add clients/cli/src/index.ts
git commit -m "feat(cli): wire resolveChain into all CLI commands for case-insensitive chain matching"
\`\`\`

---

### Task 4: Fix chains.ts — no-op semantics for add/remove

**Files:**
- Modify: \`src/commands/chains.ts\`

**Step 1: Update executeChains to detect already-active and not-found**

Replace the \`if (options.add)\` / \`else if (options.remove)\` block in \`executeChains\`:

\`\`\`typescript
if (options.add) {
  const alreadyActive = vault.chains.includes(options.add)
  if (alreadyActive) {
    if (isJsonOutput()) {
      outputJson({ chain: options.add, action: 'already-active' })
      return
    }
    success(\`Chain \${options.add} is already active\`)
    const address = await vault.address(options.add)
    info(\`Address: \${address}\`)
    return
  }
  await vault.addChain(options.add)
  if (isJsonOutput()) {
    outputJson({ chain: options.add, action: 'added' })
    return
  }
  success(\`\\n+ Added chain: \${options.add}\`)
  const address = await vault.address(options.add)
  info(\`Address: \${address}\`)
} else if (options.remove) {
  if (!vault.chains.includes(options.remove)) {
    throw Errors.invalidUsage(
      \`Chain \${options.remove} is not active on this vault\`,
      [\`Run "vultisig chains" to see active chains\`, \`Use --add \${options.remove} to add it first\`]
    )
  }
  await vault.removeChain(options.remove)
  if (isJsonOutput()) {
    outputJson({ chain: options.remove, action: 'removed' })
    return
  }
  success(\`\\n+ Removed chain: \${options.remove}\`)
}
\`\`\`

**Step 2: Add the Errors import**

Add to chains.ts imports:

\`\`\`typescript
import { Errors } from '../lib/errors'
\`\`\`

**Step 3: Build to verify**

Run: \`cd /home/sean/Repos/vultisig/vultisig-sdk && yarn build:fast\`
Expected: Build succeeds

**Step 4: Commit**

\`\`\`bash
git add clients/cli/src/commands/chains.ts
git commit -m "feat(cli): fix chains add/remove no-op semantics — idempotent add, strict remove"
\`\`\`

---

### Task 5: Fix tokens.ts — strict remove semantics

**Files:**
- Modify: \`src/commands/tokens.ts\`

**Step 1: Add token existence check before remove**

In the \`removeToken\` function, add a check before calling \`vault.removeToken\`:

\`\`\`typescript
import { Errors } from '../lib/errors'

export async function removeToken(ctx: CommandContext, chain: Chain, tokenId: string): Promise<void> {
  const vault = await ctx.ensureActiveVault()

  const tokens = vault.getTokens(chain)
  const exists = tokens?.some(t => t.id === tokenId || t.contractAddress === tokenId)
  if (!exists) {
    throw Errors.invalidUsage(
      \`Token "\${tokenId}" not found on \${chain}\`,
      [\`Run "vultisig tokens \${chain}" to list tokens\`, 'Check the token ID or contract address']
    )
  }

  await vault.removeToken(chain, tokenId)
  success(\`\\n+ Removed token \${tokenId} from \${chain}\`)
}
\`\`\`

**Step 2: Build to verify**

Run: \`cd /home/sean/Repos/vultisig/vultisig-sdk && yarn build:fast\`
Expected: Build succeeds

**Step 3: Commit**

\`\`\`bash
git add clients/cli/src/commands/tokens.ts
git commit -m "feat(cli): strict remove semantics for tokens — error if token not found"
\`\`\`

---

### Task 6: Fix transaction.ts — amount validation + EVM memo rejection

**Files:**
- Modify: \`src/commands/transaction.ts\`

**Step 1: Replace inline validation with resolveChain/parseAmount and add EVM memo check**

In \`executeSend\`, replace the existing validation block:

\`\`\`typescript
// Old:
// if (!Object.values(Chain).includes(params.chain)) {
//   throw new Error(\`Invalid chain: \${params.chain}\`)
// }
// const isMax = params.amount === 'max'
// if (!isMax && (isNaN(parseFloat(params.amount)) || parseFloat(params.amount) <= 0)) {
//   throw new Error('Invalid amount')
// }

// New:
import { isEvmChain, parseAmount } from '../lib'
import { Errors } from '../lib/errors'

// (keep the existing Chain import for runtime use)

export async function executeSend(ctx: CommandContext, params: SendParams): Promise<TransactionResult> {
  const vault = await ctx.ensureActiveVault()

  if (!Object.values(Chain).includes(params.chain)) {
    throw new Error(\`Invalid chain: \${params.chain}\`)
  }

  // Validate amount (parseAmount throws on invalid)
  if (params.amount !== 'max') {
    parseAmount(params.amount)
  }

  // Reject memo on EVM chains
  if (params.memo && isEvmChain(params.chain)) {
    throw Errors.invalidUsage(
      'Memos are not supported on EVM chains',
      ['EVM transactions use the data field for contract calls', 'Remove the --memo flag']
    )
  }

  return sendTransaction(vault, params)
}
\`\`\`

**Step 2: Build to verify**

Run: \`cd /home/sean/Repos/vultisig/vultisig-sdk && yarn build:fast\`
Expected: Build succeeds

**Step 3: Commit**

\`\`\`bash
git add clients/cli/src/commands/transaction.ts
git commit -m "feat(cli): validate send amount and reject memos on EVM chains"
\`\`\`

---

### Task 7: Fix swap.ts — use parseAmount for validation

**Files:**
- Modify: \`src/commands/swap.ts\`

**Step 1: Replace inline amount validation in executeSwapQuote and executeSwap**

In both \`executeSwapQuote\` and \`executeSwap\`, replace the amount validation:

\`\`\`typescript
// Old:
// const isMax = options.amount === 'max'
// if (!isMax && (isNaN(options.amount as number) || (options.amount as number) <= 0)) {
//   throw new Error('Invalid amount')
// }

// New:
import { parseAmount } from '../lib'

// In executeSwapQuote:
const isMax = options.amount === 'max'
if (!isMax) {
  parseAmount(String(options.amount))
}

// In executeSwap:
const isMax = options.amount === 'max'
if (!isMax) {
  parseAmount(String(options.amount))
}
\`\`\`

**Step 2: Build to verify**

Run: \`cd /home/sean/Repos/vultisig/vultisig-sdk && yarn build:fast\`
Expected: Build succeeds

**Step 3: Commit**

\`\`\`bash
git add clients/cli/src/commands/swap.ts
git commit -m "feat(cli): use parseAmount for swap amount validation"
\`\`\`

---

### Task 8: Update interactive session.ts to use resolveChain

**Files:**
- Modify: \`src/interactive/session.ts\`

**Step 1: Replace findChainByName with resolveChain in session.ts**

Change the import:

\`\`\`typescript
// Old:
import { createCompleter, findChainByName } from './completer'

// New:
import { createCompleter } from './completer'
import { resolveChain } from '../lib'
\`\`\`

Replace each \`findChainByName(x) || (x as Chain)\` with \`resolveChain(x)\`.

For the chains command handler where \`findChainByName\` returns null and prints an error message, use a try/catch:

\`\`\`typescript
// Old:
const chain = findChainByName(args[i + 1])
if (!chain) {
  console.log(chalk.red(\`Unknown chain: \${args[i + 1]}\`))
  console.log(chalk.gray('Use tab completion to see available chains'))
  return
}
addChain = chain

// New:
try {
  addChain = resolveChain(args[i + 1])
} catch (err: any) {
  console.log(chalk.red(err.message))
  return
}
\`\`\`

Apply similar pattern for the remove case.

**Step 2: Build to verify**

Run: \`cd /home/sean/Repos/vultisig/vultisig-sdk && yarn build:fast\`
Expected: Build succeeds

**Step 3: Commit**

\`\`\`bash
git add clients/cli/src/interactive/session.ts
git commit -m "feat(cli): use resolveChain in interactive shell for consistent chain resolution"
\`\`\`

---

### Task 9: Final build and manual testing

**Step 1: Full build**

Run: \`cd /home/sean/Repos/vultisig/vultisig-sdk && yarn build:fast\`
Expected: Build succeeds with no errors

**Step 2: Test chain resolution**

Run: \`cd /home/sean/Repos/vultisig/vultisig-sdk && node clients/cli/dist/index.js balance ethereum -o json\`
Expected: Resolves "ethereum" to "Ethereum", returns balance or vault error (not chain error)

**Step 3: Test unknown chain error**

Run: \`cd /home/sean/Repos/vultisig/vultisig-sdk && node clients/cli/dist/index.js balance foobar -o json\`
Expected: JSON error output with "Unknown chain" message and suggestions

**Step 4: Test chains --add already-active**

Run: \`cd /home/sean/Repos/vultisig/vultisig-sdk && node clients/cli/dist/index.js chains --add Bitcoin -o json\`
Expected: If Bitcoin already active, returns \`{ action: "already-active" }\` with exit 0

**Step 5: Test chains --remove inactive**

Run: \`cd /home/sean/Repos/vultisig/vultisig-sdk && node clients/cli/dist/index.js chains --remove Cardano -o json\`
Expected: Error with "not active on this vault" message, exit 1

**Step 6: Commit (if any fixes needed)**

Fix any issues found, commit each fix separately.

**2026-03-22T11:55:58Z**

Implementation complete: validation.ts, cleanSdkMessage, resolveChain wired into all commands, no-op semantics fixed, tests passing
