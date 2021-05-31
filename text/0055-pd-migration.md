# PD migration to TiKV Org

## Summary

This doc proposes to move the PD library from PingCAP entirely to TiKV org.

## Rationale

There have been debates in the community over PD regarding its affiliation to
TiDB and TiKV. Currently it is under PingCAP organization
(https://github.com/pingcap/pd), as a dependency library to many of the TiDB
eco-system projects. However, it’s also indisputable that PD functions as the
cluster manager, meta store and scheduling center to TiKV, which is a CNCF
project up for graduation. To clear the concern in the community, we decide to
move the PD library entirely to TiKV org (https://github.com/tikv).

## Process

- [ ] Announce to TiDB/TiKV communities. (11 AM, UTC+8, August 17, 2020)
- [ ] Update all internal `github.com/pingcap/pd` import paths in all branches
- [ ] Transfer ownership
- [ ] Set up new CIs in TiKV
- [ ] Reconfigure automated testing: Travis, Jenkins, Circle...
- [ ] Update the `import paths` and `go.mod` of all downstream dependent
    projects in `pingcap`

## Potential issues

- Can all the released TiDB versions still compile successfully?

Yes. github will [redirect all requests to the new URL](https://github.blog/2013-05-16-repository-redirects-are-here/)
and go mod download still works.

- Consider canceling the [semantic Import mechanism](https://github.com/golang/go/wiki/Modules#semantic-import-versioning)
    in PD

## Drawbacks

Possible CI downtime

## Alternatives

None.

## Unresolved questions

None.
