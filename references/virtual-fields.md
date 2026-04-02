# Okyline Virtual Fields — `$field`

Virtual fields are computed values that exist only during validation. Use them when conditional logic depends on a **derived value not present in the data**.

## Syntax

```
"$field <name>": "%<ComputeExpressionName>"
```

```json
{
  "$oky": {
    "order": {
      "$field tier": "%ComputeTier",
      "total": 1500.00,
      "$appliedIf tier('GOLD')": {
        "loyaltyBonus|@": 50.00
      }
    }
  },
  "$compute": {
    "ComputeTier": "total >= 1000 ? 'GOLD' : 'STANDARD'"
  }
}
```

## Rules

- Value MUST reference a `$compute` expression (`%Name`)
- Names MUST NOT collide with actual field names or `$appliedIf` payload fields
- No forward references — a `$field` can only reference virtual fields declared **before** it
- Declared at object level only, not inside `$appliedIf` payloads
- **Scope:** local to the declaring object — not visible in children, not inherited via `$ref`
- **Restriction:** MUST NOT be used in existence-based directives (`$requiredIfExist`, etc.) — use `fieldName(null)` instead

## Chaining

Virtual fields are evaluated sequentially. Each one can reference previously declared ones:

```json
"$field subtotal": "%ComputeSubtotal",
"$field tax": "%ComputeTax",
"$field total": "%ComputeTotal"
```

Where `ComputeTax` uses `subtotal` and `ComputeTotal` uses `subtotal + tax`.
