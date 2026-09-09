# Agent Note: Folded Workspace Running Activity

Status: implemented

English | [中文](2026-08-31-folded-workspace-running-activity.zh.md)

## Problem

A folded Workspace group renders zero Session rows, so a user who collapses groups to reclaim sidebar space loses the per-Session running spinner that normally signals an in-flight loop. Glancing at a folded tree gives no way to tell that work inside one of those Workspaces is still running.

## Decision

`deriveGroups` adds one derived fact per group, `GroupNode.hasRunningActivity`, computed from every visible member Session summary before the folded projection drops the row list: true when any member runs its own loop (`running`) or has a running subagent descendant (`runningSubagentCount > 0`). The folded `ProjectRowItem` renders a single `ongoing` spinner immediately ahead of the directory name whenever the fact is true; expanded groups keep the per-row spinners and render none on the header.

The hoisted spinner reuses the same `StateDot` ongoing presentation as a Session row (the blue pixel-chase matrix) and the same `status.running` locale string for its visually-hidden label, so no new copy enters the dictionary. It keys off the fold state only: with the group open the header stays bare and the row-level dots remain the one source of activity presentation. The derivation stays a pure projection — `hasRunningActivity` is a boolean fact, not a scan performed by the renderer.

## Alternatives considered

**Scan rendered rows from the row component.** The folded group carries no child rows, so the component cannot inspect what it no longer renders; the fact must derive where the member list still exists.

**Hoist every possible status (pending interactions, completion reminder).** The request is the running spinner; approval, question, and plan-review wait states remain a separate concern and stay visible only once the group is opened (the package README keeps that limitation).

**Show the spinner on the header even while expanded.** That duplicates the row-level dot next to the folder and adds layout noise for no signal gain; the header indicator exists only to stand in for rows that are not rendered.

## Consequences

- A folded-tree user can see at a glance that a Workspace is still working without unfolding it; the spinner disappears the moment the last running member (or descendant) stops.
- The aggregate is independent of the five-row show-all overflow: `hasRunningActivity` reflects the full visible membership, not the rendered slice, so a group folded by the toolbar toggle or by the per-group chevron reports the same fact.
- Real Workspace groups and the Ungrouped bucket share one code path through `buildGroup`, so both gain the indicator with no special case.
- No new locale strings or store state: the fact is a derived field and the copy reuses `status.running`.

## Testing

Derivation tests cover own-running and descendant-running groups — including a folded group whose rendered rows are empty — and an idle group staying false. Row tests cover the spinner appearing directly ahead of the directory name only while the group is folded and running, and its removal once the group expands or stops running.

## Related

- [Workspace Sidebar Order and Folding](../../archived/feature/2026-08-11-workspace-sidebar-order-and-folding.md)