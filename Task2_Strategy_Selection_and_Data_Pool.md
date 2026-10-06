# Task 2 — Filtering Strategy and Data Pool

**Project:** Evaluating Data-Quality Filtering Strategies for Bengali Instruction Tuning
**Inputs:** PROJECT_BRIEF.md §4 (taxonomy) and §7 (candidate data sources), plus the extraction findings in Task 1.
**Aligned with:** the submitted write-up, [bengali_filtering_tasks_1-4_humanized.md](bengali_filtering_tasks_1-4_humanized.md) (Task 2).

---

## Part A — Data Pool (as submitted)

**Pool: TigerLLM's Bangla-Instruct** (Raihan & Zampieri, 2025): 100,000 native Bengali instruction-response pairs generated through a self-instruct pipeline (500 seed tasks, GPT-4o and Claude-3.5-Sonnet as teacher models).

Every pair in the released set has already passed the authors' multi-stage filter, which checks four things:

| TigerLLM check | What it does |
|---|---|
| Language adherence | Bengali script and word-ratio checks, grammar scoring |
| Cultural sensitivity | Screens for culturally inappropriate content |
| Content quality | Response coherence and factual checks |
| Novelty | Similarity-based deduplication and lexical-diversity checks |

**Decision on the baseline question.** An earlier draft flagged a choice: use the unfiltered BanglaLlama pools (Bangla-Orca / Bangla-Alpaca) as V0, or use Bangla-Instruct even though it is already filtered. The submission takes the second option. Bangla-Instruct is the pool, and TigerLLM's own release (all 100K) is the baseline (Task 3, Arm A). The question is therefore: **does score-based re-selection improve on an already-filtered pool?**, not "does filtering beat no filtering?" BanglaLlama's pools are no longer used for training. They stay in Task 1 only as background on machine-translated Bengali data.

---

## Part B — How the Taxonomy Maps onto TigerLLM's Filter (as submitted)

Three of TigerLLM's four checks already cover three steps from our taxonomy:

| Our taxonomy step | Covered by TigerLLM check | Status in our pipeline |
|---|---|---|
| Semantic deduplication (A3) | Novelty | Already applied, kept as a pre-filter |
| Script and translationese filtering (B1/B2/B5) | Language adherence | Already applied, kept as a pre-filter |
| LLM-as-judge quality scoring, response (C1) | Content quality | Already applied as a binary pass/fail. We re-score it as a graded *q* (below) |

What TigerLLM's pipeline **doesn't** do is score instruction complexity separately from response quality, or apply an explicit diversity-aware selection step over a combined score. That's the gap this project's filtering strategy fills.

---

## Part C — Filtering Strategy (as submitted)

The strategy borrows DEITA's (Liu et al., 2024) score-first, diversity-aware framework:

1. **Complexity scoring.** An LLM-as-judge complexity score *c(i)* for each instruction, following DEITA's Evol-Complexity approach.
2. **Combined scoring.** Score *s = c × q*, where *q* is the response-quality score. This is DEITA's published formula (complexity × quality), not "quality squared." Squaring quality alone would drop the complexity signal entirely, which contradicts the task note's instruction to use quality and complexity together. *(Flagged in the submission in case a quality-only, squared score was actually what was intended. This needs supervisor confirmation.)*
3. **Diversity-aware selection.** Rank the pool by *s*, then greedily add pairs whose embedding distance to every already-selected pair exceeds a threshold *τ*, continuing until the target subset size is reached (40K in Task 3). This is DEITA's diversity step.

**Result:** a re-ranked, re-selected subset of Bangla-Instruct, chosen by complexity × quality plus diversity. It replaces TigerLLM's original approach of keeping everything that passes the binary filter. The semantic-dedup and script/translationese stages TigerLLM already applied stay in effect as a pre-filter on the pool itself.

**How this differs from DEITA itself:** DEITA trains separate LLaMA-7B scorer models for complexity and quality. We score both with **LLM-as-judge prompts** directly. That removes the cost that ruled out C2 in the earlier draft (see Part D), at the price of depending on how reliable the judge is in Bengali, which the Task 4 pilot checks first.

---

## Part D — Status of Every Candidate Strategy in the Taxonomy

