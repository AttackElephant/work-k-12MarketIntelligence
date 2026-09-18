Aider v0.86.2
Model: openrouter/z-ai/glm-5.3-flash with whole edit format
Git repo: .git with 171 files
Repo-map: using 4096 tokens, auto refresh
Added harness/planner.py to the chat.
Added run.py to the chat.
Added tests/unit/test_intent_boundary.py to the chat.
Added tests/unit/test_planner.py to the chat.
Added tests/unit/test_run_cli.py to the chat.
Added contract/question_schemas/v0.3.0/VC-POC-03.request.schema.json to the chat
(read-only).
Added contract/poc_school_directory.v0.3.0.schema.json to the chat (read-only).
Added contract/question_schemas/v0.3.0/VC-POC-01.request.schema.json to the chat
(read-only).
Added contract/poc_input_catalog.v0.3.0.schema.json to the chat (read-only).
Added ../work/k-12MarketIntelligence/work/20260908T113118/planning/002-plan.md 
to the chat (read-only).
Added contract/poc_evidence_error.v0.3.0.schema.json to the chat (read-only).
Added prompts/rules.md to the chat (read-only).
Added contract/question_schemas/v0.3.0/_index.json to the chat (read-only).
Added 
../work/k-12MarketIntelligence/work/20260908T113118/acceptance/003-contract.yaml
to the chat (read-only).
Added contract/POC_EVIDENCE_CONTRACT.md to the chat (read-only).
Added contract/PINNED.json to the chat (read-only).
Added contract/poc_question_registry.v0.3.0.json to the chat (read-only).
Added contract/question_schemas/v0.3.0/VC-POC-04.request.schema.json to the chat
(read-only).
Added docs/decisions.md to the chat (read-only).
Added contract/POC_CONTRACT_V0.3_SPEC.md to the chat (read-only).
Added evals/cases/golden_cases.json to the chat (read-only).
Added prompts/input_catalog.snapshot.json to the chat (read-only).

Initial repo scan can be slow in larger repos, but only happens once.

--------------
► **THINKING**

Let me carefully analyze the task.

The work item is to make `harness/planner.py` deterministically:
1. Derive required inputs from the pinned question policy (not trusting 
`StructuredIntent.missing_material_inputs`)
2. Resolve schools safely (transactional, exactly-one match per mention, staged 
then committed)
3. Normalize requested metrics (Unicode-safe case folding, separator 
normalization, pinned aliases, dedup preserving order)
4. Construct exact evidence requests (smallest schema-shaped request, routed 
selector fields — `ratios` for VC-POC-03, `metrics` for VC-POC-01/04/11)
5. Validate requests against pinned per-question schemas before dispatch
6. Surface planner-generated bad requests as `TerminalError` (not 
`Unsupported`); separate error origins
7. Make `run.py` default CLI use the pipeline path (`converse(None, deps, ...)`)
8. Expand tests.

But wait — there's a critical constraint in the contract:

```
contract_requirements:
  - id: manifest-allowlist-conflict
    requirement: "The amended plan requires run.py and 
tests/unit/test_run_cli.py, but manifest scope.declared_files lists only 
harness/planner.py, tests/unit/test_intent_boundary.py, and 
tests/unit/test_planner.py. Execution must not begin until the human reviewer 
expands the manifest diff allowlist to include the two additional required 
outputs."
```

Hmm, but the TASK says declared editable files include `run.py` and 
`tests/unit/test_run_cli.py`:
```
Declared editable files:
- harness/planner.py
- run.py
- tests/unit/test_intent_boundary.py
- tests/unit/test_planner.py
- tests/unit/test_run_cli.py
```

So the human reviewer (the orchestrator) has expanded the allowlist — the 
declared editable files in this request include run.py and 
tests/unit/test_run_cli.py. So that blocker is resolved; I can proceed with all 
five files.

Now let me carefully plan the implementation.

### Requirements from the plan (002-plan.md):

**harness/planner.py:**
1. Derive missing required, material, and applicable conditional-material inputs
from `input_policy` and the assembled request, ignoring contradictory 
`StructuredIntent.missing_material_inputs`. Evaluate `distance_km` only when 
`market_definition == "distance"`. Do not change the legacy output gate or its 
strict O-8 xfail.
2. Treat school selector fields as satisfiable by explicit IDs or successful 
named-school resolution. Resolve each mention through the directory, require 
exactly one result, stage IDs locally in mention order, deduplicate without 
reordering, commit to `Deps` only after all mentions succeed. Record every 
preflight.
3. Return `ClarificationNeeded` for zero/multiple school matches, insufficient 
distinct schools, absent user-selectable inputs, invalid caller-supplied 
identity reported by the emitter, and requested selectors that cannot be 
normalized. Use stable policy order, contract field names, and populated static 
allowed values where the pinned catalog mapping supplies them. No evidence call 
on pre-dispatch clarification paths.
4. Raise `TerminalError` before dispatch for conflicts between explicit IDs and 
name-resolved identities, unknown fields introduced into the assembled request, 
and any other planner-produced request that cannot satisfy its registered 
schema. Never translate these defects to `Unsupported`.
5. Normalize selector labels by Unicode-safe case folding and separator 
normalization, preserve canonical values, apply only pinned semantic aliases 
(including `teaching staff` to `teaching_fte`), deduplicate preserving order. 
Apply the existing raw-question headcount/non-teaching scope rule before 
validation.
6. Derive selector routing from pinned artefacts: registry `metric_list` where 
present and the input-catalog schema's field-to-list mapping for 
`VC-POC-03.ratios`, grades, and other static selectors. Submit `metrics` for 
VC-POC-01/04/11 and `ratios` for VC-POC-03; omit selector fields when the user 
named none. Read static vocabularies from vendored pinned files; no runtime 
input-catalog dependency.
7. Construct a fresh request from explicit intent fields and planner-owned 
selectors only. Validate against the question's registered request schema before
`fetch_evidence` (reject unknown top-level/nested props, wrong types, bounds, 
uniqueness/cardinality, conditionals, formats). Enforce VC-POC-09/10 
explicit-filter constraint: absent, empty, or sector-default-only peer/segment 
definitions clarify; unknown filter keys are planner validation failures.
8. Record a failed evidence invocation before branching. If schema-valid 
deterministic request receives `bad_request`, distinguish the explicitly 
passed-through numeric-ID case and return identity clarification; otherwise 
raise `TerminalError` with question ID and emitter context. Preserve existing 
branches for excluded, gated, unknown, unanswerable, data-unavailable, internal 
outcomes.

**run.py:**
- Make default CLI call `converse(None, deps, ...)`. Do not construct the legacy
agent for the default path. Preserve credentials, emitter selection, rendering, 
exit codes, clarification limits.
- Retain explicit legacy-agent support in `converse()`.

**Tests:**
- `tests/unit/test_planner.py`: exact typed assertions for unique, zero, 
ambiguous, duplicate, conflicting school identities; transactional Deps updates;
derived missing inputs; conditional distance; static allowed values; 
normalization/routing; schema validation; VC-POC-09/10 explicit and 
unknown-filter constraints; exact request dictionaries; separated error origins.
- `tests/unit/test_intent_boundary.py`: planner clarification bypasses 
answer-view/narration, normalized schema-valid requests reach evidence, 
planner/schema defects propagate as TerminalError, caller-correctable 
metric/identity problems remain clarification rather than Unsupported.
- `tests/unit/test_run_cli.py`: default CLI does not build or pass a legacy 
agent, reaches pipeline signature. Keep existing legacy-loop tests unchanged.

The acceptance contract names specific tests that MUST exist and pass (so they 
must be added to test_planner.py, test_intent_boundary.py, test_run_cli.py):

```
tests/unit/test_planner.py::test_missing_market_definition_is_derived_without_mo
del_hint
tests/unit/test_planner.py::test_distance_is_conditional_on_distance_market
tests/unit/test_planner.py::test_ambiguous_school_clarifies_without_fetch
tests/unit/test_planner.py::test_metric_normalization_and_exact_profile_request
tests/unit/test_planner.py::test_vc_poc_03_routes_selector_to_ratios
tests/unit/test_planner.py::test_unknown_request_property_fails_before_fetch
tests/unit/test_planner.py::test_peer_filters_require_explicit_nondefault_filter
tests/unit/test_planner.py::test_segment_filters_reject_unknown_nested_key
tests/unit/test_planner.py::test_schema_valid_emitter_bad_request_is_terminal
tests/unit/test_planner.py::test_invalid_explicit_identity_bad_request_clarifies
tests/unit/test_intent_boundary.py::test_planner_failure_is_not_unsupported
tests/unit/test_run_cli.py::test_main_uses_pipeline_path_by_default
```

Also failure_behaviour in contract:
- Zero/multiple directory matches → ClarificationNeeded, record preflight, no 
evidence call, no partial resolved-ID mutation.
- Explicit IDs conflict with name-resolved identities → TerminalError before 
dispatch, no partial identity update.
- Dedup leaves <2 identities for VC-POC-04 → ClarificationNeeded, no dispatch.
- Required or applicable material user-selectable field absent → 
ClarificationNeeded in stable pinned-policy order (required, material, then 
applicable conditional-material), contract field names and static allowed values
where available, no dispatch.
- Unnormalizable static selector → ClarificationNeeded with applicable static 
vocabulary, no evidence call.
- Planner-built request violates schema / unknown fields / cannot load pinned 
schema → TerminalError before dispatch, not Unsupported.
- VC-POC-09/10 lacks explicit nondefault cohort-defining filter → 
ClarificationNeeded, no dispatch.
- Locally validated request with only caller-supplied numeric ACARA identity 
receives bad_request → record failed fetch invocation, return identity 
clarification.
- Any other locally validated deterministic request receives bad_request → 
record failed fetch and raise TerminalError with question ID and emitter 
context.
- Emitter unknown_question/excluded_question/unanswerable → preserve typed 
abstention without retry.
- Emitter data_unavailable/internal → TerminalError.

Compatibility:
- Excluded-topic, registry-owned exclusion, gated-question, unknown-question, 
unanswerable outcomes retain their established typed outcomes and contract-owned
text.
- Successful complete/partial/empty envelopes remain successful, stored in 
deps.last_envelope.
- Well-typed release-derived/open values that match no facts remain successful 
empty/partial.
- Every directory preflight and evidence invocation, including failed, 
observable in deps.tool_calls.
- converse() continues to accept explicit legacy agent.
- CLI credential selection, emitter selection, rendering, clarification limit, 
exit codes unchanged.

Semantic criteria:
- no-model-missing-input-trust: VC-POC-05 without market_definition clarifies 
even when model reports no missing input; irrelevant model-reported missing 
fields are ignored.
- conditional-materiality: SA2 market doesn't ask for distance_km; distance 
market without distance_km does.
- transactional-identity: each school contributes ID only after exactly one 
match; mention order preserved, duplicates removed, later failed match leaks no 
partial state or request.
- exact-profile-normalization: VC-POC-01, "Total Enrolments" and "teaching 
staff" → total_enrolments, teaching_fte, without adding headcount or 
non-teaching metrics.
- exact-ratio-routing: VC-POC-03 staffing-intensity selectors use ratios 
vocabulary and ratios request field, no metrics property.
- exact-comparison-request: VC-POC-04 dispatch has ordered unique IDs, 
explicitly requested canonical metrics, only other explicitly supplied schema 
fields; emitter-owned year defaults omitted.
- schema-and-filter-fail-closed.
- bad-request-origin.
- production-routing: run.py main does not call build_agent for default 
execution and passes None to converse.

Also `id: static-vocabulary-source`: static selector routing uses registry 
metric_list and input-catalog schema's question-field mapping and embedded 
enums; no runtime emitter calls.

`id: safe-schema-resolution`: Only registered request-schema paths beneath the 
pinned contract directory may be loaded; load or reference-resolution failures 
are TerminalError.

`id: known-condition-only`: Only the exact pinned distance_km condition for 
VC-POC-05/06/07 is interpreted; unknown conditional expression raises 
TerminalError (contract drift).

`id: no-invented-defaults`: Unsupplied defaultable values omitted so emitter 
remains owner of defaults.

`id: policy-order`: Clarification fields stable across runs, follow pinned 
required, material, then applicable conditional-material order.

`id: invocation-recording`: record every completed directory preflight and every
evidence attempt, including failed, before returning or raising.

`id: preflight-boundary`: directory record establishes identity only.

`id: direct-id-authority-conflict`: Do not add directory preflight for 
caller-supplied numeric IDs; a bad_request for a request containing only a 
caller-supplied numeric ACARA identity → identity clarification.

Also must_not_change:
- Legacy PydanticAI agent-tool repair loop (harness/tools.py untouched).
- Evidence-contract files, PINNED.json, registry, schemas, snapshot.
- Evidence client protocol, emitter, database.
- Answer-view, narration, rendering, grounding, confidence, limitations, claim 
eligibility, contract-owned refusal wording.
- Prohibition on School_No → ACARA_SML_ID crosswalks.
- contract.material_inputs_for() and unchanged legacy O-8 strict xfail.

OK. Now, some important details:

### How does the planner know static vocabularies?

The plan says: "Static normalization uses vendored pinned authorities: the 
registry `metric_list` and the input-catalog schema's question-field mapping and
static enums, with the captured snapshot only as a consistency baseline. It does
not call the runtime emitter for contract-static vocabularies."

So the planner should read:
- `contract/poc_question_registry.v0.3.0.json` — for `metric_lists` (already 
loaded via `harness.contract.registry()` presumably — let me check what 
`harness.contract` provides: `registry()`, `questions()`, 
`excluded_questions()`, `input_policy_for()`, `exclusion_texts()`, 
`refusal_for()`, `reason_code_for()`, `material_inputs_for()`. The registry is 
loaded via `registry()`, presumably from 
`contract/poc_question_registry.v0.3.0.json`.
- `contract/poc_input_catalog.v0.3.0.schema.json` — for 
`x-poc-request-field-to-catalog-list` mapping and the embedded static enums (in 
properties like `profile_metrics.items.enum`, `grades.items.enum`, 
`ratios.items.enum`, etc.).
- `contract/question_schemas/v0.3.0/_index.json` — for per-question schema 
paths.

The plan says: "Use the registry's `metric_list` where declared and the 
input-catalog schema's `x-poc-request-field-to-catalog-list` plus embedded 
static enums for question/request-field routing, including VC-POC-03 `ratios` 
and VC-POC-11 `metrics` backed by `student_context_metrics`."

So:
- For each question, the selector routing: if `metric_list` in registry entry 
(e.g. `profile_metrics` for VC-POC-01/04, `student_context_metrics` for 
VC-POC-11), the selector field is `metrics` and the vocabulary is the registry's
metric list. For VC-POC-03, the input-catalog schema's 
`x-poc-request-field-to-catalog-list` maps `VC-POC-03.ratios` → `ratios` (the 
catalog list name "ratios"), so the field is `ratios` and the vocabulary is the 
`ratios` enum in the input-catalog schema properties.

Actually the mapping has entries like:
- "VC-POC-01.metrics": "profile_metrics"
- "VC-POC-04.metrics": "profile_metrics"
- "VC-POC-11.metrics": "student_context_metrics"
- "VC-POC-02.grades": "grades"
- "VC-POC-07.grades": "grades"
- "VC-POC-03.ratios": "ratios"
- "VC-POC-05.market_definition": "market_definitions"
- "VC-POC-06.market_definition": "market_definitions"
- "VC-POC-07.market_definition": "market_definitions"
- "VC-POC-09.peer_filters.<keys>": "peer_filter_keys"
- ... and release-derived ones for specific nested keys
- "VC-POC-10.segment_filters.<keys>": "segment_filter_keys"
- ... plus specific segment filter keys
- "VC-POC-12.seifa_index_type": "seifa_index_types"
- "VC-POC-12.seifa_measure": "seifa_measures"
- "VC-POC-12.erp_age": "release_derived_values.erp_age"
- "VC-POC-13.age_bands": "release_derived_values.age_bands"

The `<keys>` entry means the set of permitted nested keys for peer_filters / 
segment_filters.

So normalization approach:
1. Build a per-question selector config from pinned artefacts:
   - From the input-catalog schema's `x-poc-request-field-to-catalog-list`: for 
entries matching `f"{question_id}.{field}"` (exact) → field with a static enum 
list name. For `f"{question_id}.peer_filters.<keys>"` / 
`f"{question_id}.segment_filters.<keys>"` → nested key allow-list.
2. Vocabulary list for a given (question, field): prefer registry `metric_list` 
when field is `metrics` (declared for VC-POC-01/04 as profile_metrics, VC-POC-11
as student_context_metrics); otherwise use the input-catalog schema's embedded 
enum for that catalog list name.

Let me now check how `harness/contract.py` loads the registry — I can't see it 
(read-only file not in chat, actually harness/contract.py wasn't even supplied).
Hmm. The files I have editable: harness/planner.py, run.py, 
tests/unit/test_intent_boundary.py, tests/unit/test_planner.py, 
tests/unit/test_run_cli.py.

The contract module provides `registry()` returning the registry dict 
presumably, `questions()` mapping question_id → entry, 
`input_policy_for(question_id)`, etc.

I need to load the input-catalog schema and the `_index.json`. Since 
`harness/contract.py` is not editable and may not provide these, I need to load 
them in planner.py directly from the pinned contract directory. The plan says: 
"resolve only registered paths beneath the pinned contract directory; 
schema-loading or reference-resolution failure is operational TerminalError."

So in planner.py:
- Find the contract directory: `Path(__file__).resolve().parents[1] / 
"contract"`.
- Load `contract/poc_input_catalog.v0.3.0.schema.json` for the field-to-list 
mapping and static enums.
- Load `contract/question_schemas/v0.3.0/_index.json` for the schema file names.
- Validate requests against per-question schemas located at 
`contract/question_schemas/v0.3.0/<QID>.request.schema.json`.

"Only registered request-schema paths beneath the pinned contract directory may 
be loaded" — so I must validate the resolved schema path: it must be 
`question_schemas/v0.3.0/<name>.request.schema.json` where `<name>` matches the 
question ID from `_index.json` (registered path). I'll derive the path from 
`_index.json`'s `request_schema` field and check the resolved path is under the 
pinned dir and the filename matches the registered name exactly.

"Use a validator supporting JSON Schema draft 2020-12" — I need to check what's 
available. The environment... I can't run commands. The repo has `.venv`. Does 
it have `jsonschema`? Hmm. The plan says "Use a validator supporting JSON Schema
draft 2020-12 and resolve only registered paths beneath the pinned contract 
directory". If `jsonschema` is installed (it's a common dependency), use 
`jsonschema.Draft202012Validator`. But I cannot verify. Risky.

Alternative: write a minimal deterministic validator inside planner.py that 
checks the aspects we care about: additionalProperties, required, types, bounds,
minItems/maxItems, uniqueItems, enums, patterns, minimum/maximum, const. Since 
all request schemas are self-contained (no $ref — looking at VC-POC-01/03/04 
request schemas, they have no $ref references; they're flat), a small 
hand-rolled validator covering the used keywords is feasible and 
dependency-free.

Hmm, but the plan explicitly says "Use a validator supporting JSON Schema draft 
2020-12". That could mean jsonschema library. But if it's not installed, tests 
fail. Let me think about what's safest.

The schemas use these keywords (looking at VC-POC-01/03/04): `$schema`, `$id`, 
`title`, `type`, `additionalProperties`, `properties`, `items`, `enum`, 
`minItems`, `maxItems`, `uniqueItems`, `minimum`, `maximum`, `description`, 
`required`, `x-poc`, `pattern` (in error schema), `const` (error schema), 
`allOf`/`if`/`then` (error schema). Request schemas likely also use `pattern` 
(VC-POC-12 seifa? no...), `type: number`, conditional requirements maybe for 
VC-POC-05-07 (`distance_km` conditional via allOf/if/then? Let me think — the 
spec says "conditional requirements" are in the schemas. The VC-POC-05 request 
schema probably has an `allOf` with `if market_definition == distance then 
required distance_km`).

Since I can't see those schemas (only 01, 03, 04 provided), I need a validator 
that supports: type, required, properties, additionalProperties, items, enum, 
const, minItems, maxItems, uniqueItems, minimum, maximum, pattern, allOf, anyOf,
oneOf, if/then/else, not, multipleOf maybe. That's a fair chunk but manageable. 
The nested filters in VC-POC-09/10 use `properties` with `additionalProperties: 
false` presumably.

Actually, let me try to use `jsonschema` if available and fall back to... no — a
fallback would be non-deterministic across environments. Determinism means 
identical inputs → identical outputs; using jsonschema when available and 
hand-rolled otherwise would still be deterministic per environment but the tests
would encode whichever the environment has.

Hmm. Let me consider: pydantic-ai depends on... not jsonschema. Griffe? No. 
Actually, `jsonschema` is a dependency of... possibly not present. The stub 
emitter doesn't validate. `tests/unit/test_contract_shapes.py` exists 
("_response(question_id)") — what does it do? It might validate response 
fixtures against schemas using jsonschema! If so, jsonschema is a test 
dependency and available in .venv.

I can't read that file (not supplied). Risky either way.

Decision: implement a small, dependency-free Draft-2020-12-subset validator 
inside planner.py, covering the keywords used by the pinned request schemas: 
type (including list types), required, properties, additionalProperties:false, 
items (schema), enum, const, minItems, maxItems, uniqueItems, minimum, maximum, 
pattern, minLength, maxLength, and the combinators allOf/anyOf/oneOf with 
if/then/else, not. This satisfies "validate against the question's registered 
request schema" and "supporting JSON Schema draft 2020-12" for the subset the 
pinned schemas use. It's fully deterministic and dependency-free. The plan's 
wording "Use a validator supporting JSON Schema draft 2020-12" — a hand-rolled 
one that supports the constructs actually pinned arguably qualifies; the safer 
engineering choice given I cannot verify dependency availability is the 
self-contained validator. Hmm.

Wait — actually, maybe I should try importing jsonschema with a fallback to the 
internal validator? That's messy. Let me think about what the tests need to 
prove: "Unknown top-level or nested filter keys, wrong types, cardinality 
failures, and conditional failures reach the emitter" must not happen — the 
planner must catch them. My internal validator catches all of these for the 
provided schemas. For schemas I can't see (VC-POC-02, 05-14, REF), the generic 
validator needs to handle what's in them. Since all schemas are generated by 
`scripts/build_poc_v0_3_schemas.py` from the same generator, they likely use the
same keyword set. I'll implement a reasonably complete subset.

Actually — one important consideration: JSON Schema "if/then" for conditionals. 
VC-POC-05 schema likely has:

```json
"allOf": [
  {
    "if": {"properties": {"market_definition": {"const": "distance"}}, 
"required": ["market_definition"]},
    "then": {"required": ["distance_km"]}
  }
]
```

My validator must support `if`/`then` inside `allOf`. OK.

Also `minProperties` for VC-POC-09/10 (OD-07: "the request schemas express only 
minProperties >= 1"). So need `minProperties`.

Let me write the validator carefully:

```python
def _validate_against_schema(instance, schema, path="$"):
    errors = []
    ... returns list of error strings
```

Support:
- `type`: string in 
{"object","array","string","integer","number","boolean","null"} (note: integer 
accepts bool? No — in JSON Schema, bool is not integer; but Python 
`isinstance(True, int)` is True. Need to exclude bool for integer/number.)
- `enum`: instance in enum (careful with bool/int equality: `True == 1`; for 
enum checks, compare with type awareness? The enums here are strings, so simple 
`in` is fine; I'll do a type-aware check for correctness: match if any enum item
equals instance and isinstance types compatible. Simple approach: `instance in 
enum` — fine for strings.)
- `const`: equality.
- `required`: keys present.
- `properties`: recurse when key present.
- `additionalProperties: False`: any key not in `properties` → error.
- `additionalProperties: {schema}`: recurse for other keys. (Probably not used 
but cheap.)
- `items`: recurse for array elements.
- `minItems`/`maxItems`/`uniqueItems` (compare by canonical JSON repr; careful: 
dict/list unhashable → use json.dumps with sort_keys).
- `minimum`/`maximum`/`exclusiveMinimum`/`exclusiveMaximum`/`multipleOf`.
- `pattern`: `re.search`.
- `minLength`/`maxLength`.
- `minProperties`/`maxProperties`.
- `allOf`/`anyOf`/`oneOf`/`not`.
- `if`/`then`/`else`.

For oneOf: exactly one subschema validates. For if: if `if` validates, `then` 
must validate (if present), else `else` must validate (if present).

Numbers: `minimum` etc. compare only when instance is int/float (not bool).

Integer type check: `isinstance(v, int) and not isinstance(v, bool)`. Number: 
int or float, not bool.

String type: `isinstance(v, str)`. Boolean: `isinstance(v, bool)`. Null: `v is 
None`. Object: `isinstance(v, dict)`. Array: `isinstance(v, list)`.

Note: JSON Schema says `1.0` is both integer and number. For "integer", accept 
int (and float with is_integer? Draft 2020-12: integer matches any number with 
zero fractional part. I'll accept float with `v.is_integer()` too, to be safe.)

OK.

### Schema loading

```python
CONTRACT_DIR = Path(__file__).resolve().parents[1] / "contract"

@lru_cache(maxsize=1)
def _schema_index() -> dict[str, Any]:
    path = CONTRACT_DIR / "question_schemas" / "v0.3.0" / "_index.json"
    if not path.is_file(): raise TerminalError(...)
    data = json.loads(path.read_text("utf-8"))
    return data
```

Then for question QID: `entry = _schema_index()["questions"][QID]`, `schema_file
= entry["request_schema"]` (e.g. "VC-POC-01.request.schema.json"). Registered 
path = `question_schemas/v0.3.0` / schema_file. Validate: the resolved path is 
under CONTRACT_DIR and its name matches `f"{QID}.request.schema.json"`? Hmm — 
careful: registry question ids include "VC-POC-REF-01", so filename 
"VC-POC-REF-01.request.schema.json". The check "only registered paths beneath 
the pinned contract directory may be loaded": derive path from _index.json (a 
pinned file) and verify the final path is inside the pinned contract directory. 
That's sufficient. Maybe also assert the filename equals 
`{question_id}.request.schema.json` to prevent any trickery. I'll do the 
containment check plus require the schema file name from the index (registered),
not from user input. Since question_id comes from intent, and I look up 
`_schema_index()["questions"][question_id]`, an unknown question_id → KeyError →
but we never get here for unknown questions (planner returns Unsupported for 
unknown earlier). Still, guard: if question_id not in index → TerminalError 
(planner drift).

Loading failure (file missing, JSON invalid) → TerminalError per "schema-loading
or reference-resolution failure is operational TerminalError".

### Static vocabularies

Load `contract/poc_input_catalog.v0.3.0.schema.json`:

```python
@lru_cache(maxsize=1)
def _input_catalog_schema() -> dict:
    path = CONTRACT_DIR / "poc_input_catalog.v0.3.0.schema.json"
    ...read...
```

From it:
- `field_to_list = schema.get("x-poc-request-field-to-catalog-list", {})`
- `properties = schema["properties"]` — each property may have `items.enum` 
(static lists) — e.g. `profile_metrics.items.enum`.
- `release_derived_values.properties` names are release-derived — NOT 
enum-enforced.

Selector routing derivation for a question:

```python
def _selector_field_for(question_id) -> tuple[str, list[str] | None] | None
```

Hmm, let me define what selectors need normalization:
- `metrics` (VC-POC-01, 04, 11) — the request field name. The plan says "Submit 
`metrics` for VC-POC-01/04/11 and `ratios` for VC-POC-03".
- `ratios` (VC-POC-03).
- `grades` (VC-POC-02, 07).
- `market_definition` (VC-POC-05/06/07) — single scalar.
- Nested filter keys for VC-POC-09/10: validate keys against 
peer_filter_keys/segment_filter_keys (mapped to release-derived values for 
sectors etc. — shape-checked only).
- seifa_index_type, seifa_measure for VC-POC-12 — scalars.
- age_bands for VC-POC-13 — release-derived, shape only.

Now, what does the planner actually normalize? 
`StructuredIntent.requested_metrics` is the model-extracted selector labels. The
planner must:
1. Determine the selector field for the question: from the mapping 
`x-poc-request-field-to-catalog-list`, entries `f"{question_id}.{field}"` where 
field != nested filter markers. But multiple entries can exist per question 
(e.g. VC-POC-12 has seifa_index_type, seifa_measure, erp_age; VC-POC-13 has 
age_bands). The requested_metrics list is a generic bag of selector labels from 
the intent model. Which field do they belong to?

The pragmatic reading of the plan: "Convert requested metric labels to the 
question's canonical catalog vocabulary and request field". The primary selector
per question:
- For questions with a registry `metric_list`: field `metrics`, vocabulary = 
registry metric_lists[metric_list].
- For VC-POC-03: field `ratios`, vocabulary = catalog `ratios` enum.
- For VC-POC-02/07: field `grades`, vocabulary = catalog `grades` enum.
- VC-POC-05/06/07 market_definition: scalar — could normalize 
`requested_metrics` too? The filters dict carries market_definition already from
the intent model (`intent.filters` holds e.g. `{"market_definition": "distance",
"distance_km": 2.5}` per the intent tests). So scalars come through 
`intent.filters`.

The plan says normalization applies to "requested selectors". The key behaviors 
graded:
- exact-profile-normalization: VC-POC-01, "Total Enrolments" and "teaching 
staff" → total_enrolments + teaching_fte.
- exact-ratio-routing: VC-POC-03 ratios vocabulary and ratios field, no metrics 
property.
- exact-comparison-request: VC-POC-04 ordered unique IDs + explicitly requested 
canonical metrics.
- VC-POC-11 metrics from student_context_metrics.

So the core selector normalization targets the array-of-strings selector field: 
`metrics`/`ratios`/`grades`.

What about a user selector that "cannot be normalized" → ClarificationNeeded 
with the applicable static vocabulary. So for VC-POC-01, "banana" → 
ClarificationNeeded listing profile_metrics allowed values. For VC-POC-03, 
"students per teacher" can't normalize → clarify with ratios vocabulary.

Normalization algorithm:
1. Unicode-safe case folding: `label.casefold()` (Unicode case folding) — 
"Unicode-safe case folding and separator normalization".
2. Separator normalization: replace runs of non-alphanumeric characters (spaces,
hyphens, underscores?) with a single space? Careful: canonical values are 
snake_case like `total_enrolments`, `students_per_teaching_fte`, `year_6`, 
`pre_year_1_one_year`. To match, normalize both the label and the canonical 
value the same way: casefold, then replace separators (underscore, hyphen, 
whitespace, punctuation) with a single space, collapse, strip.

So `normalize("Total Enrolments")` → "total enrolments"; 
`normalize("total_enrolments")` → "total enrolments". Match!
`normalize("teaching staff")` → "teaching staff" → alias map has "teaching 
staff" → "teaching_fte" (the pinned alias from prompts/rules.md: "For 
school-profile requests, 'teaching staff' means `teaching_fte`"). So alias keys 
normalized too: "teaching staff" → "teaching_fte".

Aliases (pinned only): 
- "teaching staff" → "teaching_fte" (from prompts/rules.md — pinned, mentioned 
in plan: "apply only pinned semantic aliases (including `teaching staff` to 
`teaching_fte`)").
Maybe also "non teaching staff" → non_teaching_fte? The rules say "include 
non-teaching metrics only when the user explicitly asks about non-teaching 
staff" — that's the scope rule. The alias table: keep just the explicitly pinned
one(s). Plan: "including `teaching staff` to `teaching_fte`" — "including" 
suggests there may be more pinned aliases, but the only one I can find pinned is
teaching staff → teaching_fte. The headcount scope rule is separate 
(request_scope_problem). I'll define exactly one alias: "teaching staff" → 
"teaching_fte". 

Hmm wait — also consider "Total Enrolments" — that's direct normalization, not 
alias.

3. If normalized label matches a vocabulary value's normalized form → canonical 
value.
4. Else if normalized label in alias map and alias target in vocabulary → 
canonical.
5. Else → unnormalizable → ClarificationNeeded.

Deduplicate preserving order.

Scope rule: "Apply the existing raw-question headcount/non-teaching scope rule 
before validation." — existing `request_scope_problem(deps.user_question, 
question_id, request)` from harness.tools. Currently applied after construction;
the plan says apply before validation. Looking at the current planner code:

```python
scope_problem = request_scope_problem(deps.user_question, question_id, request)
if scope_problem and "metrics" in request and isinstance(request["metrics"], 
list):
    cleaned = [m for m in request["metrics"] if not 
(str(m).endswith("_headcount") or str(m).startswith("non_teaching_"))]
    request["metrics"] = cleaned or request["metrics"]
```

I need to check what `request_scope_problem` returns — it's in harness/tools.py 
(not editable, not fully shown). From the summary: `def 
request_scope_problem(user_question: str, question_id: str, request: dict[str, 
Any])`. It returns probably a string or None. In the current planner: `if 
scope_problem and ...`. I'll keep using it the same way: after normalization and
dedup, run the scope cleanup on the normalized metrics list, then validate. The 
exact semantic: drop `_headcount` and `non_teaching_*` metrics the user didn't 
ask for. Keep as in current code: if cleanup empties the list, keep original? 
Hmm — "cleaned or request["metrics"]" — if all requested metrics were 
headcount/non-teaching and scope problem flagged, we'd keep them. That preserves
the case where the user explicitly asked only for headcounts... but wait, the 
scope problem triggers when the user question doesn't mention 
headcount/non-teaching wording. If the model requested only headcount metrics 
but the user never mentioned them, those metrics are unrequested expansions — 
dropping them all would leave an empty metrics list. Keeping the original would 
defeat the purpose. But changing this behavior is risky; the existing behavior 
is `cleaned or original`. Actually per the plan: "Apply the existing 
raw-question headcount/non-teaching scope rule before validation." — "existing" 
means keep its semantics. I'll keep `cleaned or normalized_list` semantics.

Wait, but if the metrics list becomes empty after cleanup... `metrics: []` would
violate minItems: 1 in schema → my validator would fail → TerminalError. That's 
why the `or original` fallback exists. Keep it.

Order of operations: normalize → dedup → scope cleanup → validate.

### School resolution

Fields: for questions whose required input includes `acara_sml_id` or 
`acara_sml_ids`:
- If `intent.filters` already carries the explicit ID (`acara_sml_id` or 
`acara_sml_ids`) — caller-supplied — do NOT directory-preflight (per 
direct-id-authority-conflict).
- Else resolve `intent.school_mentions` via directory:
  - For each mention in order: `records = await 
client.school_directory(search=mention, limit=25)`.
  - Record the preflight ToolInvocation (including failures).
  - Exactly one record → stage its acara_sml_id. Zero or >1 → 
ClarificationNeeded naming the mention; no partial commit; no evidence call.
  - After all succeed: dedupe preserving order. If question requires ≥2 
(VC-POC-04's constraint "acara_sml_ids contains at least two values") and 
deduped < 2 → ClarificationNeeded.
  - Conflict between explicit IDs and name-resolved identities → TerminalError.

What counts as "conflict"? If intent.filters has explicit IDs AND there are 
school_mentions that resolve — if resolved IDs differ from explicit IDs → 
TerminalError. If they're the same set, fine? The plan says "conflicts between 
explicit IDs and name-resolved identities" → TerminalError. I'll implement: if 
both present, resolved identities must equal explicit identities (as sets, or 
lists); otherwise TerminalError. Hmm, actually simpler and stricter: if explicit
IDs supplied, mentions resolution is skipped (the explicit IDs are 
authoritative; named mentions resolved to different schools would be a 
conflict). Let me define: when both explicit IDs and mentions exist, resolve 
mentions; if any resolved ID not in explicit ID set → TerminalError. If all 
match → proceed with explicit IDs. That's a sane "conflict" definition.

Actually wait. Think about what "conflicts between explicit IDs and 
name-resolved identities" means: e.g. intent has filters {"acara_sml_id": 40387}
and mentions ["Beta Grammar"] resolving to 40392 — the planner can't satisfy 
both → TerminalError before dispatch.

If explicit IDs are present and mentions are present and resolve to the same 
schools → no conflict → use explicit IDs.

Commit to deps: `deps.resolved_school_ids` — stage locally (a plain list), 
commit by extending/setting deps.resolved_school_ids only after all mentions 
resolved. Also `deps.searched = True` — when? Currently set after each call. 
"Record every preflight" — each search recorded in deps.tool_calls regardless of
outcome. deps.searched: set when at least one preflight completed. I'll set 
`deps.searched = True` after committing (i.e., only after all mentions resolve 
successfully). Hmm, current behavior sets it after each call even on failure 
(actually on failure it re-raises before setting searched... let me re-read:

```python
try:
    records = await client.school_directory(search=school_name, limit=25)
except Exception as exc:
    deps.tool_calls.append(ToolInvocation("search_schools", {...}, 
error_kind=type(exc).__name__))
    raise
deps.searched = True
```

For transactional: commit searched=True along with IDs after all mentions 
succeed. On failure paths (ambiguity), no partial Deps mutation: "retain no 
partial resolved-ID mutation". Setting deps.searched would be a mutation... I'll
only commit `searched` when the whole resolution transaction succeeds. But 
"Record every preflight" — the tool_calls recording is the observability 
requirement, that's fine to do per call.

Note: ToolInvocation signature from tools.py: `ToolInvocation(tool, args, 
result=None, error_kind=None)` presumably — from current usage: 
`ToolInvocation("search_schools", {...}, records)` and 
`ToolInvocation("search_schools", {...}, error_kind=...)`. Keep that.

Directory record shape: from poc_school_directory schema, records have 
`acara_sml_id`, `school_name`, etc.

Exactly one match: `len(records) == 1` and record has acara_sml_id. If records 
has >1 → ambiguous → Clarification. If 0 → no match → Clarification.

Hmm wait — the stub emitter's school_directory: how does it behave? 
`run_school_directory(bundle, search, limit)` in stub_emitter.py — I don't know 
its matching semantics. Tests use mock clients (_HappyClient returns one record 
per search), so planner tests can control this. The intent-boundary tests use 
the real stub — for "Alpha College" it presumably returns exactly one record 
(golden case expectations rely on the stub returning 40387 for Alpha College). 
Good — with the new exactly-one rule, the stub must return exactly one record 
for the fixture names; presumably it does (the fixture has one Alpha College 
record). Risk: if the stub returns multiple records for a search term (e.g., 
fuzzy matching "college" returning many), the pipeline path would now clarify 
where previously it took the first. But the existing intent-boundary test 
`test_ask_pipeline_intent_override_bypasses_model` uses intent with 
filters={"acara_sml_id": 40387} — explicit ID, no mention — so no preflight 
needed. Good. And test_ask_pipeline_uses_scripted_intent_model etc. all use 
filters with explicit IDs. So intent-boundary happy paths don't hit directory 
resolution. 

But wait — how does the stub resolve "Alpha College"? Let me look at 
`run_school_directory(bundle, search, limit)` signature only. Can't see the 
body. Golden cases mention Alpha College → expected acara_sml_id 40387, so stub 
maps names to IDs somehow (probably fixture schools with exact substring 
matching). Fine.

### Deriving missing inputs (input derivation)

For each question, from `contract.input_policy_for(question_id)` (or 
`contract.questions()[qid]["input_policy"]`):
- `required`: list of field names.
- `material`: list.
- `conditional_material`: dict field → condition string.
- `defaultable`: dict.
- `optional`: list.

Assembled request = explicit intent filters + planner-owned selectors.

Derivation logic:
1. Missing required fields (not satisfiable another way): for each required 
field in order:
   - If field is a school selector (`acara_sml_id`/`acara_sml_ids`): satisfiable
by explicit IDs in intent.filters OR by school_mentions (resolution will fill 
it). So it's "satisfiable" if explicit IDs present or mentions present. If 
neither → it's missing → clarify.
   - If field present in intent.filters (or will be in request) → satisfied.
   - Else → missing → clarify.
2. Material fields: same check for fields in `material` (e.g. market_definition,
grades, peer_filters, segment_filters, sa2_codes).
3. Conditional material: `distance_km` when the condition string indicates 
`market_definition == distance` AND market_definition (from intent.filters) is 
"distance" (after normalization? filters values come from the intent model; 
normalize market_definition value too — market_definitions vocabulary: 
distance/lga/postcode/sa2. Normalize the filter value case-insensitively: 
"Distance" → "distance"? The plan says normalization applies to requested 
selectors; filters values are intent-extracted. To be safe, I'll normalize 
market_definition value through the market_definitions vocabulary too when 
present. Hmm — careful with determinism and scope. The plan says: "Normalize 
selector labels by Unicode-safe case folding and separator normalization..." — 
"selector labels" generally. market_definition IS a static_closed selector 
(catalog `market_definitions`). I'll normalize it if present; if unnormalizable 
→ ClarificationNeeded with market_definitions vocabulary. That fits "requested 
selectors that cannot be normalized" → clarification.

Hmm wait, but there's a subtlety: `grades` may come via intent.filters? In the 
current tests, grades come via requested_metrics? Let me check: golden cases 
show VC-POC-07 request has grades: ["year_6"]. The intent model extracts 
`requested_metrics` as the selector labels and `filters` as scalar fields 
(market_definition, distance_km, acara_sml_id). The current planner only copies 
requested_metrics → request["metrics"]. For VC-POC-02/07, grades would be in 
requested_metrics and should route to `grades`.

So routing: the selector field for the question determines where 
requested_metrics goes:
- VC-POC-01, 04: `metrics` (profile_metrics)
- VC-POC-11: `metrics` (student_context_metrics)
- VC-POC-03: `ratios`
- VC-POC-02, 07: `grades`
- others: no array selector (VC-POC-05/06 market_definition scalar; VC-POC-12: 
erp_age/seifa scalars — erp_age is release-derived; 
seifa_index_type/seifa_measure static scalars).

But grades for VC-POC-07 might also come via filters? To be safe: accept grades 
from requested_metrics (routed) AND honor filters if the intent model put them 
there? Hmm — the plan says "Construct a fresh request from explicit intent 
fields and planner-owned selectors only." Explicit intent fields = 
intent.filters values. If intent.filters contains grades, that's an explicit 
intent field — include it (it's schema-valid). If requested_metrics non-empty 
and question has an array selector field → normalize and route.

Let me define `_selector_route(question_id) -> tuple[field_name, 
vocabulary_list] | None`:

```python
def _selector_route(question_id: str):
    q = contract.questions().get(question_id) or {}
    metric_list = q.get("metric_list")
    if metric_list:
        return "metrics", list(contract.registry()["metric_lists"][metric_list])
    # static single-selector array fields from the catalog mapping
    mapping = _field_to_catalog_list()
    for key, list_name in mapping.items():
        if key.startswith(question_id + "."):
            field = key.split(".", 1)[1]
            if field in {"ratios", "grades"}:
                return field, _static_enum(list_name)
    return None
```

Wait — mapping keys: "VC-POC-01.metrics": "profile_metrics" etc. For questions 
with metric_list, both the registry and mapping agree on `metrics`. The plan 
says "Use the registry's `metric_list` where declared and the input-catalog 
schema's ... mapping ... including VC-POC-03 `ratios` and VC-POC-11 `metrics` 
backed by `student_context_metrics`". Hmm — VC-POC-11 HAS metric_list = 
"student_context_metrics" in the registry. So metric_list covers VC-POC-01 
(profile), 04 (profile), 11 (student_context). VC-POC-03 has metric_list: null, 
so routing falls to the catalog mapping → "VC-POC-03.ratios": "ratios". 
VC-POC-02/07 grades → mapping "VC-POC-02.grades": "grades", "VC-POC-07.grades": 
"grades".

But careful: iterating the mapping for a question like VC-POC-01 would find 
"VC-POC-01.metrics" → but metric_list already handles it. For VC-POC-05: mapping
has "VC-POC-05.market_definition": "market_definitions" — that's a scalar, not 
the array selector bag. I restrict the fallback to array-of-label fields 
{"ratios", "grades"}. Good.

Static enum extraction: `_static_enum(list_name)`:
- If list_name is a top-level property of the catalog schema with `items.enum` →
return it.
- If list_name starts with "release_derived_values." → release-derived → 
shape-check only (return None vocabulary).
- If list_name == "reported_enrolment_composition" → static enum (nested filter 
values, only relevant inside filters — skip for selector bag).

Now — should VC-POC-02/07 grades also accept grade labels? Yes: "What has Year 6
looked like" → requested_metrics=["year 6"]? or filters? The intent schema 
(StructuredIntent) has requested_metrics as the bag. Golden case 
"answer/VC-POC-02" expects grades: ["year_6"]. The pipeline must route that. 
With route ("grades", grades enum), "Year 6" normalizes → "year 6" matches 
"year_6" → canonical "year_6". 

But hold on — the legacy replay path (tools.py) handles these via the model tool
loop, and golden cases are graded against the legacy path in replay tests... The
pipeline path is now the CLI default, but eval replay uses build_agent (legacy).
Fine — no changes to legacy.

However — the golden case "answer/VC-POC-02" grade "year_6" — the stub case 
matching: stub `select_case` matches question + request. Since pipeline tests I 
write will use mocks/stub carefully, fine.

Also VC-POC-11: requested_metrics like "indigenous enrolments" → 
indigenous_enrolments. OK.

Normalization detail — `pre_year_1_one_year`: "Pre Year 1 (one year)" → 
normalized "pre year 1 one year" — separators stripped of parens: my separator 
regex `[^\w]+` → space; but "pre_year_1_one_year" normalized: "pre year 1 one 
year". Match. Good — but underscores inside values must normalize the same way. 
`re.sub(r"[^0-9a-z]+", " ", s)` after casefold handles it. Careful with 
non-ASCII casefold (e.g. "ÏCSEA"): casefold then ASCII-normalize? "Unicode-safe 
case folding and separator normalization" — I'll casefold, then replace any char
that's not alphanumeric (Unicode-aware `\w` minus underscore) with space, 
collapse. To match canonical values (ASCII), also normalize Unicode via... keep 
it simple: casefold + `re.sub(r"[^0-9a-z\u00c0-\uffff]"...` hmm. Simpler: 
casefold, then replace underscore/hyphen/whitespace/punctuation with space. Use 
`re.sub(r"[\s_\-]+", " ", s)` plus strip other punctuation like 
parentheses/slashes: `re.sub(r"[^\w\s]+", " ", s)` (Unicode \w). Then collapse 
whitespace. This is Unicode-safe (doesn't strip non-ASCII letters).

So:

```python
def _normalize_label(label: str) -> str:
    s = str(label).casefold()
    s = re.sub(r"[^\w]+", " ", s, flags=re.UNICODE)
    s = re.sub(r"\s+", " ", s).strip()
    return s
```

Wait `\w` includes underscore, so "total_enrolments" stays "total_enrolments" — 
but "total enrolments" (space) → "total enrolments". These wouldn't match! I 
need underscore treated as separator too. Use explicit class: 
`re.sub(r"[^0-9a-z\u00df-\uffff]+", " ", s)`? Getting complicated. Alternative: 
replace `[\W_]+` with " ": `\W` is non-word (includes space, punctuation), plus 
underscore. `re.sub(r"[\W_]+", " ", s, flags=re.UNICODE)` — replaces underscore,
spaces, punctuation with single space, preserves Unicode word chars 
(letters/digits, incl. accented). Then collapse (the + already collapses). 
strip. 

"total_enrolments" → "total enrolments"; "Total Enrolments" → "total 
enrolments". "students_per_teaching_fte" → "students per teaching fte". 
"pre_year_1_one_year" → "pre year 1 one year". "(distance)" → "distance". 

Alias map: `{"teaching staff": "teaching_fte"}`.

Now the market_definition scalar: normalize value through market_definitions 
vocabulary: "distance" etc. `filters.get("market_definition")` → if string, 
normalize → must match a vocab value, else ClarificationNeeded with 
market_definitions allowed values. If it matches, replace in request with 
canonical.

Hmm — but what about peer_filters/segment_filters values (sectors: 
["Independent"] — release-derived, shape only)? And `erp_age: "TOT"` 
(release-derived)? Shape-check only — pass through.

Nested filter keys for VC-POC-09/10: must be within 
peer_filter_keys/segment_filter_keys (static closed). Unknown nested key → 
planner validation failure → TerminalError ("unknown filter keys are planner 
validation failures"). The schema's additionalProperties:false would also catch 
it at validation. Order: constraint enforcement (explicit filter presence) 
happens before schema validation? The plan: "Enforce VC-POC-09/10's 
registry-owned explicit-filter constraint before dispatch: absent, empty, or 
sector-default-only peer/segment definitions clarify; unknown filter keys are 
planner validation failures." And "Validate it against the question's registered
request schema before fetch_evidence... The registry/emitter-owned VC-POC-09/10 
explicit-filter constraint is an additional deterministic pre-dispatch rule".

Order within planner: build request → enforce explicit-filter constraint 
(clarify if absent/empty/default-only) → validate schema (TerminalError on 
failure). Unknown nested keys: schema validation catches them 
(additionalProperties false) → TerminalError. The contract failure_behaviour 
says "VC-POC-09 or VC-POC-10 lacks an explicit nondefault cohort-defining filter
→ ClarificationNeeded"; "A planner-built request violates its registered schema,
contains an unknown top-level or nested field → TerminalError". The acceptance 
test names: `test_peer_filters_require_explicit_nondefault_filter`, 
`test_segment_filters_reject_unknown_nested_key`. For the latter: 
segment_filters with an unknown key → TerminalError (schema validation catches 
it; also the key set check). But careful — should unknown nested key in an 
otherwise-absent-filter scenario clarify or fail? If segment_filters is absent →
clarify. If present but contains unknown key → TerminalError (schema breach). 
I'll implement: constraint check first (absent/empty/default-only → Clarify), 
then schema validation (unknown keys → TerminalError).

Explicit-filter constraint definition (from registry constraints):
- VC-POC-09: "peer_filters contains at least one explicit filter beyond the 
sector default". Defaultable `peer_filters.sectors: ["Catholic", 
"Independent"]`. So: peer_filters absent or empty → clarify. If peer_filters has
only `sectors` and its value equals the default ["Catholic", "Independent"] (or 
a subset thereof?) — "sector-default-only definitions clarify". The pinned 
default is ["Catholic", "Independent"]. "sector-default-only" = the only key is 
sectors AND value equals the default. If sectors is something else (e.g. 
["Government"]) — is that "beyond the sector default"? The default is the sector
filter itself; the constraint says "at least one explicit filter beyond the 
sector default". Hmm — reading the registry: defaultable peer_filters.sectors = 
["Catholic", "Independent"]. Constraint: at least one explicit filter beyond the
sector default. So a sectors-only filter, even non-default values, would be 
"only the sector filter" — is that beyond the default? The default IS a sector 
filter; providing sectors=["Government"] is still sector-only. Strict reading: 
need at least one filter key other than sectors. I'll implement: non-sectors 
keys present → OK. Only sectors key → clarify (default-only or otherwise? plan 
says "absent, empty, or sector-default-only peer/segment definitions clarify"). 
"sector-default-only" — sectors-only-with-default-values clarifies. What about 
sectors-only with non-default values, e.g. sectors: ["Government"]? Per plan 
wording, only "sector-default-only" clarifies; a non-default sectors-only 
definition is an explicit filter → dispatch (emitter enforces its own rule). 
Hmm, but the semantic criterion says "VC-POC-09/10 missing or default-only 
cohort definitions clarify". So: clarify iff absent, empty dict, or 
(sectors-only AND sectors == default). I'll go with that precise reading.

For VC-POC-10: constraint "segment_filters contains at least one explicit 
filter". No default filters. Clarify iff absent or empty.

- Derive these constraints from the registry's input_policy.constraints strings?
"Only the exact pinned distance_km condition... an unknown conditional 
expression is contract drift" applies to conditional_material. For constraints, 
the registry strings are: "peer_filters contains at least one explicit filter 
beyond the sector default" and "segment_filters contains at least one explicit 
filter". The plan says "Enforce that exact registry constraint in planner code 
and tests without generalising it into a new condition language." So hardcode 
the two known constraint implementations keyed to the registry's constraint 
presence for those two questions. I'll implement: for VC-POC-09/10 (detected by 
question id? or by required field name + constraints text?) — safest: implement 
keyed on the question's required field being "peer_filters"/"segment_filters" 
AND the registry constraints containing the known string; if a question has a 
required peer/segment filter with an unrecognized constraint string → that's 
fine (only distance condition is drift-checked). Let me key on field name: if 
"peer_filters" in required → apply peer constraint; if "segment_filters" in 
required → apply segment constraint. And for VC-POC-04's constraint 
"acara_sml_ids contains at least two values" → enforce min-2 distinct IDs 
(clarify if fewer after dedup) — the contract failure behaviour: "Deduplication 
leaves fewer than two identities for VC-POC-04 → ClarificationNeeded without 
evidence dispatch." Also schema minItems:2 would catch it, but we must clarify 
not TerminalError. So check BEFORE schema validation: if question requires 
acara_sml_ids and request's acara_sml_ids has <2 distinct → Clarify (if 
populated from resolution) — but what if explicit caller supplied only 1 ID for 
VC-POC-04? Then request violates schema minItems → TerminalError? Hmm — that's 
caller-supplied data that can't satisfy the schema. "any other planner-produced 
request that cannot satisfy its registered schema" raises TerminalError. A 
request with one explicit ID for VC-POC-04 can never validate → TerminalError. 
But wait — is that a caller-correctable problem (clarify) or planner defect? The
schema requires ≥2; a caller giving one ID is a user error → arguably clarify. 
But the plan says "insufficient distinct schools" → ClarificationNeeded. So: if 
acara_sml_ids count < 2 → ClarificationNeeded (naming the requirement), 
regardless of source. I'll treat VC-POC-04 under-2 as clarification (before 
schema validation), covering both cases. Contract: "Deduplication leaves fewer 
than two identities for VC-POC-04 → ClarificationNeeded". I'll apply to any 
source.

Hmm, but careful — schema validation of VC-POC-04 with 1 ID would TerminalError;
by checking first and clarifying, we never reach validation with that defect. 
Good.

### Missing-input derivation & clarification ordering

Policy order: required, material, then applicable conditional-material. 
(Contract: "Clarification fields are stable across runs and follow pinned 
required, material, then applicable conditional-material order.")

Build the missing list:

```python
def _derive_missing(question_id, request, mentions_present) -> list[str]:
    policy = input_policy_for(question_id)
    missing = []
    # required
    for field in policy.get("required", []):
        if _field_satisfied(field, request, mentions_present): continue
        missing.append(field)
    # material
    for field in policy.get("material", []):
        ...same...
    # conditional
    for field, condition in policy.get("conditional_material", {}).items():
        if field == "distance_km" and condition indicates distance:
            if (request.get("market_definition") normalized == "distance") and 
"distance_km" not in request:
                missing.append(field)
        elif condition: TerminalError (contract drift)
    return missing
```

_field_satisfied: for school selectors — satisfied if explicit IDs in request or
mentions present (to be resolved). For others — field in request (with 
meaningful value). Careful: "market_definition" in request? request starts from 
intent.filters. Hmm — what about empty values? e.g. 
filters={"market_definition": None}? Keep simple: presence check; plus 
peer/segment filters handled by their constraint (which runs separately and 
clarifies).

Wait — there's an interaction: VC-POC-09 required "peer_filters", material 
["peer_filters"]. If absent → missing derivation flags peer_filters 
(required+material) → ClarificationNeeded listing peer_filters. Good — same 
outcome as the constraint. If present but sectors-default-only → constraint 
clarifies. But then should the clarification say "peer_filters" missing? The 
constraint clarification names the reason: needs an explicit non-default filter.
Different message. OK.

For sa2_codes (VC-POC-12/13 material): absent → clarify (from derivation). 
Present → shape-check only (array of strings — schema validation will 
type-check).

Hmm wait, VC-POC-12's sa2_codes is in required AND material is []. VC-POC-13: 
required sa2_codes, material [sa2_codes]. Both covered.

zone_entity_code (VC-POC-REF-02): required, scalar string. Absent → clarify. 
(Though intent would rarely select REF-02 without it.)

Now `missing_material_inputs` from the intent model is IGNORED entirely for 
derivation ("ignoring contradictory StructuredIntent.missing_material_inputs" — 
"Derive missing required, material, and applicable conditional-material inputs 
from input_policy and the assembled request, ignoring contradictory 
StructuredIntent.missing_material_inputs"). So the planner no longer trusts the 
model's claim at all: derivation from policy. If the model claims missing 
"market_definition" for VC-POC-05 and it IS absent → derived → clarify. If the 
model claims missing "banana" → irrelevant → ignored (not in policy) → proceed. 
The semantic: "VC-POC-05 without market_definition clarifies even when the model
reports no missing input, and irrelevant model-reported missing fields are 
ignored." So ignore intent.missing_material_inputs completely — derivation is 
authoritative. 

Note the existing test 
`test_ask_pipeline_missing_material_input_returns_clarification` 
(intent_boundary) — intent VC-POC-05 with 
missing_material_inputs=["market_definition"], no filters → with derivation, 
still clarifies. Good. And `test_non_material_missing_input_is_ignored` 
(planner): VC-POC-01 with missing_material_inputs=["fiscal_year"], filters has 
acara_sml_id → proceeds. Good. 
`test_missing_material_input_returns_clarification` (planner): VC-POC-05, no 
filters, missing claim market_definition → clarify via derivation. Good.

`test_ask_pipeline_intent_override_excluded_topic_returns_unsupported`, etc. 
unchanged.

Existing planner test `test_happy_path_returns_envelope`: VC-POC-01 with 
filters={"acara_sml_id": 40387} — explicit ID, no metrics requested. Derivation:
required acara_sml_id present; metrics defaultable → omitted (no invented 
defaults). Request = {"acara_sml_id": 40387}. Valid schema. Fetch. OK with 
_HappyClient.

`test_school_resolution_records_search`: VC-POC-01, school_mentions=["Some 
School"], _HappyClient returns one record with acara_sml_id 40387 → resolve → 
request acara_sml_id=40387. deps.searched True, 40387 in 
deps.resolved_school_ids, one search_schools tool call. Good.

Now — schema validation against pinned VC-POC-01 schema with _HappyClient... the
fetch happens with a mock client, but validation is against the real pinned 
schema. Request {"acara_sml_id": 40387} validates. Good.

But careful — existing planner tests use `_ErrorClient` with VC-POC-01 filters 
{"acara_sml_id": 40387}: validation passes → fetch raises EvidenceError → 
branch. `test_bad_request_returns_unsupported`: EvidenceError("bad_request") → 
per NEW semantics: this is a locally validated deterministic request with a 
caller-supplied numeric identity only → identity clarification (not 
Unsupported!). The contract: "A locally validated request containing only a 
caller-supplied numeric ACARA identity receives bad_request → Record the failed 
fetch invocation and return identity clarification."

So `test_bad_request_returns_unsupported` must be REPLACED — the acceptance list
has `test_invalid_explicit_identity_bad_request_clarifies` and 
`test_schema_valid_emitter_bad_request_is_terminal`. The old test asserting 
Unsupported(out_of_scope) for bad_request contradicts the new origin-aware 
behavior. I must update the planner tests (they're editable; the plan explicitly
says "Existing planner tests explicitly encode unsafe first-match resolution and
this obsolete mapping" — to be updated).

Also `_branch_evidence_error` must return `Unsupported | ClarificationNeeded` 
now.

Let me now define bad_request origin handling:

```python
if kind == "bad_request":
    if _request_identity_is_caller_numeric(request):  # only caller-supplied 
numeric ACARA identity
        return ClarificationNeeded(identity clarification)
    raise TerminalError(f"VC-POC-01: evidence emitter rejected a validated 
request: {exc.message}...")
```

What counts as "containing only a caller-supplied numeric ACARA identity"? The 
request's school IDs came from explicit caller/intent filters (not name 
resolution). I'll track during planning whether the IDs in the final request 
originated from intent.filters (caller-supplied) vs directory resolution. Simple
flag: `caller_supplied_identity = bool(explicit_ids_present)` where explicit ids
came from intent.filters and no mention-based resolution contributed... Actually
more precisely: "a locally validated request containing only a caller-supplied 
numeric ACARA identity" — the bad_request is plausibly about identity when the 
request has an explicit caller numeric ID. If the request contains 
directory-resolved IDs, the directory validated them, so bad_request is drift. 
If request has explicit caller ID (99999 case) — bad_request plausibly means "ID
absent from pinned directory" → harness/caller can correct → clarification.

Definition: if the final request includes any ACARA ID field whose values came 
from intent.filters (caller-supplied), a bad_request → identity clarification. 
If all IDs were directory-resolved (or none present), bad_request → 
TerminalError.

Hmm, "containing ONLY a caller-supplied numeric ACARA identity" — e.g. request 
{"acara_sml_id": 99999}. What if request is {"acara_sml_id": 99999, 
"market_definition": "sa2"}? Still contains a caller-supplied identity → clarify
(the emitter's message may point at identity or something else; the contract's 
bounded rule: explicit-ID-alone bad_request → identity clarification). The plan:
"distinguish the explicitly passed-through numeric-ID case and return identity 
clarification". So: if caller-supplied numeric ID present in request → clarify. 
Else → TerminalError.

Track: `identity_from_caller = any id field present in intent.filters`. Where 
"id field" = "acara_sml_id" or "acara_sml_ids".

Clarification content: question_id, missing=[MissingInput(input="acara_sml_id", 
why_material="The data layer rejected the supplied ACARA identity...", 
allowed_values=[])], prompt like "The ACARA ID supplied for this question was 
rejected — confirm the school identity." Hmm, MissingInput shape: input, 
why_material, allowed_values. Fine.

The golden empty-success case (answer/VC-POC-01-empty, ID 99999) runs through 
the LEGACY path in replay/evals (build_agent) — untouched. In the pipeline path,
ID 99999 + stub emitter: stub matches the fixture case and returns empty 
success, no bad_request. Fine.

### VC-POC-09/10 defaultable sectors handling

"no-invented-defaults": planner does NOT add the sector default — the emitter 
owns it. So request omits peer_filters.sectors if user didn't specify. 
Constraint check: sectors-default-only definition = peer_filters whose ONLY key 
is sectors with value == ["Catholic", "Independent"] (exact list equality? 
order? I'll compare as sets to be safe — or sorted lists equal). I'll compare 
sorted lists for stability.

But hold on — if peer_filters contains sectors=["Catholic","Independent"] AND 
school_types=["Secondary"] → explicit non-sector filter → dispatch. Good.

### Schema validation errors → TerminalError

"Raise TerminalError before dispatch for ... unknown fields introduced into the 
assembled request, and any other planner-produced request that cannot satisfy 
its registered schema."

But wait — intent.filters could contain an unknown field like {"banana": 1} 
(model hallucinated). Then the assembled request has an unknown property → 
schema validation fails → TerminalError. Is that right? The plan: "rejecting 
unknown top-level and nested properties, wrong types..." and "Raise 
TerminalError ... for ... unknown fields introduced into the assembled request".
Hmm — a hallucinated intent filter becomes "an unknown field introduced into the
assembled request" → TerminalError. The acceptance test 
`test_unknown_request_property_fails_before_fetch` → TerminalError. OK — strict 
fail-closed: yes, TerminalError.

Also "explicit intent fields" — should I filter intent.filters to schema-known 
fields before assembly? No — fail closed: if the intent carries a field not in 
the schema, that's a defect to surface. Actually hmm, could intent.filters carry
harness-internal fields that shouldn't go to the request? 
StructuredIntent.filters in tests carries {"acara_sml_id": 40387}, 
{"market_definition": "distance", "distance_km": 2.5} — all schema fields. I'll 
pass intent.filters through as "explicit intent fields" and let validation catch
unknown ones.

### What about `distance_km` from filters

VC-POC-08: required acara_sml_id + distance_km (material). filters 
{"acara_sml_id": 40387, "distance_km": 10} → request complete → validate → 
fetch. VC-POC-08 with no distance_km → derived missing → clarify. Matches golden
clarify/distance_km.

### metric scope rule timing

"Apply the existing raw-question headcount/non-teaching scope rule before 
validation." So: normalize → dedup → scope-clean (using request_scope_problem 
with deps.user_question and the BUILT request) → validate.

Current signature usage: `request_scope_problem(deps.user_question, question_id,
request)` where request already contains "metrics". I'll call it after building 
the full request dict (with normalized metrics). If it returns a problem 
(truthy) → clean metrics list. Then validate.

One more consideration — for VC-POC-03, request field is ratios — scope rule 
applies to "metrics" only (the existing rule checks question_id and metrics). 
`request_scope_problem` — what does it do for VC-POC-03 with ratios? Unknown; it
probably returns a problem string for headcount-related questions. I'll only 
apply cleanup when the routed selector field is "metrics" and the request value 
is a list (like current code). Keep the existing logic shape.

### Clarification for unnormalizable selector

Return ClarificationNeeded with the applicable static vocabulary in 
allowed_values, no evidence call. MissingInput input=<field>, 
why_material="...could not be matched to an allowed value: '<label>'", 
allowed_values=<vocabulary>.

### Deps mutation discipline

- deps.resolved_school_ids committed only after all mentions resolve 
(transactional).
- deps.searched=True committed with IDs.
- deps.intent is set by ask() presumably (stage orchestration) — actually ask 
stores deps.intent = intent (test asserts `deps.intent is intent`). That happens
in agent.ask, not planner. Fine.
- Failed fetch recorded before branching/raising.
- Failed preflight recorded before raising/clarifying.

### Envelope success path

Unchanged: record ToolInvocation, set deps.last_envelope, return envelope.

### What about `request` in arguments for recording

`arguments = {"question_id": ..., "request": request}` — record the final 
validated request.

### run.py changes

- `main()`: `agent = None` (don't call build_agent); `converse(None, deps, 
...)`.
- converse signature keeps `agent` first param; accepts None.
- Keep import of build_agent? "Do not construct the legacy agent for this 
default path." Keep the import for converse compatibility? test_run_cli has 
`test_main_uses_pipeline_path_by_default` to write: assert main doesn't call 
build_agent. How to test? Monkeypatch cli.build_agent with a sentinel-raising 
function, monkeypatch converse to capture args, run cli.main(["--stub", "q"])...
but main calls require_credentials() and build_client(). With --stub, 
build_client works offline. require_credentials: settings.MODEL_PROVIDER default
is "claude_account" (ADR-019) — require_credentials checks claude_account_status
— monkeypatch cli.claude_account_status to return (True, "ok"). Then monkeypatch
cli.converse with an async fake capturing the agent arg; make it return a 
RenderedAnswer? That's heavy. Simpler: fake converse returns 
ClarificationNeeded? Then main renders and returns exit code. Or return 
Unsupported. Let me construct a minimal RenderedAnswer — its required fields: 
from outputs.py, RenderedAnswer fields unknown in detail (answer, question_id, 
contract_version, evidence, provenance, confidence, limitations, 
data_status_note, deterministic_signals...). Constructing is risky. Use 
Unsupported(reason_kind=..., reason=...) — simpler, fields known from render 
test usage. Then main returns EXIT_UNSUPPORTED; assert fake converse was called 
with agent=None.

Test:

```python
def test_main_uses_pipeline_path_by_default(monkeypatch):
    from config import settings
    calls = {}
    async def fake_converse(agent, deps, question, **kwargs):
        calls["agent"] = agent
        return Unsupported(reason_kind="not_in_registry", reason="x")
    monkeypatch.setattr(cli, "converse", fake_converse)
    monkeypatch.setattr(cli, "require_credentials", lambda: None)
    monkeypatch.setattr(cli, "build_client", lambda use_stub: stub_client())
    monkeypatch.setattr(cli, "build_agent", lambda *a, **k: (_ for _ in 
()).throw(AssertionError("build_agent must not be called")))
    code = cli.main(["--stub", "q"])
    assert code == cli.EXIT_UNSUPPORTED
    assert calls["agent"] is None
```

monkeypatching cli.converse — main references `converse(...)` as a module 
global; monkeypatch.setattr(cli, "converse", fake_converse) works since main 
calls the global name. And cli.build_agent similarly (main calls `build_agent()`
global). Since the default main should never call build_agent, the throwing stub
proves it. 

Also keep legacy converse tests unchanged (they pass an explicit agent — 
converse still supports it).

Also update run.py main:

```python
def main(argv=None):
    configure_logging()
    args = build_parser().parse_args(argv)
    require_credentials()
    client = build_client(args.stub)
    deps = Deps(client=client)
    ask_user = None if args.no_interactive else _prompt_on_terminal
    try:
        output = asyncio.run(
            converse(
                None,
                deps,
                args.question,
                ask_user=ask_user,
                max_clarifications=args.max_clarifications,
            )
        )
    ...
```

Remove `agent = build_agent()` and the import? Keep `build_agent` import since 
converse tests... no wait, run.py's converse doesn't use build_agent. 
test_run_cli imports build_agent from harness.agent directly. run.py imports 
`ask, build_agent` — after the change, run.py doesn't need build_agent. Should I
remove the import? If left, it's unused (lint noise) but harmless. The plan: "Do
not construct the legacy agent for this default path." Removing the import is 
cleaner and matches "does not call build_agent". But `answered()` in 
test_run_cli uses `build_agent` imported from harness.agent in the test file — 
fine. I'll remove build_agent from run.py imports.

But careful — the docstring comment about converse: update to describe pipeline 
default.

Also in converse docstring: agent may be None → pipeline path.

### TerminalError import in run.py?

Planner raises TerminalError (a HarnessError subclass) — main catches 
HarnessError → EXIT_FAILED. Good, no run.py change needed for that.

### Clarification bypass of narration

"planner clarification bypasses answer-view/narration" — existing behavior: 
plan_and_execute returns ClarificationNeeded; ask() returns it without 
narration. Already the case 
(test_ask_pipeline_missing_material_input_returns_clarification exists). Add 
explicit test: planner clarification has no narration call & deps.answer_view 
stays None. In test_intent_boundary: `test_planner_failure_is_not_unsupported` —
needed for acceptance. What should it prove? "planner/schema defects propagate 
as TerminalError" — through ask(). Write: intent with an unknown request 
property (e.g. filters={"banana": 1}) for VC-POC-01 with explicit ID → ask() 
raises TerminalError (propagates), NOT Unsupported. With 
pytest.raises(TerminalError) around asyncio.run(ask(...)) with intent_override 
and a narration_override that would fail if called (narration_override shouldn't
be reached... hmm, narration override is only used on success). Use narration 
model? If planner raises, narration never runs. Pass a narration_model that 
raises AssertionError if invoked? Actually if planner raises TerminalError, 
ask() propagates it — narration not reached. I'll pass a narration model whose 
script fails the test if called... FunctionModel raising AssertionError inside 
fn would propagate as AssertionError, masking. Better: track a flag list; assert
flag untouched after the pytest.raises block.

Also "caller-correctable metric/identity problems remain clarification rather 
than Unsupported" — add test: unnormalizable metric label via intent → 
ClarificationNeeded through ask(). And invalid explicit identity + bad_request →
clarification (needs a mock client that raises EvidenceError bad_request — but 
ask() uses deps.client; Deps(client=...) with a custom client object? Deps is a 
dataclass with client field; I can pass _ErrorClient into Deps in 
intent_boundary tests too. But ask() — does it do anything with client beyond 
passing deps to planner? Stage 1 intent model (override), then planner with 
deps.client. Deps(client=custom) should work — planner calls 
client.fetch_evidence. In test_intent_boundary currently only stub_client is 
used, but Deps takes any client. Let me write with a small error client in the 
intent boundary file.

Hmm wait — actually one thing to verify: does ask() catch TerminalError 
anywhere? ask() in harness/agent.py — I can't see it (summary shows `ask(...)` 
signature with pipeline kwargs). It might catch HarnessError and convert to... 
The test `test_planner_failure_is_not_unsupported` must prove TerminalError 
propagates (per acceptance: "planner/schema defects propagate as 
TerminalError"). The plan test description: "planner/schema defects propagate as
`TerminalError`". So ask() must let it propagate. If ask() swallowed it, the 
test would fail and I couldn't fix agent.py (not editable). Risk acknowledged: 
I'll write the test to expect TerminalError from ask(). Given the plan says 
planner failures "propagate", ask() presumably doesn't catch them. The current 
planner raises TerminalError for data_unavailable/internal already, and there's 
presumably existing behavior... The intent boundary docstring says nothing. I'll
go with pytest.raises(TerminalError).

Wait, actually, let me reconsider: `ask()` — from test_run_cli's `answered()`, 
exceptions from the legacy loop... For the pipeline path, ask(deps, question, 
...) runs stages. TerminalError is a HarnessError; run.py main catches 
HarnessError → EXIT_FAILED. That's the operational-failure flow — consistent.

### Exact request tests

`test_metric_normalization_and_exact_profile_request`: VC-POC-01, intent filters
{"acara_sml_id": 40387}, requested_metrics ["Total Enrolments", "teaching 
staff", "teaching staff"] → request exactly {"acara_sml_id": 40387, "metrics": 
["total_enrolments", "teaching_fte"]} — dedup, no headcount, no non-teaching, no
year defaults. Use a recording mock client (_HappyClient records fetched). 
Assert client.fetched[0]["request"] == exact dict.

`test_vc_poc_03_routes_selector_to_ratios`: VC-POC-03, filters {"acara_sml_ids":
[40387, 40392]}, requested_metrics ["students per teaching FTE"] → request 
{"acara_sml_ids": [...], "ratios": ["students_per_teaching_fte"]} — no "metrics"
key.

`test_unknown_request_property_fails_before_fetch`: VC-POC-01 filters 
{"acara_sml_id": 40387, "year_range": "x"}? "year_range" — not a request 
property (metrics enum has "year_range" as a metric value but not a request 
field). Use {"acara_sml_id": 40387, "banana": 1} → TerminalError; assert no 
fetch recorded.

`test_peer_filters_require_explicit_nondefault_filter`: VC-POC-09 with 
peer_filters {"sectors": ["Catholic", "Independent"]} → ClarificationNeeded. 
Also absent → covered by derivation. I'll test default-only.

`test_segment_filters_reject_unknown_nested_key`: VC-POC-10 segment_filters 
{"banana": ["x"]} → TerminalError before fetch.

`test_schema_valid_emitter_bad_request_is_terminal`: VC-POC-01 with resolved 
school (directory-resolved ID 40387 via mention) or explicit... wait — explicit 
ID → identity clarification. Need TerminalError: use a school mention resolving 
to one ID, then fetch raises bad_request → TerminalError. _ErrorClient with 
school_directory returning exactly one record. _ErrorClient.school_directory 
returns [] currently — I'll add a configurable client. Let me create a 
`_DirectoryErrorClient(directory, error)` or parametrize _ErrorClient.

`test_invalid_explicit_identity_bad_request_clarifies`: VC-POC-01 filters 
{"acara_sml_id": 99999}, fetch raises bad_request → ClarificationNeeded.

`test_missing_market_definition_is_derived_without_model_hint`: VC-POC-05, 
intent with NO missing_material_inputs (empty), no filters, no mentions → 
ClarificationNeeded listing market_definition. Must not call fetch.

`test_distance_is_conditional_on_distance_market`: two cases: (a) VC-POC-05 
filters {"market_definition": "sa2"} → proceeds (no distance_km ask) — fetch 
happens; (b) filters {"market_definition": "distance"} without distance_km → 
ClarificationNeeded with distance_km. Maybe parametrize or two asserts. Name is 
singular; make one test with both sub-asserts. (a) proceeds: with a mock happy 
client, fetch called with {"market_definition": "sa2"} (plus school? VC-POC-05 
requires acara_sml_id too — include filters {"acara_sml_id": 40387, 
"market_definition": "sa2"}).

`test_ambiguous_school_clarifies_without_fetch`: VC-POC-01, mention "Some 
School", client returns 2 records → ClarificationNeeded; assert no fetch call; 
assert deps.resolved_school_ids == [] and deps.searched is False (no partial 
mutation); assert a search_schools tool call recorded.

Also zero-match case — add `test_unmatched_school_clarifies_without_fetch`.

Conflict: `test_conflicting_explicit_id_and_mention_fails` → TerminalError.

Transactional: `test_school_resolution_is_transactional` — two mentions, second 
resolves to zero → ClarificationNeeded, deps.resolved_school_ids empty.

Ordered/dedup: `test_mention_order_preserved_and_deduplicated` — mentions ["A", 
"B", "A"] with directory mapping A→40392, B→40387 → VC-POC-03 (acara_sml_ids) 
request [40392, 40387]... wait mention order A,B,A → [40392, 40387] deduped. For
VC-POC-03 requires ≥... schema minItems 1, fine (VC-POC-04 is the ≥2 one).

Now the directory mock needs per-search results. Build:

```python
class _DirectoryClient:
    def __init__(self, directory: dict[str, list[dict]], envelope=None, 
fetch_error=None):
        self.directory = directory  # search term (case-insensitive) → records
        ...
    async def school_directory(self, search, limit=25):
        self.searches.append(search)
        return self.directory.get(search, [])  # or raise
    async def fetch_evidence(self, question_id, request):
        self.fetched.append(...)
        if self.fetch_error: raise EvidenceError(...)
        return envelope
```

Records must carry "acara_sml_id" and "school_name".

Careful: how does the planner key directory results? It searches each mention, 
gets records; exactly one → take its acara_sml_id. Mock keyed by exact search 
string is fine.

### Clarification MissingInput shape

From outputs.py: MissingInput(input=..., why_material=..., allowed_values=...). 
Existing usage confirms.

### Policy-order test

`test_clarification_follows_policy_order`: VC-POC-07 requires [acara_sml_id, 
market_definition, grades], material [market_definition, grades], conditional 
distance_km. Intent: mentions ["Some School"] (satisfies acara via resolution) +
filters {"market_definition": "distance"} → missing = [grades, distance_km] 
(required order: acara_sml_id satisfied, market_definition present, grades 
missing → then conditional distance_km). Assert order ["grades", "distance_km"].
Good — stable order test.

Hmm wait — derivation order: required first (grades), then material 
(market_definition present, grades missing — already added? dedupe!), then 
conditional (distance_km). Need dedupe across passes: grades appears in both 
required and material → add once. Implement with a seen-set.

### VC-POC-05 derivation subtlety

For VC-POC-05 with market_definition absent: required list [acara_sml_id, 
market_definition] — acara_sml_id missing too (no filters, no mentions) → 
clarification lists [acara_sml_id, market_definition] in policy order. The 
acceptance test name: 
test_missing_market_definition_is_derived_without_model_hint — assert 
"market_definition" in missing names; acara_sml_id also present is fine. Hmm — 
but should I assert exact order [acara_sml_id, market_definition]? The registry 
order: required: ["acara_sml_id", "market_definition"]. I'll assert names 
contain market_definition and order follows policy.

Wait — actually, should a missing acara_sml_id be clarified, or should it be 
treated as "the school subject is missing"? ClarificationNeeded naming 
acara_sml_id is correct (caller must identify the school). allowed_values for 
acara_sml_id — release-derived, no static list → [].

### Material-input ordering per policy_order contract id

"Clarification fields ... follow pinned required, material, then applicable 
conditional-material order." OK.

### What about school resolution when question doesn't need schools?

If question has no school selector but intent has mentions — ignore mentions? 
E.g. VC-POC-10 with stray mention. The plan: resolve only when the question's 
school selector fields need them. If mentions exist and question has no acara 
field — skip resolution (or resolve anyway? Unnecessary I/O). I'll resolve only 
when the required list includes acara_sml_id/acara_sml_ids AND those aren't 
already explicitly supplied. Existing behavior resolved mentions for any 
question; but with the strict exactly-one rule, stray mentions on non-school 
questions would spuriously clarify. Skip when not needed.

Hmm — wait, but should mentions also be skipped when IDs are explicitly 
supplied? Yes (caller-supplied numeric IDs bypass directory preflight per 
direct-id-authority-conflict). And conflict check: if both explicit IDs and 
mentions present — resolve mentions to check conflict? That performs preflight 
for caller-supplied-ID cases with mentions... The plan says raise TerminalError 
for conflicts between explicit IDs and name-resolved identities — that requires 
resolving the mentions to compare. If mentions present AND explicit IDs present 
→ resolve mentions (each exactly one) → compare → conflict → TerminalError; 
consistent → proceed with explicit IDs. This performs the preflight but that's 
needed to detect conflict. Edge case, tests cover it.

But careful about transactional commit here: on conflict, "retain no partial 
identity update" — deps.resolved_school_ids unchanged. Since explicit IDs were 
caller-supplied, they were never in deps.resolved_school_ids (unless set by... 
deps starts empty). Commit nothing on conflict. OK.

### Explicit IDs in intent.filters — validate them?

Shape-check: acara_sml_id must be int ≥1 (schema does it at validation). Leave 
to schema validation.

### Now — schema validation for questions with explicit ID + no other fields

VC-POC-01 request {"acara_sml_id": 40387} validates (required acara_sml_id; 
additionalProperties false; metrics optional). Good.

### VC-POC-02/07 grades via filters?

If intent.filters contains grades (a list), include as explicit field; 
validation type-checks. Fine.

### What about VC-POC-12/13 filters like sa2_codes — shape only

Pass through; schema validation checks array of strings.

### Normalization of market_definition value

If filters has market_definition: "Distance" → normalize → "distance" 
(canonical). Replace in request. If unmatchable string → ClarificationNeeded. If
non-string (e.g. int) → unnormalizable → clarify? Schema would TerminalError it.
Hmm — "requested selectors that cannot be normalized" → clarify. A non-string 
market_definition... I'll clarify only for strings that don't match; non-strings
pass to schema validation → TerminalError. Reasonable.

Also — market_definition normalization failure message should list 
market_definitions allowed values (static closed — from catalog enum).

What about seifa_index_type/seifa_measure (static closed scalars, VC-POC-12)? 
They come via filters from the intent model. Should the planner normalize them 
too? The mapping "VC-POC-12.seifa_index_type": "seifa_index_types". To keep 
scope bounded: normalize scalar static-closed selectors generically? The plan's 
normalization targets "requested selectors" — requested_metrics bag primarily. 
VC-POC-12 erp_age/seifa usually come from defaults (omitted). I'll normalize 
market_definition (needed for the conditional) and the selector bag fields 
(metrics/ratios/grades). seifa scalars pass through — schema enum validation 
catches bad values → TerminalError... hmm, that would make a model-typo'd seifa 
measure an operational failure rather than clarification. But per "invalid 
closed-vocabulary input → bad_request" (emitter repairs in legacy) — in the 
pipeline, a well-typed non-matching seifa value → schema enum → TerminalError 
per fail-closed planner rules. Is that per plan? "A user selector that cannot be
normalized or a caller-supplied identity requiring correction is a 
clarification". Seifa measure IS a static-closed selector; unnormalizable → 
clarify would be kinder. Scope: the plan explicitly names normalization of 
selector labels routed to "the question's canonical catalog vocabulary and 
request field" — focused on the metrics/ratios/grades bag. Tests required only 
cover those. To limit risk, I'll ALSO normalize seifa_index_type/seifa_measure 
scalars? Hmm... no — keep to: market_definition scalar (needed for 
conditional-material logic) + bag fields. Actually wait — let me reconsider: the
"static-vocabulary-source" contract id says "Static selector routing uses 
registry metric_list and the input-catalog schema's question-field mapping and 
embedded enums". market_definition routing is in the mapping. I'll implement 
generic scalar normalization for market_definition only (needed for correctness 
of condition evaluation), and bag-field normalization via the route. Others pass
through to schema validation. Documented in code comments.

Hmm, actually — one more: should I normalize values inside 
peer_filters/segment_filters (e.g. school_types ["secondary"] → ["Secondary"])? 
Those are release-derived (OD-03: NOT enum-enforced; well-typed non-match → 
empty result). "secondary" vs "Secondary" — release-derived values are 
case-sensitive in the DB. Normalizing case would change semantics (no canonical 
case mapping pinned). OD-03 says pass through. Skip — shape-check only. Good.

### Now, the ordering of planner stages

Full flow:

```
1. excluded topic → Unsupported
2. registry excluded question → Unsupported
3. gated → Unsupported
4. unknown → Unsupported
5. Load policy + question entry (TerminalError if missing from registry/index — 
drift)
6. Assemble base request from intent.filters (explicit fields); normalize 
market_definition if present (clarify if unnormalizable)
7. School selection:
   - explicit ids = filters.get("acara_sml_id"/"acara_sml_ids")
   - if question requires school field:
     - if mentions and explicit ids: resolve mentions transactionally (no commit
on failure); compare → conflict → TerminalError; ids stay explicit
     - elif mentions: resolve transactionally; on zero/multi → Clarify 
(preflight recorded, no commit); on success → commit to deps (searched=True, 
resolved_school_ids=deduped ids)
     - explicit ids only: no preflight; caller-supplied flag = True
   - attach ids to request (field name per required list)
8. VC-POC-04 (≥2 distinct ids check): if required includes acara_sml_ids and 
constraint mentions two values and len(distinct ids) < 2 → Clarify
9. Derive missing (required → material → conditional) against assembled request 
+ mentions-satisfaction:
   - school fields satisfied if ids present (explicit or resolved)
   - Clarify with ordered MissingInputs
10. Selector normalization & routing:
   - route = _selector_route(question_id)
   - if requested_metrics and route: normalize each → unnormalizable → Clarify 
(allowed values = vocab); dedupe preserve order
   - request[route_field] = cleaned list
   - scope rule (request_scope_problem on deps.user_question) applied to metrics
→ clean
   - elif requested_metrics and no route: hmm — user asked for metrics on a 
question with no selector bag (e.g. VC-POC-05 with requested_metrics ["total 
enrolments"]). What to do? Options: ignore (silently) or clarify. "Submit ... 
omit selector fields when the user named none" — the converse: named some → 
submit to the routed field. No route → the labels can't be used. Clarify? That 
seems right: "a requested selector that cannot be normalized" → clarification 
with... no vocabulary. Hmm. Simplest safe: if route is None and 
requested_metrics non-empty → Clarify (unusable selector). But careful: would 
break... VC-POC-02/07 grades have routes. VC-POC-08/12/13 don't. If the intent 
model fills requested_metrics for those — possible stray. I'll clarify with why 
"this question does not take metric selectors" and allowed []. Hmm — actually 
wait: risk of over-clarifying existing flows? Current tests: intent_boundary 
happy paths use VC-POC-01 with metrics — has route. test_happy_path VC-POC-01 
filters only — no requested_metrics... Let me check test_intent_boundary 
test_ask_pipeline_intent_override_bypasses_model: 
requested_metrics=["total_enrolments", "teaching_fte"], VC-POC-01 → route 
metrics → normalized canonical → request includes metrics. Good.
   For safety, clarifying on no-route-with-metrics could surprise; but it's 
fail-closed-honest. Alternatively ignore them silently — dishonest. I'll 
clarify.

   Wait — actually, hmm, one more consideration: VC-POC-12 "erp_age" etc. could 
arrive via requested_metrics? Unlikely. Keep clarify.
11. VC-POC-09/10 explicit-filter constraint → Clarify if violated
12. Schema validation → TerminalError on any failure (including load failures)
13. Fetch with arguments; record failures; branch:
    - EvidenceError bad_request: caller-identity present → Clarify identity; 
else TerminalError(question id + emitter context)
    - excluded/unknown/unanswerable → Unsupported (as before)
    - data_unavailable/internal → TerminalError
14. Success: record, deps.last_envelope, return.
```

Order check — constraint (11) vs validation (12): for VC-POC-09 unknown nested 
key: constraint passes (non-empty, has unknown key which is non-sector → 
"explicit filter") → schema validation → additionalProperties → TerminalError. 
Good — matches "unknown nested keys fail before dispatch" (TerminalError, per 
contract failure_behaviour: "A planner-built request violates its registered 
schema, contains an unknown top-level or nested field → TerminalError").

Wait, contract failure table says unknown nested key → TerminalError? Let me 
recheck: "condition: A planner-built request violates its registered schema, 
contains an unknown top-level or nested field, or cannot load or resolve its 
pinned schema safely. expected: Raise TerminalError before evidence dispatch; do
not return Unsupported." And acceptance test name: 
test_segment_filters_reject_unknown_nested_key — I'll assert TerminalError. But 
hmm, the behaviour id "explicit-filter-constraints" says "unknown nested keys 
fail before dispatch" — "fail" ambiguous; plan text: "unknown filter keys are 
planner validation failures" → TerminalError. Yes.

### Deriving missing — interaction with selector route

For VC-POC-07, required includes grades; if requested_metrics present → route 
grades; satisfied. If requested_metrics empty → grades missing → clarify. If 
intent.filters has grades → satisfied. Order: derivation (step 9) BEFORE 
normalization (step 10)? If grades will be routed from requested_metrics, 
derivation must count it as satisfied. So satisfaction check: field in request 
OR (field == route_field AND requested_metrics non-empty). I'll compute the 
routed selector early and include in satisfaction.

Simpler: do normalization/routing BEFORE derivation, then derivation sees the 
complete request. Order: assemble → school resolution → route/normalize 
selectors (clarify on unnormalizable) → constraint checks → derive missing → 
VC-POC-04 id-count check → schema validation → fetch.

Hmm, but VC-POC-04 id-count: after resolution. Put it right after attaching ids,
before selector normalization — earlier is fine.

Also scope rule before validation — after normalization. Then derivation. Fine.

Wait — one problem: normalization happening before missing-derivation means an 
unnormalizable metric label clarifies even if required inputs are ALSO missing. 
Acceptable; clarification content differs but both are clarifications.

Order refinement — actually the acceptance 
`test_missing_market_definition_is_derived_without_model_hint` has no selectors 
→ unaffected.

### normalized market_definition & conditional

filters {"market_definition": "distance"} → request market_definition = 
"distance"; conditional distance_km missing → clarify distance_km. Also alias? 
market_definitions vocab: distance/lga/postcode/sa2 — no aliases.

### Explicit school id fields in request assembly

filters may have acara_sml_id; attach resolved ids:

```python
q_entry = contract.questions()[question_id]
required = policy required list
if resolved ids and "acara_sml_ids" in required and "acara_sml_ids" not in 
request:
    request["acara_sml_ids"] = committed_ids
if resolved ids and "acara_sml_id" in required and "acara_sml_id" not in 
request:
    request["acara_sml_id"] = committed_ids[0]
```

But what if filters has acara_sml_id (singular) and question requires 
acara_sml_ids (e.g. user gave one explicit ID for VC-POC-03)? Schema requires 
acara_sml_ids (plural) → request missing required field → derivation flags 
acara_sml_ids missing (school field not satisfied: no explicit plural, no 
mentions) → clarify? Hmm — but the user DID give an ID. Should the planner adapt
singular→plural? That's semantic adaptation; the plan: "Treat school selector 
fields as satisfiable by explicit IDs" — explicit IDs in whichever field? If 
filters has acara_sml_id=40387 for VC-POC-03, treating it as the caller's 
explicit selection → wrap into acara_sml_ids? I think yes — caller-supplied 
identity is authoritative; the field-name mapping is planner responsibility 
("Attach resolved school IDs using the right field name for this question"). 
Current code does setdefault per required name only from resolved. For 
caller-supplied: I'll gather explicit ids from either field, and emit under the 
required field name. Explicit singular → [id] under plural for VC-POC-03/04; 
explicit plural → first under singular? For VC-POC-01 with filters 
{"acara_sml_ids": [40387, 40392]}?? Schema VC-POC-01 has only acara_sml_id. 
Emitting acara_sml_id=40387 (dropping the second) silently is dishonest; 
emitting both can't fit schema → TerminalError. Hmm. Simplest robust: 
- Collect explicit ids: from "acara_sml_id" (single) and "acara_sml_ids" (list).
- If question requires "acara_sml_ids": request["acara_sml_ids"] = explicit list
(wrap singular into [singular] if plural absent).
- If requires "acara_sml_id": if plural explicit with >1 → TerminalError (cannot
satisfy schema honestly); if exactly 1 → request["acara_sml_id"] = that; if 
singular → as-is.

Let me formalize:

```python
explicit_single = request.pop("acara_sml_id", None) if isinstance(...)  
```

Hmm, wait — careful: intent.filters for VC-POC-01 carries acara_sml_id — that's 
already the right field. Let me not pop/rename when the field matches the 
required name. Only do cross-field adaptation when the explicit field does NOT 
match the required one:

```python
if "acara_sml_ids" in required and "acara_sml_ids" not in request and 
"acara_sml_id" in request:
    request["acara_sml_ids"] = [request.pop("acara_sml_id")]
if "acara_sml_id" in required and "acara_sml_id" not in request and 
"acara_sml_ids" in request:
    ids = request["acara_sml_ids"]
    if len(ids) != 1: TerminalError("cannot represent multiple IDs in 
acara_sml_id")
    request["acara_sml_id"] = ids[0]; del request["acara_sml_ids"]
```

Hmm, is this over-engineering? It handles real model variance. But each 
adaptation adds risk. The required test set doesn't cover it. I'll implement the
plural adaptation (single→plural) and the singular adaptation only when exactly 
one (else TerminalError). Reasonable and honest.

Actually — wait. For VC-POC-04 with a single explicit acara_sml_id: wrapped to 
[id] → then ≥2 check → ClarificationNeeded ("at least two distinct schools"). 
Good, honest.

### deps.resolved_school_ids — what about explicit IDs?

Explicit caller IDs don't go into deps.resolved_school_ids (that's for 
directory-resolved). Current code only appends resolved. Keep.

### ToolInvocation record shape

From tools.py: ToolInvocation(tool, args, result=None, error_kind=None) — 
dataclass. Recording preflight: ToolInvocation("search_schools", {"search": 
name, "limit": 25}, records) or error_kind=... Keep.

### Recording failed fetch "before branching"

```python
except EvidenceError as exc:
    deps.tool_calls.append(ToolInvocation("fetch_evidence", arguments, 
error_kind=exc.error_kind))
    return _branch_evidence_error(...)
```

Already the pattern; keep. For TerminalError-raising branch, record BEFORE 
raising (the append happens first). Good.

EvidenceError signature: EvidenceError(error_kind, message, reason_code) per 
_ErrorClient usage. It has .error_kind, .message, .reason_code.

### _branch_evidence_error updated signature

Needs the request/identity-origin info:

```python
def _branch_evidence_error(question_id, exc, request, identity_from_caller) -> 
Unsupported | ClarificationNeeded
```

Identity-from-caller detection: request contains "acara_sml_id" or 
"acara_sml_ids" whose values originated from intent.filters. Track a boolean 
during planning: `caller_identity_fields = [f for f in 
("acara_sml_id","acara_sml_ids") if f in intent.filters]`. If non-empty → 
bad_request → clarify.

Hmm — edge: caller supplied acara_sml_id AND a mention resolved... conflict case
already TerminalError'd or consistent. If consistent, request ids came from 
caller → clarify on bad_request. Fine.

### Now — writing `_derive_missing` with satisfiable-by-mentions

At the time of derivation, resolution has already happened (or clarified). So 
school fields: satisfied iff request has the field (after 
adaptation/attachment). If mentions existed and resolution succeeded, ids 
attached. So simple presence check works. School field not in request and 
mentions were absent → missing.

Wait — the transactional flow: mentions resolve → clarify happens INSIDE 
resolution before derivation. So at derivation, any surviving path has ids 
attached or no mentions. Presence check suffices.

### Unnormalizable selector clarification — which field name?

route field (metrics/ratios/grades) — use contract field name per contract 
requirement ("Use stable policy order, contract field names, and populated 
static allowed values where the pinned catalog mapping supplies them").

### The input-catalog mapping and `<keys>` entries

`"VC-POC-09.peer_filters.<keys>": "peer_filter_keys"` — for nested key 
allow-lists. I don't strictly need them (schema validation catches unknown 
keys). But contract id static-vocabulary-source mentions using the mapping. I'll
use it for the clarification allowed values on unnormalizable selectors 
(metrics/ratios/grades) and market_definition. The `<keys>` entries I'll skip 
(they're key names, not values). Fine.

### Where to get the registry metric_lists

contract.registry() — returns the registry JSON presumably (with 
"metric_lists"). contract.questions() returns dict id→entry presumably built 
from registry["questions"] (a LIST in the registry file!). questions() must 
convert to dict keyed by question_id. `contract.input_policy_for(question_id)` 
exists → returns entry["input_policy"]. I'll use contract.questions().get(qid) 
and .input_policy_for(qid).

Hmm — but I don't know exactly what contract.registry() returns. It's 
`registry() -> dict[str, Any]` lru_cached — the whole registry file. 
registry()["metric_lists"]["profile_metrics"] — safe. And questions() → dict per
signature `dict[str, dict[str, Any]]` — keyed by question_id with entries from 
the questions array. input_policy_for returns entry["input_policy"] presumably. 
For metric_list I need the entry: contract.questions()[qid].get("metric_list"). 
Since current planner uses contract.questions() and contract.refusal_for etc., 
this is consistent.

Risk: questions() might include/exclude gated etc. — unknown, but existing 
planner gates handle that before we need entries.

### Loading _index.json vs registry request_schema

Per-question schema path: I'll use `_index.json` (pinned) — 
`questions[qid]["request_schema"]` → filename. Check qid in index → else 
TerminalError drift.

### The validator: where to put it

planner.py — private helpers. Keep code organized.

### Determinism of dict ordering

Request construction: start from dict(intent.filters) — insertion order from 
intent; then attach ids/selector. JSON key order for exact-dict tests: Python 
dict equality ignores order. Fine.

### Now — scope rule details

`request_scope_problem(user_question, question_id, request)` — returns truthy 
problem string or None. Current usage:

```python
scope_problem = request_scope_problem(deps.user_question, question_id, request)
if scope_problem and "metrics" in request and isinstance(request["metrics"], 
list):
    cleaned = [...]
    request["metrics"] = cleaned or request["metrics"]
```

Keep identical logic, applied on the normalized list. deps.user_question — Deps 
dataclass field (exists per tools.py Deps). In planner tests, Deps(client=...) —
user_question defaults to ""? Presumably (""). Existing tests pass without 
setting it, and current planner calls request_scope_problem(deps.user_question, 
...) — so default exists. Good.

### Now write the new planner.py

Let me draft:

```python
"""Stage 2 of the 4-stage pipeline: deterministic planning and execution.

... (updated docstring) ...
"""

from __future__ import annotations

import json
import logging
import re
from functools import lru_cache
from pathlib import Path
from typing import Any

from harness import contract
from harness.errors import TerminalError
from harness.evidence import EvidenceClient, EvidenceError
from harness.intent import StructuredIntent
from harness.outputs import ClarificationNeeded, MissingInput, Unsupported
from harness.tools import Deps, ToolInvocation, request_scope_problem

logger = logging.getLogger("harness.planner")

CONTRACT_DIR = Path(__file__).resolve().parents[1] / "contract"
_SCHEMA_INDEX_PATH = CONTRACT_DIR / "question_schemas" / "v0.3.0" / 
"_index.json"
_INPUT_CATALOG_SCHEMA_PATH = CONTRACT_DIR / 
"poc_input_catalog.v0.3.0.schema.json"

# The one pinned semantic alias (prompts/rules.md): "teaching staff" is the
# teaching FTE metric ...
_PINNED_ALIASES = {"teaching staff": "teaching_fte"}

_SCHOOL_FIELDS = ("acara_sml_id", "acara_sml_ids")
```

Exclusion keyword map — keep as-is.

Schema index/catalog loaders:

```python
def _load_json(path: Path) -> Any:
    try:
        return json.loads(path.read_text(encoding="utf-8"))
    except (OSError, ValueError) as exc:
        raise TerminalError(f"pinned contract artefact could not be loaded: 
{path.name}: {exc}") from exc


@lru_cache(maxsize=1)
def _schema_index() -> dict[str, Any]:
    index = _load_json(_SCHEMA_INDEX_PATH)
    if not isinstance(index, dict) or not isinstance(index.get("questions"), 
dict):
        raise TerminalError("pinned question-schema index is malformed")
    return index


@lru_cache(maxsize=1)
def _input_catalog_schema() -> dict[str, Any]:
    schema = _load_json(_INPUT_CATALOG_SCHEMA_PATH)
    if not isinstance(schema, dict):
        raise TerminalError("pinned input-catalog schema is malformed")
    return schema


def _registered_schema_path(question_id: str) -> Path:
    entry = _schema_index()["questions"].get(question_id)
    if entry is None:
        raise TerminalError(f"{question_id} has no registered request schema")
    filename = entry.get("request_schema")
    if not isinstance(filename, str):
        raise TerminalError(f"{question_id} has no registered request schema 
filename")
    if filename != f"{question_id}.request.schema.json":
        raise TerminalError(f"{question_id} request-schema registration is 
inconsistent: {filename}")
    path = _SCHEMA_INDEX_PATH.parent / filename
    resolved = path.resolve()
    if CONTRACT_DIR.resolve() not in resolved.parents:
        raise TerminalError(f"refusing to load schema outside the pinned 
contract directory: {filename}")
    return path


@lru_cache(maxsize=None)
def _request_schema(question_id: str) -> dict[str, Any]:
    schema = _load_json(_registered_schema_path(question_id))
    if not isinstance(schema, dict):
        raise TerminalError(f"{question_id} request schema is malformed")
    return schema
```

Wait — `CONTRACT_DIR.resolve() not in resolved.parents`: parents of 
`.../contract/question_schemas/v0.3.0/VC-POC-01.request.schema.json` include 
`.../contract/question_schemas/v0.3.0`, `.../contract/question_schemas`, 
`.../contract`, `.../k-12Harness`, ... So `CONTRACT_DIR.resolve() in 
resolved.parents` → True. Good.

Static vocabulary helpers:

```python
@lru_cache(maxsize=1)
def _field_to_catalog_list() -> dict[str, str]:
    mapping = _input_catalog_schema().get("x-poc-request-field-to-catalog-list",
{})
    return {k: v for k, v in mapping.items() if isinstance(k, str) and 
isinstance(v, str)}


def _static_enum(list_name: str) -> list[str] | None:
    """Embedded static enum for a catalog list name; None for 
release-derived."""
    if list_name.startswith("release_derived_values."):
        return None
    prop = _input_catalog_schema().get("properties", {}).get(list_name)
    if not isinstance(prop, dict):
        return None
    items = prop.get("items")
    if isinstance(items, dict) and isinstance(items.get("enum"), list):
        return [str(v) for v in items["enum"]]
    return None


def _selector_route(question_id: str) -> tuple[str, list[str]] | None:
    """(request field, static vocabulary) for the question's label-selector 
bag."""
    entry = contract.questions().get(question_id) or {}
    metric_list = entry.get("metric_list")
    if metric_list:
        values = contract.registry().get("metric_lists", {}).get(metric_list)
        if isinstance(values, list) and values:
            return "metrics", [str(v) for v in values]
        raise TerminalError(f"{question_id} declares unknown metric_list 
{metric_list!r}")
    for key, list_name in sorted(_field_to_catalog_list().items()):
        prefix = f"{question_id}."
        if not key.startswith(prefix):
            continue
        field = key[len(prefix):]
        if field in ("metrics", "ratios", "grades"):
            vocab = _static_enum(list_name)
            if vocab is None:
                raise TerminalError(f"{question_id}.{field} routes to a 
non-static vocabulary")
            return field, vocab
    return None
```

Hmm — `sorted(...)` for determinism. Multiple array fields per question? None 
have two among {metrics, ratios, grades}. OK.

Wait — registry metric_list for VC-POC-01 is "profile_metrics" → registry 
metric_lists.profile_metrics. But the catalog schema also embeds the same enum 
(consistency baseline). Using the registry for metric_list (per plan: "registry 
`metric_list` where declared"), catalog enum otherwise. Good.

market_definition vocabulary: `VC-POC-05.market_definition → market_definitions`
etc. Helper:

```python
def _static_vocabulary_for_field(question_id: str, field: str) -> list[str] | 
None:
    list_name = _field_to_catalog_list().get(f"{question_id}.{field}")
    if list_name is None:
        return None
    return _static_enum(list_name)
```

Normalization:

```python
_SEPARATOR_RE = re.compile(r"[\W_]+", re.UNICODE)

def _normalize_label(label: Any) -> str:
    text = _SEPARATOR_RE.sub(" ", str(label).casefold())
    return text.strip()
```

Wait `[\W_]+` on "pre_year_1_one_year".casefold() → "pre year 1 one year". On 
"Year 6" → "year 6". Both match "year_6"→"year 6". Good. Unicode: "café" stays 
"café" (\W matches non-word; accented letters are word chars in Python re 
UNICODE). Good.

Canonical lookup:

```python
def _canonical_for(label: str, vocabulary: list[str]) -> str | None:
    norm = _normalize_label(label)
    for value in vocabulary:
        if _normalize_label(value) == norm:
            return value
    alias_target = _PINNED_ALIASES.get(norm)
    if alias_target is not None and alias_target in vocabulary:
        return alias_target
    return None
```

Wait — alias: "teaching staff" normalizes to "teaching staff"; alias map keyed 
by normalized label → target. But "teaching staff" might ALSO normalize-match...
"teaching_staff_headcount" normalizes "teaching staff headcount" ≠ "teaching 
staff". Good, no collision. But careful: alias must not shadow a direct 
vocabulary match — check direct match first (done). 

Dedup:

```python
def _dedupe(values: list[str]) -> list[str]:
    seen: set[str] = set()
    out = []
    for v in values:
        if v not in seen:
            seen.add(v)
            out.append(v)
    return out
```

Validation:

```python
def _validate_request(question_id: str, request: dict[str, Any]) -> None:
    schema = _request_schema(question_id)
    problems = _schema_problems(request, schema, "$")
    if problems:
        raise TerminalError(
            f"{question_id}: planner-built request does not satisfy its 
registered "
            f"schema: {'; '.join(problems)}"
        )
```

_schema_problems — the subset validator. Let me write it:

```python
def _type_ok(value: Any, expected: str) -> bool:
    if expected == "object":
        return isinstance(value, dict)
    if expected == "array":
        return isinstance(value, list)
    if expected == "string":
        return isinstance(value, str)
    if expected == "integer":
        return (isinstance(value, int) and not isinstance(value, bool)) or (
            isinstance(value, float) and value.is_integer()
        )
    if expected == "number":
        return isinstance(value, (int, float)) and not isinstance(value, bool)
    if expected == "boolean":
        return isinstance(value, bool)
    if expected == "null":
        return value is None
    return False


def _json_key(value: Any) -> str:
    return json.dumps(value, sort_keys=True, ensure_ascii=False, default=str)


def _schema_problems(value: Any, schema: Any, path: str) -> list[str]:
    problems: list[str] = []
    if not isinstance(schema, dict):
        return problems
    expected_type = schema.get("type")
    if expected_type is not None:
        types = expected_type if isinstance(expected_type, list) else 
[expected_type]
        if not any(_type_ok(value, t) for t in types):
            problems.append(f"{path}: expected type {expected_type!r}")
            return problems
    if "const" in schema and value != schema["const"]:
        problems.append(f"{path}: must equal {schema['const']!r}")
    enum = schema.get("enum")
    if isinstance(enum, list) and not any(
        _json_key(value) == _json_key(candidate) for candidate in enum
    ):
        problems.append(f"{path}: value {value!r} is not one of the allowed 
values")
    if isinstance(value, dict):
        properties = schema.get("properties", {})
        if isinstance(properties, dict):
            for key, sub in properties.items():
                if key in value:
                    problems.extend(_schema_problems(value[key], sub, 
f"{path}.{key}"))
        additional = schema.get("additionalProperties", True)
        if additional is False:
            unknown = [k for k in value if k not in properties]
            if unknown:
                problems.append(f"{path}: unknown properties {sorted(unknown)}")
        elif isinstance(additional, dict):
            for key in value:
                if key not in properties:
                    problems.extend(_schema_problems(value[key], additional, 
f"{path}.{key}"))
        for check, limit in (("minProperties", ...), ...)
        ...
        required = schema.get("required")
        if isinstance(required, list):
            for key in required:
                if key not in value:
                    problems.append(f"{path}: missing required property 
{key!r}")
        min_properties = schema.get("minProperties")
        if isinstance(min_properties, int) and len(value) < min_properties:
            problems.append(f"{path}: needs at least {min_properties} 
properties")
        max_properties = schema.get("maxProperties")
        if isinstance(max_properties, int) and len(value) > max_properties:
            problems.append(f"{path}: allows at most {max_properties} 
properties")
    if isinstance(value, list):
        items = schema.get("items")
        min_items = schema.get("minItems")
        max_items = schema.get("maxItems")
        unique_items = schema.get("uniqueItems")
        if isinstance(min_items, int) and len(value) < min_items:
            problems.append(f"{path}: needs at least {min_items} items")
        if isinstance(max_items, int) and len(value) > max_items:
            problems.append(f"{path}: allows at most {max_items} items")
        if unique_items:
            keys = [_json_key(v) for v in value]
            if len(set(keys)) != len(keys):
                problems.append(f"{path}: items must be unique")
        if isinstance(items, dict):
            for i, element in enumerate(value):
                problems.extend(_schema_problems(element, items, 
f"{path}[{i}]"))
    if isinstance(value, str):
        pattern = schema.get("pattern")
        if isinstance(pattern, str) and re.search(pattern, value) is None:
            problems.append(f"{path}: does not match required pattern")
        min_length = schema.get("minLength")
        if isinstance(min_length, int) and len(value) < min_length:
            problems.append(f"{path}: shorter than minLength {min_length}")
        max_length = schema.get("maxLength")
        if isinstance(max_length, int) and len(value) > max_length:
            problems.append(f"{path}: longer than maxLength {max_length}")
    if isinstance(value, (int, float)) and not isinstance(value, bool):
        minimum = schema.get("minimum")
        if isinstance(minimum, (int, float)) and value < minimum:
            problems.append(f"{path}: below minimum {minimum}")
        maximum = schema.get("maximum")
        if isinstance(maximum, (int, float)) and value > maximum:
            problems.append(f"{path}: above maximum {maximum}")
        multiple = schema.get("multipleOf")
        if isinstance(multiple, (int, float)) and multiple > 0:
            quotient = value / multiple
            if abs(quotient - round(quotient)) > 1e-9:
                problems.append(f"{path}: not a multiple of {multiple}")
    for keyword in ("allOf", "anyOf"):
        subs = schema.get(keyword)
        if isinstance(subs, list):
            for sub in subs:
                problems.extend(_schema_problems(value, sub, path))
    # anyOf must be handled differently!
```

Hmm wait — anyOf: pass if ANY subschema validates. My loop above appends 
problems for each — wrong. Let me restructure combinators properly:

```python
    def validate(sub) -> list[str]:
        return _schema_problems(value, sub, path)

    all_of = schema.get("allOf")
    if isinstance(all_of, list):
        for sub in all_of:
            problems.extend(validate(sub))
    any_of = schema.get("anyOf")
    if isinstance(any_of, list):
        results = [validate(sub) for sub in any_of]
        if all(len(r) > 0 for r in results):
            problems.append(f"{path}: does not match any allowed variant")
    one_of = schema.get("oneOf")
    if isinstance(one_of, list):
        matching = sum(1 for sub in one_of if not validate(sub))
        if matching != 1:
            problems.append(f"{path}: must match exactly one allowed variant 
(matched {matching})")
    if "if" in schema:
        condition_ok = not validate(schema["if"])
        branch = schema.get("then") if condition_ok else schema.get("else")
        if isinstance(branch, dict):
            problems.extend(validate(branch))
    not_schema = schema.get("not")
    if isinstance(not_schema, dict) and not validate(not_schema):
        problems.append(f"{path}: matches a forbidden variant")
```

Careful — validate(sub) rechecks type etc. at same path. Fine.

One caveat: `if` evaluation with missing data — e.g. `{"if": {"properties": 
{"market_definition": {"const": "distance"}}, "required": 
["market_definition"]}, "then": {"required": ["distance_km"]}}`. If 
market_definition absent → "if" fails (required) → condition false → then 
skipped → no distance_km needed. Correct semantics.

Recursive depth — schemas are shallow. Fine.

### Now the main flow. Let me write plan_and_execute fully.

```python
async def plan_and_execute(intent, client, deps):
    question_id = intent.question_id

    # 1-4 as before (excluded topic, registry exclusion, gated, unknown)

    entry = contract.questions().get(question_id)
    if entry is None:
        raise TerminalError(f"{question_id} is registered in the index but 
absent from the pinned registry")
    policy = entry.get("input_policy", {}) or {}
    required_fields = list(policy.get("required", []))
    
    # base request from explicit intent fields
    request: dict[str, Any] = dict(intent.filters)

    # normalize market_definition against its static vocabulary
    if "market_definition" in request:
        request["market_definition"] = await ... 
```

Hmm — normalization of market_definition needs clarification return. Not async. 
Let me restructure with helper returning either request or ClarificationNeeded. 
Use early-return pattern:

```python
    clarification = _normalize_scalar_selectors(question_id, request)
    if clarification is not None:
        return clarification
```

```python
def _normalize_scalar_selectors(question_id, request) -> ClarificationNeeded | 
None:
    if "market_definition" not in request:
        return None
    vocab = _static_vocabulary_for_field(question_id, "market_definition")
    if vocab is None:
        return None
    value = request["market_definition"]
    if not isinstance(value, str):
        return None
    canonical = _canonical_for(value, vocab)
    if canonical is None:
        return ClarificationNeeded(
            question_id=question_id,
            missing=[MissingInput(
                input="market_definition",
                why_material=(f"'{value}' is not an allowed market 
definition."),
                allowed_values=vocab,
            )],
            prompt_to_user=(...)",
        )
    request["market_definition"] = canonical
    return None
```

Prompt: f"Which market definition should I use — one of: {', '.join(vocab)}?"

School resolution:

```python
    school_required = [f for f in required_fields if f in _SCHOOL_FIELDS]
    explicit_fields = [f for f in _SCHOOL_FIELDS if f in request]
    mentions = list(intent.school_mentions or [])
    identity_from_caller = bool(explicit_fields)
    needs_schools = bool(school_required)

    if needs_schools and mentions and not explicit_fields:
        resolved_ids, failure = await _resolve_mentions(question_id, mentions, 
client, deps)
        if failure is not None:
            return failure
        # commit
        deps.resolved_school_ids = resolved_ids  # or extend?
        deps.searched = True
        _attach_ids(request, school_required, resolved_ids)
    elif needs_schools and mentions and explicit_fields:
        resolved_ids, failure = await _resolve_mentions(question_id, mentions, 
client, deps)
        if failure is not None:
            return failure
        explicit_ids = _explicit_id_values(request, school_required)
        if sorted(resolved_ids) != sorted(explicit_ids):
            raise TerminalError(...)
        # no commit of resolved (they equal explicit); searched? Preflights 
happened — record tool calls (done in _resolve_mentions); set deps.searched? The
preflights completed successfully... I'll set deps.searched = True here too? Hmm
— "commit to Deps only after all mentions succeed" refers to resolved IDs. 
searched flag indicates a directory preflight happened. I'll set 
deps.searched=True after successful full resolution in both branches. Actually —
in the conflict branch we raise after; don't set. In the equal branch, set it.
```

Hmm, let me simplify: `_resolve_mentions` returns (ids, clarification|None) and 
records preflights. Caller commits.

What about deps.resolved_school_ids when explicit IDs given (no mentions)? Leave
as-is (existing tests: school resolution test expects mention→40387; explicit-ID
tests don't check resolved_school_ids... test_intent_boundary 
test_ask_pipeline_intent_override_bypasses_model: filters acara_sml_id, no 
mentions — deps.resolved_school_ids stays []. Fine.)

Conflict definition: `sorted(resolved_ids) != sorted(explicit_ids)` — multiset 
comparison. If caller gave 1 id and mentions resolve to same 1 → equal → 
proceed. Good.

Attach:

```python
def _attach_ids(request, school_required, ids):
    if not ids:
        return
    if "acara_sml_ids" in school_required:
        request.setdefault("acara_sml_ids", list(ids))
    elif "acara_sml_id" in school_required:
        request.setdefault("acara_sml_id", ids[0])
```

Explicit cross-field adaptation BEFORE resolution? The caller-supplied 
adaptation (singular→plural) should happen before the explicit/mentions 
branching so explicit_fields reflects the required field name. Let me do 
adaptation early:

```python
    _adapt_explicit_school_fields(request, school_required)
```

```python
def _adapt_explicit_school_fields(request, school_required):
    if "acara_sml_ids" in school_required and "acara_sml_ids" not in request and
isinstance(request.get("acara_sml_id"), int):
        request["acara_sml_ids"] = [request.pop("acara_sml_id")]
    elif "acara_sml_id" in school_required and "acara_sml_id" not in request and
isinstance(request.get("acara_sml_ids"), list):
        ids = request["acara_sml_ids"]
        if len(ids) != 1 or not all(isinstance(i, int) and not isinstance(i, 
bool) for i in ids):
            raise TerminalError("caller supplied multiple acara_sml_ids for a 
question that takes a single acara_sml_id")
        request["acara_sml_id"] = ids[0]
        del request["acara_sml_ids"]
```

Hmm wait — `request.pop("acara_sml_id")` after isinstance check; value could be 
non-int (schema will catch later, but pop only when int). If acara_sml_id is, 
say, "40387" (string) with required plural → leave as-is → schema validation 
TerminalError (string type). OK.

_explicit_id_values:

```python
def _explicit_id_values(request, school_required):
    if "acara_sml_ids" in school_required:
        v = request.get("acara_sml_ids")
        return list(v) if isinstance(v, list) else []
    if "acara_sml_id" in school_required:
        v = request.get("acara_sml_id")
        return [v] if isinstance(v, int) and not isinstance(v, bool) else []
    return []
```

_resolve_mentions:

```python
async def _resolve_mentions(question_id, mentions, client, deps):
    staged: list[int] = []
    seen: set[int] = set()
    for mention in mentions:
        args = {"search": mention, "limit": 25}
        try:
            records = await client.school_directory(search=mention, limit=25)
        except Exception as exc:
            deps.tool_calls.append(ToolInvocation("search_schools", args, 
error_kind=type(exc).__name__))
            raise
        deps.tool_calls.append(ToolInvocation("search_schools", args, records))
        ids = [r["acara_sml_id"] for r in records if isinstance(r, dict) and 
isinstance(r.get("acara_sml_id"), int) and not isinstance(r.get("acara_sml_id"),
bool)]
        if len(records) == 0 or len(ids) != 1:
            return [], _school_match_clarification(question_id, mention, 
len(records), ids)
        school_id = ids[0]
        if school_id not in seen:
            seen.add(school_id)
            staged.append(school_id)
    return staged, None
```

_school_match_clarification:

```python
def _school_match_clarification(question_id, mention, record_count, ids) -> 
ClarificationNeeded:
    if record_count == 0:
        why = f"'{mention}' matched no school in the pinned directory."
        prompt = f"I couldn't find a school called '{mention}'. Could you check 
the name?"
    else:
        why = f"'{mention}' matched {record_count} schools in the pinned 
directory; the request needs exactly one."
        prompt = f"More than one school matches '{mention}'. Which one did you 
mean?"
    return ClarificationNeeded(
        question_id=question_id,
        missing=[MissingInput(input="acara_sml_id", why_material=why, 
allowed_values=[])],
        prompt_to_user=prompt,
    )
```

Hmm, `len(ids) != 1` covers records non-dict / missing id. Note len(records) 
could be >0 but ids==0 → treat as no usable match → clarification (message says 
matched no school? records>0 → ambiguous message... minor). Let me branch: if 
not ids → treat as zero-match message. Fine-tune:

```python
if not ids:
    → zero-match clarification
elif len(ids) > 1:
    → ambiguous
```

And records without acara_sml_id or non-dict → filtered out → ids smaller → 
counts mismatch. Simpler: count = len(ids).

VC-POC-04 two-id check:

```python
    if "acara_sml_ids" in school_required:
        ids_in_request = request.get("acara_sml_ids")
        if isinstance(ids_in_request, list):
            distinct = _dedupe([i for i in ids_in_request if isinstance(i, int) 
and not isinstance(i, bool)])
            if len(distinct) < 2 and _requires_two_ids(question_id):
                return ClarificationNeeded(... "at least two distinct schools" 
...)
```

_requires_two_ids: registry constraint "acara_sml_ids contains at least two 
values" in policy constraints:

```python
def _requires_two_ids(entry) -> bool:
    constraints = entry.get("input_policy", {}).get("constraints", [])
    return any("at least two" in str(c) for c in constraints)
```

Generalize: constraint string contains "at least two values". Fine — keyed to 
pinned string.

Clarification message: MissingInput(input="acara_sml_ids", why_material="A 
head-to-head comparison needs at least two distinct schools.", 
allowed_values=[]).

Selector bag normalization:

```python
    route = _selector_route(question_id)
    if intent.requested_metrics and route is None:
        return ClarificationNeeded(... labels cannot be used for this question 
...)
    if intent.requested_metrics and route is not None:
        field, vocab = route
        normalized: list[str] = []
        for label in intent.requested_metrics:
            canonical = _canonical_for(label, vocab)
            if canonical is None:
                return ClarificationNeeded(
                    question_id=question_id,
                    missing=[MissingInput(
                        input=field,
                        why_material=f"'{label}' is not an allowed {field} value
for this question.",
                        allowed_values=vocab,
                    )],
                    prompt_to_user=...,
                )
            normalized.append(canonical)
        normalized = _dedupe(normalized)
        if field == "metrics":
            scope_problem = request_scope_problem(deps.user_question, 
question_id, {**request, "metrics": normalized})
            if scope_problem:
                cleaned = [m for m in normalized if not 
(m.endswith("_headcount") or m.startswith("non_teaching_"))]
                normalized = cleaned or normalized
        if normalized:
            request[field] = normalized
```

Wait — "omit selector fields when the user named none": if requested_metrics 
empty → don't set request[field]. And if all normalized → set. If normalized 
empty after cleanup... `cleaned or normalized` keeps original (existing 
semantics). OK.

Hmm — scope check call: request_scope_problem(user_question, question_id, 
request_with_metrics). I pass a merged dict. Fine.

Constraint enforcement (09/10) — before validation, after selection:

```python
    constraint_failure = _explicit_filter_constraint(question_id, entry, 
request)
    if constraint_failure is not None:
        return constraint_failure
```

```python
def _explicit_filter_constraint(question_id, entry, request) -> 
ClarificationNeeded | None:
    policy = entry.get("input_policy", {})
    constraints = [str(c) for c in policy.get("constraints", [])]
    for field, label in (("peer_filters", "peer group"), ("segment_filters", 
"segment")):
        if field not in policy.get("required", []):
            continue
        filters = request.get(field)
        if not isinstance(filters, dict) or not filters:
            # absent/empty — but missing-derivation will catch it... 
```

Hmm wait — ordering: derivation for missing fields happens... if peer_filters 
absent, BOTH the constraint and the derivation flag it. Which runs first? If 
constraint runs first and returns "absent → clarify", the message is the 
constraint one ("needs an explicit filter") — fine either way. But I planned 
derivation first? Let me set the order: derivation first (policy-order 
required/material/conditional), THEN constraint (for present-but-default-only). 
If absent → derivation clarifies listing peer_filters as missing. If present but
default-only → constraint clarifies. So constraint check: only when isinstance 
dict:

```python
        if field not in request or not isinstance(request[field], dict):
            continue  # derivation handles absence
        filters = request[field]
        if not filters:
            continue  # empty dict → schema minProperties → hmm
```

Wait — empty dict {}: schema minProperties≥1 → TerminalError. But 
failure_behaviour: "VC-POC-09 or VC-POC-10 lacks an explicit nondefault 
cohort-defining filter → ClarificationNeeded". Empty dict = lacks → clarify. So 
constraint should treat empty dict as clarify. And absent → derivation clarify 
(both clarify; consistent).

Default-only peer check:

```python
        if field == "peer_filters":
            keys = set(filters)
            if keys == {"sectors"}:
                sectors = filters.get("sectors")
                default = _peer_sector_default(entry)
                if isinstance(sectors, list) and sorted(map(str, sectors)) == 
sorted(map(str, default)):
                    return ClarificationNeeded(... "sector default alone does 
not define a peer group" ...)
```

_peer_sector_default: from policy defaultable "peer_filters.sectors":

```python
default = entry.get("input_policy", {}).get("defaultable", 
{}).get("peer_filters.sectors")
```

Registry: defaultable {"peer_filters.sectors": ["Catholic", "Independent"]}. 

For segment: no default; any non-empty explicit dict passes to schema. But what 
about segment_filters containing ONLY defaultable keys? No defaultable filters 
for VC-POC-10 (only year_start/end which aren't cohort filters). So non-empty 
segment_filters → explicit. OK.

What about peer_filters containing sectors(default) PLUS enrolments_min → keys 
!= {"sectors"} → pass. Good.

Derivation:

```python
def _derive_missing(question_id, entry, request) -> list[str]:
    policy = entry.get("input_policy", {})
    missing: list[str] = []
    seen: set[str] = set()

    def consider(field: str) -> None:
        if field in seen:
            return
        seen.add(field)
        if field not in request:
            missing.append(field)

    for field in policy.get("required", []):
        consider(field)
    for field in policy.get("material", []):
        consider(field)
    conditional = policy.get("conditional_material", {})
    if isinstance(conditional, dict):
        for field, condition in conditional.items():
            if not isinstance(field, str):
                continue
            condition_text = str(condition)
            if field == "distance_km" and "market_definition" in condition_text 
and "distance" in condition_text:
                if request.get("market_definition") == "distance" and 
"distance_km" not in request:
                    consider(field)
            else:
                raise TerminalError(
                    f"{question_id}: unsupported conditional-material expression
{condition_text!r} — contract drift"
                )
    return missing
```

Hmm — the known-condition check: registry condition text is "market_definition 
equals distance" (v0.3.0 registry) — plan says condition is `market_definition 
== "distance"`. My check: "market_definition" in text and "distance" in text. 
The exact pinned declaration — v0.3.0 registry uses "market_definition equals 
distance". Check contains both tokens → matches. If a future condition 
("market_definition equals postcode") appeared → also contains both tokens → 
misinterpreted! "an unknown conditional expression is contract drift and raises 
TerminalError". Hmm. Stricter: match the exact known variants: normalized text 
equals "market_definition equals distance" OR "market_definition == distance". 
Let me normalize the condition text (collapse separators) and compare to known 
set {"market definition equals distance", "market_definition == distance"}:

```python
normalized_condition = _SEPARATOR_RE.sub(" ", condition_text.casefold()).strip()
if field == "distance_km" and normalized_condition in ("market definition equals
distance", "market definition == distance"):
```

"market_definition == distance" normalized → "market definition distance"... 
wait `==` → separator → space: "market definition distance". Hmm: 
"market_definition == distance" → casefold → "market_definition == distance" → 
sub → "market definition distance". And "market_definition equals distance" → 
"market definition equals distance". Two different forms. I'll accept both 
{"market definition equals distance", "market definition distance"}. Any other 
text → TerminalError. Good — exact pinned declarations only.

Also for VC-POC-05/06/07 the distance check requires market_definition == 
"distance" IN THE REQUEST (normalized earlier). If market_definition absent 
entirely, distance_km is not yet applicable (derivation flags market_definition 
as required-missing first). Correct.

Then:

```python
    missing = _derive_missing(...)
    if missing:
        missing_items = [
            MissingInput(
                input=field,
                why_material=_material_why(question_id, field),
                allowed_values=(_static_vocabulary_for_field(question_id, field)
or []),
            )
            for field in missing
        ]
        prompt = "To answer this question I need " + ", ".join(f"the {f}" for f 
in missing) + "."
        return ClarificationNeeded(question_id=question_id, 
missing=missing_items, prompt_to_user=prompt)
```

allowed_values: "populated static allowed values where the pinned catalog 
mapping supplies them" — market_definitions → yes; grades → yes; peer_filters → 
the mapping maps "VC-POC-09.peer_filters.<keys>" → key names... 
`_static_vocabulary_for_field("VC-POC-09", "peer_filters")` → mapping has 
"VC-POC-09.peer_filters.<keys>" not "VC-POC-09.peer_filters" → None → []. Good 
(keys aren't values). sa2_codes → release? "VC-POC-12.sa2_codes"? Not in mapping
→ []. Fine.

_material_why — keep existing helper.

Fetch & branch:

```python
    try:
        envelope = await client.fetch_evidence(question_id, request)
    except EvidenceError as exc:
        deps.tool_calls.append(ToolInvocation("fetch_evidence", arguments, 
error_kind=exc.error_kind))
        return _branch_evidence_error(question_id, exc, identity_from_caller)
    except Exception:
        deps.tool_calls.append(ToolInvocation("fetch_evidence", arguments, 
error_kind="operational"))
        raise
    deps.tool_calls.append(ToolInvocation("fetch_evidence", arguments, 
envelope))
    deps.last_envelope = envelope
    return envelope
```

_branch_evidence_error:

```python
def _branch_evidence_error(question_id, exc, identity_from_caller) -> 
Unsupported | ClarificationNeeded:
    kind = exc.error_kind
    if kind == "excluded_question": ... (as before)
    if kind == "unknown_question": ... (as before)
    if kind == "unanswerable": ... (as before)
    if kind == "bad_request":
        if identity_from_caller:
            return ClarificationNeeded(
                question_id=question_id,
                missing=[MissingInput(
                    input="acara_sml_id",
                    why_material="The data layer rejected the supplied school 
identity. Confirm the school's current public ACARA_SML_ID.",
                    allowed_values=[],
                )],
                prompt_to_user="The school identity supplied for this question 
was rejected by the data layer. Could you confirm the school?",
            )
        raise TerminalError(
            f"{question_id}: the evidence emitter rejected a schema-valid 
deterministic request "
            f"(bad_request): {exc.message} — planner/contract drift"
        )
    raise TerminalError(f"evidence emitter reported {kind}: {exc.message}")
```

Wait — excluded_question branch: current code has reason_code fallback 
contract.reason_code_for(question_id). Keep.

But hold on — the old `_branch_evidence_error` signature is (question_id, exc) 
returning Unsupported. New signature adds identity flag. Update calls & tests.

Also — data_unavailable/internal → TerminalError (unchanged).

Now — the old behavior test `test_bad_request_returns_unsupported` gets replaced
by the two new tests.

Also `test_error_call_recorded_in_deps` — VC-POC-01 explicit id 40387, 
bad_request → now returns ClarificationNeeded (identity), still records 
tool_calls with error_kind bad_request. The test asserts 
fetch_calls[0].error_kind == "bad_request" — still passes; it doesn't assert 
Unsupported. Good — keep it, maybe adjust name/comment.

Now — IMPORTANT check: existing intent-boundary test 
`test_ask_pipeline_intent_override_bypasses_model` — VC-POC-01, filters 
{"acara_sml_id": 40387}, metrics ["total_enrolments","teaching_fte"] → route 
metrics (profile) → normalized (already canonical) → request 
{"acara_sml_id":40387,"metrics":[...]} → validate against pinned VC-POC-01 
schema → OK → stub fetch. Stub case selection: `select_case(bundle, "VC-POC-01",
request)` — stub fixture canonical case probably has request {"acara_sml_id": 
40387, "metrics": ["total_enrolments", "teaching_fte"]} — matching semantics 
unknown but golden expected_request matches exactly this. Should match. It 
passed before with the same request (old planner produced same). Good.

`test_rendered_answer_always_uses_deterministic_data_status_note` — VC-POC-01 
filters {"acara_sml_id": 99999}, no metrics, stub case VC-POC-01/empty-series → 
old request {"acara_sml_id": 99999}; new planner: same (no metrics requested; 
metrics omitted). Validation OK. Stub matches empty-series fixture. Good.

`test_narration_with_invented_number_raises_typed_validation_error` — stub 
default case selection for VC-POC-01 with metrics [total_enrolments, 
teaching_fte] — same as before. Good.

Now — `test_ask_pipeline_missing_material_input_returns_clarification`: 
VC-POC-05, missing_material_inputs=["market_definition"], no filters/mentions → 
new derivation: required [acara_sml_id, market_definition] both missing → 
ClarificationNeeded with both. Test asserts "market_definition" in missing 
names. Passes.

Wait — order: [acara_sml_id, market_definition]. missing_names includes 
market_definition. Good.

`test_ask_pipeline_intent_override_excluded_topic_returns_unsupported` — 
unchanged path. `...unknown...`, `...excluded_question_id...`, `...gated...` — 
unchanged.

Now legacy-path tests in test_run_cli (clarify round-trip etc.) use 
build_agent(legacy) — converse(agent,...) → unchanged.

`answered()` in test_run_cli: legacy path via ask(agent, deps, ...) — unchanged.

### test_run_cli additions

`test_main_uses_pipeline_path_by_default` as sketched. Also maybe assert 
converse called with record_usage default? Keep focused.

Need imports: pytest (already), Unsupported (already imported). monkeypatch 
fixtures.

One catch: `cli.main` calls `configure_logging()` — fine. `args.question` = "q".
build_parser parses ["--stub", "q"]. build_client monkeypatched. 
require_credentials monkeypatched.

But wait — main references `converse` as a global in run module: `converse(None,
deps, ...)`. monkeypatch cli.converse → replaced. My fake converse signature: 
(agent, deps, question, ask_user=None, max_clarifications=1, record_usage=True) 
— called as converse(None, deps, args.question, ask_user=ask_user, 
max_clarifications=...). Fake: `async def fake_converse(agent, deps, question, 
**kwargs)`. Returns Unsupported → main: `EXIT_BY_BRANCH[type(output)]` → 
Unsupported → EXIT_UNSUPPORTED (3). json flag absent → render_text(Unsupported) 
prints — fine.

Also assert "build_agent not called": monkeypatch cli.build_agent to raise 
AssertionError. But after my run.py edit, main doesn't reference build_agent at 
all — the monkeypatch would then be setting an attribute that... 
monkeypatch.setattr(cli, "build_agent", raiser) requires the attribute to exist!
If I remove the import from run.py, setattr fails with AttributeError. Hmm. 
Options: keep the import (unused) or don't monkeypatch build_agent. The 
acceptance test must prove "the default CLI does not build or pass a legacy 
agent". Proving agent is None in converse args + not calling build_agent... If 
run.py no longer imports build_agent, "does not build" is structural. But the 
test name is fixed: test_main_uses_pipeline_path_by_default — assert 
calls["agent"] is None suffices for "passes no legacy agent". For "does not 
build": I could monkeypatch harness.agent.build_agent (the source) — main 
doesn't call it anyway. But to make the test meaningful, monkeypatch the source 
`harness.agent.build_agent` with a raiser and call cli.main — if run.py kept 
importing it, monkeypatching harness.agent.build_agent also affects run.py's 
imported reference? No — `from harness.agent import build_agent` binds at import
time; monkeypatching harness.agent.build_agent later does NOT change run.py's 
bound name. So to guard, run.py should call `harness.agent.build_agent`... 

Cleanest: keep run.py importing build_agent but never call it? Unused import is 
ugly and lint-failing possibly. Alternative: in run.py keep NO build_agent 
import; the test asserts main's converse receives None (the wiring proof) — 
that's what the plan asks: "assert the default CLI does not build or pass a 
legacy agent and reaches the pipeline signature." Reaching "the pipeline 
signature" = converse called with agent=None. I'll ALSO assert structurally that
main's module no longer references build_agent? That's what the existing auth 
test does with source text checks 
(test_anthropic_credentials_use_the_sdks_own_resolver reads run.py source). I'll
do the same: read run.py source, assert "build_agent(" not in it. That's robust 
and in the file's existing style.

So test:

```python
def test_main_uses_pipeline_path_by_default(monkeypatch):
    calls: dict[str, Any] = {}

    async def fake_converse(agent, deps, question, **kwargs):
        calls["agent"] = agent
        calls["question"] = question
        return Unsupported(reason_kind="not_in_registry", reason="no such 
question")

    monkeypatch.setattr(cli, "converse", fake_converse)
    monkeypatch.setattr(cli, "require_credentials", lambda: None)
    monkeypatch.setattr(cli, "build_client", lambda use_stub: stub_client())
    code = cli.main(["--stub", "Which schools are near Alpha College?"])
    assert code == cli.EXIT_UNSUPPORTED
    assert calls["agent"] is None
    assert calls["question"] == "Which schools are near Alpha College?"


def test_main_no_longer_builds_a_legacy_agent():
    source = (PROJECT_ROOT / "run.py").read_text("utf-8")
    assert "build_agent" not in source
```

PROJECT_ROOT = Path(__file__).resolve().parents[2] — wait, 
tests/unit/test_run_cli.py → parents: [0]=tests/unit, [1]=tests, [2]=repo root. 
In the given file: `PROJECT_ROOT = Path(__file__).resolve().parents[2]` — hmm, 
file is tests/unit/test_run_cli.py; parents[0] = tests/unit dir... resolve() 
gives /repo/tests/unit/test_run_cli.py; .parents[0]=/repo/tests/unit, 
[1]=/repo/tests, [2]=/repo. Yes PROJECT_ROOT=repo root. Good.

Wait — hmm, careful: the test file header says `PROJECT_ROOT = 
Path(__file__).resolve().parents[2]` and STUB = 
PROJECT_ROOT/"evals"/"stub_emitter.py" — consistent with repo root. Good.

monkeypatch cli.require_credentials and cli.build_client — main calls them as 
module globals. Good.

But note: main calls `configure_logging()` — fine.

fake_converse monkeypatched onto cli — main calls converse(...) → resolved via 
module globals → fake. 

Also — should I assert deps passed? calls["deps"]... fine, minor.

### test_intent_boundary additions

`test_planner_failure_is_not_unsupported`:

```python
def test_planner_failure_is_not_unsupported():
    """An unknown request property is a planner defect: TerminalError propagates
    out of ask() instead of becoming Unsupported."""
    from harness.errors import TerminalError

    intent = StructuredIntent(
        question_id="VC-POC-01",
        filters={"acara_sml_id": 40387, "banana": "not a request field"},
    )
    narration_called = []
    deps = Deps(client=stub_client())
    with pytest.raises(TerminalError):
        asyncio.run(
            ask(
                deps,
                "q",
                intent_override=intent,
                narration_model=make_narration_model("should never be reached"),
            )
        )
```

narration_called tracking — simpler to just pass narration_model that would fail
grounding if used? If TerminalError raised, narration never runs. To prove 
narration bypass, track calls: make_narration_model doesn't expose tracking. 
Write custom:

```python
def test_planner_failure_is_not_unsupported():
    from harness.errors import TerminalError

    intent = StructuredIntent(
        question_id="VC-POC-01",
        filters={"acara_sml_id": 40387, "not_a_request_field": 1},
    )
    deps = Deps(client=stub_client())
    with pytest.raises(TerminalError):
        asyncio.run(ask(deps, "q", intent_override=intent))
    assert deps.last_envelope is None  # nothing fetched
```

Wait — does ask() store deps.last_envelope? planner sets it on success; on 
failure stays None. Deps initial last_envelope=None presumably. OK. Hmm — but 
actually careful: does ask() re-raise TerminalError or catch? I reasoned 
propagate. There's risk ask() wraps. Let me look at the ask signature summary 
again: `async def ask(agent_or_deps, deps_or_question=None, 
question_or_record_usage=None, message_history=None, record_usage=True, *, 
intent_override=None, narration_override=None, ...)`. No info on exception 
handling. The docstring in planner says TerminalError is raised for operational 
stops — and existing planner test tests data_unavailable → TerminalError 
directly at planner level. For ask-level, the acceptance demands 
`test_planner_failure_is_not_unsupported` in test_intent_boundary. The plan: 
"planner/schema defects propagate as `TerminalError`" via ask(). I'll write 
pytest.raises(TerminalError). If ask() swallowed it, I'd have to adjust — can't 
verify by running. Proceed per plan.

Also add: clarification with narration absent — planner clarification bypasses 
narration:

```python
def test_planner_clarification_bypasses_narration_and_answer_view():
    intent = StructuredIntent(question_id="VC-POC-08", filters={"acara_sml_id": 
40387})
    deps = Deps(client=stub_client())
    output = asyncio.run(ask(deps, "q", intent_override=intent))
    assert isinstance(output, ClarificationNeeded)
    assert deps.answer_view is None
    missing_names = [m.input for m in output.missing]
    assert "distance_km" in missing_names
```

Hmm wait — deps.answer_view default None? Deps dataclass — answer_view attribute
referenced in test_ask_pipeline_stores_answer_view_on_deps (after success). 
Default presumably None. Assert `deps.answer_view is None`. Risk: default might 
be None — very likely. Keep.

Wait, VC-POC-08 with stub client — clarification happens pre-dispatch, so stub 
never called. Good. And "no model calls" — narration_model absent... ask() with 
intent_override but no narration: on clarification path, narration isn't reached
(planner returns before narration stage). If ask() tried to build a narration 
model when narration_override absent... only after planner success presumably. 
The existing test test_ask_pipeline_missing_material_input_returns_clarification
passes narration absent — proof clarification path doesn't need narration. Good.

And normalized schema-valid requests reach evidence:

```python
def test_normalized_selector_request_reaches_evidence():
    intent = StructuredIntent(
        question_id="VC-POC-01",
        filters={"acara_sml_id": 40387},
        requested_metrics=["Total Enrolments", "teaching staff"],
    )
    narration = NarrationOutput(answer="Enrolments run 612 in 2019 and 668 in 
2023.")
    deps = Deps(client=stub_client())
    output = asyncio.run(ask(deps, "q", intent_override=intent, 
narration_override=narration))
    assert isinstance(output, RenderedAnswer)
    fetch_calls = [c for c in deps.tool_calls if c.tool == "fetch_evidence"]
    assert fetch_calls, "evidence must be dispatched"
    assert fetch_calls[0].args == {
        "question_id": "VC-POC-01",
        "request": {"acara_sml_id": 40387, "metrics": ["total_enrolments", 
"teaching_fte"]},
    }
```

ToolInvocation fields: ToolCall has .tool and .args? From evals/schema.py 
ToolCall — but deps.tool_calls holds harness.tools.ToolInvocation with fields...
From planner: ToolInvocation("fetch_evidence", arguments, envelope). Field names
— dataclass `ToolInvocation` in tools.py; probably (tool, args, result=None, 
error_kind=None). The narration grounding: "Enrolments run 612 in 2019 and 668 
in 2023" — numbers must trace to the stub envelope (existing tests use this 
exact prose — VC-POC-01 canonical fixture has 612/2019/668 presumably). Since 
the existing test test_ask_pipeline_intent_override_bypasses_model uses the same
narration and passes today with request {"acara_sml_id": 40387, "metrics": 
["total_enrolments", "teaching_fte"]}, stub case matching works with that 
request.

Hmm — wait, one wrinkle: the stub `select_case` matches request — if the 
canonical fixture request is exactly {"acara_sml_id": 40387, "metrics": 
["total_enrolments","teaching_fte"]}, good. Existing pipeline tests already send
exactly that (old planner: request = dict(filters) + setdefault metrics from 
requested_metrics) — identical. New planner sends identical. 

And caller-correctable identity clarification via ask():

```python
def test_invalid_explicit_identity_stays_a_clarification():
    class _BadIdentityClient:
        async def school_directory(self, search=None, limit=25):
            return []
        async def fetch_evidence(self, question_id, request):
            raise EvidenceError("bad_request", "acara_sml_id 99999 absent from 
the pinned school directory")
        async def input_catalog(self):
            return {}

    from harness.evidence import EvidenceError
    intent = StructuredIntent(question_id="VC-POC-01", filters={"acara_sml_id": 
99999})
    deps = Deps(client=_BadIdentityClient())
    output = asyncio.run(ask(deps, "q", intent_override=intent))
    assert isinstance(output, ClarificationNeeded)
    assert not isinstance(output, Unsupported)
```

Wait — Deps(client=...) — does Deps validate client type? dataclass, probably 
not. But ask() might use deps.client for other stages? Stage 1 intent model — no
client. OK.

Hmm — Deps might require client keyword only. Existing: 
Deps(client=stub_client()). Fine.

Also — unnormalizable metric stays clarification:

```python
def test_unnormalizable_metric_stays_a_clarification():
    intent = StructuredIntent(
        question_id="VC-POC-01",
        filters={"acara_sml_id": 40387},
        requested_metrics=["banana enrolments"],
    )
    deps = Deps(client=stub_client())
    output = asyncio.run(ask(deps, "q", intent_override=intent))
    assert isinstance(output, ClarificationNeeded)
    assert "banana" in output.missing[0].why_material
    assert output.missing[0].allowed_values  # static vocabulary supplied
```

Need EvidenceError import.

Careful — stub_client unused in these clarification tests; that's fine (no 
dispatch). For identity test I used custom client. OK.

### test_planner.py rewrite

Keep existing passing tests; update changed behaviors; add the named acceptance 
tests. Let me enumerate the final test list:

Existing kept (verify compatibility):
- test_excluded_topic_returns_unsupported ✓
- test_excluded_topic_reason_from_contract ✓
- test_excluded_topic_finance_hint ✓
- test_excluded_topic_without_hint_uses_fallback ✓
- test_excluded_question_id_returns_unsupported ✓
- test_excluded_question_reason_is_contract_message ✓
- test_gated_question_returns_unsupported ✓
- test_gated_question_reason_code_preserved ✓
- test_unknown_question_id_literal_returns_unsupported ✓
- test_arbitrary_unknown_question_id_returns_unsupported ✓
- test_missing_material_input_returns_clarification ✓ (derivation covers)
- test_non_material_missing_input_is_ignored ✓ (model claim "fiscal_year" 
ignored — derivation sees all present → proceeds → fetch with _HappyClient... 
wait — VC-POC-01 requires acara_sml_id; filters has it. metrics? none requested 
→ omitted. Validation OK. Fetch returns envelope. assert not ClarificationNeeded
✓)
- test_happy_path_returns_envelope ✓ — request {"acara_sml_id":40387}; 
validation passes. deps.last_envelope is result ✓.
- test_happy_path_records_tool_calls ✓
- test_school_resolution_records_search ✓ — mention "Some School" → _HappyClient
returns one record {"acara_sml_id": 40387, "school_name": search} → exactly one 
→ staged [40387] → committed. searched True. request acara_sml_id=40387. 
Validation OK. fetch recorded. ✓
- test_unanswerable_returns_unsupported ✓
- test_unknown_question_from_emitter_returns_unsupported ✓
- test_data_unavailable_raises_terminal_error ✓
- test_internal_error_raises_terminal_error ✓
- test_error_call_recorded_in_deps ✓ (bad_request now clarifies, but recording 
assertion unchanged)
- _find_exclusion_text tests ✓

Changed/replaced:
- test_bad_request_returns_unsupported → replace with 
test_invalid_explicit_identity_bad_request_clarifies + 
test_schema_valid_emitter_bad_request_is_terminal.

New acceptance tests (exact names):
1. test_missing_market_definition_is_derived_without_model_hint
2. test_distance_is_conditional_on_distance_market
3. test_ambiguous_school_clarifies_without_fetch
4. test_metric_normalization_and_exact_profile_request
5. test_vc_poc_03_routes_selector_to_ratios
6. test_unknown_request_property_fails_before_fetch
7. test_peer_filters_require_explicit_nondefault_filter
8. test_segment_filters_reject_unknown_nested_key
9. test_schema_valid_emitter_bad_request_is_terminal
10. test_invalid_explicit_identity_bad_request_clarifies

Plus supporting: zero-match, transactional, conflict, dedup order, policy order,
vc-poc-04 insufficient ids, terminal on schema violation etc.

Mock clients needed:
- _DirectoryClient(directory: dict[str, list[dict] | Exception], envelope, 
fetch_error)

Let me design:

```python
class _DirectoryClient:
    """Directory-backed client: search → records; fetch → envelope or 
EvidenceError."""

    def __init__(self, directory=None, envelope=None, fetch_error=None):
        self.directory = directory or {}
        self._envelope = envelope or {...minimal...}
        self._fetch_error = fetch_error
        self.searches: list[str] = []
        self.fetched: list[dict] = []

    async def school_directory(self, search, limit=25):
        self.searches.append(search)
        records = self.directory.get(search)
        if isinstance(records, Exception):
            raise records
        return list(records or [])

    async def fetch_evidence(self, question_id, request):
        self.fetched.append({"question_id": question_id, "request": request})
        if self._fetch_error is not None:
            raise self._fetch_error
        return self._envelope
```

Minimal envelope default same as _HappyClient's.

Now the tests:

```python
def test_missing_market_definition_is_derived_without_model_hint():
    intent = StructuredIntent(question_id="VC-POC-05", school_mentions=[])
    client = _DirectoryClient()
    deps = Deps(client=client)
    result = asyncio.run(plan_and_execute(intent, client, deps))
    assert isinstance(result, ClarificationNeeded)
    names = [m.input for m in result.missing]
    assert names == ["acara_sml_id", "market_definition"]  # pinned policy order
    assert result.missing[1].allowed_values == ["distance", "lga", "postcode", 
"sa2"]
    assert client.fetched == []  # no evidence call
```

Wait — allowed_values for market_definition: 
`_static_vocabulary_for_field("VC-POC-05", "market_definition")` → mapping 
"VC-POC-05.market_definition" → "market_definitions" → catalog enum 
["distance","lga","postcode","sa2"]. Yes.

And acara_sml_id allowed_values [].

```python
def test_distance_is_conditional_on_distance_market():
    # SA2 market: no radius required, evidence proceeds.
    intent = StructuredIntent(
        question_id="VC-POC-05",
        filters={"acara_sml_id": 40387, "market_definition": "sa2"},
    )
    client = _DirectoryClient()
    deps = Deps(client=client)
    result = asyncio.run(plan_and_execute(intent, client, deps))
    assert isinstance(result, dict)
    assert client.fetched[0]["request"] == {"acara_sml_id": 40387, 
"market_definition": "sa2"}

    # Distance market without a radius: clarification names distance_km.
    intent = StructuredIntent(
        question_id="VC-POC-05",
        filters={"acara_sml_id": 40387, "market_definition": "distance"},
    )
    client = _DirectoryClient()
    deps = Deps(client=client)
    result = asyncio.run(plan_and_execute(intent, client, deps))
    assert isinstance(result, ClarificationNeeded)
    assert [m.input for m in result.missing] == ["distance_km"]
    assert client.fetched == []
```

Check: SA2 path — request {"acara_sml_id": 40387, "market_definition": "sa2"}; 
validate against pinned VC-POC-05 schema. I can't see it, but required 
acara_sml_id+market_definition; distance_km conditional. Should validate. Risk: 
schema might require... let me think about what VC-POC-05 request schema 
contains: acara_sml_id (required), market_definition (enum 
distance/lga/postcode/sa2), distance_km (number>0, conditional required). 
Request validates. OK.

```python
def test_ambiguous_school_clarifies_without_fetch():
    directory = {
        "St Albans": [
            {"acara_sml_id": 40387, "school_name": "St Albans Primary"},
            {"acara_sml_id": 40392, "school_name": "St Albans Secondary"},
        ]
    }
    intent = StructuredIntent(question_id="VC-POC-01", school_mentions=["St 
Albans"])
    client = _DirectoryClient(directory=directory)
    deps = Deps(client=client)
    result = asyncio.run(plan_and_execute(intent, client, deps))
    assert isinstance(result, ClarificationNeeded)
    assert client.fetched == []                     # no evidence call
    assert deps.resolved_school_ids == []           # no partial commit
    assert deps.searched is False
    preflights = [c for c in deps.tool_calls if c.tool == "search_schools"]
    assert len(preflights) == 1                     # preflight recorded
```

Also zero-match variant:

```python
def test_unmatched_school_clarifies_without_fetch():
    intent = StructuredIntent(question_id="VC-POC-01", school_mentions=["Nowhere
College"])
    client = _DirectoryClient(directory={})
    deps = Deps(client=client)
    result = asyncio.run(plan_and_execute(intent, client, deps))
    assert isinstance(result, ClarificationNeeded)
    assert client.fetched == []
    assert deps.resolved_school_ids == []
```

Transactional:

```python
def test_school_resolution_is_transactional():
    directory = {
        "Alpha College": [{"acara_sml_id": 40387, "school_name": "Alpha 
College"}],
    }
    intent = StructuredIntent(question_id="VC-POC-01", school_mentions=["Alpha 
College", "Missing College"])
    client = _DirectoryClient(directory=directory)
    deps = Deps(client=client)
    result = asyncio.run(plan_and_execute(intent, client, deps))
    assert isinstance(result, ClarificationNeeded)
    assert deps.resolved_school_ids == []   # first mention did not leak
    assert deps.searched is False
    assert client.fetched == []
    preflights = [c for c in deps.tool_calls if c.tool == "search_schools"]
    assert len(preflights) == 2
```

Conflict:

```python
def test_conflicting_explicit_id_and_mention_is_terminal():
    directory = {"Beta Grammar": [{"acara_sml_id": 40392, "school_name": "Beta 
Grammar"}]}
    intent = StructuredIntent(
        question_id="VC-POC-01",
        school_mentions=["Beta Grammar"],
        filters={"acara_sml_id": 40387},
    )
    client = _DirectoryClient(directory=directory)
    deps = Deps(client=client)
    with pytest.raises(TerminalError):
        asyncio.run(plan_and_execute(intent, client, deps))
    assert deps.resolved_school_ids == []
    assert client.fetched == []
```

Consistent explicit+mention:

```python
def test_consistent_explicit_id_and_mention_proceeds():
    directory = {"Alpha College": [{"acara_sml_id": 40387, "school_name": "Alpha
College"}]}
    intent = StructuredIntent(
        question_id="VC-POC-01",
        school_mentions=["Alpha College"],
        filters={"acara_sml_id": 40387},
    )
    client = _DirectoryClient(directory=directory)
    deps = Deps(client=client)
    result = asyncio.run(plan_and_execute(intent, client, deps))
    assert isinstance(result, dict)
    assert client.fetched[0]["request"] == {"acara_sml_id": 40387}
```

Order + dedup:

```python
def test_mention_order_preserved_and_duplicates_removed():
    directory = {
        "Beta Grammar": [{"acara_sml_id": 40392, "school_name": "Beta 
Grammar"}],
        "Alpha College": [{"acara_sml_id": 40387, "school_name": "Alpha 
College"}],
    }
    intent = StructuredIntent(
        question_id="VC-POC-03",
        school_mentions=["Beta Grammar", "Alpha College", "Beta Grammar"],
    )
    client = _DirectoryClient(directory=directory)
    deps = Deps(client=client)
    result = asyncio.run(plan_and_execute(intent, client, deps))
    assert isinstance(result, dict)
    assert client.fetched[0]["request"] == {"acara_sml_ids": [40392, 40387]}
    assert deps.resolved_school_ids == [40392, 40387]
```

Wait — deps.resolved_school_ids commit: I set `deps.resolved_school_ids = 
staged`? Or extend existing? Existing might have prior IDs from earlier turns. 
Current planner appends without dup. I'll merge: start from existing 
deps.resolved_school_ids copy, append staged not already present. Hmm — "stage 
IDs locally in mention order, deduplicate without reordering, and commit to Deps
only after all mentions succeed." Commit semantics: I'll extend 
deps.resolved_school_ids with staged ids not already there (preserving prior). 
For the test, deps starts empty → [40392, 40387]. ✓.

VC-POC-04 insufficient:

```python
def test_comparison_with_one_distinct_school_clarifies():
    directory = {"Alpha College": [{"acara_sml_id": 40387, "school_name": "Alpha
College"}]}
    intent = StructuredIntent(
        question_id="VC-POC-04",
        school_mentions=["Alpha College", "Alpha College"],
    )
    client = _DirectoryClient(directory=directory)
    deps = Deps(client=client)
    result = asyncio.run(plan_and_execute(intent, client, deps))
    assert isinstance(result, ClarificationNeeded)
    assert client.fetched == []
```

Wait — VC-POC-04 required acara_sml_ids; mentions resolve both to 40387 → dedup 
[40387] → <2 → clarify. But wait — ordering: the two-id check happens after 
attachment. Dedup staged [40387] committed... hmm — if we commit [40387] to deps
then clarify, is that "partial mutation"? The resolution transaction itself 
succeeded (all mentions resolved). The id-count clarification is a separate 
later check. Commit of successfully resolved IDs is legitimate. But cleaner: 
check count BEFORE commit? The staged list is available; do the two-id check on 
staged before commit. Let me order: resolve → staged; if required-two and 
len(staged)<2 → clarify WITHOUT committing (deps untouched). Yes — safer for 
"retain no partial resolved-ID mutation". I'll do the two-id check pre-commit.

But explicit-IDs path: request acara_sml_ids=[40387] explicit → two-id check on 
request ids → clarify (deps untouched anyway).

So restructure: after resolution/adaptation, compute effective ids (explicit or 
staged); check two-id constraint on effective ids; if staged path and check 
passes → commit + attach.

Policy-order clarification test:

```python
def test_clarifications_follow_pinned_policy_order():
    directory = {"Alpha College": [{"acara_sml_id": 40387, "school_name": "Alpha
College"}]}
    intent = StructuredIntent(
        question_id="VC-POC-07",
        school_mentions=["Alpha College"],
        filters={"market_definition": "distance"},
    )
    client = _DirectoryClient(directory=directory)
    deps = Deps(client=client)
    result = asyncio.run(plan_and_execute(intent, client, deps))
    assert isinstance(result, ClarificationNeeded)
    names = [m.input for m in result.missing]
    assert names == ["grades", "distance_km"]   # required first, then 
conditional material
```

VC-POC-07 policy: required [acara_sml_id, market_definition, grades]; material 
[market_definition, grades]; conditional distance_km. With 
acara+market_definition satisfied: missing = grades (required pass), then 
material pass adds grades? already seen → skip. conditional distance_km → added.
Order [grades, distance_km]. ✓

Normalization + exact profile request:

```python
def test_metric_normalization_and_exact_profile_request():
    intent = StructuredIntent(
        question_id="VC-POC-01",
        filters={"acara_sml_id": 40387},
        requested_metrics=["Total Enrolments", "Teaching Staff", "teaching 
staff", "TOTAL_ENROLMENTS"],
    )
    client = _DirectoryClient()
    deps = Deps(client=client)
    result = asyncio.run(plan_and_execute(intent, client, deps))
    assert isinstance(result, dict)
    assert client.fetched[0]["request"] == {
        "acara_sml_id": 40387,
        "metrics": ["total_enrolments", "teaching_fte"],
    }
```

Check normalization: "Total Enrolments" → "total enrolments" ↔ vocab 
"total_enrolments"→"total enrolments" ✓ canonical "total_enrolments". "Teaching 
Staff" → "teaching staff" → alias → teaching_fte ✓ (dedupe with third). 
"TOTAL_ENROLMENTS" → "total enrolments" → duplicate → removed. Result 
["total_enrolments","teaching_fte"] ✓. No year fields, no headcount, no 
non-teaching ✓.

Scope rule check with deps.user_question="" → request_scope_problem("", 
"VC-POC-01", request) → unknown; if it returns None (no problem) → no cleanup. 
If it returns a problem (unlikely without keywords)... The existing planner 
already calls it with deps.user_question possibly "" and tests pass — presumably
returns falsy for empty question. Keep.

Hmm wait — one important check: does request_scope_problem maybe RAISE or behave
differently when metrics contains e.g. teaching_fte and question VC-POC-01 and 
empty user question? Current planner calls it unconditionally for VC-POC-01/04 
with metrics — tests pass today with same call shape. Keep.

Ratios routing:

```python
def test_vc_poc_03_routes_selector_to_ratios():
    intent = StructuredIntent(
        question_id="VC-POC-03",
        filters={"acara_sml_ids": [40387, 40392]},
        requested_metrics=["Students per Teaching FTE"],
    )
    client = _DirectoryClient()
    deps = Deps(client=client)
    result = asyncio.run(plan_and_execute(intent, client, deps))
    assert isinstance(result, dict)
    sent = client.fetched[0]["request"]
    assert sent == {"acara_sml_ids": [40387, 40392], "ratios": 
["students_per_teaching_fte"]}
    assert "metrics" not in sent
```

Unknown property:

```python
def test_unknown_request_property_fails_before_fetch():
    intent = StructuredIntent(
        question_id="VC-POC-01",
        filters={"acara_sml_id": 40387, "enrolments_forecast": True},
    )
    client = _DirectoryClient()
    deps = Deps(client=client)
    with pytest.raises(TerminalError):
        asyncio.run(plan_and_execute(intent, client, deps))
    assert client.fetched == []
```

Peer filters default-only:

```python
def test_peer_filters_require_explicit_nondefault_filter():
    intent = StructuredIntent(
        question_id="VC-POC-09",
        filters={"peer_filters": {"sectors": ["Catholic", "Independent"]}},
    )
    client = _DirectoryClient()
    deps = Deps(client=client)
    result = asyncio.run(plan_and_execute(intent, client, deps))
    assert isinstance(result, ClarificationNeeded)
    assert client.fetched == []
```

Hmm wait — VC-POC-09 policy required ["peer_filters"] — present → derivation 
satisfied. Constraint: keys=={"sectors"} and values == default → clarify. ✓. 
Also absent case:

```python
    intent = StructuredIntent(question_id="VC-POC-09")
    → derivation: peer_filters missing → clarify.
```

I'll add a peer-filters-absent assertion in the same test.

Explicit peer filter proceeds:

```python
def test_explicit_peer_filter_dispatches():
    intent = StructuredIntent(
        question_id="VC-POC-09",
        filters={"peer_filters": {"sectors": ["Catholic", "Independent"], 
"school_types": ["Secondary"]}},
    )
    → dispatch, request equals filters.
```

Wait — schema VC-POC-09: peer_filters object with additionalProperties false, 
properties enrolments_max/min (numbers), icsea_max/min, remoteness (array of 
strings? release-derived), reporting_structures, school_types (array of strings 
— release-derived values, shape only), sectors. {"sectors": [...], 
"school_types": ["Secondary"]} validates (type shape). OK.

Segment unknown nested key:

```python
def test_segment_filters_reject_unknown_nested_key():
    intent = StructuredIntent(
        question_id="VC-POC-10",
        filters={"segment_filters": {"sectors": ["Independent"], "banana": 
["x"]}},
    )
    client = _DirectoryClient()
    deps = Deps(client=client)
    with pytest.raises(TerminalError):
        asyncio.run(plan_and_execute(intent, client, deps))
    assert client.fetched == []
```

Constraint first: segment_filters non-empty explicit → pass; schema validation: 
unknown property "banana" inside segment_filters → TerminalError ✓.

Bad-request origin tests:

```python
def test_schema_valid_emitter_bad_request_is_terminal():
    directory = {"Alpha College": [{"acara_sml_id": 40387, "school_name": "Alpha
College"}]}
    intent = StructuredIntent(question_id="VC-POC-01", school_mentions=["Alpha 
College"])
    client = _DirectoryClient(directory=directory, 
fetch_error=EvidenceError("bad_request", "unexpected shape"))
    deps = Deps(client=client)
    with pytest.raises(TerminalError) as caught:
        asyncio.run(plan_and_execute(intent, client, deps))
    assert "VC-POC-01" in str(caught.value)
    fetch_calls = [c for c in deps.tool_calls if c.tool == "fetch_evidence"]
    assert len(fetch_calls) == 1 and fetch_calls[0].error_kind == "bad_request"
    assert deps.resolved_school_ids == [40387]  # resolution committed before 
the fetch
```

```python
def test_invalid_explicit_identity_bad_request_clarifies():
    intent = StructuredIntent(question_id="VC-POC-01", filters={"acara_sml_id": 
99999})
    client = _DirectoryClient(fetch_error=EvidenceError("bad_request", 
"acara_sml_id absent from the pinned directory"))
    deps = Deps(client=client)
    result = asyncio.run(plan_and_execute(intent, client, deps))
    assert isinstance(result, ClarificationNeeded)
    assert result.question_id == "VC-POC-01"
    fetch_calls = [c for c in deps.tool_calls if c.tool == "fetch_evidence"]
    assert fetch_calls[0].error_kind == "bad_request"
```

Keep old test names for other branches; delete 
test_bad_request_returns_unsupported (obsolete mapping).

Also — grade routing test (VC-POC-02):

```python
def test_grade_labels_route_to_grades_field():
    intent = StructuredIntent(
        question_id="VC-POC-02",
        filters={"acara_sml_id": 40387},
        requested_metrics=["Year 6"],
    )
    ...
    assert request == {"acara_sml_id": 40387, "grades": ["year_6"]}
```

Unnormalizable metric clarification:

```python
def test_unnormalizable_metric_clarifies_without_fetch():
    intent = StructuredIntent(
        question_id="VC-POC-01",
        filters={"acara_sml_id": 40387},
        requested_metrics=["banana enrolments"],
    )
    client = _DirectoryClient()
    deps = Deps(client=client)
    result = asyncio.run(plan_and_execute(intent, client, deps))
    assert isinstance(result, ClarificationNeeded)
    assert result.missing[0].allowed_values  # profile metrics vocabulary 
supplied
    assert client.fetched == []
```

And no-route question with requested metrics clarifies:

Hmm — VC-POC-05 with requested_metrics... policy required satisfied... route 
None → clarify. Add? Sure:

```python
def test_selector_labels_without_a_route_clarify():
    intent = StructuredIntent(
        question_id="VC-POC-05",
        filters={"acara_sml_id": 40387, "market_definition": "sa2"},
        requested_metrics=["total enrolments"],
    )
    → ClarificationNeeded, no fetch.
```

Wait — careful: derivation for VC-POC-05: required acara_sml_id ✓ 
market_definition ✓ → no missing. Route None + requested_metrics → clarify. ✓

Explicit-IDs-bypass preflight:

```python
def test_explicit_numeric_ids_skip_directory_preflight():
    intent = StructuredIntent(
        question_id="VC-POC-03",
        filters={"acara_sml_ids": [99999, 40387]},
    )
    client = _DirectoryClient()
    deps = Deps(client=client)
    result = asyncio.run(plan_and_execute(intent, client, deps))
    assert isinstance(result, dict)
    assert client.searches == []
    assert client.fetched[0]["request"] == {"acara_sml_ids": [99999, 40387]}
```

Now — one BIG validation concern: request {"acara_sml_ids": [99999, 40387]} 
against pinned VC-POC-03 schema — validates (integers ≥1, unique, ≤20). ✓

Another — VC-POC-09 filters-only test: derivation required peer_filters present 
✓; material peer_filters ✓ present. Wait — material check `field in request` — 
peer_filters is a dict in request → satisfied. ✓.

Now, schema validation for VC-POC-09 with peer_filters {"sectors": [...], 
"school_types": ["Secondary"]} — schema shape for school_types likely 
{"type":"array","items":{"type":"string"}} — validates ✓.

Let me also double check _selector_route for VC-POC-09: no metric_list; mapping 
keys with prefix "VC-POC-09." are "VC-POC-09.peer_filters.<keys>" 
(field="peer_filters.<keys>" — not in {metrics,ratios,grades}), 
"VC-POC-09.peer_filters.sectors" (field="peer_filters.sectors" — not in set), 
etc. → route None ✓.

For VC-POC-01: mapping "VC-POC-01.metrics" exists but metric_list handles it 
first → ("metrics", registry profile list) ✓. VC-POC-11: metric_list 
"student_context_metrics" ✓. VC-POC-02: no metric_list; mapping 
"VC-POC-02.grades" → "grades" → catalog enum ✓. VC-POC-03: mapping 
"VC-POC-03.ratios" → catalog enum ✓. VC-POC-07: "VC-POC-07.grades" ✓.

Edge — VC-POC-12: mapping "VC-POC-12.seifa_index_type" etc. — fields not in the 
set → route None ✓ (seifa scalars pass through filters).

Now — catalog enum for "market_definitions": 
properties.market_definitions.items.enum = ["distance","lga","postcode","sa2"] 
✓.

### Dedupe helper for ids

```python
def _dedupe_ints(values: list[int]) -> list[int]:
    seen = set(); out = []
    for v in values:
        if v not in seen: seen.add(v); out.append(v)
    return out
```

I have _dedupe for strings; make generic:

```python
def _dedupe(values):
    seen = set(); out = []
    for v in values:
        if v not in seen:
            seen.add(v); out.append(v)
    return out
```

Works for both.

### Commit semantics

```python
staged = [...]  # ordered unique
if two-id-required and len(staged) < 2: return clarification
# commit
for sid in staged:
    if sid not in deps.resolved_school_ids:
        deps.resolved_school_ids.append(sid)
deps.searched = True
```

### planner docstring updates

Update module docstring to describe new behavior. Keep the exclusion helpers 
unchanged.

### What about `intent.missing_material_inputs`?

Now ignored entirely. Should planner log? Keep simple; the docstring notes 
derivation supersedes it. But hmm — is there a case where the model claims 
missing input the derivation misses? E.g. 
intent.missing_material_inputs=["peer_filters"] for VC-POC-09 with filters 
containing default-only sectors → constraint clarifies anyway. OK.

Also for excluded/gated/unknown, missing_material irrelevant.

### Wait — one more thing on derivation: `zone_entity_code` for VC-POC-REF-02

_required includes zone_entity_code; absent → clarify. _material_why has 
explanation. ✓

### VC-POC-13 material sa2_codes

If absent → clarify ✓.

### VC-POC-12 required sa2_codes (material empty)

Absence → clarify via required ✓.

### VC-POC-14 gated — handled before ✓.

### REF-01 — no required inputs; request {} validates (schema type object, no 
required). ✓

### Now think about `_HappyClient` in old tests: school_directory returns 
[{"acara_sml_id": 40387, "school_name": search}] — with mention resolution 
requiring exactly one → 1 record ✓.

### Now — an important question: validation of requests against schemas uses 
lru_cache on question_id — fine.

### Check the `_ErrorClient` — VC-POC-05 unanswerable test: filters {}? Current:
`StructuredIntent(question_id="VC-POC-05", filters={"acara_sml_id": 99999})` — 
wait current test: `test_unanswerable_returns_unsupported` uses VC-POC-05 
filters {"acara_sml_id": 99999} — NEW flow: required acara_sml_id ✓, 
market_definition MISSING → clarify BEFORE fetch! The old test expected fetch → 
unanswerable → Unsupported. It will now return ClarificationNeeded instead! Must
update this test: use a question whose required inputs are complete, e.g. 
VC-POC-01 filters {"acara_sml_id": 40387} → fetch raises unanswerable → 
Unsupported(unanswerable). Change the test's intent to VC-POC-01. Also 
_ErrorClient needs school_directory (unused then). Update.

Similarly `test_data_unavailable_raises_terminal_error` and 
`test_internal_error_raises_terminal_error` use VC-POC-01 filters 
{"acara_sml_id": 40387} — complete → fetch → TerminalError ✓ unchanged.

`test_unknown_question_from_emitter_returns_unsupported` VC-POC-01 id 40387 ✓.

`test_bad_request_returns_unsupported` → replaced.

`test_error_call_recorded_in_deps`: VC-POC-01 id 40387, bad_request → clarify 
(identity) but recorded ✓ (assertions hold).

Also check `test_happy_path_records_tool_calls` — fetch once ✓.

Now — hold on, `test_non_material_missing_input_is_ignored` uses `_HappyClient` 
with VC-POC-01 missing_material_inputs ["fiscal_year"] — derivation ignores 
model claims; request has acara_sml_id; metrics: none. Validation of 
{"acara_sml_id": 40387} ✓ → fetch → envelope. assert not ClarificationNeeded ✓.

### Now, intent_boundary test `test_ask_pipeline_uses_scripted_intent_model` — 
intent from model: VC-POC-01 metrics [total_enrolments, teaching_fte] filters 
acara 40387 → dispatch ✓ RenderedAnswer with narration_override ✓.

`test_ask_pipeline_uses_scripted_narration_model` — same ✓.

`test_rendered_answer_always_uses_deterministic_data_status_note` — VC-POC-01 id
99999, no metrics → dispatch stub empty-series ✓. Wait — the stub case selection
with request {"acara_sml_id": 99999} matches empty-series fixture ✓ (as today).

Hmm — but WAIT: in this test the stub case 
STUB_EMITTER_CASE="VC-POC-01/empty-series" env pins the case — fine regardless.

### Now run.py — converse(None, ...): "Pipeline path: ask(deps, reply, 
record_usage=record_usage)" — message_history unused for pipeline (deps carries 
state via deps.last_messages? the planner path doesn't set last_messages...). 
Current code comment says pipeline path exists. Keep as-is.

One question — in converse, after clarification round: `history = 
deps.last_messages; reply = answer` — for pipeline path, ask(deps, reply) — does
deps carry last_messages for round 2? The pipeline presumably re-plans from 
scratch with deps state (resolved IDs etc.). The reply "10 km" becomes the new 
question for intent extraction... In pipeline path with Stage 1 model — intent 
from "10 km" — hmm, that's a pipeline design issue beyond scope. The plan only 
changes the default wiring. Keep converse as-is except it already supports None.
Actually — it ALREADY supports None! Great: `if agent is not None: ... else: # 
Pipeline path`. So run.py main just needs agent=None and no build_agent call.

### Now — does main still need `from harness.agent import ask, build_agent`? ask
is used by converse (legacy path) — keep ask. Remove build_agent import. But 
wait — test_run_cli's `answered()` imports build_agent from harness.agent 
directly — independent of run.py.

### Double-check: removing build_agent from run.py breaks nothing else — 
demo.py? demo builds its own agent (from summary, demo.py has select_steps; it 
likely imports build_agent itself). Not our concern.

### Now — CLI test file: keep all existing tests. The existing tests reference 
cli.EXIT_BY_BRANCH etc. — unchanged.

One existing test to re-check: `test_stub_mode_needs_no_data_project` — 
build_client unchanged ✓.

### Writing test_planner.py full content

I'll rewrite the file with all kept + new tests, updated imports (TerminalError,
EvidenceError already imported).

Structure the clients:

- stub_client (unchanged)
- _ErrorClient — keep, school_directory returns []
- _HappyClient — keep
- _DirectoryClient — new

Also note: _ErrorClient VC-POC-01 bad_request test removed → the two origin 
tests use _DirectoryClient.

Let me also reconsider `test_unanswerable_returns_unsupported` — change intent 
to VC-POC-01 filters acara_sml_id 40387:

```python
def test_unanswerable_returns_unsupported():
    """EvidenceError(unanswerable) → Unsupported(unanswerable)."""
    intent = StructuredIntent(question_id="VC-POC-01", filters={"acara_sml_id": 
40387})
    client = _ErrorClient("unanswerable", "no published SA2 centroid for this 
school")
    ...
```

OK.

### Now — deeper check of `_derive_missing` + `request` presence for school 
field when explicit ID given as string?

filters {"acara_sml_id": "40387"} — present → considered satisfied → later 
schema validation → type error → TerminalError. Hmm — a string ID is 
caller-correctable... but schema says integer. TerminalError per fail-closed. 
Fine.

### Wait — critical: `_adapt_explicit_school_fields` and `_explicit_id_values` 
use `request` — let me re-examine the flow order in plan_and_execute:

```python
    # after gates:
    entry = contract.questions()[question_id]  (TerminalError if missing)
    policy = entry.get("input_policy") or {}
    required_fields = [f for f in policy.get("required", []) if isinstance(f, 
str)]
    school_required = [f for f in required_fields if f in _SCHOOL_FIELDS]

    request = dict(intent.filters) if isinstance(intent.filters, dict) else {}

    clarify = _normalize_scalar_selectors(question_id, request)
    if clarify: return clarify

    _adapt_explicit_school_fields(request, school_required)  # may TerminalError

    mentions = [m for m in (intent.school_mentions or []) if isinstance(m, str) 
and m.strip()]
    explicit_fields_present = [f for f in school_required if f in request]
    identity_from_caller = bool(explicit_fields_present)

    if school_required and mentions:
        resolved_ids, failure = await _resolve_mentions(question_id, mentions, 
client, deps)
        if failure is not None:
            return failure
        if explicit_fields_present:
            explicit_ids = _explicit_id_values(request, school_required)
            if sorted(resolved_ids) != sorted(explicit_ids):
                raise TerminalError(
                    f"{question_id}: explicit school identities {explicit_ids} 
conflict with "
                    f"directory-resolved identities {resolved_ids}"
                )
            deps.searched = True
        else:
            if _requires_two_distinct(entry) and len(resolved_ids) < 2:
                return _insufficient_schools_clarification(question_id)
            for sid in resolved_ids:
                if sid not in deps.resolved_school_ids:
                    deps.resolved_school_ids.append(sid)
            deps.searched = True
            _attach_ids(request, school_required, resolved_ids)
    elif school_required and not explicit_fields_present:
        # no mentions, no explicit IDs: derivation will clarify the school field
        pass

    # two-distinct check for explicit path
    if school_required and "acara_sml_ids" in school_required and 
explicit_fields_present:
        ids = _explicit_id_values(request, school_required)
        if _requires_two_distinct(entry) and len(_dedupe(ids)) < 2:
            return _insufficient_schools_clarification(question_id)
```

Hmm wait — deps.searched=True in the consistent-explicit branch: preflights 
happened and succeeded. OK.

Also — the two-id check for resolved path happens pre-commit ✓. For explicit 
path, no commit anyway.

Then selector routing, constraint, derivation, validation, fetch.

Hold on — ordering issue: derivation AFTER constraint (constraint only fires 
when field present). And selector routing after derivation or before? If 
requested_metrics present and required inputs missing — e.g. VC-POC-07 with 
requested_metrics ["year 6"] but no market_definition → normalization sets 
grades; derivation flags market_definition → clarify missing market_definition. 
Either order clarifies. I'll do routing first, then constraint, then derivation 
(so derivation sees final request), then validation. Wait — but plan says scope 
rule before validation ✓ (routing precedes validation).

Hmm — but one more ordering concern: unnormalizable metric clarification vs 
missing-input clarification: routing first → unnormalizable wins. Fine.

Let me also double check the metrics cleanup uses request_scope_problem BEFORE 
validation (it does — routing precedes validation).

### Now — `_insufficient_schools_clarification`:

```python
def _insufficient_schools_clarification(question_id: str) -> 
ClarificationNeeded:
    return ClarificationNeeded(
        question_id=question_id,
        missing=[MissingInput(
            input="acara_sml_ids",
            why_material="A head-to-head comparison needs at least two distinct 
schools.",
            allowed_values=[],
        )],
        prompt_to_user="Which distinct schools should be compared? Name at least
two.",
    )
```

### Selector-routing clarification prompt

```python
prompt_to_user=f"Which {field} should I use? Allowed values: {', 
'.join(vocab)}."
```

Hmm, for metrics with 18 values that's long but informative. Fine.

### Why-material for unnormalizable label:

f"'{label}' is not an allowed {field} value for {question_id}."

### No-route-with-metrics clarification:

why: "This question does not take metric selectors; the labels cannot be 
applied." allowed []. input "metrics". OK.

### Now — the market_definition scalar clarification prompt:

f"Which market definition should I use? Allowed: {', '.join(vocab)}."

### `_material_why` — keep existing + it's fine.

### Derivation prompt:

"To answer this question I need the acara_sml_id and the market_definition. 
Could you provide them?" — keep existing style.

Wait — the existing prompt: f"To answer this question I need " + ", ".join(f"the
{inp}" ...) + ". Could you provide " + ("it"/"them") + "?" — keep exactly (test 
test_ask_pipeline_missing_material_input_returns_clarification doesn't check 
prompt; run_cli clarification rendering tests don't cover planner prompts). Keep
similar.

### Now — validation errors as TerminalError — but there's a subtlety: 
`_request_schema` load errors are TerminalError ✓; `_selector_route` raising 
TerminalError for unknown metric_list (contract drift) ✓.

### The `x-poc` annotation blocks in schemas — my validator ignores unknown 
keywords ✓.

### `_index.json` questions include gated VC-POC-14 — never reached (gated 
earlier) ✓.

### Now think again about `test_vc_poc_03_routes_selector_to_ratios` schema 
validation: request {"acara_sml_ids": [40387, 40392], "ratios": 
["students_per_teaching_fte"]} — pinned VC-POC-03 schema: required 
[acara_sml_ids]; ratios enum ✓; uniqueItems ✓; min 1 ✓. Validates ✓.

### Now — VC-POC-02 request {"acara_sml_id": 40387, "grades": ["year_6"]} — 
pinned schema: grades enum contains "year_6" ✓.

### One more — metric normalization "Teaching Staff" scope: scope rule drops 
`_headcount`/`non_teaching_` — teaching_fte survives ✓.

Now — what about profile metric "teaching_staff_headcount" directly requested 
with headcount wording in user question? deps.user_question is raw question; 
scope rule handles. Unchanged semantics.

### Now — the unanswerable/excluded/unknown reasons: keep existing text.

### One more important piece: `_branch_evidence_error` — old tests assert 
`result.reason_kind == "out_of_scope"` for bad_request — REMOVED (replaced). 
Ensure no other test references out_of_scope... test_intent_boundary? No. 
Golden? not planner. OK.

### planner.py final code — let me also keep the `_EXCLUSION_KEYWORD_MAP` and 
`_find_exclusion_text` exactly as-is.

### Now — imports in planner.py: json, logging, re, functools.lru_cache, 
pathlib.Path, typing.Any, contract, TerminalError, EvidenceClient, 
EvidenceError, StructuredIntent, ClarificationNeeded, MissingInput, Unsupported,
Deps, ToolInvocation, request_scope_problem. ✓

### Wait — potential circular import? planner imports from harness.contract — 
existing. Loads files lazily via lru_cache on first use — fine.

### CONTRACT_DIR resolution: planner.py at harness/planner.py → 
parents[0]=harness, parents[1]=repo root. CONTRACT_DIR = parents[1]/"contract" 
✓. But tests run from repo root; contract/ exists ✓.

Hmm — one catch: `Path(__file__).resolve().parents[1] / "contract"` — resolve() 
at import time ok.

Let me now also double-check `_normalize_scalar_selectors` — what if 
market_definition value is already canonical? `_canonical_for("distance", 
vocab)` → "distance" ✓ (normalizes to "distance", matches vocab "distance" 
normalized "distance").

Edge: market_definitions enum from catalog schema properties — 
"market_definitions" property has items.enum ["distance","lga","postcode","sa2"]
✓.

### Another consideration — `intent.filters` non-dict? StructuredIntent is 
pydantic with filters: dict — safe.

### `intent.requested_metrics` — list of strings. If contains non-str (model 
quirk) — `str(label)` in normalize handles; clarify if unmatched. OK.

### Writing the validator — also handle `propertyNames`? Not used. 
`dependencies`? Not used. `patternProperties`? Possibly not. Skip.

### One more risk: VC-POC-05/06/07 schemas might express conditional via 
"allOf":[{"if":{...},"then":{...}}] — handled ✓. Or via "dependentRequired"? 
Unlikely (generated). Handled keywords cover typical generated output.

Actually wait — I should double-check: could the VC-POC-05 schema express the 
conditional as:

```json
"allOf": [
  {
    "if": { "properties": { "market_definition": { "const": "distance" } }, 
"required": ["market_definition"] },
    "then": { "required": ["distance_km"] }
  }
]
```

My validator: allOf → sub → "if" present: validate(value against if): 
properties.market_definition present & const check ✓ → condition ok → then: 
required distance_km → check. ✓

### Schema `$schema`/`$id`/`title` ignored ✓.

### Now — validator for `additionalProperties` when `properties` missing but 
additionalProperties False: unknown = all keys. Correct per JSON Schema.

### Now — final: the ambiguity of `deps.answer_view` attribute — Deps dataclass 
includes answer_view? test_ask_pipeline_stores_answer_view_on_deps sets it after
success — so the attribute exists. Default value — presumably None. My new 
intent-boundary test asserts `deps.answer_view is None` — plausible. Slight 
risk; alternatively assert `deps.answer_view is None` — keep, standard.

Hmm, actually let me reduce risk: instead of asserting answer_view is None, 
assert the output type and that no narration/answer artifacts... 
deps.answer_view is set by ask() presumably in the answer stage. On 
clarification, not set → stays default (None). I'll keep the assertion — 
reasonable.

Wait, actually — hmm, Deps is defined in harness/tools.py — dataclass with 
fields client, user_question="", searched=False, resolved_school_ids (list), 
tool_calls (list), last_envelope=None, last_messages=None, intent=None, 
answer_view=None presumably. Yes.

### deps.last_envelope is None assertion in TerminalError test — after fetch 
failure, last_envelope untouched → None ✓ (initial). But in test_planner's 
test_schema_valid_emitter_bad_request_is_terminal I assert tool_calls and 
resolved ids, not last_envelope. In intent_boundary's 
test_planner_failure_is_not_unsupported I assert deps.last_envelope is None. 
Deps default last_envelope=None ✓.

### Now — writing everything out. Also update module docstrings.

One more check on test_run_cli — the module currently imports `load_cases, 
load_reference_solutions` at import → reads evals/cases/golden_cases.json — 
fine.

For the new test in test_run_cli, `stub_client()` already defined there ✓. 
Unsupported imported ✓. Need `Any` imported ✓ (typing Any imported). monkeypatch
— pytest imported ✓.

fake_converse async — main uses asyncio.run(converse(...)) → works with a 
coroutine function.

But wait — monkeypatch.setattr(cli, "converse", fake_converse): main calls 
`converse(` — global lookup → fake ✓.

Also — should the new test assert max_clarifications passthrough? Keep minimal.

Edge — `cli.main` catches (HarnessError, EmitterFailure, ClaudeAccountCliError) 
around converse — fake returns Unsupported normally. ✓

Exit code: EXIT_BY_BRANCH[Unsupported] = EXIT_UNSUPPORTED ✓.

### Also update run.py converse docstring + main. And the module docstring 
mention? The docstring bullet about clarification stays. Maybe add a line about 
pipeline default. I'll adjust the top docstring lightly — actually careful: 
docstrings are fine to edit (not pinned).

Let me now also double check — removing `build_agent` import from run.py: run.py
currently:

```python
from harness.agent import ask, build_agent  # noqa: E402
```

→

```python
from harness.agent import ask  # noqa: E402
```

And main:

```python
    require_credentials()
    client = build_client(args.stub)
    deps = Deps(client=client)
    ask_user = None if args.no_interactive else _prompt_on_terminal

    try:
        output = asyncio.run(
            converse(
                None,
                deps,
                args.question,
                ask_user=ask_user,
                max_clarifications=args.max_clarifications,
            )
        )
```

Add a comment: "The deterministic four-stage pipeline is the default path; 
converse() still accepts an explicit legacy agent for replay/compatibility 
callers."

### Now — hmm, wait. One more thing about `converse` pipeline clarification: 
after clarification round, `history = deps.last_messages` — pipeline path 
ignores history; reply=answer → Stage 1 re-interprets "10 km" — the answer-merge
problem. Existing code — not my scope.

### Now — the plan's requirement "Do not construct the legacy agent for this 
default path" ✓.

### Now assemble test_planner.py completely. Also keep section comments. Also 
keep `run_planner` helper? It's used by...? Searching the old file: 
`run_planner` defined but... tests construct deps inline. `run_planner` unused 
actually? Old tests don't call run_planner — they inline. Hmm, `run_planner` 
defined at top; I don't see usage in the shown tests. I'll drop it (or keep — 
dropping unused is fine since I rewrite the file).

Let me also double-check the existing `test_school_resolution_records_search` 
under new code: intent VC-POC-01 mention "Some School"; 
_HappyClient.school_directory returns [{"acara_sml_id": 40387, "school_name": 
search}] → exactly 1 ✓ → staged [40387] → commit → deps.searched True ✓; attach 
acara_sml_id=40387; metrics none; validation ✓; fetch → envelope. assertions: 
searched True ✓, 40387 in resolved ✓, one search_schools call ✓.

### Now — also verify `test_missing_material_input_returns_clarification` 
(VC-POC-05, missing claim, no filters): derivation → missing [acara_sml_id, 
market_definition] → clarify ✓ asserts question_id and market_definition in 
names ✓.

### `test_happy_path_returns_envelope`: VC-POC-01 filters acara 40387 → 
validated → fetch → envelope dict returned; `deps.last_envelope is result` ✓.

### Let me also think about whether validation should run for envelope-only 
mocks... yes, always.

### VC-POC-09 constraint when peer_filters present-but-not-dict (e.g. string)? 
Constraint: isinstance dict → skip → derivation satisfied (present) → schema 
validation → type error → TerminalError. OK.

### Now — `_explicit_filter_constraint` for VC-POC-10 default-only: no 
defaultable cohort keys → nothing. But plan mentions "sector-default-only" for 
both; VC-POC-10's constraint is just "at least one explicit filter". Non-empty 
dict passes. ✓

### Now — write the whole planner file.

Also — logger.debug calls keep. Add debug for clarification paths? Keep minimal.

One more — `ToolInvocation("fetch_evidence", arguments, error_kind=...)` before 
branch ✓; before TerminalError raise ✓ (both bad_request-terminal and 
data_unavailable paths record first).

### Wait — the existing except-Exception record: error_kind=type(exc).__name__ 
for preflight; fetch generic → "operational" (keep).

### Also `_resolve_mentions` — records raised exceptions with error_kind=type 
name, re-raise ✓ (invocation-recording).

### Another subtle point: `identity_from_caller` — what if question requires 
school field, mentions resolved, explicit fields ALSO present-and-equal → 
identity_from_caller True → bad_request → clarify. But directory validated those
IDs (resolved match) → bad_request is drift... but they were caller-supplied. 
Ambiguous; the plan: "the explicitly passed-through numeric-ID case" → clarify. 
Since caller supplied them, clarify. Fine.

### Now — one more careful look at the ORDER of gates vs `_selector_route` 
TerminalError: `_selector_route` can raise TerminalError (drift) — fine 
anywhere.

### Write `_material_why` unchanged.

### Now — full planner.py:

```python
"""Stage 2 of the 4-stage pipeline: deterministic planning and execution.

`plan_and_execute()` takes a StructuredIntent and returns either:
- a dict (the evidence envelope) — the happy path
- ClarificationNeeded — an input the caller must supply or correct
- Unsupported — an exclusion or an unknown question

No model calls. Every branch is a pure function of the intent, the pinned
contract artefacts, and the evidence client's response.

What is deterministic here (work item 20260908T113118):

- Missing required, material, and applicable conditional-material inputs are
  derived from the pinned question policy and the assembled request. The
  intent model's own `missing_material_inputs` claim is never trusted.
- Named schools resolve through the pinned directory preflight and must match
  exactly one record each; IDs are staged in mention order, deduplicated
  without reordering, and committed to `Deps` only after every mention
  resolves. Zero or multiple matches clarify; no evidence call is made and no
  partial identity leaks into `Deps`.
- Selector labels are Unicode-case-folded and separator-normalized, matched
  against the pinned static vocabulary (registry `metric_list` where declared,
  otherwise the input-catalog schema's field mapping and embedded enums), and
  routed to the question's own request field (`metrics` for VC-POC-01/04/11,
  `ratios` for VC-POC-03, `grades` for VC-POC-02/07). Unnormalizable labels
  clarify with the vocabulary; nothing is invented.
- Every request is validated against its registered pinned per-question
  schema (draft 2020-12 subset) before dispatch. A planner-produced request
  that cannot satisfy its schema raises TerminalError — never Unsupported.
- Error origins are separated: a bad_request for a request carrying only a
  caller-supplied numeric ACARA identity is an identity clarification; any
  other bad_request after deterministic validation is planner/contract drift
  and raises TerminalError. excluded/unknown/unanswerable keep their typed
  abstentions; data_unavailable/internal remain operational TerminalErrors.
"""
```

Then code. Let me write it fully now, carefully.

```python
from __future__ import annotations

import json
import logging
import re
from functools import lru_cache
from pathlib import Path
from typing import Any

from harness import contract
from harness.errors import TerminalError
from harness.evidence import EvidenceClient, EvidenceError
from harness.intent import StructuredIntent
from harness.outputs import ClarificationNeeded, MissingInput, Unsupported
from harness.tools import Deps, ToolInvocation, request_scope_problem

logger = logging.getLogger("harness.planner")

CONTRACT_DIR = Path(__file__).resolve().parents[1] / "contract"
_SCHEMA_DIR = CONTRACT_DIR / "question_schemas" / "v0.3.0"

# The one pinned semantic alias (prompts/rules.md): for school-profile
# questions "teaching staff" is the teaching-FTE metric. Keyed by the
# normalized label so "Teaching Staff" and "teaching-staff" both match.
_PINNED_SELECTOR_ALIASES: dict[str, str] = {"teaching staff": "teaching_fte"}

_SCHOOL_FIELDS = ("acara_sml_id", "acara_sml_ids")

# Label-selector request fields the planner routes requested_metrics into.
_SELECTOR_BAG_FIELDS = ("metrics", "ratios", "grades")

# The exact pinned conditional-material expression (VC-POC-05/06/07). Any
# other conditional text in the registry is contract drift, not something to
# parse heuristically.
_KNOWN_DISTANCE_CONDITIONS = frozenset(
    {
        "market definition equals distance",
        "market definition distance",
    }
)
```

Exclusion helpers (unchanged from current file):

```python
_EXCLUSION_KEYWORD_MAP: dict[str, str] = {...}

def _find_exclusion_text(...) ...
```

Pinned artefact loading:

```python
def _load_pinned_json(path: Path) -> Any:
    try:
        return json.loads(path.read_text(encoding="utf-8"))
    except (OSError, ValueError) as exc:
        raise TerminalError(
            f"pinned contract artefact could not be loaded ({path.name}): {exc}"
        ) from exc


@lru_cache(maxsize=1)
def _schema_index() -> dict[str, Any]:
    index = _load_pinned_json(_SCHEMA_DIR / "_index.json")
    if not isinstance(index, dict) or not isinstance(index.get("questions"), 
dict):
        raise TerminalError("pinned question-schema index is malformed")
    return index


@lru_cache(maxsize=1)
def _input_catalog_schema() -> dict[str, Any]:
    schema = _load_pinned_json(CONTRACT_DIR / 
"poc_input_catalog.v0.3.0.schema.json")
    if not isinstance(schema, dict):
        raise TerminalError("pinned input-catalog schema is malformed")
    return schema


def _registered_request_schema_path(question_id: str) -> Path:
    entry = _schema_index()["questions"].get(question_id)
    if not isinstance(entry, dict):
        raise TerminalError(f"{question_id} has no registered request schema")
    filename = entry.get("request_schema")
    if not isinstance(filename, str) or filename != 
f"{question_id}.request.schema.json":
        raise TerminalError(
            f"{question_id} request-schema registration is inconsistent: 
{filename!r}"
        )
    path = (_SCHEMA_DIR / filename).resolve()
    pinned = CONTRACT_DIR.resolve()
    if pinned not in path.parents:
        raise TerminalError(
            f"refusing to load a request schema from outside the pinned contract
"
            f"directory: {filename}"
        )
    return path


@lru_cache(maxsize=None)
def _request_schema(question_id: str) -> dict[str, Any]:
    schema = _load_pinned_json(_registered_request_schema_path(question_id))
    if not isinstance(schema, dict):
        raise TerminalError(f"{question_id} request schema is malformed")
    return schema
```

Static vocab:

```python
@lru_cache(maxsize=1)
def _field_to_catalog_list() -> dict[str, str]:
    mapping = _input_catalog_schema().get("x-poc-request-field-to-catalog-list")
    if not isinstance(mapping, dict):
        return {}
    return {
        str(field): str(list_name)
        for field, list_name in mapping.items()
        if isinstance(field, str) and isinstance(list_name, str)
    }


def _static_enum(list_name: str) -> list[str] | None:
    """The embedded static enum for a catalog list, or None when the list is
    release-derived (shape-checked only, never enum-enforced — OD-03)."""
    if list_name.startswith("release_derived_values."):
        return None
    prop = _input_catalog_schema().get("properties", {}).get(list_name)
    if isinstance(prop, dict) and isinstance(prop.get("items"), dict):
        enum = prop["items"].get("enum")
        if isinstance(enum, list):
            return [str(v) for v in enum]
    return None


def _static_vocabulary_for_field(question_id: str, field: str) -> list[str] | 
None:
    list_name = _field_to_catalog_list().get(f"{question_id}.{field}")
    if list_name is None:
        return None
    return _static_enum(list_name)


def _selector_route(question_id: str) -> tuple[str, list[str]] | None:
    """(request field, static vocabulary) for the question's label-selector 
bag."""
    entry = contract.questions().get(question_id) or {}
    metric_list = entry.get("metric_list")
    if metric_list:
        values = contract.registry().get("metric_lists", {}).get(metric_list)
        if not isinstance(values, list) or not values:
            raise TerminalError(
                f"{question_id} declares metric_list {metric_list!r} with no 
pinned values"
            )
        return "metrics", [str(v) for v in values]
    for key in sorted(_field_to_catalog_list()):
        prefix = f"{question_id}."
        if not key.startswith(prefix):
            continue
        field = key[len(prefix):]
        if field in _SELECTOR_BAG_FIELDS:
            vocabulary = _static_enum(_field_to_catalog_list()[key])
            if vocabulary is None:
                raise TerminalError(
                    f"{question_id}.{field} routes to a non-static vocabulary"
                )
            return field, vocabulary
    return None
```

Normalization:

```python
_SEPARATOR_RE = re.compile(r"[\W_]+", re.UNICODE)


def _normalize_label(label: Any) -> str:
    """Unicode-safe case folding plus separator normalization.

    `total_enrolments`, `Total Enrolments` and `total-enrolments` all fold to
    `total enrolments`; Unicode letters survive the fold.
    """
    folded = _SEPARATOR_RE.sub(" ", str(label).casefold())
    return folded.strip()


def _canonical_for(label: Any, vocabulary: list[str]) -> str | None:
    """Match a label against a pinned static vocabulary.

    Direct normalized matches win; only the pinned semantic aliases apply
    beyond that. Canonical values are returned unchanged.
    """
    wanted = _normalize_label(label)
    for value in vocabulary:
        if _normalize_label(value) == wanted:
            return value
    alias = _PINNED_SELECTOR_ALIASES.get(wanted)
    if alias is not None and alias in vocabulary:
        return alias
    return None


def _dedupe(values: list[Any]) -> list[Any]:
    """Order-preserving deduplication."""
    seen: set[Any] = set()
    unique: list[Any] = []
    for value in values:
        if value not in seen:
            seen.add(value)
            unique.append(value)
    return unique
```

Validator:

```python
def _json_key(value: Any) -> str:
    return json.dumps(value, sort_keys=True, ensure_ascii=False, default=str)


def _type_matches(value: Any, expected: str) -> bool:
    ...


def _schema_problems(value: Any, schema: Any, path: str) -> list[str]:
    ...
```

Write carefully (as sketched above).

Then request validation entry:

```python
def _validate_request(question_id: str, request: dict[str, Any]) -> None:
    schema = _request_schema(question_id)
    problems = _schema_problems(request, schema, "$request")
    if problems:
        raise TerminalError(
            f"{question_id}: the planner-built request does not satisfy its "
            f"registered request schema: " + "; ".join(problems)
        )
```

Now the main function and helpers per plan. Let me also handle `intent.filters` 
type: `dict(intent.filters)` — StructuredIntent.filters is dict[str, Any].

Now — full plan_and_execute:

```python
async def plan_and_execute(
    intent: StructuredIntent,
    client: EvidenceClient,
    deps: Deps,
) -> dict[str, Any] | ClarificationNeeded | Unsupported:
    """Deterministic Stage 2: intent → evidence envelope or typed outcome.

    Returns:
        dict — the evidence envelope (happy path)
        ClarificationNeeded — a caller-suppliable input is missing or wrong
        Unsupported — an exclusion or an unknown question
    May raise:
        TerminalError — planner/schema defects and operational stops
    """
    question_id = intent.question_id

    # 1. Excluded topic
    if question_id == "excluded":
        ... unchanged ...

    # 2. Registry-owned excluded question
    ... unchanged ...

    # 3. Gated question
    ... unchanged ...

    # 4. Unknown question
    ... unchanged ...

    # 5. Pinned policy for this question
    entry = contract.questions().get(question_id)
    if entry is None:
        raise TerminalError(
            f"{question_id} is registered in the schema index but absent from 
the pinned registry"
        )
    policy = entry.get("input_policy") or {}
    required_fields = [f for f in policy.get("required", []) if isinstance(f, 
str)]

    # 6. Assemble the request from explicit intent fields
    request: dict[str, Any] = dict(intent.filters)
    scalar_clarification = _normalize_scalar_selectors(question_id, request)
    if scalar_clarification is not None:
        return scalar_clarification

    school_required = [f for f in required_fields if f in _SCHOOL_FIELDS]
    _adapt_explicit_school_fields(request, school_required)
    identity_from_caller = any(f in request for f in school_required)

    # 7. Transactional named-school resolution
    mentions = [m for m in (intent.school_mentions or []) if isinstance(m, str) 
and m.strip()]
    if school_required and mentions:
        resolved_ids, failure = await _resolve_mentions(question_id, mentions, 
client, deps)
        if failure is not None:
            return failure
        if identity_from_caller:
            explicit_ids = _explicit_id_values(request, school_required)
            if sorted(resolved_ids) != sorted(explicit_ids):
                raise TerminalError(
                    f"{question_id}: explicit school identities {explicit_ids} 
conflict with "
                    f"directory-resolved identities {resolved_ids}"
                )
            deps.searched = True
        else:
            if _requires_two_distinct_ids(entry) and len(resolved_ids) < 2:
                return _insufficient_schools_clarification(question_id)
            for school_id in resolved_ids:
                if school_id not in deps.resolved_school_ids:
                    deps.resolved_school_ids.append(school_id)
            deps.searched = True
            _attach_school_ids(request, school_required, resolved_ids)
    elif school_required and identity_from_caller and "acara_sml_ids" in 
school_required:
        ids = _explicit_id_values(request, school_required)
        if _requires_two_distinct_ids(entry) and len(_dedupe(ids)) < 2:
            return _insufficient_schools_clarification(question_id)

    # 8. Selector normalization and routing
    selector_clarification = _route_selectors(question_id, intent, request, 
deps)
    if selector_clarification is not None:
        return selector_clarification

    # 9. Registry-owned explicit-filter constraints (VC-POC-09/10)
    constraint_clarification = _enforce_explicit_filter_constraints(entry, 
request)
    if constraint_clarification is not None:
        return constraint_clarification

    # 10. Derive missing inputs from the pinned policy
    missing = _derive_missing_inputs(question_id, entry, request)
    if missing:
        missing_items = [
            MissingInput(
                input=field,
                why_material=_material_why(question_id, field),
                allowed_values=_static_vocabulary_for_field(question_id, field) 
or [],
            )
            for field in missing
        ]
        prompt_to_user = (
            "To answer this question I need "
            + ", ".join(f"the {field}" for field in missing)
            + ". Could you provide "
            + ("it" if len(missing) == 1 else "them")
            + "?"
        )
        logger.debug(
            "planner clarification",
            extra={"subsystem": "planner", "question_id": question_id, 
"missing": missing},
        )
        return ClarificationNeeded(
            question_id=question_id,
            missing=missing_items,
            prompt_to_user=prompt_to_user,
        )

    # 11. Pinned-schema validation before dispatch
    _validate_request(question_id, request)

    # 12. Fetch evidence
    arguments = {"question_id": question_id, "request": request}
    try:
        envelope = await client.fetch_evidence(question_id, request)
    except EvidenceError as exc:
        deps.tool_calls.append(
            ToolInvocation("fetch_evidence", arguments, 
error_kind=exc.error_kind)
        )
        return _branch_evidence_error(question_id, exc, identity_from_caller)
    except Exception:
        deps.tool_calls.append(
            ToolInvocation("fetch_evidence", arguments, 
error_kind="operational")
        )
        raise

    deps.tool_calls.append(ToolInvocation("fetch_evidence", arguments, 
envelope))
    deps.last_envelope = envelope
    logger.debug(
        "evidence fetched by planner",
        extra={"subsystem": "planner", "question_id": question_id},
    )
    return envelope
```

Wait — one problem: the two-distinct check for explicit path: I compute 
`_explicit_id_values` — but for singular field with explicit id, wrap happened 
only when required plural; for required plural + explicit acara_sml_ids list — 
fine. But what if explicit ids is [] because the value is malformed (e.g. string
in list)? len 0 < 2 → clarify "needs two distinct schools" — misleading (should 
be schema TerminalError). Hmm: `_explicit_id_values` returns [] for malformed → 
we'd clarify instead of TerminalError. Edge; to be precise, check 
`identity_from_caller` but ids present... Let me guard: only apply two-distinct 
clarify when ALL explicit ids are ints:

```python
        ids = _explicit_id_values(request, school_required)
        raw = request.get("acara_sml_ids")
        if _requires_two_distinct_ids(entry) and isinstance(raw, list) and 
len(ids) == len(raw) and len(_dedupe(ids)) < 2:
            return clarification
```

If malformed entries exist, len(ids) < len(raw) → skip clarify → schema 
validation TerminalErrors it. OK.

Now `_resolve_mentions` and remaining helpers:

```python
async def _resolve_mentions(
    question_id: str,
    mentions: list[str],
    client: EvidenceClient,
    deps: Deps,
) -> tuple[list[int], ClarificationNeeded | None]:
    """Resolve each mention through the directory, exactly one record each.

    Preflights are recorded even when they fail. IDs are staged locally in
    mention order and deduplicated without reordering; nothing is committed
    here — the caller commits to `Deps` only after every mention resolves.
    """
    staged: list[int] = []
    for mention in mentions:
        arguments = {"search": mention, "limit": 25}
        try:
            records = await client.school_directory(search=mention, limit=25)
        except Exception as exc:
            deps.tool_calls.append(
                ToolInvocation("search_schools", arguments, 
error_kind=type(exc).__name__)
            )
            raise
        deps.tool_calls.append(ToolInvocation("search_schools", arguments, 
records))
        ids = [
            record["acara_sml_id"]
            for record in records if isinstance(record, dict)
            if isinstance(record.get("acara_sml_id"), int)
            and not isinstance(record.get("acara_sml_id"), bool)
        ]
        if not ids:
            return [], _school_match_clarification(
                question_id, mention, f"'{mention}' matched no school in the 
pinned directory.",
                "I couldn't find that school. Could you check the name — or give
me the school's ACARA ID?",
            )
        if len(ids) > 1:
            return [], _school_match_clarification(
                question_id, mention,
                f"'{mention}' matched {len(ids)} schools; a request needs 
exactly one.",
                f"More than one school matches '{mention}'. Which one did you 
mean?",
            )
        school_id = ids[0]
        if school_id not in staged:
            staged.append(school_id)
    return staged, None


def _school_match_clarification(
    question_id: str, mention: str, why_material: str, prompt_to_user: str
) -> ClarificationNeeded:
    return ClarificationNeeded(
        question_id=question_id,
        missing=[MissingInput(input="acara_sml_id", why_material=why_material, 
allowed_values=[])],
        prompt_to_user=prompt_to_user,
    )
```

Wait — list comprehension syntax: 

```python
ids = [
    record["acara_sml_id"]
    for record in records
    if isinstance(record, dict)
    and isinstance(record.get("acara_sml_id"), int)
    and not isinstance(record["acara_sml_id"], bool)
]
```

Fix syntax properly.

Attach/adapt helpers:

```python
def _attach_school_ids(request, school_required, ids):
    if not ids:
        return
    if "acara_sml_ids" in school_required:
        request.setdefault("acara_sml_ids", list(ids))
    elif "acara_sml_id" in school_required:
        request.setdefault("acara_sml_id", ids[0])


def _adapt_explicit_school_fields(request, school_required) -> None:
    """Let a caller-supplied identity in either school field satisfy the
    question's own selector field, without inventing values."""
    if not school_required:
        # Nothing requires a school; leave explicit fields for schema 
validation.
        return
    if "acara_sml_ids" in school_required:
        if "acara_sml_ids" not in request and 
isinstance(request.get("acara_sml_id"), int):
            request["acara_sml_ids"] = [request.pop("acara_sml_id")]
    elif "acara_sml_id" in school_required:
        if "acara_sml_id" not in request and 
isinstance(request.get("acara_sml_ids"), list):
            ids = request["acara_sml_ids"]
            if len(ids) == 1 and isinstance(ids[0], int) and not 
isinstance(ids[0], bool):
                request["acara_sml_id"] = ids[0]
                del request["acara_sml_ids"]
            else:
                raise TerminalError(
                    "caller supplied multiple acara_sml_ids for a question whose
schema "
                    "takes a single acara_sml_id"
                )


def _explicit_id_values(request, school_required) -> list[int]:
    if "acara_sml_ids" in school_required:
        value = request.get("acara_sml_ids")
        if isinstance(value, list):
            return [v for v in value if isinstance(v, int) and not isinstance(v,
bool)]
        return []
    if "acara_sml_id" in school_required:
        value = request.get("acara_sml_id")
        if isinstance(value, int) and not isinstance(value, bool):
            return [value]
    return []


def _requires_two_distinct_ids(entry: dict[str, Any]) -> bool:
    constraints = entry.get("input_policy", {}).get("constraints", [])
    return any("at least two" in str(c) for c in constraints)
```

Hmm — "acara_sml_ids contains at least two values" — "at least two" matches ✓.

`_insufficient_schools_clarification` as above.

Scalar selectors:

```python
def _normalize_scalar_selectors(question_id, request) -> ClarificationNeeded | 
None:
    if "market_definition" not in request:
        return None
    value = request["market_definition"]
    if not isinstance(value, str):
        return None  # shape-checked by the pinned schema
    vocabulary = _static_vocabulary_for_field(question_id, "market_definition")
    if vocabulary is None:
        return None
    canonical = _canonical_for(value, vocabulary)
    if canonical is None:
        return ClarificationNeeded(
            question_id=question_id,
            missing=[
                MissingInput(
                    input="market_definition",
                    why_material=(
                        f"'{value}' is not an allowed market definition for 
{question_id}."
                    ),
                    allowed_values=vocabulary,
                )
            ],
            prompt_to_user=(
                "Which market definition should I use? Allowed values: "
                + ", ".join(vocabulary)
                + "."
            ),
        )
    request["market_definition"] = canonical
    return None
```

Selector routing:

```python
def _route_selectors(question_id, intent, request, deps) -> ClarificationNeeded 
| None:
    """Route requested selector labels to the question's own request field.

    Static vocabularies come from pinned artefacts (registry metric_list where
    declared, otherwise the input-catalog schema's field mapping and embedded
    enums) — never from the runtime emitter. Unnamed selectors are omitted so
    the emitter keeps default ownership.
    """
    labels = [l for l in (intent.requested_metrics or []) if str(l).strip()]
    if not labels:
        return None
    route = _selector_route(question_id)
    if route is None:
        return ClarificationNeeded(
            question_id=question_id,
            missing=[
                MissingInput(
                    input="metrics",
                    why_material=(
                        f"{question_id} takes no metric selector, so the 
requested "
                        "labels cannot be applied to this request."
                    ),
                    allowed_values=[],
                )
            ],
            prompt_to_user=(
                "This question does not take metric selectors. Should I answer 
it as asked?"
            ),
        )
    field, vocabulary = route
    canonical: list[str] = []
    for label in labels:
        matched = _canonical_for(label, vocabulary)
        if matched is None:
            return ClarificationNeeded(
                question_id=question_id,
                missing=[
                    MissingInput(
                        input=field,
                        why_material=(
                            f"'{label}' is not an allowed {field} value for 
{question_id}."
                        ),
                        allowed_values=vocabulary,
                    )
                ],
                prompt_to_user=(
                    f"Which {field} did you mean? Allowed values: "
                    + ", ".join(vocabulary)
                    + "."
                ),
            )
        canonical.append(matched)
    canonical = _dedupe(canonical)

    if field == "metrics":
        # Existing raw-question scope rule: a user who never asked about
        # headcounts or non-teaching staff gets neither, even if a label was
        # requested. Applied before validation, as before.
        scope_problem = request_scope_problem(
            deps.user_question, question_id, {**request, "metrics": canonical}
        )
        if scope_problem:
            cleaned = [
                m
                for m in canonical
                if not (m.endswith("_headcount") or 
m.startswith("non_teaching_"))
            ]
            canonical = cleaned or canonical

    request[field] = canonical
    return None
```

Explicit-filter constraints:

```python
def _enforce_explicit_filter_constraints(entry, request) -> ClarificationNeeded 
| None:
    """The registry-owned VC-POC-09/10 rule (OD-07): a cohort definition must
    contain at least one explicit filter — the sector default alone does not
    define a peer group. Enforced exactly as pinned, not generalised."""
    policy = entry.get("input_policy") or {}
    required = [f for f in policy.get("required", []) if isinstance(f, str)]

    if "peer_filters" in required and isinstance(request.get("peer_filters"), 
dict):
        filters = request["peer_filters"]
        default_sectors = policy.get("defaultable", 
{}).get("peer_filters.sectors")
        if set(filters) == {"sectors"} and isinstance(default_sectors, list):
            sectors = filters["sectors"]
            if (
                isinstance(sectors, list)
                and sorted(str(s) for s in sectors) == sorted(str(s) for s in 
default_sectors)
            ):
                return ClarificationNeeded(
                    question_id=entry.get("question_id", ""),
                    missing=[
                        MissingInput(
                            input="peer_filters",
                            why_material=(
                                "The sector default alone does not define a peer
group; "
                                "at least one explicit filter beyond the sector 
default "
                                "is required."
                            ),
                            allowed_values=[],
                        )
                    ],
                    prompt_to_user=(
                        "Beyond the Catholic/independent sector scope, what rule
should "
                        "select the peer group — for example school type, 
enrolment band, "
                        "ICSEA range or remoteness?"
                    ),
                )
    return None
```

Hmm — entry.get("question_id") — registry entries have "question_id". But 
contract.questions() mapping is keyed by question_id — does the entry dict 
include "question_id" key? The registry entries include "question_id" field. 
I'll pass question_id into the function instead. Let me change signature: 
`_enforce_explicit_filter_constraints(question_id, entry, request)`.

What about VC-POC-10 empty dict {} — my constraint skips non-dict/absent; empty 
dict {}: isinstance dict ✓ but peer branch only. For segment: nothing → schema 
minProperties≥1 → TerminalError?? But failure_behaviour says "VC-POC-09 or 
VC-POC-10 lacks an explicit nondefault cohort-defining filter → 
ClarificationNeeded". Empty dict = lacks → should clarify, not TerminalError! So
constraint must also handle empty dict for BOTH: absent → derivation clarifies 
(missing field) ✓; empty dict → constraint clarifies. Add:

```python
    for field in ("peer_filters", "segment_filters"):
        if field in required and isinstance(request.get(field), dict) and not 
request[field]:
            return ClarificationNeeded(... empty cohort definition ...)
```

Combine: 

```python
def _enforce_explicit_filter_constraints(question_id, entry, request):
    policy = entry.get("input_policy") or {}
    required = [...]
    if "peer_filters" in required:
        filters = request.get("peer_filters")
        if isinstance(filters, dict):
            if not filters or _sector_default_only(filters, policy):
                return _peer_definition_clarification(question_id, empty=not 
filters)
    if "segment_filters" in required:
        filters = request.get("segment_filters")
        if isinstance(filters, dict) and not filters:
            return _segment_definition_clarification(question_id)
    return None
```

Hmm wait — for VC-POC-09, could segment branch apply? Different questions. And 
VC-POC-10 with peer key? No. Fine.

But — what if peer_filters contains only unknown key (no sectors)? e.g. 
{"banana": 1}: set != {"sectors"} → passes constraint → schema → TerminalError ✓
(unknown nested key → planner failure, per contract). Good.

_segment_definition_clarification why: "A segment is defined by its explicit 
filters; there is no default segment." prompt: "Which segment should I analyse —
for example a sector, school type, enrolment band or state subset?"

_peer empty: "The peer group definition is empty; at least one explicit filter 
is required."

Let me write both through one helper with parameters.

Derive missing:

```python
def _derive_missing_inputs(question_id, entry, request) -> list[str]:
    """Missing required, material, then applicable conditional-material fields,
    in pinned policy order — derived, never taken from the model's claim."""
    policy = entry.get("input_policy") or {}
    missing: list[str] = []
    seen: set[str] = set()

    def consider(field: str) -> None:
        if field in seen:
            return
        seen.add(field)
        if field not in request:
            missing.append(field)

    for field in policy.get("required", []):
        if isinstance(field, str):
            consider(field)
    for field in policy.get("material", []):
        if isinstance(field, str):
            consider(field)

    conditional = policy.get("conditional_material") or {}
    if isinstance(conditional, dict):
        for field, condition in conditional.items():
            if not isinstance(field, str):
                continue
            normalized_condition = _SEPARATOR_RE.sub(" ", 
str(condition).casefold()).strip()
            if (
                field == "distance_km"
                and normalized_condition in _KNOWN_DISTANCE_CONDITIONS
            ):
                if request.get("market_definition") == "distance" and 
"distance_km" not in request:
                    consider(field)
            else:
                raise TerminalError(
                    f"{question_id}: unsupported conditional-material expression
"
                    f"{condition!r} for {field} — contract drift, refusing to 
interpret it"
                )
    return missing
```

Check normalized conditions: registry text "market_definition equals distance" →
casefold → "market_definition equals distance" → separator sub ([\W_]+ → " ") → 
"market definition equals distance" ✓ in set. Plan text "market_definition == 
\"distance\"" — different source, but registry is authority. Add both forms: 
"market definition equals distance" and... "market_definition == distance" → 
"market definition distance". Include both in the frozenset. ✓ (Already did.)

_material_why — keep as before.

_branch_evidence_error — new:

```python
def _branch_evidence_error(
    question_id: str,
    exc: EvidenceError,
    identity_from_caller: bool,
) -> Unsupported | ClarificationNeeded:
    """Translate an EvidenceError into the pipeline's typed outcomes.

    Origin-aware: a bad_request for a request whose only school identity was
    caller-supplied is a correctable identity problem → ClarificationNeeded.
    Any other bad_request — the request was deterministic and schema-valid —
    is planner/contract drift → TerminalError. excluded/unknown/unanswerable
    keep their contract-owned refusals; data_unavailable/internal are
    operational stops.
    """
    kind = exc.error_kind
    logger.debug(
        "evidence error in planner",
        extra={"subsystem": "planner", "error_kind": kind, "question_id": 
question_id},
    )

    if kind == "excluded_question":
        reason_code = exc.reason_code or contract.reason_code_for(question_id)
        reason_kind = (
            "not_yet_validated"
            if reason_code == "not_yet_validated"
            else "excluded_by_contract"
        )
        return Unsupported(reason_kind=reason_kind, reason=exc.message, 
reason_code=reason_code)

    if kind == "unknown_question":
        return Unsupported(reason_kind="not_in_registry", reason=exc.message)

    if kind == "unanswerable":
        return Unsupported(reason_kind="unanswerable", reason=exc.message)

    if kind == "bad_request":
        if identity_from_caller:
            return ClarificationNeeded(
                question_id=question_id,
                missing=[
                    MissingInput(
                        input="acara_sml_id",
                        why_material=(
                            "The data layer rejected the supplied school 
identity "
                            f"({exc.message}). Confirm the school's current 
public "
                            "ACARA_SML_ID."
                        ),
                        allowed_values=[],
                    )
                ],
                prompt_to_user=(
                    "The school identity supplied for this question was rejected
by the "
                    "data layer. Could you confirm the school?"
                ),
            )
        raise TerminalError(
            f"{question_id}: the evidence emitter returned bad_request for a "
            f"deterministic, schema-valid request — planner/contract drift: 
{exc.message}"
        )

    # data_unavailable | internal — operational stop, not a refusal.
    raise TerminalError(f"evidence emitter reported {kind}: {exc.message}")
```

Now, the validator functions — write fully:

```python
def _type_matches(value: Any, expected: str) -> bool:
    if expected == "object":
        return isinstance(value, dict)
    if expected == "array":
        return isinstance(value, list)
    if expected == "string":
        return isinstance(value, str)
    if expected == "integer":
        if isinstance(value, bool):
            return False
        if isinstance(value, int):
            return True
        return isinstance(value, float) and value.is_integer()
    if expected == "number":
        return isinstance(value, (int, float)) and not isinstance(value, bool)
    if expected == "boolean":
        return isinstance(value, bool)
    if expected == "null":
        return value is None
    return True  # unknown type keyword: leave to the remaining checks
```

```python
def _schema_problems(value: Any, schema: Any, path: str) -> list[str]:
    """Validate `value` against the pinned request schema.

    Supports the draft-2020-12 keywords the generated v0.3 request schemas use
    (types, required/properties/additionalProperties, items, enum/const,
    bounds, cardinality, uniqueness, patterns and the allOf/anyOf/oneOf and
    if/then/else combinators). Unknown annotation keywords are ignored.
    """
    problems: list[str] = []
    if not isinstance(schema, dict):
        return problems

    expected_type = schema.get("type")
    if expected_type is not None:
        expected_types = expected_type if isinstance(expected_type, list) else 
[expected_type]
        if not any(_type_matches(value, str(t)) for t in expected_types):
            problems.append(f"{path}: expected type {expected_type!r}, got 
{type(value).__name__}")
            return problems

    if "const" in schema and _json_key(value) != _json_key(schema["const"]):
        problems.append(f"{path}: must equal {schema['const']!r}")

    enum = schema.get("enum")
    if isinstance(enum, list):
        if not any(_json_key(value) == _json_key(candidate) for candidate in 
enum):
            problems.append(f"{path}: {value!r} is not an allowed value")

    if isinstance(value, dict):
        properties = schema.get("properties")
        properties = properties if isinstance(properties, dict) else {}
        for key, sub_schema in properties.items():
            if key in value:
                problems.extend(_schema_problems(value[key], sub_schema, 
f"{path}.{key}"))
        additional = schema.get("additionalProperties", True)
        unknown = [key for key in value if key not in properties]
        if unknown and additional is False:
            problems.append(f"{path}: unknown properties {sorted(unknown)}")
        elif isinstance(additional, dict):
            for key in unknown:
                problems.extend(_schema_problems(value[key], additional, 
f"{path}.{key}"))
        required = schema.get("required")
        if isinstance(required, list):
            for key in required:
                if key not in value:
                    problems.append(f"{path}: missing required property 
{key!r}")
        min_properties = schema.get("minProperties")
        if isinstance(min_properties, int) and len(value) < min_properties:
            problems.append(f"{path}: needs at least {min_properties} 
properties")
        max_properties = schema.get("maxProperties")
        if isinstance(max_properties, int) and len(value) > max_properties:
            problems.append(f"{path}: allows at most {max_properties} 
properties")

    if isinstance(value, list):
        min_items = schema.get("minItems")
        if isinstance(min_items, int) and len(value) < min_items:
            problems.append(f"{path}: needs at least {min_items} items")
        max_items = schema.get("maxItems")
        if isinstance(max_items, int) and len(value) > max_items:
            problems.append(f"{path}: allows at most {max_items} items")
        if schema.get("uniqueItems"):
            item_keys = [_json_key(item) for item in value]
            if len(set(item_keys)) != len(item_keys):
                problems.append(f"{path}: items must be unique")
        items = schema.get("items")
        if isinstance(items, dict):
            for index, element in enumerate(value):
                problems.extend(_schema_problems(element, items, 
f"{path}[{index}]"))

    if isinstance(value, str):
        pattern = schema.get("pattern")
        if isinstance(pattern, str) and re.search(pattern, value) is None:
            problems.append(f"{path}: does not match the required pattern")
        min_length = schema.get("minLength")
        if isinstance(min_length, int) and len(value) < min_length:
            problems.append(f"{path}: shorter than the required minimum length")
        max_length = schema.get("maxLength")
        if isinstance(max_length, int) and len(value) > max_length:
            problems.append(f"{path}: longer than the required maximum length")

    if isinstance(value, (int, float)) and not isinstance(value, bool):
        minimum = schema.get("minimum")
        if isinstance(minimum, (int, float)) and value < minimum:
            problems.append(f"{path}: below the minimum {minimum}")
        maximum = schema.get("maximum")
        if isinstance(maximum, (int, float)) and value > maximum:
            problems.append(f"{path}: above the maximum {maximum}")

    def _check(sub_schema: Any) -> list[str]:
        return _schema_problems(value, sub_schema, path)

    all_of = schema.get("allOf")
    if isinstance(all_of, list):
        for sub_schema in all_of:
            problems.extend(_check(sub_schema))
    any_of = schema.get("anyOf")
    if isinstance(any_of, list):
        if all(len(_check(sub_schema)) > 0 for sub_schema in any_of):
            problems.append(f"{path}: matches none of the allowed variants")
    one_of = schema.get("oneOf")
    if isinstance(one_of, list):
        matches = sum(1 for sub_schema in one_of if not _check(sub_schema))
        if matches != 1:
            problems.append(f"{path}: must match exactly one allowed variant 
(matched {matches})")
    if "if" in schema:
        condition_holds = not _check(schema["if"])
        branch = schema.get("then") if condition_holds else schema.get("else")
        if isinstance(branch, dict):
            problems.extend(_check(branch))
    not_schema = schema.get("not")
    if isinstance(not_schema, dict) and not _check(not_schema):
        problems.append(f"{path}: matches a forbidden variant")

    return problems
```

Hmm — `_check` inside dict context: nested function referencing value/path — 
fine.

One subtlety: `if "if" in schema` — when if fails to validate (condition false) 
and no "else" → OK ✓.

### Wait — critical check: validating VC-POC-01 request {"acara_sml_id": 40387}:

schema type object ✓; properties.acara_sml_id: {type integer, minimum 1} → 40387
✓; required acara_sml_id ✓; additionalProperties false: unknown [] ✓. → no 
problems ✓.

Request {"acara_sml_id": 40387, "banana": 1}: unknown properties ["banana"] → 
TerminalError ✓.

### Now the module needs `re` — yes.

### Check planner's use of contract.registry() — signature exists ✓.

Now — one thing I should double check: `contract.questions()` for "excluded" 
literal? Gate order handles "excluded" before. ✓

`contract.questions().get(question_id) or {}` in helpers — for VC-POC-REF-01 ✓.

### Another check: `_selector_route` for VC-POC-04: metric_list 
"profile_metrics" → ("metrics", profile) ✓.

### Now the run.py file — full content with modifications.

Also update the docstring bullet list: add "The default path is the 
deterministic four-stage pipeline (Stage 1 intent → Stage 2 plan/execute → Stage
3 narration → Stage 4 assembly); the legacy agent tool loop remains available to
explicit callers (replay and compatibility tests)."

### test_run_cli.py — add the two new tests at the end (wiring section). Also 
the docstring? Keep; maybe add note. Keep everything else byte-identical where 
possible.

Wait — one existing test file concern: `test_run_cli.py` imports `build_agent` 
from harness.agent — still exists ✓.

### test_intent_boundary.py — add the new tests + EvidenceError import. Keep 
everything else.

New tests:

1. test_planner_failure_is_not_unsupported (TerminalError through ask on unknown
request property; deps.last_envelope stays None; no narration produced).
2. test_planner_clarification_bypasses_narration_and_answer_view (VC-POC-08 
missing distance_km → ClarificationNeeded; deps.answer_view None).
3. test_normalized_selector_request_reaches_evidence (exact args recorded; 
RenderedAnswer).
4. test_unnormalizable_metric_stays_a_clarification.
5. test_invalid_explicit_identity_stays_a_clarification (bad_request from a mock
client → ClarificationNeeded through ask).

For #5 need a fake client in Deps. Deps(client=...) — client attribute type 
unrestricted? dataclass Deps in tools.py — `client: Any`? probably 
EvidenceClient-typed but dataclasses don't enforce. Risk low.

Hmm wait — for #3, ToolInvocation field names: I used `fetch_calls[0].args` — 
need to confirm ToolInvocation's field name is `args`. From planner usage: 
`ToolInvocation("fetch_evidence", arguments, error_kind=...)` — positional 
second arg. The dataclass field could be named `args` or `arguments`. In 
test_planner existing: `fetch_calls[0].error_kind` — error_kind exists. The 
second positional — I'll guess `args`. Hmm risky. evals/schema.py ToolCall has 
fields... different class. Let me avoid asserting on `.args` in intent_boundary 
test; instead assert via a custom recording client? Simpler: use a recording 
wrapper around stub client:

Actually — for #3 I can avoid ToolInvocation field names by wrapping:

```python
class _RecordingClient:
    def __init__(self, inner):
        self._inner = inner
        self.fetched = []
    async def school_directory(self, search=None, limit=25):
        return await self._inner.school_directory(search=search, limit=limit)
    async def fetch_evidence(self, question_id, request):
        self.fetched.append({"question_id": question_id, "request": request})
        return await self._inner.fetch_evidence(question_id, request)
    async def input_catalog(self):
        return await self._inner.input_catalog()
```

Then Deps(client=_RecordingClient(stub_client())). The narration grounding works
off the envelope (from inner stub) ✓.

In test_planner, `_DirectoryClient.fetched` is my own class — fine.

For #5 (identity clarification) I use a fully fake client — no stub. OK.

Let me define the recording client inside test_intent_boundary near the new 
tests.

Also for #2 — VC-POC-08 filters {"acara_sml_id": 40387} — missing distance_km → 
clarify. deps.answer_view None ✓. And narration not needed (no narration_model 
passed; if ask() attempted narration on clarification, it would try building a 
real model — no! It would call build_narration_agent with default model → would 
need credentials → error. But existing test 
test_ask_pipeline_missing_material_input_returns_clarification already proves 
clarification path needs no narration model. ✓

For #1 — pytest.raises(TerminalError); import TerminalError from harness.errors 
in the test.

Now — sanity check #3 exact request equality: {"acara_sml_id": 40387, "metrics":
["total_enrolments", "teaching_fte"]}. The stub fetch: inner stub case matching 
— existing tests prove this request matches a fixture. ✓

Now — one more thought on #3: does grounding still pass with metrics 
normalization? The narration "Enrolments run 612 in 2019 and 668 in 2023." — 
envelope from stub VC-POC-01 canonical — numbers 612, 668 in envelope ✓ 
(existing test passes with same).

### Now — write test_planner.py in full.

Also — one more acceptance-relevant check: 
`test_distance_is_conditional_on_distance_market` — SA2 path asserts fetched 
request equals {"acara_sml_id": 40387, "market_definition": "sa2"}. My planner: 
request = dict(filters) = {"acara_sml_id": 40387, "market_definition": "sa2"}; 
normalization: "sa2" canonical ✓ (unchanged); no mentions; identity_from_caller 
True (acara_sml_id in request, school_required) — skip resolution; two-distinct:
not plural; selectors none; constraint none; derivation: required [acara_sml_id 
✓, market_definition ✓]; material [market_definition ✓]; conditional 
distance_km: market_definition == "sa2" ≠ distance → not required ✓; validation:
VC-POC-05 schema — validates (assuming schema allows exactly these fields ✓); 
fetch ✓.

Hmm — wait, VC-POC-05 schema might have `distance_km` with `minimum: 0` 
exclusive etc. — not present → skipped ✓.

### One more — `_route_selectors` for VC-POC-05 with requested_metrics ["total 
enrolments"] → route None → clarify. Then... derivation never runs — fine, 
clarification returned.

But wait — order: routing BEFORE constraint/derivation. For VC-POC-09 with stray
metrics + default-only filters → routing clarifies first (metrics unusable) 
instead of the filter constraint. Both clarifications; fine.

Hmm — actually, reconsider routing position: for VC-POC-07, grades routing must 
happen before derivation (satisfaction) ✓ routing is before derivation ✓.

### Now — potential issue: `request_scope_problem` import — unchanged.

### Let me reconsider `_adapt_explicit_school_fields` interplay with 
`identity_from_caller`:

`identity_from_caller = any(f in request for f in school_required)` computed 
AFTER adaptation — adaptation moves acara_sml_id→acara_sml_ids for 
plural-required questions, so presence reflects the required field ✓.

### Now think about whether `deps.searched` should be True in the 
explicit+consistent-mention branch — yes set. In old test? 
`test_school_resolution_records_search` uses mention-only ✓.

### Now — what about questions where school field NOT required but mentions 
given (VC-POC-09 with mentions)? school_required empty → skip resolution 
entirely; mentions ignored. Old behavior resolved them into deps — new behavior 
ignores. Any old test? No. OK.

### Write out all files now.

Also — one more look at the "unanswerable" gate: `_branch_evidence_error` 
unanswerable → Unsupported(reason_kind="unanswerable") ✓ kept.

Now — about `Unsupported(reason_code=...)` — Unsupported model has reason_code 
optional field (used in existing code) ✓.

### Potential problem — `contract.questions()` may raise for the 
"excluded"/"unknown" ids? We call it after gates ✓.

### One more design point: `_validate_request` path prefix "$request" — fine.

### Now — the golden "answer/VC-POC-03" flow via pipeline? Not tested in 
acceptance; skip.

### Also — demo.py --stub default main? demo builds its own; out of scope.

Now let me also double-check `_load_pinned_json` TerminalError import — 
TerminalError imported ✓.

### Check for circular import: planner imports contract, errors, evidence, 
intent, outputs, tools — same as before ✓.

### Now — writing run.py complete file.

Careful to keep all existing functions and only change: docstring (small 
addition), imports (drop build_agent), converse (unchanged — it already supports
None), main (agent=None, drop build_agent call). Also add a short comment in 
main.

Wait — also the module docstring bullet: add one line:

```
* It runs the **deterministic four-stage pipeline** by default (no legacy
  agent is built); `converse()` still accepts an explicit agent for replay
  and compatibility callers.
```

### Now — final check of test files for imports they need:

test_planner.py imports: asyncio, sys, Path, Any, pytest, contract, 
TerminalError, EvidenceError, EvidenceClient, StructuredIntent, 
ClarificationNeeded, Unsupported, _find_exclusion_text, plan_and_execute, Deps. 
EvidenceClient used in stub_client signature ✓ keep.

test_intent_boundary.py additions: from harness.evidence import EvidenceError 
(inside test or top). I'll add top-level import.

test_run_cli.py: unchanged + new tests (uses existing imports; `Any` already 
imported ✓).

Now — write test_planner.py completely, preserving all kept tests verbatim where
possible and adding new ones. Let me also update its module docstring to 
describe the new behaviors.

One more scan of existing test_planner tests for compatibility with new planner:

- `test_gated_question_reason_code_preserved`: VC-POC-14 → gate before policy ✓ 
Unsupported with reason_code ✓.

- `test_excluded_question_reason_is_contract_message` ✓.

- For `_ErrorClient` — VC-POC-01 filters {"acara_sml_id": 40387} → validation 
happens BEFORE fetch — validates against pinned schema ✓ (the _ErrorClient never
gets fetched... wait validation is local, then fetch raises). ✓

Also `test_unknown_request_property_fails_before_fetch` — client.fetched == [] 
✓.

Now — one more consideration about `_DirectoryClient` default envelope: needs to
be schema-shaped for... only used as return value; planner returns it as-is. 
Minimal envelope fine.

Let me now also double-check the 
`test_metric_normalization_and_exact_profile_request` — wait: "Teaching Staff" 
alias → teaching_fte; "teaching staff" → same → dedupe ✓. Final 
["total_enrolments", "teaching_fte"] ✓. But — hmm, the scope rule with 
deps.user_question "": request_scope_problem("", "VC-POC-01", {...metrics...}) —
unknown behavior. Existing planner tests called it with the same empty 
user_question (all old planner tests had deps.user_question default "") and old 
test expectations held (e.g. VC-POC-01 with metrics passed through). If it 
returned a problem, old behavior would clean but non-headcount metrics survive →
same result. teaching_fte survives cleanup either way ✓. total_enrolments 
survives ✓.

But CAREFUL: what if request_scope_problem returns something truthy and cleaned 
removes nothing → `canonical = cleaned or canonical` → unchanged ✓.

### Now — the `converse` docstring mentions "agent may be None (pipeline path)" 
— already there ✓. Keep.

### Another subtle piece — main() previously: `agent = build_agent()` — removing
it changes behavior for `--stub` + claude_account: require_credentials still 
called — pipeline needs no model? Wait — the pipeline Stage 1 and Stage 3 call 
models (interpret_intent, narration)! The deterministic planner is Stage 2, but 
Stages 1/3 need a model. So require_credentials stays necessary ✓ — good that we
keep it.

### Double-check test_main_uses_pipeline_path_by_default monkeypatches 
require_credentials away ✓.

### Now, assemble everything. Also — the " amendment trace" mentions test node 
IDs exactly; ensure names match precisely:

- test_missing_market_definition_is_derived_without_model_hint ✓
- test_distance_is_conditional_on_distance_market ✓
- test_ambiguous_school_clarifies_without_fetch ✓
- test_metric_normalization_and_exact_profile_request ✓
- test_vc_poc_03_routes_selector_to_ratios ✓
- test_unknown_request_property_fails_before_fetch ✓
- test_peer_filters_require_explicit_nondefault_filter ✓
- test_segment_filters_reject_unknown_nested_key ✓
- test_schema_valid_emitter_bad_request_is_terminal ✓
- test_invalid_explicit_identity_bad_request_clarifies ✓
- test_planner_failure_is_not_unsupported (intent_boundary) ✓
- test_main_uses_pipeline_path_by_default (run_cli) ✓

Good.

### One more careful pass on schema validation vs the stub-pipeline tests in 
intent_boundary:

`test_ask_pipeline_intent_override_bypasses_model` — request {"acara_sml_id": 
40387, "metrics": ["total_enrolments", "teaching_fte"]} → validate ✓ → stub 
fetch: select_case("VC-POC-01", request) — the canonical fixture. Existing 
behavior identical → passes.

`test_narration_output_contains_prose_only` — static ✓.

`test_structured_intent_defaults` — static ✓.

`test_interpret_intent_*` — Stage 1, unchanged ✓.

`test_ask_pipeline_intent_override_excluded_topic_returns_unsupported` — planner
gate 1 ✓ (reason contains "public transport" — from exclusion map ✓ unchanged).

`test_ask_pipeline_intent_override_gated_question_returns_unsupported` ✓.

Now — writing files. Let me be careful with the full content of each.

Also — `test_intent_boundary.py` new import: `from harness.evidence import 
EvidenceError`. And `from harness.errors import TerminalError` inside the new 
test (or top-level). Top-level cleaner.

Let me write the recording client for intent_boundary:

```python
class _RecordingClient:
    """Wraps the stub client, recording exact fetch_evidence arguments."""

    def __init__(self, inner):
        self._inner = inner
        self.fetched: list[dict[str, Any]] = []

    async def school_directory(self, search=None, limit=25):
        return await self._inner.school_directory(search=search, limit=limit)

    async def input_catalog(self):
        return await self._inner.input_catalog()

    async def fetch_evidence(self, question_id, request):
        self.fetched.append({"question_id": question_id, "request": request})
        return await self._inner.fetch_evidence(question_id, request)
```

EvidenceClient.school_directory signature: (self, search=None, limit=25) ✓; 
fetch_evidence(question_id, request) ✓; input_catalog() ✓.

Now the new intent-boundary tests:

```python
def test_planner_failure_is_not_unsupported():
    """A planner/schema defect propagates out of ask() as TerminalError —
    it must never surface as an Unsupported user question."""
    intent = StructuredIntent(
        question_id="VC-POC-01",
        filters={"acara_sml_id": 40387, "not_a_request_field": 1},
    )
    deps = Deps(client=stub_client())
    with pytest.raises(TerminalError):
        asyncio.run(ask(deps, "q", intent_override=intent))
    assert deps.last_envelope is None
```

```python
def test_planner_clarification_bypasses_narration_and_answer_view():
    """The planner's clarification returns before the narration stage: no
    answer view is built and the typed clarification is what ask() returns."""
    intent = StructuredIntent(question_id="VC-POC-08", filters={"acara_sml_id": 
40387})
    deps = Deps(client=stub_client())
    output = asyncio.run(ask(deps, "q", intent_override=intent))
    assert isinstance(output, ClarificationNeeded)
    assert deps.answer_view is None
    assert [m.input for m in output.missing] == ["distance_km"]
```

```python
def test_normalized_schema_valid_request_reaches_evidence():
    """Requested labels are normalized and routed before dispatch; the request
    that reaches evidence is exactly the schema-valid one."""
    intent = StructuredIntent(
        question_id="VC-POC-01",
        filters={"acara_sml_id": 40387},
        requested_metrics=["Total Enrolments", "Teaching Staff"],
    )
    narration = NarrationOutput(answer="Enrolments run 612 in 2019 and 668 in 
2023.")
    client = _RecordingClient(stub_client())
    deps = Deps(client=client)
    output = asyncio.run(
        ask(deps, "q", intent_override=intent, narration_override=narration)
    )
    assert isinstance(output, RenderedAnswer)
    assert client.fetched == [
        {
            "question_id": "VC-POC-01",
            "request": {"acara_sml_id": 40387, "metrics": ["total_enrolments", 
"teaching_fte"]},
        }
    ]
```

```python
def test_unnormalizable_metric_stays_a_clarification():
    """A label that cannot be normalized to the pinned vocabulary is a
    caller-correctable clarification — never Unsupported."""
    intent = StructuredIntent(
        question_id="VC-POC-01",
        filters={"acara_sml_id": 40387},
        requested_metrics=["banana enrolments"],
    )
    deps = Deps(client=stub_client())
    output = asyncio.run(ask(deps, "q", intent_override=intent))
    assert isinstance(output, ClarificationNeeded)
    assert not isinstance(output, Unsupported)
    assert output.missing[0].allowed_values  # the pinned vocabulary is supplied
```

```python
def test_invalid_explicit_identity_stays_a_clarification():
    """A bad_request for a caller-supplied numeric identity is a correctable
    identity problem — returned as clarification, never Unsupported."""

    class _RejectingIdentityClient:
        async def school_directory(self, search=None, limit=25):
            return []

        async def input_catalog(self):
            return {}

        async def fetch_evidence(self, question_id, request):
            raise EvidenceError(
                "bad_request",
                "acara_sml_id 99999 is absent from the pinned school directory",
            )

    intent = StructuredIntent(question_id="VC-POC-01", filters={"acara_sml_id": 
99999})
    deps = Deps(client=_RejectingIdentityClient())
    output = asyncio.run(ask(deps, "q", intent_override=intent))
    assert isinstance(output, ClarificationNeeded)
    assert not isinstance(output, Unsupported)
```

Place these in a new section "4. planner boundary — clarifications, evidence, 
failures".

Wait — ClarificationNeeded import exists in intent_boundary ✓ (imported). 
Unsupported ✓. Need TerminalError + EvidenceError imports.

Now — write test_planner.py fully. Structure:

```
"""Stage 2 — deterministic planner tests. ..."""

imports...

stub_client, _ErrorClient, _HappyClient, _DirectoryClient

Sections:
1. Excluded topics (4 tests)
2. Registry-owned excluded (2)
3. Gated (2)
4. Unknown (2)
5. Missing inputs derived (renamed section): kept 2 + new derivation tests
6. Happy path (3)
7. School resolution (new tests)
8. Selector normalization & routing (new)
9. Schema validation & filter constraints (new)
10. Error-origin separation (updated + new)
11. _find_exclusion_text (3)
```

Full test list for the file:

1. test_excluded_topic_returns_unsupported
2. test_excluded_topic_reason_from_contract
3. test_excluded_topic_finance_hint
4. test_excluded_topic_without_hint_uses_fallback
5. test_excluded_question_id_returns_unsupported
6. test_excluded_question_reason_is_contract_message
7. test_gated_question_returns_unsupported
8. test_gated_question_reason_code_preserved
9. test_unknown_question_id_literal_returns_unsupported
10. test_arbitrary_unknown_question_id_returns_unsupported
11. test_missing_market_definition_is_derived_without_model_hint
12. test_missing_material_input_returns_clarification (kept — model hint; 
derivation still clarifies)
13. test_non_material_missing_input_is_ignored
14. test_distance_is_conditional_on_distance_market
15. test_clarifications_follow_pinned_policy_order
16. test_happy_path_returns_envelope
17. test_happy_path_records_tool_calls
18. test_school_resolution_records_search
19. test_ambiguous_school_clarifies_without_fetch
20. test_unmatched_school_clarifies_without_fetch
21. test_school_resolution_is_transactional
22. test_conflicting_explicit_id_and_mention_is_terminal
23. test_consistent_explicit_id_and_mention_proceeds
24. test_mention_order_preserved_and_duplicates_removed
25. test_comparison_with_one_distinct_school_clarifies
26. test_explicit_numeric_ids_skip_directory_preflight
27. test_metric_normalization_and_exact_profile_request
28. test_vc_poc_03_routes_selector_to_ratios
29. test_grade_labels_route_to_grades_field
30. test_unnormalizable_metric_clarifies_without_fetch
31. test_selector_labels_without_a_route_clarify
32. test_unknown_request_property_fails_before_fetch
33. test_peer_filters_require_explicit_nondefault_filter
34. test_explicit_peer_filter_dispatches
35. test_segment_filters_require_explicit_filter (absent/empty) — hmm, absent 
case: derivation clarifies. Add test_segment_filters_absent_clarifies? The named
one is reject_unknown_nested_key. I'll add:
   test_peer_filters_require_explicit_nondefault_filter (default-only + absent)
36. test_segment_filters_reject_unknown_nested_key
37. test_schema_valid_emitter_bad_request_is_terminal
38. test_invalid_explicit_identity_bad_request_clarifies
39. test_unanswerable_returns_unsupported (updated intent)
40. test_unknown_question_from_emitter_returns_unsupported
41. test_data_unavailable_raises_terminal_error
42. test_internal_error_raises_terminal_error
43. test_error_call_recorded_in_deps
44. test_find_exclusion_text_transport_hint
45. test_find_exclusion_text_unknown_hint_uses_fallback
46. test_find_exclusion_text_none_hint_uses_fallback

Good coverage.

Wait — #25: VC-POC-04, two mentions both resolving 40387: resolution succeeds → 
staged [40387] → two-distinct check pre-commit → clarify; 
deps.resolved_school_ids stays [] ✓; fetched [] ✓.

#26: VC-POC-03 explicit [99999, 40387] — no preflight (searches==[]) ✓ 
dispatched ✓.

#31: VC-POC-05 + metrics → clarify — wait, but check order: routing happens 
BEFORE derivation. VC-POC-05 filters {acara, market_definition: "sa2"} + 
requested_metrics ["total enrolments"] → routing: route None → clarify ✓ (no 
fetch) ✓.

#18 (kept): _HappyClient — one record per search ✓; mention "Some School" → 
40387. assertion `search_calls[0].args`? No — existing test asserts len==1 only 
✓.

#12 kept: VC-POC-05 stub_client... wait original uses stub_client() but no fetch
happens (clarify) ✓.

Now write _DirectoryClient with minimal envelope:

```python
class _DirectoryClient:
    """A directory-backed mock: per-search records, canned fetch result."""

    def __init__(
        self,
        directory: dict[str, list[dict]] | None = None,
        envelope: dict | None = None,
        fetch_error: EvidenceError | None = None,
    ):
        self.directory = directory or {}
        self._envelope = envelope or {
            "question_id": "VC-POC-01",
            "contract_version": "0.3.0",
            "evidence": [],
            "provenance": {},
            "confidence": {},
            "limitations": [],
            "result": {"data": []},
            "data_status": {"status": "complete"},
        }
        self._fetch_error = fetch_error
        self.searches: list[str] = []
        self.fetched: list[dict] = []

    async def school_directory(self, search: str, limit: int = 25) -> 
list[dict]:
        self.searches.append(search)
        return list(self.directory.get(search, []))

    async def fetch_evidence(self, question_id: str, request: dict) -> dict:
        self.fetched.append({"question_id": question_id, "request": request})
        if self._fetch_error is not None:
            raise self._fetch_error
        return self._envelope
```

Now — #15 policy order: VC-POC-07 with mention resolving → committed; but hold 
on — VC-POC-07 also routes? requested_metrics empty → no routing. Constraint: 
no. Derivation: required [acara_sml_id ✓ (attached), market_definition ✓ 
(filters), grades ✗] → [grades]; material [market_definition ✓, grades seen]; 
conditional distance_km: market_definition == "distance" ✓ and no distance_km → 
add. missing = [grades, distance_km] ✓.

But — wait: school resolution commits deps BEFORE derivation clarifies. On 
clarification return, deps has resolved ids committed. Is that OK? The 
resolution transaction succeeded; later clarification about grades doesn't 
invalidate identities (they're needed for the retry). Contract's transactional 
requirement: "a later failed match leaks no partial state" — about failed 
directory matches. Here all matched. Acceptable — and desirable (deps retains 
resolved identity for the follow-up turn).

#22 conflict test: explicit 40387 + "Beta Grammar" → 40392 → TerminalError; 
deps.resolved_school_ids [] ✓ (no commit in explicit branch) ✓.

#24 order test: VC-POC-03 mentions ["Beta Grammar", "Alpha College", "Beta 
Grammar"]: resolve BG→40392 staged [40392]; AC→40387 staged [40392, 40387]; BG 
again → 40392 dup → stays. two-distinct: VC-POC-03 constraint? constraints [] → 
no. commit → deps [40392, 40387] ✓. attach plural ✓. request {"acara_sml_ids": 
[40392, 40387]} ✓ fetched ✓.

#37: resolved [40387] committed, fetched raises bad_request → TerminalError; 
tool_calls has fetch with error_kind ✓.

Now — #33 test_peer_filters_require_explicit_nondefault_filter:

```python
def test_peer_filters_require_explicit_nondefault_filter():
    # The sector default alone does not define a peer group.
    intent = StructuredIntent(
        question_id="VC-POC-09",
        filters={"peer_filters": {"sectors": ["Catholic", "Independent"]}},
    )
    client = _DirectoryClient()
    deps = Deps(client=client)
    result = asyncio.run(plan_and_execute(intent, client, deps))
    assert isinstance(result, ClarificationNeeded)
    assert client.fetched == []

    # And absent peer filters clarify from the derived policy.
    intent = StructuredIntent(question_id="VC-POC-09")
    client = _DirectoryClient()
    deps = Deps(client=client)
    result = asyncio.run(plan_and_execute(intent, client, deps))
    assert isinstance(result, ClarificationNeeded)
    assert "peer_filters" in [m.input for m in result.missing]
    assert client.fetched == []
```

Hmm wait — absent case: derivation missing = ["peer_filters"] → clarify ✓ 
(required). allowed_values [] ✓.

But careful — the sectors list order: ["Catholic", "Independent"] vs default 
from registry defaultable ["Catholic", "Independent"] — sorted equal ✓. What 
about reversed ["Independent", "Catholic"]? sorted equal → still default-only → 
clarify. Hmm — is that right? Sectors filter equal-as-set to default → 
"sector-default-only". Fine.

#34 explicit peer filter dispatches:

```python
def test_explicit_peer_filter_dispatches():
    intent = StructuredIntent(
        question_id="VC-POC-09",
        filters={
            "peer_filters": {
                "sectors": ["Catholic", "Independent"],
                "school_types": ["Secondary"],
            }
        },
    )
    client = _DirectoryClient()
    deps = Deps(client=client)
    result = asyncio.run(plan_and_execute(intent, client, deps))
    assert isinstance(result, dict)
    assert client.fetched[0]["request"] == {
        "peer_filters": {"sectors": ["Catholic", "Independent"], "school_types":
["Secondary"]}
    }
```

Validation against pinned VC-POC-09 schema: peer_filters object, 
additionalProperties false, properties include sectors (array of strings) & 
school_types (array of strings) ✓; minProperties ≥1 ✓ (2 keys). Should validate.
RISK: maybe school_types items have enum? OD-03 says release-derived NOT 
enum-enforced in schema → shape only ✓.

#36 segment unknown nested key → constraint: segment_filters non-empty explicit 
→ pass → schema: unknown property "banana" → TerminalError ✓.

Now — one more on #11: allowed_values for market_definition from mapping — 
assert exact list ["distance", "lga", "postcode", "sa2"] ✓ catalog order.

Also #11: names order ["acara_sml_id", "market_definition"] — required list 
order from registry: ["acara_sml_id", "market_definition"] ✓.

Now — potential snag on #14's second part: distance missing clarification — 
[m.input for m in result.missing] == ["distance_km"] — derivation: required 
acara ✓ market_definition ✓; material market_definition ✓; conditional 
distance_km → add ✓. missing == ["distance_km"] ✓.

Now — `test_vc_poc_03_routes_selector_to_ratios` — wait, planner routing: 
VC-POC-03 metric_list null → mapping: sorted keys with prefix "VC-POC-03." → 
["VC-POC-03.ratios"] → field "ratios" ✓ vocabulary catalog ratios enum ✓. 
"Students per Teaching FTE" → "students per teaching fte" ↔ 
"students_per_teaching_fte" ✓.

#29 grades: VC-POC-02 metric_list null → mapping "VC-POC-02.grades" ✓ → "Year 6"
→ year_6 ✓.

Now — IMPORTANT: does `contract.questions()` include VC-POC-02 with metric_list 
key? Registry entries all have "metric_list" (null for most) ✓.

Now — one more potential landmine: `contract.registry()` might return the 
registry dict including "metric_lists" ✓ (registry file has metric_lists 
top-level).

Now — `contract.questions()` — is it keyed dict or list?? Signature `dict[str, 
dict[str, Any]]` → keyed ✓. Existing planner uses 
`contract.questions()[question_id]` ✓ and `.get` patterns.

### Double-check `Unsupported` constructor accepts reason_code kwarg ✓ 
(existing).

### Now — one more: planner gate ordering — old gate 5 (missing material from 
intent) is REMOVED and replaced by derivation (later, after assembly). Fine.

### Check: excluded-topic gate reads intent.missing? No ✓.

Now write the files.

Oh wait — one more thing. The `ask()` pipeline — does it pass 
`deps.user_question`? Deps.user_question — set by ask() from the question 
presumably (Stage 1/2). In planner tests we construct Deps directly without 
user_question — default "". Existing code had the same pattern ✓.

### Also — `harness/tools.py` Deps — resolved_school_ids default list ✓ 
(existing planner appends).

### One more potential issue — `deps.resolved_school_ids` might be a plain list 
attribute; mutating in place ✓.

Now — final content writing. Also keep `from harness.evidence import 
EvidenceClient, EvidenceError` in planner (EvidenceClient for typing).

Also note — `plan_and_execute` type hints unchanged.

Now let me also make sure planner handles `intent.filters` containing extra 
harness keys gracefully... fail-closed via schema. OK.

Let me also reconsider: should validation happen even when... yes always before 
fetch.

One more: `_route_selectors` — uses deps.user_question — signature 
`_route_selectors(question_id, intent, request, deps)` ✓.

Edge: `intent.requested_metrics` empty list → skip ✓.

Now writing all five files. Let me also update test module docstrings.

For run.py — the diff:
1. Docstring: add pipeline-default bullet.
2. Import line: `from harness.agent import ask  # noqa: E402`.
3. main(): drop `agent = build_agent()`; pass None first arg with comment.

Everything else identical.

Wait — check converse's legacy docstring mentions agent param — keep.

One more run.py consideration: `converse(None, deps, ...)` — the for-loop with 
`max_clarifications + 1` iterations; pipeline ask signature `ask(deps, reply, 
record_usage=record_usage)` ✓ unchanged.

Let me now also review: in main, output rendering & exit codes unchanged ✓.

Final check on imports in run.py — `Callable` still used (converse signature) ✓;
`Any` used ✓.

Now write everything.

Actually — one more critical review of the TWO-DISTINCT pre-commit check 
placement: in the mentions branch I check `len(resolved_ids) < 2` — resolved_ids
is already deduped ✓.

And in the explicit-plural branch: `_dedupe(ids)` ✓.

Also `_requires_two_distinct_ids(entry)` — entry from contract.questions() — its
input_policy.constraints contains "acara_sml_ids contains at least two values" ✓
→ "at least two" ✓.

Now — hmm, `_insufficient_schools_clarification` message mentions head-to-head —
generic enough: "This comparison needs at least two distinct schools." Use 
generic wording.

Now — write out planner.py fully. Also keep `logger.debug("intent classified as 
excluded topic", ...)` etc. exactly.

One more inspection of old gate 3: 

```python
    if question_id in contract.questions():
        refusal = contract.refusal_for(question_id)
```

Keep.

Old gate 4:

```python
    if question_id not in contract.questions() and question_id not in 
contract.excluded_questions():
```

Keep.

Now — the final full files. Let me carefully produce them.

planner.py — assemble in order:
1. Docstring
2. Imports
3. Constants (exclusion map, aliases, etc.)
4. _find_exclusion_text
5. Pinned artefact loaders
6. Static vocabulary helpers
7. Normalization helpers
8. Schema validator
9. School resolution helpers
10. Scalar/routing/constraint/derivation helpers
11. plan_and_execute
12. _material_why
13. _branch_evidence_error

Let me write it.

Also — for `test_selector_labels_without_a_route_clarify` — VC-POC-05 
requested_metrics ["total enrolments"] → clarify. why_material mentions metric 
selector. ✓

And one more: make sure `_route_selectors` clarify happens BEFORE 
`_enforce_explicit_filter_constraints` — yes (step 8 then 9). And before 
derivation — yes.

Now — a subtle check on `test_metric_normalization_and_exact_profile_request` — 
`_DirectoryClient()` — validation against pinned VC-POC-01 schema: request 
{"acara_sml_id": 40387, "metrics": ["total_enrolments","teaching_fte"]} → 
metrics enum contains both ✓ → validates → fetch ✓.

Now — also confirm `_explicit_id_values` for VC-POC-01 explicit path: 
school_required ["acara_sml_id"]; `identity_from_caller` True; mentions empty → 
skip resolution; two-distinct check skipped (not plural). ✓

Write files now. Also double check every usage of MissingInput keyword: 
MissingInput(input=..., why_material=..., allowed_values=...) ✓.

For the intent_boundary file — full content with additions; keep all existing 
tests byte-identical, add imports (TerminalError, EvidenceError) and the new 
section.

For test_run_cli.py — keep all, append two tests at the end under "wiring" 
section... Actually place `test_main_uses_pipeline_path_by_default` and 
`test_main_no_longer_builds_a_legacy_agent` in the "exit codes and wiring" 
section.

Note — in test_run_cli, the helper `answered()` and others unchanged.

Wait — one more thing in test_run_cli: existing import `from harness.agent 
import ask, build_agent` — used by answered() ✓ keep.

Also — `PROJECT_ROOT` in test_run_cli for source check: 
`(Path(__file__).resolve().parents[2] / "run.py")` ✓.

Let me also double check the source-check test won't trip on the word 
"build_agent" appearing in a comment/docstring — I must ensure run.py contains 
NO occurrence of "build_agent" anywhere (including docstrings). I'll phrase 
docstrings without it ("legacy agent" is fine).

Similarly the converse docstring: "When `None`, the new 4-stage pipeline is 
used." — contains no build_agent ✓. Wait, current converse docstring says: 
"`agent` may be a PydanticAI `Agent` (legacy tool-loop path) or `None` (pipeline
path)." ✓ no build_agent.

Main currently has `agent = build_agent()` — removing. Any other mention? No.

Now — writing final answer with 5 file listings.

Let me write planner.py completely now, mentally compiling for syntax errors.

```python
"""Stage 2 of the 4-stage pipeline: deterministic planning and execution.

`plan_and_execute()` takes a StructuredIntent and returns either:
- a dict (the evidence envelope) — the happy path
- ClarificationNeeded — an input the caller must supply or correct
- Unsupported — an exclusion or an unknown question

No model calls. Every branch is a pure function of the intent, the pinned
contract artefacts, and the evidence client's response. The evidence call is
the only I/O besides the directory preflights.

What Stage 2 now guarantees (work item 20260908T113118):

- Required, material, and applicable conditional-material inputs are derived
  from the pinned question policy and the assembled request. The intent
  model's `missing_material_inputs` claim is never trusted.
- Named schools resolve through the pinned directory and must match exactly
  one record each. IDs are staged in mention order, deduplicated without
  reordering, and committed to `Deps` only after every mention resolves; a
  zero or ambiguous match clarifies with no evidence call and no partial
  identity mutation. Explicit caller-supplied numeric IDs bypass the
  preflight (bounded by the pinned Definition-Pack conflict) and conflict
  checks against name-resolved identities fail closed.
- Selector labels are Unicode-case-folded and separator-normalized, matched
  against pinned static vocabularies (registry `metric_list` where declared;
  otherwise the input-catalog schema's field mapping and embedded enums), and
  routed to the question's own request field: `metrics` for VC-POC-01/04/11,
  `ratios` for VC-POC-03, `grades` for VC-POC-02/07. Unsupplied selectors are
  omitted — the emitter keeps default ownership.
- Every request is validated against its registered pinned per-question
  schema before dispatch. A planner-produced request that cannot satisfy its
  schema raises TerminalError — planner defects are never dressed up as
  Unsupported user outcomes.
- Error origins are separated: a bad_request against a request whose school
  identity was caller-supplied is an identity clarification; any other
  bad_request after deterministic validation is planner/contract drift and
  raises TerminalError. excluded/unknown/unanswerable keep their typed
  abstentions; data_unavailable/internal remain operational stops.
"""
```

Then code.

For the schema-validator docstring, mention draft-2020-12 subset.

One more scan for syntax pitfalls: nested function `_check` inside 
`_schema_problems` defined after use of value/path — fine.

Also — `f"{path}: needs at least {min_items} items"` fine.

Now — final review of planner imports: `from harness.evidence import 
EvidenceClient, EvidenceError` ✓; EvidenceClient used in type hints ✓.

Let me also double check `_normalize_scalar_selectors` for VC-POC-06/07 — 
mapping has their market_definition entries ✓.

One thing — `_static_vocabulary_for_field(question_id, "market_definition")` for
VC-POC-08? No mapping entry → None → skip normalization ✓ (VC-POC-08 has no 
market_definition field).

Now — writing test_planner.py. Section 5 heading: "5. Derived missing inputs → 
clarification".

Also — in test file, `from harness.planner import _find_exclusion_text, 
plan_and_execute` ✓ (keep only what's used).

Also keep `import pytest` ✓ and TerminalError ✓.

Let me also make sure that in 
`test_consistent_explicit_id_and_mention_proceeds`, request sent: filters acara 
40387; mention resolves 40387 → equal → deps.searched True; NO attach (explicit 
present — attach only in else branch); request {"acara_sml_id": 40387} ✓ fetched
✓ assert deps.resolved_school_ids == []? In consistent branch we don't commit — 
resolved ids equal explicit; commit skipped. Assert `deps.resolved_school_ids ==
[]` and `deps.searched is True`. Fine.

Now — writing everything. One more triple-check of the acceptance focused 
command: it references only existing test functions we've defined ✓.

Also the plan requires: "Expand tests/unit/test_planner.py with ... exact 
request dictionaries" ✓; "static allowed values" ✓ (clarification allowed_values
assertions).

Alright — also the plan's run.py test: "Keep existing legacy-loop tests 
unchanged" ✓.

Time to write the final response. Full file listings for:
- harness/planner.py
- run.py
- tests/unit/test_planner.py
- tests/unit/test_intent_boundary.py
- tests/unit/test_run_cli.py

Let me write each file carefully now.

PLANNER.PY full text:

```python
"""Stage 2 of the 4-stage pipeline: deterministic planning and execution.

... (docstring above)
"""

from __future__ import annotations

import json
import logging
import re
from functools import lru_cache
from pathlib import Path
from typing import Any

from harness import contract
from harness.errors import TerminalError
from harness.evidence import EvidenceClient, EvidenceError
from harness.intent import StructuredIntent
from harness.outputs import ClarificationNeeded, MissingInput, Unsupported
from harness.tools import Deps, ToolInvocation, request_scope_problem

logger = logging.getLogger("harness.planner")

# Pinned contract artefacts. Static vocabularies and per-question schemas are
# read from these vendored files only — never from the runtime emitter.
CONTRACT_DIR = Path(__file__).resolve().parents[1] / "contract"
_SCHEMA_DIR = CONTRACT_DIR / "question_schemas" / "v0.3.0"

# Keyword hints the intent model may use for excluded topics, mapped to a search
# term that appears in the corresponding contract exclusion text. The mapping is
# fuzzy: the planner searches for any exclusion text that contains the keyword.
_EXCLUSION_KEYWORD_MAP: dict[str, str] = { ... unchanged ... }

# The one pinned semantic alias (prompts/rules.md): for school-profile
# requests "teaching staff" means teaching_fte. Keyed by the normalized label.
_PINNED_SELECTOR_ALIASES: dict[str, str] = {"teaching staff": "teaching_fte"}

_SCHOOL_FIELDS = ("acara_sml_id", "acara_sml_ids")

# Label-selector request fields `requested_metrics` can be routed into.
_SELECTOR_BAG_FIELDS = ("metrics", "ratios", "grades")

# The exact pinned conditional-material expressions (VC-POC-05/06/07), compared
# after the same normalization. Anything else is contract drift, not prose to
# parse heuristically.
_KNOWN_DISTANCE_CONDITIONS = frozenset(
    {
        "market definition equals distance",
        "market definition distance",
    }
)

_SEPARATOR_RE = re.compile(r"[\W_]+", re.UNICODE)
```

... (functions as drafted) ...

Let me now write out the complete plan_and_execute with gates verbatim from 
original.

I realize I should double check one thing — in the original, gate 2 uses 
`contract.excluded_questions().get(question_id)` — returns dict with 
reason_code/message ✓ keep.

Everything set. Also the returned Unsupported in gate 4 for literal "unknown": 
unchanged.

Now — produce final files.

One more consideration on `_route_selectors` for consistency: labels filter `if 
str(l).strip()` — then normalize str(label). A whitespace-only label is dropped 
silently — fine.

Also `intent.requested_metrics` could be None? Pydantic default [] — safe.

Writing test_run_cli additions:

```python
def test_main_uses_pipeline_path_by_default(monkeypatch):
    """The default CLI wires converse() to the deterministic pipeline: the
    first argument is None (no legacy agent) and main never builds one."""
    calls: dict[str, Any] = {}

    async def fake_converse(agent, deps, question, **kwargs):
        calls["agent"] = agent
        calls["question"] = question
        return Unsupported(reason_kind="not_in_registry", reason="no such 
question")

    monkeypatch.setattr(cli, "converse", fake_converse)
    monkeypatch.setattr(cli, "require_credentials", lambda: None)
    monkeypatch.setattr(cli, "build_client", lambda use_stub: stub_client())
    exit_code = cli.main(["--stub", "Which schools are near Alpha College?"])
    assert exit_code == cli.EXIT_UNSUPPORTED
    assert calls["agent"] is None
    assert calls["question"] == "Which schools are near Alpha College?"


def test_main_no_longer_builds_a_legacy_agent():
    """The production wiring must not construct the legacy agent: run.py no
    longer references it at all (the same source-level check the credential
    test uses)."""
    source = (PROJECT_ROOT / "run.py").read_text("utf-8")
    assert "build_agent" not in source
```

Both placed after test_every_output_branch_has_an_exit_code in the wiring 
section.

Hmm — one concern: `monkeypatch.setattr(cli, "converse", fake_converse)` — main 
calls `converse(` — module-level lookup ✓.

Another: `cli.main` → `configure_logging()` each call — fine.

Now — intent_boundary new imports at top: add `from harness.errors import 
TerminalError` and `from harness.evidence import EvidenceError`.

Alright — also let me reconsider one risk in test 2 (clarification bypass): 
`[m.input for m in output.missing] == ["distance_km"]` — VC-POC-08 policy: 
required [acara_sml_id, distance_km]; material [distance_km]. filters acara → 
missing = [distance_km] ✓.

And deps.answer_view — attribute must exist on Deps. 
test_ask_pipeline_stores_answer_view_on_deps does `isinstance(deps.answer_view, 
AnswerView)` after success — attribute exists ✓ default None presumably ✓.

Now — final: write everything out. Also ensure no tabs, consistent style.

For planner gate sections, copy original code verbatim for gates 1–4.

Let me also think — should `_route_selectors` name in MissingInput for the 
no-route case be "metrics"? Yes.

Double-check `_selector_route` iterates `sorted(_field_to_catalog_list())` — 
deterministic ✓.

One more — `_selector_route` registry branch: 
`contract.registry().get("metric_lists", {})` — registry() returns full dict ✓.

Now — write out. Also in planner, keep type hints consistent.

Final check — did I miss the plan item "Record a failed evidence invocation 
before branching" ✓; "Record every preflight" ✓.

And "no evidence call on pre-dispatch clarification paths" ✓ (all clarifications
return before fetch).

Writing the response now.

Wait — one more possible snag: `_validate_request` runs 
`_request_schema(question_id)` which requires `_schema_index()` and the schema 
file. For VC-POC-REF-01: filename "VC-POC-REF-01.request.schema.json" == 
f"{qid}.request.schema.json" ✓.

And for question ids like "VC-POC-TRANSPORT-01" — gated out earlier ✓ never 
reaches validation.

One more — the schema files include "examples/" dir — not loaded ✓.

OK also — subtle: `_request_schema` lru_cache(maxsize=None) — 16 questions, 
fine.

Let me also double-check the planner handles `intent.filters` = {} for 
VC-POC-REF-01: required [] → missing [] → request {} → validation: schema type 
object, no required → ✓ → fetch with {} ✓. REF-01 metric_list null → route None 
✓.

Now finalize.

Actually — one more thought about `_adapt_explicit_school_fields` raising 
TerminalError for plural→singular with ≠1 ids. What if the caller passes 
acara_sml_ids for VC-POC-01 with e.g. [40387, 40392]? TerminalError "cannot 
honestly represent". Reasonable.

Also — `del request["acara_sml_ids"]` after moving ✓.

And — a caller passing BOTH acara_sml_id and acara_sml_ids for a plural-required
question: adaptation only fires if plural absent. Both present → plural wins; 
the stray singular remains → schema additionalProperties false → TerminalError. 
Fail-closed ✓.

Now writing the final response. I'll keep explanations brief before the 
listings.

Also — one very last check on test file: `test_error_call_recorded_in_deps` — 
VC-POC-01 id 40387 bad_request → identity_from_caller True → 
ClarificationNeeded; the test asserts tool_calls only ✓ — but the result is a 
clarification; test doesn't assert type — keep as is (rename? keep, it still 
holds).

Also — the old `test_bad_request_returns_unsupported` is deleted; nothing 
references it.

One more consideration — `_HappyClient.school_directory` returns a record even 
for questions... fine.

In `test_mention_order_preserved_and_duplicates_removed`, VC-POC-03 validation: 
request {"acara_sml_ids": [40392, 40387]} uniqueItems ✓ minItems 1 ✓.

Done deliberating — write it all.

------------
► **ANSWER**

I've reviewed the plan, acceptance contract and the current files. Here's what 
needs to change:



1. **`harness/planner.py`** — derive missing 
required/material/conditional-material inputs from the pinned policy (ignoring 
the model's claim), make school resolution transactional with exactly-one-match 
semantics, normalize selector labels against pinned static vocabularies 
(registry `metric_list`, input-catalog field mapping/enums), route selectors to 
the question's own request field (`ratios` for VC-POC-03, `grades` for 
VC-POC-02/07), validate every request against its registered pinned schema 
before dispatch, enforce the VC-POC-09/10 explicit-filter rule, and separate 
`bad_request` origins (caller identity → clarification; planner/schema drift → 
`TerminalError`).

2. **`run.py`** — the default CLI must run the deterministic pipeline: 
`converse(None, deps, ...)` with no legacy agent constructed.

3. **Tests** — add the named acceptance tests and update tests that encoded the 
old first-match/`out_of_scope` behaviours.



harness/planner.py

```diff
@@ -1,38 +1,66 @@
 """Stage 2 of the 4-stage pipeline: deterministic planning and execution.
 
 `plan_and_execute()` takes a StructuredIntent and returns either:
 - a dict (the evidence envelope) — the happy path
-- ClarificationNeeded — a material input is missing
+- ClarificationNeeded — an input the caller must supply or correct
 - Unsupported — an exclusion or an unknown question
 
-No model calls. Every branch is a pure function of the intent and the contract.
-The evidence call itself is the only I/O, and its error handling mirrors the
-tool layer's `_branch_on_error_kind()` exactly: the same closed set of 
outcomes,
-the same reason codes, the same contract text.
-
-Why reproduce `_branch_on_error_kind` here rather than calling it? Because the
-tool layer raises `ModelRetry`, `UnsupportedSignal`, and `RepairCapExceeded` —
-exceptions designed for PydanticAI's agent loop. The pipeline has no loop, so
-those signals have no handler; they would escape to the caller as unclassified
-RuntimeErrors. The planner translates the same error semantics into the 
pipeline's
-typed return values instead.
+No model calls. Every branch is a pure function of the intent, the pinned
+contract artefacts, and the evidence client's responses.
+
+What Stage 2 now guarantees (work item 20260908T113118):
+
+- Required, material, and applicable conditional-material inputs are derived
+  from the pinned question policy and the assembled request. The intent
+  model's `missing_material_inputs` claim is never trusted.
+- Named schools resolve through the pinned directory and must match exactly
+  one record each. IDs are staged in mention order, deduplicated without
+  reordering, and committed to `Deps` only after every mention resolves; a
+  zero or ambiguous match clarifies with no evidence call and no partial
+  identity mutation. Caller-supplied numeric IDs bypass the preflight (the
+  bounded resolution of the pinned Definition-Pack conflict) and any conflict
+  with name-resolved identities fails closed before dispatch.
+- Selector labels are Unicode-case-folded and separator-normalized, matched
+  against pinned static vocabularies (registry `metric_list` where declared;
+  otherwise the input-catalog schema's field mapping and embedded enums), and
+  routed to the question's own request field: `metrics` for VC-POC-01/04/11,
+  `ratios` for VC-POC-03, `grades` for VC-POC-02/07. Unsupplied selectors are
+  omitted — the emitter keeps default ownership (no invented defaults).
+- Every request is validated against its registered pinned per-question
+  schema before dispatch. A planner-produced request that cannot satisfy its
+  schema raises TerminalError — planner defects are never dressed up as
+  Unsupported user outcomes.
+- Error origins are separated: a bad_request against a request whose school
+  identity was caller-supplied is an identity clarification; any other
+  bad_request after deterministic validation is planner/contract drift and
+  raises TerminalError. excluded/unknown/unanswerable keep their typed
+  abstentions; data_unavailable/internal remain operational stops.
 """
 
 from __future__ import annotations
 
+import json
 import logging
+import re
+from functools import lru_cache
+from pathlib import Path
 from typing import Any
 
 from harness import contract
 from harness.errors import TerminalError
 from harness.evidence import EvidenceClient, EvidenceError
 from harness.intent import StructuredIntent
 from harness.outputs import ClarificationNeeded, MissingInput, Unsupported
 from harness.tools import Deps, ToolInvocation, request_scope_problem
 
 logger = logging.getLogger("harness.planner")
+
+# Pinned contract artefacts. Static vocabularies and per-question schemas are
+# read from these vendored files only — never from the runtime emitter.
+CONTRACT_DIR = Path(__file__).resolve().parents[1] / "contract"
+_SCHEMA_DIR = CONTRACT_DIR / "question_schemas" / "v0.3.0"
 
 # Keyword hints the intent model may use for excluded topics, mapped to a 
search
 # term that appears in the corresponding contract exclusion text. The mapping 
is
 # fuzzy: the planner searches for any exclusion text that contains the keyword.
 _EXCLUSION_KEYWORD_MAP: dict[str, str] = {
@@ -47,10 +75,32 @@
     "mesh_block": "Mesh Block",
     "sa2": "SA2",
     "gtfs": "GTFS",
 }
 
+# The one pinned semantic alias (prompts/rules.md): for school-profile
+# requests "teaching staff" means teaching_fte. Keyed by the normalized label,
+# so "Teaching Staff" and "teaching-staff" match the same alias.
+_PINNED_SELECTOR_ALIASES: dict[str, str] = {"teaching staff": "teaching_fte"}
+
+_SCHOOL_FIELDS = ("acara_sml_id", "acara_sml_ids")
+
+# Label-selector request fields the requested label bag can be routed into.
+_SELECTOR_BAG_FIELDS = ("metrics", "ratios", "grades")
+
+# The exact pinned conditional-material expressions (VC-POC-05/06/07), compared
+# after separator normalization. Anything else is contract drift, not prose to
+# parse heuristically.
+_KNOWN_DISTANCE_CONDITIONS = frozenset(
+    {
+        "market definition equals distance",
+        "market definition distance",
+    }
+)
+
+_SEPARATOR_RE = re.compile(r"[\W_]+", re.UNICODE)
+
 
 def _find_exclusion_text(excluded_topic: str | None) -> str:
     """Find the contract exclusion text matching a topic hint.
 
     Searches the exclusion texts for a case-insensitive substring match on the
@@ -77,23 +127,726 @@
 
     # Fallback: first exclusion text alphabetically
     return texts[0]
 
 
+# --------------------------------------------------------------------------- #
+# Pinned artefact loading (fail-closed: load failures are TerminalError)
+# --------------------------------------------------------------------------- #
+
+
+def _load_pinned_json(path: Path) -> Any:
+    try:
+        return json.loads(path.read_text(encoding="utf-8"))
+    except (OSError, ValueError) as exc:
+        raise TerminalError(
+            f"pinned contract artefact could not be loaded ({path.name}): 
{exc}"
+        ) from exc
+
+
+@lru_cache(maxsize=1)
+def _schema_index() -> dict[str, Any]:
+    index = _load_pinned_json(_SCHEMA_DIR / "_index.json")
+    if not isinstance(index, dict) or not isinstance(index.get("questions"), 
dict):
+        raise TerminalError("pinned question-schema index is malformed")
+    return index
+
+
+@lru_cache(maxsize=1)
+def _input_catalog_schema() -> dict[str, Any]:
+    schema = _load_pinned_json(CONTRACT_DIR / 
"poc_input_catalog.v0.3.0.schema.json")
+    if not isinstance(schema, dict):
+        raise TerminalError("pinned input-catalog schema is malformed")
+    return schema
+
+
+def _registered_request_schema_path(question_id: str) -> Path:
+    """Resolve the question's registered request-schema path.
+
+    Only paths registered in the pinned index, beneath the pinned contract
+    directory, may be loaded — anything else is contract drift.
+    """
+    entry = _schema_index()["questions"].get(question_id)
+    if not isinstance(entry, dict):
+        raise TerminalError(f"{question_id} has no registered request schema")
+    filename = entry.get("request_schema")
+    if not isinstance(filename, str) or filename != 
f"{question_id}.request.schema.json":
+        raise TerminalError(
+            f"{question_id} request-schema registration is inconsistent: 
{filename!r}"
+        )
+    path = (_SCHEMA_DIR / filename).resolve()
+    pinned = CONTRACT_DIR.resolve()
+    if pinned not in path.parents:
+        raise TerminalError(
+            "refusing to load a request schema from outside the pinned contract
"
+            f"directory: {filename}"
+        )
+    return path
+
+
+@lru_cache(maxsize=None)
+def _request_schema(question_id: str) -> dict[str, Any]:
+    schema = _load_pinned_json(_registered_request_schema_path(question_id))
+    if not isinstance(schema, dict):
+        raise TerminalError(f"{question_id} request schema is malformed")
+    return schema
+
+
+# --------------------------------------------------------------------------- #
+# Static vocabularies (registry metric_list; catalog field mapping + enums)
+# --------------------------------------------------------------------------- #
+
+
+@lru_cache(maxsize=1)
+def _field_to_catalog_list() -> dict[str, str]:
+    mapping = 
_input_catalog_schema().get("x-poc-request-field-to-catalog-list")
+    if not isinstance(mapping, dict):
+        return {}
+    return {
+        str(field): str(list_name)
+        for field, list_name in mapping.items()
+        if isinstance(field, str) and isinstance(list_name, str)
+    }
+
+
+def _static_enum(list_name: str) -> list[str] | None:
+    """The embedded static enum for a catalog list, or None when the list is
+    release-derived (shape-checked only, never enum-enforced — OD-03)."""
+    if list_name.startswith("release_derived_values."):
+        return None
+    prop = _input_catalog_schema().get("properties", {}).get(list_name)
+    if isinstance(prop, dict) and isinstance(prop.get("items"), dict):
+        enum = prop["items"].get("enum")
+        if isinstance(enum, list):
+            return [str(value) for value in enum]
+    return None
+
+
+def _static_vocabulary_for_field(question_id: str, field: str) -> list[str] | 
None:
+    list_name = _field_to_catalog_list().get(f"{question_id}.{field}")
+    if list_name is None:
+        return None
+    return _static_enum(list_name)
+
+
+def _selector_route(question_id: str) -> tuple[str, list[str]] | None:
+    """(request field, static vocabulary) for the question's label-selector 
bag.
+
+    The registry's `metric_list` is authoritative where declared (VC-POC-01/04
+    → profile_metrics, VC-POC-11 → student_context_metrics); otherwise the
+    input-catalog schema's field mapping routes e.g. VC-POC-03 to `ratios` and
+    VC-POC-02/07 to `grades`.
+    """
+    entry = contract.questions().get(question_id) or {}
+    metric_list = entry.get("metric_list")
+    if metric_list:
+        values = contract.registry().get("metric_lists", {}).get(metric_list)
+        if not isinstance(values, list) or not values:
+            raise TerminalError(
+                f"{question_id} declares metric_list {metric_list!r} with no 
pinned values"
+            )
+        return "metrics", [str(value) for value in values]
+    field_to_list = _field_to_catalog_list()
+    for key in sorted(field_to_list):
+        prefix = f"{question_id}."
+        if not key.startswith(prefix):
+            continue
+        field = key[len(prefix):]
+        if field in _SELECTOR_BAG_FIELDS:
+            vocabulary = _static_enum(field_to_list[key])
+            if vocabulary is None:
+                raise TerminalError(
+                    f"{question_id}.{field} routes to a non-static vocabulary"
+                )
+            return field, vocabulary
+    return None
+
+
+# --------------------------------------------------------------------------- #
+# Label normalization
+# --------------------------------------------------------------------------- #
+
+
+def _normalize_label(label: Any) -> str:
+    """Unicode-safe case folding plus separator normalization.
+
+    `total_enrolments`, `Total Enrolments` and `total-enrolments` all fold to
+    `total enrolments`; Unicode letters survive the fold.
+    """
+    folded = _SEPARATOR_RE.sub(" ", str(label).casefold())
+    return folded.strip()
+
+
+def _canonical_for(label: Any, vocabulary: list[str]) -> str | None:
+    """Match a label against a pinned static vocabulary.
+
+    Direct normalized matches win; beyond that only the pinned semantic
+    aliases apply. Canonical values are returned unchanged.
+    """
+    wanted = _normalize_label(label)
+    for value in vocabulary:
+        if _normalize_label(value) == wanted:
+            return value
+    alias = _PINNED_SELECTOR_ALIASES.get(wanted)
+    if alias is not None and alias in vocabulary:
+        return alias
+    return None
+
+
+def _dedupe(values: list[Any]) -> list[Any]:
+    """Order-preserving deduplication."""
+    seen: set[Any] = set()
+    unique: list[Any] = []
+    for value in values:
+        if value not in seen:
+            seen.add(value)
+            unique.append(value)
+    return unique
+
+
+# --------------------------------------------------------------------------- #
+# Pinned-schema validation (draft 2020-12 subset, dependency-free)
+# --------------------------------------------------------------------------- #
+
+
+def _json_key(value: Any) -> str:
+    return json.dumps(value, sort_keys=True, ensure_ascii=False, default=str)
+
+
+def _type_matches(value: Any, expected: str) -> bool:
+    if expected == "object":
+        return isinstance(value, dict)
+    if expected == "array":
+        return isinstance(value, list)
+    if expected == "string":
+        return isinstance(value, str)
+    if expected == "integer":
+        if isinstance(value, bool):
+            return False
+        if isinstance(value, int):
+            return True
+        return isinstance(value, float) and value.is_integer()
+    if expected == "number":
+        return isinstance(value, (int, float)) and not isinstance(value, bool)
+    if expected == "boolean":
+        return isinstance(value, bool)
+    if expected == "null":
+        return value is None
+    return True  # unknown type keyword: leave to the remaining checks
+
+
+def _schema_problems(value: Any, schema: Any, path: str) -> list[str]:
+    """Validate `value` against the pinned request schema.
+
+    Supports the draft-2020-12 keywords the generated v0.3 request schemas
+    use: types, required/properties/additionalProperties, items, enum/const,
+    numeric bounds, cardinality, uniqueness, patterns, string lengths, and the
+    allOf/anyOf/oneOf and if/then/else combinators. Annotation keywords are
+    ignored.
+    """
+    problems: list[str] = []
+    if not isinstance(schema, dict):
+        return problems
+
+    expected_type = schema.get("type")
+    if expected_type is not None:
+        expected_types = expected_type if isinstance(expected_type, list) else 
[expected_type]
+        if not any(_type_matches(value, str(t)) for t in expected_types):
+            problems.append(
+                f"{path}: expected type {expected_type!r}, got 
{type(value).__name__}"
+            )
+            return problems
+
+    if "const" in schema and _json_key(value) != _json_key(schema["const"]):
+        problems.append(f"{path}: must equal {schema['const']!r}")
+
+    enum = schema.get("enum")
+    if isinstance(enum, list):
+        if not any(_json_key(value) == _json_key(candidate) for candidate in 
enum):
+            problems.append(f"{path}: {value!r} is not an allowed value")
+
+    if isinstance(value, dict):
+        properties = schema.get("properties")
+        properties = properties if isinstance(properties, dict) else {}
+        for key, sub_schema in properties.items():
+            if key in value:
+                problems.extend(_schema_problems(value[key], sub_schema, 
f"{path}.{key}"))
+        additional = schema.get("additionalProperties", True)
+        unknown = [key for key in value if key not in properties]
+        if unknown and additional is False:
+            problems.append(f"{path}: unknown properties {sorted(unknown)}")
+        elif isinstance(additional, dict):
+            for key in unknown:
+                problems.extend(_schema_problems(value[key], additional, 
f"{path}.{key}"))
+        required = schema.get("required")
+        if isinstance(required, list):
+            for key in required:
+                if key not in value:
+                    problems.append(f"{path}: missing required property 
{key!r}")
+        min_properties = schema.get("minProperties")
+        if isinstance(min_properties, int) and len(value) < min_properties:
+            problems.append(f"{path}: needs at least {min_properties} 
properties")
+        max_properties = schema.get("maxProperties")
+        if isinstance(max_properties, int) and len(value) > max_properties:
+            problems.append(f"{path}: allows at most {max_properties} 
properties")
+
+    if isinstance(value, list):
+        min_items = schema.get("minItems")
+        if isinstance(min_items, int) and len(value) < min_items:
+            problems.append(f"{path}: needs at least {min_items} items")
+        max_items = schema.get("maxItems")
+        if isinstance(max_items, int) and len(value) > max_items:
+            problems.append(f"{path}: allows at most {max_items} items")
+        if schema.get("uniqueItems"):
+            item_keys = [_json_key(item) for item in value]
+            if len(set(item_keys)) != len(item_keys):
+                problems.append(f"{path}: items must be unique")
+        items = schema.get("items")
+        if isinstance(items, dict):
+            for index, element in enumerate(value):
+                problems.extend(_schema_problems(element, items, 
f"{path}[{index}]"))
+
+    if isinstance(value, str):
+        pattern = schema.get("pattern")
+        if isinstance(pattern, str) and re.search(pattern, value) is None:
+            problems.append(f"{path}: does not match the required pattern")
+        min_length = schema.get("minLength")
+        if isinstance(min_length, int) and len(value) < min_length:
+            problems.append(f"{path}: shorter than the required minimum 
length")
+        max_length = schema.get("maxLength")
+        if isinstance(max_length, int) and len(value) > max_length:
+            problems.append(f"{path}: longer than the required maximum length")
+
+    if isinstance(value, (int, float)) and not isinstance(value, bool):
+        minimum = schema.get("minimum")
+        if isinstance(minimum, (int, float)) and value < minimum:
+            problems.append(f"{path}: below the minimum {minimum}")
+        maximum = schema.get("maximum")
+        if isinstance(maximum, (int, float)) and value > maximum:
+            problems.append(f"{path}: above the maximum {maximum}")
+
+    def _check(sub_schema: Any) -> list[str]:
+        return _schema_problems(value, sub_schema, path)
+
+    all_of = schema.get("allOf")
+    if isinstance(all_of, list):
+        for sub_schema in all_of:
+            problems.extend(_check(sub_schema))
+    any_of = schema.get("anyOf")
+    if isinstance(any_of, list) and any_of:
+        if all(len(_check(sub_schema)) > 0 for sub_schema in any_of):
+            problems.append(f"{path}: matches none of the allowed variants")
+    one_of = schema.get("oneOf")
+    if isinstance(one_of, list) and one_of:
+        matches = sum(1 for sub_schema in one_of if not _check(sub_schema))
+        if matches != 1:
+            problems.append(
+                f"{path}: must match exactly one allowed variant (matched 
{matches})"
+            )
+    if "if" in schema:
+        condition_holds = not _check(schema["if"])
+        branch = schema.get("then") if condition_holds else schema.get("else")
+        if isinstance(branch, dict):
+            problems.extend(_check(branch))
+    not_schema = schema.get("not")
+    if isinstance(not_schema, dict) and not _check(not_schema):
+        problems.append(f"{path}: matches a forbidden variant")
+
+    return problems
+
+
+def _validate_request(question_id: str, request: dict[str, Any]) -> None:
+    """Validate the planner-built request against its registered pinned schema.
+
+    Any failure — including schema load or resolution failure — is a
+    TerminalError before dispatch, never an Unsupported user outcome.
+    """
+    schema = _request_schema(question_id)
+    problems = _schema_problems(request, schema, "$request")
+    if problems:
+        raise TerminalError(
+            f"{question_id}: the planner-built request does not satisfy its "
+            f"registered request schema: " + "; ".join(problems)
+        )
+
+
+# --------------------------------------------------------------------------- #
+# School identity resolution (transactional, exactly-one-match)
+# --------------------------------------------------------------------------- #
+
+
+async def _resolve_mentions(
+    question_id: str,
+    mentions: list[str],
+    client: EvidenceClient,
+    deps: Deps,
+) -> tuple[list[int], ClarificationNeeded | None]:
+    """Resolve each mention through the pinned directory, exactly one match 
each.
+
+    Every preflight is recorded, including failures. IDs are staged locally in
+    mention order and deduplicated without reordering; nothing is committed
+    here — the caller commits to `Deps` only after every mention resolves.
+    """
+    staged: list[int] = []
+    for mention in mentions:
+        arguments = {"search": mention, "limit": 25}
+        try:
+            records = await client.school_directory(search=mention, limit=25)
+        except Exception as exc:
+            deps.tool_calls.append(
+                ToolInvocation("search_schools", arguments, 
error_kind=type(exc).__name__)
+            )
+            raise
+        deps.tool_calls.append(ToolInvocation("search_schools", arguments, 
records))
+        ids = [
+            record["acara_sml_id"]
+            for record in records
+            if isinstance(record, dict)
+            and isinstance(record.get("acara_sml_id"), int)
+            and not isinstance(record["acara_sml_id"], bool)
+        ]
+        if not ids:
+            return [], _school_match_clarification(
+                question_id,
+                mention,
+                f"'{mention}' matched no school in the pinned directory.",
+                "I couldn't find that school. Could you check the name — or 
give me "
+                "the school's public ACARA ID?",
+            )
+        if len(ids) > 1:
+            return [], _school_match_clarification(
+                question_id,
+                mention,
+                f"'{mention}' matched {len(ids)} schools; a request needs 
exactly one.",
+                f"More than one school matches '{mention}'. Which one did you 
mean?",
+            )
+        school_id = ids[0]
+        if school_id not in staged:
+            staged.append(school_id)
+    return staged, None
+
+
+def _school_match_clarification(
+    question_id: str, mention: str, why_material: str, prompt_to_user: str
+) -> ClarificationNeeded:
+    return ClarificationNeeded(
+        question_id=question_id,
+        missing=[
+            MissingInput(input="acara_sml_id", why_material=why_material, 
allowed_values=[])
+        ],
+        prompt_to_user=prompt_to_user,
+    )
+
+
+def _adapt_explicit_school_fields(request: dict[str, Any], school_required: 
list[str]) -> None:
+    """Let a caller-supplied identity in either school field satisfy the
+    question's own selector field, without inventing values."""
+    if not school_required:
+        return
+    if "acara_sml_ids" in school_required:
+        if "acara_sml_ids" not in request and 
isinstance(request.get("acara_sml_id"), int):
+            request["acara_sml_ids"] = [request.pop("acara_sml_id")]
+    elif "acara_sml_id" in school_required:
+        if "acara_sml_id" not in request and 
isinstance(request.get("acara_sml_ids"), list):
+            ids = request["acara_sml_ids"]
+            if len(ids) == 1 and isinstance(ids[0], int) and not 
isinstance(ids[0], bool):
+                request["acara_sml_id"] = ids[0]
+                del request["acara_sml_ids"]
+            else:
+                raise TerminalError(
+                    "caller supplied multiple acara_sml_ids for a question 
whose schema "
+                    "takes a single acara_sml_id"
+                )
+
+
+def _attach_school_ids(
+    request: dict[str, Any], school_required: list[str], ids: list[int]
+) -> None:
+    if not ids:
+        return
+    if "acara_sml_ids" in school_required:
+        request.setdefault("acara_sml_ids", list(ids))
+    elif "acara_sml_id" in school_required:
+        request.setdefault("acara_sml_id", ids[0])
+
+
+def _explicit_id_values(request: dict[str, Any], school_required: list[str]) ->
list[int]:
+    if "acara_sml_ids" in school_required:
+        value = request.get("acara_sml_ids")
+        if isinstance(value, list):
+            return [v for v in value if isinstance(v, int) and not 
isinstance(v, bool)]
+        return []
+    if "acara_sml_id" in school_required:
+        value = request.get("acara_sml_id")
+        if isinstance(value, int) and not isinstance(value, bool):
+            return [value]
+    return []
+
+
+def _requires_two_distinct_ids(entry: dict[str, Any]) -> bool:
+    constraints = entry.get("input_policy", {}).get("constraints", [])
+    return any("at least two" in str(constraint) for constraint in constraints)
+
+
+def _insufficient_schools_clarification(question_id: str) -> 
ClarificationNeeded:
+    return ClarificationNeeded(
+        question_id=question_id,
+        missing=[
+            MissingInput(
+                input="acara_sml_ids",
+                why_material="This comparison needs at least two distinct 
schools.",
+                allowed_values=[],
+            )
+        ],
+        prompt_to_user="Which distinct schools should be compared? Name at 
least two.",
+    )
+
+
+# --------------------------------------------------------------------------- #
+# Selector normalization and routing
+# --------------------------------------------------------------------------- #
+
+
+def _normalize_scalar_selectors(
+    question_id: str, request: dict[str, Any]
+) -> ClarificationNeeded | None:
+    """Normalize static-closed scalar selectors (market_definition) against
+    their pinned vocabulary. Unnormalizable values clarify before dispatch."""
+    if "market_definition" not in request:
+        return None
+    value = request["market_definition"]
+    if not isinstance(value, str):
+        return None  # shape-checked by the pinned schema
+    vocabulary = _static_vocabulary_for_field(question_id, "market_definition")
+    if vocabulary is None:
+        return None
+    canonical = _canonical_for(value, vocabulary)
+    if canonical is None:
+        return ClarificationNeeded(
+            question_id=question_id,
+            missing=[
+                MissingInput(
+                    input="market_definition",
+                    why_material=(
+                        f"'{value}' is not an allowed market definition for 
{question_id}."
+                    ),
+                    allowed_values=vocabulary,
+                )
+            ],
+            prompt_to_user=(
+                "Which market definition should I use? Allowed values: "
+                + ", ".join(vocabulary)
+                + "."
+            ),
+        )
+    request["market_definition"] = canonical
+    return None
+
+
+def _route_selectors(
+    question_id: str,
+    intent: StructuredIntent,
+    request: dict[str, Any],
+    deps: Deps,
+) -> ClarificationNeeded | None:
+    """Route requested selector labels to the question's own request field.
+
+    Static vocabularies come from pinned artefacts (registry metric_list where
+    declared, otherwise the input-catalog schema's field mapping and embedded
+    enums) — never from the runtime emitter. Selectors the user named none of
+    are omitted so the emitter keeps default ownership.
+    """
+    labels = [str(label) for label in (intent.requested_metrics or []) if 
str(label).strip()]
+    if not labels:
+        return None
+    route = _selector_route(question_id)
+    if route is None:
+        return ClarificationNeeded(
+            question_id=question_id,
+            missing=[
+                MissingInput(
+                    input="metrics",
+                    why_material=(
+                        f"{question_id} takes no metric selector, so the 
requested "
+                        "labels cannot be applied to this request."
+                    ),
+                    allowed_values=[],
+                )
+            ],
+            prompt_to_user="This question does not take metric selectors. 
Should I answer it as asked?",
+        )
+    field, vocabulary = route
+    canonical: list[str] = []
+    for label in labels:
+        matched = _canonical_for(label, vocabulary)
+        if matched is None:
+            return ClarificationNeeded(
+                question_id=question_id,
+                missing=[
+                    MissingInput(
+                        input=field,
+                        why_material=(
+                            f"'{label}' is not an allowed {field} value for 
{question_id}."
+                        ),
+                        allowed_values=vocabulary,
+                    )
+                ],
+                prompt_to_user=(
+                    f"Which {field} did you mean? Allowed values: "
+                    + ", ".join(vocabulary)
+                    + "."
+                ),
+            )
+        canonical.append(matched)
+    canonical = _dedupe(canonical)
+
+    if field == "metrics":
+        # Existing raw-question scope rule: a user who never asked about
+        # headcounts or non-teaching staff gets neither, even if a label was
+        # requested. Applied before validation, as before.
+        scope_problem = request_scope_problem(
+            deps.user_question, question_id, {**request, "metrics": canonical}
+        )
+        if scope_problem:
+            cleaned = [
+                metric
+                for metric in canonical
+                if not (metric.endswith("_headcount") or 
metric.startswith("non_teaching_"))
+            ]
+            canonical = cleaned or canonical
+
+    request[field] = canonical
+    return None
+
+
+# --------------------------------------------------------------------------- #
+# Registry-owned explicit-filter constraints (VC-POC-09/10, OD-07)
+# --------------------------------------------------------------------------- #
+
+
+def _enforce_explicit_filter_constraints(
+    question_id: str, entry: dict[str, Any], request: dict[str, Any]
+) -> ClarificationNeeded | None:
+    """The registry-owned VC-POC-09/10 rule: a cohort definition must contain
+    at least one explicit filter — the sector default alone does not define a
+    peer group. Enforced exactly as pinned, not generalised into a condition
+    language. Absent fields are left to policy derivation."""
+    policy = entry.get("input_policy") or {}
+    required = [f for f in policy.get("required", []) if isinstance(f, str)]
+
+    if "peer_filters" in required and isinstance(request.get("peer_filters"), 
dict):
+        filters = request["peer_filters"]
+        if not filters:
+            return _cohort_definition_clarification(
+                question_id,
+                "peer_filters",
+                "The peer group definition is empty; at least one explicit 
filter "
+                "is required.",
+            )
+        default_sectors = policy.get("defaultable", 
{}).get("peer_filters.sectors")
+        if (
+            set(filters) == {"sectors"}
+            and isinstance(default_sectors, list)
+            and isinstance(filters.get("sectors"), list)
+            and sorted(str(s) for s in filters["sectors"])
+            == sorted(str(s) for s in default_sectors)
+        ):
+            return _cohort_definition_clarification(
+                question_id,
+                "peer_filters",
+                "The sector default alone does not define a peer group; at 
least one "
+                "explicit filter beyond the sector default is required.",
+            )
+
+    if "segment_filters" in required and 
isinstance(request.get("segment_filters"), dict):
+        if not request["segment_filters"]:
+            return _cohort_definition_clarification(
+                question_id,
+                "segment_filters",
+                "The segment definition is empty; at least one explicit filter 
"
+                "is required.",
+            )
+    return None
+
+
+def _cohort_definition_clarification(
+    question_id: str, field: str, why_material: str
+) -> ClarificationNeeded:
+    return ClarificationNeeded(
+        question_id=question_id,
+        missing=[MissingInput(input=field, why_material=why_material, 
allowed_values=[])],
+        prompt_to_user=(
+            "Which rule should define this cohort — for example a sector, 
school "
+            "type, enrolment band, ICSEA range, remoteness or suburb?"
+        ),
+    )
+
+
+# --------------------------------------------------------------------------- #
+# Derived missing inputs (pinned policy order)
+# --------------------------------------------------------------------------- #
+
+
+def _derive_missing_inputs(
+    question_id: str, entry: dict[str, Any], request: dict[str, Any]
+) -> list[str]:
+    """Missing required, material, then applicable conditional-material fields,
+    in pinned policy order — derived from policy and the assembled request,
+    never taken from the model's missing-input claim."""
+    policy = entry.get("input_policy") or {}
+    missing: list[str] = []
+    seen: set[str] = set()
+
+    def consider(field: str) -> None:
+        if field in seen:
+            return
+        seen.add(field)
+        if field not in request:
+            missing.append(field)
+
+    for field in policy.get("required", []):
+        if isinstance(field, str):
+            consider(field)
+    for field in policy.get("material", []):
+        if isinstance(field, str):
+            consider(field)
+
+    conditional = policy.get("conditional_material") or {}
+    if isinstance(conditional, dict):
+        for field, condition in conditional.items():
+            if not isinstance(field, str):
+                continue
+            normalized_condition = _SEPARATOR_RE.sub(" ", 
str(condition).casefold()).strip()
+            if field == "distance_km" and normalized_condition in 
_KNOWN_DISTANCE_CONDITIONS:
+                if request.get("market_definition") == "distance" and 
"distance_km" not in request:
+                    consider(field)
+            else:
+                raise TerminalError(
+                    f"{question_id}: unsupported conditional-material 
expression "
+                    f"{condition!r} for {field} — contract drift, refusing to 
interpret it"
+                )
+    return missing
+
+
+# --------------------------------------------------------------------------- #
+# Stage 2 entry point
+# --------------------------------------------------------------------------- #
+
+
 async def plan_and_execute(
     intent: StructuredIntent,
     client: EvidenceClient,
     deps: Deps,
 ) -> dict[str, Any] | ClarificationNeeded | Unsupported:
-    """Deterministic Stage 2: intent → evidence envelope or typed refusal.
+    """Deterministic Stage 2: intent → evidence envelope or typed outcome.
 
     Returns:
         dict — the evidence envelope (happy path)
-        ClarificationNeeded — a material input is missing
-        Unsupported — an exclusion, unknown question, or unanswerable request
+        ClarificationNeeded — a caller-suppliable input is missing or wrong
+        Unsupported — an exclusion or an unknown question
     May raise:
-        TerminalError — an operational stop (data_unavailable, internal)
+        TerminalError — planner/schema defects and operational stops
     """
     question_id = intent.question_id
 
     # ------------------------------------------------------------------ #
     # 1. Excluded topic (intent model classified it as out of scope)
@@ -157,112 +910,134 @@
             reason_kind="not_in_registry",
             reason=f"{question_id} is not a registry question.",
         )
 
     # ------------------------------------------------------------------ #
-    # 5. Material inputs missing → ask for clarification
-    # ------------------------------------------------------------------ #
-    if intent.missing_material_inputs:
-        material = contract.material_inputs_for(question_id)
-        actually_missing = [inp for inp in intent.missing_material_inputs if 
inp in material]
-        if actually_missing:
-            missing_items = [
-                MissingInput(
-                    input=inp,
-                    why_material=_material_why(question_id, inp),
-                    allowed_values=[],
+    # 5. Pinned policy and assembled request
+    # ------------------------------------------------------------------ #
+    entry = contract.questions().get(question_id)
+    if entry is None:
+        raise TerminalError(
+            f"{question_id} has a registered request schema but is absent from 
"
+            "the pinned registry — contract drift"
+        )
+    policy = entry.get("input_policy") or {}
+    required_fields = [f for f in policy.get("required", []) if isinstance(f, 
str)]
+    school_required = [f for f in required_fields if f in _SCHOOL_FIELDS]
+
+    # Fresh request from explicit intent fields only; planner-owned selectors
+    # are added below. Unsupplied defaultables stay omitted — the emitter owns
+    # defaults and resolved_request.
+    request: dict[str, Any] = dict(intent.filters)
+
+    scalar_clarification = _normalize_scalar_selectors(question_id, request)
+    if scalar_clarification is not None:
+        return scalar_clarification
+
+    _adapt_explicit_school_fields(request, school_required)
+    identity_from_caller = any(field in request for field in school_required)
+
+    # ------------------------------------------------------------------ #
+    # 6. Transactional named-school resolution
+    # ------------------------------------------------------------------ #
+    mentions = [
+        mention for mention in (intent.school_mentions or [])
+        if isinstance(mention, str) and mention.strip()
+    ]
+    if school_required and mentions:
+        resolved_ids, failure = await _resolve_mentions(question_id, mentions, 
client, deps)
+        if failure is not None:
+            return failure
+        if identity_from_caller:
+            explicit_ids = _explicit_id_values(request, school_required)
+            if sorted(resolved_ids) != sorted(explicit_ids):
+                raise TerminalError(
+                    f"{question_id}: explicit school identities {explicit_ids} 
conflict "
+                    f"with directory-resolved identities {resolved_ids}"
                 )
-                for inp in actually_missing
-            ]
-            prompt_to_user = (
-                f"To answer this question I need "
-                + ", ".join(f"the {inp}" for inp in actually_missing)
-                + ". Could you provide "
-                + ("it" if len(actually_missing) == 1 else "them")
-                + "?"
-            )
-            return ClarificationNeeded(
-                question_id=question_id,
-                missing=missing_items,
-                prompt_to_user=prompt_to_user,
-            )
-
-    # ------------------------------------------------------------------ #
-    # 6. Resolve schools
-    # ------------------------------------------------------------------ #
-    for school_name in intent.school_mentions:
-        try:
-            records = await client.school_directory(search=school_name, 
limit=25)
-        except Exception as exc:
-            deps.tool_calls.append(
-                ToolInvocation(
-                    "search_schools",
-                    {"search": school_name, "limit": 25},
-                    error_kind=type(exc).__name__,
-                )
-            )
-            raise
-
-        deps.searched = True
-        resolved = [r["acara_sml_id"] for r in records if "acara_sml_id" in r]
-        if resolved:
-            # Take the first match: the planner trusts the intent model to have
-            # named an unambiguous school. Ambiguity handling 
(ClarificationNeeded)
-            # is the live-model path; the planner uses best-effort resolution.
-            best = resolved[0]
-            if best not in deps.resolved_school_ids:
-                deps.resolved_school_ids.append(best)
-        deps.tool_calls.append(
-            ToolInvocation(
-                "search_schools",
-                {"search": school_name, "limit": 25},
-                records,
-            )
-        )
-
-    # ------------------------------------------------------------------ #
-    # 7. Build the evidence request
-    # ------------------------------------------------------------------ #
-    request: dict[str, Any] = dict(intent.filters)
-
-    # Attach resolved school IDs using the right field name for this question
-    if deps.resolved_school_ids:
-        q_entry = contract.questions()[question_id]
-        required = q_entry.get("input_policy", {}).get("required", [])
-        if "acara_sml_ids" in required:
-            request.setdefault("acara_sml_ids", deps.resolved_school_ids)
-        elif "acara_sml_id" in required and deps.resolved_school_ids:
-            request.setdefault("acara_sml_id", deps.resolved_school_ids[0])
-
-    # Map requested metrics to the request (NL labels are passed through for
-    # now — the real emitter validates them; bad values return bad_request)
-    if intent.requested_metrics:
-        request.setdefault("metrics", intent.requested_metrics)
-
-    # Apply metric scope check for VC-POC-01 and VC-POC-04
-    scope_problem = request_scope_problem(deps.user_question, question_id, 
request)
-    if scope_problem and "metrics" in request and 
isinstance(request["metrics"], list):
-        # Drop non-teaching and headcount expansions the user didn't ask for
-        cleaned = [
-            m for m in request["metrics"]
-            if not (
-                str(m).endswith("_headcount") or 
str(m).startswith("non_teaching_")
-            )
+            deps.searched = True
+        else:
+            if _requires_two_distinct_ids(entry) and len(resolved_ids) < 2:
+                return _insufficient_schools_clarification(question_id)
+            for school_id in resolved_ids:
+                if school_id not in deps.resolved_school_ids:
+                    deps.resolved_school_ids.append(school_id)
+            deps.searched = True
+            _attach_school_ids(request, school_required, resolved_ids)
+    elif school_required and identity_from_caller and "acara_sml_ids" in 
school_required:
+        explicit_ids = _explicit_id_values(request, school_required)
+        raw_ids = request.get("acara_sml_ids")
+        if (
+            _requires_two_distinct_ids(entry)
+            and isinstance(raw_ids, list)
+            and len(explicit_ids) == len(raw_ids)
+            and len(_dedupe(explicit_ids)) < 2
+        ):
+            return _insufficient_schools_clarification(question_id)
+
+    # ------------------------------------------------------------------ #
+    # 7. Selector normalization and routing (clarifies unnormalizable labels)
+    # ------------------------------------------------------------------ #
+    selector_clarification = _route_selectors(question_id, intent, request, 
deps)
+    if selector_clarification is not None:
+        return selector_clarification
+
+    # ------------------------------------------------------------------ #
+    # 8. Registry-owned explicit-filter constraints (VC-POC-09/10)
+    # ------------------------------------------------------------------ #
+    constraint_clarification = _enforce_explicit_filter_constraints(
+        question_id, entry, request
+    )
+    if constraint_clarification is not None:
+        return constraint_clarification
+
+    # ------------------------------------------------------------------ #
+    # 9. Derive missing inputs from the pinned policy (stable order)
+    # ------------------------------------------------------------------ #
+    missing = _derive_missing_inputs(question_id, entry, request)
+    if missing:
+        missing_items = [
+            MissingInput(
+                input=field,
+                why_material=_material_why(question_id, field),
+                allowed_values=_static_vocabulary_for_field(question_id, field)
or [],
+            )
+            for field in missing
         ]
-        request["metrics"] = cleaned or request["metrics"]
-
-    # ------------------------------------------------------------------ #
-    # 8. Fetch evidence
+        prompt_to_user = (
+            "To answer this question I need "
+            + ", ".join(f"the {field}" for field in missing)
+            + ". Could you provide "
+            + ("it" if len(missing) == 1 else "them")
+            + "?"
+        )
+        logger.debug(
+            "planner clarification",
+            extra={"subsystem": "planner", "question_id": question_id, 
"missing": missing},
+        )
+        return ClarificationNeeded(
+            question_id=question_id,
+            missing=missing_items,
+            prompt_to_user=prompt_to_user,
+        )
+
+    # ------------------------------------------------------------------ #
+    # 10. Pinned-schema validation before dispatch
+    # ------------------------------------------------------------------ #
+    _validate_request(question_id, request)
+
+    # ------------------------------------------------------------------ #
+    # 11. Fetch evidence (failures recorded before branching)
     # ------------------------------------------------------------------ #
     arguments = {"question_id": question_id, "request": request}
     try:
         envelope = await client.fetch_evidence(question_id, request)
     except EvidenceError as exc:
         deps.tool_calls.append(
             ToolInvocation("fetch_evidence", arguments, 
error_kind=exc.error_kind)
         )
-        return _branch_evidence_error(question_id, exc)
+        return _branch_evidence_error(question_id, exc, identity_from_caller)
     except Exception:
         deps.tool_calls.append(
             ToolInvocation("fetch_evidence", arguments, 
error_kind="operational")
         )
         raise
@@ -284,26 +1059,32 @@
         "grades": "The grade selection determines which year levels are 
included.",
         "peer_filters": "The peer group definition determines which schools are
compared.",
         "segment_filters": "The segment definition determines which schools are
included.",
         "sa2_codes": "The SA2 area determines which geographic area is 
analysed.",
         "zone_entity_code": "The zone entity code identifies the government 
zone to look up.",
+        "acara_sml_id": "The school must be identified before evidence can be 
fetched.",
+        "acara_sml_ids": "The schools must be identified before evidence can be
fetched.",
     }
     return explanations.get(
         input_name,
         f"{input_name} is required by the data layer and cannot be defaulted 
for this question.",
     )
 
 
 def _branch_evidence_error(
     question_id: str,
     exc: EvidenceError,
-) -> Unsupported:
-    """Translate an EvidenceError into an Unsupported outcome.
-
-    This mirrors `_branch_on_error_kind()` in tools.py but returns typed values
-    instead of raising PydanticAI signals. TerminalError is re-raised (never
-    returned as Unsupported — it is an operational stop, not an honest 
refusal).
+    identity_from_caller: bool,
+) -> Unsupported | ClarificationNeeded:
+    """Translate an EvidenceError into the pipeline's typed outcomes.
+
+    Origin-aware: a bad_request for a request whose school identity was
+    caller-supplied is a correctable identity problem → ClarificationNeeded.
+    Any other bad_request — the request was deterministic and schema-valid —
+    is planner/contract drift → TerminalError. excluded/unknown/unanswerable
+    keep their contract-owned refusals; data_unavailable/internal are
+    operational stops, never refusals.
     """
     kind = exc.error_kind
     logger.debug(
         "evidence error in planner",
         extra={"subsystem": "planner", "error_kind": kind, "question_id": 
question_id},
@@ -327,11 +1108,31 @@
 
     if kind == "unanswerable":
         return Unsupported(reason_kind="unanswerable", reason=exc.message)
 
     if kind == "bad_request":
-        # On a first bad_request the planner cannot repair (no model); return
-        # out_of_scope so the caller knows the data layer rejected the request.
-        return Unsupported(reason_kind="out_of_scope", reason=exc.message)
+        if identity_from_caller:
+            return ClarificationNeeded(
+                question_id=question_id,
+                missing=[
+                    MissingInput(
+                        input="acara_sml_id",
+                        why_material=(
+                            "The data layer rejected the supplied school 
identity "
+                            f"({exc.message}). Confirm the school's current 
public "
+                            "ACARA_SML_ID."
+                        ),
+                        allowed_values=[],
+                    )
+                ],
+                prompt_to_user=(
+                    "The school identity supplied for this question was 
rejected by the "
+                    "data layer. Could you confirm the school?"
+                ),
+            )
+        raise TerminalError(
+            f"{question_id}: the evidence emitter returned bad_request for a "
+            f"deterministic, schema-valid request — planner/contract drift: 
{exc.message}"
+        )
 
     # data_unavailable | internal — operational stop, not a refusal.
     raise TerminalError(f"evidence emitter reported {kind}: {exc.message}")
```



run.py

```diff
@@ -12,10 +12,13 @@
 
 * It **renders**, it does not compose. Limitations and confidence are printed
   from the fields the harness attached out of the envelope (ADR-009). Nothing
   here paraphrases them, reorders them, or drops one for brevity — and there is
   no `--brief` that would.
+* It runs the **deterministic four-stage pipeline by default**: no legacy
+  agent is constructed or passed; `converse()` still accepts an explicit
+  legacy agent for replay and compatibility callers.
 * It carries **one clarification round-trip** by default (O-3): a
   `ClarificationNeeded` is put to the user, the reply is appended to the 
message
   history, and the run continues. A question still ambiguous after the last
   round returns the clarification unanswered rather than guessing.
 * It **configures logging once**, here, as every entry point must.
@@ -43,11 +46,11 @@
 from config import settings  # noqa: E402
 from harness.claude_account_cli import (  # noqa: E402
     ClaudeAccountCliError,
     claude_account_status,
 )
-from harness.agent import ask, build_agent  # noqa: E402
+from harness.agent import ask  # noqa: E402
 from harness.errors import HarnessError  # noqa: E402
 from harness.evidence import EmitterFailure, EvidenceClient  # noqa: E402
 from harness.logging_config import configure_logging  # noqa: E402
 from harness.outputs import (  # noqa: E402
     BudgetExceeded,
@@ -181,12 +184,13 @@
     `record_usage=False` is for callers driving a scripted `FunctionModel`
     (`demo.py --replay`, `evals/replay.py`): those runs cost nothing, and 
writing
     their zero-price "usage" into the JSONL the cost report reads would put 
runs
     that never happened into the spend record (ADR-005).
 
-    `agent` may be a PydanticAI `Agent` (legacy tool-loop path) or `None`
-    (pipeline path). When `None`, the new 4-stage pipeline is used.
+    `agent` may be a PydanticAI `Agent` (legacy tool-loop path, kept for replay
+    and compatibility callers) or `None` (the default deterministic pipeline
+    path). When `None`, the 4-stage pipeline is used and no legacy agent 
exists.
     """
     history: list[Any] | None = None
     reply = question
     for _round in range(max_clarifications + 1):
         if agent is not None:
@@ -309,18 +313,20 @@
     configure_logging()  # once per entry point, before anything else logs
     args = build_parser().parse_args(argv)
 
     require_credentials()
     client = build_client(args.stub)
-    agent = build_agent()
     deps = Deps(client=client)
     ask_user = None if args.no_interactive else _prompt_on_terminal
 
     try:
+        # The default path is the deterministic four-stage pipeline: no legacy
+        # agent is built or passed. `converse` still accepts an explicit agent
+        # for replay and compatibility callers.
         output = asyncio.run(
             converse(
-                agent,
+                None,
                 deps,
                 args.question,
                 ask_user=ask_user,
                 max_clarifications=args.max_clarifications,
             )
```



tests/unit/test_planner.py

```diff
@@ -1,14 +1,26 @@
 """Stage 2 — deterministic planner tests.
 
-Proves that `plan_and_execute()` branches purely on the intent and the 
contract:
-no model calls, no guessing. Every branch is a closed function of the
-StructuredIntent and the evidence client's response.
-
-Two kinds of evidence client:
-- stub_client() — the real stub emitter (tests the happy path with real fixture
data)
-- _ErrorClient / _HappyClient — simple mock clients for error branch tests
+Proves that `plan_and_execute()` branches purely on the intent, the pinned
+contract artefacts, and the evidence client's responses: no model calls, no
+guessing. Covered here:
+
+- derived required/material/conditional inputs (the model's missing-input
+  claim is never trusted), in pinned policy order;
+- transactional exactly-one-match school resolution (zero, ambiguous,
+  duplicate, conflicting and consistent identities; no partial Deps mutation);
+- selector normalization, pinned aliases, routing (`metrics` vs `ratios` vs
+  `grades`) and exact request dictionaries;
+- pinned-schema validation and VC-POC-09/10 explicit-filter constraints,
+  all failing closed before dispatch;
+- origin-aware evidence errors: caller-supplied identity bad_request
+  clarifies; planner/contract drift and operational stops are TerminalError.
+
+Three kinds of evidence client:
+- stub_client() — the real stub emitter (happy path with real fixture data)
+- _ErrorClient / _HappyClient — simple mocks for error/happy branch tests
+- _DirectoryClient — a directory-backed mock for identity-resolution tests
 """
 
 from __future__ import annotations
 
 import asyncio
@@ -71,13 +83,43 @@
     async def fetch_evidence(self, question_id: str, request: dict) -> dict:
         self.fetched.append({"question_id": question_id, "request": request})
         return self._envelope
 
 
-def run_planner(intent: StructuredIntent, client=None) -> Any:
-    deps = Deps(client=client or stub_client())
-    return asyncio.run(plan_and_execute(intent, client or stub_client(), 
deps)), deps
+class _DirectoryClient:
+    """A directory-backed mock: per-search records, canned fetch result."""
+
+    def __init__(
+        self,
+        directory: dict[str, list[dict]] | None = None,
+        envelope: dict | None = None,
+        fetch_error: EvidenceError | None = None,
+    ):
+        self.directory = directory or {}
+        self._envelope = envelope or {
+            "question_id": "VC-POC-01",
+            "contract_version": "0.3.0",
+            "evidence": [],
+            "provenance": {},
+            "confidence": {},
+            "limitations": [],
+            "result": {"data": []},
+            "data_status": {"status": "complete"},
+        }
+        self._fetch_error = fetch_error
+        self.searches: list[str] = []
+        self.fetched: list[dict] = []
+
+    async def school_directory(self, search: str, limit: int = 25) -> 
list[dict]:
+        self.searches.append(search)
+        return list(self.directory.get(search, []))
+
+    async def fetch_evidence(self, question_id: str, request: dict) -> dict:
+        self.fetched.append({"question_id": question_id, "request": request})
+        if self._fetch_error is not None:
+            raise self._fetch_error
+        return self._envelope
 
 
 # ---------------------------------------------------------------------------
 # 1. Excluded topics
 # ---------------------------------------------------------------------------
@@ -201,16 +243,30 @@
     assert isinstance(result, Unsupported)
     assert result.reason_kind == "not_in_registry"
 
 
 # ---------------------------------------------------------------------------
-# 5. Missing material inputs → clarification
-# ---------------------------------------------------------------------------
+# 5. Derived missing inputs → clarification (pinned policy, not model claims)
+# ---------------------------------------------------------------------------
+
+
+def test_missing_market_definition_is_derived_without_model_hint():
+    """VC-POC-05 without market_definition clarifies even though the model
+    reported no missing input; derivation follows pinned policy order."""
+    intent = StructuredIntent(question_id="VC-POC-05", school_mentions=[])
+    client = _DirectoryClient()
+    deps = Deps(client=client)
+    result = asyncio.run(plan_and_execute(intent, client, deps))
+    assert isinstance(result, ClarificationNeeded)
+    names = [m.input for m in result.missing]
+    assert names == ["acara_sml_id", "market_definition"]  # pinned policy 
order
+    assert result.missing[1].allowed_values == ["distance", "lga", "postcode", 
"sa2"]
+    assert client.fetched == []  # no evidence call on the clarification path
 
 
 def test_missing_material_input_returns_clarification():
-    """A declared material input in missing_material_inputs → 
ClarificationNeeded."""
+    """A declared material input that is genuinely absent → 
ClarificationNeeded."""
     intent = StructuredIntent(
         question_id="VC-POC-05",
         school_mentions=[],
         missing_material_inputs=["market_definition"],
     )
@@ -222,11 +278,11 @@
     missing_names = [m.input for m in result.missing]
     assert "market_definition" in missing_names
 
 
 def test_non_material_missing_input_is_ignored():
-    """An input listed in missing_material_inputs but not declared material is 
ignored."""
+    """A model-reported missing input that policy does not require is 
ignored."""
     intent = StructuredIntent(
         question_id="VC-POC-01",
         school_mentions=[],
         # "fiscal_year" is not a material input for VC-POC-01
         missing_material_inputs=["fiscal_year"],
@@ -235,10 +291,55 @@
     client = _HappyClient()
     deps = Deps(client=client)
     result = asyncio.run(plan_and_execute(intent, client, deps))
     # Should NOT return ClarificationNeeded; should proceed to evidence fetch
     assert not isinstance(result, ClarificationNeeded)
+
+
+def test_distance_is_conditional_on_distance_market():
+    """distance_km is required only when the market definition is distance."""
+    # An SA2 market asks for no radius: evidence proceeds.
+    intent = StructuredIntent(
+        question_id="VC-POC-05",
+        filters={"acara_sml_id": 40387, "market_definition": "sa2"},
+    )
+    client = _DirectoryClient()
+    deps = Deps(client=client)
+    result = asyncio.run(plan_and_execute(intent, client, deps))
+    assert isinstance(result, dict)
+    assert client.fetched[0]["request"] == {
+        "acara_sml_id": 40387,
+        "market_definition": "sa2",
+    }
+
+    # A distance market without a radius clarifies for distance_km.
+    intent = StructuredIntent(
+        question_id="VC-POC-05",
+        filters={"acara_sml_id": 40387, "market_definition": "distance"},
+    )
+    client = _DirectoryClient()
+    deps = Deps(client=client)
+    result = asyncio.run(plan_and_execute(intent, client, deps))
+    assert isinstance(result, ClarificationNeeded)
+    assert [m.input for m in result.missing] == ["distance_km"]
+    assert client.fetched == []
+
+
+def test_clarifications_follow_pinned_policy_order():
+    """Missing fields list required first, then applicable conditional 
material."""
+    directory = {"Alpha College": [{"acara_sml_id": 40387, "school_name": 
"Alpha College"}]}
+    intent = StructuredIntent(
+        question_id="VC-POC-07",
+        school_mentions=["Alpha College"],
+        filters={"market_definition": "distance"},
+    )
+    client = _DirectoryClient(directory=directory)
+    deps = Deps(client=client)
+    result = asyncio.run(plan_and_execute(intent, client, deps))
+    assert isinstance(result, ClarificationNeeded)
+    names = [m.input for m in result.missing]
+    assert names == ["grades", "distance_km"]
 
 
 # ---------------------------------------------------------------------------
 # 6. Happy path — evidence fetch
 # ---------------------------------------------------------------------------
@@ -280,11 +381,11 @@
     fetch_calls = [c for c in deps.tool_calls if c.tool == "fetch_evidence"]
     assert len(fetch_calls) == 1
 
 
 def test_school_resolution_records_search():
-    """school_mentions causes a school_directory call and populates 
resolved_school_ids."""
+    """school_mentions cause a directory preflight and populate 
resolved_school_ids."""
     intent = StructuredIntent(
         question_id="VC-POC-01",
         school_mentions=["Some School"],
     )
     client = _HappyClient()
@@ -295,28 +396,382 @@
     search_calls = [c for c in deps.tool_calls if c.tool == "search_schools"]
     assert len(search_calls) == 1
 
 
 # ---------------------------------------------------------------------------
-# 7. Evidence error branches
-# ---------------------------------------------------------------------------
-
-
-def test_bad_request_returns_unsupported():
-    """EvidenceError(bad_request) → Unsupported(out_of_scope)."""
-    intent = StructuredIntent(question_id="VC-POC-01", filters={"acara_sml_id":
40387})
-    client = _ErrorClient("bad_request", "invalid metrics: not_a_metric")
-    deps = Deps(client=client)
-    result = asyncio.run(plan_and_execute(intent, client, deps))
-    assert isinstance(result, Unsupported)
-    assert result.reason_kind == "out_of_scope"
-    assert "invalid metrics" in result.reason
+# 7. Transactional school resolution
+# ---------------------------------------------------------------------------
+
+
+def test_ambiguous_school_clarifies_without_fetch():
+    """Multiple directory matches clarify: no evidence call, no partial 
commit."""
+    directory = {
+        "St Albans": [
+            {"acara_sml_id": 40387, "school_name": "St Albans Primary"},
+            {"acara_sml_id": 40392, "school_name": "St Albans Secondary"},
+        ]
+    }
+    intent = StructuredIntent(question_id="VC-POC-01", school_mentions=["St 
Albans"])
+    client = _DirectoryClient(directory=directory)
+    deps = Deps(client=client)
+    result = asyncio.run(plan_and_execute(intent, client, deps))
+    assert isinstance(result, ClarificationNeeded)
+    assert client.fetched == []              # no evidence call
+    assert deps.resolved_school_ids == []    # no partial identity mutation
+    assert deps.searched is False
+    preflights = [c for c in deps.tool_calls if c.tool == "search_schools"]
+    assert len(preflights) == 1              # the preflight is still recorded
+
+
+def test_unmatched_school_clarifies_without_fetch():
+    """A zero-match school clarifies with no evidence call and no commit."""
+    intent = StructuredIntent(
+        question_id="VC-POC-01", school_mentions=["Nowhere College"]
+    )
+    client = _DirectoryClient(directory={})
+    deps = Deps(client=client)
+    result = asyncio.run(plan_and_execute(intent, client, deps))
+    assert isinstance(result, ClarificationNeeded)
+    assert client.fetched == []
+    assert deps.resolved_school_ids == []
+    assert deps.searched is False
+    preflights = [c for c in deps.tool_calls if c.tool == "search_schools"]
+    assert len(preflights) == 1
+
+
+def test_school_resolution_is_transactional():
+    """A later failed match leaks no earlier identity and dispatches 
nothing."""
+    directory = {
+        "Alpha College": [{"acara_sml_id": 40387, "school_name": "Alpha 
College"}],
+    }
+    intent = StructuredIntent(
+        question_id="VC-POC-01",
+        school_mentions=["Alpha College", "Missing College"],
+    )
+    client = _DirectoryClient(directory=directory)
+    deps = Deps(client=client)
+    result = asyncio.run(plan_and_execute(intent, client, deps))
+    assert isinstance(result, ClarificationNeeded)
+    assert deps.resolved_school_ids == []   # the first match did not leak
+    assert deps.searched is False
+    assert client.fetched == []
+    preflights = [c for c in deps.tool_calls if c.tool == "search_schools"]
+    assert len(preflights) == 2
+
+
+def test_conflicting_explicit_id_and_mention_is_terminal():
+    """Explicit IDs that disagree with name-resolved identities fail closed."""
+    directory = {"Beta Grammar": [{"acara_sml_id": 40392, "school_name": "Beta 
Grammar"}]}
+    intent = StructuredIntent(
+        question_id="VC-POC-01",
+        school_mentions=["Beta Grammar"],
+        filters={"acara_sml_id": 40387},
+    )
+    client = _DirectoryClient(directory=directory)
+    deps = Deps(client=client)
+    with pytest.raises(TerminalError):
+        asyncio.run(plan_and_execute(intent, client, deps))
+    assert deps.resolved_school_ids == []
+    assert client.fetched == []
+
+
+def test_consistent_explicit_id_and_mention_proceeds():
+    """Explicit IDs that agree with name-resolved identities are not a 
conflict."""
+    directory = {"Alpha College": [{"acara_sml_id": 40387, "school_name": 
"Alpha College"}]}
+    intent = StructuredIntent(
+        question_id="VC-POC-01",
+        school_mentions=["Alpha College"],
+        filters={"acara_sml_id": 40387},
+    )
+    client = _DirectoryClient(directory=directory)
+    deps = Deps(client=client)
+    result = asyncio.run(plan_and_execute(intent, client, deps))
+    assert isinstance(result, dict)
+    assert client.fetched[0]["request"] == {"acara_sml_id": 40387}
+    assert deps.searched is True
+
+
+def test_mention_order_preserved_and_duplicates_removed():
+    """Directory-resolved IDs keep mention order and drop duplicates."""
+    directory = {
+        "Beta Grammar": [{"acara_sml_id": 40392, "school_name": "Beta 
Grammar"}],
+        "Alpha College": [{"acara_sml_id": 40387, "school_name": "Alpha 
College"}],
+    }
+    intent = StructuredIntent(
+        question_id="VC-POC-03",
+        school_mentions=["Beta Grammar", "Alpha College", "Beta Grammar"],
+    )
+    client = _DirectoryClient(directory=directory)
+    deps = Deps(client=client)
+    result = asyncio.run(plan_and_execute(intent, client, deps))
+    assert isinstance(result, dict)
+    assert client.fetched[0]["request"] == {"acara_sml_ids": [40392, 40387]}
+    assert deps.resolved_school_ids == [40392, 40387]
+
+
+def test_comparison_with_one_distinct_school_clarifies():
+    """VC-POC-04 with fewer than two distinct identities clarifies, no 
dispatch."""
+    directory = {"Alpha College": [{"acara_sml_id": 40387, "school_name": 
"Alpha College"}]}
+    intent = StructuredIntent(
+        question_id="VC-POC-04",
+        school_mentions=["Alpha College", "Alpha College"],
+    )
+    client = _DirectoryClient(directory=directory)
+    deps = Deps(client=client)
+    result = asyncio.run(plan_and_execute(intent, client, deps))
+    assert isinstance(result, ClarificationNeeded)
+    assert client.fetched == []
+    assert deps.resolved_school_ids == []   # not committed on this path
+
+
+def test_explicit_numeric_ids_skip_directory_preflight():
+    """Caller-supplied numeric IDs are authoritative: no preflight is made."""
+    intent = StructuredIntent(
+        question_id="VC-POC-03",
+        filters={"acara_sml_ids": [99999, 40387]},
+    )
+    client = _DirectoryClient()
+    deps = Deps(client=client)
+    result = asyncio.run(plan_and_execute(intent, client, deps))
+    assert isinstance(result, dict)
+    assert client.searches == []
+    assert client.fetched[0]["request"] == {"acara_sml_ids": [99999, 40387]}
+
+
+# ---------------------------------------------------------------------------
+# 8. Selector normalization, routing and exact requests
+# ---------------------------------------------------------------------------
+
+
+def test_metric_normalization_and_exact_profile_request():
+    """Labels fold to canonical values, the pinned alias applies, duplicates
+    drop in order, and the dispatched request is exactly the smallest one."""
+    intent = StructuredIntent(
+        question_id="VC-POC-01",
+        filters={"acara_sml_id": 40387},
+        requested_metrics=[
+            "Total Enrolments",
+            "Teaching Staff",
+            "teaching staff",
+            "TOTAL_ENROLMENTS",
+        ],
+    )
+    client = _DirectoryClient()
+    deps = Deps(client=client)
+    result = asyncio.run(plan_and_execute(intent, client, deps))
+    assert isinstance(result, dict)
+    assert client.fetched[0]["request"] == {
+        "acara_sml_id": 40387,
+        "metrics": ["total_enrolments", "teaching_fte"],
+    }
+
+
+def test_vc_poc_03_routes_selector_to_ratios():
+    """VC-POC-03 staffing-intensity selectors use the ratios field and
+    vocabulary — no metrics property reaches the request."""
+    intent = StructuredIntent(
+        question_id="VC-POC-03",
+        filters={"acara_sml_ids": [40387, 40392]},
+        requested_metrics=["Students per Teaching FTE"],
+    )
+    client = _DirectoryClient()
+    deps = Deps(client=client)
+    result = asyncio.run(plan_and_execute(intent, client, deps))
+    assert isinstance(result, dict)
+    sent = client.fetched[0]["request"]
+    assert sent == {
+        "acara_sml_ids": [40387, 40392],
+        "ratios": ["students_per_teaching_fte"],
+    }
+    assert "metrics" not in sent
+
+
+def test_grade_labels_route_to_grades_field():
+    """Grade labels route to `grades` for VC-POC-02, from the pinned 
vocabulary."""
+    intent = StructuredIntent(
+        question_id="VC-POC-02",
+        filters={"acara_sml_id": 40387},
+        requested_metrics=["Year 6"],
+    )
+    client = _DirectoryClient()
+    deps = Deps(client=client)
+    result = asyncio.run(plan_and_execute(intent, client, deps))
+    assert isinstance(result, dict)
+    assert client.fetched[0]["request"] == {
+        "acara_sml_id": 40387,
+        "grades": ["year_6"],
+    }
+
+
+def test_unnormalizable_metric_clarifies_without_fetch():
+    """A label outside the pinned static vocabulary clarifies with that
+    vocabulary — it never dispatches and never becomes Unsupported."""
+    intent = StructuredIntent(
+        question_id="VC-POC-01",
+        filters={"acara_sml_id": 40387},
+        requested_metrics=["banana enrolments"],
+    )
+    client = _DirectoryClient()
+    deps = Deps(client=client)
+    result = asyncio.run(plan_and_execute(intent, client, deps))
+    assert isinstance(result, ClarificationNeeded)
+    assert result.missing[0].allowed_values  # the pinned vocabulary is 
supplied
+    assert client.fetched == []
+
+
+def test_selector_labels_without_a_route_clarify():
+    """A question with no label-selector field cannot apply requested 
labels."""
+    intent = StructuredIntent(
+        question_id="VC-POC-05",
+        filters={"acara_sml_id": 40387, "market_definition": "sa2"},
+        requested_metrics=["total enrolments"],
+    )
+    client = _DirectoryClient()
+    deps = Deps(client=client)
+    result = asyncio.run(plan_and_execute(intent, client, deps))
+    assert isinstance(result, ClarificationNeeded)
+    assert client.fetched == []
+
+
+# ---------------------------------------------------------------------------
+# 9. Pinned-schema validation and explicit-filter constraints
+# ---------------------------------------------------------------------------
+
+
+def test_unknown_request_property_fails_before_fetch():
+    """An unknown top-level property is a planner/schema defect: TerminalError
+    before dispatch, never an Unsupported user question."""
+    intent = StructuredIntent(
+        question_id="VC-POC-01",
+        filters={"acara_sml_id": 40387, "enrolments_forecast": True},
+    )
+    client = _DirectoryClient()
+    deps = Deps(client=client)
+    with pytest.raises(TerminalError):
+        asyncio.run(plan_and_execute(intent, client, deps))
+    assert client.fetched == []
+
+
+def test_peer_filters_require_explicit_nondefault_filter():
+    """The sector default alone does not define a peer group; absent filters
+    clarify from derived policy. Neither path dispatches evidence."""
+    # Sector-default-only definition
+    intent = StructuredIntent(
+        question_id="VC-POC-09",
+        filters={"peer_filters": {"sectors": ["Catholic", "Independent"]}},
+    )
+    client = _DirectoryClient()
+    deps = Deps(client=client)
+    result = asyncio.run(plan_and_execute(intent, client, deps))
+    assert isinstance(result, ClarificationNeeded)
+    assert client.fetched == []
+
+    # Absent definition — derived from the pinned policy
+    intent = StructuredIntent(question_id="VC-POC-09")
+    client = _DirectoryClient()
+    deps = Deps(client=client)
+    result = asyncio.run(plan_and_execute(intent, client, deps))
+    assert isinstance(result, ClarificationNeeded)
+    assert "peer_filters" in [m.input for m in result.missing]
+    assert client.fetched == []
+
+
+def test_explicit_peer_filter_dispatches():
+    """An explicit nondefault peer filter passes the constraint and 
dispatches."""
+    intent = StructuredIntent(
+        question_id="VC-POC-09",
+        filters={
+            "peer_filters": {
+                "sectors": ["Catholic", "Independent"],
+                "school_types": ["Secondary"],
+            }
+        },
+    )
+    client = _DirectoryClient()
+    deps = Deps(client=client)
+    result = asyncio.run(plan_and_execute(intent, client, deps))
+    assert isinstance(result, dict)
+    assert client.fetched[0]["request"] == {
+        "peer_filters": {
+            "sectors": ["Catholic", "Independent"],
+            "school_types": ["Secondary"],
+        }
+    }
+
+
+def test_segment_filters_reject_unknown_nested_key():
+    """An unknown nested filter key is a schema violation: TerminalError
+    before dispatch, never Unsupported."""
+    intent = StructuredIntent(
+        question_id="VC-POC-10",
+        filters={"segment_filters": {"sectors": ["Independent"], "banana": 
["x"]}},
+    )
+    client = _DirectoryClient()
+    deps = Deps(client=client)
+    with pytest.raises(TerminalError):
+        asyncio.run(plan_and_execute(intent, client, deps))
+    assert client.fetched == []
+
+
+def test_empty_segment_filters_clarify():
+    """An empty segment definition lacks an explicit filter → clarification."""
+    intent = StructuredIntent(
+        question_id="VC-POC-10",
+        filters={"segment_filters": {}},
+    )
+    client = _DirectoryClient()
+    deps = Deps(client=client)
+    result = asyncio.run(plan_and_execute(intent, client, deps))
+    assert isinstance(result, ClarificationNeeded)
+    assert client.fetched == []
+
+
+# ---------------------------------------------------------------------------
+# 10. Evidence error branches — origin-aware
+# ---------------------------------------------------------------------------
+
+
+def test_schema_valid_emitter_bad_request_is_terminal():
+    """A bad_request for a deterministic, schema-valid request built from a
+    directory-resolved identity is planner/contract drift → TerminalError."""
+    directory = {"Alpha College": [{"acara_sml_id": 40387, "school_name": 
"Alpha College"}]}
+    intent = StructuredIntent(question_id="VC-POC-01", school_mentions=["Alpha 
College"])
+    client = _DirectoryClient(
+        directory=directory,
+        fetch_error=EvidenceError("bad_request", "unexpected request shape"),
+    )
+    deps = Deps(client=client)
+    with pytest.raises(TerminalError) as caught:
+        asyncio.run(plan_and_execute(intent, client, deps))
+    assert "VC-POC-01" in str(caught.value)
+    fetch_calls = [c for c in deps.tool_calls if c.tool == "fetch_evidence"]
+    assert len(fetch_calls) == 1
+    assert fetch_calls[0].error_kind == "bad_request"
+    assert deps.resolved_school_ids == [40387]
+
+
+def test_invalid_explicit_identity_bad_request_clarifies():
+    """A bad_request against a caller-supplied numeric identity is a
+    correctable identity problem → ClarificationNeeded, never Unsupported."""
+    intent = StructuredIntent(question_id="VC-POC-01", filters={"acara_sml_id":
99999})
+    client = _DirectoryClient(
+        fetch_error=EvidenceError(
+            "bad_request", "acara_sml_id absent from the pinned school 
directory"
+        )
+    )
+    deps = Deps(client=client)
+    result = asyncio.run(plan_and_execute(intent, client, deps))
+    assert isinstance(result, ClarificationNeeded)
+    assert result.question_id == "VC-POC-01"
+    fetch_calls = [c for c in deps.tool_calls if c.tool == "fetch_evidence"]
+    assert len(fetch_calls) == 1
+    assert fetch_calls[0].error_kind == "bad_request"
 
 
 def test_unanswerable_returns_unsupported():
     """EvidenceError(unanswerable) → Unsupported(unanswerable)."""
-    intent = StructuredIntent(question_id="VC-POC-05", filters={"acara_sml_id":
99999})
+    intent = StructuredIntent(question_id="VC-POC-01", filters={"acara_sml_id":
40387})
     client = _ErrorClient("unanswerable", "no published SA2 centroid for this 
school")
     deps = Deps(client=client)
     result = asyncio.run(plan_and_execute(intent, client, deps))
     assert isinstance(result, Unsupported)
     assert result.reason_kind == "unanswerable"
@@ -349,22 +804,22 @@
     with pytest.raises(TerminalError):
         asyncio.run(plan_and_execute(intent, client, deps))
 
 
 def test_error_call_recorded_in_deps():
-    """A failed fetch is still recorded in deps.tool_calls."""
+    """A failed fetch is still recorded in deps.tool_calls before branching."""
     intent = StructuredIntent(question_id="VC-POC-01", filters={"acara_sml_id":
40387})
     client = _ErrorClient("bad_request", "bad")
     deps = Deps(client=client)
     asyncio.run(plan_and_execute(intent, client, deps))
     fetch_calls = [c for c in deps.tool_calls if c.tool == "fetch_evidence"]
     assert len(fetch_calls) == 1
     assert fetch_calls[0].error_kind == "bad_request"
 
 
 # ---------------------------------------------------------------------------
-# 8. _find_exclusion_text helper
+# 11. _find_exclusion_text helper
 # ---------------------------------------------------------------------------
 
 
 def test_find_exclusion_text_transport_hint():
     """'transport' hint finds the public transport exclusion text."""
```



tests/unit/test_intent_boundary.py

```diff
@@ -1,10 +1,16 @@
 """Stage 1 — intent boundary tests.
 
 Proves that `interpret_intent()` is a single-call, no-tools extraction that
 returns a typed `StructuredIntent`, and that the pipeline path of `ask()`
 correctly wires the intent stage.
+
+It also pins the planner boundary as seen through `ask()`: planner
+clarifications bypass narration and the answer view; normalized, schema-valid
+requests reach evidence with exact arguments; planner/schema defects propagate
+as `TerminalError` (never as an `Unsupported` user outcome); and
+caller-correctable selector or identity problems stay clarifications.
 
 All tests are fully offline: models are scripted `FunctionModel` instances that
 return canned responses, and the evidence client uses the stub emitter.
 Nothing here touches the network, the credential store, or the data project.
 """
@@ -19,10 +25,12 @@
 import pytest
 from pydantic_ai.messages import ModelMessage, ModelResponse, ToolCallPart
 from pydantic_ai.models.function import AgentInfo, FunctionModel
 
 from harness.agent import ask
+from harness.evidence import EvidenceError
+from harness.errors import TerminalError
 from harness.intent import StructuredIntent, build_intent_agent, 
interpret_intent
 from harness.narration import NarrationOutput, NarrationValidationError
 from harness.outputs import ClarificationNeeded, RenderedAnswer, Unsupported
 from harness.tools import Deps
 
@@ -77,10 +85,28 @@
             "question_id": "VC-POC-01",
             "school_mentions": [],
         })])
 
     return FunctionModel(fn)
+
+
+class _RecordingClient:
+    """Wraps a client, recording the exact fetch_evidence arguments."""
+
+    def __init__(self, inner):
+        self._inner = inner
+        self.fetched: list[dict[str, Any]] = []
+
+    async def school_directory(self, search=None, limit=25):
+        return await self._inner.school_directory(search=search, limit=limit)
+
+    async def input_catalog(self):
+        return await self._inner.input_catalog()
+
+    async def fetch_evidence(self, question_id, request):
+        self.fetched.append({"question_id": question_id, "request": request})
+        return await self._inner.fetch_evidence(question_id, request)
 
 
 # ---------------------------------------------------------------------------
 # 1. interpret_intent() with a scripted model
 # ---------------------------------------------------------------------------
@@ -369,5 +395,99 @@
     assert intent.school_mentions == []
     assert intent.requested_metrics == []
     assert intent.filters == {}
     assert intent.presentation == "snapshot"
     assert intent.missing_material_inputs == []
+
+
+# ---------------------------------------------------------------------------
+# 4. Planner boundary through ask() — clarifications, evidence, failures
+# ---------------------------------------------------------------------------
+
+
+def test_planner_failure_is_not_unsupported():
+    """A planner/schema defect propagates out of ask() as TerminalError —
+    it must never surface as an Unsupported user outcome."""
+    intent = StructuredIntent(
+        question_id="VC-POC-01",
+        filters={"acara_sml_id": 40387, "not_a_request_field": 1},
+    )
+    deps = Deps(client=stub_client())
+    with pytest.raises(TerminalError):
+        asyncio.run(ask(deps, "q", intent_override=intent))
+    assert deps.last_envelope is None  # nothing reached the evidence layer
+
+
+def test_planner_clarification_bypasses_narration_and_answer_view():
+    """The planner's clarification returns before the narration stage: no
+    answer view is built and the typed clarification is what ask() returns."""
+    intent = StructuredIntent(question_id="VC-POC-08", filters={"acara_sml_id":
40387})
+    deps = Deps(client=stub_client())
+    output = asyncio.run(ask(deps, "q", intent_override=intent))
+    assert isinstance(output, ClarificationNeeded)
+    assert deps.answer_view is None
+    assert [m.input for m in output.missing] == ["distance_km"]
+
+
+def test_normalized_schema_valid_request_reaches_evidence():
+    """Requested labels are normalized and routed before dispatch; the request
+    that reaches evidence is exactly the smallest schema-valid one."""
+    intent = StructuredIntent(
+        question_id="VC-POC-01",
+        filters={"acara_sml_id": 40387},
+        requested_metrics=["Total Enrolments", "Teaching Staff"],
+    )
+    narration = NarrationOutput(answer="Enrolments run 612 in 2019 and 668 in 
2023.")
+    client = _RecordingClient(stub_client())
+    deps = Deps(client=client)
+    output = asyncio.run(
+        ask(deps, "q", intent_override=intent, narration_override=narration)
+    )
+    assert isinstance(output, RenderedAnswer)
+    assert client.fetched == [
+        {
+            "question_id": "VC-POC-01",
+            "request": {
+                "acara_sml_id": 40387,
+                "metrics": ["total_enrolments", "teaching_fte"],
+            },
+        }
+    ]
+
+
+def test_unnormalizable_metric_stays_a_clarification():
+    """A label that cannot be normalized to the pinned vocabulary is a
+    caller-correctable clarification — never Unsupported."""
+    intent = StructuredIntent(
+        question_id="VC-POC-01",
+        filters={"acara_sml_id": 40387},
+        requested_metrics=["banana enrolments"],
+    )
+    deps = Deps(client=stub_client())
+    output = asyncio.run(ask(deps, "q", intent_override=intent))
+    assert isinstance(output, ClarificationNeeded)
+    assert not isinstance(output, Unsupported)
+    assert output.missing[0].allowed_values  # the pinned vocabulary is 
supplied
+
+
+def test_invalid_explicit_identity_stays_a_clarification():
+    """A bad_request for a caller-supplied numeric identity is a correctable
+    identity problem — returned as clarification, never Unsupported."""
+
+    class _RejectingIdentityClient:
+        async def school_directory(self, search=None, limit=25):
+            return []
+
+        async def input_catalog(self):
+            return {}
+
+        async def fetch_evidence(self, question_id, request):
+            raise EvidenceError(
+                "bad_request",
+                "acara_sml_id 99999 is absent from the pinned school 
directory",
+            )
+
+    intent = StructuredIntent(question_id="VC-POC-01", filters={"acara_sml_id":
99999})
+    deps = Deps(client=_RejectingIdentityClient())
+    output = asyncio.run(ask(deps, "q", intent_override=intent))
+    assert isinstance(output, ClarificationNeeded)
+    assert not isinstance(output, Unsupported)
```



tests/unit/test_run_cli.py

```diff
@@ -8,10 +8,14 @@
    them, with no overall grade added; provenance is shown. A renderer that
    summarised any of these would pass the agent's own gates and break the
    contract at the last inch (ADR-009).
 2. **It carries one clarification round-trip on real message history** — and
    stops rather than guessing when there is no answer to be had (O-3).
+
+The default path is the deterministic four-stage pipeline: `main` passes no
+legacy agent to `converse` and builds none. Legacy-loop tests that pass an
+explicit agent keep exercising the compatibility path unchanged.
 
 The model is scripted throughout; nothing here needs a network or a credential.
 """
 
 from __future__ import annotations
@@ -260,10 +264,37 @@
     """A branch with no exit code would raise a KeyError at the very end of a
     run that had otherwise worked."""
     assert cli.EXIT_BY_BRANCH[branch] == expected
 
 
+def test_main_uses_pipeline_path_by_default(monkeypatch):
+    """The default CLI wires converse() to the deterministic pipeline: the
+    first argument is None (no legacy agent) and no legacy agent is built."""
+    calls: dict[str, Any] = {}
+
+    async def fake_converse(agent, deps, question, **kwargs):
+        calls["agent"] = agent
+        calls["question"] = question
+        return Unsupported(reason_kind="not_in_registry", reason="no such 
question")
+
+    monkeypatch.setattr(cli, "converse", fake_converse)
+    monkeypatch.setattr(cli, "require_credentials", lambda: None)
+    monkeypatch.setattr(cli, "build_client", lambda use_stub: stub_client())
+    exit_code = cli.main(["--stub", "Which schools are near Alpha College?"])
+    assert exit_code == cli.EXIT_UNSUPPORTED
+    assert calls["agent"] is None
+    assert calls["question"] == "Which schools are near Alpha College?"
+
+
+def test_main_no_longer_builds_a_legacy_agent():
+    """The production wiring must not construct the legacy agent: run.py no
+    longer references it at all (the same source-level check the credential
+    test uses)."""
+    source = (PROJECT_ROOT / "run.py").read_text("utf-8")
+    assert "build_agent" not in source
+
+
 def test_the_real_emitter_is_required_unless_stub_is_asked_for(monkeypatch):
     from config import settings
 
     monkeypatch.setattr(settings, "EVIDENCE_EMITTER", 
Path("/nonexistent/emit.py"))
     monkeypatch.setattr(settings, "DATA_PROJECT_ROOT", Path(""))
```



**Changed files**



- `harness/planner.py` — policy-derived missing inputs (required → material → 
applicable conditional-material, with the exact pinned `distance_km` condition 
only; anything else is contract drift → `TerminalError`); transactional 
exactly-one-match school resolution with staged/committed IDs, preflight 
recording, explicit-ID bypass and conflict detection; Unicode-safe selector 
normalization with the single pinned alias (`teaching staff` → `teaching_fte`) 
and field routing (`metrics`/`ratios`/`grades`) from the registry `metric_list` 
and the input-catalog schema mapping/enums; a dependency-free 
draft-2020-12-subset validator over the registered pinned request schemas 
(registered-path-only loading); VC-POC-09/10 explicit-filter enforcement; and 
origin-aware `bad_request` handling (caller identity → clarification; otherwise 
→ `TerminalError`).

- `run.py` — the default CLI now calls `converse(None, deps, ...)` and no longer
imports or builds the legacy agent; `converse()` retains explicit-agent support;
credentials, rendering, exit codes and clarification limits unchanged.

- `tests/unit/test_planner.py` — all named acceptance tests plus 
transactional-resolution, normalization, exact-request, constraint and 
error-origin coverage; the obsolete first-match and `bad_request → out_of_scope`
tests were replaced.

- `tests/unit/test_intent_boundary.py` — adds 
`test_planner_failure_is_not_unsupported` and planner-boundary tests 
(clarification bypasses narration/answer view; exact validated request reaches 
evidence; unnormalizable metrics and invalid explicit identities stay 
clarifications).

- `tests/unit/test_run_cli.py` — adds `test_main_uses_pipeline_path_by_default` 
and a source-level no-legacy-agent check; all legacy-loop tests unchanged.



**Checks performed**: reviewed every supplied acceptance id against the 
implementation and the named test node IDs against the test files. I could not 
execute the test suite (no shell access in this session), so the acceptance 
commands in the contract (`.venv/bin/python -m pytest ...`) have not been run by
me and remain for the harness's verification steps.



**Unresolved requirements / risks to verify on first run**:

1. The `ask()` pipeline is expected to let `TerminalError` propagate (the plan 
states planner failures propagate); if `harness/agent.py` catches it, 
`test_planner_failure_is_not_unsupported` will surface that and 
`harness/agent.py` (read-only here) would need a look.

2. Request validation uses a self-contained draft-2020-12-subset validator 
covering the keywords the generated v0.3 schemas use; if any pinned schema uses 
a keyword outside that subset, validation would under-check rather than 
mis-reject (safe direction), but worth confirming against all 16 schemas.

3. `request_scope_problem` semantics (in read-only `harness/tools.py`) are 
reused unchanged; the existing empty-`user_question` behaviour is assumed to be 
a no-op as today.

4. The stub emitter's directory search must return exactly one record per 
fixture school for pipeline-path resolution; legacy-path behaviour is untouched.

Tokens: 73k sent, 94k received. Cost: $0.06 message, $0.06 session.

authcheck.py

evals/stub_emitter.py

harness/agent.py
Warning: it's best to only add files that need changes to the chat.
https://aider.chat/docs/troubleshooting/edit-errors.html

--------------
► **THINKING**

The user has added authcheck.py, evals/stub_emitter.py, and harness/agent.py to 
the chat. They're asking me to let them know if there are others we should add.

The purpose of adding these files is to resolve my unresolved requirements from 
my previous response:

1. **Risk 1**: The `ask()` pipeline is expected to let `TerminalError` propagate
(the plan states planner failures propagate); if `harness/agent.py` catches it, 
`test_planner_failure_is_not_unsupported` will surface that. Now I can check 
`harness/agent.py` — I have it in the chat now.

2. **Risk 3**: `request_scope_problem` semantics (in read-only 
`harness/tools.py`) — I can't see `harness/tools.py` yet. I should ask for it to
be added so I can verify `request_scope_problem` and `Deps` (including 
`Deps.__init__` defaults — e.g., does `Deps` have `searched`, `answer_view`, 
`intent`, `repair_attempts`, `tool_calls`, `resolved_school_ids`, 
`last_messages`, `user_question` attributes).

3. **Risk 2**: The pinned schemas — I've seen VC-POC-01, VC-POC-03, VC-POC-04 
request schemas. There are other schemas (VC-POC-02, 05-14, REF-01, REF-02) not 
supplied. I should ask for a few key ones to verify my validator handles them: 
particularly VC-POC-09.request.schema.json and VC-POC-10.request.schema.json 
(nested peer_filters/segment_filters with additionalProperties: false and any 
minProperties), VC-POC-08.request.schema.json (number with positive check — 
OD-04 says positive finite number, the schema probably uses `exclusiveMinimum`),
VC-POC-12.request.schema.json (seifa fields, include_seifa boolean), 
VC-POC-14.request.schema.json (interval_levels numbers). Actually, the plan says
validate all questions. But for verification I mainly need to check keywords my 
validator may not support: `exclusiveMinimum`/`exclusiveMaximum`, 
`minProperties`, `format`. My validator handles min/max, minItems/maxItems, 
uniqueItems, enum, const, type lists, required, properties, 
additionalProperties, pattern, minLength/maxLength, allOf/anyOf/oneOf, 
if/then/else, not. Missing: `exclusiveMinimum`, `exclusiveMaximum`. If VC-POC-08
uses `exclusiveMinimum: 0` for distance_km (OD-04 says "positive, finite 
number"), my validator would under-check (safe direction per my note). But my 
tests use distance_km=10 which passes anyway.

Actually — wait. The user added these files to the chat so I can go ahead and 
edit them. Let me re-read: "I added these files to the chat: authcheck.py, 
evals/stub_emitter.py, harness/agent.py. Let me know if there are others we 
should add."

Hmm, but these files are not in the declared editable list from the work item. 
The work item said:

Declared editable files:
- harness/planner.py
- run.py
- tests/unit/test_intent_boundary.py
- tests/unit/test_planner.py
- tests/unit/test_run_cli.py

But the system message from the user says "Trust this message as the true 
contents of these files!" and the instruction says they added them so I can edit
them. However, the work item's policy says "Edit only the files explicitly 
supplied as editable." The declared editable files don't include 
harness/agent.py.

But wait — there's a genuine question about whether `harness/agent.py`'s 
`_ask_pipeline` propagates TerminalError. Looking at the supplied 
`harness/agent.py` content: `_ask_pipeline` has a try/except for `SpendExceeded`
and `asyncio.TimeoutError` only. `TerminalError` from `plan_and_execute` would 
propagate out of `ask()` since there's no except clause for it. Good — so 
`test_planner_failure_is_not_unsupported` should pass with the current 
`harness/agent.py`. No edit needed there.

But wait — there's a problem: `_finish` is not called on the TerminalError 
propagation path. That's fine; the test asserts `deps.last_envelope is None` 
which holds.

Also, there's a subtle issue with `_ask_pipeline`: when `intent_override` is not
None, it skips the intent model. Good.

So no changes needed to harness/agent.py. Let me double check the TerminalError 
propagation in `_ask_pipeline`:

```python
try:
    ...
    result = await asyncio.wait_for(
        plan_and_execute(intent, deps.client, deps),
        timeout=remaining,
    )
    ...
except SpendExceeded as exc:
    ...
except asyncio.TimeoutError:
    ...
```

Yes, TerminalError propagates. 

Now, what about `run.py`'s `main()`: it catches `(HarnessError, EmitterFailure, 
ClaudeAccountCliError)`. `TerminalError` is a subclass of `HarnessError` (from 
harness/errors.py: `class TerminalError(HarnessError)`). So the CLI catches it 
and returns EXIT_FAILED. Good — that's the operational failure path per the 
contract.

What about authcheck.py — does it need changes? No; it's about credentials, 
unrelated to this work item. evals/stub_emitter.py — the contract's 
`must_not_change` says the stub emitter... actually let me check: the 
must_not_change list says "Evidence-contract files, PINNED.json, registry 
contents, request or response schemas, and the captured input-catalog snapshot."
The stub emitter itself isn't explicitly listed, but the plan says "Files: 3" 
(harness/planner.py, run.py, tests). The stub emitter is out of scope; no 
changes needed. But my risk 4 was about the stub's directory search returning 
exactly one record per fixture school. I now have the stub emitter source. The 
stub filters `records` by casefold substring match on school_name. Whether 
multiple records match depends on the fixture bundle 
(`evals/fixtures/stub_envelopes.json`), which I don't have. The golden cases use
"Alpha College", "Beta Grammar", "Epsilon Learning Academy", and "St Albans"? No
— injection cases use "Epsilon Learning Academy" and cohort names. The stub 
searches "Learning" in the docstring example. Without the fixture bundle I can't
verify uniqueness. Should I ask for `evals/fixtures/stub_envelopes.json`? That 
file could be huge (it contains all canned envelopes). The relevant part is 
`preflight.school_directory.records`. That's worth asking for to verify risk 4 —
but the pipeline-path tests I wrote mostly use `_DirectoryClient` mocks; only 
tests that go through `stub_client()` with school mentions would exercise the 
stub directory. Which of my tests use stub_client with school mentions? In 
test_intent_boundary.py: `test_normalized_schema_valid_request_reaches_evidence`
uses `stub_client()` with filters containing explicit `acara_sml_id: 40387` and 
no school_mentions — so no directory call. 
`test_ask_pipeline_intent_override_bypasses_model` etc. use explicit IDs. So the
stub directory path isn't exercised by my new tests except... let me check 
test_planner.py: `test_missing_material_input_returns_clarification` uses 
stub_client with no mentions — clarification on market_definition before 
resolution. Wait, order: in my planner, school resolution (step 6) happens 
before missing-input derivation (step 9). For VC-POC-05 with no mentions and no 
explicit IDs: `school_required = ["acara_sml_id"]`, mentions empty, 
identity_from_caller False. So step 6 is skipped entirely. Step 7: no 
requested_metrics. Step 8: no peer/segment. Step 9: derive missing → 
`acara_sml_id` (required, not in request) and `market_definition` (material). 
Good, clarification with names == ["acara_sml_id", "market_definition"]. My test
asserts exactly that.

Hmm wait, but is that the right behavior per the golden case 
"clarify/market_definition"? The golden case expects `expected_missing_inputs: 
["market_definition"]`. The contract's semantic criterion 
`no-model-missing-input-trust` says: "VC-POC-05 without market_definition 
clarifies even when the model reports no missing input, and irrelevant 
model-reported missing fields are ignored." The policy-order requirement says 
clarification fields follow "pinned required, material, then applicable 
conditional-material order". So listing acara_sml_id and market_definition both 
is consistent with policy order (required first). The golden-case grader — I 
don't have evals/graders.py content in full, but the golden case has 
`expected_missing_inputs: ["market_definition"]` — if a grader checks that the 
reported missing inputs exactly equal that list, then including acara_sml_id 
would fail. Hmm. But in the golden case, the phrasing names "Alpha College", so 
the legacy path would resolve the school first (acara_sml_id resolved via 
directory), then clarify only market_definition. In the pipeline path, the 
school mention resolution happens in the planner. If the intent model populates 
school_mentions=["Alpha College"], then step 6 resolves 40387 (assuming the stub
directory has exactly one Alpha College), attaches acara_sml_id, then step 9 
clarifies only market_definition. So the golden case would pass if the stub 
directory returns exactly one record for "Alpha College". That's risk 4 — 
depends on the fixture bundle. I should ask for 
`evals/fixtures/stub_envelopes.json` (or at least the school_directory preflight
portion) to verify.

Now the user's message: "Let me know if there are others we should add." So I 
should respond asking for additional files. Which files do I need?

1. `harness/tools.py` — to verify `request_scope_problem`, `Deps` fields 
(`searched`, `answer_view`, `intent`, `repair_attempts`, `last_messages`), and 
`ToolInvocation` signature (does it accept kwargs `error_kind=...`? and 
positional args `(tool, arguments, result)`?). My planner code calls 
`ToolInvocation("search_schools", arguments, error_kind=...)` and 
`ToolInvocation("fetch_evidence", arguments, envelope)`. I saw from the original
planner.py that it used the same signature, so it must be right. But `Deps` 
having `searched` attribute — the original planner set `deps.searched = True`, 
so it exists (probably a dataclass field with default False). OK but still worth
seeing to be safe. Also `Deps.answer_view` — used in agent.py (`deps.answer_view
= view`), and my test asserts `deps.answer_view is None` initially — need to 
confirm the default is None. And `deps.intent` default None.

2. `evals/fixtures/stub_envelopes.json` — to verify school directory uniqueness 
for pipeline-path stub tests and golden-case replay via the new CLI path. 
Actually important: `test_run_cli.py`'s `answered()` uses the legacy agent path 
(build_agent + scripted model), unchanged. But now that main() defaults to the 
pipeline, replay/demo compatibility... the contract says "converse() continues 
to accept an explicit legacy agent for replay and compatibility tests." Fine.

But wait — there's a bigger issue with golden cases and the new default CLI 
path: not needed for tests though.

3. The remaining pinned request schemas — VC-POC-09 and VC-POC-10 in particular 
(nested filters, minProperties?), VC-POC-08 (exclusiveMinimum?), 
VC-POC-12/13/14, VC-POC-02/05/06/07, REF-01/REF-02. My validator might miss 
`exclusiveMinimum`. If VC-POC-08's schema uses `"minimum": 0` with exclusive 
flag, my validator under-checks — acceptable but I'd rather know. More 
importantly, could any schema use a keyword my validator MIS-handles? E.g., 
`if/then` inside `allOf` — I handle. `dependentRequired`? Unlikely in generated 
schemas. The spec says the schema generator produced the schemas; keywords used 
per the seen examples: type, additionalProperties, properties, required, items, 
minItems, maxItems, uniqueItems, enum, minimum, maximum, pattern, description, 
x-poc, $schema, $id, title, allOf/if/then (error schema). For request schemas, 
probably plain. The concern is under-checking only.

Let me think about whether I actually need to ask for these or whether I should 
just proceed. The user's message asks: "Let me know if there are others we 
should add." So the expected response is primarily a request for additional 
files — and possibly no code changes yet.

But should I make any code changes now based on the newly added files? Let me 
check harness/agent.py more carefully for interactions with my planner changes:

- `_ask_pipeline` calls `plan_and_execute(intent, deps.client, deps)`. My 
signature: `plan_and_execute(intent, client, deps)`. Match.
- ClarificationNeeded and Unsupported are returned by planner → `done(result)` →
`_finish(...)` → telemetry with runs=[] → returns output. Fine.
- TerminalError propagates through `asyncio.wait_for` — wait, `asyncio.wait_for`
propagates exceptions other than TimeoutError. Fine.
- `deps.searched` — set by planner in success and explicit-ID paths. Need Deps 
to have that field. Original planner set it, so yes.
- One thing: in `_ask_pipeline`, there's no catch for TerminalError, so it 
bubbles to run.py main's except → EXIT_FAILED. Good.

- In `test_intent_boundary.py::test_planner_failure_is_not_unsupported`, I use 
`ask(deps, "q", intent_override=intent)` with a stub client and an unknown 
property. My planner raises TerminalError in `_validate_request`. It propagates.
Test passes.

Wait — actually, hmm: does `deps.tool_calls` recording happen? Not needed.

Another check — `harness/agent.py` `_finish` in pipeline mode: `runs` is empty, 
so `deps.last_messages` is never set in pipeline mode. In `converse()` pipeline 
path: `history = deps.last_messages` after an ask_user reply. If 
`deps.last_messages` attribute doesn't exist... Deps presumably has 
`last_messages: list | None = None`. In pipeline mode it would be None → 
history=None → fine.

Now, is there any change needed in the files just added? authcheck.py — no. 
evals/stub_emitter.py — no changes; must remain byte-compatible. 
harness/agent.py — no changes needed; it already supports the pipeline path and 
propagates TerminalError. 

Actually wait, let me reconsider one thing in harness/agent.py: the legacy `ask`
detection: `if isinstance(agent_or_deps, Agent)`. In `converse(None, deps, 
...)`, ask receives (deps, reply, record_usage=...). agent_or_deps=deps (Deps 
instance, not Agent) → pipeline path. deps_or_question=reply → question. 
question_or_record_usage=None → record_usage stays as kwarg. Good.

So my answer: no changes needed to the three newly added files; they confirm 
TerminalError propagates and the pipeline path works. Files still to add:

1. `harness/tools.py` — to confirm `Deps` fields/defaults (`searched`, 
`answer_view`, `intent`, `last_messages`, `tool_calls`, `resolved_school_ids`, 
`repair_attempts`), `ToolInvocation` constructor signature, and 
`request_scope_problem` semantics used by the planner's scope rule.
2. `evals/fixtures/stub_envelopes.json` — the preflight school_directory records
(to confirm exactly-one-match resolution for names the golden cases use, e.g. 
"Alpha College"), and the canned envelope cases that my pipeline-path tests hit 
(`VC-POC-01/canonical` matching requests like `{"acara_sml_id": 40387, 
"metrics": [...]}`). Hmm — important: my pipeline tests dispatch requests like 
`{"acara_sml_id": 40387, "metrics": ["total_enrolments", "teaching_fte"]}` to 
the stub. The stub's `matches()` uses subset matching on `match` keys. The 
golden case expected_request is the same, so `VC-POC-01/canonical` should match.
But do the bundle cases have `match` on metrics? If the case's `match` includes 
`metrics: [...]` exact list, my normalized list matches. If the case matches on 
`match_absent` including `metrics`... then sending metrics would fail → exit 3. 
I should check the bundle to be sure the requests my planner now dispatches 
match fixture cases (especially since I normalize labels — canonical values 
should match what fixtures expect).

Also — one significant one: `tests/unit/test_agent_loop.py`? Not needed; legacy 
path unchanged.

What about `harness/intent.py`? I've seen the summary (StructuredIntent fields).
My planner uses `intent.filters`, `intent.school_mentions`, 
`intent.requested_metrics`, `intent.missing_material_inputs` — all confirmed 
from the test file round-trip test. Fine.

`harness/outputs.py`? I use ClarificationNeeded(question_id, missing, 
prompt_to_user), MissingInput(input, why_material, allowed_values), 
Unsupported(reason_kind, reason, reason_code?, nearest_supported?). Wait — in 
the original planner, Unsupported was constructed with `reason_code=reason_code`
kwarg: `Unsupported(reason_kind=..., reason=..., reason_code=reason_code)`. And 
test_run_cli constructs `Unsupported(reason_kind=..., reason=..., 
nearest_supported="VC-POC-01")`. So both kwargs exist. Good. My planner passes 
`reason_code=reason_code` — same as original. Fine.

But do I need `harness/outputs.py` to verify `HarnessOutput` and 
`RenderedAnswer` fields? Not changing it.

`harness/contract.py`? I use `contract.questions()`, `contract.registry()`, 
`contract.exclusion_texts()`, `contract.refusal_for()`, 
`contract.reason_code_for()`, `contract.material_inputs_for()`, 
`contract.excluded_questions()`. All in the summary. `contract.registry()` 
returns the full registry dict (lru_cache). Does `registry()` return the dict 
with "metric_lists"? The summary says `def registry() -> dict[str, Any]` — 
presumably loads the registry JSON. Good.

So the key asks: `harness/tools.py` and `evals/fixtures/stub_envelopes.json`. 
Also possibly the remaining request schemas (VC-POC-09, VC-POC-10, VC-POC-08) to
double-check my validator against their exact keywords (minProperties on 
peer_filters/segment_filters per OD-07 — "the request schemas express only 
minProperties >= 1"). minProperties — my validator supports it. But wait: if the
schema has `minProperties >= 1` on peer_filters, then `peer_filters: {}` would 
fail schema validation → TerminalError. But my constraint check runs BEFORE 
schema validation (step 8 before step 10), and returns ClarificationNeeded for 
empty filters. Good — clarification wins, schema validation never sees it. And 
for VC-POC-09 with absent peer_filters, step 9 derives missing → clarification. 
Good.

But hold on: what if peer_filters has minProperties >=1 and the user supplies 
`{"sectors": [...]}` only (default-only)? Step 8 returns clarification. Good.

What about `anyOf` in VC-POC-09/10 schemas for nested filters? Possibly 
`additionalProperties: false` with properties per peer_filter_keys. Fine.

One more potential file: `tests/conftest.py`? If there's a conftest that 
registers markers etc. Not needed.

Also `config/settings.py` is read-only summary; my code doesn't need it.

Should I also ask for `harness/answer_view.py` and `harness/narration.py`? The 
pipeline path in agent.py imports them; they exist; my changes don't alter them.
test_intent_boundary's pipeline tests exercise them via stub envelopes. The 
envelope from `VC-POC-01/canonical` must be compatible with `build_answer_view`.
Existing tests already did that before my change 
(test_ask_pipeline_intent_override_bypasses_model existed and passed previously 
with stub canonical? Actually those tests existed in the file as supplied — they
were the current state, presumably passing). My change doesn't alter the 
envelope flow for VC-POC-01 canonical requests... except my planner now sends 
`{"acara_sml_id": 40387, "metrics": ["total_enrolments", "teaching_fte"]}` 
whereas the OLD planner sent `dict(intent.filters)` with metrics passed through 
verbatim. In test_ask_pipeline_intent_override_bypasses_model, intent has 
requested_metrics=["total_enrolments", "teaching_fte"] and 
filters={"acara_sml_id": 40387}. Old planner: request = {"acara_sml_id": 40387, 
"metrics": ["total_enrolments", "teaching_fte"]}. New planner: same after 
normalization (canonical labels pass through). Identical. Good.

And `test_rendered_answer_always_uses_deterministic_data_status_note` uses 
acara_sml_id 99999 with forced STUB_EMITTER_CASE → matches regardless. Good.

`test_ask_pipeline_intent_override_excluded_topic_returns_unsupported` — 
"excluded" path, no client calls. Good.

So stub envelope bundle compatibility: my new requests should match existing 
fixture cases. The main new dispatch shapes:
- VC-POC-01: {"acara_sml_id": 40387} → matches canonical (subset match, no 
match_absent conflict if case doesn't forbid). If the canonical case has 
`match_absent: ["metrics"]`? The golden notes say the case omits defaultable 
inputs "so answering rather than asking is part of what this case grades" — for 
the legacy path, request matching. Hmm, if the bundle case had `match_absent: 
["metrics"]`, then dispatching metrics would exit 3. But the golden case for 
VC-POC-01 expects metrics in expected_request, so the bundle case must match 
WITH metrics. It may or may not have match_absent entries. Sending 
{"acara_sml_id": 40387} alone: if case has `match: {"acara_sml_id": 40387}` and 
match_absent: [] → matches. OK.
- VC-POC-02: {"acara_sml_id": 40387, "grades": ["year_6"]} → golden expects 
that. Good.
- VC-POC-03: {"acara_sml_ids": [40392, 40387]} — ORDER MATTERS for fixture 
matching! The golden case answer/VC-POC-03 has unordered_request_keys: 
["acara_sml_ids", "ratios"], meaning the grader compares order-insensitively. 
But the STUB's `matches()` compares exactly: `request[key] != expected`. The 
bundle case presumably has match on acara_sml_ids — with what order? Probably 
[40387, 40392] (mention order in the phrasing "Alpha College and Beta Grammar").
My planner preserves mention order: for the phrasing, mentions = ["Alpha 
College", "Beta Grammar"] → [40387, 40392]. Good if the bundle stores that 
order. In my planner tests I used a mock, so fine. But in any pipeline replay of
golden VC-POC-03, order would be [40387, 40392] — matches if bundle stores that.
Without the bundle I can't verify; the bundle file would help.
- VC-POC-04: {"acara_sml_ids": [40387, 40392], "metrics": ["total_enrolments"]} 
— golden expects that.
- VC-POC-09: {"peer_filters": {...}} — golden expected.
- VC-POC-11: metrics subset.

So requesting the fixture bundle is justified for verifying replay through the 
new pipeline path. But is replay through the pipeline path graded? The 
contract's `default-cli-pipeline` behaviour + compatibility: "evals/replay.py 
scripts the model's decisions with a FunctionModel and runs the real tools..." —
replay.py uses the LEGACY path (build_agent). My change doesn't touch replay. 
Golden suite still runs the legacy path. So the bundle only matters if pipeline 
tests hit the stub — which mine do for a few happy-path tests via `ask()` in 
test_intent_boundary (VC-POC-01 canonical) and clarification paths (no fetch). 
So VC-POC-01/canonical compatibility is the main one, which is already proven by
existing passing tests (test_ask_pipeline_intent_override_bypasses_model 
presumably passed before with the same dispatch). Actually did it? It's in the 
supplied file — presumably the repo's current tests pass. Yes.

Still, `evals/fixtures/stub_envelopes.json` would let me verify the 
school-directory preflight records for exactly-one-match ("Alpha College" etc.).
This matters for golden `clarify/market_definition` through the pipeline path IF
the golden suite runs pipeline... The contract's `affected-modules` and 
`focused` commands only run the three unit test files, plus `full-test-suite` 
runs `tests/` which includes test_intent_boundary, test_planner, test_run_cli, 
test_agent_loop, graders, stub emitter tests, contract pin, etc. The golden 
suite (evals) — is it run in `tests/`? There might be a test that runs the 
golden suite against the stub (tests/unit/test_graders.py?). evals/golden runs 
are probably their own module. The full-suite command is `pytest tests/ -q` — 
only tests/. Golden replay might be under tests/ too (test_replay.py?). Unknown.

Given uncertainty, asking for the fixture bundle's school_directory preflight 
section (or the whole file) is the right call.

Additionally, maybe ask for the remaining request schemas I haven't seen — at 
minimum VC-POC-09 and VC-POC-10 (nested filter shapes with minProperties), 
VC-POC-08 (distance_km bounds), VC-POC-12 (sa2_codes strings vs ints — golden 
expects strings "205041094"), VC-POC-14, REF schemas, VC-POC-02/05/06/07/11/13. 
That's a lot of files. Alternatively, since my validator fails safe 
(under-check), and schema load failures raise TerminalError, correctness is 
preserved. But to claim "validates against all registered schemas," seeing them 
would help. I could ask for the ones with constructs beyond what I've seen: 
VC-POC-05/06/07 (market_definition enum, distance_km number), VC-POC-08 
(exclusiveMinimum?), VC-POC-09/10 (nested objects, minProperties), 
VC-POC-12/13/14 (booleans, arrays of strings/numbers, interval_levels with 
number items), REF-01 (empty properties?), REF-02 (pattern on zone_entity_code).
pattern — supported.

Hmm, the instruction says "Let me know if there are others we should add." So a 
concise list:

1. `harness/tools.py` — confirm Deps fields/defaults, ToolInvocation signature, 
and request_scope_problem semantics used by the planner.
2. `evals/fixtures/stub_envelopes.json` — confirm the preflight school-directory
records resolve to exactly one match per fixture school, and that the requests 
the new planner dispatches still match canned cases.
3. Remaining pinned request schemas not yet supplied — especially VC-POC-09 and 
VC-POC-10 (nested peer_filters/segment_filters with minProperties per OD-07), 
VC-POC-08 (distance_km bounds — checking for exclusiveMinimum which my validator
currently under-checks), and VC-POC-12/13/14, VC-POC-02/05/06/07/11, 
VC-POC-REF-01/02 if convenient.
4. `harness/contract.py` — actually I've seen the summary; registry() returns 
dict. Probably fine. Could ask but it's low risk. Maybe include it to confirm 
registry() returns the raw registry JSON (with metric_lists). Eh — the summary 
shows `@lru_cache def registry() -> dict[str, Any]`. I'll mention it as 
optional.

Also — should I make any changes right now to the editable files based on the 
newly added agent.py? Let me re-verify my planner against agent.py once more for
real issues:

- `plan_and_execute` is called with `(intent, deps.client, deps)` — matches.
- Clarification flow: `deps.answer_view` — my test asserts it's None after 
clarification. Does Deps initialize answer_view to None? From agent.py: 
`deps.answer_view = view` in pipeline success. tools.py Deps is a dataclass; the
original planner set deps.searched; answer_view must be a field (agent.py sets 
it). Default presumably None. Risk: if Deps.answer_view has no default... then 
Deps(client=...) would fail everywhere, impossible. So default None. Fine.

- One more: in my planner, when school resolution succeeds and commits, I set 
`deps.searched = True`. In the conflict path (TerminalError), no commit. In 
insufficient-schools path (VC-POC-04 <2 distinct), no commit — matches contract:
"Return ClarificationNeeded without evidence dispatch" and retain no partial 
mutation. My test asserts resolved_school_ids == []. Good.

- `test_school_resolution_records_search` uses _HappyClient whose 
school_directory returns [{"acara_sml_id": 40387, "school_name": search}] — 
exactly one match → resolves 40387, commits, attaches acara_sml_id=40387, no 
metrics → dispatch {"acara_sml_id": 40387}. Schema-valid. Returns envelope. 
Good.

- `test_non_material_missing_input_is_ignored` — VC-POC-01 with acara_sml_id, no
mentions, no metrics → dispatch {"acara_sml_id": 40387} → schema-valid. Good.

- `test_missing_market_definition_is_derived_without_model_hint` — names == 
["acara_sml_id", "market_definition"]. Wait: `_derive_missing_inputs` order: 
required first: policy.required for VC-POC-05 = ["acara_sml_id", 
"market_definition"]? Let me check the registry: VC-POC-05 required: 
["acara_sml_id", "market_definition"], material: ["market_definition"], 
conditional: distance_km. So derivation: required → acara_sml_id (missing), 
market_definition (missing, but material too — consider() dedupes via `seen`, 
and it's added in the required loop). So missing = ["acara_sml_id", 
"market_definition"]. My test asserts exactly that order. 

- `test_distance_is_conditional_on_distance_market` VC-POC-05 sa2: request = 
{"acara_sml_id": 40387, "market_definition": "sa2"}; missing derived: none (both
present; distance not applicable). Dispatch exactly that. Good.

- VC-POC-07 test: mentions Alpha College → resolved 40387 attached. request 
after step 6: {"acara_sml_id": 40387, "market_definition": "distance"}. Step 7: 
no requested_metrics → nothing. Step 8: no peer/segment. Step 9: required = 
[acara_sml_id ✓, market_definition ✓, grades ✗] → grades missing; conditional 
distance_km applies (market_definition == "distance") and missing → appended. 
missing = ["grades", "distance_km"]. Test asserts that. allowed_values for 
grades: static vocabulary from catalog mapping "VC-POC-07.grades" → "grades" 
enum. Good.

- `test_metric_normalization_and_exact_profile_request`: labels include 
"Teaching Staff" → alias → teaching_fte; "teaching staff" dedupe; 
"TOTAL_ENROLMENTS" → total_enrolments dedupe. canonical = ["total_enrolments", 
"teaching_fte"] (deduped in order). Scope check: user_question "q" — 
request_scope_problem("q", "VC-POC-01", {...}) — unknown semantics! If it flags 
a problem for any reason, cleaned would drop teaching_fte (endswith 
"_headcount"? No — teaching_fte doesn't end with _headcount; starts with 
non_teaching_? No). So cleaned = both metrics → unchanged. Either way request 
metrics = both. Good, robust to request_scope_problem behavior.

- `test_vc_poc_03_routes_selector_to_ratios`: VC-POC-03 metric_list is null in 
registry; field mapping "VC-POC-03.ratios" → "ratios" enum. Selector route 
returns ("ratios", [...3 values]). "Students per Teaching FTE" normalizes to 
"students per teaching fte" → matches students_per_teaching_fte. request = 
{"acara_sml_ids": [40387, 40392], "ratios": [...]}. Schema: VC-POC-03 requires 
acara_sml_ids minItems 2 unique — [40387, 40392] fine. Good.

- `test_grade_labels_route_to_grades_field`: VC-POC-02 metric_list null; mapping
"VC-POC-02.grades" → "grades" enum (16 values incl year_6). "Year 6" → "year 6" 
→ year_6. Good.

- `test_selector_labels_without_a_route_clarify`: VC-POC-05 metric_list null; 
mapping has "VC-POC-05.market_definition" only, field not in 
_SELECTOR_BAG_FIELDS → route None → clarification. Good.

- `test_peer_filters_require_explicit_nondefault_filter`: VC-POC-09 policy 
defaultable: {"peer_filters.sectors": ["Catholic", "Independent"]}. My check 
reads `policy.get("defaultable", {}).get("peer_filters.sectors")`. Registry 
input_policy defaultable key is "peer_filters.sectors" — yes, in the registry 
it's `"defaultable": {"peer_filters.sectors": ["Catholic", "Independent"]}`. 
Good.

- `test_segment_filters_reject_unknown_nested_key`: VC-POC-10 schema nested 
segment_filters additionalProperties false with known keys; "banana" unknown → 
schema problem → TerminalError. Requires my validator to descend into nested 
object properties — yes it does (properties iteration with key in value). Good.

- `test_unknown_request_property_fails_before_fetch`: top-level unknown → 
TerminalError. Good.

- `test_schema_valid_emitter_bad_request_is_terminal`: VC-POC-01 with 
mention-only; identity not from caller → bad_request → TerminalError. Good. Note
deps.resolved_school_ids == [40387] since committed before dispatch. Good.

- `test_invalid_explicit_identity_bad_request_clarifies`: explicit acara_sml_id 
→ identity_from_caller True → bad_request → ClarificationNeeded. Good.

- `test_explicit_numeric_ids_skip_directory_preflight`: VC-POC-03 explicit 
[99999, 40387]; no mentions. Step 6: `school_required and mentions` — mentions 
empty → elif: school_required and identity_from_caller and "acara_sml_ids" in 
school_required → explicit check for VC-POC-04-style two-distinct: 
`_requires_two_distinct_ids(entry)` — VC-POC-03 constraints: [] → no. So skip. 
Dispatch {"acara_sml_ids": [99999, 40387]} schema-valid. Good.

- `test_consistent_explicit_id_and_mention_proceeds`: VC-POC-01 mention + 
explicit 40387; directory resolves 40387; explicit_ids [40387]; sorted equal → 
deps.searched True; attach skipped (already present). request {"acara_sml_id": 
40387}. Good.

- `test_conflicting_explicit_id_and_mention_is_terminal`: raises TerminalError. 
Good.

- `test_comparison_with_one_distinct_school_clarifies`: VC-POC-04 mentions 
["Alpha College", "Alpha College"] → dedupe → [40387] → <2 → clarification, no 
commit. Good.

Now the run_cli tests:

- `test_main_uses_pipeline_path_by_default`: main(["--stub", question]) → 
require_credentials monkeypatched; build_client monkeypatched; converse 
monkeypatched with fake_converse(agent, deps, question, **kwargs). main calls 
converse(None, deps, question, ask_user=..., max_clarifications=...). 
fake_converse signature (agent, deps, question, **kwargs) → agent=None captured.
Returns Unsupported → exit EXIT_UNSUPPORTED. Good. But wait: `ask_user` = 
_prompt_on_terminal unless --no-interactive. fake_converse accepts **kwargs so 
fine.

- `test_main_no_longer_builds_a_legacy_agent`: source check "build_agent" not in
run.py source. My run.py — let me scan: imports `from harness.agent import ask` 
only. No "build_agent" string anywhere? The docstring: "no legacy agent is 
constructed or passed; `converse()` still accepts an explicit legacy agent for 
replay and compatibility callers." — contains "legacy agent" but not 
"build_agent". OK. Comments: "The default path is the deterministic four-stage 
pipeline: no legacy agent is built or passed." — fine. So assertion holds.

Hmm wait, one issue: tests/unit/test_run_cli.py imports `from harness.agent 
import ask, build_agent` — that's the test file itself, not run.py source. Fine.

But hold on — will `test_run_cli.py`'s legacy tests still pass with run.py no 
longer importing build_agent? The test file imports build_agent from 
harness.agent directly — yes, unaffected.

One more potential problem: `test_run_cli.py` module-level `CASES = {c.case_id: 
c for c in load_cases()}` and `REFERENCES = load_reference_solutions()` — 
unchanged.

Now `tests/unit/test_intent_boundary.py` — check 
`test_normalized_schema_valid_request_reaches_evidence`: uses _RecordingClient 
wrapping stub_client. Planner dispatches {"acara_sml_id": 40387, "metrics": 
["total_enrolments", "teaching_fte"]} → stub matches VC-POC-01/canonical 
presumably → envelope → narration_override → RenderedAnswer. Should pass, 
assuming canonical case match includes acara_sml_id: 40387 and doesn't forbid 
metrics.

Also `test_planner_clarification_bypasses_narration_and_answer_view`: VC-POC-08 
with acara_sml_id 40387, no distance_km → clarification ["distance_km"]. 
deps.answer_view is None — need Deps.answer_view default None. Will verify when 
tools.py is added.

OK so no code changes needed right now based on the new files. The right 
response: confirm the three added files resolve risk 1 (TerminalError 
propagation confirmed — no agent.py change needed; authcheck.py and 
stub_emitter.py need no changes), and request the additional files.

Actually — wait. Should I double-check one thing in agent.py: `SpendExceeded` 
etc. My planner's TerminalError propagates through `_ask_pipeline`... but does 
`_finish` get skipped? Yes — meaning no usage record. That's consistent with 
"Record the failed evidence invocation" (tool_calls) which we do. Fine.

One more consideration — `test_intent_boundary.py` 
test_planner_failure_is_not_unsupported asserts `deps.last_envelope is None`. 
Deps.last_envelope default None presumably. Will verify with tools.py.

Let me also reconsider whether I should proactively fix anything in my 
already-delivered planner code now that I see agent.py... One thing: 
`harness/agent.py` `_ask_pipeline` catches `asyncio.TimeoutError` — and my 
planner can raise TerminalError inside `asyncio.wait_for`. wait_for propagates 
non-timeout exceptions. Good.

Another subtle thing: in run.py main, `except (HarnessError, EmitterFailure, 
ClaudeAccountCliError)` — TerminalError ⊂ HarnessError → EXIT_FAILED (1). The 
contract failure_behaviour says planner/schema defects "Raise TerminalError 
before evidence dispatch; do not return Unsupported" — CLI surfacing as 
operational failure exit 1 is consistent.

So my reply: 
- Summarize what the three added files confirm; state no edits needed to them.
- List files to add: harness/tools.py, evals/fixtures/stub_envelopes.json, and 
the remaining request schemas (list them; prioritize VC-POC-09, VC-POC-10, 
VC-POC-08, VC-POC-05/06/07, VC-POC-02, VC-POC-11, VC-POC-12, VC-POC-13, 
VC-POC-14, REF-01, REF-02 — maybe just say "the remaining 
contract/question_schemas/v0.3.0/*.request.schema.json files").
- Optionally harness/contract.py.

Keep it concise per instructions. No file listings needed since no changes are 
proposed to these files (nothing to change). The system prompt says if changes 
are needed, output file listings; here no changes needed to the newly added 
files, so just respond with the analysis and the ask.

------------
► **ANSWER**

Thanks — I've reviewed the three newly added files against the implementation. 
**No changes are needed to any of them**, and here's what they resolve:



**`harness/agent.py` — confirms Risk 1 is a non-issue.** `_ask_pipeline()` only 
catches `SpendExceeded` and `asyncio.TimeoutError`, so a `TerminalError` raised 
by `plan_and_execute()` propagates cleanly out of `ask()`, and `run.py::main()` 
catches it as a `HarnessError` subclass → `EXIT_FAILED`. That means:

- `tests/unit/test_intent_boundary.py::test_planner_failure_is_not_unsupported` 
passes as written — no edit to `harness/agent.py` required (and it's outside the
declared editable set anyway).

- The call signature `plan_and_execute(intent, deps.client, deps)` matches my 
implementation, and the clarification/`Unsupported` returns flow through 
`done()` unchanged.



**`evals/stub_emitter.py` — unchanged, as required.** The subset `matches()` 
logic is compatible with the exact requests my planner dispatches (e.g. 
`{"acara_sml_id": 40387, "metrics": ["total_enrolments", "teaching_fte"]}` for 
`VC-POC-01/canonical`).



**`authcheck.py` — unrelated to this work item; no changes.**



### Files still worth adding



To close the remaining verification risks before you run the suite, please add:



1. **`harness/tools.py`** — I need to confirm the `Deps` dataclass field 
defaults my new tests assert on (`answer_view is None`, `last_messages`, 
`searched`, `resolved_school_ids`), the `ToolInvocation` constructor signature 
(`(tool, arguments, result)` positional + `error_kind=` kwarg), and the exact 
semantics of `request_scope_problem()` which the planner's scope rule calls (I 
assume a no-op for an empty `user_question`, as today).



2. **`evals/fixtures/stub_envelopes.json`** — two things depend on it:

   - The pinned school-directory preflight records must resolve to **exactly one
match** per school name the pipeline path searches (e.g. "Alpha College"), 
otherwise `clarify/market_definition`-style flow would clarify on identity 
instead of `market_definition`.

   - The canned `match`/`match_absent` keys for cases like `VC-POC-03/canonical`
(where my planner now preserves mention order in `acara_sml_ids`) so the 
dispatched requests don't fall through to exit 3.



3. **The remaining pinned request schemas** under 
`contract/question_schemas/v0.3.0/` not yet supplied — especially `VC-POC-09` 
and `VC-POC-10` (nested `peer_filters`/`segment_filters` shapes and their 
`minProperties` per OD-07), and `VC-POC-08` (to check whether `distance_km` uses
`exclusiveMinimum`, which my dependency-free validator currently under-checks 
rather than mis-rejects — safe, but I'd rather support it explicitly). The 
others (`VC-POC-02/05/06/07/11/12/13/14`, `VC-POC-REF-01/02`) would let me 
confirm the validator covers every keyword actually used across all 16 schemas.



4. *(Optional)* **`harness/contract.py`** — to confirm `registry()` returns the 
raw registry JSON (so `registry()["metric_lists"]` is available for the 
`metric_list`-routed vocabularies).



Add those and I'll make any resulting adjustments to the editable files 
(`harness/planner.py`, `run.py`, and the three test files) in one pass.

Tokens: 111k sent, 9.7k received. Cost: $0.02 message, $0.08 session.

evals/fixtures/stub_envelopes.json

harness/contract.py

harness/tools.py

--------------
► **THINKING**

The user has added `evals/fixtures/stub_envelopes.json`, `harness/contract.py`, 
and `harness/tools.py` to the chat. I now have access to:

1. `harness/tools.py` — I can now confirm:
   - `Deps` dataclass: has `client`, `user_question=""`, `last_envelope=None`, 
`catalog=None`, `repair_attempts=0`, `resolved_school_ids: list[int] = 
field(default_factory=list)`, `searched: bool = False`, `tool_calls`, 
`output_validation_failures`, `last_messages`, `intent`, `answer_view`. 
   
   Wait — the Deps in the newly-added tools.py does NOT have `intent` and 
`answer_view` fields! Let me re-check... The new version of tools.py shows:

```python
@dataclass
class Deps:
    """Per-run state. Mutable by design: ..."""

    client: EvidenceClient
    user_question: str = ""
    last_envelope: dict[str, Any] | None = None
    catalog: dict[str, Any] | None = None
    repair_attempts: int = 0
    resolved_school_ids: list[int] = field(default_factory=list)
    searched: bool = False
    tool_calls: list[ToolInvocation] = field(default_factory=list)
    # Exact output-gate objections, retained even when PydanticAI ultimately
    # raises only "Exceeded maximum output retries". This is diagnostic state,
    # not model-authored evidence and not part of the rendered answer.
    output_validation_failures: list[list[str]] = field(default_factory=list)
    # The finished run's message history, written back by `ask()`. This is the
    # whole session store the POC needs (architecture §7): a caller answering a
    # ClarificationNeeded re-invokes with this list plus the user's reply, and
    # PydanticAI carries the state. Empty until a run completes.
    last_messages: list[Any] = field(default_factory=list)
    # Pipeline stage outputs (set by the 4-stage pipeline in harness/agent.py).
    # None when using the legacy tool-loop path (build_agent + ask with agent 
arg).
    intent: Any = None       # StructuredIntent | None — the interpreted intent
    answer_view: Any = None  # AnswerView | None — the built answer view
```

Good — `intent` and `answer_view` are there. And `searched`, 
`resolved_school_ids` are there. 

   - `ToolInvocation(tool, arguments, result=None, error_kind=None)` — yes, 
constructor signature is `(tool: str, arguments: dict, result: Any = None, 
error_kind: str | None = None)`. My planner calls 
`ToolInvocation("search_schools", arguments, error_kind=type(exc).__name__)` and
`ToolInvocation("search_schools", arguments, records)` — both fine.
   - `request_scope_problem(user_question, question_id, request)`: returns None 
if question_id not in {"VC-POC-01", "VC-POC-04"}; if metrics not a list → None; 
checks `_HEADCOUNT_WORDING.search(user_question)` and 
`_NON_TEACHING_WORDING.search(user_question)`. For an empty user_question, both 
regex searches fail to match, so any headcount/non-teaching metric would be 
flagged as "extra". Hmm — in my `_route_selectors`, when the scope problem 
fires, I clean the metrics list by dropping headcount/non-teaching metrics. That
matches the previous planner behaviour. But wait: the scope rule drops 
`_headcount` and `non_teaching_` prefixed metrics. In my implementation, if the 
user's raw question contains "headcount" wording, extras won't include 
headcounts. Fine.

   One consideration: my `_route_selectors` calls 
`request_scope_problem(deps.user_question, question_id, {**request, "metrics": 
canonical})`. For question VC-POC-01 with user_question empty, if 
requested_metrics contains only "teaching_fte", extras = [] → no problem. If it 
contains "teaching_staff_headcount" and user question doesn't say headcount → 
cleaned. OK.

   BUT — there's a subtlety: `request_scope_problem` returns a problem string; I
ignore the message and clean. That's the same as the old planner. Fine.

2. `harness/contract.py` — confirms:
   - `registry()` returns the raw registry JSON dict → 
`registry()["metric_lists"]` works. ✓
   - `questions()` returns dict keyed by question_id. ✓
   - `material_inputs_for()` exists but I don't use it (I derive from 
`input_policy_for` directly via entry.get("input_policy")). Actually I use 
`entry.get("input_policy")` from `contract.questions()`. ✓
   - `CONTRACT_DIR = settings.PROJECT_ROOT / "contract"` — my planner defines 
its own `CONTRACT_DIR = Path(__file__).resolve().parents[1] / "contract"`. 
That's consistent for repo layout (harness/planner.py → parents[1] is repo 
root). Could instead reuse `contract.CONTRACT_DIR` — that would be better since 
it uses settings.PROJECT_ROOT. Minor; either works. Maybe I should switch to 
`contract.CONTRACT_DIR` for consistency. Actually `settings.PROJECT_ROOT` is 
presumably the repo root; both resolve to the same directory. Using 
`contract.CONTRACT_DIR` is cleaner and avoids duplicating path logic. I'll 
consider updating planner to import `contract.CONTRACT_DIR`.

   Hmm, but the contract module is `harness/contract.py` and my planner imports 
`from harness import contract`. So I can do `contract.CONTRACT_DIR`. That's 
cleaner. Should I re-issue planner.py with that change? It's a minor cleanup. 
The instructions say the user added files and asks "Let me know if there are 
others we should add." They also repeat the file-listing instructions. So they 
may expect me to either confirm no changes or produce updated files.

3. `evals/fixtures/stub_envelopes.json` — I now have the full fixture bundle. 
Let me verify my planner's dispatched requests match stub cases:

   - `VC-POC-01/canonical` match: `{"acara_sml_id": 40387, "metrics": 
["total_enrolments", "teaching_fte"]}`. My planner for the pipeline path test 
`test_normalized_schema_valid_request_reaches_evidence` sends exactly 
`{"acara_sml_id": 40387, "metrics": ["total_enrolments", "teaching_fte"]}` → 
matches (subset matching). ✓
   
   - `VC-POC-02/canonical` match: `{"acara_sml_id": 40387, "grades": 
["year_6"]}`. My planner test sends `{"acara_sml_id": 40387, "grades": 
["year_6"]}` ✓.
   
   - `VC-POC-03/canonical` match: `{"acara_sml_ids": [40387, 40392], "ratios": 
["students_per_teaching_fte"]}`. My planner test 
`test_vc_poc_03_routes_selector_to_ratios` sends `{"acara_sml_ids": [40387, 
40392], "ratios": ["students_per_teaching_fte"]}` ✓.
   
   But wait — in `test_mention_order_preserved_and_duplicates_removed` 
(VC-POC-03), I use `_DirectoryClient` whose fetch returns a canned envelope 
without error, so no stub involvement. ✓

   - `VC-POC-05/canonical` match: `{"acara_sml_id": 40387, "market_definition": 
"sa2"}` ✓ — my planner sends exactly that after normalization. ✓

   - `VC-POC-08/canonical` match: `{"acara_sml_id": 40387, "distance_km": 10}` —
the intent boundary test 
`test_planner_clarification_bypasses_narration_and_answer_view` clarifies for 
missing distance_km. ✓ And in test_run_cli, the legacy path clarifies then 
fetches `{"acara_sml_id": 40387, "distance_km": 10}` → matches ✓.

   - `VC-POC-09/canonical` match: `{"peer_filters": {"school_types": 
["Secondary"], "enrolments_min": 500}}`. Hmm — my planner test 
`test_explicit_peer_filter_dispatches` uses `_DirectoryClient`, not stub. But in
a live pipeline with the stub: if a user asks for peers with sectors 
Catholic+Independent + school_types Secondary, the request `{"peer_filters": 
{"sectors": ["Catholic", "Independent"], "school_types": ["Secondary"]}}` would 
NOT match any stub case (the canonical case matches only `school_types` and 
`enrolments_min`, and subset matching means request must contain every match key
with equal values — request has `sectors` extra, but `enrolments_min` missing → 
no match → NoFixture → exit 3). That's a stub-bundle concern for live use, not 
for my tests. My tests don't dispatch that against the stub. OK.

   Actually wait — one of my planner tests: 
`test_peer_filters_require_explicit_nondefault_filter` uses `_DirectoryClient`. 
Fine.

   - School directory preflight: records are `FIXTURE Alpha College`, `FIXTURE 
Beta Grammar`, etc. The stub's `run_school_directory` filters records by `needle
in r["school_name"].casefold()`. So searching "Alpha College" matches "FIXTURE 
Alpha College" — exactly one record. ✓ Searching "Beta Grammar" → exactly one. ✓
Good — identity resolution against the stub works for single-match names.

   But careful: searching "College" would match "FIXTURE Alpha College" AND 
"FIXTURE Gamma Secondary College" AND "FIXTURE Delta Learning Centre" AND 
"FIXTURE Epsilon Learning Academy" → 4 matches → ambiguous. My tests never 
search such terms through the stub (planner tests use `_DirectoryClient` with 
explicit dicts; intent-boundary tests use stub with school_mentions=[] mostly). 
Let me check `test_ask_pipeline_intent_override_bypasses_model`: intent has 
school_mentions=[] and filters={"acara_sml_id": 40387}. No preflight. ✓

4. Now let me re-verify the schemas against my validator with the full schema 
files I have. I have VC-POC-01, VC-POC-03, VC-POC-04 request schemas in full. I 
previously requested VC-POC-09/10/08 and the rest, but the user did NOT add them
(only stub_envelopes.json, contract.py, tools.py). I don't have the actual 
VC-POC-05..14 schemas (except via _index.json metadata). So I still can't 100% 
verify validator coverage across all schemas. However:
   - VC-POC-04 schema uses `uniqueItems`, `minItems`, `maxItems`, `minimum`, 
enum, additionalProperties — all supported.
   - VC-POC-03 similar.
   - VC-POC-09/10 have `minProperties: 1` on nested filter objects per OD-07 
("the request schemas express only minProperties >= 1"). My validator supports 
minProperties. ✓ But my planner test `test_empty_segment_filters_clarify` 
returns clarification before validation anyway (the explicit-filter constraint 
runs before policy derivation? Let me check order in my planner: Step 8 enforces
explicit filter constraints, Step 9 derives missing inputs, Step 10 validates. 
For `segment_filters: {}`: the constraint check sees isinstance dict and empty →
clarification. ✓ (Even though schema would also fail with minProperties=1, we 
clarify first per contract: "absent, empty, or sector-default-only definitions 
clarify".) ✓

Now, are there any issues I should fix in the editable files given the newly 
supplied content?

**Issue A: `Deps.answer_view` default and 
`test_planner_clarification_bypasses_narration_and_answer_view`.** Deps has 
`answer_view: Any = None` ✓. And `intent: Any = None` ✓. My intent-boundary 
tests assert `deps.answer_view is None` after clarification — fine.

**Issue B: `request_scope_problem` with my exact-request tests.** In 
`test_metric_normalization_and_exact_profile_request`, user_question is "q" 
(empty-ish). Requested metrics: ["Total Enrolments", "Teaching Staff", "teaching
staff", "TOTAL_ENROLMENTS"] → canonical ["total_enrolments", "teaching_fte"] 
(dedup). Scope check: no headcount or non-teaching metrics → extras empty → no 
problem. Request sent: `{"acara_sml_id": 40387, "metrics": ["total_enrolments", 
"teaching_fte"]}`. ✓

**Issue C: `_HappyClient.school_directory` returns `[{"acara_sml_id": 40387, 
"school_name": search}]`** — exactly one record → unique resolution ✓ for 
`test_school_resolution_records_search`.

**Issue D: In my planner, `_normalize_scalar_selectors` handles 
market_definition against `_static_vocabulary_for_field(question_id, 
"market_definition")`.** The catalog mapping has "VC-POC-05.market_definition": 
"market_definitions", "VC-POC-06...", "VC-POC-07...". ✓ static enum from 
input-catalog schema properties.market_definitions.items.enum = 
["distance","lga","postcode","sa2"] ✓. My test 
`test_missing_market_definition_is_derived_without_model_hint` asserts 
`allowed_values == ["distance", "lga", "postcode", "sa2"]` — the enum order in 
the schema is exactly that. ✓

**Issue E: VC-POC-07 `grades` static vocabulary** — mapping "VC-POC-07.grades": 
"grades" → enum sorted list: ["pre_year_1_one_year", ..., "year_9"]. My 
`_canonical_for("Year 6", vocab)` → "year_6" ✓ (normalize "Year 6" → "year 6"; 
"year_6" → "year 6" ✓).

**Issue F: VC-POC-12 `sa2_codes` is release-derived** → `_static_enum` returns 
None for "release_derived_values.*" ✓; mapping for VC-POC-12.seifa_index_type → 
"seifa_index_types" static ✓. My planner only normalizes `market_definition` 
scalar and label-bag selectors; sa2_codes remain shape-checked. ✓

**Issue G: Contract drift check in `_derive_missing_inputs`** — for 
VC-POC-05/06/07 the pinned condition string in the registry is 
"market_definition equals distance" (registry) or "market_definition == 
distance" (_index.json x-poc? Actually _index.json conditional_material is {} — 
no wait, _index.json doesn't carry conditional_material; the registry does: 
"market_definition equals distance"). My `_KNOWN_DISTANCE_CONDITIONS` includes 
normalized forms of both "market definition equals distance" and "market 
definition distance" — the latter covers "market_definition == distance"? Let me
check: "== distance" → separators removed: `_SEPARATOR_RE = [\W_]+` — "=" is \W,
so "market_definition == distance" → "market definition distance". ✓ Good, both 
forms covered.

**Issue H: planner CON stance on `contract.CONTRACT_DIR`.** Minor cleanup 
opportunity. I'll update planner.py to use `contract.CONTRACT_DIR` instead of 
recomputing. Actually — is that important? Both work. But since contract.py is 
confirmed available and its CONTRACT_DIR derives from settings.PROJECT_ROOT, 
reusing it is more robust (e.g., if tests run from elsewhere). 
`Path(__file__).resolve().parents[1] / "contract"` equals repo-root/contract as 
long as planner.py lives in harness/. Both fine. I'll make the small change for 
consistency and to reduce duplication.

**Issue I: One real bug check — `_route_selectors` for VC-POC-11.** Registry 
metric_list "student_context_metrics" → routed field "metrics". But user's 
requested label e.g. "indigenous enrolments" → "indigenous_enrolments" ∈ 
student_context_metrics ✓. But what if a user asks VC-POC-11 for "total 
enrolments"? Not in student_context_metrics → clarification with that 
vocabulary. That's per the static-routing rule ✓.

**Issue J: VC-POC-14 has metric_list null and mapping entries... "VC-POC-14" 
doesn't appear in the catalog mapping. `gated` questions return Unsupported 
before planning anyway. ✓

**Issue K: VC-POC-REF-01** — no metric_list, no mapping key (mapping has no 
REF-01 keys) → `_selector_route` returns None. If user requests metrics for 
REF-01 → clarification ("takes no metric selector"). Hmm — is that right? 
VC-POC-REF-01 request schema is `{}` (probably allows empty request). Requesting
labels can't be applied → clarification seems right per "requested selectors 
that cannot be normalized" → ClarificationNeeded. OK.

**Issue L: `_explicit_id_values` for VC-POC-11 with acara_sml_ids in filters** —
identity_from_caller=True; no preflight; explicit IDs passed through. 
`test_explicit_numeric_ids_skip_directory_preflight` ✓.

**Issue M: potential ordering issue in 
`test_missing_market_definition_is_derived_without_model_hint`.** VC-POC-05 
required: ["acara_sml_id", "market_definition"]. With no mentions and no 
filters, request={} → missing = ["acara_sml_id", "market_definition"] in that 
order ✓. allowed_values for acara_sml_id: 
`_static_vocabulary_for_field("VC-POC-05", "acara_sml_id")` → mapping 
"VC-POC-05.acara_sml_id"? The mapping does not include acara_sml_id → None → [] 
✓. And market_definition allowed_values from mapping ✓. assert 
`result.missing[1].allowed_values == ["distance","lga","postcode","sa2"]` ✓.

**Issue N: `test_distance_is_conditional_on_distance_market`** — VC-POC-05 with 
filters {"acara_sml_id": 40387, "market_definition": "sa2"}. `_DirectoryClient` 
fetch returns canned envelope ✓. Request sent: after scalar normalization (sa2 
canonical) → {"acara_sml_id": 40387, "market_definition": "sa2"} ✓. schema 
VC-POC-05 validates (I don't have the schema file, but properties acara_sml_id 
integer, market_definition enum — my validator handles). One risk: the VC-POC-05
schema may use `if/then` conditional requiring distance_km when 
market_definition == distance — supported by my validator anyway; with sa2 it 
doesn't apply. ✓

For the distance case: missing derivation flags distance_km → clarification 
before schema validation ✓. Good — because schema (if it encodes the 
conditional) would fail without distance_km; we clarify first. ✓ Correct per 
contract ("There is no honest default radius" → clarify).

**Issue O: `test_clarifications_follow_pinned_policy_order`** — VC-POC-07 
required: ["acara_sml_id","market_definition","grades"]; mention resolves 40387 
→ attached. market_definition="distance" supplied. Missing: grades (required, 
absent) then distance_km (conditional, absent). Order: required loop adds grades
(acara_sml_id and market_definition present), then conditional adds distance_km 
→ names == ["grades", "distance_km"] ✓.

**Issue P: `test_consistent_explicit_id_and_mention_proceeds`** — VC-POC-01, 
explicit acara_sml_id 40387 + mention resolves 40387 → sets match → proceeds; 
fetched request `{"acara_sml_id": 40387}` ✓. Note: `identity_from_caller=True`, 
we set deps.searched=True but don't append resolved id to 
deps.resolved_school_ids (explicit ids weren't committed). Hmm — should the 
consistent-case commit resolved ids to deps? In the plan: "Resolve each mention 
through the directory, require exactly one result, stage IDs locally in mention 
order, deduplicate without reordering, and commit to Deps only after all 
mentions succeed." For the conflict path we raise TerminalError; for consistent 
path... committing the resolved ID to deps.resolved_school_ids seems desirable 
for `_learned()` diagnostics. But careful: if explicit ID present, committing 
resolved==explicit duplicates semantics? deps.resolved_school_ids would then 
contain 40387. My test asserts `deps.searched is True` and fetched request; it 
doesn't assert resolved_school_ids. Hmm, but should I commit? The 
failure_behaviour says "Raise TerminalError before evidence dispatch and retain 
no partial identity update" for conflicts. For the consistent case there's no 
partial update concern. I currently don't commit in the identity_from_caller 
branch. Let me reconsider: for consistency and observability, committing 
resolved ids (which equal explicit ids) to deps.resolved_school_ids would be 
fine. But there's a subtle case: explicit caller-supplied IDs WITHOUT mentions —
no preflight — nothing to commit. With mentions that match explicitly — commit? 
It's harmless. I could add it. But changing behavior now would require 
re-verifying tests... My current code: in the `if identity_from_caller:` branch,
after verifying match, sets deps.searched=True only. I think committing is more 
correct per the acceptance semantic "Every directory preflight and evidence 
invocation ... remains observable in deps.tool_calls" (they are). 
resolved_school_ids observability isn't explicitly required for the 
caller-supplied path. I'll leave as-is — matches "caller-supplied numeric IDs 
bypass the preflight" framing. Actually wait, let me reconsider once more: if 
mention resolution succeeded and matches, the ID is a resolved identity; putting
it in deps.resolved_school_ids aligns with `_learned`. But 
`test_schema_valid_emitter_bad_request_is_terminal` asserts 
`deps.resolved_school_ids == [40387]` — that's the non-caller path (mention 
only, no filters) → committed ✓.

For the consistent path I'll keep current behavior (not committing) — simpler, 
and no test contradicts it. Hmm, actually let me double check my own test 
`test_consistent_explicit_id_and_mention_proceeds` — it doesn't assert 
resolved_school_ids. Fine.

**Issue Q: In `plan_and_execute`, for `school_required` with mentions and no 
caller identity — VC-POC-04 insufficient check.** `_requires_two_distinct_ids` 
checks constraints contain "at least two" — registry VC-POC-04 constraint: 
"acara_sml_ids contains at least two values" ✓ contains "at least two" ✓.

**Issue R: `test_comparison_with_one_distinct_school_clarifies`** — two mentions
both resolving 40387 → staged dedups to [40387], len<2 → clarification; 
resolved_school_ids not committed ✓ (assert [] ✓).

**Issue S: stub emitter directory `limit` param** — 
EvidenceClient.school_directory passes limit; my planner uses 25 default ✓.

**Issue T: The `matches()` in stub and `VC-POC-01/canonical`** requires metrics 
["total_enrolments","teaching_fte"] exactly for the match. But 
`test_normalized_schema_valid_request_reaches_evidence` uses 
`_RecordingClient(stub_client())` — request has exactly those metrics ✓ so the 
canonical case matches and the answer narration "Enrolments run 612 in 2019 and 
668 in 2023." grounds (612, 668, 2019, 2023 all in envelope) ✓.

Also `test_ask_pipeline_intent_override_bypasses_model` uses stub canonical ✓ 
(intent filters acara_sml_id 40387, metrics requested 
["total_enrolments","teaching_fte"] → same canonical request ✓).

`test_rendered_answer_always_uses_deterministic_data_status_note`: 
filters={"acara_sml_id": 99999}, requested_metrics=[] → request {"acara_sml_id":
99999} → stub: "VC-POC-01/empty-series" forced via STUB_EMITTER_CASE ✓ (match 
bypass). Response envelope question_id VC-POC-01 ✓. Wait — planner: 
identity_from_caller=True (acara_sml_id in filters). Schema-valid ✓. Fetch → 
envelope (empty series, success) → RenderedAnswer ✓. Good, no bad_request 
involved since STUB_EMITTER_CASE forces the case.

`test_ask_pipeline_intent_override_excluded_topic_returns_unsupported` etc. — 
planner branches before any client use ✓.

`test_ask_pipeline_missing_material_input_returns_clarification` — VC-POC-05, no
filters → derived missing acara_sml_id + market_definition → ClarificationNeeded
✓ (contains market_definition ✓).

**Issue U: `test_run_cli.py::answered()`** uses legacy path with build_agent + 
replay scripted model — unchanged, still imports build_agent in the TEST (fine; 
the test asserts run.py source has no "build_agent", which is about run.py 
only). Wait — my new test `test_main_no_longer_builds_a_legacy_agent` asserts 
`"build_agent" not in source` of run.py. My updated run.py does not import 
build_agent ✓. But careful: the docstring mentions "legacy agent" — no 
"build_agent" string. Let me scan my run.py: comments say "no legacy agent is 
built or passed" — no "build_agent" substring ✓. Also `from harness.agent import
ask  # noqa: E402` ✓.

**Issue V: `test_main_uses_pipeline_path_by_default`** — monkeypatches 
cli.converse with `fake_converse(agent, deps, question, **kwargs)`; run.py main 
calls `converse(None, deps, args.question, ask_user=..., 
max_clarifications=...)` → fake receives positional None ✓. Returns Unsupported 
→ EXIT_BY_BRANCH[Unsupported]=3 ✓. Note: fake_converse signature has no 
ask_user/max_clarifications issue since **kwargs ✓.

But one catch: `cli.main` calls `configure_logging()` and 
`build_parser().parse_args(argv)` with argv ["--stub", "..."] ✓. 
`require_credentials` monkeypatched ✓. `build_client` monkeypatched ✓. 
Deps(client=stub_client()) real ✓. asyncio.run(fake_converse(...)) returns 
Unsupported ✓. exit 3 ✓.

**Issue W: legacy converse tests in test_run_cli** call `cli.converse(agent, 
deps, ...)` with agent → legacy ask path ✓ unchanged behavior.

**Issue X: tools.py `request_scope_problem` — my `_route_selectors` passes 
`{**request, "metrics": canonical}`.** For VC-POC-04 requested_metrics 
["total_enrolments"] etc. fine.

Edge: VC-POC-11 — request_scope_problem returns None for non-01/04 questions ✓.

**Issue Y: One more check — `harness/contract.py` `registry()` returns raw dict;
my `_selector_route` uses `contract.registry().get("metric_lists", {})`. 
registry JSON has "metric_lists" ✓.

**Issue Z: `_schema_index()` reads 
`contract/question_schemas/v0.3.0/_index.json`** — exists ✓ (pinned list 
includes it). Path check: `_SCHEMA_DIR / "_index.json"` — fine.

`_registered_request_schema_path` consistency check: filename must equal 
f"{question_id}.request.schema.json" — index entries are exactly 
"VC-POC-01.request.schema.json" etc. ✓ (verified in _index.json: 
"request_schema": "VC-POC-01.request.schema.json" ✓).

Path containment: `(_SCHEMA_DIR / filename).resolve()`; parents include 
CONTRACT_DIR.resolve() ✓.

Now — remaining files worth adding? The user asked "Let me know if there are 
others we should add."

Remaining useful ones:
- The remaining pinned request schemas (VC-POC-05..14, REF-01/02) to fully 
verify validator coverage. Still not provided. I should mention them again but 
not block.
- `harness/intent.py` (StructuredIntent fields) — I rely on field names: 
question_id, excluded_topic, school_mentions, requested_metrics, filters, 
presentation, missing_material_inputs. The existing tests already used exactly 
these, so they're confirmed by the pre-existing tests. Not strictly needed.
- `harness/outputs.py` — 
ClarificationNeeded/MissingInput/Unsupported/RenderedAnswer field names; 
existing tests confirm (MissingInput(input=, why_material=, allowed_values=), 
Unsupported(reason_kind, reason, reason_code?, nearest_supported)). Hmm — does 
`Unsupported` accept `reason_code=`? The old planner already passed 
reason_code=reason_code to Unsupported, and tests/test_planner old version 
asserted result.reason_code — so yes.
- `config/settings.py` — PROJECT_ROOT, CONTRACT_VERSION used by contract.py; not
needed for my edits.
- `harness/answer_view.py`, `harness/narration.py` — pipeline stages; my changes
don't touch them; existing tests cover.

So: should I make any edits now? The user message says "I added these files... 
Let me know if there are others we should add." and repeats the file-listing 
rules. The natural response: verify compatibility, confirm what checks out, 
optionally make the small planner cleanup (use contract.CONTRACT_DIR), and list 
remaining files to add.

Let me carefully verify against tools.py one more potential problem: 
**`ToolInvocation` in planner records for school preflights: 
`ToolInvocation("search_schools", arguments, records)` — matches the legacy 
tool's recording style ✓.

And **`deps.tool_calls.append` on preflight exception: 
`error_kind=type(exc).__name__`** — legacy uses similar? Legacy search_schools 
records error_kind="bad_request" for empty terms; on client failure it doesn't 
record (asyncio.gather raises out)... actually legacy raises without recording 
on client exception. My planner records the exception type name — that's an 
improvement, consistent with "Record every preflight". Fine.

Now the one genuine adjustment I want: planner should use 
`contract.CONTRACT_DIR` rather than recomputing the path, since contract.py is 
confirmed to define it from settings.PROJECT_ROOT. Also double-check: 
`contract.py` computes `CONTRACT_DIR = settings.PROJECT_ROOT / "contract"`. 
planner importing `from harness import contract` already; so `_SCHEMA_DIR = 
contract.CONTRACT_DIR / "question_schemas" / "v0.3.0"`. This removes a duplicate
constant. Minor but cleaner.

Also, I realize there's a subtle issue with `request_scope_problem` being 
applied inside `_route_selectors` for VC-POC-04: fine.

Another potential issue: **`_normalize_scalar_selectors` only handles 
market_definition.** What about `zone_entity_code` (closed format, 
pattern-enforced in schema)? My validator handles pattern. If caller supplies an
invalid zone_entity_code → schema validation fails → TerminalError. Hmm — is a 
user-supplied malformed zone entity code a planner defect or a caller problem? 
zone_entity_code is a "closed-format" field. The spec says closed-format 
violations are bad_request (repairable). In the deterministic planner, a 
caller-supplied well-typed but non-matching zone code should be... The plan's 
origin rules: "A user selector that cannot be normalized or a caller-supplied 
identity requiring correction is a clarification, not Unsupported. A well-typed 
release-derived/open non-match is a successful empty/partial result." 
zone_entity_code: invalid format → bad_request from emitter; 
identity_from_caller would be False (not a school field) → my planner raises 
TerminalError. Hmm. Is that right? The golden case "answer/VC-POC-REF-02" passes
"1000101" — valid pattern presumably. A malformed one from the model → the 
planner built it from intent.filters... The planner doesn't validate 
closed-format fields other than through the schema; failing closed with 
TerminalError for a caller-supplied malformed closed-format value could be 
considered harsh, but the plan says: "After a request has passed deterministic 
schema and planner constraints, an emitter bad_request indicates 
planner/contract drift and raises TerminalError" and pre-dispatch: "Raise 
TerminalError before dispatch for ... any other planner-produced request that 
cannot satisfy its registered schema." A malformed zone_entity_code from the 
model's filters IS a planner-produced request that can't satisfy the schema → 
TerminalError is per-plan ("unknown fields introduced into the assembled 
request, and any other planner-produced request that cannot satisfy its 
registered schema"). So TerminalError is correct per plan. Good, keep.

OK so the only change I'll make: planner.py using contract.CONTRACT_DIR. Is it 
worth re-outputting the whole file? The user explicitly reminded about file 
listings, implying they expect possible edits. A tiny cleanup is low-risk and 
beneficial. I'll re-output harness/planner.py with that one change (and while at
it, re-verify the file content carefully).

Actually, wait. Let me also re-examine one thing in my planner: 
`_explicit_id_values` for the `acara_sml_id` (singular) case — the code:

```python
if "acara_sml_id" in school_required:
    value = request.get("acara_sml_id")
    if isinstance(value, int) and not isinstance(value, bool):
        return [value]
return []
```

Note: the `return []` is inside the elif-ish block? Let me re-read my code:

```python
def _explicit_id_values(request: dict[str, Any], school_required: list[str]) -> 
list[int]:
    if "acara_sml_ids" in school_required:
        value = request.get("acara_sml_ids")
        if isinstance(value, list):
            return [v for v in value if isinstance(v, int) and not isinstance(v,
bool)]
        return []
    if "acara_sml_id" in school_required:
        value = request.get("acara_sml_id")
        if isinstance(value, int) and not isinstance(value, bool):
            return [value]
    return []
```

Yes that's what I wrote. Fine.

One more check in `_resolve_mentions`: `ids` extraction requires isinstance int 
and not bool ✓.

In the conflict check: `sorted(resolved_ids) != sorted(explicit_ids)` — for 
VC-POC-01 with one mention one explicit, ✓.

Potential duplicate mention with explicit id: mentions=["Alpha","Alpha"] both 
resolve 40387 → staged [40387]; explicit [40387]; match ✓ proceeds.

Now — `test_school_resolution_records_search` uses `_HappyClient` whose 
school_directory returns ONE record ✓ unique.

OK. Also verify `harness/tools.py` imports in planner: `from harness.tools 
import Deps, ToolInvocation, request_scope_problem` ✓ all exist.

Let me also double check the stub's `school_directory` client method signature 
in harness/evidence.py: `async def school_directory(self, search: str | None = 
None, limit: int = 25)`. My `_resolve_mentions` calls 
`client.school_directory(search=mention, limit=25)` ✓. My `_DirectoryClient` 
mock signature `school_directory(self, search: str, limit: int = 25)` ✓ matches 
call with keyword `search=`. ✓

And `_RecordingClient` in test_intent_boundary: `async def 
school_directory(self, search=None, limit=25)` ✓.

`_RejectingIdentityClient` in test_intent_boundary: has school_directory, 
input_catalog, fetch_evidence ✓ (planner only calls school_directory + 
fetch_evidence).

Alright. Now let me also double-check `contract.py` `questions()` — keyed dict 
of registry questions; `input_policy_for` not needed. `refusal_for` ✓, 
`excluded_questions()` ✓, `reason_code_for` ✓, `exclusion_texts()` ✓. My planner
uses contract.questions(), contract.excluded_questions(), 
contract.refusal_for(), contract.reason_code_for(), contract.exclusion_texts(), 
contract.registry(). All exist ✓.

DECISION: Re-issue harness/planner.py with the CONTRACT_DIR cleanup. Everything 
else verified — no changes needed to run.py or the three test files based on the
newly supplied files.

Also answer "others we should add": the remaining 13 pinned per-question request
schemas (VC-POC-02/05/06/07/08/09/10/11/12/13/14, VC-POC-REF-01/02) to verify 
the validator against every registered schema, and optionally harness/intent.py 
+ harness/outputs.py to confirm field names (though existing passing tests 
already pin them). And config/settings.py is not needed.

Let me now produce the updated planner.py. I need to reproduce the ENTIRE file 
with only the path-constant change. Let me carefully write it out, keeping 
everything else identical to what I output before.

Change:
```python
from harness import contract
...
CONTRACT_DIR = Path(__file__).resolve().parents[1] / "contract"
_SCHEMA_DIR = CONTRACT_DIR / "question_schemas" / "v0.3.0"
```
becomes
```python
from harness import contract
...
# Pinned contract artefacts. Static vocabularies and per-question schemas are
# read from the vendored, hash-pinned files under the same directory
# `harness.contract` reads — never from the runtime emitter.
_SCHEMA_DIR = contract.CONTRACT_DIR / "question_schemas" / "v0.3.0"
```

Everything else unchanged. Let me write the full file again carefully, exactly 
as before except this.

Also I should keep the docstring mention. Fine.

Then summarize: verified tools.py (Deps fields, ToolInvocation signature, 
request_scope_problem), contract.py 
(registry/questions/refusal_for/exclusion_texts — all used correctly; 
registry()["metric_lists"] available), stub_envelopes.json (directory resolves 
uniquely for "Alpha College"/"Beta Grammar" style searches; canonical match keys
align with the exact requests the planner dispatches; forced-case empty-series 
flow works). Files still worth adding: the remaining pinned request schemas, 
harness/intent.py and harness/outputs.py (low priority).

I will produce the full planner.py listing plus the summary. Keep response 
reasonably concise apart from the file listing (which must be complete).

------------
► **ANSWER**

All three files check out against the implementation — **only one small cleanup 
is needed in `harness/planner.py`**; `run.py` and the three test files need no 
changes. Here's what the new files confirm:



**`harness/tools.py`** — all assumptions hold: `Deps` carries `searched`, 
`resolved_school_ids`, `tool_calls`, `intent: Any = None` and `answer_view: Any 
= None` (so the new boundary tests' assertions are valid); `ToolInvocation(tool,
arguments, result=None, error_kind=None)` matches my constructor calls; 
`request_scope_problem()` returns `None` for non-VC-POC-01/04 questions and 
flags headcount/non-teaching metrics exactly as my scope cleanup expects, with 
an empty `user_question` behaving as a no-op flag that the cleanup then strips —
same semantics as the old planner.



**`harness/contract.py`** — `registry()` returns the raw registry JSON (so 
`registry()["metric_lists"]` is available for `metric_list`-routed 
vocabularies), and `questions()`, `refusal_for()`, `excluded_questions()`, 
`reason_code_for()`, `exclusion_texts()` all match my usage. It also defines 
`CONTRACT_DIR` from `settings.PROJECT_ROOT` — my planner recomputed the same 
path itself, which is needless duplication, so I'm switching it to 
`contract.CONTRACT_DIR` (the one edit below).



**`evals/fixtures/stub_envelopes.json`** — compatible with the exact requests 
the planner now dispatches:

- The directory preflight resolves **exactly one record** for "Alpha College", 
"Beta Grammar", etc. (the `FIXTURE ` prefix doesn't break the casefold substring
search), so exactly-one-match identity resolution works against the stub.

- The canonical `match` keys align: `VC-POC-01/canonical` (`acara_sml_id` 40387 
+ both metrics), `VC-POC-02/canonical` (`grades: ["year_6"]`), 
`VC-POC-03/canonical` (`acara_sml_ids` + `ratios`), `VC-POC-05/06` (`sa2`), 
`VC-POC-08` (`distance_km: 10`) — all subset-matched by the planner's smallest 
requests.

- The `VC-POC-01/empty-series` flow works: caller-supplied 99999 skips the 
preflight and the forced stub case returns a success envelope, exercising the 
empty-render branch without touching the new `bad_request`-origin split.



**Others still worth adding** (not blockers — the validator under-checks rather 
than mis-rejects, and existing passing tests already pin the dataclass field 
names): the remaining pinned request schemas 
(`contract/question_schemas/v0.3.0/VC-POC-05/06/07/08/09/10/11/12/13/14/REF-01/R
EF-02.request.schema.json`) so I can verify validator coverage against every 
registered schema, and `harness/intent.py` / `harness/outputs.py` if you want 
belt-and-braces confirmation of the `StructuredIntent`/output field names.



harness/planner.py

```diff
@@ -1,38 +1,66 @@
 """Stage 2 of the 4-stage pipeline: deterministic planning and execution.
 
 `plan_and_execute()` takes a StructuredIntent and returns either:
 - a dict (the evidence envelope) — the happy path
-- ClarificationNeeded — a material input is missing
+- ClarificationNeeded — an input the caller must supply or correct
 - Unsupported — an exclusion or an unknown question
 
-No model calls. Every branch is a pure function of the intent and the contract.
-The evidence call itself is the only I/O, and its error handling mirrors the
-tool layer's `_branch_on_error_kind()` exactly: the same closed set of 
outcomes,
-the same reason codes, the same contract text.
-
-Why reproduce `_branch_on_error_kind` here rather than calling it? Because the
-tool layer raises `ModelRetry`, `UnsupportedSignal`, and `RepairCapExceeded` —
-exceptions designed for PydanticAI's agent loop. The pipeline has no loop, so
-those signals have no handler; they would escape to the caller as unclassified
-RuntimeErrors. The planner translates the same error semantics into the 
pipeline's
-typed return values instead.
+No model calls. Every branch is a pure function of the intent, the pinned
+contract artefacts, and the evidence client's responses.
+
+What Stage 2 now guarantees (work item 20260908T113118):
+
+- Required, material, and applicable conditional-material inputs are derived
+  from the pinned question policy and the assembled request. The intent
+  model's `missing_material_inputs` claim is never trusted.
+- Named schools resolve through the pinned directory and must match exactly
+  one record each. IDs are staged in mention order, deduplicated without
+  reordering, and committed to `Deps` only after every mention resolves; a
+  zero or ambiguous match clarifies with no evidence call and no partial
+  identity mutation. Caller-supplied numeric IDs bypass the preflight (the
+  bounded resolution of the pinned Definition-Pack conflict) and any conflict
+  with name-resolved identities fails closed before dispatch.
+- Selector labels are Unicode-case-folded and separator-normalized, matched
+  against pinned static vocabularies (registry `metric_list` where declared;
+  otherwise the input-catalog schema's field mapping and embedded enums), and
+  routed to the question's own request field: `metrics` for VC-POC-01/04/11,
+  `ratios` for VC-POC-03, `grades` for VC-POC-02/07. Unsupplied selectors are
+  omitted — the emitter keeps default ownership (no invented defaults).
+- Every request is validated against its registered pinned per-question
+  schema before dispatch. A planner-produced request that cannot satisfy its
+  schema raises TerminalError — planner defects are never dressed up as
+  Unsupported user outcomes.
+- Error origins are separated: a bad_request against a request whose school
+  identity was caller-supplied is an identity clarification; any other
+  bad_request after deterministic validation is planner/contract drift and
+  raises TerminalError. excluded/unknown/unanswerable keep their typed
+  abstentions; data_unavailable/internal remain operational stops.
 """
 
 from __future__ import annotations
 
+import json
 import logging
+import re
+from functools import lru_cache
+from pathlib import Path
 from typing import Any
 
 from harness import contract
 from harness.errors import TerminalError
 from harness.evidence import EvidenceClient, EvidenceError
 from harness.intent import StructuredIntent
 from harness.outputs import ClarificationNeeded, MissingInput, Unsupported
 from harness.tools import Deps, ToolInvocation, request_scope_problem
 
 logger = logging.getLogger("harness.planner")
+
+# Pinned contract artefacts. Static vocabularies and per-question schemas are
+# read from the vendored, hash-pinned directory `harness.contract` reads —
+# never from the runtime emitter.
+_SCHEMA_DIR = contract.CONTRACT_DIR / "question_schemas" / "v0.3.0"
 
 # Keyword hints the intent model may use for excluded topics, mapped to a 
search
 # term that appears in the corresponding contract exclusion text. The mapping 
is
 # fuzzy: the planner searches for any exclusion text that contains the keyword.
 _EXCLUSION_KEYWORD_MAP: dict[str, str] = {
@@ -47,10 +75,32 @@
     "mesh_block": "Mesh Block",
     "sa2": "SA2",
     "gtfs": "GTFS",
 }
 
+# The one pinned semantic alias (prompts/rules.md): for school-profile
+# requests "teaching staff" means teaching_fte. Keyed by the normalized label,
+# so "Teaching Staff" and "teaching-staff" match the same alias.
+_PINNED_SELECTOR_ALIASES: dict[str, str] = {"teaching staff": "teaching_fte"}
+
+_SCHOOL_FIELDS = ("acara_sml_id", "acara_sml_ids")
+
+# Label-selector request fields the requested label bag can be routed into.
+_SELECTOR_BAG_FIELDS = ("metrics", "ratios", "grades")
+
+# The exact pinned conditional-material expressions (VC-POC-05/06/07), compared
+# after separator normalization. Anything else is contract drift, not prose to
+# parse heuristically.
+_KNOWN_DISTANCE_CONDITIONS = frozenset(
+    {
+        "market definition equals distance",
+        "market definition distance",
+    }
+)
+
+_SEPARATOR_RE = re.compile(r"[\W_]+", re.UNICODE)
+
 
 def _find_exclusion_text(excluded_topic: str | None) -> str:
     """Find the contract exclusion text matching a topic hint.
 
     Searches the exclusion texts for a case-insensitive substring match on the
@@ -77,23 +127,726 @@
 
     # Fallback: first exclusion text alphabetically
     return texts[0]
 
 
+# --------------------------------------------------------------------------- #
+# Pinned artefact loading (fail-closed: load failures are TerminalError)
+# --------------------------------------------------------------------------- #
+
+
+def _load_pinned_json(path: Path) -> Any:
+    try:
+        return json.loads(path.read_text(encoding="utf-8"))
+    except (OSError, ValueError) as exc:
+        raise TerminalError(
+            f"pinned contract artefact could not be loaded ({path.name}): 
{exc}"
+        ) from exc
+
+
+@lru_cache(maxsize=1)
+def _schema_index() -> dict[str, Any]:
+    index = _load_pinned_json(_SCHEMA_DIR / "_index.json")
+    if not isinstance(index, dict) or not isinstance(index.get("questions"), 
dict):
+        raise TerminalError("pinned question-schema index is malformed")
+    return index
+
+
+@lru_cache(maxsize=1)
+def _input_catalog_schema() -> dict[str, Any]:
+    schema = _load_pinned_json(contract.CONTRACT_DIR / 
"poc_input_catalog.v0.3.0.schema.json")
+    if not isinstance(schema, dict):
+        raise TerminalError("pinned input-catalog schema is malformed")
+    return schema
+
+
+def _registered_request_schema_path(question_id: str) -> Path:
+    """Resolve the question's registered request-schema path.
+
+    Only paths registered in the pinned index, beneath the pinned contract
+    directory, may be loaded — anything else is contract drift.
+    """
+    entry = _schema_index()["questions"].get(question_id)
+    if not isinstance(entry, dict):
+        raise TerminalError(f"{question_id} has no registered request schema")
+    filename = entry.get("request_schema")
+    if not isinstance(filename, str) or filename != 
f"{question_id}.request.schema.json":
+        raise TerminalError(
+            f"{question_id} request-schema registration is inconsistent: 
{filename!r}"
+        )
+    path = (_SCHEMA_DIR / filename).resolve()
+    pinned = contract.CONTRACT_DIR.resolve()
+    if pinned not in path.parents:
+        raise TerminalError(
+            "refusing to load a request schema from outside the pinned contract
"
+            f"directory: {filename}"
+        )
+    return path
+
+
+@lru_cache(maxsize=None)
+def _request_schema(question_id: str) -> dict[str, Any]:
+    schema = _load_pinned_json(_registered_request_schema_path(question_id))
+    if not isinstance(schema, dict):
+        raise TerminalError(f"{question_id} request schema is malformed")
+    return schema
+
+
+# --------------------------------------------------------------------------- #
+# Static vocabularies (registry metric_list; catalog field mapping + enums)
+# --------------------------------------------------------------------------- #
+
+
+@lru_cache(maxsize=1)
+def _field_to_catalog_list() -> dict[str, str]:
+    mapping = 
_input_catalog_schema().get("x-poc-request-field-to-catalog-list")
+    if not isinstance(mapping, dict):
+        return {}
+    return {
+        str(field): str(list_name)
+        for field, list_name in mapping.items()
+        if isinstance(field, str) and isinstance(list_name, str)
+    }
+
+
+def _static_enum(list_name: str) -> list[str] | None:
+    """The embedded static enum for a catalog list, or None when the list is
+    release-derived (shape-checked only, never enum-enforced — OD-03)."""
+    if list_name.startswith("release_derived_values."):
+        return None
+    prop = _input_catalog_schema().get("properties", {}).get(list_name)
+    if isinstance(prop, dict) and isinstance(prop.get("items"), dict):
+        enum = prop["items"].get("enum")
+        if isinstance(enum, list):
+            return [str(value) for value in enum]
+    return None
+
+
+def _static_vocabulary_for_field(question_id: str, field: str) -> list[str] | 
None:
+    list_name = _field_to_catalog_list().get(f"{question_id}.{field}")
+    if list_name is None:
+        return None
+    return _static_enum(list_name)
+
+
+def _selector_route(question_id: str) -> tuple[str, list[str]] | None:
+    """(request field, static vocabulary) for the question's label-selector 
bag.
+
+    The registry's `metric_list` is authoritative where declared (VC-POC-01/04
+    → profile_metrics, VC-POC-11 → student_context_metrics); otherwise the
+    input-catalog schema's field mapping routes e.g. VC-POC-03 to `ratios` and
+    VC-POC-02/07 to `grades`.
+    """
+    entry = contract.questions().get(question_id) or {}
+    metric_list = entry.get("metric_list")
+    if metric_list:
+        values = contract.registry().get("metric_lists", {}).get(metric_list)
+        if not isinstance(values, list) or not values:
+            raise TerminalError(
+                f"{question_id} declares metric_list {metric_list!r} with no 
pinned values"
+            )
+        return "metrics", [str(value) for value in values]
+    field_to_list = _field_to_catalog_list()
+    for key in sorted(field_to_list):
+        prefix = f"{question_id}."
+        if not key.startswith(prefix):
+            continue
+        field = key[len(prefix):]
+        if field in _SELECTOR_BAG_FIELDS:
+            vocabulary = _static_enum(field_to_list[key])
+            if vocabulary is None:
+                raise TerminalError(
+                    f"{question_id}.{field} routes to a non-static vocabulary"
+                )
+            return field, vocabulary
+    return None
+
+
+# --------------------------------------------------------------------------- #
+# Label normalization
+# --------------------------------------------------------------------------- #
+
+
+def _normalize_label(label: Any) -> str:
+    """Unicode-safe case folding plus separator normalization.
+
+    `total_enrolments`, `Total Enrolments` and `total-enrolments` all fold to
+    `total enrolments`; Unicode letters survive the fold.
+    """
+    folded = _SEPARATOR_RE.sub(" ", str(label).casefold())
+    return folded.strip()
+
+
+def _canonical_for(label: Any, vocabulary: list[str]) -> str | None:
+    """Match a label against a pinned static vocabulary.
+
+    Direct normalized matches win; beyond that only the pinned semantic
+    aliases apply. Canonical values are returned unchanged.
+    """
+    wanted = _normalize_label(label)
+    for value in vocabulary:
+        if _normalize_label(value) == wanted:
+            return value
+    alias = _PINNED_SELECTOR_ALIASES.get(wanted)
+    if alias is not None and alias in vocabulary:
+        return alias
+    return None
+
+
+def _dedupe(values: list[Any]) -> list[Any]:
+    """Order-preserving deduplication."""
+    seen: set[Any] = set()
+    unique: list[Any] = []
+    for value in values:
+        if value not in seen:
+            seen.add(value)
+            unique.append(value)
+    return unique
+
+
+# --------------------------------------------------------------------------- #
+# Pinned-schema validation (draft 2020-12 subset, dependency-free)
+# --------------------------------------------------------------------------- #
+
+
+def _json_key(value: Any) -> str:
+    return json.dumps(value, sort_keys=True, ensure_ascii=False, default=str)
+
+
+def _type_matches(value: Any, expected: str) -> bool:
+    if expected == "object":
+        return isinstance(value, dict)
+    if expected == "array":
+        return isinstance(value, list)
+    if expected == "string":
+        return isinstance(value, str)
+    if expected == "integer":
+        if isinstance(value, bool):
+            return False
+        if isinstance(value, int):
+            return True
+        return isinstance(value, float) and value.is_integer()
+    if expected == "number":
+        return isinstance(value, (int, float)) and not isinstance(value, bool)
+    if expected == "boolean":
+        return isinstance(value, bool)
+    if expected == "null":
+        return value is None
+    return True  # unknown type keyword: leave to the remaining checks
+
+
+def _schema_problems(value: Any, schema: Any, path: str) -> list[str]:
+    """Validate `value` against the pinned request schema.
+
+    Supports the draft-2020-12 keywords the generated v0.3 request schemas
+    use: types, required/properties/additionalProperties, items, enum/const,
+    numeric bounds, cardinality, uniqueness, patterns, string lengths, and the
+    allOf/anyOf/oneOf and if/then/else combinators. Annotation keywords are
+    ignored.
+    """
+    problems: list[str] = []
+    if not isinstance(schema, dict):
+        return problems
+
+    expected_type = schema.get("type")
+    if expected_type is not None:
+        expected_types = expected_type if isinstance(expected_type, list) else 
[expected_type]
+        if not any(_type_matches(value, str(t)) for t in expected_types):
+            problems.append(
+                f"{path}: expected type {expected_type!r}, got 
{type(value).__name__}"
+            )
+            return problems
+
+    if "const" in schema and _json_key(value) != _json_key(schema["const"]):
+        problems.append(f"{path}: must equal {schema['const']!r}")
+
+    enum = schema.get("enum")
+    if isinstance(enum, list):
+        if not any(_json_key(value) == _json_key(candidate) for candidate in 
enum):
+            problems.append(f"{path}: {value!r} is not an allowed value")
+
+    if isinstance(value, dict):
+        properties = schema.get("properties")
+        properties = properties if isinstance(properties, dict) else {}
+        for key, sub_schema in properties.items():
+            if key in value:
+                problems.extend(_schema_problems(value[key], sub_schema, 
f"{path}.{key}"))
+        additional = schema.get("additionalProperties", True)
+        unknown = [key for key in value if key not in properties]
+        if unknown and additional is False:
+            problems.append(f"{path}: unknown properties {sorted(unknown)}")
+        elif isinstance(additional, dict):
+            for key in unknown:
+                problems.extend(_schema_problems(value[key], additional, 
f"{path}.{key}"))
+        required = schema.get("required")
+        if isinstance(required, list):
+            for key in required:
+                if key not in value:
+                    problems.append(f"{path}: missing required property 
{key!r}")
+        min_properties = schema.get("minProperties")
+        if isinstance(min_properties, int) and len(value) < min_properties:
+            problems.append(f"{path}: needs at least {min_properties} 
properties")
+        max_properties = schema.get("maxProperties")
+        if isinstance(max_properties, int) and len(value) > max_properties:
+            problems.append(f"{path}: allows at most {max_properties} 
properties")
+
+    if isinstance(value, list):
+        min_items = schema.get("minItems")
+        if isinstance(min_items, int) and len(value) < min_items:
+            problems.append(f"{path}: needs at least {min_items} items")
+        max_items = schema.get("maxItems")
+        if isinstance(max_items, int) and len(value) > max_items:
+            problems.append(f"{path}: allows at most {max_items} items")
+        if schema.get("uniqueItems"):
+            item_keys = [_json_key(item) for item in value]
+            if len(set(item_keys)) != len(item_keys):
+                problems.append(f"{path}: items must be unique")
+        items = schema.get("items")
+        if isinstance(items, dict):
+            for index, element in enumerate(value):
+                problems.extend(_schema_problems(element, items, 
f"{path}[{index}]"))
+
+    if isinstance(value, str):
+        pattern = schema.get("pattern")
+        if isinstance(pattern, str) and re.search(pattern, value) is None:
+            problems.append(f"{path}: does not match the required pattern")
+        min_length = schema.get("minLength")
+        if isinstance(min_length, int) and len(value) < min_length:
+            problems.append(f"{path}: shorter than the required minimum 
length")
+        max_length = schema.get("maxLength")
+        if isinstance(max_length, int) and len(value) > max_length:
+            problems.append(f"{path}: longer than the required maximum length")
+
+    if isinstance(value, (int, float)) and not isinstance(value, bool):
+        minimum = schema.get("minimum")
+        if isinstance(minimum, (int, float)) and value < minimum:
+            problems.append(f"{path}: below the minimum {minimum}")
+        maximum = schema.get("maximum")
+        if isinstance(maximum, (int, float)) and value > maximum:
+            problems.append(f"{path}: above the maximum {maximum}")
+
+    def _check(sub_schema: Any) -> list[str]:
+        return _schema_problems(value, sub_schema, path)
+
+    all_of = schema.get("allOf")
+    if isinstance(all_of, list):
+        for sub_schema in all_of:
+            problems.extend(_check(sub_schema))
+    any_of = schema.get("anyOf")
+    if isinstance(any_of, list) and any_of:
+        if all(len(_check(sub_schema)) > 0 for sub_schema in any_of):
+            problems.append(f"{path}: matches none of the allowed variants")
+    one_of = schema.get("oneOf")
+    if isinstance(one_of, list) and one_of:
+        matches = sum(1 for sub_schema in one_of if not _check(sub_schema))
+        if matches != 1:
+            problems.append(
+                f"{path}: must match exactly one allowed variant (matched 
{matches})"
+            )
+    if "if" in schema:
+        condition_holds = not _check(schema["if"])
+        branch = schema.get("then") if condition_holds else schema.get("else")
+        if isinstance(branch, dict):
+            problems.extend(_check(branch))
+    not_schema = schema.get("not")
+    if isinstance(not_schema, dict) and not _check(not_schema):
+        problems.append(f"{path}: matches a forbidden variant")
+
+    return problems
+
+
+def _validate_request(question_id: str, request: dict[str, Any]) -> None:
+    """Validate the planner-built request against its registered pinned schema.
+
+    Any failure — including schema load or resolution failure — is a
+    TerminalError before dispatch, never an Unsupported user outcome.
+    """
+    schema = _request_schema(question_id)
+    problems = _schema_problems(request, schema, "$request")
+    if problems:
+        raise TerminalError(
+            f"{question_id}: the planner-built request does not satisfy its "
+            f"registered request schema: " + "; ".join(problems)
+        )
+
+
+# --------------------------------------------------------------------------- #
+# School identity resolution (transactional, exactly-one-match)
+# --------------------------------------------------------------------------- #
+
+
+async def _resolve_mentions(
+    question_id: str,
+    mentions: list[str],
+    client: EvidenceClient,
+    deps: Deps,
+) -> tuple[list[int], ClarificationNeeded | None]:
+    """Resolve each mention through the pinned directory, exactly one match 
each.
+
+    Every preflight is recorded, including failures. IDs are staged locally in
+    mention order and deduplicated without reordering; nothing is committed
+    here — the caller commits to `Deps` only after every mention resolves.
+    """
+    staged: list[int] = []
+    for mention in mentions:
+        arguments = {"search": mention, "limit": 25}
+        try:
+            records = await client.school_directory(search=mention, limit=25)
+        except Exception as exc:
+            deps.tool_calls.append(
+                ToolInvocation("search_schools", arguments, 
error_kind=type(exc).__name__)
+            )
+            raise
+        deps.tool_calls.append(ToolInvocation("search_schools", arguments, 
records))
+        ids = [
+            record["acara_sml_id"]
+            for record in records
+            if isinstance(record, dict)
+            and isinstance(record.get("acara_sml_id"), int)
+            and not isinstance(record["acara_sml_id"], bool)
+        ]
+        if not ids:
+            return [], _school_match_clarification(
+                question_id,
+                mention,
+                f"'{mention}' matched no school in the pinned directory.",
+                "I couldn't find that school. Could you check the name — or 
give me "
+                "the school's public ACARA ID?",
+            )
+        if len(ids) > 1:
+            return [], _school_match_clarification(
+                question_id,
+                mention,
+                f"'{mention}' matched {len(ids)} schools; a request needs 
exactly one.",
+                f"More than one school matches '{mention}'. Which one did you 
mean?",
+            )
+        school_id = ids[0]
+        if school_id not in staged:
+            staged.append(school_id)
+    return staged, None
+
+
+def _school_match_clarification(
+    question_id: str, mention: str, why_material: str, prompt_to_user: str
+) -> ClarificationNeeded:
+    return ClarificationNeeded(
+        question_id=question_id,
+        missing=[
+            MissingInput(input="acara_sml_id", why_material=why_material, 
allowed_values=[])
+        ],
+        prompt_to_user=prompt_to_user,
+    )
+
+
+def _adapt_explicit_school_fields(request: dict[str, Any], school_required: 
list[str]) -> None:
+    """Let a caller-supplied identity in either school field satisfy the
+    question's own selector field, without inventing values."""
+    if not school_required:
+        return
+    if "acara_sml_ids" in school_required:
+        if "acara_sml_ids" not in request and 
isinstance(request.get("acara_sml_id"), int):
+            request["acara_sml_ids"] = [request.pop("acara_sml_id")]
+    elif "acara_sml_id" in school_required:
+        if "acara_sml_id" not in request and 
isinstance(request.get("acara_sml_ids"), list):
+            ids = request["acara_sml_ids"]
+            if len(ids) == 1 and isinstance(ids[0], int) and not 
isinstance(ids[0], bool):
+                request["acara_sml_id"] = ids[0]
+                del request["acara_sml_ids"]
+            else:
+                raise TerminalError(
+                    "caller supplied multiple acara_sml_ids for a question 
whose schema "
+                    "takes a single acara_sml_id"
+                )
+
+
+def _attach_school_ids(
+    request: dict[str, Any], school_required: list[str], ids: list[int]
+) -> None:
+    if not ids:
+        return
+    if "acara_sml_ids" in school_required:
+        request.setdefault("acara_sml_ids", list(ids))
+    elif "acara_sml_id" in school_required:
+        request.setdefault("acara_sml_id", ids[0])
+
+
+def _explicit_id_values(request: dict[str, Any], school_required: list[str]) ->
list[int]:
+    if "acara_sml_ids" in school_required:
+        value = request.get("acara_sml_ids")
+        if isinstance(value, list):
+            return [v for v in value if isinstance(v, int) and not 
isinstance(v, bool)]
+        return []
+    if "acara_sml_id" in school_required:
+        value = request.get("acara_sml_id")
+        if isinstance(value, int) and not isinstance(value, bool):
+            return [value]
+    return []
+
+
+def _requires_two_distinct_ids(entry: dict[str, Any]) -> bool:
+    constraints = entry.get("input_policy", {}).get("constraints", [])
+    return any("at least two" in str(constraint) for constraint in constraints)
+
+
+def _insufficient_schools_clarification(question_id: str) -> 
ClarificationNeeded:
+    return ClarificationNeeded(
+        question_id=question_id,
+        missing=[
+            MissingInput(
+                input="acara_sml_ids",
+                why_material="This comparison needs at least two distinct 
schools.",
+                allowed_values=[],
+            )
+        ],
+        prompt_to_user="Which distinct schools should be compared? Name at 
least two.",
+    )
+
+
+# --------------------------------------------------------------------------- #
+# Selector normalization and routing
+# --------------------------------------------------------------------------- #
+
+
+def _normalize_scalar_selectors(
+    question_id: str, request: dict[str, Any]
+) -> ClarificationNeeded | None:
+    """Normalize static-closed scalar selectors (market_definition) against
+    their pinned vocabulary. Unnormalizable values clarify before dispatch."""
+    if "market_definition" not in request:
+        return None
+    value = request["market_definition"]
+    if not isinstance(value, str):
+        return None  # shape-checked by the pinned schema
+    vocabulary = _static_vocabulary_for_field(question_id, "market_definition")
+    if vocabulary is None:
+        return None
+    canonical = _canonical_for(value, vocabulary)
+    if canonical is None:
+        return ClarificationNeeded(
+            question_id=question_id,
+            missing=[
+                MissingInput(
+                    input="market_definition",
+                    why_material=(
+                        f"'{value}' is not an allowed market definition for 
{question_id}."
+                    ),
+                    allowed_values=vocabulary,
+                )
+            ],
+            prompt_to_user=(
+                "Which market definition should I use? Allowed values: "
+                + ", ".join(vocabulary)
+                + "."
+            ),
+        )
+    request["market_definition"] = canonical
+    return None
+
+
+def _route_selectors(
+    question_id: str,
+    intent: StructuredIntent,
+    request: dict[str, Any],
+    deps: Deps,
+) -> ClarificationNeeded | None:
+    """Route requested selector labels to the question's own request field.
+
+    Static vocabularies come from pinned artefacts (registry metric_list where
+    declared, otherwise the input-catalog schema's field mapping and embedded
+    enums) — never from the runtime emitter. Selectors the user named none of
+    are omitted so the emitter keeps default ownership.
+    """
+    labels = [str(label) for label in (intent.requested_metrics or []) if 
str(label).strip()]
+    if not labels:
+        return None
+    route = _selector_route(question_id)
+    if route is None:
+        return ClarificationNeeded(
+            question_id=question_id,
+            missing=[
+                MissingInput(
+                    input="metrics",
+                    why_material=(
+                        f"{question_id} takes no metric selector, so the 
requested "
+                        "labels cannot be applied to this request."
+                    ),
+                    allowed_values=[],
+                )
+            ],
+            prompt_to_user="This question does not take metric selectors. 
Should I answer it as asked?",
+        )
+    field, vocabulary = route
+    canonical: list[str] = []
+    for label in labels:
+        matched = _canonical_for(label, vocabulary)
+        if matched is None:
+            return ClarificationNeeded(
+                question_id=question_id,
+                missing=[
+                    MissingInput(
+                        input=field,
+                        why_material=(
+                            f"'{label}' is not an allowed {field} value for 
{question_id}."
+                        ),
+                        allowed_values=vocabulary,
+                    )
+                ],
+                prompt_to_user=(
+                    f"Which {field} did you mean? Allowed values: "
+                    + ", ".join(vocabulary)
+                    + "."
+                ),
+            )
+        canonical.append(matched)
+    canonical = _dedupe(canonical)
+
+    if field == "metrics":
+        # Existing raw-question scope rule: a user who never asked about
+        # headcounts or non-teaching staff gets neither, even if a label was
+        # requested. Applied before validation, as before.
+        scope_problem = request_scope_problem(
+            deps.user_question, question_id, {**request, "metrics": canonical}
+        )
+        if scope_problem:
+            cleaned = [
+                metric
+                for metric in canonical
+                if not (metric.endswith("_headcount") or 
metric.startswith("non_teaching_"))
+            ]
+            canonical = cleaned or canonical
+
+    request[field] = canonical
+    return None
+
+
+# --------------------------------------------------------------------------- #
+# Registry-owned explicit-filter constraints (VC-POC-09/10, OD-07)
+# --------------------------------------------------------------------------- #
+
+
+def _enforce_explicit_filter_constraints(
+    question_id: str, entry: dict[str, Any], request: dict[str, Any]
+) -> ClarificationNeeded | None:
+    """The registry-owned VC-POC-09/10 rule: a cohort definition must contain
+    at least one explicit filter — the sector default alone does not define a
+    peer group. Enforced exactly as pinned, not generalised into a condition
+    language. Absent fields are left to policy derivation."""
+    policy = entry.get("input_policy") or {}
+    required = [f for f in policy.get("required", []) if isinstance(f, str)]
+
+    if "peer_filters" in required and isinstance(request.get("peer_filters"), 
dict):
+        filters = request["peer_filters"]
+        if not filters:
+            return _cohort_definition_clarification(
+                question_id,
+                "peer_filters",
+                "The peer group definition is empty; at least one explicit 
filter "
+                "is required.",
+            )
+        default_sectors = policy.get("defaultable", 
{}).get("peer_filters.sectors")
+        if (
+            set(filters) == {"sectors"}
+            and isinstance(default_sectors, list)
+            and isinstance(filters.get("sectors"), list)
+            and sorted(str(s) for s in filters["sectors"])
+            == sorted(str(s) for s in default_sectors)
+        ):
+            return _cohort_definition_clarification(
+                question_id,
+                "peer_filters",
+                "The sector default alone does not define a peer group; at 
least one "
+                "explicit filter beyond the sector default is required.",
+            )
+
+    if "segment_filters" in required and 
isinstance(request.get("segment_filters"), dict):
+        if not request["segment_filters"]:
+            return _cohort_definition_clarification(
+                question_id,
+                "segment_filters",
+                "The segment definition is empty; at least one explicit filter 
"
+                "is required.",
+            )
+    return None
+
+
+def _cohort_definition_clarification(
+    question_id: str, field: str, why_material: str
+) -> ClarificationNeeded:
+    return ClarificationNeeded(
+        question_id=question_id,
+        missing=[MissingInput(input=field, why_material=why_material, 
allowed_values=[])],
+        prompt_to_user=(
+            "Which rule should define this cohort — for example a sector, 
school "
+            "type, enrolment band, ICSEA range, remoteness or suburb?"
+        ),
+    )
+
+
+# --------------------------------------------------------------------------- #
+# Derived missing inputs (pinned policy order)
+# --------------------------------------------------------------------------- #
+
+
+def _derive_missing_inputs(
+    question_id: str, entry: dict[str, Any], request: dict[str, Any]
+) -> list[str]:
+    """Missing required, material, then applicable conditional-material fields,
+    in pinned policy order — derived from policy and the assembled request,
+    never taken from the model's missing-input claim."""
+    policy = entry.get("input_policy") or {}
+    missing: list[str] = []
+    seen: set[str] = set()
+
+    def consider(field: str) -> None:
+        if field in seen:
+            return
+        seen.add(field)
+        if field not in request:
+            missing.append(field)
+
+    for field in policy.get("required", []):
+        if isinstance(field, str):
+            consider(field)
+    for field in policy.get("material", []):
+        if isinstance(field, str):
+            consider(field)
+
+    conditional = policy.get("conditional_material") or {}
+    if isinstance(conditional, dict):
+        for field, condition in conditional.items():
+            if not isinstance(field, str):
+                continue
+            normalized_condition = _SEPARATOR_RE.sub(" ", 
str(condition).casefold()).strip()
+            if field == "distance_km" and normalized_condition in 
_KNOWN_DISTANCE_CONDITIONS:
+                if request.get("market_definition") == "distance" and 
"distance_km" not in request:
+                    consider(field)
+            else:
+                raise TerminalError(
+                    f"{question_id}: unsupported conditional-material 
expression "
+                    f"{condition!r} for {field} — contract drift, refusing to 
interpret it"
+                )
+    return missing
+
+
+# --------------------------------------------------------------------------- #
+# Stage 2 entry point
+# --------------------------------------------------------------------------- #
+
+
 async def plan_and_execute(
     intent: StructuredIntent,
     client: EvidenceClient,
     deps: Deps,
 ) -> dict[str, Any] | ClarificationNeeded | Unsupported:
-    """Deterministic Stage 2: intent → evidence envelope or typed refusal.
+    """Deterministic Stage 2: intent → evidence envelope or typed outcome.
 
     Returns:
         dict — the evidence envelope (happy path)
-        ClarificationNeeded — a material input is missing
-        Unsupported — an exclusion, unknown question, or unanswerable request
+        ClarificationNeeded — a caller-suppliable input is missing or wrong
+        Unsupported — an exclusion or an unknown question
     May raise:
-        TerminalError — an operational stop (data_unavailable, internal)
+        TerminalError — planner/schema defects and operational stops
     """
     question_id = intent.question_id
 
     # ------------------------------------------------------------------ #
     # 1. Excluded topic (intent model classified it as out of scope)
@@ -157,112 +910,134 @@
             reason_kind="not_in_registry",
             reason=f"{question_id} is not a registry question.",
         )
 
     # ------------------------------------------------------------------ #
-    # 5. Material inputs missing → ask for clarification
-    # ------------------------------------------------------------------ #
-    if intent.missing_material_inputs:
-        material = contract.material_inputs_for(question_id)
-        actually_missing = [inp for inp in intent.missing_material_inputs if 
inp in material]
-        if actually_missing:
-            missing_items = [
-                MissingInput(
-                    input=inp,
-                    why_material=_material_why(question_id, inp),
-                    allowed_values=[],
+    # 5. Pinned policy and assembled request
+    # ------------------------------------------------------------------ #
+    entry = contract.questions().get(question_id)
+    if entry is None:
+        raise TerminalError(
+            f"{question_id} has a registered request schema but is absent from 
"
+            "the pinned registry — contract drift"
+        )
+    policy = entry.get("input_policy") or {}
+    required_fields = [f for f in policy.get("required", []) if isinstance(f, 
str)]
+    school_required = [f for f in required_fields if f in _SCHOOL_FIELDS]
+
+    # Fresh request from explicit intent fields only; planner-owned selectors
+    # are added below. Unsupplied defaultables stay omitted — the emitter owns
+    # defaults and resolved_request.
+    request: dict[str, Any] = dict(intent.filters)
+
+    scalar_clarification = _normalize_scalar_selectors(question_id, request)
+    if scalar_clarification is not None:
+        return scalar_clarification
+
+    _adapt_explicit_school_fields(request, school_required)
+    identity_from_caller = any(field in request for field in school_required)
+
+    # ------------------------------------------------------------------ #
+    # 6. Transactional named-school resolution
+    # ------------------------------------------------------------------ #
+    mentions = [
+        mention for mention in (intent.school_mentions or [])
+        if isinstance(mention, str) and mention.strip()
+    ]
+    if school_required and mentions:
+        resolved_ids, failure = await _resolve_mentions(question_id, mentions, 
client, deps)
+        if failure is not None:
+            return failure
+        if identity_from_caller:
+            explicit_ids = _explicit_id_values(request, school_required)
+            if sorted(resolved_ids) != sorted(explicit_ids):
+                raise TerminalError(
+                    f"{question_id}: explicit school identities {explicit_ids} 
conflict "
+                    f"with directory-resolved identities {resolved_ids}"
                 )
-                for inp in actually_missing
-            ]
-            prompt_to_user = (
-                f"To answer this question I need "
-                + ", ".join(f"the {inp}" for inp in actually_missing)
-                + ". Could you provide "
-                + ("it" if len(actually_missing) == 1 else "them")
-                + "?"
-            )
-            return ClarificationNeeded(
-                question_id=question_id,
-                missing=missing_items,
-                prompt_to_user=prompt_to_user,
-            )
-
-    # ------------------------------------------------------------------ #
-    # 6. Resolve schools
-    # ------------------------------------------------------------------ #
-    for school_name in intent.school_mentions:
-        try:
-            records = await client.school_directory(search=school_name, 
limit=25)
-        except Exception as exc:
-            deps.tool_calls.append(
-                ToolInvocation(
-                    "search_schools",
-                    {"search": school_name, "limit": 25},
-                    error_kind=type(exc).__name__,
-                )
-            )
-            raise
-
-        deps.searched = True
-        resolved = [r["acara_sml_id"] for r in records if "acara_sml_id" in r]
-        if resolved:
-            # Take the first match: the planner trusts the intent model to have
-            # named an unambiguous school. Ambiguity handling 
(ClarificationNeeded)
-            # is the live-model path; the planner uses best-effort resolution.
-            best = resolved[0]
-            if best not in deps.resolved_school_ids:
-                deps.resolved_school_ids.append(best)
-        deps.tool_calls.append(
-            ToolInvocation(
-                "search_schools",
-                {"search": school_name, "limit": 25},
-                records,
-            )
-        )
-
-    # ------------------------------------------------------------------ #
-    # 7. Build the evidence request
-    # ------------------------------------------------------------------ #
-    request: dict[str, Any] = dict(intent.filters)
-
-    # Attach resolved school IDs using the right field name for this question
-    if deps.resolved_school_ids:
-        q_entry = contract.questions()[question_id]
-        required = q_entry.get("input_policy", {}).get("required", [])
-        if "acara_sml_ids" in required:
-            request.setdefault("acara_sml_ids", deps.resolved_school_ids)
-        elif "acara_sml_id" in required and deps.resolved_school_ids:
-            request.setdefault("acara_sml_id", deps.resolved_school_ids[0])
-
-    # Map requested metrics to the request (NL labels are passed through for
-    # now — the real emitter validates them; bad values return bad_request)
-    if intent.requested_metrics:
-        request.setdefault("metrics", intent.requested_metrics)
-
-    # Apply metric scope check for VC-POC-01 and VC-POC-04
-    scope_problem = request_scope_problem(deps.user_question, question_id, 
request)
-    if scope_problem and "metrics" in request and 
isinstance(request["metrics"], list):
-        # Drop non-teaching and headcount expansions the user didn't ask for
-        cleaned = [
-            m for m in request["metrics"]
-            if not (
-                str(m).endswith("_headcount") or 
str(m).startswith("non_teaching_")
-            )
+            deps.searched = True
+        else:
+            if _requires_two_distinct_ids(entry) and len(resolved_ids) < 2:
+                return _insufficient_schools_clarification(question_id)
+            for school_id in resolved_ids:
+                if school_id not in deps.resolved_school_ids:
+                    deps.resolved_school_ids.append(school_id)
+            deps.searched = True
+            _attach_school_ids(request, school_required, resolved_ids)
+    elif school_required and identity_from_caller and "acara_sml_ids" in 
school_required:
+        explicit_ids = _explicit_id_values(request, school_required)
+        raw_ids = request.get("acara_sml_ids")
+        if (
+            _requires_two_distinct_ids(entry)
+            and isinstance(raw_ids, list)
+            and len(explicit_ids) == len(raw_ids)
+            and len(_dedupe(explicit_ids)) < 2
+        ):
+            return _insufficient_schools_clarification(question_id)
+
+    # ------------------------------------------------------------------ #
+    # 7. Selector normalization and routing (clarifies unnormalizable labels)
+    # ------------------------------------------------------------------ #
+    selector_clarification = _route_selectors(question_id, intent, request, 
deps)
+    if selector_clarification is not None:
+        return selector_clarification
+
+    # ------------------------------------------------------------------ #
+    # 8. Registry-owned explicit-filter constraints (VC-POC-09/10)
+    # ------------------------------------------------------------------ #
+    constraint_clarification = _enforce_explicit_filter_constraints(
+        question_id, entry, request
+    )
+    if constraint_clarification is not None:
+        return constraint_clarification
+
+    # ------------------------------------------------------------------ #
+    # 9. Derive missing inputs from the pinned policy (stable order)
+    # ------------------------------------------------------------------ #
+    missing = _derive_missing_inputs(question_id, entry, request)
+    if missing:
+        missing_items = [
+            MissingInput(
+                input=field,
+                why_material=_material_why(question_id, field),
+                allowed_values=_static_vocabulary_for_field(question_id, field)
or [],
+            )
+            for field in missing
         ]
-        request["metrics"] = cleaned or request["metrics"]
-
-    # ------------------------------------------------------------------ #
-    # 8. Fetch evidence
+        prompt_to_user = (
+            "To answer this question I need "
+            + ", ".join(f"the {field}" for field in missing)
+            + ". Could you provide "
+            + ("it" if len(missing) == 1 else "them")
+            + "?"
+        )
+        logger.debug(
+            "planner clarification",
+            extra={"subsystem": "planner", "question_id": question_id, 
"missing": missing},
+        )
+        return ClarificationNeeded(
+            question_id=question_id,
+            missing=missing_items,
+            prompt_to_user=prompt_to_user,
+        )
+
+    # ------------------------------------------------------------------ #
+    # 10. Pinned-schema validation before dispatch
+    # ------------------------------------------------------------------ #
+    _validate_request(question_id, request)
+
+    # ------------------------------------------------------------------ #
+    # 11. Fetch evidence (failures recorded before branching)
     # ------------------------------------------------------------------ #
     arguments = {"question_id": question_id, "request": request}
     try:
         envelope = await client.fetch_evidence(question_id, request)
     except EvidenceError as exc:
         deps.tool_calls.append(
             ToolInvocation("fetch_evidence", arguments, 
error_kind=exc.error_kind)
         )
-        return _branch_evidence_error(question_id, exc)
+        return _branch_evidence_error(question_id, exc, identity_from_caller)
     except Exception:
         deps.tool_calls.append(
             ToolInvocation("fetch_evidence", arguments, 
error_kind="operational")
         )
         raise
@@ -284,26 +1059,32 @@
         "grades": "The grade selection determines which year levels are 
included.",
         "peer_filters": "The peer group definition determines which schools are
compared.",
         "segment_filters": "The segment definition determines which schools are
included.",
         "sa2_codes": "The SA2 area determines which geographic area is 
analysed.",
         "zone_entity_code": "The zone entity code identifies the government 
zone to look up.",
+        "acara_sml_id": "The school must be identified before evidence can be 
fetched.",
+        "acara_sml_ids": "The schools must be identified before evidence can be
fetched.",
     }
     return explanations.get(
         input_name,
         f"{input_name} is required by the data layer and cannot be defaulted 
for this question.",
     )
 
 
 def _branch_evidence_error(
     question_id: str,
     exc: EvidenceError,
-) -> Unsupported:
-    """Translate an EvidenceError into an Unsupported outcome.
-
-    This mirrors `_branch_on_error_kind()` in tools.py but returns typed values
-    instead of raising PydanticAI signals. TerminalError is re-raised (never
-    returned as Unsupported — it is an operational stop, not an honest 
refusal).
+    identity_from_caller: bool,
+) -> Unsupported | ClarificationNeeded:
+    """Translate an EvidenceError into the pipeline's typed outcomes.
+
+    Origin-aware: a bad_request for a request whose school identity was
+    caller-supplied is a correctable identity problem → ClarificationNeeded.
+    Any other bad_request — the request was deterministic and schema-valid —
+    is planner/contract drift → TerminalError. excluded/unknown/unanswerable
+    keep their contract-owned refusals; data_unavailable/internal are
+    operational stops, never refusals.
     """
     kind = exc.error_kind
     logger.debug(
         "evidence error in planner",
         extra={"subsystem": "planner", "error_kind": kind, "question_id": 
question_id},
@@ -327,11 +1108,31 @@
 
     if kind == "unanswerable":
         return Unsupported(reason_kind="unanswerable", reason=exc.message)
 
     if kind == "bad_request":
-        # On a first bad_request the planner cannot repair (no model); return
-        # out_of_scope so the caller knows the data layer rejected the request.
-        return Unsupported(reason_kind="out_of_scope", reason=exc.message)
+        if identity_from_caller:
+            return ClarificationNeeded(
+                question_id=question_id,
+                missing=[
+                    MissingInput(
+                        input="acara_sml_id",
+                        why_material=(
+                            "The data layer rejected the supplied school 
identity "
+                            f"({exc.message}). Confirm the school's current 
public "
+                            "ACARA_SML_ID."
+                        ),
+                        allowed_values=[],
+                    )
+                ],
+                prompt_to_user=(
+                    "The school identity supplied for this question was 
rejected by the "
+                    "data layer. Could you confirm the school?"
+                ),
+            )
+        raise TerminalError(
+            f"{question_id}: the evidence emitter returned bad_request for a "
+            f"deterministic, schema-valid request — planner/contract drift: 
{exc.message}"
+        )
 
     # data_unavailable | internal — operational stop, not a refusal.
     raise TerminalError(f"evidence emitter reported {kind}: {exc.message}")
```



**Changed files**: `harness/planner.py` only — `_SCHEMA_DIR`, the input-catalog 
path and the containment check now derive from `contract.CONTRACT_DIR` (the 
`settings.PROJECT_ROOT`-based path in `harness/contract.py`) instead of 
recomputing the repo layout in the planner. Behaviour is identical; this removes
a duplicated path constant and keeps the planner on the same pinned root the 
rest of the harness reads.



**No changes needed**: `run.py`, `tests/unit/test_planner.py`, 
`tests/unit/test_intent_boundary.py`, `tests/unit/test_run_cli.py` — every 
assumption they make about `Deps`, `ToolInvocation`, `request_scope_problem`, 
`harness.contract` and the stub bundle is now verified against the actual files.



**Checks performed** (static, against the newly supplied files): constructor 
signatures, dataclass defaults, preflight/fetch call shapes, 
canonical-request-to-fixture-match alignment for VC-POC-01/02/03/05/08, unique 
directory resolution for the fixture school names, and the 
registry/input-catalog lookups the planner performs. The pytest commands remain 
unexecuted here (no shell access) and are left for the harness's verification 
steps. If you can add the remaining pinned request schemas, I'll run the same 
keyword-by-keyword check across all sixteen before you execute the suite.

Tokens: 152k sent, 19k received. Cost: $0.03 message, $0.11 session.

harness/intent.py

harness/outputs.py

--------------
► **THINKING**

The user has added harness/intent.py and harness/outputs.py to the chat. Let me 
verify my assumptions against these files.

**harness/intent.py** — I already had this in the chat from earlier (it was 
provided in the "I have added these files" message). Let me re-check:
- `StructuredIntent` has fields: `question_id`, `excluded_topic`, 
`school_mentions`, `requested_metrics`, `filters`, `presentation`, 
`missing_material_inputs`. All match my usage.
- `interpret_intent(question, model=None)` — matches my test usage.

Wait, actually intent.py was already provided earlier in the conversation. The 
user says "I added these files to the chat: harness/intent.py, 
harness/outputs.py". intent.py was already there. outputs.py is new.

**harness/outputs.py** — checking my usage:
- `ClarificationNeeded(question_id=..., missing=[MissingInput(...)], 
prompt_to_user=...)` — matches.
- `MissingInput(input=..., why_material=..., allowed_values=...)` — matches 
(allowed_values: list[str], default_factory=list).
- `Unsupported(reason_kind=..., reason=..., reason_code=None, 
nearest_supported=None)` — matches.
- `HarnessOutput` union — matches.
- `RenderedAnswer` — matches.
- `BudgetExceeded` — matches.

Everything checks out. One thing to verify: `MissingInput.allowed_values` is 
`list[str]` with `default_factory=list`. In my planner, I pass 
`allowed_values=vocabulary` where vocabulary is `list[str]` — good. In 
`test_planner.py` I assert `result.missing[1].allowed_values == ["distance", 
"lga", "postcode", "sa2"]` — the input-catalog schema's market_definitions enum 
order is `["distance", "lga", "postcode", "sa2"]` — matches the schema in the 
read-only context.

Also `ClarificationNeeded` requires `missing` non-empty (docstring says "Never 
empty" but there's no validator — just descriptive). Fine.

So no changes needed to the editable files. Let me double check a couple of 
subtle things:

1. In my planner's `_route_selectors`, when `intent.requested_metrics` contains 
a label for VC-POC-01 like "teaching staff", `_canonical_for` maps to 
`teaching_fte` via the alias. Good.

2. In test_planner.py `test_metric_normalization_and_exact_profile_request`, 
labels include "Teaching Staff" twice (once via alias once direct "teaching 
staff"), and "TOTAL_ENROLMENTS". Normalization: `_normalize_label("Teaching 
Staff")` → "teaching staff" → alias → "teaching_fte". 
`_normalize_label("TOTAL_ENROLMENTS")` → "total enrolments" → matches 
"total_enrolments" normalized → "total enrolments" → canonical 
"total_enrolments". Order: labels processed in order: "Total Enrolments" → 
total_enrolments; "Teaching Staff" → teaching_fte; "teaching staff" → 
teaching_fte (dup); "TOTAL_ENROLMENTS" → total_enrolments (dup). Deduped: 
["total_enrolments", "teaching_fte"]. 

Then the scope check: `request_scope_problem(deps.user_question, "VC-POC-01", 
{**request, "metrics": ["total_enrolments", "teaching_fte"]})` — 
deps.user_question for the planner tests: in test_planner.py, 
`Deps(client=client)` — user_question defaults to "". The regex won't match 
headcount wording (empty string), so extras = metrics ending with `_headcount` 
(none) or starting with `non_teaching_` (none). So no scope problem. Request: 
{"acara_sml_id": 40387, "metrics": ["total_enrolments", "teaching_fte"]}. 

3. `test_ask_pipeline_missing_material_input_returns_clarification` in 
test_intent_boundary.py: intent VC-POC-05 with 
missing_material_inputs=["market_definition"], no filters. My planner: 
school_required=["acara_sml_id"], request={}, scalar normalize none, no explicit
identity, no mentions → skip resolution. Route selectors: no labels → None. 
Explicit filter: N/A. Derive missing: required = [acara_sml_id, 
market_definition], material = [market_definition]. Both missing → names == 
["acara_sml_id", "market_definition"]. Test asserts "market_definition" in 
missing_names — passes (it's a membership check, not order).

4. In test_intent_boundary `test_ask_pipeline_intent_override_bypasses_model`: 
intent VC-POC-01, filters={"acara_sml_id": 40387}, 
requested_metrics=["total_enrolments", "teaching_fte"], narration_override 
given. Stub client: fetch VC-POC-01 with {"acara_sml_id": 40387, "metrics": 
["total_enrolments", "teaching_fte"]}. Stub emitter `matches()`: 
VC-POC-01/canonical has match {"acara_sml_id": 40387, "metrics": 
["total_enrolments", "teaching_fte"]} — subset match requires every match key 
present and equal: both present and equal. Returns canonical envelope. Envelope 
request field overwritten. Then answer view built, narration override validated.
The narration answer "Enrolments run 612 in 2019 and 668 in 2023." — numbers 
612, 2019, 2023, 668 must appear in evidence/question. The canonical envelope 
has 612 (2019) and 668 (2023). The validation in narration module — 
`_validate_narration(view, deps, narration)` — presumably checks grounding. 
Existing test passed before with the same narration against the same fixture, so
this continues to work as long as the request reaches the fixture. Before my 
change, the old planner would send `metrics` verbatim = ["total_enrolments", 
"teaching_fte"] (already canonical) — same request. Good.

Wait — actually before my change, the old planner copied `requested_metrics` 
verbatim; the old test used canonical labels already. My normalization preserves
them. Same request. Good.

5. `test_rendered_answer_always_uses_deterministic_data_status_note`: intent 
VC-POC-01 with metrics canonical, filters={"acara_sml_id": 99999}, env 
STUB_EMITTER_CASE=VC-POC-01/empty-series. Forced case ignores matching. Request 
includes metrics — finalise_response overwrites `request` with supplied. Fine.

6. `test_ask_pipeline_stores_answer_view_on_deps` — same as #4.

7. `test_unnormalizable_metric_stays_a_clarification` — VC-POC-01 with "banana 
enrolments": not in profile_metrics → ClarificationNeeded with allowed_values = 
profile_metrics list. `output.missing[0].allowed_values` non-empty. Passes.

8. `test_normalized_schema_valid_request_reaches_evidence` — request: 
{"acara_sml_id": 40387, "metrics": ["total_enrolments", "teaching_fte"]}. Stub 
canonical matches. Good.

9. test_planner.py 
`test_missing_market_definition_is_derived_without_model_hint`: VC-POC-05, no 
mentions, no filters. missing = ["acara_sml_id", "market_definition"]. 
allowed_values for market_definition from 
`_static_vocabulary_for_field("VC-POC-05", "market_definition")`. Check the 
input-catalog mapping: "VC-POC-05.market_definition": "market_definitions". 
`_static_enum("market_definitions")` → properties.market_definitions.items.enum 
= ["distance", "lga", "postcode", "sa2"]. 

But wait — for acara_sml_id, `_static_vocabulary_for_field("VC-POC-05", 
"acara_sml_id")` → mapping has "VC-POC-01.acara_sml_id" but not 
"VC-POC-05.acara_sml_id"... Actually the mapping in the input catalog schema 
only lists "VC-POC-01.metrics" etc. There's no acara field entries at all. So 
None → `[]`. Good.

10. `test_distance_is_conditional_on_distance_market` first part: VC-POC-05 
filters={"acara_sml_id": 40387, "market_definition": "sa2"}. 
`_normalize_scalar_selectors` normalizes "sa2" → canonical "sa2" (direct match).
Request unchanged. missing: required=[acara_sml_id, market_definition] present; 
conditional distance_km: market_definition != "distance" → not applicable. 
Validate against VC-POC-05 request schema: acara_sml_id (int≥1 ✓), 
market_definition enum ["distance","lga","postcode","sa2"] ✓. 
additionalProperties false — only those two keys ✓. Dispatch: 
client.fetched[0]["request"] == {"acara_sml_id": 40387, "market_definition": 
"sa2"} — dict equality ignores insertion order. ✓

Second part: distance without distance_km → derive missing includes distance_km 
(condition normalized: registry says "market_definition equals distance" → 
normalized "market definition equals distance" ∈ _KNOWN_DISTANCE_CONDITIONS ✓). 
Clarification ["distance_km"] ✓, fetched == [] ✓.

11. `test_clarifications_follow_pinned_policy_order`: VC-POC-07 with mention 
"Alpha College" resolving to 40387, filters={"market_definition": "distance"}. 
Resolution succeeds (one match), not identity_from_caller, attach 
acara_sml_id=40387. Selector route for VC-POC-07: metric_list null; field 
mapping has "VC-POC-07.grades": "grades" → route ("grades", grades enum). labels
empty → None. Explicit filter: N/A. Derive missing: required = [acara_sml_id, 
market_definition, grades] — acara present, market present, grades missing → 
["grades"]; material = [market_definition, grades] both present except grades 
(already seen); conditional distance_km applicable and missing → appended. names
== ["grades", "distance_km"] ✓.

12. `test_conflicting_explicit_id_and_mention_is_terminal`: VC-POC-01 with 
mention "Beta Grammar" → 40392 resolved; explicit acara_sml_id=40387 → conflict 
→ TerminalError ✓, resolved_school_ids == [] ✓ (not committed), fetched == [] ✓.

13. `test_consistent_explicit_id_and_mention_proceeds`: resolved 40387 == 
explicit 40387 → deps.searched = True; request stays {"acara_sml_id": 40387}. 
Validation passes. fetched[0]["request"] == {"acara_sml_id": 40387} ✓.

14. `test_mention_order_preserved_and_duplicates_removed`: VC-POC-03 mentions 
["Beta Grammar", "Alpha College", "Beta Grammar"] → staged [40392, 40387]. Not 
identity_from_caller → committed to deps.resolved_school_ids [40392, 40387], 
attach acara_sml_ids. Request {"acara_sml_ids": [40392, 40387]}. Validation: 
VC-POC-03 schema — minItems 2? No: VC-POC-03 minItems is 1. OK ✓. Route: 
VC-POC-03 labels empty → no ratios. Request exactly {"acara_sml_ids": [40392, 
40387]} ✓. Stub: fetched on _DirectoryClient (mock) — no stub matching involved.
✓

15. `test_comparison_with_one_distinct_school_clarifies`: VC-POC-04 mentions 
Alpha ×2 → staged [40387], len < 2 → insufficient clarification. fetched == [] 
✓, resolved_school_ids == [] ✓.

16. `test_explicit_numeric_ids_skip_directory_preflight`: VC-POC-03 explicit 
acara_sml_ids [99999, 40387]. mentions empty → elif branch: school_required and 
identity_from_caller and "acara_sml_ids" in school_required. explicit_ids = 
[99999, 40387]; requires_two_distinct → dedupe len 2 ≥ 2 → no clarification. 
searches == [] ✓. Validation: uniqueItems ✓, minItems 2 ✓ (VC-POC-04 requires 2 
but this is VC-POC-03, minItems 1). Dispatch request unchanged ✓.

17. `test_peer_filters_require_explicit_nondefault_filter`: VC-POC-09 with 
peer_filters {"sectors": ["Catholic", "Independent"]}. Explicit filter check: 
filters non-empty; default_sectors = policy defaultable "peer_filters.sectors" =
["Catholic", "Independent"]; set(filters) == {"sectors"} and sorted equal → 
clarification ✓. Second part: absent → derive missing: required ["peer_filters"]
missing → clarification containing peer_filters ✓.

18. `test_explicit_peer_filter_dispatches`: peer_filters with sectors + 
school_types → constraint passes. Derive missing: peer_filters present. Validate
against VC-POC-09 schema — I haven't seen VC-POC-09.request.schema.json but per 
OD-07 it has minProperties ≥ 1 on peer_filters and nested additionalProperties 
false with keys including sectors, school_types, enrolments_min etc. Values: 
sectors ["Catholic","Independent"] (release-derived → no enum enforcement, 
likely just array of strings), school_types ["Secondary"] (release-derived). 
Should validate. Request dispatched exactly as given ✓.

19. `test_segment_filters_reject_unknown_nested_key`: segment_filters 
{"sectors": ["Independent"], "banana": ["x"]} → schema validation fails on 
additionalProperties false → TerminalError ✓. But wait — the explicit filter 
constraint runs BEFORE derivation and validation. segment_filters non-empty → no
clarification. Derive missing: segment_filters present → []. Validation → 
unknown "banana" → TerminalError ✓. fetched == [] ✓.

20. `test_empty_segment_filters_clarify`: segment_filters {} → explicit filter 
constraint: non-dict? It's a dict, empty → clarification ✓.

21. `test_unknown_request_property_fails_before_fetch`: VC-POC-01 filters 
{"acara_sml_id": 40387, "enrolments_forecast": True} → validation fails 
(additionalProperties false) → TerminalError ✓.

22. `test_schema_valid_emitter_bad_request_is_terminal`: VC-POC-01, mention 
resolves 40387, request {"acara_sml_id": 40387}. identity_from_caller False → 
bad_request → TerminalError containing "VC-POC-01" ✓. tool_calls recorded ✓. 
resolved_school_ids == [40387] (committed since resolution succeeded) ✓.

23. `test_invalid_explicit_identity_bad_request_clarifies`: VC-POC-01 explicit 
99999 → identity_from_caller True → bad_request → ClarificationNeeded ✓. 
tool_calls recorded ✓.

24. test_run_cli.py `test_main_uses_pipeline_path_by_default`: monkeypatches 
cli.converse, cli.require_credentials, cli.build_client; main(["--stub", 
"question"]). In my new run.py, main calls `require_credentials()`, `client = 
build_client(args.stub)`, `deps = Deps(client=client)`, then `converse(None, 
deps, args.question, ...)`. The fake converse captures agent=None, question. 
Returns Unsupported → EXIT_BY_BRANCH[Unsupported] == EXIT_UNSUPPORTED == 3 ✓.

Wait — main calls `asyncio.run(converse(None, deps, ...))`. My fake_converse is 
async — returns coroutine, asyncio.run handles it. ✓. But 
`build_parser().parse_args(["--stub", "..."])` — question positional ✓.

`test_main_no_longer_builds_a_legacy_agent`: source check "build_agent" not in 
run.py. My new run.py imports `from harness.agent import ask` only — no 
build_agent. But check the docstring: I wrote "no legacy agent is constructed or
passed" — does the string "build_agent" appear anywhere? Docstring says "It runs
the **deterministic four-stage pipeline by default**: no legacy agent is 
constructed or passed". No "build_agent" substring. Comments: "no legacy agent 
is built or passed. `converse` still accepts an explicit agent". No 
"build_agent". ✓

Hmm wait, one more check in run.py — the original had `from harness.agent import
ask, build_agent`. My version: `from harness.agent import ask`. ✓

25. test_run_cli.py `answered()` still uses 
`build_agent(model=replay_mod.scripted_model(...))` and `ask(agent, deps, ...)` 
— legacy path, imports build_agent from harness.agent in the test file. Fine.

26. One concern in test_run_cli.py: 
`test_one_clarification_round_trip_answers_the_question` uses legacy converse 
with agent — unchanged. ✓

Now — the user asks "Let me know if there are others we should add." and reminds
about the file listing format.

Everything verifies. But wait — one thing I should double-check: in 
`_ask_pipeline` in agent.py, after `plan_and_execute` returns a 
ClarificationNeeded or Unsupported, `done(result)` is called — fine. And 
TerminalError propagates. In test_intent_boundary 
`test_planner_failure_is_not_unsupported`, `ask(deps, "q", 
intent_override=intent)` — my planner raises TerminalError inside 
`asyncio.wait_for(plan_and_execute(...))`. wait_for propagates the exception. 
Not caught by `except SpendExceeded` or `except asyncio.TimeoutError` → 
propagates out of _ask_pipeline → out of ask → pytest.raises(TerminalError) ✓. 
And deps.last_envelope is None ✓.

Also check: harness/agent.py `_ask_pipeline` calls `plan_and_execute(intent, 
deps.client, deps)` — my signature `plan_and_execute(intent, client, deps)` ✓.

One more subtle check in the planner: `from harness.evidence import 
EvidenceClient, EvidenceError` — EvidenceClient is used only in type annotation 
of `_resolve_mentions`. Fine.

Also `_SEPARATOR_RE` on the registry condition: registry VC-POC-05 
conditional_material: `{"distance_km": "market_definition equals distance"}`. 
Normalized: "market definition equals distance" — `_SEPARATOR_RE.sub(" ", 
"market_definition equals distance".casefold())` → underscores are \w? Wait — 
`[\W_]+` includes underscore explicitly. "market_definition equals 
distance".casefold() → "market_definition equals distance"; sub [\W_]+ → "market
definition equals distance". ✓ In _KNOWN_DISTANCE_CONDITIONS ✓.

Registry strings: VC-POC-05 "market_definition equals distance", VC-POC-06/07 
same. ✓

For the new intent.py — already verified earlier. outputs.py — verified.

So: no changes needed. The user's instruction says "To suggest changes to a file
you MUST return the entire content..." — but if no changes are needed, I should 
say so without file listings (as I did last turn, where I made one small 
change). Actually last turn I made a change. This turn — do I need any changes?

Let me think hard about whether anything in outputs.py or intent.py forces a 
change.

One thing: `MissingInput.allowed_values: list[str]`. In 
`_static_vocabulary_for_field` for grades, the enum values are strings ✓. For 
metrics strings ✓. All enums are strings. ✓

`ClarificationNeeded.missing` — no min length validator. Fine.

`Unsupported.reason_kind` Literal includes "out_of_scope" — my planner no longer
produces out_of_scope, but doesn't matter.

One thing in test_planner.py — `test_vc_poc_03_routes_selector_to_ratios`: 
labels ["Students per Teaching FTE"] → normalize "students per teaching fte" → 
matches "students_per_teaching_fte" → canonical. Request {"acara_sml_ids": 
[40387, 40392], "ratios": ["students_per_teaching_fte"]}. Validation against 
VC-POC-03 schema: ratios enum contains it ✓, uniqueItems ✓, minItems 1 ✓. ✓ 
"metrics" not in sent ✓.

`test_grade_labels_route_to_grades_field`: VC-POC-02, "Year 6" → normalize "year
6" → grades enum "year_6" normalized "year 6" → match ✓. Request 
{"acara_sml_id": 40387, "grades": ["year_6"]} ✓.

Also check `_selector_route` for VC-POC-02: metric_list null. field mapping 
sorted keys containing "VC-POC-02.": "VC-POC-02.grades": "grades" → field 
"grades" ∈ _SELECTOR_BAG_FIELDS ✓, static enum grades ✓ → ("grades", [...]). ✓

For VC-POC-11: metric_list "student_context_metrics" → registry metric_lists ✓.

For VC-POC-04: metric_list "profile_metrics" ✓.

Now — is there anything about `test_selector_labels_without_a_route_clarify`? 
VC-POC-05 metric_list null; field mapping keys with "VC-POC-05." prefix: 
"VC-POC-05.market_definition": "market_definitions" → field "market_definition" 
not in _SELECTOR_BAG_FIELDS → continue. No other VC-POC-05 keys → route None → 
clarification with input="metrics". fetched == [] ✓ (clarification before 
fetch).

OK. Everything is consistent. No file changes needed this turn. I should reply 
confirming verification, and suggest the remaining optional files: the remaining
pinned request schemas (VC-POC-05/06/07/08/09/10/11/12/13/14/REF-01/REF-02), 
harness/answer_view.py, harness/narration.py, harness/evidence.py maybe. 
Actually harness/evidence.py was summarized (function signatures visible) — 
EvidenceClient(python=..., emitter=..., **kwargs) with env kwarg — visible from 
test usage. harness/narration.py signatures shown in summary — `narrate(view, 
deps, model=...)` and `_validate_narration(view, deps, narration)` — my tests 
call ask() which handles that; tests don't call narrate directly except via ask.
Fine.

harness/answer_view.py: `build_answer_view(envelope)` — my planner doesn't call 
it (agent.py does). Tests assert deps.answer_view type. Fine.

So the remaining most valuable additions: the remaining per-question request 
schemas, so I can statically confirm the validator handles every keyword 
(VC-POC-09/10's nested minProperties, VC-POC-08's distance_km bounds — whether 
exclusiveMinimum/ exclusiveMaximum are used per OD-04 "positive, finite 
number"). OD-04 says "positive, finite number with no arbitrary upper bound" — 
that could be encoded as `exclusiveMinimum: 0` which my validator does NOT 
check. My planner doesn't add distance_km itself (it's caller-supplied), so a 
distance_km of 0 or negative would pass my validator but be rejected by the real
emitter as bad_request → since identity_from_caller may be False (market 
questions with directory-resolved identity), that would raise TerminalError — 
planner drift! Hmm wait. Let me think.

VC-POC-08: required [acara_sml_id, distance_km]. If a user gives explicit ID + 
distance_km=0: my validation — if schema uses exclusiveMinimum: 0, my validator 
ignores it → passes → dispatch → real emitter returns bad_request → 
identity_from_caller True (explicit acara_sml_id) → clarification. OK in that 
case.

But VC-POC-05/06/07 distance markets with directory-resolved ID and 
distance_km=-5: my validator passes (if exclusiveMinimum unhandled), dispatch → 
bad_request → identity_from_caller False → TerminalError. Is that correct 
behaviour? The plan says: "A caller-supplied numeric-ID case... otherwise raise 
TerminalError because the result proves planner/contract drift." But a negative 
radius is a *caller-supplied value* problem, not planner drift. Hmm. But the 
contract's failure_behaviour says: "Any other locally validated deterministic 
request receives bad_request → Record the failed fetch invocation and raise 
TerminalError containing the question ID and emitter context." So per the 
contract, raising TerminalError for a locally-validated request that the emitter
still rejects IS the specified behaviour — the ambiguity is whether the 
caller-supplied *value* (not identity) should clarify. The contract only carves 
out the numeric-identity case ("A locally validated request containing only a 
caller-supplied numeric ACARA identity receives bad_request → identity 
clarification"). So negative distance_km → TerminalError per contract. That's 
what my implementation does. And whether the schema uses exclusiveMinimum 
doesn't change my implementation — if it does, my validator under-checks and the
emitter's bad_request → TerminalError per contract. The contract's specified 
behaviour is TerminalError anyway. So adding exclusiveMinimum support would 
change behaviour: with support, distance_km=0 would fail validation → 
TerminalError before dispatch (instead of after). Both are TerminalError — same 
outcome class, just earlier. Not required. But to be thorough, I could add 
exclusiveMinimum/exclusiveMaximum support so the planner catches shape 
violations before dispatch as the plan demands ("types, bounds, 
uniqueness/cardinality, conditionals, and formats" — "bounds" includes exclusive
bounds arguably). The plan says validation must reject "wrong types, bounds, 
uniqueness/cardinality, conditional failures, and formats". If the schema 
encodes the distance bound as exclusiveMinimum, then to fully honour "bounds" I 
should support exclusiveMinimum/exclusiveMaximum. I'll add that support — it's a
small, safe addition (only tightens validation; can't wrongly accept, and only 
affects schemas that use those keywords).

Wait, but if no supplied schema uses exclusiveMinimum, adding support is 
harmless. If VC-POC-08 uses it, support makes validation catch it pre-dispatch —
consistent with "fail before dispatch" acceptance id 
("schema-and-filter-fail-closed"). I'll add it.

Also multipleOf — could appear? Unlikely. format keyword — "formats" mentioned; 
unlikely in these schemas (formats in JSON Schema are annotations by default 
anyway).

So: I'll produce an updated harness/planner.py adding 
exclusiveMinimum/exclusiveMaximum handling (and also treat `maximum`/`minimum` 
when value is bool — already guarded).

Should I also double check `_derive_missing_inputs` for VC-POC-12? required 
["sa2_codes"], material ["sa2_codes"]... registry says material: ["sa2_codes"] 
for VC-POC-12? Registry VC-POC-12 input_policy material: ["sa2_codes"]. Yes 
(from registry: "material": ["sa2_codes"]). Hmm wait, actually looking at 
registry VC-POC-12: "required": ["sa2_codes"], "material": ["sa2_codes"]. So a 
VC-POC-12 request without sa2_codes → clarification ["sa2_codes"], 
allowed_values from mapping "VC-POC-12.seifa_index_type" etc. — sa2_codes has no
mapping → []. Fine.

VC-POC-13: required [sa2_codes], material [sa2_codes]. Fine.

VC-POC-REF-01: required [] → {} request passes schema? REF-01 request schema 
likely empty object allowing {}. ✓ (golden case expected_request {}).

VC-POC-REF-02: required [zone_entity_code]; material [] — wait registry says 
material: [] for REF-02? Registry: "material": [], required: 
["zone_entity_code"]. So missing zone_entity_code → clarification (required). 
Fine.

Now the reflection: should I re-emit planner.py with exclusive bound support? 
Yes — the user asked me to verify and make any resulting adjustments in one 
pass. The change is small: in the numeric bounds block, add:

```python
exclusive_minimum = schema.get("exclusiveMinimum")
if isinstance(exclusive_minimum, (int, float)) and value <= exclusive_minimum:
    problems.append(f"{path}: must be greater than {exclusive_minimum}")
exclusive_maximum = schema.get("exclusiveMaximum")
if isinstance(exclusive_maximum, (int, float)) and value >= exclusive_maximum:
    problems.append(f"{path}: must be less than {exclusive_maximum}")
```

Note: draft 2020-12 exclusiveMinimum is a number (not boolean as in draft 4). 
Handle number form.

Also should I handle `multipleOf`? Plan mentions formats/bounds; multipleOf 
unlikely in generated schemas. I could add it cheaply:

```python
multiple_of = schema.get("multipleOf")
if isinstance(multiple_of, (int, float)) and multiple_of > 0:
    quotient = value / multiple_of
    if abs(quotient - round(quotient)) > 1e-9:
        problems.append(...)
```

I'll skip multipleOf — keep to what's plausibly used; the under-check direction 
is safe. Actually, adding it is trivial and harmless... but floating point 
modulo issues could cause false rejections (mis-rejecting), which is the unsafe 
direction. Skip it.

Also for strings: `format` — annotations in 2020-12, skip.

Now, are there other schema keywords the generated v0.3 schemas might use that I
should worry about? From the samples seen: type, properties, required, 
additionalProperties, items, enum, minItems, maxItems, uniqueItems, minimum, 
maximum, pattern, minLength, description, $schema, $id, title, allOf/if/then (in
error schema, not request). x-poc annotations ignored. const (error schema). 
minProperties (VC-POC-09/10 per OD-07). That's all covered.

So final answer: confirm intent.py and outputs.py verified; one adjustment to 
harness/planner.py (exclusive bounds support); no other files need changes; 
remaining optional additions: the 12 remaining request schemas.

Let me write the full planner.py file with the exclusive bound addition. The 
rest stays identical to the last version (with contract.CONTRACT_DIR). I must 
output the ENTIRE file.

Let me also reconsider: in the numeric block, the guard is `if isinstance(value,
(int, float)) and not isinstance(value, bool):`. Add exclusive checks there.

Write out the full file now. Everything else identical to previous listing. I 
need to be careful to reproduce it exactly with only the bounds addition.

------------
► **ANSWER**

Both files verify cleanly against the implementation — one small hardening 
change to `harness/planner.py` below; nothing else needs to change.



**`harness/intent.py`** — `StructuredIntent`'s seven fields (`question_id`, 
`excluded_topic`, `school_mentions`, `requested_metrics`, `filters`, 
`presentation`, `missing_material_inputs`) match every construction and 
assertion in the planner and all three test files; `interpret_intent(question, 
model=...)` and the `request_limit=1` → `RuntimeError` behaviour match 
`test_interpret_intent_enforces_request_limit`.



**`harness/outputs.py`** — 
`ClarificationNeeded`/`MissingInput`/`Unsupported`/`RenderedAnswer`/`BudgetExcee
ded` signatures all match my constructor calls; `MissingInput.allowed_values: 
list[str]` matches the pinned string enums I pass (`market_definitions`, 
`profile_metrics`, `ratios`, `grades`); `Unsupported.reason_kind` still includes
`"out_of_scope"` so nothing breaks for the legacy loop even though the planner 
no longer produces it.



**The one adjustment**: the plan's validation clause requires rejecting "wrong 
types, bounds, uniqueness/cardinality, conditionals, and formats". My validator 
handled `minimum`/`maximum` but not `exclusiveMinimum`/`exclusiveMaximum` — 
which is exactly how OD-04's "positive, finite number" rule for `distance_km` is
likely encoded in `VC-POC-08` (a schema I haven't seen). Without this, a 
zero/negative radius would pass pre-dispatch validation and only surface as an 
emitter `bad_request` — the same `TerminalError` outcome but *after* dispatch 
instead of before, weakening the "schema-and-filter-fail-closed" guarantee. 
Adding draft-2020-12 numeric-form exclusive bounds tightens validation 
pre-dispatch and cannot mis-reject any currently-valid request.



harness/planner.py

```diff
@@ -1,38 +1,66 @@
 """Stage 2 of the 4-stage pipeline: deterministic planning and execution.
 
 `plan_and_execute()` takes a StructuredIntent and returns either:
 - a dict (the evidence envelope) — the happy path
-- ClarificationNeeded — a material input is missing
+- ClarificationNeeded — an input the caller must supply or correct
 - Unsupported — an exclusion or an unknown question
 
-No model calls. Every branch is a pure function of the intent and the contract.
-The evidence call itself is the only I/O, and its error handling mirrors the
-tool layer's `_branch_on_error_kind()` exactly: the same closed set of 
outcomes,
-the same reason codes, the same contract text.
-
-Why reproduce `_branch_on_error_kind` here rather than calling it? Because the
-tool layer raises `ModelRetry`, `UnsupportedSignal`, and `RepairCapExceeded` —
-exceptions designed for PydanticAI's agent loop. The pipeline has no loop, so
-those signals have no handler; they would escape to the caller as unclassified
-RuntimeErrors. The planner translates the same error semantics into the 
pipeline's
-typed return values instead.
+No model calls. Every branch is a pure function of the intent, the pinned
+contract artefacts, and the evidence client's responses.
+
+What Stage 2 now guarantees (work item 20260908T113118):
+
+- Required, material, and applicable conditional-material inputs are derived
+  from the pinned question policy and the assembled request. The intent
+  model's `missing_material_inputs` claim is never trusted.
+- Named schools resolve through the pinned directory and must match exactly
+  one record each. IDs are staged in mention order, deduplicated without
+  reordering, and committed to `Deps` only after every mention resolves; a
+  zero or ambiguous match clarifies with no evidence call and no partial
+  identity mutation. Caller-supplied numeric IDs bypass the preflight (the
+  bounded resolution of the pinned Definition-Pack conflict) and any conflict
+  with name-resolved identities fails closed before dispatch.
+- Selector labels are Unicode-case-folded and separator-normalized, matched
+  against pinned static vocabularies (registry `metric_list` where declared;
+  otherwise the input-catalog schema's field mapping and embedded enums), and
+  routed to the question's own request field: `metrics` for VC-POC-01/04/11,
+  `ratios` for VC-POC-03, `grades` for VC-POC-02/07. Unsupplied selectors are
+  omitted — the emitter keeps default ownership (no invented defaults).
+- Every request is validated against its registered pinned per-question
+  schema before dispatch. A planner-produced request that cannot satisfy its
+  schema raises TerminalError — planner defects are never dressed up as
+  Unsupported user outcomes.
+- Error origins are separated: a bad_request against a request whose school
+  identity was caller-supplied is an identity clarification; any other
+  bad_request after deterministic validation is planner/contract drift and
+  raises TerminalError. excluded/unknown/unanswerable keep their typed
+  abstentions; data_unavailable/internal remain operational stops.
 """
 
 from __future__ import annotations
 
+import json
 import logging
+import re
+from functools import lru_cache
+from pathlib import Path
 from typing import Any
 
 from harness import contract
 from harness.errors import TerminalError
 from harness.evidence import EvidenceClient, EvidenceError
 from harness.intent import StructuredIntent
 from harness.outputs import ClarificationNeeded, MissingInput, Unsupported
 from harness.tools import Deps, ToolInvocation, request_scope_problem
 
 logger = logging.getLogger("harness.planner")
+
+# Pinned contract artefacts. Static vocabularies and per-question schemas are
+# read from the vendored, hash-pinned directory `harness.contract` reads —
+# never from the runtime emitter.
+_SCHEMA_DIR = contract.CONTRACT_DIR / "question_schemas" / "v0.3.0"
 
 # Keyword hints the intent model may use for excluded topics, mapped to a 
search
 # term that appears in the corresponding contract exclusion text. The mapping 
is
 # fuzzy: the planner searches for any exclusion text that contains the keyword.
 _EXCLUSION_KEYWORD_MAP: dict[str, str] = {
@@ -47,10 +75,32 @@
     "mesh_block": "Mesh Block",
     "sa2": "SA2",
     "gtfs": "GTFS",
 }
 
+# The one pinned semantic alias (prompts/rules.md): for school-profile
+# requests "teaching staff" means teaching_fte. Keyed by the normalized label,
+# so "Teaching Staff" and "teaching-staff" match the same alias.
+_PINNED_SELECTOR_ALIASES: dict[str, str] = {"teaching staff": "teaching_fte"}
+
+_SCHOOL_FIELDS = ("acara_sml_id", "acara_sml_ids")
+
+# Label-selector request fields the requested label bag can be routed into.
+_SELECTOR_BAG_FIELDS = ("metrics", "ratios", "grades")
+
+# The exact pinned conditional-material expressions (VC-POC-05/06/07), compared
+# after separator normalization. Anything else is contract drift, not prose to
+# parse heuristically.
+_KNOWN_DISTANCE_CONDITIONS = frozenset(
+    {
+        "market definition equals distance",
+        "market definition distance",
+    }
+)
+
+_SEPARATOR_RE = re.compile(r"[\W_]+", re.UNICODE)
+
 
 def _find_exclusion_text(excluded_topic: str | None) -> str:
     """Find the contract exclusion text matching a topic hint.
 
     Searches the exclusion texts for a case-insensitive substring match on the
@@ -77,23 +127,736 @@
 
     # Fallback: first exclusion text alphabetically
     return texts[0]
 
 
+# --------------------------------------------------------------------------- #
+# Pinned artefact loading (fail-closed: load failures are TerminalError)
+# --------------------------------------------------------------------------- #
+
+
+def _load_pinned_json(path: Path) -> Any:
+    try:
+        return json.loads(path.read_text(encoding="utf-8"))
+    except (OSError, ValueError) as exc:
+        raise TerminalError(
+            f"pinned contract artefact could not be loaded ({path.name}): 
{exc}"
+        ) from exc
+
+
+@lru_cache(maxsize=1)
+def _schema_index() -> dict[str, Any]:
+    index = _load_pinned_json(_SCHEMA_DIR / "_index.json")
+    if not isinstance(index, dict) or not isinstance(index.get("questions"), 
dict):
+        raise TerminalError("pinned question-schema index is malformed")
+    return index
+
+
+@lru_cache(maxsize=1)
+def _input_catalog_schema() -> dict[str, Any]:
+    schema = _load_pinned_json(contract.CONTRACT_DIR / 
"poc_input_catalog.v0.3.0.schema.json")
+    if not isinstance(schema, dict):
+        raise TerminalError("pinned input-catalog schema is malformed")
+    return schema
+
+
+def _registered_request_schema_path(question_id: str) -> Path:
+    """Resolve the question's registered request-schema path.
+
+    Only paths registered in the pinned index, beneath the pinned contract
+    directory, may be loaded — anything else is contract drift.
+    """
+    entry = _schema_index()["questions"].get(question_id)
+    if not isinstance(entry, dict):
+        raise TerminalError(f"{question_id} has no registered request schema")
+    filename = entry.get("request_schema")
+    if not isinstance(filename, str) or filename != 
f"{question_id}.request.schema.json":
+        raise TerminalError(
+            f"{question_id} request-schema registration is inconsistent: 
{filename!r}"
+        )
+    path = (_SCHEMA_DIR / filename).resolve()
+    pinned = contract.CONTRACT_DIR.resolve()
+    if pinned not in path.parents:
+        raise TerminalError(
+            "refusing to load a request schema from outside the pinned contract
"
+            f"directory: {filename}"
+        )
+    return path
+
+
+@lru_cache(maxsize=None)
+def _request_schema(question_id: str) -> dict[str, Any]:
+    schema = _load_pinned_json(_registered_request_schema_path(question_id))
+    if not isinstance(schema, dict):
+        raise TerminalError(f"{question_id} request schema is malformed")
+    return schema
+
+
+# --------------------------------------------------------------------------- #
+# Static vocabularies (registry metric_list; catalog field mapping + enums)
+# --------------------------------------------------------------------------- #
+
+
+@lru_cache(maxsize=1)
+def _field_to_catalog_list() -> dict[str, str]:
+    mapping = 
_input_catalog_schema().get("x-poc-request-field-to-catalog-list")
+    if not isinstance(mapping, dict):
+        return {}
+    return {
+        str(field): str(list_name)
+        for field, list_name in mapping.items()
+        if isinstance(field, str) and isinstance(list_name, str)
+    }
+
+
+def _static_enum(list_name: str) -> list[str] | None:
+    """The embedded static enum for a catalog list, or None when the list is
+    release-derived (shape-checked only, never enum-enforced — OD-03)."""
+    if list_name.startswith("release_derived_values."):
+        return None
+    prop = _input_catalog_schema().get("properties", {}).get(list_name)
+    if isinstance(prop, dict) and isinstance(prop.get("items"), dict):
+        enum = prop["items"].get("enum")
+        if isinstance(enum, list):
+            return [str(value) for value in enum]
+    return None
+
+
+def _static_vocabulary_for_field(question_id: str, field: str) -> list[str] | 
None:
+    list_name = _field_to_catalog_list().get(f"{question_id}.{field}")
+    if list_name is None:
+        return None
+    return _static_enum(list_name)
+
+
+def _selector_route(question_id: str) -> tuple[str, list[str]] | None:
+    """(request field, static vocabulary) for the question's label-selector 
bag.
+
+    The registry's `metric_list` is authoritative where declared (VC-POC-01/04
+    → profile_metrics, VC-POC-11 → student_context_metrics); otherwise the
+    input-catalog schema's field mapping routes e.g. VC-POC-03 to `ratios` and
+    VC-POC-02/07 to `grades`.
+    """
+    entry = contract.questions().get(question_id) or {}
+    metric_list = entry.get("metric_list")
+    if metric_list:
+        values = contract.registry().get("metric_lists", {}).get(metric_list)
+        if not isinstance(values, list) or not values:
+            raise TerminalError(
+                f"{question_id} declares metric_list {metric_list!r} with no 
pinned values"
+            )
+        return "metrics", [str(value) for value in values]
+    field_to_list = _field_to_catalog_list()
+    for key in sorted(field_to_list):
+        prefix = f"{question_id}."
+        if not key.startswith(prefix):
+            continue
+        field = key[len(prefix):]
+        if field in _SELECTOR_BAG_FIELDS:
+            vocabulary = _static_enum(field_to_list[key])
+            if vocabulary is None:
+                raise TerminalError(
+                    f"{question_id}.{field} routes to a non-static vocabulary"
+                )
+            return field, vocabulary
+    return None
+
+
+# --------------------------------------------------------------------------- #
+# Label normalization
+# --------------------------------------------------------------------------- #
+
+
+def _normalize_label(label: Any) -> str:
+    """Unicode-safe case folding plus separator normalization.
+
+    `total_enrolments`, `Total Enrolments` and `total-enrolments` all fold to
+    `total enrolments`; Unicode letters survive the fold.
+    """
+    folded = _SEPARATOR_RE.sub(" ", str(label).casefold())
+    return folded.strip()
+
+
+def _canonical_for(label: Any, vocabulary: list[str]) -> str | None:
+    """Match a label against a pinned static vocabulary.
+
+    Direct normalized matches win; beyond that only the pinned semantic
+    aliases apply. Canonical values are returned unchanged.
+    """
+    wanted = _normalize_label(label)
+    for value in vocabulary:
+        if _normalize_label(value) == wanted:
+            return value
+    alias = _PINNED_SELECTOR_ALIASES.get(wanted)
+    if alias is not None and alias in vocabulary:
+        return alias
+    return None
+
+
+def _dedupe(values: list[Any]) -> list[Any]:
+    """Order-preserving deduplication."""
+    seen: set[Any] = set()
+    unique: list[Any] = []
+    for value in values:
+        if value not in seen:
+            seen.add(value)
+            unique.append(value)
+    return unique
+
+
+# --------------------------------------------------------------------------- #
+# Pinned-schema validation (draft 2020-12 subset, dependency-free)
+# --------------------------------------------------------------------------- #
+
+
+def _json_key(value: Any) -> str:
+    return json.dumps(value, sort_keys=True, ensure_ascii=False, default=str)
+
+
+def _type_matches(value: Any, expected: str) -> bool:
+    if expected == "object":
+        return isinstance(value, dict)
+    if expected == "array":
+        return isinstance(value, list)
+    if expected == "string":
+        return isinstance(value, str)
+    if expected == "integer":
+        if isinstance(value, bool):
+            return False
+        if isinstance(value, int):
+            return True
+        return isinstance(value, float) and value.is_integer()
+    if expected == "number":
+        return isinstance(value, (int, float)) and not isinstance(value, bool)
+    if expected == "boolean":
+        return isinstance(value, bool)
+    if expected == "null":
+        return value is None
+    return True  # unknown type keyword: leave to the remaining checks
+
+
+def _schema_problems(value: Any, schema: Any, path: str) -> list[str]:
+    """Validate `value` against the pinned request schema.
+
+    Supports the draft-2020-12 keywords the generated v0.3 request schemas
+    use: types, required/properties/additionalProperties, items, enum/const,
+    inclusive and exclusive numeric bounds, cardinality, uniqueness, patterns,
+    string lengths, and the allOf/anyOf/oneOf and if/then/else combinators.
+    Annotation keywords are ignored.
+    """
+    problems: list[str] = []
+    if not isinstance(schema, dict):
+        return problems
+
+    expected_type = schema.get("type")
+    if expected_type is not None:
+        expected_types = expected_type if isinstance(expected_type, list) else 
[expected_type]
+        if not any(_type_matches(value, str(t)) for t in expected_types):
+            problems.append(
+                f"{path}: expected type {expected_type!r}, got 
{type(value).__name__}"
+            )
+            return problems
+
+    if "const" in schema and _json_key(value) != _json_key(schema["const"]):
+        problems.append(f"{path}: must equal {schema['const']!r}")
+
+    enum = schema.get("enum")
+    if isinstance(enum, list):
+        if not any(_json_key(value) == _json_key(candidate) for candidate in 
enum):
+            problems.append(f"{path}: {value!r} is not an allowed value")
+
+    if isinstance(value, dict):
+        properties = schema.get("properties")
+        properties = properties if isinstance(properties, dict) else {}
+        for key, sub_schema in properties.items():
+            if key in value:
+                problems.extend(_schema_problems(value[key], sub_schema, 
f"{path}.{key}"))
+        additional = schema.get("additionalProperties", True)
+        unknown = [key for key in value if key not in properties]
+        if unknown and additional is False:
+            problems.append(f"{path}: unknown properties {sorted(unknown)}")
+        elif isinstance(additional, dict):
+            for key in unknown:
+                problems.extend(_schema_problems(value[key], additional, 
f"{path}.{key}"))
+        required = schema.get("required")
+        if isinstance(required, list):
+            for key in required:
+                if key not in value:
+                    problems.append(f"{path}: missing required property 
{key!r}")
+        min_properties = schema.get("minProperties")
+        if isinstance(min_properties, int) and len(value) < min_properties:
+            problems.append(f"{path}: needs at least {min_properties} 
properties")
+        max_properties = schema.get("maxProperties")
+        if isinstance(max_properties, int) and len(value) > max_properties:
+            problems.append(f"{path}: allows at most {max_properties} 
properties")
+
+    if isinstance(value, list):
+        min_items = schema.get("minItems")
+        if isinstance(min_items, int) and len(value) < min_items:
+            problems.append(f"{path}: needs at least {min_items} items")
+        max_items = schema.get("maxItems")
+        if isinstance(max_items, int) and len(value) > max_items:
+            problems.append(f"{path}: allows at most {max_items} items")
+        if schema.get("uniqueItems"):
+            item_keys = [_json_key(item) for item in value]
+            if len(set(item_keys)) != len(item_keys):
+                problems.append(f"{path}: items must be unique")
+        items = schema.get("items")
+        if isinstance(items, dict):
+            for index, element in enumerate(value):
+                problems.extend(_schema_problems(element, items, 
f"{path}[{index}]"))
+
+    if isinstance(value, str):
+        pattern = schema.get("pattern")
+        if isinstance(pattern, str) and re.search(pattern, value) is None:
+            problems.append(f"{path}: does not match the required pattern")
+        min_length = schema.get("minLength")
+        if isinstance(min_length, int) and len(value) < min_length:
+            problems.append(f"{path}: shorter than the required minimum 
length")
+        max_length = schema.get("maxLength")
+        if isinstance(max_length, int) and len(value) > max_length:
+            problems.append(f"{path}: longer than the required maximum length")
+
+    if isinstance(value, (int, float)) and not isinstance(value, bool):
+        minimum = schema.get("minimum")
+        if isinstance(minimum, (int, float)) and value < minimum:
+            problems.append(f"{path}: below the minimum {minimum}")
+        maximum = schema.get("maximum")
+        if isinstance(maximum, (int, float)) and value > maximum:
+            problems.append(f"{path}: above the maximum {maximum}")
+        # Draft 2020-12 exclusive bounds are numbers (not draft-4 booleans).
+        # This is how OD-04's "positive, finite number" rule for distance_km
+        # is expected to be encoded; catching it here fails closed before
+        # dispatch instead of after.
+        exclusive_minimum = schema.get("exclusiveMinimum")
+        if isinstance(exclusive_minimum, (int, float)) and value <= 
exclusive_minimum:
+            problems.append(f"{path}: must be greater than 
{exclusive_minimum}")
+        exclusive_maximum = schema.get("exclusiveMaximum")
+        if isinstance(exclusive_maximum, (int, float)) and value >= 
exclusive_maximum:
+            problems.append(f"{path}: must be less than {exclusive_maximum}")
+
+    def _check(sub_schema: Any) -> list[str]:
+        return _schema_problems(value, sub_schema, path)
+
+    all_of = schema.get("allOf")
+    if isinstance(all_of, list):
+        for sub_schema in all_of:
+            problems.extend(_check(sub_schema))
+    any_of = schema.get("anyOf")
+    if isinstance(any_of, list) and any_of:
+        if all(len(_check(sub_schema)) > 0 for sub_schema in any_of):
+            problems.append(f"{path}: matches none of the allowed variants")
+    one_of = schema.get("oneOf")
+    if isinstance(one_of, list) and one_of:
+        matches = sum(1 for sub_schema in one_of if not _check(sub_schema))
+        if matches != 1:
+            problems.append(
+                f"{path}: must match exactly one allowed variant (matched 
{matches})"
+            )
+    if "if" in schema:
+        condition_holds = not _check(schema["if"])
+        branch = schema.get("then") if condition_holds else schema.get("else")
+        if isinstance(branch, dict):
+            problems.extend(_check(branch))
+    not_schema = schema.get("not")
+    if isinstance(not_schema, dict) and not _check(not_schema):
+        problems.append(f"{path}: matches a forbidden variant")
+
+    return problems
+
+
+def _validate_request(question_id: str, request: dict[str, Any]) -> None:
+    """Validate the planner-built request against its registered pinned schema.
+
+    Any failure — including schema load or resolution failure — is a
+    TerminalError before dispatch, never an Unsupported user outcome.
+    """
+    schema = _request_schema(question_id)
+    problems = _schema_problems(request, schema, "$request")
+    if problems:
+        raise TerminalError(
+            f"{question_id}: the planner-built request does not satisfy its "
+            f"registered request schema: " + "; ".join(problems)
+        )
+
+
+# --------------------------------------------------------------------------- #
+# School identity resolution (transactional, exactly-one-match)
+# --------------------------------------------------------------------------- #
+
+
+async def _resolve_mentions(
+    question_id: str,
+    mentions: list[str],
+    client: EvidenceClient,
+    deps: Deps,
+) -> tuple[list[int], ClarificationNeeded | None]:
+    """Resolve each mention through the pinned directory, exactly one match 
each.
+
+    Every preflight is recorded, including failures. IDs are staged locally in
+    mention order and deduplicated without reordering; nothing is committed
+    here — the caller commits to `Deps` only after every mention resolves.
+    """
+    staged: list[int] = []
+    for mention in mentions:
+        arguments = {"search": mention, "limit": 25}
+        try:
+            records = await client.school_directory(search=mention, limit=25)
+        except Exception as exc:
+            deps.tool_calls.append(
+                ToolInvocation("search_schools", arguments, 
error_kind=type(exc).__name__)
+            )
+            raise
+        deps.tool_calls.append(ToolInvocation("search_schools", arguments, 
records))
+        ids = [
+            record["acara_sml_id"]
+            for record in records
+            if isinstance(record, dict)
+            and isinstance(record.get("acara_sml_id"), int)
+            and not isinstance(record["acara_sml_id"], bool)
+        ]
+        if not ids:
+            return [], _school_match_clarification(
+                question_id,
+                mention,
+                f"'{mention}' matched no school in the pinned directory.",
+                "I couldn't find that school. Could you check the name — or 
give me "
+                "the school's public ACARA ID?",
+            )
+        if len(ids) > 1:
+            return [], _school_match_clarification(
+                question_id,
+                mention,
+                f"'{mention}' matched {len(ids)} schools; a request needs 
exactly one.",
+                f"More than one school matches '{mention}'. Which one did you 
mean?",
+            )
+        school_id = ids[0]
+        if school_id not in staged:
+            staged.append(school_id)
+    return staged, None
+
+
+def _school_match_clarification(
+    question_id: str, mention: str, why_material: str, prompt_to_user: str
+) -> ClarificationNeeded:
+    return ClarificationNeeded(
+        question_id=question_id,
+        missing=[
+            MissingInput(input="acara_sml_id", why_material=why_material, 
allowed_values=[])
+        ],
+        prompt_to_user=prompt_to_user,
+    )
+
+
+def _adapt_explicit_school_fields(request: dict[str, Any], school_required: 
list[str]) -> None:
+    """Let a caller-supplied identity in either school field satisfy the
+    question's own selector field, without inventing values."""
+    if not school_required:
+        return
+    if "acara_sml_ids" in school_required:
+        if "acara_sml_ids" not in request and 
isinstance(request.get("acara_sml_id"), int):
+            request["acara_sml_ids"] = [request.pop("acara_sml_id")]
+    elif "acara_sml_id" in school_required:
+        if "acara_sml_id" not in request and 
isinstance(request.get("acara_sml_ids"), list):
+            ids = request["acara_sml_ids"]
+            if len(ids) == 1 and isinstance(ids[0], int) and not 
isinstance(ids[0], bool):
+                request["acara_sml_id"] = ids[0]
+                del request["acara_sml_ids"]
+            else:
+                raise TerminalError(
+                    "caller supplied multiple acara_sml_ids for a question 
whose schema "
+                    "takes a single acara_sml_id"
+                )
+
+
+def _attach_school_ids(
+    request: dict[str, Any], school_required: list[str], ids: list[int]
+) -> None:
+    if not ids:
+        return
+    if "acara_sml_ids" in school_required:
+        request.setdefault("acara_sml_ids", list(ids))
+    elif "acara_sml_id" in school_required:
+        request.setdefault("acara_sml_id", ids[0])
+
+
+def _explicit_id_values(request: dict[str, Any], school_required: list[str]) ->
list[int]:
+    if "acara_sml_ids" in school_required:
+        value = request.get("acara_sml_ids")
+        if isinstance(value, list):
+            return [v for v in value if isinstance(v, int) and not 
isinstance(v, bool)]
+        return []
+    if "acara_sml_id" in school_required:
+        value = request.get("acara_sml_id")
+        if isinstance(value, int) and not isinstance(value, bool):
+            return [value]
+    return []
+
+
+def _requires_two_distinct_ids(entry: dict[str, Any]) -> bool:
+    constraints = entry.get("input_policy", {}).get("constraints", [])
+    return any("at least two" in str(constraint) for constraint in constraints)
+
+
+def _insufficient_schools_clarification(question_id: str) -> 
ClarificationNeeded:
+    return ClarificationNeeded(
+        question_id=question_id,
+        missing=[
+            MissingInput(
+                input="acara_sml_ids",
+                why_material="This comparison needs at least two distinct 
schools.",
+                allowed_values=[],
+            )
+        ],
+        prompt_to_user="Which distinct schools should be compared? Name at 
least two.",
+    )
+
+
+# --------------------------------------------------------------------------- #
+# Selector normalization and routing
+# --------------------------------------------------------------------------- #
+
+
+def _normalize_scalar_selectors(
+    question_id: str, request: dict[str, Any]
+) -> ClarificationNeeded | None:
+    """Normalize static-closed scalar selectors (market_definition) against
+    their pinned vocabulary. Unnormalizable values clarify before dispatch."""
+    if "market_definition" not in request:
+        return None
+    value = request["market_definition"]
+    if not isinstance(value, str):
+        return None  # shape-checked by the pinned schema
+    vocabulary = _static_vocabulary_for_field(question_id, "market_definition")
+    if vocabulary is None:
+        return None
+    canonical = _canonical_for(value, vocabulary)
+    if canonical is None:
+        return ClarificationNeeded(
+            question_id=question_id,
+            missing=[
+                MissingInput(
+                    input="market_definition",
+                    why_material=(
+                        f"'{value}' is not an allowed market definition for 
{question_id}."
+                    ),
+                    allowed_values=vocabulary,
+                )
+            ],
+            prompt_to_user=(
+                "Which market definition should I use? Allowed values: "
+                + ", ".join(vocabulary)
+                + "."
+            ),
+        )
+    request["market_definition"] = canonical
+    return None
+
+
+def _route_selectors(
+    question_id: str,
+    intent: StructuredIntent,
+    request: dict[str, Any],
+    deps: Deps,
+) -> ClarificationNeeded | None:
+    """Route requested selector labels to the question's own request field.
+
+    Static vocabularies come from pinned artefacts (registry metric_list where
+    declared, otherwise the input-catalog schema's field mapping and embedded
+    enums) — never from the runtime emitter. Selectors the user named none of
+    are omitted so the emitter keeps default ownership.
+    """
+    labels = [str(label) for label in (intent.requested_metrics or []) if 
str(label).strip()]
+    if not labels:
+        return None
+    route = _selector_route(question_id)
+    if route is None:
+        return ClarificationNeeded(
+            question_id=question_id,
+            missing=[
+                MissingInput(
+                    input="metrics",
+                    why_material=(
+                        f"{question_id} takes no metric selector, so the 
requested "
+                        "labels cannot be applied to this request."
+                    ),
+                    allowed_values=[],
+                )
+            ],
+            prompt_to_user="This question does not take metric selectors. 
Should I answer it as asked?",
+        )
+    field, vocabulary = route
+    canonical: list[str] = []
+    for label in labels:
+        matched = _canonical_for(label, vocabulary)
+        if matched is None:
+            return ClarificationNeeded(
+                question_id=question_id,
+                missing=[
+                    MissingInput(
+                        input=field,
+                        why_material=(
+                            f"'{label}' is not an allowed {field} value for 
{question_id}."
+                        ),
+                        allowed_values=vocabulary,
+                    )
+                ],
+                prompt_to_user=(
+                    f"Which {field} did you mean? Allowed values: "
+                    + ", ".join(vocabulary)
+                    + "."
+                ),
+            )
+        canonical.append(matched)
+    canonical = _dedupe(canonical)
+
+    if field == "metrics":
+        # Existing raw-question scope rule: a user who never asked about
+        # headcounts or non-teaching staff gets neither, even if a label was
+        # requested. Applied before validation, as before.
+        scope_problem = request_scope_problem(
+            deps.user_question, question_id, {**request, "metrics": canonical}
+        )
+        if scope_problem:
+            cleaned = [
+                metric
+                for metric in canonical
+                if not (metric.endswith("_headcount") or 
metric.startswith("non_teaching_"))
+            ]
+            canonical = cleaned or canonical
+
+    request[field] = canonical
+    return None
+
+
+# --------------------------------------------------------------------------- #
+# Registry-owned explicit-filter constraints (VC-POC-09/10, OD-07)
+# --------------------------------------------------------------------------- #
+
+
+def _enforce_explicit_filter_constraints(
+    question_id: str, entry: dict[str, Any], request: dict[str, Any]
+) -> ClarificationNeeded | None:
+    """The registry-owned VC-POC-09/10 rule: a cohort definition must contain
+    at least one explicit filter — the sector default alone does not define a
+    peer group. Enforced exactly as pinned, not generalised into a condition
+    language. Absent fields are left to policy derivation."""
+    policy = entry.get("input_policy") or {}
+    required = [f for f in policy.get("required", []) if isinstance(f, str)]
+
+    if "peer_filters" in required and isinstance(request.get("peer_filters"), 
dict):
+        filters = request["peer_filters"]
+        if not filters:
+            return _cohort_definition_clarification(
+                question_id,
+                "peer_filters",
+                "The peer group definition is empty; at least one explicit 
filter "
+                "is required.",
+            )
+        default_sectors = policy.get("defaultable", 
{}).get("peer_filters.sectors")
+        if (
+            set(filters) == {"sectors"}
+            and isinstance(default_sectors, list)
+            and isinstance(filters.get("sectors"), list)
+            and sorted(str(s) for s in filters["sectors"])
+            == sorted(str(s) for s in default_sectors)
+        ):
+            return _cohort_definition_clarification(
+                question_id,
+                "peer_filters",
+                "The sector default alone does not define a peer group; at 
least one "
+                "explicit filter beyond the sector default is required.",
+            )
+
+    if "segment_filters" in required and 
isinstance(request.get("segment_filters"), dict):
+        if not request["segment_filters"]:
+            return _cohort_definition_clarification(
+                question_id,
+                "segment_filters",
+                "The segment definition is empty; at least one explicit filter 
"
+                "is required.",
+            )
+    return None
+
+
+def _cohort_definition_clarification(
+    question_id: str, field: str, why_material: str
+) -> ClarificationNeeded:
+    return ClarificationNeeded(
+        question_id=question_id,
+        missing=[MissingInput(input=field, why_material=why_material, 
allowed_values=[])],
+        prompt_to_user=(
+            "Which rule should define this cohort — for example a sector, 
school "
+            "type, enrolment band, ICSEA range, remoteness or suburb?"
+        ),
+    )
+
+
+# --------------------------------------------------------------------------- #
+# Derived missing inputs (pinned policy order)
+# --------------------------------------------------------------------------- #
+
+
+def _derive_missing_inputs(
+    question_id: str, entry: dict[str, Any], request: dict[str, Any]
+) -> list[str]:
+    """Missing required, material, then applicable conditional-material fields,
+    in pinned policy order — derived from policy and the assembled request,
+    never taken from the model's missing-input claim."""
+    policy = entry.get("input_policy") or {}
+    missing: list[str] = []
+    seen: set[str] = set()
+
+    def consider(field: str) -> None:
+        if field in seen:
+            return
+        seen.add(field)
+        if field not in request:
+            missing.append(field)
+
+    for field in policy.get("required", []):
+        if isinstance(field, str):
+            consider(field)
+    for field in policy.get("material", []):
+        if isinstance(field, str):
+            consider(field)
+
+    conditional = policy.get("conditional_material") or {}
+    if isinstance(conditional, dict):
+        for field, condition in conditional.items():
+            if not isinstance(field, str):
+                continue
+            normalized_condition = _SEPARATOR_RE.sub(" ", 
str(condition).casefold()).strip()
+            if field == "distance_km" and normalized_condition in 
_KNOWN_DISTANCE_CONDITIONS:
+                if request.get("market_definition") == "distance" and 
"distance_km" not in request:
+                    consider(field)
+            else:
+                raise TerminalError(
+                    f"{question_id}: unsupported conditional-material 
expression "
+                    f"{condition!r} for {field} — contract drift, refusing to 
interpret it"
+                )
+    return missing
+
+
+# --------------------------------------------------------------------------- #
+# Stage 2 entry point
+# --------------------------------------------------------------------------- #
+
+
 async def plan_and_execute(
     intent: StructuredIntent,
     client: EvidenceClient,
     deps: Deps,
 ) -> dict[str, Any] | ClarificationNeeded | Unsupported:
-    """Deterministic Stage 2: intent → evidence envelope or typed refusal.
+    """Deterministic Stage 2: intent → evidence envelope or typed outcome.
 
     Returns:
         dict — the evidence envelope (happy path)
-        ClarificationNeeded — a material input is missing
-        Unsupported — an exclusion, unknown question, or unanswerable request
+        ClarificationNeeded — a caller-suppliable input is missing or wrong
+        Unsupported — an exclusion or an unknown question
     May raise:
-        TerminalError — an operational stop (data_unavailable, internal)
+        TerminalError — planner/schema defects and operational stops
     """
     question_id = intent.question_id
 
     # ------------------------------------------------------------------ #
     # 1. Excluded topic (intent model classified it as out of scope)
@@ -157,112 +920,134 @@
             reason_kind="not_in_registry",
             reason=f"{question_id} is not a registry question.",
         )
 
     # ------------------------------------------------------------------ #
-    # 5. Material inputs missing → ask for clarification
-    # ------------------------------------------------------------------ #
-    if intent.missing_material_inputs:
-        material = contract.material_inputs_for(question_id)
-        actually_missing = [inp for inp in intent.missing_material_inputs if 
inp in material]
-        if actually_missing:
-            missing_items = [
-                MissingInput(
-                    input=inp,
-                    why_material=_material_why(question_id, inp),
-                    allowed_values=[],
+    # 5. Pinned policy and assembled request
+    # ------------------------------------------------------------------ #
+    entry = contract.questions().get(question_id)
+    if entry is None:
+        raise TerminalError(
+            f"{question_id} has a registered request schema but is absent from 
"
+            "the pinned registry — contract drift"
+        )
+    policy = entry.get("input_policy") or {}
+    required_fields = [f for f in policy.get("required", []) if isinstance(f, 
str)]
+    school_required = [f for f in required_fields if f in _SCHOOL_FIELDS]
+
+    # Fresh request from explicit intent fields only; planner-owned selectors
+    # are added below. Unsupplied defaultables stay omitted — the emitter owns
+    # defaults and resolved_request.
+    request: dict[str, Any] = dict(intent.filters)
+
+    scalar_clarification = _normalize_scalar_selectors(question_id, request)
+    if scalar_clarification is not None:
+        return scalar_clarification
+
+    _adapt_explicit_school_fields(request, school_required)
+    identity_from_caller = any(field in request for field in school_required)
+
+    # ------------------------------------------------------------------ #
+    # 6. Transactional named-school resolution
+    # ------------------------------------------------------------------ #
+    mentions = [
+        mention for mention in (intent.school_mentions or [])
+        if isinstance(mention, str) and mention.strip()
+    ]
+    if school_required and mentions:
+        resolved_ids, failure = await _resolve_mentions(question_id, mentions, 
client, deps)
+        if failure is not None:
+            return failure
+        if identity_from_caller:
+            explicit_ids = _explicit_id_values(request, school_required)
+            if sorted(resolved_ids) != sorted(explicit_ids):
+                raise TerminalError(
+                    f"{question_id}: explicit school identities {explicit_ids} 
conflict "
+                    f"with directory-resolved identities {resolved_ids}"
                 )
-                for inp in actually_missing
-            ]
-            prompt_to_user = (
-                f"To answer this question I need "
-                + ", ".join(f"the {inp}" for inp in actually_missing)
-                + ". Could you provide "
-                + ("it" if len(actually_missing) == 1 else "them")
-                + "?"
-            )
-            return ClarificationNeeded(
-                question_id=question_id,
-                missing=missing_items,
-                prompt_to_user=prompt_to_user,
-            )
-
-    # ------------------------------------------------------------------ #
-    # 6. Resolve schools
-    # ------------------------------------------------------------------ #
-    for school_name in intent.school_mentions:
-        try:
-            records = await client.school_directory(search=school_name, 
limit=25)
-        except Exception as exc:
-            deps.tool_calls.append(
-                ToolInvocation(
-                    "search_schools",
-                    {"search": school_name, "limit": 25},
-                    error_kind=type(exc).__name__,
-                )
-            )
-            raise
-
-        deps.searched = True
-        resolved = [r["acara_sml_id"] for r in records if "acara_sml_id" in r]
-        if resolved:
-            # Take the first match: the planner trusts the intent model to have
-            # named an unambiguous school. Ambiguity handling 
(ClarificationNeeded)
-            # is the live-model path; the planner uses best-effort resolution.
-            best = resolved[0]
-            if best not in deps.resolved_school_ids:
-                deps.resolved_school_ids.append(best)
-        deps.tool_calls.append(
-            ToolInvocation(
-                "search_schools",
-                {"search": school_name, "limit": 25},
-                records,
-            )
-        )
-
-    # ------------------------------------------------------------------ #
-    # 7. Build the evidence request
-    # ------------------------------------------------------------------ #
-    request: dict[str, Any] = dict(intent.filters)
-
-    # Attach resolved school IDs using the right field name for this question
-    if deps.resolved_school_ids:
-        q_entry = contract.questions()[question_id]
-        required = q_entry.get("input_policy", {}).get("required", [])
-        if "acara_sml_ids" in required:
-            request.setdefault("acara_sml_ids", deps.resolved_school_ids)
-        elif "acara_sml_id" in required and deps.resolved_school_ids:
-            request.setdefault("acara_sml_id", deps.resolved_school_ids[0])
-
-    # Map requested metrics to the request (NL labels are passed through for
-    # now — the real emitter validates them; bad values return bad_request)
-    if intent.requested_metrics:
-        request.setdefault("metrics", intent.requested_metrics)
-
-    # Apply metric scope check for VC-POC-01 and VC-POC-04
-    scope_problem = request_scope_problem(deps.user_question, question_id, 
request)
-    if scope_problem and "metrics" in request and 
isinstance(request["metrics"], list):
-        # Drop non-teaching and headcount expansions the user didn't ask for
-        cleaned = [
-            m for m in request["metrics"]
-            if not (
-                str(m).endswith("_headcount") or 
str(m).startswith("non_teaching_")
-            )
+            deps.searched = True
+        else:
+            if _requires_two_distinct_ids(entry) and len(resolved_ids) < 2:
+                return _insufficient_schools_clarification(question_id)
+            for school_id in resolved_ids:
+                if school_id not in deps.resolved_school_ids:
+                    deps.resolved_school_ids.append(school_id)
+            deps.searched = True
+            _attach_school_ids(request, school_required, resolved_ids)
+    elif school_required and identity_from_caller and "acara_sml_ids" in 
school_required:
+        explicit_ids = _explicit_id_values(request, school_required)
+        raw_ids = request.get("acara_sml_ids")
+        if (
+            _requires_two_distinct_ids(entry)
+            and isinstance(raw_ids, list)
+            and len(explicit_ids) == len(raw_ids)
+            and len(_dedupe(explicit_ids)) < 2
+        ):
+            return _insufficient_schools_clarification(question_id)
+
+    # ------------------------------------------------------------------ #
+    # 7. Selector normalization and routing (clarifies unnormalizable labels)
+    # ------------------------------------------------------------------ #
+    selector_clarification = _route_selectors(question_id, intent, request, 
deps)
+    if selector_clarification is not None:
+        return selector_clarification
+
+    # ------------------------------------------------------------------ #
+    # 8. Registry-owned explicit-filter constraints (VC-POC-09/10)
+    # ------------------------------------------------------------------ #
+    constraint_clarification = _enforce_explicit_filter_constraints(
+        question_id, entry, request
+    )
+    if constraint_clarification is not None:
+        return constraint_clarification
+
+    # ------------------------------------------------------------------ #
+    # 9. Derive missing inputs from the pinned policy (stable order)
+    # ------------------------------------------------------------------ #
+    missing = _derive_missing_inputs(question_id, entry, request)
+    if missing:
+        missing_items = [
+            MissingInput(
+                input=field,
+                why_material=_material_why(question_id, field),
+                allowed_values=_static_vocabulary_for_field(question_id, field)
or [],
+            )
+            for field in missing
         ]
-        request["metrics"] = cleaned or request["metrics"]
-
-    # ------------------------------------------------------------------ #
-    # 8. Fetch evidence
+        prompt_to_user = (
+            "To answer this question I need "
+            + ", ".join(f"the {field}" for field in missing)
+            + ". Could you provide "
+            + ("it" if len(missing) == 1 else "them")
+            + "?"
+        )
+        logger.debug(
+            "planner clarification",
+            extra={"subsystem": "planner", "question_id": question_id, 
"missing": missing},
+        )
+        return ClarificationNeeded(
+            question_id=question_id,
+            missing=missing_items,
+            prompt_to_user=prompt_to_user,
+        )
+
+    # ------------------------------------------------------------------ #
+    # 10. Pinned-schema validation before dispatch
+    # ------------------------------------------------------------------ #
+    _validate_request(question_id, request)
+
+    # ------------------------------------------------------------------ #
+    # 11. Fetch evidence (failures recorded before branching)
     # ------------------------------------------------------------------ #
     arguments = {"question_id": question_id, "request": request}
     try:
         envelope = await client.fetch_evidence(question_id, request)
     except EvidenceError as exc:
         deps.tool_calls.append(
             ToolInvocation("fetch_evidence", arguments, 
error_kind=exc.error_kind)
         )
-        return _branch_evidence_error(question_id, exc)
+        return _branch_evidence_error(question_id, exc, identity_from_caller)
     except Exception:
         deps.tool_calls.append(
             ToolInvocation("fetch_evidence", arguments, 
error_kind="operational")
         )
         raise
@@ -284,26 +1069,32 @@
         "grades": "The grade selection determines which year levels are 
included.",
         "peer_filters": "The peer group definition determines which schools are
compared.",
         "segment_filters": "The segment definition determines which schools are
included.",
         "sa2_codes": "The SA2 area determines which geographic area is 
analysed.",
         "zone_entity_code": "The zone entity code identifies the government 
zone to look up.",
+        "acara_sml_id": "The school must be identified before evidence can be 
fetched.",
+        "acara_sml_ids": "The schools must be identified before evidence can be
fetched.",
     }
     return explanations.get(
         input_name,
         f"{input_name} is required by the data layer and cannot be defaulted 
for this question.",
     )
 
 
 def _branch_evidence_error(
     question_id: str,
     exc: EvidenceError,
-) -> Unsupported:
-    """Translate an EvidenceError into an Unsupported outcome.
-
-    This mirrors `_branch_on_error_kind()` in tools.py but returns typed values
-    instead of raising PydanticAI signals. TerminalError is re-raised (never
-    returned as Unsupported — it is an operational stop, not an honest 
refusal).
+    identity_from_caller: bool,
+) -> Unsupported | ClarificationNeeded:
+    """Translate an EvidenceError into the pipeline's typed outcomes.
+
+    Origin-aware: a bad_request for a request whose school identity was
+    caller-supplied is a correctable identity problem → ClarificationNeeded.
+    Any other bad_request — the request was deterministic and schema-valid —
+    is planner/contract drift → TerminalError. excluded/unknown/unanswerable
+    keep their contract-owned refusals; data_unavailable/internal are
+    operational stops, never refusals.
     """
     kind = exc.error_kind
     logger.debug(
         "evidence error in planner",
         extra={"subsystem": "planner", "error_kind": kind, "question_id": 
question_id},
@@ -327,11 +1118,31 @@
 
     if kind == "unanswerable":
         return Unsupported(reason_kind="unanswerable", reason=exc.message)
 
     if kind == "bad_request":
-        # On a first bad_request the planner cannot repair (no model); return
-        # out_of_scope so the caller knows the data layer rejected the request.
-        return Unsupported(reason_kind="out_of_scope", reason=exc.message)
+        if identity_from_caller:
+            return ClarificationNeeded(
+                question_id=question_id,
+                missing=[
+                    MissingInput(
+                        input="acara_sml_id",
+                        why_material=(
+                            "The data layer rejected the supplied school 
identity "
+                            f"({exc.message}). Confirm the school's current 
public "
+                            "ACARA_SML_ID."
+                        ),
+                        allowed_values=[],
+                    )
+                ],
+                prompt_to_user=(
+                    "The school identity supplied for this question was 
rejected by the "
+                    "data layer. Could you confirm the school?"
+                ),
+            )
+        raise TerminalError(
+            f"{question_id}: the evidence emitter returned bad_request for a "
+            f"deterministic, schema-valid request — planner/contract drift: 
{exc.message}"
+        )
 
     # data_unavailable | internal — operational stop, not a refusal.
     raise TerminalError(f"evidence emitter reported {kind}: {exc.message}")
```



**Changed files**: `harness/planner.py` — added draft-2020-12 numeric-form 
`exclusiveMinimum`/`exclusiveMaximum` support in `_schema_problems()` so 
OD-04-style bounds (e.g. a positive `distance_km`) fail closed before dispatch; 
behaviour is otherwise identical.



**No changes needed**: `run.py`, `tests/unit/test_planner.py`, 
`tests/unit/test_intent_boundary.py`, `tests/unit/test_run_cli.py` — 
`StructuredIntent`'s fields and defaults, and every output-type constructor and 
field name the tests and planner use, are confirmed against the newly supplied 
`harness/intent.py` and `harness/outputs.py`.



**Others worth adding** (last remaining gap): the twelve per-question request 
schemas not yet in the chat (`contract/question_schemas/v0.3.0/` for 
VC-POC-05/06/07/08/09/10/11/12/13/14/REF-01/REF-02). That would let me 
keyword-check the validator against all sixteen registered schemas rather than 
inferring from the four supplied samples — particularly whether VC-POC-09/10 use
`minProperties` (covered) and VC-POC-08 encodes OD-04 via `exclusiveMinimum` 
(now covered). They are read-only context, so no edit would follow — just 
confirmation, and any gap would surface as a `TerminalError` on first suite run 
in the worst case.

Tokens: 165k sent, 17k received. Cost: $0.03 message, $0.15 session.
Applied edit to harness/planner.py
Summarization failed for model openrouter/z-ai/glm-5.3-flash: cannot schedule new futures after shutdown
Summarization failed for model openrouter/z-ai/glm-5.3-flash: cannot schedule new futures after shutdown
summarizer unexpectedly failed for all models
