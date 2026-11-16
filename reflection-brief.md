# Reflection Brief — Evaluation and Observability Capstone

**Name:** Jolinna Mae Tibulan
**Date:** November 16, 2026

> Ground every answer in your own run. When a question asks for a number, file name, or line, paste
> it from your artifacts — a reviewer should be able to find it. Answers that are correct in the
> abstract but cite nothing do not meet the bar. Keep it short and specific.

---

## 0. Environment

| Field | Value |
|---|---|
| OS & version | Linux 6.6.97+ x86_64 GNU/Linux |
| Python version | Python 3.13.0 |
| Date run | November 16, 2026 |
| Ran any system live? (which) | No. System 1 had no API key available (`key set: False`), so I used the project-approved offline routing fallback. Systems 2 and 3 were run using replay/offline mode. |

Evidence: `environment.txt`.

---

## 1. Validated, routed pipeline

| Evidence | Value |
|---|---|
| Passing test count | 45 passed, 3 skipped |
| Routing output file | `01-policy-pipeline/routing.txt` — approved offline fallback because the live API key was unavailable |
| auto_approve / human_review / spot_check counts | Exercised deterministically by `tests/test_us04_routing.py`; 9/9 routing tests passed. Live pipeline counts were not produced because `ANTHROPIC_API_KEY` was unavailable. |

### 1a. Retry boundary

From `01-policy-pipeline/perturbation.txt`, the key result was:

`tests/test_us01_retry.py::test_ac_01_04_missing_source_halts_immediately PASSED`

The missing-required-field case makes one extraction attempt and then halts because the validator classifies the failure as `missing_source`. The same run shows that format failures and consistency failures are recoverable and retry, while missing-source failures do not. Retrying a value that was never present in the source would only add cost and latency and could encourage fabrication, so immediate escalation is safer. The perturbation suite completed with **13 passed, 1 skipped**.

### 1b. Reading the router

Because the live API key was unavailable, I used the approved routing fallback captured in `01-policy-pipeline/routing.txt`. One human-review example is:

`test_ac_04_06_high_confidence_plus_reviewer_disagreement_still_routes_to_human_review PASSED`

The independent **reviewer** signal sends this case to human review despite high model confidence. The same artifact independently verifies low-confidence and integration-failure routing through `test_ac_04_02_low_confidence_routes_to_human_review` and `test_ac_04_02_integration_failure_routes_to_human_review`. If routing trusted confidence alone, the high-confidence case could have been auto-approved despite disagreement from the independent reviewer.

### 1c. Where the aggregate lies

The calibration artifact `01-policy-pipeline/calibration.txt` contains:

`umbrella  exclusions  n=2 conf=0.93 acc=0.00 brier=0.865`

while the overall report contains:

`OVERALL brier=0.291`

The `umbrella × exclusions` slice therefore had **0.93 predicted confidence but 0.00 observed accuracy**, even though the aggregate Brier score appeared much less severe. Slicing by `policy_type × field` exposes a concentrated reliability problem that the overall score hides.

---

## 2. Schema-enforced two-pass extraction

| Evidence | Value |
|---|---|
| Passing test count | 25 passed |
| Document run | `02-mortgage/appraisal_informal_sqft.txt`, `02-mortgage/income_missing_bonus.txt`, `02-mortgage/income_sum_mismatch.txt` |
| Classified type | Appraisal for the appraisal fixture; income for the income fixtures |

### 2a. Two guarantees

In `02-mortgage/income_sum_mismatch.txt`, extraction succeeded structurally, but validation reported:

`"consistent": false`

with:

`"calculated": 9642.17`

`"stated": 10892.17`

`"delta": -1250.0`

Schema-enforced tool use and mathematical consistency checking provide different guarantees. The schema can force the response into valid structured fields and types, but structurally valid JSON can still contain values that contradict one another mathematically. The consistency validator catches that arithmetic problem, while it cannot by itself determine that a correctly typed text value was extracted from the wrong sentence. Both layers are therefore necessary.

### 2b. Refusing to fabricate

