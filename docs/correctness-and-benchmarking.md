# Correctness and Benchmarking Strategy

Date: 2026-04-16

## Status

Draft — open for discussion

## TL;DR

- We built a small correctness toolchain using BigCloneBench (Java) and ran an
  initial experiment. It found two concrete bugs in treepeat's detection logic.
- Three performance optimizations are ready but held pending a correctness
  discussion — one is zero-risk, two need evaluation. See
  [optimization-experiments.md](optimization-experiments.md).
- Several open questions below are worth a quick conversation before we invest
  further. The most important ones are marked **Question for maintainer**.

## Background

The maintainer noted apprehension that detection results are not "100% kosher".
This document attempts to frame what correctness means for a clone detector,
survey what has already been tried, and propose directions for building
systematic, machine-verified confidence in treepeat's output.

This is not an ADR proposing a single decision. It is a discussion document
intended to surface trade-offs before significant implementation work.

## What "Correct" Means for a Clone Detector

Unlike a compiler, a clone detector does not have a binary right/wrong answer.
The relevant dimensions are:

- **Precision**: of reported clone groups, how many are genuine?
- **Recall**: of actual clones in the corpus, how many were found?
- **Score accuracy**: does the reported similarity percentage reflect ground truth?
- **Stability**: does the same input always produce the same output?
- **Invariant preservation**: do semantics-preserving edits (renaming a variable,
  adding whitespace) leave detection results unchanged?

These dimensions trade off against each other. A lower LSH threshold improves
recall but may reduce precision. A stricter verification step improves precision
but may miss borderline clones.

Optimization work might impact correctness. Likewise, correctness work might
impact performance, so these two efforts are intermingled.

## What Has Already Been Tried

The git history shows a sustained effort to improve correctness alongside feature
development. Notable correctness-related work includes:

- **False positive fixes** — self-overlapping regions producing false 100%
  similarity (commit `e3667f0`), incorrect union-find transitivity grouping
  (`9872ce4`), false positives where only function names differ (`f35b955`)
- **Threshold fixes** — default CLI threshold lowered from 1.0 to 0.5
  (`919671f`), LSH threshold separated from filter threshold (`907c202`),
  MinHash threshold corrected (`0896923`)
- **Overselection** — region extraction tuned to avoid over-reporting
  (`0cec308`, `8db3e14`)
- **Verification layer** — LCS-based order-sensitive verification added on top
  of MinHash similarity to catch cases where set similarity is high but structural
  order differs (`03d4541`)
- **Rule-by-rule tests** — individual language rule tests added for Python, Java,
  JavaScript, Kotlin (`a053d8c`, `f594a2e`, `ce4ab27`)

Question for maintainer: Are there specific false positive or false negative cases
on your mind, or behind these earlier fixes? Knowing more details would help
prioritize the approach below.

## Clone Taxonomy

The research literature uses a standard taxonomy:

- **Type I** — Exact copies (modulo whitespace/comments)
- **Type II** — Structurally identical, renamed identifiers
- **Type III** — Near-miss: added, removed, or changed statements
- **Type IV** — Semantic equivalents with different structure

Treepeat's approach (AST shingling + MinHash LSH + order-sensitive verification)
is designed to detect Type I through III. Type IV is generally considered out of
scope for structural detectors and is left to semantic/ML approaches.

Question for maintainer: Is there a target clone type or similarity threshold that
defines the intended detection envelope? e.g. "reliably detect Type I/II,
best-effort for Type III, make no claims about Type IV"?

## An Experiment: BigCloneBench via CodeXGLUE

