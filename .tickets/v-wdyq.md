---
id: v-wdyq
status: closed
deps: []
links: [tic-ef85]
created: 2026-03-22T00:38:08Z
type: feature
priority: 1
assignee: Jibles
---
# CLI: Agent-friendly Vultisig CLI with keyring auth (vultiagent-cli)

## Objective

Build a standalone, agent-first CLI for Vultisig (`vultiagent-cli`, command: `vasig`) that enables AI agents like Claude Code to perform crypto operations (balance, send, swap) without interactive prompts. Includes a keyring-based auth system where humans authenticate once in a real terminal, and agents consume stored credentials transparently.

Repo: https://github.com/premiumjibles/vultiagent-cli
Published name: `vultiagent-cli` (binary: `vasig`)

## User Story

An AI agent (Claude Code, MCP server, or any automated tool) needs to check balances, send tokens, and execute swaps through Vultisig vaults without interactive password prompts or access to raw credentials.

## Design Constraints

- Depends on `@vultisig/sdk@0.5.0+` and `@vultisig/rujira@1.0.0+` from npm (not workspace links)
- Must be fully non-interactive when credentials are available — zero `inquirer` prompts in agent mode
- `vasig auth` is the ONLY interactive command — it's always run by a human in a real terminal
- All other commands fail with structured errors if auth is missing
- References `vultisig-sdk/clients/cli/` for SDK usage patterns (swap flow, send flow, balance queries) but is a fresh implementation, not a fork
- Auth system stores passwords in OS system keyring (not plaintext files) — designed so swapping to server-issued session tokens later is a small change in one function
- Must handle both encrypted and unencrypted .vult files

## Context & Findings

### Why a new repo instead of modifying the existing CLI

The existing `@vultisig/cli` at `vultisig-sdk/clients/cli/` works but is built for human-interactive use:
- Uses `inquirer` for password prompts, confirmations, vault selection — all of which hang or fail when run by an agent
- `password-manager.ts` resolution chain: cache → env var → interactive prompt (no keyring)
- Import flow (`vault-management.ts:249`) always prompts for password interactively
- No structured error output (errors are chalk-formatted strings, not JSON)
- Limited `--help` descriptions without examples

We don't have full write access to the vultisig-sdk repo. A standalone repo gives full control and avoids blocking on upstream approval. Both packages consume `@vultisig/sdk` from npm, so they're independent.

### Auth system design

**`vasig auth` flow:**
1. Searches common locations for `.vult` files: `~/.vultisig/`, `~/Documents/Vultisig/`, current directory, and any path in existing config
2. Presents found vaults, or accepts `--vault-file <path>` to skip discovery
3. Loads the vault file:
   - If unencrypted: loads directly, no decryption password needed
   - If encrypted: prompts for decryption password (masked), decrypts to validate
4. Prompts for VultiServer password (masked) — this is the password sent to the server during 2-of-2 MPC signing
5. Validates server auth works (test request to VultiServer)
6. Stores in system keyring:
   - `vultisig/<vaultId>/server` → server password
   - `vultisig/<vaultId>/decrypt` → decryption password (only if vault is encrypted)
7. Stores in `~/.vultisig/config.json`:
   - Vault file path, vault ID, vault name (not sensitive)
8. Prints summary: vault name, type, chains, "Ready for agent use"

**Credential resolution (all non-auth commands):**
```
getServerPassword(vaultId):
  1. keyring lookup (vultisig/<vaultId>/server)
  2. VAULT_PASSWORD env var (fallback for CI/testing)
  3. throw AuthRequiredError with message "Run vasig auth"

getDecryptionPassword(vaultId):
  1. keyring lookup (vultisig/<vaultId>/decrypt)
  2. VAULT_DECRYPT_PASSWORD env var
  3. throw AuthRequiredError with message "Run vasig auth"
```

**Future upgrade path (when server session tokens land via tic-ef85):**
Change `getServerPassword` to `getServerCredential` — look up session token from keyring first, fall back to password exchange. One function change, not a refactor.

### CLI agent-friendliness requirements

- Every command has `--help` with usage, required/optional args, and 2-3 examples
- `--output json` gives structured JSON with consistent schema across all commands
- `--yes` skips confirmation on send/swap (agent passes this flag)
- No interactive prompts anywhere except `vasig auth`
- Grouped help categories: Authentication, Wallet, Trading, Vault Management
- Exit codes: 0 = success, 1 = bad args/usage, 2 = auth required, 3 = network/server error
- JSON errors always include `code`, `message`, and `hint` fields

### Existing CLI patterns to reference

The existing CLI at `vultisig-sdk/clients/cli/src/commands/` has correct SDK usage patterns:
- `swap.ts` — full swap flow: quote → preview → prepare tx → sign → broadcast, handles approval payloads
- `transaction.ts` — send flow: prepare → confirm → sign → broadcast
- `balance.ts` — balance/portfolio queries
- `vault-management.ts` — import vault from .vult file (`sdk.importVault(content, password)`)
- `sign.ts` — arbitrary byte signing

These are the reference for how to correctly call `@vultisig/sdk` methods. The SDK patterns (vault.getSwapQuote, vault.prepareSwapTx, vault.sign, vault.broadcastTx) are the same regardless of which CLI wraps them.

### Keyring implementation

Use `keytar` npm package (or `@aspect-build/keytar`) for cross-platform keyring access:
- Linux: libsecret / GNOME Keyring / KDE Wallet
- macOS: Keychain
- Windows: Credential Manager

The keyring is encrypted at rest by the OS, tied to the user's login session. This is the same mechanism `gh` uses for GitHub tokens.

## Files

New repo structure (`premiumjibles/vultiagent-cli`):
```
src/
  index.ts              — Entry point, Commander.js program setup
  commands/
    auth.ts             — vasig auth, auth status, auth logout
    balance.ts          — vasig balance (portfolio view)
    send.ts             — vasig send (token transfers)
    swap.ts             — vasig swap, swap-quote, swap-chains
    vault.ts            — vasig vaults, vault info
    addresses.ts        — vasig addresses
  auth/
    credential-store.ts — Keyring read/write, env var fallback
    vault-discovery.ts  — Find .vult files on disk
    config.ts           — ~/.vultisig/config.json read/write
  lib/
    output.ts           — JSON/table output formatting
    errors.ts           — Structured error types with codes
  types.ts              — Shared types
package.json
tsconfig.json
README.md
```

