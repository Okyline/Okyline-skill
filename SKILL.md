---
name: okyline
description: >
  Expert assistant for the Okyline schema language - create, edit, and convert
  JSON validation schemas using declarative, example-driven syntax with inline constraints.
metadata:
  version: "2.3.0"
  author: Okyline
  repository: Okyline/Okyline-skill
---

# Okyline Schema Language v1.8.0

Okyline is a declarative language for describing and validating JSON structures using inline constraints on field names. A schema is a valid JSON document made of **real example values**: the example gives the type, the key carries the constraints.

## Before any schema generation

**MANDATORY**: read the reference files BEFORE producing an Okyline schema:

1. `references/syntax-reference.md` - complete syntax of constraints
2. `references/internal-references.md` - `$defs`, references (`&Name`), `$entries`
3. `references/conditional-directives.md` - if conditional logic
4. `references/expression-language.md` - if `$compute` is necessary
5. `references/virtual-fields.md` - if `$field` (virtual fields) is necessary
6. `references/external-imports.md` - ONLY if `$deps`/`$import` are explicitly requested or already present in a loaded schema

Never generate a schema based solely on this SKILL.md file. The examples here are a summary, not an exhaustive reference.

## What makes a schema fail to load

Since 1.8.0 the engine checks a schema when it is loaded, not only when data is validated. A schema that carries any of the following is **rejected**, even if the same text loaded under 1.7.0:

1. **A directive names a field that is not declared.** Every field cited by a trigger, a target list, `$required`, `$forbidden`, `$atLeastOne`, `$mutuallyExclusive`, `$exactlyOne` or `$allOrNone` must be declared in the same object. `$additionalProperties: true` does not help.
   ❌ `"$exactlyOne": ["email", "phon"]` when the object declares `phone`
   ✅ `"$exactlyOne": ["email", "phone"]`
2. **A condition literal has the wrong type.** No conversion, in either direction.
   ❌ `"$requiredIf active('true')": [...]` on a boolean ❌ `age('18')` on an integer ❌ `code(12)` on a string
   ✅ `active(true)`, `age(18)`, `code('12')`
3. **A condition uses a value the field can never take**: outside its nomenclature, enum, range or pattern. Each listed value is judged on its own.
   ❌ `"$appliedIf status('LEGACY')": {...}` when `$STATUS` is `ACTIVE,INACTIVE`
   ✅ use a value of `$STATUS`, or add `LEGACY` to it
4. **A value condition on an object or a collection.** Only a type guard may test such a field.
   ❌ `"$requiredIf address('X')": [...]` ❌ `"$requiredIf items(>0)": [...]`
   ✅ `"$requiredIf items(_ListOfObject_)": [...]`
5. **A path that crosses a list, or `parent` used at the root.**
   ❌ `"$requiredIf lines.total(>0)": [...]` ❌ `"$requiredIf parent.x('A')": [...]` written in the root object
   ✅ write the directive inside the list element: `"lines|[*]": [{ ..., "$requiredIf total(>0)": [...] }]`
6. **A `$` key in a conditional block that the block does not accept.** A block accepts field declarations, `$required`, `$forbidden`, the four structural groups, `$requiredIf*` / `$forbiddenIf*`, `$else` and `$notExist`. Nothing else.
   ❌ `"$appliedIf type('B')": { "$ref": "&Business" }` ❌ `{ "$additionalProperties": true }` in a branch
   ✅ declare the fields directly in the branch
7. **A typed constraint on an empty example.** An empty example leaves the elements untyped.
   ❌ `"tags|[0,10] -> {2,20}": []` ❌ `"tags|[*] -> !": []`
   ✅ `"tags|[0,10] -> {2,20}": ["eco"]` ✅ `"tags|[0,10]": []` (any element accepted)
8. **`$amend` on an object or a collection with a non-empty value.** The value is `{}` or `[]`, everything below comes from the base.
   ❌ `"address|$amend @": { "city|@": "Paris" }`
   ✅ `"address|$amend @": {}`
