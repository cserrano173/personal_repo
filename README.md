# Serif Health Take-Home: Data Integration

Integrating Hospital Price Transparency (HPT) and Transparency in Coverage (TIC) data
for three NYC hospitals: Montefiore Medical Center, Mount Sinai Hospital, and NYU Langone Tisch.

### Data Sources

- **HPT** (Hospital Price Transparency): Published by hospitals. Contains negotiated rates
between hospitals and payers, along with gross charges, discounted cash prices, and
methodology notes. 2,950 rows across 3 hospitals.
- **TIC** (Transparency in Coverage): Published by health insurers. Contains negotiated
rates from the payer's perspective, with provider-level (NPI) granularity and Medicare
benchmarks. 222 rows across 3 payers (UnitedHealthcare, Aetna, Cigna).

## Findings

### Hospital Identification

- HPT `license_number` is **not a reliable EIN**. Mount Sinai's value (`330024`) is a state
license number; NYU Langone's (`7002053H`) is a facility code. The actual EINs are
embedded in the `source_file_name` field.
- All three hospitals (EINs: 131740114, 131624096, 133971298) are present in both datasets.

### Payer Normalization

- HPT uses 73 distinct payer names. TIC uses 3 canonical names: `unitedhealthcare`,
`aetna`, `cigna-corporation`.
- HPT variants mapped to UHC: `United`, `UHC`, `United Healthcare`, `Oxford`.
Oxford is a UHC subsidiary (Oxford Health Plans); HPT plan names like "United Oxford"
confirm this relationship.
- After filtering to common payers, HPT reduces from 2,950 to 401 rows (14%).

### Code Normalization

- Three procedure codes in the dataset: CPT 99283 (ER visit), CPT 43239 (endoscopy),
MS-DRG 872 (sepsis inpatient stay).
- NYU Langone embeds the code type prefix in `raw_code` (e.g. `"MS-DRG 872"` instead
of `"872"`). Stripped during cleaning.
- NYU Langone uses `LOCAL` code type for hospital-internal supply codes (e.g. a shoulder
implant) that reuse the same numeric code as standard CPT codes but refer to completely
different items. These have no TIC counterpart and are excluded.

### Rate Multiplicity

- **TIC**: A single (hospital, payer, code) combination yields multiple rates, driven by:
  - `taxonomy_filtered_npi_list` — different providers/groups negotiate different rates
  - `billing_class` — professional vs. institutional fee schedules
  - Some true duplicates exist (same NPI list, different rates, all other columns identical)
  — likely unexposed contract tiers or data quality issues.
- **HPT**: Rate multiplicity is driven by:
  - `plan_name` — sub-tiers within a payer (HMO, PPO, EPO, Medicare Advantage, etc.)
  - `setting` — inpatient vs. outpatient vs. both
  - `standard_charge_methodology` — case rate vs. fee schedule vs. percent of charges
  - `description` — same code can have multiple chargemaster descriptions (see below)

### Network Coverage

- The TIC extract contains exactly **one network per payer**: UHC "choice-plus", Aetna
"open-access-managed-choice", Cigna "national-oap".
- HPT lists rates across many plan tiers (HMO, PPO, EPO, Medicare Advantage, etc.) that
may span multiple payer networks. Since the TIC extract covers only one network per payer,
many HPT plan-tier rates will have no TIC counterpart — not because the data disagrees,
but because the TIC extract simply doesn't include that network.
- This is a fundamental coverage limitation, not a matching failure.

### Billing Class Inference

- TIC has an explicit `billing_class` field (professional vs. institutional).
- HPT does not. We infer it conservatively from two reliable signals:
  - `code_type = MS-DRG` → institutional (DRGs are always bundled inpatient facility charges)
  - Description prefix `PR`  → professional (physician fee), used consistently at Montefiore
- The `HC` prefix was initially considered as a facility indicator, but investigation showed
it is **not reliable**: at NYU Langone, non-HC rows sometimes have higher rates than HC
rows for the same code, suggesting HC is a generic chargemaster prefix rather than a
billing class signal. All non-PR, non-MSDRG rows are left as NaN.
- Result: ~66 institutional (DRGs), ~27 professional (PR prefix), ~305 unknown.

### Description Variability

