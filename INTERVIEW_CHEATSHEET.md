# INTERVIEW_CHEATSHEET.md — Rapid Revision (read this the morning of)

## The pitch (30s)
> "Agent-assisted reconciliation for a Razorpay buildathon. Merchants get paid in batches, not per order — fees + 18% GST, refunds in future batches, USD conversion make three data sources (order ledger, settlement report, bank statement) never agree line-by-line. My engine matches them deterministically with batch-first matching; an AI layer narrates the exceptions, never decides, and every AI figure is auto-fact-checked (fails closed). 100% classification accuracy on 591 cases, 96.1% clean rate, 64 tests, zero silent failures."

## THE NUMBERS (know cold)
| | |
|---|---|
| Cases | **591** = 500 orders + 91 settlements |
| Accuracy | **100%** (591/591 vs ground truth, 0 mismatches) |
| Match rate | **96.1%** operational (568/591) · 95.3% clean (563/591) · gap 0.85pp = the 5 with-note cases (3 REFUND_SPLIT + 2 CURRENCY_MISMATCH) |
| Flagged | **14** Needs Human Review (12 UNRECORDED_REFUND + 1 DUPLICATE + 1 GHOST) |
| Unresolved | **9** (8 UNMATCHED_ORDER + 1 NEFT_FAILED) |
| AI narrated | **28** cases, all verified |
| FPR / FNR | **0.0018** (the ghost batch — known, documented) / **0.0** |
| Tests | **64** tests, 14 classes, ~0.1s, no API calls |
| Data | 501 ledger rows (1 dup) · 504 settlement rows · 100 bank rows (90 credits + noise) |
| Money rules | fee 2% domestic / 3% intl · GST **18% on fee only** · FX 83.00 · T+2 · UTR = bank ref |
| Tolerances | batch ±0.05 · order ±0.01 · full-refund detection: strict equality |
| NEFT failure case | set_7oqQnmBR7evr0ci5 · **₹62,386.14** · UTR 9503100649340391 · 5 orders |
| Duplicate order | ord_EnDJiS9HvlxNgbb1 · ₹1130.56 vs ₹1202.36 · settlement supports 1130.56 |

## Exception codes → counts
UNMATCHED_ORDER 8 · UNRECORDED_REFUND 12 · REFUND_SPLIT 3 · CURRENCY_MISMATCH 2 · DUPLICATE_ORDER 1 · GHOST_TRANSACTION 1 · NEFT_FAILED 1 · NO_CREDIT_EXPECTED 1 · (AMOUNT_MISMATCH / UNEXPECTED_SETTLEMENT / REFUND_ONLY implemented, 0 in data)

## Stack
Python 3.8+ **stdlib-first** (csv/json/hashlib/unittest/re/math) · Groq `openai/gpt-oss-120b`, **temp 0** (free tier; openai SDK + base_url) · Streamlit (dashboard only) · pandas (chart only) · python-dotenv · **no LangChain, no vector DB, no agent framework** (deliberate)

## Architecture in one line each
1. **Batch-first**: bank sees only batch credits (one UTR = one settlement = N orders) → Layer 1 batch↔bank, Layer 2 order↔settlement, Layer 3 refund classification
2. **Preprocessing**: label alias map (visa_mc_domestic→card) + USD×83 — normalization, not exceptions
3. **Confidence**: matched / matched_with_note / needs_review / hard_exception → Phase 4 → Reconciled / (with note) / (no credit due) / Needs Human Review / Unresolved
4. **Full refund residual** = −(fee+GST) — MDR non-refundable; it's correct behavior, not variance
5. **AI layer**: fixed domain-facts block + per-case real data → 2–4 sentences → figure extraction → numeric verify (`math.isclose`, abs_tol 0.01) vs case data + domain facts → **fails closed**
6. **Q&A agent**: classify (single_case / aggregate / out_of_scope) → retrieve by case_id (LLM never counts raw data) → answer → verify → structured fallbacks (never crashes)
7. **Pipeline**: frozen CSVs → match_log.json (591) → explanations.json (28) → reconciliation_report.json (591) → metrics_report → audit_trail.md; phases talk only via hashed, committed JSON; dashboard reads, never re-runs
8. **Chain of custody**: live SHA-256 of all 9 files; `.gitattributes` LF + CRLF-normalizing hash; reproducibility pin test (fresh run == committed match_log hash)

