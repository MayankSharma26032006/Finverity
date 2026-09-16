# INTERVIEW_PREP.md — Full Deep-Dive
### Finance Controller — Razorpay AI Buildathon (reconciliation system)

This is your complete interview war chest for this project. Everything here is verified against the actual repo (state as of commit `d1bb4c2`, Sep 4, 2026). Read this top-to-bottom once, then keep INTERVIEW_CHEATSHEET.md for revision and INTERVIEW_QA_BANK.md for drilling.

---

## Part 1 — The 30-second and 2-minute pitches (memorize these)

### 30-second version
> "I built an agent-assisted reconciliation system for a Razorpay buildathon. Merchants receive settlement money in batches, not per order — fees, GST, refunds, and currency conversion make matching genuinely hard. My engine deterministically matches orders across three noisy data sources — an internal order ledger, Razorpay's settlement report, and a bank statement — and an AI layer narrates the exceptions, never decides them. Every AI statement is automatically fact-checked against source data. Final result: 100% classification accuracy on 591 cases against ground truth, 96.1% clean reconciliation rate, 64 regression tests, zero silent failures."

### 2-minute version (structure: problem → why hard → what I built → guardrails → results)
> "The problem: when you pay a merchant through Razorpay, the money doesn't arrive per order. Razorpay batches multiple orders into one settlement, deducts a 2% fee plus 18% GST on the fee, and sends a single NEFT bank credit with one UTR reference. Refunds are deducted from *future* batches, a partial refund can split one order across two settlement batches, and international orders convert USD to INR. So the three sources — the merchant's order ledger, Razorpay's settlement report, and the bank statement — see the same money through completely different lenses.
>
> I designed a synthetic dataset of 500 orders with every edge case deliberately baked in, plus ground-truth labels generated in the same pass as the data. Then a three-layer deterministic matcher: batch-level (settlement to bank credit by UTR with a ±₹0.50 tolerance), order-level (distributing batches back to orders, with currency conversion and label normalization as preprocessing), and refund classification (full, partial, and cross-batch splits).
>
> The key architectural decision: the matcher is fully deterministic and testable. The AI layer — Groq's gpt-oss-120b at temperature 0 — only narrates the 28 exception cases in plain English. It never re-classifies anything. And every number it states is automatically extracted and cross-verified against the case data and a fixed domain-facts block; verification fails closed. The Q&A agent lets you ask natural-language questions about any of the 591 cases with the same guardrails.
>
> Results: 100% classification accuracy on 591 cases, 96.1% reconciled without human help, 23 cases honestly flagged for review or escalation instead of guessed on, zero false negatives, 64 regression tests in ~0.1 seconds. Along the way the audit process caught real bugs — including 12 refunds silently missing from settlement that the matcher had classified as clean matches — which I fixed and regression-locked."

---

## Part 2 — The story arc (this is what interviewers actually remember)

This is the single most important section. Interviews about projects go **chronologically**: "walk me through how you built this." Here's the true story, from commit history:

### Chapter 0 — Research before code (Aug 25, 2026, Phase 0)
- First commit was research, not code: *how does Razorpay settlement actually work?*
- Findings that shaped everything: 2% platform fee + 18% GST on the fee (GST is on the fee, not the amount); T+2 working-day settlement cycle with ~5PM IST cutoff; batching into one NEFT credit with one UTR; **MDR is non-refundable even on full refunds**; USD payments convert to INR before settlement.
- **Why interviewers care:** you understood the domain before writing a line. This is what separates senior candidates.

### Chapter 1 — Design the mess first (Aug 25, Phase 1)
- Wrote `docs/DESIGN_PHASE1.md`: full schema for 3 CSVs + ground truth, with a *messiness matrix* mapping every edge case to which datasets it touches and what it tests.
- Built `data/generate_data.py`: seeded, deterministic generator producing 500 orders, 91 settlement batches, 100 bank rows, and ground-truth JSON **in the same pass** — "the answer key is generated alongside the data, never inferred after."
- Edge cases deliberately baked in: 5 failed payments, 2 authorized-not-captured, 3 refund splits, 2 USD orders, 10 near-cutoff orders, 1 duplicate order, 1 ghost transaction, 1 missing settlement, 1 failed NEFT, 1 duplicate UTR, ~12 rounding-variance batches, label mismatches between systems.
- `data/validate_data.py`: 28 independent validation checks, all PASS.
- **Killer line:** "I generated my own ground truth *with* the data, so scoring was never circular or post-hoc."

