# Task 1 — Literature Screening, Extraction, and Research Gap

**Project:** Evaluating Data-Quality Filtering Strategies for Bengali Instruction Tuning
**Source:** NeuralSight Assignment 2 — Paper Tracker (14 papers currently in the knowledge base)
**Inclusion criteria used:** PROJECT_BRIEF.md, Section 3
**Aligned with:** the submitted write-up, [bengali_filtering_tasks_1-4_humanized.md](bengali_filtering_tasks_1-4_humanized.md) (Task 1). The submitted gap statement comes first. Parts A and B are the supporting literature work behind it. Parts C and D are revised to match the submitted design.

---

## Submitted Research Gap (as submitted)

Across recent work on English instruction tuning, filtering methods like DEITA (Liu et al., 2024), AlpaGasus (Chen et al., 2024), and large-scale selection studies such as Ivison et al. (2025) have shown that cutting a large instruction pool down to a smaller subset, selected for response quality, instruction complexity, and sample diversity, can match or beat training on the full, unfiltered pool. That result holds up for English. It hasn't been tested under controlled conditions for Bengali.

Bengali instruction data is a different setting altogether. It uses a different script (the Bengali abugida instead of Latin script), the NLP tooling for it, embeddings, scorers, tokenizers, is less mature, and the instruction corpora available, including community-built and machine-translated datasets, still lean heavily on machine translation. That dependency brings its own quality problems: translationese, script inconsistency, cultural mismatch, none of which have a clean English equivalent.

So whether the English filtering result actually transfers, whether filtering improves downstream model performance on Bengali instruction data rather than just shrinking the dataset, is still an open question. This project tests that directly, using a fixed base model and training pipeline so the data-selection method is the only thing that varies across conditions.

