# GT Recipe Display Spec — Schema

> 字典文件按语言拆分（zh_CN.json / en_US.json / ...），后端通过 `dataset.displaySpecPath` 告诉前端哪个 dataset 用哪份 spec。前端启动时 GET 拉一次缓存，按 `recipe.handler` 取 lines，跑解析器输出 `DisplayItem[]`。

## 文件结构

```json
{
  "version": 1,
  "language": "zh_CN",
  "tables": {
    "<table_name>": { "<key>": "<value>", ... }
  },
  "handlers": {
    "<handler_id>": {
      "name_cn": "<注释，人类可读，不参与渲染>",
      "lines": [ { ...line... }, ... ]
    }
  }
}
```

- `version`: schema 版本号。本文档为 v1
- `language`: BCP 47 风格的 dataset 语言标识；用于校验
- `tables`: 共享映射表（材料名 / tier 名 / spacetime fancy name 等）。所有 handler 都能引用，避免在每段 spec 里重复
- `handlers`: 按 DB 实际出现的 handler id（`split_part(recipe_id, ':', 2)`）一段独立 lines

**handler 必须和 DB 一一对得上**。DB 有但 spec 没列的 handler，前端日志 `[displaySpec] no spec for handler={id}`，displayItems 返回空数组（不糊弄）。

---

## ctx：解析器读到的数据

前端解析器拿到一条 recipe 后，先把所有 DB 字段读进 ctx，line 通过 `kind` 决定取哪段：

```ts
type RecipeCtx = {
  // 顶层
  handler: string,
  recipe_kind: 'PROCESSING' | 'FUEL',
  voltage: number | null,
  voltage_tier: string | null,    // "ULV"/"LV"/.../"MAX"
  amperage: number | null,
  duration_ticks: number,
  special_value: number | null,
  // 流体 / 物品输入（部分 handler 用）
  fluid_inputs: Array<{ amount: number, fluid_id: string }>,
  item_inputs: Array<{ item_id: string, amount: number }>,
  // 特殊物品（specialItems）
  special_items: Array<{ item_id: string }>,
  // metadata（所有 key 全读进来）
  metadata: Record<string, ScalarOrJson>,
}
```

---

## line — 一行的形状

```typescript
type Line = {
  // ─── 必填 ───
  kind: LineKind,                  // 决定取数据的方式（详见下）

  // ─── 取数据相关 ───
  key?: string,                    // kind=metadata/flag/particle_icon 时：metadata.{key}
  list_index?: number,             // kind=special_item / fluid_input 时数组下标，默认 0

  // ─── 渲染相关 ───
  label?: string | null,           // 行 label；null 表示无 label（整行就是 value）
  expr?: string,                   // 可选 JS 表达式（expr-eval 沙箱），输入 value/tables，输出新 value
  scale?: number,                  // 数值乘法（在 format 之前），常用 1000 (keV→eV) / 365 (days→mc_days)
  format?: FormatName,             // 数值→字符串格式化
  prefix?: string,                 // 字面拼前缀
  suffix?: string,                 // 字面拼后缀（如 " EU" / " K"）
  suffix_table?: string,           // 引用 tables.{name}，默认按 raw value 当 key 查
  suffix_table_lookup?: 'exact' | 'ceil', // exact 默认；ceil 取最小 table_key >= raw value
  suffix_table_format?: string,    // 查表后套模板（{0} 占位），如 " ({0})"
  literal?: string,                // kind=flag/text 时输出的字面文字
  color_code?: string,             // 颜色码前缀（如 "§9§n§l"）

  // ─── 显示条件 ───
  show_if_true?: boolean,          // value 为 true 才显示（bool 字段用）
  show_if_present?: boolean,       // metadata 不存在时整行不显示（默认 true）
  show_if?: string,                // expr 返回 true 才显示，否则跳过
};
```

---

## LineKind — 11 种处理类型

每种 kind 决定**从 ctx 取什么数据**，取出来后统一进入"格式化 + 拼接"流程。

### 顶层派生

| kind | 取什么 | 输出几行 |
|---|---|---|
| `total_eu` | `voltage × duration_ticks × amperage` | 1 行（除非 voltage 为 null） |
| `voltage_block` | `voltage / voltage_tier / amperage` | amp=1 → 1 行；amp>1 → 3 行（使用 / 电压 / 电流） |
| `duration` | `duration_ticks` | 1 行（按 unit 转秒/tick/天） |
| `fuel_heat` | `special_value`（× scale，常配 1000） | 1 行 |
| `large_boiler_table` | `special_value` 走大锅炉公式 | 4 行（青铜/钢/钛/钨钢） |

