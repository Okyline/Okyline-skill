---
name: okyline
description: >
  Expert assistant for the Okyline schema language — create, edit, and convert
  JSON validation schemas using declarative, example-driven syntax with inline constraints.
metadata:
  version: "2.0.0"
  author: Okyline
  repository: Okyline/Okyline-skill
---

# Okyline Schema Language v1.6.0

Okyline is a declarative language for describing and validating JSON structures using inline constraints on field names. Schemas are valid JSON documents with real example values.

## ⚠️ Before any schema generation

**MANDATORY**: Read the reference files BEFORE producing an Okyline schema:

1. `references/syntax-reference.md` — complete syntax of constraints
2. `references/internal-references.md` — $defs and $ref
3. `references/conditional-directives.md` — if conditional logic
4. `references/expression-language.md` — if $compute is necessary
5. `references/virtual-fields.md` — if $field (virtual fields) is necessary

Never generate a schema based solely on this SKILL.md file.
The examples here are a summary, not an exhaustive reference.

## $ref: when to use

Use `$defs`/`$ref` when:
1. **Recursion** (mandatory — only way to express recursive structures)
2. **Repeated structures** identical across multiple usages (e.g. `Period`, `Address`)
3. **Template pattern** — base structure specialized per usage via `$override` or `$amend` (e.g. `Coding`)
4. **Explicit user request**

Default → inline in `$oky`. Don't over-abstract — factorize only when it reduces real duplication or expresses a meaningful shared type.


## Core Syntax

```
"fieldName | constraints | label": exampleValue
```

- **fieldName**: JSON field name
- **constraints**: Validation rules (space-separated)
- **label**: Optional human-readable description
- **exampleValue**: Determines the inferred type

If a label is present without constraints, use `| |`:

  ❌ `"acheteur|Client"` → "Client" parsed as constraint!
  ✅ `"acheteur| |Client"` → "Client" is the label

## Minimal Schema Structure

```json
{
  "$oky": {
    "name|@ {2,50}|User name": "Alice",
    "email|@ ~$Email~": "alice@example.com",
    "age|(18..120)": 30
  }
}
```

## Essential Constraints

| Symbol | Meaning | Example |
|--------|---------|---------|
| `@` | Required field | `"name\|@": "Alice"` |
| `?` | Nullable (can be null) | `"middle\|?": "John"` |
| `{min,max}` | String length | `"code\|{5,10}": "ABC123"` |
| `(min..max)` | Numeric range | `"age\|(18..65)": 30` |
| `('a','b')` | Enum values | `"status\|('ACTIVE','INACTIVE')": "ACTIVE"` |
| `~pattern~` | Regex or format | `"email\|~$Email~": "a@b.com"` |
| `[min,max]` | Array size | `"tags\|[1,5]": ["eco"]` |
| `[*]` | Array (no size constraint) | `"items\|[*]": [1, 2]` |
| `->` | Element validation constraint | `"tags\|[*] -> {2,10}": ["eco"]` |
| `!` | Unique elements | `"codes\|[*]!": ["A","B"]` |
| `#` | Key field (for object uniqueness) | `"id\|#": 123` |
| `$ref` | Reference to definition (type indicator, not a validation constraint) | `"address\|$ref": "&Address"` |


### Open-ended numeric constraints

When only one bound is needed, use comparison operators:

| Syntax | Meaning | Example |
|--------|---------|---------|
| `(>0)` | Strictly positive | `"quantity\|(>0)": 5` |
| `(>=0)` | Positive or zero | `"price\|(>=0)": 29.99` |
| `(<100)` | Strictly < 100 | `"percentage\|(<100)": 50` |
| `(<=100)` | ≤ 100 | `"score\|(<=100)": 85` |

❌ Never invent syntax with missing bound: `(0..)`, `(..100)`
✅ Use comparisons: `(>=0)`, `(<=100)`

❌ Never use huge placeholder bounds: `(0..99999999)`, `(-99999999..100)`
✅ Use comparisons: `(>=0)`, `(<=100)`


⚠️ **INVALID SYNTAX** — Okyline does NOT support open ranges:
- ❌ `(0..)`, `(..100)`, `(1..*)`, `(0..*)`
- These will cause validation errors

### Unbounded collection size

Use `*` to indicate "no limit":

| Syntax | Meaning |
|--------|---------|
| `[*]` | Array with no size constraint |
| `[1,*]` | At least 1 element, no maximum |
| `[~pattern~:*]` | Map with validated keys, unlimited entries |

