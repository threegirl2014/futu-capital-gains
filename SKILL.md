---
name: futu-capital-gains
description: 从富途证券年度账单和资金明细计算资本利得（个税）。使用 FIFO 先进先出法匹配买卖，处理股票拆股，转换人民币汇率。当用户提到"富途"、"资本利得"、"个税计算"、"FIFO"、"年度账单"时使用。
version: 1.0.0
---

# 富途证券资本利得计算

## 概述

从富途证券（Futu Securities）的年度账单 Excel 和资金明细 CSV 中，提取买卖交易记录，使用 FIFO（先进先出）方法计算每笔卖出的资本利得，并按人民币汇率中间价转换为人民币金额。

## 输入数据源

### 1. 年度账单 Excel（主要来源）
- 文件格式：`{year}_年度账单_{account}.xlsx`
- 使用 Sheet：`证券-交易流水`
- 列：成交时间, 账户名称, 账户号码, 品类, 代码名称, 交易所/市场, 方向, 交收日期, 币种, 数量/面值, 价格, 成交金额, 总费用, 变动金额
- 筛选条件：方向 = "卖出平仓" 或 "买入开仓"

### 2. 资金明细 CSV（补充早期数据）
- 文件格式：`资金明细-{账户类型}({账号})-{日期}-{时间}.csv`
- 列：交易类型, 代码, 名称, 金额, 余额, 币种, 创建时间
- 筛选条件：交易类型 in (股票买入成交, 股票卖出成交, 股票买入, 股票卖出, ETFs买入成交, ETFs卖出成交)
- 注意：CSV 只有总金额，没有数量和单价，需要通过历史价格估算

### 3. 用户手工改动
- 用户可在生成的 Excel 中修正数据（标记"手工改动"列为"是"）
- 重新计算时需读取修改后的数据

## 核心算法

### FIFO 匹配规则
1. 按成交时间排序所有交易
2. 每只股票维护一个买入队列（deque）
3. 卖出时从队列头部匹配（先买的先卖）
4. 一笔卖出可能对应多笔买入 → 拆分卖出行，标记"拆分 i/n"
5. 浮点精度处理：EPSILON = 1e-8，忽略极小残余数量

### 股票拆股处理（关键）
拆股必须按时间顺序应用到买入队列：

```python
split_events = {
    'TSLA': [
        ('2020-08-31', 5, '2020-08-31 5:1拆股'),
        ('2022-08-25', 3, '2022-08-25 3:1拆股'),
    ],
    'AAPL': [
        ('2020-08-31', 4, '2020-08-31 4:1拆股'),
    ],
    'GOOG': [
        ('2022-07-18', 20, '2022-07-18 20:1拆股'),
    ],
}

def apply_pending_splits(stock, current_date):
    """在处理每笔交易前，检查是否有未应用的拆股事件"""
    if stock not in split_events:
        return
    for split_date_str, ratio, note in split_events[stock]:
        split_date = pd.Timestamp(split_date_str)
        if current_date >= split_date and split_date_str not in splits_applied[stock]:
            for entry in buy_queues[stock]:
                entry['remaining_qty'] *= ratio
                entry['buy_price'] = round(entry['buy_price'] / ratio, 6)
                entry['split_notes'].append(note)
            splits_applied[stock].add(split_date_str)
```

关键原则：
- 用户手工修正的 CSV 数据是交易时实际值（未拆股）
- 年度账单中的数据也是交易时实际值
- 拆股在处理到拆股日期时才应用到队列中的历史买入记录
- "对应买入"列需标注拆股调整信息

### 人民币转换
- 使用**卖出年份前一年最后一个交易日**的人民币汇率中间价
- 数据源：东方财富 API（Eastmoney），secid=120.USDCNYC（美元）或 120.HKDCNYC（港元）
- API: `push2his.eastmoney.com/api/qt/stock/kline/get`
- 需要 curl_cffi 绕过反爬

```python
# 获取汇率示例
from curl_cffi import requests
url = "https://push2his.eastmoney.com/api/qt/stock/kline/get"
params = {
    'secid': '120.USDCNYC',  # 或 120.HKDCNYC
    'fields1': 'f1,f2,f3',
    'fields2': 'f51,f52',
    'klt': '101',  # 日K
    'fqt': '0',
    'beg': '20171201',
    'end': '20241231',
}
resp = requests.get(url, params=params, impersonate='chrome')
```

