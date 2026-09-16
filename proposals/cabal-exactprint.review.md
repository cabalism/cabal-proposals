# Review of "Cabal Exactprint" (haskell/cabal-proposals#7)

Reviewer: Phil de Joux.
Reviewed at commit `5088cfd` of this branch. Line references (`L123`) point into
`proposals/cabal-exactprint.md` at that commit. Because the markdown is
generated from `cabal-exactprint.typ` with pandoc, fixes belong in the typst
source.

Besides the text, I read the two haskell/cabal PRs the proposal rests on:

- haskell/cabal#11252 "Retain comments in field parser" (the parser change),
- haskell/cabal#12316 "Exactprinter & modification framework, take 2" (the
  proof of concept, stacked on #11252).

Where the proposal and the proof of concept disagree I say so, since the
proposal is what the cabal developers are being asked to accept.

## Overall

I support the direction. Using `[Field ann]` as the concrete syntax tree is the
right call, and the "Alternatives Considered" section is a genuinely useful
record of why the annotated-GPD approaches failed. The proof of concept is
small and readable.

The proposal is not yet ready to be decided on, for three reasons.

1. It does not state the actual data-model changes to `Distribution.Fields`
   that the proof of concept makes, and its backwards-compatibility section
   ("we don't foresee any issues", L663) is wrong as a result.
2. The round-trip property it advertises (`print . parse == id`, L24) is not
   the property that is implemented or tested. The implemented property is
   `print . parse == normalize`, and `normalize` should be spelled out.
3. It does not say what it is asking the developers to decide. I have listed
   the decisions I think are implicit in section A.

Sections B and C are the substantive questions. Section D is the
motivation/prior-art gaps, E is document structure, F is line-level nits.

## A. Decisions the proposal should ask for explicitly

The proposal reads as a status report. A proposal should end with a short list
of yes/no decisions. From the text and the proof of concept, I think these are:

1. Adopt `[Field ann]` (with annotations) as the concrete syntax tree for exact
   printing and editing, rather than `GenericPackageDescription`.
2. Accept the breaking changes to `Distribution.Fields.Field` needed to carry
   the extra information (see B3). Say which release they land in.
3. Accept a normalisation policy: tabs become spaces, trailing whitespace is
   dropped, whitespace-only lines become empty lines, a single line ending is
   used per file, a final newline is added. gbaz already said yes to most of
   this on the PR; write it into the proposal.
4. Decide where the modification framework lives (Cabal-syntax, Cabal, or a new
   package) and its stability status for the first release (see B9).
5. Decide the initial user-facing consumers (which of `cabal add`,
   `gen-bounds`, `format`, `init` are in scope for the funded work) so that the
   API is driven by at least one real command.

## B. Design questions

### B1. The round-trip property is `print . parse == normalize`, not `== id`

L24 defines the goal as `print . parse == id` (and calls this "idempotency",
which it is not; idempotency would be `f . f == f`). L26 then says it only
holds "for valid package descriptions", but the real qualifications are
elsewhere: L104 (CRLF and tabs), L741 (trailing whitespace and whitespace-only
lines are lost), and in the proof of concept the Hackage round-trip test
normalises the *input* before comparing (tabs to spaces, CR stripped, trailing
whitespace stripped, blank lines emptied, a final newline appended). The
renderer also unconditionally appends a trailing newline.

Please state the property that is actually tested:

    exactRenderFields . readFieldsConcrete == normalize

