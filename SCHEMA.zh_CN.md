# GT Recipe Display Spec v2

语言：[English](SCHEMA.md) | [中文](SCHEMA.zh_CN.md)

本文档是 GregTech 配方 metadata 展示 spec 的 v2 合约说明。

v2 把展示资源拆成两类：

- 语言无关规则：`spec/display.json`
- 本地化消息：`spec/i18n/<locale>.json`

前端会按当前 dataset 同时加载这两个文件，把 locale messages 合并进
`vue-i18n`，再渲染成 `DisplayItem[]`。组件层只接收最终文本，不直接感知
spec key。

## 资源文件

源文件：

- `spec/display.json`
- `spec/i18n/zh_CN.json`

由 Vite/nginx serve 的导出副本：

- `exports/gtnh/2.8.4/official/spec/display.json`
- `exports/gtnh/2.8.4/official/spec/i18n/zh_CN.json`

后端 `DatasetSummary` 必须返回两个 URL：

```json
{
  "displaySpecUrl": "/assets/gtnh/2.8.4/official/spec/display.json",
  "displaySpecMessagesUrl": "/assets/gtnh/2.8.4/official/spec/i18n/zh_CN.json"
}
```

第一阶段只维护 `zh_CN`。`zh_CN` 文件里包含当前所有可见文本，包括目前仍然是英文的文案。

## 顶层结构

`display.json`：

