# INTERVIEW_QA_BANK.md — Question Bank with Model Answers
### ~70 questions an interviewer could ask about this project, with answers

How to use: don't memorize answers word-for-word — memorize the **fact** each answer is anchored to. Numbers: 591 cases (500 orders + 91 settlements), 100% classification accuracy, 96.1% operational match rate, 95.3% clean rate, 14 needs review, 9 unresolved, 28 narrated cases, 64 tests, 12 unrecorded refunds, FX 83.00, fee 2%/3% + 18% GST, tolerances ±0.05 batch / ±0.01 order.

Legend: 🟢 opener · 🟡 follow-up · 🔴 hard/curveball

---

## A. Openers & storytelling

**A1. 🟢 Walk me through this project.**
Give the 2-minute pitch from INTERVIEW_PREP.md Part 1. End with the bug-hunt sentence — it invites the follow-up you're best prepared for.

**A2. 🟢 What problem does it solve?**
Merchants receive Razorpay money in batches, not per order. Fees (2–3% + 18% GST), refunds deducted from future batches, USD conversion, and banking noise mean the merchant's ledger, Razorpay's settlement report, and the bank statement never agree line-by-line. My system matches them deterministically, classifies every discrepancy, and explains exceptions in plain English — with AI narration that is fact-checked and never decides.

**A3. 🟢 Why is this hard? Why not just VLOOKUP order IDs?**
Because the bank has no order IDs at all — one bank credit (one UTR) covers 3–6 orders net of fees. A naive join fails on: batching (N:1), refund deductions in future batches (one order spans two settlement_ids), label mismatches (`visa_mc_domestic` vs `card`), USD vs INR for the same order, ±0.02 rounding drift, and rows that exist in only one source (ghosts, missing settlements, failed NEFT). The match path is a two-hop graph: order → settlement rows → UTR → bank credit.

**A4. 🟢 What are you most proud of?**
Two things: the architecture decision that the deterministic engine decides and the LLM only narrates — so an AI failure can never corrupt a financial result; and catching 12 refunds that the matcher itself had classified as clean matches — the system now finds the discrepancies its own success metrics were hiding.

**A5. 🟢 What was the hardest part?**
Resisting the urge to let the LLM "help" with classification. It would have been faster to build and would have impressed in a demo, but it makes the system unauditable. The second-hardest: the unrecorded refunds — the matcher looked perfect until I compared refund *intent* (ledger) against refund *evidence* (settlement) as separate sources.

