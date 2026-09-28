# 1. Business Overview

## 1.1 Problem
Hospitals send bills (claims) to insurance companies (payers). Many are rejected (denied). Staff have to read each rejection by hand, work out the reason, and write an appeal letter. This is slow, deadlines get missed, and money is lost.

## 1.2 Goal
- Recover more money from denied claims
- Cut the manual work per denial
- Miss zero appeal deadlines
- Find the root causes so fewer denials happen in future

## 1.3 Example
1. The hospital bills $4,000 for an MRI.
2. The payer denies it: **CARC 50, not medically necessary**.
3. The system reads the denial, predicts a 70% chance of winning, finds "back pain for 8 weeks" in the doctor's notes, and checks the payer's rule ("MRI allowed after 6 weeks").
4. It writes the appeal letter. A reviewer approves it and it is sent.
5. The payer pays, and the result is recorded so the model can learn from it.

## 1.4 The 5 steps
```
READ → UNDERSTAND → DECIDE → PREPARE → SEND & LEARN
```

| Step | What happens | Technology |
|---|---|---|
| Read | Pull fields out of the denial letter or remittance (835/EOB) | OCR + GenAI |
| Understand | Map the reason code (CARC/RARC) to a category | Rules + GenAI |
| Decide | Appeal, fix and resend, bill the patient, or write off | Rules + ML |
| Prepare | Collect notes, search payer policy, write the letter | Agent + RAG + GenAI |
| Send & Learn | Human approval, submission, tracking, retraining | Workflow + ML |

## 1.5 Root-cause categories
| Category | Example CARC | Action |
|---|---|---|
| Eligibility / coverage | 27, 26, 31 | Re-check insurance, resubmit |
| Authorization missing | 197, 15, 62 | Find the authorization or appeal |
| Medical necessity | 50, 55, 56 | Appeal with clinical proof |
| Coding error | 4, 11, 16, 181 | Send a corrected claim |
| Duplicate | 18 | Verify, usually close |
| Timely filing | 29 | Appeal only if there is proof |
| Bundling | 97, 234, B15 | Correct or appeal |
| Coordination of benefits | 22, 23 | Bill the right payer |

## 1.6 Decision rules
**Hard rules (applied first):**
1. Group code `PR` → bill the patient, stop.
2. Appeal deadline passed → write off, unless there is proof of timely filing.
3. Balance below $50 (configurable) → write off or batch.
4. All appeal levels used → close or send to external review.

**ML scoring:**
- `P(overturn)` = chance of winning
- `Expected Recovery = P(overturn) × Denied Amount`
- `Priority = Expected Recovery ÷ Effort`, boosted when the deadline is close

| Condition | Action |
|---|---|
| P ≥ 0.6 and amount ≥ $500 | Agent drafts the appeal |
| 0.3 ≤ P < 0.6 | A human decides |
| P < 0.3 and low amount | Write off |
| Fixable error | Corrected claim |
| ≤ 14 days to deadline | Top of the queue |

## 1.7 Guardrails
- Every fact in a letter must cite a source document.
- Nothing goes to a payer without human approval.
- Only the minimum patient data needed is used (HIPAA).
- Every action is written to an audit log.

## 1.8 Glossary
| Term | Simple meaning |
|---|---|
| Claim | The bill sent to insurance |
| Payer | The insurance company |
| Provider | The hospital or doctor |
| Denial | Insurance refuses to pay |
| Appeal | A letter asking them to reconsider |
| Overturn / Upheld | We won / we lost |
| Write-off | Give up on the money |
| Corrected claim | Fix the bill and resend it |
| 835 / EOB | The payer's reply: what was paid and why |
| 837 | Electronic bill format |
| CARC / RARC | Denial reason code / extra detail |
| Group code | Who pays: CO = provider, PR = patient |
| CPT / ICD-10 | Treatment code / illness code |
| Prior authorization | Permission from the payer before treatment |
| LCD / NCD | Medicare coverage rulebooks |
| A/R | Money still owed to the hospital |
| PHI / HIPAA | Private patient data / the law that protects it |
| GenAI / LLM | AI that reads and writes text |
| ML model | Predicts outcomes from past data |
| Agent | AI that carries out multi-step tasks using tools |
| RAG | AI searches documents before answering |
| Human-in-the-loop | A person reviews the AI's output |
