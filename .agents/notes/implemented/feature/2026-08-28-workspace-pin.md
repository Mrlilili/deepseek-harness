# Agent Note: Workspace Pin

Status: implemented

English | [中文](2026-08-28-workspace-pin.zh.md)

## Problem

A workspace list grows past the few projects a user touches daily. The browser's two existing orders cannot express "keep this one on top": the durable registry order moves only by explicit drag, and recency sorting belongs to Sessions, not Workspaces. Users need a persistent, one-gesture way to hold a Workspace at the top of the grouped tree that survives restarts, reconnects, and manual reordering of everything else.

## Decision

Pin state lives on the workspace record as an optional ISO-8601 `pinnedAt` instant; `undefined` means unpinned. `Workspace.setPinned(pinned)` writes it through the entity's shared mutate path, stamps `updatedAt`, and resolves without writing when the request matches the current state. Pinning never touches `workspaceIds` — the durable display order stays exactly what dragging produced.

The wire gains one unary verb, `workspace/setPinned({ workspaceId, pinned })`, returning the updated `WorkspaceView` projection that now carries the optional `pinnedAt` field. The `follow()` stream publishes the pin like any record mutation: one `upsert` increment, so reconnecting clients converge through the baseline. No new stream frame type exists.

The browser applies pinning at render time: `deriveGroups` partitions pinned Workspaces ahead of unpinned ones, each partition in stable Host order, and the tree marks each group `pinned`. The Workspace row menu carries the Pin/Unpin action (label flips with state), committing without a dialog and carrying a persistent pin mark on the row; failures stay non-fatal console diagnostics like reorder rejections. The record schema extension is additive, so the `workspace` domain stays at version 2 and records written before the field parse as unpinned. The browser-local view-state ownership split (durable order versus presentation preference) is the [sidebar order and folding Agent Note](2026-08-11-workspace-sidebar-order-and-folding.md); pin state joins the durable side while the pinned-first partition stays a rendering rule.

## Alternatives considered

**Pin = prepend to the durable `workspaceIds` order.** Rejected: the registry order is the drag-owned layout. Prepending on pin makes unpin ambiguous (where does the row go back?) and silently rewrites a layout the user arranged by hand.

**Browser-local pin set.** Rejected: pin state would live outside the Host-authoritative rows, diverging across browsers and devices, and the sidebar projection would need a second source of truth to merge with Host order.

**Reuse `updatedAt` recency for prominence.** Rejected: `updatedAt` moves on every accepted mutation (rename, session attach, prune), so "recently mutated" is not "deliberately promoted".

## Consequences

Old media open unchanged and unparse-free; the domain version stays 2, and no migration exists — the projection cache discards stale record documents, but `pinnedAt` is optional so no document goes stale. Drag semantics remain meaningful inside each partition, while dragging an unpinned Workspace above a pinned one cannot visually stick — the partition, not `insertBefore`, decides rendered position. The partition logic lives in the browser tree derivation, so any future grouping surface must repeat the pinned-first rule rather than inherit it from Host order.