❌ Never omit a bound: `[1,]`, `[~pattern~:]`
✅ Use `*` explicitly: `[1,*]`, `[~pattern~:*]`


## Built-in Formats

Use with `~$FormatName~`: `$Date`, `$DateTime`, `$Time`, `$Email`, `$Uri`, `$Uuid`, `$Ipv4`, `$Ipv6`, `$Hostname`

## Type Inference Rules

- Type is inferred from example value (no explicit declarations)
- `42` → Integer, `3.14` → Number, `"text"` → String, `true` → Boolean
- Arrays must have at least one element for type inference
- **`null` cannot be used as example** — no type can be inferred. Use `?` for nullable fields with a real example value:

❌ `"middleName|?": null`
✅ `"middleName|?{1,50}": "Marie"`

❌ `"discount|?(0..100)": null`
✅ `"discount|?(0..100)": 15`

### Decimal Values Ending in .00

JSON serializers drop trailing zeros: `78.00` becomes `78`. 
To preserve decimal precision in examples, wrap in quotes:

❌ `"amount": 78.00`   → Serialized as 78, type inferred as Integer
✅ `"amount": "78.00"` → Preserved, type inferred as Number

This only affects decimals with zero fractional parts (.00, .0).
Regular decimals (45.5) and integers (10) are unaffected.


## ⚠️ `?` vs Absence — Critical Distinction

`?` and "optional field" are NOT the same thing in Okyline:

| | Meaning | When to use |
|--|---------|-------------|
| No `@` | Field may be **absent** from the document | FHIR optional fields, nullable in most APIs |
| `?` | Field may be explicitly **`null`** | Only when `null` is a meaningful intentional value |

```json
// ❌ Wrong — using ? for optional fields (common LLM mistake)
"meta|?|Resource metadata": { ... }
"subType|?|Claim subtype": { ... }

// ✅ Correct — optional = no @, never ?
"meta||Resource metadata": { ... }
"subType||Claim subtype": { ... }

// ✅ ? only when null is intentional and meaningful
"active|?|Explicitly unknown status": true
```

> In FHIR R4 and most REST APIs, absent fields are simply omitted — they are never sent as `null`. Use `?` only when the standard explicitly allows `null` as a distinct semantic value (e.g. "unknown" vs "not provided").

## Legacy Null Tolerance

```json
{ "$nullAsAbsentIfUndeclared": true }
```
When `true`, a `null` value on a field not marked `?` is treated as absent instead of invalid. Use only for legacy systems that send `null` instead of omitting fields. Default: `false`.

## Polymorphism vs Structural Directives — Key Distinction

These look similar but serve different purposes:

| | Target | Meaning |
|--|--------|---------|
| `$exactlyOne` / `$mutuallyExclusive` | Fields of an object | Constrain **presence** of sibling fields |
| `$oneOf` / `$anyOf` | A single field | Constrain the **structure** of that field's value |

```json
// $exactlyOne — exactly one of these sibling fields must be present
"diagnosisCodeableConcept": { ... },
"diagnosisReference": { ... },
"$exactlyOne": ["diagnosisCodeableConcept", "diagnosisReference"]

// $oneOf — the payment field itself must match exactly one variant
"payment|@ $oneOf": [
  { "type|@ ('card')": "card", "cardNumber|@ {16}": "1234567812345678" },
  { "type|@ ('paypal')": "paypal", "email|@ ~$Email~": "user@example.com" }
]
```

## Reusable Definitions — `$defs` and `$ref`

Okyline supports **internal references** to promote reuse and consistency.

### `$defs` — Definition Repository

`$defs` is placed at root level (alongside `$oky`) and contains reusable schema fragments:

```json
{
  "$oky": {
    "person": {
      "address | $ref": "&Address",
      "contact | $ref @": "&Email"
    }
  },
  "$defs": {
    "Address": {
      "street|@ {2,100}": "12 rue du Saule",
      "city|@ {2,100}": "Lyon"
    },
    "Email|~$Email~ {5,100}": "user@example.com"
  }
}
```

Definitions can be **objects** or **scalars** (with constraints in the key).

### Reference Syntax — `&Name`

References use `&` prefix: `&Address`, `&Email`, `&Person`

### Property-Level Reference — `field | $ref`

A field uses another schema as its type:

```json
"address | $ref @": "&Address"        // Required address
"backup | $ref ?": "&Address"         // Optional, nullable
"emails | $ref [1,5]": ["&Email"]     // Array of 1-5 emails
```

