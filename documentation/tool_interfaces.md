# Tool interfaces

The three tools accept one string argument and return one string. The English examples in [`prompt_examples.md`](prompt_examples.md) are readable representations of the requests; the implementation uses the field markers described below. In screening, the complete return from TOOL 1 is passed to TOOL 2. TOOL 2 returns its report or fallback output directly.

## TOOL 1 — Manufacturer screening

**Entrypoint:** `calculate_topsis_data_json`

**Input.** The request contains task information, optional preference information, and manufacturer records. Manufacturer identifiers are written as `Manufacturer A:`, `Manufacturer B:`, and so forth; records are separated by semicolons. Required manufacturer information includes distance, qualified rate, capacity interval, and fulfillment rate. An optional performance score is distinct from fulfillment rate.

**Defaults and weight handling.** If omitted, demand defaults to 1,000 units, unit price to CNY 100, delivery time to seven days, and response time to 24 hours. If a performance score is not supplied, the fulfillment-rate value is used for that manufacturer. The initial criterion weights are:

| Criterion | Initial weight |
| --- | ---: |
| Distance | 0.20 |
| Response time | 0.15 |
| Qualified rate | 0.20 |
| Capacity match | 0.20 |
| Fulfillment rate | 0.15 |
| Performance score | 0.10 |

A recognized qualitative preference adds 0.25 to the corresponding initial weight. Recognized explicit weights replace the corresponding initial values. The resulting weights are normalized by their sum.

**Successful return.** The return is a JSON string with the following top-level entries:

- `task_details`: task quantity, price, deadline, and the normalized weights.
- `selection_summary`: selection status, candidate count, selected identifiers, selected details, and aggregate selected capacities.
- `full_ranking`: the ordered candidate ranking with TOPSIS relative closeness and the input indicators.
- `contingency_context`: unselected-candidate information and the distance reference used for targeted report checks.

The selected-detail fields use `fulfillment_rate` for the fulfillment rate and `performance_score` for the performance score. The latter is not a second copy of the fulfillment-rate field. `is_selected` indicates whether a mutual-aid set was formed; when no set is formed, the selected list and selected details are empty while the complete ranking remains available.

**Diagnostic return.** Missing manufacturer records, missing required manufacturer fields, and recognized invalid explicit weights produce diagnostic text instead of screening JSON. The routing instructions require the agent to report a TOOL 1 error and stop without invoking TOOL 2. Exceptions outside the implemented handlers terminate the current invocation.

## TOOL 2 — Decision report generation

**Entrypoint:** `generate_decision_analysis`

**Input.** TOOL 2 receives the complete TOOL 1 JSON string; no separate preference argument is supplied. The required top-level entries are `task_details`, `selection_summary`, `full_ranking`, and `contingency_context`. The tool checks object/array types, required fields, finite nonnegative numeric values, a positive demand, and consistency between selection status, selected identifiers, and selected details.

**Successful return.** The LLM-generated English report contains four sections in this order:

1. `Execution Recommendation`
2. `Capacity Risk Analysis`
3. `Key Supplier Evaluation`
4. `Alternative Strategy`

The implementation checks section order and nonempty content, overall length, the stated selected set, manufacturer coverage, and recognized nearest-manufacturer claims against the numerical distance reference. These are structural and targeted checks, not comprehensive semantic verification.

**Failure handling.** JSON parsing failures, missing required fields, or failed input checks return diagnostic messages. If the first report fails the report checks, the diagnostic is added to the prompt and the report is regenerated once. A second failed check, a model-call or response-extraction exception, or unusable response text produces a failure notice beginning with `LLM report unavailable`, followed by the numerical screening summary. TOOL 2 returns this result directly without subsequent rewriting by the agent.

## TOOL 3 — Proposal coordination

**Entrypoint:** `Proposal_Coordinator`

**Input.** The request contains total demand and, for each participating manufacturer, capacity bounds, proposed quantity, and concession coefficient. At least one manufacturer record is required. Manufacturer identifiers and record separators follow the TOOL 1 conventions. Proposal and concession coefficient are required within each record.

**Defaults and conversion.** Missing demand defaults to 1,000 units. Capacity bounds default to `[0, 100000]` when absent or not successfully parsed. Proposed quantities are converted to integers.

**Successful return.** The return is a text coordination report containing the demand, initial proposal total and signed gap, each manufacturer’s initial proposal, concession coefficient and capacity bounds, each revised quantity, and the final total. It is not a JSON allocation object.

**Diagnostic return.** Missing manufacturer records or required proposal/concession fields produce diagnostic text. Caught execution exceptions also return diagnostic text. Uncaught exceptions terminate the current invocation.
