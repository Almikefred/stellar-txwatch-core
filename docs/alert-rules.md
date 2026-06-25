# Alert Rules Reference

Rules are evaluated per-transaction for each watched contract.
Multiple rules can match the same transaction — each fires an independent webhook call.
A rule evaluation error is logged as a warning and skipped; it never stops the engine.

## Rule types

### `AnyTransaction`
Matches every transaction that appears in the contract's Horizon history.

**Use case:** full audit trail, low-volume contracts.

```toml
[[contracts.rules]]
type = "AnyTransaction"
```

### `TransactionFailed`
Matches transactions where `successful = false`.

**Use case:** detect reverted Soroban invocations or fee-bump failures.

```toml
[[contracts.rules]]
type = "TransactionFailed"
```

### `LargeTransfer`

| Field           | Type | Required | Description                        |
|-----------------|------|----------|------------------------------------|
| `threshold_xlm` | u64  | yes      | Minimum transfer amount in XLM (> 0) |

Matches when the payment amount (extracted from Horizon operations) is ≥ `threshold_xlm` XLM.
The `amount_xlm` field in the webhook payload contains the actual transferred amount.

**Note:** Amount is extracted from `payment` operation records. Soroban token transfers
that do not produce a native `payment` operation will not populate `amount_xlm`.

```toml
[[contracts.rules]]
type          = "LargeTransfer"
threshold_xlm = 10000
```

### `FunctionCalled`

| Field           | Type   | Required | Description                          |
|-----------------|--------|----------|--------------------------------------|
| `function_name` | string | yes      | Exact function name (case-sensitive) |

Matches when the Soroban `invoke_host_function` operation calls exactly `function_name`.

```toml
[[contracts.rules]]
type          = "FunctionCalled"
function_name = "withdraw"
```

### `AdminFunctionCalled`

| Field            | Type     | Required | Description                              |
|------------------|----------|----------|------------------------------------------|
| `function_names` | [string] | yes      | Non-empty list of function names to watch |

Matches when the invoked function is any entry in `function_names`.
Equivalent to multiple `FunctionCalled` rules but produces a single
`AdminFunctionCalled([...])` label in the alert.

```toml
[[contracts.rules]]
type           = "AdminFunctionCalled"
function_names = ["set_admin", "upgrade", "initialize"]
```

### `HighFee`

| Field                | Type | Required | Description                           |
|----------------------|------|----------|---------------------------------------|
| `threshold_stroops`  | u64  | yes      | Fee threshold in stroops (> 0)        |

Matches when the transaction's total fee exceeds `threshold_stroops`.
The `fee_charged` field in the webhook payload contains the actual fee paid in stroops.

**Note:** Stroops are the smallest unit of XLM (1 XLM = 10,000,000 stroops).

```toml
[[contracts.rules]]
type               = "HighFee"
threshold_stroops  = 100000
```

### `OperationCountExceeds`

| Field             | Type | Required | Description                    |
|-------------------|------|----------|--------------------------------|
| `max_operations`  | u64  | yes      | Operation threshold (> 0)      |

Matches when the transaction contains more than `max_operations` operations.
The rule fires only when `operation_count > max_operations` (strict greater-than).

**Use case:** detect complex multi-step transactions or potential attack patterns.

```toml
[[contracts.rules]]
type            = "OperationCountExceeds"
max_operations  = 5
```

### `MultipleFailuresInWindow`

| Field            | Type | Required | Description                       |
|------------------|------|----------|-----------------------------------|
| `failure_count`  | u64  | yes      | Number of failures to trigger (≥ 1) |
| `window_seconds` | u64  | yes      | Time window in seconds (> 0)      |

**Stateful rule:** Fires when a contract experiences `failure_count` failed transactions
within `window_seconds`. After firing, the failure counter resets. Failed transactions
(where `successful = false`) are tracked; other transactions are ignored.

**Use case:** alert on sudden bursts of transaction failures or repeated errors.

**Memory:** Maintains a sliding window of up to `failure_count` recent failure timestamps per rule instance.

```toml
[[contracts.rules]]
type             = "MultipleFailuresInWindow"
failure_count    = 3
window_seconds   = 300
```

## Time Window Filter

| Field        | Type | Required | Description                 |
|--------------|------|----------|-----------------------------|
| `start_hour` | u8   | no       | Start hour (UTC, 0-23)      |
| `end_hour`   | u8   | no       | End hour (UTC, 0-23)        |

