# PeerDB fork of golang.org/x/crypto

PeerDB's fork of [golang/crypto](https://github.com/golang/crypto), carrying
a single patch: the SSH channel receive window in the `ssh` package is raised
from 2 MiB to 8 MiB (`channelWindowSize` in `ssh/channel.go`), which upstream
doesn't make configurable. PeerDB moves bulk CDC traffic through SSH tunnels;
on high-bandwidth, high-latency links the 2 MiB window caps throughput.

## How it works

- `peerdb` (default branch) carries the fork-local commits: the patch, this
  README, and the sync workflow. It changes only via pull request and is
  never force-pushed. `master` mirrors upstream and is never patched.
- Releases are tags, cut by automation: fork tag `vX.Y.0` = upstream tag
  `vX.Y.0` + the fork-local commits cherry-picked on top. Fork-only fixups
  use the patch slot (`vX.Y.1`). Tags are immutable — the Go module proxy
  pins tag→hash on first fetch; never move one.
- A daily workflow (`.github/workflows/sync-upstream.yml`) looks for new
  upstream release tags, cherry-picks the fork-local commits onto them, runs
  the `ssh` package tests with PeerDB's Go version, and pushes the matching
  fork tag. On failure it opens (or bumps) a `sync-failure` issue here.
- [PeerDB](https://github.com/PeerDB-io/peerdb) consumes this fork via
  `replace golang.org/x/crypto => github.com/PeerDB-io/crypto` in
  `flow/go.mod`; Renovate there follows this repo's tags.

## Diff vs upstream

`git diff vX.Y.0 <fork tag vX.Y.z>` against the matching upstream tag —
expected to stay a handful of lines in `ssh/channel.go` plus this README and
the sync workflow.

## Maintenance

- **Sync failure**: check the open `sync-failure` issue and the failed run.
  Usually a cherry-pick conflict with a new upstream release or a test
  failure: adjust the fork-local commits on `peerdb` via PR until they apply
  cleanly, then re-run the workflow (`workflow_dispatch`).
- **Changing the patch**: PR against `peerdb`. To release the change without
  waiting for the next upstream tag, cut a patch-slot tag by hand: cherry-pick
  the fork-local commits onto the current upstream base tag and push the tag
  (if `peerdb` already sits on the current base, just tag `peerdb`). The
  PeerDB repo picks it up via Renovate.
- Keep the delta minimal.

## Not a general-purpose fork

Don't depend on this module — use `golang.org/x/crypto`. Issues in this repo
are only for the fork's automation; report `ssh` package issues
[upstream](https://go.dev/issues).

## License

Same BSD-style license as upstream; see `LICENSE`.