Each candidate was originally scored on evidence strength, implementation cost, independence, and Bengali relevance (High/Medium/Low). The table below keeps those scores where they still hold and records each strategy's status in the submitted design. An earlier draft recommended a stacked A3 → B2/B5 → C1 pipeline on BanglaLlama's unfiltered data. That plan no longer applies: on Bangla-Instruct, TigerLLM's filter already covers A3, B, and C1.

| Code | Strategy | Evidence strength | Implementation cost | Bengali relevance | Status in submitted design |
|---|---|---|---|---|---|
| A1 | Exact / hash-based duplicate removal | High | Low | Low | Basic hygiene. Covered by TigerLLM's novelty check |
| A2 | Near-duplicate removal (MinHash+LSH, n-gram) | Medium | Medium | Low–Medium | Covered by TigerLLM's novelty check (lexical-diversity) |
| A3 | Semantic deduplication (embedding clustering) | Medium | Low | Medium | **Pre-filter, already applied** (TigerLLM novelty check) |
| B1 | Language identification / script purity | Low direct, best-documented Bengali problem | Low | High | **Pre-filter, already applied** (TigerLLM language adherence) |
| B2 | Unicode normalization / malformed-text removal | Low direct | Low | High | **Pre-filter, already applied** (TigerLLM language adherence) |
| B3 | Perplexity filtering against a Bengali reference LM | Low | High (no reliable Bengali reference LM) | Medium | Dropped. No suitable reference LM, and it would add a domain confound |
| B4 | Heuristic quality rules (length, symbols, repetition) | Medium | Low | Low | Basic hygiene. Implicit in TigerLLM's pipeline |
| B5 | Translationese / MT-artifact detection | Low direct, most-repeated Bengali concern | Medium | High | **Pre-filter, already applied** (TigerLLM language adherence). Lower risk anyway, since Bangla-Instruct is natively generated, not translated |
| C1 | LLM-as-judge response-quality scoring | High (AlpaGasus, DEITA) | Medium (API cost) | Medium (judge reliability in Bengali unverified) | **Selected as the *q* component.** TigerLLM's binary check is already applied. We re-score *q* on a graded scale. Arm D (quality-only) tests it alone |
| C2 | Complexity + quality scoring (DEITA-style) | High on English | Medium. LLM-judge prompts replace DEITA's trained scorers | Medium | **Selected, core of the strategy** (*s = c × q*). Earlier rejected as too costly because it needed trained scorers. That objection no longer applies |
| C3 | Instruction-following difficulty / loss-based | Medium | Medium–High | Low | Not selected |
| C4 | Reward-model scoring | Low | High (no Bengali reward model) | Low | Dropped |
| C5 | Response-groundedness / alignment checks | Low | Medium | Low | Not selected. Partly covered by TigerLLM's factual checks |
| C6 | Template-artifact / refusal / degenerate removal | Medium | Low | Low | Basic hygiene |
| D1 | Embedding-space coverage maximization | Low for Bengali (DEITA on English) | Medium | Medium | **Selected as the diversity step** (greedy selection with threshold *τ*). Arm E (no diversity) tests whether it helps |
| D2 | Task-type / domain balancing | Medium ("Data Diversity Matters", partial read) | Medium | Low–Medium | Not selected. Worth checking post hoc, given AlpaGasus's category-collapse risk (see Task 3 threats) |

**In short:** our experimental variables are complexity (C2), quality (C1), and diversity (D1). Deduplication and language quality (A3, B) are held fixed as TigerLLM's pre-filter.

---

## Part E — License / Provenance Items to Verify Before Training

These must be checked against the actual files and dataset cards, not taken from memory (PROJECT_BRIEF §7):

- [ ] Bangla-Instruct Hugging Face license terms (TigerLLM / `md-nishat-008` collection dataset card).
- [ ] Whether Bangla-Instruct pairs are tagged by teacher model (GPT-4o vs. Claude-3.5-Sonnet). This matters for the judge-circularity threat in Task 3 if our judge is from either family.
- [ ] Row count and fields of the downloadable release. Confirm it is actually 100K before fixing the 40K subset size, and check whether TigerLLM's filter scores are included or only the pass/fail outcome.
- [ ] Terms of use for whichever judge-model API scores *c* and *q* (output usage for training-data selection).
- [ ] License of the embedding model chosen for the diversity step.

No longer needed: Bangla-Orca / Bangla-Alpaca and CulturaX license checks, since neither is used for training in the submitted design.
