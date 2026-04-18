# Task 1 – Naive Generation Insights

## Dataset
- 100 rows after filtering (dropped metrics-generated)
- 50 domain-relevant, 50 novel-generated
- 42 unique documents

## What the question types mean

**domain-relevant** — questions that test financial *reasoning and concepts*. Things like "is working capital a useful metric for this company?" or "is this business capital intensive?". The model can often handle these from general financial knowledge alone, without needing the actual filing.

**novel-generated** — questions that require *exact figures from a specific filing*. Things like "what was the cash drop between FY2023 and Q2 FY2024?". The model has no reliable way to answer these without the document — so it either hallucinates a number or refuses. This is exactly what RAG is designed to fix.

---

## The 10 Questions – Current Run Results

### financebench_id_00005 · domain-relevant · **partially correct**

**Q:** Does Corning have positive working capital based on FY2022 data?  
**Model:** Current assets $7.85B, current liabilities $4.93B → positive WC of $2.92B.  
**Ground truth:** Positive WC of $831M.  
**Verdict:** Partially correct. Got the direction right (positive) but overstated by ~3.5×. The discrepancy is structural: the ground truth uses only *operating* current assets/liabilities explicitly shown in the balance sheet, while the model summed all current items. The model doesn't know this narrower definition without the filing.

---

### financebench_id_00070 · domain-relevant · **partially correct**

**Q:** Does American Water Works have positive working capital based on FY2022 data?  
**Model:** Current assets ~$2.43B, liabilities ~$2.55B → negative WC ~-$120M.  
**Ground truth:** Negative WC of -$1,561M.  
**Verdict:** Partially correct. Got the sign right (negative) but is off by ~13× on the magnitude. Also correctly added nuance that WC may not be the most useful metric for a utility — good financial reasoning.

---

### financebench_id_00080 · domain-relevant · **wrong**

**Q:** Does PayPal have positive working capital based on FY2022 data?  
**Model:** Current assets $63.8B, liabilities $76.4B → negative WC of -$12.6B.  
**Ground truth:** Positive WC of $1.6B.  
**Verdict:** Wrong — and in the wrong direction. The model inflated both figures to ~40× their actual size and flipped the sign. Likely hallucinated the payment float as a massive current liability. Interesting that it actually argued convincingly *why* negative WC is fine for PayPal (payment processing float reasoning), making the wrong answer sound authoritative.

---

### financebench_id_00206 · domain-relevant · **correct**

**Q:** Are JPM's gross margins historically consistent?  
**Model:** Gross margins are not a relevant metric for a bank; discussed NIM instead.  
**Ground truth:** Gross margin is not a relevant metric for a financial institution.  
**Verdict:** Correct. The model nailed the core answer and went further with a reasonable explanation of NIM as the appropriate alternative metric. No retrieval needed — this is pure financial domain knowledge.

---

### financebench_id_00215 · domain-relevant · **partially correct**

**Q:** Is Verizon a capital intensive business based on FY 2022 data?  
**Model:** CapEx ~$23.4B / Revenue ~$136.4B = 17.2% CapEx-to-revenue → yes, capital intensive.  
**Ground truth:** Capital intensity ratio ≈ 2.77 (Assets / Revenue) → yes, capital intensive.  
**Verdict:** Partially correct. Reached the right conclusion but used the wrong metric (CapEx/Revenue instead of Assets/Revenue). The model's figures themselves are also hallucinated — actual FY2022 CapEx was ~$18.2B. Good instinct, wrong calculation.

---

### financebench_id_00283 · novel-generated · **wrong (hallucinated)**

**Q:** How much does Pfizer expect to pay to spin off Upjohn in the future in USD million?  
**Model:** ~$12 billion USD.  
**Ground truth:** $77.78 million.  
**Verdict:** Hallucinated — off by ~150×. The model likely confused the spin-off *transaction cost* ($77.78M) with the approximate *market value* of the Viatris deal (~$12B). Confidently stated without any hedging, which is the dangerous pattern for RAG motivation.

---

### financebench_id_00288 · novel-generated · **refused**

**Q:** Was there any drop in Cash & Cash equivalents between FY 2023 and Q2 of FY2024?  
**Model:** "I don't have access to real-time financial data." Offered guidance on how to look it up.  
**Ground truth:** Yes, ~42% decline.  
**Verdict:** Refused. This is the only clean refusal. The model correctly flagged that FY2024 data is beyond its training horizon — a legitimate and honest refusal. Notably the refusal was still somewhat useful (explained what comparison to make).