9. **A `$compute` the engine cannot resolve**: unknown function, wrong number of arguments, unknown compute, unknown rounding mode written as a literal, `matches()` with a format that is not written `'$Name'` or does not exist.
   ❌ `round(x, 2, "NEAREST")` ❌ `matches(ref, fmt)` ❌ `length()`
   ✅ `round(x, 2, "HALF_EVEN")` ✅ `matches(ref, '$Email')` ✅ `length(ref)`
10. **A field redefined in a branch without `$override` or `$amend`**, two `(...)` blocks on one field, `$oneOf` and `$anyOf` on the same field.
    ❌ `"$appliedIf type('B')": { "name|@ {2,100}": "…" }` when `name` is declared at the parent level
    ✅ `"$appliedIf type('B')": { "name|$amend {2,100}": "…" }`

## Core syntax

```
"fieldName | constraints | label": exampleValue
```

- **fieldName**: JSON field name
- **constraints**: validation rules, space-separated
- **label**: optional human-readable description
- **exampleValue**: determines the inferred type

A label without constraints needs the empty middle slot:

  ❌ `"buyer|Client"` → "Client" parsed as a constraint
  ✅ `"buyer| |Client"` → "Client" is the label

A key starting with `$` is a directive, unless the first non-blank character after the initial word is a `|`: `"$oid|@ {24}"` is a field named `$oid` (MongoDB). A name starting with `@` needs no such mark: `"@type|@": "Person"` (JSON-LD).

## Minimal schema

```json
{
  "$oky": {
    "name|@ {2,50}|User name": "Alice",
    "email|@ ~$Email~": "alice@example.com",
    "age|(18..120)": 30
  }
}
```

## Essential constraints

| Symbol | Meaning | Example |
|--------|---------|---------|
| `@` | Required field | `"name\|@": "Alice"` |
| `?` | Nullable (can be `null`) | `"middle\|?": "John"` |
| `{min,max}` | String length | `"code\|{5,10}": "ABC123"` |
| `(min..max)` | Numeric range | `"age\|(18..65)": 30` |
| `(>0)` `(>=0)` `(<100)` `(<=100)` | One-sided bound | `"quantity\|(>0)": 5` |
| `('a','b')` | Enum values | `"status\|('ACTIVE','INACTIVE')": "ACTIVE"` |
| `($NAME)` | Nomenclature reference | `"status\|@ ($STATUS)": "ACTIVE"` |
| `~pattern~` | Regex (searched, not anchored: write `^…$`) | `"zip\|~^[0-9]{5}$~": "75001"` |
| `~$Name~` | Named or built-in format | `"email\|~$Email~": "a@b.com"` |
| `[min,max]` `[*]` `[1,*]` | Array size | `"tags\|[1,5]": ["eco"]` |
| `-> constraints` | Constraints on each element | `"tags\|[*] -> {2,10}": ["eco"]` |
| `-> !` | Unique elements (by `#` keys for objects) | `"codes\|[*] -> !": ["A","B"]` |
| `#` | Key field, for object uniqueness | `"id\|#": 123` |
| `[~pattern~:max]` `[*:*]` | Map: key pattern, max entries | `"i18n\|[~^[a-z]{2}$~:10]": {"en": "Hi"}` |
| `&Name` | Reference to a definition, as the value | `"address": "&Address"` |

Rules of thumb:
- ❌ Never invent open ranges `(0..)`, `(..100)`, `(1..*)` → ✅ `(>=0)`, `(<=100)`.
- ❌ Never use huge placeholder bounds `(0..99999999)` → ✅ `(>=0)`.
- ❌ Never omit a bound `[1,]` → ✅ `[1,*]`.
- Only **one** `(...)` block per field: put extra bounds inside the compute when a field carries `(%Name)`.
- Built-in formats: `$Date`, `$DateTime`, `$Time`, `$Email`, `$Uri`, `$Uuid`, `$Ipv4`, `$Ipv6`, `$Hostname`. A `$format` of the same name replaces the built-in.

## Type inference

