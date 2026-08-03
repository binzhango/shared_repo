# Evidently Design for Finance Transformation-Output Evaluation

Version: 1.0  
Date: 2026-08-02  
Status: implementation design

## 1. Design objective

Build Evidently into a reusable evaluation framework for DataFrames produced by a finance transformation agent.

The framework must answer:

> Is the transformed output financially meaningful, representative of the intended population, stable across releases, robust in important slices and tails, and fit for its downstream use?

## 2. Framework boundary

### Inputs

The evaluation runner receives:

1. Candidate transformed DataFrame
2. Agent/transformation version manifest
3. Gold output reference, where available
4. Target-population reference
5. Previous approved release reference
6. Versioned Evidently evaluation specification
7. Optional downstream evaluation results

### Outputs

Every run produces:

- Evidently metric values
- Evidently Test results
- Results by registered financial slice
- Hard-gate result
- Weighted dimension scores
- Final state: `NOT_EVALUABLE`, `PASS`, `CONDITIONAL_PASS`, `FAIL`, or `ERROR`
- HTML evaluation reports
- JSON/dictionary results
- Evidence tables for failed or material cases
- Immutable evaluation manifest and checksums

### Out of scope

The evaluation framework does not perform transformation-contract validation such as:

- Schema and datatype validation
- Required-column and nullability contracts
- Exact formula implementation tests
- Referential integrity and join cardinality
- Aggregate reconciliation required by the transformation specification
- Uniqueness and identifier-format checks

## 3. Runtime architecture

The framework contains eight logical services.

| Component | Responsibility |
|---|---|
| Run-manifest builder | Freezes versions, hashes, intended use, references, metric spec and timestamps |
| Data adapter | Converts DataFrames into explicit Evidently `Dataset` objects |
| Reference registry | Resolves immutable gold, population and release references |
| Evaluation registry | Resolves data definitions, metrics, slices, thresholds and score policy |
| Report factory | Builds the registered Evidently Reports and Tests |
| Finance metric pack | Implements custom dataset- and column-level financial metrics |
| Decision engine | Applies hard gates, dimension floors, weighted score and approval policy |
| Artifact publisher | Retains Evidently reports, results, evidence, decision and checksums |

Core flow:

```text
Candidate transformed output
        ↓
Run manifest + Dataset/DataDefinition adapter
        ↓
Gold + population + previous-release comparisons
        ↓
Built-in Evidently metrics + custom finance metrics + Tests
        ↓
Hard gates → dimension floors → weighted score
        ↓
Decision and evidence artifacts
```

## 4. Evaluation contracts

### 4.1 Evaluation run manifest

```yaml
evaluation_run_id: eval_20260802_001
evaluation_spec_version: finance_output_eval_v1
use_case: portfolio_concentration_analysis
candidate_dataset_id: transformed_holdings_20260802
candidate_sha256: "..."
agent_version: transformation_agent_v7
transformation_policy_version: portfolio_transform_v11
references:
  gold: gold_holdings_cases_v4
  population: portfolio_population_2024_2026_v3
  previous_release: transformed_holdings_release_v6
started_at: "2026-08-02T14:16:00Z"
```

### 4.2 Evaluation specification

The specification is configuration, not code. It defines:

- Dataset grain and primary alignment keys
- Explicit Evidently `DataDefinition`
- Material and excluded columns
- Reference bindings
- Report definitions
- Metric definitions and owners
- Financial slice definitions
- Hard gates
- Dimension weights and floors
- Overall threshold
- Evidence and retention policy

Example:

```yaml
spec_version: finance_output_eval_v1
dataset:
  grain: portfolio_id + valuation_timestamp + security_id
  keys: [portfolio_id, valuation_timestamp, security_id]
  evaluation_timestamp: information_available_at

data_definition:
  id_column: evaluation_row_id
  timestamp: valuation_timestamp
  numerical_columns:
    - market_value_base
    - portfolio_weight
    - gross_exposure
    - net_exposure
  categorical_columns:
    - asset_class
    - sector
    - currency
    - instrument_type
    - portfolio_type
  datetime_columns:
    - valuation_timestamp
    - information_available_at

excluded_from_drift:
  - evaluation_row_id
  - portfolio_id
  - security_id
  - valuation_timestamp
  - information_available_at

slices:
  - name: asset_class
    column: asset_class
    critical_values: [equity, fixed_income, derivative, cash]
  - name: market_regime
    column: market_regime
    critical_values: [normal, stress]
  - name: exposure_tail
    expression: abs(gross_exposure) >= population_p99

score_policy:
  overall_minimum: 90
  dimension_floor: 80
  hard_gates_must_pass: true
```