and define `normalize` in one place. Then the 60% / 99% numbers (L99, and
Jappie's comment on the PR) have a precise meaning. Right now the 60% figure
is measured against raw input and the 99% figure against normalised input, so
they are not comparable.

Related: the summary says the work "gives downstream users a stability
guarantee" (L22). A guarantee of what? Of the output format of `Pretty`
instances? Of the `Field` type? Please say.

### B2. Absolute positions as the layout representation

The design keeps every node's absolute `Position` and reconstructs whitespace
by padding up to the next position. Every insertion therefore has to renumber
every later sibling (steps 6.1 to 6.3, L139-L150), and every removal either
leaves a gap or renumbers.

This is exactly the situation ghc-exactprint was in before GHC 9.2, and it is
why ghc-exactprint now converts anchors to relative `DeltaPos` before editing:
with relative positions an insertion is local and nothing else needs to move.
Was a delta-based annotation considered? The alternatives section discusses
three CST candidates but not this choice, and it is the choice that determines
how hard the modification framework is to get right.

The proof of concept shows the sharp edges already:

- `offsetFieldRow` shifts the field's own annotation but not the comments
  attached to it (the `HasPosition (WithComments ann)` instance only touches
  `unComments`), and `fieldRowRange` has a TODO saying it ignores comment
  height.
- The checked-in golden file `edit-printed/simple_add-field-start.expr` in
  #12316 is visibly wrong. Inserting one field at the top of a 13-line file
  produces `-- before` at column 9, `build-depends` and its colon on separate
  lines, and the `-- after` comment printed above the `base > 4.8` line it
  used to follow. That output is currently the *expected* output of the test.
- Removal does not renumber (`removeField` leaves later positions untouched,
  `modifyField` clamps the shift with `max 0`), so removing a field leaves
  blank lines behind. Step 6.2 (L146) presents this as an open choice; the
  proposal should say which behaviour is proposed, because `cabal add` /
  `format` users will see it.

Either the proposal should explain how the absolute-position design is going
to be made robust (a renumbering pass that includes comments, tested by
property), or it should consider switching to relative positions before the API
is fixed.

### B3. The data model is not described

The proposal never shows what `Field ann` looks like after the change. From
the two PRs, the shape is:

    data Comment ann      = Comment !ByteString !ann
    data WithComments ann = WithComments { justComments :: [Comment ann], unComments :: ann }
    data Field ann        = Field Position !(Name ann) [FieldLine ann]   -- new colon Position
                          | Section !(Name ann) [SectionArg ann] [Field ann]
    data SrcSpan          = SrcSpan !Position !Position
    data Located a        = MkLocated { getSrcSpan :: SrcSpan, unLocated :: a }

plus `Name` no longer being lower-cased by `mkName` (the invariant is deleted
and `readFields` lower-cases afterwards). None of this is in the proposal, yet
it is most of what a reviewer needs to evaluate: which nodes carry comments,
where "preceding vs succeeding" comments attach (the changelog in #11252 says
each field gets both), how field-line positions map to file positions after
`joinFieldLines`, and how the colon position is represented.

Please add a "Data model" subsection under "Proposed Change" with these types
and their invariants. `Located` is used at L166 and L205 without ever being
introduced.

### B4. Backwards compatibility section is wrong

L663 says "the changes are local to the parser, we don't foresee any
backwards-compatibility issues". The proof of concept:

- adds a field to the `Field` constructor (breaking every pattern match in
  cabal-fmt, cabal-gild, haskell-ci, the HLS cabal plugin,
  cabal-plan-bounds, ...),
- removes the "lower-case ASCII" invariant on `Name` and changes `mkName`,
- reimplements `readFields` as `readFieldsWithComments` followed by stripping
  (#11252), so comment collection now happens on every parse, including the
  parse of every package in the Hackage index. Has this been benchmarked?
  `Cabal-benchmarks` and the `hackage-tests parsec` target exist for this.

Even if the final design differs, the section should list the exported types
and functions that change, and say whether the plan is one breaking release or
a deprecation path (for example, keep `Field` as is and put the colon position
into the annotation).

### B5. Cabal spec version threading

In the proof of concept, `Edit a = CabalSpecVersion -> a -> EditResult a`, and
the typed helpers parse and print with `runParsecParser' spec` /
`prettyVersioned spec`. The proposal never mentions the spec version, but it
matters: leading commas are only accepted from 2.2, optional commas from 3.0,
so the same `build-depends` bytes parse differently depending on
`cabal-version`. Questions:

- Where does the version come from? Read from the `cabal-version` field before
  editing, presumably, but the proposal's own headline example (L116) is
  editing `cabal-version`. What happens to edits sequenced after that one?
- Does `prettyVersioned` guarantee that a value rendered for version `v` is
  accepted by the parser for version `v`? If not, step 8 (L154) can fail after
  a "successful" edit.

### B6. Braces syntax and signalling non-exactness

L27 excludes brace syntax. Fine, but the parser still accepts it, so what does
`exactRenderFields` do with a brace-syntax file: refuse, or silently print the
layout form? `cabal add` needs to know when the exactness guarantee is off so
it can warn or bail. Suggest a lexer warning or a flag from
`readFieldsConcrete` that consumers can inspect, and a stated property for the
non-exact case (at minimum semantics preserving: `parseGPD . print . parse ==
parseGPD`).

### B7. Focus semantics

The `Edit` sketch at L326-L355 focuses with predicates. The cases a real
`cabal add` hits are not discussed:

- A monoidal field split across several `build-depends:` occurrences in one
  stanza (this is the very example used at L570). Which one does
  `AddField (hasFieldName "build-depends")` target? The proof of concept has
  `ModifyFirst` / `ModifyLast` / `RemoveAll` / `RemoveFirst`; the proposal
  should present that vocabulary and its defaults.
- Common stanzas: the dependency the user wants may belong in a `common`
  block that the target stanza `import`s. Does the framework resolve imports,
  or is that the caller's job?
- Conditionals: the same field name occurs in several `if` branches of one
  component.
- Field creation (L345 "create it should it not exist"): where in the section
  is the new field placed, and with what indentation? (See B8.)
- Case: since `Name` is no longer lower-cased in the concrete tree,
  `hasFieldName "build-depends"` must compare case-insensitively. Say so.

### B8. Indentation and style inference

`mkFieldAt` in the proof of concept hard-codes `indentedCol = col0 + 2` with a
TODO to read it from config. For a tool that promises not to "mangle the
format", new fields and new list items should inherit the surrounding style
(indent width, leading vs trailing comma, one item per line vs packed). The
"Prettiness" open question (L785) touches this for list items only. Please
state the intended policy, even if it is "copy the style of the nearest
sibling, else use a configurable default".

### B9. Where the code lives and what stability it has

Cabal-syntax is a GHC boot library. Whatever API ships there is effectively
frozen by the next GHC release, and the proof of concept still has TODOs about
labels for match failures, partiality of `prependValueListBS`, and moving
functions to an internal module. Options worth stating:

- ship the parser/data-model change in Cabal-syntax (it is needed there) but
  ship the editing framework in a separate package or an explicitly
  `Internal`/experimental module for one release cycle;
- or commit to the API now and say why.

Also note that `Distribution.Fields.Transform` imports `Text.Parsec` directly,
whereas the rest of Cabal-syntax goes through `Distribution.Parsec`.

### B10. Validation (step 8, L154) should be semantic

Bodigrim's review comment asked for a sanity check after editing. The step
added says "parses to the transformed value", which validates the field-line
text. The check `cabal add` really needs is at the GPD level:

    parseGPD (print (edit fields)) == semanticEdit (parseGPD (print fields))

That is a property test that can run against Hackage for the simple edits
(add a dependency, bump a bound) and would catch B2-style position bugs
automatically. Suggest listing it under testing (L317-L324) alongside the
golden tests.

### B11. Comments inside edited field lines

`modifyValueList` extracts the comments from the field lines, edits the joined
bytes, then re-interleaves the comments. What is the rule when the edit changes
the number of lines before a comment (a dependency rendered as two lines, or a
multi-line item collapsed to one)? The L237-L260 example only shows the case
where line counts are unchanged.

## C. Where the proposal and the proof of concept disagree

These should be reconciled, in whichever direction is intended.

| Proposal | Proof of concept (#12316) |
|---|---|
| `Newtype b a` constraints (L194, L205, L215, L226) | `Coercible a b`; `Distribution.Compat.Newtype` was removed from cabal master in July 2026 (commit `75d677f8c`), as Kleidukos noted |
| `CommaVSep` separator (L251, L267, L284, L304) | no such type; the separators are `CommaVCat`, `CommaFSep`, `VCat`, `FSep`, `NoCommaFSep` |
| `addValueList` with `InsertPosition` (L213-L221) and `removeValueList` (L224-L232) | only `prependValueList` exists; there is no append and no item removal |
| `readFieldsWithComments` (L699) | renamed to `readFieldsConcrete` |
| `[FieldLine Position] -> [FieldLine Position]` (L191 onwards) | `CabalSpecVersion -> [FieldLine (WithComments Position)] -> EditResult [FieldLine (WithComments Position)]` |
| `Edit` described only as focus + transformation (L326) | `Edit` has an `EditOk` / `EditUnchanged` / `EditErr` result algebra with `andThen`, `orFallback`, `failIfUnchanged`. This is a good design and worth a paragraph |
| 119662 / 194557 round-trips (L99) | Jappie reports ~99%; update the number and say what normalisation it assumes (B1) |

## D. Motivation, prior art, interested parties

- L39: `cabal format` is a hidden command in cabal-install. Worth saying, since
  "it drops your comments" reads as a criticism of a supported feature.
- L56: HLS already offers "add missing module to cabal file" and "add
  dependency" code actions, implemented on top of cabal-add. That strengthens
  the motivation (the ecosystem is routing around Cabal) and should be cited.
- Prior art is missing cabal-gild (tfausak), which is a formatter that keeps
  comments, and cabal-plan-bounds (nomeata), which rewrites bounds in place.
  Both are closer to this work than hpack (L75), which is a generator rather
  than an editor. hpack belongs in "interested parties" as a potential consumer
  of the printer, not as a symptom.
- The template asks "Have you contacted them?" (L667). Bodigrim has reviewed.
  Suggest naming who has been asked or should be: phadej (cabal-fmt,
  haskell-ci), tfausak (cabal-gild), fendor / VeryMilkyJoe (HLS cabal plugin),
  sol (hpack), nomeata (cabal-plan-bounds).
- L715: the template asks for a timeline. Give the remaining funded period and
  a target Cabal release for the parser change and for the first command.

## E. Document structure

- "Alternatives Considered" is about 45% of the document and each alternative
  appears twice (a bullet at L362-L415 and a subsection at L418-L660). Keep
  the bullets to one paragraph each and move the detailed post-mortems to an
  appendix. The proposal will be read by people deciding, the appendix by
  people implementing.
- "Proposed Change" should gain, in this order: data model (B3), round-trip
  property with normalisation (B1), position handling (B2), the `Edit` algebra
  (C), focus semantics (B7), then the typed field-line helpers and examples.
- The examples at L237-L313 should compile against the proof of concept, or
  be clearly labelled pseudo-code. Today they are neither (see F).
- Add a "Decisions requested" section (A) before "Open Questions".

## F. Line-level nits

- L24: "idempotency" -> "round-trip property" (see B1).
- L34: "loselessly" -> "losslessly". L36: "through out" -> "throughout".
- L72: "an non-official" -> "an unofficial".
- L126: joining field lines "with indentation and newlines" into "a single
  field line" violates the documented `FieldLine` invariant (no newlines).
  Say "into a single ByteString" or describe `joinFieldLines` explicitly.
- L150: "Modification is be a hybrid" -> "Modification is a hybrid".
- L191: haddock says "given a function `a -> a`" but the type is
  `a -> Maybe a` (same at L203).
- L224-L232: `removeValueList` is missing `=>` after the constraint tuple
  (Kleidukos).
- L250-L252: `setBaseVersionTo :: Version -> ...` but `Dependency` takes a
  `VersionRange`; and `Depedency` is misspelled twice (also L284).
- L263 says "append" but the code at L267 uses `Prepend` and the output at
  L270-L275 shows a prepend (Kleidukos).
- L302: `Dependency -> Dependency -> Ord` -> `Ordering`.
- L304: `@(List CommaVSep @Identity) @Dependency` is not valid; something like
  `@(List CommaVCat (Identity Dependency) Dependency) @[Dependency]` matches
  the proof of concept's `Coercible` formulation.
- L308-L314: the sorted output has a trailing comma after the last item and
  moves `-- interleaved comments` between different items than before. If
  that is the intended behaviour it should be said in the sentence above it,
  not left for the reader to spot.
- L371: "imtate" -> "imitate". L461: "engeerning" -> "engineering".
  L722: "evoving" -> "evolving". L746: "subsituting" -> "substituting".
- L421: stray "+" in "along with its associated trivia. + To construct".
- L584: "and and".
- L749: "1.9141 percent" -> "1.9%".
- L699: "can be simplified the new" -> "can be simplified with the new".
- Throughout: "september" -> "September"; "Cabal Exactprint" vs "exactprint"
  vs "Exact-printer" are all used; pick one.
