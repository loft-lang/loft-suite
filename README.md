<!--
Copyright (c) 2026 Jurjen Stellingwerff
SPDX-License-Identifier: LGPL-3.0-or-later
-->

# loft-suite — the language suite, outside the compiler

A black-box test suite for the [loft](https://github.com/loft-lang/loft) language: every
test here is a `.loft` program a released `loft` binary can run, and nothing here links loft
as a Rust crate.  The suite is the record of what the language promises, kept apart from
the repository that implements it, so a change to loft is checked against a promise that
did not move with it.  Design and phases: loft-lang/plans#193.

## The tree is the rule register

```
rules/<chapter>/<rule>/<cell>.loft     one directory per formal chapter, one per rule
attic/scripts/<program>.loft           the inherited corpus — kept, run nightly only
loft-suite                             the runner
```

`<chapter>` is a chapter of loft's formal definition (`doc/claude/formal/<chapter>.md`) and
`<rule>` a rule it defines (`B-Copy`, `L-Null`, `D-bind-11`).  A cell cites the rule it sits
under in its header (`@FR-B-Copy`), and may cite others; `loft-suite check` fails a cell
that sits under a rule it does not cite.  A rule with no directory is a visible gap, not an
absence.

## A cell

A self-contained program `loft --tests` can run: every zero-parameter function is an entry
point (`fn main` alone when it exists, `fn test_*` when any is declared), `assert` is the
oracle, and the annotations are the ones `loft test` reads:

| header line | meaning |
|---|---|
| `// @EXPECT_ERROR: <substring>` | the file (or the function below) must be refused with a diagnostic containing it |
| `// @EXPECT_WARNING: <substring>` | a warning containing it must be emitted; any other warning fails the cell |
| `// @EXPECT_FAIL[: <substring>]` | the function below (or every one) must fault at run time |
| `// @IGNORE` | the function below is not run |
| `// @ARGS: <argv…>` | positional arguments; `--lib`, `--project`, `--production`, `--log-conf` as flags |
| `// @STORE_LIMIT: <n>` | a ceiling on live stores, handed to the binary as `LOFT_STORE_LIMIT` |
| `// @falsified-at: <sha> — …` | the receipt that the cell FAILED on the build it was written to catch |

Where an annotation sits decides its scope.  Everything before the file's first `fn`,
`struct` or `enum` is the HEADER, and an annotation there applies to the whole file — so an
`@IGNORE` written above the FIRST function of a file with no declarations before it ignores
the file, not that function.  To ignore one function, put a declaration or another function
above it.

Every cell runs with `LOFT_LEAK_CHECK=1`: a store left behind by a function that returned
cleanly fails it.  A warning the cell claims must fire; one it does not claim is printed,
not failed — the inherited cells were never warning-clean, and a never-read local is often
the shape a cell needs.  A chapter that has been built out claims every warning its cells
raise.  A cell's expected value is hand-computed; two backends agreeing is not a pass.

## Running

```
loft-suite run                                 every rule cell, interpreter, 8 at a time
loft-suite run @FR-B-Copy                      one rule (refinements B-Copy-* included; --exact)
loft-suite run --chapter ownership --mode both both backends, one chapter
loft-suite run --attic --junit out.xml         the inherited corpus too (the nightly)
loft-suite list @FR-L-Null                     the cells a selection names
loft-suite attic --unclaimed                   attic programs no rule has claimed
```

`--loft <binary>` (or `$LOFT`) names the binary under test.  The interpreter lane runs
from a release bundle as is.  The native lane needs loft's runtime library beside the
binary, which a bundle does not carry: run it from a source checkout (`make install`) or
inside loft's own build job.  One process per cell, no inner pool.

## Where it runs

1. On every loft PR, against the release binary that PR builds — the `Language suite`
   job in loft's `ci.yml`, a required check.  The same job runs on every push to loft
   `main`, which is the nightly reading of the rules.
2. Nightly, the attic too (`--attic`; a report, never a gate).
3. Against each release bundle — the compatibility check an older version never had.

loft pins the suite's commit the way it pins a library: `tests/loft-suite.pin` in the loft
repository names the commit its CI checks out.  A behaviour change is a loft PR plus a
suite PR, and the loft PR bumps the pin.  From a loft checkout with this repository beside
it, `make suite ARGS="@FR-B-Copy --mode both"` runs a selection against that tree's binary.

## The attic

`attic/` holds the corpus loft carried before the suite existed, verbatim.  Nothing is
deleted from it except by PROMOTION: when a rule is built out, each attic program the rule
explains is either subsumed (the rule's cells already cover it — deleted, the commit naming
the cell) or promoted (moved under the rule with a hand-computed answer).  The attic only
shrinks.
