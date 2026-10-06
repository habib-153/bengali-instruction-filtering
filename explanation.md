# সহজ ব্যাখ্যা — Supervisor ও Team-এর জন্য
### (Plain-language guide to the submitted Tasks 1–4)

এই ফাইলটা লেখা হয়েছে যাতে আমরা supervisor-কে এবং একে অপরকে সহজে বুঝিয়ে বলতে পারি আমরা কী জমা দিয়েছি। জমা দেওয়া আসল document হলো `bengali_filtering_tasks_1-4_humanized.docx` (markdown কপি: `bengali_filtering_tasks_1-4_humanized.md`)। Technical file গুলো (Task1–Task4) এখন সেই submission অনুযায়ী update করা। এই ফাইলটা সেগুলোর "human explanation", মিটিং-এ বলার মতো ভাষায়।

> **আগের draft থেকে কী বদলেছে (এক নজরে):** আগে আমরা BanglaLlama-এর unfiltered data (Bangla-Orca/Alpaca) নিয়ে dedup → language filter → LLM-judge, এভাবে ৩টা stacked filter আর V0–V3 / R1–R3 run-এর plan করেছিলাম। জমা দেওয়া version-এ এগুলো বদলে গেছে: **pool এখন TigerLLM-এর Bangla-Instruct (100K)**, **strategy এখন DEITA-style (complexity × quality + diversity)**, আর **experiment এখন ৫টা arm (A–E)**, TigerLLM-এর model, recipe আর benchmark হুবহু রেখে।

---

## ১. পুরো Project-টা আসলে কী নিয়ে?

**সহজ কথায়:** ইংরেজিতে দেখা গেছে, বড় instruction dataset থেকে ভালো, জটিল আর বৈচিত্র্যময় উদাহরণগুলো বেছে একটা ছোট subset বানালে, সেটা দিয়ে train করা model পুরো dataset দিয়ে train করা model-এর সমান বা তার চেয়ে ভালো হয় (DEITA, AlpaGasus, Ivison et al. 2025)। প্রশ্ন হলো:

> **বাংলাতেও কি একই জিনিস কাজ করে?** নাকি ভালো result আসলে শুধু data কম হওয়ার কারণে?

বাংলা আলাদা কারণ এর script আলাদা, tool (embedding, scorer, tokenizer) দুর্বল, আর data-র বড় অংশ machine translation থেকে আসা। তাই ইংরেজির result বাংলায় খাটবে, এটা ধরে নেওয়া যায় না। আমরা এটা সরাসরি test করব: model আর training একদম fixed রেখে, **শুধু data কীভাবে বাছাই হলো সেটা বদলে**।

---

## ২. Task 1 — Research Gap
*(ফাইল: `Task1_Screening_Extraction_and_Gap.md`)*

**এক লাইনে gap:** ইংরেজিতে data selection কাজ করে বলে প্রমাণিত, বাংলায় controlled ভাবে কেউ test করেনি।

ফাইলে যা আছে:
- **জমা দেওয়া gap statement** (একদম উপরে)।
- **Screening:** ১৪টা paper-এর মধ্যে ১০টা Include, ৪টা Unsure। আগের মতোই আছে।
- **Extraction cards:** প্রতিটা paper-এর dataset, method, result, limitation। TigerLLM, DEITA, AlpaGasus-এর "কেন দরকারি" অংশ নতুন design অনুযায়ী update করা।
- **Gap synthesis:** TigerLLM-এর filter শুধু pass/fail করে, সব রেখে দেয়। Instruction complexity আলাদা করে score করে না, diversity দেখে বাছাইও করে না। আর পুরো 100K-কে কখনো ছোট selected subset বা random subset-এর সাথে compare করেনি। **এটাই আমাদের entry point।**

**খেয়াল রাখার মতো:** submission-এ "Ivison et al., 2025" cite করা হয়েছে, যেটা আমাদের tracker-এর "Large-Scale Data Selection for Instruction Tuning" paper। এটা এখনো পুরো পড়া হয়নি, **পড়া দরকার**।

---

## ৩. Task 2 — Filtering Strategy ও Data Pool
*(ফাইল: `Task2_Strategy_Selection_and_Data_Pool.md`)*

**Data pool:** TigerLLM-এর **Bangla-Instruct**, 100,000টা native বাংলা instruction-response pair (GPT-4o আর Claude-3.5-Sonnet দিয়ে generate করা, 500টা seed task থেকে)। TigerLLM আগেই ৪টা জিনিস check করে filter করেছে: language adherence, cultural sensitivity, content quality, আর novelty (duplicate)।