已知汇率参考值（卖出年份前一年最后一天）：
- 2017-12-29 USD: 6.5342, HKD: 0.8359
- 2018-12-28 USD: 6.8632, HKD: 0.8762
- 2019-12-31 USD: 6.9762, HKD: 0.8958
- 2020-12-31 USD: 6.5249, HKD: 0.8416
- 2021-12-31 USD: 6.3757, HKD: 0.8176
- 2022-12-30 USD: 6.9646, HKD: 0.8933
- 2023-12-29 USD: 7.0827, HKD: 0.9062
- 2024-12-31 USD: 7.1884, HKD: 0.9260

### CSV 数据估算股数
CSV 只有金额没有数量，需要：
1. 通过东方财富 API 获取该股票当天的前复权收盘价
2. 用 `金额 / 价格` 估算股数，四舍五入到整数
3. 反算单价 = `金额 / 股数`
4. 用户后续可手工修正

东方财富 secid 格式：
- 美股 NASDAQ: 105.{code}
- 美股 NYSE: 106.{code}
- 美股 AMEX: 107.{code}
- 美股 OTC: 153.{code}（如 TCEHY）
- 港股: 116.{code}（如 116.03690）
- 人民币汇率: 120.{code}

搜索 API（查找正确 secid）：
```
https://searchapi.eastmoney.com/api/suggest/get?input={code}&type=14&count=5
```

## 输出格式

### Sheet 1: 证券-交易流水
原始合并数据，包含：
- 14 列原始字段 + "来源"列 + "手工改动"列
- 按成交时间排序
- 手工改动行标绿色背景

### Sheet 2: 资本利得
FIFO 计算结果，21 列：
成交时间, 账户名称, 账户号码, 品类, 代码名称, 交易所/市场, 方向, 交收日期, 币种, 数量/面值, 价格, 成交金额, 总费用, 变动金额, 来源, 手工改动, 资本利得, 资本利得(人民币), 卖出年份, 对应买入, 拆分情况

样式：
- 手工改动行：浅绿色 (#D9EAD3)
- 拆分行：浅黄色 (#FFF2CC)
- 表头：加粗居中

"对应买入"列格式：
- 正常：`2020-01-01 14:19:30 买入10股@20.22`
- 含拆股：`2019-10-27 19:59:10 买入1股@280.5333 [2020-01-31 5:1拆股,2022-02-25 3:1拆股已调整]`
- 无匹配：`无对应买入记录(更早期持仓)`

## 依赖

```
pip install pandas openpyxl curl_cffi
```

## 步骤

1. 读取用户当前的"计算个税.xlsx"中的"证券-交易流水" Sheet（含手工改动）
2. 如果是首次生成，则合并年度账单 + CSV 资金明细
3. 获取所需汇率（如本地无缓存）
4. 按成交时间排序所有交易
5. 逐笔处理：买入入队，卖出 FIFO 匹配（处理前先 apply 拆股）
6. 计算资本利得 = 卖出金额 - 买入金额 - 按比例分摊的费用
7. 转换人民币：利得 × 汇率（卖出年份前一年年末）
8. 生成 Excel 两个 Sheet，保留格式和颜色标记
9. 输出年度汇总

## Pitfalls

- CSV 金额可能包含费用（变动金额 vs 成交金额），要区分
- 浮点精度问题：matched_qty 接近 0 时用 EPSILON 判断，避免出现 2.29e-16 股的残余
- 拆股必须按时间顺序应用，不能预先调整所有历史数据
- 东方财富 API 可能被封，必须用 curl_cffi 的 impersonate='chrome'
- yfinance/Yahoo/Stooq 都已被封（403或JS挑战），不要尝试
- 部分股票 secid 特殊（TCEHY=153, MINT=107），需先搜索确认
- 费用按比例分摊：一笔卖出拆分时，总费用按 matched_qty / original_sell_qty 分配
- 港股代码在东方财富需要补零：3690 → 03690

## Verification

1. 检查"无对应买入记录"的卖出笔数是否合理（应尽可能少）
2. 验证拆股后的数量和价格关系（如 TSLA 5:1 后数量×5、价格÷5）
3. 对比年度汇总的人民币总额与分笔之和
4. 确认手工改动行被正确保留和标记
