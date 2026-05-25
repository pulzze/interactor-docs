# Expressions

**Version:** 1.0.0
**Last Updated:** 2026-05-24

---

## Overview

Interactor uses **JSONata** as its expression language for data transformation and condition evaluation throughout workflows and tools. JSONata is a lightweight, safe query and transformation language designed for JSON data.

### Where Expressions Are Used

| Context | Syntax | Purpose | Example |
|---------|--------|---------|---------|
| Step configurations | `{{ expression }}` | Inject data into URLs, headers, bodies, prompts | `"{{ input.user_id }}"` |
| Tool `input_mapping` | Bare JSONata | Transform workflow data before calling a tool | `$sum(data.items.price)` |
| Tool `output_mapping` | Bare JSONata | Transform tool response before storing | `response.data.id` |
| Workflow conditions | Bare JSONata | Control workflow transitions | `input.amount > 10000` |

Interactor uses expressions in two forms:
- **Template expressions** (`{{ }}`) — embedded in step configuration strings (URLs, headers, bodies, prompts). Simple dot-paths are resolved instantly; complex expressions use JSONata.
- **Bare JSONata** — used directly in tool mappings and workflow conditions.

Both forms use the same JSONata language for complex expressions.

---

## Template Expressions

Step configurations (HTTP steps, AI steps, agent steps, etc.) use `{{ }}` delimiters to inject dynamic values into strings:

```json
{
  "type": "http",
  "config": {
    "url": "https://api.example.com/users/{{ input.user_id }}",
    "headers": {
      "Authorization": "Bearer {{ input.api_key }}"
    },
    "body": {
      "name": "{{ data.user.name }}",
      "count": "{{ data.item_count }}"
    }
  }
}
```

### Available Sources

| Source | Description | Mutability | Example |
|--------|-------------|------------|---------|
| `input` | Values passed at instance creation | Immutable | `{{ input.order_id }}` |
| `data` | Runtime workflow data | Mutable | `{{ data.status }}` |

**Note:** `input` values cannot be modified by workflow steps. Use `data` for values that change during execution.

### Type Preservation

When the entire string is a single expression, the raw typed value is returned:

```json
{
  "count": "{{ data.item_count }}",
  "user": "{{ data.user_profile }}",
  "tags": "{{ data.tag_list }}"
}
```

Resolves to:
```json
{
  "count": 42,
  "user": {"name": "Alice", "email": "alice@example.com"},
  "tags": ["billing", "priority"]
}
```

When an expression is embedded in surrounding text, values are coerced to strings:

```json
{
  "message": "Hello {{ data.name }}, you have {{ data.count }} items"
}
```

Resolves to:
```json
{
  "message": "Hello Alice, you have 42 items"
}
```

### Simple vs Complex Expressions

**Simple dot-path expressions** (e.g., `{{ data.user.name }}`) are resolved instantly via direct lookup with no overhead.

**Complex expressions** containing array access, filters, functions, or operators are evaluated via JSONata:

```json
{
  "first_item": "{{ data.items[0].name }}",
  "active_count": "{{ $count(data.items[active = true]) }}",
  "total": "{{ $sum(data.items.price) }}",
  "is_high_value": "{{ input.amount > input.threshold }}"
}
```

Any valid JSONata expression can be used inside `{{ }}` delimiters.

---

## Bare JSONata Expressions

Tool mappings and workflow conditions use JSONata expressions directly, without `{{ }}` delimiters.

### Data Access

```jsonata
data.customer.email          // Nested property access
data.items[0].name           // Array index
data.items[-1]               // Last array element
data.items[status='active']  // Filter by condition
```

### Operators

| Operation | JSONata | Example |
|-----------|---------|---------|
| Equality | `=` | `data.status = 'approved'` |
| Not equal | `!=` | `data.status != 'pending'` |
| Greater/less | `>`, `>=`, `<`, `<=` | `data.amount > 1000` |
| Logical AND | `and` | `data.valid and data.complete` |
| Logical OR | `or` | `data.urgent or data.priority = 'high'` |
| Logical NOT | `not` | `not data.deleted` |
| String concat | `&` | `data.first & ' ' & data.last` |
| Ternary | `? :` | `data.vip ? 'priority' : 'standard'` |

### Common Functions

```jsonata
// Aggregation
$sum(items.price)            // Sum numeric values
$count(items)                // Count array elements
$average(items.score)        // Calculate average
$min(items.price)            // Minimum value
$max(items.price)            // Maximum value

// String manipulation
$uppercase(name)             // Convert to uppercase
$lowercase(name)             // Convert to lowercase
$trim(text)                  // Remove whitespace
$contains(text, 'search')    // Check if contains substring
$split(text, ',')            // Split into array
$join(items, ', ')           // Join array to string

// Type checking
$type(value)                 // Returns type name
$string(value)               // Convert to string
$number(value)               // Convert to number
$boolean(value)              // Convert to boolean

// Date/time
$now()                       // Current timestamp (ms)
$fromMillis(ts)              // Format timestamp
$toMillis(dateStr)           // Parse to timestamp
```

---

## Examples

### Tool Input Mapping

Transform workflow data before calling a tool:

