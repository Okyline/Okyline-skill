# Okyline Expression Language

The expression language is used in `$compute` blocks for business rules, cross-field validation, and calculated constraints.

## Declaration & Usage

```json
{
  "$oky": {
    "subtotal": 100.0,
    "taxRate": 0.2,
    "total|(%ValidTotal)": 120.0
  },
  "$compute": {
    "ValidTotal": "total == subtotal * (1 + taxRate)"
  }
}
```

**Reference compute in validation:** `|(%ComputeName)`
**Reference compute in expression:** `%ComputeName`

## Context Rule

A `$compute` expression is always evaluated in the context of the **object that directly contains the annotated field**. All properties of that object are accessible — including sibling arrays and nested objects.

```json
// The annotated field is inside a line item — context = the current element
"lignes|[*]": [{
  "quantite": 5,
  "prix": 100.0,
  "montantHT|(%Check)": 500.0
}],
"$compute": { "Check": "montantHT == quantite * prix" }
```

> The same rule applies at any depth. When the annotated field is at the root, the containing object is the document root — all root-level arrays and fields are accessible.

---

## Operators

| Operator | Description | Example | Null Behavior |
|----------|-------------|---------|---------------|
| `+` | Addition | `2 + 3 → 5` | Null propagates (except string concat: null → "") |
| `-` | Subtraction | `5 - 2 → 3` | Null propagates |
| `*` | Multiplication | `3 * 2 → 6` | Null propagates |
| `/` | Division | `6 / 2 → 3.0` | Null propagates; div by zero → null |
| `>` `<` `>=` `<=` | Comparison | `age > 18` | Returns null if operand is null |
| `==` `!=` | Equality | `"A" == "A"` | null == null → true |
| `===` `!==` | Strict equality | `1.0 === 1.0` | IEEE-754 bit-exact |
| `&&` | Logical AND | `a && b` | null → false |
| `||` | Logical OR | `a || b` | null → false |
| `!` | Negation | `!true → false` | !null → true |
| `??` | Null coalescing | `price ?? 0` | Returns right if left is null |
| `? :` | Ternary | `x > 10 ? "hi" : "lo"` | Condition null → false |

---

## Referencing Other Computes

Use `%ComputeName` to reference other computed expressions.

```json
{
  "$compute": {
    "BasePrice": "1000",
    "Tax": "%BasePrice * 0.2",
    "Shipping": "50",
    "Total": "%BasePrice + %Tax + %Shipping"
  }
}
```

**In aggregations:**
```json
{
  "$compute": {
    "LineTotal": "netAmount * (1 + vat)",
    "InvoiceTotal": "sum(items, %LineTotal)"
  }
}
```

---

## Numeric Functions

| Function | Description | Example |
|----------|-------------|---------|
| `abs(x)` | Absolute value | `abs(-5) → 5` |
| `sqrt(x)` | Square root | `sqrt(9) → 3.0` |
| `floor(x, scale?)` | Round down | `floor(3.1415, 2) → 3.14` |
| `ceil(x, scale?)` | Round up | `ceil(3.1415, 2) → 3.15` |
| `round(x, scale?, mode?)` | Round (HALF_UP default) | `round(3.5, 0) → 4.0` |
| `mod(a, b)` | Remainder | `mod(10, 3) → 1` |
| `pow(base, exp)` | Power | `pow(2, 3) → 8.0` |
| `log(x)` | Natural logarithm | `log(2.71828) → 1.0` |
| `log10(x)` | Base-10 logarithm | `log10(1000) → 3.0` |
| `toInt(v)` | Convert to integer | `toInt(3.7) → 4` |
| `toNum(v)` | Convert to number | `toNum("42") → 42.0` |
| `toStr(v)` | Convert to string | `toStr(42) → "42"` |

---

## String Functions

