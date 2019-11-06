# Unified TiKV binary

## Summary

This RFC proposes a unified TiKV binary named `tikv` that combines the
functionalities of `tikv-server`, `tikv-ctl`, and `tikv-importer` (that was
already merged into `tikv-server`).

This change would result in smaller binary sizes, a simpler build graph,
and improved usability. This change also defines backwards compatible aliases,
ensuring users of previous versions will not be burdened by any relearning or
workflow changes as a result of the upgrade.

## Motivation

### Simpler maintenance

After this change `tikv` will only build and distribute a single binary, reducing
the number of binaries we must maintain and test.

For users, this would mean they only needed to distribute one binary, and only
keep one binary up to date.

### Smaller build graph

Running `cargo +nightly build -Z timings --release` allows for the measurement
of the build time of a near-distributable build.

Timings measurements showed both `tikv-ctl` and `tikv-server` generated build
tasks that ran in parallel. Making this change would make these a single job
which would take slightly longer. While this would result in a minor slowdown in
compile time, the overall outcome would be less CPU time and less disk writes.

### Reduced learning complexity

Currently the `tikv-server` binary has a fairly simple invocation with no subcommands.
All options are passed as flags (eg. `tikv-server --log-level trace`).

`tikv-ctl` has a much more complex API service. Consider the subcommands of `tikv-ctl`:

```man
SUBCOMMANDS:
    bad-regions           Get all regions with corrupt raft
    cluster               Print the cluster id
    compact               Compact a column family in a specified range
    compact-cluster       Compact the whole cluster in a specified range in one or more column families
    consistency-check     Force a consistency-check for a specified region
    diff                  Calculate difference of region keys from different dbs
    dump-snap-meta        Dump snapshot meta file
    fail                  Inject failures to TiKV and recovery
    help                  Prints this message or the help of the given subcommand(s)
    metrics               Print the metrics
    modify-tikv-config    Modify tikv config, eg. tikv-ctl --host ip:port modify-tikv-config -m kvdb -n
                          default.disable_auto_compactions -v true
    mvcc                  Print the mvcc value
    print                 Print the raw value
    raft                  Print a raft log entry
    random-hex            Generate random bytes with specified length and print as hex
    raw-scan              Print all raw keys in the range
    recover-mvcc          Recover mvcc data on one node by deleting corrupted keys
    recreate-region       Recreate a region with given metadata, but alloc new id for it
    region-properties     Show region properties
    scan                  Print the range db range
    size                  Print region size
    split-region          Split the region
    store                 Print the store id
    tombstone             Set some regions on the node to tombstone by manual
    unsafe-recover        Unsafely recover the cluster when the majority replicas are failed
```

Moving `tikv-server` to a `tikv server --flags` and keeping all the `tikv-ctl *`
commands as `tikv *` would have minimal changes for the user.

This would allow the user to only have to consider one `tikv` binary. For
a long deprecation phase we can maintain aliases at `tikv-server` and `tikv-ctl`.

### Binary Sizes

Currently, TiKV's binaries and Docker images are fairly large, and we have been working
in issues like [#5303](https://github.com/tikv/tikv/issues/5303) to improve it.

Our k8s/Docker images:

```tsv
$ docker pull pingcap/tikv
$ docker images
REPOSITORY             TAG                                        IMAGE ID            CREATED             SIZE
pingcap/tikv           latest                                     f46ebde09c95        7 hours ago         416MB
```

Our `master` binaries are doing well:

```bash
 ls -lah bin/tikv-*
-rwxr-xr-x 1 user user  92M Nov  6 12:43 bin/tikv-ctl
-rwxr-xr-x 1 user user 134M Nov  6 12:43 bin/tikv-server
```

Our 3.0 binaries were a bit worse:

```bash
$ curl  http://download.pingcap.org/tidb-latest-linux-amd64.tar.gz | tar xz
$ ls -lah tidb-latest-linux-amd64/bin/tikv-*
-rwxr-xr-x 1 user user 225M Nov  6 11:41 tidb-latest-linux-amd64/bin/tikv-ctl
-rwxr-xr-x 1 user user 384M Nov  6 11:41 tidb-latest-linux-amd64/bin/tikv-server
```

Previously, we also had `tikv-importer` which was roughly the same size as `tikv-ctl`:

```bash
$ curl http://download.pingcap.org/tidb-v2.1-linux-amd64.tar.gz | tar xz
$ ls -lah tidb-v2.1-linux-amd64/bin/tikv-*
-rwxr-xr-x 1 user user  70M Nov  3 23:49 tidb-v2.1-linux-amd64/bin/tikv-ctl
-rwxr-xr-x 1 user user  75M Nov  3 23:49 tidb-v2.1-linux-amd64/bin/tikv-importer
-rwxr-xr-x 1 user user 175M Nov  3 23:49 tidb-v2.1-linux-amd64/bin/tikv-server
```

It is likely that moving `tikv-ctl` and `tikv-server` into the same binary
would reduce the overall binary size, similar to what happened to `tikv-importer`.

## Detailed design

These changes should be released as part of a **major** TiKV version.

1. Aliases for `tikv-server` and `tikv-ctl` are included in `etc` and distributed
2. The `tikv-ctl` codebase would be renamed to `tikv`, the `tikv-ctl` alias
   becomes active.
3. The `tikv-server` codebase will be moved to `tikv` under the subcommand
  `tikv server`. The `tikv-server` alias becomes active.
4. The `tikv.org` and associated documentation is updated.
5. Dependencies like `tidb-ansible` are updated to use `tikv`.
6. Our `scripts/gen-dockerfile.sh` is updated.
7. Possible build process simplification is explored.

## Drawbacks

* It's annoying to deal with the complexities of migration.
* We may prefer process separation (eg `dockerd` vs `docker`, `journald` vs `journalctl`)

## Alternatives

* We could continue to have process separation (eg `dockerd` vs `docker`,
  `journald` vs `journalctl`)
* Perhapops we could distribute a shared library which `tikv-server` and
  `tikv-ctl` linked to.

## Unresolved questions

* Is this the user experience we want?
