## fn workspace_upgrades<T: RefMetadata>(
Why can we have empty workspace segments?

Does the `to_workspace_state` call end up reading a bunch of commit messages? I'm a little surprised by this. Perhaps some explanation of _why_ this makes sense would be nice.

I guess this is doing something to disambiguate between two stacked references and what should be two independent, but the logic is a tad confusing, and seems to rely on a primitive version of constructing the workspace which seems a little circular logic wise.

I would love to know more about what this is trying to achieve and the algorithm it uses to achieve it.
### fn candidates_for_independent_branches_in_workspace(
It's not clear to me what makes something a candidate.