**Structural constraints** (`@`, `?`, `[min,max]`, `!`) are defined locally at each usage.
**Value constraints** (`{min,max}`, `(min..max)`, `~pattern~`) are inherited from the definition.

### Important: `$ref` vs `->` for arrays

`$ref` and `->` serve different purposes:
- `$ref` = type indicator (what the elements ARE) — comes BEFORE `->`
- `->` = validation constraints (rules elements must satisfy) — comes AFTER `$ref`

For arrays of referenced types, `$ref` is placed with the array constraint, never after `->`:
```json
// Correct syntax
"items|[*] $ref": ["&Item"]                    // Array of Item references
"items|[1,10] $ref": ["&Item"]                 // Array of 1-10 Item references  
"items|[*] $ref -> {2,50}": ["&Item"]          // With additional element constraint

// Wrong syntax - $ref must never be after ->
❌ "items|[*] -> $ref": ["&Item"]
```

### Object-Level Reference — Inheritance

Include all fields from a base schema:

```json
{
  "$oky": {
    "Article": {
      "$ref": "&Auditable",
      "title|@ {1,200}": "Mon article"
    }
  },
  "$defs": {
    "Auditable": {
      "createdAt|@ ~$DateTime~": "2025-01-01T00:00:00Z",
      "updatedAt|@ ~$DateTime~": "2025-01-01T00:00:00Z"
    }
  }
}
```

### Modifying Inherited Fields

| Directive | Usage | Description |
|-----------|-------|-------------|
| `$remove` | `"$remove": ["field1", "field2"]` | Exclude inherited fields |
| `$override` | `"field \| $override ...": value` | Replace inherited field entirely (unspecified blocks are erased) |
| `$amend` | `"field \| $amend ...": value` | Adapt inherited field block-by-block (unspecified blocks are kept from base) |

Both `$override` and `$amend` preserve type, collection nature and `$ref` target. In an `$appliedIf` branch, redefining a parent field without `$override` or `$amend` is a parsing error.

## Choosing the Right Conditional Mechanism

| Need | Mechanism | Example |
|------|-----------|---------|
| Field A required when field B has a specific value | `$requiredIf` | `"$requiredIf status('ACTIVE')": ["email"]` |
| Field A forbidden when field B has a specific value | `$forbiddenIf` | `"$forbiddenIf status('CLOSED')": ["lastLogin"]` |
| Field A required when field B exists/is absent | `$requiredIfExist` / `$requiredIfNotExist` | `"$requiredIfExist shipping": ["address"]` |
| Different fields depending on a value | `$appliedIf` switch | `"$appliedIf paymentMethod": { "('CARD')": {...}, "('PAYPAL')": {...} }` |
| Extra fields when a condition is met | `$appliedIf` simple | `"$appliedIf status('ACTIVE')": { "workDays\|@": 20 }` |
| Exactly one field among N must be present | `$exactlyOne` | `"$exactlyOne": ["email", "phone"]` |
| At most one field among N | `$mutuallyExclusive` | `"$mutuallyExclusive": ["optionA", "optionB"]` |
| At least one field among N | `$atLeastOne` | `"$atLeastOne": ["email", "phone", "fax"]` |
| All or none of a group | `$allOrNone` | `"$allOrNone": ["street", "city", "zip"]` |
| Condition based on a computed/derived value | `$field` + `$appliedIf` | `"$field tier": "%ComputeTier"` then `"$appliedIf tier('GOLD')": {...}` |
| Validation depends on field's runtime type | Type Guard | `"$appliedIf data(_String_)": {...}` |
| Condition on null value | Null literal | `"$requiredIf status(null)": ["fallback"]` |

## `$compute` vs `$field`

Use `$compute` alone to validate field values against business rules (`"total|(%CheckTotal)": 120.0`).

Introduce `$field` only when a conditional directive needs a **derived value not present in the data** (e.g., a computed tier, classification, or flag). If the condition can reference an existing field directly, don't use `$field`.

## $compute Context Rules

Expressions in `$compute` must be **attached to a field** using `|(%ComputeName)` syntax. The expression is evaluated in the context of the **object that directly contains the annotated field** — all properties of that object are accessible, including sibling arrays.

**Path resolution:** `parent` always refers to the parent of the evaluation context. Context fields are accessed directly (no prefix needed), so `parent` is always one level up from the context. Use `parent.parent` to go two levels up, or `root` to reach the document root directly. Use `this.` to disambiguate when a field name collides with a reserved keyword or a compute parameter (`parent`, `root`, `prev`, `next`, `first`, `last`, `origin`).

