---
name: cpn-modeling
description: 从自然语言业务场景描述中自动提取 CPN（着色 Petri 网）模型，输出 JSON、CPN Tools XML（.cpn）和 HTML 可视化三种格式。当用户描述业务流程并提到 CPN、Petri 网、建模、FS 规则、工序依赖、库所、变迁等关键词时触发。也适用于电商订单、餐厅点餐、审批流程、工程工序等场景，即使用户未明确提到 CPN，只要描述了实体间的先后依赖关系也应触发。当用户询问"如何建模"、"怎么画 Petri 网"、"CPN 怎么用"时，优先输出建模指导。
---

# CPN 建模

## 模式判断

收到用户输入后，先判断意图：

| 用户意图 | 处理方式 |
|---------|---------|
| 描述了一个业务流程，要求建模 | → 执行**建模输出**（三部分） |
| 询问"如何建模"、"怎么识别库所" | → 输出**建模指导**，参考 `references/modeling-guide.md` |
| 要求看示例 | → 展示 `references/example-restaurant.md` |

---

## 建模输出

从自然语言业务场景描述中提取 CPN 模型，依次输出三种格式。

### 提取规则

读取用户输入，识别以下四类元素：

**1. 库所（Place）** — 名词性实体 + 状态词
- 命名格式：`{实体名}_{状态}`
- 示例：`订单_已提交`、`库存_充足`、`审批_待处理`

**2. 变迁（Transition）** — 动词性业务动作
- 命名格式：`{动作}{实体名}`
- 示例：`提交订单`、`扣减库存`、`审批通过`

**3. 着色集合（Color Set）** — 从实体分类属性中提取
- 只对影响流程走向的属性建模

**4. 依赖关系（Dependency）** — 含"才能"、"完成后"、"依赖"等语义
- `arc_sequence`：同链内顺序，token 自然流动
- `guard_condition`：跨链依赖，变迁加守卫条件
- `fusion_place`：跨子项目状态共享，融合库所

### 输出步骤

提取完成后，**依次输出以下三部分**：

**第一部分：JSON**（格式参考 `references/json-schema.md`）

**第二部分：CPN XML**（格式参考 `references/cpn-xml-template.md`）
- 每个子项目对应一个 `<page>`
- `fusion_place` → 目标 page 创建同名融合库所
- `guard_condition` → 变迁的 `<condition>` 写入守卫

**第三部分：HTML 可视化**（格式参考 `references/html-viz-template.md`）
- 将 JSON 对象替换模板中的 `__CPN_DATA__`
- 输出完整 HTML 后告知用户保存为 `.html` 文件用浏览器打开

### 输出质量检查

1. `dependency_rules` 是否覆盖所有"才能/完成后"依赖
2. 多输入变迁的所有输入库所是否都能获得 token（防死锁）
3. HTML 中 `__CPN_DATA__` 是否已替换为实际 JSON
4. 每个 `<page>` 是否与 JSON 子项目一一对应

---

## 建模指导

当用户询问如何建模时，读取 `references/modeling-guide.md` 并按以下结构回答：

1. **识别库所**：名词+状态，哪些需要初始 token
2. **识别变迁**：动词，触发条件
3. **连接弧**：输入弧消耗 token，输出弧产生 token
4. **识别依赖**：三种机制的选择标准
5. **死锁检查**：多输入变迁的常见陷阱

结合用户描述的具体场景举例说明，不要只给抽象定义。

---

## 参考文件

- 建模方法论：`references/modeling-guide.md`
- 经典示例（餐厅点餐）：`references/example-restaurant.md`
- JSON 格式：`references/json-schema.md`
- CPN XML 格式：`references/cpn-xml-template.md`
- HTML 可视化模板：`references/html-viz-template.md`