**Citation keys used in the submission:** DEITA = Liu et al., 2024 (#12 below); AlpaGasus = Chen et al., 2024 (#13); Ivison et al., 2025 = "Large-Scale Data Selection for Instruction Tuning" (#14); TigerLLM = Raihan & Zampieri, 2025 (#11).

---

## Part A — Screening

Screening decision for every paper currently logged in the tracker, checked against the include/exclude/unsure rules in PROJECT_BRIEF.md §3 (a data-selection or filtering method for instruction tuning; a quality-vs-quantity study; low-resource/Indic instruction tuning; Bengali resources or benchmarks; deduplication; translationese/MT artifacts; or a usable Bengali/multilingual benchmark).

| # | Paper | Logged by | Link | Decision | Reason |
|---|---|---|---|---|---|
| 1 | BanglaLlama: LLaMA for Bangla Language | Mukti | arxiv.org/abs/2410.21200 | **Include** | Releases the Bangla-Orca / Bangla-Alpaca instruction sets with no post-hoc quality filtering. It is the main example of the machine-translated Bengali instruction data the submitted gap describes. It is background only: the submitted design does not train on it. |
| 2 | Superfiltering: Weak-to-Strong Data Filtering for Fast Instruction-Tuning | Mukti | aclanthology.org/2024.acl-long.769 | **Include** | Proposes a filtering method for instruction-tuning data (weak-model-scores-strong-model-data); falls under Strategy family C (instruction–response quality filtering). |
| 3 | CoachLM: Automatic Instruction Revisions Improve the Data Quality in LLM Instruction Tuning | Hasan | ieeexplore.ieee.org/document/10597991 | **Include** | Improves instruction data quality via automatic revision rather than pure filtering — relevant comparison point for our C-family strategies, and for what "quality improvement" can mean beyond keep/discard. |
| 4 | Data Diversity Matters for Robust Instruction Tuning | Hasan | aclanthology.org/2024.findings-emnlp.195 | **Include** | Directly studies data diversity's effect on instruction-tuning robustness — grounds Strategy family D (diversity/coverage) and the composability question (RQ3). |
| 5 | An Instruction-Response Perspective on Large Language Models in Information Retrieval Tasks | Hasan | dl.acm.org/10.1145/3726302.3730346 | **Unsure** | Title suggests a downstream IR application of instruction-response formatting rather than a data-selection or quality method. PDF is accessible but not yet read in depth — needs a first pass before a final call; likely candidate for exclusion under "task-specific fine-tuning with no instruction-following component" if it turns out to be purely about IR ranking. |
| 6 | INSTRUCTEVAL: Towards Holistic Evaluation of Instruction-Tuned Large Language Models | Hasan | aclanthology.org/2024.scalellm-1.4 | **Unsure** | An evaluation-framework paper, not a filtering method, and (based on the title) English-focused. Per the brief's rule for "English-only but the method is clearly transferable," keep as Unsure — potentially useful for shaping our automatic-evaluation protocol in Section 6, not as a filtering strategy source. |
| 7 | Too late to train, too early to use? A study on necessity and viability of low-resource Bengali LLMs | Habibur Rahman | aclanthology.org/2025.coling-main.79 | **Include** | Directly addresses whether dedicated low-resource Bengali LLMs are still needed given cross-lingual transfer — core evidence for RQ2 (do English-developed interventions transfer to Bengali) and for motivating the thesis itself. |
| 8 | Evaluating LLMs' Multilingual Capabilities for Bengali: Benchmark Creation and Performance Analysis | Habibur Rahman | arxiv.org/abs/2507.23248 | **Include** | Builds a Bengali evaluation benchmark and documents a tokenization-efficiency-vs-accuracy relationship — directly usable in Section 6 (evaluation plan) and Section 8 (tokenizer fertility check for base-model selection). |
| 9 | TituLLMs: A Family of Bangla LLMs with Comprehensive Benchmarking | Habibur Rahman | aclanthology.org/2025.findings-acl.1279 | **Unsure** | PDF access not yet confirmed in the tracker (no accessibility status logged). Title indicates it is a Bengali-LLM resource/benchmarking paper, which would qualify it for Include once access and content are confirmed. |
| 10 | BanglaLlama: LLaMA for Bangla Language (published venue version) | Habibur Rahman | aclanthology.org/2026.loreslm-1.7 | **Unsure — duplicate/venue-version of #1** | Same paper as #1, logged twice under different links (arXiv preprint vs. LoResLM 2026 proceedings version). PDF access not yet confirmed for this version. Action: confirm whether this is the camera-ready version of #1 and merge the two tracker rows into one before the spreadsheet is finalized, citing the published version. |
| 11 | TigerLLM: A Family of Bangla Large Language Models | Shadman Kabir | arxiv.org/abs/2503.10995 | **Include** | The anchor for the whole submitted design: Bangla-Instruct (100K) is our data pool, and TigerLLM's base model (LLaMA-3.2 1B), fine-tuning recipe, and six-benchmark evaluation suite are held fixed (Task 3). |
| 12 | What Makes Good Data for Alignment? A Comprehensive Study of Automatic Data Selection in Instruction Tuning (DEITA) | Shadman Kabir | arxiv.org/pdf/2312.15685 | **Include** | Cited in the submitted gap (Liu et al., 2024). Our filtering strategy borrows its framework directly: complexity × quality score plus greedy diversity-aware selection (Task 2). |
| 13 | AlpaGasus: Training a Better Alpaca with Fewer Data | Shadman Kabir | arxiv.org/pdf/2307.08701 | **Include** | Cited in the submitted gap (Chen et al., 2024). Canonical LLM-as-judge single-score filter. Its size-matched random subset is the template for our Arm B random control, and its documented skill-category imbalance after filtering matters for the threat audit in Task 3. |
| 14 | Large-Scale Data Selection for Instruction Tuning | Shadman Kabir | arxiv.org/pdf/2503.01807 | **Include** | Cited in the submitted gap (Ivison et al., 2025) as a large-scale English selection study. PDF accessible but not yet read in depth; needs full extraction because the submission now cites it. |
| 15 | A Survey on Data Selection for LLM Instruction Tuning | Shadman Kabir | arxiv.org/pdf/2402.05123 | **Include** | Survey covering the method landscape our taxonomy (PROJECT_BRIEF §4) is built on; used as the structural backbone for the gap synthesis below, not as a primary empirical source. |

**Tally:** 10 Include, 4 Unsure (#5, #6, #9, #10), 0 Exclude. Row #10 is a duplicate of row #1 and should be merged once confirmed — so the working count of distinct papers is 14, matching the knowledge base.

**Action items before the spreadsheet is final:** resolve the 4 Unsure rows (confirm PDF access for #9 and #10, do a first read of #5 and #6 to decide relevance) and complete full extraction on #2, #3, #4, and #14, which are currently Include but only partially read (see Part B).

---

## Part B — Extraction

Extraction cards for the 10 Include papers, grouped in batches of 4 as requested, formatted to paste as rows into the Literature Review Spreadsheet. Columns follow the tracker's existing structure: Problem/Motivation, Data Used, Method/Pipeline, Baseline(s), Evaluation Approach, Results, Limitations, Reproducibility.

Seven of these ten already have a full read logged in the tracker (BanglaLlama, Too Late to Train, Evaluating LLMs' Multilingual Capabilities, TigerLLM, DEITA, AlpaGasus, and the Survey). The other three (Superfiltering, CoachLM, Data Diversity Matters) and one more (Large-Scale Data Selection) only have partial or no extraction logged — those cards are marked **Partial** and should not be treated as final until someone on the team does a full read.

### Batch 1

**1. BanglaLlama: LLaMA for Bangla Language** (Mukti, 2024/2026 — arxiv.org/abs/2410.21200)
- *Problem:* Bengali lacks reproducible, openly documented instruction-tuning resources built on LLaMA.
- *Data used:* Bangla-Orca (172k samples, translated from OpenOrca, code/math excluded), Bangla-Alpaca (52k pairs, translated from Stanford Alpaca), CulturaX Bengali subset for pretraining. Translated via Google Cloud Translation API (~$10k cost) with manual injection of Bangladeshi cultural references.
- *Method/pipeline:* Machine translation of existing English instruction corpora, no post-translation quality filtering — only random human spot-checks.
- *Baseline(s):* Prior LLaMA 2/3/3.1/3.2 base checkpoints (1B–8B), compared as base vs. instruct variants.
- *Evaluation approach:* Not detailed in the extracted notes beyond model release; treat as a resource paper for this axis.
- *Results:* Five base/instruct model variants released; no ablation of filtered vs. unfiltered data.
- *Limitations:* No data-quality filtering step; translation-loss artifacts and literal-translation errors acknowledged by the authors but not measured or corrected.
- *Reproducibility:* Code and model weights on Hugging Face (hf.co/collections/BanglaLLM/banglallama).
- **Why it matters here:** It is the clearest example of the machine-translated Bengali data the submitted gap describes: 224k translated samples with no filtering, and translation artifacts the authors acknowledge but do not measure. We do not train on it. The submitted design uses TigerLLM's already-filtered, natively generated Bangla-Instruct instead (Task 2).

**2. Superfiltering: Weak-to-Strong Data Filtering for Fast Instruction-Tuning** (ACL 2024 — aclanthology.org/2024.acl-long.769) — **Partial**
- *What's logged:* PDF accessible; no extraction fields filled in yet in the tracker.
- *What we know from the title/venue:* Proposes using a small ("weak") model's scoring signal to filter data for a larger ("strong") model's instruction tuning, aimed at making filtering fast/cheap.
- *Relevance if confirmed:* Would sit in Strategy family C, and matters specifically because it targets filtering *cost*, which is one of our practical constraints (§9 compute budget in PROJECT_BRIEF).
- **Action:** Needs a full read before it can support any claim in the gap or strategy sections.

**3. CoachLM: Automatic Instruction Revisions Improve the Data Quality in LLM Instruction Tuning** (IEEE — ieeexplore.ieee.org/document/10597991) — **Partial**
- *What's logged:* PDF accessible; dataset used = Alpaca-52k; no other fields filled in.
- *What we know from the title:* Revises (rewrites) low-quality instructions/responses automatically rather than discarding them — a "repair" strategy, distinct from keep/discard filtering.
- *Relevance if confirmed:* Offers an alternative framing to our filtering-only design — worth naming explicitly as future work / out-of-scope in our limitations, since our variant matrix (Task 3) only filters, it does not revise.
- **Action:** Needs a full read to confirm the revision mechanism and whether it reports a quality delta comparable to filtering-only methods.

**4. Data Diversity Matters for Robust Instruction Tuning** (EMNLP Findings 2024 — aclanthology.org/2024.findings-emnlp.195) — **Partial**
- *What's logged:* PDF accessible; datasets used = Alpaca-52k, Dolly-15k (small-scale) and UltraChat-1.3M, LMSYS-Chat-1M (large-scale); no other fields filled in.
- *What we know from the title/dataset pairing:* Compares small curated sets against much larger chat corpora to isolate the effect of diversity on robustness.
- *Relevance if confirmed:* Direct evidence source for Strategy family D (diversity/coverage) and for RQ3 (do interventions compose, or does diversity alone explain gains attributed to other filters).
- **Action:** Needs a full read to extract the actual diversity metric used and whether it is transferable to a Bengali embedding space.

### Batch 2

**5. Too late to train, too early to use? A study on necessity and viability of low-resource Bengali LLMs** (COLING 2025 — aclanthology.org/2025.coling-main.79)
- *Problem:* Whether it is still worth building dedicated low-resource Bengali LLMs given how well English-oriented LLMs now transfer cross-lingually.
- *Data used:* A compiled benchmark of 7 Bengali NLU/NLG tasks — BanglaNMT, XLSum, CrossSum, BanglaParaphrase, SQuAD-bn/BQA, BanglaRQA, BEnQA, XNLI-bn.
- *Method/pipeline:* Compares state-of-the-art open-weight and closed-source LLMs (e.g., LLaMA-3, GPT-4) against traditional fine-tuned encoder-decoder baselines on these tasks.
- *Baseline(s):* Fine-tuned encoder-decoder models (the pre-LLM standard for these tasks).
- *Evaluation approach:* Reasoning capability, Bengali script generation accuracy, and computational cost from tokenization.
- *Results:* LLMs are strong at reasoning but inconsistent at generating correct Bengali script; inefficient tokenization raises both cost and error rate.
- *Limitations:* The field still lacks high-quality Bengali pretraining and instruction-tuning data; widely used datasets carry machine-translation bias.
- *Reproducibility:* No code/data repository mentioned.
- **Why it matters here:** Directly supports RQ2 — it shows English-LLM transfer is not a substitute for good Bengali-specific data, which is the premise our whole thesis rests on.

**6. Evaluating LLMs' Multilingual Capabilities for Bengali: Benchmark Creation and Performance Analysis** (arxiv.org/abs/2507.23248)
- *Problem:* No standardized Bengali evaluation benchmark exists, and poor tokenization quietly degrades model performance.
- *Data used:* 8 major English NLP benchmarks (HellaSwag, Winogrande, CommonsenseQA, BoolQ, OpenBookQA, and others spanning commonsense/science/math/multidomain) machine-translated into Bengali using GPT-4o-mini, chosen over Google Translate/Azure after a blind human review. Total translation cost ~$200.
- *Method/pipeline:* Automated LLM translation → regex-based cleanup of malformed JSON outputs → benchmarking of 10 open-source LLMs (including Mistral and DeepSeek families).
- *Baseline(s):* The same 10 LLMs' performance on the original English benchmark versions, to isolate the multilingual performance gap.
- *Evaluation approach:* Inference accuracy on translated questions, plus correlation analysis between tokenization granularity (bytes/token, tokens/word) and accuracy.
- *Results:* Bengali performance is consistently worse than English across models; Mistral-family models degrade most, DeepSeek is most stable; higher tokens-per-word correlates with lower accuracy.
- *Limitations:* Benchmark data is itself machine-translated by an LLM, so it can carry translationese and the source model's alignment biases rather than natural Bengali.
- *Reproducibility:* Datasets on Hugging Face; translation and evaluation code on GitHub.
- **Why it matters here:** Gives us a concrete, reusable finding — tokenizer fertility predicts performance loss. The base model is now fixed to TigerLLM's LLaMA-3.2 (1B), so this finding no longer drives model choice. It does matter for the token-count confound in Task 3: arms with the same example count can have different token counts.

### Batch 3

**7. TigerLLM: A Family of Bangla Large Language Models** (arxiv.org/abs/2503.10995)
- *Problem:* Existing Bangla LLMs suffer from poor reproducibility, low data quality, and heavy reliance on translated synthetic data, despite Bangla having ~237M native speakers.
- *Data used:* Bangla-TextBook corpus (~9.9M tokens / 697,903 sentences from 163 National Curriculum and Textbook Board textbooks, Grades 6–12) for pretraining; Bangla-Instruct (100K native Bengali instruction-response pairs, expanded from 500 volunteer-written seed tasks via GPT-4o/Claude-3.5-Sonnet self-instruct) for fine-tuning. Its multi-stage filter checks four things: language adherence (Bengali script and word-ratio checks, grammar scoring), cultural sensitivity, content quality (response coherence and factual checks), and novelty (similarity-based deduplication and lexical-diversity checks).
- *Method/pipeline:* Continual pretraining of LLaMA-3.2-1B and Gemma-2-9B on Bangla-TextBook, then full fine-tuning (no LoRA, Flash Attention) on Bangla-Instruct.
- *Baseline(s):* Prior open Bangla LLMs (Titu-Gemma, Titu-LLaMA, Bangla-LLaMA, BongLLama) and proprietary models (GPT-3.5, GPT-4o-mini).
- *Evaluation approach:* Pass@1 accuracy on Bangla benchmarks (MMLU-bn, PangBench-bn, BanglaQuaD, mHumanEval-bn, BEnQA, BanglaRQA); no significance testing or user study.
- *Results:* TigerLLM-9B leads across all six benchmarks, beating GPT-3.5 and mostly beating GPT-4o-mini; TigerLLM-1B outperforms all prior open Bangla LLMs despite its size.
- *Limitations:* Textbook corpus restricted to Grade 6–12 academic register; Bangla-Instruct has no unanswerable-question category; model scale capped at 9B; regional/dialectal variation underrepresented. **No ablation comparing filtered vs. unfiltered Bangla-Instruct, and no random-subset control** — the paper treats its "native + curated" pipeline as inherently high quality without testing that claim empirically.
- *Reproducibility:* Both datasets and both models open-sourced on Hugging Face; pipeline code on GitHub (github.com/mraihan-gmu/TigerLLM).
- **Why it matters here:** This is our data pool, our fixed training and evaluation setup, and our closest related work. Its filter is binary: it keeps every pair that passes. It never scores instruction complexity separately from response quality and never selects for diversity over a combined score. The submitted design fills that gap (Task 2) and tests it with the missing ablation above (Task 3, Arms A–C).

**8. What Makes Good Data for Alignment? A Comprehensive Study of Automatic Data Selection in Instruction Tuning (DEITA)** (arxiv.org/pdf/2312.15685)
- *Problem:* No principled method existed for automatically selecting instruction-tuning data; prior work showed small curated sets can align models well without explaining *why* or *how* to select systematically.
- *Data used:* Two pools built from existing public instruction sets — a "high-quality" pool (ShareGPT, UltraChat, WizardLM, ~300K) and a "lower-quality/redundant" pool (Alpaca, Dolly, OpenAssistant, FLAN 2022, ~100K).
- *Method/pipeline:* Three signals — complexity and quality (each scored by evolving samples through iterations and training a LLaMA-7B scorer on ChatGPT-ranked seed data) and diversity (embedding distance to the nearest already-selected sample) — combined into one score, then greedily selected under a diversity threshold.
- *Baseline(s):* Random selection, instruction length, perplexity, instruction-following difficulty, InsTag complexity/diversity, and full models like LIMA, AlpaGasus, Vicuna, WizardLM, Zephyr, Tülu 2.
- *Evaluation approach:* MT-Bench, AlpacaEval, the Open LLM Leaderboard, plus a 4-annotator human study on 100 examples; no formal significance testing reported.
- *Results:* A 6K-sample DEITA-selected set matches or beats Zephyr-beta (trained on 260K samples, ~30x more data) on MT-Bench/AlpacaEval.
- *Limitations:* Entirely reliant on ChatGPT/GPT-4 for evolution, scoring, and judging (untested outside English); narrow quality definition (no factuality/safety check); some design choices unablated; judge circularity (GPT judging GPT-trained outputs); small human panel.
- *Reproducibility:* Model checkpoints and selected datasets released on GitHub (github.com/hkust-nlp/deita).
- **Why it matters here:** Our filtering strategy is DEITA's framework applied to Bangla-Instruct: score s = c × q, then greedy diversity-aware selection under a threshold τ (Task 2). One difference: we score c and q with LLM-as-judge prompts instead of training separate LLaMA-7B scorers, which keeps the cost manageable. Its untested-outside-English limitation is exactly what the submitted gap asks about.

### Batch 4

**9. AlpaGasus: Training a Better Alpaca with Fewer Data** (arxiv.org/pdf/2307.08701)
- *Problem:* Widely used instruction datasets (e.g., Alpaca-52k) contain low-quality, incorrect, or irrelevant responses that mislead instruction tuning and waste compute.
- *Data used:* Alpaca (52,002 pairs), Dolly-15k, GPT4LLM (Alpaca instructions with GPT-4 responses) as training pools; four human-curated test sets (Self-Instruct, Vicuna, WizardLM, Koala) for evaluation.
- *Method/pipeline:* ChatGPT (or Claude-2) scores each (instruction, input, response) triplet 0–5 for accuracy on a single dimension; triplets scoring ≥4.5 are kept (~9K of 52K); LLaMA-1/2 fine-tuned on the filtered subset with Alpaca's original hyperparameters.
- *Baseline(s):* Full unfiltered Alpaca-52k, and a size-matched random 9K subset (Alpaca-9k-random) — this is the closest existing example of an R-control in the literature we reviewed.
- *Evaluation approach:* GPT-4-as-judge pairwise win/tie/lose across four test sets (order-swapped to control position bias), a 3-rater human study (160 prompts), and standard benchmarks (MMLU, DROP, HumanEval, BBH).
- *Results:* The 9K-filtered model beats the 52K-unfiltered model on all four test sets; 13B model reaches >90% of text-davinci-003 capability; training cost drops from $27–225 to $5–41 depending on model size.
- *Limitations:* Single-dimension scoring with no category-balance control disproportionately strips coding examples (~88% filtered vs. ~82% average); relies entirely on proprietary LLM judges; the 4.5 score threshold is chosen heuristically, not theoretically justified.
- *Reproducibility:* Code, the filtered 9K dataset, and checkpoints public on GitHub, plus an independent community reimplementation.
- **Why it matters here:** AlpaGasus is the strongest existing template for our random control (Task 3, Arm B: a uniform random 40K sample matched to Arm C). Its documented coding-category collapse is the concrete cautionary example behind our threat audit: a single-score filter can silently distort category balance. Our quality-only ablation (Arm D) is close to an AlpaGasus-style filter, and mHumanEval-bn is the benchmark most likely to show that same coding collapse.

**10. A Survey on Data Selection for LLM Instruction Tuning** (Wang et al., arxiv.org/pdf/2402.05123)
- *Problem:* Frames the field-level question this whole project sits inside — how to automatically select a small, high-quality subset from large instruction sets, since quality outweighs quantity and manual selection is costly and biased.
- *Data used:* None collected (survey); catalogs existing sets — Self-Instruct (52K/252), Alpaca (52,002), WizardLM (250K via Evol-Instruct), LIMA (1,000/300/50), Dolly-v2 (15,000, Wikipedia-restricted), P3 (170 datasets, 2,052 templates).
- *Method/pipeline (of the papers it surveys):* Instruction length, perplexity, reward-model scores, k-nearest-neighbor distance, CLIP score, instruction-following-difficulty ratios, sentence embeddings, InsTag labels, complexity/quality scores, semantic-parse-tree node counts.
- *Baseline(s):* N/A (survey).
- *Evaluation approach:* N/A (survey; aggregates reported numbers across surveyed papers).
- *Results:* N/A (survey).
- *Limitations (as stated by the authors):* No uniform evaluation standard across selection methods; heavy time/API cost when filtering hundreds of thousands of instructions with strong LLMs; quality-assessment models are built almost entirely for English and general domains. Additional limitation we note: the survey's comparison tables aggregate numbers reported under different base models and benchmarks rather than a single controlled re-run, and it leans on GPT-4-as-judge throughout.
- *Reproducibility:* No new code/data; maintains a curated paper list at github.com/Bolin97/awesome-instruction-selector.
- **Why it matters here:** Confirms, from a field-wide view, the exact hole this thesis is aimed at — the survey's own stated limitation ("quality-assessment models... built almost entirely for English") is our research gap, stated by someone else first.

---

## Part C — Gap Synthesis, Organized by the Strategy Taxonomy (PROJECT_BRIEF §4)

The submission uses TigerLLM's Bangla-Instruct as the pool (Task 2), so the gap is now judged against what TigerLLM's filter already does. TigerLLM checks language adherence, cultural sensitivity, content quality, and novelty, and keeps every pair that passes.

**Family A — Deduplication.** No paper in our set tests deduplication as a measured intervention on Bengali. For our pool, TigerLLM's novelty check (similarity-based dedup plus lexical-diversity checks) already does this job, so semantic deduplication is a **pre-filter that is already applied**, not one of our experimental variables.

**Family B — Language-quality filtering.** "Too late to train, too early to use?" and "Evaluating LLMs' Multilingual Capabilities for Bengali" document the Bengali-specific symptoms: script errors, machine-translation bias, and tokenizer fragmentation. Neither proposes a filter. On Bangla-Instruct, TigerLLM's language-adherence check (script and word-ratio checks, grammar scoring) covers script and translationese filtering, so this is also an **already-applied pre-filter**. Bangla-Instruct is natively generated rather than machine-translated, which lowers the translationese risk compared with pools like BanglaLlama's.

**Family C — Instruction–response quality filtering.** This family has the most coverage, but all of it is on English data: AlpaGasus (single LLM-judge score), DEITA (complexity × quality), Superfiltering (weak-model scoring, pending full read), and CoachLM (revision, pending full read). TigerLLM's content-quality check scores the *response* only. No Bengali work scores **instruction complexity** separately from response quality, and none combines the two into one ranking score. That is the first half of what our strategy adds.

**Family D — Diversity/coverage.** DEITA's greedy embedding-distance selection and "Data Diversity Matters" (pending full read) are the evidence here, again on English. TigerLLM's pipeline has no diversity-aware *selection* step over a combined score. It deduplicates, then keeps everything. That is the second half of what our strategy adds.

**Consensus across sources.** Every comparison-reporting method paper (AlpaGasus, DEITA, and per the submission Ivison et al., 2025) finds that a smaller selected subset matches or beats the full pool, with far less training compute.

**Divergence / the actual hole.** All of those results come from English or English-sourced data, scored by judges that were trained mostly on English. TigerLLM, the strongest Bengali pool, treats "passes the binary filter" as good enough. It never compares its full 100K against a smaller selected subset, and it never compares against a same-size random subset. So nobody has tested whether score-based selection beats keeping everything on Bengali, and whether any gain comes from selection or just from having less data.

**Research gap statement.** See the submitted text at the top of this file: the English result (a selected subset matches or beats the full pool) has not been tested under controlled conditions for Bengali. Bengali has a different script, less mature tooling, and MT-heavy corpora, so we cannot assume the result transfers.

**Contribution claim.** A controlled comparison on TigerLLM's Bangla-Instruct, with TigerLLM's base model (LLaMA-3.2 1B), fine-tuning recipe, and six-benchmark evaluation suite held fixed. Only the data-selection method varies: the full 100K pool (Arm A), a size-matched 40K random sample (Arm B), and a 40K DEITA-style complexity × quality + diversity selection (Arm C), with optional ablations that remove complexity (Arm D) or diversity (Arm E). See Task 3.

---

## Part D — Pressure-Testing the Gap Against the Three Sub-Questions (PROJECT_BRIEF §1)

**RQ1 — Does selection improve downstream performance beyond what an equal-size random subset achieves?**
The gap survives. AlpaGasus is the only paper in our set with a size-matched random control, and only for English. In the submitted design, **Arm C vs. Arm B** (both 40K) answers RQ1 directly, and **Arm C vs. Arm A** (40K vs. 100K) answers whether the selected subset matches or beats the full pool, which is the claim the gap statement makes.

**RQ2 — Do English-developed methods transfer to Bengali?**
This is the sub-question with the most evidence behind it. The two Bengali-specific papers document failure modes with no English equivalent, and DEITA's own stated limitation is that it is untested outside English. The submitted design tests transfer directly: DEITA's method, unchanged in structure, applied to a native Bengali pool. The Task 4 pilot first checks that the LLM judge scores Bengali text consistently and directly, not via an English translation.

**RQ3 — Do the parts of the method compose, or do their gains overlap?**
The design's question here has changed from the old stacked dedup → language → quality chain. It now asks whether complexity and diversity each add anything beyond quality alone. **Arm D** (quality only) isolates complexity, and **Arm E** (complexity × quality, no diversity step) isolates diversity. Both are budget-dependent and run only after A–C. RQ3 is still the weakest-supported question in the literature we have read. "Data Diversity Matters" is still a partial read.

**Verdict:** the gap holds under all three tests. RQ1 and RQ2 are answered by the core arms (A–C), which are budgeted. RQ3 depends on the optional ablations (D–E). If they are not run, the thesis should say plainly that it cannot separate the complexity and diversity contributions.
