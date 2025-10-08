# Autokey Unknown-Alphabet Brute-Forcer Optimizations

This note explains why the current worker occasionally falls back to an expensive search and outlines concrete optimizations we can pursue.

## Why the solver sometimes performs a full search

When we test a candidate crib in the `autokey_custom_alpha` mode we have to recover **both**:

1. the permutation that maps plaintext letters to ciphertext letters, and
2. the initial autokey seed of length *m*.

The helper `solveAutokeyUnknownAlphabet` starts by consuming the fixed crib assignments and building a mixed list of constraints that relate plaintext symbols, ciphertext symbols, and the key residues (`r = step mod m`).【F:index.html†L1047-L1084】 After deterministic propagation has finished, the solver uses a full depth-first search (`dfs`) to assign the remaining residues and plaintext symbols while preserving the bijective alphabet mapping.【F:index.html†L1118-L1247】 If propagation cannot fix all variables (because the crib coverage is sparse or leaves mutually unconstrained components), the DFS must explore alternative assignments until it finds one that satisfies every constraint. That backtracking work is what causes the long “stuck” periods.

## Existing pruning steps

Before the DFS runs, the worker performs two lighter-weight filters:

- During the quick trial phase we run the same solver with `propagateOnly` enabled, which performs deterministic propagation followed by a small number of randomized micro-searches (bounded by `quickTrials` and `trialDepth`).【F:index.html†L1007-L1096】 Words that fail this phase are rejected cheaply.
- When the quick trial succeeds we perform the full solve once per plausible key length and, if successful, decrypt and score the result.【F:index.html†L2463-L2505】

The quick trial catches many impossible words, but when it returns “maybe” we still need the exhaustive search to guarantee correctness.

## Optimization directions

Several improvements can reduce how often we reach the expensive DFS or how much work it has to do:

1. **Stronger deterministic propagation.** We currently derive residue candidates from pairwise differences inside the quick trial, but the full solver only enforces arc consistency through repeated scans. Implementing a constraint queue (e.g. AC-3 style) or tracking dependency graph components would push more consequences without branching, shrinking the search space before DFS begins.【F:index.html†L1001-L1045】【F:index.html†L1085-L1096】
2. **Reusable base state per selection.** For every candidate word we rebuild `collectContiguousCribs`, recompute `stepPlain`, and recreate the constraint arrays from scratch.【F:index.html†L2467-L2486】 We can cache the “static” constraints implied by the fixed cribs once, then clone and extend them with the word-specific assignments, avoiding repeated map/set construction.
3. **Residue ordering informed by coverage.** The DFS already picks the residue with the highest constraint count, but we can feed it better heuristics, e.g. preferring residues that participate in the longest contiguous runs or have feed dependencies (`depSym`). That tends to lock the key faster and reduces branching.【F:index.html†L1098-L1116】【F:index.html†L1169-L1211】
4. **Incremental alphabet pruning.** Because the permutation is bijective, as soon as a ciphertext symbol is assigned we can eliminate that value from all other plaintext symbols. Maintaining explicit candidate sets per letter (instead of recomputing `used`) enables forward checking so the DFS can abort earlier when a branch exhausts a symbol’s domain.
5. **Search restarts guided by diagnostics.** The helper `autokeyCoverageDiagnostics` already reports residue coverage and missing feed links.【F:index.html†L1315-L1348】 We can expose this data in the UI and use it to auto-adjust `quickTrials` or refuse to start the worker if the coverage is so sparse that DFS would explode, prompting the user to place additional cribs.
6. **Memoizing failed partial assignments.** When DFS backtracks it often revisits identical `(pos, k)` patterns produced in different orders. Hashing the assignments for connected components and caching the fact that they lead to contradictions would prune symmetric branches.

## Takeaways

The brute-forcer is correct but occasionally slow because the underlying constraint problem genuinely requires search when crib coverage is thin. By pushing more work into deterministic propagation, caching shared structures, and guiding the DFS with richer heuristics we can significantly reduce those worst-case stalls without changing the solver’s guarantees.
