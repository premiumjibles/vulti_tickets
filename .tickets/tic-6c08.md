---
id: tic-6c08
status: open
type: task
priority: 2
assignee: Jibles
created: 2026-04-13T12:07:22.280484066Z
---
# Unified tool execution lifecycle with historical guard

## Objective
Consolidate the fragmented tool execution state (pendingTx/txResult/useToolLifecycle/useTransactionSigning/buildToolDetection) into a unified step-based execution model where tool components own their signing lifecycle. Add an explicit historical tool guard to prevent re-execution of tools from loaded conversations.

## Context & Findings
Comparative analysis with shapeshift-agentic revealed that vultiagent-app's tool execution is split across 4-5 state containers and hooks doing what a single ToolExecutionState + useToolExecution pattern handles in ShapeShift. The ShapeShift pattern is signing-method-agnostic — TSS/MPC signing works fine inside a step machine; the "sign" step just takes longer.

**Current flow (store-based indirection):**
buildToolDetection scans messages → writes to chatSessionStore.pendingTx → BuildTxCard reads from store → useTransactionSigning reads from store → signs → writes txResult/recentActions to store

**Target flow (component-owned lifecycle):**
BuildTxCard receives tool output as prop → calls requestApproval() from context → gets password → calls signing function → reports result via callback

**Key findings from deep analysis:**
- useTransactionSigning already accepts pendingTx as a parameter (90% decoupled) — only writes txResult/recentActions directly to store
- AgentApprovalBar is already fully controlled (no store access) — works with either pattern
- BuildTxCard is purely display-driven — reads pendingTx/txResult from store but could take props
- useConfirmationFlow (scheduled proposals) has ZERO overlap with tx signing — separate concern, no unification needed
- recentActions CANNOT be eliminated (backend doesn't receive message history) but can be callback-based
- A real bug exists: loading a conversation can re-trigger build_* tool detection because buildToolDetection scans ALL messages without distinguishing historical from new. The buildToolPendingTxIdRef guard only prevents the same tool from firing twice in a session — not historical tools from prior sessions.

**Rejected approaches:**
- Promise-based approval modal: React Native doesn't have true blocking modals; current store-driven bar is more practical
- Eliminating recentActions: blocked by backend architecture (only receives current message, not history)
- Client-side conversation persistence: blocked by cross-device sync requirements

## Files
- src/features/agent/hooks/useTransactionSigning.ts — make completeTxApproval/confirmTx callback-based (onTxCompleted, onActionLogged) instead of direct store mutation
- src/features/agent/hooks/useAgentChat.ts — replace auto-setPendingTx effect (lines 185-196) with callback; add historical tool ID tracking
- src/features/agent/lib/buildToolDetection.ts — no changes needed (detection logic is fine, consumption changes)
- src/features/agent/stores/chatSessionStore.ts — shrink: pendingTx/txResult may move to component-local or context state
- src/features/agent/components/tools/BuildTxCard.tsx — receive pendingTx/txResult as props instead of reading from store
- src/features/agent/components/tools/SignTypedDataTool.tsx — same: props instead of store reads
- src/features/agent/screens/AgentHomeScreen.tsx — simplify orchestration after phases complete
- src/features/agent/components/AgentApprovalBar.tsx — no changes needed (already fully controlled)
- src/features/agent/hooks/useConfirmationFlow.ts — reference only, no changes (separate concern)

## Acceptance Criteria
- [ ] ToolExecutionState type defined with step-based state machine (currentStep, completedSteps, skippedSteps, failedStep, terminal, meta, substatus)
- [ ] useToolExecution hook provides ExecutionContext with advanceStep/skipStep/failAndPersist/markTerminal
- [ ] Historical tool guard: Set<string> of tool call IDs populated on conversation load, checked before any tool execution
- [ ] BuildTxCard receives tool output data as props (not from chatSessionStore)
- [ ] useTransactionSigning reports results via callbacks (onTxCompleted, onActionLogged) instead of direct store mutation
- [ ] buildToolDetection auto-fire effect in useAgentChat replaced with callback-based pattern
- [ ] Loading a saved conversation with completed build_* tools does NOT show approval UI
- [ ] Switching conversations within a session does NOT re-trigger historical build tools
- [ ] All existing signing flows (EVM, Solana, UTXO, XRP, Cosmos, etc.) work through new step machine
- [ ] SignTypedDataTool receives state as props instead of reading from store
- [ ] Existing tests updated to reflect new patterns

## Gotchas
- useTransactionSigning's completeTxApproval has a text-based chain detection fallback (lines 245-323) that regex-matches chain from conversation text when parseServerTx returns null — must be preserved
- Biometric prompt is wrapped in InteractionManager.runAfterInteractions() to prevent navigation race — don't lose this
- The tx_ready path (scheduled proposals via getTxProposalApi) is separate from build_* detection — both need historical guard but have different data sources
- recentActions must still flow to backend on next message send — callback pattern changes WHERE it's written, not WHETHER
- pendingTx store may still be needed for the approval bar visibility check in AgentHomeScreen unless replaced with context

