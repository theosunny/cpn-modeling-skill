# CPN 建模 Skill 设计文档

**日期：** 2026-05-07  
**领域：** 装修 BOM 业务流程建模  
**方案：** 直接提取法（方案 A）

---

## 概述

该 skill 使 Claude 能够从自然语言业务场景描述中，自动提取 CPN（着色 Petri 网）模型元素，并同时输出三种格式：JSON、CPN Tools 兼容 XML（.cpn）、HTML 可视化。

目标领域为装修 BOM 业务流程，核心实体包括项目、子项目、交易链、单据，以及跨子项目和跨交易链的 FS（Finish-Start）依赖规则。

---

## 触发条件

用户描述业务流程时，出现以下关键词之一即触发：

- CPN、Petri 网、着色 Petri 网
- 建模、流程建模、业务建模
- FS 规则、依赖关系、工序约束
- 库所、变迁、令牌

---

## 处理流程

```
自然语言输入 → 实体提取 → 三路并行输出
                              ├── JSON
                              ├── CPN XML (.cpn)
                              └── HTML 可视化
```

---

## 实体提取规则

### 库所（Place）
- 识别名词性实体 + 状态词
- 状态词：完成、待发起、进行中、已验收、已分配
- 示例：`TaskDA_完成`、`MaterialsSI_待发起`

### 变迁（Transition）
- 识别动词性业务动作
- 动作词：发起、验收、分配、采购、施工、收款
- 示例：`发起MaterialsSI`、`完成TaskDA`

### 着色（Color Set）
- 实体的分类属性作为 token 颜色
- 子项目类型：新建、水电、瓦工、木工、油漆、安装
- 交易链类型：SAL（销售链）、PKG（预算链）、MPK（材料采购链）、CPK（施工链）
- 单据类型：TaskDA（施工验收单）、MaterialsSI（材料要货单）

### FS 规则
- 识别含依赖语义的句子
- 关键词：必须等待、才能、先于、依赖、完成后
- 两类规则：
  - **跨子项目 FS**：不同子项目间的单据依赖
  - **跨交易链 FS**：同一子项目内不同交易链间的依赖

---

## 输出格式

### 1. JSON Schema

```json
{
  "project_id": "PRJ2025001",
  "color_sets": [
    {
      "name": "SubProjectType",
      "values": ["新建", "水电", "瓦工", "木工", "油漆", "安装"]
    },
    {
      "name": "ChainType",
      "values": ["SAL", "PKG", "MPK", "CPK"]
    }
  ],
  "places": [
    {
      "id": "P1",
      "name": "TaskDA_完成",
      "color": "SubProjectType",
      "initial_marking": [],
      "subproject": "新建阶段"
    }
  ],
  "transitions": [
    {
      "id": "T1",
      "name": "发起MaterialsSI",
      "guard": "TaskDA_完成 ∈ marking"
    }
  ],
  "arcs": [
    {
      "id": "A1",
      "from": "P1",
      "to": "T1",
      "type": "input",
      "annotation": "1`新建"
    }
  ],
  "fs_rules": [
    {
      "id": "FS1",
      "type": "cross_subproject",
      "predecessor": "新建.TaskDA",
      "successor": "水电.MaterialsSI",
      "description": "墙体验收后才能开槽布管"
    },
    {
      "id": "FS2",
      "type": "cross_chain",
      "predecessor": "SAL.收款",
      "successor": "PKG.预算分配",
      "description": "资金到位后才能分配预算"
    }
  ]
}
```

### 2. CPN XML (.cpn)

遵循 CPN Tools 标准格式：

```
<cpnet>
  <globbox>
    <!-- colorset 定义 -->
    <colorset id="SubProjectType" .../>
  </globbox>
  <page id="新建阶段">
    <!-- 每个子项目对应一个 page -->
    <place id="P1" .../>
    <trans id="T1" .../>
    <arc id="A1" .../>
  </page>
  ...
</cpnet>
```

- 每个子项目对应一个 `<page>`
- colorset 统一定义在 `<globbox>`
- FS 规则表达为变迁的 `<guard>` 条件

### 3. HTML 可视化

- 渲染引擎：SVG（内嵌于 HTML）
- 圆形节点 = 库所，按子项目着色区分
- 矩形节点 = 变迁
- 实线箭头 = 弧（输入/输出）
- 虚线箭头 = FS 依赖关系
- 交互：支持缩放、拖拽、节点悬停显示详情

---

## 层级结构

```
项目 Project
└── 子项目 SubProject（1:N，最多6个阶段）
    └── 交易链 Chain（SAL / PKG / MPK / CPK）
        └── 单据 Document（TaskDA / MaterialsSI 等）
```

---

## FS 规则类型

| 类型 | 说明 | 示例 |
|------|------|------|
| cross_subproject | 跨子项目依赖 | 新建.TaskDA → 水电.MaterialsSI |
| cross_chain | 跨交易链依赖 | SAL.收款 → PKG.预算分配 → MPK/CPK.执行 |

---

## 约束与边界

- 输入为中文自然语言描述
- 领域限定为装修 BOM 业务流程（可扩展到其他行业）
- CPN XML 兼容 CPN Tools 4.x 格式
- HTML 可视化为静态文件，无需服务器
