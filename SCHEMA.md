# GT Recipe Display Spec v2

Language: [English](SCHEMA.md) | [中文](SCHEMA.zh_CN.md)

This document is the source-of-truth contract for GregTech recipe metadata
display specs.

The v2 design splits display behavior into:

- language-neutral rules: `spec/display.json`
- locale messages: `spec/i18n/<locale>.json`

The frontend loads both files for the active dataset, merges the locale messages
into `vue-i18n`, then renders `DisplayItem[]`. Components receive final text;
they do not know about spec keys.

## Resource Files

Source files:

- `spec/display.json`
- `spec/i18n/zh_CN.json`

Exported files served by Vite/nginx:

- `exports/gtnh/2.8.4/official/spec/display.json`
- `exports/gtnh/2.8.4/official/spec/i18n/zh_CN.json`

Backend `DatasetSummary` must return both URLs:

```json
{
  "displaySpecUrl": "/assets/gtnh/2.8.4/official/spec/display.json",
  "displaySpecMessagesUrl": "/assets/gtnh/2.8.4/official/spec/i18n/zh_CN.json"
}
```

First-stage locale support only ships `zh_CN`. The `zh_CN` file contains all
current visible text, including text that is currently English.

## Top-Level Shape

`display.json`:

```json
{
  "schema": "recipe-display-spec",
  "version": 2,
  "tables": {
    "table_name": {
      "raw_value": "spec.gregtech.table.table_name.rawValue"
    }
  },
  "handlers": {
    "macerator": {
      "nameKey": "spec.gregtech.handler.macerator.name",
      "recipe_kind": "PROCESSING",
      "recipe_count": 5228,
      "lines": []
    }
  }
}
```

Fields:

- `schema`: required. Must be exactly `recipe-display-spec`.
- `version`: required. Must be `2`.
- `tables`: required object. Shared lookup tables. Values are i18n keys.
- `handlers`: required object. Keyed by GregTech handler short name.

The handler short name is the recipe category without the `gregtech:` prefix.
For example, category `gregtech:macerator` maps to handler key `macerator`.

## Handler Spec

Handler object:

```json
{
  "nameKey": "spec.gregtech.handler.<handler>.name",
  "recipe_kind": "PROCESSING",
  "recipe_count": 1,
  "lines": []
}
```

Fields:

- `nameKey`: optional i18n key for handler display name.
- `recipe_kind`: optional documentation/debug field. Current values are
  `PROCESSING` or `FUEL`.
- `recipe_count`: optional documentation/debug field copied from current data.
- `lines`: required array of line specs. Rendered in order.

`name_cn` is not valid in v2.

## Runtime Context

Each recipe is converted to a runtime context before rendering:

```ts
interface RecipeCtx {
  handler: string
  recipe_kind: 'PROCESSING' | 'FUEL'
  voltage: number | null
  voltage_tier: string | null
  amperage: number | null
  duration_ticks: number
  special_value: number | null
  fluid_inputs: Array<{ amount: number; fluid_id: string }>
  item_inputs: Array<{ item_id: string; amount: number }>
  special_items: GregTechSpecialItem[]
  metadata: Record<string, unknown>
}
```

Expression fields can access `ctx`, `tables`, and the current raw `value`.

## Line Spec

Common line object:

```json
{
  "kind": "metadata",
  "key": "some_metadata_key",
  "labelKey": "spec.gregtech.handler.example.line.someMetadata.label",
  "expr": "comma(value)",
  "format": "comma_int",
  "prefixKey": "spec.gregtech.prefix.example",
  "suffixKey": "spec.gregtech.suffix.example",
  "show_if_present": true,
  "show_if_true": false,
  "show_if": "value > 0",
  "color_code": "GOLD"
}
```

All possible fields:

- `kind`: required. Determines where the raw value comes from.
- `key`: metadata key for `metadata`, `metadata_json`, and `flag`.
- `list_index`: zero-based index for `special_item` and `fluid_input`.
- `field`: `RecipeCtx` field name for `top_field`.
- `labelKey`: i18n key for the label. `null` or missing means no label.
- `expr`: expression that transforms the raw value.
- `valueKeyExpr`: expression that returns an i18n key or raw display value.
- `scale`: numeric multiplier applied after `expr` or before `format`.
- `format`: named formatter.
- `prefixKey`: i18n key prepended to the rendered value.
- `suffixKey`: i18n key appended to the rendered value.
- `suffix_table`: table name in top-level `tables`.
- `suffix_table_lookup`: lookup mode for `suffix_table`. Values: `exact`,
  `ceil`.
- `suffixTableFormatKey`: i18n template used after suffix-table lookup. The
  selected table text replaces `{0}`.
