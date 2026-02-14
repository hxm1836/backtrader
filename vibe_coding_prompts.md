# MiniTrader - 精简版回测框架 Vibe Coding 提示词全集

> 目标：通过 10 轮对话，用 vibe coding 构建一个功能完整的量化回测框架
> 预计代码量：5,000-8,000 行
> 技术栈：Python 3.10+, pandas, numpy, matplotlib

---

## 架构总览

```
minitrader/
├── __init__.py          # 包入口，暴露公共 API
├── cerebro.py           # 回测引擎
├── strategy.py          # 策略基类
├── broker.py            # 模拟经纪商（订单撮合、资金管理）
├── feed.py              # 数据源（CSV/pandas）
├── order.py             # 订单模型
├── position.py          # 持仓模型
├── indicator.py         # 指标基类
├── indicators/          # 内置指标
│   ├── __init__.py
│   ├── sma.py           # 简单移动平均
│   ├── ema.py           # 指数移动平均
│   ├── rsi.py           # RSI
│   ├── macd.py          # MACD
│   ├── bollinger.py     # 布林带
│   ├── atr.py           # ATR
│   ├── crossover.py     # 交叉信号
│   └── stochastic.py    # KDJ/随机指标
├── analyzers/           # 绩效分析
│   ├── __init__.py
│   ├── returns.py       # 收益率
│   ├── sharpe.py        # 夏普比率
│   ├── drawdown.py      # 最大回撤
│   └── trade_analyzer.py# 交易统计
├── plot.py              # 可视化
├── sizer.py             # 仓位管理
└── utils.py             # 工具函数
```

---

## 第 1 轮：项目初始化 + 数据层

### Prompt 1

```
我要用 Python 构建一个名为 MiniTrader 的量化交易回测框架。请帮我完成项目初始化和数据层。

要求：

1. 创建项目结构：
   minitrader/
   ├── __init__.py
   ├── feed.py
   └── utils.py

2. feed.py - 数据源模块：
   - 定义 DataFeed 基类，内部用 pandas DataFrame 存储 OHLCV 数据
   - DataFrame 必须包含列：datetime, open, high, low, close, volume
   - 提供属性访问：feed.open, feed.high, feed.low, feed.close, feed.volume, feed.datetime
   - 每个属性返回的是一个 LineSeries 对象（类似数组，支持通过 [0] 获取当前值，[-1] 获取前一个值）
   - 实现 CSVFeed 子类：从 CSV 文件加载数据，支持指定日期格式、日期列名
   - 实现 PandasFeed 子类：直接从 pandas DataFrame 创建
   - DataFeed 需要支持迭代：有一个内部指针 _idx，每次调用 advance() 向前推进一步

3. utils.py - 工具模块：
   - LineSeries 类：包装一个 numpy 数组和一个指针 _idx
     - __getitem__(0) 返回当前位置的值
     - __getitem__(-1) 返回前一个位置的值
     - __getitem__(n) 当 n > 0 时，抛出异常（不能看到未来数据）
     - 支持 > < == != 等比较操作，返回布尔值（比较当前值）
     - 支持 len() 返回当前已经可见的数据长度

4. __init__.py：
   - 导出 CSVFeed, PandasFeed

请生成完整代码，包含类型注解和必要的 docstring。
```

---

## 第 2 轮：订单与持仓模型

### Prompt 2

```
继续开发 MiniTrader 框架。现在实现订单和持仓模型。

要求：

1. order.py - 订单模块：
   - Order 类，包含以下属性：
     - order_type: 枚举，支持 MARKET（市价单）, LIMIT（限价单）, STOP（止损单）
     - direction: 枚举，BUY 或 SELL
     - size: 下单数量（股数）
     - price: 限价/止损价格（市价单为 None）
     - status: 枚举，CREATED -> SUBMITTED -> ACCEPTED -> COMPLETED / CANCELED / REJECTED
     - executed_price: 实际成交价
     - executed_size: 实际成交量
     - executed_dt: 成交时间
     - commission: 佣金
     - ref: 唯一订单编号（自增整数）
   - Order 提供方法：
     - execute(price, size, dt, commission) - 标记订单为已成交
     - cancel() - 取消订单
     - isbuy() / issell() - 判断方向
     - alive() - 判断订单是否还未完结

2. position.py - 持仓模块：
   - Position 类：
     - size: 当前持仓数量（正数=多头，负数=空头，0=空仓）
     - price: 持仓均价
     - update(size, price) - 更新持仓（买入加仓、卖出减仓）
       - 同方向加仓：计算加权平均价
       - 反方向减仓：计算已实现盈亏
     - 返回已实现盈亏 (realized PnL)

确保所有枚举使用 Python 的 enum 模块。代码要包含类型注解。
```