## 5. Reference model

Maintain three references because each answers a different question.

### 5.1 Gold reference

Question:

> For the same source input, does the agent output agree with independently approved financial output?

Construction:

- Independently calculated or expert-adjudicated
- Same source cases as the candidate
- Key-aligned before evaluation
- Includes acceptable-alternative annotations where more than one methodology is valid
- Includes materiality and severity per case

Do not pass the raw gold and candidate tables directly to a distribution report when row-level alignment matters. Build an aligned comparison DataFrame:

```text
keys
candidate_<field>
expected_<field>
absolute_error
relative_error
material_difference
acceptable_method
slice columns
```

For numerical outputs, the aligned frame may use Evidently regression mappings with `expected_<field>` as target and `candidate_<field>` as prediction. Use custom metrics for categorical agreement, acceptable alternatives and materiality-aware differences.

### 5.2 Target-population reference

Question:

> Is the candidate output representative of the financial population and conditions for which it will be used?

Construction:

- Defines target entities, products, currencies, periods and regimes
- Includes failed, defaulted, delisted, merged, liquidated or otherwise adverse cases where relevant
- Preserves legitimate tail events
- Uses peer groups appropriate to the financial concept

Use it for:

- Coverage
- Distribution distance
- Category-share comparison
- Tail representation
- Correlation and relationship changes
- Bias and missingness-pattern analysis

### 5.3 Previous approved release

Question:

> Did the new transformation release change output behavior beyond the approved release tolerance?

Construction:

- Immutable output from the last approved release
- Comparable source population and period, or an explicitly fixed regression corpus
- Linked to its agent, policy and evaluation versions

Release drift is diagnostic. A difference from the previous release is not automatically bad, and similarity is not proof of correctness.

## 6. Evidently dataset design

Current Evidently uses `Dataset` and `DataDefinition`. Use explicit mapping; do not rely on automatic type inference for finance data.

Conceptual factory:

```python
from evidently import Dataset, DataDefinition

definition = DataDefinition(
    id_column="evaluation_row_id",
    timestamp="valuation_timestamp",
    numerical_columns=[...],
    categorical_columns=[...],
    datetime_columns=[...],
)

candidate = Dataset.from_pandas(
    candidate_df,
    data_definition=definition,
)

population_reference = Dataset.from_pandas(
    population_reference_df,
    data_definition=definition,
)
```

Rules:

- Candidate and reference in one Report must use identical definitions.
- IDs and timestamps should normally be excluded from drift metrics.
- Numerically encoded categories must be declared categorical.
- Units and currencies must already be normalized in the candidate output.
- Evaluation-only helper columns must be explicitly named and documented.
- Slice and evidence columns should be retained in the source DataFrame even if not included in all Evidently reports.

## 7. Report suite

Create five Evidently report families. Each has a distinct purpose and reference.

### 7.1 Candidate profile report

Input:

- Candidate only

Purpose:

- Create an interpretable profile of the evaluated output
- Capture counts, distributions, quantiles and categories needed for evidence

Contents:

- `DataSummaryPreset`
- Selected material-column metrics
- Quantiles for tails and important ratios
- Category counts for critical financial populations
- Custom population-coverage metrics where the denominator is external

This report is descriptive and focused on fitness evidence rather than transformation-contract checks.

### 7.2 Population fitness report

Input:

- Current: candidate
- Reference: target population

Purpose:

- Measure representativeness and economically meaningful distribution differences

Contents:

- `DataDriftPreset` restricted to material columns
- Explicit `ValueDrift` metrics for critical outputs
- Quantile and category-share comparison
- Tail representation
- Peer-group/slice reports

Do not use Evidently's default share-of-drifting-columns decision as the finance release gate. Register methods and thresholds per material column or peer group.

### 7.3 Release regression report

Input:

- Current: candidate from new release
- Reference: previous approved release on the fixed regression corpus

Purpose:

- Detect unexpected transformation changes

Contents:

- Material output drift
- Category-share changes
- Correlation changes
- Custom rank-stability and decision-stability metrics
- Release-change evidence table

Expected changes must be linked to an approved change request. Unexplained material change is a gate failure or review trigger.

### 7.4 Gold agreement report

Input:

- Row-aligned candidate/gold comparison frame

Purpose:

- Measure agreement with independent financial output

Contents:

- Numerical error metrics where appropriate
- Material disagreement rate
- Categorical agreement
- Acceptable-method agreement
- Critical-case pass rate
- Expert-review override rate