```json
{
  "name": "send_invoice",
  "input_mapping": {
    "recipient_email": "data.customer.billing_email",
    "amount": "$sum(data.line_items.price) * 1.1",
    "invoice_number": "'INV-' & $string(data.order_id)",
    "items": "data.line_items.{ 'description': name, 'total': quantity * unit_price }"
  }
}
```

### Tool Output Mapping

Extract and transform tool response:

```json
{
  "name": "create_user",
  "output_mapping": {
    "user_id": "response.data.id",
    "created_at": "response.data.created_at",
    "success": "response.status = 'ok'"
  }
}
```

### Workflow Conditions

Control state transitions using `input` (immutable) and `data` (runtime):

```json
{
  "states": {
    "check_amount": {
      "type": "decision",
      "transitions": [
        {
          "to": "high_value_approval",
          "condition": "input.amount > 50000"
        },
        {
          "to": "manager_approval",
          "condition": "input.amount > 10000 and input.amount <= 50000"
        },
        {
          "to": "auto_approve",
          "condition": "input.amount <= 10000"
        }
      ]
    }
  }
}
```

### Complex Transformations

```jsonata
// Filter and transform an array
data.orders[status = 'pending'].{
  "id": order_id,
  "customer": customer_name,
  "total": $sum(items.price),
  "item_count": $count(items)
}

// Conditional object construction
{
  "level": data.total > 100000 ? "enterprise" :
           data.total > 10000 ? "business" : "starter",
  "discount": data.is_vip ? 0.2 : 0
}

// Aggregation with grouping
$sum(data.items[category = 'electronics'].price)
```

---

## Expression Context

Expressions have access to the following context variables depending on where they're used:

### In Workflow Steps and Conditions

| Variable | Description | Mutability |
|----------|-------------|------------|
| `input` | Values passed at instance creation | Immutable |
| `data` | Runtime workflow data | Mutable |
| `instance` | Instance metadata (id, workflow_name, current_state) | Read-only |
| `thread` | Thread metadata (id) | Read-only |

### In Tool Mappings

| Variable | Description |
|----------|-------------|
| `input` | Workflow instance input |
| `data` | Current workflow data |
| `response` | Tool response (in output_mapping only) |

---

## Testing Expressions

Use the expressions playground API to test your expressions before deploying:

### Evaluate Expression

```bash
curl -X POST https://core.interactor.com/api/v1/expressions/evaluate \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "expression": "$sum(items.price) * 1.1",
    "input": {
      "items": [
        {"name": "Widget", "price": 10},
        {"name": "Gadget", "price": 25}
      ]
    }
  }'
```

**Response:**
```json
{
  "result": 38.5,
  "execution_time_ms": 2,
  "cached": false,
  "no_match": false
}
```

The `no_match` field is `true` when the expression resolved to `undefined` (e.g., accessing a missing field). The service normalizes this to `result: null` in the response.

### Validate Expression

Check expression syntax without evaluating:

```bash
curl -X POST https://core.interactor.com/api/v1/expressions/validate \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "expression": "$sum(items.price"
  }'
```

**Response (invalid):**
```json
{
  "valid": false,
  "error": "EXPR_SYNTAX_ERROR",
  "message": "Expected ')' at position 16",
  "position": 16
}
```

---

## Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `EXPR_SYNTAX_ERROR` | 400 | Expression has invalid syntax |
| `EXPR_TIMEOUT` | 408 | Expression execution exceeded time limit (100ms) |
| `EXPR_OUTPUT_TOO_LARGE` | 413 | Result exceeded size limit (1MB) |
| `EXPR_SERVICE_UNAVAILABLE` | 503 | JSONata service is not available |

---

## Safety & Limits

JSONata expressions are safe by design:

- No file system access
- No network access
- No code execution or eval
- No global state mutation
- Deterministic (same input = same output)

### Execution Limits

| Limit | Value |
|-------|-------|
| Max execution time | 100ms (default; configurable per deployment) |
| Max output size | 1MB |

---

## Error Handling

### Undefined Variables

Accessing undefined variables returns `null` (standard JSONata behavior):

```jsonata
input.nonexistent           // Returns null
input.user.missing.deeply   // Returns null (no error)
data.items[99]              // Returns null if index out of bounds
```

This means expressions won't fail on missing data, but you may get unexpected `null` values.

### Defensive Patterns

Use these patterns to handle potentially missing data:

```jsonata
// Default values via existence check (JSONata has no ?? operator)
$exists(input.count) ? input.count : 0     // Use 0 if count is missing
$exists(input.name) ? input.name : "Unknown" // Use default string

// Conditional checks
input.items ? $sum(input.items.price) : 0  // Check before aggregating
data.user ? data.user.email : null         // Safe property access

// Type checking
$type(input.value) = 'number'              // Verify type before operations
```

### Expression Errors in Workflows

When an expression fails during workflow execution, the error is recorded in the workflow history:

```json
{
  "type": "error",
  "subtype": "expression_failed",
  "state": "calculate_total",
  "error_code": "EXPR_TIMEOUT",
  "message": "Expression execution exceeded time limit (100ms)",
  "expression": "$sum(data.items.price)"
}
```

---

## Resources

- [JSONata Documentation](https://docs.jsonata.org/) - Full language reference
- [JSONata Exerciser](https://try.jsonata.org/) - Interactive playground
- [JSONata Functions](https://docs.jsonata.org/overview#functions) - Built-in function reference
