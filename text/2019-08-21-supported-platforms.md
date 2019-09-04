# Supported Platforms

## Summary

Formally define TiKV's supported build and deployment platform support. Modelled
after https://forge.rust-lang.org/platform-support.html, we define 3 tiers of
support. Tier 1 support defines "guaranteed to work" or "well tested" targets,
tier 2 defines "gauranteed to build" or "ok to experiment with" targets, and
tier 3 defines "limited support" or "ok with issues" targets.

## Motivation

Recently we noticed in [#4843](https://github.com/tikv/tikv/issues/4843) we
weren't entirely clear about which platforms, architectures, operating systems,
and other variables which we do, and do not, support. In
[#5181](https://github.com/tikv/tikv/pull/5181) we examined this from a
performance angle. If we can clearly define our minimum requirements it means we
can better tune our builds to get the best performance and reliability possible.

While working on [#4514](https://github.com/tikv/tikv/pull/4514/) we reinforced
that the current build process uses a version of CentOS 7 which will become
outside of support on
[30 June 2024](https://en.wikipedia.org/wiki/CentOS#End-of-support_schedule).
Regarding the Westmere architecture, Intel does not list planned
discontinuations, they do list any [Intel Discontinued Processors]. We may in
the future find that the westmere architecture is part of this list, and in this
case we should move to a new lower-bound.

Supporting a large variety of architectures, CPU features, and operating systems
is difficult and places a large burden on the release and testing process.
Clearly defining what must be be tested gives our team the ability to focus our
efforts, while still inviting others to help us improve support.

For production deployments, our recommendations (Tier 1) remain the same, and
will remain limited to only a small selection of targets we have a long history
with. While TiKV is compatible with many targets, our community members have
years of experience maintaining critical workloads on these tier 1 targets and
feel comfortable tackling the most in-depth problems. On other (tier 2) targets
there may be differing best practices about how to manage services, how to
control system limits, or other factors which may not be well documented. We are
hopeful that over time we can build a sufficient knowledge base with other
targets to adopt them as tier 1 targets.

Still, we must recognize that while some platforms are either not well tested
(Mac OS X), or completely untested (Windows), folks still *work* on those
machines. Folks working with projects that talk to TiKV will want to have a copy
of TiKV they can access and use. Folks working on TiKV will probably not be
running a copy of their preferred deployment platform. These are *development*
targets, where users benefit from being able to harness tooling or IDE features
that require the TiKV source to at least *build* and be able to handle a couple
operations a second. So it makes sense to define a second, less rigorous
*Development platform* with tiers as well.

A good example of this is the Nix package manager. While we are quite certain
TiKV can operate on in Nix, our community has not yet buiilt TiKV with Nix. So
we cannot be sure that Nix works, and it is not a platform we can recommend to
use for development or deployment. If a community member added support, then
TiKV could easily say that Nix is a perfectly reasonable development platform
immediately. In the deployment platform case, even saying we gaurantee Nix to
build (Tier 2) could take several months before our team is confident enough to
provide official builds.

## Detailed design

This RFC proposes the following terms to be defined for TiKV:

* **Deployment Platforms:** Deployment targets for release binaries produced by
  the build process. These platforms can *run* TiKV, but not necessarily *build*
  it. This platform is the concern of end user community.
* **Development Platforms:** Development targets for interactively developing
  TiKV on. These platforms can build, debug, and run TiKV, but may not have all
  features to run a production workload. These platforms are only the concern of
  our development community.
* **Tier 1:** Official build processes build, test, and benchmark these targets,
  and they are considered endorsed by the maintainers for all usage.
      + Example: CentOS 7, our official build OS, on X86-64 with SSE4
      + Full instructions present and regularly tested
      + Tested on each PR
      + Binaries provided
* **Tier 2:** Official build processes build, but *do not require testing or
  benchmarking these platforms to suceed*. Users can feel comfortable
  experimenting or exploring TiKV with this tier, but they should *expect
  problems if used in production*.
      + Example: Mac OS, which several contributors use for testing, but we
        don't feel comfortable recommending for deployment
      + Only recommended for experimentation
      + Intructions are presented best effort, but may be incomplete or not
        regularly tested
      + Tested during major releases only
      + Binaries provided when possible and functional (Eg. Releasing OS X
        builds is not something we may always be able to do, due to new changes
        to their architecture)
* **Tier 3:** Official build processes do not build, test, or benchmark these
  targets. TiKV *should probably* build on these platforms (through official or
  ad-hoc tooling), and we'll contributor accept patches to fix or improve these
  builds. Users are strongly discouraged from using these targets for any
  purpose other than R&D.
      + Example: NixOS, a Linux distro with a slightly exotic packaging concept
      that none of our maintainers have experience with
      + Not recommended
      + No instructions provided
      + No binaries/containers provided
* **Not supported:** Testing has found the target not viable in the past. These
  targets are either not possible to build TiKV on, or known to have serious
  problems. Maintainers may accept patches to add or improve support if they are
  uninvasive, but open an issue first. *Please, don't use these.*
      + Example: Windows, we have never managed to build or successfully run
        tests on this OS
      + Not recommended
      + No instructions provided
      + No support available, bug reports about this platform may be closed
        without reason

### Target Tier/Platform Promotion

Tier promotion on the **development platform** is possible via an RFC. An RFC of
this nature should be accompanied by a series of of PRs adding content depending
on the tier promotion. Tier promotion on the **deployment platform** is
determined by maintainer consensus.

Development Platform Promotions:

* **Not supported to Tier 3:**
      + The target can successfully run `cargo check`, `cargo clippy`, `rls`, and
        `cargo fmt` on TiKV without patches or modifications
      + The target can successfully run `cargo build` and `cargo build --release`
        on TiKV without patches or modifications
      + The target can produce development builds of TiKV, they do not
        necessarily run
* **Tier 3 to Tier 2:**
      + The target can host and run TiKV as a service, and serve requests
      + The target can actively particpate in a TiKV cluster containing TiKV
        nodes running on tier 1 targets
      + The target can pass TiKV's test suite and benchmark suite
      + The proposal is submitted by a regular contributor to TiKV, and they
        state they intend to maintain support
* **Tier 2 to Tier 1:**
      + The target has reliably passed TiKVs test and benchmark suites for a
        period of 3 months
      + The target can use an interactive debugger on TiKV and documentation is present
      + The target supports at least one fuzzer and fuzzing has been performed
      + The target is used as a development machine by at least one maintainer
        or three regular contributors

### Tier Definitions

This RFC proposes the following definitions for the tiers of platforms and targets:

* **Deployment Platforms**
      + **Tier 1**:
        - CentOS 7.x.x on x86-64 on x84-64 with SSE4.2
        - Ubuntu 16.04 through 19.04 on x84-64 with SSE4.2
      + **Tier 2**:
        - Any Linux Kernel >3.x with glibc >2.17 on x86-64 with SSE4.2
        - Any Linux Kernel >3.x with glibc >2.17 on ARM64/AARCH64
      + **Tier 3**:
        - Any Linux Kernel >3.x with glibc >2.17 on x86-64 without SSE4.2
        - Any Linux Kernel >3.x with musl libc on X86-64 or ARM64/AARCH64, with
          or without SSE4.2
        - The latest Mac OS X release on X86-64 (Caution: Apple offers no
          written support cycle)
      + **Not supported**:
        - Windows
        - FreeBSD
        - Any Linux using the 'Unbreakable Enterprise Kernel' by Oracle.
* **Development Platforms**
      + **Tier 1**:
        - Any Linux kernel >3.x on X84-64 with glibc 2.17 with SSE4.2 or without
        - The latest Mac OS X release on X86-64
      + **Tier 2**:
        - Any machine with access to a Docker installation which can support
          Linux containers
      + **Tier 3**:
        - Nix (the package manager) environments
        - Any Linux Kernel >3.x with musl libc on X86-64 or ARM64/AARCH64 with
          SSE4.2 or without
      + **Not supported**:
        - Windows
        - FreeBSD
        - Any Linux using the 'Unbreakable Enterprise Kernel' by Oracle.

This list does not attempt to list all posibilities, it lists all major targets
we are asked about on a regular basis and special cases we are aware of. If
something is not listed, it should be considered untested and in the "Not
supported" tier.

### RFC Effects

If accepted, this RFC would:

* **discourage** adding this data to the TiKV documentation.
      - This should be a living development reference for our contributors and
        community. We should not encourage new or end users to try unsupported
        platforms, we'll limit our messaging to only Tier 1 platforms.
      - These targets and how we support them may change in the future, and
        maintaining separate lists is risks out of date data.
* **encourage** stating and listing our tier 1 supported targets where
  appropriate in documentation, as these specifics are of great importance to
  our readers.
* **encourage** any future changes be ammended to this RFC itself.

This RFC as currently defined does not change any of our *current* support, it
only clarifies them and defines a path forward for new targets.

## Drawbacks

* We must maintain this compatability specification and ensure correctness
* Contributors may feel encouraged to try to promote certain platforms up a tier
and require us to do more testing on releases.
* This list does not talk about hardware recommendations like CPUs and memory
since they are dependent on workload and too specific for this RFC. This is
about what hardware TiKV *can* and *can not* operate on, when not limited by
computational resources. Nor does it include specifics about which cloud, VM, or
container platforms we support. Users should refer to
https://tikv.org/docs/3.0/tasks/deploy/introduction/ for system specification
guidelines around these topics.

## Alternatives

* We could merge development and deployment tiers, but this would not be the
  full story, since TiKV builds in more places than we support deployment.
* We could not specify what we support and let our users enjoy a fun guessing game
* We could only specify supported (Tier 1) targets and not worry about partial support.
* We could limit our Tier 1 to platforms we run Jepsen/Schrodinger tests.

## Unresolved questions

The targets on each tier are currently a best guess based on author experiences.

[Intel Discontinued Processors]: https://www.intel.com/content/www/us/en/support/articles/000022396/processors.html