### Chapter 2 — The matching engine (Aug 26–28, Phase 2)
- The core insight (say this proudly): **the bank never sees orders — it sees batch credits. So matching must be batch-first, top-down, not order-by-order.**
- Three layers: `batch_matcher.py` (Layer 1: settlement_id ↔ bank UTR, ±0.05 tolerance), `order_matcher.py` (Layer 2: order_id ↔ settlement rows, with payment_status pre-checks), `refund_classifier.py` (Layer 3).
- Preprocessing: label alias map (ledger says `visa_mc_domestic`, settlement says `card` — normalization, NOT exceptions), USD→INR at FX 83.00.
- A 4-status confidence model: `matched` / `matched_with_note` / `needs_review` / `hard_exception`.
- Double-counting prevention rules (ghost rows count in batch net but not per-order; refund rows are negative nets, not new charges).

### Chapter 3 — AI that never decides (Aug 28, Phase 3)
- `agent/explainer.py`: exactly 28 non-trivial cases narrated (at design time it was 17; the unrecorded-refund discovery grew it to 28). The other 563 plain-matched cases never touch an LLM.
- Groq `openai/gpt-oss-120b`, temperature 0, max_tokens 800. Prompt = system instructions + fixed domain-facts block + case data. "Quote, don't invent."
- Hallucination safeguard: extract every figure the LLM states → cross-verify against case data + domain facts. Fail-closed behavior.

### Chapter 4 — Honest reporting (Aug 27, Phase 4)
- `engine/reconciler.py`: collapses the 4-status model into 5 judge-readable labels: Reconciled / Reconciled (with note) / Reconciled (no credit due) / Needs Human Review / Unresolved.
- Pure read-merge-write, zero decisions. Completeness assertions abort if not exactly 500 + 91.
- Labels chosen for honesty: "Needs Human Review" not "Pending"; "Unresolved" not "Error"; "Reconciled (no credit due)" is semantically opposite of unresolved.

### Chapter 5 — Measure yourself (Sep 3, Phase 5)
- `engine/metrics_scorer.py`: compares every case's exception_code against ground truth. Per-code TP/FP/FN/TN, precision/recall/F1, FPR/FNR. Every mismatch listed individually with case_id — no anonymous aggregate counts, ever.
- Handles the subtle cases: duplicate order has 2 ground-truth entries → exception entry wins; GHOST_TRANSACTION has no GT vocabulary → scored via credit-status mapping.
- Result: 591/591 correct, mismatches = [].

### Chapter 6 — The bug hunt (Sep 3–4, the differentiator)
Four real findings from external audit (detailed in Part 8 below): duplicate-order suppression, 12 unrecorded refunds, negative-net mislabeling, and one refuted claim. **Also a documentation-integrity bug:** hashes were unstable across platforms because of CRLF line endings → `.gitattributes` + CRLF-normalizing sha256.

### Chapter 7 — Make it talk and show it (Aug 28 → Sep 3, Phases 7–8)
- `agent/qa_agent.py`: classify question → single_case / aggregate / out_of_scope → ground with real data → LLM → verify figures. Structured fallbacks (never crash, never guess).
- `dashboard.py`: Streamlit, 5 sections, dark fintech theme, all data cached, Q&A is the only live component.
- `run_pipeline.py`: unified runner with flags; `--regenerate-data` guarded behind an interactive "type yes" prompt because data is frozen for a reason.

### Chapter 8 — Hardening (Sep 3–4)
- Hallucination-checker numeric fix (the "2 vs 47,500.00" bug — Part 8), explanation validation (fail-safe failure states), fail-closed verification, reproducibility pin (fresh matcher run must hash-match committed match_log.json), test suite grown to 64.

---

## Part 3 — The stack (and why each choice)

| Choice | What | Why (say it like this) |
|---|---|---|
| Python 3.8+ | Entire codebase | Fintech logic + data wrangling; stdlib-first |
| **stdlib-first** (csv, json, hashlib, unittest, re, math, datetime) | All core processing | Zero framework lock-in, fully transparent, everything auditable. "If I can do it with stdlib, I don't add a dependency." |
| Groq + openai SDK | LLM narration (Phase 3) + Q&A (Phase 7) | Free tier was enough for 28 one-shot calls; openai SDK pointed at Groq's base_url = no new dependency |
| `openai/gpt-oss-120b` | The model | Sufficient for *narration of pre-computed facts* — the hard reasoning was already done deterministically |
| **Temperature 0** | All LLM calls | Reproducibility. An explanation pipeline must not be creative. |
| Streamlit | Dashboard | Fastest path to a judge-usable UI; `@st.cache_data` for frozen data |
| pandas | Dashboard chart + table styling only | Kept out of the engine deliberately — engine is stdlib |
| python-dotenv | API key loading | `.env` gitignored, template committed |
| **No LangChain, no vector DB, no agent framework** | — | "The task was 28 independent one-shot calls with no chaining or retrieval. A framework would add complexity with zero benefit. This is a deliberate minimalism decision, not an omission." |

