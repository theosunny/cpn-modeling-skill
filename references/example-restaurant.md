# 经典示例：餐厅点餐流程

## 场景描述

顾客进入餐厅后，服务员接单并提交厨房；厨房在库存充足时开始备餐，完成后出餐；收银员在出餐完成后为顾客结账；结账完成后顾客离开。

模型按**角色链**划分，每条链对应一个参与者：顾客链追踪顾客状态，服务员链管理服务员资源，厨房链管理备餐过程和库存，收银链管理结账过程。资源库所（服务员_空闲、厨房_空闲、收银员_空闲、库存_充足）初始持有 token，被变迁消耗后在适当时机归还，防止资源竞争导致死锁。

**依赖关系：**
- 接单完成后才能开始备餐（arc_sequence，token 流动）
- 备餐完成后才能出餐（arc_sequence，token 流动）
- 出餐完成后才能结账（arc_sequence，token 流动）
- 备餐时直接消耗库存 token（arc_sequence，T2 同时输入订单和库存）
- 服务员、厨房、收银员均为资源库所（fusion_place，占用后归还）

---

## JSON 模型

```json
{
  "project_id": "REST-ORDER-002",
  "color_sets": [
    { "name": "OrderType",    "values": ["堂食", "外卖"] },
    { "name": "ResourceType", "values": ["空闲"] }
  ],
  "places": [
    { "id": "P1",  "name": "顾客_待点餐",  "color": "OrderType",    "initial_marking": ["1`堂食"], "subproject": "顾客链", "chain": "顾客链" },
    { "id": "P2",  "name": "订单_已接收",  "color": "OrderType",    "initial_marking": [],         "subproject": "顾客链", "chain": "顾客链" },
    { "id": "P3",  "name": "出餐_已完成",  "color": "OrderType",    "initial_marking": [],         "subproject": "顾客链", "chain": "顾客链" },
    { "id": "P4",  "name": "顾客_已离开",  "color": "OrderType",    "initial_marking": [],         "subproject": "顾客链", "chain": "顾客链" },
    { "id": "P5",  "name": "服务员_空闲",  "color": "ResourceType", "initial_marking": ["1`空闲"], "subproject": "服务员链", "chain": "服务员链" },
    { "id": "P6",  "name": "备餐_进行中",  "color": "OrderType",    "initial_marking": [],         "subproject": "厨房链", "chain": "厨房链" },
    { "id": "P7",  "name": "备餐_已完成",  "color": "OrderType",    "initial_marking": [],         "subproject": "厨房链", "chain": "厨房链" },
    { "id": "P8",  "name": "厨房_空闲",    "color": "ResourceType", "initial_marking": ["1`空闲"], "subproject": "厨房链", "chain": "厨房链" },
    { "id": "P9",  "name": "库存_充足",    "color": "ResourceType", "initial_marking": ["1`空闲"], "subproject": "厨房链", "chain": "厨房链" },
    { "id": "P10", "name": "库存_已扣减",  "color": "ResourceType", "initial_marking": [],         "subproject": "厨房链", "chain": "厨房链" },
    { "id": "P11", "name": "收银员_空闲",  "color": "ResourceType", "initial_marking": ["1`空闲"], "subproject": "收银链", "chain": "收银链" }
  ],
  "transitions": [
    { "id": "T1", "name": "接单",     "guard": "true", "subproject": "服务员链", "chain": "服务员链" },
    { "id": "T2", "name": "开始备餐", "guard": "true", "subproject": "厨房链",  "chain": "厨房链"  },
    { "id": "T3", "name": "完成备餐", "guard": "true", "subproject": "厨房链",  "chain": "厨房链"  },
    { "id": "T4", "name": "出餐",     "guard": "true", "subproject": "厨房链",  "chain": "厨房链"  },
    { "id": "T5", "name": "结账",     "guard": "true", "subproject": "收银链",  "chain": "收银链"  }
  ],
  "arcs": [
    { "id": "A1",  "from": "P1",  "to": "T1", "type": "input",  "annotation": "1`堂食" },
    { "id": "A2",  "from": "P5",  "to": "T1", "type": "input",  "annotation": "1`空闲" },
    { "id": "A3",  "from": "T1",  "to": "P2", "type": "output", "annotation": "1`堂食" },
    { "id": "A4",  "from": "T1",  "to": "P5", "type": "output", "annotation": "1`空闲" },
    { "id": "A5",  "from": "P2",  "to": "T2", "type": "input",  "annotation": "1`堂食" },
    { "id": "A6",  "from": "P8",  "to": "T2", "type": "input",  "annotation": "1`空闲" },
    { "id": "A7",  "from": "P9",  "to": "T2", "type": "input",  "annotation": "1`空闲" },
    { "id": "A8",  "from": "T2",  "to": "P6", "type": "output", "annotation": "1`堂食" },
    { "id": "A9",  "from": "T2",  "to": "P10","type": "output", "annotation": "1`空闲" },
    { "id": "A10", "from": "P6",  "to": "T3", "type": "input",  "annotation": "1`堂食" },
    { "id": "A11", "from": "T3",  "to": "P7", "type": "output", "annotation": "1`堂食" },
    { "id": "A12", "from": "T3",  "to": "P8", "type": "output", "annotation": "1`空闲" },
    { "id": "A13", "from": "P7",  "to": "T4", "type": "input",  "annotation": "1`堂食" },
    { "id": "A14", "from": "T4",  "to": "P3", "type": "output", "annotation": "1`堂食" },
    { "id": "A15", "from": "P3",  "to": "T5", "type": "input",  "annotation": "1`堂食" },
    { "id": "A16", "from": "P11", "to": "T5", "type": "input",  "annotation": "1`空闲" },
    { "id": "A17", "from": "T5",  "to": "P4", "type": "output", "annotation": "1`堂食" },
    { "id": "A18", "from": "T5",  "to": "P11","type": "output", "annotation": "1`空闲" }
  ],
  "dependency_rules": [
    {
      "id": "DEP1",
      "mechanism": "arc_sequence",
      "predecessor": "顾客链.接单",
      "successor": "厨房链.开始备餐",
      "description": "接单完成后才能开始备餐（P2 token 流动）"
    },
    {
      "id": "DEP2",
      "mechanism": "arc_sequence",
      "predecessor": "厨房链.完成备餐",
      "successor": "厨房链.出餐",
      "description": "备餐完成后才能出餐（P7 token 流动）"
    },
    {
      "id": "DEP3",
      "mechanism": "arc_sequence",
      "predecessor": "厨房链.出餐",
      "successor": "收银链.结账",
      "description": "出餐完成后才能结账（P3 token 流动）"
    },
    {
      "id": "DEP4",
      "mechanism": "arc_sequence",
      "predecessor": "厨房链.库存_充足",
      "successor": "厨房链.开始备餐",
      "description": "T2 直接消耗库存 token，库存不足则 T2 无法触发（arc_sequence）"
    },
    {
      "id": "DEP5",
      "mechanism": "fusion_place",
      "predecessor": "服务员链.服务员_空闲",
      "successor": "服务员链.接单",
      "description": "服务员资源占用：接单时消耗，接单完成后立即归还（P5）"
    },
    {
      "id": "DEP6",
      "mechanism": "fusion_place",
      "predecessor": "厨房链.厨房_空闲",
      "successor": "厨房链.开始备餐",
      "description": "厨房资源占用：开始备餐时消耗，完成备餐后归还（P8）"
    },
    {
      "id": "DEP7",
      "mechanism": "fusion_place",
      "predecessor": "收银链.收银员_空闲",
      "successor": "收银链.结账",
      "description": "收银员资源占用：结账时消耗，结账完成后归还（P11）"
    }
  ]
}
```

---

## 建模要点说明

### 为什么按角色链划分，而不是按阶段划分？

按阶段划分（点餐阶段/备餐阶段/出餐阶段/结账阶段）是流程图思维，描述的是"事情发生的顺序"。CPN 建模的核心是**并发与资源竞争**，应该描述"谁在做什么、谁在等什么"。

按角色链划分后，每条链的库所和变迁归属清晰：
- 顾客链：追踪顾客状态（待点餐 → 已点餐 → 已离开）
- 服务员链：管理服务员资源（空闲 → 接单占用 → 归还）
- 厨房链：管理备餐过程和库存（空闲 → 备餐占用 → 归还；库存消耗）
- 收银链：管理结账过程（空闲 → 结账占用 → 归还）

### 为什么删除独立的 T6（扣减库存）？

原模型中 T6 只有 P8→T6→P9，没有来自订单的输入弧，可以在任意时刻独立触发，与主流程完全脱节。这是建模错误：库存扣减必须与备餐动作绑定。

正确做法是让 T2（开始备餐）同时消耗库存 token：T2 的输入 = P2(订单_已接收) + P8(厨房_空闲) + P9(库存_充足)。这样库存不足时 T2 无法触发，自然阻塞备餐，无需额外的守卫条件。

### 为什么守卫条件全部改为 `"true"`？

守卫条件（guard）的作用是在变迁**已满足输入弧条件**的前提下，附加额外的过滤逻辑。如果守卫条件只是重复检查"某输入库所有 token"，那它与输入弧完全冗余。

新模型中所有依赖都通过输入弧表达：T2 需要 P2+P8+P9 同时有 token，T5 需要 P3+P11 同时有 token。输入弧已经保证了这些条件，守卫条件设为 `"true"` 即可。

### 资源库所模式（Resource Place Pattern）

服务员_空闲（P5）、厨房_空闲（P8）、收银员_空闲（P11）、库存_充足（P9）都是资源库所，特征是：
1. 初始持有 token（`initial_marking` 非空）
2. 被变迁消耗后，在流程的某个后续步骤归还

归还时机：
- 服务员：接单（T1）时消耗，T1 完成后立即归还（服务员只负责接单，不跟单）
- 厨房：开始备餐（T2）时消耗，完成备餐（T3）后归还
- 收银员：结账（T5）时消耗，T5 完成后立即归还
- 库存：开始备餐（T2）时消耗，流入 P10（库存_已扣减），不归还（库存是一次性消耗）

### 防死锁验证

逐一检查每个变迁的输入库所在流程中能否获得 token：

| 变迁 | 输入库所 | token 来源 |
|------|----------|-----------|
| T1 接单 | P1 顾客_待点餐 | 初始 token | 
| T1 接单 | P5 服务员_空闲 | 初始 token |
| T2 开始备餐 | P2 订单_已接收 | T1 输出 |
| T2 开始备餐 | P8 厨房_空闲 | 初始 token，T3 归还 |
| T2 开始备餐 | P9 库存_充足 | 初始 token |
| T3 完成备餐 | P6 备餐_进行中 | T2 输出 |
| T4 出餐 | P7 备餐_已完成 | T3 输出 |
| T5 结账 | P3 出餐_已完成 | T4 输出 |
| T5 结账 | P11 收银员_空闲 | 初始 token，T5 归还 |

所有输入库所均有明确的 token 来源，无死锁。
