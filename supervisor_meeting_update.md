# Supervisor update — Bengali instruction-data filtering

Oct 8, 2026 · Team NeuralSight

All four tasks have moved from plan to working code. The Kaggle feasibility pilot (1,001 Bangla-Instruct rows) passed every judge check; the full 100K run now needs a compute decision.

## Status at a glance

| Task | Status | Evidence |
| --- | --- | --- |
| 1. Research gap | Done | Submitted text unchanged. Profiling the dataset adds evidence: machine-translation artefacts sit inside Bangla-Instruct itself. |
| 2. Filtering strategy and data pool | Done, now in code | Notebook `iteration03_filtering_selection.ipynb`: Bengali-aware cleaning, 11 rule filters, dedup, benchmark decontamination, LLM-judge scores c and q, DEITA-style selection. |
| 3. Experimental comparison | Code ready, not run | Notebook `iteration04_finetune_eval.ipynb`: LLaMA-3.2-1B fine-tuning and evaluation on MMLU-bn, BanglaRQA, mHumanEval-bn. Tested end to end on CPU only. |
| 4. Feasibility pilot | Mostly done | Kaggle run on 1,001 rows (2×T4 GPU, ~1 hour): 4 of 6 steps done, 1 partial, the toy fine-tune still to run. |

## Task 1 — Research gap

The gap stands as submitted: data selection (DEITA, AlpaGasus, Ivison et al. 2025) beats full-pool training in English, but nobody has tested it under controlled conditions for Bengali.

Profiling the real dataset now backs the "translation-heavy" claim with evidence from Bangla-Instruct itself:

- Alpaca's `<noinput>` placeholder survives as the Bengali word **নোইনপুট** and as "কোন ইনপুট নেই" (~0.6% of the first sub-corpus).
- FLAN-style rows keep English template words inside Bengali prompts (`Choices: - Dogs - Leaves`).
- The math sub-corpus reads as translated Chinese competition problems (WeChat red packets, full-width spaces).

So even a "native, filtered" Bengali pool carries translation artefacts, which is exactly the setting the gap describes.

## Task 2 — Data pool and filtering strategy

The strategy is unchanged (complexity × quality, then diversity-aware selection), but the real dataset differs from the paper's description in ways that change the plan.

| Finding (profiled on 12,530 rows from 5 points in the file) | Number | Consequence |
| --- | --- | --- |
| Size of the Hugging Face release | **342,391 rows**, not 100K | We must choose which 100K to use (decision below) |
| Licence on Hugging Face | MIT (our earlier notebook said CC-BY-4.0) | Corrected |
| Structure | 4 sub-corpora joined end to end; translated LaTeX math is the last ~40% | Sample by domain, report results per domain |
| Old language filter (iteration 02) | dropped 24% of rows, mostly math (LaTeX counted as English) | Replaced: language is checked on prose only |
| Old 0–100 quality score | median 100, 98.8% "excellent" | Cannot rank anything; replaced by LLM-judge scores |
| Responses cut off mid-sentence | ~2.5% | Trimmed to the last full sentence instead of dropped |
| Llama-3.2 tokenizer on Bengali | 0.94 characters per token; 14% of rows exceed 2,048 tokens | Drives the compute cost (see Compute) |

The filtering pipeline now implemented, in order:

