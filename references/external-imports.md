# External Imports (Annex E)

Cross-schema composition: depend on other contracts and import their named
definitions. Only relevant when modularization or reuse across projects is
explicitly required — otherwise keep everything inline in a single schema.

Identity keys (`$id`, `$version`, `$state`) and multi-payload entries
(`$entries`) are NOT specific to external imports — see SKILL.md Document
Metadata.

## Dependencies — `$deps`

```json
"$deps": {
  "common":  "~1.3.0",
  "billing": "^2.0.0",
  "geo":     "1.5.0"
}
```

| Constraint | Matches |
|------------|---------|
| `1.3.4` | Exact |
| `~1.3.0` | Same MAJOR.MINOR, any PATCH |
| `^1.3.0` | Same MAJOR, any MINOR/PATCH |

## External Imports — `$import`

Map a local alias to a definition from a dependency. `$import` lives **inside
the sub-block whose kind it imports**: `$defs` for type/object definitions,
`$nomenclature` / `$format` / `$compute` for the others. A `$import` at the
document root fails loading. Inside `$oky`/`$defs`, references use the **local
alias** only (`&Address`), never the external form (`&common.Address`).

```json
"$deps": { "common": "~1.3.0" },
"$defs": { "$import": { "Address": "&common.Address", "Email": "&common.Email" } },
"$oky": {
  "ship|@": "&Address"
}
```

Other kinds import the same way, each in its own sub-block:

```json
"$nomenclature": { "$import": { "COUNTRY": "&geo.COUNTRY" } }
"$format":       { "$import": { "Phone":   "&common.Phone" } }
"$compute":      { "$import": { "Tax":     "&billing.Tax" } }
```

Once imported, aliases share the namespace of `$defs` and accept the full
Annex D toolkit (`$ref`, `$override`, `$amend`, `$remove`).

## Visibility — `$public` / `$private`

Per-sub-block (`$defs`, `$nomenclature`, `$format`, `$compute`). Controls
what is exposable to external `$import` imports — mutually exclusive.

```json
"$defs": {
  "Animal":  { ... },
  "_Helper": { ... },
  "$private": ["_Helper"]
}
```

| Mode | Effect |
|------|--------|
| Neither | All entries exposed (default) |
| `$public: [...]` | Strict allow-list. `[]` locks all. |
| `$private: [...]` | Deny-list. Empty `[]` rejected (use absence). |

Names are bare (no `&` sigil). Hidden entries appear absent to importers.
Visibility is a barrier to **import**, not to runtime validation — an
exposed type can still internally use hidden helpers.

## Re-export through Alias Chains

A `$import` alias is itself a valid import target — an intermediate library
can re-export from its own deps. Applies to all four kinds. Cycles fail
at load time. Re-exports respect the intermediate's visibility rules.

```
common.Address  ←  region.$import.Addr  ←  app imports &region.Addr
```

## When to activate

- ✅ Explicit user request for modularization / shared library
- ✅ User loads a schema that already declares `$deps` / `$import`
- ❌ Default — keep everything inline in `$oky` + `$defs`
