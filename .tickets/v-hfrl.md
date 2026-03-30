---
id: v-hfrl
status: open
deps: []
links: [v-mehr]
created: 2026-03-30T02:26:53Z
type: task
priority: 2
assignee: Jibles
---
# Unify vultiagent-app onto @vultisig/sdk — pluggable MPC provider + migration

## Problem Statement

vultiagent-app (React Native/Expo) reimplements ~10k lines of chain registry, transaction building, MPC signing, and relay orchestration logic that already exists in @vultisig/sdk. vultiagent-cli correctly consumes the SDK as a thin wrapper. The goal was for both apps to share a common SDK, but vultiagent-app went native instead. "Solved" means both apps import @vultisig/sdk, with the only app-specific code being UI and platform bindings (expo-dkls for native MPC on mobile, WASM for Node/browser).

## Research Findings

### Current State

**Three repos, two implementations of the same logic:**

| | vultiagent-cli | vultiagent-app |
|---|---|---|
| Uses @vultisig/sdk? | Yes (`file:../vultisig-sdk/packages/sdk`) | No — full reimplementation |
| Package manager | npm | npm |
| DKLS/MPC | SDK's Rust WASM (10.8 MB) via `@lib/dkls` | Go native via `expo-dkls` (gomobile xcframework/aar) |
| Chain logic | SDK's chain adapters | Own `src/lib/chains.ts` (31 chains), `src/lib/coins.ts` |
| TX building | SDK's `vault.prepareSendTx()` | Own per-chain builders: `evmTx.ts`, `cosmosTx.ts`, `solanaTx.ts`, `suiTx.ts`, `tonTx.ts`, `tronTx.ts` |
| Signing orchestration | SDK's `vault.sign()` | Own relay coordination in `src/services/auth/fastVaultSign.ts`, `src/services/relay.ts` |
| Domain logic LOC | ~2.5k (thin wrapper) | ~10k (full reimplementation) |

**vultiagent-app code layout:**
- `src/lib/` (10 files, ~2.6k LOC) — pure domain logic (chains, coins, CAIP, schedules). No RN/Expo deps. Clean.
- `src/services/` (38 files, ~7.2k LOC) — TX builders, MPC signing, vault creation, auth, relay. 22 of ~30 files import Expo things (`expo-crypto`, `expo-secure-store`, `expo-dkls`, `react-native-sse`).
- `modules/expo-dkls/` — Expo native module wrapping Go gomobile binaries for DKLS (ECDSA) and Schnorr (EdDSA).
- Code comments acknowledge duplication: "Mirrors vultisig-sdk DKLS.processOutbound", "Reference: vultisig-sdk/packages/..."

**Two separate DKLS implementations:**
- **Rust** (`vultisig-windows/lib/dkls/`) → WASM via wasm-pack → used by SDK in Node/browser
- **Go** (`vultisig/mobile-tss-lib`) → native via gomobile → used by vultiagent-app on iOS/Android
- Both implement the same MPC protocol (`bnb-chain/tss-lib/v2` underneath), different source languages

### Available Tools & Patterns

**SDK platform abstraction (already exists):**
- `packages/sdk/src/platforms/` — node, browser, react-native, electron-main, chrome-extension
- Each platform registers: crypto (`configureCrypto()`), storage, polyfills, WASM init (`configureWasm()`)
- Rollup builds separate bundles: `index.node.esm.js`, `index.browser.js`, `index.react-native.js`, etc.
- DI pattern via global context (`context/wasmRuntime.ts`) — callback-based registration

**SDK MPC layer (where the coupling is):**
- `core/mpc/lib/signSession.ts` — **directly imports** `@lib/dkls/vs_wasm` and `@lib/schnorr/vs_schnorr_wasm`
- `core/mpc/lib/initialize.ts` — directly imports WASM init functions
- `core/mpc/keysign/index.ts` — calls `makeSignSession()` which returns WASM session objects
- The `SignSession` interface is implicit (duck-typed from WASM classes): `outputMessage()`, `inputMessage()`, `finish()`

**expo-dkls native module API (`modules/expo-dkls/src/ExpoDklsModule.ts`):**
- Exposes: keygen setup/session/messages, keysign setup/session/messages, Schnorr equivalents, key import
- iOS: Swift wrapping `godkls.xcframework` + `goschnorr.xcframework` via C FFI
- Android: Kotlin wrapping `dkls-release.aar` + `goschnorr-release.aar` via JNI/SWIG
- Frameworks are pre-built binaries, not compiled from source in this repo

