# Work pieces
## Metadata persistance
This portion should be pluggable. We should assume an initial implementation can use gmeta.
The structure can be as follows:
commit:
	diverged_from: Set\<oid\>
	 dropped: bool (always true or unset)

We may write metadata for commits that don't exist in the remote. The graph that this creates is sparsely populated.
## Reconciling graphs
We are always working with two graphs - the graph referenced by the remote reference and the graph referenced by the local reference. We want to combine the graphs with the following goals:
- Preserving the remote structure as much as possible
- Preserving the ordering of nodes in the remote

## Combining divergent changes
(problem statement)
In the basic case, where the remote and the local branches have both amended a commit, we can try to find a common amended ancestor, rebase all three trees onto where the desired parent and then perform a three way merge on those to produce the final combined tree.

We however need to consider how to handle split commits which in the evolution graph are represented by having N children splitting off of the same ancestor.

When pulling, we try to preserve the remote ordering, so if the remote has a commit split into 3, and the local split into 2, we will try to combine the contents of the two local commits into the three remote commits.

(Aside on the rebase engine)
Since we may have multiple remote commits and those remote commits can be spread arbitrarily through a graph, IE (T1 -> other commit -> T2 -> other commit -> T3), we need to have pretty much a full rebase context when producing the final graph. This is because we need to track which hunks have been absorbed into the other targets from our source tree.

To facilitate this, we could have a new system for rebasing history specifically for this problem set. Doing so would result in a lot of overlap with the rebase engine, and would make composability with the rebase engine harder. As such, I'm leaning towards the idea of having a special `PickDivergent` step to the rebase engine.

A `PickDivergent` step contains:
- The local commit ids provided in topological order
- Optionally the common ancestor
- The id of the current remote commit
- A 20 byte string family identifier used to associate remote commits that belong to the same split family.

Each `PickDivergent` represents one remote commit within a single remote commit family. Each step specifies the local commits it is working with, and all `PickDivergent` steps that belong to the same remote family consistently list the same local commits. The algorithm only considers those listed local commits. If an optional ancestor is present, it is the same for every `PickDivergent` in that remote family.

The rebase engine might need some extra state to support picking divergent commits.
Once a `PickDivergent` has been processed, it is lowered into a regular `Pick` step in the output step graph that is being built. We do not need to preserve any extra divergent-resolution metadata in that resulting graph.

(the general algorithm)
The hunk pool - We take all the local commits in the provided order, from most parent to most child, and rebase them on to the first remote commit's parent. Conceptually, these commits are squashed into a single hunk pool tree by merging them while preferring the side with the local changes (that is, taking the "theirs" side during the merge). We always flatten the local side this way, even if doing so loses per-commit structure. This same preference for the local side also applies when rebasing the ancestor onto the new parent and when rebasing the squashed hunk pool forward through the remote family. In other words, the intermediate rebases should prefer the side that prioritizes the intended final content.

For each PickDivergent, we will also cherry-pick the original remote and common ancestor. If there is no common ancestor found, the final merges use the rebasing base instead: instead of merging `(filtered + remote - ancestor)`, we merge `(filtered + remote - base)`, where `base` is the parent we are rebasing everything onto (that is, the pick step beneath this step).
If we are dealing with merge commits, similar to the `cherry_pick` implementation, we will create a virtual parent. If creating that virtual merge base fails, we should error.

The intended behavior is to preserve the remote split and assign local amendments across that split in remote order.
If remote commits `A`, `B`, and `C` are all descended from a common ancestor `X`, and a local commit `P` also diverged from `X`, then the resolved graph should still contain `A`, `B`, and `C`. Amendments from `P` should be incorporated into whichever remote commits their hunks overlap with, and any hunks that do not get incorporated earlier should be carried forward to the last remote commit in the family.

For each remote commit in the family except the last. For now, a family is identified by a 20 byte string identifier attached to each related `PickDivergent` step:

If the hunk pool is not based on the current remote commit's parent, we first rebase it on top of the current PickDivergent's parent.
We then split the hunk pool into two trees:
- `P_filtered`, containing hunks that overlap with `diff current_remote~ current_remote`
- `P_rest`, containing hunks that do not overlap

Overlap is determined from a zero-context diff. For regular file edits, we use the changed ranges from that diff. For whole-file operations such as add, remove, or rename, we treat the entire file operation as overlapping. For now, rename-plus-edit cases are also treated as full overlap rather than trying to distribute line-level edits separately.

Remote commits in a family are processed in a defined order, from most parent to most child. That order is the global traversal order for those commits; there is no separate family-local ordering. The same order determines hunk assignment: when a hunk overlaps the current remote commit, it is taken at that point in the traversal even if it might also overlap with a later remote commit.

We use `P_filtered` in a three way merge with the current remote commit, using the common ancestor as the base when one exists. Otherwise, we use the rebasing base. For example, for commit `A`, we produce `A'` by merging `A` with `P_filtered` relative to `X`.
The resulting tree can be written as the final commit for that remote position. If the final merge of the divergent commits conflicts, we use our conflict representation to represent that result.

We then carry `P_rest` forward, rebasing it onto the next remote commit's parent before repeating the same overlap split. For example, after producing `A'`, we rebase `P_rest` onto `B~`, split it again into overlapping and non-overlapping hunks relative to `diff B~ B`, and merge the overlapping portion into `B`.

For the last PickDivergent step for the family, we use the remaining hunk pool tree directly after rebasing it onto that final remote parent. This guarantees that any hunks not incorporated by earlier overlapping matches are still incorporated into the final remote commit in the family.
