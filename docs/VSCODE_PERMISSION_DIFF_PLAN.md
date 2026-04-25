# VS Code Post-Apply Edit Review Plan (Upstream-Friendly)

## Goal

Keep edit-heavy agent workflows fast in OpenChamber's VS Code sidebar UI and surface VS Code-native per-hunk review controls.

## Final Direction

- Do not block on pre-apply permission review for `edit` asks in VS Code.
- Auto-approve edit permissions in VS Code.
- Keep existing permission behavior for non-edit permissions.
- Keep this behavior default-on via VS Code setting.
- Reuse existing UI and permission card behavior (no new card/action).

## Non-Goals

- No OpenCode backend protocol changes.
- No permission payload/schema changes.
- No behavior changes for web/desktop runtimes.
- No redesign of permission UI components.

## Settings Contract

- New VS Code setting: `openchamber.vscode.postApplyEditReview`.
- Type: `boolean`.
- Default: `true`.
- Runtime-exposed field: `vscodePostApplyEditReview`.

## Implementation (Completed)

1. VS Code setting contribution and runtime exposure.
   - `packages/vscode/package.json`
   - `packages/vscode/src/bridge-settings-runtime.ts`

2. Settings propagation into shared UI config flow.
   - `packages/ui/src/lib/api/types.ts`
   - `packages/ui/src/lib/desktop.ts`
   - `packages/ui/src/lib/persistence.ts`
   - `packages/ui/src/stores/useConfigStore.ts`
   - `packages/vscode/webview/main.tsx`
   - `packages/vscode/src/extension.ts` (live config-change push)

3. VS Code-only edit auto-approve behavior in sync layer.
   - `packages/ui/src/sync/sync-context.tsx`
   - Applies in live `permission.asked` flow.
   - Applies again in reconnect pending-permissions resync flow.

4. Removed SCM auto-open nudge from sync flow.
   - No longer force-opens `workbench.view.scm` after auto-approve.

5. Added sidebar-compatible post-apply diff auto-open for auto-approved edit permissions in VS Code.
   - `packages/ui/src/sync/sync-context.tsx`
   - When an `edit` permission is auto-approved, queue one post-apply review token per request.
   - On matching finalized edit tool part (`edit`/`multiedit`/`apply_patch`), open `editor.openDiff` with patch-backed virtual original.
   - Keeps review in VS Code diff UI without relying on chat participant APIs.

## Follow-up Design Note

- Native chat participant-only hunk controls (`textEdit`/`externalEdit`) are still not directly available from sidebar webview flows.
- Current implementation uses VS Code diff editor auto-open as the sidebar-compatible native review surface.
- If stricter Copilot-style hunk controls are required beyond diff editor capabilities, that remains a future exploration item.

## Validation

Executed checks:
- `bun run --cwd packages/ui type-check`
- `bun run --cwd packages/vscode type-check`
- `bun run --cwd packages/ui lint`
- `bun run --cwd packages/vscode lint`
- `bun run --cwd packages/vscode build`

## Manual Verification Remaining

In VS Code runtime with setting enabled:
1. Trigger an edit permission ask from agent.
2. Confirm permission card is auto-skipped for edit request.
3. Confirm edits apply and run continues.
4. Validate final native per-hunk review integration once design is finalized.
5. Validate that each auto-approved edit permission opens exactly one diff review and does not duplicate on repeated part updates.

## Rollback Safety

- Behavior is isolated to VS Code runtime checks.
- Setting can disable auto-approve behavior without backend changes.
