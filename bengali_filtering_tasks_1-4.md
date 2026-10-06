# Evaluating Data-Quality Filtering Strategies for Bengali Instruction Tuning

*Tasks 1–4: Research Gap, Filtering Strategy, Initial Experimental Comparison, and Feasibility Test*

> Markdown copy of the submitted document `bengali_filtering_tasks_1-4_humanized.docx`. Text is unchanged from the submission.

---

## Task 1: Research Gap

Across recent work on English instruction tuning, filtering methods like DEITA (Liu et al., 2024), AlpaGasus (Chen et al., 2024), and large-scale selection studies such as Ivison et al. (2025) have shown that cutting a large instruction pool down to a smaller subset, selected for response quality, instruction complexity, and sample diversity, can match or beat training on the full, unfiltered pool. That result holds up for English. It hasn't been tested under controlled conditions for Bengali.

Bengali instruction data is a different setting altogether. It uses a different script (the Bengali abugida instead of Latin script), the NLP tooling for it, embeddings, scorers, tokenizers, is less mature, and the instruction corpora available, including community-built and machine-translated datasets, still lean heavily on machine translation. That dependency brings its own quality problems: translationese, script inconsistency, cultural mismatch, none of which have a clean English equivalent.

So whether the English filtering result actually transfers, whether filtering improves downstream model performance on Bengali instruction data rather than just shrinking the dataset, is still an open question. This project tests that directly, using a fixed base model and training pipeline so the data-selection method is the only thing that varies across conditions.

---

## Task 2: Filtering Strategy and Data Pool

**Data pool.** The starting pool is TigerLLM's released Bangla-Instruct dataset (Raihan & Zampieri, 2025): 100,000 native Bengali instruction-response pairs generated through a self-instruct pipeline (500 seed tasks, GPT-4o and Claude-3.5-Sonnet as teacher models). Every pair in the released set has already passed the authors' own multi-stage filter, which checks four things: language adherence (Bengali script and word-ratio checks, grammar scoring), cultural sensitivity, content quality (response coherence and factual checks), and novelty (similarity-based deduplication and lexical-diversity checks).

Three of those four TigerLLM axes map onto this project's own filtering steps:

- **Semantic deduplication**: TigerLLM's novelty check
- **Script and translationese filtering**: TigerLLM's language-adherence check
- **LLM-as-judge quality scoring (response)**: TigerLLM's content-quality check

What TigerLLM's pipeline doesn't do is score instruction complexity separately from response quality, or apply an explicit diversity-aware selection step over a combined score. That's the gap this project's filtering strategy fills, borrowing DEITA's (Liu et al., 2024) score-first, diversity-aware framework:

- **Complexity scoring**: an LLM-as-judge complexity score c(i) for each instruction, following DEITA's Evol-Complexity approach.
- **Combined scoring**: score s = c × q, where q is the response-quality score. This is DEITA's published formula (complexity × quality), not "quality squared." Squaring quality alone would drop the complexity signal entirely, which contradicts the same task note's instruction to use quality and complexity together: flagging this in case a quality-only, squared score was actually what was intended.
- **Diversity-aware selection**: rank the pool by s, then greedily add pairs whose embedding distance to every already-selected pair exceeds a threshold τ, continuing until the target subset size is reached. This is DEITA's diversity step.

The result is a re-ranked, re-selected subset of Bangla-Instruct, chosen by complexity × quality plus diversity, used in place of TigerLLM's original "keep everything that passes the binary filter" approach. The semantic-dedup and script/translationese stages TigerLLM already applied stay in effect as a pre-filter on the pool itself.

---

## Task 3: Initial Experimental Comparison

This sets up the first controlled comparison, anchored to TigerLLM: its released dataset, base model, fine-tuning recipe, and evaluation suite all stay fixed, and only the data-selection method varies, isolating exactly the variable Task 1 flags as untested.

### 3.1 Fixed Elements (Anchored to TigerLLM)

