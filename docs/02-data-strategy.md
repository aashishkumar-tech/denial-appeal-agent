# 2. Data Strategy

> Check every dataset name, link, size and license before use. The list below is from general knowledge and has not been verified.

## 2.1 How much data each part needs
| Component | Data needed | Why |
|---|---|---|
| GenAI extraction and letters | 50–200 test examples | The model is pre-trained; we only evaluate it |
| RAG (payer policies) | Thousands of policy documents | Public CMS documents |
| Agent workflow | 20–100 test scenarios | Tests the workflow, not a model |
| ML overturn model | 50K–1M labeled claims | The only component that needs volume |

## 2.2 Public sources
| Source | Content | Used for |
|---|---|---|
| CMS DE-SynPUF | Synthetic Medicare claims (inpatient, outpatient, carrier, drugs) | Base claims, ML features |
| Synthea | Generator for patients, encounters, claims, notes | Scalable claims + linked patients |
| Kaggle: Healthcare Provider Fraud Detection | Medicare-style claims | Extra claims and features |
| CMS Transparency in Coverage PUF | Real denial rates by insurer | Setting realistic denial rates |
| CMS CERT reports | Real improper-payment reasons | Setting realistic reason mix |
| CMS Medicare Coverage Database (LCD/NCD) | Coverage policies | RAG knowledge base |
| X12 / WPC CARC & RARC lists | Denial reason codes | Code mapping table |
| CMS ICD-10-CM, HCPCS files | Code descriptions | Validation, features |
| Hugging Face: `starmpcc/Asclepius-Synthetic-Clinical-Notes` | Synthetic clinical notes | Appeal evidence |
| Hugging Face: `aharley/rvl_cdip`, `nielsr/funsd` | Scanned documents, forms | Document classification and extraction tests |

## 2.3 Generating denial labels (automated)
Public claims don't include denials or appeal outcomes, so a script adds them:

1. **Load claims** from SynPUF or Synthea.
2. **Set the denial rate** per payer type from the published statistics (configurable, e.g. 10–20%).
3. **Assign reasons with rules**, for example:
   - High-cost imaging with no authorization flag → CARC 197
   - Diagnosis not on the policy's covered list for the CPT → CARC 50
   - Submission date later than the filing limit → CARC 29
   - Same patient, date and CPT twice → CARC 18
   - Everything else → random from the reason mix
4. **Assign the group code** (mostly CO, some PR).
5. **Simulate appeal outcomes** with a probability per category, adjusted by features (documentation present, amount, payer).
6. **Add noise** so the model can't just memorize the rules.
7. **Link notes**: attach a synthetic note to each medical-necessity case, with evidence present or missing.

All rates and probabilities go in `config/denial_rules.yaml` so the business team can change them without code changes.

## 2.4 Output tables
| Table | Key fields |
|---|---|
| `claims` | claim_id, patient_id, payer, dos, cpt, icd10, billed_amt, submit_date |
| `denials` | claim_id, group_code, carc, rarc, denial_date, denied_amt, category |
| `appeals` | appeal_id, claim_id, level, submit_date, outcome, recovered_amt |
| `documents` | doc_id, claim_id, type, text, source |
| `policies` | policy_id, title, cpt_list, icd10_list, criteria_text |

## 2.5 Path to real data
| Phase | Data |
|---|---|
| Development and testing | Public + synthetic (no PHI) |
| Initial model training | Public + synthetic, plus de-identified internal historical denials with real outcomes when available |
| Production | Live feeds (835, EHR, billing) and user uploads; models retrained on real outcomes |

## 2.6 Data quality checks
- Schema checks on every table (types, required fields)
- Code validity (CARC, CPT, ICD-10 exist in the reference lists)
- Distribution checks (denial rate and reason mix match the config)
- No real PHI in the proof-of-concept data

## 2.7 Document types users work with

### Groups
| Group | Documents | Format | System action |
|---|---|---|---|
| **1. Bill** | Claim (837), UB-04, CMS-1500, itemized bill | EDI, PDF, paper | EDI parsed directly; forms via OCR + LLM |
| **2. Insurer reply** | Remittance (835), EOB, claim status (277) | EDI, scanned PDF | EDI parsed directly; EOB via OCR + LLM |
| **3. Denial / payer letters** | Denial letter, records request, appeal decision, audit request | PDF, fax, scan | OCR + LLM extraction |
| **4. Medical records** | Progress notes, discharge summary, operative report, lab/imaging, orders, medications, therapy notes | EHR PDF export, scan | LLM searches for evidence and cites page |
| **5. Insurance rules** | Medical policies, LCD/NCD, contracts, payer manuals | PDF, web | Indexed for RAG (pgvector) |
| **6. Admin / patient** | Insurance card, registration form, prior auth approval, eligibility (271), consent | Image, PDF, EDI | OCR + LLM; 271 parsed directly |
| **7. Appeal paperwork** | Appeal letter, payer appeal form, proof of submission, corrected claim | Generated PDF, fax receipt | Letter drafted by AI, approved by a person |

### Challenges
- Mixed quality: clean EDI, PDFs, poor scans, faxes, handwriting
- Long records (50–500 pages) with the key evidence in one line
- Every payer uses a different letter layout
- PHI on almost every page

### Document classes for the classifier
`claim_form`, `eob`, `denial_letter`, `records_request`, `appeal_decision`, `clinical_note`, `discharge_summary`, `operative_report`, `lab_imaging`, `order`, `prior_auth`, `insurance_card`, `payer_policy`, `contract`, `other`

### Key fields to extract (denial letter / EOB)
claim_id, member_id, patient_name, payer, date_of_service, denial_date, billed_amt, paid_amt, denied_amt, group_code, carc, rarc, cpt, icd10, denial_reason_text, appeal_deadline, appeal_address_or_fax

### Public stand-ins for development and testing
| Document group | Stand-in |
|---|---|
| Bill, insurer reply (EDI) | Generated from SynPUF / Synthea claims |
| Denial letters, EOBs | **Generated from templates** (several payer layouts), some rendered as noisy scan-like PDFs to test OCR |
| Medical records | Hugging Face synthetic notes (e.g. Asclepius) |
| Insurance rules | CMS LCD/NCD documents |
| Forms / scans (OCR testing) | Hugging Face `nielsr/funsd`, `aharley/rvl_cdip` |
| Prior auth, insurance card | Generated from templates |
