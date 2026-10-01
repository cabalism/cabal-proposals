# Package sources at a chosen path

## Summary

Give `cabal get` a way to unpack a package's source to exactly the directory
the user names, and let a project declare such **source trees** — package
sources it needs in its tree but does not build — in `cabal.project`, so
that cabal can fetch, pin and verify them. This covers tooling delivered as
Hackage packages (Updo), and package sources that another build system
consumes at a path of its choosing (a Buck2 build of a Hackage package),
without stretching `cabal vendor`, which is a function of the build plan.

## Motivation

Two arrangements in and around the cabal repository need a package's source
at a chosen path, and neither can use cabal for it today:

- **Updo** ([up-do/updo](https://github.com/up-do/updo)) is a Hackage package
  used as a make-and-Dhall toolkit: a consuming project `include`s its
  makefiles and runs its scripts in place, and never compiles it. It
  bootstraps itself with a make rule that fetches an archive — the Hackage
  source distribution of a pinned version, or a GitHub archive of a pinned
  commit — unpacks it, and renames `updo-1.0.0/` to `updo/`. The
  documentation explains the workaround: "`cabal get` doesn't support
  unpacking to a specific directory that does not match the package name
  with a version suffix", and `--destdir` only picks the parent. `stack
  unpack --to` has the same limitation. Pinning and provenance live in
  makefile variables (`UPDO_VERSION`, `UPDO_COMMIT_HASH`) and an `$(info)`
  line, with no verification of what was downloaded.
- **Buck2 builds of this repository** need the source of `hackage-security`
  — a Hackage dependency that cabal builds in-place, because it depends on
  the repository's own `Cabal-syntax` — at `project-buck2/vendor/hackage-security/src/`,
  where a hand-written `BUCK` file globs it. Buck2 rules can only see their
  own directory, so the path is fixed by the consumer and must not carry a
  version. A script copies the tree out of cabal's private
  `dist-newstyle/src/`.

The common shape is: *a package's source, at a version or commit the project
pins, unpacked at a path the project chooses, kept up to date, with cabal
doing the fetching and verifying.* Today cabal offers three partial answers,
none of which fits:

- `cabal get PKG` — fetches and verifies from any configured repository and
  applies the index revision, but always unpacks to `<pkg>-<version>/`; and
  it is not project-aware ([#8584](https://github.com/haskell/cabal/issues/8584)),
  so it does not see repositories or `index-state` from `cabal.project`.
- `source-repository-package` — pins a VCS ref, but only for packages the
  project builds; the checkout lives in `dist-newstyle/src/<mangled>-<hash>/`
  and is gone with `cabal clean`.
- `cabal vendor --unpack` (companion proposal) — unpacks a dependency, but by
  construction only one that is in the build plan; Updo is not.

What is deliberately *not* in scope is fetching arbitrary non-package trees.
The Buck2 rules themselves (a `haskell-buck2` fork: Starlark and toolchains,
no `.cabal` file) are pulled in as a git submodule, and that is the right
tool: cabal knows how to fetch and verify packages, and should not become a
general downloader.

## Proposed Change

### 1. `cabal get --into DIR`

A new option on `cabal get` naming the directory to unpack into, replacing
the `<pkg>-<version>` name rather than nesting under it:

```
cabal get updo --into updo
cabal get hackage-security --into project-buck2/vendor/hackage-security
```

Semantics: the package's source distribution is unpacked so that its
top-level files land directly in `DIR` (created as needed; refused if it
exists and is not empty, as `cabal get` refuses today), with the index
revision applied to the `.cabal` file as usual. `--into` and `--destdir` are
mutually exclusive. It combines with `--source-repository`, in which case the
clone goes to `DIR` (today it goes to `<pkg>/`, which is already unversioned
— `--into` just makes the two modes consistent). With
[#8584](https://github.com/haskell/cabal/issues/8584) done, the package is
resolved against the project's repositories and `index-state`, so a vendored
no-index repository works as a source without extra flags.

This alone replaces Updo's make rule with a one-liner and the Buck2 copy
script with `cabal get hackage-security --into …`, at the cost of still
keeping the version pin and the invocation in a script.

### 2. Declared source trees (for discussion)

To move the pin and the invocation into the project file, a project could
declare the trees it needs:

```
source-tree
  path: updo
  package: updo ==1.0.0

source-tree
  path: project-buck2/vendor/hackage-security
  package: hackage-security
  -- version taken from the build plan, like cabal vendor --unpack

source-tree
  path: tools/updo
  type: git
  location: https://github.com/up-do/updo
  tag: 60545b108b7a6a2f802ec7a161aa4b9eb7441baf
```

A `source-tree` names a path and a source: a package with a version
constraint (resolved against the project's repositories at its `index-state`;
an unconstrained package that is also a dependency takes the plan's version),
or a VCS location and ref in the `source-repository-package` vocabulary. A
new command — `cabal sync-sources` (name open) — materialises every declared
tree that is missing or whose recorded provenance no longer matches the
stanza, writing a small marker file (`.cabal-source-tree`: origin, version or
commit, tarball hash) that later runs compare against. Trees are otherwise
opaque to cabal: they are not packages of the project, never built, never
part of the plan, and `cabal build` does not touch them. That last point is
what distinguishes this from `packages:` and `source-repository-package`.

Because the trees are opaque, the feature needs no solver involvement and no
change to the plan; it is fetching plus bookkeeping, reusing `fetchRepoTarball`
(hackage-security verification for secure repositories), `unpackPackage`
(revision applied), and the VCS syncing used for `source-repository-package`.

## Alternatives Considered

- **Extend `cabal vendor --unpack` with a destination** (`--unpack pkg=path`).
  Serves the Buck2 case, since `hackage-security` is in the plan, and the
  companion proposal lists it as an option. It cannot serve Updo, which is in
  no plan, and using the vendor command for something that is not vendoring
  muddles both. `--into` on `get` serves both and is the primitive the vendor
  command would use anyway.
- **Only `cabal get --into`, no declared trees.** The smallest change, and
  perhaps enough: the pin stays in a script or makefile, as Updo has it
  today. Declared trees add project-file surface and a command; the case for
  them is a single place for pins, verification via hackage-security instead
  of a bare `curl`, and `cabal` knowing what is in the tree. This proposal
  asks for the first part firmly and the second for discussion.
- **Generalise to arbitrary archives or git trees** (a `type: archive` with
  a URL and hash, covering the `buck2/` submodule). Rejected: cabal would be a
  downloader for things it knows nothing about; git submodules and Nix-style
  fetchers exist for that, with better semantics for non-package trees.
- **Make `cabal get` write the version-less name by default.** Changes the
  behaviour every existing script relies on; an explicit `--into` is
  backwards compatible.
- **`stack unpack --to` style** (choose the parent only) is what exists and
  is the thing being fixed.

## Backwards Compatibility / Migration

- `--into` is a new option; `cabal get` is unchanged without it.
- `source-tree` is a new stanza; projects without it are unaffected. Older
  cabal versions warn about an unknown section, as they do for any new
  project-file syntax.
- Updo could replace its bootstrap rule with `cabal get updo==1.0.0 --into updo`
  (or a `source-tree`) while keeping the makefile rule as a fallback for
  users on older cabal.

## Interested parties

- Updo's author and users, who currently maintain the bootstrap makefiles.
- Users of other build systems (Buck2, Bazel's `rules_haskell`, Shake-based
  builds) that consume Hackage sources at fixed paths.
- The `cabal get` maintainers and the discussion on #8584, since `--into` and
  project-awareness are the two halves of making `get` fit for this.

## Implementation Notes

Part 1 is small: `Get.hs` already has `unpackPackage verbosity prefix pkgid
descOverride tarball`, which unpacks to `prefix </> prettyShow pkgid`; `--into`
needs a variant that unpacks to a given directory (extract to a temporary
sibling, then rename, to keep the "refuse if non-empty" check and atomicity).
`clonePackagesFromSourceRepo` takes the destination already. Documentation
and a testsuite case using `withRepo`.

Part 2 is a project-file stanza in both parsers (as `repository` is), a
command modelled on the `cabal vendor` prototype's structure, and a marker
file format. It should wait for the discussion on whether declared trees are
wanted at all.

The author is willing to implement part 1 alongside the `cabal vendor` work,
and part 2 if accepted.

## Open Questions

1. Should declared trees exist, or is `cabal get --into` (plus a script) the
   right size of solution?
2. Command name for materialising declared trees (`sync-sources`,
   `fetch-sources`, or folding it into `cabal update`).
3. Whether a `source-tree` for a package that is also a dependency should be
   allowed to pin a *different* version from the plan, or always follow it.
4. Marker file: name, format, and whether `cabal` should warn when a tree
   has been modified since it was written (cargo's `.cargo-checksum.json`
   does; Updo's users edit nothing in `updo/`, but the Buck2 case is a
   source tree someone might patch).
5. Whether `cabal vendor --unpack` should be defined as "a `source-tree`
   for a plan dependency", so the two features share the marker and the
   layout rules.

## References

- [up-do/updo](https://github.com/up-do/updo), "Downloading and Setup"
  (`docs/boot.qmd`): the bootstrap rule and the `cabal get`/`stack unpack`
  limitations it works around.
- [haskell/cabal#8584](https://github.com/haskell/cabal/issues/8584) —
  Extend `cabal get` to be affected by v2- projects.
- The `cabal vendor` proposal (`vendor-command-proposal.md`), "Two real
  cases from this repository" and Open Question 2.
- `project-buck2/README.md` and `project-buck2/fetch-inplace-deps.py` on the
  `add/haskell-buck2` branch of this repository.
- [cargo vendor](https://doc.rust-lang.org/cargo/commands/cargo-vendor.html)
  and [Go module vendoring](https://go.dev/ref/mod#vendoring), for the
  versioned vs unversioned directory conventions.
