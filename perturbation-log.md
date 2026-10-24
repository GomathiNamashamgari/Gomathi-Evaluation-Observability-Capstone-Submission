# Perturbation Log

For each system, make one deliberate change to an input or configuration, predict the outcome, run
it, and record what actually happened. See the starters in the Instructions, or design your own (your
own experiment earns more credit).

---

### System 1 — validated, routed pipeline

- **Change I made (file + what I changed):** Copied `data/policies/POL-2025-001.txt` to `perturbations/POL-2025-001-missing-endorsements.txt` and removed the `ENDORSEMENTS` section from the copy. The original policy was left unchanged.
- **Command I ran:** Attempted `.venv/bin/policy-extractor extract perturbations/POL-2025-001-missing-endorsements.txt --policy-id POL-2025-001`; the live run failed before extraction because no API authentication was configured. Then ran `.venv/bin/pytest tests/test_us01_retry.py::test_ac_01_04_missing_source_halts_immediately -v`.
- **What I predicted:** When extraction returns `endorsements=None`, validation will classify it as `missing_source` with pattern `endorsements_absent` and escalate immediately rather than retrying.
- **What actually happened:** The live CLI attempt stopped with an authentication error, so it did not verify extraction of the modified document. The targeted recorded-client test passed: `test_ac_01_04_missing_source_halts_immediately PASSED` (`1 passed`). That test asserts escalation for the missing endorsements field after one recorded-client call.
- **How this differs from the unperturbed run:** The modified input omits the endorsements section, while the original retains it. The controlled test verifies the pipeline's missing-source escalation behavior, but it is not a live extraction of the modified file. Output is saved in `01-policy-pipeline/perturbation-run.txt`.

---

### System 2 — schema-enforced two-pass extraction

- **Change I made (file + what I changed):** Created `/tmp/income_sum_corrected.txt` from `fixtures/documents/income_sum_mismatch.txt` and changed only the stated `TOTAL MONTHLY EARNINGS` from 10,892.17 to 9,642.17. The original fixture was not modified.
- **Command I ran:** `.venv/bin/mortgage-extract /tmp/income_sum_corrected.txt --mode replay` (full extraction could not run because no recorded response existed for the changed input); then ran an offline Python check using the project's `Income`, `MortgageExtraction`, and `validate()` implementation to compare the original and corrected totals.
- **What I predicted:** The original total would fail consistency validation by $1,250.00, while the corrected total would match the calculated component sum and pass.
- **What actually happened (paste the key output line):** Original: `Consistent: False`, discrepancy `{'field': 'total_monthly_income', 'calculated': 9642.17, 'stated': 10892.17, 'delta': -1250.0}`. Corrected: `Consistent: True`, `Discrepancies: []`.
- **How this differs from the unperturbed run:** Correcting the stated total removed the $1,250.00 arithmetic discrepancy in the real offline validator. The changed document's full replay extraction was not verified because its request had no cached response; the result above isolates and verifies the validator behavior without an API call.

---

### System 3 — multi-source synthesis

- **Change I made (file + what I changed):** Enabled the supply-chain coordinator's simulated logistics timeout using the `--simulate-timeout` option; no source files were edited.
- **Command I ran:** `.venv/bin/supply-chain-investigate meridian --offline --simulate-timeout`
- **What I predicted:** The coordinator would mark logistics-dependent evidence unavailable, avoid presenting the missing logistics values as confirmed facts, and identify metrics that could no longer be corroborated.
- **What actually happened (paste the key output line):** `Sources unavailable: logistics unavailable (timeout)`. The briefing marked `late_shipment_count` as `[missing source: timeout reading logistics]`; `on_time_delivery_rate` had only the supplier_audit value of 95.0 percent; and `production_capacity_utilization` remained incomplete because no source reported it.
- **How this differs from the unperturbed run:** The normal offline run included logistics evidence and showed a contested on-time delivery rate of 95.0 percent from supplier_audit versus 78.0 percent from logistics. With the timeout, the logistics value was unavailable, so the conflict disappeared from the report but the uncertainty did not mean the 95.0 percent value was independently confirmed. The briefing explicitly reported the source timeout.
