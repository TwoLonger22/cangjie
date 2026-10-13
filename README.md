# 智能AA记账本 - 后端接口文档

## 📁 代码结构(在 `entry\src\main\cangjie`中 )

logic/  
└── SettlementSolver.cj     # 核心结算算法

parser/  
└── JsonParser.cj           # JSON解析器，解析账单输入BillInput

model/  
└── BillModels.cj               # 数据模型（Person, Bill, 转账 Transfer, BillInput, 结算结果模型 SettlementResult）

pages/  
├── BillEntryPage.cj    入口界面  
└── SettlementPage.cj   结果展示界面

## 📦 数据模型 （详见`BillModels.cj`）

### Person
```cangjie
public class Person {
    public init(name: String)
    public mut prop name: String
}
```

### Bill
```cangjie
public class Bill {
  public init(payer: Person, amount: Float64, consumers: ArrayList<Person>)
  public init(payer: Person, amount: Float64, consumers: ArrayList<Person>, note: String)
  public init(payer: Person, amount: Float64, consumers: ArrayList<Person>, weights: ArrayList<Float64>)
  public init(payer: Person, amount: Float64, consumers: ArrayList<Person>, weights: ArrayList<Float64>, note: String)
    public mut prop payer: Person
    public mut prop amount: Float64
    public mut prop note: String  // 备注
    public mut prop consumers: ArrayList<Person>
  public mut prop weights: ArrayList<Float64>  // 可选，和 consumers 对齐
}
```

### BillInput
```cangjie
public class BillInput {
    public init(participants: ArrayList<Person>, bills: ArrayList<Bill>)
    public mut prop participants: ArrayList<Person>
    public mut prop bills: ArrayList<Bill>
}
```

### Transfer
```cangjie
public class Transfer {
    public init(from: Person, to: Person, amount: Float64)
    public mut prop from: Person
    public mut prop to: Person
    public mut prop amount: Float64
}
```

### SettlementResult
```cangjie
public class SettlementResult {
  public init(transfers: ArrayList<Transfer>, balances: HashMap<String, Float64>)
  public init(transfers: ArrayList<Transfer>, balances: HashMap<String, Float64>, paidTotals: HashMap<String, Float64>, consumedTotals: HashMap<String, Float64>)
    public mut prop transfers: ArrayList<Transfer>
    public mut prop balances: HashMap<String, Float64>  // key: 人名, value: 净余额
  public mut prop paidTotals: HashMap<String, Float64>  // key: 人名, value: 支付总额
  public mut prop consumedTotals: HashMap<String, Float64>  // key: 人名, value: 消费总额
}
```

## ✨ 新增功能（后端）

- **按权重分摊**：`Bill.weights` 可选，长度需与 `consumers` 一致；权重和 $> 0$ 时按比例分摊，否则退回均摊。
- **自动补全参与者**：即使 `participants` 未完整列出付款人/消费人，也能正常结算。
- **支付/消费统计**：`SettlementResult` 新增 `paidTotals` 与 `consumedTotals`。
- **账单备注**：`Bill.note` 可选，用于记录用途说明，不参与结算计算。

## 🔧 核心算法 API

### SettlementSolver

```cangjie
public class SettlementSolver {
    // 一站式结算：输入账单，输出结算结果
    public func settle(input: BillInput): SettlementResult
    
    // 计算净余额（一般无需单独调用）
    public func computeNetBalances(input: BillInput): HashMap<Person, Float64>
    
    // 最小化转账（一般无需单独调用）
    public func minimizeTransfers(balances: HashMap<Person, Float64>): ArrayList<Transfer>
}
```

**使用示例：**
```cangjie
let solver = SettlementSolver()
let result = solver.settle(billInput)
// result.transfers: 转账列表
// result.balances: 每人净余额
// result.paidTotals: 每人支付总额
// result.consumedTotals: 每人消费总额
```

### JsonParser

```cangjie
public class JsonParser {
    // JSON字符串 → BillInput 对象
    public static func parseBillInput(str: String): Option<BillInput>
}
```

**输入 JSON 格式：**
```json
{
  "participants": [
    {"name": "Alice"},
    {"name": "Bob"},
    {"name": "Charlie"},
    {"name": "David"},
    {"name": "Eve"}
  ],
  "bills": [
    {
      "payer": {"name": "Alice"},
      "amount": 120.5,
      "note": "晚餐聚餐",
      "consumers": [
        {"name": "Alice"},
        {"name": "Bob"},
        {"name": "Charlie"}
      ],
      "weights": [1, 1, 2]
    },
    {
      "payer": {"name": "Bob"},
      "amount": 75.0,
      "note": "电费均摊",
      "consumers": [
        {"name": "Bob"},
        {"name": "David"},
        {"name": "Eve"}
      ],
      "weights": [1, 1, 1]
    },
    {
      "payer": {"name": "Charlie"},
      "amount": 60.0,
      "note": "资料打印",
      "consumers": [
        {"name": "Alice"},
        {"name": "Charlie"},
        {"name": "Eve"}
      ],
      "weights": [2, 1, 1]
    }
  ]
}
```

## 📤 输出格式

```json
{
  "transfers": [
    {"from": {"name": "David"}, "to": {"name": "Alice"}, "amount": 12.5}
  ],
  "balances": {
    "Alice": 40.0,
    "Bob": -10.0
  },
  "paidTotals": {
    "Alice": 120.5,
    "Bob": 75.0
  },
  "consumedTotals": {
    "Alice": 60.25,
    "Bob": 35.0
  }
}
```