- `literalKey`: i18n key used by text-like lines.
- `valueTemplateKey`: i18n template applied around the rendered value.
- `templateArgs`: named expression map passed to `valueTemplateKey`.
- `color_code`: optional color code forwarded to the display item.
- `show_if_true`: if `true`, render only when raw value is exactly `true`.
- `show_if_present`: defaults to enabled. When enabled, missing/null raw values
  are skipped. Set to `false` only when a line must render for null.
- `show_if`: expression condition. Falsey result skips the line.
- `single`: single-voltage segment for `voltage_block`.
- `split`: split-voltage segments for `voltage_block`.

## Line Kinds

`total_eu`

- Raw value: `ctx.voltage * ctx.duration_ticks * ctx.amperage`.
- Skips when `ctx.voltage` or `ctx.amperage` is null.
- Usually formatted with `comma_long`.

`voltage_block`

- Special renderer. Does not use the normal raw-value pipeline.
- Uses `single` when recipe amperage is 1 or missing.
- Uses `split` when `ctx.amperage > 1`.
- If only `split` exists, `split` is always used.
- Skips when `ctx.voltage` is null.

`duration`

- Raw value: `ctx.duration_ticks`.
- Skips when ticks are `<= 0`.
- Usually formatted with `duration_seconds` or `duration_ticks`.

`fuel_heat`

- Raw value: `ctx.special_value`.
- Skips when `special_value` is null.

`large_boiler_table`

- Special renderer for large boiler fuel burn time.
- Uses `ctx.special_value / 40` as base.
- Emits fixed rows using i18n labels:
  `spec.gregtech.largeBoiler.header`,
  `spec.gregtech.largeBoiler.bronze`,
  `spec.gregtech.largeBoiler.steel`,
  `spec.gregtech.largeBoiler.titanium`,
  `spec.gregtech.largeBoiler.tungstensteel`.
- Uses `spec.gregtech.largeBoiler.disabled` when the computed cell is below
  `0.05`.
- Skips when `special_value` is null.

`metadata`

- Raw value: `ctx.metadata[key]`.
- Returns null when the metadata key is absent, which normally skips the line.
- Use for scalar metadata that should be formatted or transformed.

`metadata_json`

- Raw value: `ctx.metadata[key]`.
- Same lookup behavior as `metadata`.
- Intended for JSON metadata consumed by expressions.

`flag`

- Raw value: `ctx.metadata[key]`.
- Usually combined with `show_if_true: true` and `literalKey`.

`text`

- Raw value: empty string.
- Use `literalKey` or `valueTemplateKey`.
- Does not depend on recipe data.

`special_item`

- Raw value: `ctx.special_items[list_index ?? 0]?.itemVariantId`.
- Returns null if the item does not exist.

`fluid_input`

- Raw value: `ctx.fluid_inputs[list_index ?? 0]`.
- Returns null if the fluid input does not exist.
- Expressions can read `value.amount` and `value.fluid_id`.

`top_field`

- Raw value: `ctx[field]`.
- Returns null if the field does not exist.

## Voltage Block Segment

`single` and each entry of `split` use this shape:

```json
{
  "labelKey": "spec.gregtech.handler.example.line.voltage.label",
  "expr": "comma(ctx.voltage)",
  "valueKeyExpr": "'spec.gregtech.some.key'",
  "format": "comma_int",
  "prefixKey": "spec.gregtech.prefix.example",
  "suffixKey": "spec.gregtech.suffix.euPerTick",
  "valueTemplateKey": "spec.gregtech.template.voltageWithTier",
  "templateArgs": {
    "tier": "ctx.voltage_tier"
  }
}
```

Fields:

- `labelKey`: required i18n key.
- `expr`: expression evaluated with `value = ctx.voltage`.
- `valueKeyExpr`: expression returning an i18n key or raw display value.
- `format`: named formatter applied to `expr` result.
- `prefixKey`: translated prefix.
- `suffixKey`: translated suffix.
- `valueTemplateKey`: translated template applied to the segment value.
- `templateArgs`: named expression map passed to the template.

## Render Pipeline

Normal lines render in this order:

1. Read raw value according to `kind`.
2. Skip if raw value is `undefined`.
3. Skip null raw values unless `show_if_present` is `false`.
4. If `show_if_true` is `true`, skip unless raw value is exactly `true`.
5. If `show_if` exists, evaluate it with `{ value, ctx, tables }` and skip on
   falsey result.
6. Produce base value text:
   - `valueKeyExpr`: evaluate and translate if it returns `spec.gregtech.*`.
   - `expr`: evaluate, optionally apply `scale`, then apply `format` or
     translate if the result is an i18n key.
   - `literalKey`: translate the literal key.
   - `format`: optionally apply `scale`, then format the raw value.
   - fallback: `String(value)`.