**Base model:** LLaMA-3.2 (1B), TigerLLM's smaller published variant. Full parity would also mean testing the Gemma-2 (9B) variant, but that needs the 8×A100 cluster scale TigerLLM used for continual pretraining. The 1B path is what's actually feasible on available compute, with 9B as a stretch goal.

**Fine-tuning recipe:** full fine-tuning, no LoRA, matching TigerLLM's published hyperparameters for the 1B model: 2,048 max sequence length, batch size 16, gradient accumulation 4, 3 epochs, learning rate 1e-5, weight decay 0.02, 10% warm-up, AdamW (8-bit), cosine schedule, BF16 precision.

**Evaluation suite:** the same six Bangla-specific benchmarks TigerLLM reports: MMLU-bn (understanding), PangBench-bn (multitasking), BanglaQuaD (question answering), mHumanEval-bn (coding), BEnQA (knowledge), and BanglaRQA (reasoning), scored as Pass@1%, so results sit directly against TigerLLM's own published table.

### 3.2 Experimental Arms

| Arm | Data size | Selection method | Role |
|---|---|---|---|
| A: TigerLLM anchor | 100K (full pool) | TigerLLM's original release, no re-filtering | Reproduces the published baseline |
| B: Random control | 40K (matched to C) | Uniform random sample of the 100K pool | Isolates a pure data-size effect from a selection effect |
| C: Ours (DEITA-style) | 40K | complexity × quality score + diversity-aware selection | Tests this project's core hypothesis |
| D: Quality-only ablation | 40K | Top-K by quality score alone | Isolates the contribution of complexity |
| E: No-diversity ablation | 40K | Top-K by complexity × quality, no diversity step | Isolates the contribution of diversity |

Arms D and E run only if compute budget allows after A–C are complete.

### 3.3 Evaluation Metrics

**Primary:** Pass@1 on the six benchmarks above, compared arm-by-arm and against TigerLLM's published numbers.

**Diagnostic:** training loss curves per arm (does a smaller, well-selected set converge faster or lower, as TigerLLM's own findings suggest for quality-first data), plus pool-level diagnostics per arm (score distributions, Type-Token Ratio, embedding-space coverage) to get a sense of what "good" Bengali instruction data actually looks like under this framework.

**Optional (budget-permitting):** pairwise LLM-as-judge win rate between arms on a small held-out instruction set, closer to how DEITA itself was evaluated (MT-Bench/AlpacaEval-style) and a more direct read on instruction-following quality than the downstream knowledge benchmarks alone.

---

## Task 4: Small-Scale Feasibility Test

Before scoring the full 100K pool or committing to a full fine-tuning run, a small pilot checks that the pipeline actually works on Bengali text end to end.

1. **Sample a small slice**: pull 500–1,000 pairs at random from Bangla-Instruct to test every downstream step at low cost.
2. **Test the LLM-judge scorers on Bengali**: run the complexity and quality prompts on this slice and check that the judge model returns consistent, parseable scores directly on Bengali instructions and responses, not on an English translation of them.
3. **Sanity-check embeddings**: embed the slice with a multilingual or Bengali-capable sentence-embedding model and confirm distances behave sensibly (near-duplicate pairs sit close, unrelated pairs sit far) before trusting the diversity step at scale.
4. **Run a toy selection pass**: apply the greedy diversity-aware selection with a small threshold τ and budget, then manually spot-check a handful of selected vs. rejected pairs to confirm the subset actually looks more diverse and higher-quality than a same-size random sample.
5. **Run a toy fine-tune**: fine-tune LLaMA-3.2 (1B) with LoRA for a few hundred steps on the small selected slice, just to confirm the training script, data format, and hardware setup run without errors. Not to draw performance conclusions yet.
6. **Record time and cost**: log the wall-clock time and API/compute cost for scoring the small slice, and extrapolate to the full 100K pool, to catch anything (judge-model cost, embedding time) that needs to change before committing to the full Task 3 run.
