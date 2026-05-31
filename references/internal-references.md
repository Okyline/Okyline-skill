# Okyline Internal References — `$defs` and `&Name`

## Overview

Okyline supports **internal schema references** to promote reuse and consistency. References allow reusing schema fragments defined in `$defs`.

Two use cases:
1. **Property-level reference** — a field whose value `&Name` (or `["&Name"]`) qualifies the field as having the type of the referenced definition
2. **Object-level reference** — an object that includes all fields from another schema (inheritance), via the special key `$ref` inside the object

References are an **inclusion mechanism**: referenced schemas can be extended, overridden, or partially removed.

---

## `$defs` — Definition Repository

`$defs` is a container for reusable schema fragments, placed at the root level like the name `$oky`.

### Basic Structure

```json
{
  "$oky": {
    
    "person": {
      "$ref": "&Address",
      "name|@ {2,50}": "Dupond"
    }
  },
  "$defs": {
      "Address": {
        "street|@ {2,100}": "12 rue du Saule",
        "city|@ {2,100}": "Lyon"
      }
  }
}
```

### Scalar Definitions

`$defs` supports both object schemas and scalar type definitions:

```json
{
  "$defs": {
    "Email|~$Email~ {5,100}": "user@example.com",
    "Percentage|(0..100)": 50,
    "Address": {
      "street|@ {2,100}": "12 rue du Saule",
      "city|@ {2,100}": "Lyon"
    }
  }
}
```

### Rules

- `$defs` is optional
- Entries are **not** JSON properties of validated instances — they are reusable definitions only
- References target the **first level** of `$defs` only (no nested paths)

### Collection Definitions

The key of a `$defs` entry accepts the same constraint grammar as a field key (excluding `@` and `#`). A definition can therefore name a **collection type** (list or map, including leaf constraints) and be reached via `&Name`:

```json
{
  "$oky": { "matrix|@ [*]": ["&Row"] },
  "$defs": {
    "Row|[*] -> (>=0)": [0]
  }
}
```
↳ `matrix` is `List<List<int>>`. Chains further (`&Cell` → `&Row` → `&Matrix`) to compose map-of-map, list-of-map, etc. at any depth.

---

## Reference Syntax — `&Name`

Internal definitions are referenced via:

```
&Name
```

Where `Name` is defined in `$defs`.

- `&` denotes the current document's definition namespace
- Resolution is **case-sensitive**

Examples: `&Address`, `&Email`, `&Person`

---

## Property-Level References

### Basic Syntax

A field uses another schema as its type when its value is `&Name`:

```json
{
  "$oky": {
    "person": {
      "address": "&Address",
      "name|@ {2,50}": "Dupond"
    }
  },
  "$defs": {
    "Address": {
      "street|@ {2,100}": "12 rue du Saule",
      "city|@ {2,100}": "Lyon"
    }
  }
}
```

The field behaves as if the referenced schema had been written inline.

### Target Types

A property-level reference MAY target:
- a scalar schema (number, string, boolean)
- an object schema
- an array schema

### Lists and Maps of Referenced Elements

If the value is a **single-element array containing a reference**, the field is a list:

```json
{
  "$oky": {
    "addresses": ["&Address"]
  },
  "$defs": {
    "Address": {
      "street|@ {2,100}": "12 rue du Saule",
      "city|@ {2,100}": "Lyon"
    }
  }
}
```

Array size constraints are added as standard structural constraints:

```json
"addresses|[1,10]": ["&Address"]
```

A **map** field works the same way — a single-entry object whose value is a reference:

```json
"statsByRegion|[*:*]": { "region-1": "&Stat" }
```

### Polymorphic References

A list or map field whose example holds **more than one** `&Name` (optionally mixed with inline values) is polymorphic — each element / value matches any of the variants:

```json
"events|@ [*]":       ["&Login", "&Logout"]
"indicators|@ [*:*]": { "k1": "&Counter", "k2": "&Gauge" }
```

