# CPN JSON Schema 参考

## 完整结构

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
    },
    {
      "name": "DocType",
      "values": ["TaskDA", "MaterialsSI", "收款单", "预算单"]
    }
  ],
  "places": [
    {
      "id": "P1",
      "name": "TaskDA_完成",
      "color": "SubProjectType",
      "initial_marking": [],
      "subproject": "新建阶段",
      "chain": "CPK"
    }
  ],
  "transitions": [
    {
      "id": "T1",
      "name": "发起MaterialsSI",
      "guard": "TaskDA_完成 ∈ marking",
      "subproject": "水电阶段",
      "chain": "MPK"
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
  "dependency_rules": [
    {
      "id": "DEP1",
      "mechanism": "fusion_place",
      "predecessor": "新建.TaskDA",
      "successor": "水电.MaterialsSI",
      "description": "墙体验收后才能开槽布管（跨页面融合库所）"
    },
    {
      "id": "DEP2",
      "mechanism": "guard_condition",
      "predecessor": "SAL.收款",
      "successor": "PKG.预算分配",
      "description": "资金到位后才能分配预算（变迁守卫条件）"
    },
    {
      "id": "DEP3",
      "mechanism": "arc_sequence",
      "predecessor": "CPK.施工",
      "successor": "CPK.验收",
      "description": "施工完成后才能验收（输入弧 token 流动）"
    }
  ]
}
```

## 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| project_id | string | 项目唯一标识 |
| color_sets | array | 着色集合定义 |
| places | array | 库所列表 |
| transitions | array | 变迁列表 |
| arcs | array | 弧列表，type 为 input 或 output |
| dependency_rules | array | 依赖规则，mechanism 为 fusion_place / guard_condition / arc_sequence |
