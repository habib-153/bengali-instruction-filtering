# Task 4 — Small-Scale Feasibility Test

**Project:** Evaluating Data-Quality Filtering Strategies for Bengali Instruction Tuning
**Purpose:** before scoring the full 100K pool or committing to a full fine-tuning run, check that the pipeline works on Bengali text end to end.
**Aligned with:** the submitted write-up, [bengali_filtering_tasks_1-4_humanized.md](bengali_filtering_tasks_1-4_humanized.md) (Task 4). Part A is as submitted. Parts B–C are supporting notes that go beyond the submission.

---

## Part A — Pilot Steps (as submitted)

1. **Sample a small slice.** Pull 500–1,000 pairs at random from Bangla-Instruct to test every downstream step at low cost.
2. **Test the LLM-judge scorers on Bengali.** Run the complexity and quality prompts on this slice. Check that the judge model returns consistent, parseable scores directly on Bengali instructions and responses, not on an English translation of them.
3. **Sanity-check embeddings.** Embed the slice with a multilingual or Bengali-capable sentence-embedding model. Confirm that distances behave sensibly (near-duplicate pairs sit close, unrelated pairs sit far) before trusting the diversity step at scale.
4. **Run a toy selection pass.** Apply the greedy diversity-aware selection with a small threshold *τ* and budget. Then manually spot-check a handful of selected vs. rejected pairs to confirm the subset actually looks more diverse and higher-quality than a same-size random sample.
5. **Run a toy fine-tune.** Fine-tune LLaMA-3.2 (1B) with LoRA for a few hundred steps on the small selected slice, only to confirm that the training script, data format, and hardware setup run without errors. This step draws no performance conclusions.
6. **Record time and cost.** Log the wall-clock time and API/compute cost for scoring the small slice, and extrapolate to the full 100K pool. This catches anything (judge-model cost, embedding time) that needs to change before committing to the full Task 3 run.

**Note on LoRA vs. full fine-tuning:** the toy fine-tune uses LoRA only to check the pipeline cheaply. The real Task 3 runs use full fine-tuning with TigerLLM's recipe. Once the toy run passes, do one short full-fine-tuning smoke test as well, to confirm memory fits at batch size 16 × gradient accumulation 4 with 2,048-token sequences.

---

## Part B — Stop-and-Redesign Criteria (supporting notes, not in the submission)

These results mean the pipeline itself is broken, not just that the data is noisy. Any one of them should stop the scale-up and trigger a fix first.

**Judge scores are not consistent or parseable.** Re-score a subset of ~50 pairs a second time. If scores often move by more than one point on the same input, or a noticeable share of outputs fail to parse, fix the prompt or output format before going further. Do not scale up a noisy scorer, because *s = c × q* multiplies the noise from both scores.

**The judge only works on translated text.** If scores on Bengali inputs are flat (near-constant) or clearly unrelated to quality on a manual check, while English translations of the same pairs score sensibly, then the judge does not read Bengali well enough. Switch to a judge model with better Bengali ability. Scoring translations instead would break the experiment's premise.

**Embeddings fail the near-duplicate test.** If hand-picked near-duplicate pairs are not clearly closer than unrelated pairs, the embedding model is not Bengali-capable. Swap the model. Re-tuning *τ* will not fix this.

**The diversity step cannot reach the budget, or accepts nearly everything.** If at a given *τ* the greedy pass runs out of candidates before the target size, or rejects almost nothing, *τ* is miscalibrated. Find a *τ* range that reaches 40% of the slice (mirroring 40K of 100K) while still rejecting some pairs.

**The selected subset doesn't look better than random.** If the manual spot-check (step 4) finds the selected pairs no better or more diverse than a random sample of the same size, or finds clearly good pairs among the rejected ones, look for what the rejected pairs have in common (length, topic, task type, e.g., coding) before running at scale. This is the AlpaGasus category-collapse risk from Task 1.

**Extrapolated cost exceeds the budget.** If step 6 projects that scoring 100K pairs (two judge calls each, for *c* and *q*) goes beyond the API budget, reduce cost before Task 3: use a cheaper judge, score *c* and *q* in one call, or score only part of the pool.

**What does *not* trigger a stop:** score distributions that are skewed high. Bangla-Instruct is already filtered, so most pairs should score reasonably well. Skew alone is expected. It only becomes a problem if scores no longer separate pairs at all.

---

## Part C — Cost and Timeline Note

Run the pilot right after the Bangla-Instruct license and row-count checks (Task 2, Part E) are done. Its main costs are the judge API calls for 500–1,000 pairs (×2 for *c* and *q*) and a short GPU session for embeddings and the toy fine-tune. It should fit inside one working session. The time and cost numbers from step 6 set the budget for the full 100K scoring pass and for deciding whether Arms D and E are affordable (Task 3).
