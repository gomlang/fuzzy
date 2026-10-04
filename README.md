# fuzzy

Unicode-aware ranked subsequence matching in native GoML, inspired by
[nucleo](https://github.com/helix-editor/nucleo). The score is this library's
documented policy, not a claim of numerical compatibility with nucleo or fzf.

```goml
use ecosystem::fuzzy;
use std::context;

fn files(query: string, candidates: Vec[string]) -> Result[fuzzy::Report, fuzzy::Error] {
    let matcher = fuzzy::Matcher::new(fuzzy::Options::new())?;
    matcher.search_parallel(query, candidates, 20, 4, context::Context::background())
}
```

## Matching and offsets

`Matcher::find` performs fuzzy matching. `find_with` also accepts `Substring`,
`Prefix`, `Suffix`, and `Exact` plus a cancellation context. `pattern` and
`prepare` cache pattern normalization and candidate segmentation for repeated
`match_prepared` calls. Prepared values are immutable and reusable concurrently.
They may cross matcher instances when preprocessing policies agree: a candidate
requires the same `path_bonus`, and a pattern requires the same `CaseMode`.
Candidates retain both case representations, so their originating case mode does
not restrict reuse; patterns are independent of path scoring. Incompatible
preparation returns `IncompatiblePreparation`. Budget settings may differ: the
receiving matcher rechecks original UTF-8 byte counts, both candidate unit counts
and the normalized pattern unit count before any empty/impossible-match shortcut.
Reprepare the candidate or pattern when changing its relevant matching policy.

`CaseMode::Sensitive` compares original scalars; `Fold` uses standard Unicode
full case folding; `Smart` folds unless the pattern contains an uppercase scalar.
An expansion such as `ß` to `ss` retains its original grapheme location. Returned
`Match.ranges` are merged half-open UTF-8 byte ranges covering complete extended
graphemes; `graphemes` contains unique zero-based grapheme indices. Combining marks
and emoji sequences are never partially highlighted. Matching itself compares
scalars and can select a scalar inside a grapheme. Canonical normalization,
accent stripping, transliteration, and locale-specific folding are not applied.
Segmentation uses `unicode_text` Unicode 16; case folding/classification use the
standard library's Unicode version.

The dynamic program finds the maximum-score subsequence. Each matched scalar
earns 16 points, with 12 at the input start, 14 after a path separator when enabled,
10 after other non-alphanumeric scalars, or 8 at a lowercase-to-uppercase boundary.
Consecutive matches gain 8. Leading skipped scalars cost 1 each; internal gaps
cost 3 plus their length; trailing skips cost their length divided by four.
Boundary rewards are applied once per original grapheme. Equal scores prefer the
earliest final position, and stable candidate ties preserve input order.

## Search and incremental queries

`search` and `search_parallel` return bounded Top-K hits, total matching candidate
count and scanned count. Limit zero still counts all matches. Parallel workers
produce the same ranking, indices, scores and highlights as serial execution.
Input vectors are copied before parallel work; callers must not concurrently
mutate a vector while the copy is being taken. There are at most 256 workers.

`Session::new(matcher, values)` caches candidate preprocessing. `update(query,
limit, context)` returns `(next_session, report)`; old sessions remain usable.
When a query extends its predecessor with unchanged case policy, only previously
matching candidates are scanned. All eligible candidates are retained regardless
of the previous Top-K limit. Backspacing, replacement and smart-case transitions
rescan the index. Sessions have no mutable shared cache and can branch or be
queried concurrently.
The matrix admission budget applies only to candidates actually scanned. A
narrowed update can therefore succeed where a fresh search returns `WorkLimit`
for a candidate already excluded by the previous query.

## Bounds and errors

Options validate input byte/unit counts, pattern size, matrix cells and result
count. The defaults are 1 MiB per string, 32,768 normalized candidate scalars,
256 pattern scalars, 2,097,152 DP cells and 10,000 results. Fuzzy matching uses O(MN)
time and parent storage, plus two O(N) score rows. Parent indices use four bytes
per cell: the supported maximum of 1,048,576 candidate units fits in a signed
32-bit index. Scores retain their full integer width. Top-K uses a bounded heap,
with O(log K) insertion cost. Sessions retain O(total candidate size) preprocessing.
Matrix budget exhaustion is an error, never a silently degraded match.
Prefix, suffix and exact matches evaluate their fixed alignment directly after
the same feasibility and work-budget checks. They avoid the scoring matrix and
retain only O(M) highlight metadata; scoring and byte/grapheme ranges are unchanged.
Substring matching scans all overlapping occurrences using a pattern prefix
table and a rolling boundary score in O(M + N) time and O(M) auxiliary space.
It retains the highest score and earliest ending position on ties. The same
normalized matrix-cell budget remains an admission check for every match mode.

Cancellation/deadlines are checked before matching, each row, every 1,024 columns
and between candidates. Segmentation and case folding of one bounded candidate
are synchronous. `Session::new_with` accepts a context and checks it between
candidates; `new` uses a background context. Searches check the context before
preparing each candidate. A failed update does not modify
the previous session. The scoped parallel search joins every worker on failure.

## Validation

`(cd ../verification && just ecosystem-test fuzzy)` checks the library and example, repeated builds
and race detection. Native tests cover scores, anchors, Unicode expansions and
graphemes, all budgets, stable Top-K, persistent incremental search, cancellation,
cross-matcher preparation compatibility and budget validation,
parallel equivalence, and 1,270 exhaustive short-input comparisons against an
independent recursive subsequence enumerator. The example demonstrates file
search and query refinement. No Python or native adapter is required.
Anchored matching adds 3,810 fixed-position oracle comparisons and Unicode
folding/grapheme cases. `goml test prepared_anchor_scaling --ignored --nocapture`
runs an optional benchmark of prepared prefix and suffix matches; it has no
timing assertions.
Substring matching adds 1,270 comparisons against exhaustive matching windows,
plus overlapping Unicode folds, path bonuses and score ties.
`goml test prepared_substring_scaling --ignored --nocapture` measures repeated
substring searches over prepared input without timing assertions.

## Development and examples

Requires GoML 0.1.56 or newer. The `examples/basic/` example shares the root manifest. From the library root, run:

```sh
goml run --example basic
goml test
goml verify --timeout 300s
```

`goml test` builds the example and runs its tests. `goml verify` repeats the example checks as an independent module against an isolated registry snapshot. `(cd ../verification && just ecosystem-test fuzzy)` also retains the library-specific smoke and compatibility checks.

Before allocating its scoring matrix, matching performs a linear subsequence
feasibility scan over the prepared case-sensitive or folded units. Impossible
candidates return `None` even when their potential matrix exceeds `max_cells`.
Candidates passing that scan still obey the matrix budget and retain the same
scores and tie rules. The scan checks cancellation every 1,024 units.

Top-K retention uses a bounded worst-first heap: each admitted/replaced hit takes
O(log K) work instead of shifting O(K) entries. Only the final retained hits are
sorted, in O(K log K), by descending score then original candidate index. Equal
scores therefore keep stable input order. Zero-limit searches still count all
matches without retaining hits; limits above the corpus size return every match.
Parallel workers use the same bounded policy and their merge preserves serial
ordering. The change does not alter scoring or corpus preparation limits.

### Collection match modes

`search_with(query, values, kind, limit, context)` and
`search_parallel_with(query, values, kind, limit, workers, context)` accept every
`MatchKind`, using the same scoring, highlights, stable input-index ties, Top-K
limits and cancellation rules as single-candidate `find_with`. The existing
`search` and `search_parallel` remain fuzzy by default.

`Session.update_with(query, kind, limit, context)` selects the mode for a query
without preparing the corpus again. Persistent branches and failed updates leave
older sessions usable. A mode or smart-case change scans all candidates. With
unchanged mode and case policy, fuzzy/substring/prefix queries can reuse eligible
candidates when text is appended; suffix queries can reuse them when text is
prepended; exact queries can reuse them only when unchanged. Backspacing or other
changes rescan the corpus. `update` always selects fuzzy mode, including after an
`update_with` call. Limit zero retains the complete eligibility set.
