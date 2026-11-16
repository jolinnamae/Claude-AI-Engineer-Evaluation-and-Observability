# Perturbation Log

For each system, make one deliberate change to an input or configuration, predict the outcome, run
it, and record what actually happened. See the starters in the Instructions, or design your own (your
own experiment earns more credit).

---

### System 1 — validated, routed pipeline

- **Change I made (file + what I changed):** Exercised the provided perturbation case in `tests/test_us01_retry.py` where a required source field is null/missing. This represents changing a normally valid required field to a missing-source condition.
- **Command I ran:** `.venv/bin/pytest tests/test_us01_retry.py -v`
- **What I predicted:** A missing required value should be classified as `missing_source` and halt immediately rather than retrying, because another API call cannot recover information that is absent from the source document.
- **What actually happened (paste the key output line):** `tests/test_us01_retry.py::test_ac_01_04_missing_source_halts_immediately PASSED`. The perturbation suite completed with **13 passed, 1 skipped**. The same run also showed `format_failure_retries_with_error_appended PASSED` and `consistency_failure_also_retries PASSED`.
- **How this differs from the unperturbed run:** A valid extraction is accepted normally, while recoverable format or consistency errors are retried. The missing-source perturbation is treated differently: the system stops immediately rather than performing a futile retry. Evidence: `01-policy-pipeline/perturbation.txt`.

---

### System 2 — schema-enforced two-pass extraction

- **Change I made (file + what I changed):** Used the supplied `fixtures/documents/income_sum_mismatch.txt` perturbation, where the stated monthly income total does not match the sum of the extracted income components, and contrasted it with the clean `appraisal_informal_sqft.txt` fixture.
- **Command I ran:** `.venv/bin/mortgage-extract fixtures/documents/income_sum_mismatch.txt --mode replay`
- **What I predicted:** The extraction should still complete, but the deterministic consistency validator should mark the result inconsistent and report the arithmetic difference instead of silently accepting the stated total.
- **What actually happened (paste the key output line):** `"consistent": false` with `"calculated": 9642.17`, `"stated": 10892.17`, and `"delta": -1250.0`.
- **How this differs from the unperturbed run:** The clean appraisal run returned `"consistent": true` with no discrepancies and normalized the informal square-foot value to `gross_living_area_sqft: 2400`. The mismatch fixture preserved both calculated and stated values and surfaced the **-1250.0** discrepancy. Evidence: `02-mortgage/income_sum_mismatch.txt` and `02-mortgage/appraisal_informal_sqft.txt`.

---

### System 3 — multi-source synthesis

- **Change I made (file + what I changed):** Changed the investigation configuration by adding `--simulate-timeout`, deliberately forcing the logistics source to become unavailable.
- **Command I ran:** `.venv/bin/supply-chain-investigate meridian --offline --simulate-timeout`
- **What I predicted:** The investigation should complete rather than abort. Claims dependent on logistics should be marked Incomplete with an explicit source-failure annotation, while information supported by the remaining sources should still appear.
- **What actually happened (paste the key output line):** `Sources unavailable: logistics unavailable (timeout)`. The run still completed, and `late_shipment_count` appeared under **Incomplete** with `missing source: timeout reading logistics`.
- **How this differs from the unperturbed run:** With logistics available, the normal briefing retained both conflicting `on_time_delivery_rate` values: **95.0 percent — supplier_audit (as of 2026-04-10)** and **78.0 percent — logistics (as of 2026-04-05)** under Contested. With the simulated timeout, the logistics-dependent evidence was unavailable rather than fabricated, and the system explicitly represented that uncertainty in the Incomplete section. Evidence: `03-supply-chain/investigation.txt` and `03-supply-chain/perturbation-timeout.txt`.