**Parameterized computes:** a compute can declare formal parameters: `"Name(p1, p2)": "body"` and be called with `%Name(arg1, arg2)`. Args on a field constraint are literals or dotted paths; full expressions are allowed from another compute body. Params shadow sibling fields inside the body — use `this.field` to unshadow. A param bound to an object exposes its members via `p.field`; a list param composes with `firstOf`/`at`/`findFirst`/`sum`/…

**List element access:** `firstOf(list)`, `lastOf(list)`, `findFirst(list, pred)`, `findLast(list, pred)`, `at(list, idx)` return a single element (null if empty / not found / out of bounds). Chain `.field` directly: `findFirst(lines, cat == 'S').amount`.

**Compute reference with dotted access:** `%Name.field.deep` accesses members of the compute's result, useful when the compute returns an object (e.g. `"%DocRoot.DocumentCurrencyCode"` where `DocRoot = "root.Invoice ?? root.CreditNote"`).

See `references/expression-language.md` for full details and examples.

## Schema Validation Checklist

Before delivering a schema, verify:

- [ ] `$oky` wrapper present
- [ ] All required fields marked with `@`
- [ ] No `?` on fields that are simply optional (use `?` only for intentional null)
- [ ] No `null` example values — use a real example with `?` if nullable
- [ ] No empty arrays `[]` — at least one element for type inference
- [ ] Decimals ending in .00 quoted as strings (`"78.00"` not `78.00`)
- [ ] Element constraints use `->` (not applied directly to array field)
- [ ] `$ref` before `->`, never after
- [ ] `$compute` expressions attached to fields with `|(%Name)`
- [ ] No two `(...)` constraint blocks on the same field
- [ ] Labels use `| |Label` syntax (not `|Label`)
- [ ] `$id` uses only letters, digits, underscores and dots
- [ ] Enums use `$nomenclature` or `('A','B')`, not regex patterns

## Reference Files

For detailed syntax and features, consult these references:

- **Constraint syntax & patterns**: See [references/syntax-reference.md](references/syntax-reference.md)
- **Conditional directives**: See [references/conditional-directives.md](references/conditional-directives.md)
- **Expression language ($compute)**: See [references/expression-language.md](references/expression-language.md)
- **Internal references ($defs, $ref)**: See [references/internal-references.md](references/internal-references.md)
- **Virtual fields ($field)**: See [references/virtual-fields.md](references/virtual-fields.md)

## Quick Patterns

```json
// Required email
"email|@ ~$Email~": "user@example.com"

// Required string 2-50 chars
"name|@ {2,50}": "Alice"

// Optional number range
"discount|(0..100)": 15

// Enum from nomenclature
"status|@ ($STATUS)": "ACTIVE"

// Array of unique strings, 1-10 items, each 2-20 chars
"tags|@ [1,10] -> {2,20}!": ["eco", "bio"]

// Required but nullable
"middleName|@ ?{1,50}": "Marie"

// Map with pattern keys
"translations|[~^[a-z]{2}$~:10]": {"en": "Hello", "fr": "Bonjour"}

// Reference to definition (required)
"address | $ref @": "&Address"

// Array of referenced type ($ref before ->, never after)
"items | $ref @ [1,100]": ["&OrderItem"]

// Array of referenced type with element constraint
"codes | $ref [1,10] -> {2,20}": ["&Code"]

// Object inheritance
"$ref": "&Auditable"

// Template pattern — $ref + $override inline in array element
"coding|[*]": [{
  "$ref": "&Coding",
  "code|$override @ ($MARITAL_STATUS)|Code": "M"
}]

// Property-level $ref (field IS the type, no specialization)
"period|$ref||Validity period": "&Period"

// $amend — adapt inherited field keeping base constraints
"email|$amend @": "alice@corp.com"

// Parameterized compute — one rule, many categories
"$compute": { "RateCheck(cat)": "category != cat || rate > 0" }
"rate|(%RateCheck('S'))": 20

// List access with field chain
"$compute": { "FirstPaid": "findFirst(payments, status == 'OK').amount > 0" }
```

## Complete Example — E-commerce Order

