# Resume Screening Search: When Vector Search Beats SQL LIKE Matching

Short answer: use exact lexical matching for mandatory terms, semantic retrieval for varied descriptions of the same capability, and a hybrid path when missed evidence is costlier than a second retrieval pass. For an education company answering screening questions over a folder of resume PDFs, neither vector search nor SQL `LIKE` returns universally better matches. The right choice follows the question: `LIKE` is precise for a literal license or course code; vectors are better candidates for meaning expressed with different words. Final eligibility must remain deterministic and traceable to the source PDF.

This architecture decision record adopts hybrid retrieval for exploratory candidate discovery, with structured filters applied before retrieval and quoted evidence required before a result reaches a reviewer. The decision favors retrieval quality, but it gives latency a hard boundary: lexical and semantic searches run concurrently, fusion is bounded, and generation is outside the eligibility decision.

## Should resume screening search use vector search or SQL LIKE?

Three invariants control the design. First, a required credential, work authorization value, location constraint, or date range is a structured field or an exact evidence check; semantic similarity cannot prove it. Second, every returned match carries a document identifier, page number, extraction version, and text span so a reviewer can inspect the original PDF. Third, access control is evaluated before any resume text is retrieved. A fluent answer without those boundaries is a deliverability failure in another costume: the response arrived, but it reached the wrong destination or cannot be trusted.

The failure boundaries matter more than the index brand. PDF extraction may scramble columns, headers may be repeated on every page, acronyms may be ambiguous, and an embedding can place neighboring concepts close together without establishing that a candidate meets a requirement. Literal search fails differently. `LIKE '%learning management system%'` misses `LMS administration`, while `LIKE '%C++%'` can be useful because the punctuation and exact token carry meaning.

Keep abstention explicit. If no passage clears the retrieval and evidence thresholds, return no supported match rather than letting an answer model fill the gap.

Exactness wins sometimes.

## Decision table

| Path | Strongest use | Predictable miss | Latency shape | Evidence policy |
|---|---|---|---|---|
| SQL `LIKE` | Exact phrases, identifiers, uncommon credential names | Synonyms, paraphrases, reordered terms | One lexical query; leading wildcards may prevent efficient prefix lookup | Return the matched span |
| Vector similarity | Duties and skills described in varied language | Exact constraints, negation, numeric requirements | Embedding plus nearest-neighbor retrieval | Recheck the source span |
| Hybrid fusion | Mixed screening questions | Bad extraction or missing metadata can hurt both paths | Two bounded searches plus fusion | Require provenance from either path |

The table is a routing policy, not a leaderboard. Query classes should be labeled before tuning: exact constraint, concept discovery, or mixed. That label also makes evaluation legible to compliance reviewers because the system can explain why a search path ran.

The main limitation of the hybrid choice is operational: two indexes can drift, two branches can fail independently, and fusion adds work to every mixed query. A folder dominated by standardized application forms may not need that complexity; structured columns plus literal search are easier to audit and often sufficient. A collection of narrative resumes has the opposite trade-off because equivalent experience can be worded dozens of ways. This is why the routing label belongs beside the query rather than inside a global feature flag. It lets the team reserve the slower path for questions that can benefit from semantic recall, while a page-level evidence requirement contains the risk that a plausible neighbor is mistaken for a qualification.

Latency counts.

## Critical path in Python

The critical path below uses generic interfaces. It filters the authorized document set first, runs both retrieval methods concurrently, fuses ranks without pretending their raw scores share a scale, and returns evidence rather than an automated hiring verdict. Reciprocal rank fusion uses positions, which avoids comparing a lexical score directly with a vector distance. The constant is a tuning input and must be validated on the local judgment set.

```python
from concurrent.futures import ThreadPoolExecutor
from dataclasses import dataclass


@dataclass(frozen=True)
class Hit:
    chunk_id: str
    document_id: str
    page: int
    text: str


def reciprocal_rank_fusion(result_lists: list[list[Hit]], rank_constant: int = 60) -> list[Hit]:
    scores: dict[str, float] = {}
    evidence: dict[str, Hit] = {}
    for results in result_lists:
        for rank, hit in enumerate(results, start=1):
            scores[hit.chunk_id] = scores.get(hit.chunk_id, 0.0) + 1.0 / (rank_constant + rank)
            evidence[hit.chunk_id] = hit
    ordered_ids = sorted(scores, key=scores.get, reverse=True)
    return [evidence[chunk_id] for chunk_id in ordered_ids]


def retrieve(query: str, authorized_document_ids: set[str], limit: int = 20) -> list[Hit]:
    if not authorized_document_ids:
        return []

    with ThreadPoolExecutor(max_workers=2) as pool:
        lexical_job = pool.submit(lexical_search, query, authorized_document_ids, limit)
        semantic_job = pool.submit(semantic_search, query, authorized_document_ids, limit)
        lexical_hits = lexical_job.result()
        semantic_hits = semantic_job.result()

    fused = reciprocal_rank_fusion([lexical_hits, semantic_hits])
    return fused[:limit]
```

Production code also needs per-branch deadlines, cancellation, bounded input sizes, and a declared partial-result policy. If semantic retrieval times out, an exact-match result may still be useful, but the response must say that semantic recall was unavailable. Do not silently present a degraded path as a complete search. Log query class, branch latency, result identifiers, extraction version, and reviewer-visible reason codes; avoid logging raw resume text or the original free-form query unless retention and access policies explicitly permit it.

## How should better matches be measured?

Build a judgment set from real screening questions with sensitive values removed. Each item needs relevant passages, not merely a relevant resume, because a document-level label can hide retrieval that landed on the wrong page. Include acronym expansions, paraphrased job duties, exact certifications, negated experience, date constraints, OCR damage, and resumes with multi-column layouts. Split tuning questions from the final evaluation set.

Measure recall at the evidence cutoff and precision among passages shown to reviewers. Add p50 and p95 retrieval latency for each query class, plus the rate of searches that abstain or degrade to one branch. No invented universal threshold belongs here. Select cutoffs from the consequence of a false negative versus the review burden of a false positive, then record that choice in the decision log.

A small slice can mislead. Report results by query class and PDF extraction quality, and compare lexical, semantic, and hybrid paths on the same judgments. Reviewer disagreement should be retained rather than averaged away; it identifies ambiguous requirements that retrieval tuning cannot settle.

Operationally, index a new extraction version beside the active one, validate it, then switch an alias or version pointer. Roll back the pointer if evidence recall or latency breaches the agreed budget. Deletion must remove the document from every retrieval path and invalidate cached answers. These controls keep a candidate who withdrew consent from resurfacing through an older semantic index.

## Rejected option and where it still belongs

The rejected default is vector-only screening. It broadens recall for paraphrases, but it makes exact gates too easy to blur and creates unnecessary embedding work for literal identifiers. It remains valid for a separately labeled discovery feature where reviewers are looking for adjacent experience, every result exposes its supporting passage, and nobody treats similarity as proof of eligibility.

`LIKE`-only search is also rejected as the general path because resume language varies too much. It still belongs in the system. Use it for exact strings, for a diagnostic fallback when semantic retrieval is unavailable, and for smoke tests that confirm newly indexed text is actually searchable.

The resulting rule is deliberately plain: structure hard constraints, retrieve soft evidence through both lexical and semantic paths, fuse bounded result sets, and make a human inspect the cited passage. Better matches are the ones that survive that evidence chain within the latency budget, not the ones with the most persuasive similarity score.

## References

- https://arxiv.org/abs/2005.11401