All variants MUST be the same kind — all scalar **or** all object; mixing fails at load.

### Constraint Categories

#### Structural Constraints (local, at usage)

NOT inherited — defined at each usage point:

| Constraint | Description |
|------------|-------------|
| `@` | Required |
| `?` | Nullable |
| `[min,max]` | List size |
| `!` | Uniqueness in list |
| `%` | Default value |
| Label | Field description |

#### Value Constraints (inherited)

Inherited from the referenced schema. Modified only via `$override`:

| Constraint | Description |
|------------|-------------|
| `#` | Key field(s) |
| `{min,max}` | String length |
| `(min..max)` | Numeric range |
| `('A','B')` | Enumeration |
| `~pattern~` | Regex/format |
| `(%Compute)` | Computed validation |

Example:

```json
{
  "$oky": {
    "user": {
      "primaryEmail|@": "&Email",
      "backupEmail|?": "&Email"
    }
  },
  "$defs": {
    "Email|~$Email~ {5,100}": "user@example.com"
  }
}
```

- `~$Email~` and `{5,100}` → inherited from `Email`
- `@` vs `?` → defined locally per usage

---

## Object-Level References — Inheritance

### Basic Inclusion

An object includes another schema as a base using a top-level `$ref` field:

```json
{
  "$oky": {
    "Person": {
      "$ref": "&Address",
      "name|@ {2,50}": "Dupond"
    }
  },
  "$defs": {
    "Address": {
      "street|@ {2,100}": "12 rue du Saule",
      "city|@ {2,100}": "Lyon"
    }
  }
}
```

All fields from the referenced schema are **injected** into the current object.

Effective `Person`:
```json
{
  "street|@ {2,100}": "12 rue du Saule",
  "city|@ {2,100}": "Lyon",
  "name|@ {2,50}": "Dupond"
}
```

### Rules

- Object-level `$ref` **MUST** target an **object schema** (not scalar)
- Definitions containing conditional rules (`$requiredIf`, etc.) or `$compute` expressions **can be inherited** via object-level `$ref`, provided `$remove` is not used on that inclusion.
- Object-level `$ref` targets **exactly one** template (single reference string, not an array).

### Field Collision Rules

| Situation | Result |
|-----------|--------|
| Local field same name as inherited (no `$override` or `$amend`) | Schema rejected |

### Cycles

- **Object-level cycles** (A includes B which includes A) → **forbidden**
- **Property-level recursion** (A has a property of type A) → **allowed**

---

## `$remove` — Excluding Inherited Fields

Exclude fields inherited from referenced schemas:

```json
{
  "$oky": {
    "AnonymousPerson": {
      "$ref": "&Person",
      "$remove": ["email", "ssn"]
    }
  },
  "$defs": {
    "Person": {
      "name|@ {1,50}": "John",
      "age|@ (0..150)": 42,
      "email|@ ~$Email~": "john@example.com",
      "ssn|@": "123-45-6789"
    }
  }
}
```

Effective schema:
```json
{
  "name|@ {1,50}": "John",
  "age|@ (0..150)": 42
}
```

### Rules

- `$remove` MUST be an array of field names
- Each field MUST exist in the referenced schema (otherwise → rejected)
- `$remove` MUST NOT be used when the referenced schema contains conditional rules (`$requiredIf`, etc.) or `$compute` expressions

---

## `$override` and `$amend` — Adapting Inherited Fields

Two directives to adapt a field inherited from a template:

- **`$override`** — replaces the field entirely. Unspecified blocks are **erased**.
- **`$amend`** — replaces only the specified blocks. Unspecified blocks are **kept from the base**.

Both preserve field type, collection nature and reference target (structural invariants).

```json
{
  "$oky": {
    "Employee": {
      "$ref": "&Person",
      "name | $amend @": "John Doe",
      "salary|@ (>=0)": 3000
    }
  },
  "$defs": {
    "Person": {
      "name|? {1,50}": "John",
      "age|@ (0..150)": 42
    }
  }
}
```