---

## 第 3 轮：模拟经纪商

### Prompt 3

```
继续开发 MiniTrader。现在实现模拟经纪商。

要求：

1. broker.py - 模拟经纪商：
   - Broker 类：
     - 初始化参数：cash（初始资金），commission（佣金比率，默认 0.001 即千分之一）
     - 属性：
       - cash: 当前可用现金
       - value: 当前总资产（现金 + 所有持仓市值）
       - positions: dict[data_name -> Position] 多个标的各自的持仓
     - 核心方法：
       - submit_order(order) -> Order：提交订单
       - execute_pending_orders(current_data, dt)：在每个 bar 尝试撮合挂单
         - 市价单：以当前 bar 的开盘价成交
         - 限价买单：当前 bar 最低价 <= 限价时，以限价成交
         - 限价卖单：当前 bar 最高价 >= 限价时，以限价成交
         - 止损买单：当前 bar 最高价 >= 止损价时，以止损价成交
         - 止损卖单：当前 bar 最低价 <= 止损价时，以止损价成交
       - get_value(current_prices: dict) -> float：计算当前总资产
     - 资金检查：
       - 买入前检查现金是否足够（price * size * (1 + commission)）
       - 资金不足时拒绝订单（设为 REJECTED）
     - 成交后处理：
       - 扣减/增加现金
       - 更新对应标的的 Position
       - 计算并扣除佣金

2. 在 __init__.py 中导出 Broker

代码逻辑要严谨，特别是资金计算和持仓更新的部分。
```

---

## 第 4 轮：策略基类

### Prompt 4

```
继续开发 MiniTrader。现在实现策略基类。

要求：

1. strategy.py - 策略基类：
   - Strategy 类：
     - params 类属性：一个字典，定义策略的可配置参数及其默认值
       - 子类可以覆盖 params 来定义自己的参数
       - 初始化时将 params 中的键值对设为实例属性，方便用 self.p.xxx 访问
     - Params 内部类：将字典转为可用 . 访问的对象
     - 初始化时接收：
       - datas: List[DataFeed]，数据源列表
       - broker: Broker 实例
       - 用户自定义参数（覆盖 params 默认值）
     - 数据访问快捷方式：
       - self.data 或 self.data0 = datas[0]
       - self.data1 = datas[1]（如果有的话）
       - self.datas = 完整列表
     - 核心生命周期方法（子类覆写）：
       - __init__(): 初始化指标
       - next(): 每个 bar 调用，编写交易逻辑
       - start(): 回测开始时调用
       - stop(): 回测结束时调用
       - notify_order(order): 订单状态变更时回调
       - notify_trade(trade): 交易完成时回调
     - 交易快捷方法：
       - buy(size=None, price=None, exectype=Order.MARKET, data=None) -> Order
       - sell(size=None, price=None, exectype=Order.MARKET, data=None) -> Order
       - close(data=None) -> Order：平掉指定标的的全部持仓
     - 属性：
       - self.position: 主数据的当前持仓
       - self.broker: 经纪商引用
     - 如果 size 为 None，使用 sizer 来决定下单量（默认下1股）

2. sizer.py - 仓位管理：
   - Sizer 基类：
     - getsizing(strategy, data, isbuy) -> int
   - FixedSizer：固定手数
   - PercentSizer：按资金百分比计算

在 __init__.py 中导出 Strategy, FixedSizer, PercentSizer
```

---

## 第 5 轮：回测引擎 Cerebro

### Prompt 5

```
继续开发 MiniTrader。现在实现核心回测引擎。

要求：

1. cerebro.py - 回测引擎：
   - Cerebro 类：
     - adddata(data: DataFeed, name: str = None)：添加数据源
     - addstrategy(strategy_cls: type, **kwargs)：添加策略类及其参数
     - addsizer(sizer_cls: type, **kwargs)：设置仓位管理器
     - addanalyzer(analyzer_cls: type, **kwargs)：添加分析器
     - broker 属性：
       - cerebro.broker.setcash(amount)
       - cerebro.broker.setcommission(commission)
       - 也可以直接访问 cerebro.broker
     - run() 方法 - 核心回测循环：
       1. 创建 Broker 实例
       2. 实例化 Strategy（传入 datas 和 broker）
       3. 调用 strategy.start()
       4. 对所有数据做预加载（全部数据加载到 DataFrame 中）
       5. 计算所有指标的最小预热期（warmup period）
       6. 主循环：从第一个有效 bar 开始，逐 bar 推进
          a. 推进所有 DataFeed 的指针
          b. 更新所有指标的值
          c. 执行 broker 的挂单撮合 (execute_pending_orders)
          d. 如果已过预热期，调用 strategy.next()
          e. 记录每个 bar 的资产净值（用于后续分析）
       7. 调用 strategy.stop()
       8. 运行所有 analyzer
       9. 返回策略实例列表
     - plot(**kwargs) 方法：调用 plot 模块绘图（后面实现）

   - 关键设计：
     - 预热期 = 所有指标中最大的 period 值
     - 多个数据源时，以最短的数据为准，保证日期对齐
     - 每个 bar 的处理顺序：数据推进 -> 指标更新 -> 订单撮合 -> 策略逻辑

2. 更新 __init__.py：
   - 导出 Cerebro
   - 让用户可以写 import minitrader as mt; cerebro = mt.Cerebro()

确保 run() 的循环逻辑正确，特别注意指针同步和预热期处理。
```