- The example value gives the type: `42` → Integer, `3.14` → Number, `"text"` → String, `true` → Boolean, `{…}` → Object, `[…]` → List.
- **`null` is never an example.** Use `?` with a real value: `"middleName|?": "Marie"`.
- **Decimals ending in `.00`**: serializers drop the zeros, `78.00` becomes `78` and is inferred Integer. Quote them: `"amount": "78.00"` → Number. `45.5` and `10` are unaffected.
- **Empty `[]` or `{ }`** is allowed and leaves the elements untyped: any value passes, only the size and key pattern apply. Prefer one real element when the elements have a type.
- **`$obj`**: an array example under `$obj` is a list of examples for a **single** value, not a list. `"payment|@ $obj $oneOf": [ {…}, {…} ]` is one object matching one variant; without `$obj` it is a list.

## `?` versus absence

| | Meaning | When to use |
|--|---------|-------------|
| no `@` | the field may be **absent** | optional fields in most APIs (FHIR, REST) |
| `?` | the field may be explicitly **`null`** | only when `null` is a meaningful value |

```json
// ❌ ? for a merely optional field (common mistake)
"meta|?|Resource metadata": { ... }
// ✅ optional = no @, never ?
"meta| |Resource metadata": { ... }
// ✅ ? only when null carries meaning
"active|?|Explicitly unknown status": true
```

For legacy producers that send `null` instead of omitting a field, declare `"$nullAsAbsentIfUndeclared": true` at the root: a `null` on a field without `?` is then treated as absent. Default `false`.

## Decision tables

**When to use `$defs` + `&Name`:** recursion (the only way), a structure repeated identically (`Address`, `Period`), a template specialized per usage with `$override`/`$amend`, or an explicit request. Otherwise inline in `$oky`; do not over-abstract.

**Which conditional mechanism:**

| Need | Mechanism | Example |
|------|-----------|---------|
| A required when B has a value | `$requiredIf` | `"$requiredIf status('ACTIVE')": ["email"]` |
| A forbidden when B has a value | `$forbiddenIf` | `"$forbiddenIf status('CLOSED')": ["lastLogin"]` |
| A required when B exists / is absent | `$requiredIfExist` / `$requiredIfNotExist` | `"$requiredIfExist shipping": ["address"]` |
| Different fields per value of B | `$appliedIf` switch | `"$appliedIf paymentMethod": { "('CARD')": {...}, "('PAYPAL')": {...} }` |
| Extra fields when a condition holds | `$appliedIf` simple | `"$appliedIf status('ACTIVE')": { "workDays\|@": 20 }` |
| Exactly one of N present | `$exactlyOne` | `"$exactlyOne": ["email", "phone"]` |
| At most one of N | `$mutuallyExclusive` | `"$mutuallyExclusive": ["optionA", "optionB"]` |
| At least one of N | `$atLeastOne` | `"$atLeastOne": ["email", "phone", "fax"]` |
| All or none of a group | `$allOrNone` | `"$allOrNone": ["street", "city", "zip"]` |
| Condition on a computed value of the field | compute as operand | `"$requiredIf amount(%IsHigh)": ["approver"]` |
| Condition on a derived value not in the data | `$field` + `$appliedIf` | `"$field tier": "%ComputeTier"` then `"$appliedIf tier('GOLD')": {...}` |
| Validation depends on the runtime type | type guard | `"$appliedIf data(_String_)": {...}` |
| Condition on `null` | null literal | `"$requiredIf status(null)": ["fallback"]` |

**Sibling presence versus value structure:** `$exactlyOne` / `$mutuallyExclusive` constrain which **sibling fields** are present; `$oneOf` / `$anyOf` constrain the **shape of one field's value**.

**`$compute` versus `$field`:** `$compute` alone validates a field against a rule (`"total|(%CheckTotal)": 120.0`) or serves as a condition operand (`amount(%IsHigh)`). Introduce `$field` only when a directive needs a derived value that exists in no field.

## `$compute` in short

- An expression is evaluated in the object that **directly contains** the annotated field; sibling fields, arrays and nested objects are reachable by name, `parent.` and `root.` go up.
- `"total|(%CheckTotal)": 120.0` attaches the check to a field; the expression must yield a boolean. `%Name` references another compute; `%Name(args)` calls a parameterized one.
- Every number is rounded to `$decimalScale` decimals (default 6) after each operation, and `==` compares at that scale exactly. To compare an amount in cents, round it: `total == round(sum(lines, qty * price), 2)`.
- A container check goes **before** `->`: `"lines|[1,*] (%OneFocal)"`; a compute placed after `->` runs per element.