As an investigation toward systematic evaluation, we extracted a sample from
[BigCloneBench (BCB)](https://github.com/clementbirkle/BigCloneBench) as
distributed by
[Microsoft CodeXGLUE](https://github.com/microsoft/CodeXGLUE/tree/90f11dc34f14fb3995631f4cab53d9b8ed9d2ecd/Code-Code/Clone-detection-BigCloneBench)
(pinned to a specific commit for reproducibility).

BCB contains 9,126 Java method-level functions with 56,820 labeled clone pairs
and 358,596 labeled non-clone pairs in its test split. Each function is a method
body; the label is binary (clone / not-clone) without per-pair type annotation.

### What we prototyped:

A small toolchain (`tools/correctness/`) that:

1. **Downloads** the BCB corpus (`scripts/download_corpus.py`)
2. **Samples** pairs stratified by: same method name (likely Type I/II), high
   text similarity with different names (likely Type III), and hard negatives
   (same name, labeled non-clone) (`scripts/sample_bcb.py`)
3. **Generates** Java test suite directories with multiple methods per file,
   clone pair members shuffled among filler functions so position provides no
   hint (`scripts/build_suites.py`)

The generated suites live in `tools/correctness/suites/` alongside
`expected.json` files listing which method pairs should be detected as clone
groups. These are committed; the large BCB corpus files are gitignored.

### Initial result

Running treepeat against the `clone_same_name` suite (20 labeled clone pairs
among 60 total methods across 3 Java files) found 5 of 20 expected groups —
25% recall on pairs that should be near-exact matches. We investigated the
misses and found at least two distinct failure modes.

**Issue 1: class regions competing with method regions in LSH**

The 4 detected pairs ranged from 20–35 lines. The 16 missed pairs ranged from
8–42 lines. Investigating short missed pairs (e.g. `sha1`, 8 lines,
text-similarity 1.00) with DEBUG logging showed:

```
Query for sha1:2-9 returned 4 similar key(s)
Found similar group of 4 region(s) with 91.7% similarity
  - sha1 [A.java]
  - sha1 [B.java]
  - class A          ← wrapper class bundled in
  - class B          ← wrapper class bundled in
Filtered: below similarity_percent (100.0%) after verification
```

The `class_declaration` region for a file containing one short method has
shingles that are nearly identical to that method's shingles — the class body
*is* the method. LSH bundles them into a single 4-way group; the average
pairwise similarity across that group drops below the 100% threshold; and the
genuine clone pair is discarded along with it.

This is both a test fixture issue (single-method-per-file amplifies the
problem) and a treepeat issue (class regions probably should not compete with
their own contained method regions in LSH candidate generation).

**Issue 2: structurally similar functions cross-contaminating clone groups**

Longer pairs that should be unambiguous also missed — e.g. `decodeFileToFile`
and `encodeFileToFile` (both 27 lines, text-similarity 1.00). These are in
multi-method files so the thin-wrapper issue does not apply. DEBUG logging
showed:

```
Query for decodeFileToFile returned 6 similar key(s)
Found similar group of 4:
  - decodeFileToFile [A]
  - encodeFileToFile [A]   ← cross-contamination
  - decodeFileToFile [B]
  - encodeFileToFile [B]
Verified: LSH=100%, Ordered=99.8%
Filtered: below 100%
```

`decodeFileToFile` and `encodeFileToFile` are both file I/O loops using the
same Base64 library. After AST normalization they are structurally
indistinguishable, so LSH merges what should be two separate 2-way clone groups
into one 4-way group. The 4-way group then fails the 100% similarity threshold
and both pairs are lost.

This would affect real codebases too — files containing multiple similar
utility functions (several hash variants, several file-copy implementations)
will exhibit the same behavior.

**What remains unexplained**

These two issues account for some of the 15 misses but we have not traced all
of them. The lower-similarity pairs (`copy` at 0.86, `hash` at 0.83, `SHA1` at
0.75, `getMD5` at 0.45) may be genuine near-misses that fall below treepeat's
effective detection range, BCB label noise, or further instances of the
cross-contamination problem. Further investigation is needed before drawing
conclusions about the full recall picture.

Question for maintainer: The class-region-vs-method-region LSH pooling issue
(Issue 1) looks like a correctness bug independent of any benchmark. Does this
match anything you've seen, and is it the kind of thing you'd want a fix for
before further correctness work?

### Limitations of BCB for treepeat

- **Java only.** Treepeat supports 10+ languages; BCB exercises only one.
- **Method-level only.** BCB contains no class-level or file-level clones.
- **Binary labels.** The CodeXGLUE distribution does not include per-pair clone
  type (I/II/III/IV). The original BCB dataset has type annotations but requires
  a separate download and other work.
- **No ground truth for score accuracy.** BCB tells us whether two functions
  are clones, not what their similarity score should be.

## Possible Future Directions

These are starting points for discussion, not proposals. The intent is to avoid
investing heavily in a direction the maintainer has already explored or ruled out.

### Mechanical transforms on existing corpora to produce approximate clones

Given a confirmed clone pair (e.g. from BCB or hand-written), apply programmatic
(tree-sitter) transformations to generate Type II and III variants:

- Insert/remove blank lines and comments
- Rename local variables and parameters
- Add no-op statements (logging, assertions)
- Restructure equivalent loop forms

This gives controllable test cases where the expected similarity impact of each
transform is known in advance, making it easier to reason about regression.

### LLM-assisted cross-language translation

Take Java method pairs from BCB and use an LLM to translate both into Python
and TypeScript. This would give multi-language clone pairs with known ground
truth, directly addressing the Java-only limitation. Different LLMs or prompt
seeding or temperature settings could generate Type II/III variants naturally.

This is novel relative to published benchmarks and would be a meaningful
contribution to the test infrastructure.

### Same-prompt LLM generation (semantic clones)

Given a natural language description (from CodeSearchNet, CONCODE, or written
directly), generate multiple implementations using different LLMs or prompting
strategies. The resulting functions are semantic clones (Type IV) that treepeat
is not expected to detect — but measuring how often it does (or does not) would
characterize its Type IV behavior boundary.

### Existing multi-language datasets

**CodeSearchNet** — (docstring, code) pairs in Python, Java, JavaScript, PHP,
Ruby, Go. Not labeled for clones, but could be mined for near-duplicates or
used as a filler/negative corpus.

**POJ-104** — 104 competitive programming problems × ~500 C/C++ student
solutions each. Same-problem solutions are semantic clones. Useful for
characterizing Type IV behavior. Filtering may reveal type III clones.

Question for maintainer: Are any of these directions ones you've already explored
or ruled out? Any other benchmarks or evaluation strategies you'd suggest?

## Open Questions: Observability and Output Format

### Should timing and memory reporting be in the product?

The external performance harness (`tools/perf/`) wraps `/usr/bin/time` and
parses log output to reconstruct per-stage timing and peak RSS. This works for
benchmarking but means that field reports — a user sharing their SARIF output
when something looks wrong — arrive without any performance context. There is
no way to know whether a surprising result came from a slow run that may have
been interrupted, a memory-pressured run that may have degraded, or normal
execution.

Baking basic observability into treepeat itself would mean:
- Per-stage elapsed time already partially present as `(Xs)` suffixes in log
  lines; making this machine-readable and always-on rather than log-level
  dependent
- Peak RSS self-reported via `resource.getrusage(RUSAGE_SELF)` at pipeline end
  — avoids the `/usr/bin/time` wrapper requirement entirely
- Region counts per stage already logged at INFO; a structured summary line
  would make these parseable without regex

Question for maintainer: Is this the kind of instrumentation you'd want baked
into the product, or does it belong in external tooling?

### Should this information live in the SARIF output?

SARIF has first-class support for execution metadata via the `invocations`
array (`startTimeUtc`, `endTimeUtc`, `executionSuccessful`,
`toolExecutionNotifications`). Beyond that, `run.properties` accepts arbitrary
key-value pairs for tool-specific data.

The case for embedding timing and memory in SARIF: a shared SARIF file becomes
a complete field report. Someone filing a bug can attach one file and it
contains both the detection results and the execution context needed to
reproduce or diagnose the issue.

The case against: SARIF output is consumed by IDEs, CI tools, and downstream
pipelines that may not expect or want performance metadata. Keeping it in a
separate structured log avoids polluting the primary output format.

A middle path: emit a lean `run.properties` block with a small fixed set of
keys (elapsed time, peak RSS, region counts).

### Should there be a spec for treepeat-specific SARIF metadata?

Treepeat already emits custom properties on individual results (e.g.
`similarity`, `similarityPercent`, `groupSize`, `regions`). There is currently
no published spec for what these fields mean, what values they can take, or
which are stable across versions.

A short spec document — even just a table of field names, types, and semantics
— would:
- Allow external tooling to consume treepeat SARIF reliably
- Make it possible to write validation tests against the output schema
- Provide a forcing function for deciding which fields are stable API vs.
  internal detail

Question for maintainer: Is a SARIF metadata spec something you'd find valuable,
and if so, where should it live (inline in the SARIF output as `$schema`,
as a separate doc, or as a JSON Schema file)?

## Relationship to Performance Work

Three performance optimizations are ready but not yet submitted as PRs. Two of
them touch detection semantics and the sequencing question — land them before or
after establishing a correctness baseline — is relevant here. Details and
measured numbers are in [optimization-experiments.md](optimization-experiments.md).

The short version:

| Optimization | Speed impact | Correctness risk |
|---|---|---|
| Rules engine pre-partition | Large | None — land anytime |
| LCS → difflib swap | nestjs 136s → 3.7s | Subtle algorithm change |
| LSH threshold reduction | Django: timeout → 188s | Recall may decrease |