**A6. 🟢 What would you do differently?**
Build the DB layer earlier (the DATA_ARCHITECTURE_REPORT identifies what files can't give: atomicity, run history, idempotent ingestion). Also: structured LLM outputs from day one instead of prose + figure extraction, and a small UI earlier to demo interim results.

---

## B. Domain: Razorpay settlement mechanics

**B1. 🟢 Explain how Razorpay settlement works.**
Customer pays → capture (T+0, cutoff ~5PM IST) → Razorpay batches all captures in a settlement window → deducts platform fee (2% domestic / 3% international) + 18% GST **on the fee only** → one settlement_id per batch → single NEFT credit to the bank with one UTR, typically T+2 working days. `net = gross − fee − (fee × 0.18)`.

**B2. 🟡 What's GST applied on?**
Only the platform fee, not the transaction. ₹10,000 at 2%: fee ₹200, GST ₹36, net ₹9,764. Merchants can claim input tax credit on it. (Getting this wrong was a designed trap — it changes every amount in the dataset.)

**B3. 🟡 What happens on refunds?**
Refunds are deducted from *future* settlement batches (5–7 working days later), appearing as negative line items against the order. **MDR is non-refundable** — a full refund of ₹15,000 still costs the merchant the ₹300 fee + ₹54 GST, so the residual on a full-refund order is exactly −(fee + GST). A partial refund can land in a *different* batch than the original — one order spanning two batches is the hardest edge case.

**B4. 🟡 What's a UTR?**
A 16-digit NEFT reference issued by the bank, not Razorpay. All rows of one settlement share one UTR; the bank statement shows one credit per UTR. It's the join key between Layer 1 (settlement) and the bank.

**B5. 🟡 What about international payments?**
Customer pays USD; Razorpay converts to INR (base_amount) at processing rate + FX markup, then fees apply on the INR base. In our synthetic data: fixed 1 USD = 83.00 INR, so ledger shows $424.24 and settlement shows ₹35,211.92 for the same order — the matcher must convert before comparing.

**B6. 🟡 What's a settlement vs a payment vs an order?**
order_id = merchant/Razorpay order; payment_id = Razorpay transaction (one order can have retries); settlement_id = batch; UTR = bank's reference for the NEFT credit of that batch; refund_id links a refund to its payment.

---

## C. Data & synthetic generation

**C1. 🟢 Why synthetic data?**
Buildathon constraint — no real merchant data. Synthetic let me plant every edge case deliberately with a traceable origin, and generate ground truth in the same pass so scoring is honest. The design doc has a messiness matrix: every artifact → which datasets it touches → what it tests.

**C2. 🟡 Walk me through the dataset.**
500 orders over 2 weeks (Aug 4–17, 2025): 493 captured, 5 failed, 2 authorized. 91 settlement batches (504 rows — some orders have 2 rows: original + refund). 100 bank rows: 90 Razorpay credits + ~10 noise (salary debits, vendor payments). Edge cases: 12 unrecorded refunds, 3 refund splits, 2 USD orders, 1 duplicate order, 1 ghost, 1 missing settlement, 1 failed NEFT, 1 duplicate UTR, 10 near-cutoff orders.

**C3. 🟡 How did you generate ground truth?**
The generator writes ground_truth.json (per order) and ground_truth_settlements.json (per settlement) **in the same pass** as the CSVs — the answer key exists before any matching, never inferred after. Later, one manual audit pass corrected the labels once; then everything was frozen. Ground truth hashes are in the audit trail.

**C4. 🟡 How do you validate generated data?**
data/validate_data.py: 28 checks, one per messiness item from the design doc — counts of failed payments, duplicate detection, USD rows, narration formats, etc. All PASS before any engine work.

**C5. 🟡 Why is data frozen? What if you need new data?**
Every hash, score, and audit entry anchors to the committed dataset. `run_pipeline.py --regenerate-data` exists only to prove the generator is correct — it deletes and regenerates data/raw, so it requires typing "yes". Freezing is what makes the audit trail verifiable.

**C6. 🔴 How would you move from synthetic to real data?**
The engine doesn't care where CSVs come from. Steps: adapters for Razorpay's real export schema (it has fee/tax/refund/net columns — my schema adds gst_on_fee explicitly); replace fixed FX with dated rate tables; label-alias map extended from real export values; then *hand-score a sample* of real cases to build an initial ground truth and measure — expect the 100% to drop, and that's the honest number.

**C7. 🔴 How did you decide on the noise in the bank statement?**
~10 non-Razorpay rows (salary debits, vendor credits). Tests noise filtering: the matcher only considers credits, joins by UTR, so noise is naturally unmatched and never force-fitted. Real statements have far more noise; the design shows the principle.

---

## D. Matching engine (architecture & algorithms)

**D1. 🟢 Explain the matching algorithm.**
Three layers, batch-first: (1) group settlement rows by settlement_id, sum net_amount, match to bank credit by UTR within ±0.05 → batch_credited / neft_failed / no_credit; (2) per order, find its settlement rows, validate amounts after FX conversion and label normalization, pre-check payment_status; (3) classify refunds (full/split/partial/only/unrecorded). Ghost detection and consistency checks (soft flags) run cross-source. Output: match_log.json, one entry per order and per settlement.

**D2. 🟡 Why batch-first?**
The bank statement is the constraint: it has no order information, only batch credits. Order-first matching against the bank is structurally impossible; you must match batch→bank, then distribute the batch across orders. A bank credit of ₹88,022.53 corresponds to 6 orders in one batch — no order-level key exists at the bank.

**D3. 🟡 How do tolerances work?**
Batch: ±0.05 (12/91 batches have ±0.01–0.02 rounding drift from per-row rounding). Order: ±0.01 (both sides are 2dp). Full refund detection: strict equality |refund| == gross. Between 0.05 and 1.00 batch diff → matched but needs_review (ambiguous band — flag, don't guess).

**D4. 🟡 How do you handle the label mismatches?**
Alias map in preprocessing: visa_mc_domestic→card, international_card→intl_card, amex_diners→amex. It's systemic (189 shared orders), so it's normalization, not exceptions — flagging it would flood the exception list and hide real issues. Post-normalization mismatches *would* be exceptions and are checked as soft flags.

**D5. 🟡 Explain the full-refund residual logic.**
Two settlement rows: original (gross>0) + refund deduction (−gross). Sum of nets = −(fee + gst) because MDR is non-refundable. The matcher *confirms* residual == −(fee+gst) and calls it matched. Flagging that −17.33 as "unexplained variance" would be a false positive.

**D6. 🟡 How does a refund split across batches work?**
Original charge settles in batch A; refund initiated later lands as a negative row in batch B. Matcher links both settlement_ids to the order, classifies REFUND_SPLIT, |refund| < gross. Both batches still reconcile independently because the refund row contributes negative net to batch B.

**D7. 🟡 How do you prevent double-counting?**
order_id is the join key for settlement rows; refund rows (gross=0, refund<0) are never counted as new charges (their negative net handles the math naturally); ghosts count toward batch net (they affect the bank credit) but are flagged, not attributed to any order; cross-batch splits contribute to both batches independently.

**D8. 🟡 What's the confidence model?**
4 statuses: matched (no special logic), matched_with_note (resolved but required special-case logic — REFUND_SPLIT, CURRENCY_MISMATCH), needs_review (human judgment — duplicates, ghosts, unrecorded refunds), hard_exception (no match possible — UNMATCHED_ORDER, NEFT_FAILED). Phase 4 maps these to 5 simplified labels, exception-code-aware (NO_CREDIT_EXPECTED stays "Reconciled (no credit due)").

**D9. 🔴 What happens if a settlement's UTR appears twice in the bank?**
Deliberate edge case (bank glitch). The matcher picks the credit closest to batch_net. In production I'd add date proximity and amount-multiple checks, and flag ambiguity rather than silently choose.

**D10. 🔴 What if two batches shared one UTR?**
Not in this dataset (91 IDs → 91 UTRs is validated). If it happened, per-batch amount matching still disambiguates, but I'd flag it — same UTR for different settlement_ids is a data-integrity smell.

**D11. 🔴 Complexity? Scale to 1M orders?**
Current: dict lookups, effectively O(n) per layer with grouping; 591 cases run in milliseconds; even 10⁵ rows is fine in-memory. At 10⁶+: DB indexes on order_id/settlement_id/utr (schema already written), incremental matching of new rows against open batches, and partitioning by settlement_date. The matching logic itself doesn't change.

**D12. 🔴 What's a soft flag vs an exception?**
Soft flags = data-quality signals (date tolerance violations, post-normalization label mismatches) — informational, never change classification. Exceptions = financial discrepancies that need action. Separating them protects trust in the exception list; mixing "this looks slow" with "money is missing" trains users to ignore both.

---

## E. The AI layer

**E1. 🟢 Why does the AI layer exist at all if the engine is deterministic?**
Exceptions are only valuable if a human can act on them fast. 2–4 sentences of plain English per case ("the refund of ₹2,558.65 was deducted in batch set_ZxZ… after the original ₹14,802.45 settled in set_V4w…") turns a flag into an action. And the Q&A agent makes all 591 cases queryable in natural language. Narration scales human attention; it never replaces judgment.

**E2. 🟢 "LLM narrates, never decides" — why?**
Three reasons: auditability (finance results must be explainable; the explanation *is* the audit, not the decision); testability (deterministic classification is regression-tested; LLM output is verified but not deterministic); blast radius (AI failure = flagged/unverified narration, never a changed result). The README's one-line version: "an AI failure can never change a correct reconciliation result."

**E3. 🟡 How do you prevent hallucinations?**
Defense in depth: (1) grounding — fixed domain-facts block + per-case real data in every prompt; (2) instruction — "quote, don't invent; don't re-classify"; (3) temperature 0; (4) automated verification — extract every stated figure, verify numerically (isclose, abs_tol 0.01) against case data + domain facts; (5) fail-closed — unextractable decimal figures = unverified; (6) validation — empty/malformed/overlong responses become explicit failure states. All 28 explanations verified.

**E4. 🟡 What can the checker still miss?**
Relational claims: "both duplicate rows appear in one batch" — both numbers real, the relationship wrong; figure checks pass. Caught once by manual review. Mitigation: deterministic data builders now state relationships explicitly so the LLM paraphrases instead of infers. Full fix: structured outputs with schema validation or a judge pass. I volunteer this limitation proactively.

**E5. 🟡 Why temperature 0? Doesn't creativity help explanations?**
Narration of pre-computed facts isn't creative work. Temperature 0 gives maximal reproducibility and kills a whole class of variance; a reconciliation narration that changes tone or *numbers* between runs is a defect.

**E6. 🟡 Walk me through the figure-extraction logic.**
Normalize Unicode first (narrow no-break space, thin space, ₹ → "Rs" — these caused false mismatches once), then regexes: currency-marked amounts (Rs/$/INR/USD prefixes/suffixes), bare decimals ("residual of 2196.99"), 2025-XX-XX dates. Dedupe. Bare amounts were added after noticing marker-less figures would have been silently skipped — now they're verified like any other, and if decimals exist but extraction yields nothing, verification fails closed.

**E7. 🟡 Why numeric comparison instead of string matching?**
The original substring matcher was a real bug: "2" verified against 47,500.00. Fixed with math.isclose(abs_tol=0.01) for numerics (83 matches 83.00), exact string only for non-numerics (IDs, dates). Regression tests pin the negative cases: 2↛47500.00, 18↛1180.00, 83↛8383.00.

**E8. 🟡 What's the Q&A agent's architecture?**
Pure read-retrieve-respond: load 4 artifacts once (singleton, lazy); classify question via regex — single_case (ord_/set_), aggregate (default), out_of_scope (future/external keywords); retrieve — full case entry + match_log enrichment + existing explanation reused verbatim for single_case; precomputed summary + metrics for aggregate (the LLM never counts raw data); generate at temp 0; verify figures; return structured {answer, source_case_ids, verified, category, fallback_reason}. Writes nothing.

**E9. 🟡 What happens when the API is down mid-demo?**
Graceful: retries with exponential backoff (429 → 2/4/8s; 5xx → one retry; 401 → hard stop with clear message), then answer_question catches everything and returns the structured api_error fallback — polite message, verified=false, fallback_reason set. The dashboard shows a warning, never a traceback; the rest of the app is unaffected. Explanations that fail mid-run are stored with validation.reason="api_error" — visible failure states, not silent gaps.

**E10. 🔴 Why not use embeddings/RAG?**
No unstructured corpus to search — the knowledge is 591 structured records. Direct dict lookups by case_id and precomputed aggregates are exact, instant, and auditable. RAG would add a lossy retrieval step to a system whose whole point is exactness. If documents (Razorpay policy PDFs) entered scope, retrieval over *documentation* with structured data still injected directly would make sense.

**E11. 🔴 Why not function-calling / tool-use agents?**
The pipeline is a fixed DAG over frozen files; "tools" would be the four JSON loads I already do deterministically. Tool-calling earns its complexity when the *sequence* of data access is unknown at design time. Here it's known, so determinism beats flexibility. I'd introduce tools for multi-step investigation of needs_review cases (pull bank rows, recompute residuals) — that's a real future extension.

**E12. 🔴 How do you evaluate LLM output quality beyond the figure check?**
Currently: automated figure verification + format validation + human review of all 28 outputs (small enough to review exhaustively). The relational-claim miss is the documented gap. Next steps I'd build: golden-set regression prompts, structured-output schemas, LLM-as-judge on a sample, and tracking verified-rate + mismatch-rate per run as first-class metrics.

**E13. 🔴 gpt-oss-120b on Groq — why this model? Would a bigger model help?**
The task is grounded narration of facts my engine computed — the hard reasoning is deterministic. Requirements: instruction-following (stay in 2–4 sentences, don't re-classify), stable numerics, low cost/latency. gpt-oss-120b met all four on the free tier. A frontier model wouldn't reduce errors the checker catches; better *grounding and verification* would. Model is one swappable string.

**E14. 🔴 What if the model changes or Groq deprecates it?**
The provider is abstracted behind the openai SDK with a base_url; model is a config string. The verification layer is model-agnostic — any model's output gets the same figure checks. Worst case: explanations fail validation/verification and get flagged; the deterministic pipeline is unaffected. That's the architecture working as designed.

---

## F. Metrics & honesty

**F1. 🟢 How do you measure accuracy?**
Compare each case's exception_code against ground truth (vocabulary bridge for settlements: report code → expected bank-credit status). Per-code TP/FP/FN/TN with precision/recall/F1 — the "none" (matched) class uses the same generic formula. Plus FPR (flagged-clean things wrongly) and FNR (silently matched exceptions). Every mismatch listed individually with case_id, expected, actual — no anonymous counts.

**F2. 🟡 100% accuracy — really? Defend that.**
Against synthetic ground truth, and the README scopes it explicitly: generator and matcher share assumptions, so it proves internal consistency with the reference set (which was also manually audited once), not real-world accuracy. The claims I stand behind: 0 false negatives, and the 3.9% flagged cases are genuine ambiguities surfaced, not guessed. Ask me how I'd measure on real data — adapters, then hand-scored sample. (F2 paired with F6.)

**F3. 🟡 Why are there two match rates?**
96.1% operational: resolved without human help (includes 5 correctly-handled note cases + 1 no-credit-due). 95.3% clean: cases needing no exception at all. Gap = 0.85pp = the 5 with-note cases (3 REFUND_SPLIT + 2 CURRENCY_MISMATCH), itemized by code in the report. Definitional difference, documented, not an error. I report both because single flattering numbers invite mistrust.

**F4. 🟡 FPR is 0.0018 but you claim 100% accuracy — contradiction?**
No — different lenses. The ghost batch is *correctly credited* (batch level) but flagged needs_review (one of its 6 orders isn't in the ledger). GT has no GHOST vocabulary, so code-equality scoring says correct while the binary matched-vs-exception view counts one FP. Documented in the metrics report. Deliberate choice: surface ambiguity rather than suppress it. FNR = 0.0 is the metric that matters in finance.

**F5. 🟡 What's a false negative here and why is zero important?**
Ground truth says exception, system says matched — silently swallowed money movement. In reconciliation, a missed exception is real money unaccounted for; a false positive just costs review time. The scorer lists every FN individually. Result: FNR = 0.0 — nothing was silently matched.

**F6. 🔴 How do you know your ground truth is right?**
It was generated with the data from the same deterministic rules, then one manual audit pass corrected it once (recorded in the audit trail: "updated once"), then frozen with hashes. The known residual risk: generator and matcher could share a wrong assumption — which is exactly why I scope the 100% as internal consistency + audit, not universal truth. Independent checks that partially mitigate: 28-validator suite, the audit that *changed* ground truth once (proof the audit had teeth), and the refuted-claim investigation.

---

## G. Testing & quality

**G1. 🟢 Describe your testing strategy.**
64 tests, 14 classes, stdlib unittest, ~0.1s, zero API calls. Unit: matcher layers, FX, refunds, exceptions, status mapping, metrics arithmetic, QA parsing, explanation validation, fail-closed verification. Integration: full frozen dataset — counts, completeness, determinism, report structure — plus a reproducibility pin (fresh matcher run hash-matches committed match_log.json). LLM calls deliberately excluded.

**G2. 🟡 Why no pytest?**
 unittest covers everything needed (fixtures via setUp, subTest, assertions) with zero dependency; the project's rule is stdlib-first. pytest would buy slightly nicer ergonomics, not more safety. If the suite grew to need fixtures/plugins/parametrize at scale, I'd switch without sentimentality.

**G3. 🟡 Why exclude LLM calls from tests?**
Determinism, speed, cost, offline. LLM-dependent behavior is instead guarded at runtime (validation, figure verification, fail-closed states) and the deterministic helpers (extract, verify, validate) are tested directly with fixed fixtures. Testing "the model said something nice" is testing a random variable.

**G4. 🟡 What does the reproducibility pin do?**
TestMatcherOutputReproducibility runs the matcher in-memory on the frozen CSVs, serializes exactly like compile_match_log, and asserts SHA-256 equals the committed match_log.json. Any engine change that alters results fails the suite instead of silently drifting from the audit trail. It converts "we think it's deterministic" into a CI-enforced invariant.

**G5. 🟡 What do the integration tests check on the full dataset?**
500 + 91 counts exactly; every entry has required fields; two runs produce identical confidence per order (determinism); the committed reconciliation report loads with the right structure; entry counts match the audit trail.

**G6. 🔴 If you had one day to improve quality, what would you do?**
Mutation-style testing on the matcher (perturb a CSV amount, assert the right exception fires); property-based tests for refund/residual invariants (residual == −(fee+gst) must hold for *any* full refund); a golden-set of Q&A prompts with expected categories; and CI running the suite + hash pin on every push.

---

## H. Integrity, security, ops

**H1. 🟢 Explain the audit trail.**
audit_trail.md = chain of custody (SHA-256 of all 9 files, computed live, never hand-copied), the bug-fix narrative with verification steps, the NEFT_FAILED graceful-failure case, and 3 fully-traced examples showing one case across all 5 phases. Each phase's metadata embeds its inputs' hashes; each generator re-verifies inputs after writing. Claim: the reported numbers were computed from exactly these inputs, by exactly this code.

**H2. 🟡 Why did hashes break across platforms?**
CRLF: Windows checkouts produced different bytes → different hashes. Fix: .gitattributes forces LF, and sha256() normalizes \r\n→\n as defense in depth. Lesson: content-addressing requires byte-level discipline (writes use newline="" and sorted IDs; timestamped outputs are *documented* as hash-unstable).

**H3. 🟡 How are API keys handled?**
GROQ_API_KEY in .env, gitignored; .env.example committed as template; loaded via python-dotenv; clear error and exit if missing; tests never need keys. Never hardcoded, never logged.

**H4. 🟡 What's NOT production-ready about it?**
I wrote the answer in DATA_ARCHITECTURE_REPORT.md: file-based storage (no atomicity, no run history, no idempotent ingestion, no concurrency control), fixed FX, no auth layer on the dashboard, explanations semantically non-deterministic across runs. The matcher logic is production-shaped; the storage and ingestion layers are the gap.

**H5. 🔴 How would you monitor this in production?**
Business metrics as alerts: exception-rate SLO (spike = upstream data shift), UNRECORDED_REFUND count (any new one = incident), NEFT_FAILED SLA timers to bank ops, verified-rate of AI narration (drop = grounding regression), input-hash drift (schema change detection). Plus operational: pipeline duration, retry rates, per-phase failure alarms. Every phase already emits structured counts — the hooks exist.

---

## I. Curveballs & pressure questions

**I1. 🔴 Your ground truth came from the same rules as your matcher. Isn't the 100% circular?**
Partially — and I say so in the README. The score proves the engine is internally consistent with the reference set. What breaks the circularity: the manual audit pass that corrected ground truth once (independent human judgment), the audit trail that records *both* fixes and a refuted claim (independent reasoning), and edge cases whose classification required cross-source reasoning (unrecorded refunds weren't knowable from generator rules alone — they were found by comparing intent vs evidence). But the honest framing stands: synthetic GT = internal consistency, not real-world accuracy.

**I2. 🔴 An interviewer says: "I could do this with SQL joins in an afternoon."**
"You could get the easy 90% — order_id join between ledger and settlement works for plain rows. The last 10% is the product: batch→UTR→bank joins that SQL does awkwardly (N:1 aggregation with tolerance), refund intent vs evidence cross-checks, currency and label normalization, and a decision of what to do with ambiguity. My matcher is ~the same set operations, but with explicit classification of *why* anything doesn't match, tests, and hashes. And the SQL version still has no answer layer for humans — that's where the AI part lives. I'd genuinely enjoy seeing the SQL version; it would be a great validator to run against my engine."

**I3. 🔴 Why is this resume-worthy? It's 500 rows of fake data.**
"The dataset size is irrelevant; every mechanism is real: three-source reconciliation with batching/refunds/FX, a decision-vs-narration AI architecture with automated verification, ground-truth evaluation methodology, audit-grade provenance, and a documented bug hunt that caught the system lying to itself. Those transfer directly to real reconciliation volumes — and the architecture report sketches the production path."

**I4. 🔴 What's the weakest part of the project?**
"The verification ceiling: figure-checking catches wrong numbers but not wrong relationships — I have one documented miss there. Second: file-based storage. I wrote both up in the repo (README 'AI Layer Evaluation' and DATA_ARCHITECTURE_REPORT) rather than paper over them."

**I5. 🔴 The buildathon rejected this project. Why do you think so, and what did you learn?**
Suggested framing (be honest, non-defensive, growth-oriented): "Judging at a buildathon is a different game than engineering correctness — scope/flash/novelty against hundreds of entries, and honest-limitation-heavy writeups read as less flashy even though they're more rigorous. What I took from it: lead with the demo and the story, keep the receipts in the appendix; and the engineering habits I built here — verification that fails closed, provenance, honest metrics — are exactly what production teams ask for in interviews. The project got stronger after submission: the numeric checker fix and the reproducibility pin came from continuing to audit it."

**I6. 🔴 A stakeholder says "the 23 flagged cases are too many, just auto-resolve them."**
"Those 23 aren't errors — they're the cases where money is ambiguous: 12 refunds recorded in our books but absent from settlement, a duplicate order with two different amounts, a ghost order, 8 orders that never settled. Auto-resolving means guessing with money: writing off refunds the processor may owe us, or picking an amount for the duplicate. Each needs a different human action — escalate to bank ops, verify against the source system, void/retry. The 96.1% automated rate *is* the value; forcing 100% automation converts a trustworthy system into an expensive random-number generator."

**I7. 🔴 What happens if I give you 10,000 more orders with messiness you didn't design for?**
Unknown messiness falls into buckets by construction: rows that don't join (→ unmatched/ghost paths with details), amounts that don't agree (→ AMOUNT_MISMATCH/review, never silent), statuses I haven't seen (→ UNEXPECTED_SETTLEMENT-style defensive codes already implemented). What I *can't* promise: correct classification of a genuinely new pattern — I'd expect a burst of needs_review cases, which is the designed behavior: flag, don't guess. Then extend rules + tests from the reviewed cases.

**I8. 🔴 Defend one design choice you'd argue about with a senior engineer.**
Example answer (LLM layer): "I'd defend narration-only AI even though a senior might say 'let the LLM classify with structured output, it's 2026.' My counter: finance classification needs per-case explainability grounded in *provable* data; an LLM's classification is a probabilistic artifact. The moment a wrong LLM classification overwrites a correct deterministic one, you've lost the audit trail. Structured outputs + verification narrow this but don't eliminate it; and my deterministic core is cheap to run. I'm open to LLM *triage* (prioritizing review queues) — a lane where errors are cheap."

**I9. 🔴 Explain something technical from this project to me as if I were a CFO.**
"The three files are three witnesses to the same money: your books, Razorpay's statement, and the bank. They never match line-by-line because Razorpay bundles payments, takes fees, and handles refunds later. My system reconstructs the money trail batch by batch, tells you exactly which rupees are unaccounted for, and explains each one in plain language. It found ₹62,386 that was supposed to arrive and never did — the kind of thing that otherwise surfaces at month-end, or never."

**I10. 🔴 You had 28 commits over 11 days. Walk me through your process.**
"Research first (Phase 0 commit is literally docs), then schema + generator with validation, then the matcher with tests, then AI on top of a proven deterministic core, then reporting/metrics, then audit, then UX — each phase frozen before the next consumed it. The later commits are almost entirely *hardening found by audit*: bug fixes, the numeric checker, hash stability, reproducibility. The shape of that history — few features at the end, mostly verification — is what I'd consider healthy."

---

## J. Rapid-fire definitions (say these in one breath each)

- **Batch-first matching** — match settlement batches to bank credits by UTR, then distribute to orders; forced by the bank's batch-level view.
- **Narrate, don't decide** — LLM explains pre-computed classifications; it can never alter them.
- **Fail-closed verification** — unverifiable = unverified = flagged; a pass never means "skipped."
- **Ground truth** — per-case expected labels generated with the data, audited once, frozen, hashed.
- **Ghost transaction** — settlement row whose order_id doesn't exist in the ledger.
- **UNRECORDED_REFUND** — ledger claims a refund; settlement has no evidence. Found via intent-vs-evidence cross-check.
- **NO_CREDIT_EXPECTED** — negative-net batch; no bank credit is *correct* behavior.
- **matched_with_note** — resolved correctly, but required special-case logic (FX conversion, refund split); kept distinguishable from plain matches.
- **Chain of custody** — SHA-256 of every pipeline file, computed live, embedded in each phase's metadata.
- **Reproducibility pin** — test asserting a fresh matcher run hash-matches the committed output.
- **Two-source verification** — a stated figure must match case data OR curated domain facts.
- **Operational vs clean match rate** — resolved-without-humans (96.1%) vs no-exception-needed (95.3%); gap explained, not hidden.
