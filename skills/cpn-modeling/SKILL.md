---
name: cpn-modeling
description: 从自然语言业务场景描述中自动提取 CPN（着色 Petri 网）模型，输出 JSON、CPN Tools XML（.cpn）和 HTML 可视化三种格式。当用户描述业务流程并提到 CPN、Petri 网、建模、FS 规则、工序依赖、库所、变迁等关键词时触发。也适用于装修 BOM、工程工序、业务流程依赖分析等场景，即使用户未明确提到 CPN，只要描述了实体间的先后依赖关系也应触发。
---

# CPN 建模

从自然语言业务场景描述中提取 CPN 模型，依次输出三种格式。

## 提取规则

读取用户输入，识别以下四类元素：

### 1. 库所（Place）
识别**名词性实体 + 状态词**的组合：
- 状态词：完成、待发起、进行中、已验收、已分配、已收款
- 命名格式：`{实体名}_{状态}`
- 示例：`TaskDA_完成`、`MaterialsSI_待发起`、`预算_已分配`

### 2. 变迁（Transition）
识别**动词性业务动作**：
- 动作词：发起、验收、分配、采购、施工、收款、审批、提交
- 命名格式：`{动作}{实体名}`
- 示例：`发起MaterialsSI`、`完成TaskDA`、`分配预算`

### 3. 着色集合（Color Set）
从实体的分类属性中提取：
- 子项目类型（SubProjectType）：从描述中提取阶段名称
- 交易链类型（ChainType）：SAL、PKG、MPK、CPK（或从描述中识别）
- 单据类型（DocType）：从描述中提取单据名称

### 4. FS 规则
识别含**依赖语义**的句子：
- 触发词：必须等待、才能、先于、依赖、完成后、验收后
- 两类规则：
  - `cross_subproject`：不同子项目间的单据依赖
  - `cross_chain`：同一子项目内不同交易链间的依赖
  - `intra_chain`：同一交易链内的工序顺序依赖（如步骤 A 必须先于步骤 B）

---

## 输出步骤

提取完成后，**依次输出以下三部分**：

### 第一部分：JSON 输出

输出完整 JSON，格式参考 `references/json-schema.md`。

用代码块包裹：
```json
{ ... }
```

### 第二部分：CPN XML 输出

输出 CPN Tools 4.x 兼容的 XML，格式参考 `references/cpn-xml-template.md`。

规则：
- 每个子项目对应一个 `<page>`
- colorset 定义在 `<globbox>`
- 跨子项目 FS 规则用融合库所（fusion place）表达
- 跨交易链 FS 规则用变迁的 `<condition>` 守卫条件表达

输出完整 XML，格式严格参考 `references/cpn-xml-template.md` 中的模板结构。

### 第三部分：HTML 可视化

生成完整 HTML 文件内容，格式参考 `references/html-viz-template.md`。

将第一部分生成的 JSON 对象赋值给模板中的 `__CPN_DATA__` 变量。

输出完整 HTML 后，告知用户：
> "将以上 HTML 内容保存为 `.html` 文件，用浏览器打开即可查看 Petri 网可视化。圆形=库所，矩形=变迁，实线=弧，红色虚线=FS依赖。"

---

## 输出质量检查

生成三种输出后，自检以下几点：
1. JSON 中的 `fs_rules` 是否覆盖了描述中所有"必须等待/才能"的依赖关系
2. CPN XML 中每个 `<page>` 是否与 JSON 中的子项目一一对应
3. HTML 中的 `__CPN_DATA__` 是否已替换为实际 JSON
4. CPN XML 中每条 `cross_subproject` FS 规则是否有对应的融合库所，每条 `cross_chain` 规则是否有对应的 `<condition>` 守卫条件

---

## 参考文件

- JSON 格式：`references/json-schema.md`
- CPN XML 格式：`references/cpn-xml-template.md`
- HTML 可视化模板：`references/html-viz-template.md`
