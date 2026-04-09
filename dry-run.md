# Dry-run support for non-legacy but-api mutations

## Goal

Add `dry_run` support to all non-legacy mutations in:

- `crates/but-api/src/commit`
- `crates/but-api/src/branch.rs`

The goal is to make these mutations capable of returning the same post-operation workspace view whether the operation is executed for real or run as a dry-run preview.

This spec is intentionally narrow: it defines API shape, return values, and caller updates required to support dry-run execution. It does **not** introduce any product or UI behavior changes.

## Requirements

### 1. Add `dry_run` to every non-legacy mutation

Every non-legacy mutation in the paths above should expose a `dry_run` flag.

- The type should be `Option<bool>`.
- The value should be unwrapped to `false` when not provided.
- `None` and `Some(false)` should behave identically.

This should be applied consistently across all non-legacy commit and branch mutations in these locations, even if some callers do not yet actively use the flag.

## 2. Dry-run should follow the real execution path

When `dry_run == true`, the mutation should perform the same planning / editing / rebase computation as the real operation, but without persisting the result.

The intended behavior is **not** a best-effort preview. Dry-run should produce the same logical operation result as a real execution, just without writing changes.

The current implementation work around `Editor` and `SuccessfulRebase` is expected to make this straightforward:

- `SuccessfulRebase` already exposes an `overlayed_graph()` representation.
- The output should be derived from that dry-run-compatible representation.
- There should be minimal additional work needed to make these operations return accurate dry-run results.

## 3. Standardize the workspace result shape

Add a standard return property named `workspace` to mutation responses.

```rs
struct WorkspaceState {
    /// Commits that were replaced by this operation. Maps `old_id -> new_id`.
    pub replaced_commits: BTreeMap<gix::ObjectId, gix::ObjectId>,
    pub head_info: but_workspace::RefInfo,
}
```

This structure should represent the workspace state **after** the mutation, regardless of whether the mutation was real or a dry-run.

### Dry-run semantics

In dry-run mode, `WorkspaceState` should still describe the **post-operation** state as if the mutation had actually happened.

In particular:

- `workspace.replaced_commits` should reflect the commits that would be replaced by the operation.
- `workspace.head_info` should reflect the hypothetical post-operation HEAD/workspace view.
- The returned `workspace` value in dry-run mode should have the same meaning and structure as in real execution.

`head_info` should be sourced from the same operation result data, and should be readily available from `SuccessfulRebase`.

## 4. Replace response-local `replaced_commits` fields

For the individual mutation response types:

- remove the existing top-level `replaced_commits` fields
- replace them with a `workspace` field

The intent is that callers consume a single standardized workspace result rather than special-casing per-mutation `replaced_commits` handling.

## 5. Update callers for compatibility only

Update callers in:

- `apps/lite`
- `apps/desktop`
- other Rust callers

These updates should be limited to wiring in the new API shape:

- pass through `dry_run` where needed
- consume `workspace` instead of response-local `replaced_commits`
- update types and field access accordingly

This pass should **not** change product behavior.

Specifically:

- no new UI flows are required
- no user-facing behavior should change
- this is API and plumbing work only

## Non-goals

This work does **not** include:

- changing product semantics
- introducing new UI for dry-run mode
- redesigning mutation behavior beyond adding dry-run support and standardizing the workspace result
- legacy mutation support

## Expected outcome

After this work:

1. Every non-legacy mutation in the targeted commit and branch APIs accepts `dry_run: Option<bool>`.
2. Real and dry-run execution both produce the same `WorkspaceState` structure.
3. `WorkspaceState` always represents the post-operation workspace view.
4. Response types use `workspace` instead of standalone `replaced_commits` fields.
5. Existing callers are updated to the new API without changing product behavior.