---

## 第 6 轮：指标体系

### Prompt 6

```
继续开发 MiniTrader。现在实现指标体系。

要求：

1. indicator.py - 指标基类：
   - Indicator 基类：
     - 接收数据源（DataFeed 或另一个 Indicator 的输出 LineSeries）
     - period 属性：指标的计算周期
     - lines：指标的输出线，是一个字典 {name: numpy_array}
     - 提供 __getitem__ 让 indicator[0] 返回主线的当前值
     - calculate() 方法：由子类实现，对全部数据预计算
     - 支持指标嵌套：一个指标的输出可以作为另一个指标的输入

2. indicators/ 目录下实现 8 个指标：

   a. SMA (简单移动平均)：
      - 输入：data (LineSeries), period (int)
      - 输出：sma 线
      - 公式：过去 period 个值的算术平均

   b. EMA (指数移动平均)：
      - 输入：data, period
      - 输出：ema 线
      - 公式：EMA_t = α * price_t + (1-α) * EMA_(t-1)，α = 2/(period+1)

   c. RSI (相对强弱指标)：
      - 输入：data, period (默认14)
      - 输出：rsi 线
      - 公式：先算每日涨跌，分别求平均上涨和平均下跌（用EMA平滑），RSI = 100 - 100/(1+RS)

   d. MACD：
      - 输入：data, fast_period=12, slow_period=26, signal_period=9
      - 输出：macd 线, signal 线, histogram 线
      - 公式：MACD = EMA(fast) - EMA(slow), Signal = EMA(MACD, signal_period), Hist = MACD - Signal

   e. BollingerBands (布林带)：
      - 输入：data, period=20, devfactor=2.0
      - 输出：mid 线 (SMA), top 线, bot 线
      - 公式：mid = SMA, top = mid + devfactor * std, bot = mid - devfactor * std

   f. ATR (真实波动幅度均值)：
      - 输入：data (需要 high, low, close), period=14
      - 输出：atr 线
      - 公式：TR = max(H-L, |H-prevC|, |L-prevC|), ATR = SMA(TR, period)

   g. CrossOver (交叉信号)：
      - 输入：line1, line2 (两个 LineSeries)
      - 输出：cross 线（1 = 金叉, -1 = 死叉, 0 = 无交叉）

   h. Stochastic (随机指标/KDJ)：
      - 输入：data (需要 high, low, close), k_period=14, d_period=3
      - 输出：k 线, d 线
      - 公式：%K = (C - LL) / (HH - LL) * 100, %D = SMA(%K, d_period)

3. indicators/__init__.py 导出所有指标，让用户可以：
   - mt.ind.SMA(data, period=20)
   - mt.ind.MACD(data)

4. 同时在主 __init__.py 中添加 ind = indicators 别名
```

---

## 第 7 轮：绩效分析器

### Prompt 7