**If asked "why not a real LLM/agent framework?"** — "The architecture I wanted is: deterministic system decides, LLM narrates. For that, prompt → call → verify is the entire surface. Frameworks shine when you need chaining, tools, or memory. I needed none. The moment the use case grows (e.g., multi-step investigation of a needs_review case), I'd reach for tool-calling, but not before."

**If asked "why Groq and not OpenAI/Anthropic?"** — "Cost (free tier, 28 calls) and latency. No quality tradeoff because the model is narrating facts my engine already computed — grounding matters more than model size here. The model is swappable via one string."

---

## Part 4 — Architecture deep-dive (the heart of the interview)

### 4.1 The three data sources — three lenses on the same money

| Source | Sees | Doesn't see |
|---|---|---|
| Order ledger (Dataset A, 501 rows incl. 1 dup) | Orders, amounts, customer, payment method, refund intent | Fees, batches, bank credits |
| Settlement report (Dataset B, 504 rows) | Fees, GST, batches, refund deductions, payment IDs | The ledger's refund *intent*, bank reality |
| Bank statement (Dataset C, 100 rows) | NEFT credits/debits with UTRs, noise transactions | Orders, fees, everything else |

Key numeric facts: 91 unique settlement_ids → 91 unique UTRs; 504 settlement rows cover 493 distinct orders (some orders have 2 rows: original + refund deduction); 99 unique UTRs in bank (includes ~10 noise); bank txn_date is typically T+1 after settlement_date.

### 4.2 The join path
```
A.order_id ──► B.order_id ──► B.bank_utr ──► C.utr
              (sum of B.net_amount per settlement_id ≈ C.amount, ±0.05)
```

### 4.3 The three matching layers

