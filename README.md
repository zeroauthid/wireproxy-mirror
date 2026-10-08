# wireproxy-mirror

Permanent, org-owned, read-only mirror of
[windtf/wireproxy](https://github.com/windtf/wireproxy) (formerly
`octeep/wireproxy` — see "A note on the upstream rename" below) — a
userspace WireGuard client/proxy.

## Why this exists

`zeroauthid/claude-proxy` (see `common/procedures/it/claude-admin-proxy-architecture.md`
in `zeroauthid/systems`) has a route ("Carrie") that needs a pinned `wireproxy`:
its deploy builds `wireproxy` from this mirror at a pinned tag, checked
against the expected commit. Claude Code sessions only get GitHub access to repositories
already connected to this org, so a third-party repository like this one
can never be reached directly from a session's usual GitHub tooling. This
mirror makes it permanently reachable, without a per-session scope request
every time.

## What this mirror does NOT include: release binary assets

`git clone`/`git push` only transfers git objects (commits, trees, blobs,
tags) — **GitHub Releases and their uploaded binary assets are a separate,
non-git object store and never travel with a git mirror.** This repository
has the `v1.1.3` **tag** (a real git ref, pointing at the exact right
commit), but not the compiled `wireproxy_*` binaries/`checksums.txt`
attached to upstream's `v1.1.3` release — those still only exist at
`https://github.com/windtf/wireproxy/releases/tag/v1.1.3` (still reachable
read-only, same as any public repo). `zeroauthid/claude-proxy` builds from
source and does not use them. If its "Carrie"
route ever needs to serve those binaries from org-controlled infrastructure
rather than fetching them from upstream at deploy time, that needs a
separate, deliberate step (e.g. mirroring the release itself onto this
repository via the GitHub Releases API, or another storage location) — not
something this git-level sync provides for free.

## Where the actual mirrored content is

**This branch (`_mirror-control`) is not the mirror.** It holds only this
README and the sync workflow, and is deliberately excluded from the sync so
the workflow can't delete itself (see below). The mirrored content —
upstream's own branches and tags, exactly as `windtf/wireproxy` has them —
lives on `master` (upstream's own default branch), `udp`, and every
`v*` tag. Browse those directly; `master` is what a real
`git clone`/checkout of this repo's *content* should use, not this branch.

## Why the sync workflow isn't on `master`

`git push --mirror` (or an equivalent force-push + prune of
`refs/heads/*`/`refs/tags/*`) makes the destination match upstream exactly,
branch-for-branch — including deleting anything upstream doesn't have. If
`.github/workflows/sync-upstream.yml` lived on `master` alongside the
mirrored content, the very first successful sync run would overwrite
`master` with upstream's own tree (which has no such workflow file),
deleting the workflow that ran it. The next scheduled trigger would then
silently never fire again — GitHub only runs `schedule`-triggered workflows
that still exist on the repository's default branch. Splitting the
mirrored content (branches/tags, upstream-owned) from the sync automation
(this branch, our own) avoids that self-destruction while keeping
everything in one repository, and this branch is set as this repository's
**default branch** specifically so GitHub reads the workflow from it.

The sync workflow itself force-updates every upstream branch/tag by exact
name and prunes anything the mirror has that upstream no longer does — full
"exact ref-for-ref copy" fidelity for everything that came from upstream —
while explicitly never touching `_mirror-control`, by name.

## Rules

- **Never push directly to this repository's mirrored branches/tags**
  (`master`, `udp`, `v*`, etc.) — the next scheduled sync will overwrite or
  delete anything that isn't also on the real upstream.
- Changes to the sync workflow itself belong on `_mirror-control`.
- Sync runs daily (`workflow_dispatch` also available for a manual run) —
  see `.github/workflows/sync-upstream.yml`.

## A note on this environment's git tag-push restriction

Setting this repository up from an interactive Claude Code session (not
this workflow) hit a pre-existing, already-documented environment quirk:
pushing *any* git tag from this kind of session's own git credential is
rejected with an HTTP 403, regardless of repository or tag-protection
settings — documented in `zeroauthid/systems`'s `AGENTS.md` §8.4 (first
recorded 2026-09-03, in the repository now named `organization`) for a
completely different repository, and reproduced identically here on a
brand-new one, confirming it really is environment-wide rather than
repository-specific. `git push --delete` of a branch hit the same 403.
Neither restriction applies to GitHub Actions' own `GITHUB_TOKEN` (a
different credential entirely) — this repo's own sync workflow pushes and
deletes tags/branches normally. The initial mirror's tags were instead
created via the GitHub Git Data API (`POST .../git/refs`, and
`POST .../git/tags` first for the one annotated tag, `v1.0.4`) — which
uses a different code path than `git push` and wasn't affected. Seven of
the sixteen upstream tags (`v1.0.0`–`v1.0.3`, `v1.0.4`, `v1.0.5`,
`wireproxy`) pointed at commits not reachable from any of upstream's three
branches (an old history rewrite orphaned them) and so didn't exist yet in
this repository's object store, which the Git Data API requires; those were
brought in first via a temporary branch push (branch pushes are unaffected
by the tag-push restriction), then deleted once the real tag ref existed
independently.

## A note on the upstream rename

The GitHub account/repository this project was originally created under,
`octeep/wireproxy`, is now `windtf/wireproxy` — confirmed via the GitHub
API (`GET /repos/octeep/wireproxy` returns `full_name: "windtf/wireproxy"`,
not a fork, no `parent`/`source`, i.e. the same repository under a new
name/owner, not a different project). GitHub currently redirects the old
name transparently for both `git clone` and the REST API, which is why
requesting `octeep/wireproxy` still worked when this mirror was set up
2026-09-24 — but a redirect like that isn't guaranteed to hold forever
(e.g. if `octeep` is ever registered by someone else, GitHub would then
resolve it to *their* repository instead of forwarding it). The sync
workflow points at `windtf/wireproxy` directly for that reason.