See `references/expression-language.md`.

## Common mistakes to avoid

Collected from real generation errors, most frequent first. Each ❌ has been produced by a model at least once.

1. **`?` on a merely optional field** - `?` means "may be `null`", absence needs nothing:
   ❌ `"meta|?|Resource metadata": { ... }`
   ✅ `"meta| |Resource metadata": { ... }`
2. **`null` as an example value** - no type can be inferred, even with `?`:
   ❌ `"name|?": null`
   ✅ `"name|?": "Charles"`
3. **A directive naming an undeclared field, or a value the field cannot take** - rejected at load since 1.8.0:
   ❌ `"$requiredIf status('LEGACY')": ["reason"]` with `"status|@ ($STATUS)"` and no `LEGACY` in `$STATUS`
   ❌ `"$exactlyOne": ["email", "phon"]` when the object declares `phone`
   ✅ declare the field, or use a value the field can take
4. **Element constraints on the array field instead of after `->`**:
   ❌ `"permis|@ ('A','B','C')[]": ["B"]`
   ✅ `"permis|@ -> ('A','B','C')": ["B"]`
   ✅ `"permis|@ [1,5] -> ('A','B','C')": ["B"]`
5. **`[1,10]` (array size) confused with `-> {1,10}` (element constraint)**:
   ❌ `"tags|{2,20}": ["eco"]`
   ✅ `"tags|[1,10] -> {2,20}": ["eco"]`
6. **`!` before `->`**:
   ❌ `"codes|[*]!": ["A","B"]`
   ✅ `"codes|[*] -> !": ["A","B"]`
7. **A decimal example ending in `.00` unquoted** - serialized as an integer, inferred Integer:
   ❌ `"amount|@": 800.00`
   ✅ `"amount|@": "800.00"`
8. **Two `(...)` blocks on one field** - when a field carries `(%Name)`, every value constraint goes inside the compute:
   ❌ `"montantTTC|@ (>=0) (%LigneTTC)": 4320`
   ✅ `"montantTTC|@ (%LigneTTC)": 4320` with `"LigneTTC": "montantTTC >= 0 && montantTTC == montantNetHT + montantTVA"`
9. **Open ranges or placeholder bounds**:
   ❌ `"price|(0..)": 29.99` ❌ `"price|(0..99999999)": 29.99`
   ✅ `"price|(>=0)": 29.99`
10. **A label without the empty constraint slot** - the label is parsed as a constraint:
    ❌ `"acheteur|Client": "…"`
    ✅ `"acheteur| |Client": "…"`
11. **A reference without `&`, or written after `->`**:
    ❌ `"address": "Address"` (a plain string example)
    ❌ `"children|[*] -> &Node": []`
    ✅ `"address": "&Address"` ✅ `"children|[*]": ["&Node"]` ✅ `"children|[1,10] -> {2,50}": ["&Node"]`
12. **`$oneOf` on an array example without `$obj`** when one object is meant - without `$obj` the field is a list:
    ❌ `"payment|@ $oneOf": [ {…}, {…} ]` for a single payment
    ✅ `"payment|@ $obj $oneOf": [ {…}, {…} ]`
13. **A `$compute` declared but attached nowhere, or attached after `->` when it checks the whole collection**:
    ❌ `"insurance|[1,*] -> ! (%FocalInsurance)"` (after `->` the compute runs on each element, not on the list)
    ✅ `"insurance|[1,*] (%FocalInsurance) -> !"`
14. **Redefining an inherited or parent-level field without `$override` / `$amend`**:
    ❌ `{ "$ref": "&Person", "name|@ {1,50}": "John" }` when `Person` declares `name`
    ✅ `{ "$ref": "&Person", "name|$amend @": "John" }`
15. **`$amend` on an object with a subtree** - the adapter's value is empty, everything below comes from the base:
    ❌ `"address|$amend @": { "city|@": "Paris" }`
    ✅ `"address|$amend @": {}`