| Function | Description | Example |
|----------|-------------|---------|
| `isNull(v)` | True if null | `isNull(null) → true` |
| `isNullOrEmpty(s)` | True if null or empty | `isNullOrEmpty("") → true` |
| `isEmpty(s)` | True if length is 0 | `isEmpty("") → true` |
| `length(s)` | String length (null → 0) | `length("hey") → 3` |
| `substring(s, start, len)` | Extract substring | `substring("Hello", 1, 3) → "ell"` |
| `substringBefore(s, delim)` | Part before first occurrence | `substringBefore("a:b:c", ":") → "a"` |
| `substringAfter(s, delim)` | Part after first occurrence | `substringAfter("a:b:c", ":") → "b:c"` |
| `substringBeforeLast(s, delim)` | Part before last occurrence | `substringBeforeLast("a:b:c", ":") → "a:b"` |
| `substringAfterLast(s, delim)` | Part after last occurrence | `substringAfterLast("a.b.txt", ".") → "txt"` |
| `replace(s, target, repl)` | Replace all occurrences | `replace("foo bar foo", "foo", "baz") → "baz bar baz"` |
| `replaceFirst(s, target, repl)` | Replace first occurrence | `replaceFirst("foo bar foo", "foo", "baz") → "baz bar foo"` |
| `replaceLast(s, target, repl)` | Replace last occurrence | `replaceLast("foo bar foo", "foo", "baz") → "foo bar baz"` |
| `trim(s)` | Remove whitespace | `trim("  hi  ") → "hi"` |
| `ltrim(s)` | Remove leading spaces | `ltrim("  hi  ") → "hi  "` |
| `rtrim(s)` | Remove trailing spaces | `rtrim("  hi  ") → "  hi"` |
| `startsWith(s, prefix)` | Check prefix | `startsWith("hello", "he") → true` |
| `endsWith(s, suffix)` | Check suffix | `endsWith("hello", "lo") → true` |
| `contains(s, search)` | Check substring | `contains("banana", "an") → true` |
| `removePrefix(s, prefix)` | Remove prefix if present | `removePrefix("Hello", "He") → "llo"` |
| `removeSuffix(s, suffix)` | Remove suffix if present | `removeSuffix("Hello", "lo") → "Hel"` |
| `removeRange(s, start, len)` | Remove len chars from start | `removeRange("Hello", 1, 3) → "Ho"` |
| `toUpperCase(s)` | To upper case | `toUpperCase("Hi") → "HI"` |
| `toLowerCase(s)` | To lower case | `toLowerCase("Hi") → "hi"` |
| `capitalize(s)` | Uppercase first char | `capitalize("hello") → "Hello"` |
| `decapitalize(s)` | Lowercase first char | `decapitalize("Hello") → "hello"` |
| `padStart(s, len, ch)` | Left pad | `padStart("7", 3, "0") → "007"` |
| `padEnd(s, len, ch)` | Right pad | `padEnd("7", 3, "0") → "700"` |
| `repeat(times, ch)` | Repeat character | `repeat(5, "*") → "*****"` |
| `indexOf(s, sub)` | First index of substring | `indexOf("abracadabra", "bra") → 1` |
| `indexOfFirst(s, sub)` | Alias for indexOf | `indexOfFirst("abracadabra", "bra") → 1` |
| `indexOfLast(s, sub)` | Last index of substring | `indexOfLast("abracadabra", "bra") → 8` |

### String Index Handling
- Negative start indices → clamped to 0
- Negative lengths → treated as 0
- Start beyond length → empty string
- End beyond length → clamped to length
- Functions never throw exceptions

---

## Date Functions

| Function | Description | Example |
|----------|-------------|---------|
| `date(str, pattern?)` | Parse date (default: yyyy-MM-dd) | `date("2024-03-15")` |
| `formatDate(date, pattern?)` | Format date | `formatDate(d, "dd/MM/yy") → "15/03/24"` |
| `today()` | Current system date | `today() → 2025-11-05` |
| `daysBetween(start, end)` | Days difference | `daysBetween("2024-03-15", "2024-03-18") → 3` |
| `plusDays(date, days)` | Add days | `plusDays("2024-02-28", 1) → "2024-02-29"` |
| `minusDays(date, days)` | Subtract days | `minusDays("2024-03-01", 1) → "2024-02-29"` |
| `plusMonths(date, months)` | Add months | `plusMonths("2024-01-31", 1) → "2024-02-29"` |
| `minusMonths(date, months)` | Subtract months | `minusMonths("2024-03-31", 1) → "2024-02-29"` |
| `plusYears(date, years)` | Add years | `plusYears("2023-03-15", 1) → "2024-03-15"` |
| `minusYears(date, years)` | Subtract years | `minusYears("2024-03-15", 1) → "2023-03-15"` |
| `isWeekend(date)` | True if Sat/Sun | `isWeekend("2024-03-16") → true` |
| `isLeapYear(date)` | True if leap year | `isLeapYear("2024-03-15") → true` |
| `year(date)` | Extract year | `year("2024-03-15") → 2024` |
| `month(date)` | Extract month | `month("2024-03-15") → 3` |
| `day(date)` | Extract day | `day("2024-03-15") → 15` |
| `dayOfWeek(date)` | Day of week (MON=1, SUN=7) | `dayOfWeek("2024-03-15") → 5` |
| `dayOfYear(date)` | Day of year (1-366) | `dayOfYear("2024-03-15") → 75` |
| `weekOfYear(date)` | ISO week number (1-53) | `weekOfYear("2024-01-01") → 1` |
| `quarter(date)` | Quarter (1-4) | `quarter("2024-09-15") → 3` |
| `semester(date)` | Semester (1-2) | `semester("2024-09-15") → 2` |
| `before(date1, date2)` | True if date1 < date2 | `before("2024-01-01", "2024-12-31") → true` |
| `after(date1, date2)` | True if date1 > date2 | `after("2024-12-31", "2024-01-01") → true` |
| `equals(date1, date2)` | True if same date | `equals("2024-03-15", "2024-03-15") → true` |