```json
{
  "schema": "gregtech-display-spec",
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

字段：

- `schema`：必填，必须是 `gregtech-display-spec`。
- `version`：必填，必须是 `2`。
- `tables`：必填对象，共用查表结构，值必须是 i18n key。
- `handlers`：必填对象，以 GregTech handler 短名为 key。

handler 短名来自 recipe category 去掉 `gregtech:` 前缀后的部分。例如
`gregtech:macerator` 对应 handler key `macerator`。

## Handler Spec

handler 对象：

```json
{
  "nameKey": "spec.gregtech.handler.<handler>.name",
  "recipe_kind": "PROCESSING",
  "recipe_count": 1,
  "lines": []
}
```

字段：

- `nameKey`：可选，handler 展示名的 i18n key。
- `recipe_kind`：可选，文档/调试字段。当前值为 `PROCESSING` 或 `FUEL`。
- `recipe_count`：可选，文档/调试字段，来自当前数据统计。
- `lines`：必填数组，按顺序渲染。

v2 不允许使用 `name_cn`。

## 运行时上下文

每条配方渲染前会被转换成运行时上下文：

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

表达式字段可以访问 `ctx`、`tables` 和当前原始值 `value`。

## Line Spec

通用 line 对象：

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

所有可用字段：

- `kind`：必填，决定 raw value 从哪里取。
- `key`：`metadata`、`metadata_json`、`flag` 使用的 metadata key。
- `list_index`：`special_item` 和 `fluid_input` 使用的从 0 开始的索引。
- `field`：`top_field` 使用的 `RecipeCtx` 字段名。
- `labelKey`：label 的 i18n key。`null` 或缺省表示没有 label。
- `expr`：转换 raw value 的表达式。
- `valueKeyExpr`：返回 i18n key 或原始展示值的表达式。
- `scale`：数值倍率，应用在 `expr` 之后或 `format` 之前。
- `format`：命名 formatter。
- `prefixKey`：追加到 value 前面的 i18n key。
- `suffixKey`：追加到 value 后面的 i18n key。
- `suffix_table`：顶层 `tables` 里的表名。
- `suffix_table_lookup`：`suffix_table` 查询模式，可选 `exact` 或 `ceil`。
- `suffixTableFormatKey`：suffix table 命中后的 i18n 模板，表文本会替换 `{0}`。
- `literalKey`：text-like line 使用的 i18n key。
- `valueTemplateKey`：包裹最终 value 的 i18n 模板。
- `templateArgs`：传给 `valueTemplateKey` 的命名表达式。
- `color_code`：可选颜色码，会透传给 display item。
- `show_if_true`：为 `true` 时，只有 raw value 严格等于 `true` 才渲染。
- `show_if_present`：默认启用。启用时 raw value 为 null 会跳过。只有需要 null 也渲染时才设为 `false`。
- `show_if`：显示条件表达式，结果为 falsey 时跳过。
- `single`：`voltage_block` 的单行电压段。
- `split`：`voltage_block` 的拆分电压段。

## Line Kind

`total_eu`

- Raw value：`ctx.voltage * ctx.duration_ticks * ctx.amperage`。
- 当 `ctx.voltage` 或 `ctx.amperage` 为 null 时跳过。
- 通常使用 `comma_long` 格式化。

`voltage_block`

- 专用 renderer，不走普通 raw-value 流程。
- amperage 为 1 或缺省时使用 `single`。
- `ctx.amperage > 1` 时使用 `split`。
- 如果只提供 `split`，永远使用 `split`。
- `ctx.voltage` 为 null 时跳过。

`duration`

- Raw value：`ctx.duration_ticks`。
- ticks `<= 0` 时跳过。
- 通常使用 `duration_seconds` 或 `duration_ticks`。

`fuel_heat`

- Raw value：`ctx.special_value`。
- `special_value` 为 null 时跳过。

`large_boiler_table`

- 大型锅炉燃料燃烧时间专用 renderer。
- 使用 `ctx.special_value / 40` 作为基础值。
- 固定输出这些 i18n label：
  `spec.gregtech.largeBoiler.header`、
  `spec.gregtech.largeBoiler.bronze`、
  `spec.gregtech.largeBoiler.steel`、
  `spec.gregtech.largeBoiler.titanium`、
  `spec.gregtech.largeBoiler.tungstensteel`。
- 单元格计算值低于 `0.05` 时使用 `spec.gregtech.largeBoiler.disabled`。
- `special_value` 为 null 时跳过。

`metadata`

- Raw value：`ctx.metadata[key]`。
- metadata key 不存在时返回 null，默认会跳过该行。
- 用于需要格式化或表达式转换的标量 metadata。

`metadata_json`

- Raw value：`ctx.metadata[key]`。
- 查询行为和 `metadata` 相同。
- 用于表达式需要读取的 JSON metadata。

`flag`

- Raw value：`ctx.metadata[key]`。
- 通常和 `show_if_true: true`、`literalKey` 一起使用。

`text`

- Raw value：空字符串。
- 使用 `literalKey` 或 `valueTemplateKey`。
- 不依赖配方数据存在性。

`special_item`

- Raw value：`ctx.special_items[list_index ?? 0]?.itemVariantId`。
- 对应物品不存在时返回 null。

`fluid_input`

- Raw value：`ctx.fluid_inputs[list_index ?? 0]`。
- 对应 fluid input 不存在时返回 null。
- 表达式可以读取 `value.amount` 和 `value.fluid_id`。

`top_field`

- Raw value：`ctx[field]`。
- 字段不存在时返回 null。

## Voltage Block Segment

`single` 和 `split` 中每个元素使用这个结构：

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

字段：

- `labelKey`：必填 i18n key。
- `expr`：表达式，求值时 `value = ctx.voltage`。
- `valueKeyExpr`：返回 i18n key 或原始展示值的表达式。
- `format`：应用到 `expr` 结果的命名 formatter。
- `prefixKey`：翻译后的前缀。
- `suffixKey`：翻译后的后缀。
- `valueTemplateKey`：应用到 segment value 的翻译模板。
- `templateArgs`：传给模板的命名表达式。

## 渲染顺序

普通 line 按这个顺序渲染：

1. 根据 `kind` 读取 raw value。
2. raw value 为 `undefined` 时跳过。
3. 除非 `show_if_present` 是 `false`，否则 raw value 为 null 时跳过。
4. 如果 `show_if_true` 是 `true`，只有 raw value 严格等于 `true` 才继续。
5. 如果存在 `show_if`，用 `{ value, ctx, tables }` 求值，falsey 时跳过。
6. 生成基础 value text：
   - `valueKeyExpr`：求值，如果返回 `spec.gregtech.*` 则翻译。
   - `expr`：求值，可选应用 `scale`，再应用 `format`，否则如果结果是 i18n key 则翻译。
   - `literalKey`：翻译 literal key。
   - `format`：可选应用 `scale`，再格式化 raw value。
   - fallback：`String(value)`。
7. 应用 `valueTemplateKey`，参数为 `{ value, raw, result, ...templateArgs }`。
8. 应用 `prefixKey` 和 `suffixKey`。
9. 应用 `suffix_table` 和 `suffixTableFormatKey`。
10. 翻译 `labelKey`，除非它为 null 或缺省。
11. 输出 `{ label, value, colorCode }`。

涉及不同语言语序的内容，应优先使用 `valueTemplateKey`。不要在 `expr` 里拼自然语言片段。

## Tables

顶层 `tables` 是语言无关查表结构：

```json
{
  "coolant_type": {
    "water": "spec.gregtech.table.coolantType.water",
    "ic2coolant": "spec.gregtech.table.coolantType.ic2Coolant"
  }
}
```

规则：

- table 名是内部标识。
- table key 是配方或 metadata 的 raw value。
- table value 必须是 i18n key。
- 表达式里的 `lookup(table, key)` 返回 table value。如果返回值是
  `spec.gregtech.*` key，renderer 会翻译。

Suffix table 示例：

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

查询模式：

- `exact`：使用 `table[String(raw)]`。
- `ceil`：把数字 table key 当阈值，选择最小的 `>= raw` 的 key；没有命中时回退到 `*`。

`suffixTableFormatKey` 通过 `{0}` 接收已翻译的 table 文本。缺省时默认模板是 `{0}`。

## 表达式语言

表达式使用 `expr-eval`。

可用绑定：

- `value`：line raw value；对 voltage-block segment 来说是 `ctx.voltage`。
- `result`：只在 `templateArgs` 里可用，表示 `expr` 求值后的结果。
- `ctx`：运行时配方上下文。
- `tables`：顶层查表对象。

`raw` 不是表达式绑定。应用 `valueTemplateKey` 时，它会作为最终 i18n message 的模板参数传入。

字符串拼接：

- 用 `||` 做字符串拼接。
- 不要用 `+` 拼字符串，`+` 保持数值加法语义。

可用 helper 函数：

- `str(value)`：转字符串。
- `int(value)`：截断为整数。
- `comma(value)`：整数千分位格式化。
- `lookup(table, key)`：把 key 转字符串后查表。
- `fixed1(value)`：保留一位小数。
- `tier(value)`：按 EU/t 计算电压等级名。
- `floor(value)`：`Math.floor`。
- `ceil(value)`：`Math.ceil`。
- `round(value)`：`Math.round`。
- `tanh(value)`：`Math.tanh`。
- `band(a, b)`：按位 AND。
- `bor(a, b)`：按位 OR。
- `shr(a, b)`：无符号右移。
- `len(value)`：数组长度，非数组返回 0。

表达式示例：

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

命名 formatter：

- `direct`：`String(value)`。
- `as_string`：`String(value)`。
- `comma_int`：整数千分位。
- `comma_long`：整数千分位。
- `comma_double_1`：千分位并保留一位小数。
- `percent_int_x100`：把数值原样加 `%`。
- `percent_double_x100`：数值乘 100、四舍五入后加 `%`。
- `decimal_0`：四舍五入整数文本。
- `decimal_1`：固定一位小数。
- `decimal_2`：固定两位小数。
- `scientific`：JavaScript 两位小数科学计数法。
- `duration_seconds`：输入是 ticks。低于 20 ticks 显示 ticks，否则显示秒。
- `duration_ticks`：输入是 ticks，始终显示 ticks。
- `duration_auto_unit`：输入是秒，自动选择秒、分钟、小时或天。
- `java_double`：接近 Java double 的显示文本，用于 GT metadata。
- `bool_yes_no`：翻译后的是/否。

需要文案的 formatter 从 i18n 取文本：

- `spec.gregtech.format.duration.ticks`
- `spec.gregtech.format.duration.seconds`
- `spec.gregtech.format.duration.autoSeconds`
- `spec.gregtech.format.duration.autoMinutes`
- `spec.gregtech.format.duration.autoHours`
- `spec.gregtech.format.duration.autoDays`
- `spec.gregtech.format.bool.yes`
- `spec.gregtech.format.bool.no`

不要添加 `duration_seconds_en`、`bool_yes_no_cn` 这类语言特化 format 名称。

## I18n Messages

message 文件是标准 `vue-i18n` locale object：

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

对应 flat key：

```text
spec.gregtech.handler.macerator.line.duration.label
```

key 规则：

- `display.json` 引用的所有 key 必须以 `spec.gregtech.` 开头。
- handler 名称建议使用 `spec.gregtech.handler.<handler>.name`。
- handler line label 建议使用
  `spec.gregtech.handler.<handler>.line.<semanticName>.label`。
- handler line literal 建议使用
  `spec.gregtech.handler.<handler>.line.<semanticName>.literal`。
- 共用模板建议使用 `spec.gregtech.template.<name>`。
- 共用前后缀建议使用 `spec.gregtech.prefix.<name>` 和
  `spec.gregtech.suffix.<name>`。
- 共用 table 文本建议使用 `spec.gregtech.table.<tableName>.<valueName>`。
- 缺失 key 会显示 key 本身并输出 `console.warn`，方便校对时发现问题。

`valueTemplateKey` 使用命名参数。至少有这些参数：

- `{value}`：套模板前的 value 文本。
- `{raw}`：line kind 取到的 raw value。
- `{result}`：表达式结果；没有表达式时是 raw value。
- `templateArgs` 中声明的任意名字。

示例：

```json
{
  "valueTemplateKey": "spec.gregtech.template.percentChance",
  "templateArgs": {
    "chance": "value * 100"
  }
}
```

message：

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

如果不同 locale 可能需要不同语序，使用模板，不要在表达式里拼接短语。

## 语言无关规则

`display.json` 不能包含任何用户可见自然语言。

允许出现在 `display.json` 里：

- i18n key，例如 `spec.gregtech.handler.macerator.name`。
- handler id 和 metadata key。
- 源数据中的枚举值，例如 `PROCESSING`、`FUEL`、`BIO`。
- 电压等级名，例如 `LV`、`HV`、`LuV`、`MAX`。
- 公式和表达式语法。
- 作为公式或游戏常量一部分的单位符号，前提是它们不是自然语言文案。

禁止出现在 `display.json` 里：

- 中文 label 或 literal。
- 英文 label 或 literal，例如 `Duration`、`Steam output shown`、`Hydrogen`。
- 在 `expr` 里拼接自然语言片段。
- v1 旧文本字段，例如 `name_cn`、`label`、`literal`、`prefix`、`suffix`、
  `suffix_table_format`。
- 语言特化 formatter 名称。

所有可见文本都必须进入 `spec/i18n/<locale>.json`，即使当前文本本身是英文。

## 前端加载流程

`GregTechMetadataStrip` 的流程：

1. 加载 `dataset.displaySpecUrl`。
2. 按当前 dataset locale 加载 `dataset.displaySpecMessagesUrl`。
3. 通过 `i18n.global.mergeLocaleMessage(locale, messages)` 合并 messages。
4. 调用 `renderRecipe({ spec, ctx, t })` 渲染配方行。
5. 返回最终 `DisplayItem[]`。

Spec 缓存 key：

- `displaySpecUrl`

Message 缓存 key：

- `<locale>:<displaySpecMessagesUrl>`

dataset locale 控制当前 `vue-i18n` locale。第一阶段 fallback 也是 `zh_CN`。

## 校验

编辑 spec 后运行：

```bash
node tools/validate-display-spec-i18n.mjs
```

校验脚本检查：

- `display.json` 的 `schema` 是 `gregtech-display-spec`。
- `display.json` 的 `version` 是 `2`。
- 每个被引用的 `spec.gregtech.*` key 都存在于 `spec/i18n/zh_CN.json`。
- 表达式字符串 literal 内的 `spec.gregtech.*` key 也存在。
- `display.json` 没有明显 CJK 文本残留。
- `expr`、`show_if`、`valueKeyExpr` 里的可疑自然语言字符串会被报告，除非已加入白名单。

校验脚本故意偏保守。如果它误报真实公式或源数据枚举，优先加窄范围白名单，不要放宽整体扫描。

## 维护清单

新增展示字段时：

1. 在正确 handler 的 `spec/display.json` 里添加 line。
2. 把所有可见文本写入 `spec/i18n/zh_CN.json`。
3. 短语或 locale 相关语序使用 `valueTemplateKey`。
4. `expr` 只用于计算和源数据 lookup。
5. 共用查表文本写入 `tables` 和 `spec.gregtech.table.*` messages。
6. 如果 dev server 需要立刻 serve 编辑后的版本，同步源文件到
   `exports/gtnh/2.8.4/official/spec/...`。
7. 运行 `node tools/validate-display-spec-i18n.mjs`。

在 `tools/build-display-spec.py` 明确升级到 v2 前，不要把它当作当前 spec 的权威来源。