Numerical closeness alone is insufficient when the wrong financial method happens to produce a similar answer.

### 7.5 Finance risk and utility report

Input:

- Candidate plus evaluation helper data
- Optional external downstream results

Purpose:

- Evaluate risks and benefits not captured by generic distributions

Contents:

- Point-in-time leakage rate
- Survivorship or adverse-event coverage
- Critical-slice coverage
- Financial-tail coverage
- Economic-method agreement
- Downstream non-inferiority
- Prohibited-output exposure rate

## 8. Slice execution model

Average performance must not hide a critical failure.

Use an explicit slice registry. For each registered slice:

1. Filter candidate and applicable reference DataFrames.
2. Enforce a minimum sample-size policy.
3. Run the applicable report definition.
4. Store the slice metric values and Test results.
5. Apply the slice threshold or mark `INSUFFICIENT_EVIDENCE`.

Required slice types:

- Product or asset class
- Fund strategy or portfolio type
- Industry and issuer-size band
- Currency and jurisdiction
- Source/provider
- Market regime and period
- Active/adverse status
- High-value and tail cases
- Transformation-method path

Rules:

- `INSUFFICIENT_EVIDENCE` is not a pass for a critical slice.
- Slice definitions and minimum sizes must be registered before acceptance testing.
- Multiple-comparison effects should be considered when large numbers of statistical tests are used.
- Prefer effect-size or distance thresholds over p-values alone for large datasets.

## 9. Finance metric pack

Implement finance logic as Evidently custom dataset- or column-level Metrics with attached Tests where appropriate.

Each metric must expose:

- Stable metric ID and version
- Display name and business question
- Numerator, denominator and exclusions
- Scalar metric value used by Evidently
- Affected row keys in a separate evidence table
- Applicable slices
- Default visualization or simple counter
- Test condition, severity and owner

### 9.1 Core custom metrics

| Metric ID | Calculation | Interpretation |
|---|---|---|
| `FIN-COV-001 target_universe_coverage` | represented applicable entities / applicable target entities | Population coverage |
| `FIN-COV-002 critical_slice_coverage` | represented applicable critical entities/events / target critical entities/events | Adverse/tail coverage |
| `FIN-TIME-001 point_in_time_leakage_rate` | outputs using data available after decision time / tested outputs | Temporal fitness |
| `FIN-TAIL-001 tail_representation_score` | registered similarity function over tail quantiles and event rates | Tail preservation |
| `FIN-METHOD-001 economic_method_agreement` | acceptable-method cases / adjudicated cases | Method suitability |
| `FIN-DIFF-001 material_output_disagreement` | materially different conclusions / aligned gold cases | Output agreement |
| `FIN-SLICE-001 slice_distribution_distance` | registered distance within financial peer group | Slice representativeness |
| `FIN-UTIL-001 downstream_noninferiority` | downstream candidate result − reference result using registered direction and margin | Fitness for use |
| `FIN-STAB-001 rank_stability` | registered rank correlation or top-k overlap | Decision stability |
| `FIN-RISK-001 prohibited_output_rate` | outputs containing prohibited information or classifications / tested outputs | Output-control fitness |

### 9.2 Domain extensions

Fund metrics:

- Liquidated-fund coverage
- Survivorship-bias impact
- Backfill-period coverage
- Benchmark suitability agreement
- Net/gross fee-basis fitness
- Return-tail preservation

Accounting metrics:

- Point-in-time filing/restatement use
- Default and bankruptcy coverage
- GAAP/IFRS peer comparability
- Sector-specific methodology agreement
- Negative/near-zero denominator impact
- Restatement impact distribution

Portfolio metrics:

- Derivative and short-position coverage
- Market-value versus risk-exposure distinction rate
- Gross/net exposure distribution fitness
- Concentration-rank stability
- Look-through impact
- Stress-period and illiquidity coverage

## 10. Drift-method selection

Choose methods per column; do not apply one global default.

| Data/evaluation need | Starting method | Additional control |
|---|---|---|
| Continuous return/exposure distributions | Wasserstein or registered KS/effect-size combination | Quantile and tail comparison |
| Stable categorical composition | Jensen–Shannon, PSI or chi-square as appropriate | Minimum category counts |
| Large datasets | Effect size or normalized distance | Avoid p-value-only gates |
| Small samples | Exact/robust method selected during calibration | Mark insufficient evidence |
| Tail-sensitive fields | Tail quantile and exceedance-rate metric | Dedicated hard floor |
| Ordered/ranking outputs | Rank correlation and top-k overlap | Material decision-change rate |
| Relationship preservation | Correlation or dependency-change metric | Peer-group evaluation |

