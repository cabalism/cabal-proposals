# A `cabal vendor` command

## Summary

Add a project-aware `cabal vendor` command that copies the source of every
dependency in a project's build plan into a directory laid out as a
`file+noindex` package repository, so that the project can be built without
network access. Alongside it, let `file+noindex:` URLs in project files use
paths relative to the project root, so that the vendored repository can be
committed together with the project.

## Motivation

[haskell/cabal#10935](https://github.com/haskell/cabal/issues/10935) asks for
a `cabal vendor` command, prompted by the question in
[#10934](https://github.com/haskell/cabal/issues/10934) of how to build a
project with `cabal-install` on a machine that has GHC and cabal but no
network. That is the situation of distribution packagers (Fedora, ALT Linux
and others commented on the issue), of Nix and Guix style builders, and of
anyone building in a sandbox. Other ecosystems have a first-class answer:
`cargo vendor` and `go mod vendor` both materialise every dependency into a
`vendor/` directory that is then used instead of the network.

Cabal already has all the pieces, but no command that assembles them:

- A *local no-index repository* is a plain directory of
  `<package>-<version>.tar.gz` files, with optional `<package>-<version>.cabal`
  files next to them that act as revisions
  (`doc/config.rst`, "Local no-index repositories"). Nothing needs to be
  generated for cabal to use it; it builds the index from the directory.
- The elaborated install plan knows, for every dependency, where its source
  comes from and what the revised `.cabal` file from the index is (the same
  bytes whose hash `plan.json` reports as `pkg-cabal-sha256`).
- `cabal-install` can fetch and verify any repository tarball
  (`fetchRepoTarball`), and it already turns every
  `source-repository-package` checkout into a source distribution tarball
  before building it.

What people do today is scripting around `cabal get`, `cabal sdist` and a
hand-written `repository` stanza (for instance the
[`cabal-vendor`](https://github.com/lRespublica/cabal-vendor) script, or
[`nix-build-cabal-project`](https://sr.ht/~fgaz/nix-build-cabal-project/)),
and they run into gaps that the issue thread lists:

- `cabal fetch` is not project-aware
  ([#10977](https://github.com/haskell/cabal/issues/10977)), so it cannot
  fetch "what this project needs".
- `plan.json` did not record which revision of a `.cabal` file was used
  ([#6186](https://github.com/haskell/cabal/issues/6186), since fixed by
  [#10980](https://github.com/haskell/cabal/pull/10980)), so external tools
  could not reproduce the plan exactly.
- A `file+noindex:` URL cannot be relative
  ([#10934](https://github.com/haskell/cabal/issues/10934)): the path is used
  as written, relative to the current working directory, and cabal warns that
  this is fragile. A repository stanza with an absolute path cannot be
  committed to a project that is built on other machines.

## Proposed Change

### The command

```
cabal vendor [PACKAGES] [FLAGS]
```

`cabal vendor` is a nix-style command (registered as `v2-vendor`, with
`vendor` and `new-vendor` as aliases like every other v2 command). It runs
the same first phase as `cabal build`: establish the project, bring the
install plan up to date with the solver, using the project's current
configuration (`cabal.project`, `.local`, `.freeze`, `--constraint`,
`--index-state`, and so on). It then walks the plan and, for every package
that is not local to the project:

- **Package repository packages** (`RepoTarballPackage`): fetch the tarball if
  it is not already in the package cache — this goes through the normal
  hackage-security verification for secure repositories — and copy it to
  `<vendor-dir>/<package>-<version>.tar.gz`. If the plan carries a revised
  `.cabal` file for the package (the `elabPkgDescriptionOverride`), write it
  as `<vendor-dir>/<package>-<version>.cabal`. Because the revised file keeps
  its `x-revision` field, `plan.json` still reports the right `pkg-revision`
  when the project is later built from the vendored repository.
- **`source-repository-package` dependencies**: see the next section.
- **Everything else** — packages given as local or remote tarballs in
  `packages:` — is not vendored; the command warns about them, because their
  `packages:` entry would still be used and would still need the tarball.
- **Local packages** of the project are silently skipped: they are the
  project.

Build tool dependencies and setup dependencies are part of the plan and are
vendored like any other dependency.

The directory defaults to `vendor` in the project root; `-o`/`--output-directory`
changes it (as for `cabal sdist`). Every package in the plan is written,
overwriting a file of the same name; files for packages not in the plan are
left alone, so the directory accumulates rather than being regenerated (see
"Repeated runs" below), unless `--prune` is given, which removes the
`<package>-<version>.tar.gz` and `.cabal` files of packages that are not in
the plan. Pruning goes by the whole plan even when package names are given,
and only ever touches files named like package files: `preferred-versions`,
`noindex.cache` or anything else a user keeps in the directory is left alone.
The directory's `noindex.cache` is removed after writing so that cabal
rebuilds the index on next use (cabal never invalidates that cache by
itself). `--dry-run` lists what would be vendored (and removed) and writes
nothing.

Finally the command prints the configuration needed to use the directory:

```
$ cabal vendor
Vendored 12 packages into /home/me/proj/vendor

To build using the vendored packages, add the following to the project file
(the 'url' line must stay indented under 'repository'):

repository vendored
  url: file+noindex:vendor

active-repositories: vendored
```

The path is relative to the project root whenever the directory is inside
the project. With `active-repositories` naming only the vendored repository,
the solver sees exactly the vendored versions and no other repository is
read, so the project builds without network access.

When package names are given (`cabal vendor aeson text`), only those
dependencies are vendored, at the versions in the plan. This is for pinning a
few packages to exact sources while the rest keep coming from Hackage. The
printed stanza then keeps the other repositories active and makes the
vendored one the sole provider of the packages it holds:

```
active-repositories: :rest, vendored:override
```

Names that are not source dependencies of the project, or that name a local
package, are errors.

### Repeated runs

`cabal vendor` is a function of the current plan: each run writes the
packages of the plan it computes, and nothing else changes. That gives the
following behaviour for the ways it gets run more than once.

- **The same full run twice** is idempotent: the same files are written
  again with the same contents.
- **Two partial runs** (`cabal vendor aeson`, later `cabal vendor text`)
  accumulate; both print the `:rest, vendored:override` stanza, which is the
  right one as long as the directory does not hold the whole plan.
- **A partial run after a full run**, or a full run after partial ones, also
  accumulates. The stanza printed describes the run that was just made, not
  the directory: after `cabal vendor` has vendored everything, the directory
  can be used with `active-repositories: vendored` even though a later
  `cabal vendor aeson` prints the `:rest, ...:override` form; conversely a
  partial directory is not made complete by a partial run, whatever the
  stanza says.
- **Once the project uses the vendored repository** (`active-repositories:
  vendored` in the project file), the plan is solved from the vendored
  repository alone, so `cabal vendor` finds every package already in place
  and does nothing — it cannot pick up newer versions from Hackage, because
  it no longer sees Hackage. To refresh, run it with the other repositories
  re-enabled for that one invocation, for example

  ```
  cabal --active-repositories=:rest vendor
  ```

  after `cabal update`, or with `cabal vendor --index-state=...`. Newer
  versions are then added next to the old ones; the old files stay until
  removed by hand or with `--prune`, and with several versions in the
  directory the solver picks among them as it would in any repository, so a
  `cabal.project.freeze` remains the way to pin exact versions.
- **`--prune` is relative to one plan.** A directory that serves several
  configurations of the project (with and without tests, two compilers) is
  pruned to whichever configuration `cabal vendor --prune` runs with; the
  other configuration's packages have to be vendored again. This is the
  same trade-off as a regenerating `cargo vendor`, made explicit by the
  flag rather than the default. Once the project builds from the vendored
  repository, the plan is solved from the directory itself, so `--prune`
  is a no-op except for versions the solver no longer chooses (typically
  older ones superseded by a refresh) — the prototype confirms this on a
  250-package directory.
- **A `source-repository-package` dependency vendored earlier**, whose stanza
  has since been removed, now comes from the vendored repository itself and
  is left untouched; if the stanza is still there, the source distribution is
  written again from the current checkout.
- A file that is its own source (the plan took the package from the vendor
  directory) is never copied over itself.
- **Several `.cabal` files in one directory are not a problem.** A local
  package directory in `packages:` must contain exactly one `.cabal` file,
  but the vendor directory is never read that way. A `file+noindex`
  repository is read by listing the directory, taking every
  `<package>-<version>.tar.gz` as an entry and looking up the file with the
  exact name `<package>-<version>.cabal` for it; unmatched `.cabal` files are
  ignored. So each package id has at most one sidecar, which every run
  overwrites in place, and different versions of a package are different
  entries with different file names. Repeated runs cannot produce a
  "multiple `.cabal` files" conflict.

### Dependencies from git and other version control

Today a `source-repository-package` stanza makes cabal clone the repository
into `dist-newstyle/src/<mangled-location>-<hash>/`, pick the package in
`subdir:`, and turn it into a source distribution tarball next to the
checkout. That tarball, not the checkout, is what gets built: the package is
a non-local dependency installed into the store, exactly like a Hackage
package. With `--offline`, the sync is skipped, so a fresh clone of the
project cannot be built at all.

`cabal vendor` copies that source distribution into
`<vendor-dir>/<package>-<version>.tar.gz`. No `.cabal` sidecar is needed since
the tarball's own `.cabal` file is authoritative. The command lists the
stanzas that can then be removed:

```
The vendored repository now provides these source-repository-package
dependencies, whose stanzas can be removed from the project file:
  - bar-1.0 (git https://github.com/foo/bar, tag 1a2b3c)
```

After removing the stanza, the dependency comes from the vendored repository
like every other package, keeps its non-local status (store-built, not a
target of `cabal build all`), and the project no longer needs `git` at all.
This works uniformly for every VCS type cabal supports (git, darcs, mercurial,
subversion) because it operates on the source distribution, not on the
checkout; the output of `post-checkout-command` is included since the
distribution is made after it runs.

Note that what is vendored is the *package* as it would be built (the files
a source distribution contains), not the repository: `.git`, sibling packages
in a multi-package repository and files outside the package are excluded, in
the same way as `cargo vendor` excludes files not part of the crate.

### Relative `file+noindex:` paths

A `repository` stanza in `cabal.project`, or in a file it imports, may use a
relative path in its `file+noindex:` URL:

```
repository vendored
  url: file+noindex:vendor
```

The path is resolved against the project root when the project file is read,
so a vendored repository can be committed with the project and used from any
working directory. (Note the single colon: `file+noindex://vendor` would
parse `vendor` as a URI authority.) Repository stanzas in the global
`~/.config/cabal/config` are not affected; a relative path there keeps its
current, warned-about, behaviour.

### Working on a vendored package locally

A common reason to want a dependency's source at hand is to try a change in
it: a fix, a debug print, a different default. The usual tool for that is a
git submodule (or a `source-repository-package` pointing at a fork), which
gives the full history and a way to push, at the price of needing the
upstream repository, the network and git at build time. The vendored
directory offers a lighter alternative: a snapshot of exactly the source that
the plan builds, with its revised `.cabal` file, which can be turned into a
local package, edited in place, and later turned back. Nothing can be pushed
upstream from it — a change that should go upstream is sent as a patch by
hand — which is the same trade-off `cargo vendor` makes with its `[patch]`
sections.

This works today with existing commands, without changes to `cabal vendor`:

1. Unpack the package from the vendored repository. `cabal get` applies the
   sidecar `.cabal` file as it does for an index revision, so the result is
   what the plan builds:

   ```
   cabal --local-no-index-repo=vendored:$PWD/vendor --active-repositories=vendored \
     get assoc-1.1.1 --destdir=patched
   ```

   (`cabal get` is not project-aware, so the repository is passed on the
   command line. The same command without the two global flags unpacks from
   Hackage directly, for those who want to go to a local copy without
   vendoring first.)

2. List the directory in the project file:

   ```
   packages: ., patched/assoc-1.1.1
   ```

   A `packages:` entry shadows every repository version of that package name,
   so the vendored tarball is ignored for as long as the line is there. The
   package is now local: built in place, subject to local package
   configuration, part of `cabal build all` — which is what one wants while
   working on it.

3. Edit, build, test. To go back, remove the `packages:` line (and the
   directory); the plan returns to the vendored tarball. A `cabal vendor`
   run in the meantime is unaffected, since local packages are never
   vendored.

The same steps apply to a vendored `source-repository-package` dependency,
whose tarball is the source distribution of its checkout.

**Getting a patch back out.** Because the pristine source stays in the vendor
directory, a patch for upstream is the difference between a fresh unpack and
the edited directory:

```
cabal --local-no-index-repo=vendored:$PWD/vendor --active-repositories=vendored \
  get assoc-1.1.1 --destdir=pristine
git diff --no-index pristine/assoc-1.1.1 patched/assoc-1.1.1 > assoc.patch
```

Simpler still is to `git init` the unpacked directory and commit it as
vendored before editing; `git diff` and `git format-patch` then produce the
patch at any time, and the edits have history without any upstream
involvement. Two things to know when sending such a patch upstream: the
baseline must be the *revised* package (as `cabal get` produces it), not the
bare tarball, or the diff also contains Hackage's revision to the `.cabal`
file — `x-revision:` and changed bounds — which is not the author's; and a
source distribution is not the repository (files not shipped in the
distribution are absent, and a package from a multi-package repository sits
at a subdirectory there), so the patch may need `git apply --directory=`
or a different `-p` level. The `source-repository head` field of the `.cabal`
file says where upstream is.

The prototype makes this a one-liner: `cabal vendor --unpack PACKAGES`
vendors the named packages as usual and additionally unpacks each into
`vendor/src/<package>-<version>/`, with the revised `.cabal` file applied
(the same step `cabal get` performs), then prints the `packages:` entry to
add:

```
$ cabal vendor --unpack assoc
...
Unpacked into vendor/src. To work on these as local packages,
which take precedence over the vendored ones, add to the project file:

packages:
  vendor/src/assoc-1.1.1
```

Package names are required (`--unpack` alone is an error: unpacking every
dependency would turn them all into local packages). A package directory
that already exists is left alone with a warning, since it may hold edits;
the tarball and sidecar next to it are still refreshed. The `src/`
subdirectory is ignored by the no-index reader, so it does not disturb the
repository it sits in, and `--prune` never touches it. Version control of
the unpacked copy is left to the user; the `git init` recipe above is the
suggested one. A project-aware `cabal get` (#8584) would give the same
unpack step without `--local-no-index-repo`, and is a separate change.

### Two real cases from this repository, and where vendoring stops

While prototyping, two arrangements in the cabal repository itself were
examined as candidates for `cabal vendor`. They are instructive because one
is in scope and one is not, and the line between them is the line around
this proposal.

**Buck2 builds of this repository** (`project-buck2/`, on a branch). Buck2
builds the packages in the repository against Hackage dependencies that
cabal has already built into its store; a generator turns the store into
`haskell_prebuilt_library` targets. One Hackage dependency,
`hackage-security`, cannot be a prebuilt library: it depends on `Cabal-syntax`
from this repository, so cabal builds it in-place and there is nothing in the
store. A script (`fetch-inplace-deps.py`) copies its source from cabal's
private `dist-newstyle/src/hackage-security-<version>/` into
`project-buck2/vendor/hackage-security/src/`, where a hand-written `BUCK` file
builds it. Two things are notable:

- The script does what `cabal vendor --unpack hackage-security` does — the
  package is in the plan, the source is the revised source distribution —
  except that it also depends on cabal's internal directory layout, which is
  exactly what a command should spare it. This case is in scope.
- The consumer needs the tree at a path *it* chooses, without a version in
  the name, because the `BUCK` file globs `src/**/*.hs` next to itself and
  Buck2 rules can only see their own directory. `--unpack` today writes
  `vendor/src/<package>-<version>/`, so the build file would have to know the
  version. That is a layout question for `--unpack` (see Open Question 2),
  not a reason to leave the script.

The same branch also carries a git submodule, `buck2/`, for the Buck2 rules
themselves (a fork of `haskell-buck2`: Starlark, toolchains, a Python
generator). It contains no `.cabal` file and is never built by cabal; it is
build tooling. `cabal vendor` has nothing to say about it, and should not: a
submodule, or a pinned archive fetched by a script, is the right tool for a
tree that is not a package. This case is out of scope, and the temptation to
grow the command into a general fetcher is worth resisting explicitly.

**Updo** ([up-do/updo](https://github.com/up-do/updo)) is a Hackage package
that projects use as a make-and-Dhall toolkit to generate `cabal.project`
and `stack.yaml`. It is never compiled by the consuming project; its
makefiles are `include`d and its scripts run in place. It bootstraps itself
with one make rule, kept in the consuming project's `project-bootstrap.mk`:
fetch an archive (the Hackage source distribution of a pinned version, or a
GitHub archive of a pinned commit), unpack it, and rename the result to
`updo/`. Its documentation explains why cabal was not used: "`cabal get`
doesn't support unpacking to a specific directory that does not match the
package name with a version suffix", and `--destdir` only chooses the parent.
So updo wants, for a package that is not in any build plan, the same thing
the Buck2 case wants for a package that is: a package's source, pinned,
unpacked at a chosen path. `cabal vendor` cannot serve it, because the
command is a function of the build plan and updo is not in the plan. What
cabal could offer instead — a destination for `cabal get`, and possibly a
declared "source tree" in the project file — is a separate question, taken
up in the companion draft "Package sources at a chosen path"
(`source-trees-proposal.md`).

The two cases together give a second, independent data point on the layout
of unpacked sources: both a Buck2 build file and a Makefile want an
unversioned path they chose. That is recorded under Open Question 2.

### Everything is already there

Two existing functions define the on-disk contract, so the command adds no
new file format:

- `FetchUtils.packageFile` for a `RepoLocalNoIndex` repository is
  `<dir>/<package>-<version>.tar.gz`, and
- `IndexUtils.withIndexEntries` reads `<package>-<version>.cabal` next to it
  as the revision override.

Getting those two names right is all it takes for cabal to index the
directory.

## Alternatives Considered

The choices below were each weighed for the first version. Where an
alternative is attractive as a later extension, it is called out as such.

1. **How the project picks up the vendored repository.**
   - *Print the stanza for the user to add* (chosen). This is what
     `cargo vendor` does. It changes nothing about how project files are
     found and read, keeps the user in control of `cabal.project`, and works
     with imports and conditionals.
   - *A new auto-loaded `cabal.project.vendor` file*, written by the command
     and read like `cabal.project.freeze`. Zero friction (`go mod vendor`
     style), but it adds a third implicit project file to every command's
     configuration loading and change monitoring, and to the documentation of
     what a project consists of. Could be added later if the manual step
     proves to be a nuisance.
   - *Editing `cabal.project` in place.* Rejected: cabal does not rewrite
     hand-written project files, and it cannot do so safely in the presence
     of imports and conditionals.
   - *A `--vendor-dir` flag on `build`* that injects the repository.
     Rejected: the configuration would live outside the project file, which
     is the opposite of what committed vendoring is for.

2. **Relative paths.**
   - *Resolve against the project root* (chosen). Matches how `packages:`,
     `builddir` and imports are resolved.
   - *Absolute paths only*, printing the absolute path in the stanza. Rejected:
     the stanza could not be committed.
   - *Also resolve paths in the global config against the config file's
     directory.* Not done here; a natural small follow-up.

3. **`source-repository-package` dependencies.**
   - *Copy the source distribution into the no-index repository* (chosen).
     One file copy, no change to how the package is built, works for every
     VCS.
   - *Unpack into `vendor/<package>-<version>/` and point `packages:` at it.*
     The checkout would be browsable and patchable in place, but the
     dependency would become a local package: it is then built in place
     rather than in the store, local package configuration (`ghc-options`,
     `-Wall` and friends in `package *`) applies to it, and `cabal build all`
     and `cabal test all` include it. That is a change in behaviour the user
     did not ask for — unless they want to work on the package, for which
     see "Working on a vendored package locally"; it is the right thing to
     do on request, not by default.
   - *Skip them with a warning.* Rejected: they are exactly the dependencies
     that most often need the network at build time.
   - *A first-class vendored location for checkouts* — see the follow-up
     below.

4. **Remote and local tarballs in `packages:`.** Skipped with a warning
   (chosen). Copying them into the directory would not stop the `packages:`
   entry from being used.

5. **Output form.** A directory (chosen), rather than a tarball of the
   directory (the `cabal-vendor` script offers `--output-mode tarball`) or a
   full `01-index.tar` secure repository. A directory is what cabal reads
   natively and what version control handles well; wrapping it is a one-liner
   for anyone who needs an archive.

6. **Pruning.** Files for packages not in the plan are left alone by
   default, and removed on request with `--prune` (chosen). `cargo vendor`
   always regenerates the directory from scratch. Leaving files lets several
   configurations (or several `cabal vendor` runs with different flags or
   package names) accumulate in one repository, and avoids deleting anything
   the user put there; `--prune` covers the case where stale versions linger
   after a refresh. Pruning against the packages named on the command line
   rather than the whole plan was rejected: it would delete legitimately
   vendored dependencies on every partial run.

7. **Reproducibility.** With only the vendored repository active there is one
   version of every package, so no freeze file is needed. Also writing
   `cabal.project.freeze`, or a `preferred-versions` file into the directory,
   was considered and not done: users who want a freeze file can run
   `cabal freeze`, and the vendored repository is not a substitute for it when
   other repositories are also active.

8. **Which packages.** The whole plan by default, with optional package names
   (chosen). `cargo vendor` and `go mod vendor` only do whole-plan vendoring.
   Names-plus-dependency-closure (as `cabal fetch --dependencies`) is not
   offered: with `:rest` active the closure is not needed to build, and the
   flag can be added later without changing the command line.

9. **Relation to `cabal fetch` and `build --only-download`.** `cabal vendor`
   is in effect the project-aware `fetch` of #10977 with a destination. A
   later v2 `fetch` could share its plan traversal. `build --only-download`
   fetches into the shared package cache, which is not a repository and not
   something to commit.

## Backwards Compatibility / Migration

- The command is new; nothing changes for projects that do not use it.
- A relative `file+noindex:` path in a project file changes meaning: it was
  resolved against the current working directory (and warned about), it is
  now resolved against the project root. The two coincide when cabal is run
  from the project root, which is the only case in which the old behaviour
  worked reliably.
- The vendored directory is the existing no-index repository format; any
  cabal that supports `file+noindex` (3.4 and later) can build from it, only
  the relative path needs a cabal with this change.

## Interested parties

- The commenters on #10935: distribution packagers (Fedora, ALT Linux) and
  the author of `nix-build-cabal-project`.
- Authors of the `cabal-vendor` script and similar tooling, who could
  replace it.
- Maintainers of Nix/Guix Haskell infrastructure, who need exactly the
  "sources of the plan, with revisions" that the vendored directory contains.
- Nobody has been contacted yet; this proposal is the first step.

## Implementation Notes

An implementation exists (`cabal-install/src/Distribution/Client/CmdVendor.hs`,
about 350 lines, plus a one-function change in `ProjectConfig.hs` for
relative paths) with documentation, changelog entries and three
`cabal-testsuite` tests: whole-plan vendoring followed by a build with the
original repository deleted, partial vendoring with `:rest, vendored:override`,
and vendoring a `source-repository-package` from a local git repository
followed by a build with the repository deleted. The author is willing to
see it through review.

One lesson from the prototype worth recording: the stanza the command
prints is indentation-sensitive (`url:` must sit under `repository`), and
cabal's ordinary `notice` re-wraps messages to the terminal width and drops
leading whitespace, so the first version printed a stanza that did not
parse when pasted. The output now uses the unwrapped variant, and the
message says explicitly that the `url` line must stay indented. The
testsuite did not catch this because it runs cabal with `+nowrap`; a
reviewer looking at the recorded output would not have seen it either.

A small related fix is included: `cabal build --offline` used to refuse to
build any repository package that was not yet in the store, even when it
came from a `file+noindex` repository where nothing is downloaded
(`ProjectBuilding.hs`, `packagesToDownload`). No-index repositories are now
treated as local there, so `--offline` works with a vendored repository as
one would expect. The existing `OfflineFlag` test relied on the old
behaviour and now uses a real remote repository (`withRemoteRepo`) to test
what it always meant to test: refusal to download.

## Open Questions

1. **Browsable checkouts for VCS dependencies.** The checkout of a
   `source-repository-package` lives at
   `dist-newstyle/src/<mangled-location>-<hash>/`, which is hard to find and
   is lost with `cabal clean`. A follow-up could add a project-level field,
   for example `vendor-source-repositories: vendor/src`, under which cabal
   keeps each checkout's package as a browsable directory
   `vendor/src/<package>-<version>/` (the source-distribution-filtered tree).
   When the directory exists and matches the stanza's `tag`/`commit` (a
   marker file, in the spirit of the existing `<pathStem>.cache` monitor),
   the build reads it instead of cloning, so the stanza can stay in
   `cabal.project` and the directory can be committed; when it is absent,
   cabal clones as today and `cabal vendor` populates it. This gives
   `go`-style "use `vendor/` when present" for VCS dependencies without
   changing how repository packages work. Whether to spell it per stanza
   (`vendored-in: vendor/bar`) or per project, and what the field is called,
   is open.
2. **`--unpack` and `vendor/src/`.** The prototype implements
   `cabal vendor --unpack PACKAGES` (see "Working on a vendored package
   locally") and deliberately does not initialise the unpacked directory as
   a git repository: a nested repository inside the project's working tree
   is a surprise, and the recipe is one line for those who want it. Open:
   - the **name of the unpacked directory**. Two independent consumers (the
     Buck2 build files and updo's makefiles, see "Two real cases") want an
     unversioned path of their choosing, not `vendor/src/<package>-<version>/`.
     Options: an unversioned default (`vendor/src/<package>/`, as Go lays out
     `vendor/` by module path; cargo keeps the version), an explicit
     destination per package (`--unpack aeson=third-party/aeson`), or both.
     A versioned name is safer for the local-hacking workflow (two versions
     can coexist while switching); an unversioned one is what build files
     want to reference. This is the question the prototype most wants an
     answer to;
   - whether the same `vendor/src/` location should also hold the browsable
     VCS checkouts of question 1, so that "the source of dependency X, in the
     tree" means one thing;
   - whether a project-aware `cabal get` (#8584) should subsume the unpack
     step.
3. Should vendored source distributions of VCS dependencies record their
   provenance (location, commit) in a small manifest, so that a later
   `cabal vendor` can tell when a stanza moved on?
4. Should `cabal vendor` verify tarballs already in the directory against
   `pkg-src-sha256` from the plan, or is leaving files alone enough?
5. The default names: `vendor` for the directory and `vendored` for the
   repository in the printed stanza.
6. Windows: the relative form is `file+noindex:vendor`; absolute paths keep
   the documented `file+noindex:C:/...` form.
7. Whether the no-index cache (`noindex.cache`) should be invalidated by
   cabal when the directory changes, which would make the "remove the cache"
   step unnecessary for everyone, not only for `cabal vendor`.

## References

- [haskell/cabal#10935](https://github.com/haskell/cabal/issues/10935) — Add a `cabal vendor` command
- [haskell/cabal#10934](https://github.com/haskell/cabal/issues/10934) — Proper way to vendor dependencies?
- [haskell/cabal#10977](https://github.com/haskell/cabal/issues/10977) — Implement v2-fetch (project support)
- [haskell/cabal#6186](https://github.com/haskell/cabal/issues/6186) / [#10980](https://github.com/haskell/cabal/pull/10980) — revision information in `plan.json`
- [`cabal-vendor`](https://github.com/lRespublica/cabal-vendor) — a shell script doing this today
- [`nix-build-cabal-project`](https://sr.ht/~fgaz/nix-build-cabal-project/)
- [cargo vendor](https://doc.rust-lang.org/cargo/commands/cargo-vendor.html)
- [Go module vendoring](https://go.dev/ref/mod#vendoring)
- Cabal User Guide: [Local no-index repositories](https://cabal.readthedocs.io/en/latest/config.html#local-no-index-repositories),
  [`active-repositories`](https://cabal.readthedocs.io/en/latest/cabal-project-description-file.html#cfg-field-active-repositories)