- The same CPT code has multiple descriptions even within a single hospital.
For example, Montefiore lists CPT 99283 as "EMERGENCY DEPT VISIT LOW MDM",
"HC EMERGENCY DEPT VISIT LVL 3", and "PR EMERGENCY DEPARTMENT VISIT LOW MDM".
- PR prefix reliably indicates professional (physician) charges at Montefiore. HC prefix
is unreliable as a facility indicator (see Billing Class Inference above).
- Mount Sinai's "HC Psych ED Visit Level 3" reuses CPT 99283 for psychiatric ER visits — a
different clinical context. These rows are dropped.
- Description is not a reliable join key; code + code_type is the canonical identifier.

### Rate Cleaning

- HPT has three interrelated rate fields: `negotiated_rate` (dollars), `negotiated_pct`
(percentage), and `gross_charge`. The relationship `rate = pct × gross / 100` holds
exactly (max diff = $0.01) for all rows where all three are present.
- Missing values are filled using this relationship where two of the three are available.
- TIC has 3 rows where `negotiation_type = 'percentage'` — the rate value (e.g. 70)
represents 70% of billed charges, not a dollar amount and not a percentage of Medicare.
These are dropped (simplification; converting to dollars would require the HPT gross charge).
- "Lesser of 100% of billed charges applies" is a common contract clause in HPT
`additional_payer_notes`. It caps the actual payment at the gross charge, meaning
negotiated rates that exceed gross charges are contractually valid but capped in practice.

### CMS Benchmarking

- TIC provides `cms_baseline_rate` (Medicare reimbursement) for each row.
- Typical commercial-to-Medicare ratios range from 1.5× to 5×.
- 8 TIC rows exceed 20× Medicare — flagged as potential outliers for review.

## Assumptions & Simplifications

1. **Oxford = UHC**: Oxford Health Plans is treated as a UnitedHealthcare subsidiary based
  on HPT plan names ("United Oxford") and domain knowledge.
2. **EIN from filename**: EINs are extracted from `source_file_name` via a hardcoded map
  for the three known hospitals. This would not scale without a systematic EIN lookup.
3. **Billing class inference is conservative**: Only PR-prefixed descriptions (professional)
  and MS-DRG codes (institutional) are classified. The HC prefix was found to be unreliable
   as a facility indicator. ~77% of HPT rows have NaN billing class.
4. **TIC percentage rates dropped**: 3 rows with `negotiation_type = 'percentage'` are
  excluded rather than converted to dollar amounts.
5. **Psych ED rows dropped**: 3 Mount Sinai rows using CPT 99283 for psychiatric ER
  visits are excluded as not comparable to general ER visit rates.
6. **Single time period**: Both datasets represent a single snapshot (early 2025).
  Rate changes over time are not captured.
7. **FFS only**: All TIC rows use `arrangement = 'ffs'` (fee-for-service). Capitation
  or bundled payment arrangements would require different handling.
8. **HC prefix not used**: The `HC` description prefix is not treated as a facility/
  institutional indicator due to inconsistent usage across hospitals.

## Matching Methodology

### Step 1: Deterministic Blocking

Generate candidate pairs by inner-joining on four hard keys:
**EIN × Code × Code_Type × Payer**

These fields are unambiguous in both datasets and isolate the matching space to specific
hospital-procedure-payer triples. This produces a many-to-many set of candidates
(~7 TIC candidates per HPT row on average).

### Step 2: Confidence Scoring (0–100)

Each candidate pair is scored on three components:


| Component               | Max | Logic                                                                                                     |
| ----------------------- | --- | --------------------------------------------------------------------------------------------------------- |
| **Rate proximity**      | 50  | Exact match (<0.1%): 50, within 5%: 40, within 20%: 25, within 50%: 10, else: 0                           |
| **Billing class**       | 30  | Match: 30, HPT unknown (NaN): 10, known mismatch: 0                                                       |
| **Setting consistency** | 20  | inpatient↔institutional: 20, outpatient↔professional: 20, outpatient↔institutional: 15, both: 10, else: 5 |


Rate proximity is the dominant component because, within a block (same hospital, code,
payer), matching dollar amounts are the strongest evidence that two rows describe the
same negotiated arrangement.

### Step 3: Classification


| Tier         | Score  | Interpretation                                                   |
| ------------ | ------ | ---------------------------------------------------------------- |
| **High**     | 70–100 | Strong structural + rate alignment                               |
| **Medium**   | 50–69  | Partial alignment (e.g. rates within 20%, billing class unknown) |
| **Low**      | 30–49  | Weak alignment (same block but rates diverge)                    |
| **Unlikely** | 0–29   | Same block but billing class mismatch or rates >50% apart        |