Restricts any rule's evaluation to a specific UTC hour window. Rules with a time window defined
will only fire during `[start_hour, end_hour)` in UTC. If `start_hour >= end_hour`, the window
is invalid and validation will fail.

**Use case:** reduce alert noise outside business hours, defer sensitive operations to off-hours.

**Note:** The time window is evaluated at the UTC hour level. A rule fires only if the current
UTC hour `h` satisfies `h >= start_hour && h < end_hour`.

```toml
[[contracts.rules]]
type          = "FunctionCalled"
function_name = "withdraw"
start_hour    = 9
end_hour      = 17
```

This rule fires for `withdraw` calls only between 09:00 and 16:59 UTC.

### `FunctionNamePattern`

| Field     | Type   | Required | Description                              |
|-----------|--------|----------|------------------------------------------|
| `pattern` | string | yes      | Glob or regex pattern for function names |
| `mode`    | string | yes      | Pattern mode: `glob` or `regex`          |

Matches when any invoked Soroban function matches the provided pattern in the specified mode.

**Use case:** match function names by prefix/suffix (glob) or complex patterns (regex).

**Glob examples:**
- `set_*` matches `set_admin`, `set_fee`, `set_threshold`, etc.
- `*_v2` matches `init_v2`, `execute_v2`, etc.
- `?fetch*` matches `prefetch_`, `isfetch_anything`, etc.

**Regex examples:**
- `set_.*` matches any function starting with `set_`
- `(init|execute)` matches either `init` or `execute`
- `.*_v[0-9]+` matches versioned function names like `foo_v1`, `bar_v2`

```toml
[[contracts.rules]]
type    = "FunctionNamePattern"
pattern = "set_*"
mode    = "glob"
```

## Evaluation order

Rules are evaluated in the order they appear in the config file.
All matching rules fire; there is no short-circuit.

## Webhook payload fields

| Field              | Type        | Always present | Description                              |
|--------------------|-------------|----------------|------------------------------------------|
| `label`            | string      | yes            | Contract label from config               |
| `contract_id`      | string      | yes            | Stellar C-address                        |
| `network`          | string      | yes            | `mainnet` / `testnet` / `futurenet`      |
| `rule_triggered`   | string      | yes            | Human-readable rule description          |
| `transaction_hash` | string      | yes            | Stellar transaction hash                 |
| `function_name`    | string/null | no             | Soroban function name if available; `null` indicates a non-Soroban transaction |
| `amount_xlm`       | u64/null    | no             | Transfer amount in XLM if available      |
| `timestamp`        | i64         | yes            | Unix timestamp (seconds) of transaction  |
| `horizon_link`     | string      | yes            | Direct link to transaction on Horizon    |
| `explorer_link`    | string      | yes            | Stellar Expert explorer link for the transaction |

> `horizon_link` and `explorer_link` are always present in every alert payload, even when `function_name` is `null` for a non-Soroban transaction.

## Stable rule_type values

The webhook payload includes two rule-related fields:

| Field | Purpose | Example |
|-------|---------|---------|
| `rule_type` | Machine-readable, stable rule variant name; use for programmatic routing | `"LargeTransfer"` |
| `rule_triggered` | Human-readable description with parameters; use for display | `"LargeTransfer(>=10000XLM)"` |

### Rule type table

| Rule | `rule_type` value |
|------|-------------------|
| `AnyTransaction` | `"AnyTransaction"` |
| `TransactionFailed` | `"TransactionFailed"` |
| `LargeTransfer` | `"LargeTransfer"` |
| `FunctionCalled` | `"FunctionCalled"` |
| `AdminFunctionCalled` | `"AdminFunctionCalled"` |
| `HighFee` | `"HighFee"` |
| `OperationCountExceeds` | `"OperationCountExceeds"` |
| `MultipleFailuresInWindow` | `"MultipleFailuresInWindow"` |
| `FunctionNamePattern` | `"FunctionNamePattern"` |

## Adding a new rule type

1. Add a variant to `AlertRule` in `crates/config/src/lib.rs`
2. Add field validation in `AlertRule::validate()` in the same file
3. Add the match arm in `eval_rule()` in `crates/rules/src/lib.rs`
4. Add the label string in `rule_label()` in the same file
5. Add a stable `rule_type` string in `rule_type()` in the same file
6. Add unit tests in `crates/rules/src/lib.rs`
7. Update the rule type table in this section
8. Update the webhook payload example in README.md (if adding a new example)

No other crates need changes.
