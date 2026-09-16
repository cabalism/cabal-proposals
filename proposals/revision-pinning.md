# Pin package revisions in project configuration

Draft for a pull request to
[`haskell/cabal-proposals`](https://github.com/haskell/cabal-proposals) as
`proposals/revision-pinning.md`.

## Summary

Let project configuration select a specific `.cabal` file revision of a
package version, in the notation Stack uses (`pkg-1.2.3@rev:N`,
`pkg-1.2.3@sha256:HASH`), and have `cabal freeze` record the revision of
every package in the plan. Pins go in `constraints:` (and so `--constraint`)
and in a new `revisions:` project field. Nothing changes in `.cabal` files or
in the solver.

## Motivation

A Hackage revision can change a build plan after the fact, and the only
protection today is `index-state`, which freezes the entire index:

- A revision that tightens bounds breaks a previously working plan. See the
  production incident in #7833 and the `QuickCheck` revision that broke CI in
  #7277.
- `index-state` is all-or-nothing. It cannot express "take new packages, but
  keep this one known-good revision", and there is in general no index state
  that selects exactly a chosen set of revisions.
- A freeze file does not protect against revisions at all (#7277). The
  suggested workaround, `allow-newer: all` on top of the freeze file, covers
  bound tightening but not a revision that adds a dependency or changes
  anything else in the `.cabal` file.
- The information already exists. `cabal-install` keeps every revision in its
  index cache, and `plan.json` reports `pkg-revision` and `pkg-cabal-sha256`
  for each package (#5695). There is just no way to feed a choice back in.
  Stack has offered `@rev:N` and `@sha256:HASH` for years.

The discussion on #7833 settled that pins do not belong in `build-depends`,
because a revision is repository metadata rather than a property of the
package, but that pinning in project configuration is acceptable given care
(@gbaz, @andreabedini) and that the Stack notation is a good choice
(@Mikolaj). This proposal is that design, and asks for a decision on the one
part that changes existing behaviour: what `cabal freeze` writes.

## Proposed Change

**Notation.** A revision pin is appended to a package version:

- `@rev:N` selects the revision whose `x-revision` field is `N`, the original
  upload being revision 0. This is the number `plan.json` reports and the
  number shown on the package's revisions page on Hackage.
- `@sha256:HASH` selects the revision whose `.cabal` file text has that
  SHA-256 hash, as `plan.json` reports in `pkg-cabal-sha256`.

**In a constraint.** `constraints: foo ==1.2.3@rev:2` fixes the version and
pins the revision. The pin is accepted only with an exact version (`==`) and
only in the unqualified or `any.` scope, since a revision applies to the
package version wherever it is used. `--constraint="foo ==1.2.3@rev:2"` is
the command line form.

**In a new `revisions:` project field.** `revisions: foo-1.2.3@rev:2` pins
the revision without constraining the version. The pin only takes effect
when the solver picks that package version; otherwise it has no effect and
`cabal` warns that the pin is probably stale. There is deliberately no
`--revisions` command line flag, because `--constraint` already provides the
command line form.

**In `cabal freeze`.** The freeze file records the revision of every package
in the plan whose `.cabal` file is a revision, as `any.foo ==1.2.3@rev:N` in
its version constraint. A package that is at several versions in the plan
gets its pins in a `revisions:` field instead. A freeze file therefore fixes
the `.cabal` files of the plan, not just the versions.

**Semantics.**

- No solver change. A pin is applied when the package index is read, after
  `index-state` filtering, by selecting a different entry for that package
  version. The constraint form desugars to `==` for the solver plus a pin. The
  index cache format is unchanged.
- The pinned `.cabal` file is what everything else sees, so for example
  `allow-newer` relaxes the bounds of the pinned revision, and `plan.json`
  reports the pinned revision.
- It is an error, not a warning, when the package version exists in a
  repository but none of its revisions match the pin (the message lists the
  available `rev:N (sha256:…)` pairs), and when the same package version is
  pinned to two different revisions. Both messages name where each pin came
  from: the field or constraint, and the project file, command line or freeze
  file.

**Out of scope.** Pins in `build-depends`. `cabal get`, which does not read
project configuration and so ignores pins. The `v1-` commands accept the
constraint syntax but apply only the version part.

## Alternatives Considered

- **Pins in `build-depends`** (the original request in #7833). Rejected in
  the thread: a package would then only build against one `.cabal` file of
  its dependency, defeating the purpose of revisions.
- **`index-state` alone.** Cannot select revisions, only freeze the whole
  index, and a bad revision blocks every later index state.
- **Freeze plus `allow-newer: all`.** Only covers bound tightening.
- **A solver-aware design** where each revision is a candidate the solver
  chooses between. Far larger change, and would alter solver results for
  projects that pin nothing.
- **Other spellings**, `1.2.3-5`, `1.2.3-r5`, `1.2.3 (r5)`, discussed on
  #1929 and #7833. The Stack notation is already known to users, has a
  by-hash form, and `plan.json` already reports both identifiers.
- **A `--revisions` flag.** Redundant with `--constraint`, and a second
  spelling of the same thing on the command line.
- **Pins only in the `revisions:` field, never in constraints.** Freeze
  files would then split each pinned package across two fields, and
  "exactly this version" and "exactly this `.cabal` file" belong together.

## Backwards Compatibility / Migration

- Projects that pin nothing are unaffected. The index cache format is
  unchanged, and `plan.json` gains no fields.
- A freeze file written by a `cabal-install` with this change may contain
  `==1.2.3@rev:N`, which older releases reject as a parse error in
  `constraints`. This is the only incompatible change. There is precedent:
  3.4 started writing `active-repositories` and per-repository `index-state`
  into freeze files, which earlier releases could not read either. Migration
  is to delete the `@rev:N` suffixes or regenerate the file with the older
  release. If the developers prefer, `cabal freeze` could take a flag to
  leave revisions out, or pin only when asked.
- Stack's by-hash form carries a size suffix, `@sha256:HASH,SIZE`. This
  proposal accepts the hash alone, as `plan.json` reports it, and the
  suffix would have to be dropped when copying from a Stack file.

## Interested parties

- The participants on #7833: those who asked for the feature (@hasufell,
  @tfausak, @Kleidukos) and those who shaped the design in the thread
  (@Mikolaj, @gbaz, @andreabedini). Not yet contacted beyond the thread.
- Stack and pantry maintainers, for the shared notation.
- Hackage trustees, since pins give users a way to opt out of one revision
  without freezing the index.
- Tools that parse project or freeze files or `plan.json`, such as
  haskell.nix, which will meet the new constraint syntax and field.

## Implementation Notes

I am the implementer. The change is complete on a branch and will be opened
as a pull request against #7833 alongside this proposal:

- new `Distribution.Client.Types.PackageRevision` module, pin handling in
  `IndexUtils`, `Targets`, `ProjectConfig` and `CmdFreeze`;
- two new error codes, `Cabal-7169` (no matching revision) and `Cabal-7170`
  (conflicting pins);
- documentation for `constraints`, the new `revisions` field and `freeze`;
- unit tests for parsing and printing, a parser-tests fixture, and
  cabal-testsuite packages against a `file+noindex` repository and a secure
  repository with a real second revision.

Target release: `cabal-install` 3.20.

## Open Questions

1. Should `cabal freeze` pin revisions by default, or only with a flag? The
   proposal says by default, since a freeze file that does not fix the
   `.cabal` files does not reproduce the plan.
2. Should the by-hash form also accept Stack's `,SIZE` suffix, for
   copy-paste compatibility?
3. Is `revisions` the right field name, or should it say what it does more
   directly (`pinned-revisions`)?
4. When a pinned package version is carried by two active repositories, the
   pin must match in each. Is that the right rule, or should the first
   repository that has the version win?

## References

- #7833 Adding support for revision pinning in dependency version
- #7277 `cabal freeze` should freeze either the index or the revisions
- #5695 Report revisions in `plan.json`
- #6186, #1929, #2222 earlier discussions of revisions and pinning
- commercialhaskell/stack#2217 and the pantry documentation of
  `pkg-1.2.3@rev:N` / `@sha256:HASH,SIZE`
- haskell/ecosystem-proposals#6 Uncurated Hackage Layer
