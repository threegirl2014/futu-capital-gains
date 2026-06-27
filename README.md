# Futu Capital Gains 富途股票资本利得计算

> 从富途证券（Futu Securities）年度账单与资金明细中，按 FIFO 方法计算每笔卖出的资本利得，并按央行中间价换算为人民币，输出可直接用于个税申报的 Excel 报表。

> 当前仅支持股票，不支持期权。

> 由于富途官方已给到股息的数据，所以此skill只用于计算买卖的资本利得。


[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

## ✨ 特性

- **FIFO 先进先出匹配**：自动按买入时间顺序匹配卖出，支持一笔卖出拆分对应多笔买入
- **拆股自动调整**：内置 TSLA / AAPL / GOOG 等历史拆股事件，按时间顺序应用到买入队列
- **人民币换算**：使用东方财富 API 抓取 USD/HKD 对 CNY 的官方中间价
- **数据源合并**：自动合并「年度账单 Excel」+「资金明细 CSV」，覆盖富途多年历史
- **手工改动保留**：用户在 Excel 中标记 `手工改动=是` 的行不会被覆盖
- **可读性输出**：双 Sheet Excel，含原始流水 + 资本利得明细，并对拆分/手工改动行高亮

## 📋 适用场景

- 富途证券账户持有人需要计算 **境外股票投资所得个人所得税**
- 跨多个年度的买卖记录，需要按 **FIFO** 方法匹配成本
- 持仓涉及 **拆股**（如特斯拉、苹果、谷歌等历史拆股）
- 同时持有 **美股 + 港股**，需要分币种换算人民币

## 🚀 快速开始


### 输入文件准备
《年度账单》文件源：富途牛牛App或电脑版→账户→全部→年度报表
《资金明细》文件源：富途牛牛电脑版→账户→资金明细（选择账户、选择时间区间、点击导出）

将以下文件准备好（默认路径可在脚本中修改）：

```
~/富途资金/
├── 年度报表/
│   ├── 2021_年度账单_xx.xlsx
│   ├── 2022_年度账单_xx.xlsx
│   ├── 2023_年度账单_xx.xlsx
│   └── 2024_年度账单_xx.xlsx
└── 资金明细-融资融券(xx)-20100101-20251231.csv
```

### 一键执行

在 QoderWork 中直接说：

> 帮我用富途账单算一下 2024 年度的资本利得个税

QoderWork 会自动加载此 skill 并完成全部流程。


## 📊 输出格式

生成的 `计算个税.xlsx` 包含两个 Sheet：

### Sheet 1 - 证券-交易流水

合并所有数据源后的原始流水：

| 成交时间 | 代码名称 | 方向 | 数量/面值 | 价格 | 成交金额 | 来源 | 手工改动 |
|---|---|---|---|---|---|---|---|
| 2020-01-09 11:19:30 | TSLA 特斯拉 | 买入开仓 | 30 | 38.97 | 1169.10 | 年度账单 | 否 |
| 2020-08-31 09:30:00 | TSLA 特斯拉 | 买入开仓 | 150 | 7.794 | 1169.10 | CSV 估算 | 是 |

### Sheet 2 - 资本利得

FIFO 匹配后的盈亏明细（21 列）：

| 字段 | 说明 |
|---|---|
| 资本利得 | 卖出价 × 数量 − 对应买入价 × 数量 − 分摊费用（原币种） |
| 资本利得(人民币) | 按卖出年份前一年最后一个交易日的中间价换算 |
| 卖出年份 | 用于汇总申报 |
| 对应买入 | 例：`2019-12-27 09:59:10 买入12股@28.5333 [2020-08-31 5:1拆股已调整]` |
| 拆分情况 | 一笔卖出拆分对应多笔买入时标注 `拆分 i/n` |

### 颜色标记

| 颜色 | 含义 |
|---|---|
| 🟩 浅绿色 `#D9EAD3` | 用户手工修正的行 |
| 🟨 浅黄色 `#FFF2CC` | FIFO 拆分行 |

## 🧮 核心算法

### FIFO 匹配伪代码

```python
buy_queues = defaultdict(deque)

for tx in sorted(transactions, key=lambda x: x.time):
    apply_pending_splits(tx.stock, tx.time)   # ⚠️ 先应用未处理的拆股

    if tx.direction == "买入开仓":
        buy_queues[tx.stock].append({
            'remaining_qty': tx.qty,
            'buy_price': tx.price,
            'split_notes': [],
            ...
        })
    else:  # 卖出平仓
        sell_qty = tx.qty
        while sell_qty > EPSILON and buy_queues[tx.stock]:
            head = buy_queues[tx.stock][0]
            matched = min(sell_qty, head.remaining_qty)
            gain = (tx.price - head.buy_price) * matched - prorated_fee
            ...
            head.remaining_qty -= matched
            sell_qty -= matched
            if head.remaining_qty <= EPSILON:
                buy_queues[tx.stock].popleft()
```

### 拆股处理（关键）

⚠️ **不要预先调整历史买入数据**，而是在处理到拆股日期的交易时再应用：

```python
split_events = {
    'TSLA': [
        ('2020-08-31', 5, '5:1拆股'),
        ('2022-08-25', 3, '3:1拆股'),
    ],
    'AAPL': [('2020-08-31', 4, '4:1拆股')],
    'GOOG': [('2022-07-18', 20, '20:1拆股')],
}
```

### 人民币换算

使用**卖出年份前一年最后一个交易日**的中间价（与国税总局口径一致）：

| 年份末 | USD/CNY | HKD/CNY |
|---|---|---|
| 2020-12-31 | 6.5249 | 0.8416 |
| 2021-12-31 | 6.3757 | 0.8176 |
| 2022-12-30 | 6.9646 | 0.8933 |
| 2023-12-29 | 7.0827 | 0.9062 |
| 2024-12-31 | 7.1884 | 0.9260 |

数据来源：东方财富 `push2his.eastmoney.com/api/qt/stock/kline/get`

```python
from curl_cffi import requests
resp = requests.get(
    "https://push2his.eastmoney.com/api/qt/stock/kline/get",
    params={
        'secid': '120.USDCNYC',   # 港元: 120.HKDCNYC
        'fields1': 'f1,f2,f3',
        'fields2': 'f51,f52',
        'klt': '101',
        'fqt': '0',
        'beg': '20171201',
        'end': '20241231',
    },
    impersonate='chrome',         # 必须，绕过反爬
)
```

## 🗂 东方财富 secid 速查

| 市场 | 前缀 | 示例 |
|---|---|---|
| NASDAQ | `105.` | `105.TSLA` |
| NYSE | `106.` | `106.BABA` |
| AMEX | `107.` | `107.MINT` |
| OTC | `153.` | `153.TCEHY` |
| 港股 | `116.` | `116.03690`（注意补零） |
| 汇率 | `120.` | `120.USDCNYC` / `120.HKDCNYC` |

未知 secid 时可调用搜索接口：

```
https://searchapi.eastmoney.com/api/suggest/get?input={code}&type=14&count=5
```

## ⚠️ 注意事项

1. **CSV 缺数量列**：富途资金明细 CSV 只有总金额，需通过当日前复权收盘价反推股数（`round(金额/价格)`），后续可手工修正
2. **费用分摊**：一笔卖出拆分对应多笔买入时，总费用按 `matched_qty / original_qty` 比例分摊
3. **浮点精度**：使用 `EPSILON = 1e-8` 判断队列残余，避免出现 `2.29e-16 股` 的伪精度
4. **拆股按时间应用**：用户手工填的 CSV 数据是交易时实际值（未拆股），切勿一次性预调整
5. **数据源限制**：yfinance / Yahoo / Stooq 在中国大陆环境均被封锁（403 或 JS Challenge），统一使用东方财富
6. **港股代码补零**：如美团 `3690` → 东方财富 secid `116.03690`

## 📁 文件结构

```
futu-capital-gains/
├── SKILL.md              # Skill 入口（QoderWork 自动加载）
└── README.md             # 你正在看的这份文档
```

## 🔍 验证清单

完成后请检查：

- [ ] "无对应买入记录" 的卖出笔数 ≤ 早期未导入的持仓数
- [ ] 拆股后数量与价格关系正确（TSLA 5:1 后数量 ×5、价格 ÷5）
- [ ] 年度汇总 = 分笔人民币利得之和
- [ ] 手工改动行被正确保留（绿色背景）

## 🤝 贡献

如需添加新的拆股事件、新的 secid 映射，请直接编辑 `SKILL.md` 中对应的字典。

## 📜 License

MIT License — 仅供个人税务申报参考，**不构成税务或投资建议**。如有疑问请咨询专业税务师。

---