### Metadata 派生

| kind | 取什么 | 备注 |
|---|---|---|
| `metadata` | `metadata[key]` 标量 | 数值 / 字符串 / 枚举均可；缺失则整行跳过 |
| `metadata_json` | `metadata[key]` 整个 JSON 对象（绑定到 expr 的 value） | 必须配 `expr`，由 expr 决定怎么取多字段并组合 |
| `flag` | `metadata[key]` 必须是 bool | 配 `show_if_true: true` + `literal: "需要超净间"`；false 或缺失跳过 |

### 特殊行

| kind | 取什么 | 备注 |
|---|---|---|
| `text` | 不取数据 | 纯 hardcode 提示，配 `literal: "进一步促进产出"` |
| `special_item` | `special_items[list_index]` | 用 itemId 渲染图标（前端组件，不是文字） |
| `fluid_input` | `fluid_inputs[list_index]` | 取 `.amount` 当 value |

---

## FormatName — 数值格式化

每个 format 名对应前端一个**纯函数**，输入 value（已经被 expr/scale 处理过），输出字符串。

| format | 输入 | 输出 |
|---|---|---|
| `direct` | any | `String(value)` |
| `comma_int` | int/long | `"4,501"`（千分位） |
| `comma_long` | long | `"1,000,000,000"` |
| `comma_double_1` | double | 千分位 + 一位小数 |
| `percent_int_x100` | int | `"55%"`（已是百分数，不乘 100） |
| `percent_double_x100` | double 0..1 | `"55%"`（×100 后整数） |
| `decimal_0` | double | `"55"`（去小数） |
| `decimal_1` | double | `"55.5"` |
| `decimal_2` | double | `"55.55"` |
| `scientific` | double | `"5.12e-02"` |
| `duration_seconds` | int ticks | `"160 秒"`（带"秒"单位；ticks % 20 == 0 整数否则一位小数） |
| `duration_seconds_en` | int ticks | `"160 secs"`（英文） |
| `duration_ticks` | int | `"5 tick"` / `"5 ticks"` |
| `duration_auto_unit` | double seconds | 自动选秒/分/时/天单位（isotopedecay 用） |
| `java_double` | number | 模拟 Java `Double.toString` 的展示；整数保留 `.0`，必要时用 `E` 科学计数法 |
| `bool_yes_no` | bool | `"Yes"` / `"No"` |
| `bool_yes_no_cn` | bool | `"是"` / `"否"` |

新增 format 必须在前端 displaySpec.ts 里加一个函数；不在表里的 format 名加载时报错。

---

## tables — 共享映射表

```json
{
  "tables": {
    "tier_name": {
      "0": "ULV", "1": "LV", "2": "MV", "3": "HV", "4": "EV", "5": "IV",
      "6": "LuV", "7": "ZPM", "8": "UV", "9": "UHV", "10": "UEV",
      "11": "UIV", "12": "UMV", "13": "UXV", "14": "MAX"
    },
    "coil_material": {
      "1801": "白铜", "2701": "坎塔尔合金", "3601": "镍铬合金",
      "4501": "钛铂钒", "5401": "高速钢-G", "6301": "高速钢-S",
      "7201": "硅岩", "8101": "硅岩合金", "9001": "三元金属",
      "9901": "通流琥珀金", "10801": "觉醒龙锭", "11701": "无尽",
      "12601": "海珀珍", "13501": "永恒", "*": "永恒+"
    },
    "chemplant_casing": {
      "0": "青铜", "1": "钢", "2": "不锈钢", "3": "铝",
      "4": "钛", "5": "钨钢", "6": "劳伦姆合金", "7": "铱"
    },
    "eoh_spacetime_fancy": {
      "0": "Schwarzschild", "1": "Reissner-Nordström", "2": "Kerr",
      "3": "Kerr-Newman", "4": "Lense-Thirring", "5": "Tipler",
      "6": "Alcubierre", "7": "van Stockum", "8": "Gallifreyan"
    }
  }
}
```

引用方式（在 line 里）：

```json
{ "kind": "metadata", "key": "coil_heat",
  "label": "热容",
  "format": "comma_int",
  "suffix": " K",
  "suffix_table": "coil_material",
  "suffix_table_lookup": "ceil",
  "suffix_table_format": " ({0})" }
```

