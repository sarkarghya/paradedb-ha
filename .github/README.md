# ParadeDB HA

A fork of [ParadeDB](https://github.com/paradedb/paradedb) whose `pg_search` keeps
WAL-logged BM25 indexes working across PostgreSQL physical replication and Patroni
failover. Since v0.27.0, stock Community keeps a standby replaying, but marks the
replicated indexes and refuses them after promotion until each one is rebuilt with
`REINDEX`. This build serves the replicated index immediately, with no reindex.

It is experimental and not ParadeDB Enterprise. **Search on a standby is still
unsupported**; send search traffic to the primary only.

## Layout

The `paradedb-ha` branch is an upstream release tag plus a small commit stack. Upstream
handles standby replay and refuses standby reads itself; the runtime patch only lets a
promoted server serve the indexes it replicated:

| Path | Purpose |
| --- | --- |
| `storage/src/postgres/rel.rs` | `blocked_after_promotion()` returns false in this build. Replay still writes upstream's taint, so a stock build would refuse these index files again |
| `common/src/gucs.rs`, `common/Cargo.toml`, `pg_search/Cargo.toml` | The `experimental-physical-failover` Cargo feature and the read-only `paradedb.physical_failover` setting |
| `tests/tests/replication.rs` | Physical replication and promotion regression; `PG_SEARCH_TEST_PHYSICAL_FAILOVER=1` asserts promoted indexes answer and verify without REINDEX |
| `tests/physical_failover/` | Patroni elections and rejoining smoke test |
| `docker/Dockerfile.paradedb-ha` | Build, test and export the extension |
| `.github/workflows/paradedb-ha.yml` | CI for AMD64 and ARM64; publishes releases from tags |
| `fork/release.json` | Pinned upstream tag/commit and fork revision |
| `fork/sync-upstream.sh` | Merges a new upstream release |

To see the whole fork diff: `git diff "$(jq -r .upstreamCommit fork/release.json)" -- common pg_search storage tests`.

Upstream SQL, the control file and `Cargo.lock` must stay identical to the pinned upstream
release (CI enforces this), so the extension version stays `0.27.0`; the fork revision
only identifies our binary.

## Updating to a new upstream release

```sh
fork/sync-upstream.sh          # pinned vs latest upstream release
fork/sync-upstream.sh v0.27.1  # creates sync/v0.27.1 and merges the tag
```

Resolve any conflicts (review upstream storage, WAL and planner changes carefully), then
push `sync/v0.27.1`. Upstream cuts minor releases from `main` and patch releases from
release branches, so a merge that crosses those lines can conflict in files the fork never
touched. In that case resolve to the exact upstream tree plus the fork-owned paths listed
above, reapply the runtime patch as its own commit, and check the result with
`git diff <tag> -- . ':!.github/README.md' ':!.github/workflows/paradedb-ha.yml' ':!docker/Dockerfile.paradedb-ha' ':!fork' ':!tests/physical_failover'`. CI builds both architectures and runs the replication regression and
Patroni smoke test. When it passes, merge the branch into `paradedb-ha`.

Never merge upstream `main`; only stable release tags.

## Releasing

Tag the tested commit and push the tag. The tag must match `fork/release.json`:

```sh
git tag paradedb-ha-0.27.0-r1 && git push origin paradedb-ha-0.27.0-r1
```

CI rebuilds and retests, then publishes a prerelease with
`pg_search-pg18-trixie-{amd64,arm64}.tar.gz` and `SHA256SUMS`. Each tarball holds the
`usr/` tree, `BUILD-INFO`, `LICENSE` and `SOURCE.tar.gz`. For a new binary on the same
upstream version, bump `revision` in `fork/release.json`. Never replace published assets.

Fork tags never start with `v`, so upstream's inherited release workflows (Debian, PGXN,
Docs and others), which run on `v*` tags, are never triggered. Keep them disabled in the
Actions settings anyway.

## Using a release

Every cluster member must run the same build with `pg_search` in
`shared_preload_libraries` before any logged BM25 index exists. `SHOW
paradedb.physical_failover` returns `on` only on this build. Never roll back to Community
releases before v0.27.0 while logged BM25 indexes exist: their standbys stop at the index
WAL. Stock v0.27.0 and later is a safe fallback, but promoted indexes then need `REINDEX`.

```dockerfile
ARG TARGETARCH
RUN curl -fsSL -o /tmp/pg_search.tar.gz \
      "https://github.com/sarkarghya/paradedb-ha/releases/download/paradedb-ha-0.27.0-r1/pg_search-pg18-trixie-${TARGETARCH}.tar.gz" \
    && echo "<pinned sha256>  /tmp/pg_search.tar.gz" | sha256sum -c - \
    && tar -xzf /tmp/pg_search.tar.gz -C / usr
```

The extension depends on `postgresql-18-pgvector` and `libopenblas0-pthread`, and
`CREATE EXTENSION pg_search` needs the `vector` extension first (or `CASCADE`).

When upgrading a running cluster, follow upstream's order: upgrade standbys before the
primary, because v0.27.0 primaries write an index WAL record older builds cannot replay.

## Running tests locally

```sh
cargo pgrx install --package pg_search --release \
  --features experimental-physical-failover --pg-config "$PG_CONFIG"
PG_SEARCH_TEST_PHYSICAL_FAILOVER=1 cargo test --locked --package tests \
  --test replication test_wal_streaming_replication_with_pg_search -- --exact --nocapture

docker build -f docker/Dockerfile.paradedb-ha --target patroni-test -t paradedb-ha:local .
PARADEDB_HA_IMAGE=paradedb-ha:local python3 tests/physical_failover/smoke.py
```

## License

ParadeDB HA retains ParadeDB's [AGPL-3.0 license](../LICENSE). It is not an official
ParadeDB release.