7. Apply `valueTemplateKey`, passing `{ value, raw, result, ...templateArgs }`.
8. Apply `prefixKey` and `suffixKey`.
9. Apply `suffix_table` and `suffixTableFormatKey`.
10. Translate `labelKey` unless it is null/missing.
11. Emit `{ label, value, colorCode }`.

`valueTemplateKey` is the preferred way to handle locale-dependent word order.
Do not concatenate natural-language fragments in `expr`.

## Tables

Top-level `tables` are language-neutral lookup tables:

```json
{
  "coolant_type": {
    "water": "spec.gregtech.table.coolantType.water",
    "ic2coolant": "spec.gregtech.table.coolantType.ic2Coolant"
  }
}
```

Rules:

- Table names are internal identifiers.
- Table keys are raw recipe/metadata values.
- Table values must be i18n keys.
- `lookup(table, key)` in expressions returns the table value. If that value is
  a `spec.gregtech.*` key, the renderer translates it.

Suffix tables:

```json
{
  "kind": "metadata",
  "key": "heat",
  "labelKey": "spec.gregtech.handler.example.line.heat.label",
  "format": "comma_int",
  "suffix_table": "coil_tier",
  "suffix_table_lookup": "ceil",
  "suffixTableFormatKey": "spec.gregtech.template.parenthesized"
}
```

Lookup modes:

- `exact`: uses `table[String(raw)]`.
- `ceil`: treats numeric table keys as thresholds and chooses the smallest key
  `>= raw`; falls back to `*` if no threshold matches.

`suffixTableFormatKey` receives already translated table text through `{0}`.
If it is missing, the default template is `{0}`.

## Expression Language

Expressions use `expr-eval`.

Available bindings:

- `value`: raw line value, or `ctx.voltage` for voltage-block segments.
- `result`: only inside `templateArgs`; value after `expr` evaluation.
- `ctx`: runtime recipe context.
- `tables`: top-level lookup tables.

`raw` is not an expression binding. It is passed to the final i18n message as a
template parameter when `valueTemplateKey` is applied.

String concatenation:

- Use `||` for string concatenation.
- Do not use `+` for strings; `+` remains numeric addition.

Available helper functions:

- `str(value)`: convert to string.
- `int(value)`: truncate to integer.
- `comma(value)`: integer thousands formatting.
- `lookup(table, key)`: table lookup by stringified key.
- `fixed1(value)`: one decimal place.
- `tier(value)`: voltage tier name from EU/t.
- `floor(value)`: `Math.floor`.
- `ceil(value)`: `Math.ceil`.
- `round(value)`: `Math.round`.
- `tanh(value)`: `Math.tanh`.
- `band(a, b)`: bitwise AND.
- `bor(a, b)`: bitwise OR.
- `shr(a, b)`: unsigned right shift.
- `len(value)`: array length, or 0 for non-arrays.

Expression examples:

```json
{
  "expr": "comma(ctx.voltage * ctx.amperage)",
  "valueTemplateKey": "spec.gregtech.template.euPerTickWithTier",
  "templateArgs": {
    "tier": "tier(ctx.voltage * ctx.amperage)"
  }
}
```

```json
{
  "valueKeyExpr": "lookup(tables.cleanroom_type, value)"
}
```

## Formats

Named formatters:

- `direct`: `String(value)`.
- `as_string`: `String(value)`.
- `comma_int`: integer thousands formatting.
- `comma_long`: integer thousands formatting.
- `comma_double_1`: thousands formatting with one decimal place.
- `percent_int_x100`: appends `%` to the numeric value as-is.
- `percent_double_x100`: multiplies numeric value by 100, rounds, appends `%`.
- `decimal_0`: rounded integer string.
- `decimal_1`: fixed one decimal place.
- `decimal_2`: fixed two decimal places.
- `scientific`: JavaScript exponential notation with two decimals.
- `duration_seconds`: input is ticks. Values below 20 ticks render as ticks;
  otherwise render as seconds.
- `duration_ticks`: input is ticks and always renders as ticks.
- `duration_auto_unit`: input is seconds. Chooses seconds/minutes/hours/days.
- `java_double`: Java-like double string for GT metadata.
- `bool_yes_no`: translated yes/no.

Formatter text comes from i18n where needed:

- `spec.gregtech.format.duration.ticks`
- `spec.gregtech.format.duration.seconds`
- `spec.gregtech.format.duration.autoSeconds`
- `spec.gregtech.format.duration.autoMinutes`
- `spec.gregtech.format.duration.autoHours`
- `spec.gregtech.format.duration.autoDays`
- `spec.gregtech.format.bool.yes`
- `spec.gregtech.format.bool.no`

Do not add language-specific format names such as `duration_seconds_en` or
`bool_yes_no_cn`.