1. **Bengali-aware cleaning**: Unicode NFC (41% of responses mixed two encodings of য়), keeps ZWJ/ZWNJ, code indentation and LaTeX intact.
2. **11 rule filters**, each drop logged with a reason: length, broken encoding, `নোইনপুট`-type artefacts, AI refusals, wrong language, untranslated English, echoed answers, truncation, character spam, unsegmented text, repetition. A per-domain report flags any filter that hits one domain unfairly.
3. **Deduplication**: exact hash + MinHash near-duplicates (Jaccard ≥ 0.8).
4. **Decontamination**: drop any row sharing a 13-word sequence with MMLU-bn, BanglaRQA, mHumanEval-bn or our held-out prompts.
5. **Scores**: LLM-judge complexity *c* and quality *q* (1–6, read from the judge's digit probabilities), plus perplexity and IFD from the base model.
6. **Selection**: rank by *s = c × q*, then add a row only if its embedding is not too similar (cosine < τ) to any row already chosen.

## Task 3 — Experimental comparison

The design is as submitted: TigerLLM's 1B recipe stays fixed and only the data-selection method changes. The training and evaluation code is written and passes an end-to-end test on CPU; no GPU training run yet.

| Arm | Size | Selection | Question it answers |
| --- | --- | --- | --- |
| A | full clean pool | none (TigerLLM anchor) | Baseline |
| B | 40K | random | Is any gain just from less data? |
| C | 40K | c × q + diversity (ours) | Core hypothesis |
| D | 40K | quality only | What does complexity add? |
| E | 40K | c × q, no diversity | What does diversity add? |
| F *(new, optional)* | 40K | IFD (Cherry-LLM) | A second published baseline at no extra cost |

What was added to the design while building it:

- **Held-out set**: 500 prompts removed before any filtering; never trained on. Used for held-out loss and the optional win-rate comparison.
- **Same hygiene for every arm**: arm A is the pool after cleaning and decontamination, so C vs A measures selection, not cleaning.
- **Token counts per arm** are logged, because complexity favours longer rows (pilot: C has 8% more tokens than B at the same row count).
- **Significance**: paired bootstrap confidence intervals over benchmark items, 3 seeds per arm.

| Benchmark | Status |
| --- | --- |
| MMLU-bn (OpenAI MMMLU, Bengali, 14,042 questions) | Wired in |
| BanglaRQA (1,493 test questions) | Wired in |
| mHumanEval-bn (164 Python tasks) | Wired in |
| BEnQA, BanglaQuaD, PangBench-bn | Not on Hugging Face; need the files from the authors |

## Task 4 — Feasibility pilot (Kaggle, 2×T4 GPU)

The pilot ran end to end in about one hour and the LLM judge passed every check. Of 1,001 sampled rows, 45 were dropped by the filters (16 cut-off answers were repaired instead), 0 were duplicates and 1 overlapped MMLU-bn, leaving 955.

| Step in the plan | Result | Verdict |
| --- | --- | --- |
| 1. Sample 500–1,000 rows | 1,001 rows + 50 held-out, sampled by domain from all 342,391 | Done |
| 2. Judge scores Bengali directly | Qwen2.5-7B-Instruct: **100%** parseable; English vs Bengali rubric agree at **0.89** (complexity) and **0.80** (quality) | Pass |
| 3. Embeddings behave sensibly | bge-m3: a math problem's nearest neighbours are math problems (cos 0.63); duplicate test not possible (no duplicates among 1K rows) | Partial |
| 4. Toy selection + spot-check | Arms built; blind 100-row sheet (50 from C, 50 from B) ready for human rating | Done, rating pending |
| 5. Toy fine-tune (LoRA) | Code ready; not yet run on GPU | Pending |
| 6. Time and cost | Judge 54 min, perplexity 3.5 min, embeddings 18 s for 955 rows | Done |

Selection does what it should. Arm C (ours) against random arm B, both 400 rows:

| Measure | B (random) | C (ours) |
| --- | --- | --- |
| Mean complexity c (1–6) | 3.73 | 4.28 |
| Mean quality q (1–6) | 4.68 | 5.10 |
| Total tokens | 484K | 524K |
| Math share | 35% | 43% |
| Code share | 14% | 10% |

The pilot also caught two problems, both now fixed: the DEITA threshold τ = 0.9 never fired with bge-m3 (C came out identical to E), and the language filter dropped 19% of code rows. τ is now calibrated from the pool itself.

## Compute and cost

Scoring the full 100K pool with the 7B judge would take ~47 GPU-hours on Kaggle, more than the free quota allows (30 h per week, 12 h per session; 24 h left this week). Estimates are extrapolated from the pilot's logged times.

| Job | Kaggle 2×T4 | A100 (estimate) |
| --- | --- | --- |
| LLM judge, 100K rows (7B) | ~47 h | ~3–5 h (with vLLM) |
| Perplexity / IFD, 100K rows | ~6 h | ~1 h |
| Embeddings + filters, 100K rows | < 1 h | < 0.5 h |
| Full fine-tune, arm A (100K, 3 epochs) | ~35 h | ~7 h |
| Full fine-tune, one 40K arm | ~15 h | ~3 h |

Options for the scoring step:

- **A. Cheaper judge on Kaggle (recommended):** Qwen2.5-3B judge + shorter judge inputs + skip IFD (it only feeds optional arm F): ~10–15 h across two sessions. Needs a 20-minute re-pilot to confirm the 3B judge passes the same checks.
- **B. Smaller pool:** keep the 7B judge on a 40K pool with 16K arms: ~20 h. Changes the design.
- **C. A100 access** (Colab Pro or university GPU): fastest and keeps the design as is.

Training A + B + C with 3 seeds is ~40 A100-hours; on Kaggle T4s it does not fit unless we switch to LoRA.

## Decisions needed from you

- [ ] **Which pool?** The release has 342,391 rows; the paper trained on 100K. Proposal: a 100K sample stratified by domain (or ask the TigerLLM authors which 100K they used).
- [ ] **c × q or quality²?** We use DEITA's c × q; the task note may have meant quality squared.
- [ ] **Arm A = cleaned pool?** Proposal: yes, so that C vs A isolates selection rather than cleaning.
- [ ] **Compute route** for scoring: option A, B or C above.
- [ ] **Training route**: full fine-tuning on A100s (as TigerLLM did) or LoRA on Kaggle.
- [ ] **Mixed 30K collection** (iteration 02: Bangla-Instruct + translated Alpaca + Wikipedia): keep it as a separate dataset? Mixing sources into arms A–E would add a second variable.
- [ ] **Missing benchmarks**: can you help obtain BEnQA, BanglaQuaD and PangBench-bn?

## Next steps

1. Rate the 100-row blind spot-check sheet (team, 1–5 per row) to finish Task 4 step 4.
2. Run the LoRA toy fine-tune on Kaggle (~1 h) to finish Task 4 step 5.
3. Re-run the pilot with the chosen judge and the fixed τ; confirm the duplicate test on a larger slice.
4. Score the full pool on the chosen compute route.
5. Train arms A, B, C (seed 42 first), then the remaining seeds and arms D, E, F.

---

# বাংলায় সহজ ব্যাখ্যা — Study guide

মিটিং-এর আগে এই অংশ পড়ে নিন। প্রথম অংশ (Supervisor update) supervisor-কে দেখানোর জন্য; এই অংশ আপনার নিজের বোঝার জন্য, বাংলা + English মিশিয়ে।

## পুরো গল্পটা সহজ কথায় / The whole story

**এক লাইনে:** বড় dataset থেকে ভালো আর বৈচিত্র্যময় উদাহরণ বেছে ছোট dataset বানালে, সেটা দিয়ে train করা model কি বাংলাতেও ভালো করে? ইংরেজিতে করে, বাংলায় কেউ পরীক্ষা করেনি। *In one line: does picking the best and most varied examples from a big dataset help a Bengali model, as it does in English?*

1. **Data (ডেটা):** TigerLLM-এর Bangla-Instruct — প্রশ্ন (instruction) আর উত্তর (response)-এর জোড়া। Paper বলে 100K, কিন্তু আসল file-এ 342,391টা সারি।
2. **Cleaning (পরিষ্কার):** ভাঙা, অর্ধেক কাটা, ডুপ্লিকেট, আর test-এর প্রশ্নের সাথে মিলে যাওয়া সারি বাদ দিই। *Remove broken rows, duplicates and rows that leak test questions.*
3. **Scoring (নম্বর দেওয়া):** একটা বড় AI model (judge) প্রতিটা সারিকে দুটো নম্বর দেয়: প্রশ্ন কত কঠিন (*c*, complexity) আর উত্তর কত ভালো (*q*, quality), 1–6 স্কেলে।
4. **Selection (বাছাই):** *c × q* অনুযায়ী সাজাই, তারপর উপর থেকে একটা একটা করে নিই — কিন্তু আগে নেওয়া কোনো সারির মতো হুবহু হলে বাদ (diversity)। এটাই DEITA method।
5. **Experiment (পরীক্ষা):** একই model (LLaMA-3.2-1B) আলাদা আলাদা data set (arm A–F) দিয়ে train করি, তারপর বাংলা benchmark-এ পরীক্ষা। শুধু data বদলায়, তাই ফলাফলের পার্থক্য data-র কারণেই।
6. **Pilot (ছোট পরীক্ষা):** পুরো 100K করার আগে 1,001টা সারি দিয়ে Kaggle-এ পুরো pipeline চালিয়েছি। সব ঠিকমতো চলেছে, judge বাংলা ভালোই বোঝে। এখন শুধু বড় run-এর জন্য GPU লাগবে।

## মূল শব্দগুলো / Key terms

| Term | বাংলায় সহজে | In English |
| --- | --- | --- |
| Instruction tuning | model-কে "প্রশ্ন → ভালো উত্তর" জোড়া দিয়ে শেখানো, যাতে সে কথা মেনে উত্তর দেয় | Fine-tuning on instruction–response pairs |
| Data selection | সব ডেটা না নিয়ে সবচেয়ে কাজেরগুলো বেছে নেওয়া | Training on a chosen subset instead of everything |
| DEITA | ইংরেজি একটা method: কঠিনতা × মান দিয়ে সাজাও, তারপর একই রকম গুলো বাদ দাও | Score-first, diversity-aware selection (Liu et al., 2024) |
| *c* (complexity) | প্রশ্নটা কত কঠিন, 1 (সহজ) – 6 (খুব কঠিন) | How demanding the instruction is |
| *q* (quality) | উত্তরটা কত সঠিক, সম্পূর্ণ আর সাবলীল বাংলায়, 1–6 | How good the response is |
| *s = c × q* | দুটো নম্বর গুণ করে একটা ফাইনাল নম্বর; কঠিন আর ভালো দুটোই হলে উপরে | Combined ranking score |
| LLM judge | একটা বড় AI (আমাদের Qwen2.5-7B) যে পরীক্ষকের মতো নম্বর দেয় | A model used as the grader |
| Embedding | প্রতিটা লেখাকে সংখ্যার তালিকা বানানো, যাতে দুটো লেখা কত কাছাকাছি মাপা যায় (bge-m3) | Text as a vector, for measuring similarity |
| Cosine similarity | দুটো লেখা কত একই রকম: 1 = হুবহু, 0 = কোনো মিল নেই | Similarity between two embeddings |
| τ (tau) | "কতটা মিল হলে বাদ দেব" — সীমা রেখা | Similarity cut-off for the diversity step |
| Arm | একটা data-set সংস্করণ; একই model ক্রমান্বয়ে আলাদা arm দিয়ে train হয় | One experimental condition |
| Ablation | একটা অংশ সরিয়ে দেখা সেটার অবদান কত (arm D, E) | Removing one part to measure its effect |
| Decontamination | test-এর প্রশ্ন training data-তে থাকলে সেই সারি বাদ, নইলে model "নকল" করে ভালো score করবে | Removing test-set overlap from training data |
| Held-out set | আলাদা করে রাখা 500 প্রশ্ন, কখনো training-এ যায় না | Prompts never trained on, kept for evaluation |
| Perplexity | model একটা লেখা দেখে কতটা "অবাক" হয়; খুব বেশি = সম্ভবত আবজাব লেখা | How surprised a model is by a text |
| IFD | প্রশ্নটা উত্তর বুঝতে কতটা সাহায্য করে; 1-এর বেশি = প্রশ্ন-উত্তর মেলে না | Instruction-Following Difficulty (Cherry-LLM) |
| Translationese | অনুবাদের ছাপ: ইংরেজি বাক্যগঠন, অনূদিত ইংরেজি, "নোইনপুট" | Artefacts of machine translation |
| Token | model যে টুকরোয় লেখা পড়ে; বাংলায় প্রায় প্রতি অক্ষরে একটা, তাই খরচ বেশি | The unit a model reads; cost scales with it |
| Full fine-tune vs LoRA | পুরো model update বনাম অল্প কিছু অতিরিক্ত weight; LoRA সস্তা, কিন্তু TigerLLM full করেছিল | Update all weights vs a small adapter |
| Seed | random-এর শুরুর মান; 3টা seed = একই পরীক্ষা 3 বার, ফলাফল ভাগ্য না সত্য তা বোঝার জন্য | Repeat runs to separate signal from luck |
| Spearman correlation | দুটো নম্বরের ক্রম কতটা একসাথে যায়: 1 = পুরো এক, 0 = কোনো সম্পর্ক নেই | Agreement between two rankings |

## Pilot-এর সংখ্যাগুলো কীভাবে পড়বেন / Reading the pilot numbers

| Number | মানে কী / What it means | ভালো না খারাপ? |
| --- | --- | --- |
| 1,001 → 955 rows | filter-এ 45টা বাদ (4.5%), 16টা কাটা উত্তর মেরামত, test-এর সাথে মিল 1টা | ভালো: TigerLLM আগেই filter করেছিল, তাই কম বাদ পড়াই স্বাভাবিক |
| Parse rate 100% | judge প্রতিবার ঠিক ফর্ম্যাটে (1–6 একটা সংখ্যা) উত্তর দিয়েছে | ভালো (লক্ষ্য ছিল ≥ 95%) |
| 0.89 / 0.80 | একই নির্দেশ ইংরেজিতে আর বাংলায় দিলে judge প্রায় একই নম্বর দেয় (complexity / quality) | ভালো: judge বাংলা স্থিরভাবে বোঝে (লক্ষ্য ≥ 0.5) |
| c = 3.68, q = 4.70 (গড়) | প্রশ্ন মাঝারি কঠিন, উত্তর সাধারণত ভালো | স্বাভাবিক; নম্বর ছড়ানো, তাই সাজানো যায় |
| c vs প্রশ্নের দৈর্ঘ্য = 0.49 | লম্বা প্রশ্ন কিছুটা বেশি কঠিন নম্বর পায় | ঠিক আছে (< 0.7), তবু token গুনে report করব |
| c vs q = −0.23 | কঠিন প্রশ্নের উত্তর একটু কম ভালো | আকর্ষণীয়: তাই "শুধু quality" (D) আর "c × q" (C) আলাদা সারি বাছে, overlap মাত্র 52% |
| C vs B: c 4.28 vs 3.73, q 5.10 vs 4.68 | আমাদের বাছাই random-এর চেয়ে কঠিন আর ভালো সারি নেয় | ভালো: selection কাজ করছে |
| C-তে 8% বেশি token | সমান সারি, কিন্তু লেখা বেশি | সাবধান: ফলাফল ভালো হলে এটা কি শুধু বেশি লেখার কারণে? তাই report করব |
| C ∩ E = 100% | প্রথম run-এ diversity step কোনো সারি বাদ দেয়নি | সমস্যা ছিল, এখন ঠিক (τ pool থেকে মাপা হয়) |
| IFD ≥ 1: 0.1% | প্রায় সব সারিতে প্রশ্নটা উত্তর বুঝতে সাহায্য করে | ভালো: প্রশ্ন-উত্তর অমিল প্রায় নেই |
| Judge 54 min / 955 rows | পুরো 100K-এ হিসাব করলে ~47 GPU-ঘণ্টা | সমস্যা: ফ্রি Kaggle quota-তে ধরে না, তাই সিদ্ধান্ত লাগবে |

## কী ভুল হয়েছিল, কীভাবে ঠিক করলাম / What went wrong and the fix

সব ভুল প্রকৃত data বা pilot চালিয়ে ধরা পড়েছে — এটাই pilot-এর কাজ। *Every problem below was caught by running on real data, which is the point of a pilot.*

| সমস্যা / Problem | কারণ / Cause | Fix |
| --- | --- | --- |
| Iteration 02 language filter 24% সারি ফেলে দিয়েছিল, বেশিরভাগ গণিত | `$\frac{1}{2}$` মতো LaTeX-কে ইংরেজি বলে গণ্য করেছিল | শুধু সাধারণ লেখা (prose) দেখে ভাষা মাপি; math, code, URL বাদ দিয়ে |
| Iteration 02-এর quality score সবাই প্রায় 100 পেয়েছিল | নিয়মগুলো শুধু pass/fail, ভালো-খারাপ আলাদা করতে পারে না | LLM judge-এর c আর q |
| Bengali-র ZWJ/ZWNJ মুছে যাচ্ছিল, code-এর indentation ভেঙে যাচ্ছিল | cleaning খুব কড়া ছিল | Bengali-aware cleaning: সেগুলো রেখে দেয় |
| Repetition filter গণিতের উত্তর ফেলে দিচ্ছিল | ধাপে ধাপে সমাধানে "সমাধান করি:" বারবার আসে, এটা স্বাভাবিক | math-এ এই filter বন্ধ, সীমা 0.5 |
| Test-overlap check ভুল match দিচ্ছিল | `frac 1 2 times` আর FLAN template বাক্য অনেক সারিতে একই | math আর সংখ্যা বাদ, অনেক সারিতে থাকা বাক্য = template, গণ্য নয় |
| প্রথম Kaggle run 2.7 ঘণ্টায়ও শেষ হয়নি (cancel করি) | T4 GPU-তে bf16 আসলে নেই, software দিয়ে নকল করে চলছিল | T4-এ fp16; perplexity 1,229 s থেকে 207 s (6× দ্রুত) |
| Diversity step কিছুই বাদ দেয়নি (C = E) | DEITA-র τ = 0.9 অন্য embedding model-এর জন্য; bge-m3-এ কোনো জোড়া 0.89-এর উপরে যায়নি | τ এখন pool থেকে মাপা হয় (pilot-এ 0.767) |
| Language filter code-এর 19% ফেলে দিচ্ছিল | উত্তরে code থাকলে তো ইংরেজি অক্ষরই বেশি | code প্রশ্নে শুধু প্রশ্নের ভাষা দেখি |

## Supervisor যা জিজ্ঞেস করতে পারেন / Likely questions

| Question | Say this (English) | মনে রাখুন (বাংলা) |
| --- | --- | --- |
| TigerLLM already filtered the data. Why filter again? | Their filter is pass/fail and keeps everything that passes. Ours ranks by complexity × quality and enforces diversity. Our cleaning still found artefacts like নোইনপুট and cut-off answers. | ওদের = পাস/ফেল; আমাদের = র‍্যাঙ্কিং + বৈচিত্র্য |
| Why not use their 100K directly? | The public file has 342,391 rows and doesn't say which 100K they trained on. We propose a 100K sample stratified by domain, unless the authors can tell us. | ফাইলে 342K, কোন 100K জানি না |
| Why c × q and not quality squared? | c × q is DEITA's published formula. Quality squared drops complexity entirely. The pilot shows they are different signals (correlation −0.23). | দুটো আলাদা তথ্য দেয় |
| Can an AI judge score Bengali reliably? | 100% of its answers were usable, and the English and Bengali versions of the rubric agree at 0.89 and 0.80. A blind human check of 100 rows is the next step. | সংখ্যা আছে, মানুষের যাচাই বাকি |
| Why Qwen and not GPT-4o or Claude as judge? | Bangla-Instruct was generated by GPT-4o and Claude-3.5. A judge from the same family could favour its own style. Qwen is also free to run. | নিজের লেখা নিজে নম্বর দিলে bias |
| Why a random arm? | It separates "less data" from "better data". If C beats B at the same size, the gain comes from selection. | কম data না ভালো data? |
| Couldn't C win just because it has more text? | Possible: C has 8% more tokens than B. We log tokens per arm and can add a token-matched random arm if needed. | token গুনে report করি |
| Will your numbers match TigerLLM's table? | Not exactly. They also pre-trained on Bangla textbooks first, and our MMLU-bn split may differ. We compare our arms with each other. | arm বনাম arm তুলনা |
| Why only 1B and not 9B? | 9B needs a multi-GPU cluster. 1B fits the compute we have; 9B stays a stretch goal. | compute-এর কারণে |
| Is 1,000 rows enough? | For checking that the pipeline works, yes. It is not for results. The duplicate test needs a larger slice and runs with the full pool. | pilot = চলে কিনা দেখা, ফলাফল না |
| What do you need from me? | A decision on the pool, the compute route and the training route, plus help getting BEnQA, BanglaQuaD and PangBench-bn. | সিদ্ধান্ত + GPU + benchmark |

## মিটিং-এ কী বলবেন, ক্রমানুসারে / Meeting script (~7 minutes)

প্রথম অংশ (Supervisor update) screen-এ খুলে উপর থেকে নিচে যান। প্রতিটা ধাপে একটা ইংরেজি বাক্য বলুন, বাকিটা table দেখাবে।

1. **Status (30 s):** "All four tasks are now working code. The pilot ran on Kaggle and the judge passed every check."
    - Status table দেখান।
2. **Task 1 (30 s):** "The gap is unchanged. The data itself confirms it: we found translation artefacts like নোইনপুট inside Bangla-Instruct."
3. **Task 2 (1.5 min):** "The real dataset has 342K rows, not 100K, and four sub-corpora. Our old language filter wrongly removed most of the math, so we rebuilt the cleaning."
    - Finding table-এর প্রথম তিন সারি আর "6-step pipeline" দেখান।
4. **Task 3 (1 min):** "Same design, five arms plus an optional IFD arm. We added a held-out set, the same cleaning for every arm, token counts and bootstrap significance."
5. **Task 4 (2 min, সবচেয়ে গুরুত্বপূর্ণ):** "On 1,001 rows the judge was 100% usable and stable across English and Bengali rubrics. Our arm C picks harder and better rows than random. The pilot also caught two bugs, now fixed."
    - C vs B table দেখান; সৎভাবে বলুন কোন দুটো ধাপ বাকি (human rating, toy fine-tune)।
6. **Compute (1 min):** "Scoring 100K with the 7B judge needs ~47 GPU-hours; Kaggle gives 30 per week. Three options are in the doc; we recommend a smaller judge on Kaggle, or A100 access if you can arrange it."
7. **Decisions (1 min):** checklist ধরে একটা একটা করে জিজ্ঞেস করুন, উত্তর পেলে সেখানেই tick দিন।

**মনে রাখার তিনটা সংখ্যা / Three numbers to remember:** 342,391 rows · judge agreement 0.89 / 0.80 · ~47 GPU-hours.