渲染流程：
1. ctx.metadata.coil_heat = 4500
2. format: `comma_int(4500)` → `"4,500"`
3. suffix: `+ " K"` → `"4,500 K"`
4. suffix_table + `suffix_table_lookup: "ceil"`：取最小的 `table_key >= 4500`，即 `4501` → `"钛铂钒"`
5. suffix_table_format `" ({0})"` 把 `{0}` 替换成 `"钛铂钒"` → `" (钛铂钒)"`
6. 拼到末尾 → `"4,500 K (钛铂钒)"`

---

## expr — 沙箱表达式

只在以下场景用：
- 位运算（`nke_range % 10000`）
- JSON 多字段组合（`value.minEnergy * 1000 + "-" + value.maxEnergy * 1000`）
- 条件分支（`subZero ? "加热" : "冷却"`）
- 拼接计算结果（`lftr_output_power × duration_ticks`）

**绑定到 expr 的变量**：
- `value` —— 当前 line 取到的数据（按 kind 不同）
- `ctx` —— 完整 ctx 对象（极少用，主要用于跨字段计算）
- `tables` —— spec.tables 的引用

**可用辅助函数**：
- `comma(value)` —— 千分位整数格式化
- `tier(value)` —— 按 EU/t 计算 GT 电压等级名，超过 MAX 返回 `MAX+`
- `lookup(table, key)` —— 查 `tables` 映射
- `int(value)` / `floor(value)` / `ceil(value)` / `round(value)` —— 数值取整
- `band(value, mask)` / `bor(value, mask)` / `shr(value, bits)` —— 位运算
- `len(value)` —— 数组长度

