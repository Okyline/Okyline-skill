# Okyline Syntax Reference

## Scalar Field Constraints

### `@` - Required Field
Field MUST be present in validated documents.
```json
"name|@": "Alice"
// {"name": "Bob"} → ✅  |  {} → ❌
```

### `?` - Nullable Field
Field can contain `null` values. Can combine with `@` for required-but-nullable.
```json
"middleName|?": "Marie"        // Optional, nullable
"middleName|@ ?": "Marie"      // Required, but can be null
```

### `{...}` - String Length
Restricts character length (Unicode code points).
```json
"username|{3,20}": "alice"     // min 3, max 20
"city|{50}": "Paris"           // max 50 (no minimum)
"code|{5,5}": "ABC12"          // exactly 5
"bio|{3,*}": "hi there"        // min 3, no maximum
"notes|{*}": "free text"       // any length
```

### `(...)` - Value Constraints

**Numeric range (inclusive):**
```json
"age|(18..120)": 30            // 18 to 120 inclusive
"price|(0..1000)": 49.99
```

**Comparisons:**
```json
"quantity|(>0)": 5             // strictly greater than 0
"discount|(<=50)": 20          // less than or equal to 50
"score|(>=10)": 85             // greater than or equal to 10
```

**Discrete values (enum):**
```json
"status|('ACTIVE','INACTIVE','PENDING')": "ACTIVE"
"priority|(1,2,3,5,8)": 3      // numeric enum
```

**Lexicographic range:**
```json
"letter|('A'..'Z')": "B"       // single uppercase letter
```

**Combined (OR logic):**
```json
"value|(1,2..5,>10)": 12       // equals 1 OR 2-5 OR >10
```

**Nomenclature reference:**
```json
"color|($COLORS)": "RED"       // references $nomenclature
```

### `~...~` - Format Validation

**Inline regex (ECMA-262):**
```json
"postalCode|~^[0-9]{5}$~": "75001"
"phone|~^\\+33[0-9]{9}$~": "+33612345678"
```

**Named format reference:**
```json
"code|~$ProductCode~": "AB-1234"   // references $format block
"email|~$Email~": "user@test.com"  // built-in format
```

### `#` - Key Field
Marks field as identifier for object uniqueness in arrays.
```json
"users|[*] -> !": [
  {"id|#": "u1", "name": "Alice"},
  {"id|#": "u2", "name": "Bob"}
]
```

### `%` - Default Value (Informational)
Indicates example is also the default. Does not affect validation.
```json
"country|%": "France"
"theme|%('light','dark')": "light"
```

---

## Array Constraints

### `[...]` - Array Size
```json
"tags|[1,5]": ["eco"]          // 1 to 5 items
"codes|[10,*]": ["A"]          // at least 10 items
"letters|[5]": ["A"]           // max 5 items
"items|[*]": ["x"]             // any size
"tags|[0,10]": []              // empty example: elements untyped, any value accepted
```

An empty example (`[]`, or `{ }` for a map) leaves the elements untyped: only the size and the key pattern apply. A constraint that needs a type is rejected at load: `"tags|[0,10] -> {2,20}": []` and `"tags|[*] -> !": []` are errors. Prefer one real element when the elements have a type.

### `->` - Element Constraints
Applies constraints to each element.
```json
"tags|[1,5] -> {2,10}": ["eco"]           // each: 2-10 chars
"scores|[*] -> (0..100)": [85, 92]        // each: 0-100
"emails|[*] -> ~$Email~": ["a@b.com"]     // each: email format
```

### `!` - Uniqueness
All elements must be unique.
```json
// Scalar uniqueness - `!` goes after `->`, with the other element constraints
"codes|[*] -> !": ["A", "B", "C"]
"tags|[1,5] -> {2,10}!": ["eco", "bio"]

// Object uniqueness (by # key fields - a list of objects needs at least one `#`)
"items|[*] -> !": [
  {"sku|#": "ABC", "name": "Product A"},
  {"sku|#": "DEF", "name": "Product B"}
]
```

**Composite keys:** Multiple `#` fields form composite key (URL-encoded, hyphen-separated).

---

## Map Constraints

Maps are objects with dynamic keys. Syntax: `[key_pattern:max_entries]`

```json
// Any keys, max 5 entries
"metadata|[*:5]": {"author": "Alice", "version": "1.0"}

// Keys matching pattern, unlimited entries
"products|[~^SKU-\\d{5}$~:*]": {
  "SKU-12345": {"name|@": "Product A", "price|@": 29.99}
}

// Language codes, max 10 entries, values 1-100 chars
"labels|[~^[a-z]{2}$~:10] -> {1,100}": {"en": "Hello", "fr": "Bonjour"}
```

---

## Polymorphism

### `$oneOf` - Exclusive Match
Value must match exactly ONE schema. An array example is a **list** unless `$obj` says the field is a single value: write `$obj $oneOf` for one object, `[*] $oneOf` for a list whose elements each take one of the variants.
```json
"payment|@ $obj $oneOf": [
  {"type|@ ('card')": "card", "cardNumber|@ {16}": "1234567812345678"},
  {"type|@ ('paypal')": "paypal", "email|@ ~$Email~": "user@example.com"},
  {"type|@ ('bank')": "bank", "iban|@ {15,34}": "FR76..."}
]
```

