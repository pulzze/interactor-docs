# Expressions

**Version:** 1.0.0
**Last Updated:** 2026-02-04

---

## Overview

Interactor uses **JSONata** as its expression language for data transformation and condition evaluation throughout workflows and tools. JSONata is a lightweight, safe query and transformation language designed for JSON data.

### Where Expressions Are Used

| Context | Purpose | Example |
|---------|---------|---------|
| Tool `input_mapping` | Transform workflow data before calling a tool | `{ "total": $sum(data.items.price) }` |
| Tool `output_mapping` | Transform tool response before storing | `{ "user_id": response.data.id }` |
| Workflow conditions | Control workflow transitions | `data.amount > 10000` |

---

## Quick Reference

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

Control state transitions:

```json
{
  "states": {
    "check_amount": {
      "type": "decision",
      "transitions": [
        {
          "to": "high_value_approval",
          "condition": "data.amount > 50000"
        },
        {
          "to": "manager_approval",
          "condition": "data.amount > 10000 and data.amount <= 50000"
        },
        {
          "to": "auto_approve",
          "condition": "data.amount <= 10000"
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

### In Tool Mappings

| Variable | Description |
|----------|-------------|
| `data` | Current workflow data |
| `parameters` | Workflow parameters |
| `response` | Tool response (in output_mapping only) |

### In Workflow Conditions

| Variable | Description |
|----------|-------------|
| `data` | Current workflow data |
| `parameters` | Workflow parameters |
| `instance` | Workflow instance metadata (id, workflow_name, current_state) |
| `thread` | Thread metadata (id) |

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
  "execution_time_ms": 2
}
```

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
| Max execution time | 100ms |
| Max output size | 1MB |
| Max recursion depth | 100 |

---

## Resources

- [JSONata Documentation](https://docs.jsonata.org/) - Full language reference
- [JSONata Exerciser](https://try.jsonata.org/) - Interactive playground
- [JSONata Functions](https://docs.jsonata.org/overview#functions) - Built-in function reference