**表达式引擎**：[expr-eval](https://www.npmjs.com/package/expr-eval) 沙箱化 JavaScript 表达式。**不支持** `new`、函数定义、`this`、`window` 等；支持算术 / 比较 / 三元 / 数组下标 / `.` 字段访问 / 内置函数。

**禁止**：`eval()` / `new Function()` / 任意 import / 任何 IO。

加载时所有 expr 提前编译，编译失败的 spec **拒绝加载**（前端启动报错）。

---

## 5 个完整示例

### 示例 1：研磨机（最简单，只有标准三件套）

```json
"macerator": {
  "name_cn": "研磨机",
  "lines": [
    { "kind": "total_eu", "label": "总计",
      "format": "comma_long", "suffix": " EU" },
    { "kind": "voltage_block",
      "single": { "label": "使用",
                  "expr": "value + ' EU/t (' + ctx.voltage_tier + ')'",
                  "format": "comma_int" }
    },
    { "kind": "duration", "label": "时间",
      "format": "duration_seconds" }
  ]
}
```

### 示例 2：高炉（带 coil_heat 表查 + cleanroom flag）

```json
"blastfurnace": {
  "name_cn": "高炉",
  "lines": [
    { "kind": "total_eu", "label": "总计",
      "format": "comma_long", "suffix": " EU" },
    { "kind": "voltage_block",
      "single": { "label": "使用", "expr": "value + ' EU/t (' + ctx.voltage_tier + ')'",
                  "format": "comma_int" }
    },
    { "kind": "duration", "label": "时间",
      "format": "duration_seconds" },
    { "kind": "metadata", "key": "coil_heat",
      "label": "热容",
      "format": "comma_int",
      "suffix": " K",
      "suffix_table": "coil_material",
      "suffix_table_lookup": "ceil",
      "suffix_table_format": " ({0})" },
    { "kind": "flag", "key": "cleanroom",
      "label": null,
      "show_if_true": true,
      "literal": "需要超净间" }
  ]
}
```

### 示例 3：电弧炉（amp=3，voltage_block 三行）

```json
"arcfurnace": {
  "name_cn": "电弧炉",
  "lines": [
    { "kind": "total_eu", "label": "总计",
      "format": "comma_long", "suffix": " EU" },
    { "kind": "voltage_block",
      "split": [
        { "label": "使用",
          "expr": "value",
          "format": "comma_int", "suffix": " EU/t " },
        { "label": "电压",
          "expr": "ctx.voltage / ctx.amperage",
          "format": "comma_int",
          "suffix_table_format": " EU/t ({tier_name})",
          "expr_post": "result + ' EU/t (' + ctx.voltage_tier + ')'"
        },
        { "label": "电流",
          "expr": "ctx.amperage",
          "suffix": " A" }
      ]
    },
    { "kind": "duration", "label": "时间",
      "format": "duration_seconds" }
  ]
}
```

> **注**：amp>1 时 voltage_block 用 `split` 三段，单独每段写 label/expr/format/suffix。

### 示例 4：和谐之眼（EoH，8 个新 metadata + Warning hardcode）

```json
"tt_eyeofharmony": {
  "name_cn": "和谐之眼",
  "lines": [
    { "kind": "duration", "label": "时间",
      "format": "duration_seconds" },
    { "kind": "metadata", "key": "eoh_hydrogen",
      "label": "Hydrogen",
      "format": "comma_long", "suffix": " L" },
    { "kind": "metadata", "key": "eoh_helium",
      "label": "Helium",
      "format": "comma_long", "suffix": " L" },
    { "kind": "metadata", "key": "eoh_spacetime_tier",
      "label": "Spacetime Tier",
      "expr": "tables.eoh_spacetime_fancy[String(value)]",
      "color_code": "§l" },
    { "kind": "metadata", "key": "eoh_eu_output",
      "label": "EU Output",
      "format": "comma_long", "suffix": " EU" },
    { "kind": "metadata", "key": "eoh_eu_start_cost",
      "label": "EU Input",
      "format": "comma_long", "suffix": " EU" },
    { "kind": "metadata", "key": "eoh_base_success_chance",
      "label": "Base Recipe Chance",
      "scale": 100,
      "format": "decimal_0", "suffix": "%" },
    { "kind": "metadata", "key": "eoh_energy_efficiency",
      "label": "Recipe Energy Efficiency",
      "scale": 100,
      "format": "decimal_0", "suffix": "%" }
  ]
}
```

### 示例 5：兰系列目标室（JSON metadata 多字段组合 + 粒子图标）

```json
"lanth_targetchamber": {
  "name_cn": "兰系列目标室",
  "lines": [
    { "kind": "voltage_block",
      "split": [
        { "label": "使用", "expr": "value",
          "format": "comma_int", "suffix": " EU/t " },
        { "label": "电压",
          "expr": "ctx.voltage / ctx.amperage + ' EU/t (' + ctx.voltage_tier + ')'",
          "format": "comma_int" },
        { "label": "电流", "expr": "ctx.amperage", "suffix": " A" }
      ]
    },
    { "kind": "metadata_json", "key": "target_chamber_metadata",
      "label": "能量",
      "expr": "(value.minEnergy * 1000).toFixed(0) + '-' + (value.maxEnergy * 1000).toFixed(0)",
      "suffix": " eV" },
    { "kind": "metadata_json", "key": "target_chamber_metadata",
      "label": "聚焦",
      "expr": "'>=' + value.minFocus" },
    { "kind": "metadata_json", "key": "target_chamber_metadata",
      "label": "数量",
      "expr": "value.amount",
      "format": "comma_int" },
    { "kind": "particle_icon", "key": "target_chamber_metadata",
      "json_path": "particleItem" }
  ]
}
```

---

## 验证规则（前端启动时检查）

加载 spec 后前端校验：

1. **handler id 唯一**
2. **每个 line 必有 kind**，且 kind 在已知枚举内
3. **format 在已知枚举内**
4. **expr 编译通过**（用 expr-eval 编译；失败的整段 spec 拒绝加载）
5. **suffix_table 引用的 table 存在**
6. **JSON 结构形式正确**（schema 简版校验）

任一失败：spec 拒绝加载，控制台报错，前端走"无 displayItems"分支（recipe 还是能显示，只是没有 metadata 行）。

---

## 多语言

每种语言一份 spec：
- `webnei/ui/src/components/recipe/gregtech/spec/zh_CN.json`
- `webnei/ui/src/components/recipe/gregtech/spec/en_US.json`

`dataset.language` 决定用哪份。**不同语言的 spec 不共享 lines，可以独立调整 label 甚至 line 结构**（比如 `Base success chance` 英文 hardcode，zh_CN 也保留英文）。

后端在 `/api/datasets/{id}` 响应里加 `displaySpecPath: "/spec/gregtech/zh_CN.json"`，前端按此 fetch。

---

## 何时改 spec

| 场景 | 改什么 |
|---|---|
| 加新 handler 显示 | spec.handlers 加一段 |
| 改某 handler 渲染规则 | 改该 handler 的 lines |
| 加新格式（如某行要科学计数） | 前端 displaySpec.ts 加 format 函数 + spec 用 |
| 加新 line kind | 前端解析器加 case + 本文档同步 + spec 用 |
| 修复术语翻译 | 改 zh_CN.json 的 label / literal |
| 新增 GT 版本 | exporter 重导后改 spec 适配新字段 |

不要在 Vue 组件 / 后端 Java 里硬编码 metadata key 名 / label / 单位字符串。
