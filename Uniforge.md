# Uniforge

Uniforge is a high-speed forge frontend focused on providing a fast, reliable, and responsive interface over GitHub.

Its primary goal is to make working with forge data feel immediate. First contentful paint is a first-class product metric and should be taken seriously at every layer of the system, from page rendering to data loading to live updates.

Although GitHub is the initial upstream, Uniforge should be designed around forge-agnostic internal models. In the long term, it should be capable of supporting multiple forge backends, including a native Uniforge backend.

# Stack
Ruby on Rails
Idiomatic Rails frontend using Turbo and Turbo Streams for interactivity
Ethos of never needing to refresh a page to receive updates

## MVP user flows

The initial MVP should focus on code review flows.

As a user, I can:
- list all pull requests in a project
- open a pull request and view its description
- open a pull request and view its diff
- view comments on a pull request
- add a top-level comment to a pull request
- add a comment on a diff within a pull request
- mark feedback as resolved

For diff rendering, a dedicated diff rendering library should be considered. `diffs` appears to be a strong option because it is explicitly focused on rendering code and diffs for the web, supports server-side rendering, and is performance-conscious in its architecture.

## Testing framework
### Emulated GitHub API.
- Relevant endpoints generated from GitHub REST API and WebSocket API documentation
### Emphasis on system tests
System tests give a full picture of the system's health so encoding real-life scenarios and workflows is a priority
# Basic architecture
## Owned data model
Uniforge maintains it's own copy of data that has a two way sync with GitHub or GitLab. Uniforge data structures are thought from the ground up and not tied to legacy data structures of GitHub.

#### User
Represents a real user that logs in to the application

- email (unique)
- username
- password (hashed)

#### Organisation
All repositories belong to an organisation. Unlike GitHub, a User never directly owns a repository.
- name
- token (encrypted)
#### UserOrganisationMembership
- role (owner, member)
- user_id
- organisation_id
#### Repository
We have both a repository record and a checkout of the repository that we maintain.
- name
- remote_id (JSON, used to identify the repo on the remote, this may be different per forge)
- organisation_id
#### Pull Request
Has a minimal set of properties to represent the PR, oids to display the diff, ect...
- description
- remote_id (JSON, used to identify the PR on the remote, this may be different per forge)
- repository_id
- author_id (optional)
- remote_author_id (required)
#### Review Thread
Code review discussion should be modeled around threads rather than individual comments.

A review thread may either be attached to the pull request generally or anchored to a location in the diff. Resolution state should belong to the thread.

- subject_type (general, diff)
- repository_id
- pull_request_id
- resolved_at (optional)
- metadata (JSON, provider-specific details)

#### Comment
Comments belong to a review thread. A thread may contain a single top-level comment or a small conversation.

- content
- author_id (optional)
- remote_author_id (required)
- review_thread_id
- parent_comment_id (optional)
- metadata (JSON, provider-specific details)

For MVP purposes, comments should support both top-level pull request discussion and diff-associated feedback without exposing forge-specific terminology in the core model.

### Syncing data
There is a two way syncing problem, and records can be invalidated on both sides.

Initially there are three sets of data we need to sync:

- Repository contents
	- One way, from the true forge
- Pull request details
	- Two way, updates to the description get reflected into GH, and vice versa
- Pull request comments

We should have a robust setup that is error resilient and is as responsive as possible without abusing APIs.

Review discussion should be normalized into a forge-agnostic thread model. Different providers represent pull request discussion differently, so provider-specific adapters should translate remote data into a shared internal representation rather than leaking provider concepts throughout the application.

In general:
- top-level pull request discussion should be representable in the same system as diff discussion
- diff location should be modeled separately from comment content
- resolution should be modeled at the thread level
- provider-specific metadata should be preserved when needed without becoming part of the core domain model

The goal is not to erase differences between providers completely, but to define a stable internal model that can support GitHub first while remaining adaptable to GitLab and a future native backend.

## Minimal viable sync architecture

The initial sync layer should be intentionally simple.

Uniforge should maintain a local copy of the data it needs to render quickly and update live, but it does not need a large distributed-systems architecture in the MVP.

A minimal viable sync architecture should include:

- local tables for repositories, pull requests, review threads, and comments
- provider-specific sync code at the boundary of the system
- webhook ingestion to detect that a repository or pull request has changed
- background jobs to re-sync changed data
- a small amount of periodic reconciliation so missed webhook events do not leave the system stale
- write actions that update the remote forge first and then re-fetch the affected pull request state

For MVP purposes, the simplest mental model is:
- GitHub remains the source of truth for forge data
- Uniforge keeps a local mirror optimized for reads
- user actions are sent to GitHub and then reconciled back into local state

The sync layer should be organized around a few coarse jobs rather than a large number of fine-grained pipelines. A good starting point is:
- sync repository
- sync pull request
- sync review discussion for a pull request

Webhooks should be treated as triggers, not as the source of truth. When a webhook arrives, Uniforge should enqueue a refresh of the affected repository or pull request rather than trying to fully apply remote state directly from the webhook payload.

Likewise, a lightweight periodic reconciliation pass should re-fetch recently active pull requests and repositories. This keeps the system correct if a webhook is delayed, dropped, or incomplete.

The MVP should also prefer idempotent sync operations. Re-running a sync job for the same repository or pull request should be safe and should converge on the same local state.

This approach is deliberately conservative. It avoids over-engineering early, while still establishing the important architectural boundaries:
- the forge adapter is isolated from the core application model
- the UI reads from local state rather than from live provider requests
- remote writes are reconciled through sync rather than assumed locally
- future providers can be added behind the same basic pattern

More advanced concerns such as fine-grained event logs, sophisticated rate-limit scheduling, or highly specialized projection pipelines can be added later if the product proves they are necessary.