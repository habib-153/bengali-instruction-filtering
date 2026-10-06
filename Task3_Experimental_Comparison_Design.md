# Task 3 — Initial Experimental Comparison

**Project:** Evaluating Data-Quality Filtering Strategies for Bengali Instruction Tuning
**Builds on:** Task 2's strategy (complexity × quality scoring + diversity-aware selection on Bangla-Instruct) and PROJECT_BRIEF §5–6.
**Aligned with:** the submitted write-up, [bengali_filtering_tasks_1-4_humanized.md](bengali_filtering_tasks_1-4_humanized.md) (Task 3). Parts A–C are as submitted. Parts D–E are supporting notes that go beyond the submission.

This sets up the first controlled comparison, anchored to TigerLLM. Its released dataset, base model, fine-tuning recipe, and evaluation suite all stay fixed, and only the data-selection method varies. That isolates the variable Task 1 flags as untested.

---

## Part A — Fixed Elements (Anchored to TigerLLM)

**Base model:** LLaMA-3.2 (1B), TigerLLM's smaller published variant. Full parity would also mean testing the Gemma-2 (9B) variant, but that needs the 8×A100 cluster scale TigerLLM used for continual pretraining. The 1B path is what's feasible on available compute. 9B is a stretch goal.

**Fine-tuning recipe:** full fine-tuning, no LoRA, matching TigerLLM's published hyperparameters for the 1B model:

| Hyperparameter | Value |
|---|---|
| Max sequence length | 2,048 |
| Batch size | 16 |
| Gradient accumulation | 4 |
| Epochs | 3 |
| Learning rate | 1e-5 |
| Weight decay | 0.02 |
| Warm-up | 10% |
| Optimizer | AdamW (8-bit) |
| Schedule | Cosine |
| Precision | BF16 |

**Evaluation suite:** the same six Bangla-specific benchmarks TigerLLM reports, scored as Pass@1 (%), so results sit directly against TigerLLM's own published table:

| Benchmark | Capability |
|---|---|
| MMLU-bn | Understanding |
| PangBench-bn | Multitasking |
| BanglaQuaD | Question answering |
| mHumanEval-bn | Coding |
| BEnQA | Knowledge |
| BanglaRQA | Reasoning |

---

## Part B — Experimental Arms

| Arm | Data size | Selection method | Role |
|---|---|---|---|
| **A: TigerLLM anchor** | 100K (full pool) | TigerLLM's original release, no re-filtering | Reproduces the published baseline |
| **B: Random control** | 40K (matched to C) | Uniform random sample of the 100K pool | Isolates a pure data-size effect from a selection effect |
| **C: Ours (DEITA-style)** | 40K | complexity × quality score + diversity-aware selection | Tests this project's core hypothesis |
| **D: Quality-only ablation** | 40K | Top-K by quality score alone | Isolates the contribution of complexity |
| **E: No-diversity ablation** | 40K | Top-K by complexity × quality, no diversity step | Isolates the contribution of diversity |

Arms D and E run only if compute budget allows after A–C are complete.

**How the arms answer the research questions (see Task 1, Part D):**

- **C vs. B** (same size, different selection): is there a real selection effect, or is it just data size? (RQ1)
- **C vs. A** (40K selected vs. 100K full): does a selected subset match or beat the full pool on Bengali, as it does on English? (the core gap / RQ2)
- **C vs. D** and **C vs. E**: does complexity add anything beyond quality, and does diversity add anything beyond the score ranking? (RQ3)

**Comparison unit:** equal example count (40K) across B–E, with A at full size as the reference. Under a fixed recipe of 3 epochs, a 40K arm trains on 40% of A's examples, so a C ≥ A result also means less training compute.

---

## Part C — Evaluation Metrics

**Primary:** Pass@1 on the six benchmarks above, compared arm-by-arm and against TigerLLM's published numbers.

**Diagnostic:**

- Training loss curves per arm: does a smaller, well-selected set converge faster or lower, as TigerLLM's own findings suggest for quality-first data?
- Pool-level diagnostics per arm: score distributions (*c*, *q*, *s*), Type-Token Ratio, and embedding-space coverage. These show what "good" Bengali instruction data looks like under this framework.

**Optional (budget-permitting):** pairwise LLM-as-judge win rate between arms on a small held-out instruction set. This is closer to how DEITA itself was evaluated (MT-Bench/AlpacaEval-style) and gives a more direct read on instruction-following quality than the downstream knowledge benchmarks alone. If run: swap response order across two runs per pair and average the results to control position bias, and report the judge model, the exact prompt, and the swap procedure with the number.

---

## Part D — Threat Audit (supporting notes, not in the submission)

**Arm A may not reproduce TigerLLM's published number.** TigerLLM-1B was continually pretrained on Bangla-TextBook *before* fine-tuning on Bangla-Instruct. If we fine-tune raw LLaMA-3.2 (1B), Arm A will likely score below TigerLLM's published table, and the gap would come from pretraining, not data selection. Before running, decide whether we start from TigerLLM's continually pretrained checkpoint (if released) or from base LLaMA-3.2. Either way, compare arms against our own Arm A first, and against the published table second.

**Circular judge bias.** Bangla-Instruct was generated by GPT-4o and Claude-3.5-Sonnet. If the judge scoring *c* and *q* comes from either family, it may favor text in its own style rather than better Bengali. The same applies to the optional pairwise judge, and that judge should not be the same model as the scoring judge. Name both judge models before running. If they cannot be kept apart within budget, state the circularity as a limitation.

**Judge reliability in Bengali.** The whole strategy depends on the LLM judge scoring Bengali instructions and responses consistently. If Arm C underperforms Arm B, we cannot tell at first whether selection doesn't help or the judge misread Bengali quality. Mitigation: the Task 4 consistency check, and ideally judge agreement against a small human-annotated sample.

**Token count differs at equal example count.** Complexity scoring tends to favor longer, multi-part instructions, so Arm C's 40K examples may contain noticeably more tokens than Arm B's 40K. Part of a C > B gain could then come from token volume, not selection. Mitigation: report token counts alongside example counts for every arm (PROJECT_BRIEF §5), and flag large differences. Bengali tokenizer fertility (Task 1, "Evaluating LLMs' Multilingual Capabilities") makes this worse.

**Category imbalance from score-based selection.** AlpaGasus's single-score filter removed coding examples far more often than other categories. Arms C and D could do the same, which would show up as a drop on mHumanEval-bn specifically. Mitigation: compare task-type composition of each arm's 40K against the full pool, and read per-benchmark results, not just the average.

**Benchmark contamination.** Bangla-Instruct was generated by GPT-4o and Claude-3.5-Sonnet, and several benchmarks (e.g., MMLU-bn) are translations of public English sets. Check for overlap between benchmark items and Bangla-Instruct before final evaluation. This affects all arms equally, but it can inflate absolute numbers compared with TigerLLM's table.

---

## Part E — Open Items Not Specified in the Submission

- **Seeds.** Not stated. Recommend 3 seeds per arm if compute allows. With 1 seed, make no significance claims and state this as a limitation.
- **Choice of 40K.** Not justified in the submission. Confirm it against the real pool size (Task 2, Part E). Keep the threshold *τ* loose enough that the diversity step can actually reach 40K.
- **Judge and embedding models.** Not named yet. The Task 4 pilot should settle both.
- **Reporting format per arm:** pool size, selected size, token count, wall-clock time, Pass@1 on each benchmark, diagnostics, and seed variance where available (PROJECT_BRIEF §5).