## I18n Messages

Message files are normal `vue-i18n` locale objects:

```json
{
  "spec": {
    "gregtech": {
      "handler": {
        "macerator": {
          "name": "Macerator",
          "line": {
            "duration": {
              "label": "Duration"
            }
          }
        }
      }
    }
  }
}
```

The corresponding flat key is:

```text
spec.gregtech.handler.macerator.line.duration.label
```

Key rules:

- All keys referenced by `display.json` must start with `spec.gregtech.`.
- Handler names should use `spec.gregtech.handler.<handler>.name`.
- Handler line labels should use
  `spec.gregtech.handler.<handler>.line.<semanticName>.label`.
- Handler line literals should use
  `spec.gregtech.handler.<handler>.line.<semanticName>.literal`.
- Shared templates should use `spec.gregtech.template.<name>`.
- Shared suffixes/prefixes should use `spec.gregtech.suffix.<name>` and
  `spec.gregtech.prefix.<name>`.
- Shared table values should use `spec.gregtech.table.<tableName>.<valueName>`.
- Missing keys intentionally render as the key itself and log
  `console.warn`, so bad keys are visible during review.

`valueTemplateKey` uses named parameters. At minimum these are available:

- `{value}`: rendered value text before the template.
- `{raw}`: raw value from the line kind.
- `{result}`: expression result, or raw value when no expression was used.
- any names from `templateArgs`.

Example:

```json
{
  "valueTemplateKey": "spec.gregtech.template.percentChance",
  "templateArgs": {
    "chance": "value * 100"
  }
}
```

Message:

```json
{
  "spec": {
    "gregtech": {
      "template": {
        "percentChance": "{value} chance"
      }
    }
  }
}
```

Use templates instead of expression concatenation when different locales may
need different word order.

## Language-Neutral Rules

`display.json` must not contain user-visible prose in any language.

Allowed in `display.json`:

- i18n keys, for example `spec.gregtech.handler.macerator.name`.
- handler ids and metadata keys.
- enum-like values from source data, for example `PROCESSING`, `FUEL`, `BIO`.
- voltage tier names, for example `LV`, `HV`, `LuV`, `MAX`.
- formulas and expression syntax.
- unit symbols that are part of formulas or game constants, when not used as
  prose.

Forbidden in `display.json`:

- Chinese labels or literals.
- English labels or literals such as `Duration`, `Steam output shown`, or
  `Hydrogen`.
- concatenated natural-language fragments in `expr`.
- old v1 text fields such as `name_cn`, `label`, `literal`, `prefix`, `suffix`,
  and `suffix_table_format`.
- language-specific formatter names.

All visible text belongs in `spec/i18n/<locale>.json`, even if the current text
is English.

## Frontend Loading Flow

For `GregTechMetadataStrip`:

1. Load `dataset.displaySpecUrl`.
2. Load `dataset.displaySpecMessagesUrl` for the active dataset locale.
3. Merge messages through `i18n.global.mergeLocaleMessage(locale, messages)`.
4. Render recipe lines with `renderRecipe({ spec, ctx, t })`.
5. Return final `DisplayItem[]`.

Spec cache key:

- `displaySpecUrl`

Message cache key:

- `<locale>:<displaySpecMessagesUrl>`

The dataset locale controls the active `vue-i18n` locale. First-stage fallback is
also `zh_CN`.

## Validation

Structure is validated by the JSON Schema `spec/recipe-display.schema.json`, referenced from
`display.json` via its `$schema` field. Editors (VS Code etc.) provide completion and inline
validation automatically.

The schema checks:

- `display.json` has `schema: "recipe-display-spec"` and `version: 2`.
- every line has a valid `kind` and only known fields.
- `format` values are from the known formatter set.

i18n key existence (that every referenced `spec.<mod>.*` key exists in `spec/i18n/<locale>.json`)
is not enforced by the schema; missing keys render as the key itself and log a `console.warn` at
runtime, so bad keys surface during review.

## Maintenance Checklist

When adding a new displayed field:

1. Add a line to the correct handler in `spec/display.json` (the `$schema` ref gives completion).
2. Put every visible word in `spec/i18n/<locale>.json` under the owning mod's namespace
   (`spec.gregtech.*`, `spec.bloodmagic.*`, ...).
3. Use `valueTemplateKey` for phrases or locale-dependent order.
4. Use `expr` only for computation and source-data lookup.
5. Add shared lookup text to `tables` plus `spec.<mod>.table.*` messages.
6. `display.json` and `i18n/` are served directly from `exports/gtnh/2.8.4/official/spec/...`
   (same files), so edits take effect immediately.


Do not use `tools/build-display-spec.py` as the authority for v2 until it is
explicitly updated for this schema.
