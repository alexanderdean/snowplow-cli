# Spec: Merge `snowplow-tracking-cli` into `snowplow-cli`

**Status:** Draft
**Owner:** TBD
**Branch:** `spec/merge-tracking-cli`

## Summary

Absorb the standalone `snowplow-tracking-cli` (currently at `github.com/snowplow/snowplow-tracking-cli`, v0.7.0, Apache 2.0) into this repository as a new `events` command family. The result: one Snowplow CLI binary that can both manage tracking design via Console (existing `data-products` / `data-structures`) *and* track events to a collector (new `events track-one`). The standalone repo is then archived.

## Motivation

- A single Snowplow CLI is the right user experience long-term; today users discover two tools with overlapping names and no clear relationship.
- The tracking CLI has been effectively unmaintained since 2022 (last release v0.7.0, Go 1.19). Folding it into an actively maintained binary gets it free dependency and Go upgrades.
- Shell-script tracking and Snowplow tracking-design workflows naturally co-occur in CI/CD. Having one binary on `$PATH` simplifies pipelines.

## Goals

1. All current `snowplow-tracking-cli` functionality is available as `snowplow-cli events track-one ...` with no loss of capability.
2. The merged binary uses Cobra (consistent with the rest of the CLI), not urfave/cli.
3. The tracking subcommand requires **no Console credentials** — it remains stateless, taking collector and event details as flags. It must not break for users who run it without any Snowplow Console config present.
4. Exit-code semantics from the tracking CLI are preserved (0/4/5/1 by HTTP class).
5. The old repo is archived with a `README` pointer to the new home, and the old Docker image / binaries are clearly deprecated.

## Non-goals

- Redesigning the tracking flag surface. The first cut keeps flag names and semantics 1:1 with the existing tool (modulo Cobra-style long flags); naming cleanup is a follow-up.
- Adding batching, buffering, retries, or persistent storage beyond what the existing tool does.
- Building any Console-integrated event tracking (e.g. auto-discovering schemas from the org). Out of scope.
- Reimplementing the Snowplow Go tracker. We keep using `snowplow-golang-tracker/v3`.

## Current state (what we're absorbing)

`snowplow-tracking-cli` is a single-file `main` package (`snowplowtrk.go`, ~292 lines) with:

- urfave/cli v1 as the framework
- `github.com/snowplow/snowplow-golang-tracker/v3 v3.0.0` for transport
- Flags: `--collector`, `--appid`, `--method`, `--protocol`, `--sdjson`, `--schema`, `--json`, `--ipaddress`, `--contexts`
- Behaviour: send exactly one self-describing event, block on collector response, exit with 0/4/5/1
- Build: Makefile + gox cross-compile, Dockerfile (Alpine), distributed via GitHub Releases and Docker Hub (`snowplow/snowplow-tracking-cli`)
- License: Apache 2.0 (compatible-to-absorb under our SLULA — see "License" below)
- Tests: `snowplowtrk_test.go` with HTTP mocking

## Proposed command surface

```
snowplow-cli events
snowplow-cli events track-one [flags]    # primary command — tracks one event, configured inline via flags
```

`track-one` is the only leaf at launch. The name encodes three things: the canonical Snowplow verb (*track*, matching every tracker SDK), the count (*one*), and — by contrast with the future siblings below — the fact that the event is composed inline from flags rather than read from a file.

### Namespace and future siblings

`events` is a noun-shaped family (the thing being acted on) and leaves are verbs (or verb-noun phrases) operating on it. This gives us a stable namespace as the family grows. The siblings we already anticipate:

```
snowplow-cli events track-one  [flags]                 # singular, inline-configured (this spec)
snowplow-cli events track-bulk <file> [flags]          # many events from a JSONL/NDJSON file
snowplow-cli events replay     <recording> [flags]     # re-emit a captured event stream
snowplow-cli events validate   <file>                  # check schema references locally without sending
```

Two conventions worth pinning down so additions stay coherent:

- **`track-*` clusters the "send to a collector" operations.** Anything that produces real traffic against a collector belongs under a `track-` prefix; this keeps related commands adjacent in `--help` output and makes the safety implication ("this will emit events") visible at a glance.
- **Bare verbs are reserved for non-emitting operations on events** (`replay` is borderline because it does emit, but it's named after the source action; `validate` is the canonical bare-verb shape — local-only, no network).

We deliberately did **not** add a `track` alias on the `events` family. With `track` already inside the leaf name, an alias would yield `snowplow-cli track track-one`, which reads badly. Users type the verb where it belongs — in the leaf.

### Flags on `events track-one`

| Long flag      | Short | Required | Default       | Notes                                                           |
|----------------|-------|----------|---------------|-----------------------------------------------------------------|
| `--collector`  | `-c`  | yes      | —             | Collector domain, e.g. `collector.acme.com`                     |
| `--app-id`     |       | no       | `snowplow-cli`| Renamed from `--appid` for Cobra-style; keep `--appid` as hidden alias for one release |
| `--method`     | `-m`  | no       | `POST`        | `POST` or `GET`. **Default flipped from GET to POST** — see "Behavioural changes" |
| `--protocol`   | `-p`  | no       | `https`       | `http` or `https`                                               |
| `--sdjson`     |       | no       | —             | Full self-describing JSON: `{"schema":"iglu:...","data":{...}}` |
| `--schema`     |       | no       | —             | Schema URI; used with `--json`                                  |
| `--json`       |       | no       | —             | Event data JSON; used with `--schema`                           |
| `--ip-address` |       | no       | —             | Custom IP. Renamed from `--ipaddress`; keep old as hidden alias |
| `--contexts`   |       | no       | `[]`          | JSON array of self-describing contexts                          |

Mutual exclusion: exactly one of (`--sdjson`) or (`--schema` + `--json`) must be supplied. Validate in `RunE` and return a Cobra error.

No Console credentials. The `events` command family does **not** call `config.InitConsoleConfig` in its `PersistentPreRunE` — it only calls `snplog.InitLogging`.

### Exit codes

Preserve existing semantics:
- `0` — collector returned 2xx/3xx
- `4` — collector returned 4xx
- `5` — collector returned 5xx
- `1` — any other error (bad flags, network, JSON parse)

Implementation: leaf `RunE` returns a typed error and `cmd.Execute()` maps it to the right exit code. May require a small wrapper in `cmd.Execute()` since today it just does `os.Exit(1)` on any error — see "Implementation plan" step 4.

## Repository layout changes

```
cmd/
  events/                         NEW package
    events.go                       Family root command, no Console config init
    track_one.go                    The `track-one` leaf
    track_one_test.go               Port of snowplowtrk_test.go
internal/
  tracker/                        NEW package
    tracker.go                      initTracker, trackSelfDescribingEvent (extracted from snowplowtrk.go)
    json.go                         getSdJSON, getContexts, stringToMap helpers
    tracker_test.go
go.mod                            Add `github.com/snowplow/snowplow-golang-tracker/v3`
cmd/root.go                       Register events.EventsCmd
```

Rationale for splitting `cmd/events/` and `internal/tracker/`: matches the established pattern (see `cmd/dp/` + `internal/console`, `cmd/ds/` + `internal/console`). Command code stays thin; the tracker plumbing lives in `internal/`.

## Absorption strategy: copy, not subtree/submodule

**Decision: plain `cp` of the source files into this repo, with attribution preserved via `NOTICE` + `CHANGELOG`. Do not use git submodule, git subtree, or import the upstream repo as a Go module.**

Stage the absorption as two commits so the relicensing is auditable:

1. **Verbatim copy commit** — bring `snowplowtrk.go` and `snowplowtrk_test.go` into the repo unchanged: original Apache 2.0 headers intact, original file names, dropped into a temporary location (e.g. `internal/tracker/_import/`). Commit message names the source commit SHA of the upstream repo so provenance is recorded in git.
2. **Relicense + refactor commit** — split the file into the layout above (`cmd/events/`, `internal/tracker/`), swap Apache 2.0 headers for the SLULA v1.0 header used elsewhere, rewrite imports, rewire the CLI from urfave/cli to Cobra. The diff against commit 1 is exactly the set of changes legal would want to see in a relicensing review.

Both commits land in the same PR.

### Why not the alternatives

- **`git submodule`** — wrong tool for source code we intend to actively maintain. Pins us to a soon-to-be-archived upstream, complicates the build/CI, and blocks the refactor into Cobra (you can't restructure code without breaking the submodule contract). Submodules are appropriate for vendored binaries or third-party deps we don't own — neither applies.
- **`git subtree add`** — preserves upstream commit history in a subdirectory; the "respectful" option. Rejected because (a) we're relicensing the absorbed code from Apache 2.0 to SLULA v1.0 and reorganising it into Cobra-shaped files, so the preserved history would not `git blame` cleanly to the new layout anyway; (b) it pulls Apache-licensed commits into a SLULA repo, which muddies licence-scanner output downstream; (c) the upstream history is small enough (a few dozen commits, last meaningful change in 2022) that the archaeology value over a `NOTICE`-file pointer to the archived repo is negligible.
- **Go module dependency (`go get github.com/snowplow/snowplow-tracking-cli`)** — non-starter. The upstream is `package main`, not an importable library. Even if we extracted the tracker plumbing into a `package` in the upstream first, that would create a dependency on a repo we're about to archive — strictly worse than just copying the ~292 lines in.
- **`git merge --allow-unrelated-histories` at the top level** — would interleave two unrelated commit trees, produce a confusing log, and gain us nothing the two-commit copy approach doesn't.

### Attribution

The original authors (Joshua Beemster, Ronny Yabar, et al.) get credit via:
- A `NOTICE` file at the repo root naming them and pointing at the upstream Apache 2.0 licence and source repo.
- A `CHANGELOG` entry for the release that introduces `events track-one`, explicitly crediting the upstream tool.
- Source file comments are *not* used for per-author attribution — the `NOTICE` file is the authoritative record. This matches how most Apache-2.0-to-other-licence absorptions are handled in practice.

## Behavioural changes vs. old tool

1. **Default `--method` flips from `GET` to `POST`.** GET-tracking is essentially never the right default in 2026 — POST handles larger payloads, doesn't leak data into collector access logs, and matches what every Snowplow tracker SDK does today. Call this out prominently in the migration note. Users who genuinely need GET pass `--method GET`.
2. **`--appid` → `--app-id`**, **`--ipaddress` → `--ip-address`** for consistency with the rest of the CLI's kebab-case flags. Keep old names as hidden aliases for one release, emitting a deprecation warning on use.
3. **Default `--app-id`** changes from `snowplowtrk` to `snowplow-cli` so events from the merged binary are identifiable as such.
4. **License header on absorbed code** changes from Apache 2.0 to the Snowplow Limited Use License Agreement v1.0 to match the rest of this repo. The original Apache copyright notice for 2016–2022 contributions is preserved in `NOTICE` (see "License" below).

No other behavioural changes. Event payload construction, callback timing, IP-address handling, and exit-code mapping are byte-for-byte the same.

## License

The tracking CLI is Apache 2.0; this repo is SLULA v1.0. Apache 2.0 permits relicensing of forks/copies under different terms provided attribution is preserved. The mechanics of relicensing (verbatim-copy commit, header swap, `NOTICE` file) are detailed under "Absorption strategy" above.

Before merging the PR: confirm with Snowplow legal that this is the desired posture (existing SLULA-licensed binary absorbing an Apache-2.0-licensed tool). If there's a preference to keep the tracking subcommand independently Apache-licensed, the alternative is to keep `internal/tracker/` and `cmd/events/` under Apache 2.0 with file-level headers — Go's build doesn't care, but downstream licence-scanning will pick up the mix.

## Implementation plan

Each step is a single PR.

1. **Vendor the tracker dependency.** Add `github.com/snowplow/snowplow-golang-tracker/v3 v3.0.0` to `go.mod`. Verify it builds clean on Go 1.25.5 (it's been pinned to 1.19 in the source repo — likely fine, but a smoke test catches API drift).
2. **Absorb the upstream source — two commits in one PR** (see "Absorption strategy" above):
   - **2a. Verbatim copy.** Drop `snowplowtrk.go` and `snowplowtrk_test.go` into `internal/tracker/_import/` unchanged, Apache 2.0 headers intact. Commit message records the upstream commit SHA for provenance.
   - **2b. Relicense + refactor.** Split into `cmd/events/` (family root + `track-one` leaf) and `internal/tracker/` (`initTracker`, `trackSelfDescribingEvent`, `getSdJSON`, `getContexts`, `stringToMap`, `parseStatusCode`, plus the ported test). Replace Apache 2.0 headers with the SLULA v1.0 header used elsewhere. Rewire urfave/cli flag parsing into Cobra. Delete `internal/tracker/_import/`. Wire flags as per the table above, including hidden aliases for `--appid` / `--ipaddress`. Register `EventsCmd` in `cmd/root.go` (no Console config init in its `PersistentPreRunE`).
3. **Add `NOTICE` file.** Credit the original `snowplow-tracking-cli` authors, link to the upstream Apache 2.0 licence and archived repo.
4. **Exit-code wiring.** Introduce a typed error in `internal/tracker` (e.g. `HTTPStatusError{Class int}`) and update `cmd.Execute()` in `cmd/root.go` to map it to `os.Exit(4|5|1)`. Default existing path to `1` (no change for current commands).
5. **README + docs.** Add an `Events` section to the main README with examples mirroring the old tool's docs. Update `cmd/docs.go` output expectations if needed.
6. **CHANGELOG entry.** Record the absorption, credit upstream authors, list the flag renames and the GET→POST default flip.
7. **Deprecate the upstream repo.** Once the first release of `snowplow-cli` containing `events track-one` is out: rewrite `snowplow-tracking-cli`'s README to point at this binary, push a final v0.8.0 release with the deprecation notice, and archive the GitHub repo. Push a final `snowplow/snowplow-tracking-cli:deprecated` Docker tag with the same README.

## Distribution

No new artefacts needed. The existing release pipeline (`cd.yaml` → GitHub Releases, Homebrew, npm) ships the merged binary automatically. Users invoke `snowplow-cli events track-one ...` instead of running a separate binary.

Docker: the existing `snowplow/snowplow-tracking-cli` image keeps working for legacy users. We do *not* need to publish a new image for the merged CLI as part of this work — but if Docker distribution is desired going forward, that's a separate spec.

## Migration guide for users (for inclusion in README)

```
# Before
snowplow-tracking-cli --collector c.acme.com --schema iglu:.../1-0-0 --json '{"hello":"world"}'

# After
snowplow-cli events track-one --collector c.acme.com --schema iglu:.../1-0-0 --json '{"hello":"world"}'
```

Heads-up notes:
- Default `--method` is now `POST` (was `GET`). Pass `--method GET` to restore old behaviour.
- `--appid` → `--app-id`, `--ipaddress` → `--ip-address`. Old names still work for now but emit a deprecation warning.
- The default app ID in collector logs changes from `snowplowtrk` to `snowplow-cli`. Pass `--app-id snowplowtrk` to match historical data.

## Open questions

1. **License posture** — confirm SLULA absorption is what legal wants (see "License" section).
2. **Tracker Go version drift** — does `snowplow-golang-tracker/v3 v3.0.0` build cleanly on Go 1.25.5, or do we need a v3.0.1 release first?
3. **GET-default flip** — comfortable doing this as a silent behaviour change in the migrated tool, or do we want `events track-one` to *error* if `--method` is omitted, for one release, to force users to make an explicit choice?
4. **MCP exposure** — should `events track-one` be exposed as an MCP tool? Probably yes (sending a test event from an LLM agent is a natural use case) but out of scope for the initial merge.