**আগের ৩টা strategy কোথায় গেল?** TigerLLM-এর filter আগেই সেগুলোর কাজ করে দিয়েছে:

| আমাদের আগের strategy | TigerLLM-এর কোন check সেটা করে |
|---|---|
| Semantic deduplication | Novelty check |
| Script / translationese filtering | Language-adherence check |
| LLM-judge response quality | Content-quality check |

তাই এগুলো এখন **pre-filter হিসেবে আগে থেকেই apply করা**, আমাদের experiment-এর variable না।

**আমাদের নতুন strategy (DEITA থেকে নেওয়া):**
1. **Complexity score (c):** LLM দিয়ে প্রতিটা instruction কতটা জটিল তার score।
2. **Combined score s = c × q:** q হলো response-এর quality score। এটা DEITA-র formula। **"quality squared" না**, কারণ তাতে complexity পুরোপুরি বাদ পড়ে যায়। *(submission-এ এটা flag করা আছে। Supervisor আসলে কোনটা চেয়েছিলেন confirm করতে হবে।)*
3. **Diversity-aware selection:** s অনুযায়ী সাজিয়ে একটা একটা করে pair নেওয়া হবে, কিন্তু শুধু তখনই, যখন সেটা আগে নেওয়া সবগুলো থেকে embedding-এ যথেষ্ট দূরে (threshold τ)। এভাবে 40K পর্যন্ত নেওয়া হবে।

**আগে DEITA-style বাদ দিয়েছিলাম, এখন কেন নিলাম?** আগে ভেবেছিলাম আলাদা scorer model train করতে হবে, যেটা খরচসাপেক্ষ। এখন আমরা সরাসরি LLM-judge prompt দিয়ে score করব, তাই সেই খরচ নেই।

**V0 নিয়ে আগের প্রশ্নের সমাধান:** আগে প্রশ্ন ছিল baseline হবে BanglaLlama-এর unfiltered data, নাকি TigerLLM-এর filtered data। জমা দেওয়া version-এ **TigerLLM-এর Bangla-Instruct** নেওয়া হয়েছে। তাই আমাদের প্রশ্নটা এখন: "already-filtered pool থেকে আরও ভালোভাবে বাছাই করলে কি উন্নতি হয়?"

---

## ৪. Task 3 — Experimental Comparison
*(ফাইল: `Task3_Experimental_Comparison_Design.md`)*

**যা fixed থাকবে (TigerLLM-এর হুবহু):**
- **Model:** LLaMA-3.2 (1B)। 9B-র জন্য বড় GPU cluster লাগে, তাই সেটা stretch goal।
- **Training:** full fine-tuning (LoRA না), TigerLLM-এর hyperparameter-এ: 3 epoch, lr 1e-5, batch 16, seq len 2048, ইত্যাদি।
- **Evaluation:** TigerLLM-এর ৬টা বাংলা benchmark (MMLU-bn, PangBench-bn, BanglaQuaD, mHumanEval-bn, BEnQA, BanglaRQA), Pass@1।

**৫টা arm:**

| Arm | Data | মানে সহজ ভাষায় |
|---|---|---|
| A | 100K (পুরোটা) | TigerLLM যা release করেছে, হুবহু। এটাই baseline |
| B | 40K random | Random ভাবে বাছা 40K। শুধু data কম হলে কী হয়, সেটা দেখতে |
| C | 40K (আমাদের) | complexity × quality + diversity দিয়ে বাছা। **আমাদের মূল hypothesis** |
| D | 40K | শুধু quality score দিয়ে বাছা। complexity-র অবদান দেখতে |
| E | 40K | complexity × quality, কিন্তু diversity step ছাড়া। diversity-র অবদান দেখতে |

D আর E শুধু তখনই চালানো হবে, যদি A–C শেষ হওয়ার পর compute বাকি থাকে।

**মূল তুলনা:**
- **C vs B** (দুটোই 40K): উন্নতি কি আসলে ভালো বাছাইয়ের জন্য, নাকি শুধু কম data-র জন্য?
- **C vs A** (40K vs 100K): ছোট selected set কি পুরো set-এর সমান বা ভালো?

