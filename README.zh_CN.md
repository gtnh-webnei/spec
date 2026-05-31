# GregTech 展示 Spec

语言：[English](README.md) | [中文](README.zh_CN.md)

这个仓库存放 WebNEI 使用的 GregTech 配方展示 spec。它是人工维护的规则源，用来描述每个 GregTech handler 的 metadata 行应该如何渲染。

## 文件说明

- `display.json`：语言无关的展示规则。这里保存 handler 结构、line kind、metadata key、公式、查表、format 名称、显示条件和 i18n key。
- `i18n/zh_CN.json`：`display.json` 引用的本地化消息。当前只维护 `zh_CN`，里面包含所有可见文本，包括目前 UI 中仍然是英文的文本。
- `SCHEMA.md`：完整英文 v2 schema 合约。
- `SCHEMA.zh_CN.md`：完整中文 v2 schema 合约。

旧版单文件 `zh_CN.json` 已废弃，不要再恢复。

## 修改流程

修改 GregTech metadata 的可见文案时：

1. 把文案写到 `i18n/zh_CN.json`。
2. 在 `display.json` 里用 `spec.gregtech.*` key 引用。
3. 涉及语序、短语、单位组合时，优先使用 `valueTemplateKey`。
4. `expr` 只放计算逻辑，不拼自然语言。
5. 提交前运行校验。

修改展示行为时：

1. 修改 `display.json`。
2. 如有新增可见文本，同步写入 `i18n/zh_CN.json`。
3. 如果新增字段、format、line kind 或表达式函数，同步更新 schema 文档。
4. 如果本地 dev server 需要立刻 serve 新内容，同步更新导出目录副本。

## 校验

在 workspace 根目录运行：

```bash
node tools/validate-display-spec-i18n.mjs
```

校验脚本会检查：

- `display.json` 是 v2。
- 所有被引用的 `spec.gregtech.*` key 都存在于 `i18n/zh_CN.json`。
- 语言无关的 `display.json` 里没有明显中文或自然语言文本残留。

## 运行时副本

前端在 dev/prod 下不会直接读取这个仓库路径。实际被 serve 的运行时副本在：

```text
exports/gtnh/2.8.4/official/spec/display.json
exports/gtnh/2.8.4/official/spec/i18n/zh_CN.json
```

后端 `DatasetSummary` 通过这两个字段告诉前端资源地址：

- `displaySpecUrl`
- `displaySpecMessagesUrl`

这个仓库是人工维护源文件。需要本地运行验证时，要保持导出目录副本同步。
