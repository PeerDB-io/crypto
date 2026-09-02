# PeerDB fork of golang.org/x/crypto

PeerDB's fork of [golang/crypto](https://github.com/golang/crypto). It
carries one patch: the SSH channel receive window in the `ssh` package is
raised from upstream's hardcoded 2 MiB to 8 MiB (`channelWindowSize` in
`ssh/channel.go`). PeerDB moves bulk CDC (change data capture) traffic
through SSH tunnels, and SSH throughput caps at window ÷ round-trip time, so
on high-bandwidth, high-latency links the 2 MiB window holds throughput
below link capacity.

## Layout

| Ref | Contents | How it changes |
|---|---|---|
| `peerdb` (default branch) | Latest upstream release tag + the fork's commits: the patch, this README, the sync workflow | Sync workflow rebases and force-pushes it on each upstream release; manual changes via pull request |
| `master` | The upstream base of the current release, unpatched | Fast-forwarded by the sync workflow |
| `vX.Y.N` tags | Upstream tag `vX.Y.0` + the fork's N commits | Cut by the sync workflow on upstream releases; by hand for releases in between |

The patch version is the number of extra commits in the fork. Upstream only
ever tags `vX.Y.0` and the fork always has at least one extra commit, so
every fork tag
gets a fresh name and a published tag is never moved or recreated — which is
what the Go module proxy requires: it pins tag→hash on the first fetch.
(`v0.55.0` predates this scheme.) The full fork delta is
[`master...peerdb`](https://github.com/PeerDB-io/crypto/compare/master...peerdb).

## Staying current with upstream

`.github/workflows/sync-upstream.yml` runs daily (and on manual dispatch):

1. Computes the target tag: the latest upstream release's `vX.Y` plus the
   number of extra commits in the fork. A release is due whenever that tag
   doesn't exist yet — a new upstream release, freshly merged fork PRs, or
   both.
2. Rebases the fork's commits onto the upstream tag when the base moved.
3. Builds all packages and tests the root `ssh` package, using the Go
   version from PeerDB's `flow/go.mod`.
4. Force-pushes `peerdb` and pushes the target tag.

A fully green run pings a heartbeat monitor as its last step; when pings
stop — a failing run or a schedule that stopped firing — the team is
alerted.

Validation covers the root `ssh` package, which runs full in-memory
handshakes against the patched code. The `ssh/test` and `ssh/agent`
packages replay recorded transcripts that embed upstream's 2 MiB window, so
they fail against this patch and are excluded.

## How PeerDB consumes it

`flow/go.mod` in [PeerDB](https://github.com/PeerDB-io/peerdb) keeps
`require golang.org/x/crypto` and adds
`replace golang.org/x/crypto => github.com/PeerDB-io/crypto`. Renovate in
that repo follows this repo's tags and opens the bump PRs.

## Maintenance

- **Sync failure**: check the latest run under Actions. Typical causes: a
  rebase conflict with a new upstream release, or an `ssh` test failure.
  Adjust the fork's commits on `peerdb` via PR until they apply cleanly,
  then re-run the workflow.
- **Changing the patch**: PR against `peerdb`, then re-run the workflow
  (manual dispatch) to cut the tag.
- Keep the delta minimal.

## Scope

This fork exists for PeerDB. Other projects should depend on
`golang.org/x/crypto`, and `ssh` package issues belong
[upstream](https://go.dev/issues). Issues in this repo cover the fork's
automation only.

## License

Upstream's BSD-style license applies; see `LICENSE`.
