# Override constraints

## Summary

Add an `override-constraints` field to project files and a `--override-constraint` command-line flag. Plain
`constraints` stay additive: they are intersected, as today. An override *replaces* other constraints of the same kind
on the same package: every such constraint at a weaker position, and plain constraints at the same position. A
position is weaker if it comes from a less authoritative configuration layer, or sits deeper in the import tree. Two
different overrides at the same position are an error. Overrides are resolved before the solver runs, so the result
doesn't depend on the order of lines or imports.

## Motivation

Importing a package set, such as a Stackage snapshot's `cabal.config`, pins hundreds of packages with `==` and
`installed`. Cabal constraints only intersect, so a project cannot use a different version of even one pinned package.
The user guide documents a workaround: download the `cabal.config`, then repeatedly comment out conflicting lines
(`doc/nix-local-build.rst`, "Limitations"). [Updo](https://github.com/up-do/updo), a tool that generates cabal
projects from a package set plus per-project constraints, automates exactly that edit.

The same need shows up beyond snapshots:
- company-wide constraint files;
- CI-only configuration;
- a local `cabal.project.local` that bumps one dependency without editing shared files.

Earlier attempts and discussion:

- #9511 is the issue.
- #9510 tried implicit "shallowest import wins" for `==`, applied inside the solver behind a `--version-win` option
  (`VersionWin = ShallowWins | LastWins`). It later experimented with "last wins".
- The review of #9510 raised these points:
  - Keep project semantics out of the solver, and pick one semantics rather than a mode flag (gbaz).
  - "Imported" should not implicitly mean "overridable"; express intent explicitly, for example with an `override`
    keyword that can be grepped for (michaelpj, andreabedini).
  - Against an `override` keyword: an override could be "buried at the bottom" of a chain of imports, and with two
    overrides "it would not be clear which would win" (gbaz). Some prefer order, where later overrides earlier
    (gbaz, Mikolaj). Others want conflicts at the same level to stay errors.
- Not raised in that review, but visible in every snapshot: constraints are more than versions. Snapshots pin flags
  too, and those need overriding as well.
- A separate, unmerged branch proposes a `hide-constraints` import modifier that removes an import's constraints
  wholesale, by package name. It's coarse: it drops flags along with versions, and must be attached to each import.

This proposal answers the objections to an `override` keyword directly: a total order on positions says which
override wins, a disagreement at the same position is an error, and every replacement is reported. It aims for:
- explicit intent;
- deterministic results that don't depend on order;
- same-level conflicts reported as errors;
- no change to the solver algorithm;
- per-property granularity.

## Prior art

| Tool | Mechanism | Who may override | Conflicts |
|---|---|---|---|
| npm (`overrides`), Yarn (`resolutions`), pnpm (`pnpm.overrides`) | Replace a dependency's version anywhere in the tree, optionally scoped to a parent (nested keys in npm, `foo>bar` in pnpm, `**/bar` in Yarn) | Root `package.json` only | npm rejects an override that contradicts a direct dependency unless it references it |
| Dart pub (`dependency_overrides`) | Ignore every constraint on a package and use the given one | Root package only; ignored in dependencies | pub reports overridden packages at resolution |
| Elixir Mix (`override: true`) | A dependency declared in the root wins over requirements from other dependencies | Root project | Without `override`, diverging requirements are an error |
| Go modules (`replace`, `exclude`) | Replace or exclude module versions | Main module only; ignored in dependencies | n/a |
| Bazel bzlmod (`single_version_override`, …) | Pin or patch a module version | Root module only; ignored elsewhere | n/a |
| Cargo (`[patch]`) | Replace a crate's source, still within semver | Root workspace only | Must satisfy requirements |
| uv (`constraint-dependencies` vs `override-dependencies`) | *Additive* constraints and *replacing* overrides are two separate fields | Workspace root only | Overrides are "absolute": they replace all requirements on that package |
| Maven (`dependencyManagement`, BOM import) | "Nearest wins" mediation; local declarations beat imported BOMs | Nearer in the tree | At equal depth, and between BOMs, first declaration wins (order) |
| Gradle (rich versions `require`/`strictly`/`reject`, `force`) | Explicit strength per declaration | Any module, including published metadata | Conflicting `strictly` fails the build |
| Stack (snapshot layering, `extra-deps`, `flags`, `drop-packages`) | `stack.yaml` and child snapshots override parent snapshot versions *and flags*, per package | Closer layer wins | n/a |
| Nix module system (`mkDefault`, `mkForce`, `mkOverride n`) | Numeric priority; the lowest number (highest priority) wins | Anyone | Same priority with different values is an error |
| nixpkgs overlays | A later overlay wins over `prev` | Anyone | Order-based |
| Docker Compose (multiple `-f` files; `!override`, `!reset`) | Later files merge; explicit tags replace or remove | Later files | Order-based |
| Spack configuration scopes (`::` suffix) | Scopes ranked defaults < … < user < environment < command line; `::` replaces instead of merging | Higher scope | Scope rank |
| cabal (`active-repositories: r:override`) | An existing per-repository `:override` modifier in project files | Project | n/a |

Four themes run through these tools:

1. **Replacing is marked explicitly, separately from merging.** Examples are uv's two fields, Compose `!override`,
   Spack `::`, Gradle `strictly` and Nix `mkForce`.
2. **Authority sits with the root, or with whatever is nearer to it.** npm, Yarn, pnpm, Dart, Mix, Go, Bazel, Cargo
   and uv honour overrides only in the root. Maven's nearest-wins is the graded version of the same idea, though it
   falls back to order at equal depth.
3. **Conflicts at equal strength are errors**, as in Nix and Gradle, and in Mix without `override`.
4. **Order-based systems exist**, such as overlays, Compose and Maven BOMs, but order is a known source of surprise.

This proposal takes themes 1–3. Stack is the closest analogue for Haskell users: it overrides versions and flags per
package from a closer layer.

## Proposed Change

### Syntax

```cabal
-- cabal.project
import: https://www.stackage.org/lts-23.0/cabal.config

override-constraints: any.hashable ==1.4.2.0, any.text ==2.0.2, any.foo -bar
```

```console
$ cabal build --override-constraint="any.hashable ==1.4.2.0"
```

`override-constraints` uses exactly the syntax of `constraints`. An override is itself a constraint: it goes to the
solver like any other. Snapshots pin with `any.` scopes, so overrides of snapshot pins are written with `any.` too; see
"Scopes" below for why.

### Position

Every constraint has a **position**, made of a layer and a depth.

The **layers**, strongest first, are:
1. the command line, and user targets such as `cabal install foo-1.2` (depth 0);
2. `cabal.project.local` and its imports;
3. `cabal.project`, `cabal.project.freeze` and their imports;
4. the global config file (`~/.config/cabal/config`; `ConstraintSourceMainConfig` in the code).

The **depth** is the number of imports between the layer's root file (depth 0) and the file that contains the
constraint. Constraints inside `if` blocks have the position of the file that contains them.

Position A is **stronger** than position B when either:
- A's layer is stronger than B's; or
- the layers are the same and A has the smaller depth.

Layer is compared first, so a file imported by `cabal.project.local`, at any depth, outranks `cabal.project` itself.

For example, with `cabal.project` importing `stackage.config`, which in turn imports `ghc-9.8.config`:

| Constraint | Layer | Depth |
|---|---|---|
| `--constraint="any.hashable ==1.4.2.0"` on the command line | 1 | 0 |
| `cabal install foo-1.2` (a user target) | 1 | 0 |
| `constraints:` in `cabal.project.local` | 2 | 0 |
| `constraints:` in a file imported by `cabal.project.local` | 2 | 1 |
| `constraints:` in `cabal.project` | 3 | 0 |
| `constraints:` in `cabal.project.freeze` | 3 | 0 |
| `constraints:` in `stackage.config` | 3 | 1 |
| `constraints:` in `ghc-9.8.config` | 3 | 2 |
| `constraints:` in the global config file | 4 | 0 |
| `flags:` in a `package` stanza, `--flags`, and constraints cabal adds itself | outside the position system | — |

`cabal.project.local` outranks `cabal.project`. It holds uncommitted, per-developer configuration, so a developer can
override constraints from the shared project, its freeze file and its imports without editing any of them. This is
already how the rest of the project configuration is layered: the user guide says the sources are combined with
"later entries override earlier ones", in the order global config, `cabal.project`, `cabal.project.freeze`,
`cabal.project.local` (`doc/cabal-project-description-file.rst`; `readProjectConfig` merges
`global <> local <> freeze <> extra`). The layers above are that order, read strongest first. It is also the usual
pattern in configuration tools, where local settings win over shared ones (Spack's scopes and Docker Compose override
files, for example).

Constraints with no file provenance are outside the position system. These are the constraints cabal adds itself
(rule R4), and flags from `flags:` in `package` stanzas and from `--flags`, which reach the solver as
`PackagePropertyFlags` constraints tagged `ConstraintSourceConfigFlagOrTarget` (see Open Questions).

### Kinds

An override replaces constraints of the same *kind* on the same package:

- **version:** version ranges, `installed` and `source`. An override `any.Cabal ==3.12.1.0` replaces a snapshot's
  `any.Cabal installed`.
- **flag *f*:** one kind per flag name. A multi-flag constraint such as `any.foo +bar -baz` counts as one constraint
  per flag, so an override `any.foo -bar` replaces only the `bar` part, leaving `any.foo -baz`.
- **stanzas:** stanza constraints such as `foo test` only ever enable a stanza, so they are not overridable.

Flags from `package` stanzas are not constraints of any kind here: an override never replaces them, and they never
replace an override.

### Scopes

A scope **contains** another scope when every package instance the second applies to, the first applies to as well:
- `any.p` contains every scope for `p`;
- `setup.p` contains `setup.p` and `q:setup.p` for any `q`;
- any other scope contains only itself.

An override replaces a constraint only when the override's scope contains the constraint's scope. This is a
containment rule, not an overlap rule, on purpose. The solver's scope language has no "every instance except the
top-level one", so if a top-level `hashable ==x` were allowed to replace a snapshot's `any.hashable ==y`, the setup
and executable instances of `hashable` would silently lose their pin. Instead:

- an override whose scope is narrower than a same-kind constraint it would otherwise replace (weaker or same
  position, scopes overlapping but not containing) is an **error** that names both and suggests the wider scope:

  ```
  Error: override-constraint hashable ==1.4.2.0 (cabal.project) is narrower than
    any.hashable ==1.4.3.0 (stackage.config, imported by cabal.project).
  Write any.hashable ==1.4.2.0 to override every instance, or setup.hashable ==1.4.2.0 for setup instances only.
  ```

### Rules

- **R1:** Plain constraints never remove anything. They intersect, as today.
- **R2:** An override replaces:
  - every constraint of the same kind, whose scope it contains, at a *weaker* position;
  - *plain* constraints of the same kind, whose scope it contains, at the *same* position.
- **R3:** Two different overrides of the same kind, with overlapping scopes, at the same position are an error that
  names both files. Identical overrides are merged. Two overrides are identical when they are equal as constraints
  after simplifying version ranges, regardless of which file or import path they came from.
- **R4:** Constraints that cabal adds itself are never replaced. These include non-reinstallable packages, `setup`
  Cabal version bounds, profiled-dynamic and the constraints derived from `package` stanzas and `--flags`.
- **R5:** Resolution is a pure pass over the collected constraints, after conditionals are instantiated and before
  solving. The solver is unchanged.

The pass proceeds from the strongest position to the weakest. At each position, overrides first replace everything
R2 says they replace at weaker positions; a constraint that has been replaced is gone before R3 is checked. So a root
override silences a disagreement between two imported snapshots, which is exactly its job, and R3 only reports
disagreements among overrides that survive.

Duplicate imports need no special treatment. Neither parser deduplicates an import reached by two paths, so its
constraints appear twice with different import chains. Under R3's definition of identity, copies at the same depth
merge, and a shallower copy replaces a deeper one.

### Examples

1. **Sibling imports, as updo lays out a project.** `cabal.project` imports `stackage.config` and
   `constraints.config`, both at depth 1. `constraints.config` has `override-constraints: any.hashable ==1.4.2.0`,
   which replaces the snapshot's plain `any.hashable ==1.4.3.0` at the same position.
2. **A root override beats a deep import.** An override in `cabal.project` replaces a snapshot pin that is imported
   three levels down.
3. **An import cannot overrule the root.** Suppose `cabal.project` has a plain `constraints: any.hashable ==1.4.3.0`
   and a remote import has `override-constraints: any.hashable ==1.4.2.0`.
   - The import is at a weaker position, so nothing is replaced.
   - The solver reports the conflict with both sources, as it does today.
   - The root can resolve it by making its own constraint an override, or, if `hide-constraints` lands, by hiding
     the package on that import.
4. **Conflict.** Two sibling imports both override `hashable`, with different versions, and nothing above them
   overrides it:

   ```
   Error: conflicting override-constraints at the same position:
     any.hashable ==1.4.2.0 from a.config (imported by cabal.project)
     any.hashable ==1.4.3.0 from b.config (imported by cabal.project)
   ```

   Adding `override-constraints: any.hashable ==1.4.2.0` to `cabal.project` replaces both and removes the error.
5. **Local tweak.** `cabal.project.local` overrides a version pinned in `cabal.project` or its freeze file, without
   editing either file. A plain `--constraint` on the command line is still stronger than a `.local` override, so the
   two conflict in the solver as today.
6. **Freeze file.** `cabal.project.freeze` is at the same position as `cabal.project`, so an override in
   `cabal.project` replaces a stale freeze pin. `cabal freeze` then records the overridden version as a plain
   constraint. The freeze file records the *effect* of overrides, not the overrides themselves, so removing an
   override later reintroduces the conflict between the freeze file and the snapshot until the project is frozen
   again.

### Reporting

- With `-v2`, every replacement is listed with the override, what it replaced, and both sources.
- Replacing a constraint from `cabal.project.freeze` is reported at normal verbosity, since a freeze file is meant
  to be authoritative.
- An override that replaces a plain constraint in the *same file* gets a warning, since that is likely a mistake.
- An override that replaces nothing gets a warning, as `hide-constraints` proposes for unused package names. It is
  usually a stale override left behind after a snapshot bump.
- Solver failure messages keep showing constraint sources, now including overrides.

### Relation to `hide-constraints`

If both land, the two features are complementary:
- `hide-constraints` is a per-import veto. It removes both plain and override constraints on the named packages
  from that import's subtree.
- `override-constraints` is a precise replacement for one property of one package, and isn't tied to a particular
  import.

## Alternatives Considered

- **Last wins, based on order.**
  - Simple to state, and it is how single-valued fields such as `optimization:` already combine, both within a file
    and across the global/project/freeze/local files. But list-valued fields such as `constraints` are
    order-independent today, and that is the property users rely on when they import snapshots.
  - Moving an `import:` would silently change which constraint wins.
  - It cannot report conflicts at the same level.
  - Maven's BOM ordering and nixpkgs overlays show how order confuses people.
- **Implicit shallowest-wins for `==` (#9510).** It ties overriding to the import structure without any stated intent.
  It ignores flags and `installed`, and it was implemented in the solver.
- **Root-only overrides, as npm, Dart, Go, Bazel and uv do.** This is the simplest model. It could be a first phase of
  this proposal, allowing `override-constraints` only at depth 0 and on the command line. However, it cannot express
  updo's layout, where overrides live in an imported, generated `constraints.config`.
- **Whole-package replacement, as uv's "absolute" overrides do.** An override on `p` would drop every constraint on
  `p`, whatever its kind. Simpler to specify, but bumping a version would silently drop a snapshot's flag pins for
  the same package, which is the "constraints are more than versions" objection to #9510 all over again.
- **Overrides attached to an import.** `import: X` with an indented `override-constraints:` modifier, combining
  masking and replacement for one import. It has to be repeated on every import that pins the package, and it does
  not help `cabal.project.local`, which has no import to attach to.
- **Strength levels, like Gradle's `require`/`prefer`/`strictly`.** A middle ground between one `override` marker
  and numeric priorities. Two levels, plain and override, cover the cases raised so far; more can be added later.
- **Numeric priorities, like Nix's `mkOverride`.** Maximally flexible, but heavyweight for cabal users, and priorities
  in remote files become an arms race.
- **Importing as preferences**, where a modifier on `import: X` turns its `==` constraints into `preferences`. The
  solver silently deviates when it can't satisfy them, which hides mistakes and can't be reported deterministically.
- **Retrying the solver after dropping conflicting constraints.** The #9510 review rejected this, on the grounds that
  the solver's inputs should be fixed.
- **An `override` marker inside `constraints:`**, for example `constraints: override hashable ==1.4.2.0` as
  suggested in the #9510 review. A separate field is easier to grep for, easier for both project parsers to handle,
  and mirrors naturally as a command-line flag.
- **Masking only, with `hide-constraints`.** It drops flags along with versions, and must be repeated on every import
  that pins the package.

## Backwards Compatibility / Migration

- The change is purely additive: without `override-constraints`, behaviour is unchanged.
- Older cabal-install versions warn about an unknown field, then report the same solver conflict as today.
- Existing workarounds, such as edited downloaded snapshots, keep working and can be replaced gradually. So would
  `hide-constraints`, if it lands.
- `cabal freeze` output keeps its form. It records the plan that was solved with overrides applied, as plain
  constraints (Example 6).
- `cabal configure --override-constraint=…` persists the override into `cabal.project.local`, as `--constraint`
  does today. The override then sits in layer 2 rather than layer 1, which is still above `cabal.project`.

## Interested parties

- People and tools that consume Stackage `cabal.config` files: updo, haskell.nix users and Stackage curators.
- Participants in the #9511 and #9510 discussions: gbaz, michaelpj, int-index, andreabedini, phadej and Mikolaj.
- None have been contacted about this text yet; the proposal PR is the invitation.

## Implementation Notes

I'm willing to implement this, following the prototypes below. Timeline: to be agreed once the proposal is
accepted.

- **Field.** Add `projectConfigOverrideConstraints :: [(UserConstraint, ConstraintSource)]` to `ProjectConfigShared`,
  next to `projectConfigConstraints` in `ProjectConfig/Types.hs`, with:
  - a lens in `ProjectConfig/Lens.hs`;
  - a legacy field description in `ProjectConfig/Legacy.hs`, which is also the printer that `cabal configure` uses
    to write `cabal.project.local`;
  - a grammar entry in `ProjectConfig/FieldGrammar.hs`, tagging the source as the `constraints` entry does.
- **Command line.** Add `--override-constraint` to `configureExOptions` in `Setup.hs`, tagged
  `ConstraintSourceCommandlineFlag`. Note the reach of `configureExOptions`: v2 commands pick it up through
  `NixStyleOptions`, but so do v1 `configure` and `install`, and `ConfigExFlags` is part of `SavedConfig`, so the
  field is also accepted in the global config file. That is consistent with layer 4, but it means v1 commands see the
  flag whether or not they honour it (Open Question 4).
- **Position.**
  - The depth is the length of the `ProjectConfigPath` minus one; the path already records the import chain.
  - The layer is not recorded today. Constraints from `cabal.project`, `cabal.project.freeze` and
    `cabal.project.local` are all tagged `ConstraintSourceProjectConfig path`, distinguishable only by the root file
    name, and `resolveSolverSettings :: ProjectConfig -> SolverSettings` has no `DistDirLayout` to compare against.
    `readProjectConfig` reads each root with a `ProjectFileKey` (`Main`, `Local`, `Freeze`) and merges the skeletons,
    so the layer is known there. Two options:
    - add a layer to `ConstraintSourceProjectConfig`. `ConstraintSource` and `ProjectConfigPath` live in
      `cabal-install-solver`, so this is a type change in that package, though the solver algorithm is untouched;
    - keep the layer in `ProjectConfigShared`, for example by tagging the override list per layer in
      `readProjectConfig` before the skeletons are merged.
    The first is simpler and also improves error messages.
- **Resolution.**
  - A new pure module, `Distribution.Client.ProjectConfig.Override`, called from `resolveSolverSettings` on
    `projectConfigConstraints` and `projectConfigOverrideConstraints`, before the profiled-dynamic constraint is
    prepended to `solverSettingConstraints`. Setup Cabal bounds, non-reinstallable packages and `package` stanza flags
    are added later, in `ProjectPlanning` and `Dependency`, so R4 holds for them without special handling.
  - Conditionals are instantiated before `resolveSolverSettings` runs, so the pass only sees active branches.
  - Conflicts and narrow-scope errors raise a `CabalInstallException`.
  - The solver algorithm is unchanged. `cabal-install-solver` changes only if the layer is recorded in
    `ConstraintSource`.
- **Tests.**
  - QuickCheck properties:
    - with no overrides, the result is the identity;
    - the result, as a set, doesn't depend on input order;
    - command-line plain constraints are never removed by overrides from files;
    - resolution is idempotent;
    - an override never removes a constraint whose scope it does not contain.
  - cabal-testsuite package tests against a local repo, as the `hide-constraints` branch does. Cases:
    - sibling imports;
    - a deep import;
    - the root beating an import;
    - the root silencing a conflict between two imports;
    - a conflict;
    - a narrow-scope error;
    - flag granularity;
    - a `.local` override against a freeze file;
    - the same file imported twice.
- **Prototypes.** #9510 (depth tracking) and the `hide-constraints` branch (tree filtering in both parsers) show that
  the plumbing is small.

## Open Questions

1. Is the containment rule for scopes right, or should a top-level override be allowed to replace an `any.p`
   constraint, accepting that setup and executable instances of `p` lose the pin?
2. Flags set in `package p` stanzas (`flags:`) and by `--flags` lose their file provenance before solving. Should
   they get positions so that overrides can replace them, or stay out of scope, as proposed?
3. Should overrides be allowed in remote (URI) imports? The position rule already stops them from beating the root.
4. Should v1 commands such as `v1-install` and `v1-freeze` honour overrides, ignore them with a warning, or reject
   them?
5. Should `installed` and `source` share a kind with version ranges, as proposed?
6. Should user targets (`cabal install foo-1.2`) be in the command-line layer, as proposed, or never replaced?
7. Should a plain `--constraint` on the command line be replaceable by an override in `cabal.project.local`? As
   proposed, no: the command line is the strongest layer.
8. Should `cabal freeze` write overrides through, so that the freeze file stays consistent when an override is later
   removed, or keep writing only plain constraints (Example 6)?
9. What should the field be called: `override-constraints`, `constraint-overrides` or `force-constraints`?
10. Should a first phase ship root-only overrides (depth 0 and the command line) before depth-graded ones?

## References

- [#9511 Override version equality constraints](https://github.com/haskell/cabal/issues/9511)
- [#9510 Override imported package version equalities](https://github.com/haskell/cabal/pull/9510) (prototype and
  review)
- [#7556 Conditionals and imports in project files](https://github.com/haskell/cabal/pull/7556)
- The `hide-constraints` branch (masking on imports)
- [Updo](https://github.com/up-do/updo)
- Tool documentation:
  - [npm `overrides`](https://docs.npmjs.com/cli/v10/configuring-npm/package-json#overrides),
    [Yarn `resolutions`](https://yarnpkg.com/configuration/manifest#resolutions) and
    [pnpm `overrides`](https://pnpm.io/package_json#pnpmoverrides)
  - [Dart `dependency_overrides`](https://dart.dev/tools/pub/dependencies#dependency-overrides)
  - [Mix `override`](https://hexdocs.pm/mix/Mix.Tasks.Deps.html)
  - [Go `replace` and `exclude`](https://go.dev/ref/mod#go-mod-file-replace)
  - [Bazel `single_version_override`](https://bazel.build/rules/lib/globals/module#single_version_override)
  - [Cargo `[patch]`](https://doc.rust-lang.org/cargo/reference/overriding-dependencies.html)
  - [uv `override-dependencies` and `constraint-dependencies`](https://docs.astral.sh/uv/reference/settings/)
  - [Maven dependency mediation](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html)
  - [Gradle rich versions](https://docs.gradle.org/current/userguide/rich_versions.html)
  - [Stack snapshots](https://docs.haskellstack.org/en/stable/topics/snapshot_location/)
  - [Nix module priorities](https://nixos.org/manual/nixos/stable/#sec-option-definitions-setting-priorities)
  - [Docker Compose merging (`!override`, `!reset`)](https://docs.docker.com/reference/compose-file/merge/)
  - [Spack configuration scopes](https://spack.readthedocs.io/en/latest/configuration.html)
