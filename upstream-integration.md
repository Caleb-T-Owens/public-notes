# Current implementation
Currently the upstream integration code takes on the following responsibilities:
- Dropping changes from the commit graph that have been included upstream
- Rebasing commits on top of the new master branch if they have been integrated upstream
- Merging changes into each branch (if the user has chosen not to use a rebasing approach)
- "stashing" (Unapplying) branches from your workspace.
- Resolving divergence between the local target reference and the remote target reference. IE `main` and `origin/main`.

For the new algorithm, we probably want to drop the support for resolving divergence in favour of having 
# New algorithm
We need the new algorithm to work in both single branch mode and when we've got a managed workspace. IE regardless of what `WorkspaceKind` is set to, we should be able to handle it.
We do require the workspace to have an upstream target.