```json
{
  "$version": "1.0.0",
  "$id": "ecommerce.order",
  "$title": "E-commerce order",
  "$description": "E-commerce order - Full Okyline features showcase",
  "$oky": {
    "orderId|# ~^ORD-[0-9]{8}$~|Order identifier": "ORD-20250107",
    "createdAt|@ ~$DateTime~|Creation date": "2025-01-07T14:30:00Z",
    "//createdBy|~^[A-Z]{2}[0-9]{5}$~|Created By": "PF97877",
    "customer|@|Customer info": {
      "id|# (>0)": 42,
      "email|@ ~$Email~": "alice@example.com",
      "phone|? ~$Phone~|Optional phone": "+33612345678",
      "type|@ ($CUSTOMER_TYPE)": "PREMIUM",
      "$appliedIf type": {
        "('PREMIUM')": {
          "loyaltyPoints|@ (>=0)": 1500,
          "discountRate|@ (0..30)": 15
        },
        "('BUSINESS')": {
          "companyName|@ {2,100}": "Acme Corp",
          "vatNumber|@ ~$VatNumber~": "FR12345678901"
        }
      }
    },
    "shipping|@ $ref|Shipping address": "&Address",
    "billing|? $ref|Billing if different": "&Address",
    "lines|@ [1,50] -> !|Order lines": [
      {
        "sku|# ~$Sku~": "PRD-00123",
        "name|@ {2,200}": "Wireless Headphones",
        "quantity|@ (1..999)": 2,
        "unitPrice|@ (>0)": 79.99,
        "lineTotal|(%LineTotal)": 159.98,
        "category|($CATEGORY)": "ELECTRONICS"
      }
    ],
    "payment|@": {
      "method|@ ($PAYMENT_METHOD)": "CARD",
      "status|@ ($PAYMENT_STATUS)": "PAID",
      "$requiredIf status('PAID')": ["paidAt", "transactionId"],
      "$forbiddenIf status('PENDING')": ["paidAt", "transactionId"],
      "paidAt|~$DateTime~": "2025-01-07T14:32:00Z",
      "transactionId|~$TransactionId~": "TXN-A1B2C3D4E5F6"
    },
    "amounts|@|Amounts": {
      "subtotal|@ (%ValidSubtotal)": 159.98,
      "shippingCost|@ (>=0)": 5.99,
      "discount|@ (>=0)": 24.0,
      "tax|@ (>=0)": 28.39,
      "total|@ (%ValidTotal)": 170.36
    },
    "status|@ ($ORDER_STATUS)": "CONFIRMED",
    "tags|? [0,10] -> {1,30}!|Optional tags": ["gift", "express"],
    "notes|? {0,500}|Internal notes": "Handle with care"
  },
  "$defs": {
    "Address": {
      "street|@ {5,200}": "123 Main Street",
      "city|@ {2,100}": "Paris",
      "postalCode|@ ~^[0-9]{5}$~": "75001",
      "country|@ ~^[A-Z]{2}$~": "FR"
    }
  },
  "$format": {
    "Phone": "^\\+[0-9]{11,14}$",
    "VatNumber": "^[A-Z]{2}[0-9]{9,12}$",
    "TransactionId": "^TXN-[A-Z0-9]{12}$",
    "Sku": "^[A-Z]{3}-[0-9]{5}$"
  },
  "$nomenclature": {
    "CUSTOMER_TYPE": "STANDARD, PREMIUM, BUSINESS",
    "CATEGORY": "ELECTRONICS, CLOTHING, HOME, FOOD, OTHER",
    "PAYMENT_METHOD": "CARD, PAYPAL, TRANSFER, CRYPTO",
    "PAYMENT_STATUS": "PENDING, PAID, FAILED, REFUNDED",
    "ORDER_STATUS": "DRAFT, CONFIRMED, SHIPPED, DELIVERED, CANCELLED"
  },
  "$compute": {
    "LineTotal": "lineTotal == quantity * unitPrice",
    "ValidSubtotal": "subtotal > 0 && subtotal == sum(parent.lines, lineTotal)",
    "ValidTotal": "total > 0 && total == subtotal + shippingCost + tax - discount"
  }
}
```

## Document Metadata (Optional) - Must be generated in this order

```json
{
  "$okylineVersion": "1.5.0",
  "$version": "1.0.0",
  "$id": "my-schema",
  "$title": "My Schema",
  "$description": "Schema description",
  "$additionalProperties": false,
  "$sequence": false,
  "$oky": {
    ...
  },
  "$defs": { ... },
  "$format": { "Code": "^[A-Z]{3}-\\d{4}$" },
  "$compute": { "Total": "price * quantity" },
  "$nomenclature": { "STATUS": "ACTIVE,INACTIVE" }
}


**`$id` format**: Only letters, digits, underscores and dots allowed. Pattern: `^[a-zA-Z][a-zA-Z0-9_]*(\.[a-zA-Z][a-zA-Z0-9_]*)*$`
 ❌ `"personne-vehicules"` (hyphen not allowed)
 ✅ `"personne.vehicules"` or `"personne_vehicules"`
```