**ঝুঁকি যা মাথায় রাখা আছে (ফাইলের Part D):**
- TigerLLM-1B fine-tuning-এর আগে textbook data দিয়ে extra pretraining করেছিল। আমরা সেটা না করলে Arm A তাদের published number-এর চেয়ে কম আসতে পারে।
- Bangla-Instruct GPT-4o/Claude দিয়ে বানানো। Score করার judge-ও যদি একই family-র হয়, তাহলে bias (circular) হতে পারে।
- Complexity বেশি মানে সাধারণত লম্বা instruction। তাই C-তে token বেশি থাকতে পারে, যে কারণে token count-ও report করতে হবে।
- AlpaGasus-এ দেখা গেছে score দিয়ে filter করলে coding example বেশি বাদ পড়ে। তাই mHumanEval-bn আলাদা করে দেখতে হবে।

---

## ৫. Task 4 — Feasibility Test (ছোট পরীক্ষা)
*(ফাইল: `Task4_Feasibility_Pilot.md`)*

**সহজ ভাষায়:** পুরো 100K score করার আগে **500–1,000টা pair** দিয়ে ছোট একটা পরীক্ষা চালাব, পুরো pipeline ঠিকমতো চলে কিনা দেখতে:

1. Bangla-Instruct থেকে random 500–1,000টা pair নেওয়া।
2. LLM judge **সরাসরি বাংলায়** (অনুবাদ করে না) consistent আর parseable score দেয় কিনা দেখা।
3. Embedding model বাংলায় ঠিক কাজ করে কিনা দেখা: প্রায় একই রকম pair কাছাকাছি, আলাদা pair দূরে থাকে কিনা।
4. ছোট একটা selection চালিয়ে হাতে দেখা, বাছাই করা গুলো random-এর চেয়ে ভালো আর বৈচিত্র্যময় লাগে কিনা।
5. LLaMA-3.2 (1B)-কে LoRA দিয়ে কয়েকশো step train করা। শুধু code আর hardware চলে কিনা দেখতে, result দেখার জন্য না।
6. সময় আর খরচ লিখে রেখে 100K-এর জন্য হিসাব করা।

**কখন থামতে হবে (ফাইলের Part B):** judge-এর score অস্থির বা parse না হলে, judge বাংলা ঠিকমতো না বুঝলে, embedding duplicate ধরতে না পারলে, বাছাই করা data random-এর চেয়ে ভালো না লাগলে, বা খরচ budget ছাড়িয়ে গেলে।

---

## ৬. Supervisor Meeting-এ যা বলা যেতে পারে (Talking points)

1. "আমাদের gap হলো: ইংরেজিতে data selection কাজ করে প্রমাণিত, বাংলায় controlled ভাবে কেউ test করেনি।" *(Task 1)*
2. "আমরা TigerLLM-এর Bangla-Instruct (100K) নিয়েছি। TigerLLM-এর filter আগেই dedup আর language check করে, কিন্তু complexity আর diversity দিয়ে বাছাই করে না। আমরা DEITA-র method দিয়ে সেটা যোগ করছি।" *(Task 2)*
3. "Score হলো complexity × quality, DEITA-র formula। Task note-এ 'quality squared' লেখা ছিল বলে মনে হয়েছে। আপনি কি সেটাই চেয়েছিলেন, নাকি c × q ঠিক আছে?" *(Task 2: **supervisor-কে জিজ্ঞেস করার প্রশ্ন**)*
4. "TigerLLM-এর model, training recipe আর ৬টা benchmark হুবহু রেখে ৫টা arm চালাব: পুরো 100K, random 40K, আমাদের 40K, আর budget থাকলে দুটো ablation।" *(Task 3)*
5. "শুরুর আগে 500–1,000টা pair দিয়ে pilot চালাব, যাতে দেখা যায় LLM judge বাংলায় ঠিকমতো score করে কিনা আর খরচ কত।" *(Task 4)*

---

## ৭. এখনো যা Confirm করা বাকি (Open items)

- **c × q নাকি quality²:** supervisor-এর কাছ থেকে confirm করতে হবে।
- **Base checkpoint:** raw LLaMA-3.2-1B, নাকি TigerLLM-এর textbook-pretrained checkpoint? Arm A-র number এর উপর নির্ভর করে।
- **Judge model আর embedding model:** এখনো ঠিক করা হয়নি। Pilot-এ ঠিক হবে।
- **Seeds:** submission-এ উল্লেখ নেই। ৩টা seed সম্ভব হলে ভালো।
- **Ivison et al. (2025)** paper-টা submission-এ cite করা, কিন্তু এখনো পুরো পড়া হয়নি। ৪টা Unsure screening আর ৪টা Partial extraction এখনো বাকি।
- Bangla-Instruct-এর license আর আসল row count এখনো verify করা হয়নি।