16. **Structure errors**: `$defs` inside `$oky` (root level only); object-level `$ref` to a scalar definition; object-level cycles (A includes B includes A); `$ref`, `$remove` or `$additionalProperties` inside an `$appliedIf` branch.
17. **A hyphen in `$id`**:
    ❌ `"$id": "personne-vehicules"`
    ✅ `"$id": "personne.vehicules"`
18. **Regex slips**: unescaped backslashes (`\d` → `\\d`); no `^…$` when full coverage is meant (a pattern is searched, not anchored).
19. **Absolute paths in `$compute` instead of the containing object** - fields are reached by name from the object holding the annotated field; `parent.` and `root.` only when needed.
20. **A typed constraint on an empty example**:
    ❌ `"tags|[*] -> {2,20}": []`
    ✅ `"tags|[*] -> {2,20}": ["eco"]` (or `"tags|[*]": []` if the elements are untyped on purpose)

## Quick patterns

```json
"email|@ ~$Email~": "user@example.com"              // required email
"name|@ {2,50}": "Alice"                             // required string 2-50
"discount|(0..100)": 15                              // optional number in range
"status|@ ($STATUS)": "ACTIVE"                       // enum from nomenclature
"tags|@ [1,10] -> {2,20}!": ["eco", "bio"]           // 1-10 unique strings, 2-20 chars each
"middleName|@ ?{1,50}": "Marie"                      // required but nullable
"translations|[~^[a-z]{2}$~:10]": {"en": "Hello"}    // map with pattern keys
"address|@": "&Address"                              // reference, required
"items|@ [1,100]": ["&OrderItem"]                    // list of a referenced type
"period| |Validity period": "&Period"                // reference with a label
"$ref": "&Auditable"                                 // object-level inclusion
"email|$amend @": "alice@corp.com"                   // adapt an inherited field, keep its blocks
"code|$override @ ($MARITAL_STATUS)": "M"            // replace an inherited field
"address|$amend @": {}                               // adapt an inherited object: key only, value empty
"$oid|@ {24}": "507f1f77bcf86cd799439011"            // a data field named $oid
"$compute": { "RateCheck(cat)": "category != cat || rate > 0" }
"rate|(%RateCheck('S'))": 20                         // parameterized compute
"$requiredIf amount(%IsHigh)": ["approver"]          // compute as condition operand
```

## Complete example - e-commerce order

```json
{
  "$version": "1.0.0",
  "$id": "ecommerce.order",
  "$title": "E-commerce order",
  "$oky": {
    "orderId|# ~^ORD-[0-9]{8}$~|Order identifier": "ORD-20250107",
    "createdAt|@ ~$DateTime~|Creation date": "2025-01-07T14:30:00Z",
    "customer|@|Customer info": {
      "id|# (>0)": 42,
      "email|@ ~$Email~": "alice@example.com",
      "phone|? ~$Phone~|Optional phone": "+33612345678",
      "type|@ ($CUSTOMER_TYPE)": "PREMIUM",
      "$appliedIf type": {
        "('PREMIUM')": { "loyaltyPoints|@ (>=0)": 1500, "discountRate|@ (0..30)": 15 },
        "('BUSINESS')": { "companyName|@ {2,100}": "Acme Corp", "vatNumber|@ ~$VatNumber~": "FR12345678901" }
      }
    },
    "shipping|@|Shipping address": "&Address",
    "billing|?|Billing if different": "&Address",
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
      "discount|@ (>=0)": "24.00",
      "tax|@ (>=0)": 28.39,
      "total|@ (%ValidTotal)": 170.36
    },
    "status|@ ($ORDER_STATUS)": "CONFIRMED",
    "tags|? [0,10] -> {1,30}!|Optional tags": ["gift", "express"]
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
    "CUSTOMER_TYPE": "STANDARD,PREMIUM,BUSINESS",
    "CATEGORY": "ELECTRONICS,CLOTHING,HOME,FOOD,OTHER",
    "PAYMENT_METHOD": "CARD,PAYPAL,TRANSFER,CRYPTO",
    "PAYMENT_STATUS": "PENDING,PAID,FAILED,REFUNDED",
    "ORDER_STATUS": "DRAFT,CONFIRMED,SHIPPED,DELIVERED,CANCELLED"
  },
  "$compute": {
    "LineTotal": "lineTotal == round(quantity * unitPrice, 2)",
    "ValidSubtotal": "subtotal > 0 && subtotal == round(sum(parent.lines, lineTotal), 2)",
    "ValidTotal": "total > 0 && total == round(subtotal + shippingCost + tax - discount, 2)"
  }
}
```