Calibration determines the final method and threshold. Evidently defaults are useful for exploration but are not automatically finance acceptance policy.

## 11. Evidently Tests and gates

Attach explicit Tests to metrics after threshold calibration.

Illustrative pattern:

```python
from evidently import Report
from evidently.tests import eq, gte, lte

report = Report([
    PointInTimeLeakageRate(tests=[eq(0)]),
    CriticalSliceCoverage(tests=[gte(configured_minimum)]),
    MaterialOutputDisagreement(
        share_tests=[lte(configured_maximum)]
    ),
])
```

Test classes:

| Class | Behavior |
|---|---|
| Hard gate | Any failure makes the evaluation `FAIL` |
| Dimension floor | Aggregated dimension must meet its minimum |
| Review trigger | Requires human review but may allow conditional release |
| Diagnostic | Reported for interpretation and monitoring only |

Default hard gates:

- Point-in-time leakage rate equals zero observed
- Critical economic-method cases pass
- Required critical slices meet minimum coverage
- No prohibited output exposure
- No unexplained material release change beyond tolerance
- No critical downstream degradation

## 12. Scoring and decision policy

Recommended dimensions:

| Dimension | Weight |
|---|---:|
| Distribution preservation | 20% |
| Population coverage | 20% |
| Financial semantic quality | 25% |
| Representativeness and tails | 15% |
| Temporal integrity | 10% |
| Downstream usefulness | 10% |

Decision order:

1. Evidently execution completeness
2. Hard-gate Tests
3. Critical-slice Tests
4. Dimension floors
5. Weighted overall score
6. Required human approval

States:

| State | Rule |
|---|---|
| `NOT_EVALUABLE` | A required reference, population definition or evaluation evidence is unavailable |
| `ERROR` | Evidently or framework execution failed; no quality conclusion |
| `FAIL` | Any hard gate, critical slice, dimension floor or overall threshold fails |
| `CONDITIONAL_PASS` | Only bounded noncritical exceptions with owner, control and expiry |
| `PASS` | All required Tests, floors, score and approvals pass |

Rules:

- Hard gates remain separate from the weighted score.
- Missing required metrics and empty critical slices fail closed.
- Skipped and not-applicable metrics are distinct.
- Thresholds are frozen before hidden acceptance testing.
- Score calculation reads versioned Evidently results; it must not silently recompute finance metrics differently.

## 13. Threshold calibration

Do not start with arbitrary production thresholds.

Calibration set:

- Several previously approved outputs
- Known-bad transformation outputs
- Synthetic perturbations with known severity
- Normal and stressed periods
- Critical slices and tail cases
- At least one release-change corpus

Procedure:

1. Run exploratory Evidently reports without acceptance Tests.
2. Review distributions and metric behavior with finance experts.
3. Select methods and thresholds that separate acceptable and unacceptable outputs.
4. Quantify false positives and false negatives.
5. Add confidence intervals to sampled rates.
6. Freeze the threshold registry.
7. Run a hidden acceptance set.

Statistical rules:

- Use an upper confidence bound for maximum-error thresholds.
- Use a lower confidence bound for minimum-success thresholds.
- Zero observed defects does not prove zero risk.
- Treat a too-small critical slice as insufficient evidence, not success.
- Document regime-sensitive thresholds and when each applies.

## 14. Package design

```text
finance_evaluation/
  contracts/
    run_manifest.py
    evaluation_spec.py
  adapters/
    dataframe_adapter.py
    gold_alignment.py
    slice_builder.py
  registry/
    reference_registry.py
    metric_registry.py
    threshold_registry.py
  reports/
    candidate_profile.py
    population_fitness.py
    release_regression.py
    gold_agreement.py
    finance_risk_utility.py
  metrics/
    coverage.py
    temporal.py
    tails.py
    methodology.py
    disagreement.py
    stability.py
    downstream.py
    prohibited_output.py
  decision/
    gates.py
    scoring.py
    policy.py
  artifacts/
    evidence.py
    publisher.py
  runner.py
```

Module rules:

- Report modules compose Evidently Metrics and Tests; they do not calculate policy independently.
- Metric modules contain calculation logic and version information.
- Decision modules consume Evidently results and registered policy.
- Adapters are responsible for key alignment and helper columns.
- Registries resolve immutable versions and never infer “latest” during a release decision.

## 15. Run orchestration