Reference (read-only, for SDK usage patterns):
- `vultisig-sdk/clients/cli/src/commands/swap.ts` — swap flow
- `vultisig-sdk/clients/cli/src/commands/transaction.ts` — send flow
- `vultisig-sdk/clients/cli/src/commands/balance.ts` — balance queries
- `vultisig-sdk/clients/cli/src/commands/vault-management.ts` — vault import
- `vultisig-sdk/clients/cli/src/core/password-manager.ts` — current password resolution (what we're replacing)

## Acceptance Criteria

- [ ] `vasig auth` discovers vault files, handles encrypted/unencrypted, stores credentials in system keyring
- [ ] `vasig auth status` shows authenticated vaults
- [ ] `vasig auth logout` clears keyring entries
- [ ] `vasig balance` shows balances (table + JSON output)
- [ ] `vasig send` prepares, signs, and broadcasts a transfer with `--yes` flag
- [ ] `vasig swap` gets quote, prepares, signs, and broadcasts a swap with `--yes` flag
- [ ] `vasig addresses` shows derived addresses for active chains
- [ ] All commands work fully non-interactively when keyring credentials exist
- [ ] All commands fail with structured JSON error (code + message + hint) when auth is missing
- [ ] `vasig --help` and `vasig <command> --help` show grouped categories, args, and examples
- [ ] `--output json` works on all commands with consistent schema
- [ ] Exit codes: 0 success, 1 bad args, 2 auth required, 3 network error
- [ ] Published to npm and installable via `npm install -g vultiagent-cli`
- [ ] Lint and type-check pass

## Gotchas

- `@vultisig/sdk` WASM modules (DKLS, Schnorr) need Node.js >=20 — set in engines field
- The SDK's `Vultisig` class requires `onPasswordRequired` callback at init — wire this to the keyring credential store, NOT to an interactive prompt
- `sdk.importVault(content, password)` handles both encrypted and unencrypted — pass `undefined` for password if unencrypted
- `sdk.isVaultEncrypted(content)` exists and should be used during `vasig auth` to detect encryption
- The server password and the vault decryption password may be different — `vasig auth` should prompt for them separately
- `keytar` requires native compilation — consider `@aspect-build/keytar` or prebuild binaries for easier install
- The CLI's `coordinateFastSigning` in ServerManager sends `vault_password` in plaintext over HTTPS — this is the current server API, not something we can change
- For SecureVault (N-of-M), signing requires multiple devices via relay — initial scope should focus on FastVault (2-of-2) which is the agent-friendly path

## Notes

**2026-03-22T00:49:08Z**

# vultiagent-cli Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use /run tk:v-wdyq to implement this plan task-by-task via subagent-driven-development.

**Goal:** Build an agent-first CLI (vasig) that enables AI agents to perform crypto operations via Vultisig vaults without interactive prompts, using OS keyring for credential storage.

**Architecture:** Commander.js CLI with keyring-based auth (keytar). SDK operations (balance, send, swap) wrap @vultisig/sdk methods with structured JSON output and consistent error codes. The only interactive command is \`vasig auth\` — all others consume stored credentials transparently.

**Tech Stack:** TypeScript, Commander.js, keytar (OS keyring), @vultisig/sdk@0.5.0+, @vultisig/rujira@1.0.0+, vitest (testing), tsup (bundling)

---

### Task 1: Project Scaffolding

**Files:**
- Create: \`package.json\`
- Create: \`tsconfig.json\`
- Create: \`.gitignore\`
- Create: \`src/index.ts\` (placeholder)

**Step 1: Initialize package.json**

\`\`\`bash
cd /home/sean/Repos/vultisig/vultiagent-cli
\`\`\`

Create \`package.json\`:
\`\`\`json
{
  "name": "vultiagent-cli",
  "version": "0.1.0",
  "description": "Agent-friendly Vultisig CLI with keyring auth",
  "type": "module",
  "bin": {
    "vasig": "./dist/index.js"
  },
  "scripts": {
    "build": "tsup src/index.ts --format esm --dts --clean",
    "dev": "tsx src/index.ts",
    "test": "vitest run",
    "test:watch": "vitest",
    "lint": "tsc --noEmit",
    "prepublishOnly": "npm run build"
  },
  "engines": {
    "node": ">=20.0.0"
  },
  "keywords": ["vultisig", "crypto", "cli", "agent", "mpc"],
  "license": "MIT",
  "dependencies": {
    "@vultisig/sdk": "^0.5.0",
    "@vultisig/rujira": "^1.0.0",
    "commander": "^12.0.0",
    "keytar": "^7.9.0",
    "inquirer": "^9.0.0",
    "ora": "^8.0.0",
    "chalk": "^5.3.0"
  },
  "devDependencies": {
    "tsup": "^8.0.0",
    "tsx": "^4.0.0",
    "typescript": "^5.4.0",
    "vitest": "^2.0.0",
    "@types/node": "^20.0.0",
    "@types/inquirer": "^9.0.0"
  }
}
\`\`\`

**Step 2: Create tsconfig.json**

\`\`\`json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "outDir": "dist",
    "rootDir": "src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
\`\`\`

**Step 3: Create .gitignore**

\`\`\`
node_modules/
dist/
*.tsbuildinfo
.env
\`\`\`

**Step 4: Create placeholder entry point**

Create \`src/index.ts\`:
\`\`\`typescript
#!/usr/bin/env node
console.log('vasig placeholder')
\`\`\`

**Step 5: Install dependencies**

Run: \`npm install\`
Expected: Clean install, lockfile generated

**Step 6: Verify build**

Run: \`npm run build\`
Expected: \`dist/index.js\` created

**Step 7: Verify lint**

Run: \`npm run lint\`
Expected: No errors

**Step 8: Commit**

\`\`\`bash
git add package.json tsconfig.json .gitignore src/index.ts package-lock.json
git commit -m "chore: scaffold project with dependencies"
\`\`\`

---

### Task 2: Error Types and Exit Codes

**Files:**
- Create: \`src/lib/errors.ts\`
- Create: \`tests/lib/errors.test.ts\`

**Step 1: Write the failing tests**

Create \`tests/lib/errors.test.ts\`:
\`\`\`typescript
import { describe, it, expect } from 'vitest'
import {
  VasigError,
  UsageError,
  AuthRequiredError,
  NetworkError,
  ExitCode,
  toErrorJson,
} from '../../src/lib/errors.js'

describe('error types', () => {
  it('UsageError has exit code 1', () => {
    const err = new UsageError('bad argument')
    expect(err.exitCode).toBe(ExitCode.USAGE)
    expect(err.code).toBe('USAGE_ERROR')
    expect(err.message).toBe('bad argument')
  })

  it('AuthRequiredError has exit code 2', () => {
    const err = new AuthRequiredError()
    expect(err.exitCode).toBe(ExitCode.AUTH_REQUIRED)
    expect(err.code).toBe('AUTH_REQUIRED')
    expect(err.hint).toBe('Run: vasig auth')
  })

  it('NetworkError has exit code 3', () => {
    const err = new NetworkError('timeout')
    expect(err.exitCode).toBe(ExitCode.NETWORK)
    expect(err.code).toBe('NETWORK_ERROR')
  })

  it('toErrorJson returns structured JSON', () => {
    const err = new AuthRequiredError()
    const json = toErrorJson(err)
    expect(json).toEqual({
      ok: false,
      error: {
        code: 'AUTH_REQUIRED',
        message: 'Authentication required. Run vasig auth to set up credentials.',
        hint: 'Run: vasig auth',
      },
    })
  })

  it('toErrorJson handles plain Error', () => {
    const json = toErrorJson(new Error('something broke'))
    expect(json.ok).toBe(false)
    expect(json.error.code).toBe('UNKNOWN_ERROR')
    expect(json.error.message).toBe('something broke')
  })
})
\`\`\`

**Step 2: Run test to verify it fails**

Run: \`npx vitest run tests/lib/errors.test.ts\`
Expected: FAIL — modules not found

**Step 3: Implement error types**

Create \`src/lib/errors.ts\`:
\`\`\`typescript
export enum ExitCode {
  SUCCESS = 0,
  USAGE = 1,
  AUTH_REQUIRED = 2,
  NETWORK = 3,
}

export abstract class VasigError extends Error {
  abstract readonly exitCode: ExitCode
  abstract readonly code: string
  readonly hint?: string

  constructor(message: string, hint?: string) {
    super(message)
    this.name = this.constructor.name
    this.hint = hint
  }
}

export class UsageError extends VasigError {
  readonly exitCode = ExitCode.USAGE
  readonly code = 'USAGE_ERROR'

  constructor(message: string, hint?: string) {
    super(message, hint)
  }
}

export class AuthRequiredError extends VasigError {
  readonly exitCode = ExitCode.AUTH_REQUIRED
  readonly code = 'AUTH_REQUIRED'

  constructor(message?: string) {
    super(
      message ?? 'Authentication required. Run vasig auth to set up credentials.',
      'Run: vasig auth'
    )
  }
}

export class NetworkError extends VasigError {
  readonly exitCode = ExitCode.NETWORK
  readonly code = 'NETWORK_ERROR'

  constructor(message: string, hint?: string) {
    super(message, hint)
  }
}

export interface ErrorJson {
  ok: false
  error: {
    code: string
    message: string
    hint?: string
  }
}

export function toErrorJson(err: Error): ErrorJson {
  if (err instanceof VasigError) {
    return {
      ok: false,
      error: {
        code: err.code,
        message: err.message,
        hint: err.hint,
      },
    }
  }
  return {
    ok: false,
    error: {
      code: 'UNKNOWN_ERROR',
      message: err.message,
    },
  }
}
\`\`\`

**Step 4: Run test to verify it passes**

Run: \`npx vitest run tests/lib/errors.test.ts\`
Expected: PASS (all 5 tests)

**Step 5: Commit**

\`\`\`bash
git add src/lib/errors.ts tests/lib/errors.test.ts
git commit -m "feat: add structured error types with exit codes"
\`\`\`

---

### Task 3: Output Formatting

**Files:**
- Create: \`src/lib/output.ts\`
- Create: \`tests/lib/output.test.ts\`

**Step 1: Write the failing tests**

Create \`tests/lib/output.test.ts\`:
\`\`\`typescript
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest'
import { OutputFormat, formatOutput, printResult, printError } from '../../src/lib/output.js'
import { AuthRequiredError, toErrorJson } from '../../src/lib/errors.js'

describe('output formatting', () => {
  let stdoutSpy: ReturnType<typeof vi.spyOn>
  let stderrSpy: ReturnType<typeof vi.spyOn>

  beforeEach(() => {
    stdoutSpy = vi.spyOn(process.stdout, 'write').mockImplementation(() => true)
    stderrSpy = vi.spyOn(process.stderr, 'write').mockImplementation(() => true)
  })

  afterEach(() => {
    stdoutSpy.mockRestore()
    stderrSpy.mockRestore()
  })

  it('formatOutput json returns stringified JSON', () => {
    const data = { balance: '1.5', chain: 'ETH' }
    const result = formatOutput(data, 'json')
    expect(JSON.parse(result)).toEqual({ ok: true, data })
  })

  it('formatOutput table returns human-readable text', () => {
    const data = { balance: '1.5', chain: 'ETH' }
    const result = formatOutput(data, 'table')
    expect(result).toContain('balance')
    expect(result).toContain('1.5')
  })

  it('printResult writes to stdout with newline', () => {
    printResult({ value: 1 }, 'json')
    expect(stdoutSpy).toHaveBeenCalled()
    const output = stdoutSpy.mock.calls[0][0] as string
    expect(JSON.parse(output)).toEqual({ ok: true, data: { value: 1 } })
  })

  it('printError writes JSON error to stderr', () => {
    const err = new AuthRequiredError()
    printError(err, 'json')
    expect(stderrSpy).toHaveBeenCalled()
    const output = stderrSpy.mock.calls[0][0] as string
    const parsed = JSON.parse(output)
    expect(parsed.ok).toBe(false)
    expect(parsed.error.code).toBe('AUTH_REQUIRED')
  })
})
\`\`\`

**Step 2: Run test to verify it fails**

Run: \`npx vitest run tests/lib/output.test.ts\`
Expected: FAIL — module not found

**Step 3: Implement output formatting**

Create \`src/lib/output.ts\`:
\`\`\`typescript
import { VasigError, toErrorJson } from './errors.js'

export type OutputFormat = 'json' | 'table'

export function formatOutput(data: unknown, format: OutputFormat): string {
  if (format === 'json') {
    return JSON.stringify({ ok: true, data }, null, 2)
  }
  return formatTable(data)
}

function formatTable(data: unknown): string {
  if (data === null || data === undefined) return ''
  if (Array.isArray(data)) {
    if (data.length === 0) return '(empty)'
    const keys = Object.keys(data[0])
    const header = keys.join('\t')
    const rows = data.map((row) => keys.map((k) => String(row[k] ?? '')).join('\t'))
    return [header, ...rows].join('\n')
  }
  if (typeof data === 'object') {
    return Object.entries(data as Record<string, unknown>)
      .map(([k, v]) => \`\${k}: \${typeof v === 'object' ? JSON.stringify(v) : String(v)}\`)
      .join('\n')
  }
  return String(data)
}

export function printResult(data: unknown, format: OutputFormat): void {
  process.stdout.write(formatOutput(data, format) + '\n')
}

export function printError(err: Error, format: OutputFormat): void {
  if (format === 'json') {
    process.stderr.write(JSON.stringify(toErrorJson(err), null, 2) + '\n')
  } else {
    const prefix = err instanceof VasigError ? \`Error [\${err.code}]\` : 'Error'
    process.stderr.write(\`\${prefix}: \${err.message}\n\`)
    if (err instanceof VasigError && err.hint) {
      process.stderr.write(\`Hint: \${err.hint}\n\`)
    }
  }
}
\`\`\`

**Step 4: Run test to verify it passes**

Run: \`npx vitest run tests/lib/output.test.ts\`
Expected: PASS (all 4 tests)

**Step 5: Commit**

\`\`\`bash
git add src/lib/output.ts tests/lib/output.test.ts
git commit -m "feat: add output formatting (JSON and table)"
\`\`\`

---

### Task 4: Credential Store (Keyring + Env Var Fallback)

**Files:**
- Create: \`src/auth/credential-store.ts\`
- Create: \`tests/auth/credential-store.test.ts\`

**Step 1: Write the failing tests**

Create \`tests/auth/credential-store.test.ts\`:
\`\`\`typescript
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest'

// Mock keytar before importing credential-store
vi.mock('keytar', () => ({
  default: {
    getPassword: vi.fn(),
    setPassword: vi.fn(),
    deletePassword: vi.fn(),
  },
}))

import keytar from 'keytar'
import {
  getServerPassword,
  getDecryptionPassword,
  setServerPassword,
  setDecryptionPassword,
  clearCredentials,
  SERVICE_NAME,
} from '../../src/auth/credential-store.js'
import { AuthRequiredError } from '../../src/lib/errors.js'

const mockedKeytar = vi.mocked(keytar)

describe('credential-store', () => {
  const vaultId = 'vault-abc-123'

  beforeEach(() => {
    vi.clearAllMocks()
    delete process.env.VAULT_PASSWORD
    delete process.env.VAULT_DECRYPT_PASSWORD
  })

  describe('getServerPassword', () => {
    it('returns keyring value when available', async () => {
      mockedKeytar.getPassword.mockResolvedValue('keyring-pw')
      const result = await getServerPassword(vaultId)
      expect(result).toBe('keyring-pw')
      expect(mockedKeytar.getPassword).toHaveBeenCalledWith(SERVICE_NAME, \`\${vaultId}/server\`)
    })

    it('falls back to VAULT_PASSWORD env var', async () => {
      mockedKeytar.getPassword.mockResolvedValue(null)
      process.env.VAULT_PASSWORD = 'env-pw'
      const result = await getServerPassword(vaultId)
      expect(result).toBe('env-pw')
    })

    it('throws AuthRequiredError when no credential found', async () => {
      mockedKeytar.getPassword.mockResolvedValue(null)
      await expect(getServerPassword(vaultId)).rejects.toThrow(AuthRequiredError)
    })
  })

  describe('getDecryptionPassword', () => {
    it('returns keyring value when available', async () => {
      mockedKeytar.getPassword.mockResolvedValue('decrypt-pw')
      const result = await getDecryptionPassword(vaultId)
      expect(result).toBe('decrypt-pw')
      expect(mockedKeytar.getPassword).toHaveBeenCalledWith(SERVICE_NAME, \`\${vaultId}/decrypt\`)
    })

    it('falls back to VAULT_DECRYPT_PASSWORD env var', async () => {
      mockedKeytar.getPassword.mockResolvedValue(null)
      process.env.VAULT_DECRYPT_PASSWORD = 'env-decrypt'
      const result = await getDecryptionPassword(vaultId)
      expect(result).toBe('env-decrypt')
    })

    it('throws AuthRequiredError when no credential found', async () => {
      mockedKeytar.getPassword.mockResolvedValue(null)
      await expect(getDecryptionPassword(vaultId)).rejects.toThrow(AuthRequiredError)
    })
  })

  describe('setServerPassword', () => {
    it('stores in keyring', async () => {
      await setServerPassword(vaultId, 'my-pw')
      expect(mockedKeytar.setPassword).toHaveBeenCalledWith(SERVICE_NAME, \`\${vaultId}/server\`, 'my-pw')
    })
  })

  describe('setDecryptionPassword', () => {
    it('stores in keyring', async () => {
      await setDecryptionPassword(vaultId, 'my-decrypt')
      expect(mockedKeytar.setPassword).toHaveBeenCalledWith(SERVICE_NAME, \`\${vaultId}/decrypt\`, 'my-decrypt')
    })
  })

  describe('clearCredentials', () => {
    it('deletes both keyring entries', async () => {
      mockedKeytar.deletePassword.mockResolvedValue(true)
      await clearCredentials(vaultId)
      expect(mockedKeytar.deletePassword).toHaveBeenCalledWith(SERVICE_NAME, \`\${vaultId}/server\`)
      expect(mockedKeytar.deletePassword).toHaveBeenCalledWith(SERVICE_NAME, \`\${vaultId}/decrypt\`)
    })
  })
})
\`\`\`

**Step 2: Run test to verify it fails**

Run: \`npx vitest run tests/auth/credential-store.test.ts\`
Expected: FAIL — module not found

**Step 3: Implement credential store**

Create \`src/auth/credential-store.ts\`:
\`\`\`typescript
import keytar from 'keytar'
import { AuthRequiredError } from '../lib/errors.js'

export const SERVICE_NAME = 'vultisig'

export async function getServerPassword(vaultId: string): Promise<string> {
  const fromKeyring = await keytar.getPassword(SERVICE_NAME, \`\${vaultId}/server\`)
  if (fromKeyring) return fromKeyring

  const fromEnv = process.env.VAULT_PASSWORD
  if (fromEnv) return fromEnv

  throw new AuthRequiredError()
}

export async function getDecryptionPassword(vaultId: string): Promise<string> {
  const fromKeyring = await keytar.getPassword(SERVICE_NAME, \`\${vaultId}/decrypt\`)
  if (fromKeyring) return fromKeyring

  const fromEnv = process.env.VAULT_DECRYPT_PASSWORD
  if (fromEnv) return fromEnv

  throw new AuthRequiredError()
}

export async function setServerPassword(vaultId: string, password: string): Promise<void> {
  await keytar.setPassword(SERVICE_NAME, \`\${vaultId}/server\`, password)
}

export async function setDecryptionPassword(vaultId: string, password: string): Promise<void> {
  await keytar.setPassword(SERVICE_NAME, \`\${vaultId}/decrypt\`, password)
}

export async function clearCredentials(vaultId: string): Promise<void> {
  await keytar.deletePassword(SERVICE_NAME, \`\${vaultId}/server\`)
  await keytar.deletePassword(SERVICE_NAME, \`\${vaultId}/decrypt\`)
}
\`\`\`

**Step 4: Run test to verify it passes**

Run: \`npx vitest run tests/auth/credential-store.test.ts\`
Expected: PASS (all 8 tests)

**Step 5: Commit**

\`\`\`bash
git add src/auth/credential-store.ts tests/auth/credential-store.test.ts
git commit -m "feat: add keyring credential store with env var fallback"
\`\`\`

---

### Task 5: Vault Discovery and Config

**Files:**
- Create: \`src/auth/vault-discovery.ts\`
- Create: \`src/auth/config.ts\`
- Create: \`tests/auth/vault-discovery.test.ts\`
- Create: \`tests/auth/config.test.ts\`

**Step 1: Write the failing tests for vault-discovery**

Create \`tests/auth/vault-discovery.test.ts\`:
\`\`\`typescript
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { discoverVaultFiles } from '../../src/auth/vault-discovery.js'
import * as fs from 'node:fs/promises'
import * as os from 'node:os'
import * as path from 'node:path'

vi.mock('node:fs/promises')
const mockedFs = vi.mocked(fs)

describe('vault-discovery', () => {
  beforeEach(() => {
    vi.clearAllMocks()
  })

  it('finds .vult files in standard locations', async () => {
    const home = os.homedir()
    // Mock readdir to return .vult files in the first search path
    mockedFs.readdir.mockImplementation(async (dirPath) => {
      const dir = String(dirPath)
      if (dir === path.join(home, '.vultisig')) {
        return ['myvault.vult', 'other.txt'] as any
      }
      throw new Error('ENOENT')
    })
    mockedFs.stat.mockResolvedValue({ isFile: () => true } as any)

    const files = await discoverVaultFiles()
    expect(files.length).toBeGreaterThanOrEqual(1)
    expect(files[0]).toContain('.vult')
  })

  it('returns empty array when no vaults found', async () => {
    mockedFs.readdir.mockRejectedValue(new Error('ENOENT'))
    const files = await discoverVaultFiles()
    expect(files).toEqual([])
  })
})
\`\`\`

**Step 2: Write the failing tests for config**

Create \`tests/auth/config.test.ts\`:
\`\`\`typescript
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { loadConfig, saveConfig, type VasigConfig } from '../../src/auth/config.js'
import * as fs from 'node:fs/promises'

vi.mock('node:fs/promises')
const mockedFs = vi.mocked(fs)

describe('config', () => {
  beforeEach(() => {
    vi.clearAllMocks()
  })

  it('loadConfig returns default when file missing', async () => {
    mockedFs.readFile.mockRejectedValue(new Error('ENOENT'))
    const config = await loadConfig()
    expect(config).toEqual({ vaults: [] })
  })

  it('loadConfig parses existing config', async () => {
    const existing: VasigConfig = {
      vaults: [{ id: 'v1', name: 'MyVault', filePath: '/path/to/vault.vult' }],
    }
    mockedFs.readFile.mockResolvedValue(JSON.stringify(existing))
    const config = await loadConfig()
    expect(config.vaults).toHaveLength(1)
    expect(config.vaults[0].id).toBe('v1')
  })

  it('saveConfig writes JSON to disk', async () => {
    mockedFs.mkdir.mockResolvedValue(undefined)
    mockedFs.writeFile.mockResolvedValue()
    const config: VasigConfig = {
      vaults: [{ id: 'v1', name: 'Test', filePath: '/test.vult' }],
    }
    await saveConfig(config)
    expect(mockedFs.writeFile).toHaveBeenCalled()
    const written = JSON.parse(mockedFs.writeFile.mock.calls[0][1] as string)
    expect(written.vaults[0].id).toBe('v1')
  })
})
\`\`\`

**Step 3: Run tests to verify they fail**

Run: \`npx vitest run tests/auth/vault-discovery.test.ts tests/auth/config.test.ts\`
Expected: FAIL — modules not found

**Step 4: Implement vault-discovery**

Create \`src/auth/vault-discovery.ts\`:
\`\`\`typescript
import * as fs from 'node:fs/promises'
import * as path from 'node:path'
import * as os from 'node:os'

const SEARCH_DIRS = [
  path.join(os.homedir(), '.vultisig'),
  path.join(os.homedir(), 'Documents', 'Vultisig'),
]

export async function discoverVaultFiles(extraDirs: string[] = []): Promise<string[]> {
  const dirs = [...SEARCH_DIRS, process.cwd(), ...extraDirs]
  const found: string[] = []

  for (const dir of dirs) {
    try {
      const entries = await fs.readdir(dir)
      for (const entry of entries) {
        if (typeof entry === 'string' && entry.endsWith('.vult')) {
          const fullPath = path.join(dir, entry)
          try {
            const stat = await fs.stat(fullPath)
            if (stat.isFile()) found.push(fullPath)
          } catch {
            // skip inaccessible files
          }
        }
      }
    } catch {
      // directory doesn't exist or not accessible
    }
  }

  return [...new Set(found)]
}
\`\`\`

**Step 5: Implement config**

Create \`src/auth/config.ts\`:
\`\`\`typescript
import * as fs from 'node:fs/promises'
import * as path from 'node:path'
import * as os from 'node:os'

const CONFIG_DIR = path.join(os.homedir(), '.vultisig')
const CONFIG_PATH = path.join(CONFIG_DIR, 'config.json')

export interface VaultEntry {
  id: string
  name: string
  filePath: string
}

export interface VasigConfig {
  vaults: VaultEntry[]
}

export async function loadConfig(): Promise<VasigConfig> {
  try {
    const raw = await fs.readFile(CONFIG_PATH, 'utf-8')
    return JSON.parse(raw) as VasigConfig
  } catch {
    return { vaults: [] }
  }
}

export async function saveConfig(config: VasigConfig): Promise<void> {
  await fs.mkdir(CONFIG_DIR, { recursive: true })
  await fs.writeFile(CONFIG_PATH, JSON.stringify(config, null, 2), 'utf-8')
}

export function getConfigPath(): string {
  return CONFIG_PATH
}
\`\`\`

**Step 6: Run tests to verify they pass**

Run: \`npx vitest run tests/auth/vault-discovery.test.ts tests/auth/config.test.ts\`
Expected: PASS (all 5 tests)

**Step 7: Commit**

\`\`\`bash
git add src/auth/vault-discovery.ts src/auth/config.ts tests/auth/vault-discovery.test.ts tests/auth/config.test.ts
git commit -m "feat: add vault discovery and config persistence"
\`\`\`

---

### Task 6: Shared Types

**Files:**
- Create: \`src/types.ts\`

**Step 1: Create shared types**

Create \`src/types.ts\`:
\`\`\`typescript
import type { Vultisig } from '@vultisig/sdk'

export type OutputFormat = 'json' | 'table'

export interface CommandContext {
  sdk: InstanceType<typeof Vultisig>
  format: OutputFormat
}

export interface SuccessJson<T = unknown> {
  ok: true
  data: T
}

export interface BalanceResult {
  chain: string
  symbol: string
  amount: string
  fiatValue?: string
  fiatCurrency?: string
}

export interface SendResult {
  txHash: string
  chain: string
  explorerUrl: string
}

export interface SwapQuoteResult {
  fromChain: string
  fromToken: string
  toChain: string
  toToken: string
  inputAmount: string
  estimatedOutput: string
  provider: string
}

export interface SwapResult {
  txHash: string
  chain: string
  explorerUrl: string
  approvalTxHash?: string
}

export interface AddressResult {
  chain: string
  address: string
}

export interface VaultInfoResult {
  id: string
  name: string
  type: string
  chains: string[]
  isEncrypted: boolean
  threshold: number
  totalSigners: number
}
\`\`\`

**Step 2: Verify lint passes**

Run: \`npm run lint\`
Expected: No errors

**Step 3: Commit**

\`\`\`bash
git add src/types.ts
git commit -m "feat: add shared types for command results"
\`\`\`

---

### Task 7: CLI Entry Point and Commander Setup

**Files:**
- Modify: \`src/index.ts\`

**Step 1: Write the failing test**

Create \`tests/cli.test.ts\`:
\`\`\`typescript
import { describe, it, expect } from 'vitest'
import { execSync } from 'node:child_process'

describe('CLI entry point', () => {
  it('shows help with --help flag', () => {
    const output = execSync('npx tsx src/index.ts --help', {
      encoding: 'utf-8',
      cwd: process.cwd(),
    })
    expect(output).toContain('vasig')
    expect(output).toContain('Authentication')
  })

  it('shows version with --version flag', () => {
    const output = execSync('npx tsx src/index.ts --version', {
      encoding: 'utf-8',
      cwd: process.cwd(),
    })
    expect(output.trim()).toMatch(/^\d+\.\d+\.\d+$/)
  })
})
\`\`\`

**Step 2: Run test to verify it fails**

Run: \`npx vitest run tests/cli.test.ts\`
Expected: FAIL — help output doesn't match

**Step 3: Implement CLI entry point**

Rewrite \`src/index.ts\`:
\`\`\`typescript
#!/usr/bin/env node
import { Command } from 'commander'
import { readFileSync } from 'node:fs'
import { fileURLToPath } from 'node:url'
import { dirname, join } from 'node:path'
import { VasigError, ExitCode } from './lib/errors.js'
import { printError } from './lib/output.js'
import type { OutputFormat } from './lib/output.js'

const __dirname = dirname(fileURLToPath(import.meta.url))
const pkg = JSON.parse(readFileSync(join(__dirname, '..', 'package.json'), 'utf-8'))

const program = new Command()

program
  .name('vasig')
  .description('Agent-friendly Vultisig CLI with keyring auth')
  .version(pkg.version)
  .option('--output <format>', 'output format (json, table)', 'table')

// --- Authentication ---
const authGroup = program
  .command('auth')
  .description('[Authentication] Authenticate a vault for agent use')

authGroup
  .command('setup', { isDefault: true })
  .description('Set up vault credentials in system keyring')
  .option('--vault-file <path>', 'path to .vult file (skip discovery)')
  .action(async () => {
    const { registerAuthCommand } = await import('./commands/auth.js')
    await registerAuthCommand(program)
  })

authGroup
  .command('status')
  .description('Show authenticated vaults')
  .action(async () => {
    const { registerAuthStatusCommand } = await import('./commands/auth.js')
    await registerAuthStatusCommand(program)
  })

authGroup
  .command('logout')
  .description('Clear stored credentials')
  .option('--vault-id <id>', 'specific vault to log out')
  .option('--all', 'log out all vaults')
  .action(async () => {
    const { registerAuthLogoutCommand } = await import('./commands/auth.js')
    await registerAuthLogoutCommand(program)
  })

// --- Wallet ---
program
  .command('balance')
  .description('[Wallet] Show vault balances')
  .option('--chain <chain>', 'filter by chain')
  .option('--include-tokens', 'include token balances')
  .action(async (_opts) => {
    const { balanceCommand } = await import('./commands/balance.js')
    await balanceCommand(_opts, getFormat())
  })

program
  .command('addresses')
  .description('[Wallet] Show derived addresses for active chains')
  .action(async () => {
    const { addressesCommand } = await import('./commands/addresses.js')
    await addressesCommand(getFormat())
  })

// --- Trading ---
program
  .command('send')
  .description('[Trading] Send tokens to an address')
  .requiredOption('--chain <chain>', 'blockchain (e.g., Ethereum, Bitcoin)')
  .requiredOption('--to <address>', 'recipient address')
  .requiredOption('--amount <amount>', 'amount to send (or "max")')
  .option('--token <tokenId>', 'token contract address')
  .option('--memo <memo>', 'transaction memo')
  .option('--yes', 'skip confirmation prompt')
  .action(async (opts) => {
    const { sendCommand } = await import('./commands/send.js')
    await sendCommand(opts, getFormat())
  })

program
  .command('swap')
  .description('[Trading] Execute a token swap')
  .requiredOption('--from <chain:token>', 'source chain and optional token (e.g., Ethereum or Ethereum:USDC)')
  .requiredOption('--to <chain:token>', 'destination chain and optional token')
  .requiredOption('--amount <amount>', 'amount to swap')
  .option('--yes', 'skip confirmation prompt')
  .action(async (opts) => {
    const { swapCommand } = await import('./commands/swap.js')
    await swapCommand(opts, getFormat())
  })

program
  .command('swap-quote')
  .description('[Trading] Get a swap quote without executing')
  .requiredOption('--from <chain:token>', 'source chain and optional token')
  .requiredOption('--to <chain:token>', 'destination chain and optional token')
  .requiredOption('--amount <amount>', 'amount to swap')
  .action(async (opts) => {
    const { swapQuoteCommand } = await import('./commands/swap.js')
    await swapQuoteCommand(opts, getFormat())
  })

program
  .command('swap-chains')
  .description('[Trading] List supported swap chains')
  .action(async () => {
    const { swapChainsCommand } = await import('./commands/swap.js')
    await swapChainsCommand(getFormat())
  })

// --- Vault Management ---
program
  .command('vaults')
  .description('[Vault Management] List authenticated vaults')
  .action(async () => {
    const { vaultsCommand } = await import('./commands/vault.js')
    await vaultsCommand(getFormat())
  })

program
  .command('vault-info')
  .description('[Vault Management] Show details of active vault')
  .action(async () => {
    const { vaultInfoCommand } = await import('./commands/vault.js')
    await vaultInfoCommand(getFormat())
  })

function getFormat(): OutputFormat {
  return program.opts().output as OutputFormat
}

// Global error handler
program.exitOverride()

async function main() {
  try {
    await program.parseAsync(process.argv)
  } catch (err: unknown) {
    if (err instanceof VasigError) {
      printError(err, getFormat())
      process.exit(err.exitCode)
    }
    if (err instanceof Error && err.message !== '') {
      printError(err, getFormat())
    }
    process.exit(ExitCode.USAGE)
  }
}

main()
\`\`\`

**Step 4: Run test to verify it passes**

Run: \`npx vitest run tests/cli.test.ts\`
Expected: PASS (both tests)

**Step 5: Commit**

\`\`\`bash
git add src/index.ts tests/cli.test.ts
git commit -m "feat: add CLI entry point with commander setup"
\`\`\`

---

### Task 8: Auth Command (Interactive — vasig auth, auth status, auth logout)

**Files:**
- Create: \`src/commands/auth.ts\`
- Create: \`tests/commands/auth.test.ts\`

This is the ONLY interactive command. It uses inquirer for prompts.

**Step 1: Write the failing tests**

Create \`tests/commands/auth.test.ts\`:
\`\`\`typescript
import { describe, it, expect, vi, beforeEach } from 'vitest'

vi.mock('keytar', () => ({
  default: {
    getPassword: vi.fn(),
    setPassword: vi.fn(),
    deletePassword: vi.fn(),
  },
}))

vi.mock('node:fs/promises', async (importOriginal) => {
  const actual = await importOriginal<typeof import('node:fs/promises')>()
  return { ...actual, readFile: vi.fn(), writeFile: vi.fn(), mkdir: vi.fn(), readdir: vi.fn(), stat: vi.fn() }
})

vi.mock('inquirer', () => ({
  default: { prompt: vi.fn() },
}))

vi.mock('@vultisig/sdk', () => ({
  Vultisig: vi.fn().mockImplementation(() => ({
    initialize: vi.fn(),
    importVault: vi.fn().mockResolvedValue({
      id: 'vault-123',
      name: 'TestVault',
      type: 'fast',
      isEncrypted: false,
      chains: ['Ethereum', 'Bitcoin'],
    }),
    isVaultEncrypted: vi.fn().mockReturnValue(false),
    dispose: vi.fn(),
  })),
}))

import keytar from 'keytar'
import inquirer from 'inquirer'
import * as fs from 'node:fs/promises'
import { authSetup, authStatus, authLogout } from '../../src/commands/auth.js'

const mockedKeytar = vi.mocked(keytar)
const mockedInquirer = vi.mocked(inquirer)
const mockedFs = vi.mocked(fs)

describe('auth commands', () => {
  beforeEach(() => {
    vi.clearAllMocks()
  })

  describe('authSetup', () => {
    it('imports unencrypted vault and stores server password', async () => {
      mockedFs.readFile.mockResolvedValue('vault-file-content')
      mockedInquirer.prompt
        .mockResolvedValueOnce({ serverPassword: 'server-pass' }) // server password prompt

      const result = await authSetup({ vaultFile: '/path/vault.vult' })

      expect(result.vaultId).toBe('vault-123')
      expect(result.vaultName).toBe('TestVault')
      expect(mockedKeytar.setPassword).toHaveBeenCalledWith(
        'vultisig', 'vault-123/server', 'server-pass'
      )
    })
  })

  describe('authStatus', () => {
    it('returns configured vaults from config', async () => {
      mockedFs.readFile.mockResolvedValue(JSON.stringify({
        vaults: [{ id: 'v1', name: 'Vault1', filePath: '/test.vult' }],
      }))
      mockedKeytar.getPassword.mockResolvedValue('exists')

      const result = await authStatus()
      expect(result).toHaveLength(1)
      expect(result[0].name).toBe('Vault1')
      expect(result[0].hasCredentials).toBe(true)
    })
  })

  describe('authLogout', () => {
    it('clears credentials for a specific vault', async () => {
      mockedFs.readFile.mockResolvedValue(JSON.stringify({
        vaults: [{ id: 'v1', name: 'Vault1', filePath: '/test.vult' }],
      }))
      mockedFs.mkdir.mockResolvedValue(undefined)
      mockedFs.writeFile.mockResolvedValue()
      mockedKeytar.deletePassword.mockResolvedValue(true)

      await authLogout({ vaultId: 'v1' })
      expect(mockedKeytar.deletePassword).toHaveBeenCalledWith('vultisig', 'v1/server')
      expect(mockedKeytar.deletePassword).toHaveBeenCalledWith('vultisig', 'v1/decrypt')
    })
  })
})
\`\`\`

**Step 2: Run test to verify it fails**

Run: \`npx vitest run tests/commands/auth.test.ts\`
Expected: FAIL — module not found

**Step 3: Implement auth command**

Create \`src/commands/auth.ts\`:
\`\`\`typescript
import * as fs from 'node:fs/promises'
import inquirer from 'inquirer'
import { Vultisig } from '@vultisig/sdk'
import {
  setServerPassword,
  setDecryptionPassword,
  clearCredentials,
  getServerPassword,
} from '../auth/credential-store.js'
import { loadConfig, saveConfig } from '../auth/config.js'
import { discoverVaultFiles } from '../auth/vault-discovery.js'
import { printResult } from '../lib/output.js'
import type { OutputFormat } from '../lib/output.js'
import type { Command } from 'commander'

interface AuthSetupOpts {
  vaultFile?: string
}

export async function authSetup(opts: AuthSetupOpts): Promise<{ vaultId: string; vaultName: string }> {
  let vaultFilePath = opts.vaultFile

  // Discover or select vault file
  if (!vaultFilePath) {
    const config = await loadConfig()
    const extraDirs = config.vaults.map((v) => v.filePath).filter(Boolean)
    const files = await discoverVaultFiles(extraDirs.map((f) => f.replace(/\/[^/]+$/, '')))

    if (files.length === 0) {
      throw new Error('No .vult files found. Use --vault-file <path> to specify one.')
    }

    if (files.length === 1) {
      vaultFilePath = files[0]
    } else {
      const { selected } = await inquirer.prompt([
        {
          type: 'list',
          name: 'selected',
          message: 'Select a vault file:',
          choices: files,
        },
      ])
      vaultFilePath = selected
    }
  }

  // Read vault file
  const vaultContent = await fs.readFile(vaultFilePath!, 'utf-8')

  // Create SDK instance to check encryption and import
  const sdk = new Vultisig({})
  const isEncrypted = sdk.isVaultEncrypted(vaultContent)

  let decryptPassword: string | undefined
  if (isEncrypted) {
    const { password } = await inquirer.prompt([
      {
        type: 'password',
        name: 'password',
        message: 'Enter vault decryption password:',
        mask: '*',
      },
    ])
    decryptPassword = password
  }

  // Import vault to validate and get vault info
  const vault = await sdk.importVault(vaultContent, decryptPassword)

  // Prompt for server password
  const { serverPassword } = await inquirer.prompt([
    {
      type: 'password',
      name: 'serverPassword',
      message: 'Enter VultiServer password (for 2-of-2 signing):',
      mask: '*',
    },
  ])

  // Store credentials in keyring
  await setServerPassword(vault.id, serverPassword)
  if (isEncrypted && decryptPassword) {
    await setDecryptionPassword(vault.id, decryptPassword)
  }

  // Update config
  const config = await loadConfig()
  const existing = config.vaults.findIndex((v) => v.id === vault.id)
  const entry = { id: vault.id, name: vault.name, filePath: vaultFilePath! }
  if (existing >= 0) {
    config.vaults[existing] = entry
  } else {
    config.vaults.push(entry)
  }
  await saveConfig(config)

  if (typeof sdk.dispose === 'function') sdk.dispose()

  return { vaultId: vault.id, vaultName: vault.name }
}

export async function authStatus(): Promise<Array<{ id: string; name: string; filePath: string; hasCredentials: boolean }>> {
  const config = await loadConfig()
  const results = []

  for (const vault of config.vaults) {
    let hasCredentials = false
    try {
      await getServerPassword(vault.id)
      hasCredentials = true
    } catch {
      // no credentials
    }
    results.push({ ...vault, hasCredentials })
  }

  return results
}

export async function authLogout(opts: { vaultId?: string; all?: boolean }): Promise<void> {
  const config = await loadConfig()

  if (opts.all) {
    for (const vault of config.vaults) {
      await clearCredentials(vault.id)
    }
    config.vaults = []
  } else if (opts.vaultId) {
    await clearCredentials(opts.vaultId)
    config.vaults = config.vaults.filter((v) => v.id !== opts.vaultId)
  }

  await saveConfig(config)
}

// Commander registration helpers (called from index.ts)
export async function registerAuthCommand(program: Command): Promise<void> {
  const opts = program.commands.find((c) => c.name() === 'auth')?.commands?.find((c) => c.name() === 'setup')?.opts() ?? {}
  const format = program.opts().output as OutputFormat
  const result = await authSetup(opts)
  printResult(result, format)
}

export async function registerAuthStatusCommand(program: Command): Promise<void> {
  const format = program.opts().output as OutputFormat
  const result = await authStatus()
  printResult(result, format)
}

export async function registerAuthLogoutCommand(program: Command): Promise<void> {
  const opts = program.commands.find((c) => c.name() === 'auth')?.commands?.find((c) => c.name() === 'logout')?.opts() ?? {}
  const format = program.opts().output as OutputFormat
  await authLogout(opts)
  printResult({ message: 'Logged out' }, format)
}
\`\`\`

**Step 4: Run test to verify it passes**

Run: \`npx vitest run tests/commands/auth.test.ts\`
Expected: PASS (all 3 tests)

**Step 5: Commit**

\`\`\`bash
git add src/commands/auth.ts tests/commands/auth.test.ts
git commit -m "feat: add auth command (setup, status, logout)"
\`\`\`

---

### Task 9: SDK Initialization Helper

**Files:**
- Create: \`src/lib/sdk.ts\`
- Create: \`tests/lib/sdk.test.ts\`

This helper initializes the SDK with the keyring-backed password callback, and loads the active vault.

**Step 1: Write the failing tests**

Create \`tests/lib/sdk.test.ts\`:
\`\`\`typescript
import { describe, it, expect, vi, beforeEach } from 'vitest'

vi.mock('keytar', () => ({
  default: {
    getPassword: vi.fn().mockResolvedValue('keyring-password'),
    setPassword: vi.fn(),
    deletePassword: vi.fn(),
  },
}))

vi.mock('node:fs/promises', async (importOriginal) => {
  const actual = await importOriginal<typeof import('node:fs/promises')>()
  return { ...actual, readFile: vi.fn(), writeFile: vi.fn(), mkdir: vi.fn() }
})

const mockVault = {
  id: 'vault-123',
  name: 'TestVault',
  type: 'fast',
  isEncrypted: true,
  chains: ['Ethereum'],
}

vi.mock('@vultisig/sdk', () => ({
  Vultisig: vi.fn().mockImplementation((opts: any) => ({
    initialize: vi.fn(),
    importVault: vi.fn().mockResolvedValue(mockVault),
    dispose: vi.fn(),
    _onPasswordRequired: opts?.onPasswordRequired,
  })),
}))

import * as fs from 'node:fs/promises'
import { createSdkWithVault } from '../../src/lib/sdk.js'

const mockedFs = vi.mocked(fs)

describe('sdk helper', () => {
  beforeEach(() => {
    vi.clearAllMocks()
  })

  it('creates SDK and imports vault from config', async () => {
    mockedFs.readFile.mockImplementation(async (path) => {
      if (String(path).endsWith('config.json')) {
        return JSON.stringify({
          vaults: [{ id: 'vault-123', name: 'TestVault', filePath: '/test.vult' }],
        })
      }
      return 'vault-file-content'
    })

    const { sdk, vault } = await createSdkWithVault()
    expect(vault.id).toBe('vault-123')
  })

  it('throws when no vaults configured', async () => {
    mockedFs.readFile.mockImplementation(async (path) => {
      if (String(path).endsWith('config.json')) {
        return JSON.stringify({ vaults: [] })
      }
      throw new Error('ENOENT')
    })

    await expect(createSdkWithVault()).rejects.toThrow()
  })
})
\`\`\`

**Step 2: Run test to verify it fails**

Run: \`npx vitest run tests/lib/sdk.test.ts\`
Expected: FAIL — module not found

**Step 3: Implement SDK helper**

Create \`src/lib/sdk.ts\`:
\`\`\`typescript
import * as fs from 'node:fs/promises'
import { Vultisig } from '@vultisig/sdk'
import { loadConfig } from '../auth/config.js'
import { getServerPassword, getDecryptionPassword } from '../auth/credential-store.js'
import { AuthRequiredError } from './errors.js'

export async function createSdkWithVault(vaultId?: string) {
  const config = await loadConfig()

  if (config.vaults.length === 0) {
    throw new AuthRequiredError('No vaults configured. Run vasig auth to set up credentials.')
  }

  const vaultEntry = vaultId
    ? config.vaults.find((v) => v.id === vaultId)
    : config.vaults[0]

  if (!vaultEntry) {
    throw new AuthRequiredError(\`Vault \${vaultId} not found. Run vasig auth to set up credentials.\`)
  }

  const sdk = new Vultisig({
    onPasswordRequired: async (id: string) => {
      return getDecryptionPassword(id)
    },
  })

  await sdk.initialize()

  const vaultContent = await fs.readFile(vaultEntry.filePath, 'utf-8')
  const isEncrypted = sdk.isVaultEncrypted(vaultContent)

  let password: string | undefined
  if (isEncrypted) {
    password = await getDecryptionPassword(vaultEntry.id)
  }

  const vault = await sdk.importVault(vaultContent, password)

  return { sdk, vault, vaultEntry }
}
\`\`\`

**Step 4: Run test to verify it passes**

Run: \`npx vitest run tests/lib/sdk.test.ts\`
Expected: PASS (both tests)

**Step 5: Commit**

\`\`\`bash
git add src/lib/sdk.ts tests/lib/sdk.test.ts
git commit -m "feat: add SDK initialization helper with keyring auth"
\`\`\`

---

### Task 10: Balance and Addresses Commands

**Files:**
- Create: \`src/commands/balance.ts\`
- Create: \`src/commands/addresses.ts\`
- Create: \`tests/commands/balance.test.ts\`
- Create: \`tests/commands/addresses.test.ts\`

**Step 1: Write the failing tests for balance**

Create \`tests/commands/balance.test.ts\`:
\`\`\`typescript
import { describe, it, expect, vi, beforeEach } from 'vitest'

const mockVault = {
  id: 'vault-123',
  name: 'TestVault',
  balance: vi.fn(),
  balances: vi.fn(),
  chains: ['Ethereum', 'Bitcoin'],
}

vi.mock('../../src/lib/sdk.js', () => ({
  createSdkWithVault: vi.fn().mockResolvedValue({
    sdk: { dispose: vi.fn() },
    vault: mockVault,
  }),
}))

import { getBalances } from '../../src/commands/balance.js'

describe('balance command', () => {
  beforeEach(() => {
    vi.clearAllMocks()
  })

  it('returns balances for all chains', async () => {
    mockVault.balances.mockResolvedValue({
      Ethereum: { chain: 'Ethereum', symbol: 'ETH', amount: '1000000000000000000', formattedAmount: '1.0', decimals: 18 },
      Bitcoin: { chain: 'Bitcoin', symbol: 'BTC', amount: '100000000', formattedAmount: '1.0', decimals: 8 },
    })

    const result = await getBalances({})
    expect(result).toHaveLength(2)
    expect(result[0].chain).toBe('Ethereum')
    expect(result[1].chain).toBe('Bitcoin')
  })

  it('filters by chain when specified', async () => {
    mockVault.balance.mockResolvedValue({
      chain: 'Ethereum', symbol: 'ETH', amount: '1000000000000000000', formattedAmount: '1.0', decimals: 18,
    })

    const result = await getBalances({ chain: 'Ethereum' })
    expect(result).toHaveLength(1)
    expect(result[0].chain).toBe('Ethereum')
  })
})
\`\`\`

**Step 2: Write the failing tests for addresses**

Create \`tests/commands/addresses.test.ts\`:
\`\`\`typescript
import { describe, it, expect, vi, beforeEach } from 'vitest'

const mockVault = {
  id: 'vault-123',
  name: 'TestVault',
  address: vi.fn(),
  chains: ['Ethereum', 'Bitcoin'],
}

vi.mock('../../src/lib/sdk.js', () => ({
  createSdkWithVault: vi.fn().mockResolvedValue({
    sdk: { dispose: vi.fn() },
    vault: mockVault,
  }),
}))

import { getAddresses } from '../../src/commands/addresses.js'

describe('addresses command', () => {
  beforeEach(() => {
    vi.clearAllMocks()
    mockVault.address.mockImplementation(async (chain: string) => {
      if (chain === 'Ethereum') return '0xabc123'
      if (chain === 'Bitcoin') return 'bc1qxyz'
      return 'unknown'
    })
  })

  it('returns addresses for all chains', async () => {
    const result = await getAddresses()
    expect(result).toHaveLength(2)
    expect(result[0]).toEqual({ chain: 'Ethereum', address: '0xabc123' })
    expect(result[1]).toEqual({ chain: 'Bitcoin', address: 'bc1qxyz' })
  })
})
\`\`\`

**Step 3: Run tests to verify they fail**

Run: \`npx vitest run tests/commands/balance.test.ts tests/commands/addresses.test.ts\`
Expected: FAIL — modules not found

**Step 4: Implement balance command**

Create \`src/commands/balance.ts\`:
\`\`\`typescript
import { createSdkWithVault } from '../lib/sdk.js'
import { printResult } from '../lib/output.js'
import { NetworkError } from '../lib/errors.js'
import type { OutputFormat } from '../lib/output.js'
import type { BalanceResult } from '../types.js'

interface BalanceOpts {
  chain?: string
  includeTokens?: boolean
}

export async function getBalances(opts: BalanceOpts): Promise<BalanceResult[]> {
  const { sdk, vault } = await createSdkWithVault()

  try {
    if (opts.chain) {
      const balance = await vault.balance(opts.chain, undefined)
      return [{
        chain: balance.chain ?? opts.chain,
        symbol: balance.symbol,
        amount: balance.formattedAmount ?? balance.amount,
        fiatValue: balance.fiatValue?.formatted,
        fiatCurrency: balance.fiatCurrency,
      }]
    }

    const balances = await vault.balances(undefined, opts.includeTokens)
    return Object.values(balances).map((b: any) => ({
      chain: b.chain,
      symbol: b.symbol,
      amount: b.formattedAmount ?? b.amount,
      fiatValue: b.fiatValue?.formatted,
      fiatCurrency: b.fiatCurrency,
    }))
  } catch (err: unknown) {
    if (err instanceof Error && (err.message.includes('network') || err.message.includes('timeout'))) {
      throw new NetworkError(err.message)
    }
    throw err
  } finally {
    if (typeof sdk.dispose === 'function') sdk.dispose()
  }
}

export async function balanceCommand(opts: BalanceOpts, format: OutputFormat): Promise<void> {
  const results = await getBalances(opts)
  printResult(results, format)
}
\`\`\`

**Step 5: Implement addresses command**

Create \`src/commands/addresses.ts\`:
\`\`\`typescript
import { createSdkWithVault } from '../lib/sdk.js'
import { printResult } from '../lib/output.js'
import type { OutputFormat } from '../lib/output.js'
import type { AddressResult } from '../types.js'

export async function getAddresses(): Promise<AddressResult[]> {
  const { sdk, vault } = await createSdkWithVault()

  try {
    const results: AddressResult[] = []
    for (const chain of vault.chains) {
      const address = await vault.address(chain)
      results.push({ chain, address })
    }
    return results
  } finally {
    if (typeof sdk.dispose === 'function') sdk.dispose()
  }
}

export async function addressesCommand(format: OutputFormat): Promise<void> {
  const results = await getAddresses()
  printResult(results, format)
}
\`\`\`

**Step 6: Run tests to verify they pass**

Run: \`npx vitest run tests/commands/balance.test.ts tests/commands/addresses.test.ts\`
Expected: PASS (all 3 tests)

**Step 7: Commit**

\`\`\`bash
git add src/commands/balance.ts src/commands/addresses.ts tests/commands/balance.test.ts tests/commands/addresses.test.ts
git commit -m "feat: add balance and addresses commands"
\`\`\`

---

### Task 11: Send Command

**Files:**
- Create: \`src/commands/send.ts\`
- Create: \`tests/commands/send.test.ts\`

**Step 1: Write the failing tests**

Create \`tests/commands/send.test.ts\`:
\`\`\`typescript
import { describe, it, expect, vi, beforeEach } from 'vitest'

const mockVault = {
  id: 'vault-123',
  name: 'TestVault',
  isEncrypted: false,
  address: vi.fn().mockResolvedValue('0xSenderAddress'),
  balance: vi.fn().mockResolvedValue({ decimals: 18, symbol: 'ETH', chain: 'Ethereum' }),
  getMaxSendAmount: vi.fn().mockResolvedValue({ maxSendable: 1000000000000000000n }),
  prepareSendTx: vi.fn().mockResolvedValue({ coin: { chain: 'Ethereum' } }),
  extractMessageHashes: vi.fn().mockResolvedValue(['0xhash1']),
  sign: vi.fn().mockResolvedValue({ signature: '0xsig', recovery: 0, format: 'ecdsa' }),
  broadcastTx: vi.fn().mockResolvedValue('0xtxhash123'),
  gas: vi.fn().mockResolvedValue({ fast: '21000', price: '50' }),
  on: vi.fn(),
  removeAllListeners: vi.fn(),
  isUnlocked: vi.fn().mockReturnValue(true),
}

vi.mock('../../src/lib/sdk.js', () => ({
  createSdkWithVault: vi.fn().mockResolvedValue({
    sdk: { dispose: vi.fn() },
    vault: mockVault,
  }),
}))

vi.mock('@vultisig/sdk', () => ({
  Vultisig: { getTxExplorerUrl: vi.fn().mockReturnValue('https://etherscan.io/tx/0xtxhash123') },
}))

vi.mock('keytar', () => ({
  default: {
    getPassword: vi.fn().mockResolvedValue('server-pw'),
    setPassword: vi.fn(),
    deletePassword: vi.fn(),
  },
}))

import { executeSend } from '../../src/commands/send.js'

describe('send command', () => {
  beforeEach(() => {
    vi.clearAllMocks()
  })

  it('sends a transaction with --yes flag', async () => {
    const result = await executeSend({
      chain: 'Ethereum',
      to: '0xRecipient',
      amount: '1.0',
      yes: true,
    })

    expect(result.txHash).toBe('0xtxhash123')
    expect(result.chain).toBe('Ethereum')
    expect(mockVault.prepareSendTx).toHaveBeenCalled()
    expect(mockVault.sign).toHaveBeenCalled()
    expect(mockVault.broadcastTx).toHaveBeenCalled()
  })

  it('sends max amount when amount is "max"', async () => {
    await executeSend({
      chain: 'Ethereum',
      to: '0xRecipient',
      amount: 'max',
      yes: true,
    })

    expect(mockVault.getMaxSendAmount).toHaveBeenCalled()
    expect(mockVault.prepareSendTx).toHaveBeenCalledWith(
      expect.objectContaining({ amount: 1000000000000000000n })
    )
  })
})
\`\`\`

**Step 2: Run test to verify it fails**

Run: \`npx vitest run tests/commands/send.test.ts\`
Expected: FAIL — module not found

**Step 3: Implement send command**

Create \`src/commands/send.ts\`:
\`\`\`typescript
import { Vultisig } from '@vultisig/sdk'
import { createSdkWithVault } from '../lib/sdk.js'
import { getServerPassword } from '../auth/credential-store.js'
import { printResult } from '../lib/output.js'
import { NetworkError } from '../lib/errors.js'
import type { OutputFormat } from '../lib/output.js'
import type { SendResult } from '../types.js'

interface SendOpts {
  chain: string
  to: string
  amount: string
  token?: string
  memo?: string
  yes?: boolean
}

export async function executeSend(opts: SendOpts): Promise<SendResult> {
  const { sdk, vault } = await createSdkWithVault()

  try {
    const address = await vault.address(opts.chain)
    const balance = await vault.balance(opts.chain, opts.token)

    const coin = {
      chain: opts.chain,
      address,
      decimals: balance.decimals,
      ticker: balance.symbol,
      id: opts.token,
    }

    let amount: bigint
    if (opts.amount === 'max') {
      const maxInfo = await vault.getMaxSendAmount({
        coin,
        receiver: opts.to,
        memo: opts.memo,
      })
      amount = maxInfo.maxSendable
    } else {
      const [whole, frac = ''] = opts.amount.split('.')
      const paddedFrac = frac.padEnd(balance.decimals, '0').slice(0, balance.decimals)
      amount = BigInt(whole || '0') * 10n ** BigInt(balance.decimals) + BigInt(paddedFrac || '0')
    }

    const payload = await vault.prepareSendTx({
      coin,
      receiver: opts.to,
      amount,
      memo: opts.memo,
    })

    // Extract hashes, sign, broadcast
    const messageHashes = await vault.extractMessageHashes(payload)
    const signature = await vault.sign(
      { transaction: payload, chain: opts.chain, messageHashes },
      {}
    )
    const txHash = await vault.broadcastTx({
      chain: opts.chain,
      keysignPayload: payload,
      signature,
    })

    const explorerUrl = Vultisig.getTxExplorerUrl(opts.chain, txHash)

    return { txHash, chain: opts.chain, explorerUrl }
  } catch (err: unknown) {
    if (err instanceof Error && (err.message.includes('network') || err.message.includes('timeout'))) {
      throw new NetworkError(err.message)
    }
    throw err
  } finally {
    if (typeof sdk.dispose === 'function') sdk.dispose()
  }
}

export async function sendCommand(opts: SendOpts, format: OutputFormat): Promise<void> {
  const result = await executeSend(opts)
  printResult(result, format)
}
\`\`\`

**Step 4: Run test to verify it passes**

Run: \`npx vitest run tests/commands/send.test.ts\`
Expected: PASS (both tests)

**Step 5: Commit**

\`\`\`bash
git add src/commands/send.ts tests/commands/send.test.ts
git commit -m "feat: add send command with max amount support"
\`\`\`

---

### Task 12: Swap Command (swap, swap-quote, swap-chains)

**Files:**
- Create: \`src/commands/swap.ts\`
- Create: \`tests/commands/swap.test.ts\`

**Step 1: Write the failing tests**

Create \`tests/commands/swap.test.ts\`:
\`\`\`typescript
import { describe, it, expect, vi, beforeEach } from 'vitest'

const mockVault = {
  id: 'vault-123',
  name: 'TestVault',
  isEncrypted: false,
  getSwapQuote: vi.fn().mockResolvedValue({
    fromCoin: { chain: 'Ethereum', ticker: 'ETH', decimals: 18 },
    toCoin: { chain: 'Bitcoin', ticker: 'BTC', decimals: 8 },
    estimatedOutput: '0.05',
    provider: 'thorchain',
  }),
  getSupportedSwapChains: vi.fn().mockResolvedValue(['Ethereum', 'Bitcoin', 'THORChain']),
  isSwapSupported: vi.fn().mockResolvedValue(true),
  prepareSwapTx: vi.fn().mockResolvedValue({
    keysignPayload: { coin: { chain: 'Ethereum' } },
    approvalPayload: null,
  }),
  extractMessageHashes: vi.fn().mockResolvedValue(['0xhash1']),
  sign: vi.fn().mockResolvedValue({ signature: '0xsig' }),
  broadcastTx: vi.fn().mockResolvedValue('0xswaptx'),
  on: vi.fn(),
  removeAllListeners: vi.fn(),
  isUnlocked: vi.fn().mockReturnValue(true),
}

vi.mock('../../src/lib/sdk.js', () => ({
  createSdkWithVault: vi.fn().mockResolvedValue({
    sdk: { dispose: vi.fn() },
    vault: mockVault,
  }),
}))

vi.mock('@vultisig/sdk', () => ({
  Vultisig: { getTxExplorerUrl: vi.fn().mockReturnValue('https://etherscan.io/tx/0xswaptx') },
}))

vi.mock('keytar', () => ({
  default: {
    getPassword: vi.fn().mockResolvedValue('server-pw'),
    setPassword: vi.fn(),
    deletePassword: vi.fn(),
  },
}))

import { getSwapQuote, getSupportedChains, executeSwap } from '../../src/commands/swap.js'

describe('swap commands', () => {
  beforeEach(() => {
    vi.clearAllMocks()
  })

  describe('swap-quote', () => {
    it('returns a swap quote', async () => {
      const result = await getSwapQuote({
        from: 'Ethereum',
        to: 'Bitcoin',
        amount: '1.0',
      })

      expect(result.fromToken).toBe('ETH')
      expect(result.toToken).toBe('BTC')
      expect(result.estimatedOutput).toBe('0.05')
      expect(result.provider).toBe('thorchain')
    })
  })

  describe('swap-chains', () => {
    it('returns supported swap chains', async () => {
      const result = await getSupportedChains()
      expect(result).toEqual(['Ethereum', 'Bitcoin', 'THORChain'])
    })
  })

  describe('swap', () => {
    it('executes a swap with --yes flag', async () => {
      const result = await executeSwap({
        from: 'Ethereum',
        to: 'Bitcoin',
        amount: '1.0',
        yes: true,
      })

      expect(result.txHash).toBe('0xswaptx')
      expect(mockVault.getSwapQuote).toHaveBeenCalled()
      expect(mockVault.prepareSwapTx).toHaveBeenCalled()
      expect(mockVault.sign).toHaveBeenCalled()
      expect(mockVault.broadcastTx).toHaveBeenCalled()
    })
  })
})
\`\`\`

**Step 2: Run test to verify it fails**

Run: \`npx vitest run tests/commands/swap.test.ts\`
Expected: FAIL — module not found

**Step 3: Implement swap command**

Create \`src/commands/swap.ts\`:
\`\`\`typescript
import { Vultisig } from '@vultisig/sdk'
import { createSdkWithVault } from '../lib/sdk.js'
import { printResult } from '../lib/output.js'
import { NetworkError } from '../lib/errors.js'
import type { OutputFormat } from '../lib/output.js'
import type { SwapQuoteResult, SwapResult } from '../types.js'

interface SwapOpts {
  from: string
  to: string
  amount: string
  yes?: boolean
}

function parseChainToken(input: string): { chain: string; token?: string } {
  const parts = input.split(':')
  return { chain: parts[0], token: parts[1] }
}

export async function getSwapQuote(opts: SwapOpts): Promise<SwapQuoteResult> {
  const { sdk, vault } = await createSdkWithVault()

  try {
    const from = parseChainToken(opts.from)
    const to = parseChainToken(opts.to)

    const quote = await vault.getSwapQuote({
      fromCoin: { chain: from.chain, token: from.token },
      toCoin: { chain: to.chain, token: to.token },
      amount: parseFloat(opts.amount),
      fiatCurrency: 'usd',
    })

    return {
      fromChain: quote.fromCoin.chain,
      fromToken: quote.fromCoin.ticker,
      toChain: quote.toCoin.chain,
      toToken: quote.toCoin.ticker,
      inputAmount: opts.amount,
      estimatedOutput: quote.estimatedOutput,
      provider: quote.provider,
    }
  } finally {
    if (typeof sdk.dispose === 'function') sdk.dispose()
  }
}

export async function getSupportedChains(): Promise<string[]> {
  const { sdk, vault } = await createSdkWithVault()

  try {
    const chains = await vault.getSupportedSwapChains()
    return [...chains]
  } finally {
    if (typeof sdk.dispose === 'function') sdk.dispose()
  }
}

export async function executeSwap(opts: SwapOpts): Promise<SwapResult> {
  const { sdk, vault } = await createSdkWithVault()

  try {
    const from = parseChainToken(opts.from)
    const to = parseChainToken(opts.to)

    // Get quote
    const quote = await vault.getSwapQuote({
      fromCoin: { chain: from.chain, token: from.token },
      toCoin: { chain: to.chain, token: to.token },
      amount: parseFloat(opts.amount),
      fiatCurrency: 'usd',
    })

    // Prepare swap tx
    const { keysignPayload, approvalPayload } = await vault.prepareSwapTx({
      fromCoin: { chain: from.chain, token: from.token },
      toCoin: { chain: to.chain, token: to.token },
      amount: parseFloat(opts.amount),
      swapQuote: quote,
      autoApprove: false,
    })

    let approvalTxHash: string | undefined

    // Handle approval if needed (e.g., ERC-20 approve)
    if (approvalPayload) {
      const approvalHashes = await vault.extractMessageHashes(approvalPayload)
      const approvalSig = await vault.sign(
        { transaction: approvalPayload, chain: from.chain, messageHashes: approvalHashes },
        {}
      )
      approvalTxHash = await vault.broadcastTx({
        chain: from.chain,
        keysignPayload: approvalPayload,
        signature: approvalSig,
      })
      // Wait for approval to confirm
      await new Promise((resolve) => setTimeout(resolve, 5000))
    }

    // Sign main swap
    const messageHashes = await vault.extractMessageHashes(keysignPayload)
    const signature = await vault.sign(
      { transaction: keysignPayload, chain: from.chain, messageHashes },
      {}
    )

    // Broadcast
    const txHash = await vault.broadcastTx({
      chain: from.chain,
      keysignPayload,
      signature,
    })

    const explorerUrl = Vultisig.getTxExplorerUrl(from.chain, txHash)

    return { txHash, chain: from.chain, explorerUrl, approvalTxHash }
  } catch (err: unknown) {
    if (err instanceof Error && (err.message.includes('network') || err.message.includes('timeout'))) {
      throw new NetworkError(err.message)
    }
    throw err
  } finally {
    if (typeof sdk.dispose === 'function') sdk.dispose()
  }
}

export async function swapCommand(opts: SwapOpts, format: OutputFormat): Promise<void> {
  const result = await executeSwap(opts)
  printResult(result, format)
}

export async function swapQuoteCommand(opts: SwapOpts, format: OutputFormat): Promise<void> {
  const result = await getSwapQuote(opts)
  printResult(result, format)
}

export async function swapChainsCommand(format: OutputFormat): Promise<void> {
  const result = await getSupportedChains()
  printResult(result, format)
}
\`\`\`

**Step 4: Run test to verify it passes**

Run: \`npx vitest run tests/commands/swap.test.ts\`
Expected: PASS (all 3 tests)

**Step 5: Commit**

\`\`\`bash
git add src/commands/swap.ts tests/commands/swap.test.ts
git commit -m "feat: add swap, swap-quote, and swap-chains commands"
\`\`\`

---

### Task 13: Vault Management Commands

**Files:**
- Create: \`src/commands/vault.ts\`
- Create: \`tests/commands/vault.test.ts\`

**Step 1: Write the failing tests**

Create \`tests/commands/vault.test.ts\`:
\`\`\`typescript
import { describe, it, expect, vi, beforeEach } from 'vitest'

const mockVault = {
  id: 'vault-123',
  name: 'TestVault',
  type: 'fast',
  chains: ['Ethereum', 'Bitcoin'],
  isEncrypted: true,
  threshold: 2,
  totalSigners: 2,
}

vi.mock('../../src/lib/sdk.js', () => ({
  createSdkWithVault: vi.fn().mockResolvedValue({
    sdk: { dispose: vi.fn() },
    vault: mockVault,
  }),
}))

vi.mock('node:fs/promises', async (importOriginal) => {
  const actual = await importOriginal<typeof import('node:fs/promises')>()
  return { ...actual, readFile: vi.fn(), writeFile: vi.fn(), mkdir: vi.fn() }
})

vi.mock('keytar', () => ({
  default: { getPassword: vi.fn().mockResolvedValue('pw'), setPassword: vi.fn(), deletePassword: vi.fn() },
}))

import * as fs from 'node:fs/promises'
import { listVaults, getVaultInfo } from '../../src/commands/vault.js'

const mockedFs = vi.mocked(fs)

describe('vault commands', () => {
  beforeEach(() => {
    vi.clearAllMocks()
  })

  describe('listVaults', () => {
    it('returns configured vaults from config', async () => {
      mockedFs.readFile.mockResolvedValue(JSON.stringify({
        vaults: [
          { id: 'v1', name: 'Vault1', filePath: '/a.vult' },
          { id: 'v2', name: 'Vault2', filePath: '/b.vult' },
        ],
      }))

      const result = await listVaults()
      expect(result).toHaveLength(2)
      expect(result[0].name).toBe('Vault1')
    })
  })

  describe('getVaultInfo', () => {
    it('returns detailed vault info', async () => {
      const result = await getVaultInfo()
      expect(result.id).toBe('vault-123')
      expect(result.name).toBe('TestVault')
      expect(result.type).toBe('fast')
      expect(result.chains).toEqual(['Ethereum', 'Bitcoin'])
      expect(result.isEncrypted).toBe(true)
      expect(result.threshold).toBe(2)
    })
  })
})
\`\`\`

**Step 2: Run test to verify it fails**

Run: \`npx vitest run tests/commands/vault.test.ts\`
Expected: FAIL — module not found

**Step 3: Implement vault commands**

Create \`src/commands/vault.ts\`:
\`\`\`typescript
import { loadConfig } from '../auth/config.js'
import { createSdkWithVault } from '../lib/sdk.js'
import { printResult } from '../lib/output.js'
import type { OutputFormat } from '../lib/output.js'
import type { VaultInfoResult } from '../types.js'

export async function listVaults(): Promise<Array<{ id: string; name: string; filePath: string }>> {
  const config = await loadConfig()
  return config.vaults
}

export async function getVaultInfo(): Promise<VaultInfoResult> {
  const { sdk, vault } = await createSdkWithVault()

  try {
    return {
      id: vault.id,
      name: vault.name,
      type: vault.type,
      chains: [...vault.chains],
      isEncrypted: vault.isEncrypted,
      threshold: vault.threshold,
      totalSigners: vault.totalSigners,
    }
  } finally {
    if (typeof sdk.dispose === 'function') sdk.dispose()
  }
}

export async function vaultsCommand(format: OutputFormat): Promise<void> {
  const result = await listVaults()
  printResult(result, format)
}

export async function vaultInfoCommand(format: OutputFormat): Promise<void> {
  const result = await getVaultInfo()
  printResult(result, format)
}
\`\`\`

**Step 4: Run test to verify it passes**

Run: \`npx vitest run tests/commands/vault.test.ts\`
Expected: PASS (both tests)

**Step 5: Commit**

\`\`\`bash
git add src/commands/vault.ts tests/commands/vault.test.ts
git commit -m "feat: add vault list and vault info commands"
\`\`\`

---

### Task 14: README and Final Polish

**Files:**
- Create: \`README.md\`

**Step 1: Create README**

Create \`README.md\`:
\`\`\`markdown
# vultiagent-cli

Agent-friendly Vultisig CLI with keyring auth. Enables AI agents to perform crypto operations (balance, send, swap) through Vultisig vaults without interactive prompts.

## Install

\\\`\\\`\\\`bash
npm install -g vultiagent-cli
\\\`\\\`\\\`

Requires Node.js >= 20.

## Quick Start

\\\`\\\`\\\`bash
# 1. Authenticate (interactive, run once by human)
vasig auth --vault-file ~/path/to/vault.vult

# 2. Agent commands (non-interactive)
vasig balance --output json
vasig send --chain Ethereum --to 0xRecipient --amount 1.0 --yes --output json
vasig swap --from Ethereum --to Bitcoin --amount 1.0 --yes --output json
\\\`\\\`\\\`

## Commands

### Authentication
- \\\`vasig auth\\\` — Set up vault credentials in system keyring
- \\\`vasig auth status\\\` — Show authenticated vaults
- \\\`vasig auth logout\\\` — Clear stored credentials

### Wallet
- \\\`vasig balance\\\` — Show vault balances
- \\\`vasig addresses\\\` — Show derived addresses

### Trading
- \\\`vasig send\\\` — Send tokens
- \\\`vasig swap\\\` — Execute a swap
- \\\`vasig swap-quote\\\` — Get a quote without executing
- \\\`vasig swap-chains\\\` — List supported swap chains

### Vault Management
- \\\`vasig vaults\\\` — List authenticated vaults
- \\\`vasig vault-info\\\` — Show active vault details

## Output Formats

All commands support \\\`--output json\\\` for structured JSON output:

\\\`\\\`\\\`json
{ "ok": true, "data": { ... } }
\\\`\\\`\\\`

Errors:
\\\`\\\`\\\`json
{ "ok": false, "error": { "code": "AUTH_REQUIRED", "message": "...", "hint": "Run: vasig auth" } }
\\\`\\\`\\\`

## Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | Bad arguments / usage error |
| 2 | Authentication required |
| 3 | Network / server error |

## Environment Variables

For CI/testing, credentials can be provided via env vars:
- \\\`VAULT_PASSWORD\\\` — VultiServer password
- \\\`VAULT_DECRYPT_PASSWORD\\\` — Vault decryption password
\\\`\\\`\\\`

**Step 2: Run all tests**

Run: \`npx vitest run\`
Expected: All tests pass

**Step 3: Run lint**

Run: \`npm run lint\`
Expected: No errors

**Step 4: Run build**

Run: \`npm run build\`
Expected: Clean build, \`dist/\` created

**Step 5: Commit**

\`\`\`bash
git add README.md
git commit -m "docs: add README with usage examples"
\`\`\`

---

### Task 15: Integration Smoke Test

**Files:**
- Create: \`tests/integration/cli-smoke.test.ts\`

**Step 1: Write integration test**

Create \`tests/integration/cli-smoke.test.ts\`:
\`\`\`typescript
import { describe, it, expect } from 'vitest'
import { execSync } from 'node:child_process'

describe('CLI smoke tests', () => {
  const run = (args: string) =>
    execSync(\`npx tsx src/index.ts \${args}\`, {
      encoding: 'utf-8',
      timeout: 10000,
      env: { ...process.env, NODE_ENV: 'test' },
    })

  it('--help shows all command groups', () => {
    const output = run('--help')
    expect(output).toContain('auth')
    expect(output).toContain('balance')
    expect(output).toContain('send')
    expect(output).toContain('swap')
    expect(output).toContain('addresses')
    expect(output).toContain('vaults')
    expect(output).toContain('vault-info')
  })

  it('--version returns semver', () => {
    const output = run('--version')
    expect(output.trim()).toMatch(/^\d+\.\d+\.\d+$/)
  })

  it('balance --help shows options', () => {
    const output = run('balance --help')
    expect(output).toContain('--chain')
    expect(output).toContain('--include-tokens')
    expect(output).toContain('--output')
  })

  it('send --help shows required options', () => {
    const output = run('send --help')
    expect(output).toContain('--chain')
    expect(output).toContain('--to')
    expect(output).toContain('--amount')
    expect(output).toContain('--yes')
  })

  it('swap --help shows required options', () => {
    const output = run('swap --help')
    expect(output).toContain('--from')
    expect(output).toContain('--to')
    expect(output).toContain('--amount')
    expect(output).toContain('--yes')
  })
})
\`\`\`

**Step 2: Run integration test**

Run: \`npx vitest run tests/integration/cli-smoke.test.ts\`
Expected: PASS (all 5 tests)

**Step 3: Run full test suite**

Run: \`npx vitest run\`
Expected: All tests pass (unit + integration)

**Step 4: Commit**

\`\`\`bash
git add tests/integration/cli-smoke.test.ts
git commit -m "test: add CLI integration smoke tests"
\`\`\`

**Step 5: Final build verification**

Run: \`npm run build && npm run lint && npx vitest run\`
Expected: All pass — build, lint, and tests green

**2026-03-22T01:10:02Z**

All 15 tasks implemented: scaffolding, error types, output formatting, credential store, vault discovery, shared types, CLI entry point, auth command, SDK helper, balance/addresses, send, swap, vault management, README, smoke tests. 77 tests passing, lint clean, build clean.