Result: `name` becomes `@? {1,50}` — the `@` flag is added by `$amend`, the `?` and `{1,50}` are kept from the base. Using `$override @` instead would yield just `name|@` (everything else erased).

### Rules

- `$override` and `$amend` MUST target a field that exists in the referenced schema (after removals), or a parent-level field in an `$appliedIf` branch
- `$override` and `$amend` MUST NOT appear on the same field
- Without either directive, redefining an inherited or parent-level field → error
- In an `$appliedIf` branch, redefining a parent field requires explicit `$override` or `$amend`

---

## Order of Application

1. **Reference injection** — Resolve the single `$ref`, inject all fields
2. **Removals** — Apply `$remove`
3. **Adaptations** — Apply `$override` and `$amend` via block-by-block merge
4. **Local additions** — Add remaining local fields

---

## Error Conditions Summary

| Situation | Result |
|-----------|--------|
| `$ref` value is not a single reference string | Schema rejected |
| `$remove` targets non-existent field | Schema rejected |
| `$override` or `$amend` targets non-existent field (after removes) | Schema rejected |
| `$override` and `$amend` both on same field | Schema rejected |
| `$override` or `$amend` changes type, collection nature or reference target | Schema rejected |
| Local field collides with inherited (no `$override` or `$amend`) | Schema rejected |
| Object-level cycle detected | Schema rejected |
| Object-level `$ref` targets non-object schema | Schema rejected |
| Object-level `$ref` targets definition with conditional rules | Schema rejected |

---

## Quick Reference

| Feature | Syntax | Description |
|---------|--------|-------------|
| Definition repository | `$defs: { ... }` | Container for reusable schemas |
| Scalar definition | `"Name\|constraints": example` | Reusable scalar type |
| Object definition | `"Name": { fields }` | Reusable object schema |
| Reference syntax | `&Name` | Points to `$defs` entry |
| Property-level ref | `"field": "&Name"` | Field typed by definition |
| Property-level array | `"field": ["&Name"]` | Array of definition type |
| Object-level ref | `"$ref": "&Name"` | Include all fields from base |
| Remove field | `"$remove": ["field"]` | Exclude inherited field |
| Override field | `"field \| $override ..."` | Replace inherited definition |
| Amend field | `"field \| $amend ..."` | Adapt inherited field block-by-block |

---

## Template Pattern — Object-Level `$ref` + `$override` in Array Elements

A powerful pattern for typed structures that share a common base but need per-usage specialization. The object-level `$ref` is placed inside the array element object, not on the array field itself.

**Use case:** FHIR `Coding` — same structure everywhere, but `code` has different enum constraints per usage.

```json
{
  "$oky": {
    "maritalStatus": {
      "coding|[*]": [{
        "$ref": "&Coding",
        "code|$override @ ($MARITAL_STATUS)|Code": "M"
      }]
    },
    "gender": {
      "coding|[*]": [{
        "$ref": "&Coding",
        "code|$override @ ($GENDER)|Code": "male"
      }]
    }
  },
  "$defs": {
    "Coding": {
      "system|@ ~$Uri~|System": "http://example.org",
      "code|@ {1,50}|Code": "example",
      "display|{1,100}|Display": "Example"
    }
  }
}
```

**Key points:**
- `system` and `display` inherited from `&Coding` — validated once, applied everywhere
- Only `code` is overridden per usage to add the specific enum constraint
- The array field `coding|[*]` is a normal array — inheritance happens at element level
- Multiple fields can be overridden if needed

**Property-level reference vs object-level `$ref` — when to use which:**

| Pattern | Use when |
|---------|----------|
| `"period": "&Period"` | The field IS a Period — no specialization needed |
| `{ "$ref": "&Coding", "code\|$override ...": ... }` | The object EXTENDS a base — needs per-usage specialization |