```
继续开发 MiniTrader。现在实现绩效分析模块。

要求：

1. analyzers/ 目录：

   a. returns.py - 收益分析：
      - ReturnsAnalyzer 类：
        - 计算总收益率
        - 计算年化收益率
        - 计算每日收益率序列
        - get_analysis() 返回字典：
          {
            "total_return": 0.25,         # 总收益 25%
            "annual_return": 0.18,        # 年化收益 18%
            "daily_returns": [...]        # 日收益率列表
          }

   b. sharpe.py - 夏普比率：
      - SharpeAnalyzer 类：
        - 参数：risk_free_rate (年化无风险利率，默认 0.02)
        - 公式：Sharpe = (年化收益 - 无风险利率) / 年化波动率
        - get_analysis() 返回：{"sharpe_ratio": 1.5}

   c. drawdown.py - 回撤分析：
      - DrawdownAnalyzer 类：
        - 计算最大回撤比例
        - 计算最大回撤持续天数
        - 计算回撤序列
        - get_analysis() 返回：
          {
            "max_drawdown": 0.15,           # 最大回撤 15%
            "max_drawdown_duration": 45,     # 最长回撤持续天数
            "drawdown_series": [...]         # 回撤序列
          }

   d. trade_analyzer.py - 交易统计：
      - TradeAnalyzer 类：
        - 统计所有已完成交易（从 strategy 的交易记录中获取）
        - get_analysis() 返回：
          {
            "total_trades": 50,
            "won": 30,
            "lost": 20,
            "win_rate": 0.6,
            "avg_profit": 150.0,
            "avg_loss": -100.0,
            "profit_factor": 2.25,   # 总盈利 / 总亏损
            "largest_win": 500.0,
            "largest_loss": -300.0,
            "avg_trade_duration": 5.2  # 平均持仓天数
          }

2. 所有 Analyzer 继承自一个基类 Analyzer：
   - 接收 strategy 实例和 equity_curve (净值列表)
   - 抽象方法 run() 和 get_analysis()
   - 提供 print_analysis() 方法，格式化打印分析结果

3. analyzers/__init__.py 导出所有分析器
4. 更新 __init__.py
```

---

## 第 8 轮：可视化

### Prompt 8

```
继续开发 MiniTrader。现在实现可视化模块。

要求：

1. plot.py - 绘图模块：
   - MiniPlot 类：
     - 接收策略运行结果，绘制完整的回测报告图
     - 使用 matplotlib，图表分为多个子图（subplots），共享 x 轴（日期）

   - 主图（最大，占 60% 高度）：
     - K线图（蜡烛图）或收盘价折线图（通过参数切换）
     - 叠加显示所有在策略中使用的均线类指标（SMA, EMA, BollingerBands）
     - 买入信号：绿色上三角标记
     - 卖出信号：红色下三角标记

   - 副图1 - 成交量（占 10% 高度）：
     - 柱状图显示成交量
     - 上涨日绿色，下跌日红色

   - 副图2 - 独立指标区（占 15% 高度）：
     - 显示 RSI、Stochastic 等有独立坐标的指标
     - RSI 加上 70/30 水平参考线

   - 副图3 - 净值曲线（占 15% 高度）：
     - 显示策略资产净值变化
     - 用填充区域表示回撤

   - 样式要求：
     - 使用暗色主题（背景 #1a1a2e，前景白色）
     - 网格线用半透明灰色
     - 整体美观专业
     - 日期 x 轴自动调整密度，不重叠

   - plot(savefig=None) 方法：显示图表或保存为文件

2. 在 Cerebro 的 plot() 方法中调用 MiniPlot

最终效果应该类似 TradingView 的多面板图表风格。
```

---

## 第 9 轮：集成测试 + 示例策略

### Prompt 9

```
继续开发 MiniTrader。现在编写测试和示例。

要求：

1. 创建 tests/ 目录，编写以下测试：

   a. test_feed.py：
      - 测试 CSVFeed 能正确加载 CSV 文件
      - 测试 PandasFeed 能正确从 DataFrame 创建
      - 测试 LineSeries 的 [0], [-1] 索引行为
      - 测试 LineSeries 禁止访问未来数据 [1] 会抛异常

   b. test_indicators.py：
      - 对每个指标用已知数据验证计算结果
      - 例如：对 [1,2,3,4,5] 计算 SMA(3) 应得 [NaN, NaN, 2, 3, 4]
      - RSI, MACD 等用手动计算的结果验证

   c. test_broker.py：
      - 测试市价单成交
      - 测试限价单在价格满足时成交
      - 测试限价单在价格不满足时不成交
      - 测试资金不足时订单被拒绝
      - 测试佣金计算正确

   d. test_strategy.py：
      - 用一个简单的 SMA 交叉策略做端到端测试
      - 验证买卖信号在正确的位置触发
      - 验证最终资金合理（有盈利或合理亏损）

2. 创建 examples/ 目录，包含 3 个示例：

   a. examples/sma_cross.py - 双均线交叉策略：
      - 短期 SMA(10) 上穿长期 SMA(30) 买入
      - 短期下穿长期卖出
      - 使用示例 CSV 数据

   b. examples/rsi_strategy.py - RSI 反转策略：
      - RSI < 30 买入
      - RSI > 70 卖出

   c. examples/multi_indicator.py - 多指标组合策略：
      - 同时使用 MACD + RSI + BollingerBands
      - MACD 金叉 + RSI < 50 + 价格在布林带下轨附近 → 买入

3. 创建一个示例 CSV 数据文件 examples/sample_data.csv：
   - 包含至少 200 行日线数据
   - 列：Date,Open,High,Low,Close,Volume
   - 数据要有明显的趋势和震荡，便于验证策略
   - 可以用正弦波 + 趋势 + 随机噪声生成

4. 确保所有测试可以通过 pytest 运行
```