**vultiagent-cli as reference thin client:**
- `src/lib/sdk.ts` — initializes SDK, imports vault, overlaps WASM compilation with file I/O
- `src/lib/signing.ts` — retry wrapper around `vault.sign()`
- `src/commands/` — each command calls high-level SDK methods (`vault.balance()`, `vault.prepareSendTx()`, etc.)
- Total: ~2.5k LOC for a full-featured CLI with MCP support

### Validated Behaviors

- Confirmed: SDK's `platforms/react-native/` entry point exists and builds, but the DKLS binding is still WASM (not native)
- Confirmed: `signSession.ts` directly imports WASM classes — no abstraction point for native bindings
- Confirmed: expo-dkls and SDK WASM expose the same conceptual API (setup, outputMessage, inputMessage, finish) — just different function signatures
- Confirmed: vultiagent-app's `src/lib/` (chains, coins) has zero RN/Expo imports and could be dropped entirely in favor of SDK equivalents
- Confirmed: SDK already has `MpcProvider`-like precedent in `WasmProvider` pattern and `configureCrypto()` DI
- Confirmed: both repos use npm (package-lock.json), not pnpm/yarn

## Constraints

- expo-dkls native module must remain for React Native — WASM performance on Hermes (RN's JS engine) is poor, which is likely why they went native
- The Go (mobile-tss-lib) and Rust (vultisig-windows/lib/dkls) are separate codebases implementing the same protocol — unifying them into one source language is a separate effort (Phase 2)
- SDK's `@core/mpc` and `@lib/dkls` packages are synced from vultisig-windows (read-only upstream) — changes to core MPC must flow through the SDK's own abstraction, not by modifying synced files
- vultiagent-app uses Expo-specific APIs for secure storage (`expo-secure-store`), biometrics (`expo-local-authentication`), camera, etc. — these are app-level concerns, not SDK concerns, and should stay in the app

## Dead Ends

- **Go → WASM for unified binary**: Go's WASM output bundles the entire Go runtime (~15-20 MB), has no threading, and performs poorly vs Rust WASM. Not viable for web/Node.
- **Pure TypeScript MPC implementation**: MPC protocol involves intensive elliptic curve operations — would be orders of magnitude slower and a security risk. The heavy crypto must stay in a systems language.
- **Extracting vultiagent-app's code into a new SDK**: The app's service code is coupled to Expo (`expo-crypto` for random bytes, `expo-dkls` for signing). Would need the same platform abstraction the existing SDK already has. Building a second SDK makes no sense.

## PoC Results: Pluggable MPC Provider (2026-03-30)

### What was done
Added an `MpcProvider` interface + `configureMpc()` DI pattern to the SDK, wrapping all WASM imports behind a pluggable provider. The existing WASM path works identically through the new abstraction.

### Files changed (SDK)
- **NEW** `packages/core/mpc/lib/mpcProvider.ts` — interface + registry (55 LOC)
- **NEW** `packages/core/mpc/lib/wasmMpcProvider.ts` — WASM implementation (63 LOC)
- **MODIFIED** `packages/core/mpc/lib/initialize.ts` — delegates to provider (was 18 LOC, now 13)
- **MODIFIED** `packages/core/mpc/lib/keyshare.ts` — delegates to provider (was 22 LOC, now 15)
- **MODIFIED** `packages/core/mpc/lib/signSession.ts` — delegates to provider (was 35 LOC, now 55)
- **MODIFIED** 4 platform entry points + 2 test setups — 2-line addition each (`configureMpc(wasmMpcProvider)`)
- **UNCHANGED** `keysign/index.ts`, `keysign/setupMessage/make.ts` — zero modifications needed

### Test results
- TypeScript typecheck: PASS
- SDK unit tests: 336/336 passed (21 files)
- CLI tests: 172/172 passed (22 files)
- SDK build (node): PASS
- E2E tests: setup loads correctly, skips at vault loading (no test vault configured)

### Key findings
1. **Clean abstraction**: Only 3 files needed modification in core MPC. The keysign orchestration (`keysign/index.ts`) required zero changes — it already imported through the lib files we shimmed.
2. **Schnorr type difference**: DKLS `setup()` takes `messageHash: Uint8Array | null | undefined`, Schnorr takes `message: Uint8Array` (non-nullable). Provider interface uses nullable, WASM provider asserts non-null for Schnorr with `!`.
3. **Keygen not covered**: `dkls/dkls.ts` and `schnorr/schnorrKeygen.ts` still import WASM directly. These would need a separate `MpcKeygenProvider` interface. Fine for now since vultiagent-app creates vaults via server, not local keygen.
4. **Upstream sync risk**: Modified files in `packages/core/` are synced from vultisig-windows. Production approach: either upstream the abstraction or move the provider layer to `packages/sdk/src/` with import interception.

### Confidence assessment
- **MPC keysign pluggability**: HIGH — proven working, clean interface
- **Full vultiagent-app migration to SDK**: MEDIUM — chain registry + TX building are trivial (zero RN deps in `src/lib/`), but MPC keygen + the expo-dkls handle-based API vs WASM class-based API needs careful adapter work
- **Native MpcProvider for expo-dkls**: MEDIUM — the conceptual API maps well (setup/output/input/finish), but expo-dkls uses integer handles vs WASM objects, and all messages are base64 strings vs Uint8Array. Adapter is straightforward but needs testing with real native module.

## Goal State: Unified Rust → All Targets (Phase 2)

### Current binary landscape

Three separate build pipelines produce MPC binaries for different consumers:

| Source | Language | Build tool | Output | Consumer |
|---|---|---|---|---|
| `vultisig/dkls23-rs` | Rust | `wasm-pack` | `.wasm` + `.js` + `.d.ts` | SDK (Node/browser) via vultisig-windows sync |
| `vultisig/multi-party-schnorr` | Rust | `wasm-pack` | `.wasm` + `.js` + `.d.ts` | SDK (Node/browser) via vultisig-windows sync |
| `vultisig/mobile-tss-lib` | Go | `gomobile bind` | `.xcframework` (iOS), `.aar` (Android) | vultiagent-app via expo-dkls |
| `vultisig/go-wrappers` | Go (CGO) | `go build` + linked Rust `.so` | Go package with C FFI | vultiserver |

**Key discovery: vultiserver already runs Rust.** The Go server uses `go-wrappers` which calls into pre-compiled Rust `.so`/`.dylib` libraries via CGO. The C headers are generated by `cbindgen` from the same Rust source. So the actual cryptography on the server is Rust — Go is just the FFI glue.

**vultiagent-app's native binaries go through an unnecessary layer.** The mobile path is: Rust source → compiled Rust lib → Go CGO wrapper → gomobile bind → .xcframework/.aar → Swift/Kotlin FFI → Expo module → JS. The Go runtime gets bundled into every native binary for no reason — `dkls-release.aar` is 9.6 MB largely due to Go runtime overhead. Direct Rust → native would be significantly smaller.

**The Go repo (`mobile-tss-lib`) is NOT calling Rust** — it's a separate pure Go implementation of the same protocol (built on `bnb-chain/tss-lib/v2`). Meanwhile `go-wrappers` IS calling Rust. Two different Go repos, two different approaches, same protocol. This is the mess we're cleaning up.

Native binaries (`.xcframework`, `.aar`) are git-ignored in vultiagent-app and must be manually copied from `mobile-tss-lib` builds — no download script exists.

### Target state

Bring the Rust source (`dkls23-rs`, `multi-party-schnorr`) into the SDK monorepo. One Rust codebase produces all four output targets. The Go repos (`mobile-tss-lib`, `go-wrappers`) are eliminated as build dependencies.

```
sdk-repo/
├── rust/
│   ├── dkls/           # Rust DKLS source (from dkls23-rs)
│   ├── schnorr/        # Rust Schnorr source (from multi-party-schnorr)
│   └── Cargo.toml      # Workspace with per-target build profiles
├── packages/
│   ├── lib-dkls/       # WASM outputs consumed by TS
│   ├── lib-schnorr/    # WASM outputs consumed by TS
│   ├── native-ios/     # .xcframework outputs
│   ├── native-android/ # .aar / .so outputs
│   └── native-server/  # .so/.dylib + cbindgen C headers for vultiserver
└── scripts/
    └── build-native.sh
```

Build pipeline — one Rust source, four targets:

```bash
# 1. WASM (Node/browser/Electron) — already working today
wasm-pack build -t web --out-dir packages/lib-dkls/wasm

# 2. iOS native (device + simulator → .xcframework)
cargo build --release --target aarch64-apple-ios
cargo build --release --target aarch64-apple-ios-sim
xcodebuild -create-xcframework ...

# 3. Android native (.so for arm64, armv7, x86_64)
cargo ndk -t arm64-v8a -t armeabi-v7a -t x86_64 build --release
# Package into .aar with JNI bindings

# 4. Server native (.so/.dylib + C headers for Go CGO or direct linking)
cargo build --release --target x86_64-unknown-linux-gnu
cbindgen --config cbindgen.toml --output include/dkls.h
```

Triggered by `pnpm run build:wasm`, `pnpm run build:native`, and `pnpm run build:server`, or a CI pipeline. Most developers never run these — compiled outputs stay checked in or published as release artifacts.

### What each consumer gets

| Consumer | Target | What changes |
|---|---|---|
| **SDK (Node/browser)** | WASM | No change — same wasm-pack output, just built from in-repo source instead of synced |
| **vultiagent-app (iOS)** | .xcframework | expo-dkls Swift wrapper calls Rust C FFI directly (via cbindgen) instead of Go FFI. Same pattern, one fewer layer. Smaller binaries (no Go runtime) |
| **vultiagent-app (Android)** | .aar / .so | expo-dkls Kotlin wrapper calls Rust JNI directly instead of Go SWIG/JNI. Same pattern, one fewer layer. Smaller binaries |
| **vultiserver** | .so + C headers | `go-wrappers` either points at our .so + headers directly, or we publish a new Go package with the same CGO interface. Server code (`mpc_wrapper.go`) unchanged |

### What this eliminates

- The Go `mobile-tss-lib` repo — pure Go reimplementation of the protocol, no longer needed
- The `go-wrappers` build chain — or at minimum, it becomes a thin consumer of our published `.so` + headers rather than owning the Rust compilation
- The `sync-and-copy.ts` script that mirrors WASM from vultisig-windows
- The manual process of copying `.xcframework` / `.aar` into expo-dkls
- The unnecessary Go runtime bundled into every mobile native binary
- Two independent implementations of the same protocol diverging over time
- The dependency on vultisig-windows as the upstream source of truth for core packages

### What changes in expo-dkls

The Swift and Kotlin wrappers would need updating to call Rust C FFI (via `cbindgen` or `uniffi-rs`) instead of Go FFI. The interface shape is very similar — both are C-level function calls with buffer pointers and `tss_buffer` structs. The TypeScript API (`ExpoDklsModule.ts`) stays the same. The `MpcProvider` abstraction from Phase 1 means no changes to any TS orchestration code.

### What changes in vultiserver

Minimal. The server's `service/mpc_wrapper.go` adapter layer stays the same — it already calls C FFI functions like `dkls_keygen_setupmsg_new()`. The only change is where the `.so` and C headers come from (our repo instead of go-wrappers). The HTTP API, Redis task queue, and relay coordination are untouched.

### Tooling maturity

All four compilation targets from Rust are well-established:

| Target | Tooling | Used in production by |
|---|---|---|
| WASM | `wasm-pack` | Already working in Vultisig today |
| Android | `cargo-ndk` | Signal, Mozilla Application Services |
| iOS | `cargo-lipo` + `cbindgen` / `uniffi-rs` | Signal, Mozilla Application Services, 1Password |
| Server (.so + C FFI) | `cargo build` + `cbindgen` | Already working in Vultisig today (via go-wrappers) |

### Rust source ownership

Both `dkls23-rs` and `multi-party-schnorr` are Vultisig's own repos (not third-party). Licensed as "SLL" (Vultisig proprietary). `multi-party-schnorr` has a HashCloak security audit. No license concerns with bringing them in-repo.

### Sequencing

This is Phase 2 — depends on Phase 1 (pluggable MPC provider) being complete. The `MpcProvider` abstraction means the native provider swap (Go FFI → Rust FFI) is invisible to the TypeScript layer. Phase 1 buys the ability to do Phase 2 without touching any TS orchestration code.

Recommended order within Phase 2:
1. Bring Rust source into repo, verify WASM builds match existing output
2. Add server `.so` + C header build, verify go-wrappers compatibility
3. Add iOS/Android native builds, update expo-dkls wrappers
4. Deprecate mobile-tss-lib and go-wrappers dependencies

## Open Questions

- **MpcProvider interface shape**: The SDK's WASM classes use `new SignSession(setup, id, keyshare)` with `outputMessage()/inputMessage()/finish()`. expo-dkls uses flat functions (`dklsKeysignSetup()`, `dklsKeysignSession()`, `dklsKeysignOutbound()`, etc.). How thin should the adapter be? Should it mirror the WASM class API or define a higher-level interface?
- **Where does the MpcProvider get registered?** Current pattern is `configureWasm()` in platform entry points. Should MPC follow the same pattern (`configureMpc()`) or be part of the existing `configureWasm()` call?
- **Migration strategy for vultiagent-app**: Big bang (replace all services at once) vs incremental (start with chain registry/coins, then TX building, then MPC signing)? The chain registry and coins have zero RN deps and could be swapped trivially. MPC is the hard part.
- **expo-dkls ownership**: Should it stay in vultiagent-app's `modules/` or move into the SDK monorepo as a platform-specific package (e.g., `@vultisig/dkls-native`)?
- **Testing parity**: How do we verify the native MPC provider produces identical signatures to the WASM provider? Is there an existing test harness for this?
- **SDK React Native build target**: The SDK already has `platforms/react-native/` — is it currently tested/used by anything, or is it a skeleton? Does it need work before vultiagent-app can consume it?