```python
def evaluate_transformed_output(request):
    manifest = manifests.freeze(request)
    spec = evaluation_registry.resolve_exact(manifest.spec_version)
    refs = reference_registry.resolve_exact(manifest.references)

    datasets = adapter.build_evidently_datasets(
        candidate=request.candidate_df,
        references=refs,
        data_definition=spec.data_definition,
    )

    results = {
        "profile": reports.candidate_profile(datasets.candidate, spec),
        "population": reports.population_fitness(
            datasets.candidate, datasets.population, spec
        ),
        "release": reports.release_regression(
            datasets.release_candidate, datasets.release_reference, spec
        ),
        "gold": reports.gold_agreement(datasets.gold_aligned, spec),
        "finance": reports.finance_risk_utility(datasets, spec),
        "slices": reports.run_registered_slices(datasets, spec),
    }

    final_decision = decision.apply(results, spec.policy)
    return artifacts.publish(manifest, results, final_decision)
```

## 16. Artifact model

```text
artifacts/<evaluation_run_id>/
  manifest.yaml
  spec_reference.json
  reference_receipts.json
  results/
    candidate_profile.json
    population_fitness.json
    release_regression.json
    gold_agreement.json
    finance_risk_utility.json
    slices.json
  reports/
    candidate_profile.html
    population_fitness.html
    release_regression.html
    gold_agreement.html
    finance_risk_utility.html
  evidence/
    failed_cases.parquet
    slice_failures.parquet
    material_changes.parquet
  decision.json
  checksums.txt
```

Every result must be traceable to:

- Candidate data hash
- Agent and transformation versions
- Evaluation specification
- Reference versions
- Metric versions
- Threshold versions
- Evidently version
- Decision policy and approver

## 17. Testing the evaluation framework

### Metric unit tests

- Known numerator and denominator
- Empty and all-excluded datasets
- Zero or near-zero denominators
- Missing evidence columns
- Negative values and extreme tails
- Slice filtering
- Deterministic repeatability

### Report contract tests

- Expected metric IDs appear
- Required Tests execute
- Candidate/reference definitions match
- JSON result schema remains compatible
- HTML generation succeeds

### Decision-policy tests

- Hard gate cannot be averaged away
- Missing required metric fails closed
- Critical slice failure blocks release
- Conditional pass requires approved exception
- Score and dimension floors use the registered versions

### End-to-end golden tests

- Known-good output passes
- Known-bad financial methodology fails
- Point-in-time leakage fails
- Underrepresented critical population fails
- Legitimate market-regime drift is reported without automatic rejection when policy permits it
- Unexpected release behavior triggers review or failure

## 18. Implementation sequence

### Milestone 1 — Foundation

- Evaluation manifest
- Explicit DataDefinition
- Reference and evaluation registries
- Candidate profile and population fitness reports

### Milestone 2 — Finance evaluation

- Gold alignment
- Core finance metric pack
- Registered slice execution
- Evidence tables

### Milestone 3 — Decisioning

- Evidently Tests
- Hard gates
- Dimension scoring
- Release decision artifact

### Milestone 4 — Operationalization

- Automated runner
- HTML/JSON publication
- Production monitoring
- Reference refresh and change management

## 19. Definition of done

- [ ] Missing required references or evaluation evidence produces `NOT_EVALUABLE` and no score.
- [ ] Candidate and references use explicit, versioned `DataDefinition` objects.
- [ ] Gold, population and release references are independently registered.
- [ ] Five report families run reproducibly.
- [ ] Core finance metrics have unit and golden tests.
- [ ] Every critical slice has a minimum-evidence and acceptance rule.
- [ ] Finance thresholds are calibrated, versioned and frozen before acceptance.
- [ ] Evidently hard-gate Tests cannot be offset by the total score.
- [ ] JSON and HTML reports plus failed-row evidence are retained.
- [ ] A reviewer can reconstruct the decision from immutable artifacts.

## 20. Evidently implementation references

- Use explicit [`Dataset` and `DataDefinition`](https://docs.evidentlyai.com/docs/library/data_definition) objects for controlled finance column typing and roles.
- Compose evaluation families with Evidently [`Report`](https://docs.evidentlyai.com/docs/library/report), Presets and individual Metrics.
- Use [`DataDriftPreset`](https://docs.evidentlyai.com/metrics/preset_data_drift) selectively, with registered methods and thresholds for material columns.
- Implement financial logic through Evidently [`custom dataset- and column-level Metrics`](https://docs.evidentlyai.com/metrics/customize_metric).
- Attach explicit pass/fail conditions using Evidently [`Tests`](https://docs.evidentlyai.com/docs/library/tests) after calibration.
