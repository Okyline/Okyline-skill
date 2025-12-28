# Okyline Conditional Directives

Conditional directives apply structural changes based on field values or existence.

## Summary Table

| Directive | Condition Type | Purpose |
|-----------|----------------|---------|
| `$requiredIf` / `$requiredIfNot` | Field value | Make fields required |
| `$requiredIfExist` / `$requiredIfNotExist` | Field existence | Make fields required |
| `$forbiddenIf` / `$forbiddenIfNot` | Field value | Forbid fields |
| `$forbiddenIfExist` / `$forbiddenIfNotExist` | Field existence | Forbid fields |
| `$appliedIf` / `$appliedIfNot` | Field value | Add structure (if/switch) |
| `$appliedIfExist` / `$appliedIfNotExist` | Field existence | Add structure |

---

## Value-Based Conditions

### `$requiredIf` / `$requiredIfNot`

```json
{
  "$oky": {
    "accountType|@ ('PERSONAL','BUSINESS')": "BUSINESS",
    "taxId": "123456789",
    "personalId": "AB123",
    
    "$requiredIf accountType('BUSINESS')": ["taxId"],
    "$requiredIfNot accountType('BUSINESS')": ["personalId"]
  }
}
```
- If `accountType` is `'BUSINESS'` → `taxId` required
- If `accountType` is NOT `'BUSINESS'` → `personalId` required

**Multiple values:**
```json
"$requiredIf status('SHIPPED','DELIVERED')": ["trackingNumber"]
```

**Numeric conditions:**
```json
"$requiredIf age(<18)": ["parentConsent"]
"$requiredIfNot age(<18)": ["idCard"]
```

### `$forbiddenIf` / `$forbiddenIfNot`

```json
{
  "$oky": {
    "status|@ ('ACTIVE','CLOSED')": "CLOSED",
    "lastLogin": "2025-01-15",
    "closureReason": "User requested",
    
    "$forbiddenIf status('CLOSED')": ["lastLogin"],
    "$forbiddenIfNot status('CLOSED')": ["closureReason"]
  }
}
```
- If `status` is `'CLOSED'` → `lastLogin` must NOT be present
- If `status` is NOT `'CLOSED'` → `closureReason` must NOT be present

### `$appliedIf` - Simple If/Else

```json
{
  "$oky": {
    "status|@ ('ACTIVE','ON_LEAVE')": "ACTIVE",
    "$appliedIf status('ACTIVE')": {
      "workDays|@ (1..22)": 20
    },
    "$else": {
      "leaveReason|@": "Vacation"
    }
  }
}
```

### `$appliedIf` - Switch-Case Mode

```json
{
  "$oky": {
    "paymentMethod|@ ('CARD','BANK','PAYPAL')": "CARD",
    "$appliedIf paymentMethod": {
      "('CARD')": {
        "cardNumber|@ {16}": "1234567812345678",
        "cvv|@ {3}": "123"
      },
      "('BANK')": {
        "iban|@ {15,34}": "FR7612345678901234567890123",
        "bic|@ {8,11}": "BNPAFRPP"
      },
      "('PAYPAL')": {
        "paypalEmail|@ ~$Email~": "user@example.com"
      },
      "$else": {
        "note|@": "Unknown payment method"
      }
    }
  }
}
```

---

## Existence-Based Conditions

### `$requiredIfExist` / `$requiredIfNotExist`

```json
{
  "$oky": {
    "firstName": "John",
    "lastName": "Doe",
    "email": "john@example.com",
    "phone": "+33612345678",
    
    "$requiredIfExist firstName": ["lastName"],
    "$requiredIfNotExist email": ["phone"]
  }
}
```
- If `firstName` exists → `lastName` required
- If `email` does NOT exist → `phone` required

### `$forbiddenIfExist` / `$forbiddenIfNotExist`

```json
{
  "$oky": {
    "archived": true,
    "sku": "SKU-12345",
    "internalCode": "INT-999",
    
    "$forbiddenIfExist archived": ["active"],
    "$forbiddenIfNotExist sku": ["internalCode"]
  }
}
```
- If `archived` exists → `active` must NOT be present
- If `sku` does NOT exist → `internalCode` must NOT be present

### `$appliedIfExist` / `$appliedIfNotExist`

```json
{
  "$oky": {
    "tracking": "ABC123",
    "$appliedIfExist tracking": {
      "carrier|@": "DHL",
      "estimatedDelivery|@ ~$Date~": "2025-12-25"
    },
    "$appliedIfNotExist email": {
      "phone|@": "+33612345678",
      "phoneVerified|@": true
    }
  }
}
```

---

## Type Guards

Type Guards check the **runtime type** of a field value in conditions.

### Available Type Guards

| Type Guard | Description |
|------------|-------------|
| `_Null_` | Value is null |
| `_Boolean_` | Value is boolean |
| `_String_` | Value is string |
| `_Integer_` | Value is integer (no fractional part) |
| `_Number_` | Value is number (integer or decimal) |
| `_Object_` | Value is object |
| `_EmptyList_` | Value is empty array |
| `_ListOfNull_` | Array of only null values |
| `_ListOfBoolean_` | Array of booleans |
| `_ListOfString_` | Array of strings |
| `_ListOfInteger_` | Array of integers |
| `_ListOfNumber_` | Array of numbers |
| `_ListOfObject_` | Array of objects |

### Type Guard Examples

```json
{
  "$oky": {
    "data": "example",
    "$appliedIf data(_String_)": {
      "length": 7
    },
    "$else": {
      "type": "non-string"
    }
  }
}
```

**Multiple type guards (OR logic):**
```json
{
  "$oky": {
    "value": null,
    "$appliedIf value(_String_,_Null_)": {
      "isTextOrEmpty": true
    }
  }
}
```

**Array type check:**
```json
{
  "$oky": {
    "items": [1, 2, 3],
    "$requiredIf items(_ListOfInteger_)": ["sum"]
  }
}
```

### Type Guard Notes

- `_Integer_` is stricter than `_Number_`: `3.0` matches `_Number_` but NOT `_Integer_`
- `_Number_` includes integers: `42` matches both `_Integer_` and `_Number_`
- `_ListOfXXX_` guards ignore null elements for type inference
- Multiple guards use OR logic

---

## Combining Conditional Directives

```json
{
  "$oky": {
    "employeeStatus|@ ('ACTIVE','ON_LEAVE','TERMINATED')": "TERMINATED",
    "workDays|(1..22)": 20,
    "leaveReason|{10,200}": "Parental leave",
    "terminationDate|~$Date~": "2025-12-31",
    
    "$requiredIf employeeStatus('ACTIVE')": ["workDays"],
    "$requiredIf employeeStatus('ON_LEAVE')": ["leaveReason"],
    "$forbiddenIfNot employeeStatus('TERMINATED')": ["terminationDate"],
    
    "$appliedIf employeeStatus": {
      "('ACTIVE')": {
        "currentProject|@": "Project Alpha"
      },
      "('TERMINATED')": {
        "exitInterview|@": true
      }
    }
  }
}
```

---

## Condition Syntax Summary

**Value conditions:**
- `fieldName('value')` - exact match
- `fieldName('val1','val2')` - any of values
- `fieldName(<18)` - comparison
- `fieldName(10..50)` - range

**Existence conditions:**
- Just field name in `$xxxIfExist`/`$xxxIfNotExist`

**Type conditions:**
- `fieldName(_TypeGuard_)` - type check
- `fieldName(_Type1_,_Type2_)` - multiple types (OR)