## Root keys

Only `$oky` is required. Recommended order, for readability only (no rule imposes one):

```json
{
  "$id": "my.schema",
  "$version": "1.0.0",
  "$state": "DRAFT",
  "$title": "My Schema",
  "$description": "Schema description",
  "$additionalProperties": false,
  "$sequence": false,
  "$decimalScale": 6,
  "$oky": { ... },
  "$entries": { "Create": "&Item", "Update": "&ItemPatch" },
  "$defs": { ... },
  "$format": { "Code": "^[A-Z]{3}-\\d{4}$" },
  "$compute": { "Total": "price * quantity" },
  "$nomenclature": { "STATUS": "ACTIVE,INACTIVE" }
}
```

- **`$id`**: letters, digits, underscores and dots, `^[a-zA-Z][a-zA-Z0-9_]*(\.[a-zA-Z][a-zA-Z0-9_]*)*$`. No hyphen.
- **`$version`**: Semver 2.0.0. Required only to publish.
- **`$state`**: `"DRAFT"` (default), `"DRAFT-FINAL"` or `"FINAL"`. Omit unless the lifecycle matters.
- **`$decimalScale`**: integer, root level only, default `6`.
- **`$additionalProperties`** / **`$sequence`**: root or object level; a local value applies to that object only, never to its children, and is not inherited from an included template.
- **`$entries`**: extra validation entry points, values are local `&Name` references. A library with no root document writes `"$oky": null` and is validated through its entries.
- **`$okylineVersion`**: indicative only, do not generate it.
- A `|` after a `$nomenclature`, `$format` or `$compute` key adds a label: `"STATUS | Account status": "ACTIVE,INACTIVE"`.

## Structural group directives

```json
"$atLeastOne": ["email", "phone"]
"$mutuallyExclusive": ["deceasedBoolean", "deceasedDateTime"]
"$exactlyOne": ["diagnosisCode", "diagnosisRef"]
"$allOrNone": ["street", "city", "zip"]

// several groups of the same directive: add a suffix (structural groups only)
"$mutuallyExclusive_deceased": ["deceasedBoolean", "deceasedDateTime"],
"$mutuallyExclusive_birth": ["multipleBirthBoolean", "multipleBirthInteger"]
```

## Cross-collection check

Attach the compute to the array field, **before** `->`, so that it runs once on the whole collection:

```json
{
  "$oky": {
    "insurance|@ [1,*] (%FocalInsurance) -> !": [{ "id|#": "INS-1", "focal|@": true }]
  },
  "$compute": { "FocalInsurance": "countIf(insurance, focal) == 1" }
}
```

## Before delivering a schema, check

- [ ] `$oky` wrapper present; `$defs`, `$format`, `$nomenclature`, `$compute` at root level
- [ ] Every required field carries `@`; `?` only where `null` is meaningful; no `null` example
- [ ] Every field named by a directive or a group is declared in the same object
- [ ] Every condition value is of the field's type and inside its nomenclature, enum, range or pattern
- [ ] Array constraints: size in `[...]`, element constraints and `!` after `->`
- [ ] Decimals ending in `.00` are quoted
- [ ] One `(...)` block per field; a `(%Name)` compute carries every value bound inside the expression
- [ ] Labels use `| |Label` when there is no constraint
- [ ] References are values starting with `&`; a container compute is placed before `->`
- [ ] `$id` without hyphen; no `$okylineVersion`
- [ ] Every `$compute` is attached to a field or used as a condition operand
- [ ] Regex backslashes escaped, `^…$` where full coverage is meant
