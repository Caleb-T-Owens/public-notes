# Combining divergent changes

## Problem

When local history and remote history both evolve the same logical change, a normal cherry-pick is not enough.

The simplest case is a single amended commit on each side:

- remote amended `X` into `R`
- local amended `X` into `L`

In that case, we can rebase both sides onto the desired parent and perform a three-way merge using `X` as the base.

The harder case is when either side split the original change into multiple commits. For example:

- the remote side split `X` into `R1 -> R2 -> R3`
- the local side split `X` into `L1 -> L2`

When pulling, we want to preserve the remote shape. That means the result should still be three remote-position commits, while local amendments are distributed across those three commits in remote order.

This document defines the terminology and algorithm for doing that.

## Goals

1. **Preserve remote structure as much as possible.**
   The remote side determines the number and ordering of commits in the resolved result.
2. **Preserve remote ordering.**
   If the remote side split a logical change into multiple commits, those commits stay in that order.
3. **Incorporate all local changes exactly once.**
   Local amendments should either be assigned to an earlier matching remote commit or carried forward to a later one.
4. **Integrate with the existing rebase engine.**
   The solution should compose with normal step-graph rebasing instead of creating a separate parallel engine.
5. **Represent conflicts explicitly.**
   If the final merge for a remote position conflicts, the result should use the existing conflict representation.

## Terms

### Divergence ancestor

The commit from which the local and remote histories both descended before they diverged.

This is the preferred merge base when one exists. In earlier notes this was described as a “common amended ancestor”. In this document, we use **divergence ancestor** consistently.

### Remote family

An ordered set of one or more remote commits that represent the remote-side evolution of one logical change.

Example:

- `R1 -> R2 -> R3`

A remote family preserves remote structure. The output must contain one resolved commit for each remote family member.

### Local sequence

The ordered list of local commits that represent the local-side evolution of the same logical change.

Example:

- `L1 -> L2`

The local sequence is treated as input material to distribute across the remote family. Its internal commit structure does **not** need to be preserved.

### Family order

The processing order of commits in a remote family.

This is **not** a separate family-local ordering. It is the normal traversal order already implied by the step graph: parentmost to childmost in the output history.

### Rebase base

The commit that the current remote family member should be placed on top of in the output graph.

For the first member of a family, this is the parent produced by the preceding step in the rebase graph. For later members, it is the resolved parent created earlier in the same family (or whatever parent the step graph specifies at that position).

### Hunk pool

A tree representing the unresolved local content that still needs to be assigned into the remote family.

At the beginning of processing, the hunk pool contains the entire local sequence flattened onto the first rebase base. As each remote family member absorbs overlapping hunks, the remaining hunks are carried forward in the pool.

### Overlap

A hunk in the hunk pool **overlaps** a remote commit if it touches the same changed region as that remote commit’s delta **after that remote commit has been materialized onto its output parent**.

Overlap is determined using a **zero-context diff** of the rebased/materialized remote commit against its output parent.

In other words, for remote family member `Ri`, the classifier uses:

```text
diff(parent(Ri'), Ri')
```

Where `Ri'` is the remote commit after it has been rebased or otherwise materialized at its output position.

Rules:

- line edits overlap when their changed ranges intersect
- file add/remove/rename operations count as full-file overlap
- rename-plus-edit is treated as full overlap for now

The algorithm deliberately avoids trying to do clever sub-file distribution for rename cases.

## Why this belongs in the rebase engine

This operation needs full rebase context.

A remote family may be spread through a larger graph, and the remaining local hunks must be carried forward from one remote position to the next. That means we need the same parent tracking, commit materialization, and conflict handling that the existing rebase engine already provides.

Creating a separate divergence-specific history engine would duplicate a large amount of the rebase machinery and make composition harder.

Instead, divergent resolution should be modeled as a special step type that runs inside the rebase engine and then lowers to ordinary pick steps once resolved.

## Proposed step: `PickDivergent`