## The 4 bugs (STAR-ready)
1. **Duplicate suppression** — exception path skipped evidence collection → settlement_ids was [] though a real settlement existed → fix: gather evidence even while flagged. *Lesson: flag the decision, still collect the facts.*
2. **12 unrecorded refunds** ⭐ — ledger said refund, settlement had none; matcher said "matched" → match rate was perfect while money was missing. Caught by cross-referencing intent vs evidence; new code UNRECORDED_REFUND; fix independently re-derived from raw CSVs; regression-locked. *Lesson: the worst failures inflate your success metrics.*
3. **Negative-net mislabeling** — batch with net −₹446.18 was "hard_exception" but no credit due is *correct* → NO_CREDIT_EXPECTED / "Reconciled (no credit due)". *Lesson: don't label correct-but-weird as failure.*
4. **Hallucination-checker substring bug** — "2" verified against 47,500.00 → numeric comparison (isclose 0.01), negative-case regression tests. *Lesson: a loose verifier manufactures false confidence.*
- **+1 refuted claim** (expected_residual vs order_residual — different by design, never compared): say it unprompted, shows honesty.
- **+1 AI limitation**: relational-claim error passed figure checks; caught by human review; fix = deterministic case-data builders state relationships. *Lesson: figure checks are necessary, not sufficient.*

## The three killer follow-ups (memorized answers)
- **"100% sounds fake"** → "100% against *synthetic* ground truth — scoped explicitly in the README. Proves internal consistency + one manual audit, not real-world accuracy. The stronger claims: zero false negatives, and the 3.9% flagged were surfaced, not guessed."
- **"96.1% or 95.3%?"** → "Two honest definitions: resolved-without-humans vs no-exception-needed. The 0.85pp gap is itemized in the report (3 refund splits + 2 currency + 1 no-credit). Definitional, not an error."
- **"FPR 0.0018 vs 100%?"** → "Ghost batch: correctly credited at batch level, but flagged needs_review for the unknown order inside. GT has no GHOST vocabulary → code-equality says correct, binary view counts 1 FP. Documented; FNR = 0.0 is what matters."

## 10 senior lines (drop naturally)
1. "The bank never sees orders — it sees batch credits, so matching had to be batch-first."
2. "The deterministic engine decides; the LLM is a narrator with a reference card."
3. "Ground truth was generated *with* the data, audited once, then frozen."
4. "The most dangerous failures inflate your success metrics — 12 refunds vanished into 'matched'."
5. "Verification fails closed: a pass never means 'skipped'."
6. "Every mismatch is listed individually — anonymous counts hide exactly what you need to see."
7. "I'd rather report a 0.85pp gap with its explanation than one flattering number."
8. "One suspected bug turned out to be correct behavior — the audit trail records the refutation too."
9. "Figure-level checking is necessary but not sufficient; relational claims need deterministic grounding."
10. "The logic is production-quality; the storage layer is the gap — and I wrote the Postgres schema for it."

## Demo commands
```bash
streamlit run dashboard.py                        # demo
python run_pipeline.py --all                      # phases 2–6
python -m unittest tests.test_reconciliation -v   # 64 tests
python3 agent/qa_agent.py                         # 6 sample Q&As
```
Demo path: metrics → filter UNRECORDED_REFUND → read one AI explanation → Q&A "Why does order ord_EnDJiS9HvlxNgbb1 need human review?" → audit trail hashes. 5 min.

## If you blank out, remember just this
**Three sources, three lenses, one money trail. Batch-first because the bank is batch-level. Deterministic decides, AI narrates, verification fails closed. The bug story is the product: the system caught itself inflating its own metrics, and now it can't.**
