在使用 **Freqtrade**  进行交易策略回测时，**trailing_stop** （跟踪止损）看似能帮助提高利润，但实际操作中可能会出现相反的结果。这是因为回测机制的限制，尤其是在回测时使用的时间线可能过于粗糙，无法捕捉到短期的价格波动，从而导致跟踪止损效果不如预期。
### 问题原因 

Freqtrade回测通常使用较大的时间线（如1小时），而在实际交易中，市场往往会在较小的时间维度（如5分钟或更短）出现价格波动。例如，某个资产在1小时的K线内看似有稳定的上涨趋势，但在更小的时间周期内，可能会出现快速下跌的插针现象，这种波动可能会触发过于紧密的止损策略，从而导致过早的平仓，错失更大收益。
由于回测仅使用1小时的K线数据，它并不能准确反映在更小时间维度上的价格波动。因此，过于依赖 **trailing_stop**  来设置止损，反而可能导致利润不升反降。
### 解决方法 
为了避免这种回测止损陷阱，建议开发者使用 **custom_exit** （自定义退出条件）来代替 trailing_stop。通过自定义的退出策略，可以灵活地控制止损的条件，更好地适应市场的短期波动。
例如，以下是一个示例代码，它根据交易的最大利润与当前利润的差距来决定是否触发退出：


```python
def custom_exit(self, pair: str, trade: Trade, current_time: datetime, current_rate: float,
                current_profit: float, **kwargs) -> Optional[Union[str, bool]]:
    max_profit = trade.calc_profit_ratio(trade.max_rate)

    if not max_profit > 0.1:
        return None

    if (max_profit - current_profit) < 0.02:
        return "custom_trailing_stop"
```
在这个例子中，只有当最大利润超过10%且当前利润与最大利润的差距小于2%时，才会触发 `custom_trailing_stop` 退出策略，从而避免在没有足够利润的情况下过早止损。
### 结论 
总的来说，使用 **trailing_stop**  进行止损可能因为回测的时间线不够细致，导致其效果低于预期。为了更准确地反映市场的动态变化，建议通过 **custom_exit**  来实现更灵活的止损策略，从而避免因过度依赖回测机制带来的潜在陷阱。

### 宝贝推广

【量化策略37套附6套正向回测结果】 https://m.tb.cn/h.hqpATDP?tk=TRtH471WgWF