---

## 第 10 轮：完善与打包

### Prompt 10

```
最后一轮！完善 MiniTrader 框架，使其可以直接使用。

要求：

1. 完善 __init__.py，确保以下用法可行：
   ```python
   import minitrader as mt

   cerebro = mt.Cerebro()
   cerebro.adddata(mt.CSVFeed('data.csv'))
   cerebro.broker.setcash(100000)
   cerebro.broker.setcommission(0.001)
   cerebro.addstrategy(MyStrategy, fast_period=10, slow_period=30)
   cerebro.addsizer(mt.PercentSizer, percent=95)
   cerebro.addanalyzer(mt.analyzers.SharpeAnalyzer)
   cerebro.addanalyzer(mt.analyzers.DrawdownAnalyzer)
   results = cerebro.run()
   cerebro.plot()
   ```

2. 添加参数优化功能到 Cerebro：
   - cerebro.optstrategy(StrategyClass, fast_period=range(5,20), slow_period=range(20,40))
   - 对所有参数组合做网格搜索
   - 返回每组参数的最终资金，按收益排序
   - 打印 top 10 参数组合

3. 创建 setup.py 或 pyproject.toml：
   - 包名：minitrader
   - 版本：0.1.0
   - 依赖：pandas, numpy, matplotlib
   - 支持 pip install -e .

4. 检查并修复所有模块之间的导入关系，确保没有循环依赖

5. 给每个公开类和方法添加简洁的 docstring

6. 在项目根目录添加一个 README.md：
   - 项目简介
   - 安装方法
   - 快速上手示例（双均线策略，10行代码）
   - 支持的指标列表
   - 支持的分析器列表
```

---

## 补充轮次（可选）

### Prompt A：添加更多指标

```
给 MiniTrader 添加以下指标：
1. WMA (加权移动平均)
2. VWAP (成交量加权平均价)
3. OBV (能量潮指标)
4. CCI (商品通道指标)
5. ADX (平均趋向指标)
6. ParabolicSAR (抛物线转向)

每个指标要有正确的数学公式实现和单元测试。
```

### Prompt B：添加多数据源支持

```
增强 MiniTrader 的多数据源功能：
1. 支持同时回测多只股票
2. 在策略中通过 self.data0, self.data1 访问不同标的
3. 不同标的的日期自动对齐（跳过非共同交易日）
4. 支持针对不同标的分别下单：self.buy(data=self.data1, size=100)
5. 编写一个配对交易示例策略
```

### Prompt C：添加实时数据源

```
给 MiniTrader 添加实时数据支持：
1. 实现 YahooFeed：通过 yfinance 库拉取雅虎财经的历史数据
2. 实现 AkShareFeed：通过 akshare 库拉取 A 股数据
3. 自动处理日期范围、数据格式转换
4. 编写使用示例
```

### Prompt D：命令行界面

```
给 MiniTrader 添加 CLI 支持：
1. 使用 argparse 或 click
2. 命令：minitrader run --strategy sma_cross --data AAPL.csv --cash 100000
3. 命令：minitrader optimize --strategy sma_cross --data AAPL.csv --param fast=5:20 --param slow=20:40
4. 命令：minitrader plot --strategy sma_cross --data AAPL.csv --output chart.png
5. 策略通过文件路径指定，自动加载
```

---

## 使用建议

1. **严格按顺序执行**：每一轮都依赖前面的代码，不要跳步
2. **每轮结束后测试**：在进入下一轮前，确保当前代码能运行
3. **遇到 bug 直接粘贴报错**：把完整的错误信息给 AI，让它修复
4. **第 6 轮（指标）最容易出错**：仔细验证数学计算，可以用 Excel 或手算对照
5. **保持上下文**：如果对话太长丢失上下文，把当前所有文件的代码重新喂给 AI

## 预期成果

完成后你将拥有一个：
- 支持 CSV/Pandas 数据源的回测框架
- 8 个常用技术指标
- 市价/限价/止损三种订单类型
- 4 个绩效分析器（收益、夏普、回撤、交易统计）
- 专业级图表输出
- 参数优化功能
- 完整的测试套件和示例策略

这个框架虽然不及 backtrader 的 55,000 行代码和 122 个指标，
但核心功能完整，足以用于学习量化交易和验证策略思路。
