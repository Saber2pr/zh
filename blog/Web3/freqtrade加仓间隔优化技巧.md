在 **Freqtrade**  策略框架中，仓位调整（`adjust_trade_position`）是提高策略盈利能力和风险管理的核心要素之一。合理的仓位调整能够在市场有利时增加仓位以放大盈利，而在市场不利时减少仓位以控制风险。然而，过于频繁或不适时的仓位调整，特别是在蜡烛周期的初期，可能会带来不必要的损失。

因此，如何优化仓位调整的时机，成为了策略设计中的一个关键点。本文将重点介绍如何利用 `fresh_candle`  来优化仓位调整，并阐述其与 `process_only_new_candles` 之间的区别，帮助交易者更好地理解这两者在策略中的作用及其优化效果。

### 1. `adjust_trade_position` 简介 

`adjust_trade_position` 是 Freqtrade 框架中用于动态调整仓位的函数。策略通过该函数决定是否增加或减少仓位。仓位调整的基本逻辑包括： 

- **增加仓位** ：当市场走势对我们有利时，通过加仓来放大盈利。
- **减少仓位** ：当市场走势不利时，通过减仓来降低风险。

然而，在蜡烛周期的初期，市场波动通常较大，频繁的仓位调整可能会导致误判。因此，避免在蜡烛周期刚开始时做出仓位调整，是优化策略的重要环节。

### 2. `fresh_candle`：避免在蜡烛周期初期过早调整仓位

`fresh_candle` 是一种自定义的机制，旨在通过判断交易是否进入了新的蜡烛周期，从而控制是否执行仓位调整。其基本思路是，计算当前交易的持续时间，若持续时间小于设定的最小调整时间（例如 5 分钟），则认为当前交易属于蜡烛周期的初期，跳过仓位调整。

`fresh_candle` 方法实现: `fresh_candle` 通过计算当前时间与交易开盘时间的差值，来判断是否进入了新的一根蜡烛周期。如果交易的持续时间小于设定的 `adjustment_time_pending_minutes`（例如 5 分钟），则跳过仓位调整，避免在蜡烛周期的初期进行仓位调整。

简易代码如下：

```python
class MyStrategy(IStrategy):

    adjustment_time_pending_minutes = 5  # 设置调整时间为 5 分钟

    def is_fresh_candle(self, trade, current_time):
        # 计算交易的持续时间
        trade_duration = (current_time - trade.open_date_utc).seconds / 60 
        return trade_duration < self.adjustment_time_pending_minutes  # 小于 5 分钟为新蜡烛

    def adjust_trade_position(self, trade, current_time):
        # 判断是否为新蜡烛，如果是，则跳过仓位调整
        if self.is_fresh_candle(trade, current_time):
            return None  # 如果是新蜡烛周期，跳过仓位调整

        # 如果不是新蜡烛周期，执行仓位调整逻辑
        current_profit = self.get_profit(trade)
        if current_profit > 0.05:
            self.buy(...)  # 增加仓位
        elif current_profit < -0.05:
            self.sell(...)  # 减少仓位
```

`fresh_candle` 的作用

- **避免过早调整仓位** ：在蜡烛周期的初期（如开盘后几分钟），市场波动较大，过早进行仓位调整可能导致错误的决策。`fresh_candle` 通过判断交易的持续时间，确保仓位调整在蜡烛周期有足够时间演变之后才执行。
 
- **提高策略稳定性** ：避免频繁地在市场波动剧烈时调整仓位，能够提升策略的稳定性，并减少因过度交易带来的风险。
 
- **优化执行时机** ：通过 `fresh_candle`，策略能够有效控制在蜡烛周期的初期阶段跳过仓位调整，从而为后续的市场走势提供更多时间来确认方向，避免在市场未明确时做出仓位调整。

### 3. `process_only_new_candles`：控制蜡烛数据的处理方式

`process_only_new_candles` 是 Freqtrade 框架内置的参数，它决定了策略在处理蜡烛数据时是否仅仅依赖最新的蜡烛数据。该参数与 `fresh_candle` 的作用不同，但都涉及到控制策略在蜡烛周期内的行为。

- **功能** ：`process_only_new_candles` 控制策略是否在每个新蜡烛到来时，仅处理最新生成的蜡烛数据。设置为 `True` 时，策略只会处理最新的蜡烛数据，而不会依赖之前的蜡烛数据进行计算。这有助于简化策略，避免处理不必要的历史数据。
 
- **适用场景** ：适用于那些不依赖历史数据的策略，特别是短期交易策略，或者需要更高效计算的高频策略。

#### `process_only_new_candles` 与 `fresh_candle` 的区别

| 功能 | process_only_new_candles | fresh_candle | 
| --- | --- | --- | 
| 作用描述 | 控制策略是否仅处理每个蜡烛周期的最新蜡烛数据，避免重复计算。 | 判断交易是否进入新的蜡烛周期（即是否超过设定的最小调整时间），控制是否进行仓位调整。 | 
| 控制粒度 | 影响 populate_indicators 方法，决定是否使用最新的蜡烛数据计算技术指标。 | 影响 adjust_trade_position 方法，控制是否在蜡烛周期初期进行仓位调整。 | 
| 是否影响策略执行 | 是的，process_only_new_candles 影响整个策略的数据处理，确保策略只基于最新蜡烛数据执行。 | 影响仓位调整的时机，但不会直接影响策略的整体执行。 | 
| 使用场景 | 用于高频或短期交易策略，只依赖最新蜡烛数据，无需依赖历史数据。 | 用于避免在蜡烛周期初期频繁调整仓位，确保仓位调整发生在蜡烛周期有足够时间演变之后。 | 
| 优点 | 提高策略运行效率，避免重复计算和不必要的数据处理。 | 避免过早的仓位调整，减少由于市场波动剧烈时错误判断导致的损失。 | 


### 4. 总结：如何选择 `fresh_candle` 和 `process_only_new_candles`

- `fresh_candle` ：主要用于控制仓位调整的时机。它通过判断当前交易是否处于蜡烛周期的初期（持续时间小于设定的最小调整时间）来决定是否跳过仓位调整。`fresh_candle` 适用于避免在蜡烛周期刚开始时频繁调整仓位，从而减少误判和过度交易的风险。
 
- `process_only_new_candles` ：影响策略的数据处理方式，确保策略仅使用最新蜡烛数据进行技术指标计算。适用于高频或短期策略，能够提高策略的计算效率，避免使用历史数据进行不必要的计算。

这两者的主要区别在于：
 
- `process_only_new_candles`  影响策略整体的蜡烛数据处理，确保只计算当前蜡烛的数据；
 
- `fresh_candle`  则专注于仓位调整的时机，确保调整只在蜡烛周期有足够时间后进行。

在实际应用中，可以根据策略需求灵活配置这两个参数，以提高策略的稳定性和效率，确保在合适的时机进行仓位调整，避免市场波动带来的错误决策。

### 宝贝推广

【量化策略37套附6套正向回测结果】 https://m.tb.cn/h.hqpATDP?tk=TRtH471WgWF