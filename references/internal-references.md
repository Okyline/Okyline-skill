# Okyline Internal References — `$defs` and `$ref`

## Overview

Okyline supports **internal schema references** to promote reuse and consistency. References allow reusing schema fragments defined in `$defs`.

Two use cases:
1. **Property-level reference** — a field whose type comes from a definition (`field | $ref`)
2. **Object-level reference** — an object that includes all fields from another schema (inheritance)

In Okyline, `$ref` is an **inclusion mechanism**: referenced schemas can be extended, overridden, or partially removed.

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

## Property-Level References — `field | $ref`

### Basic Syntax

A field uses another schema as its type via the `$ref` constraint:

```json
{
  "$oky": {
    "person": {
      "address | $ref": "&Address",
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

Property-level `$ref` MAY target:
- a scalar schema (number, string, boolean)
- an object schema
- an array schema

### Arrays of Referenced Elements

If the value is a **single-element array containing a reference**, the field is a list:

```json
{
  "$oky": {
    "addresses | $ref": [
      "&Address"
    ]
  },
  "$defs": {
    "Address": {
      "street|@ {2,100}": "12 rue du Saule",
      "city|@ {2,100}": "Lyon"
    }
  }
}
```

Array size constraints can be combined:

```json
"addresses | $ref [1,10]": ["&Address"]
```

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
      "primaryEmail | $ref @": "&Email",
      "backupEmail | $ref ?": "&Email"
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
- Definitions containing conditional rules (`$requiredIf`, etc.) or `$compute` **cannot be inherited** via object-level `$ref`

### Multiple Inheritance

An object can include multiple base schemas:

```json
{
  "$oky": {
    "Article": {
      "$ref": [
        "&Auditable",
        "&Deletable"
      ],
      "title|@ {1,200}": "Mon article"
    }
  },
  "$defs": {
    "Auditable": {
      "createdAt|@ ~$DateTime~": "2025-01-01T00:00:00Z",
      "updatedAt|@ ~$DateTime~": "2025-01-01T00:00:00Z"
    },
    "Deletable": {
      "deletedAt|~$DateTime~": "2025-01-08T00:00:00Z",
      "isDeleted|@": true
    }
  }
}
```

- References are applied in order (left to right)
- Field collision between base schemas → schema rejected (unless `$keep` or `$remove` resolves it)

### Field Collision Rules

| Situation | Result |
|-----------|--------|
| Local field same name as inherited (no `$override`) | Schema rejected |
| Two base schemas define same field (not removed, no `$keep`) | Schema rejected |

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
- Each field MUST exist in at least one referenced schema (otherwise → rejected)
- `$remove` applies to **all** inherited schemas

### Resolving Collisions with `$remove`

```json
{
  "$oky": {
    "Article": {
      "$ref": ["&Auditable", "&Deletable"],
      "$remove": ["updatedAt"],
      "updatedAt|@ ~$DateTime~": "2025-06-15T10:30:00Z",
      "title|@": "Mon article"
    }
  },
  "$defs": {
    "Auditable": {
      "createdAt|@": "2025-01-01T00:00:00Z",
      "updatedAt|@": "2025-01-01T00:00:00Z"
    },
    "Deletable": {
      "deletedAt|@": "2025-01-01T00:00:00Z",
      "updatedAt|@": "2025-01-01T00:00:00Z"
    }
  }
}
```

---

## `$keep` — Resolving Inheritance Conflicts

When multiple inheritance causes field collisions, `$keep` specifies which definition's field to retain:

```json
{
  "$oky": {
    "Combined": {
      "$ref": ["&A", "&B"],
      "$keep": ["&A.config"]
    }
  },
  "$defs": {
    "A": {
      "config|@": "value-from-A",
      "name|@": "A"
    },
    "B": {
      "config|@": "value-from-B",
      "status|@": "active"
    }
  }
}
```

### Rules

- `$keep` is an array of `&DefinitionName.fieldName`
- Only valid with multiple inheritance (`$ref` as array)
- Collisions on fields NOT in `$keep` → schema rejected
- `$keep` and `$remove` can be used together

---

## `$override` — Redefining Inherited Fields

Explicitly redefine an inherited field:

```json
{
  "$oky": {
    "Employee": {
      "$ref": "&Person",
      "name | $override @ {1,100}": "Jean Dupont",
      "salary|@ (>=0)": 3000
    }
  },
  "$defs": {
    "Person": {
      "name|@ {1,50}": "John",
      "age|@ (0..150)": 42
    }
  }
}
```

### Rules

- `$override` MUST target a field that exists in the referenced schema (after removals)
- The inherited definition is **completely replaced** by the local one
- Without `$override`, redefining an inherited field → error
- `$override` applies to **value constraints only** (structural constraints are always local)

---

## Order of Application

### Single Reference

1. **Reference injection** — Resolve `$ref`, inject all fields
2. **Removals** — Apply `$remove`
3. **Overrides** — Apply `$override`
4. **Local additions** — Add remaining local fields

### Multiple References

For `"$ref": ["&A", "&B", "&C"]`:

1. Inject A → Apply removes → current field set
2. Inject B → Apply removes → check collisions
3. Inject C → Apply removes → check collisions
4. Apply `$keep` to resolve remaining collisions
5. Apply overrides on final inherited set
6. Local additions (collision = error)

---

## Error Conditions Summary

| Situation | Result |
|-----------|--------|
| `$remove` targets non-existent field | Schema rejected |
| `$override` targets non-existent field (after removes) | Schema rejected |
| Local field collides with inherited (no `$override`) | Schema rejected |
| Two base schemas define same field (not removed, no `$keep`) | Schema rejected |
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
| Property-level ref | `"field \| $ref": "&Name"` | Field typed by definition |
| Property-level array | `"field \| $ref": ["&Name"]` | Array of definition type |
| Object-level ref | `"$ref": "&Name"` | Include all fields from base |
| Multiple inheritance | `"$ref": ["&A", "&B"]` | Include from multiple bases |
| Remove field | `"$remove": ["field"]` | Exclude inherited field |
| Keep on conflict | `"$keep": ["&A.field"]` | Choose which version to keep |
| Override field | `"field \| $override ..."` | Replace inherited definition |