**Layer 1 — batch_matcher.py (settlement ↔ bank):**
For each settlement_id: `batch_net = sum(net_amount of all rows)` → look up batch's UTR among bank *credits* → classify:
- `batch_credited` — UTR found, |bank_amount − batch_net| ≤ 0.05 (handles the ±0.02 rounding drift in 12/91 batches)
- Ambiguous band: diff < 1.00 → credited but `needs_review`
- `batch_neft_failed` — positive net, UTR not in bank (money that should have arrived and didn't)
- `batch_no_credit` — non-positive net, UTR not in bank (**correct** behavior — refund deductions exceeded gross)
- Duplicate UTR handling: pick the bank credit closest to batch_net.

**Layer 2 — order_matcher.py (order ↔ settlement):**
payment_status pre-checks first (failed → expect 0 rows, so rows would be `UNEXPECTED_SETTLEMENT`; authorized → settles next cycle) → find all settlement rows by order_id → amount validation (ledger gross, converted to INR if USD, vs settlement gross, ±0.01) → refund classification.

**Layer 3 — refund_classifier.py:**
- `FULL_REFUND`: |refund_deduction| == original gross → residual must equal **−(fee + GST)** — the merchant's non-refundable MDR loss. *Not an error.*
- `REFUND_SPLIT`: refund row in a different settlement_id than the original charge (the hardest case — one order spans two batches)
- `PARTIAL_REFUND`: refund within same batch
- `REFUND_ONLY`: refund rows but no original charge
- `UNRECORDED_REFUND`: ledger claims a refund (refund_status partial/full, refund_amount > 0) but settlement has no refund row — **this code was born from a real bug** (Part 8).

### 4.4 Cross-source consistency (soft flags, never blocking)
`exceptions.py` checks: label mismatch after normalization; settlement_date more than 5 working days after order_date; captured_date ≠ order_date; bank date more than 2 working days after settlement. These produce `soft_flags` — informational, don't change classification. (Say why: "data-quality signals and hard financial exceptions are different things; mixing them erodes trust in the exception list.")

### 4.5 The confidence → status mapping (Phase 4)
```
matched                → Reconciled               (474 orders + 88 settlements = 562 plain)
matched_with_note      → Reconciled (with note)   (5: 3 REFUND_SPLIT + 2 CURRENCY_MISMATCH)
matched + NO_CREDIT_EXPECTED → Reconciled (no credit due) (1 settlement)
needs_review           → Needs Human Review       (14: 12 UNRECORDED_REFUND + 1 DUPLICATE + 1 GHOST)
hard_exception         → Unresolved               (9: 8 UNMATCHED_ORDER + 1 NEFT_FAILED)
```
562 + 5 + 1 + 14 + 9 = **591**. The reconciliation is the source of truth; the AI layer is a pure annotator. The mapping is exception-code-aware (a `matched` batch with NO_CREDIT_EXPECTED is still reconciled).

### 4.6 File/data flow (draw this on a whiteboard if asked)
```
data/raw/*.csv (frozen) ──► engine/matcher_exact.py ──► engine/output/match_log.json (591)
                                  deterministic                │
                                                               ▼
                                    agent/explainer.py ──► agent/output/explanations.json (28, verified)
                                       (LLM)                       │
                                                               ▼
                                    engine/reconciler.py ──► engine/output/reconciliation_report.json (591)
                                                               ▼
                                    engine/metrics_scorer.py ──► metrics_report.json/md  (vs ground truth)
                                                               ▼
                                    engine/generate_audit.py ──► audit_trail.md (live SHA-256 chain)

dashboard.py + agent/qa_agent.py read committed outputs only — never re-run the matcher.
```
Phases communicate **only through committed JSON files with hashes in their metadata** — no in-memory handoffs, no hidden state. Each phase verifies its input hashes after writing.

### 4.7 The exception codes (know all 9 cold)
| Code | Count | Meaning |
|---|---|---|
| UNMATCHED_ORDER | 8 | failed (5) / authorized (2) / captured-but-missing (1) |
| UNRECORDED_REFUND | 12 | ledger claims refund, settlement has none |
| REFUND_SPLIT | 3 | order spans 2 batches (original + refund) |
| CURRENCY_MISMATCH | 2 | USD order, matches after ×83 conversion |
| DUPLICATE_ORDER | 1 | two ledger rows, conflicting amounts |
| GHOST_TRANSACTION | 1 | settlement references order not in ledger |
| NEFT_FAILED | 1 | ₹62,386.14 batch, bank credit never arrived |
| NO_CREDIT_EXPECTED | 1 | negative net batch (−₹446.18), no credit is correct |
| AMOUNT_MISMATCH / UNEXPECTED_SETTLEMENT / REFUND_ONLY | 0 in data | implemented defensively, would fire on other data |

---

## Part 5 — The AI layer in depth (this is where AI-role interviews live)

### 5.1 The governing principle: "Narrate, don't decide"
The deterministic engine makes every classification. The LLM receives the *already-decided* status and the case's real data, and produces a 2–4 sentence plain-English explanation. It is instructed not to re-classify. Consequence: **an AI failure can never change a correct reconciliation result.** The worst an LLM failure can do is produce a flagged, unverified narration.

### 5.2 Grounding — three layers
1. **Domain facts block** (fixed, injected into every prompt): fee structure (2%/3% + 18% GST on fee), net formula, T+2 cycle, batching rules, refund mechanics, FX 83.00, identifier formats. Prevents the model from inventing domain rules.
2. **Case data blocks**: built by *deterministic Python code* per exception type — e.g. for REFUND_SPLIT, both settlement rows with amounts, the residual, the UTRs. The builder states relationships explicitly ("Row 2 has NO corresponding settlement row") instead of letting the LLM infer them — a lesson learned from a real relational-claim error (Part 8).
3. **System instructions**: base explanations ONLY on provided data; reference specific amounts/dates/IDs; do not re-classify; 2–4 sentences; express genuine uncertainty for needs_review cases; output only the final answer.

### 5.3 The hallucination safeguard (your crown jewel — know it cold)
Flow: `extract_figures()` → `collect_source_figures()` → `_figures_match()` → fail-closed rules.

- **Extraction**: regex for currency-marked amounts (Rs/$/INR/USD), 2025-XX-XX dates, *and bare decimal amounts* (e.g. "residual of 2196.99"). Unicode normalization first (narrow no-break spaces U+202F, thin spaces, ₹ symbol → these once caused false mismatches).
- **Verification**: numeric figures compared with `math.isclose(abs_tol=0.01)` (mirrors ORDER_TOLERANCE); non-numeric tokens (dates, IDs) by exact string. Numeric comparison killed the substring bug (Part 8).
- **Two sources**: a figure passes only if it matches the case data OR the curated domain facts (2, 3, 18, 83.00, 0.18, 2.36, 3.54...).
- **Fail-closed**: if the text contains decimal figures but none could be extracted → `verified: false`, reason `amounts_present_but_none_extracted`. A pass never means "skipped." If no figures at all → passes with reason `no_figures_stated` (nothing to check).
- **Results**: all 28 explanations verified. During development the checker caught Unicode issues and percentage equivalences; one relational-claim error slipped past figure checks and was caught by manual review — which is exactly why the fix was to make the data builders state relationships deterministically.

### 5.4 Validation & failure states (`validate_explanation`)
Empty/None/whitespace → `empty_response` (rejected); <2 sentences → `too_short`; >6 → `too_long`. Rejected explanations are stored with explicit failure states — never silently passed. If validation fails, the hallucination check is skipped and marked N/A. API failures produce `ERROR: <reason>` entries with `validation.reason = "api_error"`. The deterministic result is untouched in every failure path.

### 5.5 The Q&A agent (Phase 7) — read-retrieve-respond
- Loads 4 JSON artifacts once (lazy singleton), indexed by case_id. Writes nothing to disk.
- `classify_question()`: `ord_`/`set_` regex → single_case; future/external keywords → out_of_scope; else aggregate.
- single_case: full reconciliation_report entry + match_log enrichment + existing explanation (verbatim reuse — never regenerate) → prompt → LLM → `_verify_figures()` against the case JSON.
- aggregate: injects **pre-computed summary blocks only** — "The LLM is never asked to count raw data." Verified against the same summary stats injected into the prompt (catches the model rounding or approximating counts).
- out_of_scope: structured fallbacks with reason codes (`case_id_not_found`, `future_prediction`, `external_data`, `ambiguous`, `api_error`) — same JSON shape as real answers, `verified: true` because the fallback is rule-built, not generated.
- Retry/backoff: 429 → exponential (2/4/8s, 3 retries); 5xx/timeout → single retry; 401 → hard stop. Any unhandled exception → polite api_error fallback; the dashboard can never show a traceback.
- Limitation to volunteer proactively: the figure checker can't catch wrong *relational* claims in free-form Q&A answers; mitigation is rich deterministic case-data grounding.

### 5.6 Why "temperature 0" and "max_tokens 800/500"
Temperature 0: same facts, most reproducible output; narration isn't creative work. 800 tokens (explainer) / 500 (QA): room for 2–4 sentence answers plus any model preamble that gets stripped (the code strips `<think>` blocks and common preambles like "Here's a thinking process:" — gpt-oss is a reasoning-capable model and sometimes narrates its reasoning).

---

## Part 6 — Metrics: what the numbers actually mean (the honesty section)

This is where most candidates get destroyed: "your accuracy is 100%? Really?" Have these answers ready:

### The headline numbers
| Metric | Value |
|---|---|
| Cases scored | 591 (500 orders + 91 settlements) |
| Classification accuracy | 100% (591/591, 0 mismatches vs ground truth) |
| Clean reconciliation rate (operational) | 96.1% — 568/591 |
| Clean, no-exception rate | 95.3% — 563/591 |
| Needs Human Review | 14 (12 unrecorded refunds + 1 duplicate + 1 ghost) |
| Unresolved | 9 (8 unmatched orders + 1 NEFT failure) |
| AI-narrated | 28 cases, all verified |
| FPR | 0.0018 |
| FNR | 0.0 |
| Tests | 64, ~0.1s, no API calls |

### Pre-rehearsed answers to the three killer follow-ups

**"100%? That sounds fake."**
> "It's 100% against *synthetic* ground truth, and the README says so explicitly. The generator and matcher share the same domain assumptions, so the score proves internal consistency with that reference set plus a one-time manual audit correction — not performance on real unlabeled data. What I'd claim as more meaningful: zero false negatives — nothing was silently matched that should have been flagged — and the only false positive is the known ghost-transaction edge case, disclosed in the metrics report rather than hidden. And the 3.9% flagged cases are genuine ambiguities the system surfaced instead of guessing on."

**"You said 96.1% but also 95.3% — which is it?"**
> "Both, by two honest definitions. Operational rate counts cases resolved without human help — including 5 cases that carry ground-truth exception codes but were correctly handled with a note (3 refund splits, 2 currency conversions) and 1 settlement correctly determined to need no credit. The clean rate counts only cases that needed *no* exception at all. The 0.85pp gap is explained in the metrics report with the exact code breakdown — it's definitional, not an error."

**"FPR 0.0018 — but isn't accuracy 100%?"**
> "Yes, and this is a deliberate subtlety. The ghost-transaction batch is *correctly credited* at the batch level, but I flag it needs_review because one of its 6 orders isn't in the ledger. Ground truth has no GHOST vocabulary — it scores the batch as 'credited'. So in the binary matched-vs-exception view that's one false positive, while the exception-code comparison scores it correct. The report documents this: figure-level checks are necessary but not sufficient, and I chose to surface the ambiguity rather than suppress it." (Also: FNR = 0 — no silent misses, the metric that matters most in finance.)

### Ground truth integrity
- Ground truth generated **in the same pass** as the data (never inferred afterwards), then corrected **once** by manual audit, then frozen. Both the data and ground truth hashes are in the audit trail. `--regenerate-data` requires typing "yes".
- Scoring compares **exception_code equality**, with a vocabulary bridge for settlements (report exception_code → expected bank-credit status; GHOST → credited).
- Duplicate-order scoring rule: when one order has two GT entries (matched + exception), the exception entry wins (a ledger artifact, not two classification targets).

---

## Part 7 — Data integrity, hashing, and the audit trail

- **Chain of custody**: `audit_trail.md` records SHA-256 of all 9 pipeline files, computed **live** by `generate_audit.py` — never hand-copied. Each phase's metadata embeds its inputs' hashes, so any consumer can verify provenance. Post-write verification confirms inputs unchanged.
- **Cross-platform hash stability**: a real bug — the trail verified on Linux but not Windows because of CRLF. Fixed two ways: `.gitattributes` forcing LF, and `sha256()` normalizing `\r\n → \n` as defense-in-depth. Also: `match_log.json` is byte-deterministic (sorted keys/IDs, `newline=""` writes); reports with `generated_at` timestamps are *expected* to change hash per run — documented, not hidden.
- **Reproducibility pin**: `TestMatcherOutputReproducibility` re-runs the matcher in-memory, serializes exactly as `compile_match_log` does, and asserts its hash equals the committed `match_log.json`. Any engine change that alters results fails CI instead of silently drifting from the audit trail.
- **What the audit proves**: the reported numbers were computed from exactly the committed inputs, by exactly the committed code.
- **Deliberately frozen data**: regeneration is possible but guarded, because every hash, score, and audit entry anchors to the committed dataset.

---

## Part 8 — The bugs: your interview goldmine ("tell me about a hard bug" — have all four ready)

### Bug 1 — Duplicate-order settlement suppression (the matcher hid evidence)
- **Symptom**: `ord_EnDJiS9HvlxNgbb1` had two ledger rows (₹1130.56 vs ₹1202.36, same customer/SKU/timestamp). The DUPLICATE_ORDER classification short-circuited before order matching, so `settlement_ids: []` — even though a real settlement row for ₹1130.56 existed.
- **Catch**: manual review of match_log noticed empty settlement_ids and cross-checked the CSV.
- **Fix**: duplicate path now looks up real settlement data; matcher attaches `settlement_ids: ['set_NvO7qBhqH6y5IHWi']` while keeping needs_review; explainer's case-data builder updated to state the relationship ("settlement supports 1130.56; 1202.36 unverified").
- **Lesson**: *"Don't let an exception path prevent evidence collection. Flag the decision, but still gather the facts."*

### Bug 2 — 12 unrecorded refunds (the most important one; this is the audit trail's headline)
- **Symptom**: 12 orders with `refund_status=partial` and refund_amount > 0 in the ledger, but *no* refund_deduction row in settlement. The matcher classified all 12 as clean `matched` — the match rate looked perfect while refunds were silently missing from the books.
- **Catch**: no single source showed it; cross-referencing ledger refund intent against settlement refund rows exposed the gap immediately.
- **Fix**: new exception code `UNRECORDED_REFUND`; reclassified 12 orders matched → needs_review. These 12 are why Needs Human Review = 14 and why narrated cases = 28 (not the 17 designed).
- **Verification**: didn't trust its own fix — independently re-derived all 12 order IDs from raw CSVs with an objective rule; all 12 matched; now regression-locked.
- **Lesson**: *"The dangerous failures are the ones that don't crash — they inflate your success metrics. Cross-source intent-vs-evidence checks are how you find money that's silently wrong."*

### Bug 3 — Negative-net batch mislabeling (wrong default assumption)
- **Symptom**: batch `set_vlVzIbTfj7VNQanv` (net −₹446.18) got `hard_exception` — same label as a genuine failure. But refund deductions exceeding gross means *no bank credit is due* — correct behavior.
- **Fix**: `batch_no_credit` with confidence `matched`, exception_code `NO_CREDIT_EXPECTED` retained for traceability; Phase 4 maps it to "Reconciled (no credit due)".
- **Lesson**: *"Distinguish 'something went wrong' from 'the system behaved correctly in a weird-looking case'. Mislabeling correct behavior destroys trust in the exception list."*

### Claim 4 — Refuted (say this unprompted; it shows intellectual honesty)
- A suspected bug: `expected_residual` didn't match `order_residual` for REFUND_SPLIT. Investigation: they measure different things by design (gross-based retained value vs net-of-fees sum) and the code never compares them. **No fix needed** — documented in the audit trail as refuted.

### Bug 5 — Hallucination checker substring match (the AI-layer bug)
- **Symptom**: `_verify_figures` used substring matching, so a stated "2" verified against 47,500.00 ("2" is inside it); "18" vs 1180.00; "83" vs 8383.00.
- **Fix**: numeric figures compared numerically with `math.isclose(abs_tol=0.01)`; substring only for non-numeric tokens. Regression tests lock in exactly those three negative cases plus the positive `83 == 83.00`. Applied in both explainer and QA agent.
- **Lesson**: *"A verifier with a loose matcher is worse than no verifier — it manufactures false confidence in the safety mechanism itself."*

### Bug 6 — Hash instability (infrastructure lesson)
- CRLF line endings made hashes platform-dependent → `.gitattributes` + CRLF-normalizing sha256 (defense in depth).

### The relational-claim miss (AI limitation, caught by humans)
- The LLM once said both duplicate rows "were included in a single batch" — both *figures* were real, the *relationship* was wrong, and the figure checker passed it. Manual review caught it. Fix: deterministic data builders now state relationships explicitly. **Willingly volunteering this is a senior-level signal**: "fact-checking figures is necessary but not sufficient; relational claims need deterministic grounding."

---

## Part 9 — Testing philosophy

- **64 tests, 14 classes, stdlib unittest only, ~0.1s, zero API calls.** No pytest — no dependency, no reason.
- Unit tests: batch success/mismatch paths, FX conversion, refund classification, order exceptions, status mapping, ghost detection, metrics arithmetic, QA parsing, explanation validation, fail-closed verification.
- Integration: full frozen dataset (500 + 91) — counts, field completeness, determinism (two runs identical), report structure; plus the hash reproducibility pin.
- LLM calls are *deliberately excluded* from tests: fast, deterministic, offline, free. LLM-dependent behavior is guarded by runtime verification instead (fail-closed checks, validation), and deterministic helpers are tested directly.
- Regression-locking discipline: every real bug fixed above is now pinned by a named test.

---

## Part 10 — Production thinking (DATA_ARCHITECTURE_REPORT.md — your "what next" answer)

You wrote a production-architecture analysis in-repo. Use it when asked "how would this scale?"

- Core insight: **"The logic is production-quality; the storage layer is the gap."** The data is already relational (order_id, settlement_id, bank_utr are foreign keys enforced in code, not by structure).
- Six file-based risks you identified: duplicate ingestion (no UNIQUE constraint), non-atomic writes (crash mid-json.dump loses output), no run history (overwrites destroy previous correct outputs), timestamp hash instability (metadata breaks content addressing), concurrent write hazard (no locking), and partial AI-output loss on re-run.
- Proposed schema: PostgreSQL with immutable raw tables, separate `refunds` table, append-only `match_results`/`explanations` keyed by `run_id`, versioned ground truth, `reconciliation_runs` table holding hash provenance — plus indexes on order_id/settlement_id/utr and UNIQUE(run_id, case_id) idempotency.
- Design principle preserved in production design: **keep the deterministic core, keep AI narration-only, keep per-run provenance.**

---

## Part 11 — Decisions & trade-offs (the "why" questions)

1. **Batch-first matching** — forced by reality: bank only sees batch credits; order-first matching against the bank is structurally impossible.
2. **LLM narrates, never decides** — auditability. In finance, an unexplainable classification is worse than no classification. AI failure = annotated flag, never a wrong result.
3. **Synthetic data with baked-in ground truth** — no real data access (buildathon); lets me plant every edge case with a traceable origin and score honestly. Explicitly scoped: proves engine correctness, not real-world accuracy.
4. **Frozen, committed data + outputs** — reproducibility and a fresh-clone-runnable demo; hashes anchor everything. Trade-off: repo carries data; acceptable for a buildathon artifact.
5. **Regex/alias label normalization, not fuzzy matching** — label differences were *systemic* (all rows), so a normalization map is correct; fuzzy matching would mask genuine mismatches.
6. **Tolerances**: ±0.05 batch (absorbs ±0.02 per-row rounding drift), ±0.01 order-level, strict equality for full-refund detection. In ambiguous band (0.05–1.00) → needs_review rather than guess.
7. **Groq + gpt-oss-120b** — free, fast, sufficient for grounded narration; model is a swappable string.
8. **No LangChain/vector DB** — no chaining/retrieval/tool need; minimalism as a feature.
9. **Every mismatch listed individually, never aggregated** — anonymous counts hide exactly the cases you need to see.
10. **Fail-closed everywhere** — unextractable figures = unverified; missing data = honest exception; regeneration = guarded.
11. **Soft flags vs hard exceptions** — data-quality signals don't pollute the financial exception list (trust).
12. **Match rate defined operationally but both definitions reported with the gap explained** — no metric shopping.

### Honest limitations (volunteer 2–3 before being asked — it builds enormous credibility)
1. Synthetic ground truth → 100% proves internal consistency, not real-world accuracy. Next step: run the matcher on anonymized real Razorpay exports and hand-score a sample.
2. Figure verification can't catch wrong relational claims (mitigated by deterministic case-data builders; a full fix needs semantic checking or structured output).
3. Scales fine to ~10⁵ cases in this architecture; beyond that needs the DB layer + incremental/streaming matching.
4. FX is fixed at 83.00; real systems need dated rate tables and markup handling.
5. Duplicate UTR resolution picks the closest amount — correct for this dataset, needs a stronger rule (date proximity) in production.
6. Q&A aggregates come from precomputed stats — excellent for grounding, but novel cross-field questions fall back to out_of_scope rather than compute live.

---

## Part 12 — Possible extensions (have an opinion on each)

- **Live ingestion**: Razorpay API + bank statement parsers (CSV/MT940) feeding the same matcher; the deterministic core doesn't change.
- **Postgres layer** per Part 10; run-scoped append-only results.
- **Human-review workflow**: assign Needs Human Review cases, capture the human's decision as *new ground truth* → the system gets a growing labeled set → measure real precision over time.
- **Semantic verification**: second LLM-as-judge pass or structured JSON outputs with schema validation for relational claims.
- **Alerting**: NEFT_FAILED → Slack/email to bank ops with the UTR; SLA timers on needs_review cases.
- **Dated FX tables + per-instrument fee schedules** from config, not code.
- **Anomaly detection on top**: statistical outlier fees/delays as new soft flags.
- **Multi-acquirer**: generalize beyond Razorpay — the three-lens model (merchant/processor/bank) is generic.

---

## Part 13 — How to run it (in case they ask you to demo)

```bash
pip install -r requirements.txt
cp .env.example .env        # add GROQ_API_KEY (free from groq.com)
streamlit run dashboard.py  # the demo: metrics → 591-case explorer → live Q&A → audit story

python run_pipeline.py --all          # Phases 2–6
python -m unittest tests.test_reconciliation -v   # 64 tests, ~0.1s
python3 agent/qa_agent.py             # 6 built-in demo questions
```
Demo flow that works: headline metrics (100% / 96.1%) → filter case table to `UNRECORDED_REFUND` → open one case, read the AI explanation → ask the Q&A "Why does order ord_EnDJiS9HvlxNgbb1 need human review?" → show audit trail hash table → done in 5 minutes.

---

## Part 14 — Resume bullet alignment (make sure you can defend each line)

Whatever your resume says, ensure you can back these claims with specifics:
- "Agent-assisted reconciliation across 3 data sources" → Part 4.
- "LLM narration with automated fact-checking / fails closed" → Part 5.
- "100% classification accuracy on 591 cases" → Part 6 (including the *scope* caveat — say it before they ask).
- "Caught N real defects through external audit" → Part 8.
- "64 regression tests" → Part 9.
- If your resume mentions phases or specific numbers (28 narrated cases, 12 refunds, ₹62,386 NEFT failure), know each cold — they're all in this doc.

---

## Part 15 — The 10 lines to say when you want to sound senior

1. "The bank never sees orders — it sees batch credits. So matching had to be batch-first."
2. "The deterministic engine is the source of truth; the LLM is a narrator with a reference card."
3. "Ground truth was generated with the data, not inferred after — and corrected once by audit, then frozen."
4. "The most dangerous failures are the ones that inflate your success metrics — 12 refunds vanished into 'matched' until I cross-referenced intent against evidence."
5. "Verification fails closed: a pass never means 'skipped'."
6. "Every mismatch is listed individually with case_id — anonymous counts hide exactly what you need to see."
7. "I'd rather report a 0.85pp gap with its explanation than one flattering number."
8. "One suspected bug turned out to be correct behavior — the audit trail records the refutation too."
9. "Figure-level fact-checking is necessary but not sufficient; relational claims need deterministic grounding."
10. "The logic is production-quality; the storage layer is the gap — and I wrote the schema for it."

---

*Next: INTERVIEW_QA_BANK.md (~70 questions with model answers) and INTERVIEW_CHEATSHEET.md (one-page revision).*
