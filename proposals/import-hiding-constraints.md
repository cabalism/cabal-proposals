# Import modifiers: `hide-constraints`

## Summary

Allow an `import:` in a `cabal.project` file to carry modifiers, indented
beneath the import location. The first and only modifier is
`hide-constraints`, a comma-separated list of package names whose constraints
are dropped from the imported file and from anything that it imports in turn.

```
import: https://www.stackage.org/lts-21.25/cabal.config
  hide-constraints: hashable, text

constraints: hashable ==1.4.2.0, text ==2.0.2
```

This gives a project a way to use a different version of a package than the
one pinned by an imported snapshot, without downloading and editing the
snapshot.

## Motivation

Since cabal-install 3.8, a project can import a Stackage snapshot's
`cabal.config` directly by URL. That snapshot pins every package with an
equality constraint. Constraints in cabal are additive: every constraint on a
package, from every file, is intersected. So a project that imports a snapshot
and then wants `hashable ==1.4.2.0` where the snapshot says `==1.4.3.0` gets
an unsolvable plan:

```
[__1] rejecting: hashable-1.4.3.0
      (constraint from cabal.project requires ==1.4.2.0)
[__1] rejecting: hashable-1.4.2.0
      (constraint from stackage.config requires ==1.4.3.0)
        imported by: cabal.project
```

