# Performance Optimization Experiments

Date: 2026-04-16

## Status

Draft — changes not yet submitted as PRs

## Summary

Three optimizations were identified by profiling treepeat against five real
repositories (requests, flask, redux-toolkit, nestjs, django) using cProfile
and a purpose-built performance harness. The primary finding was that the
pipeline was **time-bounded, not memory-bounded** — an earlier hypothesis about
memory pressure and swapping was not borne out (django ran at ~5.5 GB RSS on a
32 GB machine with zero swap events).

| Optimization | File | Speed impact | Correctness risk |
|---|---|---|---|
| Rules engine pre-partition | `rules/engine.py` | Large (unquantified) | None |
| LCS → difflib swap | `verification.py` | 37× on nestjs | Subtle — see below |
| LSH threshold reduction | `lsh_stage.py` | Django: timeout → 188s | Recall may decrease |

The no-risk optimization (rules engine) could land independently at any time.
The other two are candidates for validation against a correctness benchmark
before merging. See [correctness-and-benchmarking.md](correctness-and-benchmarking.md)
for the broader context.

---

## Optimization 1: Rules engine language pre-partitioning

**File:** `treepeat/pipeline/rules/engine.py` — `_iter_matching_rules`

**Problem:** `_iter_matching_rules` iterates over all rules on every AST node
traversal and calls `rule.matches_language(language)` on each one. With 6.2
million node traversals in django, this produced 632 million redundant
`matches_language` calls costing 87s. The shingling stage as a whole was 380s
of 462s total.

**Fix:** Build a `dict[str, list[Rule]]` at `RuleEngine.__init__` time keyed
by language. `_iter_matching_rules(language)` becomes an O(1) dict lookup
returning a pre-filtered list, eliminating all per-node language checks.

```python
# in __init__, after self.rules = rules:
self._rules_by_language: dict[str, list[Rule]] = {}
for rule in rules:
    for lang in rule.languages:
        self._rules_by_language.setdefault(lang, []).append(rule)
```

**Correctness risk:** None. Same rules, same application order, looked up
differently.

**SARIF verification:** Not needed given zero semantic change.

---

## Optimization 2: Verification algorithm swap

**File:** `treepeat/pipeline/verification.py` — `_compute_lcs_length`,
`_compute_ordered_similarity`, `_compute_source_similarity`

**Problem:** `_compute_lcs_length` is a pure-Python O(m×n) DP table over
shingle lists. On nestjs (336 regions, 1801 pairs): 273s tottime + 58s in 663M
`builtins.max` calls = ~98% of total runtime.

**Fix:** Replace with `difflib.SequenceMatcher(None, s1, s2).ratio()` —
C-implemented, already used elsewhere in the codebase. The `_compute_lcs_length`
and `_normalize_similarity` functions are removed entirely.

**Measured impact:** nestjs 136s → 3.7s (37×). Peak RSS 317 MiB → 185 MiB
(full DP table allocation gone).

**Correctness risk — the subtle part:** LCS finds the longest common
*subsequence* (non-contiguous); `SequenceMatcher` finds matching *blocks*
(contiguous runs) and computes `2*M/T` as its ratio. These are mathematically
different similarity measures. The ratio formula matches the existing
normalization (`lcs / avg_length`) in many cases but diverges on inputs where
non-contiguous matches matter.

It is arguable that contiguous block matching is actually *more* semantically
appropriate for code similarity — but that is a correctness decision, not just
an optimization.

**SARIF verification:** Output verified identical on all five test repos
(requests, flask, redux-toolkit, nestjs, django). Score-level differences on
edge cases cannot be ruled out without a labeled benchmark.

---

## Optimization 3: LSH threshold reduction

**File:** `treepeat/pipeline/lsh_stage.py` — `_create_lsh_index`

**Problem:** The LSH stage was passing too many candidate pairs to the
verification stage. Before other fixes, verification was the bottleneck; after
Optimization 2, LSH candidate volume became the next constraint for large repos.

**Fix:** The `lsh_similarity_percent` used for candidate generation was reduced:

```python
lsh_similarity_percent = min(0.5, 0.7 * similarity_percent)
```

This reduces the number of candidates that reach pairwise verification.

**Measured impact:** Django: timeout at 30 minutes → completes in ~188s. SARIF
output verified identical on all five test repos.

**Correctness risk:** Reducing the LSH threshold reduces recall — some genuine
clone pairs that would have been found as candidates may no longer reach
verification. The degree of recall loss is not yet measured against a labeled
benchmark. This is the most significant correctness concern of the three.

**SARIF verification:** Output verified identical on five repos. These are not
a labeled benchmark, so absence of difference does not guarantee no recall loss
on other inputs.

---

## Open questions

**Sequencing:** Should the rules engine change land first (no risk), with the
other two following after a correctness baseline is established? Or is the SARIF
parity evidence sufficient to land all three together?

**The difflib semantic question:** Is LCS→matching-blocks a correctness concern
requiring benchmark validation, or a defensible algorithmic substitution?

**Recall measurement for the LSH change:** The only way to know the recall
impact is to run against a labeled benchmark. The BCB toolchain in
`tools/correctness/` could be built toward this goal.

**Memory at scale:** Even after the time fixes, django's 5.5 GB RSS is notable.
The ~200 KB/region scaling is linear with shingled region count and driven
primarily by shingle string retention. This is not an immediate problem on
developer hardware but is relevant for CI environments or very large repos.
Interning or hashing shingle content rather than storing repeated strings is a
candidate future optimization with no correctness risk.

