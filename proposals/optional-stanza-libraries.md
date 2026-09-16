# Libraries in optional stanzas

## Summary

A library component may declare that it belongs to one or more optional stanzas:

```cabal
library testlib
  visibility: private
  stanza: test, bench
```

Such a library is *requested* exactly when one of those stanzas is, in the same
way a test-suite is requested only under `--enable-tests`. Its dependencies
become conditional on that stanza rather than unconditional, which lets a package
share helper modules between its test-suites without those helpers'
dependencies being mandatory in every build.

Conditional is not the same as absent: a build that has not disabled the stanza
still prefers to enable it, and will solve those dependencies if it can. What
changes is that it can now decline. See
[What "not solved for" does and does not mean](#what-not-solved-for-does-and-does-not-mean).

A library naming no stanzas -- the default, and what omitting the field means --
is always requested, exactly as today.

## Motivation

A package whose test-suites share helper modules has nowhere good to put them.

`cabal-install` has four test-suites sharing `hs-source-dirs: tests`. Four modules
are listed in more than one stanza, so each is compiled once per suite that uses
it: `UnitTests.Options` and `UnitTests.Distribution.Solver.Modular.DSL` three
times each, `...DSL.TestCaseUtils` and `...Client.ArbitraryInstances` twice.

The two obvious remedies both fail.

### A sublibrary drags test-only dependencies into every solve

Moving the shared modules into a sublibrary of `cabal-install` is the natural fit:
the component graph stays acyclic, since the sublibrary depends on
`lib:cabal-install` and the test-suites depend on the sublibrary.

But `tests: False` disables *test-suite components*, and a sublibrary is a library
component. Its `build-depends` are therefore resolved unconditionally. In
`cabal-install`'s case the helpers need `Cabal-QuickCheck`, which is an
unpublished in-repo package that `cabal.bootstrap.project` deliberately excludes,
so every bootstrap solve fails:

```
Error: [Cabal-7107]
Could not resolve dependencies:
[__0] trying: cabal-install-3.19.0.0 (user goal)
[__1] unknown package: Cabal-QuickCheck (dependency of cabal-install)
```

Even where the dependencies do resolve, they are wrong to require: `tasty` and
friends enter a plan for a build that asked for no tests.

### A separate package is a package-level cycle

Moving the helpers into their own package removes them from the `tests: False`
solve, because only test-suites would depend on that package. But the helpers
need `lib:cabal-install`, so the package graph becomes
`cabal-install:test -> cabal-install-testlib -> cabal-install:lib`. The component
graph is still acyclic, but cabal resolves cycles at package granularity:

```
[_41] rejecting: cabal-install:*test (cyclic dependencies; conflict set: cabal-install, cabal-install-testlib)
```

### A common stanza solves the solver, and reintroduces the duplication

A `common` stanza carrying the shared sources and their dependencies, imported
only by the test-suites, does keep those dependencies out of the solve. Verified:
with an unsatisfiable dependency in the common stanza, `--disable-tests` resolves
cleanly and builds only the library.

It works because a common stanza is textual: its `build-depends` end up inside
each importing component, so if only test-suites import it, the dependencies
exist only in test-suite stanzas and vanish with them.

The cost is the problem this proposal started from. There is no library artifact,
so each importing test-suite compiles the shared modules again -- measured on a
two-suite package, the shared module is compiled once per suite. That is exactly
`cabal-install`'s present situation, where four test-suites share
`hs-source-dirs: tests` and four modules are compiled two or three times each.

Two further limits: without a compiled artifact there is nothing to anchor
instances, so shared orphan instances become a hazard; and common stanzas are
package-local, so this cannot serve sharing between test-suites in *different*
packages, which is the case [#10900](https://github.com/haskell/cabal/issues/10900)
is chiefly about.

So the choice without this feature is between resolving test-only dependencies
in every build and recompiling test helpers once per suite. The point of the
field is to stop having to choose.

### The workaround, and why it is not good enough

A manual flag guarding `buildable` does work:

```cabal
flag test-helpers
  default: False
  manual: True

library testlib
  if !flag(test-helpers)
    buildable: False
```

`buildable: False` removes the component's dependencies from the solve, so
bootstrap resolves again. But the flag is a hand-mirror of `--enable-tests` that
nothing keeps in sync: every project that builds the test-suites must set
`+test-helpers`, and a plain `cabal build` outside those projects fails with
`Dependency on unbuildable package cabal-install`, which does not hint at the
cause.

## Proposed Change

Let a sublibrary declare that it belongs to an optional stanza:

```cabal
library testlib
  visibility: private
  stanza: test
  hs-source-dirs: testlib
  build-depends: cabal-install, Cabal-QuickCheck, tasty, tasty-hunit
  exposed-modules: UnitTests.Options, ...
```

A library with `stanza: test` is *requested* exactly when test-suites are, and one
with `stanza: bench` exactly when benchmarks are. `stanza: always` is the default
and the existing behaviour. Only sublibraries may carry the field; the main
library is always requested.

This needs no new concept in the solver, which already models the optional
stanzas as decision variables rather than inputs.

A new field on library components, spelled here as `stanza:` but see
[Naming](#naming) -- `requested-by:` is the recommended spelling:

```
stanza: test, bench
```

a comma-separated set, empty by default.

- Default empty, which is the current behaviour: the library is always
  requested.
- Accepted only on sublibraries. On the main library it is an unrecognised
  field, warned about and ignored -- the same treatment `visibility:` already
  gets there, so this is consistent rather than special.
- Requires `cabal-version: 3.20` or later.
- A library is requested iff any stanza it names is requested. Being *requested* is necessary but not sufficient for being
  *enabled*; `buildable: False` still applies, per the existing distinction in
  `Distribution.Types.ComponentRequestedSpec`.

### What "not solved for" does and does not mean

The dependencies of a library in an optional stanza are guarded by that stanza,
not removed from the description, so how much they affect a solve depends on how
the stanza was decided. There are three cases, and only the first is absolute.

**Explicitly disabled** (`tests: False`, `--disable-tests`). `ProjectPlanning`
adds a constraint ruling the stanza out, so the library's dependencies are never
required. This is the case `cabal.bootstrap.project` is in, and the reason a
bootstrap plan is unaffected by test-only dependencies.

**Left at the default.** They are *still used*. For every local package cabal
adds a stanza preference for every stanza the user has not disabled --

```haskell
. addPreferences
  -- enable stanza preference unilaterally, regardless if the user asked
  -- accordingly or expressed no preference, to help hint the solver
```

-- and `preferPackageStanzaPreferences` in the solver reorders the stanza choice
so that enabling it is tried first. The solver therefore attempts to satisfy a
stanza-scoped library's dependencies by default, and if it can, they are part of
the plan.

What changes is that the choice is now a choice. Before, a sublibrary's
dependencies were unconditional and a solve that could not satisfy them simply
failed; that is what made `Cabal-QuickCheck` fatal to the bootstrap plan. Now the
stanza node is weak, so the solver backtracks and produces a plan without the
stanza rather than failing. Verified: a stanza library depending on a package
that does not exist leaves `cabal build` at default settings working, where the
same dependency on an ordinary sublibrary fails the solve.

**Explicitly enabled** (`--enable-tests`). The stanza is constrained on and the
dependencies are required, so an unsatisfiable one is a failure again.

Two consequences worth being plain about. Someone who merely wants to avoid
downloading a heavy test-only dependency will not get that from the default; they
need `--disable-tests`. And the preference is added for *local* packages only, so
the stanzas of a dependency taken from Hackage are never preferred -- which is
the mechanical reason a stanza-scoped library cannot usefully be `public`, as
discussed under
[Cross-package sharing](#cross-package-sharing-is-the-unresolved-conflict).

### Validation

A component in an optional stanza may only be depended upon by components in the
same stanza, or by other components of the same package that are themselves in
that stanza. In particular the main library may not depend on a `stanza: test`
sublibrary.

Without this rule a package could reintroduce the original problem by having
`lib` depend on a test-scoped sublibrary, and the solver would be left with a
dependency on a component that is not requested. With it, the dependencies of a
stanza-scoped library are needed only when something in that stanza is, which is
what makes gating the dependencies alone sufficient.

This rule is worth stating on its own merits: it is what the flag workaround
cannot express.

It is implemented as a package check reporting `PackageBuildImpossible`, which
`Distribution.Simple.Configure.checkPackageProblems` turns into an error, so the
package cannot be configured. The check walks every branch of each component's
condition tree, since a violation hidden behind a flag is still a violation, and
reads each sublibrary's stanza from its tree root. Reported as
`cross-stanza-dependency`:

```
The package will not build sanely due to these errors:
Error: [cross-stanza-dependency] The executable 'cabal' depends on the library
'testlib', which is in the test stanza. A library in an optional stanza is only
requested when that stanza is, so anything outside the stanza that depends on it
would be left with a missing dependency whenever the stanza is disabled. Either
move the dependency into the test stanza too, or take the library out of it.
```

Three further rules accompany it.

**A stanza-scoped library may not be `public`** (`public-stanza-library`, also
`PackageBuildImpossible`). Whether a stanza is requested is part of *this*
package's configuration, so a package depending on this one has no way to ask for
it; such a library could never be satisfied from outside.

**The `stanza` field may not be set inside a conditional**
(`conditional-stanza`, `PackageBuildImpossible`). A component's stanza has to be
known before conditions are resolved -- the solver reads it from the condition
tree's root -- so a conditional setting would be quietly ignored. Rejecting it is
better than silently taking the root value.

**A stanza-scoped library that nothing in its stanza depends on is warned about**
(`unused-stanza-library`, `PackageBuildWarning`, so not fatal). It would never be
requested and so never built. A library in the same stanza counts as a user, not
just a test-suite or benchmark.

Reachability is transitive. A stanza-scoped library is live only if a test-suite
or benchmark reaches it, directly or through other libraries in the same stanza,
so a dead chain is reported in full rather than one link per run. Verified on a
two-link chain: with the test-suite depending on neither, both libraries are
reported; with the chain live, neither is; and when only the tail is orphaned,
only the tail is reported. The closure keeps a visited set, so a dependency cycle
among libraries terminates rather than looping -- such a package is rejected
elsewhere as a component cycle, but the check must not hang before it gets there.

Two notes on the cross-stanza rule as implemented:

- A component's own stanza is what it may depend on: a test-suite may depend on a
  `stanza: test` library, a benchmark on a `stanza: bench` one, and any component
  on an ordinary library. Anything else is rejected.
- **A stanza-scoped library may depend on another in the same stanza.** A
  library's own stanza is what it is judged by, so `stanza: test` depending on
  `stanza: test` is fine, as is any stanza-scoped library depending on an
  ordinary one. Verified end to end: a chain of two `stanza: test` libraries
  under a test-suite builds and runs with tests enabled, and with
  `--disable-tests` the whole chain is reported as unavailable together:

  ```
  Cannot build the package two-0.1 because none of the components are available
  to build: the test suite 't', the library 'helper-b' and the library 'helper-a'
  are not available because building test suites has been disabled in the
  configuration
  ```
- Making the *main library* depend on a `stanza: test` sublibrary is already
  rejected for a different reason -- the sublibrary depends on the main library,
  so it is a component cycle -- and `cabal` reports it as such. The check is what
  catches the cases that are not cycles, such as an executable depending on it.

### Libraries shared by more than one stanza

Sharing helpers between a package's test-suites and its benchmarks is common, and
a single-valued field cannot express it. The field therefore holds a set:

```cabal
library shared
  stanza: test, bench
```

read as **requested when any of its stanzas is**. A library naming no stanzas --
the default, and what omitting the field means -- is always requested.

Nothing new was needed in the solver. `flagged_deps` is a list and each
`Stanza (SN pn s) deps` entry is independent, so emitting a library's
dependencies once per stanza it belongs to says exactly "required if tests are
on, required if benchmarks are on, not required if neither is".
`Distribution.Solver.Types.OptionalStanza` already had `OptionalStanzaSet` and
`enableStanzas` for the surrounding machinery.

Validation becomes membership: a component in stanza `S` may depend on a library
whose set contains `S`. That also repairs the advice in the error message, which
previously could not be followed. Before, a benchmark depending on a
`stanza: test` library was told to "move the dependency into the test stanza
too", which a benchmark cannot do; now it reads:

```
Either add the depending component's stanza to the library, or take the library
out of its stanzas.
```

**Repeated entries are a warning** (`repeated-stanza`, `PackageBuildWarning`).
`stanza: test, test` means the same as `stanza: test`, so nothing is broken; it
is reported because it is not what anyone intends. The precedent for warning
rather than erroring on a redundant setting is `OptRts` -- "`-rtsopts` has no
effect for libraries" -- which is also a `PackageBuildWarning`. Note this is why
the field is parsed into a list and de-duplicated at its use sites: parsing
straight into a set would swallow the repeat and leave nothing to report.

**An empty field is not reported.** `stanza:` written with nothing after it is
indistinguishable from an absent field: `monoidalFieldAla` yields `[]` for both,
and separating them would mean typing the field as `Maybe [LibraryStanza]` and
threading `fromMaybe []` through every use. It would also be without precedent --
no check in `Cabal` reports that a list field was written empty, and
`build-depends:` with nothing after it passes silently. Treating an empty
`stanza:` as equivalent to omitting it is the behaviour consistent with the rest
of the format.

**Explicit targets.** `cabal build pkg:lib:testlib` with tests disabled should
behave as `cabal build pkg:test:foo` does today, which is to fail with a specific
explanation rather than silently enabling the stanza:

```
Error: [Cabal-7127]
Cannot build the test suite 'unit-tests' because building test suites has been
explicitly disabled in the configuration. ...
```

Reaching that required a change beyond the solver, because
`componentAvailableTargetStatus` in `Distribution.Client.ProjectPlanning` asks
`componentOptionalStanza` -- which answers from the component's *name* -- whether
a component belongs to an optional stanza. For a library it answered "no", so the
component was treated as always available, and asking for it with tests disabled
produced an internal error rather than a diagnosis:

```
Error: [Cabal-7127]
Internal error when trying to build the library 'testlib' from the package
cabal-install-3.19.0.0. The package,component pair is not in the set of
available targets for the project plan, which would suggest an inconsistency
between readTargetSelectors and resolveTargets.
```

The fix is that a library must be asked, not its name. That function already
receives the whole component, so it is a local change; the shared conversion
`libraryStanzaToOptionalStanza` now lives beside `OptionalStanza`.

That produced the right error class but the wrong noun -- "because building
*libraries* has been explicitly disabled" -- because `renderTargetProblem`
derived it with `renderComponentKind Plural (componentKind cname)`, which was
only ever right while an optional component was always a test-suite or benchmark.
Naming the stanza instead means carrying it, and the stanza is *not* known where
the problem is constructed: `selectComponentTargetBasic` sees only the
`AvailableTargetStatus`. So the two disabled statuses now carry it:

```haskell
    TargetDisabledByUser (Maybe OptionalStanza)
  | TargetDisabledBySolver (Maybe OptionalStanza)
```

and `TargetOptionalStanzaDisabledByUser` / `...BySolver` carry it onward to the
renderer. The result reads correctly for both kinds of component:

```
Cannot build the library 'testlib' because building test suites has been
explicitly disabled in the configuration. ...

Cannot build the test suite 'unit-tests' because building test suites has been
explicitly disabled in the configuration. ...
```

The field is `Maybe` because `Distribution.Client.CmdHaddock` repurposes
`TargetDisabledByUser` to mean "not requested by this target filter", where no
stanza is involved; there the renderer falls back to the component kind, which is
the existing wording. That repurposing is arguably worth its own status
constructor, but that is out of scope here.

The distinction is observable, and `IntegrationTests2` pins it. In the haddock
target-problem fixture a benchmark that *was* buildable is rewritten by haddock's
filter and so carries no stanza, while a test-suite already disabled by the solver
keeps its `Just TestStanzas`:

```haskell
  [ AvailableTarget "p-0.1" (CBenchName "user-disabled")
      -- haddock's own target filter, not a disabled stanza
      (TargetDisabledByUser Nothing) True
  , AvailableTarget "p-0.1" (CTestName "solver-disabled")
      (TargetDisabledBySolver (Just TestStanzas)) True
```

Getting this wrong is caught by the suite, which is a useful property: it means
the two meanings cannot quietly merge again.

The general lesson is that the assumption "optionality is a property of the
component's name" is encoded in several places, and each has to be revisited. The
prototype does so for the solver, for planning, and for the two renderers -- the
last of which also fixes the stanza named in `TargetProblemNoneEnabled`
messages, which had the same defect.

**`cabal check`.** The validation rule above is a new check. A stanza-scoped
sublibrary that no test-suite depends on is dead weight and could warrant a
warning.


## Alternatives Considered

`cabal-install`'s solver does not receive a `ComponentRequestedSpec`.
`Distribution.Solver.Modular.IndexConversion.convGPD` converts a package's
components with `os`, `arch` and the compiler only, and expresses the optional
stanzas as a dependency constructor:

```haskell
data FlaggedDep qpn =
    Flagged (FN qpn) FInfo (TrueFlaggedDeps qpn) (FalseFlaggedDeps qpn)
  | Stanza  (SN qpn)       (TrueFlaggedDeps qpn)
  | Simple (LDep qpn) Component
```

Test-suite dependencies are already emitted under `prefix (Stanza (SN pn TestStanzas))`.
Placing a stanza-scoped sublibrary's dependencies in the same group is all the
solver change amounts to.

The alternative of adding a condition to the `.cabal` conditional language --
`if tests()`, parallel to `if impl(ghc)` -- does not work as well:

- `ConfVar` has four constructors (`OS`, `Arch`, `PackageFlag`, `Impl`) and is
  shared with `cabal.project` parsing via `Distribution.Fields.ConfVar.parseConditionConfVar`.
  A new constructor becomes syntactically valid in project files, where it is
  meaningless and where unhandled `ConfVar`s currently reach a raw `error` call in
  `Distribution.Client.ProjectConfig.Legacy`.
- The natural spelling is negative (`if !tests() buildable: False`), but `Stanza`
  carries only true-branch dependencies, unlike `Flagged`. Supporting it would mean
  extending the solver's stanza representation with a false branch.

Component scope is also the more honest model: "this component exists for the
tests" is a property of the component, not a condition on its contents.

A third option, raised alongside `test-only:` in #10900, is a new component
*kind* -- a `test-library` or `testlib` stanza beside `library`. That has one
real attraction: optionality would follow from the component's name again, so the
name-keyed functions described above would keep working untouched, and none of
the plumbing this proposal had to change would need changing.

It costs more elsewhere, though. A component kind is part of a component's
identity: `ComponentName`, the solver's `Component`, target syntax such as
`pkg:lib:foo`, unit ids, the installed package database, Backpack, and every
consumer that pattern-matches those. A field on `Library` reuses the existing
library component wholesale and leaves all of that alone. The new kind would also
need a sibling for benchmarks, and a third for a helper shared by both -- the
same dead end a boolean reaches.

### Naming

The field is spelled `stanza:` throughout this document and in the prototype, but
that spelling is provisional and probably wrong. It is recorded here so the
question is settled deliberately rather than by inheritance from the prototype.

**`stanza:` collides with an established meaning.** In user-facing documentation
"stanza" already means *a group of fields*, and the project file documentation
defines it that way outright: fields "live inside stanzas (groups of fields that
apply to only part of a project)". The docs speak of the `library` stanza, the
`source-repository-package` stanza, `common` stanzas, "a typical stanza for a
foreign library". Under that reading, `stanza: test` written inside a
`library testlib` stanza says "this group of fields is test", which is not the
intended meaning at all.

The term is also internal. `OptionalStanza` is a solver and `cabal-install` type;
no user-facing document uses the phrase "optional stanza". The corresponding
user-facing vocabulary is the project fields `tests:` and `benchmarks:` and the
flags `--enable-tests` and `--disable-tests`.

That suggests naming the field after what the user already types. Candidates:

| spelling | reads as | notes |
| --- | --- | --- |
| `enabled-by: tests` | "this library is enabled by `tests`" | values match the `tests:`/`benchmarks:` project fields and `--enable-tests` exactly; the word "enabled" is, however, already load-bearing in Cabal, where *enabled* means buildable **and** requested |
| `requested-by: tests` | "requested when `tests` are" | matches Cabal's own terminology precisely: `--enable-tests` is what *requests* a component, per `Distribution.Types.ComponentRequestedSpec` |
| `optional: tests` | "optional, along with `tests`" | short, but says nothing about which stanza without reading the value |
| `test-only: True` | "only for tests" | clearest at a glance, and what [#10900](https://github.com/haskell/cabal/issues/10900) converged on, but it is a boolean: it needs a second field for benchmarks and still cannot say "either" |
| `scope: test` | "scoped to tests" | "scope" is overloaded in Cabal already (dependency scope, visibility) |
| `stanza: test` | -- | collides as described above |

`requested-by: tests` is the recommendation. It reuses the word Cabal's own
documentation uses for exactly this state, it keeps *enabled* free for its
existing meaning, and its values are the ones users already write in
`cabal.project`.

Two sub-questions go with it:

- **Plural values.** `tests` and `benchmarks` match `--enable-tests` and the
  project fields; `test` and `bench` match the internal constructors
  (`TestStanzas`, `BenchStanzas`). The user-facing plural is preferable.
- **The default.** The prototype spells it `always`, which reads oddly against a
  `requested-by:` field. Omitting the field is the default in any case, so the
  explicit form could simply be dropped, or spelled `requested-by: none`.

Renaming is mechanical: the field name appears once in the field grammar, and the
constructor names are internal to `Distribution.Types.LibraryStanza`.

## Relation to haskell/cabal#10900

Issue [#10900](https://github.com/haskell/cabal/issues/10900) asks for the same
capability and had converged on a narrower spelling:

> - Libraries get a new field: `test-only: Bool`, by default False.
> - Libraries with `test-only: True` can only be imported by other libraries with
>   `test-only: True` or test-suites.
> - Local libraries with `test-only: True` will only be enabled if
>   `--enable-tests` is on, even if no other component depends on them. They will
>   respect the usual component naming (i.e. `all` will build them if they are
>   enabled).

The mechanism proposed here is the same one, generalised. `test-only: True` is
`stanza: test`; the import rule is the `cross-stanza-dependency` check; and
building an undepended library under `--enable-tests` is the behaviour verified
below. Three points deserve a direct answer.

### A boolean cannot express a library shared with benchmarks

`test-only: Bool` has no room for benchmarks, and none for a helper used by both
the test-suites and the benchmarks -- which is the case that pushed this proposal
to a set. Under a boolean, such a helper must either be duplicated or marked
`test-only: False` and have its dependencies solved unconditionally. The set
subsumes the boolean without losing anything: `stanza: test` is exactly
`test-only: True`.

`test-only:` is nevertheless a strong candidate for the *name*, and reads better
than `stanza:`. It does not survive generalisation, though: `test-only: True` and
`bench-only: True` cannot say "either", which is the whole point. See
[Naming](#naming).

### Undepended libraries are built, and that needed a fix

The issue asks that a `test-only` library be enabled by `--enable-tests` even
when nothing depends on it. It is, and there is a test for it: in
`ConditionalLib/Plan` the `tests` sub-test builds `lib:helper` for a package
whose test-suite happens to depend on it, and a package with an *undepended*
stanza library builds it too.

That did not work at first. `elabStanzasRequested` in
`Distribution.Client.ProjectPlanning` drops a stanza request unless the package
has components of that stanza:

```haskell
TestStanzas -> listToMaybe [v | v <- maybeToList tests, _ <- PD.testSuites elabPkgDescription]
```

A package that provides testlibs but has no test-suite of its own -- exactly the
shape #10900 describes -- therefore had its request dropped, and the library
could never be built. Worse, asking for it produced an uncaught
`renderBuildTargetProblem: unexpected status` rather than a diagnosis, because
the renderer obtained the stanza from the component's *name*. Both are fixed: a
library in a stanza now counts as the package having that stanza, and the
"not requested by default" status carries its stanza the way the disabled
statuses already did.

### Cross-package sharing is the unresolved conflict

The issue's motivating case is sharing helpers between test-suites **in different
packages** -- `unstable-*-testlib` sublibraries consumed by other packages' tests.
That requires `visibility: public`, and this proposal currently makes
`public` plus a stanza an error (`public-stanza-library`), on the grounds that
whether a stanza is requested is part of the *producing* package's configuration
and a consumer cannot ask for it.

Within a single multi-package project that objection is weak: `tests: True`
applies to every local package, so the producer's stanza is on whenever the
consumer's tests are. Across a published dependency it is not weak at all --
nothing enables the stanza of a package coming from Hackage, so the component
would simply be unavailable.

So the honest position is that this proposal serves the single-package case and
the local multi-package case, but forbids the published case, while #10900 wants
all three. Resolving it means deciding what a dependency on a stanza-scoped
library of a *non-local* package should mean. The options seem to be to keep it
an error, to allow it only for local packages, or to let a dependency on such a
library imply enabling that stanza in the dependency -- which is a much larger
change to how stanzas are selected.

### The rebuild concern

The issue notes the worry that enabling these libraries changes build plans and
causes rebuilds, and observes that this is not new -- test-suites and benchmarks
already behave this way. That carries over unchanged here, since the mechanism is
the same one: a stanza-scoped library is requested exactly when its stanza is,
and toggling a stanza has always re-planned.

### Corroboration on the flag workaround

The issue independently reaches the same conclusion about emulating this with
cabal flags: it "changes the API of a package, and as downstream components
cannot specify cabal flags in their `build-depends`, the same problem cascades
down". That matches what the flag experiment here found, and is worth citing as
evidence that the workaround is not merely inconvenient but structurally unable
to serve the multi-package case.

## Backwards Compatibility / Migration

**Backwards compatibility.** The field is gated on `cabal-version: 3.20`, so
older `cabal` reports an unsupported spec version rather than misreading the
package. A package using it cannot be built by older toolchains, which is the
normal cost of new `.cabal` syntax.

The gate is doing more work than it may appear, and the prototype demonstrates
what happens without it. An unrecognised field in a `.cabal` file is a *warning*,
not an error, and the field is then ignored -- so a `cabal` without this feature
reads a `stanza: test` library as an ordinary sublibrary and resolves its
dependencies unconditionally. Running the unpatched `cabal` 3.19 against this
repository's own bootstrap project gives:

```
warnings: cabal-install/cabal-install.cabal:324:5: Unknown field: "stanza"
...
[__1] unknown package: Cabal-QuickCheck (dependency of cabal-install)
```

The warning is buried and the failure surfaces much later as a solver error that
says nothing about the field that caused it. With the gate in place the same
`cabal` refuses the package up front, naming the spec version it cannot satisfy.
Failing loudly and early is the better of the two, and it is the reason the gate
should not be treated as a formality.

**sdist and `flattenPackageDescription`.** Flattening takes all components, so
source distributions are unaffected.

## Interested parties

The request comes from the Haskell community rather than from `cabal`'s own
needs: [#10900](https://github.com/haskell/cabal/issues/10900) was opened for the
[ouroboros-consensus](https://github.com/IntersectMBO/ouroboros-consensus)
project, which carries a set of `unstable-*-testlib` sublibraries and wants them
neither versioned nor changelogged as public API, and needs to share them between
test-suites in different packages. The participants in that issue are the
parties most directly affected, and had converged on a narrower form of this
change; they have not yet been contacted about this proposal.

`cabal` itself is a second interested party: `cabal-install` has four test-suites
sharing a source directory, and is the package the prototype uses as its subject.

Anyone maintaining a package whose test-suites share helper modules is affected
in the same way, which from a scan of Hackage is a common shape.

## Implementation Notes

A working prototype exists, and the author is willing to carry the change
through. What it does and does not cover is set out below; the remaining work is
the open questions rather than the mechanism.

A working prototype accompanies this proposal, implemented against
`cabal-install` 3.19 / GHC 9.14.1.20260728.

### Result

`cabal-install`'s four test-suites share four helper modules. With them moved
into a `stanza: test` sublibrary:

| | bootstrap plan | `testlib` / `Cabal-QuickCheck` / `tasty` in plan |
| --- | --- | --- |
| before, no sublibrary | 18 packages | no |
| sublibrary, no `stanza:` field | **fails to resolve** | -- |
| sublibrary, `stanza: test` | **18 packages, identical** | **no** |

and with tests requested, `cabal build cabal-install:tests` pulls in
`cabal-install-3.19.0.0 (lib:testlib)` and `Cabal-QuickCheck` as expected. No
project-file configuration is involved in either direction: `--enable-tests`
alone decides.

### Changes

| file | change |
| --- | --- |
| `Cabal-syntax/src/Distribution/Types/LibraryStanza.hs` | new `LibraryStanza` type, modelled on `LibraryVisibility` |
| `Cabal-syntax/src/Distribution/Types/Library.hs` | `libStanza` field, `emptyLibrary`, `Semigroup` |
| `Cabal-syntax/src/Distribution/Types/Library/Lens.hs` | `libStanza` lens |
| `Cabal-syntax/src/Distribution/PackageDescription/FieldGrammar.hs` | `stanza` field, sublibraries only |
| `Cabal-syntax/src/Distribution/Types/ComponentRequestedSpec.hs` | `libraryStanzaNotRequestedReason`, consulted by `componentDisabledReason` |
| `Cabal-syntax/src/Distribution/PackageDescription/Configuration.hs` | `overallDependencies` consults the stanza as well as the name |
| `cabal-install-solver/.../Modular/IndexConversion.hs` | stanza-scoped sublibraries emitted under `prefix (Stanza ...)` |
| `Cabal/src/Distribution/PackageDescription/Check/Target.hs`, `Cabal/src/Distribution/Simple/Build.hs` | field added to positional match / record literal |
| `Cabal-tree-diff/src/Data/TreeDiff/Instances/Cabal.hs` | `ToExpr LibraryStanza` |
| `cabal-install-solver/.../Types/OptionalStanza.hs` | `libraryStanzaToOptionalStanza` |
| `cabal-install/.../ProjectPlanning.hs` | `componentAvailableTargetStatus` asks the component; the two disabled statuses carry the stanza |
| `cabal-install/.../TargetProblem.hs`, `ProjectOrchestration.hs`, `CmdErrorMessages.hs`, `CmdHaddock.hs` | stanza threaded to the renderers; messages name the stanza |
| `cabal-install/tests/IntegrationTests2.hs` | fixtures updated with the stanza |
| `Cabal/.../Check.hs`, `Check/Warning.hs` | four checks: `cross-stanza-dependency`, `public-stanza-library`, `conditional-stanza`, `unused-stanza-library` |
| `Cabal-tests/.../Utils/Structured.hs` | golden structure hashes for `GenericPackageDescription` and `LocalBuildInfo` |
| `Cabal-tests/tests/CheckTests.hs`, `ParserTests/regressions/stanza-*.cabal`, `*.check` | golden cases for the four findings |
| `Cabal-tests/tests/ParserTests.hs`, `stanza-roundtrip.cabal`, `.format`, `.expr` | pretty-printer and round-trip coverage |
| `cabal-testsuite/PackageTests/ConditionalLib/{Plan,Deps,CrossStanza}` | end-to-end behaviour |

The solver change is the whole mechanism, and it is small:

```haskell
++ concatMap (convSubLib initDR) [sl | sl <- sub_libs, isNothing (subLibStanza sl)]
++ prefix (Stanza (SN pn TestStanzas))
     (L.map (convSubLib (addStanza TestStanzas initDR))
            [sl | sl <- sub_libs, subLibStanza sl == Just TestStanzas])
```

### Tests

Each of the four checks has a golden case under
`Cabal-tests/tests/ParserTests/regressions/`, registered in `CheckTests.hs`. The
fixtures carry version bounds and a long enough description so that the golden
output is only the finding under test, rather than incidental advice that would
break the case whenever unrelated checks change.

`stanza-cross-dep.cabal` is the discriminating one: an executable and a
test-suite both depend on the same `stanza: test` library, and only the
executable is reported. `stanza-unused.cabal` holds a two-link dead chain and its
golden names both libraries, pinning the transitive behaviour.

Parsing and printing are covered separately by `stanza-roundtrip.cabal`,
registered in `ParserTests`, which carries a library for each of the three values
plus one that omits the field. That fixture gets the `format` golden, the `expr`
golden, and the round-trip assertion -- the last being the one that matters, since
it pretty-prints and re-parses and compares `condSubLibraries`, so a field that
printed but did not survive a re-parse would fail rather than quietly round-trip
to the default. The golden output confirms the intended asymmetry: `stanza: test`
and `stanza: bench` are printed, while `stanza: always` is omitted as the default
and still reads back as `LibraryStanzaAlways`.

These are real regression tests, not just recordings: disabling the cross-stanza
rule fails exactly `stanza-cross-dep.cabal`, with a readable diff, and leaves the
other three passing.

Note that `tasty-golden` writes a missing golden file and reports the test as
passing, so a first green run proves nothing. The files were read and checked
against what each case is meant to produce.

### Integration tests

The checks above all work on a package *description*. What the feature actually
does -- keep a library's dependencies out of the solve while its stanza is not
requested -- is exercised by `cabal-testsuite`, which drives a real `cabal`
against a real package and is run by `cabal-validate`. The cases live together
under `cabal-testsuite/PackageTests/ConditionalLib/`.

| test | asserts |
| --- | --- |
| `ConditionalLib/Deps` | the load-bearing one: `helper` names a package that does not exist, and with `--disable-tests` the solve succeeds anyway. If its dependencies were required the build would fail, which is precisely what happened to `cabal.bootstrap.project` before the field existed |
| `ConditionalLib/Plan/cabal` | four named sub-tests over one package -- `neither`, `tests`, `benchmarks`, `both` -- each with its own golden. A library in both stanzas appears under `tests` and under `benchmarks`, and under neither when both are off |
| `ConditionalLib/Plan/target-disabled` | asking for `pkg:lib:helper` with tests disabled fails, and the message names *test suites* |
| `ConditionalLib/CrossStanza` | an executable depending on a `stanza: test` library fails the build with `cross-stanza-dependency` |

The first was added after checking that the others do not cover the solver: with
`subLibStanza` stubbed to `Nothing`, so that stanza-scoped libraries are treated
as ordinary ones, `ConditionalLib/Deps` fails and the plan tests still pass. They
guard different things -- the plan tests guard which components are *requested*,
which is the planner, not the solver.

An earlier version of `ConditionalLib/Deps` gave `helper` only a `base` dependency.
That fixture was useless: whether such a dependency is solved for is
unobservable. The dependency has to be one that cannot be satisfied for its
absence from the solve to be visible.

### Note on the golden data that a new field disturbs

Adding a field to `Library` moves two sets of golden data, and both are part of
the change rather than churn to be papered over.

The `Structured` hashes of `GenericPackageDescription` and `LocalBuildInfo`
change, because that hash is precisely how `cabal` invalidates its caches when
the description format changes.

The 38 `.expr` fixtures under `Cabal-tests/tests/ParserTests/regressions/` also
move, since they pretty-print a parsed `GenericPackageDescription`. Regenerating
them with `--accept` is safe here only because the result was checked: across all
38 files the diff is 88 added lines, every one of them
`libStanza = LibraryStanzaAlways,`, and nothing removed. A regeneration that
showed anything else would mean the field had changed behaviour rather than
merely appeared.

### Deliberate deviations

- **The field is spelled `stanza:`.** See [Naming](#naming); `requested-by:` is
  the recommended spelling and the rename is mechanical. The prototype predates
  that discussion.
- **The `cabal-version` gate is relaxed.** The shipped field should carry
  `availableSince CabalSpecV3_20`. The prototype omits it because the in-tree
  `Cabal` is 3.19, so no available `Cabal` can satisfy a `cabal-version: 3.20`
  package and `cabal-install` could not build itself. The one-line gate is marked
  in `FieldGrammar.hs`.

  The consequence, in this repository, is that `cabal-install` now describes
  itself with a field that only a `cabal` built from this branch understands. An
  older `cabal` ignores it and then fails to resolve the bootstrap project, as
  above. That is a chicken-and-egg peculiar to prototyping `.cabal` syntax inside
  the package that defines it, and it is why landing the feature has to sequence
  the `cabal-version` bump against the `Cabal` version: the field can only be
  used by `cabal-install` itself once a released `Cabal` supports the spec
  version that gates it.

### Not implemented

Nothing outstanding from the original list. What remains is judgement the
proposal process should settle rather than the prototype: the field's spelling
(see [Naming](#naming)), whether `CmdHaddock`'s reuse of `TargetDisabledByUser`
deserves its own status constructor, and documentation for the users' guide.
- Explicit-target behaviour (`cabal build pkg:lib:testlib` with tests disabled) is
  untested.
- `optionalStanza` (`CmdErrorMessages.hs`) still answers from the component's
  name alone. It is now only reached for statuses that carry no stanza, so it is
  no longer wrong, but it remains a name-keyed answer to a question that is not
  about names.
- `CmdHaddock`'s reuse of `TargetDisabledByUser` for "not requested by this
  target filter" would be better as its own status constructor.
- Parser round-trip tests, pretty-printer tests, documentation.

## Open Questions

- **The spelling of the field.** `stanza:` is used throughout the prototype and
  is probably wrong; see [Naming](#naming), where `requested-by:` is recommended
  and `test-only:` -- what #10900 converged on -- is discussed.
- **Cross-package sharing.** Whether a stanza-scoped library may be `public`, and
  what a dependency on such a library of a *non-local* package should mean. This
  is the substantive disagreement with #10900 and is set out under
  [Cross-package sharing is the unresolved conflict](#cross-package-sharing-is-the-unresolved-conflict).
- **An empty `stanza:` field** is currently indistinguishable from an absent one
  and so is not reported. Warning about it would mean typing the field as
  `Maybe [LibraryStanza]`, and would be the first such check in `Cabal`.
- **`CmdHaddock`'s reuse of `TargetDisabledByUser`** to mean "not requested by
  this target filter" is why the status carries a `Maybe OptionalStanza` rather
  than a stanza. It would be better as its own status constructor.
- **Sequencing the `cabal-version` bump.** The field can only be used by
  `cabal-install` itself once a released `Cabal` supports the spec version that
  gates it.

## References

- [haskell/cabal#10900](https://github.com/haskell/cabal/issues/10900) -- "New
  field on library: `test-only: Bool`", the request this proposal answers, and
  the source of the `test-only:` and `test-library` alternatives.
- [ouroboros-consensus](https://github.com/IntersectMBO/ouroboros-consensus) --
  the `unstable-*-testlib` sublibraries that motivated that issue.
- `Distribution.Types.ComponentRequestedSpec` -- the note on *buildable* versus
  *requested* versus *enabled* components, which this proposal's semantics are
  stated in terms of.
- `doc/cabal-project-description-file.rst` -- the `tests:`, `benchmarks:` and
  `semaphore:` fields, and the section on conditionals and imports.