The only workaround today is the one the user guide documents: download the
snapshot, import it by relative path, and comment out conflicting lines by
hand, repeating `cabal build --dry-run` until the solver is happy. That is
issue [#9511](https://github.com/haskell/cabal/issues/9511), opened in
December 2023, and it is the last remaining step of the
[Updo](https://blockscope.com/posts/2023-11-15-updo.html) workflow that still
needs manual editing of a downloaded file. Stack users are used to overriding
snapshot versions in place with `extra-deps`; cabal users have no equivalent.

The problem is not specific to Stackage. Any shared, imported configuration,
whether a company-wide project file, a CI configuration, or a curated set of
constraints, will eventually need to be adjusted at the point of use for one or
two packages without being copied.

## Proposed Change

### Syntax

The `import` field currently takes a single line, the location of the import.
This proposal allows further lines, indented beneath the location, each being
a modifier of the form `name: value`. This is ordinary multi-line field syntax
in the `.cabal` file format, so the location remains the first line and the
modifiers are the remaining lines.

```
import: <location>
  hide-constraints: <package>, <package>, ...
```

- Only `hide-constraints` is recognised. Any other modifier name is a parse
  error naming the unknown modifier, so a typo such as `hide-constraint` fails
  rather than being silently ignored.
- The value of `hide-constraints` is a comma-separated list of package names.
  Anything else, such as a version range, is a parse error.
- The modifier may be repeated within one import; the lists are concatenated.
- An `import` with no modifiers is unchanged.

### Semantics

For an import with `hide-constraints: p1, p2, ...`:

- Every constraint on `p1`, `p2`, ... is removed from the imported file's
  parsed configuration. This includes constraints inside conditional blocks
  of the imported file.
- Hiding is transitive along the import's own chain. If the imported file
  itself imports another file, constraints on those packages coming from that
  further import are also removed. In other words, the modifier applies to
  the whole subtree rooted at that import.
- Hiding does not reach sibling imports or the importing file. If
  `cabal.project` imports `a.config` with `hide-constraints: hashable` and
  also imports `b.config`, any `hashable` constraint in `b.config` or in
  `cabal.project` itself is kept.
- All kinds of constraint on the named packages are hidden: version, flag,
  stanza, and `installed`/`source`. This is deliberate. A snapshot that pins
  `hashable ==1.4.3.0` may also set `hashable +random-initial-seed`, and the
  point of hiding is to take the imported file's word on nothing about that
  package, then say what you want in your own `constraints:` field.
- Only the `constraints` field is affected. Other per-package settings in the
  import, such as `flags` in a `package` stanza, `allow-newer`, or
  `source-repository-package` entries, are not touched.

### Diagnostics

- A package named in `hide-constraints` for which the import subtree has no
  constraints at all produces a warning at normal verbosity, so that stale
  entries are noticed when a snapshot is bumped:

  ```
  Warning: hide-constraints found no constraints to hide for text;
    stackage.config
      imported by: cabal.project
  ```

- At `-v2`, the constraints that were hidden are listed, each with the file
  it came from, so the effect of the modifier can be audited.

### Documentation

The `cabal.project` reference gains a `hide-constraints` section next to the
existing description of `import`, and the Stackage limitations section of the
user guide replaces the download-and-edit workaround with the modifier.

## Alternatives Considered

The discussion on #9511 raised three directions. This proposal picks the
narrowest one that solves the reported problem and keeps the door open to the
others.

### Override semantics for `constraints:` ("last one wins")

Make a later constraint on a package replace, rather than intersect with, an
earlier one. This was argued for on #9511 as the uniform and principled
answer, since the same rule could apply to every field of a project file and
would solve the snapshot case as a special case.

It was not chosen here because it changes the meaning of every existing
`constraints:` field. Today it is common and correct to have several partial
constraints on one package accumulate from several files. Changing that
globally is a much larger compatibility question than the snapshot use case
requires, and it would need its own proposal with a migration story. Nothing
in this proposal prevents such a change later. If override semantics were
adopted, `hide-constraints` would remain useful for the case where you want
to drop a constraint without replacing it.

### Solve for a subset of packages and pin the rest

Pick some packages to solve freely and take the snapshot's answer for
everything else. This was suggested on #9511 as more principled than touching
`constraints:` at all.

It was not chosen because constraints are not only versions. A snapshot's
flag constraints are load-bearing and would need the same treatment, and the
solver currently uses constraints for local package sourcing too, so the
change is deep. As was also noted on #9511, this is itself a form of override,
just a more specialised one.

### A top-level field rather than an import modifier

Something like `hide-constraints: hashable` at the top level of the project
file, hiding constraints from all imports.

An import modifier was chosen because it names the thing being modified. Two
imports may both constrain a package, and a top-level field cannot say which
one to hide. Attaching the list to the import also makes the transitive rule
easy to state and easy to see in the file.

### Making `constraints:` from local packages not apply

Also raised on #9511. It does not address the snapshot case, where the
conflicting constraint is on a non-local dependency.

### Naming

`hide-constraints` was chosen over `override-constraints`, `mask-constraints`
and `ignore-constraints`. It does not itself override anything; it removes
constraints, and the user adds their own in the ordinary way. "Hide" is
already the word cabal uses for removing something from consideration, as in
hidden packages and `--hide-successes`.

## Backwards Compatibility / Migration

- Project files without import modifiers are parsed and interpreted exactly
  as before, under both the legacy and the parsec project file parsers.
- A project file that uses `hide-constraints` will not work with an older
  cabal-install. Older versions read the whole multi-line `import` field as
  the import location and will fail to find a file by that name. This is the
  same situation as any new project file field. A project that must support
  older versions can keep using the download-and-edit workaround.
- No change to the `Cabal` library, the `.cabal` file format, or the
  `cabal-version` spec.
- No change to solver behaviour. The solver sees fewer constraints; it does
  not see new kinds of input.

## Interested parties

- Users importing Stackage snapshots directly, the audience of the "Stackage
  snapshots" section of the user guide.
- Users of tools that generate project files from snapshots, such as Updo.
- The Stackage curators, since this makes importing `cabal.config` by URL
  usable without local edits. They have not been contacted.
- Haskell Language Server and hie-bios, which drive the cabal executable
  rather than parsing project files themselves, so no work is expected there.

## Implementation Notes

I am willing to implement this and have done so on the
`add/version-overriding-masking` branch of my fork, targeted at the
`cabal-install` in the next release:

- `Distribution.Client.ProjectConfig.Import` gains `ImportSpec`,
  `parseImportSpec`, `hideConstraints` and `hideImportConstraints`, plus the
  two messages.
- Both project file parsers, `Legacy` and `Parsec`, parse the import field
  through `parseImportSpec` and apply `hideImportConstraints` to the parsed
  import before merging it.
- Unit tests cover the modifier parser. A `cabal-testsuite` package test,
  `ProjectImport/HideConstraints`, covers the conflict without hiding,
  hiding, hiding through an intermediate import, sibling imports in both
  directions, the unused-package warning and the unknown-modifier error, for
  both parsers.
- A changelog entry and documentation are included.

The change is about 470 lines including tests and docs.

## Open Questions

- Should hiding be transitive through the import's own imports, as proposed,
  or apply only to constraints written directly in the imported file? The
  transitive rule matches how a user thinks of "the snapshot" as a unit, but a
  case for the non-transitive rule could be made for imports of imports that
  the user controls.
- Should the unused-package warning be an error instead? A warning was
  chosen so that a project file can be shared across snapshot bumps without
  breaking, but an error would catch typos in package names sooner.
- Should there be a way to hide only version constraints while keeping flag
  constraints, for example `hide-constraints: hashable (version)`? Nothing in
  the syntax rules it out. It is left out until there is a use case.
- Is `import:` with modifiers the right place to grow further per-import
  settings, such as pinning an index-state for a remote import or
  hiding other fields? This proposal establishes the syntax but only
  defines one modifier.

## References

- [#9511 Override version equality constraints](https://github.com/haskell/cabal/issues/9511),
  the issue this addresses, with the design discussion summarised above.
- [#7556 RFC: Cabal support for LTS Snapshots](https://github.com/haskell/cabal/issues/7556),
  where the gap was first identified when remote imports landed.
- [#7833 Adding support for revision pinning in dependency version](https://github.com/haskell/cabal/issues/7833),
  the other limitation listed in the Stackage section of the user guide, not
  addressed here.
- [Stackage snapshots in the cabal user guide](https://cabal.readthedocs.io/en/latest/nix-local-build.html#stackage-snapshots),
  documenting the current download-and-edit workaround.
- [Updo](https://blockscope.com/posts/2023-11-15-updo.html), a tool that
  generates project files from snapshots.
