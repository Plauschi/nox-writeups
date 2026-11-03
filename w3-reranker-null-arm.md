# The Null Arm Won: Falsifying Our Own Reranker

*A case study from NOX, a personal multi-agent lab. Drafted by Claude; Hermes reviewed it against the raw artifacts before publication; the owner gated both the deploy and the publication. See [the lab page](lab.md) for who does what.*

## The problem

The lab's shared knowledge vault is searched by an embedding index. Most retrieval abilities measured fine; the persistent weak spot was **cross-document recall** — queries whose answer requires pieces from two or more files. Frozen baseline: cross-document recall@3 of **0.158** against an overall recall@3 of 0.464 (104 queries, ground truth and scoring harness frozen by hash).

The obvious diagnosis was coverage: the index must be missing content. We tested that first. Three candidate index extensions — together more than 33,900 additional chunks — moved cross-document recall@3 by **exactly zero** across five paired measurements. The defect was structural, a ranking problem, not a missing-content problem.

## The design

The next hypothesis was ranking diversity, and the standard tool for that is MMR (maximal marginal relevance): penalize candidates that are too similar to what is already selected. We preregistered it as the single treatment arm:

- **λ = 0.15, frozen one-shot.** No tuning, no retry. If the arm failed, the parameter was spent.
- **Pass/fail gates declared in advance** — primary: cross-document recall@3 and strict multi-hop successes must improve; negative gates: overall recall, first-rank recall on the strongest ability, and an abstention probe must not regress.
- **A paired measurement cell** — off → collapse → MMR → off — on one frozen snapshot, with the two off arms required to produce identical ranking digests (drift control).
- **A null arm**: *collapse*, a trivial dedup that keeps only the best chunk per source document. The preregistration said explicitly that a significant effect in this arm "would be a finding about the measurement chain, not a success of the arm."

## The runs that did not count

Two of three cell runs were discarded fail-closed before any result was read as evidence. In the first, the runner killed a wrapper process while the API it was supposed to replace kept serving — one arm measured the wrong process. The fix was structural: a port-free invariant and an **arm identity probe** (the API must report the mode it is supposed to be running before a single query is scored). In the second, a harness edit of ours had silently changed the semantics of the abstention probe inside the frozen baseline; the read-back caught it, the baseline was re-frozen with byte-identical ground-truth hashes, and the run went to the invalid pile with its artifacts kept as evidence.

## The result

**MMR failed.** Overall recall@3 dropped below its negative gate (0.464 → 0.440), the single natural multi-hop success in the baseline was displaced out of the top 3, no query improved, two got worse. Under the preregistration, that is the end of MMR here: λ was one-shot, and it stays spent.

**The null arm passed every gate.** Collapse — the control — lifted overall recall@3 from 0.464 to **0.500**, cross-document recall@3 from 0.158 to **0.211**, strict multi-hop successes from 1 to 2, and reduced zero-recall queries from 31 to 28, with no negative gate violated: first-rank recall on the strongest ability and the abstention probe stayed byte-identical, and overall recall moved only upward. Cells on two different index snapshots produced identical arm deltas — a de-facto replication.

## The mechanism (verified, not narrated)

The reviewer traced the flipped queries to their sources rather than accepting the story. The pattern: several of the vault's highest-traffic files are large single documents, and multiple chunks of the *same file* were crowding the top ranks, pushing genuinely different documents just below the cutoff. Collapsing per canonical source document frees those slots. Reproduced independently: before collapse, 23 of 104 queries had a duplicated source document in their top 3; after, zero.

An earlier simulation had predicted collapse would do nothing — and it was correct *on its own data*: it had used chunk-ID prefixes, not canonical source paths, as the document key. The negative prediction was not wrong; its key semantics were. That distinction only became visible because both the simulation and the cell kept their artifacts.

## Shipping the control, by the rules

A null arm that wins is not a result you deploy on enthusiasm. The review verdict was explicit: the two existing cells count as *replicated pre-declaration findings*, not as confirmation. So the design was amended — collapse re-declared as the single treatment arm, same gates, same frozen ground truth — and a **fresh post-declaration cell on a fresh index snapshot**, now with machine-readable receipts (effective mode probe, snapshot and code hashes, full rerank trace), had to pass before a deploy proposal reached the owner. One green cell was itself discarded on the way because its index snapshot hash matched the pre-declaration one — a replay, not a fresh throw. The cell that counted ran on a third, independent snapshot and passed every gate. Only then did the owner open the deploy gate, and the dedup went live with a one-line rollback path.

The reviewer also cut one claim down on the way: the draft report said the MMR penalty "destroyed" the lost multi-hop success, but the archived artifacts only proved *selection displacement*, not the quantitative penalty attribution — so the weaker sentence is the one that survived. The same standard applies to this page.

## What this is and is not

This is one private corpus, 104 queries, one lab. It says nothing general about MMR, which works elsewhere. What it is meant to show is a working falsification loop in a personal system: a preregistered parameter that stayed spent after failing, two runs thrown away by their own guards, a control arm that was not promoted to a success until it had been re-declared and re-confirmed under the same gates — and a feature of our own design deleted because the measurement said so.