---

### financebench_id_00299 · novel-generated · **partially correct**

**Q:** Which of JPM's business segments had the lowest net revenue in 2021 Q1?  
**Model:** Corporate segment, $254M net revenue. Listed all 5 segments with figures.  
**Ground truth:** Corporate, net revenue -$473M.  
**Verdict:** Partially correct. Identified the right segment but reported a positive $254M instead of a negative -$473M — missed the sign entirely. The hallucinated figures for other segments ($12.8B CIB, etc.) are in the right ballpark but still fabricated.

---

### financebench_id_00302 · novel-generated · **partially correct**

**Q:** Did Pfizer grow its PPNE between FY20 and FY21?  
**Model:** "PPNE did grow between FY20 and FY21" then gave figures: $10.4B → $13.8B (+32.7%). Hedged with "I don't have the exact figures."  
**Ground truth:** Yes, change in PPNE was positive year over year.  
**Verdict:** Partially correct — the *direction* is right (growth) and the model committed to it despite uncertainty. The specific figures ($10.4B, $13.8B) are hallucinated. **Note:** The heuristic auto-verdict classified this as "refused" because it matched "I don't have" — that is a false positive; the model did answer substantively.

---

### financebench_id_00382 · novel-generated · **correct**

**Q:** Which region had the highest EBITDAR contribution for MGM during FY2022?  
**Model:** Las Vegas Strip region.  
**Ground truth:** Las Vegas resorts, ~90% of company-level EBITDAR.  
**Verdict:** Correct. Straightforward factual recall — MGM's Las Vegas dominance is well-known enough that the model knew it without retrieval.

---

## Summary

| Verdict | Count | IDs |
|---|---|---|
| correct | 2 | 00206, 00382 |
| partially correct | 5 | 00005, 00070, 00215, 00299, 00302 |
| wrong | 2 | 00080, 00283 |
| refused | 1 | 00288 |

**By question type:**
- domain-relevant (5): 1 correct, 3 partially correct, 1 wrong — model can reason about financial concepts but fabricates specific figures
- novel-generated (5): 1 correct, 2 partially correct, 1 wrong, 1 refused — clearly weaker without retrieval

---

## Interesting Observations

### 1. Heuristic verdict bug on 00302
The auto-verdict classified 00302 as "refused" because the text contained "I don't have the exact figures." In reality the model gave a directional answer (growth, +32.7%) — the heuristic is too aggressive. Manual verdict should be **partially correct**.

### 2. The working capital definition trap (00005, 00070, 00080)
Three questions asked about working capital. The model computed WC as total current assets minus total current liabilities (standard formula), but the ground truth uses a narrower *operating* working capital definition based only on items explicitly shown in the balance sheet. Even if the model had the right formula, it would get wrong numbers without the actual filing. RAG should help here significantly.

### 3. Confident hallucination vs honest refusal (00283 vs 00288)
The most dangerous outputs are confident wrong answers, not refusals. 00283 (Pfizer Upjohn $12B vs $77.78M) is 150× off, stated with zero hedging. 00288 at least told the user it didn't know. From a user-trust perspective, 00283 is worse than 00288.

### 4. Reasoning questions are more robust than retrieval questions
The two clean correct answers (00206 JPM gross margins, 00382 MGM EBITDAR region) required either pure financial reasoning or widely-known facts. The model fails when it needs precise numbers from a specific annual filing.

### 5. Sign errors on financial figures (00299)
00299 got the right segment (Corporate) but reported +$254M instead of -$473M — the model dropped the negative sign, likely because it's unusual to report negative net revenue. Without the filing, it defaults to plausible-looking positive numbers.

### 6. Confabulated rationale for wrong answers (00080)
PayPal (00080) is the clearest example of the model constructing a *convincing narrative* around wrong facts. It explained why PayPal's negative WC is acceptable due to payment float — reasonable logic — but applied to fabricated numbers. This is the LLM "hallucination with reasoning" failure mode.

---

## Hypothesis for Task 5 (RAG)

RAG is expected to help most on:
- **novel-generated questions** — all 5 will benefit from grounded retrieval (00283, 00288, 00299, 00302, 00382)
- **working capital questions** (00005, 00070, 00080) — the filing provides exact balance sheet line items to resolve the operating WC ambiguity

RAG will have less impact on:
- Pure reasoning questions like 00206 (JPM gross margins) — the model already knows gross margin isn't relevant for banks