For each HPT row, we select the TIC candidate with the highest score as the "best match."

### Design Decisions

- **Plan type not used as a key**: HPT plan names and TIC network names have no crosswalk.
Including plan type as a hard key would discard valid matches. Rate proximity implicitly
captures plan alignment — matching rates are evidence of the same contract tier.
- **Rate proximity > text similarity**: Dollar amounts are concrete and comparable; free-form
names are not.
- **NaN billing class gets partial credit (10/30)**: ~77% of HPT rows have unknown billing
class. Penalizing them fully would dominate the score for the majority of rows.

### Results

- 398 HPT rows → 321 with at least one TIC candidate (81%)
- 166 high-confidence matches (42% of matched), of which 146 have exact rate matches
- 77 HPT rows unmatched (LOCAL code types or plan tiers outside the TIC network extract)

### Scaling Considerations

This approach was designed for a small sample (398 × 219 rows). At production scale —
thousands of hospitals, hundreds of payers, tens of thousands of codes — several aspects
would need to change:

**Blocking complexity**: The many-to-many join on (EIN × Code × Code_Type × Payer) produces
~7 candidates per HPT row here, but this ratio grows with the number of NPI-level TIC rows
per block. A hospital with 500 contracted providers for a single code would generate 500
candidate pairs per HPT row. At scale (millions of HPT rows × hundreds of TIC rows per
block), the candidate set could reach billions of pairs.

**Mitigations**:

- **Tighter blocking keys**: Add `billing_class` to the blocking key where available to
reduce block sizes. At scale, investing in better billing class inference (e.g., using
revenue codes, charge amount ranges, or facility/professional CMS fee schedule lookups)
would pay off by shrinking blocks significantly.
- **Top-K pruning**: Instead of scoring all candidates, pre-filter to the K nearest rates
within each block using a sorted index or KD-tree on rate values. Most unlikely pairs
can be eliminated without computing the full score.
- **Partitioning**: Blocks are independent — the scoring for (Hospital A, CPT 99283, Aetna)
doesn't depend on (Hospital B, CPT 43239, Cigna). This is embarrassingly parallel and
can be distributed across workers partitioned by EIN or payer.
- **Columnar storage**: Move from CSV to Parquet for faster I/O and predicate pushdown
on blocking keys.

**Payer normalization**: The hardcoded payer name mapping (Oxford → UHC, etc.) doesn't
scale to thousands of payer variants. A production system would need a maintained payer
alias table or fuzzy matching specifically on payer names (where text similarity does work,
unlike plan↔network names).

**EIN resolution**: Extracting EINs from filenames is a sample-specific hack. At scale,
this requires a hospital registry or NPI-to-EIN crosswalk (e.g., CMS Provider of Services
file or NPPES data).

**Scoring weights**: The current weights (50/30/20) were chosen by judgment for this sample.
At scale, these could be tuned empirically using a labeled set of known matches, or replaced
with a learned model (e.g., logistic regression on the score components).

## Concrete Examples

### Example 1: CPT 43239 / Mount Sinai / UHC — The $6,438 Match

Mount Sinai's HPT file lists $6,438 for CPT 43239 (endoscopy with biopsy) under UHC.
The same rate appears in the UHC TIC extract. Both datasets contain additional rates:

**HPT** (6 rows): $6,438 ("All Payer", case rate), $4,681 (Oxford "All Payer", case rate),
$1,048.28 (Medicare Advantage / Oxford Government, fee schedule — appears 4 times across
different descriptions).

**TIC** (8 rows): Institutional rates of $6,438, $4,608, $2,670, $1,532 and professional
rates of $862.79, $514.22, $311.55, $270.93.

**What changes when the price varies?**

In HPT, rates vary by `plan_name` (different contract tiers), `standard_charge_methodology`
(case rate vs. fee schedule), and `description` (different chargemaster entries for the
same CPT code). In TIC, rates vary by `billing_class` (institutional vs. professional) and
`taxonomy_filtered_npi_list` (different provider groups negotiate different rates).

**Can other records be aligned?**

