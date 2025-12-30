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
```

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
// Scalar uniqueness
"codes|[*]!": ["A", "B", "C"]

// Object uniqueness (by # key fields)
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
Value must match exactly ONE schema.
```json
"payment|@ $oneOf": [
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
  "$okylineVersion": "1.1.0",      // Okyline spec version
  "$version": "1.0.0",             // Schema version (required for registry)
  "$id": "namespace.schema-name",  // Unique identifier, 
  "$title": "Schema Title",
  "$description": "Description",
  "$additionalProperties": false,  // Reject unknown fields (default: false)
  
  "$oky": { ... }                  // REQUIRED: schema definition
  
  "$nomenclature": { ... },
  "$format": { ... },
  "$compute": { ... },
  
}

Important : 
- `$id` format: `^[a-zA-Z][a-zA-Z0-9_]*(\.[a-zA-Z][a-zA-Z0-9_]*)*$`
    - Must not start or end with `.`
    - Must not contain consecutive dots (`..`)
    - Each segment must start with a letter
```

### `$additionalProperties` Scope
- Root level: applies globally
- Object level: applies only to that object (not recursive)
- Default: `false` (unknown fields rejected)

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