## Structural Group Directives

Constrain which fields from a group must be present. No condition required.

```json
"$atLeastOne": ["email", "phone"]               // at least one present
"$mutuallyExclusive": ["deceasedBoolean", "deceasedDateTime"]  // at most one
"$exactlyOne": ["diagnosisCode", "diagnosisRef"]  // exactly one
"$allOrNone": ["street", "city", "zip"]         // all or none

// Multiple independent groups → use suffix
"$mutuallyExclusive_deceased": ["deceasedBoolean", "deceasedDateTime"],
"$mutuallyExclusive_birth": ["multipleBirthBoolean", "multipleBirthInteger"]
```

See `references/conditional-directives.md` for full details.

## $compute — Cross-Collection Pattern

To validate a constraint across an array from root context, attach the compute to the array field itself:

```json
"insurance|@ [1,*] -> ! (%FocalInsurance)": [{
  "focal|@": true, ...
}],
"$compute": {
  "FocalInsurance": "countIf(insurance, focal) == 1"
}
```

- `countIf(collection, expr)` — counts elements where expr is truthy
- For boolean fields: `countIf(insurance, focal)` counts `true` values
- For other fields: `countIf(items, status == 'ACTIVE')`
- Always attach the compute to the most semantically relevant field

## Common Mistakes to Avoid

- Missing `$oky` wrapper
- Empty arrays (need at least one element)
- Confusing `[1,10]` (array size) with `-> {1,10}` (element constraint)
- Forgetting to escape backslashes in regex (`\d` → `\\d`)
- Defining `$compute` expressions without attaching them to fields with `|(%Name)`
- Using absolute paths in `$compute` instead of relative references to parent context
- Using `$ref` without the `&` prefix (correct: `"&Address"`, wrong: `"Address"`)
- Object-level `$ref` targeting a scalar definition (must target object schema)
- Redefining an inherited field without using `$override` or `$amend`
- Creating circular object-level references (A includes B includes A)
- Placing `$defs` inside `$oky` (must be at root level)
- Using empty brackets `[]` for arrays — use `[*]` or omit size constraint entirely
- Applying element constraints (enum, length, range) directly to array field instead of using `->`:
  ❌ `"permis|@ ('A','B','C')[]": ["B"]`
  ✅ `"permis|@ -> ('A','B','C')": ["B"]`
  ✅ `"permis|@ [1,5] -> ('A','B','C')": ["B"]`
- Using hyphens in `$id` — only letters, digits, underscores and dots are allowed
- Placing `$ref` after `->` for arrays of references (`$ref` is a type indicator, not a validation constraint):
  ❌ `"children|[*] -> $ref": ["&Node"]`
  ✅ `"children|[*] $ref": ["&Node"]`
  ✅ `"children|[1,10] $ref -> {2,50}": ["&Node"]`
- Using open-ended range syntax `(0..)` or `(..100)` — these don't exist!
  ❌ `"price|(0..)": 29.99`
  ✅ `"price|(>=0)": 29.99`
 - Using multiple value constraint blocks `(...)` on the same field:
  ❌ `"montantTTC|@ (>=0) (%LigneTTC)": 4320`
  ✅ `"montantTTC|@ (%LigneTTC)": 4320`
 - Using null as an example value, even with ? Okyline needs a valid example to infer the type.:
  ❌ `"name|? ": null`
  ✅ `"name|? ": "Charles"`
 - Using decimal number example ending in .00 without quotes.
   ❌ `"amount|? ": 800.00`
  ✅ `"amount|? ": "800.00"`
  
  When a field uses `$compute`, ALL value constraints must be inside the compute expression:
```json
  "$compute": {
    "LigneTTC": "montantTTC >= 0 && montantTTC == montantNetHT + montantTVA"
  }
```

## Silent errors (no syntax error, but validation problem)

- [ ] Decimals ending in `.00` → use `"150.00"` (otherwise inferred as Integer)
- [ ] Example `null` → cannot infer type
- [ ] Empty array `[]` → cannot infer element type
- [ ] `"field|Label"` → "Label" parsed as constraint (use `"field| |Label"`)