---

## Aggregation Functions

| Function | Description | Example |
|----------|-------------|---------|
| `sum(collection, expr)` | Sum of expression | `sum(items, price)` |
| `average(collection, expr)` | Average of expression | `average(items, quantity)` |
| `min(collection, expr)` | Minimum value | `min(scores, score)` |
| `max(collection, expr)` | Maximum value | `max(scores, score)` |
| `count(collection)` | Count non-null elements | `count(items)` |
| `countAll(collection)` | Count all elements | `countAll(items)` |
| `countIf(collection, expr)` | Count where expr is true | `countIf(users, active)` |
| `exists(collection, expr)` | True if any element matches | `exists(items, price > 100)` |
| `notExists(collection, expr)` | True if no element matches | `notExists(items, price < 0)` |
| `sumIf(collection, pred, expr)` | Sum expr where pred is true | `sumIf(items, active, price)` |
| `map(collection, expr)` | List of expr per element | `map(items, price * qty)` |
| `filter(collection, expr)` | Elements where expr is true | `filter(items, active)` |

For scalar collections (arrays of numbers/strings), aggregations work without a second argument: `sum(scores)`, `min(prices)`.

**Aggregation with compute reference:**
```json
{
  "$compute": {
    "LineTotal": "price * quantity",
    "OrderTotal": "sum(items, %LineTotal)"
  }
}
```

---

## Membership Function — `in`

Tests whether a value belongs to a set.

| Form | Example |
|------|---------|
| Inline literals | `in(status, 'DRAFT', 'SENT')` |
| Nomenclature | `in(status, '$INVOICE_STATUS')` |
| Array field | `in(code, allowedCodes)` |

- `null` value → `false`
- List as first arg → containsAll semantics: `in(myList, 'A', 'B')` checks all present

---

## List Iteration Context

Inside aggregation lambdas (`countIf`, `exists`, `filter`, `sum`, etc.), these variables are available:

| Variable | Resolves to |
|----------|-------------|
| `origin` | The item whose validation triggered the aggregation |
| `prev` | Element before current iteration element |
| `next` | Element after current iteration element |
| `first` | First element of the collection |
| `last` | Last element of the collection |

**Positional predicates:**

| Predicate | Returns true when |
|-----------|-------------------|
| `isOrigin` | Current element is the origin element |
| `isFirst` | Current element is first in collection |
| `isLast` | Current element is last in collection |

All support dotted navigation: `origin.amount`, `prev.date`, `first.id`.

**Example — uniqueness check:**
```json
{
  "$compute": {
    "IsUnique": "countIf(parent.items, id == origin.id) == 1"
  },
  "$oky": {
    "items|[*]": [{
      "id|@ (%IsUnique)": "A001"
    }]
  }
}
```

---

## Null Handling

### Arithmetic (null propagates)
```js
10 + null     → null
null * 5      → null
// Exception: string concat treats null as ""
"Hello" + null → "Hello"
```

### Comparison
```js
10 > null     → null
null <= 5     → null
```

### Equality
```js
null == null  → true
null == x     → false
null != x     → true
```

### Logical
```js
null && true  → false
null || true  → true
!null         → true
```

### Null coalescing
```js
price ?? 0              → 0 (if price is null)
a ?? b ?? c ?? 0        → first non-null or 0
(price ?? 0) * (qty ?? 1)
```

---

## Complete Example

```json
{
  "$oky": {
    "order": {
      "items|@ [1,100]": [
        {"price": 10.0, "quantity": 2, "vat": 0.2},
        {"price": 15.0, "quantity": 3, "vat": 0.1}
      ],
      "discount": 5.0,
      "shippingCost": null,
      "subtotal|(%CheckSubtotal)": 65.0,
      "total|(%CheckTotal)": 71.5
    }
  },
  "$compute": {
    "LineGross": "round(price * quantity * (1 + vat), 2)",
    "Shipping": "shippingCost ?? 10.0",
    "CheckSubtotal": "subtotal == sum(items, price * quantity)",
    "CheckTotal": "total == sum(items, %LineGross) - discount + %Shipping"
  }
}
```

**Explanation:**
1. `LineGross` calculates gross amount per line (price × quantity × (1 + vat))
2. `Shipping` defaults to 10.0 if null
3. `CheckSubtotal` validates subtotal equals sum of (price × quantity)
4. `CheckTotal` validates total equals sum of gross amounts minus discount plus shipping