The $6,438 match is high-confidence: same hospital, same payer, same code, identical dollar
amount, and TIC explicitly tags it as institutional. The HPT side labels it "case rate" —
a pricing methodology, not a billing class, but the dollar magnitude is consistent with a
facility charge. The other HPT rows are different plan tiers (Oxford, Medicare Advantage)
that likely correspond to a different UHC network than "choice-plus" (the only network in
the TIC extract). The TIC professional rates ($270–$862) represent physician fees that
could align with HPT professional charges, but Mount Sinai doesn't use PR-prefixed
descriptions for this code, so billing class remains unconfirmed.

### Example 2: DRG 872 / Mount Sinai / Aetna — Rate $29,259.18

Mount Sinai lists an Aetna HMO/POS/PPO case rate of $29,259.18 for DRG 872 (sepsis).
This rate does **not** appear in the Aetna TIC extract.

**HPT** (4 rows): $29,259.18 (HMO/POS/PPO), $24,577.49 (Blackstone/Student Health),
$13,708.20 (Medicare), $36,573.98 (Signature Administrators). All are inpatient
institutional case rates.

**TIC** (6 rows): $17,966, $13,365, $9,206, $8,757 — all institutional, all under the
"open-access-managed-choice" network, varying by NPI list.

The $29,259.18 rate exceeds every TIC rate for this block. This is a structural mismatch:
the hospital publishes one rate for a broad plan tier ("HMO/POS/PPO"), while the payer
publishes multiple NPI-specific rates under a specific network that may not correspond to
that tier. The appropriate handling is to report as unmatched (most honest), note the range
comparison ($29K vs. $8K–$18K range), or align to the nearest TIC rate ($17,966) with a
low confidence score flagging the 63% gap.

### Structural Observations

**What is DRG 872, and why do DRG rates behave differently?**

DRG 872 (sepsis/severe sepsis without MV >96 hours) is a bundled inpatient payment — a
single lump sum covering the entire hospital stay (room, nursing, labs, medications). These
rates are high-dollar ($8K–$50K+), always institutional (no professional component), and
more sensitive to plan tier. The spread between Aetna's cheapest ($13,708 Medicare) and
most expensive ($36,574 Signature Administrators) plan for the same DRG at the same
hospital is nearly 3:1.

CPT codes like 43239 and 99283 split into facility and professional components, have more
rate variants due to NPI-level negotiation on the TIC side, and are lower-dollar — making
exact matches more likely. DRG matching will structurally have fewer exact matches and
wider rate gaps.

**Should plan type (PPO, HMO, EPO) be a matching key or a confidence score component?**

It belongs in the **confidence score**, not the matching key:
- HPT uses plan names like "unitedhealthcareppo1160"; TIC uses network names like
  "choice-plus" — no clean crosswalk exists between the two
- A single TIC network often covers multiple HPT plan types
- Making plan type a hard key would discard valid matches where the plan-to-network
  mapping is unknown (the majority of cases)
- As a score component, plan type mismatches reduce confidence without eliminating the
  match — more appropriate given the data quality reality

**When matched records have materially different rates, what are the legitimate reasons?**

1. **Different plan tiers**: HPT "HMO/POS/PPO" vs. TIC's specific network — the hospital
   may publish a blended/highest-tier rate while TIC reports the network-specific rate
2. **Different providers (NPIs)**: TIC rates vary by which provider performs the service;
   HPT reports hospital-level rates
3. **Timing**: HPT may reflect a different contract effective date; renegotiations happen
   periodically
4. **Lesser-of clause**: HPT contractual rate exceeds gross charge but actual payment is
   capped; TIC may reflect the capped amount
5. **Methodology**: "Case rate" in HPT vs. "fee schedule" in TIC may produce different
   amounts for the same service
6. **Data errors**: Either dataset may contain stale, duplicated, or incorrectly mapped rates

Rate disagreements should not be discarded — the delta itself is a valuable signal. Small
deltas suggest consistent contract data; large deltas indicate different contract terms or
data quality issues warranting investigation.

## Scaling with Agentic AI

The "Scaling Considerations" above address computational scaling (blocking, partitioning,
storage). The harder bottleneck at national scale is the judgment-intensive work that
currently requires human attention for each new hospital or payer. An agentic AI system
could address this by automating the cognitive long tail:

**Payer Normalization Agent**: The hardcoded Oxford → UHC mapping required inspecting HPT
plan names ("United Oxford") and knowing Oxford Health Plans is a UHC subsidiary. At scale,
with thousands of payer name variants, an agent could maintain and continuously update a
payer alias table — identifying subsidiaries, DBAs, and regional brands by cross-referencing
plan name patterns, CMS registration data, and corporate ownership databases. Ambiguous
mappings (e.g., "Empire" which could be Empire BCBS or a local plan) would be flagged for
human review with supporting evidence.

**EIN & Entity Resolution Agent**: Extracting EINs from filenames is a sample-specific hack.
An agent could orchestrate multi-source lookups — querying NPPES, the CMS Provider of
Services file, state license databases, and hospital websites — to resolve hospital identity.
When the HPT `license_number` field contains a state facility code (like Mount Sinai's
`330024`), the agent would recognize the format and route to the appropriate state registry
rather than treating it as an EIN.

**Billing Class Inference Agent**: The HC prefix investigation (initially promising, then
found unreliable at NYU Langone) is exactly the kind of multi-step reasoning an agent
excels at. Given a new hospital's HPT data, an agent could: (1) identify description
prefixes and patterns, (2) test whether they correlate with rate magnitudes or settings,
(3) cross-reference with CMS physician/facility fee schedule rates to validate
classifications, and (4) report confidence in the inferred billing class per row.

**Data Quality & Anomaly Triage**: The 8 TIC rows exceeding 20× Medicare, the "true
duplicates" with identical NPI lists but different rates, Mount Sinai reusing CPT 99283 for
psychiatric ED visits — these edge cases surfaced through manual inspection. An agent could
monitor incoming data drops for schema drift, flag statistical outliers against Medicare
benchmarks, detect code reuse for clinically distinct services (using CPT descriptor
lookups), and route exceptions to analysts with pre-computed context.

**Adaptive Scoring**: The current 50/30/20 weights were chosen by judgment. An agent could
propose weight adjustments based on observed match quality — running the scoring pipeline
on new hospital data, identifying systematic failures (e.g., blocks where billing class
dominates rate proximity), and suggesting recalibration. With a labeled validation set,
the agent could iteratively test weight combinations and report precision/recall tradeoffs.

**Orchestration at Scale**: As new hospital files arrive monthly (the assignment notes
"several billion records"), an agent could orchestrate the end-to-end pipeline:
ingest → validate schema compliance → clean → normalize payer/codes → match → score →
flag exceptions → generate delta reports vs. previous month. The human role shifts from
executing the pipeline to reviewing agent-flagged exceptions and approving alias table
updates.

The key design principle is that the agent handles the **long tail** — the thousands of
small decisions (is this payer name a variant? is this description prefix meaningful? is
this rate an outlier or a legitimate contract?) that individually require judgment but
collectively don't scale with human analysts.

## Schema Mapping


| Concept             | HPT Column                              | TIC Column                         |
| ------------------- | --------------------------------------- | ---------------------------------- |
| Hospital ID         | `source_file_name` → `ein`              | `ein`                              |
| Payer               | `payer_name` → `payer`                  | `payer`                            |
| Plan                | `plan_name`                             | `network_name`                     |
| Procedure Code      | `raw_code` → `code` + `code_type`       | `code` + `code_type`               |
| Negotiated Rate ($) | `standard_charge_negotiated_dollar`     | `rate` (when negotiated)           |
| Rate as %           | `standard_charge_negotiated_percentage` | `rate` (when percentage)           |
| Gross / List Price  | `standard_charge_gross`                 | *(not available)*                  |
| Care Setting        | `setting`                               | `billing_class`                    |
| Place of Service    | *(not available)*                       | `place_of_service_list`            |
| Rate Methodology    | `standard_charge_methodology`           | `negotiation_type` + `arrangement` |
| Medicare Benchmark  | *(not available)*                       | `cms_baseline_rate`                |
| Provider Detail     | *(hospital-level only)*                 | `taxonomy_filtered_npi_list`       |


## Files

- `serif_health_analysis.ipynb` — Clean notebook with data inspection, cleaning, matching, and analysis
- `hpt_extract_20250213.csv` — Hospital Price Transparency data sample
- `tic_extract_20250213.csv` — Transparency in Coverage data sample
- `unified_output.csv` — Final unified dataset with match scores and rate deltas

## How to Run

```bash
pip install pandas numpy
jupyter notebook serif_health_analysis.ipynb
```

Run all cells sequentially. The notebook reads the two CSV files from the same directory
and produces `unified_output.csv` as the final output.