In `02-mortgage/income_missing_bonus.txt`, the extraction contains:

`"bonus_monthly": null`

The document did not provide a bonus value, so the extractor preserved absence rather than inventing one. The extraction schema deliberately allows nullable fields, giving “not stated in the document” an explicit machine-readable representation. This prevents downstream consumers from confusing missing evidence with a real numeric value such as zero.

### 2c. Normalization

In `02-mortgage/appraisal_informal_sqft.txt`, the informal source value “about 2,400 sq ft” was normalized to:

`"gross_living_area_sqft": 2400`

The same extraction reported `"consistent": true`. Normalizing during extraction gives downstream validation and underwriting code a stable typed value rather than requiring every subsequent component to repeatedly parse commas, qualifiers, and unit text. This keeps downstream operations simpler and more deterministic.

---

## 3. Multi-source synthesis

| Evidence | Value |
|---|---|
| Passing test count | 34 passed |
| Briefing file | `03-supply-chain/investigation.txt` |
| Section the conflict landed in | Contested |

### 3a. Annotate, don't arbitrate

The Contested section of `03-supply-chain/investigation.txt` preserves:

`95.0 percent — supplier_audit (as of 2026-04-10)`

and:

`78.0 percent — logistics (as of 2026-04-05)`

for `on_time_delivery_rate`. Rather than averaging the values or arbitrarily choosing one, the system retains both measurements with their sources and dates. This gives a decision-maker the evidence needed to judge the disagreement directly; collapsing them into one value would hide both the conflict and its provenance.

### 3b. Source goes dark

The perturbation artifact `03-supply-chain/perturbation-timeout.txt` reports:

`Sources unavailable: logistics unavailable (timeout)`

but the investigation still finishes. Logistics-dependent `late_shipment_count` appears under **Incomplete** with:

`missing source: timeout reading logistics`

An unreachable source is therefore not interpreted as a source having “nothing to report.” The coordinator preserves the distinction between unavailable evidence and absent evidence, continues with the remaining sources, and avoids fabricating a replacement result.

### 3c. Dates as a guardrail

The briefing reports the same delivery metric at different dates:

`95.0 percent — supplier_audit (as of 2026-04-10)`

and:

`78.0 percent — logistics (as of 2026-04-05)`

Dates are important provenance because observations made at different times do not necessarily describe an identical system state. Keeping the dates prevents an ordinary temporal change from automatically being interpreted as a contradiction. In this run the difference remains material enough to be classified as Contested, but a reviewer can still see exactly when each observation applies.

---

## 4. Synthesis

### 4a. One principle

The clearest example of **evaluate the output, don't trust the model's word** is `02-mortgage/income_sum_mismatch.txt`. The extractor returned structurally valid data, but the deterministic validator recomputed monthly income as **9642.17** while the stated total was **10892.17**, revealing a **-1250.0** discrepancy. A system that trusted schema-valid model output without evaluating its internal consistency could have accepted an incorrect income figure. This artifact demonstrates why successful extraction is not the same as verified extraction.

### 4b. Confidence ≠ correctness

This mattered most clearly in System 1's `01-policy-pipeline/calibration.txt`. The `umbrella / exclusions` slice had **0.93 predicted confidence**, **0.00 observed accuracy**, and a **0.865 Brier score**, while the overall Brier score was only **0.291**. The model could therefore be highly confident in a field where its observed accuracy was zero. Per-slice observability exposed that reliability failure where an aggregate metric alone could have hidden it.

### 4c. Apply it

A relevant workflow in my own technical work is extracting implementation requirements from SAP functional specifications, defect tickets, and change documentation into structured fields such as object name, affected output, required condition, change type, and implementation requirement. I would start with **validated retry with escalation** because missing or inconsistent source details should not silently become implementation assumptions. I would instrument validation-error category, retry count, nullable-required-field rate, escalation rate, field-level confidence/accuracy slices, and a sampled human-review accuracy measure. The calibration result in `01-policy-pipeline/calibration.txt` shows why those measurements should be sliced by field or work type instead of relying only on an overall score.