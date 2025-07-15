在加密货币交易中，许多山寨币（altcoins）的价格波动往往受比特币（BTC）价格的影响。因此，将比特币（BTC）的价格数据引入到策略中，可以更好地理解市场趋势，从而帮助你做出更准确的交易决策。通过使用比特币的价格动作作为辅助指标，你可以更好地把握市场入场时机，提高策略的盈利能力。
在本文中，我们将探讨如何使用比特币的价格数据和指数移动平均线（EMA）来优化交易策略。我们将使用Freqtrade的`informative`功能来获取比特币的数据，并将其应用到策略中。
### 1. 为什么使用BTC指标？ 

山寨币通常会受到比特币价格波动的影响，也就是说，当比特币处于上涨趋势时，许多山寨币也可能跟随上涨。因此，通过将比特币的价格动作纳入策略，你可以更好地判断市场的整体趋势，进而优化你的交易决策。

例如，当比特币处于上涨趋势时，可能意味着山寨币也会跟随上涨，此时你可以考虑做多。相反，如果比特币处于下跌趋势，则可能意味着市场进入调整阶段，你应该避免做多，甚至考虑做空。

### 2. 如何在Freqtrade中实现BTC指标 
步骤1：导入`informative`库首先，你需要从Freqtrade中导入`informative`功能。这个功能允许你拉取不同交易对（比如BTC/USDT）的数据，并用它们进行进一步的分析。

```python
from freqtrade.strategy import informative
```

#### 步骤2：定义Informative函数 
接下来，你需要定义几个函数来获取不同时间框架的比特币数据。我们将使用1分钟（`1M`）和1小时（`1h`）的BTC/USDT数据来跟踪比特币的价格动作。
这里是如何定义这些函数：


```python
# 将此代码添加到populate_indicators函数之前
@informative('1M')
def populate_indicators_1M(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    # 提供当前交易对的1分钟数据
    return dataframe

@informative('1M', 'BTC/USDT:USDT', fmt='{column}_{base}_{timeframe}')
def populate_indicators_btc_1M(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    # 提供BTC/USDT的1分钟数据
    return dataframe

@informative('1h', 'BTC/USDT:USDT', fmt='{column}_{base}_{timeframe}')
def populate_indicators_btc_1h(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    # 提供BTC/USDT的1小时数据
    return dataframe
```
 
- **`@informative('1M')`** : 指定了当前交易对的时间框架为1分钟。
 
- **`@informative('1M', 'BTC/USDT:USDT')`** : 获取BTC/USDT交易对的1分钟数据。
 
- **`@informative('1h', 'BTC/USDT:USDT')`** : 获取BTC/USDT交易对的1小时数据。

#### 步骤3：将BTC指标应用到入场和出场条件中 

现在，我们需要定义条件，将BTC的指标与当前交易对的指标结合起来。具体来说，我们希望检查当前山寨币的价格是否高于1分钟的收盘价，以及比特币是否处于上涨趋势（比较1小时和1分钟的收盘价）。
**对于做多条件** ，你可能希望在以下情况下入场：
- 当前山寨币价格高于1分钟收盘价。

- 比特币的1小时收盘价高于1分钟收盘价（表明比特币处于上涨趋势）。
**对于做空条件** ，你可能希望在以下情况下入场：
- 当前山寨币价格低于1分钟收盘价。

- 比特币的1小时收盘价低于1分钟收盘价（表明比特币处于下跌趋势）。

以下是如何在策略中实现这些条件：


```python
# 将这些条件添加到你的做多条件中：
(dataframe['close'] > dataframe['close_1M']) &  # 当前山寨币收盘价高于1分钟收盘价
(dataframe['close_btc_1h'] > dataframe['close_btc_1M'])  # 比特币1小时收盘价高于1分钟收盘价

# 将这些条件添加到你的做空条件中：
(dataframe['close'] < dataframe['close_1M']) &  # 当前山寨币收盘价低于1分钟收盘价
(dataframe['close_btc_1h'] < dataframe['close_btc_1M'])  # 比特币1小时收盘价低于1分钟收盘价
```

### 3. 条件解释 
 
- **`dataframe['close'] > dataframe['close_1M']`** ：该条件检查山寨币当前的收盘价是否高于1分钟的收盘价，表示山寨币价格处于上涨趋势。
 
- **`dataframe['close_btc_1h'] > dataframe['close_btc_1M']`** ：该条件检查比特币是否处于上涨趋势。通过比较比特币1小时收盘价与1分钟收盘价，若1小时收盘价更高，表明比特币处于上涨趋势。
对于**做空条件** ，我们将这些条件反过来使用： 
- **`dataframe['close'] < dataframe['close_1M']`** ：山寨币当前收盘价低于1分钟收盘价，表示山寨币价格处于下跌趋势。
 
- **`dataframe['close_btc_1h'] < dataframe['close_btc_1M']`** ：比特币处于下跌趋势，1小时收盘价低于1分钟收盘价。

### 4. 最终策略示例 

以下是一个简化的示例，展示如何将这些BTC指标集成到你的策略中：


```python
from freqtrade.strategy import IStrategy, informative
import talib.abstract as ta

class MyStrategy(IStrategy):

    # 定义交易对的时间框架
    timeframe = '5m'

    # 将此代码添加到populate_indicators函数之前
    @informative('1M')
    def populate_indicators_1M(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        # 提供当前交易对的1分钟数据
        return dataframe

    @informative('1M', 'BTC/USDT:USDT', fmt='{column}_{base}_{timeframe}')
    def populate_indicators_btc_1M(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        # 提供BTC/USDT的1分钟数据
        return dataframe

    @informative('1h', 'BTC/USDT:USDT', fmt='{column}_{base}_{timeframe}')
    def populate_indicators_btc_1h(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        # 提供BTC/USDT的1小时数据
        return dataframe

    # 定义做多入场条件
    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        dataframe.loc[
            (
                (dataframe['close'] > dataframe['close_1M']) &  # 山寨币高于1分钟收盘价
                (dataframe['close_btc_1h'] > dataframe['close_btc_1M']) &  # 比特币上涨
                (dataframe['ema_20'] > dataframe['close'])  # EMA确认
            ),
            'enter_long'] = 1
        return dataframe

    # 定义做空出场条件
    def populate_exit_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        dataframe.loc[
            (
                (dataframe['close'] < dataframe['close_1M']) &  # 山寨币低于1分钟收盘价
                (dataframe['close_btc_1h'] < dataframe['close_btc_1M'])  # 比特币下跌
            ),
            'exit_long'] = 1
        return dataframe
```

### 5. 结论 

将比特币的价格指标引入Freqtrade策略，可以帮助你更好地理解市场的整体趋势，因为山寨币往往会跟随比特币的价格波动。通过使用比特币的指标，如1小时和1分钟的收盘价，你可以更准确地把握入场和出场时机，从而提高策略的稳定性和盈利能力。