Each remote family member is represented by one `PickDivergent` step.

```text
PickDivergent {
  local_commits: Vec<ObjectId>,   // parentmost -> childmost
  ancestor: Option<ObjectId>,     // same for every step in the family
  remote_commit: ObjectId,        // the remote commit for this position
  family_id: [u8; 20],            // associates related remote commits
}
```

### Invariants

The following invariants are trusted input from the caller.

For all `PickDivergent` steps with the same `family_id`:

- `local_commits` is identical
- `ancestor` is identical
- each step has a different `remote_commit`
- the steps are processed in normal graph traversal order
- each step produces exactly one output commit

After processing, a `PickDivergent` step is lowered into an ordinary `Pick` step in the output step graph. No additional divergence-specific metadata needs to survive in the resulting graph.

## High-level idea

The remote family defines the output shape.

The local sequence is flattened into a single hunk pool. Then, for each remote family member in order:

1. identify the portion of the pool that overlaps this remote commit
2. merge that portion into this remote position
3. carry the remainder forward

Earlier remote commits get first claim on overlapping hunks. Assignment is greedy and single-claim: once a hunk is assigned to the current remote family member, it is removed from the pool and cannot be assigned again later in the traversal. Any hunks that were never claimed by earlier commits are guaranteed to land in the final remote family member.

## Detailed algorithm

Assume:

- remote family: `R1, R2, ..., Rn`
- local sequence: `L1, L2, ..., Lm`
- optional divergence ancestor: `A`
- first rebase base: `B0`

`R1..Rn` are processed in family order.

### 1. Build the initial hunk pool

Rebase the local sequence onto `B0` and flatten it into a single tree, preserving the final local content rather than local commit structure.

Conceptually:

1. start from `B0`
2. apply `L1, L2, ..., Lm` in local order
3. squash the result into one tree
4. when an intermediate merge must choose a side, prefer the side containing the intended local result

In terms of the current merge implementation, this means preferring the local / “theirs” side while constructing the pool.

Call the resulting tree `P1`.

If a divergence ancestor `A` exists, also rebase it onto `B0` to produce `A1`.

### 2. For each remote family member, move the pool to the correct parent

For step `i`, let `Bi` be the actual parent commit for that remote position in the output graph.

Before resolving `Ri`:

- if the current hunk pool is not already based on `Bi`, rebase it onto `Bi`
- if an ancestor exists and is not already based on `Bi`, rebase the ancestor onto `Bi`

This keeps the pool and optional ancestor aligned with the current output position.

### 3. Materialize the remote commit for this position

Resolve the remote commit at this step exactly as the rebase engine would normally place it:

- cherry-pick `Ri` onto `Bi`
- if `Ri` is a merge commit, first construct the virtual parent/merge base exactly as existing cherry-pick logic does
- if the required merge base cannot be constructed, fail the operation

Call the resulting commit tree for this position `Ri*`.

If the resolved commit for this position is ultimately written out, it keeps the remote commit’s message, author, and extra headers, and follows the same signing semantics as a normal `Pick`.

### 4. Partition the hunk pool

If `i < n`, compute the zero-context diff for the rebased/materialized remote delta:

```text
diff(parent(Ri'), Ri')
```

Where `Ri'` is the remote commit after it has been materialized onto its output parent.

Use that diff to partition the current pool `Pi` into:

- `Pi_match`: hunks that overlap `Ri'`
- `Pi_rest`: hunks that do not overlap `Ri'`

Assignment is greedy in family order and single-claim: if a hunk overlaps the current remote commit, it is assigned here, removed from the pool, and cannot later be assigned to another remote family member even if it would also overlap there.

If `i == n` (the last remote family member), skip filtering and define:

- `Pi_match = Pi`
- `Pi_rest = ∅`

This guarantees that any remaining local changes are incorporated into the final remote position.

### 5. Merge the matched pool into the current remote position

Produce the resolved tree for this remote position with a three-way merge:

- **base** = rebased divergence ancestor for this position, if one exists
- **otherwise base** = the current rebase base `Bi`
- **ours** = `Ri*`
- **theirs** = `Pi_match`

In shorthand:

```text
resolved_i = merge(base_i, remote_i, matched_pool_i)
```

If the merge is clean, write a normal commit for this remote position.

If `Pi_match` is empty for a non-final step, we still emit the remote commit for that position normally, because preserving the remote structure is stronger than only emitting commits that absorb local hunks.

If the merge conflicts, write a conflicted commit using the existing conflict representation.

### 6. Carry the rest of the pool forward

If `i < n`, set:

- `P(i+1) = Pi_rest`

Then continue to the next remote family member.

Because the remainder is rebased onto the next remote position before reuse, later commits see the still-unassigned local changes in the correct context.

This makes the algorithm intentionally sequential and context-sensitive: overlap for `R(i+1)` is evaluated only after `Ri` has been materialized and after the remaining pool has been carried forward.

### 7. Lower the result into ordinary rebase steps

Once each `PickDivergent` step has been resolved, replace it in the output graph with a normal `Pick` step containing the newly created commit id.

From that point on, the rest of the rebase engine does not need to know that divergent resolution happened.

## Worked example

Suppose:

- divergence ancestor: `X`
- remote family: `R1 -> R2 -> R3`
- local sequence: `L1 -> L2`

We want the final history to remain three commits on the remote side.

1. Flatten `L1 -> L2` onto the parent of `R1` to create `P1`
2. Compare `P1` against the delta of `R1`
   - overlapping hunks go into `P1_match`
   - the rest become `P1_rest`
3. Merge `P1_match` into `R1`
4. Rebase `P1_rest` onto the parent of `R2`
5. Compare the rebased remainder against the delta of `R2`
6. Merge the overlapping portion into `R2`
7. Carry the remainder forward again
8. At `R3`, merge the entire remaining pool into the final remote position

Result:

- remote commit count is still 3
- remote ordering is still `R1 -> R2 -> R3`
- local amendments are incorporated once, in order
- unclaimed local hunks end up in `R3`

## Merge-commit handling

If a remote or ancestor commit involved in divergent resolution has multiple parents, the algorithm should use the same generalized cherry-pick behavior already present in `but-rebase`:

- merge the original parents to produce the effective base when necessary
- merge the destination parents to produce the effective onto state when necessary
- fail if those prerequisite base merges themselves cannot be resolved well enough to continue

In other words, `PickDivergent` should reuse the crate’s existing N-to-M cherry-pick semantics rather than inventing separate merge behavior.

## Conflict behavior

There are two distinct failure modes:

### 1. Prerequisite merge-base construction fails

If the algorithm cannot construct the required effective base or effective destination state for a merge commit, the operation should fail immediately.

This matches existing `cherry_pick` behavior that reports `FailedToMergeBases`.

### 2. Final divergent merge conflicts

If the final merge for a remote position conflicts, the operation should still succeed by materializing a conflicted commit using the existing conflict representation.

That is the same user-visible behavior as other rebases in `but-rebase`.

## Important invariants

The implementation should preserve these invariants:

1. **Remote-family cardinality is preserved.**  
   One input remote family member becomes one output commit.

2. **Remote-family order is preserved.**  
   The output ordering matches the remote ordering exactly.

3. **Local structure may be discarded.**  
   The local sequence is flattened into content, not preserved as commits.

4. **Earlier matches win.**  
   Overlap assignment is greedy in remote order.

5. **Assignment is single-claim.**  
   Once a hunk is assigned to a remote family member, it is removed from the pool and cannot be assigned again later.

6. **Nothing is silently dropped.**  
   Whatever remains unassigned before the last family member is merged there.

7. **Normal rebase semantics remain in charge.**  
   Parent selection, commit creation, metadata retention, signing behavior, and conflict materialization should reuse the existing rebase engine.