### `$anyOf` - Non-Exclusive Match
Value must match at least one schema.
```json
"notification|$anyOf": [
  {"email|~$Email~": "user@example.com"},
  {"sms|~^\\+[0-9]{10,15}$~": "+33612345678"}
]
```

---

## Special Blocks

### `$nomenclature` - Reusable Enums
```json
{
  "$nomenclature": {
    "STATUS": "DRAFT,VALIDATED,REJECTED,ACTIVE,INACTIVE",
    "COUNTRIES": "FRA,DEU,ESP,USA,GBR"
  },
  "$oky": {
    "status|@ ($STATUS)": "ACTIVE",
    "country|($COUNTRIES)": "FRA"
  }
}
```

**Key-value form** (since 1.5.0) - associates a value with each key:
```json
{
  "$nomenclature": {
    "IbanLetters": "A:10,B:11,C:12,D:13,E:14,F:15,..."
  }
}
```
- The two forms cannot be mixed within a single entry.
- For validation (`($NAME)`), keys are the allowed values - both forms behave identically.
- Values are accessible via `lookup(key, '$NAME')` in expressions.

### `$format` - Reusable Patterns
```json
{
  "$format": {
    "PostalCode": "^[0-9]{5}$",
    "Sku": "^SKU-[A-Z]{3}[0-9]{5}$"
  },
  "$oky": {
    "zipCode|~$PostalCode~": "75001",
    "productSku|~$Sku~": "SKU-ABC12345"
  }
}
```

**Override built-ins:**
```json
{
  "$format": {
    "Date": "^(0[1-9]|[12]\\d|3[01])/(0[1-9]|1[0-2])/\\d{2}$"
  },
  "$oky": {
    "birthDate|~$Date~": "15/05/90"
  }
}
```

---

## Built-in Formats Reference

| Format | Validation | Example |
|--------|------------|---------|
| `$Date` | ISO 8601 date (semantic: validates leap years) | `"2025-05-30"` |
| `$DateTime` | ISO 8601 datetime (semantic) | `"2025-05-30T14:30:00Z"` |
| `$Time` | RFC 3339 time | `"14:30:00"` |
| `$Email` | Email address (syntactic) | `"user@example.com"` |
| `$Uri` | URI with scheme + port validation | `"https://example.com:8080"` |
| `$Ipv4` | IPv4 address | `"192.168.1.1"` |
| `$Ipv6` | IPv6 address | `"2001:db8::1"` |
| `$Uuid` | UUID v1-v5 | `"550e8400-e29b-..."` |
| `$Hostname` | RFC 1034 hostname | `"api.example.com"` |

---

## Document Structure

```json
{
  "$id": "namespace.schema_name",  // Unique identifier
  "$version": "1.0.0",             // Schema version (required only to publish)
  "$state": "DRAFT",               // "DRAFT" (default), "DRAFT-FINAL" or "FINAL"
  "$title": "Schema Title",
  "$description": "Description",
  "$additionalProperties": false,  // Reject unknown fields (default: false)
  "$sequence": false,              // Enforce field order (default: false)
  "$decimalScale": 6,              // Decimals carried by numbers in computes (default: 6)

  "$oky": { ... },                 // REQUIRED: schema definition ("$oky": null for a library of definitions)

  "$entries": { ... },             // Extra validation targets (see internal-references.md)
  "$defs": { ... },
  "$nomenclature": { ... },
  "$format": { ... },
  "$compute": { ... }
}
```

Important:
- `$id` format: `^[a-zA-Z][a-zA-Z0-9_]*(\.[a-zA-Z][a-zA-Z0-9_]*)*$` - letters, digits, underscores and dots; no hyphen; no leading, trailing or double dot.
- Root keys are closed: an unknown `$` key at the root fails loading. No order is imposed.
- `$okylineVersion` is indicative only: do not generate it.
- A `$nomenclature`, `$format` or `$compute` key may carry a label after `|`: `"STATUS | Account status": "ACTIVE,INACTIVE"`.

### Field names starting with `$` or `@`

A key starting with `$` is a directive, unless the first non-blank character after the initial word is a `|`: it is then a field, whose name may look like a directive. `@` needs no such mark. Both are reachable from conditions and expressions.

```json
"$oid|@ {24}": "507f1f77bcf86cd799439011"     // a MongoDB field named $oid
"@type|@ ('Person','Organization')": "Person" // a JSON-LD field named @type
"$appliedIf @type('Person')": { "birthDate|@ ~$Date~": "1990-05-15" }
```

### `$additionalProperties` Scope
- Root level: applies globally
- Object level: applies only to that object (not recursive, child objects keep the global setting)
- Not inherited from a template included by object-level `$ref`
- Default: `false` (unknown fields rejected)
- `$sequence` follows the same model

```json
{
  "$additionalProperties": false,
  "$oky": {
    "user": {
      "$additionalProperties": true,  // only applies to "user" object
      "name|@": "Alice"
    }
  }
}
```
