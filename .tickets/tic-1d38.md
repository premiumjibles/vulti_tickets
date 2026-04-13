---
id: tic-1d38
status: open
type: chore
priority: 2
assignee: Jibles
created: 2026-04-13T12:07:29.626270262Z
---
# Simplify agent chat hook wiring

## Objective
Reduce incidental complexity in the agent chat hook layer by consolidating context building, removing redundant cache operations, and simplifying conversation state coordination.

## Context & Findings
Comparative analysis with shapeshift-agentic revealed three areas of unnecessary layering in the chat/conversation hooks. None of these are architectural — they are historical accumulation that can be flattened without behavioral changes.

**Context building (3 layers to 1):**
- Layer 1: agentContext.ts buildAgentContext() — derives addresses, builds coins, creates instructions
- Layer 2: useAgentTools.ts — wraps buildAgentContext, injects balances from React Query cache + recentActions from Zustand
- Layer 3: chatTransport.ts — calls the callback and packages into HTTP body
- All three serve a single consumer with no branching. agentContext.ts is not used anywhere else. buildAgentContext could take balances and recentActions as optional parameters directly, eliminating Layer 2.

**Double-invalidation in onFinish:**
- useAgentChat.ts lines 81-82: calls markConversationPersisted (updates cache) then immediately invalidateConversations (throws cache away and refetches). The first call is wasted work.
- Fix: call only invalidateConversations. The 5-minute staleTime means brief staleness is acceptable.

**Callback capture refs:**
- useAgentChat.ts lines 48-53: six ref assignments to avoid stale closures in onFinish. These exist because onFinish cannot depend on conversationId without retriggering useChat. Could be simplified by extracting the onFinish logic into a stable callback.

**Persisted flag race condition:**
- useConversations.ts lines 68-69: fetchConversation returns empty messages for unpersisted conversations. If a user switches away before onFinish marks persisted=true, messages become invisible on return.
- Fix: do not branch on persisted flag in fetchConversation — always attempt to load messages from cache or server.

## Files
- src/services/agentContext.ts — add optional balances and recentActions parameters to buildAgentContext
- src/features/agent/hooks/useAgentTools.ts — simplify to thin wrapper that gathers balances + recentActions and passes to buildAgentContext
- src/features/agent/lib/chatTransport.ts — reference only (consumer, no changes needed)
- src/features/agent/hooks/useAgentChat.ts — remove double-invalidation (line 81), reduce callback refs
- src/features/agent/hooks/useConversations.ts — simplify fetchConversation to single code path (remove persisted flag branching at lines 68-69)
- src/features/agent/lib/queries.ts — reference for query options and staleTime values

## Acceptance Criteria
- [ ] buildAgentContext accepts optional balances and recentActions parameters
- [ ] useAgentTools is simplified (no longer separately fetches/injects — passes params to buildAgentContext)
- [ ] onFinish does not call both markConversationPersisted and invalidateConversations (one or the other)
- [ ] fetchConversation uses single code path regardless of persisted flag
- [ ] Callback capture refs in useAgentChat reduced (at minimum remove the redundant markConversationPersistedRef)
- [ ] No behavioral regressions in conversation loading, switching, or context sending
- [ ] Lint and type-check pass

## Gotchas
- balancesQueryOptions cache (60s staleTime) is used elsewhere in the app for balance display — do not remove the query, just change where it is accessed for context building
- recentActions are cleared immediately after extraction (useAgentTools line 46) — preserve this clear-on-read semantics wherever it moves
- The persisted flag is also used in queries.ts line 153-154 to merge unpersisted + server conversations — check that removing the branch in fetchConversation does not break this merge logic
- chatTransport.ts prepareSendMessagesRequest extracts only the latest user message text (not full history) — this is intentional, do not change it

