# Merged Markdown Files

This file contains the merged content of the following files:

- advanced-backtesting.md
- advanced-hyperopt.md
- advanced-orderflow.md
- advanced-setup.md
- backtesting.md
- bot-basics.md
- bot-usage.md
- configuration.md
- data-analysis.md
- data-download.md
- deprecated.md
- developer.md
- docker_quickstart.md
- edge.md
- exchanges.md
- faq.md
- freqai-configuration.md
- freqai-developers.md
- freqai-feature-engineering.md
- freqai.md
- freqai-parameter-table.md
- freqai-reinforcement-learning.md
- freqai-running.md
- freq-ui.md
- hyperopt.md
- index.md
- installation.md
- leverage.md
- lookahead-analysis.md
- plotting.md
- plugins.md
- producer-consumer.md
- recursive-analysis.md
- rest-api.md
- sql_cheatsheet.md
- stoploss.md
- strategy-101.md
- strategy-advanced.md
- strategy_analysis_example.md
- strategy-callbacks.md
- strategy-customization.md
- strategy_migration.md
- telegram-usage.md
- trade-object.md
- updating.md
- utils.md
- webhook-config.md
- windows_installation.md

---

## File: advanced-backtesting.md

# Advanced Backtesting Analysis

## Analyze the buy/entry and sell/exit tags

It can be helpful to understand how a strategy behaves according to the buy/entry tags used to
mark up different buy conditions. You might want to see more complex statistics about each buy and
sell condition above those provided by the default backtesting output. You may also want to
determine indicator values on the signal candle that resulted in a trade opening.

!!! Note
    The following buy reason analysis is only available for backtesting, *not hyperopt*.

We need to run backtesting with the `--export` option set to `signals` to enable the exporting of
signals **and** trades:

``` bash
freqtrade backtesting -c <config.json> --timeframe <tf> --strategy <strategy_name> --timerange=<timerange> --export=signals
```

This will tell freqtrade to output a pickled dictionary of strategy, pairs and corresponding
DataFrame of the candles that resulted in entry and exit signals.
Depending on how many entries your strategy makes, this file may get quite large, so periodically check your `user_data/backtest_results` folder to delete old exports.

Before running your next backtest, make sure you either delete your old backtest results or run
backtesting with the `--cache none` option to make sure no cached results are used.

If all goes well, you should now see a `backtest-result-{timestamp}_signals.pkl` and `backtest-result-{timestamp}_exited.pkl` files in the `user_data/backtest_results` folder.

To analyze the entry/exit tags, we now need to use the `freqtrade backtesting-analysis` command
with `--analysis-groups` option provided with space-separated arguments:

``` bash
freqtrade backtesting-analysis -c <config.json> --analysis-groups 0 1 2 3 4 5
```

This command will read from the last backtesting results. The `--analysis-groups` option is
used to specify the various tabular outputs showing the profit of each group or trade,
ranging from the simplest (0) to the most detailed per pair, per buy and per sell tag (4):

* 0: overall winrate and profit summary by enter_tag
* 1: profit summaries grouped by enter_tag
* 2: profit summaries grouped by enter_tag and exit_tag
* 3: profit summaries grouped by pair and enter_tag
* 4: profit summaries grouped by pair, enter_ and exit_tag (this can get quite large)
* 5: profit summaries grouped by exit_tag

More options are available by running with the `-h` option.

### Using export-filename

Normally, `backtesting-analysis` uses the latest backtest results, but if you wanted to go
back to a previous backtest output, you need to supply the `--export-filename` option.
You can supply the same parameter to `backtest-analysis` with the name of the final backtest
output file. This allows you to keep historical versions of backtest results and re-analyse
them at a later date:

``` bash
freqtrade backtesting -c <config.json> --timeframe <tf> --strategy <strategy_name> --timerange=<timerange> --export=signals --export-filename=/tmp/mystrat_backtest.json
```

You should see some output similar to below in the logs with the name of the timestamped
filename that was exported:

```
2022-06-14 16:28:32,698 - freqtrade.misc - INFO - dumping json to "/tmp/mystrat_backtest-2022-06-14_16-28-32.json"
```

You can then use that filename in `backtesting-analysis`:

```
freqtrade backtesting-analysis -c <config.json> --export-filename=/tmp/mystrat_backtest-2022-06-14_16-28-32.json
```

### Tuning the buy tags and sell tags to display

To show only certain buy and sell tags in the displayed output, use the following two options:

```
--enter-reason-list : Space-separated list of enter signals to analyse. Default: "all"
--exit-reason-list : Space-separated list of exit signals to analyse. Default: "all"
```

For example:

```bash
freqtrade backtesting-analysis -c <config.json> --analysis-groups 0 2 --enter-reason-list enter_tag_a enter_tag_b --exit-reason-list roi custom_exit_tag_a stop_loss
```

### Outputting signal candle indicators

The real power of `freqtrade backtesting-analysis` comes from the ability to print out the indicator
values present on signal candles to allow fine-grained investigation and tuning of buy signal
indicators. To print out a column for a given set of indicators, use the `--indicator-list`
option:

```bash
freqtrade backtesting-analysis -c <config.json> --analysis-groups 0 2 --enter-reason-list enter_tag_a enter_tag_b --exit-reason-list roi custom_exit_tag_a stop_loss --indicator-list rsi rsi_1h bb_lowerband ema_9 macd macdsignal
```

The indicators have to be present in your strategy's main DataFrame (either for your main
timeframe or for informative timeframes) otherwise they will simply be ignored in the script
output.

!!! Note "Indicator List"
    The indicator values will be displayed for both entry and exit points. If `--indicator-list all` is specified, 
    only the indicators at the entry point will be shown to avoid excessively large lists, which could occur depending on the strategy.

There are a range of candle and trade-related fields that are included in the analysis so are 
automatically accessible by including them on the indicator-list, and these include:

- **open_date     :** trade open datetime
- **close_date    :** trade close datetime
- **min_rate      :** minimum price seen throughout the position
- **max_rate      :** maximum price seen throughout the position
- **open          :** signal candle open price
- **close         :** signal candle close price
- **high          :** signal candle high price
- **low           :** signal candle low price
- **volume        :** signal candle volume
- **profit_ratio  :** trade profit ratio
- **profit_abs    :** absolute profit return of the trade 

#### Sample Output for Indicator Values

```bash
freqtrade backtesting-analysis -c user_data/config.json --analysis-groups 0 --indicator-list chikou_span tenkan_sen 
```

In this example,
we aim to display the `chikou_span` and `tenkan_sen` indicator values at both the entry and exit points of trades.

A sample output for indicators might look like this:

| pair      | open_date                 | enter_reason | exit_reason | chikou_span (entry) | tenkan_sen (entry) | chikou_span (exit) | tenkan_sen (exit) |
|-----------|---------------------------|--------------|-------------|---------------------|--------------------|--------------------|-------------------|
| DOGE/USDT | 2024-07-06 00:35:00+00:00 |              | exit_signal | 0.105               | 0.106              | 0.105              | 0.107             |
| BTC/USDT  | 2024-08-05 14:20:00+00:00 |              | roi         | 54643.440           | 51696.400          | 54386.000          | 52072.010         |

As shown in the table, `chikou_span (entry)` represents the indicator value at the time of trade entry, 
while `chikou_span (exit)` reflects its value at the time of exit. 
This detailed view of indicator values enhances the analysis.

The `(entry)` and `(exit)` suffixes are added to indicators
to distinguish the values at the entry and exit points of the trade.

!!! Note "Trade-wide Indicators"
    Certain trade-wide indicators do not have the `(entry)` or `(exit)` suffix. These indicators include: `pair`, `stake_amount`, 
    `max_stake_amount`, `amount`, `open_date`, `close_date`, `open_rate`, `close_rate`, `fee_open`, `fee_close`, `trade_duration`, 
    `profit_ratio`, `profit_abs`, `exit_reason`,`initial_stop_loss_abs`, `initial_stop_loss_ratio`, `stop_loss_abs`, `stop_loss_ratio`, 
    `min_rate`, `max_rate`, `is_open`, `enter_tag`, `leverage`, `is_short`, `open_timestamp`, `close_timestamp` and `orders`

#### Filtering Indicators Based on Entry or Exit Signals

The `--indicator-list` option, by default, displays indicator values for both entry and exit signals. To filter the indicator values exclusively for entry signals, you can use the `--entry-only` argument. Similarly, to display indicator values only at exit signals, use the `--exit-only` argument.

Example: Display indicator values at entry signals:

```bash
freqtrade backtesting-analysis -c user_data/config.json --analysis-groups 0 --indicator-list chikou_span tenkan_sen --entry-only
```

Example: Display indicator values at exit signals:

```bash
freqtrade backtesting-analysis -c user_data/config.json --analysis-groups 0 --indicator-list chikou_span tenkan_sen --exit-only
```

!!! note 
    When using these filters, the indicator names will not be suffixed with `(entry)` or `(exit)`.

### Filtering the trade output by date

To show only trades between dates within your backtested timerange, supply the usual `timerange` option in `YYYYMMDD-[YYYYMMDD]` format:

```
--timerange : Timerange to filter output trades, start date inclusive, end date exclusive. e.g. 20220101-20221231
```

For example, if your backtest timerange was `20220101-20221231` but you only want to output trades in January:

```bash
freqtrade backtesting-analysis -c <config.json> --timerange 20220101-20220201
```

### Printing out rejected signals

Use the `--rejected-signals` option to print out rejected signals.

```bash
freqtrade backtesting-analysis -c <config.json> --rejected-signals
```

### Writing tables to CSV

Some of the tabular outputs can become large, so printing them out to the terminal is not preferable.
Use the `--analysis-to-csv` option to disable printing out of tables to standard out and write them to CSV files.

```bash
freqtrade backtesting-analysis -c <config.json> --analysis-to-csv
```

By default this will write one file per output table you specified in the `backtesting-analysis` command, e.g.

```bash
freqtrade backtesting-analysis -c <config.json> --analysis-to-csv --rejected-signals --analysis-groups 0 1
```

This will write to `user_data/backtest_results`:

* rejected_signals.csv
* group_0.csv
* group_1.csv

To override where the files will be written, also specify the `--analysis-csv-path` option.

```bash
freqtrade backtesting-analysis -c <config.json> --analysis-to-csv --analysis-csv-path another/data/path/
```

---

## File: advanced-hyperopt.md

# Advanced Hyperopt

This page explains some advanced Hyperopt topics that may require higher
coding skills and Python knowledge than creation of an ordinal hyperoptimization
class.

## Creating and using a custom loss function

To use a custom loss function class, make sure that the function `hyperopt_loss_function` is defined in your custom hyperopt loss class.
For the sample below, you then need to add the command line parameter `--hyperopt-loss SuperDuperHyperOptLoss` to your hyperopt call so this function is being used.

A sample of this can be found below, which is identical to the Default Hyperopt loss implementation. A full sample can be found in [userdata/hyperopts](https://github.com/freqtrade/freqtrade/blob/develop/freqtrade/templates/sample_hyperopt_loss.py).

``` python
from datetime import datetime
from typing import Any, Dict

from pandas import DataFrame

from freqtrade.constants import Config
from freqtrade.optimize.hyperopt import IHyperOptLoss

TARGET_TRADES = 600
EXPECTED_MAX_PROFIT = 3.0
MAX_ACCEPTED_TRADE_DURATION = 300

class SuperDuperHyperOptLoss(IHyperOptLoss):
    """
    Defines the default loss function for hyperopt
    """

    @staticmethod
    def hyperopt_loss_function(
        *,
        results: DataFrame,
        trade_count: int,
        min_date: datetime,
        max_date: datetime,
        config: Config,
        processed: dict[str, DataFrame],
        backtest_stats: dict[str, Any],
        starting_balance: float,
        **kwargs,
    ) -> float:
        """
        Objective function, returns smaller number for better results
        This is the legacy algorithm (used until now in freqtrade).
        Weights are distributed as follows:
        * 0.4 to trade duration
        * 0.25: Avoiding trade loss
        * 1.0 to total profit, compared to the expected value (`EXPECTED_MAX_PROFIT`) defined above
        """
        total_profit = results['profit_ratio'].sum()
        trade_duration = results['trade_duration'].mean()

        trade_loss = 1 - 0.25 * exp(-(trade_count - TARGET_TRADES) ** 2 / 10 ** 5.8)
        profit_loss = max(0, 1 - total_profit / EXPECTED_MAX_PROFIT)
        duration_loss = 0.4 * min(trade_duration / MAX_ACCEPTED_TRADE_DURATION, 1)
        result = trade_loss + profit_loss + duration_loss
        return result
```

Currently, the arguments are:

* `results`: DataFrame containing the resulting trades.
    The following columns are available in results (corresponds to the output-file of backtesting when used with `--export trades`):  
    `pair, profit_ratio, profit_abs, open_date, open_rate, fee_open, close_date, close_rate, fee_close, amount, trade_duration, is_open, exit_reason, stake_amount, min_rate, max_rate, stop_loss_ratio, stop_loss_abs`
* `trade_count`: Amount of trades (identical to `len(results)`)
* `min_date`: Start date of the timerange used
* `min_date`: End date of the timerange used
* `config`: Config object used (Note: Not all strategy-related parameters will be updated here if they are part of a hyperopt space).
* `processed`: Dict of Dataframes with the pair as keys containing the data used for backtesting.
* `backtest_stats`: Backtesting statistics using the same format as the backtesting file "strategy" substructure. Available fields can be seen in `generate_strategy_stats()` in `optimize_reports.py`.
* `starting_balance`: Starting balance used for backtesting.

This function needs to return a floating point number (`float`). Smaller numbers will be interpreted as better results. The parameters and balancing for this is up to you.

!!! Note
    This function is called once per epoch - so please make sure to have this as optimized as possible to not slow hyperopt down unnecessarily.

!!! Note "`*args` and `**kwargs`"
    Please keep the arguments `*args` and `**kwargs` in the interface to allow us to extend this interface in the future.

## Overriding pre-defined spaces

To override a pre-defined space (`roi_space`, `generate_roi_table`, `stoploss_space`, `trailing_space`, `max_open_trades_space`), define a nested class called Hyperopt and define the required spaces as follows:

```python
from freqtrade.optimize.space import Categorical, Dimension, Integer, SKDecimal

class MyAwesomeStrategy(IStrategy):
    class HyperOpt:
        # Define a custom stoploss space.
        def stoploss_space():
            return [SKDecimal(-0.05, -0.01, decimals=3, name='stoploss')]

        # Define custom ROI space
        def roi_space() -> List[Dimension]:
            return [
                Integer(10, 120, name='roi_t1'),
                Integer(10, 60, name='roi_t2'),
                Integer(10, 40, name='roi_t3'),
                SKDecimal(0.01, 0.04, decimals=3, name='roi_p1'),
                SKDecimal(0.01, 0.07, decimals=3, name='roi_p2'),
                SKDecimal(0.01, 0.20, decimals=3, name='roi_p3'),
            ]

        def generate_roi_table(params: Dict) -> dict[int, float]:

            roi_table = {}
            roi_table[0] = params['roi_p1'] + params['roi_p2'] + params['roi_p3']
            roi_table[params['roi_t3']] = params['roi_p1'] + params['roi_p2']
            roi_table[params['roi_t3'] + params['roi_t2']] = params['roi_p1']
            roi_table[params['roi_t3'] + params['roi_t2'] + params['roi_t1']] = 0

            return roi_table

        def trailing_space() -> List[Dimension]:
            # All parameters here are mandatory, you can only modify their type or the range.
            return [
                # Fixed to true, if optimizing trailing_stop we assume to use trailing stop at all times.
                Categorical([True], name='trailing_stop'),

                SKDecimal(0.01, 0.35, decimals=3, name='trailing_stop_positive'),
                # 'trailing_stop_positive_offset' should be greater than 'trailing_stop_positive',
                # so this intermediate parameter is used as the value of the difference between
                # them. The value of the 'trailing_stop_positive_offset' is constructed in the
                # generate_trailing_params() method.
                # This is similar to the hyperspace dimensions used for constructing the ROI tables.
                SKDecimal(0.001, 0.1, decimals=3, name='trailing_stop_positive_offset_p1'),

                Categorical([True, False], name='trailing_only_offset_is_reached'),
        ]

        # Define a custom max_open_trades space
        def max_open_trades_space(self) -> List[Dimension]:
            return [
                Integer(-1, 10, name='max_open_trades'),
            ]
```

!!! Note
    All overrides are optional and can be mixed/matched as necessary.

### Dynamic parameters

Parameters can also be defined dynamically, but must be available to the instance once the [`bot_start()` callback](strategy-callbacks.md#bot-start) has been called.

``` python

class MyAwesomeStrategy(IStrategy):

    def bot_start(self, **kwargs) -> None:
        self.buy_adx = IntParameter(20, 30, default=30, optimize=True)

    # ...
```

!!! Warning
    Parameters created this way will not show up in the `list-strategies` parameter count.

### Overriding Base estimator

You can define your own estimator for Hyperopt by implementing `generate_estimator()` in the Hyperopt subclass.

```python
class MyAwesomeStrategy(IStrategy):
    class HyperOpt:
        def generate_estimator(dimensions: List['Dimension'], **kwargs):
            return "RF"

```

Possible values are either one of "GP", "RF", "ET", "GBRT" (Details can be found in the [scikit-optimize documentation](https://scikit-optimize.github.io/)), or "an instance of a class that inherits from `RegressorMixin` (from sklearn) and where the `predict` method has an optional `return_std` argument, which returns `std(Y | x)` along with `E[Y | x]`".

Some research will be necessary to find additional Regressors.

Example for `ExtraTreesRegressor` ("ET") with additional parameters:

```python
class MyAwesomeStrategy(IStrategy):
    class HyperOpt:
        def generate_estimator(dimensions: List['Dimension'], **kwargs):
            from skopt.learning import ExtraTreesRegressor
            # Corresponds to "ET" - but allows additional parameters.
            return ExtraTreesRegressor(n_estimators=100)

```

The `dimensions` parameter is the list of `skopt.space.Dimension` objects corresponding to the parameters to be optimized. It can be used to create isotropic kernels for the `skopt.learning.GaussianProcessRegressor` estimator. Here's an example:

```python
class MyAwesomeStrategy(IStrategy):
    class HyperOpt:
        def generate_estimator(dimensions: List['Dimension'], **kwargs):
            from skopt.utils import cook_estimator
            from skopt.learning.gaussian_process.kernels import (Matern, ConstantKernel)
            kernel_bounds = (0.0001, 10000)
            kernel = (
                ConstantKernel(1.0, kernel_bounds) * 
                Matern(length_scale=np.ones(len(dimensions)), length_scale_bounds=[kernel_bounds for d in dimensions], nu=2.5)
            )
            kernel += (
                ConstantKernel(1.0, kernel_bounds) * 
                Matern(length_scale=np.ones(len(dimensions)), length_scale_bounds=[kernel_bounds for d in dimensions], nu=1.5)
            )

            return cook_estimator("GP", space=dimensions, kernel=kernel, n_restarts_optimizer=2)
```

!!! Note
    While custom estimators can be provided, it's up to you as User to do research on possible parameters and analyze / understand which ones should be used.
    If you're unsure about this, best use one of the Defaults (`"ET"` has proven to be the most versatile) without further parameters.

## Space options

For the additional spaces, scikit-optimize (in combination with Freqtrade) provides the following space types:

* `Categorical` - Pick from a list of categories (e.g. `Categorical(['a', 'b', 'c'], name="cat")`)
* `Integer` - Pick from a range of whole numbers (e.g. `Integer(1, 10, name='rsi')`)
* `SKDecimal` - Pick from a range of decimal numbers with limited precision (e.g. `SKDecimal(0.1, 0.5, decimals=3, name='adx')`). *Available only with freqtrade*.
* `Real` - Pick from a range of decimal numbers with full precision (e.g. `Real(0.1, 0.5, name='adx')`

You can import all of these from `freqtrade.optimize.space`, although `Categorical`, `Integer` and `Real` are only aliases for their corresponding scikit-optimize Spaces. `SKDecimal` is provided by freqtrade for faster optimizations.

``` python
from freqtrade.optimize.space import Categorical, Dimension, Integer, SKDecimal, Real  # noqa
```

!!! Hint "SKDecimal vs. Real"
    We recommend to use `SKDecimal` instead of the `Real` space in almost all cases. While the Real space provides full accuracy (up to ~16 decimal places) - this precision is rarely needed, and leads to unnecessary long hyperopt times.

    Assuming the definition of a rather small space (`SKDecimal(0.10, 0.15, decimals=2, name='xxx')`) - SKDecimal will have 5 possibilities (`[0.10, 0.11, 0.12, 0.13, 0.14, 0.15]`).

    A corresponding real space `Real(0.10, 0.15 name='xxx')`  on the other hand has an almost unlimited number of possibilities (`[0.10, 0.010000000001, 0.010000000002, ... 0.014999999999, 0.01500000000]`).

---

## File: advanced-orderflow.md

# Orderflow data

This guide walks you through utilizing public trade data for advanced orderflow analysis in Freqtrade.

!!! Warning "Experimental Feature"
    The orderflow feature is currently in beta and may be subject to changes in future releases. Please report any issues or feedback on the [Freqtrade GitHub repository](https://github.com/freqtrade/freqtrade/issues).
    It's also currently not been tested with freqAI - and combining these two features is considered out of scope at this point.

!!! Warning "Performance"
    Orderflow requires raw trades data. This data is rather large, and can cause a slow initial startup, when freqtrade needs to download the trades data for the last X candles. Additionally, enabling this feature will cause increased memory usage. Please ensure to have sufficient resources available.

## Getting Started

### Enable Public Trades

In your `config.json` file, set the `use_public_trades` option to true under the `exchange` section.

```json
"exchange": {
   ...
   "use_public_trades": true,
}
```

### Configure Orderflow Processing

Define your desired settings for orderflow processing within the orderflow section of config.json. Here, you can adjust factors like:

- `cache_size`: How many previous orderflow candles are saved into cache instead of calculated every new candle
- `max_candles`: Filter how many candles would you like to get trades data for.
- `scale`: This controls the price bin size for the footprint chart.
- `stacked_imbalance_range`: Defines the minimum consecutive imbalanced price levels required for consideration.
- `imbalance_volume`: Filters out imbalances with volume below this threshold.
- `imbalance_ratio`: Filters out imbalances with a ratio (difference between ask and bid volume) lower than this value.

```json
"orderflow": {
    "cache_size": 1000, 
    "max_candles": 1500, 
    "scale": 0.5, 
    "stacked_imbalance_range": 3, //  needs at least this amount of imbalance next to each other
    "imbalance_volume": 1, //  filters out below
    "imbalance_ratio": 3 //  filters out ratio lower than
  },
```

## Downloading Trade Data for Backtesting

To download historical trade data for backtesting, use the --dl-trades flag with the freqtrade download-data command.

```bash
freqtrade download-data -p BTC/USDT:USDT --timerange 20230101- --trading-mode futures --timeframes 5m --dl-trades
```

!!! Warning "Data availability"
    Not all exchanges provide public trade data. For supported exchanges, freqtrade will warn you if public trade data is not available if you start downloading data with the `--dl-trades` flag.

## Accessing Orderflow Data

Once activated, several new columns become available in your dataframe:

``` python

dataframe["trades"] # Contains information about each individual trade.
dataframe["orderflow"] # Represents a footprint chart dict (see below)
dataframe["imbalances"] # Contains information about imbalances in the order flow.
dataframe["bid"] # Total bid volume 
dataframe["ask"] # Total ask volume
dataframe["delta"] # Difference between ask and bid volume.
dataframe["min_delta"] # Minimum delta within the candle
dataframe["max_delta"] # Maximum delta within the candle
dataframe["total_trades"] # Total number of trades
dataframe["stacked_imbalances_bid"] # List of price levels of stacked bid imbalance range beginnings
dataframe["stacked_imbalances_ask"] # List of price levels of stacked ask imbalance range beginnings
```

You can access these columns in your strategy code for further analysis. Here's an example:

``` python
def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    # Calculating cumulative delta
    dataframe["cum_delta"] = cumulative_delta(dataframe["delta"])
    # Accessing total trades
    total_trades = dataframe["total_trades"]
    ...

def cumulative_delta(delta: Series):
    cumdelta = delta.cumsum()
    return cumdelta

```

### Footprint chart (`dataframe["orderflow"]`)

This column provides a detailed breakdown of buy and sell orders at different price levels, offering valuable insights into order flow dynamics. The `scale` parameter in your configuration determines the price bin size for this representation

The `orderflow` column contains a dict with the following structure:

``` output
{
    "price": {
        "bid_amount": 0.0,
        "ask_amount": 0.0,
        "bid": 0,
        "ask": 0,
        "delta": 0.0,
        "total_volume": 0.0,
        "total_trades": 0
    }
}
```

#### Orderflow column explanation

- key: Price bin - binned at `scale` intervals
- `bid_amount`: Total volume bought at each price level.
- `ask_amount`: Total volume sold at each price level.
- `bid`: Number of buy orders at each price level.
- `ask`: Number of sell orders at each price level.
- `delta`: Difference between ask and bid volume at each price level.
- `total_volume`: Total volume (ask amount + bid amount) at each price level.
- `total_trades`: Total number of trades (ask + bid) at each price level.

By leveraging these features, you can gain valuable insights into market sentiment and potential trading opportunities based on order flow analysis.

### Raw trades data (`dataframe["trades"]`)

List with the individual trades that occurred during the candle. This data can be used for more granular analysis of order flow dynamics.

Each individual entry contains a dict with the following keys:

- `timestamp`: Timestamp of the trade.
- `date`: Date of the trade.
- `price`: Price of the trade.
- `amount`: Volume of the trade.
- `side`: Buy or sell.
- `id`: Unique identifier for the trade.
- `cost`: Total cost of the trade (price * amount).

### Imbalances (`dataframe["imbalances"]`)

This column provides a dict with information about imbalances in the order flow. An imbalance occurs when there is a significant difference between the ask and bid volume at a given price level.

Each row looks as follows - with price as index, and the corresponding bid and ask imbalance values as columns

``` output
{
    "price": {
        "bid_imbalance": False,
        "ask_imbalance": False
    }
}
```

---

## File: advanced-setup.md

# Advanced Post-installation Tasks

This page explains some advanced tasks and configuration options that can be performed after the bot installation and may be uselful in some environments.

If you do not know what things mentioned here mean, you probably do not need it.

## Running multiple instances of Freqtrade

This section will show you how to run multiple bots at the same time, on the same machine.

### Things to consider

* Use different database files.
* Use different Telegram bots (requires multiple different configuration files; applies only when Telegram is enabled).
* Use different ports (applies only when Freqtrade REST API webserver is enabled).

### Different database files

In order to keep track of your trades, profits, etc., freqtrade is using a SQLite database where it stores various types of information such as the trades you performed in the past and the current position(s) you are holding at any time. This allows you to keep track of your profits, but most importantly, keep track of ongoing activity if the bot process would be restarted or would be terminated unexpectedly.

Freqtrade will, by default, use separate database files for dry-run and live bots (this assumes no database-url is given in either configuration nor via command line argument).
For live trading mode, the default database will be `tradesv3.sqlite` and for dry-run it will be `tradesv3.dryrun.sqlite`.

The optional argument to the trade command used to specify the path of these files is `--db-url`, which requires a valid SQLAlchemy url.
So when you are starting a bot with only the config and strategy arguments in dry-run mode, the following 2 commands would have the same outcome.

``` bash
freqtrade trade -c MyConfig.json -s MyStrategy
# is equivalent to
freqtrade trade -c MyConfig.json -s MyStrategy --db-url sqlite:///tradesv3.dryrun.sqlite
```

It means that if you are running the trade command in two different terminals, for example to test your strategy both for trades in USDT and in another instance for trades in BTC, you will have to run them with different databases.

If you specify the URL of a database which does not exist, freqtrade will create one with the name you specified. So to test your custom strategy with BTC and USDT stake currencies, you could use the following commands (in 2 separate terminals):

``` bash
# Terminal 1:
freqtrade trade -c MyConfigBTC.json -s MyCustomStrategy --db-url sqlite:///user_data/tradesBTC.dryrun.sqlite
# Terminal 2:
freqtrade trade -c MyConfigUSDT.json -s MyCustomStrategy --db-url sqlite:///user_data/tradesUSDT.dryrun.sqlite
```

Conversely, if you wish to do the same thing in production mode, you will also have to create at least one new database (in addition to the default one) and specify the path to the "live" databases, for example:

``` bash
# Terminal 1:
freqtrade trade -c MyConfigBTC.json -s MyCustomStrategy --db-url sqlite:///user_data/tradesBTC.live.sqlite
# Terminal 2:
freqtrade trade -c MyConfigUSDT.json -s MyCustomStrategy --db-url sqlite:///user_data/tradesUSDT.live.sqlite
```

For more information regarding usage of the sqlite databases, for example to manually enter or remove trades, please refer to the [SQL Cheatsheet](sql_cheatsheet.md).

### Multiple instances using docker

To run multiple instances of freqtrade using docker you will need to edit the docker-compose.yml file and add all the instances you want as separate services. Remember, you can separate your configuration into multiple files, so it's a good idea to think about making them modular, then if you need to edit something common to all bots, you can do that in a single config file. 
``` yml
---
version: '3'
services:
  freqtrade1:
    image: freqtradeorg/freqtrade:stable
    # image: freqtradeorg/freqtrade:develop
    # Use plotting image
    # image: freqtradeorg/freqtrade:develop_plot
    # Build step - only needed when additional dependencies are needed
    # build:
    #   context: .
    #   dockerfile: "./docker/Dockerfile.custom"
    restart: always
    container_name: freqtrade1
    volumes:
      - "./user_data:/freqtrade/user_data"
    # Expose api on port 8080 (localhost only)
    # Please read the https://www.freqtrade.io/en/latest/rest-api/ documentation
    # before enabling this.
     ports:
     - "127.0.0.1:8080:8080"
    # Default command used when running `docker compose up`
    command: >
      trade
      --logfile /freqtrade/user_data/logs/freqtrade1.log
      --db-url sqlite:////freqtrade/user_data/tradesv3_freqtrade1.sqlite
      --config /freqtrade/user_data/config.json
      --config /freqtrade/user_data/config.freqtrade1.json
      --strategy SampleStrategy
  
  freqtrade2:
    image: freqtradeorg/freqtrade:stable
    # image: freqtradeorg/freqtrade:develop
    # Use plotting image
    # image: freqtradeorg/freqtrade:develop_plot
    # Build step - only needed when additional dependencies are needed
    # build:
    #   context: .
    #   dockerfile: "./docker/Dockerfile.custom"
    restart: always
    container_name: freqtrade2
    volumes:
      - "./user_data:/freqtrade/user_data"
    # Expose api on port 8080 (localhost only)
    # Please read the https://www.freqtrade.io/en/latest/rest-api/ documentation
    # before enabling this.
    ports:
      - "127.0.0.1:8081:8080"
    # Default command used when running `docker compose up`
    command: >
      trade
      --logfile /freqtrade/user_data/logs/freqtrade2.log
      --db-url sqlite:////freqtrade/user_data/tradesv3_freqtrade2.sqlite
      --config /freqtrade/user_data/config.json
      --config /freqtrade/user_data/config.freqtrade2.json
      --strategy SampleStrategy

```

You can use whatever naming convention you want, freqtrade1 and 2 are arbitrary. Note, that you will need to use different database files, port mappings and telegram configurations for each instance, as mentioned above. 

## Use a different database system

Freqtrade is using SQLAlchemy, which supports multiple different database systems. As such, a multitude of database systems should be supported.
Freqtrade does not depend or install any additional database driver. Please refer to the [SQLAlchemy docs](https://docs.sqlalchemy.org/en/14/core/engines.html#database-urls) on installation instructions for the respective database systems.

The following systems have been tested and are known to work with freqtrade:

* sqlite (default)
* PostgreSQL
* MariaDB

!!! Warning
    By using one of the below database systems, you acknowledge that you know how to manage such a system. The freqtrade team will not provide any support with setup or maintenance (or backups) of the below database systems.

### PostgreSQL

Installation:
`pip install psycopg2-binary`

Usage:
`... --db-url postgresql+psycopg2://<username>:<password>@localhost:5432/<database>`

Freqtrade will automatically create the tables necessary upon startup.

If you're running different instances of Freqtrade, you must either setup one database per Instance or use different users / schemas for your connections.

### MariaDB / MySQL

Freqtrade supports MariaDB by using SQLAlchemy, which supports multiple different database systems.

Installation:
`pip install pymysql`

Usage:
`... --db-url mysql+pymysql://<username>:<password>@localhost:3306/<database>`



## Configure the bot running as a systemd service

Copy the `freqtrade.service` file to your systemd user directory (usually `~/.config/systemd/user`) and update `WorkingDirectory` and `ExecStart` to match your setup.

!!! Note
    Certain systems (like Raspbian) don't load service unit files from the user directory. In this case, copy `freqtrade.service` into `/etc/systemd/user/` (requires superuser permissions).

After that you can start the daemon with:

```bash
systemctl --user start freqtrade
```

For this to be persistent (run when user is logged out) you'll need to enable `linger` for your freqtrade user.

```bash
sudo loginctl enable-linger "$USER"
```

If you run the bot as a service, you can use systemd service manager as a software watchdog monitoring freqtrade bot 
state and restarting it in the case of failures. If the `internals.sd_notify` parameter is set to true in the 
configuration or the `--sd-notify` command line option is used, the bot will send keep-alive ping messages to systemd 
using the sd_notify (systemd notifications) protocol and will also tell systemd its current state (Running or Stopped) 
when it changes. 

The `freqtrade.service.watchdog` file contains an example of the service unit configuration file which uses systemd 
as the watchdog.

!!! Note
    The sd_notify communication between the bot and the systemd service manager will not work if the bot runs in a Docker container.

## Advanced Logging

Freqtrade uses the default logging module provided by python.
Python allows for extensive [logging configuration](https://docs.python.org/3/library/logging.config.html#logging.config.dictConfig) in this regards - way more than what can be covered here.

Default logging (Colored terminal output) is setup by default if no `log_config` is provided.
Using `--logfile logfile.log` will enable the RotatingFileHandler.
If you're not content with the log format - or with the default settings provided for the RotatingFileHandler, you can customize logging to your liking.

The default configuration looks roughly like the below - with the file handler being provided - but not enabled.

``` json hl_lines="5-7 13-16 27"
{
  "log_config": {
      "version": 1,
      "formatters": {
          "basic": {
              "format": "%(message)s"
          },
          "standard": {
              "format": "%(asctime)s - %(name)s - %(levelname)s - %(message)s"
          }
      },
      "handlers": {
          "console": {
              "class": "freqtrade.loggers.ft_rich_handler.FtRichHandler",
              "formatter": "basic"
          },
          "file": {
              "class": "logging.handlers.RotatingFileHandler",
              "formatter": "standard",
              // "filename": "someRandomLogFile.log",
              "maxBytes": 10485760,
              "backupCount": 10
          }
      },
      "root": {
          "handlers": [
              "console",
              // "file"
          ],
          "level": "INFO",
      }
  }
}
```

!!! Note "highlighted lines"
    Highlighted lines in the above code-block define the Rich handler and belong together.
    The formatter "standard" and "file" will belong to the FileHandler.

Each handler must use one of the defined formatters (by name) - and it's class must be available and a valid logging class.
To actually use a handler - it must be in the "handlers" section inside the "root" segment.
If this section is left out, freqtrade will provide no output (in the non-configured handler, anyway).

!!! Tip "Explicit log configuration"
    We recommend to extract the logging configuration from your main configuration, and provide it to your bot via [multiple configuration files](configuration.md#multiple-configuration-files) functionality. This will avoid unnecessary code duplication.

---

On many Linux systems the bot can be configured to send its log messages to `syslog` or `journald` system services. Logging to a remote `syslog` server is also available on Windows. The special values for the `--logfile` command line option can be used for this.

### Logging to syslog

To send Freqtrade log messages to a local or remote `syslog` service use the `"log_config"` setup option to configure logging.

``` json
{
  // ...
  "log_config": {
    "version": 1,
    "formatters": {
      "syslog_fmt": {
        "format": "%(name)s - %(levelname)s - %(message)s"
      }
    },
    "handlers": {
      // Other handlers? 
      "syslog": {
         "class": "logging.handlers.SysLogHandler",
          "formatter": "syslog_fmt",
          // Use one of the other options above as address instead? 
          "address": "/dev/log"
      }
    },
    "root": {
      "handlers": [
        // other handlers
        "syslog",
        
      ]
    }

  }
}
```

[Additional log-handlers](#advanced-logging) may need to be configured to for example also have log output in the console.

#### Syslog usage

Log messages are send to `syslog` with the `user` facility. So you can see them with the following commands:

* `tail -f /var/log/user`, or
* install a comprehensive graphical viewer (for instance, 'Log File Viewer' for Ubuntu).

On many systems `syslog` (`rsyslog`) fetches data from `journald` (and vice versa), so both syslog or journald can be used and the messages be viewed with both `journalctl` and a syslog viewer utility. You can combine this in any way which suites you better.

For `rsyslog` the messages from the bot can be redirected into a separate dedicated log file. To achieve this, add

```
if $programname startswith "freqtrade" then -/var/log/freqtrade.log
```

to one of the rsyslog configuration files, for example at the end of the `/etc/rsyslog.d/50-default.conf`.

For `syslog` (`rsyslog`), the reduction mode can be switched on. This will reduce the number of repeating messages. For instance, multiple bot Heartbeat messages will be reduced to a single message when nothing else happens with the bot. To achieve this, set in `/etc/rsyslog.conf`:

```
# Filter duplicated messages
$RepeatedMsgReduction on
```

#### Syslog addressing

The syslog address can be either a Unix domain socket (socket filename) or a UDP socket specification, consisting of IP address and UDP port, separated by the `:` character.


So, the following are the examples of possible addresses:

* `"address": "/dev/log"` -- log to syslog (rsyslog) using the `/dev/log` socket, suitable for most systems.
* `"address": "/var/run/syslog"` -- log to syslog (rsyslog) using the `/var/run/syslog` socket. Use this on MacOS.
* `"address": "localhost:514"` -- log to local syslog using UDP socket, if it listens on port 514.
* `"address": "<ip>:514"` -- log to remote syslog at IP address and port 514. This may be used on Windows for remote logging to an external syslog server.


??? Info "Deprecated - configure syslog via command line"

  `--logfile syslog:<syslog_address>` -- send log messages to `syslog` service using the `<syslog_address>` as the syslog address.

  The syslog address can be either a Unix domain socket (socket filename) or a UDP socket specification, consisting of IP address and UDP port, separated by the `:` character.

  So, the following are the examples of possible usages:

  * `--logfile syslog:/dev/log` -- log to syslog (rsyslog) using the `/dev/log` socket, suitable for most systems.
  * `--logfile syslog` -- same as above, the shortcut for `/dev/log`.
  * `--logfile syslog:/var/run/syslog` -- log to syslog (rsyslog) using the `/var/run/syslog` socket. Use this on MacOS.
  * `--logfile syslog:localhost:514` -- log to local syslog using UDP socket, if it listens on port 514.
  * `--logfile syslog:<ip>:514` -- log to remote syslog at IP address and port 514. This may be used on Windows for remote logging to an external syslog server.

### Logging to journald

This needs the `cysystemd` python package installed as dependency (`pip install cysystemd`), which is not available on Windows. Hence, the whole journald logging functionality is not available for a bot running on Windows.

To send Freqtrade log messages to `journald` system service, add the following configuration snippet to your configuration.

``` json
{
  // ...
  "log_config": {
    "version": 1,
    "formatters": {
      "journald_fmt": {
        "format": "%(name)s - %(levelname)s - %(message)s"
      }
    },
    "handlers": {
      // Other handlers? 
      "journald": {
         "class": "cysystemd.journal.JournaldLogHandler",
          "formatter": "journald_fmt",
      }
    },
    "root": {
      "handlers": [
        // .. 
        "journald",
        
      ]
    }

  }
}
```

[Additional log-handlers](#advanced-logging) may need to be configured to for example also have log output in the console.

Log messages are send to `journald` with the `user` facility. So you can see them with the following commands:

* `journalctl -f` -- shows Freqtrade log messages sent to `journald` along with other log messages fetched by `journald`.
* `journalctl -f -u freqtrade.service` -- this command can be used when the bot is run as a `systemd` service.

There are many other options in the `journalctl` utility to filter the messages, see manual pages for this utility.

On many systems `syslog` (`rsyslog`) fetches data from `journald` (and vice versa), so both `--logfile syslog` or `--logfile journald` can be used and the messages be viewed with both `journalctl` and a syslog viewer utility. You can combine this in any way which suites you better.

??? Info "Deprecated - configure journald via command line"
    To send Freqtrade log messages to `journald` system service use the `--logfile` command line option with the value in the following format:

    `--logfile journald` -- send log messages to `journald`.

### Log format as JSON

You can also configure the default output stream to use JSON format instead.
The "fmt_dict" attribute defines the keys for the json output - as well as the [python logging LogRecord attributes](https://docs.python.org/3/library/logging.html#logrecord-attributes).

The below configuration will change the default output to JSON. The same formatter could however also be used in combination with the `RotatingFileHandler`.
We recommend to keep one format in human readable form.

``` json
{
  // ...
  "log_config": {
    "version": 1,
    "formatters": {
       "json": {
          "()": "freqtrade.loggers.json_formatter.JsonFormatter",
          "fmt_dict": {
              "timestamp": "asctime",
              "level": "levelname",
              "logger": "name",
              "message": "message"
          }
      }
    },
    "handlers": {
      // Other handlers? 
      "jsonStream": {
          "class": "logging.StreamHandler",
          "formatter": "json"
      }
    },
    "root": {
      "handlers": [
        // .. 
        "jsonStream",
        
      ]
    }

  }
}
```

---

## File: backtesting.md

# Backtesting

This page explains how to validate your strategy performance by using Backtesting.

Backtesting requires historic data to be available.
To learn how to get data for the pairs and exchange you're interested in, head over to the [Data Downloading](data-download.md) section of the documentation.

## Backtesting command reference

--8<-- "commands/backtesting.md"

## Test your strategy with Backtesting

Now you have good Entry and exit strategies and some historic data, you want to test it against
real data. This is what we call [backtesting](https://en.wikipedia.org/wiki/Backtesting).

Backtesting will use the crypto-currencies (pairs) from your config file and load historical candle (OHLCV) data from `user_data/data/<exchange>` by default.
If no data is available for the exchange / pair / timeframe combination, backtesting will ask you to download them first using `freqtrade download-data`.
For details on downloading, please refer to the [Data Downloading](data-download.md) section in the documentation.

The result of backtesting will confirm if your bot has better odds of making a profit than a loss.

All profit calculations include fees, and freqtrade will use the exchange's default fees for the calculation.

!!! Warning "Using dynamic pairlists for backtesting"
    Using dynamic pairlists is possible (not all of the handlers are allowed to be used in backtest mode), however it relies on the current market conditions - which will not reflect the historic status of the pairlist.
    Also, when using pairlists other than StaticPairlist, reproducibility of backtesting-results cannot be guaranteed.
    Please read the [pairlists documentation](plugins.md#pairlists) for more information.

    To achieve reproducible results, best generate a pairlist via the [`test-pairlist`](utils.md#test-pairlist) command and use that as static pairlist.

!!! Note
    By default, Freqtrade will export backtesting results to `user_data/backtest_results`.
    The exported trades can be used for [further analysis](#further-backtest-result-analysis) or can be used by the [plotting sub-command](plotting.md#plot-price-and-indicators) (`freqtrade plot-dataframe`) in the scripts directory.


### Starting balance

Backtesting will require a starting balance, which can be provided as `--dry-run-wallet <balance>` or `--starting-balance <balance>` command line argument, or via `dry_run_wallet` configuration setting.
This amount must be higher than `stake_amount`, otherwise the bot will not be able to simulate any trade.

### Dynamic stake amount

Backtesting supports [dynamic stake amount](configuration.md#dynamic-stake-amount) by configuring `stake_amount` as `"unlimited"`, which will split the starting balance into `max_open_trades` pieces.
Profits from early trades will result in subsequent higher stake amounts, resulting in compounding of profits over the backtesting period.

### Example backtesting commands

With 5 min candle (OHLCV) data (per default)

```bash
freqtrade backtesting --strategy AwesomeStrategy
```

Where `--strategy AwesomeStrategy` / `-s AwesomeStrategy` refers to the class name of the strategy, which is within a python file in the `user_data/strategies` directory.

---

With 1 min candle (OHLCV) data

```bash
freqtrade backtesting --strategy AwesomeStrategy --timeframe 1m
```

---

Providing a custom starting balance of 1000 (in stake currency)

```bash
freqtrade backtesting --strategy AwesomeStrategy --dry-run-wallet 1000
```

---

Using a different on-disk historical candle (OHLCV) data source

Assume you downloaded the history data from the Binance exchange and kept it in the `user_data/data/binance-20180101` directory. 
You can then use this data for backtesting as follows:

```bash
freqtrade backtesting --strategy AwesomeStrategy --datadir user_data/data/binance-20180101 
```

---

Comparing multiple Strategies

```bash
freqtrade backtesting --strategy-list SampleStrategy1 AwesomeStrategy --timeframe 5m
```

Where `SampleStrategy1` and `AwesomeStrategy` refer to class names of strategies.

---

Prevent exporting trades to file

```bash
freqtrade backtesting --strategy backtesting --export none --config config.json 
```

Only use this if you're sure you'll not want to plot or analyze your results further.

---

Exporting trades to file specifying a custom filename

```bash
freqtrade backtesting --strategy backtesting --export trades --export-filename=backtest_samplestrategy.json
```

Please also read about the [strategy startup period](strategy-customization.md#strategy-startup-period).

---

Supplying custom fee value

Sometimes your account has certain fee rebates (fee reductions starting with a certain account size or monthly volume), which are not visible to ccxt.
To account for this in backtesting, you can use the `--fee` command line option to supply this value to backtesting.
This fee must be a ratio, and will be applied twice (once for trade entry, and once for trade exit).

For example, if the commission fee per order is 0.1% (i.e., 0.001 written as ratio), then you would run backtesting as the following:

```bash
freqtrade backtesting --fee 0.001
```

!!! Note
    Only supply this option (or the corresponding configuration parameter) if you want to experiment with different fee values. By default, Backtesting fetches the default fee from the exchange pair/market info.

---

Running backtest with smaller test-set by using timerange

Use the `--timerange` argument to change how much of the test-set you want to use.

For example, running backtesting with the `--timerange=20190501-` option will use all available data starting with May 1st, 2019 from your input data.

```bash
freqtrade backtesting --timerange=20190501-
```

You can also specify particular date ranges.

The full timerange specification:

- Use data until 2018/01/31: `--timerange=-20180131`
- Use data since 2018/01/31: `--timerange=20180131-`
- Use data since 2018/01/31 till 2018/03/01 : `--timerange=20180131-20180301`
- Use data between POSIX / epoch timestamps 1527595200 1527618600: `--timerange=1527595200-1527618600`

## Understand the backtesting result

The most important in the backtesting is to understand the result.

A backtesting result will look like that:

```
================================================ BACKTESTING REPORT =================================================
| Pair     | Trades |   Avg Profit % |   Tot Profit BTC |   Tot Profit % | Avg Duration |  Wins Draws Loss   Win%  |
|----------+--------+----------------+------------------+----------------+--------------+--------------------------|
| ADA/BTC  |     35 |          -0.11 |      -0.00019428 |          -1.94 | 4:35:00      |    14     0    21   40.0 |
| ARK/BTC  |     11 |          -0.41 |      -0.00022647 |          -2.26 | 2:03:00      |     3     0     8   27.3 |
| BTS/BTC  |     32 |           0.31 |       0.00048938 |           4.89 | 5:05:00      |    18     0    14   56.2 |
| DASH/BTC |     13 |          -0.08 |      -0.00005343 |          -0.53 | 4:39:00      |     6     0     7   46.2 |
| ENG/BTC  |     18 |           1.36 |       0.00122807 |          12.27 | 2:50:00      |     8     0    10   44.4 |
| EOS/BTC  |     36 |           0.08 |       0.00015304 |           1.53 | 3:34:00      |    16     0    20   44.4 |
| ETC/BTC  |     26 |           0.37 |       0.00047576 |           4.75 | 6:14:00      |    11     0    15   42.3 |
| ETH/BTC  |     33 |           0.30 |       0.00049856 |           4.98 | 7:31:00      |    16     0    17   48.5 |
| IOTA/BTC |     32 |           0.03 |       0.00005444 |           0.54 | 3:12:00      |    14     0    18   43.8 |
| LSK/BTC  |     15 |           1.75 |       0.00131413 |          13.13 | 2:58:00      |     6     0     9   40.0 |
| LTC/BTC  |     32 |          -0.04 |      -0.00006886 |          -0.69 | 4:49:00      |    11     0    21   34.4 |
| NANO/BTC |     17 |           1.26 |       0.00107058 |          10.70 | 1:55:00      |    10     0     7   58.5 |
| NEO/BTC  |     23 |           0.82 |       0.00094936 |           9.48 | 2:59:00      |    10     0    13   43.5 |
| REQ/BTC  |      9 |           1.17 |       0.00052734 |           5.27 | 3:47:00      |     4     0     5   44.4 |
| XLM/BTC  |     16 |           1.22 |       0.00097800 |           9.77 | 3:15:00      |     7     0     9   43.8 |
| XMR/BTC  |     23 |          -0.18 |      -0.00020696 |          -2.07 | 5:30:00      |    12     0    11   52.2 |
| XRP/BTC  |     35 |           0.66 |       0.00114897 |          11.48 | 3:49:00      |    12     0    23   34.3 |
| ZEC/BTC  |     22 |          -0.46 |      -0.00050971 |          -5.09 | 2:22:00      |     7     0    15   31.8 |
| TOTAL    |    429 |           0.36 |       0.00762792 |          76.20 | 4:12:00      |   186     0   243   43.4 |
============================================= LEFT OPEN TRADES REPORT =============================================
| Pair     |  Trades |   Avg Profit % |   Tot Profit BTC |   Tot Profit % | Avg Duration   |  Win Draw Loss Win% |
|----------+---------+----------------+------------------+----------------+----------------+---------------------|
| ADA/BTC  |       1 |           0.89 |       0.00004434 |           0.44 | 6:00:00        |    1    0    0  100 |
| LTC/BTC  |       1 |           0.68 |       0.00003421 |           0.34 | 2:00:00        |    1    0    0  100 |
| TOTAL    |       2 |           0.78 |       0.00007855 |           0.78 | 4:00:00        |    2    0    0  100 |
==================== EXIT REASON STATS ====================
| Exit Reason        |   Exits |  Wins |  Draws |  Losses |
|--------------------+---------+-------+--------+---------|
| trailing_stop_loss |     205 |   150 |      0 |      55 |
| stop_loss          |     166 |     0 |      0 |     166 |
| exit_signal        |      56 |    36 |      0 |      20 |
| force_exit         |       2 |     0 |      0 |       2 |

================== SUMMARY METRICS ==================
| Metric                      | Value               |
|-----------------------------+---------------------|
| Backtesting from            | 2019-01-01 00:00:00 |
| Backtesting to              | 2019-05-01 00:00:00 |
| Trading Mode                | Spot                |
| Max open trades             | 3                   |
|                             |                     |
| Total/Daily Avg Trades      | 429 / 3.575         |
| Starting balance            | 0.01000000 BTC      |
| Final balance               | 0.01762792 BTC      |
| Absolute profit             | 0.00762792 BTC      |
| Total profit %              | 76.2%               |
| CAGR %                      | 460.87%             |
| Sortino                     | 1.88                |
| Sharpe                      | 2.97                |
| Calmar                      | 6.29                |
| SQN                         | 2.45                |
| Profit factor               | 1.11                |
| Expectancy (Ratio)          | -0.15 (-0.05)       |
| Avg. stake amount           | 0.001      BTC      |
| Total trade volume          | 0.429      BTC      |
|                             |                     |
| Long / Short                | 352 / 77            |
| Total profit Long %         | 1250.58%            |
| Total profit Short %        | -15.02%             |
| Absolute profit Long        | 0.00838792 BTC      |
| Absolute profit Short       | -0.00076 BTC        |
|                             |                     |
| Best Pair                   | LSK/BTC 26.26%      |
| Worst Pair                  | ZEC/BTC -10.18%     |
| Best Trade                  | LSK/BTC 4.25%       |
| Worst Trade                 | ZEC/BTC -10.25%     |
| Best day                    | 0.00076 BTC         |
| Worst day                   | -0.00036 BTC        |
| Days win/draw/lose          | 12 / 82 / 25        |
| Avg. Duration Winners       | 4:23:00             |
| Avg. Duration Loser         | 6:55:00             |
| Max Consecutive Wins / Loss | 3 / 4               |
| Rejected Entry signals      | 3089                |
| Entry/Exit Timeouts         | 0 / 0               |
| Canceled Trade Entries      | 34                  |
| Canceled Entry Orders       | 123                 |
| Replaced Entry Orders       | 89                  |
|                             |                     |
| Min balance                 | 0.00945123 BTC      |
| Max balance                 | 0.01846651 BTC      |
| Max % of account underwater | 25.19%              |
| Absolute Drawdown (Account) | 13.33%              |
| Drawdown                    | 0.0015 BTC          |
| Drawdown high               | 0.0013 BTC          |
| Drawdown low                | -0.0002 BTC         |
| Drawdown Start              | 2019-02-15 14:10:00 |
| Drawdown End                | 2019-04-11 18:15:00 |
| Market change               | -5.88%              |
=====================================================
```

### Backtesting report table

The 1st table contains all trades the bot made, including "left open trades".

The last line will give you the overall performance of your strategy,
here:

```
| TOTAL    |    429 |           0.36 |         152.41 |       0.00762792 |          76.20 | 4:12:00      |   186     0   243   43.4 |
```

The bot has made `429` trades for an average duration of `4:12:00`, with a performance of `76.20%` (profit), that means it has
earned a total of `0.00762792 BTC` starting with a capital of 0.01 BTC.

The column `Avg Profit %` shows the average profit for all trades made.
The column `Tot Profit %` shows instead the total profit % in relation to the starting balance.
In the above results, we have a starting balance of 0.01 BTC and an absolute profit of 0.00762792 BTC - so the `Tot Profit %` will be `(0.00762792 / 0.01) * 100 ~= 76.2%`.

Your strategy performance is influenced by your entry strategy, your exit strategy, and also by the `minimal_roi` and `stop_loss` you have set.

For example, if your `minimal_roi` is only `"0":  0.01` you cannot expect the bot to make more profit than 1% (because it will exit every time a trade reaches 1%).

```json
"minimal_roi": {
    "0":  0.01
},
```

On the other hand, if you set a too high `minimal_roi` like `"0":  0.55`
(55%), there is almost no chance that the bot will ever reach this profit.
Hence, keep in mind that your performance is an integral mix of all different elements of the strategy, your configuration, and the crypto-currency pairs you have set up.

### Exit reasons table

The 2nd table contains a recap of exit reasons.
This table can tell you which area needs some additional work (e.g. all or many of the `exit_signal` trades are losses, so you should work on improving the exit signal, or consider disabling it).

### Left open trades table

The 3rd table contains all trades the bot had to `force_exit` at the end of the backtesting period to present you the full picture.
This is necessary to simulate realistic behavior, since the backtest period has to end at some point, while realistically, you could leave the bot running forever.
These trades are also included in the first table, but are also shown separately in this table for clarity.

### Summary metrics

The last element of the backtest report is the summary metrics table.
It contains some useful key metrics about performance of your strategy on backtesting data.

```
================== SUMMARY METRICS ==================
| Metric                      | Value               |
|-----------------------------+---------------------|
| Backtesting from            | 2019-01-01 00:00:00 |
| Backtesting to              | 2019-05-01 00:00:00 |
| Trading Mode                | Spot                |
| Max open trades             | 3                   |
|                             |                     |
| Total/Daily Avg Trades      | 429 / 3.575         |
| Starting balance            | 0.01000000 BTC      |
| Final balance               | 0.01762792 BTC      |
| Absolute profit             | 0.00762792 BTC      |
| Total profit %              | 76.2%               |
| CAGR %                      | 460.87%             |
| Sortino                     | 1.88                |
| Sharpe                      | 2.97                |
| Calmar                      | 6.29                |
| SQN                         | 2.45                |
| Profit factor               | 1.11                |
| Expectancy (Ratio)          | -0.15 (-0.05)       |
| Avg. stake amount           | 0.001      BTC      |
| Total trade volume          | 0.429      BTC      |
|                             |                     |
| Long / Short                | 352 / 77            |
| Total profit Long %         | 1250.58%            |
| Total profit Short %        | -15.02%             |
| Absolute profit Long        | 0.00838792 BTC      |
| Absolute profit Short       | -0.00076 BTC        |
|                             |                     |
| Best Pair                   | LSK/BTC 26.26%      |
| Worst Pair                  | ZEC/BTC -10.18%     |
| Best Trade                  | LSK/BTC 4.25%       |
| Worst Trade                 | ZEC/BTC -10.25%     |
| Best day                    | 0.00076 BTC         |
| Worst day                   | -0.00036 BTC        |
| Days win/draw/lose          | 12 / 82 / 25        |
| Avg. Duration Winners       | 4:23:00             |
| Avg. Duration Loser         | 6:55:00             |
| Max Consecutive Wins / Loss | 3 / 4               |
| Rejected Entry signals      | 3089                |
| Entry/Exit Timeouts         | 0 / 0               |
| Canceled Trade Entries      | 34                  |
| Canceled Entry Orders       | 123                 |
| Replaced Entry Orders       | 89                  |
|                             |                     |
| Min balance                 | 0.00945123 BTC      |
| Max balance                 | 0.01846651 BTC      |
| Max % of account underwater | 25.19%              |
| Absolute Drawdown (Account) | 13.33%              |
| Drawdown                    | 0.0015 BTC          |
| Drawdown high               | 0.0013 BTC          |
| Drawdown low                | -0.0002 BTC         |
| Drawdown Start              | 2019-02-15 14:10:00 |
| Drawdown End                | 2019-04-11 18:15:00 |
| Market change               | -5.88%              |
=====================================================

```

- `Backtesting from` / `Backtesting to`: Backtesting range (usually defined with the `--timerange` option).
- `Max open trades`: Setting of `max_open_trades` (or `--max-open-trades`) - or number of pairs in the pairlist (whatever is lower).
- `Trading Mode`: Spot or Futures trading.
- `Total/Daily Avg Trades`: Identical to the total trades of the backtest output table / Total trades divided by the backtesting duration in days (this will give you information about how many trades to expect from the strategy).
- `Starting balance`: Start balance - as given by dry-run-wallet (config or command line).
- `Final balance`: Final balance - starting balance + absolute profit.
- `Absolute profit`: Profit made in stake currency.
- `Total profit %`: Total profit. Aligned to the `TOTAL` row's `Tot Profit %` from the first table. Calculated as `(End capital − Starting capital) / Starting capital`.
- `CAGR %`: Compound annual growth rate.
- `Sortino`: Annualized Sortino ratio.
- `Sharpe`: Annualized Sharpe ratio.
- `Calmar`: Annualized Calmar ratio.
- `SQN`: System Quality Number (SQN) - by Van Tharp.
- `Profit factor`: profit / loss.
- `Avg. stake amount`: Average stake amount, either `stake_amount` or the average when using dynamic stake amount.
- `Total trade volume`: Volume generated on the exchange to reach the above profit.
- `Best Pair` / `Worst Pair`: Best and worst performing pair, and it's corresponding `Tot Profit %`.
- `Best Trade` / `Worst Trade`: Biggest single winning trade and biggest single losing trade.
- `Best day` / `Worst day`: Best and worst day based on daily profit.
- `Days win/draw/lose`: Winning / Losing days (draws are usually days without closed trade).
- `Avg. Duration Winners` / `Avg. Duration Loser`: Average durations for winning and losing trades.
- `Max Consecutive Wins / Loss`: Maximum consecutive wins/losses in a row.
- `Rejected Entry signals`: Trade entry signals that could not be acted upon due to `max_open_trades` being reached.
- `Entry/Exit Timeouts`: Entry/exit orders which did not fill (only applicable if custom pricing is used).
- `Canceled Trade Entries`: Number of trades that have been canceled by user request via `adjust_entry_price`.
- `Canceled Entry Orders`: Number of entry orders that have been canceled by user request via `adjust_entry_price`.
- `Replaced Entry Orders`: Number of entry orders that have been replaced by user request via `adjust_entry_price`.
- `Min balance` / `Max balance`: Lowest and Highest Wallet balance during the backtest period.
- `Max % of account underwater`: Maximum percentage your account has decreased from the top since the simulation started.
Calculated as the maximum of `(Max Balance - Current Balance) / (Max Balance)`.
- `Absolute Drawdown (Account)`: Maximum Account Drawdown experienced. Calculated as `(Absolute Drawdown) / (DrawdownHigh + startingBalance)`.
- `Drawdown`: Maximum, absolute drawdown experienced. Difference between Drawdown High and Subsequent Low point.
- `Drawdown high` / `Drawdown low`: Profit at the beginning and end of the largest drawdown period. A negative low value means initial capital lost.
- `Drawdown Start` / `Drawdown End`: Start and end datetime for this largest drawdown (can also be visualized via the `plot-dataframe` sub-command).
- `Market change`: Change of the market during the backtest period. Calculated as average of all pairs changes from the first to the last candle using the "close" column.
- `Long / Short`: Split long/short values (Only shown when short trades were made).
- `Total profit Long %` / `Absolute profit Long`: Profit long trades only (Only shown when short trades were made).
- `Total profit Short %` / `Absolute profit Short`: Profit short trades only (Only shown when short trades were made).

### Daily / Weekly / Monthly breakdown

You can get an overview over daily / weekly or monthly results by using the `--breakdown <>` switch.

To visualize daily and weekly breakdowns, you can use the following:

``` bash
freqtrade backtesting --strategy MyAwesomeStrategy --breakdown day week
```

``` output
======================== DAY BREAKDOWN =========================
|        Day |   Tot Profit USDT |   Wins |   Draws |   Losses |
|------------+-------------------+--------+---------+----------|
| 03/07/2021 |           200.0   |      2 |       0 |        0 |
| 04/07/2021 |           -50.31  |      0 |       0 |        2 |
| 05/07/2021 |           220.611 |      3 |       2 |        0 |
| 06/07/2021 |           150.974 |      3 |       0 |        2 |
| 07/07/2021 |           -70.193 |      1 |       0 |        2 |
| 08/07/2021 |           212.413 |      2 |       0 |        3 |

```

The output will show a table containing the realized absolute Profit (in stake currency) for the given timeperiod, as well as wins, draws and losses that materialized (closed) on this day. Below that there will be a second table for the summarized values of weeks indicated by the date of the closing Sunday. The same would apply to a monthly breakdown indicated by the last day of the month.

### Backtest result caching

To save time, by default backtest will reuse a cached result from within the last day when the backtested strategy and config match that of a previous backtest. To force a new backtest despite existing result for an identical run specify `--cache none` parameter.

!!! Warning
    Caching is automatically disabled for open-ended timeranges (`--timerange 20210101-`), as freqtrade cannot ensure reliably that the underlying data didn't change. It can also use cached results where it shouldn't if the original backtest had missing data at the end, which was fixed by downloading more data.
    In this instance, please use `--cache none` once to force a fresh backtest.

### Further backtest-result analysis

To further analyze your backtest results, freqtrade will export the trades to file by default.
You can then load the trades to perform further analysis as shown in the [data analysis](strategy_analysis_example.md#load-backtest-results-to-pandas-dataframe) backtesting section.

### Backtest output file

The output file freqtrade produces is a zip file containing the following files:

- The backtest report in json format
- the market change data in feather format
- a copy of the strategy file
- a copy of the strategy parameters (if a parameter file was used)
- a sanitized copy of the config file

This will ensure results are reproducible - under the assumption that the same data is available.

Only the strategy file and the config file are included in the zip file, eventual dependencies are not included.

## Assumptions made by backtesting

Since backtesting lacks some detailed information about what happens within a candle, it needs to take a few assumptions:

- Exchange [trading limits](#trading-limits-in-backtesting) are respected
- Entries happen at open-price unless a custom price logic has been specified
- All orders are filled at the requested price (no slippage) as long as the price is within the candle's high/low range
- Exit-signal exits happen at open-price of the consecutive candle
- Exits free their trade slot for a new trade with a different pair
- Exit-signal is favored over Stoploss, because exit-signals are assumed to trigger on candle's open
- ROI
  - Exits are compared to high - but the ROI value is used (e.g. ROI = 2%, high=5% - so the exit will be at 2%)
  - Exits are never "below the candle", so a ROI of 2% may result in a exit at 2.4% if low was at 2.4% profit
  - ROI entries which came into effect on the triggering candle (e.g. `120: 0.02` for 1h candles, from `60: 0.05`) will use the candle's open as exit rate
  - Force-exits caused by `<N>=-1` ROI entries use low as exit value, unless N falls on the candle open (e.g. `120: -1` for 1h candles)
- Stoploss exits happen exactly at stoploss price, even if low was lower, but the loss will be `2 * fees` higher than the stoploss price
- Stoploss is evaluated before ROI within one candle. So you can often see more trades with the `stoploss` exit reason comparing to the results obtained with the same strategy in the Dry Run/Live Trade modes
- Low happens before high for stoploss, protecting capital first
- Trailing stoploss
  - Trailing Stoploss is only adjusted if it's below the candle's low (otherwise it would be triggered)
  - On trade entry candles that trigger trailing stoploss, the "minimum offset" (`stop_positive_offset`) is assumed (instead of high) - and the stop is calculated from this point. This rule is NOT applicable to custom-stoploss scenarios, since there's no information about the stoploss logic available.
  - High happens first - adjusting stoploss
  - Low uses the adjusted stoploss (so exits with large high-low difference are backtested correctly)
  - ROI applies before trailing-stop, ensuring profits are "top-capped" at ROI if both ROI and trailing stop applies
- Exit-reason does not explain if a trade was positive or negative, just what triggered the exit (this can look odd if negative ROI values are used)
- Evaluation sequence (if multiple signals happen on the same candle)
  - Exit-signal
  - Stoploss
  - ROI
  - Trailing stoploss
- Position reversals (futures only) happen if an entry signal in the other direction than the closing trade triggers at the candle the existing trade closes.

Taking these assumptions, backtesting tries to mirror real trading as closely as possible. However, backtesting will **never** replace running a strategy in dry-run mode.
Also, keep in mind that past results don't guarantee future success.

In addition to the above assumptions, strategy authors should carefully read the [Common Mistakes](strategy-customization.md#common-mistakes-when-developing-strategies) section, to avoid using data in backtesting which is not available in real market conditions.

### Trading limits in backtesting

Exchanges have certain trading limits, like minimum (and maximum) base currency, or minimum/maximum stake (quote) currency.
These limits are usually listed in the exchange documentation as "trading rules" or similar and can be quite different between different pairs.

Backtesting (as well as live and dry-run) does honor these limits, and will ensure that a stoploss can be placed below this value - so the value will be slightly higher than what the exchange specifies.
Freqtrade has however no information about historic limits.

This can lead to situations where trading-limits are inflated by using a historic price, resulting in minimum amounts > 50\$.

For example:

BTC minimum tradable amount is 0.001.
BTC trades at 22.000\$ today (0.001 BTC is related to this) - but the backtesting period includes prices as high as 50.000\$.
Today's minimum would be `0.001 * 22_000` - or 22\$.  
However the limit could also be 50$ - based on `0.001 * 50_000` in some historic setting.

#### Trading precision limits

Most exchanges pose precision limits on both price and amounts, so you cannot buy 1.0020401 of a pair, or at a price of 1.24567123123.  
Instead, these prices and amounts will be rounded or truncated (based on the exchange definition) to the defined trading precision.
The above values may for example be rounded to an amount of 1.002, and a price of 1.24567.

These precision values are based on current exchange limits (as described in the [above section](#trading-limits-in-backtesting)), as historic precision limits are not available.

## Improved backtest accuracy

One big limitation of backtesting is it's inability to know how prices moved intra-candle (was high before close, or vice-versa?).
So assuming you run backtesting with a 1h timeframe, there will be 4 prices for that candle (Open, High, Low, Close).

While backtesting does take some assumptions (read above) about this - this can never be perfect, and will always be biased in one way or the other.
To mitigate this, freqtrade can use a lower (faster) timeframe to simulate intra-candle movements.

To utilize this, you can append `--timeframe-detail 5m` to your regular backtesting command.

``` bash
freqtrade backtesting --strategy AwesomeStrategy --timeframe 1h --timeframe-detail 5m
```

This will load 1h data (the main timeframe) as well as 5m data (detail timeframe) for the selected timerange.
The strategy will be analyzed with the 1h timeframe.
Candles where activity may take place (there's an active signal, the pair is in a trade) are  evaluated at the 5m timeframe.
This will allow for a more accurate simulation of intra-candle movements - and can lead to different results, especially on higher timeframes.

Entries will generally still happen at the main candle's open, however freed trade slots may be freed earlier (if the exit signal is triggered on the 5m candle), which can then be used for a new trade of a different pair.

All callback functions (`custom_exit()`, `custom_stoploss()`, ... ) will be running for each 5m candle once the trade is opened (so 12 times in the above example of 1h timeframe, and 5m detailed timeframe).

`--timeframe-detail` must be smaller than the original timeframe, otherwise backtesting will fail to start.

Obviously this will require more memory (5m data is bigger than 1h data), and will also impact runtime (depending on the amount of trades and trade durations).
Also, data must be available / downloaded already.

!!! Tip
    You can use this function as the last part of strategy development, to ensure your strategy is not exploiting one of the [backtesting assumptions](#assumptions-made-by-backtesting). Strategies that perform similarly well with this mode have a good chance to perform well in dry/live modes too (although only forward-testing (dry-mode) can really confirm a strategy).

??? Sample "Extreme Difference Example"
    Using `--timeframe-detail` on an extreme example (all below pairs have the 10:00 candle with an entry signal) may lead to the following backtesting Trade sequence with 1 max_open_trades:

    | Pair | Entry Time | Exit Time | Duration |
    |------|------------|-----------| -------- |
    | BTC/USDT | 2024-01-01 10:00:00 | 2021-01-01 10:05:00 | 5m |
    | ETH/USDT | 2024-01-01 10:05:00 | 2021-01-01 10:15:00 | 10m |
    | XRP/USDT | 2024-01-01 10:15:00 | 2021-01-01 10:30:00 | 15m |
    | SOL/USDT | 2024-01-01 10:15:00 | 2021-01-01 11:05:00 | 50m |
    | BTC/USDT | 2024-01-01 11:05:00 | 2021-01-01 12:00:00 | 55m |

    Without timeframe-detail, this would look like:

    | Pair | Entry Time | Exit Time | Duration |
    |------|------------|-----------| -------- |
    | BTC/USDT | 2024-01-01 10:00:00 | 2021-01-01 11:00:00 | 1h |
    | BTC/USDT | 2024-01-01 11:00:00 | 2021-01-01 12:00:00 | 1h |

    The difference is significant, as without detail data, only the first `max_open_trades` signals per candle are evaluated, and the trade slots are only freed at the end of the candle, allowing for a new trade to be opened at the next candle.


## Backtesting multiple strategies

To compare multiple strategies, a list of Strategies can be provided to backtesting.

This is limited to 1 timeframe value per run. However, data is only loaded once from disk so if you have multiple
strategies you'd like to compare, this will give a nice runtime boost.

All listed Strategies need to be in the same directory, unless also `--recursive-strategy-search` is specified, where sub-directories within the strategy directory are also considered.

``` bash
freqtrade backtesting --timerange 20180401-20180410 --timeframe 5m --strategy-list Strategy001 Strategy002 --export trades
```

This will save the results to `user_data/backtest_results/backtest-result-<datetime>.json`, including results for both `Strategy001` and `Strategy002`.
There will be an additional table comparing win/losses of the different strategies (identical to the "Total" row in the first table).
Detailed output for all strategies one after the other will be available, so make sure to scroll up to see the details per strategy.

```
================================================== STRATEGY SUMMARY ===================================================================
| Strategy    |  Trades |   Avg Profit % |   Tot Profit BTC |   Tot Profit % | Avg Duration   |  Wins |  Draws | Losses | Drawdown % |
|-------------+---------+----------------+------------------+----------------+----------------+-------+--------+--------+------------|
| Strategy1   |     429 |           0.36 |       0.00762792 |          76.20 | 4:12:00        |   186 |      0 |    243 |       45.2 |
| Strategy2   |    1487 |          -0.13 |      -0.00988917 |         -98.79 | 4:43:00        |   662 |      0 |    825 |     241.68 |
```

## Next step

Great, your strategy is profitable. What if the bot can give your the optimal parameters to use for your strategy?
Your next step is to learn [how to find optimal parameters with Hyperopt](hyperopt.md)

---

## File: bot-basics.md

# Freqtrade basics

This page provides you some basic concepts on how Freqtrade works and operates.

## Freqtrade terminology

* **Strategy**: Your trading strategy, telling the bot what to do.
* **Trade**: Open position.
* **Open Order**: Order which is currently placed on the exchange, and is not yet complete.
* **Pair**: Tradable pair, usually in the format of Base/Quote (e.g. `XRP/USDT` for spot, `XRP/USDT:USDT` for futures).
* **Timeframe**: Candle length to use (e.g. `"5m"`, `"1h"`, ...).
* **Indicators**: Technical indicators (SMA, EMA, RSI, ...).
* **Limit order**: Limit orders which execute at the defined limit price or better.
* **Market order**: Guaranteed to fill, may move price depending on the order size.
* **Current Profit**: Currently pending (unrealized) profit for this trade. This is mainly used throughout the bot and UI.
* **Realized Profit**: Already realized profit. Only relevant in combination with [partial exits](strategy-callbacks.md#adjust-trade-position) - which also explains the calculation logic for this.
* **Total Profit**: Combined realized and unrealized profit. The relative number (%) is calculated against the total investment in this trade.

## Fee handling

All profit calculations of Freqtrade include fees. For Backtesting / Hyperopt / Dry-run modes, the exchange default fee is used (lowest tier on the exchange). For live operations, fees are used as applied by the exchange (this includes BNB rebates etc.).

## Pair naming

Freqtrade follows the [ccxt naming convention](https://docs.ccxt.com/#/README?id=consistency-of-base-and-quote-currencies) for currencies.
Using the wrong naming convention in the wrong market will usually result in the bot not recognizing the pair, usually resulting in errors like "this pair is not available".

### Spot pair naming

For spot pairs, naming will be `base/quote` (e.g. `ETH/USDT`).

### Futures pair naming

For futures pairs, naming will be `base/quote:settle` (e.g. `ETH/USDT:USDT`).

## Bot execution logic

Starting freqtrade in dry-run or live mode (using `freqtrade trade`) will start the bot and start the bot iteration loop.
This will also run the `bot_start()` callback.

By default, the bot loop runs every few seconds (`internals.process_throttle_secs`) and performs the following actions:

* Fetch open trades from persistence.
* Calculate current list of tradable pairs.
* Download OHLCV data for the pairlist including all [informative pairs](strategy-customization.md#get-data-for-non-tradeable-pairs)  
  This step is only executed once per Candle to avoid unnecessary network traffic.
* Call `bot_loop_start()` strategy callback.
* Analyze strategy per pair.
  * Call `populate_indicators()`
  * Call `populate_entry_trend()`
  * Call `populate_exit_trend()`
* Update trades open order state from exchange.
  * Call `order_filled()` strategy callback for filled orders.
  * Check timeouts for open orders.
    * Calls `check_entry_timeout()` strategy callback for open entry orders.
    * Calls `check_exit_timeout()` strategy callback for open exit orders.
    * Calls `adjust_order_price()` strategy callback for open orders.
      * Calls `adjust_entry_price()` strategy callback for open entry orders. *only called when `adjust_order_price()` is not implemented*
      * Calls `adjust_exit_price()` strategy callback for open exit orders. *only called when `adjust_order_price()` is not implemented*
* Verifies existing positions and eventually places exit orders.
  * Considers stoploss, ROI and exit-signal, `custom_exit()` and `custom_stoploss()`.
  * Determine exit-price based on `exit_pricing` configuration setting or by using the `custom_exit_price()` callback.
  * Before an exit order is placed, `confirm_trade_exit()` strategy callback is called.
* Check position adjustments for open trades if enabled by calling `adjust_trade_position()` and place additional order if required.
* Check if trade-slots are still available (if `max_open_trades` is reached).
* Verifies entry signal trying to enter new positions.
  * Determine entry-price based on `entry_pricing` configuration setting, or by using the `custom_entry_price()` callback.
  * In Margin and Futures mode, `leverage()` strategy callback is called to determine the desired leverage.
  * Determine stake size by calling the `custom_stake_amount()` callback.
  * Before an entry order is placed, `confirm_trade_entry()` strategy callback is called.

This loop will be repeated again and again until the bot is stopped.

## Backtesting / Hyperopt execution logic

[backtesting](backtesting.md) or [hyperopt](hyperopt.md) do only part of the above logic, since most of the trading operations are fully simulated.

* Load historic data for configured pairlist.
* Calls `bot_start()` once.
* Calculate indicators (calls `populate_indicators()` once per pair).
* Calculate entry / exit signals (calls `populate_entry_trend()` and `populate_exit_trend()` once per pair).
* Loops per candle simulating entry and exit points.
  * Calls `bot_loop_start()` strategy callback.
  * Check for Order timeouts, either via the `unfilledtimeout` configuration, or via `check_entry_timeout()` / `check_exit_timeout()` strategy callbacks.
  * Calls `adjust_order_price()` strategy callback for open orders.
    * Calls `adjust_entry_price()` strategy callback for open entry orders. *only called when `adjust_order_price()` is not implemented!*
    * Calls `adjust_exit_price()` strategy callback for open exit orders. *only called when `adjust_order_price()` is not implemented!*
  * Check for trade entry signals (`enter_long` / `enter_short` columns).
  * Confirm trade entry / exits (calls `confirm_trade_entry()` and `confirm_trade_exit()` if implemented in the strategy).
  * Call `custom_entry_price()` (if implemented in the strategy) to determine entry price (Prices are moved to be within the opening candle).
  * In Margin and Futures mode, `leverage()` strategy callback is called to determine the desired leverage.
  * Determine stake size by calling the `custom_stake_amount()` callback.
  * Check position adjustments for open trades if enabled and call `adjust_trade_position()` to determine if an additional order is requested.
  * Call `order_filled()` strategy callback for filled entry orders.
  * Call `custom_stoploss()` and `custom_exit()` to find custom exit points.
  * For exits based on exit-signal, custom-exit and partial exits: Call `custom_exit_price()` to determine exit price (Prices are moved to be within the closing candle).
  * Call `order_filled()` strategy callback for filled exit orders.
* Generate backtest report output

!!! Note
    Both Backtesting and Hyperopt include exchange default Fees in the calculation. Custom fees can be passed to backtesting / hyperopt by specifying the `--fee` argument.

!!! Warning "Callback call frequency"
    Backtesting will call each callback at max. once per candle (`--timeframe-detail` modifies this behavior to once per detailed candle).
    Most callbacks will be called once per iteration in live (usually every ~5s) - which can cause backtesting mismatches.

---

## File: bot-usage.md

# Start the bot

This page explains the different parameters of the bot and how to run it.

!!! Note
    If you've used `setup.sh`, don't forget to activate your virtual environment (`source .venv/bin/activate`) before running freqtrade commands.

!!! Warning "Up-to-date clock"
    The clock on the system running the bot must be accurate, synchronized to a NTP server frequently enough to avoid problems with communication to the exchanges.

## Bot commands

--8<-- "commands/main.md"

### Bot trading commands

--8<-- "commands/trade.md"

### How to specify which configuration file be used?

The bot allows you to select which configuration file you want to use by means of
the `-c/--config` command line option:

```bash
freqtrade trade -c path/far/far/away/config.json
```

Per default, the bot loads the `config.json` configuration file from the current
working directory.

### How to use multiple configuration files?

The bot allows you to use multiple configuration files by specifying multiple
`-c/--config` options in the command line. Configuration parameters
defined in the latter configuration files override parameters with the same name
defined in the previous configuration files specified in the command line earlier.

For example, you can make a separate configuration file with your key and secret
for the Exchange you use for trading, specify default configuration file with
empty key and secret values while running in the Dry Mode (which does not actually
require them):

```bash
freqtrade trade -c ./config.json
```

and specify both configuration files when running in the normal Live Trade Mode:

```bash
freqtrade trade -c ./config.json -c path/to/secrets/keys.config.json
```

This could help you hide your private Exchange key and Exchange secret on you local machine
by setting appropriate file permissions for the file which contains actual secrets and, additionally,
prevent unintended disclosure of sensitive private data when you publish examples
of your configuration in the project issues or in the Internet.

See more details on this technique with examples in the documentation page on
[configuration](configuration.md).

### Where to store custom data

Freqtrade allows the creation of a user-data directory using `freqtrade create-userdir --userdir someDirectory`.
This directory will look as follows:

```
user_data/
├── backtest_results
├── data
├── hyperopts
├── hyperopt_results
├── plot
└── strategies
```

You can add the entry "user_data_dir" setting to your configuration, to always point your bot to this directory.
Alternatively, pass in `--userdir` to every command.
The bot will fail to start if the directory does not exist, but will create necessary subdirectories.

This directory should contain your custom strategies, custom hyperopts and hyperopt loss functions, backtesting historical data (downloaded using either backtesting command or the download script) and plot outputs.

It is recommended to use version control to keep track of changes to your strategies.

### How to use **--strategy**?

This parameter will allow you to load your custom strategy class.
To test the bot installation, you can use the `SampleStrategy` installed by the `create-userdir` subcommand (usually `user_data/strategy/sample_strategy.py`).

The bot will search your strategy file within `user_data/strategies`.
To use other directories, please read the next section about `--strategy-path`.

To load a strategy, simply pass the class name (e.g.: `CustomStrategy`) in this parameter.

**Example:**
In `user_data/strategies` you have a file `my_awesome_strategy.py` which has
a strategy class called `AwesomeStrategy` to load it:

```bash
freqtrade trade --strategy AwesomeStrategy
```

If the bot does not find your strategy file, it will display in an error
message the reason (File not found, or errors in your code).

Learn more about strategy file in
[Strategy Customization](strategy-customization.md).

### How to use **--strategy-path**?

This parameter allows you to add an additional strategy lookup path, which gets
checked before the default locations (The passed path must be a directory!):

```bash
freqtrade trade --strategy AwesomeStrategy --strategy-path /some/directory
```

#### How to install a strategy?

This is very simple. Copy paste your strategy file into the directory
`user_data/strategies` or use `--strategy-path`. And voila, the bot is ready to use it.

### How to use **--db-url**?

When you run the bot in Dry-run mode, per default no transactions are
stored in a database. If you want to store your bot actions in a DB
using `--db-url`. This can also be used to specify a custom database
in production mode. Example command:

```bash
freqtrade trade -c config.json --db-url sqlite:///tradesv3.dry_run.sqlite
```

## Next step

The optimal strategy of the bot will change with time depending of the market trends. The next step is to
[Strategy Customization](strategy-customization.md).

---

## File: configuration.md

# Configure the bot

Freqtrade has many configurable features and possibilities.
By default, these settings are configured via the configuration file (see below).

## The Freqtrade configuration file

The bot uses a set of configuration parameters during its operation that all together conform to the bot configuration. It normally reads its configuration from a file (Freqtrade configuration file).

Per default, the bot loads the configuration from the `config.json` file, located in the current working directory.

You can specify a different configuration file used by the bot with the `-c/--config` command-line option.

If you used the [Quick start](docker_quickstart.md#docker-quick-start) method for installing
the bot, the installation script should have already created the default configuration file (`config.json`) for you.

If the default configuration file is not created we recommend to use `freqtrade new-config --config user_data/config.json` to generate a basic configuration file.

The Freqtrade configuration file is to be written in JSON format.

Additionally to the standard JSON syntax, you may use one-line `// ...` and multi-line `/* ... */` comments in your configuration files and trailing commas in the lists of parameters.

Do not worry if you are not familiar with JSON format -- simply open the configuration file with an editor of your choice, make some changes to the parameters you need, save your changes and, finally, restart the bot or, if it was previously stopped, run it again with the changes you made to the configuration. The bot validates the syntax of the configuration file at startup and will warn you if you made any errors editing it, pointing out problematic lines.

### Environment variables

Set options in the Freqtrade configuration via environment variables.
This takes priority over the corresponding value in configuration or strategy.

Environment variables must be prefixed with  `FREQTRADE__` to be loaded to the freqtrade configuration.

`__` serves as level separator, so the format used should correspond to `FREQTRADE__{section}__{key}`.
As such - an environment variable defined as  `export FREQTRADE__STAKE_AMOUNT=200` would result in `{stake_amount: 200}`.

A more complex example might be `export FREQTRADE__EXCHANGE__KEY=<yourExchangeKey>` to keep your exchange key secret. This will move the value to the `exchange.key` section of the configuration.
Using this scheme, all configuration settings will also be available as environment variables.

Please note that Environment variables will overwrite corresponding settings in your configuration, but command line Arguments will always win.

Common example:

``` bash
FREQTRADE__TELEGRAM__CHAT_ID=<telegramchatid>
FREQTRADE__TELEGRAM__TOKEN=<telegramToken>
FREQTRADE__EXCHANGE__KEY=<yourExchangeKey>
FREQTRADE__EXCHANGE__SECRET=<yourExchangeSecret>
```

Json lists are parsed as json - so you can use the following to set a list of pairs:

``` bash
export FREQTRADE__EXCHANGE__PAIR_WHITELIST='["BTC/USDT", "ETH/USDT"]'
```

!!! Note
    Environment variables detected are logged at startup - so if you can't find why a value is not what you think it should be based on the configuration, make sure it's not loaded from an environment variable.

!!! Tip "Validate combined result"
    You can use the [show-config subcommand](utils.md#show-config) to see the final, combined configuration.

??? Warning "Loading sequence"
    Environment variables are loaded after the initial configuration. As such, you cannot provide the path to the configuration through environment variables. Please use `--config path/to/config.json` for that.
    This also applies to `user_dir` to some degree. while the user directory can be set through environment variables - the configuration will **not** be loaded from that location.

### Multiple configuration files

Multiple configuration files can be specified and used by the bot or the bot can read its configuration parameters from the process standard input stream.

You can specify additional configuration files in `add_config_files`. Files specified in this parameter will be loaded and merged with the initial config file. The files are resolved relative to the initial configuration file.
This is similar to using multiple `--config` parameters, but simpler in usage as you don't have to specify all files for all commands.

!!! Tip "Validate combined result"
    You can use the [show-config subcommand](utils.md#show-config) to see the final, combined configuration.

!!! Tip "Use multiple configuration files to keep secrets secret"
    You can use a 2nd configuration file containing your secrets. That way you can share your "primary" configuration file, while still keeping your API keys for yourself.
    The 2nd file should only specify what you intend to override.
    If a key is in more than one of the configurations, then the "last specified configuration" wins (in the above example, `config-private.json`).

    For one-off commands, you can also use the below syntax by specifying multiple "--config" parameters.

    ``` bash
    freqtrade trade --config user_data/config1.json --config user_data/config-private.json <...>
    ```

    The below is equivalent to the example above - but having 2 configuration files in the configuration, for easier reuse.

    ``` json title="user_data/config.json"
    "add_config_files": [
        "config1.json",
        "config-private.json"
    ]
    ```

    ``` bash
    freqtrade trade --config user_data/config.json <...>
    ```

??? Note "config collision handling"
    If the same configuration setting takes place in both `config.json` and `config-import.json`, then the parent configuration wins.
    In the below case, `max_open_trades` would be 3 after the merging - as the reusable "import" configuration has this key overwritten.

    ``` json title="user_data/config.json"
    {
        "max_open_trades": 3,
        "stake_currency": "USDT",
        "add_config_files": [
            "config-import.json"
        ]
    }
    ```

    ``` json title="user_data/config-import.json"
    {
        "max_open_trades": 10,
        "stake_amount": "unlimited",
    }
    ```

    Resulting combined configuration:

    ``` json title="Result"
    {
        "max_open_trades": 3,
        "stake_currency": "USDT",
        "stake_amount": "unlimited"
    }
    ```

    If multiple files are in the `add_config_files` section, then they will be assumed to be at identical levels, having the last occurrence override the earlier config (unless a parent already defined such a key).

## Editor autocomplete and validation

If you are using an editor that supports JSON schema, you can use the schema provided by Freqtrade to get autocompletion and validation of your configuration file by adding the following line to the top of your configuration file:

``` json
{
    "$schema": "https://schema.freqtrade.io/schema.json",
}
```

??? Note "Develop version"
    The develop schema is available as `https://schema.freqtrade.io/schema_dev.json` - though we recommend to stick to the stable version for the best experience.

## Configuration parameters

The table below will list all configuration parameters available.

Freqtrade can also load many options via command line (CLI) arguments (check out the commands `--help` output for details).

### Configuration option prevalence

The prevalence for all Options is as follows:

* CLI arguments override any other option
* [Environment Variables](#environment-variables)
* Configuration files are used in sequence (the last file wins) and override Strategy configurations.
* Strategy configurations are only used if they are not set via configuration or command-line arguments. These options are marked with [Strategy Override](#parameters-in-the-strategy) in the below table.

### Parameters table

Mandatory parameters are marked as **Required**, which means that they are required to be set in one of the possible ways.

|  Parameter | Description |
|------------|-------------|
| `max_open_trades` | **Required.** Number of open trades your bot is allowed to have. Only one open trade per pair is possible, so the length of your pairlist is another limitation that can apply. If -1 then it is ignored (i.e. potentially unlimited open trades, limited by the pairlist). [More information below](#configuring-amount-per-trade). [Strategy Override](#parameters-in-the-strategy).<br> **Datatype:** Positive integer or -1.
| `stake_currency` | **Required.** Crypto-currency used for trading. <br> **Datatype:** String
| `stake_amount` | **Required.** Amount of crypto-currency your bot will use for each trade. Set it to `"unlimited"` to allow the bot to use all available balance. [More information below](#configuring-amount-per-trade). <br> **Datatype:** Positive float or `"unlimited"`.
| `tradable_balance_ratio` | Ratio of the total account balance the bot is allowed to trade. [More information below](#configuring-amount-per-trade). <br>*Defaults to `0.99` 99%).*<br> **Datatype:** Positive float between `0.1` and `1.0`.
| `available_capital` | Available starting capital for the bot. Useful when running multiple bots on the same exchange account. [More information below](#configuring-amount-per-trade). <br> **Datatype:** Positive float.
| `amend_last_stake_amount` | Use reduced last stake amount if necessary. [More information below](#configuring-amount-per-trade). <br>*Defaults to `false`.* <br> **Datatype:** Boolean
| `last_stake_amount_min_ratio` | Defines minimum stake amount that has to be left and executed. Applies only to the last stake amount when it's amended to a reduced value (i.e. if `amend_last_stake_amount` is set to `true`). [More information below](#configuring-amount-per-trade). <br>*Defaults to `0.5`.* <br> **Datatype:** Float (as ratio)
| `amount_reserve_percent` | Reserve some amount in min pair stake amount. The bot will reserve `amount_reserve_percent` + stoploss value when calculating min pair stake amount in order to avoid possible trade refusals. <br>*Defaults to `0.05` (5%).* <br> **Datatype:** Positive Float as ratio.
| `timeframe` | The timeframe to use (e.g `1m`, `5m`, `15m`, `30m`, `1h` ...). Usually missing in configuration, and specified in the strategy. [Strategy Override](#parameters-in-the-strategy). <br> **Datatype:** String
| `fiat_display_currency` | Fiat currency used to show your profits. [More information below](#what-values-can-be-used-for-fiat_display_currency). <br> **Datatype:** String
| `dry_run` | **Required.** Define if the bot must be in Dry Run or production mode. <br>*Defaults to `true`.* <br> **Datatype:** Boolean
| `dry_run_wallet` | Define the starting amount in stake currency for the simulated wallet used by the bot running in Dry Run mode. [More information below](#dry-run-wallet)<br>*Defaults to `1000`.* <br> **Datatype:** Float or Dict
| `cancel_open_orders_on_exit` | Cancel open orders when the `/stop` RPC command is issued, `Ctrl+C` is pressed or the bot dies unexpectedly. When set to `true`, this allows you to use `/stop` to cancel unfilled and partially filled orders in the event of a market crash. It does not impact open positions. <br>*Defaults to `false`.* <br> **Datatype:** Boolean
| `process_only_new_candles` | Enable processing of indicators only when new candles arrive. If false each loop populates the indicators, this will mean the same candle is processed many times creating system load but can be useful of your strategy depends on tick data not only candle. [Strategy Override](#parameters-in-the-strategy). <br>*Defaults to `true`.*  <br> **Datatype:** Boolean
| `minimal_roi` | **Required.** Set the threshold as ratio the bot will use to exit a trade. [More information below](#understand-minimal_roi). [Strategy Override](#parameters-in-the-strategy). <br> **Datatype:** Dict
| `stoploss` |  **Required.** Value as ratio of the stoploss used by the bot. More details in the [stoploss documentation](stoploss.md). [Strategy Override](#parameters-in-the-strategy).  <br> **Datatype:** Float (as ratio)
| `trailing_stop` | Enables trailing stoploss (based on `stoploss` in either configuration or strategy file). More details in the [stoploss documentation](stoploss.md#trailing-stop-loss). [Strategy Override](#parameters-in-the-strategy). <br> **Datatype:** Boolean
| `trailing_stop_positive` | Changes stoploss once profit has been reached. More details in the [stoploss documentation](stoploss.md#trailing-stop-loss-custom-positive-loss). [Strategy Override](#parameters-in-the-strategy). <br> **Datatype:** Float
| `trailing_stop_positive_offset` | Offset on when to apply `trailing_stop_positive`. Percentage value which should be positive. More details in the [stoploss documentation](stoploss.md#trailing-stop-loss-only-once-the-trade-has-reached-a-certain-offset). [Strategy Override](#parameters-in-the-strategy). <br>*Defaults to `0.0` (no offset).* <br> **Datatype:** Float
| `trailing_only_offset_is_reached` | Only apply trailing stoploss when the offset is reached. [stoploss documentation](stoploss.md). [Strategy Override](#parameters-in-the-strategy). <br>*Defaults to `false`.*  <br> **Datatype:** Boolean
| `fee` | Fee used during backtesting / dry-runs. Should normally not be configured, which has freqtrade fall back to the exchange default fee. Set as ratio (e.g. 0.001 = 0.1%). Fee is applied twice for each trade, once when buying, once when selling. <br> **Datatype:** Float (as ratio)
| `futures_funding_rate` | User-specified funding rate to be used when historical funding rates are not available from the exchange. This does not overwrite real historical rates. It is recommended that this be set to 0 unless you are testing a specific coin and you understand how the funding rate will affect freqtrade's profit calculations. [More information here](leverage.md#unavailable-funding-rates) <br>*Defaults to `None`.*<br> **Datatype:** Float
| `trading_mode` | Specifies if you want to trade regularly, trade with leverage, or trade contracts whose prices are derived from matching cryptocurrency prices. [leverage documentation](leverage.md). <br>*Defaults to `"spot"`.*  <br> **Datatype:** String
| `margin_mode` | When trading with leverage, this determines if the collateral owned by the trader will be shared or isolated to each trading pair [leverage documentation](leverage.md). <br> **Datatype:** String
| `liquidation_buffer` | A ratio specifying how large of a safety net to place between the liquidation price and the stoploss to prevent a position from reaching the liquidation price [leverage documentation](leverage.md). <br>*Defaults to `0.05`.*  <br> **Datatype:** Float
| | **Unfilled timeout**
| `unfilledtimeout.entry` | **Required.** How long (in minutes or seconds) the bot will wait for an unfilled entry order to complete, after which the order will be cancelled. [Strategy Override](#parameters-in-the-strategy).<br> **Datatype:** Integer
| `unfilledtimeout.exit` | **Required.** How long (in minutes or seconds) the bot will wait for an unfilled exit order to complete, after which the order will be cancelled and repeated at current (new) price, as long as there is a signal. [Strategy Override](#parameters-in-the-strategy).<br> **Datatype:** Integer
| `unfilledtimeout.unit` | Unit to use in unfilledtimeout setting. Note: If you set unfilledtimeout.unit to "seconds", "internals.process_throttle_secs" must be inferior or equal to timeout [Strategy Override](#parameters-in-the-strategy). <br> *Defaults to `"minutes"`.* <br> **Datatype:** String
| `unfilledtimeout.exit_timeout_count` | How many times can exit orders time out. Once this number of timeouts is reached, an emergency exit is triggered. 0 to disable and allow unlimited order cancels. [Strategy Override](#parameters-in-the-strategy).<br>*Defaults to `0`.* <br> **Datatype:** Integer
| | **Pricing**
| `entry_pricing.price_side` | Select the side of the spread the bot should look at to get the entry rate. [More information below](#entry-price).<br> *Defaults to `"same"`.* <br> **Datatype:** String (either `ask`, `bid`, `same` or `other`).
| `entry_pricing.price_last_balance` | **Required.** Interpolate the bidding price. More information [below](#entry-price-without-orderbook-enabled).
| `entry_pricing.use_order_book` | Enable entering using the rates in [Order Book Entry](#entry-price-with-orderbook-enabled). <br> *Defaults to `true`.*<br> **Datatype:** Boolean
| `entry_pricing.order_book_top` | Bot will use the top N rate in Order Book "price_side" to enter a trade. I.e. a value of 2 will allow the bot to pick the 2nd entry in [Order Book Entry](#entry-price-with-orderbook-enabled). <br>*Defaults to `1`.*  <br> **Datatype:** Positive Integer
| `entry_pricing. check_depth_of_market.enabled` | Do not enter if the difference of buy orders and sell orders is met in Order Book. [Check market depth](#check-depth-of-market). <br>*Defaults to `false`.* <br> **Datatype:** Boolean
| `entry_pricing. check_depth_of_market.bids_to_ask_delta` | The difference ratio of buy orders and sell orders found in Order Book. A value below 1 means sell order size is greater, while value greater than 1 means buy order size is higher. [Check market depth](#check-depth-of-market) <br> *Defaults to `0`.*  <br> **Datatype:** Float (as ratio)
| `exit_pricing.price_side` | Select the side of the spread the bot should look at to get the exit rate. [More information below](#exit-price-side).<br> *Defaults to `"same"`.* <br> **Datatype:** String (either `ask`, `bid`, `same` or `other`).
| `exit_pricing.price_last_balance` | Interpolate the exiting price. More information [below](#exit-price-without-orderbook-enabled).
| `exit_pricing.use_order_book` | Enable exiting of open trades using [Order Book Exit](#exit-price-with-orderbook-enabled). <br> *Defaults to `true`.*<br> **Datatype:** Boolean
| `exit_pricing.order_book_top` | Bot will use the top N rate in Order Book "price_side" to exit. I.e. a value of 2 will allow the bot to pick the 2nd ask rate in [Order Book Exit](#exit-price-with-orderbook-enabled)<br>*Defaults to `1`.* <br> **Datatype:** Positive Integer
| `custom_price_max_distance_ratio` | Configure maximum distance ratio between current and custom entry or exit price. <br>*Defaults to `0.02` 2%).*<br> **Datatype:** Positive float
| | **TODO**
| `use_exit_signal` | Use exit signals produced by the strategy in addition to the `minimal_roi`. <br>Setting this to false disables the usage of `"exit_long"` and `"exit_short"` columns. Has no influence on other exit methods (Stoploss, ROI, callbacks). [Strategy Override](#parameters-in-the-strategy). <br>*Defaults to `true`.* <br> **Datatype:** Boolean
| `exit_profit_only` | Wait until the bot reaches `exit_profit_offset` before taking an exit decision. [Strategy Override](#parameters-in-the-strategy). <br>*Defaults to `false`.* <br> **Datatype:** Boolean
| `exit_profit_offset` | Exit-signal is only active above this value. Only active in combination with `exit_profit_only=True`. [Strategy Override](#parameters-in-the-strategy). <br>*Defaults to `0.0`.* <br> **Datatype:** Float (as ratio)
| `ignore_roi_if_entry_signal` | Do not exit if the entry signal is still active. This setting takes preference over `minimal_roi` and `use_exit_signal`. [Strategy Override](#parameters-in-the-strategy). <br>*Defaults to `false`.* <br> **Datatype:** Boolean
| `ignore_buying_expired_candle_after` | Specifies the number of seconds until a buy signal is no longer used. <br> **Datatype:** Integer
| `order_types` | Configure order-types depending on the action (`"entry"`, `"exit"`, `"stoploss"`, `"stoploss_on_exchange"`). [More information below](#understand-order_types). [Strategy Override](#parameters-in-the-strategy).<br> **Datatype:** Dict
| `order_time_in_force` | Configure time in force for entry and exit orders. [More information below](#understand-order_time_in_force). [Strategy Override](#parameters-in-the-strategy). <br> **Datatype:** Dict
| `position_adjustment_enable` | Enables the strategy to use position adjustments (additional buys or sells). [More information here](strategy-callbacks.md#adjust-trade-position). <br> [Strategy Override](#parameters-in-the-strategy). <br>*Defaults to `false`.*<br> **Datatype:** Boolean
| `max_entry_position_adjustment` | Maximum additional order(s) for each open trade on top of the first entry Order. Set it to `-1` for unlimited additional orders. [More information here](strategy-callbacks.md#adjust-trade-position). <br> [Strategy Override](#parameters-in-the-strategy). <br>*Defaults to `-1`.*<br> **Datatype:** Positive Integer or -1
| | **Exchange**
| `exchange.name` | **Required.** Name of the exchange class to use. <br> **Datatype:** String
| `exchange.key` | API key to use for the exchange. Only required when you are in production mode.<br>**Keep it in secret, do not disclose publicly.** <br> **Datatype:** String
| `exchange.secret` | API secret to use for the exchange. Only required when you are in production mode.<br>**Keep it in secret, do not disclose publicly.** <br> **Datatype:** String
| `exchange.password` | API password to use for the exchange. Only required when you are in production mode and for exchanges that use password for API requests.<br>**Keep it in secret, do not disclose publicly.** <br> **Datatype:** String
| `exchange.uid` | API uid to use for the exchange. Only required when you are in production mode and for exchanges that use uid for API requests.<br>**Keep it in secret, do not disclose publicly.** <br> **Datatype:** String
| `exchange.pair_whitelist` | List of pairs to use by the bot for trading and to check for potential trades during backtesting. Supports regex pairs as `.*/BTC`. Not used by VolumePairList. [More information](plugins.md#pairlists-and-pairlist-handlers). <br> **Datatype:** List
| `exchange.pair_blacklist` | List of pairs the bot must absolutely avoid for trading and backtesting. [More information](plugins.md#pairlists-and-pairlist-handlers). <br> **Datatype:** List
| `exchange.ccxt_config` | Additional CCXT parameters passed to both ccxt instances (sync and async). This is usually the correct place for additional ccxt configurations. Parameters may differ from exchange to exchange and are documented in the [ccxt documentation](https://docs.ccxt.com/#/README?id=overriding-exchange-properties-upon-instantiation). Please avoid adding exchange secrets here (use the dedicated fields instead), as they may be contained in logs. <br> **Datatype:** Dict
| `exchange.ccxt_sync_config` | Additional CCXT parameters passed to the regular (sync) ccxt instance. Parameters may differ from exchange to exchange and are documented in the [ccxt documentation](https://docs.ccxt.com/#/README?id=overriding-exchange-properties-upon-instantiation) <br> **Datatype:** Dict
| `exchange.ccxt_async_config` | Additional CCXT parameters passed to the async ccxt instance. Parameters may differ from exchange to exchange  and are documented in the [ccxt documentation](https://docs.ccxt.com/#/README?id=overriding-exchange-properties-upon-instantiation) <br> **Datatype:** Dict
| `exchange.enable_ws` | Enable the usage of Websockets for the exchange. <br>[More information](#consuming-exchange-websockets).<br>*Defaults to `true`.* <br> **Datatype:** Boolean
| `exchange.markets_refresh_interval` | The interval in minutes in which markets are reloaded. <br>*Defaults to `60` minutes.* <br> **Datatype:** Positive Integer
| `exchange.skip_open_order_update` | Skips open order updates on startup should the exchange cause problems. Only relevant in live conditions.<br>*Defaults to `false`*<br> **Datatype:** Boolean
| `exchange.unknown_fee_rate` | Fallback value to use when calculating trading fees. This can be useful for exchanges which have fees in non-tradable currencies. The value provided here will be multiplied with the "fee cost".<br>*Defaults to `None`<br> **Datatype:** float
| `exchange.log_responses` | Log relevant exchange responses. For debug mode only - use with care.<br>*Defaults to `false`*<br> **Datatype:** Boolean
| `exchange.only_from_ccxt` | Prevent data-download from data.binance.vision. Leaving this as false can greatly speed up downloads, but may be problematic if the site is not available.<br>*Defaults to `false`*<br> **Datatype:** Boolean
| `experimental.block_bad_exchanges` | Block exchanges known to not work with freqtrade. Leave on default unless you want to test if that exchange works now. <br>*Defaults to `true`.* <br> **Datatype:** Boolean
| | **Plugins**
| `edge.*` | Please refer to [edge configuration document](edge.md) for detailed explanation of all possible configuration options.
| `pairlists` | Define one or more pairlists to be used. [More information](plugins.md#pairlists-and-pairlist-handlers). <br>*Defaults to `StaticPairList`.*  <br> **Datatype:** List of Dicts
| | **Telegram**
| `telegram.enabled` | Enable the usage of Telegram. <br> **Datatype:** Boolean
| `telegram.token` | Your Telegram bot token. Only required if `telegram.enabled` is `true`. <br>**Keep it in secret, do not disclose publicly.** <br> **Datatype:** String
| `telegram.chat_id` | Your personal Telegram account id. Only required if `telegram.enabled` is `true`. <br>**Keep it in secret, do not disclose publicly.** <br> **Datatype:** String
| `telegram.balance_dust_level` | Dust-level (in stake currency) - currencies with a balance below this will not be shown by `/balance`. <br> **Datatype:** float
| `telegram.reload` | Allow "reload" buttons on telegram messages. <br>*Defaults to `true`.<br> **Datatype:** boolean
| `telegram.notification_settings.*` | Detailed notification settings. Refer to the [telegram documentation](telegram-usage.md) for details.<br> **Datatype:** dictionary
| `telegram.allow_custom_messages` | Enable the sending of Telegram messages from strategies via the dataprovider.send_msg() function. <br> **Datatype:** Boolean
| | **Webhook**
| `webhook.enabled` | Enable usage of Webhook notifications <br> **Datatype:** Boolean
| `webhook.url` | URL for the webhook. Only required if `webhook.enabled` is `true`. See the [webhook documentation](webhook-config.md) for more details. <br> **Datatype:** String
| `webhook.entry` | Payload to send on entry. Only required if `webhook.enabled` is `true`. See the [webhook documentation](webhook-config.md) for more details. <br> **Datatype:** String
| `webhook.entry_cancel` | Payload to send on entry order cancel. Only required if `webhook.enabled` is `true`. See the [webhook documentation](webhook-config.md) for more details. <br> **Datatype:** String
| `webhook.entry_fill` | Payload to send on entry order filled. Only required if `webhook.enabled` is `true`. See the [webhook documentation](webhook-config.md) for more details. <br> **Datatype:** String
| `webhook.exit` | Payload to send on exit. Only required if `webhook.enabled` is `true`. See the [webhook documentation](webhook-config.md) for more details. <br> **Datatype:** String
| `webhook.exit_cancel` | Payload to send on exit order cancel. Only required if `webhook.enabled` is `true`. See the [webhook documentation](webhook-config.md) for more details. <br> **Datatype:** String
| `webhook.exit_fill` | Payload to send on exit order filled. Only required if `webhook.enabled` is `true`. See the [webhook documentation](webhook-config.md) for more details. <br> **Datatype:** String
| `webhook.status` | Payload to send on status calls. Only required if `webhook.enabled` is `true`. See the [webhook documentation](webhook-config.md) for more details. <br> **Datatype:** String
| `webhook.allow_custom_messages` | Enable the sending of Webhook messages from strategies via the dataprovider.send_msg() function. <br> **Datatype:** Boolean
| | **Rest API / FreqUI / Producer-Consumer**
| `api_server.enabled` | Enable usage of API Server. See the [API Server documentation](rest-api.md) for more details. <br> **Datatype:** Boolean
| `api_server.listen_ip_address` | Bind IP address. See the [API Server documentation](rest-api.md) for more details. <br> **Datatype:** IPv4
| `api_server.listen_port` | Bind Port. See the [API Server documentation](rest-api.md) for more details. <br>**Datatype:** Integer between 1024 and 65535
| `api_server.verbosity` | Logging verbosity. `info` will print all RPC Calls, while "error" will only display errors. <br>**Datatype:** Enum, either `info` or `error`. Defaults to `info`.
| `api_server.username` | Username for API server. See the [API Server documentation](rest-api.md) for more details. <br>**Keep it in secret, do not disclose publicly.**<br> **Datatype:** String
| `api_server.password` | Password for API server. See the [API Server documentation](rest-api.md) for more details. <br>**Keep it in secret, do not disclose publicly.**<br> **Datatype:** String
| `api_server.ws_token` | API token for the Message WebSocket. See the [API Server documentation](rest-api.md) for more details.  <br>**Keep it in secret, do not disclose publicly.** <br> **Datatype:** String
| `bot_name` | Name of the bot. Passed via API to a client - can be shown to distinguish / name bots.<br> *Defaults to `freqtrade`*<br> **Datatype:** String
| `external_message_consumer` | Enable [Producer/Consumer mode](producer-consumer.md) for more details. <br> **Datatype:** Dict
| | **Other**
| `initial_state` | Defines the initial application state. If set to stopped, then the bot has to be explicitly started via `/start` RPC command. <br>*Defaults to `stopped`.* <br> **Datatype:** Enum, either `stopped` or `running`
| `force_entry_enable` | Enables the RPC Commands to force a Trade entry. More information below. <br> **Datatype:** Boolean
| `disable_dataframe_checks` | Disable checking the OHLCV dataframe returned from the strategy methods for correctness. Only use when intentionally changing the dataframe and understand what you are doing. [Strategy Override](#parameters-in-the-strategy).<br> *Defaults to `False`*. <br> **Datatype:** Boolean
| `internals.process_throttle_secs` | Set the process throttle, or minimum loop duration for one bot iteration loop. Value in second. <br>*Defaults to `5` seconds.* <br> **Datatype:** Positive Integer
| `internals.heartbeat_interval` | Print heartbeat message every N seconds. Set to 0 to disable heartbeat messages. <br>*Defaults to `60` seconds.* <br> **Datatype:** Positive Integer or 0
| `internals.sd_notify` | Enables use of the sd_notify protocol to tell systemd service manager about changes in the bot state and issue keep-alive pings. See [here](advanced-setup.md#configure-the-bot-running-as-a-systemd-service) for more details. <br> **Datatype:** Boolean
| `strategy` | **Required** Defines Strategy class to use. Recommended to be set via `--strategy NAME`. <br> **Datatype:** ClassName
| `strategy_path` | Adds an additional strategy lookup path (must be a directory). <br> **Datatype:** String
| `recursive_strategy_search` | Set to `true` to recursively search sub-directories inside `user_data/strategies` for a strategy. <br> **Datatype:** Boolean
| `user_data_dir` | Directory containing user data. <br> *Defaults to `./user_data/`*. <br> **Datatype:** String
| `db_url` | Declares database URL to use. NOTE: This defaults to `sqlite:///tradesv3.dryrun.sqlite` if `dry_run` is `true`, and to `sqlite:///tradesv3.sqlite` for production instances. <br> **Datatype:** String, SQLAlchemy connect string
| `logfile` | Specifies logfile name. Uses a rolling strategy for log file rotation for 10 files with the 1MB limit per file. <br> **Datatype:** String
| `add_config_files` | Additional config files. These files will be loaded and merged with the current config file. The files are resolved relative to the initial file.<br> *Defaults to `[]`*. <br> **Datatype:** List of strings
| `dataformat_ohlcv` | Data format to use to store historical candle (OHLCV) data. <br> *Defaults to `feather`*. <br> **Datatype:** String
| `dataformat_trades` | Data format to use to store historical trades data. <br> *Defaults to `feather`*. <br> **Datatype:** String
| `reduce_df_footprint` | Recast all numeric columns to float32/int32, with the objective of reducing ram/disk usage (and decreasing train/inference timing in FreqAI). (Currently only affects FreqAI use-cases) <br> **Datatype:** Boolean. <br> Default: `False`.
| `log_config` | Dictionary containing the log config for python logging. [more info](advanced-setup.md#advanced-logging) <br> **Datatype:** dict. <br> Default: `FtRichHandler`

### Parameters in the strategy

The following parameters can be set in the configuration file or strategy.
Values set in the configuration file always overwrite values set in the strategy.

* `minimal_roi`
* `timeframe`
* `stoploss`
* `max_open_trades`
* `trailing_stop`
* `trailing_stop_positive`
* `trailing_stop_positive_offset`
* `trailing_only_offset_is_reached`
* `use_custom_stoploss`
* `process_only_new_candles`
* `order_types`
* `order_time_in_force`
* `unfilledtimeout`
* `disable_dataframe_checks`
* `use_exit_signal`
* `exit_profit_only`
* `exit_profit_offset`
* `ignore_roi_if_entry_signal`
* `ignore_buying_expired_candle_after`
* `position_adjustment_enable`
* `max_entry_position_adjustment`

### Configuring amount per trade

There are several methods to configure how much of the stake currency the bot will use to enter a trade. All methods respect the [available balance configuration](#tradable-balance) as explained below.

#### Minimum trade stake

The minimum stake amount will depend on exchange and pair and is usually listed in the exchange support pages.

Assuming the minimum tradable amount for XRP/USD is 20 XRP (given by the exchange), and the price is 0.6\$, the minimum stake amount to buy this pair is `20 * 0.6 ~= 12`.
This exchange has also a limit on USD - where all orders must be > 10\$ - which however does not apply in this case.

To guarantee safe execution, freqtrade will not allow buying with a stake-amount of 10.1\$, instead, it'll make sure that there's enough space to place a stoploss below the pair (+ an offset, defined by `amount_reserve_percent`, which defaults to 5%).

With a reserve of 5%, the minimum stake amount would be ~12.6\$ (`12 * (1 + 0.05)`). If we take into account a stoploss of 10% on top of that - we'd end up with a value of ~14\$ (`12.6 / (1 - 0.1)`).

To limit this calculation in case of large stoploss values, the calculated minimum stake-limit will never be more than 50% above the real limit.

!!! Warning
    Since the limits on exchanges are usually stable and are not updated often, some pairs can show pretty high minimum limits, simply because the price increased a lot since the last limit adjustment by the exchange. Freqtrade adjusts the stake-amount to this value, unless it's > 30% more than the calculated/desired stake-amount - in which case the trade is rejected.

#### Dry-run wallet

When running in dry-run mode, the bot will use a simulated wallet to execute trades. The starting balance of this wallet is defined by `dry_run_wallet` (defaults to 1000).
For more complex scenarios, you can also assign a dictionary to `dry_run_wallet` to define the starting balance for each currency.

```json
"dry_run_wallet": {
    "BTC": 0.01,
    "ETH": 2,
    "USDT": 1000
}
```

Command line options (`--dry-run-wallet`) can be used to override the configuration value, but only for the float value, not for the dictionary. If you'd like to use the dictionary, please adjust the configuration file.

!!! Note
    Balances not in stake-currency will not be used for trading, but are shown as part of the wallet balance.
    On Cross-margin exchanges, the wallet balance may be used to calculate the available collateral for trading.

#### Tradable balance

By default, the bot assumes that the `complete amount - 1%` is at it's disposal, and when using [dynamic stake amount](#dynamic-stake-amount), it will split the complete balance into `max_open_trades` buckets per trade.
Freqtrade will reserve 1% for eventual fees when entering a trade and will therefore not touch that by default.

You can configure the "untouched" amount by using the `tradable_balance_ratio` setting.

For example, if you have 10 ETH available in your wallet on the exchange and `tradable_balance_ratio=0.5` (which is 50%), then the bot will use a maximum amount of 5 ETH for trading and considers this as an available balance. The rest of the wallet is untouched by the trades.

!!! Danger
    This setting should **not** be used when running multiple bots on the same account. Please look at [Available Capital to the bot](#assign-available-capital) instead.

!!! Warning
    The `tradable_balance_ratio` setting applies to the current balance (free balance + tied up in trades). Therefore, assuming the starting balance of 1000, a configuration with `tradable_balance_ratio=0.99` will not guarantee that 10 currency units will always remain available on the exchange. For example, the free amount may reduce to 5 units if the total balance is reduced to 500 (either by a losing streak or by withdrawing balance).

#### Assign available Capital

To fully utilize compounding profits when using multiple bots on the same exchange account, you'll want to limit each bot to a certain starting balance.
This can be accomplished by setting `available_capital` to the desired starting balance.

Assuming your account has 10000 USDT and you want to run 2 different strategies on this exchange.
You'd set `available_capital=5000` - granting each bot an initial capital of 5000 USDT.
The bot will then split this starting balance equally into `max_open_trades` buckets.
Profitable trades will result in increased stake-sizes for this bot - without affecting the stake-sizes of the other bot.

Adjusting `available_capital` requires reloading the configuration to take effect. Adjusting the `available_capital` adds the difference between the previous `available_capital` and the new `available_capital`. Decreasing the available capital when trades are open doesn't exit the trades. The difference is returned to the wallet when the trades conclude. The outcome of this differs depending on the price movement between the adjustment and exiting the trades.

!!! Warning "Incompatible with `tradable_balance_ratio`"
    Setting this option will replace any configuration of `tradable_balance_ratio`.

#### Amend last stake amount

Assuming we have the tradable balance of 1000 USDT, `stake_amount=400`, and `max_open_trades=3`.
The bot would open 2 trades and will be unable to fill the last trading slot, since the requested 400 USDT are no longer available since 800 USDT are already tied in other trades.

To overcome this, the option `amend_last_stake_amount` can be set to `True`, which will enable the bot to reduce stake_amount to the available balance to fill the last trade slot.

In the example above this would mean:

* Trade1: 400 USDT
* Trade2: 400 USDT
* Trade3: 200 USDT

!!! Note
    This option only applies with [Static stake amount](#static-stake-amount) - since [Dynamic stake amount](#dynamic-stake-amount) divides the balances evenly.

!!! Note
    The minimum last stake amount can be configured using `last_stake_amount_min_ratio` - which defaults to 0.5 (50%). This means that the minimum stake amount that's ever used is `stake_amount * 0.5`. This avoids very low stake amounts, that are close to the minimum tradable amount for the pair and can be refused by the exchange.

#### Static stake amount

The `stake_amount` configuration statically configures the amount of stake-currency your bot will use for each trade.

The minimal configuration value is 0.0001, however, please check your exchange's trading minimums for the stake currency you're using to avoid problems.

This setting works in combination with `max_open_trades`. The maximum capital engaged in trades is `stake_amount * max_open_trades`.
For example, the bot will at most use (0.05 BTC x 3) = 0.15 BTC, assuming a configuration of `max_open_trades=3` and `stake_amount=0.05`.

!!! Note
    This setting respects the [available balance configuration](#tradable-balance).

#### Dynamic stake amount

Alternatively, you can use a dynamic stake amount, which will use the available balance on the exchange, and divide that equally by the number of allowed trades (`max_open_trades`).

To configure this, set `stake_amount="unlimited"`. We also recommend to set `tradable_balance_ratio=0.99` (99%) - to keep a minimum balance for eventual fees.

In this case a trade amount is calculated as:

```python
currency_balance / (max_open_trades - current_open_trades)
```

To allow the bot to trade all the available `stake_currency` in your account (minus `tradable_balance_ratio`) set

```json
"stake_amount" : "unlimited",
"tradable_balance_ratio": 0.99,
```

!!! Tip "Compounding profits"
    This configuration will allow increasing/decreasing stakes depending on the performance of the bot (lower stake if the bot is losing, higher stakes if the bot has a winning record since higher balances are available), and will result in profit compounding.

!!! Note "When using Dry-Run Mode"
    When using `"stake_amount" : "unlimited",` in combination with Dry-Run, Backtesting or Hyperopt, the balance will be simulated starting with a stake of `dry_run_wallet` which will evolve.
    It is therefore important to set `dry_run_wallet` to a sensible value (like 0.05 or 0.01 for BTC and 1000 or 100 for USDT, for example), otherwise, it may simulate trades with 100 BTC (or more) or 0.05 USDT (or less) at once - which may not correspond to your real available balance or is less than the exchange minimal limit for the order amount for the stake currency.

#### Dynamic stake amount with position adjustment

When you want to use position adjustment with unlimited stakes, you must also implement `custom_stake_amount` to a return a value depending on your strategy.
Typical value would be in the range of 25% - 50% of the proposed stakes, but depends highly on your strategy and how much you wish to leave into the wallet as position adjustment buffer.

For example if your position adjustment assumes it can do 2 additional buys with the same stake amounts then your buffer should be 66.6667% of the initially proposed unlimited stake amount.

Or another example if your position adjustment assumes it can do 1 additional buy with 3x the original stake amount then `custom_stake_amount` should return 25% of proposed stake amount and leave 75% for possible later position adjustments.

--8<-- "includes/pricing.md"

## Further Configuration details

### Understand minimal_roi

The `minimal_roi` configuration parameter is a JSON object where the key is a duration
in minutes and the value is the minimum ROI as a ratio.
See the example below:

```json
"minimal_roi": {
    "40": 0.0,    # Exit after 40 minutes if the profit is not negative
    "30": 0.01,   # Exit after 30 minutes if there is at least 1% profit
    "20": 0.02,   # Exit after 20 minutes if there is at least 2% profit
    "0":  0.04    # Exit immediately if there is at least 4% profit
},
```

Most of the strategy files already include the optimal `minimal_roi` value.
This parameter can be set in either Strategy or Configuration file. If you use it in the configuration file, it will override the
`minimal_roi` value from the strategy file.
If it is not set in either Strategy or Configuration, a default of 1000% `{"0": 10}` is used, and minimal ROI is disabled unless your trade generates 1000% profit.

!!! Note "Special case to forceexit after a specific time"
    A special case presents using `"<N>": -1` as ROI. This forces the bot to exit a trade after N Minutes, no matter if it's positive or negative, so represents a time-limited force-exit.

### Understand force_entry_enable

The `force_entry_enable` configuration parameter enables the usage of force-enter (`/forcelong`, `/forceshort`) commands via Telegram and REST API.
For security reasons, it's disabled by default, and freqtrade will show a warning message on startup if enabled.
For example, you can send `/forceenter ETH/BTC` to the bot, which will result in freqtrade buying the pair and holds it until a regular exit-signal (ROI, stoploss, /forceexit) appears.

This can be dangerous with some strategies, so use with care.

See [the telegram documentation](telegram-usage.md) for details on usage.

### Ignoring expired candles

When working with larger timeframes (for example 1h or more) and using a low `max_open_trades` value, the last candle can be processed as soon as a trade slot becomes available. When processing the last candle, this can lead to a situation where it may not be desirable to use the buy signal on that candle. For example, when using a condition in your strategy where you use a cross-over, that point may have passed too long ago for you to start a trade on it.

In these situations, you can enable the functionality to ignore candles that are beyond a specified period by setting `ignore_buying_expired_candle_after` to a positive number, indicating the number of seconds after which the buy signal becomes expired.

For example, if your strategy is using a 1h timeframe, and you only want to buy within the first 5 minutes when a new candle comes in, you can add the following configuration to your strategy:

``` json
  {
    //...
    "ignore_buying_expired_candle_after": 300,
    // ...
  }
```

!!! Note
    This setting resets with each new candle, so it will not prevent sticking-signals from executing on the 2nd or 3rd candle they're active. Best use a "trigger" selector for buy signals, which are only active for one candle.

### Understand order_types

The `order_types` configuration parameter maps actions (`entry`, `exit`, `stoploss`, `emergency_exit`, `force_exit`, `force_entry`) to order-types (`market`, `limit`, ...) as well as configures stoploss to be on the exchange and defines stoploss on exchange update interval in seconds.

This allows to enter using limit orders, exit using limit-orders, and create stoplosses using market orders.
It also allows to set the
stoploss "on exchange" which means stoploss order would be placed immediately once the buy order is fulfilled.

`order_types` set in the configuration file overwrites values set in the strategy as a whole, so you need to configure the whole `order_types` dictionary in one place.

If this is configured, the following 4 values (`entry`, `exit`, `stoploss` and `stoploss_on_exchange`) need to be present, otherwise, the bot will fail to start.

For information on (`emergency_exit`,`force_exit`, `force_entry`, `stoploss_on_exchange`,`stoploss_on_exchange_interval`,`stoploss_on_exchange_limit_ratio`) please see stop loss documentation [stop loss on exchange](stoploss.md)

Syntax for Strategy:

```python
order_types = {
    "entry": "limit",
    "exit": "limit",
    "emergency_exit": "market",
    "force_entry": "market",
    "force_exit": "market",
    "stoploss": "market",
    "stoploss_on_exchange": False,
    "stoploss_on_exchange_interval": 60,
    "stoploss_on_exchange_limit_ratio": 0.99,
}
```

Configuration:

```json
"order_types": {
    "entry": "limit",
    "exit": "limit",
    "emergency_exit": "market",
    "force_entry": "market",
    "force_exit": "market",
    "stoploss": "market",
    "stoploss_on_exchange": false,
    "stoploss_on_exchange_interval": 60
}
```

!!! Note "Market order support"
    Not all exchanges support "market" orders.
    The following message will be shown if your exchange does not support market orders:
    `"Exchange <yourexchange> does not support market orders."` and the bot will refuse to start.

!!! Warning "Using market orders"
    Please carefully read the section [Market order pricing](#market-order-pricing) section when using market orders.

!!! Note "Stoploss on exchange"
    `order_types.stoploss_on_exchange_interval` is not mandatory. Do not change its value if you are
    unsure of what you are doing. For more information about how stoploss works please
    refer to [the stoploss documentation](stoploss.md).

    If `order_types.stoploss_on_exchange` is enabled and the stoploss is cancelled manually on the exchange, then the bot will create a new stoploss order.

!!! Warning "Warning: order_types.stoploss_on_exchange failures"
    If stoploss on exchange creation fails for some reason, then an "emergency exit" is initiated. By default, this will exit the trade using a market order. The order-type for the emergency-exit can be changed by setting the `emergency_exit` value in the `order_types` dictionary - however, this is not advised.

### Understand order_time_in_force

The `order_time_in_force` configuration parameter defines the policy by which the order
is executed on the exchange. Three commonly used time in force are:

**GTC (Good Till Canceled):**

This is most of the time the default time in force. It means the order will remain
on exchange till it is cancelled by the user. It can be fully or partially fulfilled.
If partially fulfilled, the remaining will stay on the exchange till cancelled.

**FOK (Fill Or Kill):**

It means if the order is not executed immediately AND fully then it is cancelled by the exchange.

**IOC (Immediate Or Canceled):**

It is the same as FOK (above) except it can be partially fulfilled. The remaining part
is automatically cancelled by the exchange.

**PO (Post only):**

Post only order. The order is either placed as a maker order, or it is canceled.
This means the order must be placed on orderbook for at least time in an unfilled state.

#### time_in_force config

The `order_time_in_force` parameter contains a dict with entry and exit time in force policy values.
This can be set in the configuration file or in the strategy.
Values set in the configuration file overwrites values set in the strategy.

The possible values are: `GTC` (default), `FOK` or `IOC`.

``` python
"order_time_in_force": {
    "entry": "GTC",
    "exit": "GTC"
},
```

!!! Warning
    This is ongoing work. For now, it is supported only for binance, gate and kucoin.
    Please don't change the default value unless you know what you are doing and have researched the impact of using different values for your particular exchange.

### Fiat conversion

Freqtrade uses the Coingecko API to convert the coin value to it's corresponding fiat value for the Telegram reports.
The FIAT currency can be set in the configuration file as `fiat_display_currency`.

Removing `fiat_display_currency` completely from the configuration will skip initializing coingecko, and will not show any FIAT currency conversion. This has no importance for the correct functioning of the bot.

#### What values can be used for fiat_display_currency?

The `fiat_display_currency` configuration parameter sets the base currency to use for the
conversion from coin to fiat in the bot Telegram reports.

The valid values are:

```json
"AUD", "BRL", "CAD", "CHF", "CLP", "CNY", "CZK", "DKK", "EUR", "GBP", "HKD", "HUF", "IDR", "ILS", "INR", "JPY", "KRW", "MXN", "MYR", "NOK", "NZD", "PHP", "PKR", "PLN", "RUB", "SEK", "SGD", "THB", "TRY", "TWD", "ZAR", "USD"
```

In addition to fiat currencies, a range of crypto currencies is supported.

The valid values are:

```json
"BTC", "ETH", "XRP", "LTC", "BCH", "BNB"
```

#### Coingecko Rate limit problems

On some IP ranges, coingecko is heavily rate-limiting.
In such cases, you may want to add your coingecko API key to the configuration.

``` json
{
    "fiat_display_currency": "USD",
    "coingecko": {
        "api_key": "your-api",
        "is_demo": true
    }
}
```

Freqtrade supports both Demo and Pro coingecko API keys.

The Coingecko API key is NOT required for the bot to function correctly.
It is only used for the conversion of coin to fiat in the Telegram reports, which usually also work without API key.

## Consuming exchange Websockets

Freqtrade can consume websockets through ccxt.pro.

Freqtrade aims ensure data is available at all times.
Should the websocket connection fail (or be disabled), the bot will fall back to REST API calls.

Should you experience problems you suspect are caused by websockets, you can disable these via the setting `exchange.enable_ws`, which defaults to true.

```jsonc
"exchange": {
    // ...
    "enable_ws": false,
    // ...
}
```

Should you be required to use a proxy, please refer to the [proxy section](#using-proxy-with-freqtrade) for more information.

!!! Info "Rollout"
    We're implementing this out slowly, ensuring stability of your bots.
    Currently, usage is limited to ohlcv data streams.
    It's also limited to a few exchanges, with new exchanges being added on an ongoing basis.

## Using Dry-run mode

We recommend starting the bot in the Dry-run mode to see how your bot will
behave and what is the performance of your strategy. In the Dry-run mode, the
bot does not engage your money. It only runs a live simulation without
creating trades on the exchange.

1. Edit your `config.json` configuration file.
2. Switch `dry-run` to `true` and specify `db_url` for a persistence database.

```json
"dry_run": true,
"db_url": "sqlite:///tradesv3.dryrun.sqlite",
```

3. Remove your Exchange API key and secret (change them by empty values or fake credentials):

```json
"exchange": {
    "name": "binance",
    "key": "key",
    "secret": "secret",
    ...
}
```

Once you will be happy with your bot performance running in the Dry-run mode, you can switch it to production mode.

!!! Note
    A simulated wallet is available during dry-run mode and will assume a starting capital of `dry_run_wallet` (defaults to 1000).

### Considerations for dry-run

* API-keys may or may not be provided. Only Read-Only operations (i.e. operations that do not alter account state) on the exchange are performed in dry-run mode.
* Wallets (`/balance`) are simulated based on `dry_run_wallet`.
* Orders are simulated, and will not be posted to the exchange.
* Market orders fill based on orderbook volume the moment the order is placed, with a maximum slippage of 5%.
* Limit orders fill once the price reaches the defined level - or time out based on `unfilledtimeout` settings.
* Limit orders will be converted to market orders if they cross the price by more than 1%, and will be filled immediately based regular market order rules (see point about Market orders above).
* In combination with `stoploss_on_exchange`, the stop_loss price is assumed to be filled.
* Open orders (not trades, which are stored in the database) are kept open after bot restarts, with the assumption that they were not filled while being offline.

## Switch to production mode

In production mode, the bot will engage your money. Be careful, since a wrong strategy can lose all your money.
Be aware of what you are doing when you run it in production mode.

When switching to Production mode, please make sure to use a different / fresh database to avoid dry-run trades messing with your exchange money and eventually tainting your statistics.

### Setup your exchange account

You will need to create API Keys (usually you get `key` and `secret`, some exchanges require an additional `password`) from the Exchange website and you'll need to insert this into the appropriate fields in the configuration or when asked by the `freqtrade new-config` command.
API Keys are usually only required for live trading (trading for real money, bot running in "production mode", executing real orders on the exchange) and are not required for the bot running in dry-run (trade simulation) mode. When you set up the bot in dry-run mode, you may fill these fields with empty values.

### To switch your bot in production mode

**Edit your `config.json`  file.**

**Switch dry-run to false and don't forget to adapt your database URL if set:**

```json
"dry_run": false,
```

**Insert your Exchange API key (change them by fake API keys):**

```json
{
    "exchange": {
        "name": "binance",
        "key": "af8ddd35195e9dc500b9a6f799f6f5c93d89193b",
        "secret": "08a9dc6db3d7b53e1acebd9275677f4b0a04f1a5",
        //"password": "", // Optional, not needed by all exchanges)
        // ...
    }
    //...
}
```

You should also make sure to read the [Exchanges](exchanges.md) section of the documentation to be aware of potential configuration details specific to your exchange.

!!! Hint "Keep your secrets secret"
    To keep your secrets secret, we recommend using a 2nd configuration for your API keys.
    Simply use the above snippet in a new configuration file (e.g. `config-private.json`) and keep your settings in this file.
    You can then start the bot with `freqtrade trade --config user_data/config.json --config user_data/config-private.json <...>` to have your keys loaded.

    **NEVER** share your private configuration file or your exchange keys with anyone!

## Using a proxy with Freqtrade

To use a proxy with freqtrade, export your proxy settings using the variables `"HTTP_PROXY"` and `"HTTPS_PROXY"` set to the appropriate values.
This will have the proxy settings applied to everything (telegram, coingecko, ...) **except** for exchange requests.

``` bash
export HTTP_PROXY="http://addr:port"
export HTTPS_PROXY="http://addr:port"
freqtrade
```

### Proxy exchange requests

To use a proxy for exchange connections - you will have to define the proxies as part of the ccxt configuration.

``` json
{ 
  "exchange": {
    "ccxt_config": {
      "httpsProxy": "http://addr:port",
      "wsProxy": "http://addr:port",
    }
  }
}
```

For more information on available proxy types, please consult the [ccxt proxy documentation](https://docs.ccxt.com/#/README?id=proxy).

## Next step

Now you have configured your config.json, the next step is to [start your bot](bot-usage.md).

---

## File: data-analysis.md

# Analyzing bot data with Jupyter notebooks

You can analyze the results of backtests and trading history easily using Jupyter notebooks. Sample notebooks are located at `user_data/notebooks/` after initializing the user directory with `freqtrade create-userdir --userdir user_data`.

## Quick start with docker

Freqtrade provides a docker-compose file which starts up a jupyter lab server.
You can run this server using the following command: `docker compose -f docker/docker-compose-jupyter.yml up`

This will create a dockercontainer running jupyter lab, which will be accessible using `https://127.0.0.1:8888/lab`.
Please use the link that's printed in the console after startup for simplified login.

For more information, Please visit the [Data analysis with Docker](docker_quickstart.md#data-analysis-using-docker-compose) section.

### Pro tips

* See [jupyter.org](https://jupyter.org/documentation) for usage instructions.
* Don't forget to start a Jupyter notebook server from within your conda or venv environment or use [nb_conda_kernels](https://github.com/Anaconda-Platform/nb_conda_kernels)*
* Copy the example notebook before use so your changes don't get overwritten with the next freqtrade update.

### Using virtual environment with system-wide Jupyter installation

Sometimes it can be desired to use a system-wide installation of Jupyter notebook, and use a jupyter kernel from the virtual environment.
This prevents you from installing the full jupyter suite multiple times per system, and provides an easy way to switch between tasks (freqtrade / other analytics tasks).

For this to work, first activate your virtual environment and run the following commands:

``` bash
# Activate virtual environment
source .venv/bin/activate

pip install ipykernel
ipython kernel install --user --name=freqtrade
# Restart jupyter (lab / notebook)
# select kernel "freqtrade" in the notebook
```

!!! Note
    This section is provided for completeness, the Freqtrade Team won't provide full support for problems with this setup and will recommend to install Jupyter in the virtual environment directly, as that is the easiest way to get jupyter notebooks up and running. For help with this setup please refer to the [Project Jupyter](https://jupyter.org/) [documentation](https://jupyter.org/documentation) or [help channels](https://jupyter.org/community).

!!! Warning
    Some tasks don't work especially well in notebooks. For example, anything using asynchronous execution is a problem for Jupyter. Also, freqtrade's primary entry point is the shell cli, so using pure python in a notebook bypasses arguments that provide required objects and parameters to helper functions. You may need to set those values or create expected objects manually.

## Recommended workflow

| Task | Tool |
  --- | ---
Bot operations | CLI
Repetitive tasks | Shell scripts
Data analysis & visualization | Notebook

1. Use the CLI to

    * download historical data
    * run a backtest
    * run with real-time data
    * export results

1. Collect these actions in shell scripts

    * save complicated commands with arguments
    * execute multi-step operations
    * automate testing strategies and preparing data for analysis

1. Use a notebook to

    * visualize data
    * mangle and plot to generate insights

## Example utility snippets

### Change directory to root

Jupyter notebooks execute from the notebook directory. The following snippet searches for the project root, so relative paths remain consistent.

```python
import os
from pathlib import Path

# Change directory
# Modify this cell to insure that the output shows the correct path.
# Define all paths relative to the project root shown in the cell output
project_root = "somedir/freqtrade"
i=0
try:
    os.chdir(project_root)
    assert Path('LICENSE').is_file()
except:
    while i<4 and (not Path('LICENSE').is_file()):
        os.chdir(Path(Path.cwd(), '../'))
        i+=1
    project_root = Path.cwd()
print(Path.cwd())
```

### Load multiple configuration files

This option can be useful to inspect the results of passing in multiple configs.
This will also run through the whole Configuration initialization, so the configuration is completely initialized to be passed to other methods.

``` python
import json
from freqtrade.configuration import Configuration

# Load config from multiple files
config = Configuration.from_files(["config1.json", "config2.json"])

# Show the config in memory
print(json.dumps(config['original_config'], indent=2))
```

For Interactive environments, have an additional configuration specifying `user_data_dir` and pass this in last, so you don't have to change directories while running the bot.
Best avoid relative paths, since this starts at the storage location of the jupyter notebook, unless the directory is changed.

``` json
{
    "user_data_dir": "~/.freqtrade/"
}
```

### Further Data analysis documentation

* [Strategy debugging](strategy_analysis_example.md) - also available as Jupyter notebook (`user_data/notebooks/strategy_analysis_example.ipynb`)
* [Plotting](plotting.md)
* [Tag Analysis](advanced-backtesting.md)

Feel free to submit an issue or Pull Request enhancing this document if you would like to share ideas on how to best analyze the data.

---

## File: data-download.md

# Data Downloading

## Getting data for backtesting and hyperopt

To download data (candles / OHLCV) needed for backtesting and hyperoptimization use the `freqtrade download-data` command.

If no additional parameter is specified, freqtrade will download data for `"1m"` and `"5m"` timeframes for the last 30 days.
Exchange and pairs will come from `config.json` (if specified using `-c/--config`).
Without provided configuration, `--exchange` becomes mandatory.

You can use a relative timerange (`--days 20`) or an absolute starting point (`--timerange 20200101-`). For incremental downloads, the relative approach should be used.

!!! Tip "Tip: Updating existing data"
    If you already have backtesting data available in your data-directory and would like to refresh this data up to today, freqtrade will automatically calculate the missing timerange for the existing pairs and the download will occur from the latest available point until "now", neither `--days` or `--timerange` parameters are required. Freqtrade will keep the available data and only download the missing data.  
    If you are updating existing data after inserting new pairs that you have no data for, use the `--new-pairs-days xx` parameter. Specified number of days will be downloaded for new pairs while old pairs will be updated with missing data only.  

### Usage

--8<-- "commands/download-data.md"

!!! Tip "Downloading all data for one quote currency"
    Often, you'll want to download data for all pairs of a specific quote-currency. In such cases, you can use the following shorthand:
    `freqtrade download-data --exchange binance --pairs ".*/USDT" <...>`. The provided "pairs" string will be expanded to contain all active pairs on the exchange.
    To also download data for inactive (delisted) pairs, add `--include-inactive-pairs` to the command.

!!! Note "Startup period"
    `download-data` is a strategy-independent command. The idea is to download a big chunk of data once, and then iteratively increase the amount of data stored.

    For that reason, `download-data` does not care about the "startup-period" defined in a strategy. It's up to the user to download additional days if the backtest should start at a specific point in time (while respecting startup period).

### Start download

A very simple command (assuming an available `config.json` file) can look as follows.

```bash
freqtrade download-data --exchange binance
```

This will download historical candle (OHLCV) data for all the currency pairs defined in the configuration.

Alternatively, specify the pairs directly

```bash
freqtrade download-data --exchange binance --pairs ETH/USDT XRP/USDT BTC/USDT
```

or as regex (in this case, to download all active USDT pairs)

```bash
freqtrade download-data --exchange binance --pairs ".*/USDT"
```

### Other Notes

* To use a different directory than the exchange specific default, use `--datadir user_data/data/some_directory`.
* To change the exchange used to download the historical data from, either use `--exchange <exchange>` - or specify a different configuration file.
* To use `pairs.json` from some other directory, use `--pairs-file some_other_dir/pairs.json`.
* To download historical candle (OHLCV) data for only 10 days, use `--days 10` (defaults to 30 days).
* To download historical candle (OHLCV) data from a fixed starting point, use `--timerange 20200101-` - which will download all data from January 1st, 2020.
* Given starting points are ignored if data is already available, downloading only missing data up to today.
* Use `--timeframes` to specify what timeframe download the historical candle (OHLCV) data for. Default is `--timeframes 1m 5m` which will download 1-minute and 5-minute data.
* To use exchange, timeframe and list of pairs as defined in your configuration file, use the `-c/--config` option. With this, the script uses the whitelist defined in the config as the list of currency pairs to download data for and does not require the pairs.json file. You can combine `-c/--config` with most other options.

??? Note "Permission denied errors"
    If your configuration directory `user_data` was made by docker, you may get the following error:

    ```
    cp: cannot create regular file 'user_data/data/binance/pairs.json': Permission denied
    ```

    You can fix the permissions of your user-data directory as follows:

    ```
    sudo chown -R $UID:$GID user_data
    ```

### Download additional data before the current timerange

Assuming you downloaded all data from 2022 (`--timerange 20220101-`) - but you'd now like to also backtest with earlier data.
You can do so by using the `--prepend` flag, combined with `--timerange` - specifying an end-date.

``` bash
freqtrade download-data --exchange binance --pairs ETH/USDT XRP/USDT BTC/USDT --prepend --timerange 20210101-20220101
```

!!! Note
    Freqtrade will ignore the end-date in this mode if data is available, updating the end-date to the existing data start point.

### Data format

Freqtrade currently supports the following data-formats:

* `feather` - a dataformat based on Apache Arrow
* `json` -  plain "text" json files
* `jsongz` - a gzip-zipped version of json files
* `parquet` - columnar datastore (OHLCV only)

By default, both OHLCV data and trades data are stored in the `feather` format.

This can be changed via the `--data-format-ohlcv` and `--data-format-trades` command line arguments respectively.
To persist this change, you should also add the following snippet to your configuration, so you don't have to insert the above arguments each time:

``` jsonc
    // ...
    "dataformat_ohlcv": "feather",
    "dataformat_trades": "feather",
    // ...
```

If the default data-format has been changed during download, then the keys `dataformat_ohlcv` and `dataformat_trades` in the configuration file need to be adjusted to the selected dataformat as well.

!!! Note
    You can convert between data-formats using the [convert-data](#sub-command-convert-data) and [convert-trade-data](#sub-command-convert-trade-data) methods.

#### Dataformat comparison

The following comparisons have been made with the following data, and by using the linux `time` command.

```
Found 6 pair / timeframe combinations.
+----------+-------------+--------+---------------------+---------------------+
|     Pair |   Timeframe |   Type |                From |                  To |
|----------+-------------+--------+---------------------+---------------------|
| BTC/USDT |          5m |   spot | 2017-08-17 04:00:00 | 2022-09-13 19:25:00 |
| ETH/USDT |          1m |   spot | 2017-08-17 04:00:00 | 2022-09-13 19:26:00 |
| BTC/USDT |          1m |   spot | 2017-08-17 04:00:00 | 2022-09-13 19:30:00 |
| XRP/USDT |          5m |   spot | 2018-05-04 08:10:00 | 2022-09-13 19:15:00 |
| XRP/USDT |          1m |   spot | 2018-05-04 08:11:00 | 2022-09-13 19:22:00 |
| ETH/USDT |          5m |   spot | 2017-08-17 04:00:00 | 2022-09-13 19:20:00 |
+----------+-------------+--------+---------------------+---------------------+
```

Timings have been taken in a not very scientific way with the following command, which forces reading the data into memory.

``` bash
time freqtrade list-data --show-timerange --data-format-ohlcv <dataformat>
```

|  Format | Size | timing |
|------------|-------------|-------------|
| `feather` | 72Mb | 3.5s |
| `json` | 149Mb | 25.6s |
| `jsongz` | 39Mb | 27s |
| `parquet` | 83Mb | 3.8s |

Size has been taken from the BTC/USDT 1m spot combination for the timerange specified above.

To have a best performance/size mix, we recommend using the default feather format, or parquet.

### Pairs file

In alternative to the whitelist from `config.json`, a `pairs.json` file can be used.
If you are using Binance for example:

* create a directory `user_data/data/binance` and copy or create the `pairs.json` file in that directory.
* update the `pairs.json` file to contain the currency pairs you are interested in.

```bash
mkdir -p user_data/data/binance
touch user_data/data/binance/pairs.json
```

The format of the `pairs.json` file is a simple json list.
Mixing different stake-currencies is allowed for this file, since it's only used for downloading.

``` json
[
    "ETH/BTC",
    "ETH/USDT",
    "BTC/USDT",
    "XRP/ETH"
]
```

!!! Note
    The `pairs.json` file is only used when no configuration is loaded (implicitly by naming, or via `--config` flag).
    You can force the usage of this file via `--pairs-file pairs.json` - however we recommend to use the pairlist from within the configuration, either via `exchange.pair_whitelist` or `pairs` setting in the configuration.

## Sub-command convert data

--8<-- "commands/convert-data.md"

### Example converting data

The following command will convert all candle (OHLCV) data available in `~/.freqtrade/data/binance` from json to jsongz, saving diskspace in the process.
It'll also remove original json data files (`--erase` parameter).

``` bash
freqtrade convert-data --format-from json --format-to jsongz --datadir ~/.freqtrade/data/binance -t 5m 15m --erase
```

## Sub-command convert trade data

--8<-- "commands/convert-trade-data.md"

### Example converting trades

The following command will convert all available trade-data in `~/.freqtrade/data/kraken` from jsongz to json.
It'll also remove original jsongz data files (`--erase` parameter).

``` bash
freqtrade convert-trade-data --format-from jsongz --format-to json --datadir ~/.freqtrade/data/kraken --erase
```

## Sub-command trades to ohlcv

When you need to use `--dl-trades` (kraken only) to download data, conversion of trades data to ohlcv data is the last step.
This command will allow you to repeat this last step for additional timeframes without re-downloading the data.

--8<-- "commands/trades-to-ohlcv.md"

### Example trade-to-ohlcv conversion

``` bash
freqtrade trades-to-ohlcv --exchange kraken -t 5m 1h 1d --pairs BTC/EUR ETH/EUR
```

## Sub-command list-data

You can get a list of downloaded data using the `list-data` sub-command.

--8<-- "commands/list-data.md"

### Example list-data

```bash
> freqtrade list-data --userdir ~/.freqtrade/user_data/

              Found 33 pair / timeframe combinations.
┏━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━┓
┃          Pair ┃                                 Timeframe ┃ Type ┃
┡━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━┩
│       ADA/BTC │     5m, 15m, 30m, 1h, 2h, 4h, 6h, 12h, 1d │ spot │
│       ADA/ETH │     5m, 15m, 30m, 1h, 2h, 4h, 6h, 12h, 1d │ spot │
│       ETH/BTC │     5m, 15m, 30m, 1h, 2h, 4h, 6h, 12h, 1d │ spot │
│      ETH/USDT │                  5m, 15m, 30m, 1h, 2h, 4h │ spot │
└───────────────┴───────────────────────────────────────────┴──────┘

```

Show all trades data including from/to timerange

``` bash
> freqtrade list-data --show --trades
                     Found trades data for 1 pair.                     
┏━━━━━━━━━┳━━━━━━┳━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━┓
┃    Pair ┃ Type ┃                From ┃                  To ┃ Trades ┃
┡━━━━━━━━━╇━━━━━━╇━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━┩
│ XRP/ETH │ spot │ 2019-10-11 00:00:11 │ 2019-10-13 11:19:28 │  12477 │
└─────────┴──────┴─────────────────────┴─────────────────────┴────────┘

```

## Trades (tick) data

By default, `download-data` sub-command downloads Candles (OHLCV) data. Most exchanges also provide historic trade-data via their API.
This data can be useful if you need many different timeframes, since it is only downloaded once, and then resampled locally to the desired timeframes.

Since this data is large by default, the files use the feather file format by default. They are stored in your data-directory with the naming convention of `<pair>-trades.feather` (`ETH_BTC-trades.feather`). Incremental mode is also supported, as for historic OHLCV data, so downloading the data once per week with `--days 8` will create an incremental data-repository.

To use this mode, simply add `--dl-trades` to your call. This will swap the download method to download trades.
If `--convert` is also provided, the resample step will happen automatically and overwrite eventually existing OHLCV data for the given pair/timeframe combinations.

!!! Warning "Do not use"
    You should not use this unless you're a kraken user (Kraken does not provide historic OHLCV data).  
    Most other exchanges provide OHLCV data with sufficient history, so downloading multiple timeframes through that method will still proof to be a lot faster than downloading trades data.

!!! Note "Kraken user"
    Kraken users should read [this](exchanges.md#historic-kraken-data) before starting to download data.

Example call:

```bash
freqtrade download-data --exchange kraken --pairs XRP/EUR ETH/EUR --days 20 --dl-trades
```

!!! Note
    While this method uses async calls, it will be slow, since it requires the result of the previous call to generate the next request to the exchange.

## Next step

Great, you now have some data downloaded, so you can now start [backtesting](backtesting.md) your strategy.

---

## File: deprecated.md

# Deprecated features

This page contains description of the command line arguments, configuration parameters
and the bot features that were declared as DEPRECATED by the bot development team
and are no longer supported. Please avoid their usage in your configuration.

## Removed features

### the `--refresh-pairs-cached` command line option

`--refresh-pairs-cached` in the context of backtesting, hyperopt and edge allows to refresh candle data for backtesting.
Since this leads to much confusion, and slows down backtesting (while not being part of backtesting) this has been singled out as a separate freqtrade sub-command `freqtrade download-data`.

This command line option was deprecated in 2019.7-dev (develop branch) and removed in 2019.9.

### The **--dynamic-whitelist** command line option

This command line option was deprecated in 2018 and removed freqtrade 2019.6-dev (develop branch) and in freqtrade 2019.7.
Please refer to [pairlists](plugins.md#pairlists-and-pairlist-handlers) instead.

### the `--live` command line option

`--live` in the context of backtesting allowed to download the latest tick data for backtesting.
Did only download the latest 500 candles, so was ineffective in getting good backtest data.
Removed in 2019-7-dev (develop branch) and in freqtrade 2019.8.

### `ticker_interval` (now `timeframe`)

Support for `ticker_interval` terminology was deprecated in 2020.6 in favor of `timeframe` - and compatibility code was removed in 2022.3.

### Allow running multiple pairlists in sequence

The former `"pairlist"` section in the configuration has been removed, and is replaced by `"pairlists"` - being a list to specify a sequence of pairlists.

The old section of configuration parameters (`"pairlist"`) has been deprecated in 2019.11 and has been removed in 2020.4.

### deprecation of bidVolume and askVolume from volume-pairlist

Since only quoteVolume can be compared between assets, the other options (bidVolume, askVolume) have been deprecated in 2020.4, and have been removed in 2020.9.

### Using order book steps for exit price

Using `order_book_min` and `order_book_max` used to allow stepping the orderbook and trying to find the next ROI slot - trying to place sell-orders early.
As this does however increase risk and provides no benefit, it's been removed for maintainability purposes in 2021.7.

### Legacy Hyperopt mode

Using separate hyperopt files was deprecated in 2021.4 and was removed in 2021.9.
Please switch to the new [Parametrized Strategies](hyperopt.md) to benefit from the new hyperopt interface.

## Strategy changes between V2 and V3

Isolated Futures / short trading was introduced in 2022.4. This required major changes to configuration settings, strategy interfaces, ...

We have put a great effort into keeping compatibility with existing strategies, so if you just want to continue using freqtrade in spot markets, there are no changes necessary.
While we may drop support for the current interface sometime in the future, we will announce this separately and have an appropriate transition period.

Please follow the [Strategy migration](strategy_migration.md) guide to migrate your strategy to the new format to start using the new functionalities.

### webhooks - changes with 2022.4

#### `buy_tag` has been renamed to `enter_tag`

This should apply only to your strategy and potentially to webhooks.
We will keep a compatibility layer for 1-2 versions (so both `buy_tag` and `enter_tag` will still work), but support for this in webhooks will disappear after that.

#### Naming changes

Webhook terminology changed from "sell" to "exit", and from "buy" to "entry", removing "webhook" in the process.

* `webhookbuy`, `webhookentry` -> `entry`
* `webhookbuyfill`, `webhookentryfill` -> `entry_fill`
* `webhookbuycancel`, `webhookentrycancel` -> `entry_cancel`
* `webhooksell`, `webhookexit` -> `exit`
* `webhooksellfill`, `webhookexitfill` -> `exit_fill`
* `webhooksellcancel`, `webhookexitcancel` -> `exit_cancel`

## Removal of `populate_any_indicators`

version 2023.3 saw the removal of `populate_any_indicators` in favor of split methods for feature engineering and targets. Please read the [migration document](strategy_migration.md#freqai-strategy) for full details.

## Removal of `protections` from configuration

Setting protections from the configuration via `"protections": [],` has been removed in 2024.10, after having raised deprecation warnings for over 3 years.

## hdf5 data storage

Using hdf5 as data storage has been deprecated in 2024.12 and was removed in 2025.1. We recommend switching to the feather data format.

Please use the [`convert-data` subcommand](data-download.md#sub-command-convert-data) to convert your existing data to one of the supported formats before updating.

## Configuring advanced logging via config

Configuring syslog and journald via `--logfile systemd` and `--logfile journald` respectively has been deprecated in 2025.3.
Please use configuration based [log setup](advanced-setup.md#advanced-logging) instead.

---

## File: developer.md

# Development Help

This page is intended for developers of Freqtrade, people who want to contribute to the Freqtrade codebase or documentation, or people who want to understand the source code of the application they're running.

All contributions, bug reports, bug fixes, documentation improvements, enhancements and ideas are welcome. We [track issues](https://github.com/freqtrade/freqtrade/issues) on [GitHub](https://github.com) and also have a dev channel on [discord](https://discord.gg/p7nuUNVfP7) where you can ask questions.

## Documentation

Documentation is available at [https://freqtrade.io](https://www.freqtrade.io/) and needs to be provided with every new feature PR.

Special fields for the documentation (like Note boxes, ...) can be found [here](https://squidfunk.github.io/mkdocs-material/reference/admonitions/).

To test the documentation locally use the following commands.

``` bash
pip install -r docs/requirements-docs.txt
mkdocs serve
```

This will spin up a local server (usually on port 8000) so you can see if everything looks as you'd like it to.

## Developer setup

To configure a development environment, you can either use the provided [DevContainer](#devcontainer-setup), or use the `setup.sh` script and answer "y" when asked "Do you want to install dependencies for dev [y/N]? ".
Alternatively (e.g. if your system is not supported by the setup.sh script), follow the manual installation process and run `pip3 install -r requirements-dev.txt` - followed by `pip3 install -e .[all]`.

This will install all required tools for development, including `pytest`, `ruff`, `mypy`, and `coveralls`.

Then install the git hook scripts by running `pre-commit install`, so your changes will be verified locally before committing.
This avoids a lot of waiting for CI already, as some basic formatting checks are done locally on your machine.

Before opening a pull request, please familiarize yourself with our [Contributing Guidelines](https://github.com/freqtrade/freqtrade/blob/develop/CONTRIBUTING.md).

### Devcontainer setup

The fastest and easiest way to get started is to use [VSCode](https://code.visualstudio.com/) with the Remote container extension.
This gives developers the ability to start the bot with all required dependencies *without* needing to install any freqtrade specific dependencies on your local machine.

#### Devcontainer dependencies

* [VSCode](https://code.visualstudio.com/)
* [docker](https://docs.docker.com/install/)
* [Remote container extension documentation](https://code.visualstudio.com/docs/remote)

For more information about the [Remote container extension](https://code.visualstudio.com/docs/remote), best consult the documentation.

### Tests

New code should be covered by basic unittests. Depending on the complexity of the feature, Reviewers may request more in-depth unittests.
If necessary, the Freqtrade team can assist and give guidance with writing good tests (however please don't expect anyone to write the tests for you).

#### How to run tests

Use `pytest` in root folder to run all available testcases and confirm your local environment is setup correctly

!!! Note "feature branches"
    Tests are expected to pass on the `develop` and `stable` branches. Other branches may be work in progress with tests not working yet.

#### Checking log content in tests

Freqtrade uses 2 main methods to check log content in tests, `log_has()` and `log_has_re()` (to check using regex, in case of dynamic log-messages).
These are available from `conftest.py` and can be imported in any test module.

A sample check looks as follows:

``` python
from tests.conftest import log_has, log_has_re

def test_method_to_test(caplog):
    method_to_test()

    assert log_has("This event happened", caplog)
    # Check regex with trailing number ...
    assert log_has_re(r"This dynamic event happened and produced \d+", caplog)

```

### Debug configuration

To debug freqtrade, we recommend VSCode (with the Python extension) with the following launch configuration (located in `.vscode/launch.json`).
Details will obviously vary between setups - but this should work to get you started.

``` json
{
    "name": "freqtrade trade",
    "type": "debugpy",
    "request": "launch",
    "module": "freqtrade",
    "console": "integratedTerminal",
    "args": [
        "trade",
        // Optional:
        // "--userdir", "user_data",
        "--strategy", 
        "MyAwesomeStrategy",
    ]
},
```

Command line arguments can be added in the `"args"` array.
This method can also be used to debug a strategy, by setting the breakpoints within the strategy.

A similar setup can also be taken for Pycharm - using `freqtrade` as module name, and setting the command line arguments as "parameters".

??? Tip "Correct venv usage"
    When using a virtual environment (which you should), make sure that your Editor is using the correct virtual environment to avoid problems or "unknown import" errors.

    #### Vscode

    You can select the correct environment in VSCode with the command "Python: Select Interpreter" - which will show you environments the extension detected.
    If your environment has not been detected, you can also pick a path manually.

    #### Pycharm

    In pycharm, you can select the appropriate Environment in the "Run/Debug Configurations" window.
    ![Pycharm debug configuration](assets/pycharm_debug.png)

!!! Note "Startup directory"
    This assumes that you have the repository checked out, and the editor is started at the repository root level (so pyproject.toml is at the top level of your repository).

## ErrorHandling

Freqtrade Exceptions all inherit from `FreqtradeException`.
This general class of error should however not be used directly. Instead, multiple specialized sub-Exceptions exist.

Below is an outline of exception inheritance hierarchy:

```
+ FreqtradeException
|
+---+ OperationalException
|   |
|   +---+ ConfigurationError
|
+---+ DependencyException
|   |
|   +---+ PricingError
|   |
|   +---+ ExchangeError
|       |
|       +---+ TemporaryError
|       |
|       +---+ DDosProtection
|       |
|       +---+ InvalidOrderException
|           |
|           +---+ RetryableOrderError
|           |
|           +---+ InsufficientFundsError
|
+---+ StrategyError
```

---

## Plugins

### Pairlists

You have a great idea for a new pair selection algorithm you would like to try out? Great.
Hopefully you also want to contribute this back upstream.

Whatever your motivations are - This should get you off the ground in trying to develop a new Pairlist Handler.

First of all, have a look at the [VolumePairList](https://github.com/freqtrade/freqtrade/blob/develop/freqtrade/plugins/pairlist/VolumePairList.py) Handler, and best copy this file with a name of your new Pairlist Handler.

This is a simple Handler, which however serves as a good example on how to start developing.

Next, modify the class-name of the Handler (ideally align this with the module filename).

The base-class provides an instance of the exchange (`self._exchange`) the pairlist manager (`self._pairlistmanager`), as well as the main configuration (`self._config`), the pairlist dedicated configuration (`self._pairlistconfig`) and the absolute position within the list of pairlists.

```python
        self._exchange = exchange
        self._pairlistmanager = pairlistmanager
        self._config = config
        self._pairlistconfig = pairlistconfig
        self._pairlist_pos = pairlist_pos
```

!!! Tip
    Don't forget to register your pairlist in `constants.py` under the variable `AVAILABLE_PAIRLISTS` - otherwise it will not be selectable.

Now, let's step through the methods which require actions:

#### Pairlist configuration

Configuration for the chain of Pairlist Handlers is done in the bot configuration file in the element `"pairlists"`, an array of configuration parameters for each Pairlist Handlers in the chain.

By convention, `"number_assets"` is used to specify the maximum number of pairs to keep in the pairlist. Please follow this to ensure a consistent user experience.

Additional parameters can be configured as needed. For instance, `VolumePairList` uses `"sort_key"` to specify the sorting value - however feel free to specify whatever is necessary for your great algorithm to be successful and dynamic.

#### short_desc

Returns a description used for Telegram messages.

This should contain the name of the Pairlist Handler, as well as a short description containing the number of assets. Please follow the format `"PairlistName - top/bottom X pairs"`.

#### gen_pairlist

Override this method if the Pairlist Handler can be used as the leading Pairlist Handler in the chain, defining the initial pairlist which is then handled by all Pairlist Handlers in the chain. Examples are `StaticPairList` and `VolumePairList`.

This is called with each iteration of the bot (only if the Pairlist Handler is at the first location) - so consider implementing caching for compute/network heavy calculations.

It must return the resulting pairlist (which may then be passed into the chain of Pairlist Handlers).

Validations are optional, the parent class exposes a `verify_blacklist(pairlist)` and `_whitelist_for_active_markets(pairlist)` to do default filtering. Use this if you limit your result to a certain number of pairs - so the end-result is not shorter than expected.

#### filter_pairlist

This method is called for each Pairlist Handler in the chain by the pairlist manager.

This is called with each iteration of the bot - so consider implementing caching for compute/network heavy calculations.

It gets passed a pairlist (which can be the result of previous pairlists) as well as `tickers`, a pre-fetched version of `get_tickers()`.

The default implementation in the base class simply calls the `_validate_pair()` method for each pair in the pairlist, but you may override it. So you should either implement the `_validate_pair()` in your Pairlist Handler or override `filter_pairlist()` to do something else.

If overridden, it must return the resulting pairlist (which may then be passed into the next Pairlist Handler in the chain).

Validations are optional, the parent class exposes a `verify_blacklist(pairlist)` and `_whitelist_for_active_markets(pairlist)` to do default filters. Use this if you limit your result to a certain number of pairs - so the end result is not shorter than expected.

In `VolumePairList`, this implements different methods of sorting, does early validation so only the expected number of pairs is returned.

##### sample

``` python
    def filter_pairlist(self, pairlist: list[str], tickers: dict) -> List[str]:
        # Generate dynamic whitelist
        pairs = self._calculate_pairlist(pairlist, tickers)
        return pairs
```

### Protections

Best read the [Protection documentation](plugins.md#protections) to understand protections.
This Guide is directed towards Developers who want to develop a new protection.

No protection should use datetime directly, but use the provided `date_now` variable for date calculations. This preserves the ability to backtest protections.

!!! Tip "Writing a new Protection"
    Best copy one of the existing Protections to have a good example.

#### Implementation of a new protection

All Protection implementations must have `IProtection` as parent class.
For that reason, they must implement the following methods:

* `short_desc()`
* `global_stop()`
* `stop_per_pair()`.

`global_stop()` and `stop_per_pair()` must return a ProtectionReturn object, which consists of:

* lock pair - boolean
* lock until - datetime - until when should the pair be locked (will be rounded up to the next new candle)
* reason - string, used for logging and storage in the database
* lock_side - long, short or '*'.

The `until` portion should be calculated using the provided `calculate_lock_end()` method.

All Protections should use `"stop_duration"` / `"stop_duration_candles"` to define how long a pair (or all pairs) should be locked.
The content of this is made available as `self._stop_duration` to the each Protection.

If your protection requires a look-back period, please use `"lookback_period"` / `"lockback_period_candles"` to keep all protections aligned.

#### Global vs. local stops

Protections can have 2 different ways to stop trading for a limited :

* Per pair (local)
* For all Pairs (globally)

##### Protections - per pair

Protections that implement the per pair approach must set `has_local_stop=True`.
The method `stop_per_pair()` will be called whenever a trade closed (exit order completed).

##### Protections - global protection

These Protections should do their evaluation across all pairs, and consequently will also lock all pairs from trading (called a global PairLock).
Global protection must set `has_global_stop=True` to be evaluated for global stops.
The method `global_stop()` will be called whenever a trade closed (exit order completed).

##### Protections - calculating lock end time

Protections should calculate the lock end time based on the last trade it considers.
This avoids re-locking should the lookback-period be longer than the actual lock period.

The `IProtection` parent class provides a helper method for this in `calculate_lock_end()`.

---

## Implement a new Exchange (WIP)

!!! Note
    This section is a Work in Progress and is not a complete guide on how to test a new exchange with Freqtrade.

!!! Note
    Make sure to use an up-to-date version of CCXT before running any of the below tests.
    You can get the latest version of ccxt by running `pip install -U ccxt` with activated virtual environment.
    Native docker is not supported for these tests, however the available dev-container will support all required actions and eventually necessary changes.

Most exchanges supported by CCXT should work out of the box.

To quickly test the public endpoints of an exchange, add a configuration for your exchange to `tests/exchange_online/conftest.py` and run these tests with `pytest --longrun tests/exchange_online/test_ccxt_compat.py`.
Completing these tests successfully a good basis point (it's a requirement, actually), however these won't guarantee correct exchange functioning, as this only tests public endpoints, but no private endpoint (like generate order or similar).

Also try to use `freqtrade download-data` for an extended timerange (multiple months) and verify that the data downloaded correctly (no holes, the specified timerange was actually downloaded).

These are prerequisites to have an exchange listed as either Supported or Community tested (listed on the homepage).
The below are "extras", which will make an exchange better (feature-complete) - but are not absolutely necessary for either of the 2 categories.

Additional tests / steps to complete:

* Verify data provided by `fetch_ohlcv()` - and eventually adjust `ohlcv_candle_limit` for this exchange
* Check L2 orderbook limit range (API documentation) - and eventually set as necessary
* Check if balance shows correctly (*)
* Create market order (*)
* Create limit order (*)
* Cancel order (*)
* Complete trade (enter + exit) (*)
  * Compare result calculation between exchange and bot
  * Ensure fees are applied correctly (check the database against the exchange)

(*) Requires API keys and Balance on the exchange.

### Stoploss On Exchange

Check if the new exchange supports Stoploss on Exchange orders through their API.

Since CCXT does not provide unification for Stoploss On Exchange yet, we'll need to implement the exchange-specific parameters ourselves. Best look at `binance.py` for an example implementation of this. You'll need to dig through the documentation of the Exchange's API on how exactly this can be done. [CCXT Issues](https://github.com/ccxt/ccxt/issues) may also provide great help, since others may have implemented something similar for their projects.

### Incomplete candles

While fetching candle (OHLCV) data, we may end up getting incomplete candles (depending on the exchange).
To demonstrate this, we'll use daily candles (`"1d"`) to keep things simple.
We query the api (`ct.fetch_ohlcv()`) for the timeframe and look at the date of the last entry. If this entry changes or shows the date of a "incomplete" candle, then we should drop this since having incomplete candles is problematic because indicators assume that only complete candles are passed to them, and will generate a lot of false buy signals. By default, we're therefore removing the last candle assuming it's incomplete.

To check how the new exchange behaves, you can use the following snippet:

``` python
import ccxt
from datetime import datetime, timezone
from freqtrade.data.converter import ohlcv_to_dataframe
ct = ccxt.binance()  # Use the exchange you're testing
timeframe = "1d"
pair = "BTC/USDT"  # Make sure to use a pair that exists on that exchange!
raw = ct.fetch_ohlcv(pair, timeframe=timeframe)

# convert to dataframe
df1 = ohlcv_to_dataframe(raw, timeframe, pair=pair, drop_incomplete=False)

print(df1.tail(1))
print(datetime.now(timezone.utc))
```

``` output
                         date      open      high       low     close  volume  
499 2019-06-08 00:00:00+00:00  0.000007  0.000007  0.000007  0.000007   26264344.0  
2019-06-09 12:30:27.873327
```

The output will show the last entry from the Exchange as well as the current UTC date.
If the day shows the same day, then the last candle can be assumed as incomplete and should be dropped (leave the setting `"ohlcv_partial_candle"` from the exchange-class untouched / True). Otherwise, set `"ohlcv_partial_candle"` to `False` to not drop Candles (shown in the example above).
Another way is to run this command multiple times in a row and observe if the volume is changing (while the date remains the same).

### Update binance cached leverage tiers

Updating leveraged tiers should be done regularly - and requires an authenticated account with futures enabled.

``` python
import ccxt
import json
from pathlib import Path

exchange = ccxt.binance({
    'apiKey': '<apikey>',
    'secret': '<secret>',
    'options': {'defaultType': 'swap'}
    })
_ = exchange.load_markets()

lev_tiers = exchange.fetch_leverage_tiers()

# Assumes this is running in the root of the repository.
file = Path('freqtrade/exchange/binance_leverage_tiers.json')
json.dump(dict(sorted(lev_tiers.items())), file.open('w'), indent=2)

```

This file should then be contributed upstream, so others can benefit from this, too.

## Updating example notebooks

To keep the jupyter notebooks aligned with the documentation, the following should be ran after updating a example notebook.

``` bash
jupyter nbconvert --ClearOutputPreprocessor.enabled=True --inplace freqtrade/templates/strategy_analysis_example.ipynb
jupyter nbconvert --ClearOutputPreprocessor.enabled=True --to markdown freqtrade/templates/strategy_analysis_example.ipynb --stdout > docs/strategy_analysis_example.md
```

## Continuous integration

This documents some decisions taken for the CI Pipeline.

* CI runs on all OS variants, Linux (ubuntu), macOS and Windows.
* Docker images are build for the branches `stable` and `develop`, and are built as multiarch builds, supporting multiple platforms via the same tag.
* Docker images containing Plot dependencies are also available as `stable_plot` and `develop_plot`.
* Docker images contain a file, `/freqtrade/freqtrade_commit` containing the commit this image is based of.
* Full docker image rebuilds are run once a week via schedule.
* Deployments run on ubuntu.
* ta-lib binaries are contained in the build_helpers directory to avoid fails related to external unavailability.
* All tests must pass for a PR to be merged to `stable` or `develop`.

## Creating a release

This part of the documentation is aimed at maintainers, and shows how to create a release.

### Create release branch

!!! Note
    Make sure that the `stable` branch is up-to-date!

First, pick a commit that's about one week old (to not include latest additions to releases).

``` bash
# create new branch
git checkout -b new_release <commitid>
```

Determine if crucial bugfixes have been made between this commit and the current state, and eventually cherry-pick these.

* Merge the release branch (stable) into this branch.
* Edit `freqtrade/__init__.py` and add the version matching the current date (for example `2019.7` for July 2019). Minor versions can be `2019.7.1` should we need to do a second release that month. Version numbers must follow allowed versions from PEP0440 to avoid failures pushing to pypi.
* Commit this part.
* Push that branch to the remote and create a PR against the **stable branch**.
* Update develop version to next version following the pattern `2019.8-dev`.

### Create changelog from git commits

``` bash
# Needs to be done before merging / pulling that branch.
git log --oneline --no-decorate --no-merges stable..new_release
```

To keep the release-log short, best wrap the full git changelog into a collapsible details section.

```markdown
<details>
<summary>Expand full changelog</summary>

... Full git changelog

</details>
```

### FreqUI release

If FreqUI has been updated substantially, make sure to create a release before merging the release branch.
Make sure that freqUI CI on the release is finished and passed before merging the release.

### Create github release / tag

Once the PR against stable is merged (best right after merging):

* Use the button "Draft a new release" in the Github UI (subsection releases).
* Use the version-number specified as tag.
* Use "stable" as reference (this step comes after the above PR is merged).
* Use the above changelog as release comment (as codeblock).
* Use the below snippet for the new release

??? Tip "Release template"
    ````
    --8<-- "includes/release_template.md"
    ````

## Releases

### pypi

!!! Warning "Manual Releases"
    This process is automated as part of Github Actions.  
    Manual pypi pushes should not be necessary.

??? example "Manual release"
    To manually create a pypi release, please run the following commands:

    Additional requirement: `wheel`, `twine` (for uploading), account on pypi with proper permissions.

    ``` bash
    pip install -U build
    python -m build --sdist --wheel

    # For pypi test (to check if some change to the installation did work)
    twine upload --repository-url https://test.pypi.org/legacy/ dist/*

    # For production:
    twine upload dist/*
    ```

    Please don't push non-releases to the productive / real pypi instance.

---

## File: docker_quickstart.md

# Using Freqtrade with Docker

This page explains how to run the bot with Docker. It is not meant to work out of the box. You'll still need to read through the documentation and understand how to properly configure it.

## Install Docker

Start by downloading and installing Docker / Docker Desktop for your platform:

* [Mac](https://docs.docker.com/docker-for-mac/install/)
* [Windows](https://docs.docker.com/docker-for-windows/install/)
* [Linux](https://docs.docker.com/install/)

!!! Info "Docker compose install"
    Freqtrade documentation assumes the use of Docker desktop (or the docker compose plugin).  
    While the docker-compose standalone installation still works, it will require changing all `docker compose` commands from `docker compose` to `docker-compose` to work (e.g. `docker compose up -d` will become `docker-compose up -d`).

??? Warning "Docker on windows"
    If you just installed docker on a windows system, make sure to reboot your system, otherwise you might encounter unexplainable Problems related to network connectivity to docker containers.

## Freqtrade with docker

Freqtrade provides an official Docker image on [Dockerhub](https://hub.docker.com/r/freqtradeorg/freqtrade/), as well as a [docker compose file](https://github.com/freqtrade/freqtrade/blob/stable/docker-compose.yml) ready for usage.

!!! Note
    - The following section assumes that `docker` is installed and available to the logged in user.
    - All below commands use relative directories and will have to be executed from the directory containing the `docker-compose.yml` file.

### Docker quick start

Create a new directory and place the [docker-compose file](https://raw.githubusercontent.com/freqtrade/freqtrade/stable/docker-compose.yml) in this directory.

``` bash
mkdir ft_userdata
cd ft_userdata/
# Download the docker-compose file from the repository
curl https://raw.githubusercontent.com/freqtrade/freqtrade/stable/docker-compose.yml -o docker-compose.yml

# Pull the freqtrade image
docker compose pull

# Create user directory structure
docker compose run --rm freqtrade create-userdir --userdir user_data

# Create configuration - Requires answering interactive questions
docker compose run --rm freqtrade new-config --config user_data/config.json
```

The above snippet creates a new directory called `ft_userdata`, downloads the latest compose file and pulls the freqtrade image.
The last 2 steps in the snippet create the directory with `user_data`, as well as (interactively) the default configuration based on your selections.

!!! Question "How to edit the bot configuration?"
    You can edit the configuration at any time, which is available as `user_data/config.json` (within the directory `ft_userdata`) when using the above configuration.

    You can also change the both Strategy and commands by editing the command section of your `docker-compose.yml` file.

#### Adding a custom strategy

1. The configuration is now available as `user_data/config.json`
2. Copy a custom strategy to the directory `user_data/strategies/`
3. Add the Strategy' class name to the `docker-compose.yml` file

The `SampleStrategy` is run by default.

!!! Danger "`SampleStrategy` is just a demo!"
    The `SampleStrategy` is there for your reference and give you ideas for your own strategy.
    Please always backtest your strategy and use dry-run for some time before risking real money!
    You will find more information about Strategy development in the [Strategy documentation](strategy-customization.md).

Once this is done, you're ready to launch the bot in trading mode (Dry-run or Live-trading, depending on your answer to the corresponding question you made above).

``` bash
docker compose up -d
```

!!! Warning "Default configuration"
    While the configuration generated will be mostly functional, you will still need to verify that all options correspond to what you want (like Pricing, pairlist, ...) before starting the bot.

#### Accessing the UI

If you've selected to enable FreqUI in the `new-config` step, you will have freqUI available at port `localhost:8080`.

You can now access the UI by typing localhost:8080 in your browser.

??? Note "UI Access on a remote server"
    If you're running on a VPS, you should consider using either a ssh tunnel, or setup a VPN (openVPN, wireguard) to connect to your bot.
    This will ensure that freqUI is not directly exposed to the internet, which is not recommended for security reasons (freqUI does not support https out of the box).
    Setup of these tools is not part of this tutorial, however many good tutorials can be found on the internet.
    Please also read the [API configuration with docker](rest-api.md#configuration-with-docker) section to learn more about this configuration.

#### Monitoring the bot

You can check for running instances with `docker compose ps`.
This should list the service `freqtrade` as `running`. If that's not the case, best check the logs (see next point).

#### Docker compose logs

Logs will be written to: `user_data/logs/freqtrade.log`.  
You can also check the latest log with the command `docker compose logs -f`.

#### Database

The database will be located at: `user_data/tradesv3.sqlite`

#### Updating freqtrade with docker

Updating freqtrade when using `docker` is as simple as running the following 2 commands:

``` bash
# Download the latest image
docker compose pull
# Restart the image
docker compose up -d
```

This will first pull the latest image, and will then restart the container with the just pulled version.

!!! Warning "Check the Changelog"
    You should always check the changelog for breaking changes / manual interventions required and make sure the bot starts correctly after the update.

### Editing the docker-compose file

Advanced users may edit the docker-compose file further to include all possible options or arguments.

All freqtrade arguments will be available by running `docker compose run --rm freqtrade <command> <optional arguments>`.

!!! Warning "`docker compose` for trade commands"
    Trade commands (`freqtrade trade <...>`) should not be ran via `docker compose run` - but should use `docker compose up -d` instead.
    This makes sure that the container is properly started (including port forwardings) and will make sure that the container will restart after a system reboot.
    If you intend to use freqUI, please also ensure to adjust the [configuration accordingly](rest-api.md#configuration-with-docker), otherwise the UI will not be available.

!!! Note "`docker compose run --rm`"
    Including `--rm` will remove the container after completion, and is highly recommended for all modes except trading mode (running with `freqtrade trade` command).

??? Note "Using docker without docker compose"
    "`docker compose run --rm`" will require a compose file to be provided.
    Some freqtrade commands that don't require authentication such as `list-pairs` can be run with "`docker run --rm`" instead.  
    For example `docker run --rm freqtradeorg/freqtrade:stable list-pairs --exchange binance --quote BTC --print-json`.  
    This can be useful for fetching exchange information to add to your `config.json` without affecting your running containers.

#### Example: Download data with docker

Download backtesting data for 5 days for the pair ETH/BTC and 1h timeframe from Binance. The data will be stored in the directory `user_data/data/` on the host.

``` bash
docker compose run --rm freqtrade download-data --pairs ETH/BTC --exchange binance --days 5 -t 1h
```

Head over to the [Data Downloading Documentation](data-download.md) for more details on downloading data.

#### Example: Backtest with docker

Run backtesting in docker-containers for SampleStrategy and specified timerange of historical data, on 5m timeframe:

``` bash
docker compose run --rm freqtrade backtesting --config user_data/config.json --strategy SampleStrategy --timerange 20190801-20191001 -i 5m
```

Head over to the [Backtesting Documentation](backtesting.md) to learn more.

### Additional dependencies with docker

If your strategy requires dependencies not included in the default image - it will be necessary to build the image on your host.
For this, please create a Dockerfile containing installation steps for the additional dependencies (have a look at [docker/Dockerfile.custom](https://github.com/freqtrade/freqtrade/blob/develop/docker/Dockerfile.custom) for an example).

You'll then also need to modify the `docker-compose.yml` file and uncomment the build step, as well as rename the image to avoid naming collisions.

``` yaml
    image: freqtrade_custom
    build:
      context: .
      dockerfile: "./Dockerfile.<yourextension>"
```

You can then run `docker compose build --pull` to build the docker image, and run it using the commands described above.

### Plotting with docker

Commands `freqtrade plot-profit` and `freqtrade plot-dataframe` ([Documentation](plotting.md)) are available by changing the image to `*_plot` in your `docker-compose.yml` file.
You can then use these commands as follows:

``` bash
docker compose run --rm freqtrade plot-dataframe --strategy AwesomeStrategy -p BTC/ETH --timerange=20180801-20180805
```

The output will be stored in the `user_data/plot` directory, and can be opened with any modern browser.

### Data analysis using docker compose

Freqtrade provides a docker-compose file which starts up a jupyter lab server.
You can run this server using the following command:

``` bash
docker compose -f docker/docker-compose-jupyter.yml up
```

This will create a docker-container running jupyter lab, which will be accessible using `https://127.0.0.1:8888/lab`.
Please use the link that's printed in the console after startup for simplified login.

Since part of this image is built on your machine, it is recommended to rebuild the image from time to time to keep freqtrade (and dependencies) up-to-date.

``` bash
docker compose -f docker/docker-compose-jupyter.yml build --no-cache
```

## Troubleshooting

### Docker on Windows

* Error: `"Timestamp for this request is outside of the recvWindow."`  
  The market api requests require a synchronized clock but the time in the docker container shifts a bit over time into the past.
  To fix this issue temporarily you need to run `wsl --shutdown` and restart docker again (a popup on windows 10 will ask you to do so).
  A permanent solution is either to host the docker container on a linux host or restart the wsl from time to time with the scheduler.

  ``` bash
  taskkill /IM "Docker Desktop.exe" /F
  wsl --shutdown
  start "" "C:\Program Files\Docker\Docker\Docker Desktop.exe"
  ```

* Cannot connect to the API (Windows)  
  If you're on windows and just installed Docker (desktop), make sure to reboot your System. Docker can have problems with network connectivity without a restart.
  You should obviously also make sure to have your [settings](#accessing-the-ui) accordingly.

!!! Warning
    Due to the above, we do not recommend the usage of docker on windows for production setups, but only for experimentation, datadownload and backtesting.
    Best use a linux-VPS for running freqtrade reliably.

---

## File: edge.md

# Edge positioning

The `Edge Positioning` module uses probability to calculate your win rate and risk reward ratio. It will use these statistics to control your strategy trade entry points, position size and, stoploss.

!!! Danger "Deprecated functionality"
    `Edge positioning` (or short Edge) is currently in maintenance mode only (we keep existing functionality alive) and should be considered as deprecated.
    It will currently not receive new features until either someone stepped forward to take up ownership of that module - or we'll decide to remove edge from freqtrade.

!!! Warning
    When using `Edge positioning` with a dynamic whitelist (VolumePairList), make sure to also use `AgeFilter` and set it to at least `calculate_since_number_of_days` to avoid problems with missing data.

!!! Note
    `Edge Positioning` only considers *its own* buy/sell/stoploss signals. It ignores the stoploss, trailing stoploss, and ROI settings in the strategy configuration file.
    `Edge Positioning` improves the performance of some trading strategies and *decreases* the performance of others.


## Introduction

Trading strategies are not perfect. They are frameworks that are susceptible to the market and its indicators. Because the market is not at all predictable, sometimes a strategy will win and sometimes the same strategy will lose.

To obtain an edge in the market, a strategy has to make more money than it loses. Making money in trading is not only about *how often* the strategy makes or loses money.

!!! tip "It doesn't matter how often, but how much!"
    A bad strategy might make 1 penny in *ten* transactions but lose 1 dollar in *one* transaction. If one only checks the number of winning trades, it would be misleading to think that the strategy is actually making a profit.

The Edge Positioning module seeks to improve a strategy's winning probability and the money that the strategy will make *on the long run*. 

We raise the following question[^1]:

!!! Question "Which trade is a better option?"
    a) A trade with 80% of chance of losing 100\$ and 20% chance of winning 200\$<br/>
    b) A trade with 100% of chance of losing 30\$

???+ Info "Answer"
    The expected value of *a)* is smaller than the expected value of *b)*.<br/>
    Hence, *b*) represents a smaller loss in the long run.<br/>
    However, the answer is: *it depends*

Another way to look at it is to ask a similar question:

!!! Question "Which trade is a better option?"
    a) A trade with 80% of chance of winning 100\$ and 20% chance of losing 200\$<br/>
    b) A trade with 100% of chance of winning 30\$

Edge positioning tries to answer the hard questions about risk/reward and position size automatically, seeking to minimizes the chances of losing of a given strategy.

### Trading, winning and losing

Let's call $o$ the return of a single transaction $o$ where $o \in \mathbb{R}$. The collection $O = \{o_1, o_2, ..., o_N\}$ is the set of all returns of transactions made during a trading session. We say that $N$ is the cardinality of $O$, or, in lay terms, it is the number of transactions made in a trading session.

!!! Example
    In a session where a strategy made three transactions we can say that $O = \{3.5, -1, 15\}$. That means that $N = 3$ and $o_1 = 3.5$, $o_2 = -1$, $o_3 = 15$.

A winning trade is a trade where a strategy *made* money. Making money means that the strategy closed the position in a value that returned a profit, after all deducted fees. Formally, a winning trade will have a return $o_i > 0$. Similarly, a losing trade will have a return $o_j \leq 0$. With that, we can discover the set of all winning trades, $T_{win}$, as follows:

$$ T_{win} = \{ o \in O | o > 0 \} $$

Similarly, we can discover the set of losing trades $T_{lose}$ as follows:

$$ T_{lose} = \{o \in O | o \leq 0\} $$

!!! Example
    In a section where a strategy made four transactions $O = \{3.5, -1, 15, 0\}$:<br>
    $T_{win} = \{3.5, 15\}$<br>
    $T_{lose} = \{-1, 0\}$<br>

### Win Rate and Lose Rate

The win rate $W$ is the proportion of winning trades with respect to all the trades made by a strategy. We use the following function to compute the win rate:

$$W = \frac{|T_{win}|}{N}$$

Where $W$ is the win rate, $N$ is the number of trades and, $T_{win}$ is the set of all trades where the strategy made money.

Similarly, we can compute the rate of losing trades:

$$ 
    L = \frac{|T_{lose}|}{N} 
$$

Where $L$ is the lose rate, $N$ is the amount of trades made and, $T_{lose}$ is the set of all trades where the strategy lost money. Note that the above formula is the same as calculating $L = 1 – W$ or $W = 1 – L$

### Risk Reward Ratio

Risk Reward Ratio ($R$) is a formula used to measure the expected gains of a given investment against the risk of loss. It is basically what you potentially win divided by what you potentially lose. Formally:

$$ R = \frac{\text{potential_profit}}{\text{potential_loss}} $$

???+ Example "Worked example of $R$ calculation"
    Let's say that you think that the price of *stonecoin* today is 10.0\$. You believe that, because they will start mining stonecoin, it will go up to 15.0\$ tomorrow. There is the risk that the stone is too hard, and the GPUs can't mine it, so the price might go to 0\$ tomorrow. You are planning to invest 100\$, which will give you 10 shares (100 / 10).

    Your potential profit is calculated as:

    $\begin{aligned} 
        \text{potential_profit} &= (\text{potential_price} - \text{entry_price}) * \frac{\text{investment}}{\text{entry_price}} \\
                                &= (15 - 10) * (100 / 10) \\
                                &= 50
    \end{aligned}$

    Since the price might go to 0\$, the 100\$ dollars invested could turn into 0.

    We do however use a stoploss of 15% - so in the worst case, we'll sell 15% below entry price (or at 8.5$\).

    $\begin{aligned}
        \text{potential_loss} &= (\text{entry_price} - \text{stoploss}) * \frac{\text{investment}}{\text{entry_price}} \\
                                &= (10 - 8.5) * (100 / 10)\\
                                &= 15
    \end{aligned}$

    We can compute the Risk Reward Ratio as follows:

    $\begin{aligned}
        R   &= \frac{\text{potential_profit}}{\text{potential_loss}}\\
            &= \frac{50}{15}\\
            &= 3.33
    \end{aligned}$<br>
    What it effectively means is that the strategy have the potential to make 3.33\$ for each 1\$ invested.

On a long horizon, that is, on many trades, we can calculate the risk reward by dividing the strategy' average profit on winning trades by the strategy' average loss on losing trades. We can calculate the average profit, $\mu_{win}$, as follows:

$$ \text{average_profit} = \mu_{win} = \frac{\text{sum_of_profits}}{\text{count_winning_trades}} = \frac{\sum^{o \in T_{win}} o}{|T_{win}|} $$

Similarly, we can calculate the average loss, $\mu_{lose}$, as follows:

$$ \text{average_loss} = \mu_{lose} = \frac{\text{sum_of_losses}}{\text{count_losing_trades}} = \frac{\sum^{o \in T_{lose}} o}{|T_{lose}|} $$

Finally, we can calculate the Risk Reward ratio, $R$, as follows:

$$ R = \frac{\text{average_profit}}{\text{average_loss}} = \frac{\mu_{win}}{\mu_{lose}}\\ $$


???+ Example "Worked example of $R$ calculation using mean profit/loss"
    Let's say the strategy that we are using makes an average win $\mu_{win} = 2.06$ and an average loss $\mu_{loss} = 4.11$.<br>
    We calculate the risk reward ratio as follows:<br>
    $R = \frac{\mu_{win}}{\mu_{loss}} = \frac{2.06}{4.11} = 0.5012...$
    

### Expectancy

By combining the Win Rate $W$ and the Risk Reward ratio $R$ to create an expectancy ratio $E$. A expectance ratio is the expected return of the investment made in a trade. We can compute the value of $E$ as follows:

$$E = R * W - L$$

!!! Example "Calculating $E$"
    Let's say that a strategy has a win rate $W = 0.28$ and a risk reward ratio $R = 5$. What this means is that the strategy is expected to make 5 times the investment around on 28% of the trades it makes. Working out the example:<br>
    $E = R * W - L = 5 * 0.28 - 0.72 = 0.68$
    <br>

The expectancy worked out in the example above means that, on average, this strategy' trades will return 1.68 times the size of its losses. Said another way, the strategy makes 1.68\$ for every 1\$ it loses, on average. 

This is important for two reasons: First, it may seem obvious, but you know right away that you have a positive return. Second, you now have a number you can compare to other candidate systems to make decisions about which ones you employ.

It is important to remember that any system with an expectancy greater than 0 is profitable using past data. The key is finding one that will be profitable in the future.

You can also use this value to evaluate the effectiveness of modifications to this system.

!!! Note
    It's important to keep in mind that Edge is testing your expectancy using historical data, there's no guarantee that you will have a similar edge in the future. It's still vital to do this testing in order to build confidence in your methodology but be wary of "curve-fitting" your approach to the historical data as things are unlikely to play out the exact same way for future trades.

## How does it work?

Edge combines dynamic stoploss, dynamic positions, and whitelist generation into one isolated module which is then applied to the trading strategy. If enabled in config, Edge will go through historical data with a range of stoplosses in order to find buy and sell/stoploss signals. It then calculates win rate and expectancy over *N* trades for each stoploss. Here is an example:

| Pair   |      Stoploss      |  Win Rate | Risk Reward Ratio | Expectancy |
|----------|:-------------:|-------------:|------------------:|-----------:|
| XZC/ETH  |  -0.01        |   0.50       |1.176384           | 0.088      |
| XZC/ETH  |  -0.02        |   0.51       |1.115941           | 0.079      |
| XZC/ETH  |  -0.03        |   0.52       |1.359670           | 0.228      |
| XZC/ETH  |  -0.04        |   0.51       |1.234539           | 0.117      |

The goal here is to find the best stoploss for the strategy in order to have the maximum expectancy. In the above example stoploss at $3%$ leads to the maximum expectancy according to historical data.

Edge module then forces stoploss value it evaluated to your strategy dynamically.

### Position size

Edge dictates the amount at stake for each trade to the bot according to the following factors:

- Allowed capital at risk
- Stoploss

Allowed capital at risk is calculated as follows:

```
Allowed capital at risk = (Capital available_percentage) X (Allowed risk per trade)
```

Stoploss is calculated as described above with respect to historical data.

The position size is calculated as follows:

```
Position size = (Allowed capital at risk) / Stoploss
```

Example:

Let's say the stake currency is **ETH** and there is $10$ **ETH** on the wallet. The capital available percentage is $50%$ and the allowed risk per trade is $1\%$. Thus, the available capital for trading is $10 * 0.5 = 5$ **ETH** and the allowed capital at risk would be $5 * 0.01 = 0.05$ **ETH**.

-   **Trade 1:** The strategy detects a new buy signal in the **XLM/ETH** market. `Edge Positioning` calculates a stoploss of $2\%$ and a position of $0.05 / 0.02 = 2.5$ **ETH**. The bot takes a position of $2.5$ **ETH** in the **XLM/ETH** market.

-   **Trade 2:** The strategy detects a buy signal on the **BTC/ETH** market while **Trade 1** is still open. `Edge Positioning` calculates the stoploss of $4\%$ on this market. Thus, **Trade 2** position size is $0.05 / 0.04 = 1.25$ **ETH**.

!!! Tip "Available Capital $\neq$ Available in wallet"
    The available capital for trading didn't change in **Trade 2** even with **Trade 1** still open. The available capital **is not** the free amount in the wallet.

-   **Trade 3:** The strategy detects a buy signal in the **ADA/ETH** market. `Edge Positioning` calculates a stoploss of $1\%$ and a position of $0.05 / 0.01 = 5$ **ETH**. Since **Trade 1** has $2.5$ **ETH** blocked and **Trade 2** has $1.25$ **ETH** blocked, there is only $5 - 1.25 - 2.5 = 1.25$ **ETH** available. Hence, the position size of **Trade 3** is $1.25$ **ETH**. 

!!! Tip "Available Capital Updates"
    The available capital does not change before a position is sold. After a trade is closed the Available Capital goes up if the trade was profitable or goes down if the trade was a loss.

-   The strategy detects a sell signal in the **XLM/ETH** market. The bot exits **Trade 1** for a profit of $1$ **ETH**. The total capital in the wallet becomes $11$ **ETH** and the available capital for trading becomes $5.5$ **ETH**.

-   **Trade 4** The strategy detects a new buy signal int the **XLM/ETH** market. `Edge Positioning` calculates the stoploss of $2\%$, and the position size of $0.055 / 0.02 = 2.75$ **ETH**.

## Edge command reference

--8<-- "commands/edge.md"

## Configurations

Edge module has following configuration options:

|  Parameter | Description |
|------------|-------------|
| `enabled` | If true, then Edge will run periodically. <br>*Defaults to `false`.* <br> **Datatype:** Boolean
| `process_throttle_secs` | How often should Edge run in seconds. <br>*Defaults to `3600` (once per hour).* <br> **Datatype:** Integer
| `calculate_since_number_of_days` | Number of days of data against which Edge calculates Win Rate, Risk Reward and Expectancy. <br> **Note** that it downloads historical data so increasing this number would lead to slowing down the bot. <br>*Defaults to `7`.* <br> **Datatype:** Integer
| `allowed_risk` | Ratio of allowed risk per trade. <br>*Defaults to `0.01` (1%)).* <br> **Datatype:** Float
| `stoploss_range_min` | Minimum stoploss. <br>*Defaults to `-0.01`.* <br> **Datatype:** Float
| `stoploss_range_max` | Maximum stoploss. <br>*Defaults to `-0.10`.* <br> **Datatype:** Float
| `stoploss_range_step` | As an example if this is set to -0.01 then Edge will test the strategy for `[-0.01, -0,02, -0,03 ..., -0.09, -0.10]` ranges. <br> **Note** than having a smaller step means having a bigger range which could lead to slow calculation. <br> If you set this parameter to -0.001, you then slow down the Edge calculation by a factor of 10. <br>*Defaults to `-0.001`.* <br> **Datatype:** Float
| `minimum_winrate` | It filters out pairs which don't have at least minimum_winrate. <br>This comes handy if you want to be conservative and don't comprise win rate in favour of risk reward ratio. <br>*Defaults to `0.60`.* <br> **Datatype:** Float
| `minimum_expectancy` | It filters out pairs which have the expectancy lower than this number. <br>Having an expectancy of 0.20 means if you put 10\$ on a trade you expect a 12\$ return. <br>*Defaults to `0.20`.* <br> **Datatype:** Float
| `min_trade_number` | When calculating *W*, *R* and *E* (expectancy) against historical data, you always want to have a minimum number of trades. The more this number is the more Edge is reliable. <br>Having a win rate of 100% on a single trade doesn't mean anything at all. But having a win rate of 70% over past 100 trades means clearly something. <br>*Defaults to `10` (it is highly recommended not to decrease this number).* <br> **Datatype:** Integer
| `max_trade_duration_minute` | Edge will filter out trades with long duration. If a trade is profitable after 1 month, it is hard to evaluate the strategy based on it. But if most of trades are profitable and they have maximum duration of 30 minutes, then it is clearly a good sign.<br>**NOTICE:** While configuring this value, you should take into consideration your timeframe. As an example filtering out trades having duration less than one day for a strategy which has 4h interval does not make sense. Default value is set assuming your strategy interval is relatively small (1m or 5m, etc.).<br>*Defaults to `1440` (one day).* <br> **Datatype:** Integer
| `remove_pumps` | Edge will remove sudden pumps in a given market while going through historical data. However, given that pumps happen very often in crypto markets, we recommend you keep this off.<br>*Defaults to `false`.* <br> **Datatype:** Boolean

## Running Edge independently

You can run Edge independently in order to see in details the result. Here is an example:

``` bash
freqtrade edge
```

An example of its output:

| **pair**     |   **stoploss** |   **win rate** |   **risk reward ratio** |   **required risk reward** |   **expectancy** |   **total number of trades** |   **average duration (min)** |
|:----------|-----------:|-----------:|--------------------:|-----------------------:|-------------:|-----------------:|---------------:|
| **AGI/BTC**   |      -0.02 |       0.64 |                5.86 |                   0.56 |         3.41 |                       14 |                       54 |
| **NXS/BTC**   |      -0.03 |       0.64 |                2.99 |                   0.57 |         1.54 |                       11 |                       26 |
| **LEND/BTC**  |      -0.02 |       0.82 |                2.05 |                   0.22 |         1.50 |                       11 |                       36 |
| **VIA/BTC**   |      -0.01 |       0.55 |                3.01 |                   0.83 |         1.19 |                       11 |                       48 |
| **MTH/BTC**   |      -0.09 |       0.56 |                2.82 |                   0.80 |         1.12 |                       18 |                       52 |
| **ARDR/BTC**  |      -0.04 |       0.42 |                3.14 |                   1.40 |         0.73 |                       12 |                       42 |
| **BCPT/BTC**  |      -0.01 |       0.71 |                1.34 |                   0.40 |         0.67 |                       14 |                       30 |
| **WINGS/BTC** |      -0.02 |       0.56 |                1.97 |                   0.80 |         0.65 |                       27 |                       42 |
| **VIBE/BTC**  |      -0.02 |       0.83 |                0.91 |                   0.20 |         0.59 |                       12 |                       35 |
| **MCO/BTC**   |      -0.02 |       0.79 |                0.97 |                   0.27 |         0.55 |                       14 |                       31 |
| **GNT/BTC**   |      -0.02 |       0.50 |                2.06 |                   1.00 |         0.53 |                       18 |                       24 |
| **HOT/BTC**   |      -0.01 |       0.17 |                7.72 |                   4.81 |         0.50 |                      209 |                        7 |
| **SNM/BTC**   |      -0.03 |       0.71 |                1.06 |                   0.42 |         0.45 |                       17 |                       38 |
| **APPC/BTC**  |      -0.02 |       0.44 |                2.28 |                   1.27 |         0.44 |                       25 |                       43 |
| **NEBL/BTC**  |      -0.03 |       0.63 |                1.29 |                   0.58 |         0.44 |                       19 |                       59 |

Edge produced the above table by comparing `calculate_since_number_of_days` to `minimum_expectancy` to find `min_trade_number` historical information based on the config file. The timerange Edge uses for its comparisons can be further limited by using the `--timerange` switch.

In live and dry-run modes, after the `process_throttle_secs` has passed, Edge will again process `calculate_since_number_of_days` against `minimum_expectancy` to find `min_trade_number`. If no `min_trade_number` is found, the bot will return "whitelist empty". Depending on the trade strategy being deployed, "whitelist empty" may be return much of the time - or *all* of the time. The use of Edge may also cause trading to occur in bursts, though this is rare.

If you encounter "whitelist empty" a lot, condsider tuning `calculate_since_number_of_days`, `minimum_expectancy`  and `min_trade_number` to align to the trading frequency of your strategy.

### Update cached pairs with the latest data

Edge requires historic data the same way as backtesting does.
Please refer to the [Data Downloading](data-download.md) section of the documentation for details.

### Precising stoploss range

```bash
freqtrade edge --stoplosses=-0.01,-0.1,-0.001 #min,max,step
```

### Advanced use of timerange

```bash
freqtrade edge --timerange=20181110-20181113
```

Doing `--timerange=-20190901` will get all available data until September 1st (excluding September 1st 2019).

The full timerange specification:

* Use tickframes till 2018/01/31: `--timerange=-20180131`
* Use tickframes since 2018/01/31: `--timerange=20180131-`
* Use tickframes since 2018/01/31 till 2018/03/01 : `--timerange=20180131-20180301`
* Use tickframes between POSIX timestamps 1527595200 1527618600: `--timerange=1527595200-1527618600`


[^1]: Question extracted from MIT Opencourseware S096 - Mathematics with applications in Finance: https://ocw.mit.edu/courses/mathematics/18-s096-topics-in-mathematics-with-applications-in-finance-fall-2013/

---

## File: exchanges.md

# Exchange-specific Notes

This page combines common gotchas and Information which are exchange-specific and most likely don't apply to other exchanges.

## Exchange configuration

Freqtrade is based on [CCXT library](https://github.com/ccxt/ccxt) that supports over 100 cryptocurrency
exchange markets and trading APIs. The complete up-to-date list can be found in the
[CCXT repo homepage](https://github.com/ccxt/ccxt/tree/master/python).
However, the bot was tested by the development team with only a few exchanges.
A current list of these can be found in the "Home" section of this documentation.

Feel free to test other exchanges and submit your feedback or PR to improve the bot or confirm exchanges that work flawlessly..

Some exchanges require special configuration, which can be found below.

### Sample exchange configuration

A exchange configuration for "binance" would look as follows:

```json
"exchange": {
    "name": "binance",
    "key": "your_exchange_key",
    "secret": "your_exchange_secret",
    "ccxt_config": {},
    "ccxt_async_config": {},
    // ... 
```

### Setting rate limits

Usually, rate limits set by CCXT are reliable and work well.
In case of problems related to rate-limits (usually DDOS Exceptions in your logs), it's easy to change rateLimit settings to other values.

```json
"exchange": {
    "name": "kraken",
    "key": "your_exchange_key",
    "secret": "your_exchange_secret",
    "ccxt_config": {"enableRateLimit": true},
    "ccxt_async_config": {
        "enableRateLimit": true,
        "rateLimit": 3100
    },
```

This configuration enables kraken, as well as rate-limiting to avoid bans from the exchange.
`"rateLimit": 3100` defines a wait-event of 3.1s between each call. This can also be completely disabled by setting `"enableRateLimit"` to false.

!!! Note
    Optimal settings for rate-limiting depend on the exchange and the size of the whitelist, so an ideal parameter will vary on many other settings.
    We try to provide sensible defaults per exchange where possible, if you encounter bans please make sure that `"enableRateLimit"` is enabled and increase the `"rateLimit"` parameter step by step.

## Binance

!!! Warning "Server location and geo-ip restrictions"
    Please be aware that Binance restricts API access regarding the server country. The current and non-exhaustive countries blocked are Canada, Malaysia, Netherlands and United States. Please go to [binance terms > b. Eligibility](https://www.binance.com/en/terms) to find up to date list.

Binance supports [time_in_force](configuration.md#understand-order_time_in_force).

!!! Tip "Stoploss on Exchange"
    Binance supports `stoploss_on_exchange` and uses `stop-loss-limit` orders. It provides great advantages, so we recommend to benefit from it by enabling stoploss on exchange.
    On futures, Binance supports both `stop-limit` as well as `stop-market` orders. You can use either `"limit"` or `"market"` in the `order_types.stoploss` configuration setting to decide which type to use.

### Binance Blacklist recommendation

For Binance, it is suggested to add `"BNB/<STAKE>"` to your blacklist to avoid issues, unless you are willing to maintain enough extra `BNB` on the account or unless you're willing to disable using `BNB` for fees.
Binance accounts may use `BNB` for fees, and if a trade happens to be on `BNB`, further trades may consume this position and make the initial BNB trade unsellable as the expected amount is not there anymore.

If not enough `BNB` is available to cover transaction fees, then fees will not be covered by `BNB` and no fee reduction will occur. Freqtrade will never buy BNB to cover for fees. BNB needs to be bought and monitored manually to this end.

### Binance sites

Binance has been split into 2, and users must use the correct ccxt exchange ID for their exchange, otherwise API keys are not recognized.

* [binance.com](https://www.binance.com/) - International users. Use exchange id: `binance`.
* [binance.us](https://www.binance.us/) - US based users. Use exchange id: `binanceus`.

### Binance RSA keys

Freqtrade supports binance RSA API keys.

We recommend to use them as environment variable.

``` bash
export FREQTRADE__EXCHANGE__SECRET="$(cat ./rsa_binance.private)"
```

They can however also be configured via configuration file. Since json doesn't support multi-line strings, you'll have to replace all newlines with `\n` to have a valid json file.

``` json
// ...
 "key": "<someapikey>",
 "secret": "-----BEGIN PRIVATE KEY-----\nMIIEvQIBABACAFQA<...>s8KX8=\n-----END PRIVATE KEY-----"
// ...
```

### Binance Futures

Binance has specific (unfortunately complex) [Futures Trading Quantitative Rules](https://www.binance.com/en/support/faq/4f462ebe6ff445d4a170be7d9e897272) which need to be followed, and which prohibit a too low stake-amount (among others) for too many orders.
Violating these rules will result in a trading restriction.

When trading on Binance Futures market, orderbook must be used because there is no price ticker data for futures.

``` jsonc
  "entry_pricing": {
      "use_order_book": true,
      "order_book_top": 1,
      "check_depth_of_market": {
          "enabled": false,
          "bids_to_ask_delta": 1
      }
  },
  "exit_pricing": {
      "use_order_book": true,
      "order_book_top": 1
  },
```

#### Binance isolated futures settings

Users will also have to have the futures-setting "Position Mode" set to "One-way Mode", and "Asset Mode" set to "Single-Asset Mode".
These settings will be checked on startup, and freqtrade will show an error if this setting is wrong.

![Binance futures settings](assets/binance_futures_settings.png)

Freqtrade will not attempt to change these settings.

#### Binance BNFCR futures

BNFCR mode are a special type of futures mode on Binance to work around regulatory issues in Europe.  
To use BNFCR futures, you will have to have the following combination of settings:

``` jsonc
{
    // ...
    "trading_mode": "futures",
    "margin_mode": "cross",
    "proxy_coin": "BNFCR",
    "stake_currency": "USDT" // or "USDC"
    // ...
}
```

The `stake_currency` setting defines the markets the bot will be operating in. This choice is really arbitrary.

On the exchange, you'll have to use "Multi-asset Mode" - and "Position Mode set to "One-way Mode".  
Freqtrade will check these settings on startup, but won't attempt to change them.

## Bingx

BingX supports [time_in_force](configuration.md#understand-order_time_in_force) with settings "GTC" (good till cancelled), "IOC" (immediate-or-cancel) and "PO" (Post only) settings.

!!! Tip "Stoploss on Exchange"
    Bingx supports `stoploss_on_exchange` and can use both stop-limit and stop-market orders. It provides great advantages, so we recommend to benefit from it by enabling stoploss on exchange.

## Kraken

Kraken supports [time_in_force](configuration.md#understand-order_time_in_force) with settings "GTC" (good till cancelled), "IOC" (immediate-or-cancel) and "PO" (Post only) settings.

!!! Tip "Stoploss on Exchange"
    Kraken supports `stoploss_on_exchange` and can use both stop-loss-market and stop-loss-limit orders. It provides great advantages, so we recommend to benefit from it.
    You can use either `"limit"` or `"market"` in the `order_types.stoploss` configuration setting to decide which type to use.

### Historic Kraken data

The Kraken API does only provide 720 historic candles, which is sufficient for Freqtrade dry-run and live trade modes, but is a problem for backtesting.
To download data for the Kraken exchange, using `--dl-trades` is mandatory, otherwise the bot will download the same 720 candles over and over, and you'll not have enough backtest data.

To speed up downloading, you can download the [trades zip files](https://support.kraken.com/hc/en-us/articles/360047543791-Downloadable-historical-market-data-time-and-sales-) kraken provides.
These are usually updated once per quarter. Freqtrade expects these files to be placed in `user_data/data/kraken/trades_csv`.

A structure as follows can make sense if using incremental files, with the "full" history in one directory, and incremental files in different directories.
The assumption for this mode is that the data is downloaded and unzipped keeping filenames as they are.
Duplicate content will be ignored (based on timestamp) - though the assumption is that there is no gap in the data.

This means, if your "full" history ends in Q4 2022 - then both incremental updates Q1 2023 and Q2 2023 are available.
Not having this will lead to incomplete data, and therefore invalid results while using the data.

```
└── trades_csv
    ├── Kraken_full_history
    │   ├── BCHEUR.csv
    │   └── XBTEUR.csv
    ├── Kraken_Trading_History_Q1_2023
    │   ├── BCHEUR.csv
    │   └── XBTEUR.csv
    └── Kraken_Trading_History_Q2_2023
        ├── BCHEUR.csv
        └── XBTEUR.csv
```

You can convert these files into freqtrade files:

``` bash
freqtrade convert-trade-data --exchange kraken --format-from kraken_csv --format-to feather
# Convert trade data to different ohlcv timeframes
freqtrade trades-to-ohlcv -p BTC/EUR BCH/EUR --exchange kraken -t 1m 5m 15m 1h
```

The converted data also makes downloading data possible, and will start the download after the latest loaded trade.

``` bash
freqtrade download-data --exchange kraken --dl-trades -p BTC/EUR BCH/EUR 
```

!!! Warning "Downloading data from kraken"
    Downloading kraken data will require significantly more memory (RAM) than any other exchange, as the trades-data needs to be converted into candles on your machine.
    It will also take a long time, as freqtrade will need to download every single trade that happened on the exchange for the pair / timerange combination, therefore please be patient.

!!! Warning "rateLimit tuning"
    Please pay attention that rateLimit configuration entry holds delay in milliseconds between requests, NOT requests/sec rate.
    So, in order to mitigate Kraken API "Rate limit exceeded" exception, this configuration should be increased, NOT decreased.

## Kucoin

Kucoin requires a passphrase for each api key, you will therefore need to add this key into the configuration so your exchange section looks as follows:

```json
"exchange": {
    "name": "kucoin",
    "key": "your_exchange_key",
    "secret": "your_exchange_secret",
    "password": "your_exchange_api_key_password",
    // ...
}
```

Kucoin supports [time_in_force](configuration.md#understand-order_time_in_force).

!!! Tip "Stoploss on Exchange"
    Kucoin supports `stoploss_on_exchange` and can use both stop-loss-market and stop-loss-limit orders. It provides great advantages, so we recommend to benefit from it.
    You can use either `"limit"` or `"market"` in the `order_types.stoploss` configuration setting to decide which type of stoploss shall be used.

### Kucoin Blacklists

For Kucoin, it is suggested to add `"KCS/<STAKE>"` to your blacklist to avoid issues, unless you are willing to maintain enough extra `KCS` on the account or unless you're willing to disable using `KCS` for fees. 
Kucoin accounts may use `KCS` for fees, and if a trade happens to be on `KCS`, further trades may consume this position and make the initial `KCS` trade unsellable as the expected amount is not there anymore.

## HTX

!!! Tip "Stoploss on Exchange"
    HTX supports `stoploss_on_exchange` and uses `stop-limit` orders. It provides great advantages, so we recommend to benefit from it by enabling stoploss on exchange.

## OKX

OKX requires a passphrase for each api key, you will therefore need to add this key into the configuration so your exchange section looks as follows:

```json
"exchange": {
    "name": "okx",
    "key": "your_exchange_key",
    "secret": "your_exchange_secret",
    "password": "your_exchange_api_key_password",
    // ...
}
```

If you've registered with OKX on the host my.okx.com (OKX EAA)- you will need to use `"myokx"` as the exchange name.
Using the wrong exchange will result in the error "OKX Error 50119: API key doesn't exist" - as the 2 are separate entities.

!!! Warning
    OKX only provides 100 candles per api call. Therefore, the strategy will only have a pretty low amount of data available in backtesting mode.

!!! Warning "Futures"
    OKX Futures has the concept of "position mode" - which can be "Buy/Sell" or long/short (hedge mode).
    Freqtrade supports both modes (we recommend to use Buy/Sell mode) - but changing the mode mid-trading is not supported and will lead to exceptions and failures to place trades.
    OKX also only provides MARK candles for the past ~3 months. Backtesting futures prior to that date will therefore lead to slight deviations, as funding-fees cannot be calculated correctly without this data.

## Gate.io

!!! Tip "Stoploss on Exchange"
    Gate.io supports `stoploss_on_exchange` and uses `stop-loss-limit` orders. It provides great advantages, so we recommend to benefit from it by enabling stoploss on exchange..

Gate.io allows the use of `POINT` to pay for fees. As this is not a tradable currency (no regular market available), automatic fee calculations will fail (and default to a fee of 0).
The configuration parameter `exchange.unknown_fee_rate` can be used to specify the exchange rate between Point and the stake currency. Obviously, changing the stake-currency will also require changes to this value.

Gate API keys require the following permissions on top of the market type you want to trade:

* "Spot Trade" _or_ "Perpetual Futures" (Read and Write) (either select both, or the one matching the market you want to trade)
* "Wallet" (read only)
* "Account" (read only)

Without these permissions, the bot will not start correctly and show errors like "permission missing".

## Bybit

Futures trading on bybit is currently supported for USDT markets, and will use isolated futures mode.

On startup, freqtrade will set the position mode to "One-way Mode" for the whole (sub)account. This avoids making this call over and over again (slowing down bot operations), but means that changes to this setting may result in exceptions and errors.

As bybit doesn't provide funding rate history, the dry-run calculation is used for live trades as well.

API Keys for live futures trading must have the following permissions:

* Read-write
* Contract - Orders
* Contract - Positions

We do strongly recommend to limit all API keys to the IP you're going to use it from.

!!! Warning "Unified accounts"
    Freqtrade assumes accounts to be dedicated to the bot.
    We therefore recommend the usage of one subaccount per bot. This is especially important when using unified accounts.  
    Other configurations (multiple bots on one account, manual non-bot trades on the bot account) are not supported and may lead to unexpected behavior.


!!! Tip "Stoploss on Exchange"
    Bybit (futures only) supports `stoploss_on_exchange` and uses `stop-loss-limit` orders. It provides great advantages, so we recommend to benefit from it by enabling stoploss on exchange.
    On futures, Bybit supports both `stop-limit` as well as `stop-market` orders. You can use either `"limit"` or `"market"` in the `order_types.stoploss` configuration setting to decide which type to use.

## Bitmart

Bitmart requires the API key Memo (the name you give the API key) to go along with the exchange key and secret.
It's therefore required to pass the UID as well.

```json
"exchange": {
    "name": "bitmart",
    "uid": "your_bitmart_api_key_memo",
    "secret": "your_exchange_secret",
    "password": "your_exchange_api_key_password",
    // ...
}
```

!!! Warning "Necessary Verification"
    Bitmart requires Verification Lvl2 to successfully trade on the spot market through the API - even though trading via UI works just fine with just Lvl1 verification.

## Hyperliquid

!!! Tip "Stoploss on Exchange"
    Hyperliquid supports `stoploss_on_exchange` and uses `stop-loss-limit` orders. It provides great advantages, so we recommend to benefit from it.

Hyperliquid is a Decentralized Exchange (DEX). Decentralized exchanges work a bit different compared to normal exchanges. Instead of authenticating private API calls using an API key, private API calls need to be signed with the private key of your wallet (We recommend using an api Wallet for this, generated either on Hyperliquid or in your wallet of choice).
This needs to be configured like this:

```json
"exchange": {
    "name": "hyperliquid",
    "walletAddress": "your_eth_wallet_address",
    "privateKey": "your_api_private_key",
    // ...
}
```

* walletAddress in hex format: `0x<40 hex characters>` - Can be easily copied from your wallet - and should be your wallet address, not your API Wallet Address.
* privateKey in hex format: `0x<64 hex characters>` - Use the key the API Wallet shows on creation.

Hyperliquid handles deposits and withdrawals on the Arbitrum One chain, a Layer 2 scaling solution built on top of Ethereum. Hyperliquid uses USDC as quote / collateral. The process of depositing USDC on Hyperliquid requires a couple of steps, see [how to start trading](https://hyperliquid.gitbook.io/hyperliquid-docs/onboarding/how-to-start-trading) for details on what steps are needed.

!!! Note "Hyperliquid general usage Notes"
    Hyperliquid does not support market orders, however ccxt will simulate market orders by placing limit orders with a maximum slippage of 5%.  
    Unfortunately, hyperliquid only offers 5000 historic candles, so backtesting will either need to build candles historically (by waiting and downloading the data incrementally over time) - or will be limited to the last 5000 candles.

!!! Info "Some general best practices (non exhaustive)"
    * Beware of supply chain attacks, like pip package poisoning etcetera. Whenever you use your private key, make sure your environment is safe.
    * Don't use your actual wallet private key for trading. Use the Hyperliquid [API generator](https://app.hyperliquid.xyz/API) to create a separate API wallet.
    * Don't store your actual wallet private key on the server you use for freqtrade. Use the API wallet private key instead. This key won't allow withdrawals, only trading.
    * Always keep your mnemonic phrase and private key private.
    * Don't use the same mnemonic as the one you had to backup when initializing a hardware wallet, using the same mnemonic basically deletes the security of your hardware wallet.
    * Create a different software wallet, only transfer the funds you want to trade with to that wallet, and use that wallet to trade on Hyperliquid.
    * If you have funds you don't want to use for trading (after making a profit for example), transfer them back to your hardware wallet.

## All exchanges

Should you experience constant errors with Nonce (like `InvalidNonce`), it is best to regenerate the API keys. Resetting Nonce is difficult and it's usually easier to regenerate the API keys.

## Random notes for other exchanges

* The Ocean (exchange id: `theocean`) exchange uses Web3 functionality and requires `web3` python package to be installed:

```shell
pip3 install web3
```

### Getting latest price / Incomplete candles

Most exchanges return current incomplete candle via their OHLCV/klines API interface.
By default, Freqtrade assumes that incomplete candle is fetched from the exchange and removes the last candle assuming it's the incomplete candle.

Whether your exchange returns incomplete candles or not can be checked using [the helper script](developer.md#incomplete-candles) from the Contributor documentation.

Due to the danger of repainting, Freqtrade does not allow you to use this incomplete candle.

However, if it is based on the need for the latest price for your strategy - then this requirement can be acquired using the [data provider](strategy-customization.md#possible-options-for-dataprovider) from within the strategy.

### Advanced Freqtrade Exchange configuration

Advanced options can be configured using the `_ft_has_params` setting, which will override Defaults and exchange-specific behavior.

Available options are listed in the exchange-class as `_ft_has_default`.

For example, to test the order type `FOK` with Kraken, and modify candle limit to 200 (so you only get 200 candles per API call):

```json
"exchange": {
    "name": "kraken",
    "_ft_has_params": {
        "order_time_in_force": ["GTC", "FOK"],
        "ohlcv_candle_limit": 200
        }
    //...
}
```

!!! Warning
    Please make sure to fully understand the impacts of these settings before modifying them.

---

## File: faq.md

# Freqtrade FAQ

## Supported Markets

Freqtrade supports spot trading, as well as (isolated) futures trading for some selected exchanges. Please refer to the [documentation start page](index.md#supported-futures-exchanges-experimental) for an up-to-date list of supported exchanges.

### Can my bot open short positions?

Freqtrade can open short positions in futures markets.
This requires the strategy to be made for this - and `"trading_mode": "futures"` in the configuration.
Please make sure to read the [relevant documentation page](leverage.md) first.

In spot markets, you can in some cases use leveraged spot tokens, which reflect an inverted pair (eg. BTCUP/USD, BTCDOWN/USD, ETHBULL/USD, ETHBEAR/USD,...) which can be traded with Freqtrade.

### Can my bot trade options or futures?

Futures trading is supported for selected exchanges. Please refer to the [documentation start page](index.md#supported-futures-exchanges-experimental) for an up-to-date list of supported exchanges.

## Beginner Tips & Tricks

* When you work with your strategy & hyperopt file you should use a proper code editor like VSCode or PyCharm. A good code editor will provide syntax highlighting as well as line numbers, making it easy to find syntax errors (most likely pointed out by Freqtrade during startup).

## Freqtrade common questions

### Can freqtrade open multiple positions on the same pair in parallel?

No. Freqtrade will only open one position per pair at a time.
You can however use the [`adjust_trade_position()` callback](strategy-callbacks.md#adjust-trade-position) to adjust an open position.

Backtesting provides an option for this in `--eps` - however this is only there to highlight "hidden" signals, and will not work in live.

### The bot does not start

Running the bot with `freqtrade trade --config config.json` shows the output `freqtrade: command not found`.

This could be caused by the following reasons:

* The virtual environment is not active.
  * Run `source .venv/bin/activate` to activate the virtual environment.
* The installation did not complete successfully.
  * Please check the [Installation documentation](installation.md).

### The bot starts, but in STOPPED mode

Make sure you set the `initial_state` config option to `"running"` in your config.json

### I have waited 5 minutes, why hasn't the bot made any trades yet?

* Depending on the entry strategy, the amount of whitelisted coins, the
situation of the market etc, it can take up to hours or days to find a good entry
position for a trade. Be patient!

* Backtesting will tell you roughly how many trades to expect - but that won't guarantee that they'll be distributed evenly across time - so you could have 20 trades on one day, and 0 for the rest of the week.

* It may be because of a configuration error. It's best to check the logs, they usually tell you if the bot is simply not getting buy signals (only heartbeat messages), or if there is something wrong (errors / exceptions in the log).

### I have made 12 trades already, why is my total profit negative?

I understand your disappointment but unfortunately 12 trades is just
not enough to say anything. If you run backtesting, you can see that the
current algorithm does leave you on the plus side, but that is after
thousands of trades and even there, you will be left with losses on
specific coins that you have traded tens if not hundreds of times. We
of course constantly aim to improve the bot but it will _always_ be a
gamble, which should leave you with modest wins on monthly basis but
you can't say much from few trades.

### I’d like to make changes to the config. Can I do that without having to kill the bot?

Yes. You can edit your config and use the `/reload_config` command to reload the configuration. The bot will stop, reload the configuration and strategy and will restart with the new configuration and strategy.

### Why does my bot not sell everything it bought?

This is called "coin dust" and can happen on all exchanges.
It happens because many exchanges subtract fees from the "receiving currency" - so you buy 100 COIN - but you only get 99.9 COIN.
As COIN is trading in full lot sizes (1COIN steps), you cannot sell 0.9 COIN (or 99.9 COIN) - but you need to round down to 99 COIN.

This is not a bot-problem, but will also happen while manual trading.

While freqtrade can handle this (it'll sell 99 COIN), fees are often below the minimum tradable lot-size (you can only trade full COIN, not 0.9 COIN).
Leaving the dust (0.9 COIN) on the exchange makes usually sense, as the next time freqtrade buys COIN, it'll eat into the remaining small balance, this time selling everything it bought, and therefore slowly declining the dust balance (although it most likely will never reach exactly 0).

Where possible (e.g. on binance), the use of the exchange's dedicated fee currency will fix this.
On binance, it's sufficient to have BNB in your account, and have "Pay fees in BNB" enabled in your profile. Your BNB balance will slowly decline (as it's used to pay fees) - but you'll no longer encounter dust (Freqtrade will include the fees in the profit calculations).
Other exchanges don't offer such possibilities, where it's simply something you'll have to accept or move to a different exchange.

### I deposited more funds to the exchange, but my bot doesn't recognize this

Freqtrade will update the exchange balance when necessary (Before placing an order).
RPC calls (Telegram's `/balance`, API calls to `/balance`) can trigger an update at max. once per hour.

If `adjust_trade_position` is enabled (and the bot has open trades eligible for position adjustments) - then the wallets will be refreshed once per hour.
To force an immediate update, you can use `/reload_config` - which will restart the bot.

### I want to use incomplete candles

Freqtrade will not provide incomplete candles to strategies. Using incomplete candles will lead to repainting and consequently to strategies with "ghost" buys, which are impossible to both backtest, and verify after they happened.

You can use "current" market data by using the [dataprovider](strategy-customization.md#orderbookpair-maximum)'s orderbook or ticker methods - which however cannot be used during backtesting.

### Is there a setting to only Exit the trades being held and not perform any new Entries?

You can use the `/stopentry` command in Telegram to prevent future trade entry, followed by `/forceexit all` (sell all open trades).

### I want to run multiple bots on the same machine

Please look at the [advanced setup documentation Page](advanced-setup.md#running-multiple-instances-of-freqtrade).

### I'm getting "Impossible to load Strategy" when starting the bot

This error message is shown when the bot cannot load the strategy.
Usually, you can use `freqtrade list-strategies` to list all available strategies. 
The output of this command will also include a status column, showing if the strategy can be loaded.

Please check the following:

* Are you using the correct strategy name? The strategy name is case-sensitive and must correspond to the Strategy class name (not the filename!).
* Is the strategy in the `user_data/strategies` directory, and has the file-ending `.py`?
* Does the bot show other warnings before this error? Maybe you're missing some dependencies for the strategy - which would be highlighted in the log.
* In case of docker - is the strategy directory mounted correctly (check the volumes part of the docker-compose file)?

### I'm getting "Missing data fillup" messages in the log

This message is just a warning that the latest candles had missing candles in them.
Depending on the exchange, this can indicate that the pair didn't have a trade for the timeframe you are using - and the exchange does only return candles with volume.
On low volume pairs, this is a rather common occurrence.

If this happens for all pairs in the pairlist, this might indicate a recent exchange downtime. Please check your exchange's public channels for details.

Irrespectively of the reason, Freqtrade will fill up these candles with "empty" candles, where open, high, low and close are set to the previous candle close - and volume is empty. In a chart, this will look like a `_` - and is aligned with how exchanges usually represent 0 volume candles.

### I'm getting "Price jump between 2 candles detected"

This message is a warning that the candles had a price jump of > 30%.
This might be a sign that the pair stopped trading, and some token exchange took place (e.g. COCOS in 2021 - where price jumped from 0.0000154 to 0.01621).
This message is often accompanied by ["Missing data fillup"](#im-getting-missing-data-fillup-messages-in-the-log) - as trading on such pairs is often stopped for some time.

### I want to reset the bot's database

To reset the bot's database, you can either delete the database (by default `tradesv3.sqlite` or `tradesv3.dryrun.sqlite`), or use a different database url via `--db-url` (e.g. `sqlite:///mynewdatabase.sqlite`).

### I'm getting "Outdated history for pair xxx" in the log

The bot is trying to tell you that it got an outdated last candle (not the last complete candle).
As a consequence, Freqtrade will not enter a trade for this pair - as trading on old information is usually not what is desired.

This warning can point to one of the below problems:

* Exchange downtime -> Check your exchange status page / blog / twitter feed for details.
* Wrong system time -> Ensure your system-time is correct.
* Barely traded pair -> Check the pair on the exchange webpage, look at the timeframe your strategy uses. If the pair does not have any volume in some candles (usually visualized with a "volume 0" bar, and a "_" as candle), this pair did not have any trades in this timeframe. These pairs should ideally be avoided, as they can cause problems with order-filling.
* API problem -> API returns wrong data (this only here for completeness, and should not happen with supported exchanges).

### I'm getting the "Exchange XXX does not support market orders." message and cannot run my strategy

As the message says, your exchange does not support market orders and you have one of the [order types](configuration.md/#understand-order_types) set to "market". Your strategy was probably written with other exchanges in mind and sets "market" orders for "stoploss" orders, which is correct and preferable for most of the exchanges supporting market orders (but not for Gate.io).

To fix this, redefine order types in the strategy to use "limit" instead of "market":

``` python
    order_types = {
        ...
        "stoploss": "limit",
        ...
    }
```

The same fix should be applied in the configuration file, if order types are defined in your custom config rather than in the strategy.

### I'm trying to start the bot live, but get an API permission error

Errors like `Invalid API-key, IP, or permissions for action` mean exactly what they actually say.  
Your API key is either invalid (copy/paste error? check for leading/trailing spaces in the config), expired, or the IP you're running the bot from is not enabled in the Exchange's API console.  
Usually, the permission "Spot Trading" (or the equivalent in the exchange you use) will be necessary.  
Futures will usually have to be enabled specifically.

### How do I search the bot logs for something?

By default, the bot writes its log into stderr stream. This is implemented this way so that you can easily separate the bot's diagnostics messages from Backtesting, Edge and Hyperopt results, output from other various Freqtrade utility sub-commands, as well as from the output of your custom `print()`'s you may have inserted into your strategy. So if you need to search the log messages with the grep utility, you need to redirect stderr to stdout and disregard stdout.

* In unix shells, this normally can be done as simple as:
```shell
$ freqtrade --some-options 2>&1 >/dev/null | grep 'something'
```
(note, `2>&1` and `>/dev/null` should be written in this order)

* Bash interpreter also supports so called process substitution syntax, you can grep the log for a string with it as:
```shell
$ freqtrade --some-options 2> >(grep 'something') >/dev/null
```
or
```shell
$ freqtrade --some-options 2> >(grep -v 'something' 1>&2)
```

* You can also write the copy of Freqtrade log messages to a file with the `--logfile` option:
```shell
$ freqtrade --logfile /path/to/mylogfile.log --some-options
```
and then grep it as:
```shell
$ cat /path/to/mylogfile.log | grep 'something'
```
or even on the fly, as the bot works and the log file grows:
```shell
$ tail -f /path/to/mylogfile.log | grep 'something'
```
from a separate terminal window.

On Windows, the `--logfile` option is also supported by Freqtrade and you can use the `findstr` command to search the log for the string of interest:
```
> type \path\to\mylogfile.log | findstr "something"
```

## Hyperopt module

### Why does freqtrade not have GPU support?

First of all, most indicator libraries don't have GPU support - as such, there would be little benefit for indicator calculations.
The GPU improvements would only apply to pandas-native calculations - or ones written by yourself.

For hyperopt, freqtrade is using scikit-optimize, which is built on top of scikit-learn.
Their statement about GPU support is [pretty clear](https://scikit-learn.org/stable/faq.html#will-you-add-gpu-support).

GPU's also are only good at crunching numbers (floating point operations).
For hyperopt, we need both number-crunching (find next parameters) and running python code (running backtesting).
As such, GPU's are not too well suited for most parts of hyperopt.

The benefit of using GPU would therefore be pretty slim - and will not justify the complexity introduced by trying to add GPU support.

There is however nothing preventing you from using GPU-enabled indicators within your strategy if you think you must have this - you will however probably be disappointed by the slim gain that will give you (compared to the complexity).

### How many epochs do I need to get a good Hyperopt result?

Per default Hyperopt called without the `-e`/`--epochs` command line option will only
run 100 epochs, means 100 evaluations of your triggers, guards, ... Too few
to find a great result (unless if you are very lucky), so you probably
have to run it for 10000 or more. But it will take an eternity to
compute.

Since hyperopt uses Bayesian search, running for too many epochs may not produce greater results.

It's therefore recommended to run between 500-1000 epochs over and over until you hit at least 10000 epochs in total (or are satisfied with the result). You can best judge by looking at the results - if the bot keeps discovering better strategies, it's best to keep on going.

```bash
freqtrade hyperopt --hyperopt-loss SharpeHyperOptLossDaily --strategy SampleStrategy -e 1000
```

### Why does it take a long time to run hyperopt?

* Discovering a great strategy with Hyperopt takes time. Study www.freqtrade.io, the Freqtrade Documentation page, join the Freqtrade [discord community](https://discord.gg/p7nuUNVfP7). While you patiently wait for the most advanced, free crypto bot in the world, to hand you a possible golden strategy specially designed just for you.

* If you wonder why it can take from 20 minutes to days to do 1000 epochs here are some answers:

This answer was written during the release 0.15.1, when we had:

* 8 triggers
* 9 guards: let's say we evaluate even 10 values from each
* 1 stoploss calculation: let's say we want 10 values from that too to be evaluated

The following calculation is still very rough and not very precise
but it will give the idea. With only these triggers and guards there is
already 8\*10^9\*10 evaluations. A roughly total of 80 billion evaluations.
Did you run 100 000 evaluations? Congrats, you've done roughly 1 / 100 000 th
of the search space, assuming that the bot never tests the same parameters more than once.

* The time it takes to run 1000 hyperopt epochs depends on things like: The available cpu, hard-disk, ram, timeframe, timerange, indicator settings, indicator count, amount of coins that hyperopt test strategies on and the resulting trade count - which can be 650 trades in a year or 100000 trades depending if the strategy aims for big profits by trading rarely or for many low profit trades.

Example: 4% profit 650 times vs 0,3% profit a trade 10000 times in a year. If we assume you set the --timerange to 365 days.

Example:
`freqtrade --config config.json --strategy SampleStrategy --hyperopt SampleHyperopt -e 1000 --timerange 20190601-20200601`

## Edge module

### Edge implements interesting approach for controlling position size, is there any theory behind it?

The Edge module is mostly a result of brainstorming of [@mishaker](https://github.com/mishaker) and [@creslinux](https://github.com/creslinux) freqtrade team members.

You can find further info on expectancy, win rate, risk management and position size in the following sources:

- https://www.tradeciety.com/ultimate-math-guide-for-traders/
- https://samuraitradingacademy.com/trading-expectancy/
- https://www.learningmarkets.com/determining-expectancy-in-your-trading/
- https://www.lonestocktrader.com/make-money-trading-positive-expectancy/
- https://www.babypips.com/trading/trade-expectancy-matter

## Official channels

Freqtrade is using exclusively the following official channels:

* [Freqtrade discord server](https://discord.gg/p7nuUNVfP7)
* [Freqtrade documentation (https://freqtrade.io)](https://freqtrade.io)
* [Freqtrade github organization](https://github.com/freqtrade)

Nobody affiliated with the freqtrade project will ask you about your exchange keys or anything else exposing your funds to exploitation.
Should you be asked to expose your exchange keys or send funds to some random wallet, then please don't follow these instructions.

Failing to follow these guidelines will not be responsibility of freqtrade.

## "Freqtrade token"

Freqtrade does not have a Crypto token offering.

Token offerings you find on the internet referring Freqtrade, FreqAI or freqUI must be considered to be a scam, trying to exploit freqtrade's popularity for their own, nefarious gains.

---

## File: freqai-configuration.md

# Configuration

FreqAI is configured through the typical [Freqtrade config file](configuration.md) and the standard [Freqtrade strategy](strategy-customization.md). Examples of FreqAI config and strategy files can be found in `config_examples/config_freqai.example.json` and `freqtrade/templates/FreqaiExampleStrategy.py`, respectively.

## Setting up the configuration file

 Although there are plenty of additional parameters to choose from, as highlighted in the [parameter table](freqai-parameter-table.md#parameter-table), a FreqAI config must at minimum include the following parameters (the parameter values are only examples):

```json
    "freqai": {
        "enabled": true,
        "purge_old_models": 2,
        "train_period_days": 30,
        "backtest_period_days": 7,
        "identifier" : "unique-id",
        "feature_parameters" : {
            "include_timeframes": ["5m","15m","4h"],
            "include_corr_pairlist": [
                "ETH/USD",
                "LINK/USD",
                "BNB/USD"
            ],
            "label_period_candles": 24,
            "include_shifted_candles": 2,
            "indicator_periods_candles": [10, 20]
        },
        "data_split_parameters" : {
            "test_size": 0.25
        }
    }
```

A full example config is available in `config_examples/config_freqai.example.json`.

!!! Note
    The `identifier` is commonly overlooked by newcomers, however, this value plays an important role in your configuration. This value is a unique ID that you choose to describe one of your runs. Keeping it the same allows you to maintain crash resilience as well as faster backtesting. As soon as you want to try a new run (new features, new model, etc.), you should change this value (or delete the `user_data/models/unique-id` folder. More details available in the [parameter table](freqai-parameter-table.md#feature-parameters).

## Building a FreqAI strategy

The FreqAI strategy requires including the following lines of code in the standard [Freqtrade strategy](strategy-customization.md):

```python
    # user should define the maximum startup candle count (the largest number of candles
    # passed to any single indicator)
    startup_candle_count: int = 20

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:

        # the model will return all labels created by user in `set_freqai_targets()`
        # (& appended targets), an indication of whether or not the prediction should be accepted,
        # the target mean/std values for each of the labels created by user in
        # `set_freqai_targets()` for each training period.

        dataframe = self.freqai.start(dataframe, metadata, self)

        return dataframe

    def feature_engineering_expand_all(self, dataframe: DataFrame, period, **kwargs) -> DataFrame:
        """
        *Only functional with FreqAI enabled strategies*
        This function will automatically expand the defined features on the config defined
        `indicator_periods_candles`, `include_timeframes`, `include_shifted_candles`, and
        `include_corr_pairs`. In other words, a single feature defined in this function
        will automatically expand to a total of
        `indicator_periods_candles` * `include_timeframes` * `include_shifted_candles` *
        `include_corr_pairs` numbers of features added to the model.

        All features must be prepended with `%` to be recognized by FreqAI internals.

        :param df: strategy dataframe which will receive the features
        :param period: period of the indicator - usage example:
        dataframe["%-ema-period"] = ta.EMA(dataframe, timeperiod=period)
        """

        dataframe["%-rsi-period"] = ta.RSI(dataframe, timeperiod=period)
        dataframe["%-mfi-period"] = ta.MFI(dataframe, timeperiod=period)
        dataframe["%-adx-period"] = ta.ADX(dataframe, timeperiod=period)
        dataframe["%-sma-period"] = ta.SMA(dataframe, timeperiod=period)
        dataframe["%-ema-period"] = ta.EMA(dataframe, timeperiod=period)

        return dataframe

    def feature_engineering_expand_basic(self, dataframe: DataFrame, **kwargs) -> DataFrame:
        """
        *Only functional with FreqAI enabled strategies*
        This function will automatically expand the defined features on the config defined
        `include_timeframes`, `include_shifted_candles`, and `include_corr_pairs`.
        In other words, a single feature defined in this function
        will automatically expand to a total of
        `include_timeframes` * `include_shifted_candles` * `include_corr_pairs`
        numbers of features added to the model.

        Features defined here will *not* be automatically duplicated on user defined
        `indicator_periods_candles`

        All features must be prepended with `%` to be recognized by FreqAI internals.

        :param df: strategy dataframe which will receive the features
        dataframe["%-pct-change"] = dataframe["close"].pct_change()
        dataframe["%-ema-200"] = ta.EMA(dataframe, timeperiod=200)
        """
        dataframe["%-pct-change"] = dataframe["close"].pct_change()
        dataframe["%-raw_volume"] = dataframe["volume"]
        dataframe["%-raw_price"] = dataframe["close"]
        return dataframe

    def feature_engineering_standard(self, dataframe: DataFrame, **kwargs) -> DataFrame:
        """
        *Only functional with FreqAI enabled strategies*
        This optional function will be called once with the dataframe of the base timeframe.
        This is the final function to be called, which means that the dataframe entering this
        function will contain all the features and columns created by all other
        freqai_feature_engineering_* functions.

        This function is a good place to do custom exotic feature extractions (e.g. tsfresh).
        This function is a good place for any feature that should not be auto-expanded upon
        (e.g. day of the week).

        All features must be prepended with `%` to be recognized by FreqAI internals.

        :param df: strategy dataframe which will receive the features
        usage example: dataframe["%-day_of_week"] = (dataframe["date"].dt.dayofweek + 1) / 7
        """
        dataframe["%-day_of_week"] = (dataframe["date"].dt.dayofweek + 1) / 7
        dataframe["%-hour_of_day"] = (dataframe["date"].dt.hour + 1) / 25
        return dataframe

    def set_freqai_targets(self, dataframe: DataFrame, **kwargs) -> DataFrame:
        """
        *Only functional with FreqAI enabled strategies*
        Required function to set the targets for the model.
        All targets must be prepended with `&` to be recognized by the FreqAI internals.

        :param df: strategy dataframe which will receive the targets
        usage example: dataframe["&-target"] = dataframe["close"].shift(-1) / dataframe["close"]
        """
        dataframe["&-s_close"] = (
            dataframe["close"]
            .shift(-self.freqai_info["feature_parameters"]["label_period_candles"])
            .rolling(self.freqai_info["feature_parameters"]["label_period_candles"])
            .mean()
            / dataframe["close"]
            - 1
            )
        return dataframe
```

Notice how the `feature_engineering_*()` is where [features](freqai-feature-engineering.md#feature-engineering) are added. Meanwhile `set_freqai_targets()` adds the labels/targets. A full example strategy is available in `templates/FreqaiExampleStrategy.py`.

!!! Note
    The `self.freqai.start()` function cannot be called outside the `populate_indicators()`.

!!! Note
    Features **must** be defined in `feature_engineering_*()`. Defining FreqAI features in `populate_indicators()`
    will cause the algorithm to fail in live/dry mode. In order to add generalized features that are not associated with a specific pair or timeframe, you should use `feature_engineering_standard()`
    (as exemplified in `freqtrade/templates/FreqaiExampleStrategy.py`).

## Important dataframe key patterns

Below are the values you can expect to include/use inside a typical strategy dataframe (`df[]`):

|  DataFrame Key | Description |
|------------|-------------|
| `df['&*']` | Any dataframe column prepended with `&` in `set_freqai_targets()` is treated as a training target (label) inside FreqAI (typically following the naming convention `&-s*`). For example, to predict the close price 40 candles into the future, you would set `df['&-s_close'] = df['close'].shift(-self.freqai_info["feature_parameters"]["label_period_candles"])` with `"label_period_candles": 40` in the config. FreqAI makes the predictions and gives them back under the same key (`df['&-s_close']`) to be used in `populate_entry/exit_trend()`. <br> **Datatype:** Depends on the output of the model.
| `df['&*_std/mean']` | Standard deviation and mean values of the defined labels during training (or live tracking with `fit_live_predictions_candles`). Commonly used to understand the rarity of a prediction (use the z-score as shown in `templates/FreqaiExampleStrategy.py` and explained [here](#creating-a-dynamic-target-threshold) to evaluate how often a particular prediction was observed during training or historically with `fit_live_predictions_candles`). <br> **Datatype:** Float.
| `df['do_predict']` | Indication of an outlier data point. The return value is integer between -2 and 2, which lets you know if the prediction is trustworthy or not. `do_predict==1` means that the prediction is trustworthy. If the Dissimilarity Index (DI, see details [here](freqai-feature-engineering.md#identifying-outliers-with-the-dissimilarity-index-di)) of the input data point is above the threshold defined in the config, FreqAI will subtract 1 from `do_predict`, resulting in `do_predict==0`. If `use_SVM_to_remove_outliers` is active, the Support Vector Machine (SVM, see details [here](freqai-feature-engineering.md#identifying-outliers-using-a-support-vector-machine-svm)) may also detect outliers in training and prediction data. In this case, the SVM will also subtract 1 from `do_predict`. If the input data point was considered an outlier by the SVM but not by the DI, or vice versa, the result will be `do_predict==0`. If both the DI and the SVM considers the input data point to be an outlier, the result will be `do_predict==-1`. As with the SVM, if `use_DBSCAN_to_remove_outliers` is active, DBSCAN (see details [here](freqai-feature-engineering.md#identifying-outliers-with-dbscan)) may also detect outliers and subtract 1 from `do_predict`. Hence, if both the SVM and DBSCAN are active and identify a datapoint that was above the DI threshold as an outlier, the result will be `do_predict==-2`. A particular case is when `do_predict == 2`, which means that the model has expired due to exceeding `expired_hours`. <br> **Datatype:** Integer between -2 and 2.
| `df['DI_values']` | Dissimilarity Index (DI) values are proxies for the level of confidence FreqAI has in the prediction. A lower DI means the prediction is close to the training data, i.e., higher prediction confidence. See details about the DI [here](freqai-feature-engineering.md#identifying-outliers-with-the-dissimilarity-index-di). <br> **Datatype:** Float.
| `df['%*']` | Any dataframe column prepended with `%` in `feature_engineering_*()` is treated as a training feature. For example, you can include the RSI in the training feature set (similar to in `templates/FreqaiExampleStrategy.py`) by setting `df['%-rsi']`. See more details on how this is done [here](freqai-feature-engineering.md). <br> **Note:** Since the number of features prepended with `%` can multiply very quickly (10s of thousands of features are easily engineered using the multiplictative functionality of, e.g., `include_shifted_candles` and `include_timeframes` as described in the [parameter table](freqai-parameter-table.md)), these features are removed from the dataframe that is returned from FreqAI to the strategy. To keep a particular type of feature for plotting purposes, you would prepend it with `%%` (see details below). <br> **Datatype:** Depends on the feature created by the user.
| `df['%%*']` | Any dataframe column prepended with `%%` in `feature_engineering_*()` is treated as a training feature, just the same as the above `%` prepend. However, in this case, the features are returned back to the strategy for FreqUI/plot-dataframe plotting and monitoring in Dry/Live/Backtesting <br> **Datatype:** Depends on the feature created by the user. Please note that features created in `feature_engineering_expand()` will have automatic FreqAI naming schemas depending on the expansions that you configured (i.e. `include_timeframes`, `include_corr_pairlist`, `indicators_periods_candles`, `include_shifted_candles`). So if you want to plot `%%-rsi` from `feature_engineering_expand_all()`, the final naming scheme for your plotting config would be: `%%-rsi-period_10_ETH/USDT:USDT_1h` for the `rsi` feature with `period=10`, `timeframe=1h`, and `pair=ETH/USDT:USDT` (the `:USDT` is added if you are using futures pairs). It is useful to simply add `print(dataframe.columns)` in your `populate_indicators()` after `self.freqai.start()` to see the full list of available features that are returned to the strategy for plotting purposes.

## Setting the `startup_candle_count`

The `startup_candle_count` in the FreqAI strategy needs to be set up in the same way as in the standard Freqtrade strategy (see details [here](strategy-customization.md#strategy-startup-period)). This value is used by Freqtrade to ensure that a sufficient amount of data is provided when calling the `dataprovider`, to avoid any NaNs at the beginning of the first training. You can easily set this value by identifying the longest period (in candle units) which is passed to the indicator creation functions (e.g., TA-Lib functions). In the presented example, `startup_candle_count` is 20 since this is the maximum value in `indicators_periods_candles`.

!!! Note
    There are instances where the TA-Lib functions actually require more data than just the passed `period` or else the feature dataset gets populated with NaNs. Anecdotally, multiplying the `startup_candle_count` by 2 always leads to a fully NaN free training dataset. Hence, it is typically safest to multiply the expected `startup_candle_count` by 2. Look out for this log message to confirm that the data is clean:

    ```
    2022-08-31 15:14:04 - freqtrade.freqai.data_kitchen - INFO - dropped 0 training points due to NaNs in populated dataset 4319.
    ```

## Creating a dynamic target threshold

Deciding when to enter or exit a trade can be done in a dynamic way to reflect current market conditions. FreqAI allows you to return additional information from the training of a model (more info [here](freqai-feature-engineering.md#returning-additional-info-from-training)). For example, the `&*_std/mean` return values describe the statistical distribution of the target/label *during the most recent training*. Comparing a given prediction to these values allows you to know the rarity of the prediction. In `templates/FreqaiExampleStrategy.py`, the `target_roi` and  `sell_roi` are defined to be 1.25 z-scores away from the mean which causes predictions that are closer to the mean to be filtered out.

```python
dataframe["target_roi"] = dataframe["&-s_close_mean"] + dataframe["&-s_close_std"] * 1.25
dataframe["sell_roi"] = dataframe["&-s_close_mean"] - dataframe["&-s_close_std"] * 1.25
```

To consider the population of *historical predictions* for creating the dynamic target instead of information from the training as discussed above, you would set `fit_live_predictions_candles` in the config to the number of historical prediction candles you wish to use to generate target statistics.

```json
    "freqai": {
        "fit_live_predictions_candles": 300,
    }
```

If this value is set, FreqAI will initially use the predictions from the training data and subsequently begin introducing real prediction data as it is generated. FreqAI will save this historical data to be reloaded if you stop and restart a model with the same `identifier`.

## Using different prediction models

FreqAI has multiple example prediction model libraries that are ready to be used as is via the flag `--freqaimodel`. These libraries include `CatBoost`, `LightGBM`, and `XGBoost` regression, classification, and multi-target models, and can be found in `freqai/prediction_models/`.

Regression and classification models differ in what targets they predict - a regression model will predict a target of continuous values, for example what price BTC will be at tomorrow, whilst a classifier will predict a target of discrete values, for example if the price of BTC will go up tomorrow or not. This means that you have to specify your targets differently depending on which model type you are using (see details [below](#setting-model-targets)).

All of the aforementioned model libraries implement gradient boosted decision tree algorithms. They all work on the principle of ensemble learning, where predictions from multiple simple learners are combined to get a final prediction that is more stable and generalized. The simple learners in this case are decision trees. Gradient boosting refers to the method of learning, where each simple learner is built in sequence - the subsequent learner is used to improve on the error from the previous learner. If you want to learn more about the different model libraries you can find the information in their respective docs:

* CatBoost: https://catboost.ai/en/docs/
* LightGBM: https://lightgbm.readthedocs.io/en/v3.3.2/#
* XGBoost: https://xgboost.readthedocs.io/en/stable/#

There are also numerous online articles describing and comparing the algorithms. Some relatively lightweight examples would be [CatBoost vs. LightGBM vs. XGBoost — Which is the best algorithm?](https://towardsdatascience.com/catboost-vs-lightgbm-vs-xgboost-c80f40662924#:~:text=In%20CatBoost%2C%20symmetric%20trees%2C%20or,the%20same%20depth%20can%20differ.) and [XGBoost, LightGBM or CatBoost — which boosting algorithm should I use?](https://medium.com/riskified-technology/xgboost-lightgbm-or-catboost-which-boosting-algorithm-should-i-use-e7fda7bb36bc). Keep in mind that the performance of each model is highly dependent on the application and so any reported metrics might not be true for your particular use of the model.

Apart from the models already available in FreqAI, it is also possible to customize and create your own prediction models using the `IFreqaiModel` class. You are encouraged to inherit `fit()`, `train()`, and `predict()` to customize various aspects of the training procedures. You can place custom FreqAI models in `user_data/freqaimodels` - and freqtrade will pick them up from there based on the provided `--freqaimodel` name - which has to correspond to the class name of your custom model.
Make sure to use unique names to avoid overriding built-in models.

### Setting model targets

#### Regressors

If you are using a regressor, you need to specify a target that has continuous values. FreqAI includes a variety of regressors, such as the `CatboostRegressor`via the flag `--freqaimodel CatboostRegressor`. An example of how you could set a regression target for predicting the price 100 candles into the future would be

```python
df['&s-close_price'] = df['close'].shift(-100)
```

If you want to predict multiple targets, you need to define multiple labels using the same syntax as shown above.

#### Classifiers

If you are using a classifier, you need to specify a target that has discrete values. FreqAI includes a variety of classifiers, such as the `CatboostClassifier` via the flag `--freqaimodel CatboostClassifier`. If you elects to use a classifier, the classes need to be set using strings. For example, if you want to predict if the price 100 candles into the future goes up or down you would set

```python
df['&s-up_or_down'] = np.where( df["close"].shift(-100) > df["close"], 'up', 'down')
```

If you want to predict multiple targets you must specify all labels in the same label column. You could, for example, add the label `same` to define where the price was unchanged by setting

```python
df['&s-up_or_down'] = np.where( df["close"].shift(-100) > df["close"], 'up', 'down')
df['&s-up_or_down'] = np.where( df["close"].shift(-100) == df["close"], 'same', df['&s-up_or_down'])
```

## PyTorch Module

### Quick start

The easiest way to quickly run a pytorch model is with the following command (for regression task):

```bash
freqtrade trade --config config_examples/config_freqai.example.json --strategy FreqaiExampleStrategy --freqaimodel PyTorchMLPRegressor --strategy-path freqtrade/templates 
```

!!! Note "Installation/docker"
    The PyTorch module requires large packages such as `torch`, which should be explicitly requested during `./setup.sh -i` by answering "y" to the question "Do you also want dependencies for freqai-rl or PyTorch (~700mb additional space required) [y/N]?".
    Users who prefer docker should ensure they use the docker image appended with `_freqaitorch`.
    We do provide an explicit docker-compose file for this in `docker/docker-compose-freqai.yml` - which can be used via `docker compose -f docker/docker-compose-freqai.yml run ...` - or can be copied to replace the original docker file.
    This docker-compose file also contains a (disabled) section to enable GPU resources within docker containers. This obviously assumes the system has GPU resources available.

### Structure

#### Model

You can construct your own Neural Network architecture in PyTorch by simply defining your `nn.Module` class inside your custom [`IFreqaiModel` file](#using-different-prediction-models) and then using that class in your `def train()` function. Here is an example of logistic regression model implementation using PyTorch (should be used with nn.BCELoss criterion) for classification tasks.

```python

class LogisticRegression(nn.Module):
    def __init__(self, input_size: int):
        super().__init__()
        # Define your layers
        self.linear = nn.Linear(input_size, 1)
        self.activation = nn.Sigmoid()

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        # Define the forward pass
        out = self.linear(x)
        out = self.activation(out)
        return out

class MyCoolPyTorchClassifier(BasePyTorchClassifier):
    """
    This is a custom IFreqaiModel showing how a user might setup their own 
    custom Neural Network architecture for their training.
    """

    @property
    def data_convertor(self) -> PyTorchDataConvertor:
        return DefaultPyTorchDataConvertor(target_tensor_type=torch.float)

    def __init__(self, **kwargs) -> None:
        super().__init__(**kwargs)
        config = self.freqai_info.get("model_training_parameters", {})
        self.learning_rate: float = config.get("learning_rate",  3e-4)
        self.model_kwargs: dict[str, Any] = config.get("model_kwargs",  {})
        self.trainer_kwargs: dict[str, Any] = config.get("trainer_kwargs",  {})

    def fit(self, data_dictionary: dict, dk: FreqaiDataKitchen, **kwargs) -> Any:
        """
        User sets up the training and test data to fit their desired model here
        :param data_dictionary: the dictionary holding all data for train, test,
            labels, weights
        :param dk: The datakitchen object for the current coin/model
        """

        class_names = self.get_class_names()
        self.convert_label_column_to_int(data_dictionary, dk, class_names)
        n_features = data_dictionary["train_features"].shape[-1]
        model = LogisticRegression(
            input_dim=n_features
        )
        model.to(self.device)
        optimizer = torch.optim.AdamW(model.parameters(), lr=self.learning_rate)
        criterion = torch.nn.CrossEntropyLoss()
        init_model = self.get_init_model(dk.pair)
        trainer = PyTorchModelTrainer(
            model=model,
            optimizer=optimizer,
            criterion=criterion,
            model_meta_data={"class_names": class_names},
            device=self.device,
            init_model=init_model,
            data_convertor=self.data_convertor,
            **self.trainer_kwargs,
        )
        trainer.fit(data_dictionary, self.splits)
        return trainer

```

#### Trainer

The `PyTorchModelTrainer` performs the idiomatic PyTorch train loop:
Define our model, loss function, and optimizer, and then move them to the appropriate device (GPU or CPU). Inside the loop, we iterate through the batches in the dataloader, move the data to the device, compute the prediction and loss, backpropagate, and update the model parameters using the optimizer. 

In addition, the trainer is responsible for the following:
 - saving and loading the model
 - converting the data from `pandas.DataFrame` to `torch.Tensor`. 

#### Integration with Freqai module 

Like all freqai models, PyTorch models inherit `IFreqaiModel`. `IFreqaiModel` declares three abstract methods: `train`, `fit`, and `predict`. we implement these methods in three levels of hierarchy.
From top to bottom:

1. `BasePyTorchModel` - Implements the `train` method. all `BasePyTorch*` inherit it. responsible for general data preparation (e.g., data normalization) and calling the `fit` method. Sets `device` attribute used by children classes. Sets `model_type` attribute used by the parent class.
2. `BasePyTorch*` -  Implements the `predict` method. Here, the `*` represents a group of algorithms, such as classifiers or regressors. responsible for data preprocessing, predicting, and postprocessing if needed.
3. `PyTorch*Classifier` / `PyTorch*Regressor` - implements the `fit` method. responsible for the main train flaw, where we initialize the trainer and model objects.

![image](assets/freqai_pytorch-diagram.png)

#### Full example

Building a PyTorch regressor using MLP (multilayer perceptron) model, MSELoss criterion, and AdamW optimizer.

```python
class PyTorchMLPRegressor(BasePyTorchRegressor):
    def __init__(self, **kwargs) -> None:
        super().__init__(**kwargs)
        config = self.freqai_info.get("model_training_parameters", {})
        self.learning_rate: float = config.get("learning_rate",  3e-4)
        self.model_kwargs: dict[str, Any] = config.get("model_kwargs",  {})
        self.trainer_kwargs: dict[str, Any] = config.get("trainer_kwargs",  {})

    def fit(self, data_dictionary: dict, dk: FreqaiDataKitchen, **kwargs) -> Any:
        n_features = data_dictionary["train_features"].shape[-1]
        model = PyTorchMLPModel(
            input_dim=n_features,
            output_dim=1,
            **self.model_kwargs
        )
        model.to(self.device)
        optimizer = torch.optim.AdamW(model.parameters(), lr=self.learning_rate)
        criterion = torch.nn.MSELoss()
        init_model = self.get_init_model(dk.pair)
        trainer = PyTorchModelTrainer(
            model=model,
            optimizer=optimizer,
            criterion=criterion,
            device=self.device,
            init_model=init_model,
            target_tensor_type=torch.float,
            **self.trainer_kwargs,
        )
        trainer.fit(data_dictionary)
        return trainer
```

Here we create a `PyTorchMLPRegressor` class that implements the `fit` method. The `fit` method specifies the training building blocks: model, optimizer, criterion, and trainer. We inherit both `BasePyTorchRegressor` and `BasePyTorchModel`, where the former implements the `predict` method that is suitable for our regression task, and the latter implements the train method.

??? Note "Setting Class Names for Classifiers"
    When using classifiers, the user must declare the class names (or targets) by overriding the `IFreqaiModel.class_names` attribute. This is achieved by setting `self.freqai.class_names` in the FreqAI strategy inside the `set_freqai_targets` method.
    
    For example, if you are using a binary classifier to predict price movements as up or down, you can set the class names as follows:
    ```python
    def set_freqai_targets(self, dataframe: DataFrame, metadata: dict, **kwargs) -> DataFrame:
        self.freqai.class_names = ["down", "up"]
        dataframe['&s-up_or_down'] = np.where(dataframe["close"].shift(-100) >
                                                  dataframe["close"], 'up', 'down')
    
        return dataframe
    ```
    To see a full example, you can refer to the [classifier test strategy class](https://github.com/freqtrade/freqtrade/blob/develop/tests/strategy/strats/freqai_test_classifier.py).


#### Improving performance with `torch.compile()`

Torch provides a `torch.compile()` method that can be used to improve performance for specific GPU hardware. More details can be found [here](https://pytorch.org/tutorials/intermediate/torch_compile_tutorial.html). In brief, you simply wrap your `model` in `torch.compile()`:


```python
        model = PyTorchMLPModel(
            input_dim=n_features,
            output_dim=1,
            **self.model_kwargs
        )
        model.to(self.device)
        model = torch.compile(model)
```

Then proceed to use the model as normal. Keep in mind that doing this will remove eager execution, which means errors and tracebacks will not be informative.

---

## File: freqai-developers.md

# Development

## Project architecture

The architecture and functions of FreqAI are generalized to encourages development of unique features, functions, models, etc.

The class structure and a detailed algorithmic overview is depicted in the following diagram:

![image](assets/freqai_algorithm-diagram.jpg)

As shown, there are three distinct objects comprising FreqAI:

* **IFreqaiModel** - A singular persistent object containing all the necessary logic to collect, store, and process data, engineer features, run training, and inference models.
* **FreqaiDataKitchen** - A non-persistent object which is created uniquely for each unique asset/model. Beyond metadata, it also contains a variety of data processing tools.
* **FreqaiDataDrawer** - A singular persistent object containing all the historical predictions, models, and save/load methods.

There are a variety of built-in [prediction models](freqai-configuration.md#using-different-prediction-models) which inherit directly from `IFreqaiModel`. Each of these models have full access to all methods in `IFreqaiModel` and can therefore override any of those functions at will. However, advanced users will likely stick to overriding `fit()`, `train()`, `predict()`, and `data_cleaning_train/predict()`.

## Data handling

FreqAI aims to organize model files, prediction data, and meta data in a way that simplifies post-processing and enhances crash resilience by automatic data reloading. The data is saved in a file structure,`user_data_dir/models/`, which contains all the data associated with the trainings and backtests. The `FreqaiDataKitchen()` relies heavily on the file structure for proper training and inferencing and should therefore not be manually modified.

### File structure

The file structure is automatically generated based on the model `identifier` set in the [config](freqai-configuration.md#setting-up-the-configuration-file). The following structure shows where the data is stored for post processing:

| Structure | Description |
|-----------|-------------|
| `config_*.json` | A copy of the model specific configuration file. |
| `historic_predictions.pkl` | A file containing all historic predictions generated during the lifetime of the `identifier` model during live deployment. `historic_predictions.pkl` is used to reload the model after a crash or a config change. A backup file is always held in case of corruption on the main file. FreqAI **automatically** detects corruption and replaces the corrupted file with the backup. |
| `pair_dictionary.json` | A file containing the training queue as well as the on disk location of the most recently trained model. |
| `sub-train-*_TIMESTAMP` | A folder containing all the files associated with a single model, such as: <br>
|| `*_metadata.json` - Metadata for the model, such as normalization max/min, expected training feature list, etc. <br>
|| `*_model.*` - The model file saved to disk for reloading from a crash. Can be `joblib` (typical boosting libs), `zip` (stable_baselines), `hd5` (keras type), etc. <br>
|| `*_pca_object.pkl` - The [Principal component analysis (PCA)](freqai-feature-engineering.md#data-dimensionality-reduction-with-principal-component-analysis) transform (if `principal_component_analysis: True` is set in the config) which will be used to transform unseen prediction features. <br>
|| `*_svm_model.pkl` - The [Support Vector Machine (SVM)](freqai-feature-engineering.md#identifying-outliers-using-a-support-vector-machine-svm) model (if `use_SVM_to_remove_outliers: True` is set in the config) which is used to detect outliers in unseen prediction features. <br>
|| `*_trained_df.pkl` - The dataframe containing all the training features used to train the `identifier` model. This is used for computing the [Dissimilarity Index (DI)](freqai-feature-engineering.md#identifying-outliers-with-the-dissimilarity-index-di) and can also be used for post-processing. <br>
|| `*_trained_dates.df.pkl` - The dates associated with the `trained_df.pkl`, which is useful for post-processing. |

The example file structure would look like this:

```
├── models
│   └── unique-id
│       ├── config_freqai.example.json
│       ├── historic_predictions.backup.pkl
│       ├── historic_predictions.pkl
│       ├── pair_dictionary.json
│       ├── sub-train-1INCH_1662821319
│       │   ├── cb_1inch_1662821319_metadata.json
│       │   ├── cb_1inch_1662821319_model.joblib
│       │   ├── cb_1inch_1662821319_pca_object.pkl
│       │   ├── cb_1inch_1662821319_svm_model.joblib
│       │   ├── cb_1inch_1662821319_trained_dates_df.pkl
│       │   └── cb_1inch_1662821319_trained_df.pkl
│       ├── sub-train-1INCH_1662821371
│       │   ├── cb_1inch_1662821371_metadata.json
│       │   ├── cb_1inch_1662821371_model.joblib
│       │   ├── cb_1inch_1662821371_pca_object.pkl
│       │   ├── cb_1inch_1662821371_svm_model.joblib
│       │   ├── cb_1inch_1662821371_trained_dates_df.pkl
│       │   └── cb_1inch_1662821371_trained_df.pkl
│       ├── sub-train-ADA_1662821344
│       │   ├── cb_ada_1662821344_metadata.json
│       │   ├── cb_ada_1662821344_model.joblib
│       │   ├── cb_ada_1662821344_pca_object.pkl
│       │   ├── cb_ada_1662821344_svm_model.joblib
│       │   ├── cb_ada_1662821344_trained_dates_df.pkl
│       │   └── cb_ada_1662821344_trained_df.pkl
│       └── sub-train-ADA_1662821399
│           ├── cb_ada_1662821399_metadata.json
│           ├── cb_ada_1662821399_model.joblib
│           ├── cb_ada_1662821399_pca_object.pkl
│           ├── cb_ada_1662821399_svm_model.joblib
│           ├── cb_ada_1662821399_trained_dates_df.pkl
│           └── cb_ada_1662821399_trained_df.pkl

```

---

## File: freqai-feature-engineering.md

# Feature engineering

## Defining the features

Low level feature engineering is performed in the user strategy within a set of functions called `feature_engineering_*`. These function set the `base features` such as, `RSI`, `MFI`, `EMA`, `SMA`, time of day, volume, etc. The `base features` can be custom indicators or they can be imported from any technical-analysis library that you can find. FreqAI is equipped with a set of functions to simplify rapid large-scale feature engineering:

|  Function | Description |
|---------------|-------------|
| `feature_engineering_expand_all()` | This optional function will automatically expand the defined features on the config defined `indicator_periods_candles`, `include_timeframes`, `include_shifted_candles`, and `include_corr_pairs`.
| `feature_engineering_expand_basic()` | This optional function will automatically expand the defined features on the config defined `include_timeframes`, `include_shifted_candles`, and `include_corr_pairs`. Note: this function does *not* expand across `indicator_periods_candles`.
| `feature_engineering_standard()` | This optional function will be called once with the dataframe of the base timeframe. This is the final function to be called, which means that the dataframe entering this function will contain all the features and columns from the base asset created by the other `feature_engineering_expand` functions. This function is a good place to do custom exotic feature extractions (e.g. tsfresh). This function is also a good place for any feature that should not be auto-expanded upon (e.g., day of the week).
| `set_freqai_targets()` | Required function to set the targets for the model. All targets must be prepended with `&` to be recognized by the FreqAI internals.

Meanwhile, high level feature engineering is handled within `"feature_parameters":{}` in the FreqAI config. Within this file, it is possible to decide large scale feature expansions on top of the `base_features` such as "including correlated pairs" or "including informative timeframes" or even "including recent candles."

It is advisable to start from the template `feature_engineering_*` functions in the source provided example strategy (found in `templates/FreqaiExampleStrategy.py`) to ensure that the feature definitions are following the correct conventions. Here is an example of how to set the indicators and labels in the strategy:

```python
    def feature_engineering_expand_all(self, dataframe: DataFrame, period, metadata, **kwargs) -> DataFrame:
        """
        *Only functional with FreqAI enabled strategies*
        This function will automatically expand the defined features on the config defined
        `indicator_periods_candles`, `include_timeframes`, `include_shifted_candles`, and
        `include_corr_pairs`. In other words, a single feature defined in this function
        will automatically expand to a total of
        `indicator_periods_candles` * `include_timeframes` * `include_shifted_candles` *
        `include_corr_pairs` numbers of features added to the model.

        All features must be prepended with `%` to be recognized by FreqAI internals.

        Access metadata such as the current pair/timeframe/period with:

        `metadata["pair"]` `metadata["tf"]`  `metadata["period"]`

        :param df: strategy dataframe which will receive the features
        :param period: period of the indicator - usage example:
        :param metadata: metadata of current pair
        dataframe["%-ema-period"] = ta.EMA(dataframe, timeperiod=period)
        """

        dataframe["%-rsi-period"] = ta.RSI(dataframe, timeperiod=period)
        dataframe["%-mfi-period"] = ta.MFI(dataframe, timeperiod=period)
        dataframe["%-adx-period"] = ta.ADX(dataframe, timeperiod=period)
        dataframe["%-sma-period"] = ta.SMA(dataframe, timeperiod=period)
        dataframe["%-ema-period"] = ta.EMA(dataframe, timeperiod=period)

        bollinger = qtpylib.bollinger_bands(
            qtpylib.typical_price(dataframe), window=period, stds=2.2
        )
        dataframe["bb_lowerband-period"] = bollinger["lower"]
        dataframe["bb_middleband-period"] = bollinger["mid"]
        dataframe["bb_upperband-period"] = bollinger["upper"]

        dataframe["%-bb_width-period"] = (
            dataframe["bb_upperband-period"]
            - dataframe["bb_lowerband-period"]
        ) / dataframe["bb_middleband-period"]
        dataframe["%-close-bb_lower-period"] = (
            dataframe["close"] / dataframe["bb_lowerband-period"]
        )

        dataframe["%-roc-period"] = ta.ROC(dataframe, timeperiod=period)

        dataframe["%-relative_volume-period"] = (
            dataframe["volume"] / dataframe["volume"].rolling(period).mean()
        )

        return dataframe

    def feature_engineering_expand_basic(self, dataframe: DataFrame, metadata, **kwargs) -> DataFrame:
        """
        *Only functional with FreqAI enabled strategies*
        This function will automatically expand the defined features on the config defined
        `include_timeframes`, `include_shifted_candles`, and `include_corr_pairs`.
        In other words, a single feature defined in this function
        will automatically expand to a total of
        `include_timeframes` * `include_shifted_candles` * `include_corr_pairs`
        numbers of features added to the model.

        Features defined here will *not* be automatically duplicated on user defined
        `indicator_periods_candles`

        Access metadata such as the current pair/timeframe with:

        `metadata["pair"]` `metadata["tf"]`

        All features must be prepended with `%` to be recognized by FreqAI internals.

        :param df: strategy dataframe which will receive the features
        :param metadata: metadata of current pair
        dataframe["%-pct-change"] = dataframe["close"].pct_change()
        dataframe["%-ema-200"] = ta.EMA(dataframe, timeperiod=200)
        """
        dataframe["%-pct-change"] = dataframe["close"].pct_change()
        dataframe["%-raw_volume"] = dataframe["volume"]
        dataframe["%-raw_price"] = dataframe["close"]
        return dataframe

    def feature_engineering_standard(self, dataframe: DataFrame, metadata, **kwargs) -> DataFrame:
        """
        *Only functional with FreqAI enabled strategies*
        This optional function will be called once with the dataframe of the base timeframe.
        This is the final function to be called, which means that the dataframe entering this
        function will contain all the features and columns created by all other
        freqai_feature_engineering_* functions.

        This function is a good place to do custom exotic feature extractions (e.g. tsfresh).
        This function is a good place for any feature that should not be auto-expanded upon
        (e.g. day of the week).

        Access metadata such as the current pair with:

        `metadata["pair"]`

        All features must be prepended with `%` to be recognized by FreqAI internals.

        :param df: strategy dataframe which will receive the features
        :param metadata: metadata of current pair
        usage example: dataframe["%-day_of_week"] = (dataframe["date"].dt.dayofweek + 1) / 7
        """
        dataframe["%-day_of_week"] = (dataframe["date"].dt.dayofweek + 1) / 7
        dataframe["%-hour_of_day"] = (dataframe["date"].dt.hour + 1) / 25
        return dataframe

    def set_freqai_targets(self, dataframe: DataFrame, metadata, **kwargs) -> DataFrame:
        """
        *Only functional with FreqAI enabled strategies*
        Required function to set the targets for the model.
        All targets must be prepended with `&` to be recognized by the FreqAI internals.

        Access metadata such as the current pair with:

        `metadata["pair"]`

        :param df: strategy dataframe which will receive the targets
        :param metadata: metadata of current pair
        usage example: dataframe["&-target"] = dataframe["close"].shift(-1) / dataframe["close"]
        """
        dataframe["&-s_close"] = (
            dataframe["close"]
            .shift(-self.freqai_info["feature_parameters"]["label_period_candles"])
            .rolling(self.freqai_info["feature_parameters"]["label_period_candles"])
            .mean()
            / dataframe["close"]
            - 1
            )
        
        return dataframe
```

In the presented example, the user does not wish to pass the `bb_lowerband` as a feature to the model,
and has therefore not prepended it with `%`. The user does, however, wish to pass `bb_width` to the
model for training/prediction and has therefore prepended it with `%`.

After having defined the `base features`, the next step is to expand upon them using the powerful `feature_parameters` in the configuration file:

```json
    "freqai": {
        //...
        "feature_parameters" : {
            "include_timeframes": ["5m","15m","4h"],
            "include_corr_pairlist": [
                "ETH/USD",
                "LINK/USD",
                "BNB/USD"
            ],
            "label_period_candles": 24,
            "include_shifted_candles": 2,
            "indicator_periods_candles": [10, 20]
        },
        //...
    }
```

The `include_timeframes` in the config above are the timeframes (`tf`) of each call to `feature_engineering_expand_*()` in the strategy. In the presented case, the user is asking for the `5m`, `15m`, and `4h` timeframes of the `rsi`, `mfi`, `roc`, and `bb_width` to be included in the feature set.

You can ask for each of the defined features to be included also for informative pairs using the `include_corr_pairlist`. This means that the feature set will include all the features from `feature_engineering_expand_*()` on all the `include_timeframes` for each of the correlated pairs defined in the config (`ETH/USD`, `LINK/USD`, and `BNB/USD` in the presented example).

`include_shifted_candles` indicates the number of previous candles to include in the feature set. For example, `include_shifted_candles: 2` tells FreqAI to include the past 2 candles for each of the features in the feature set.

In total, the number of features the user of the presented example strategy has created is: length of `include_timeframes` * no. features in `feature_engineering_expand_*()` * length of `include_corr_pairlist` * no. `include_shifted_candles` * length of `indicator_periods_candles`
 $= 3 * 3 * 3 * 2 * 2 = 108$.
 
 !!! note "Learn more about creative feature engineering"
    Check out our [medium article](https://emergentmethods.medium.com/freqai-from-price-to-prediction-6fadac18b665) geared toward helping users learn how to creatively engineer features.

### Gain finer control over `feature_engineering_*` functions with `metadata`

All `feature_engineering_*` and `set_freqai_targets()` functions are passed a `metadata` dictionary which contains information about the `pair`, `tf` (timeframe), and `period` that FreqAI is automating for feature building. As such, a user can use `metadata` inside `feature_engineering_*` functions as criteria for blocking/reserving features for certain timeframes, periods, pairs etc.

```python
def feature_engineering_expand_all(self, dataframe: DataFrame, period, metadata, **kwargs) -> DataFrame:
    if metadata["tf"] == "1h":
        dataframe["%-roc-period"] = ta.ROC(dataframe, timeperiod=period)
```

This will block `ta.ROC()` from being added to any timeframes other than `"1h"`.

### Returning additional info from training

Important metrics can be returned to the strategy at the end of each model training by assigning them to `dk.data['extra_returns_per_train']['my_new_value'] = XYZ` inside the custom prediction model class. 

FreqAI takes the `my_new_value` assigned in this dictionary and expands it to fit the dataframe that is returned to the strategy. You can then use the returned metrics in your strategy through `dataframe['my_new_value']`. An example of how return values can be used in FreqAI are the `&*_mean` and `&*_std` values that are used to [created a dynamic target threshold](freqai-configuration.md#creating-a-dynamic-target-threshold).

Another example, where the user wants to use live metrics from the trade database, is shown below:

```json
    "freqai": {
        "extra_returns_per_train": {"total_profit": 4}
    }
```

You need to set the standard dictionary in the config so that FreqAI can return proper dataframe shapes. These values will likely be overridden by the prediction model, but in the case where the model has yet to set them, or needs a default initial value, the pre-set values are what will be returned.

### Weighting features for temporal importance

FreqAI allows you to set a `weight_factor` to weight recent data more strongly than past data via an exponential function:

$$ W_i = \exp(\frac{-i}{\alpha*n}) $$

where $W_i$ is the weight of data point $i$ in a total set of $n$ data points. Below is a figure showing the effect of different weight factors on the data points in a feature set.

![weight-factor](assets/freqai_weight-factor.jpg)

## Building the data pipeline

By default, FreqAI builds a dynamic pipeline based on user configuration settings. The default settings are robust and designed to work with a variety of methods. These two steps are a `MinMaxScaler(-1,1)` and a `VarianceThreshold` which removes any column that has 0 variance. Users can activate other steps with more configuration parameters. For example if users add `use_SVM_to_remove_outliers: true` to the `freqai` config, then FreqAI will automatically add the [`SVMOutlierExtractor`](#identifying-outliers-using-a-support-vector-machine-svm) to the pipeline. Likewise, users can add `principal_component_analysis: true` to the `freqai` config to activate PCA. The [DissimilarityIndex](#identifying-outliers-with-the-dissimilarity-index-di) is activated with `DI_threshold: 1`. Finally, noise can also be added to the data with `noise_standard_deviation: 0.1`. Finally, users can add [DBSCAN](#identifying-outliers-with-dbscan) outlier removal with `use_DBSCAN_to_remove_outliers: true`.

!!! note "More information available"
    Please review the [parameter table](freqai-parameter-table.md) for more information on these parameters.


### Customizing the pipeline

Users are encouraged to customize the data pipeline to their needs by building their own data pipeline. This can be done by simply setting `dk.feature_pipeline` to their desired `Pipeline` object inside their `IFreqaiModel` `train()` function, or if they prefer not to touch the `train()` function, they can override `define_data_pipeline`/`define_label_pipeline` functions in their `IFreqaiModel`:

!!! note "More information available"
    FreqAI uses the [`DataSieve`](https://github.com/emergentmethods/datasieve) pipeline, which follows the SKlearn pipeline API, but adds, among other features, coherence between the X, y, and sample_weight vector point removals, feature removal, feature name following. 

```python
from datasieve.transforms import SKLearnWrapper, DissimilarityIndex
from datasieve.pipeline import Pipeline
from sklearn.preprocessing import QuantileTransformer, StandardScaler
from freqai.base_models import BaseRegressionModel


class MyFreqaiModel(BaseRegressionModel):
    """
    Some cool custom model
    """
    def fit(self, data_dictionary: Dict, dk: FreqaiDataKitchen, **kwargs) -> Any:
        """
        My custom fit function
        """
        model = cool_model.fit()
        return model

    def define_data_pipeline(self) -> Pipeline:
        """
        User defines their custom feature pipeline here (if they wish)
        """
        feature_pipeline = Pipeline([
            ('qt', SKLearnWrapper(QuantileTransformer(output_distribution='normal'))),
            ('di', ds.DissimilarityIndex(di_threshold=1))
        ])

        return feature_pipeline
    
    def define_label_pipeline(self) -> Pipeline:
        """
        User defines their custom label pipeline here (if they wish)
        """
        label_pipeline = Pipeline([
            ('qt', SKLearnWrapper(StandardScaler())),
        ])

        return label_pipeline
```

Here, you are defining the exact pipeline that will be used for your feature set during training and prediction. You can use *most* SKLearn transformation steps by wrapping them in the `SKLearnWrapper` class as shown above. In addition, you can use any of the transformations available in the [`DataSieve` library](https://github.com/emergentmethods/datasieve). 

You can easily add your own transformation by creating a class that inherits from the datasieve `BaseTransform` and implementing your `fit()`, `transform()` and `inverse_transform()` methods:

```python
from datasieve.transforms.base_transform import BaseTransform
# import whatever else you need

class MyCoolTransform(BaseTransform):
    def __init__(self, **kwargs):
        self.param1 = kwargs.get('param1', 1)

    def fit(self, X, y=None, sample_weight=None, feature_list=None, **kwargs):
        # do something with X, y, sample_weight, or/and feature_list
        return X, y, sample_weight, feature_list

    def transform(self, X, y=None, sample_weight=None,
                  feature_list=None, outlier_check=False, **kwargs):
        # do something with X, y, sample_weight, or/and feature_list
        return X, y, sample_weight, feature_list

    def inverse_transform(self, X, y=None, sample_weight=None, feature_list=None, **kwargs):
        # do/dont do something with X, y, sample_weight, or/and feature_list
        return X, y, sample_weight, feature_list
```

!!! note "Hint"
    You can define this custom class in the same file as your `IFreqaiModel`.

### Migrating a custom `IFreqaiModel` to the new Pipeline

If you have created your own custom `IFreqaiModel` with a custom `train()`/`predict()` function, *and* you still rely on `data_cleaning_train/predict()`, then you will need to migrate to the new pipeline. If your model does *not* rely on `data_cleaning_train/predict()`, then you do not need to worry about this migration.

More details about the migration can be found [here](strategy_migration.md#freqai---new-data-pipeline).

## Outlier detection

Equity and crypto markets suffer from a high level of non-patterned noise in the form of outlier data points. FreqAI implements a variety of methods to identify such outliers and hence mitigate risk.

### Identifying outliers with the Dissimilarity Index (DI)

The Dissimilarity Index (DI) aims to quantify the uncertainty associated with each prediction made by the model. 

You can tell FreqAI to remove outlier data points from the training/test data sets using the DI by including the following statement in the config:

```json
    "freqai": {
        "feature_parameters" : {
            "DI_threshold": 1
        }
    }
```

Which will add `DissimilarityIndex` step to your `feature_pipeline` and set the threshold to 1. The DI allows predictions which are outliers (not existent in the model feature space) to be thrown out due to low levels of certainty. To do so, FreqAI measures the distance between each training data point (feature vector), $X_{a}$, and all other training data points:

$$ d_{ab} = \sqrt{\sum_{j=1}^p(X_{a,j}-X_{b,j})^2} $$

where $d_{ab}$ is the distance between the normalized points $a$ and $b$, and $p$ is the number of features, i.e., the length of the vector $X$. The characteristic distance, $\overline{d}$, for a set of training data points is simply the mean of the average distances:

$$ \overline{d} = \sum_{a=1}^n(\sum_{b=1}^n(d_{ab}/n)/n) $$

$\overline{d}$ quantifies the spread of the training data, which is compared to the distance between a new prediction feature vectors, $X_k$ and all the training data:

$$ d_k = \arg \min d_{k,i} $$

This enables the estimation of the Dissimilarity Index as:

$$ DI_k = d_k/\overline{d} $$

You can tweak the DI through the `DI_threshold` to increase or decrease the extrapolation of the trained model. A higher `DI_threshold` means that the DI is more lenient and allows predictions further away from the training data to be used whilst a lower `DI_threshold` has the opposite effect and hence discards more predictions.

Below is a figure that describes the DI for a 3D data set.

![DI](assets/freqai_DI.jpg)

### Identifying outliers using a Support Vector Machine (SVM)

You can tell FreqAI to remove outlier data points from the training/test data sets using a Support Vector Machine (SVM) by including the following statement in the config:

```json
    "freqai": {
        "feature_parameters" : {
            "use_SVM_to_remove_outliers": true
        }
    }
```

Which will add `SVMOutlierExtractor` step to your `feature_pipeline`. The SVM will be trained on the training data and any data point that the SVM deems to be beyond the feature space will be removed.

You can elect to provide additional parameters for the SVM, such as `shuffle`, and `nu` via the `feature_parameters.svm_params` dictionary in the config.

The parameter `shuffle` is by default set to `False` to ensure consistent results. If it is set to `True`, running the SVM multiple times on the same data set might result in different outcomes due to `max_iter` being to low for the algorithm to reach the demanded `tol`. Increasing `max_iter` solves this issue but causes the procedure to take longer time.

The parameter `nu`, *very* broadly, is the amount of data points that should be considered outliers and should be between 0 and 1.

### Identifying outliers with DBSCAN

You can configure FreqAI to use DBSCAN to cluster and remove outliers from the training/test data set or incoming outliers from predictions, by activating `use_DBSCAN_to_remove_outliers` in the config:

```json
    "freqai": {
        "feature_parameters" : {
            "use_DBSCAN_to_remove_outliers": true
        }
    }
```

Which will add the `DataSieveDBSCAN` step to your `feature_pipeline`. This is an unsupervised machine learning algorithm that clusters data without needing to know how many clusters there should be.

Given a number of data points $N$, and a distance $\varepsilon$, DBSCAN clusters the data set by setting all data points that have $N-1$ other data points within a distance of $\varepsilon$ as *core points*. A data point that is within a distance of $\varepsilon$ from a *core point* but that does not have $N-1$ other data points within a distance of $\varepsilon$ from itself is considered an *edge point*. A cluster is then the collection of *core points* and *edge points*. Data points that have no other data points at a distance $<\varepsilon$ are considered outliers. The figure below shows a cluster with $N = 3$.

![dbscan](assets/freqai_dbscan.jpg)

FreqAI uses `sklearn.cluster.DBSCAN` (details are available on scikit-learn's webpage [here](https://scikit-learn.org/stable/modules/generated/sklearn.cluster.DBSCAN.html) (external website)) with `min_samples` ($N$) taken as 1/4 of the no. of time points (candles) in the feature set. `eps` ($\varepsilon$) is computed automatically as the elbow point in the *k-distance graph* computed from the nearest neighbors in the pairwise distances of all data points in the feature set.


### Data dimensionality reduction with Principal Component Analysis

You can reduce the dimensionality of your features by activating the principal_component_analysis in the config:

```json
    "freqai": {
        "feature_parameters" : {
            "principal_component_analysis": true
        }
    }
```

This will perform PCA on the features and reduce their dimensionality so that the explained variance of the data set is >= 0.999. Reducing data dimensionality makes training the model faster and hence allows for more up-to-date models.

---

## File: freqai.md

![freqai-logo](assets/freqai_doc_logo.svg)

# FreqAI

## Introduction

FreqAI is a software designed to automate a variety of tasks associated with training a predictive machine learning model to generate market forecasts given a set of input signals. In general, FreqAI aims to be a sandbox for easily deploying robust machine learning libraries on real-time data ([details](#freqai-position-in-open-source-machine-learning-landscape)).

!!! Note
    FreqAI is, and always will be, a not-for-profit, open-source project. FreqAI does *not* have a crypto token, FreqAI does *not* sell signals, and FreqAI does not have a domain besides the present [freqtrade documentation](https://www.freqtrade.io/en/latest/freqai/).

Features include:

* **Self-adaptive retraining** - Retrain models during [live deployments](freqai-running.md#live-deployments) to self-adapt to the market in a supervised manner
* **Rapid feature engineering** - Create large rich [feature sets](freqai-feature-engineering.md#feature-engineering) (10k+ features) based on simple user-created strategies
* **High performance** - Threading allows for adaptive model retraining on a separate thread (or on GPU if available) from model inferencing (prediction) and bot trade operations. Newest models and data are kept in RAM for rapid inferencing
* **Realistic backtesting** - Emulate self-adaptive training on historic data with a [backtesting module](freqai-running.md#backtesting) that automates retraining
* **Extensibility** - The generalized and robust architecture allows for incorporating any [machine learning library/method](freqai-configuration.md#using-different-prediction-models) available in Python. Eight examples are currently available, including classifiers, regressors, and a convolutional neural network
* **Smart outlier removal** - Remove outliers from training and prediction data sets using a variety of [outlier detection techniques](freqai-feature-engineering.md#outlier-detection)
* **Crash resilience** - Store trained models to disk to make reloading from a crash fast and easy, and [purge obsolete files](freqai-running.md#purging-old-model-data) for sustained dry/live runs
* **Automatic data normalization** - [Normalize the data](freqai-feature-engineering.md#building-the-data-pipeline) in a smart and statistically safe way
* **Automatic data download** - Compute timeranges for data downloads and update historic data (in live deployments)
* **Cleaning of incoming data** - Handle NaNs safely before training and model inferencing
* **Dimensionality reduction** - Reduce the size of the training data via [Principal Component Analysis](freqai-feature-engineering.md#data-dimensionality-reduction-with-principal-component-analysis)
* **Deploying bot fleets** - Set one bot to train models while a fleet of [consumers](producer-consumer.md) use signals.

## Quick start

The easiest way to quickly test FreqAI is to run it in dry mode with the following command:

```bash
freqtrade trade --config config_examples/config_freqai.example.json --strategy FreqaiExampleStrategy --freqaimodel LightGBMRegressor --strategy-path freqtrade/templates
```

You will see the boot-up process of automatic data downloading, followed by simultaneous training and trading. 

!!! danger "Not for production"
    The example strategy provided with the Freqtrade source code is designed for showcasing/testing a wide variety of FreqAI features. It is also designed to run on small computers so that it can be used as a benchmark between developers and users. It is *not* designed to be run in production.

An example strategy, prediction model, and config to use as a starting points can be found in
`freqtrade/templates/FreqaiExampleStrategy.py`, `freqtrade/freqai/prediction_models/LightGBMRegressor.py`, and
`config_examples/config_freqai.example.json`, respectively.

## General approach

You provide FreqAI with a set of custom *base indicators* (the same way as in a [typical Freqtrade strategy](strategy-customization.md)) as well as target values (*labels*). For each pair in the whitelist, FreqAI trains a model to predict the target values based on the input of custom indicators. The models are then consistently retrained, with a predetermined frequency, to adapt to market conditions. FreqAI offers the ability to both backtest strategies (emulating reality with periodic retraining on historic data) and deploy dry/live runs. In dry/live conditions, FreqAI can be set to constant retraining in a background thread to keep models as up to date as possible.

An overview of the algorithm, explaining the data processing pipeline and model usage, is shown below.

![freqai-algo](assets/freqai_algo.jpg)

### Important machine learning vocabulary

**Features** - the parameters, based on historic data, on which a model is trained. All features for a single candle are stored as a vector. In FreqAI, you build a feature data set from anything you can construct in the strategy.

**Labels** - the target values that the model is trained toward. Each feature vector is associated with a single label that is defined by you within the strategy. These labels intentionally look into the future and are what you are training the model to be able to predict.

**Training** - the process of "teaching" the model to match the feature sets to the associated labels. Different types of models "learn" in different ways which means that one might be better than another for a specific application. More information about the different models that are already implemented in FreqAI can be found [here](freqai-configuration.md#using-different-prediction-models).

**Train data** - a subset of the feature data set that is fed to the model during training to "teach" the model how to predict the targets. This data directly influences weight connections in the model.

**Test data** - a subset of the feature data set that is used to evaluate the performance of the model after training. This data does not influence nodal weights within the model.

**Inferencing** - the process of feeding a trained model new unseen data on which it will make a prediction. 

## Install prerequisites

The normal Freqtrade install process will ask if you wish to install FreqAI dependencies. You should reply "yes" to this question if you wish to use FreqAI. If you did not reply yes, you can manually install these dependencies after the install with:

``` bash
pip install -r requirements-freqai.txt
```

!!! Note
    Catboost will not be installed on low-powered arm devices (raspberry), since it does not provide wheels for this platform.

### Usage with docker

If you are using docker, a dedicated tag with FreqAI dependencies is available as `:freqai`. As such - you can replace the image line in your docker compose file with `image: freqtradeorg/freqtrade:stable_freqai`. This image contains the regular FreqAI dependencies. Similar to native installs, Catboost will not be available on ARM based devices. If you would like to use PyTorch or Reinforcement learning, you should use the torch or RL tags, `image: freqtradeorg/freqtrade:stable_freqaitorch`, `image: freqtradeorg/freqtrade:stable_freqairl`.

!!! note "docker-compose-freqai.yml"
   We do provide an explicit docker-compose file for this in `docker/docker-compose-freqai.yml` - which can be used via `docker compose -f docker/docker-compose-freqai.yml run ...` - or can be copied to replace the original docker file. This docker-compose file also contains a (disabled) section to enable GPU resources within docker containers. This obviously assumes the system has GPU resources available.

### FreqAI position in open-source machine learning landscape

Forecasting chaotic time-series based systems, such as equity/cryptocurrency markets, requires a broad set of tools geared toward testing a wide range of hypotheses. Fortunately, a recent maturation of robust machine learning libraries (e.g. `scikit-learn`) has opened up a wide range of research possibilities. Scientists from a diverse range of fields can now easily prototype their studies on an abundance of established machine learning algorithms. Similarly, these user-friendly libraries enable "citizen scientists" to use their basic Python skills for data exploration. However, leveraging these machine learning libraries on historical and live chaotic data sources can be logistically difficult and expensive. Additionally, robust data collection, storage, and handling presents a disparate challenge. [`FreqAI`](#freqai) aims to provide a generalized and extensible open-sourced framework geared toward live deployments of adaptive modeling for market forecasting. The `FreqAI` framework is effectively a sandbox for the rich world of open-source machine learning libraries. Inside the `FreqAI` sandbox, users find they can combine a wide variety of third-party libraries to test creative hypotheses on a free live 24/7 chaotic data source - cryptocurrency exchange data. 

### Citing FreqAI

FreqAI is [published in the Journal of Open Source Software](https://joss.theoj.org/papers/10.21105/joss.04864). If you find FreqAI useful in your research, please use the following citation:

```bibtex
@article{Caulk2022, 
    doi = {10.21105/joss.04864},
    url = {https://doi.org/10.21105/joss.04864},
    year = {2022}, publisher = {The Open Journal},
    volume = {7}, number = {80}, pages = {4864},
    author = {Robert A. Caulk and Elin Törnquist and Matthias Voppichler and Andrew R. Lawless and Ryan McMullan and Wagner Costa Santos and Timothy C. Pogue and Johan van der Vlugt and Stefan P. Gehring and Pascal Schmidt},
    title = {FreqAI: generalizing adaptive modeling for chaotic time-series market forecasts},
    journal = {Journal of Open Source Software} } 
```

## Common pitfalls

FreqAI cannot be combined with dynamic `VolumePairlists` (or any pairlist filter that adds and removes pairs dynamically).
This is for performance reasons - FreqAI relies on making quick predictions/retrains. To do this effectively,
it needs to download all the training data at the beginning of a dry/live instance. FreqAI stores and appends
new candles automatically for future retrains. This means that if new pairs arrive later in the dry run due to a volume pairlist, it will not have the data ready. However, FreqAI does work with the `ShufflePairlist` or a `VolumePairlist` which keeps the total pairlist constant (but reorders the pairs according to volume).

## Additional learning materials

Here we compile some external materials that provide deeper looks into various components of FreqAI:

- [Real-time head-to-head: Adaptive modeling of financial market data using XGBoost and CatBoost](https://emergentmethods.medium.com/real-time-head-to-head-adaptive-modeling-of-financial-market-data-using-xgboost-and-catboost-995a115a7495)
- [FreqAI - from price to prediction](https://emergentmethods.medium.com/freqai-from-price-to-prediction-6fadac18b665)


## Support

You can find support for FreqAI in a variety of places, including the [Freqtrade discord](https://discord.gg/Jd8JYeWHc4), the dedicated [FreqAI discord](https://discord.gg/7AMWACmbjT), and in [github issues](https://github.com/freqtrade/freqtrade/issues).

## Credits

FreqAI is developed by a group of individuals who all contribute specific skillsets to the project.

Conception and software development:
Robert Caulk @robcaulk

Theoretical brainstorming and data analysis:
Elin Törnquist @th0rntwig

Code review and software architecture brainstorming:
@xmatthias

Software development:
Wagner Costa @wagnercosta
Emre Suzen @aemr3
Timothy Pogue @wizrds

Beta testing and bug reporting:
Stefan Gehring @bloodhunter4rc, @longyu, Andrew Lawless @paranoidandy, Pascal Schmidt @smidelis, Ryan McMullan @smarmau, Juha Nykänen @suikula, Johan van der Vlugt @jooopiert, Richárd Józsa @richardjosza

---

## File: freqai-parameter-table.md

# Parameter table

The table below will list all configuration parameters available for FreqAI. Some of the parameters are exemplified in `config_examples/config_freqai.example.json`.

Mandatory parameters are marked as **Required** and have to be set in one of the suggested ways.

### General configuration parameters

|  Parameter | Description |
|------------|-------------|
|  |  **General configuration parameters within the `config.freqai` tree**
| `freqai` | **Required.** <br> The parent dictionary containing all the parameters for controlling FreqAI. <br> **Datatype:** Dictionary.
| `train_period_days` | **Required.** <br> Number of days to use for the training data (width of the sliding window). <br> **Datatype:** Positive integer.
| `backtest_period_days` | **Required.** <br> Number of days to inference from the trained model before sliding the `train_period_days` window defined above, and retraining the model during backtesting (more info [here](freqai-running.md#backtesting)). This can be fractional days, but beware that the provided `timerange` will be divided by this number to yield the number of trainings necessary to complete the backtest. <br> **Datatype:** Float.
| `identifier` | **Required.** <br> A unique ID for the current model. If models are saved to disk, the `identifier` allows for reloading specific pre-trained models/data. <br> **Datatype:** String.
| `live_retrain_hours` | Frequency of retraining during dry/live runs. <br> **Datatype:** Float > 0. <br> Default: `0` (models retrain as often as possible).
| `expiration_hours` | Avoid making predictions if a model is more than `expiration_hours` old. <br> **Datatype:** Positive integer. <br> Default: `0` (models never expire).
| `purge_old_models` | Number of models to keep on disk (not relevant to backtesting). Default is 2, which means that dry/live runs will keep the latest 2 models on disk. Setting to 0 keeps all models. This parameter also accepts a boolean to maintain backwards compatibility. <br> **Datatype:** Integer. <br> Default: `2`.
| `save_backtest_models` | Save models to disk when running backtesting. Backtesting operates most efficiently by saving the prediction data and reusing them directly for subsequent runs (when you wish to tune entry/exit parameters). Saving backtesting models to disk also allows to use the same model files for starting a dry/live instance with the same model `identifier`. <br> **Datatype:** Boolean. <br> Default: `False` (no models are saved).
| `fit_live_predictions_candles` | Number of historical candles to use for computing target (label) statistics from prediction data, instead of from the training dataset (more information can be found [here](freqai-configuration.md#creating-a-dynamic-target-threshold)). <br> **Datatype:** Positive integer.
| `continual_learning` | Use the final state of the most recently trained model as starting point for the new model, allowing for incremental learning (more information can be found [here](freqai-running.md#continual-learning)). Beware that this is currently a naive approach to incremental learning, and it has a high probability of overfitting/getting stuck in local minima while the market moves away from your model. We have the connections here primarily for experimental purposes and so that it is ready for more mature approaches to continual learning in chaotic systems like the crypto market. <br> **Datatype:** Boolean. <br> Default: `False`.
| `write_metrics_to_disk` | Collect train timings, inference timings and cpu usage in json file. <br> **Datatype:** Boolean. <br> Default: `False`
| `data_kitchen_thread_count` | <br> Designate the number of threads you want to use for data processing (outlier methods, normalization, etc.). This has no impact on the number of threads used for training. If user does not set it (default), FreqAI will use max number of threads - 2 (leaving 1 physical core available for Freqtrade bot and FreqUI) <br> **Datatype:** Positive integer.
| `activate_tensorboard` | <br> Indicate whether or not to activate tensorboard for the tensorboard enabled modules (currently Reinforcment Learning, XGBoost, Catboost, and PyTorch). Tensorboard needs Torch installed, which means you will need the torch/RL docker image or you need to answer "yes" to the install question about whether or not you wish to install Torch. <br> **Datatype:** Boolean. <br> Default: `True`.
| `wait_for_training_iteration_on_reload` | <br> When using /reload or ctrl-c, wait for the current training iteration to finish before completing graceful shutdown. If set to `False`, FreqAI will break the current training iteration, allowing you to shutdown gracefully more quickly, but you will lose your current training iteration. <br> **Datatype:** Boolean. <br> Default: `True`.

### Feature parameters

|  Parameter | Description |
|------------|-------------|
|  |  **Feature parameters within the `freqai.feature_parameters` sub dictionary**
| `feature_parameters` | A dictionary containing the parameters used to engineer the feature set. Details and examples are shown [here](freqai-feature-engineering.md). <br> **Datatype:** Dictionary.
| `include_timeframes` | A list of timeframes that all indicators in `feature_engineering_expand_*()` will be created for. The list is added as features to the base indicators dataset. <br> **Datatype:** List of timeframes (strings).
| `include_corr_pairlist` | A list of correlated coins that FreqAI will add as additional features to all `pair_whitelist` coins. All indicators set in `feature_engineering_expand_*()` during feature engineering (see details [here](freqai-feature-engineering.md)) will be created for each correlated coin. The correlated coins features are added to the base indicators dataset. <br> **Datatype:** List of assets (strings).
| `label_period_candles` | Number of candles into the future that the labels are created for. This can be used in `set_freqai_targets()` (see `templates/FreqaiExampleStrategy.py` for detailed usage). This parameter is not necessarily required, you can create custom labels and choose whether to make use of this parameter or not. Please see `templates/FreqaiExampleStrategy.py` to see the example usage. <br> **Datatype:** Positive integer.
| `include_shifted_candles` | Add features from previous candles to subsequent candles with the intent of adding historical information. If used, FreqAI will duplicate and shift all features from the `include_shifted_candles` previous candles so that the information is available for the subsequent candle. <br> **Datatype:** Positive integer.
| `weight_factor` | Weight training data points according to their recency (see details [here](freqai-feature-engineering.md#weighting-features-for-temporal-importance)). <br> **Datatype:** Positive float (typically < 1).
| `indicator_max_period_candles` | **No longer used (#7325)**. Replaced by `startup_candle_count` which is set in the [strategy](freqai-configuration.md#building-a-freqai-strategy). `startup_candle_count` is timeframe independent and defines the maximum *period* used in `feature_engineering_*()` for indicator creation. FreqAI uses this parameter together with the maximum timeframe in `include_time_frames` to calculate how many data points to download such that the first data point does not include a NaN. <br> **Datatype:** Positive integer.
| `indicator_periods_candles` | Time periods to calculate indicators for. The indicators are added to the base indicator dataset. <br> **Datatype:** List of positive integers.
| `principal_component_analysis` | Automatically reduce the dimensionality of the data set using Principal Component Analysis. See details about how it works [here](freqai-feature-engineering.md#data-dimensionality-reduction-with-principal-component-analysis) <br> **Datatype:** Boolean. <br> Default: `False`.
| `plot_feature_importances` | Create a feature importance plot for each model for the top/bottom `plot_feature_importances` number of features. Plot is stored in `user_data/models/<identifier>/sub-train-<COIN>_<timestamp>.html`. <br> **Datatype:** Integer. <br> Default: `0`.
| `DI_threshold` | Activates the use of the Dissimilarity Index for outlier detection when set to > 0. See details about how it works [here](freqai-feature-engineering.md#identifying-outliers-with-the-dissimilarity-index-di). <br> **Datatype:** Positive float (typically < 1).
| `use_SVM_to_remove_outliers` | Train a support vector machine to detect and remove outliers from the training dataset, as well as from incoming data points. See details about how it works [here](freqai-feature-engineering.md#identifying-outliers-using-a-support-vector-machine-svm). <br> **Datatype:** Boolean.
| `svm_params` | All parameters available in Sklearn's `SGDOneClassSVM()`. See details about some select parameters [here](freqai-feature-engineering.md#identifying-outliers-using-a-support-vector-machine-svm). <br> **Datatype:** Dictionary.
| `use_DBSCAN_to_remove_outliers` | Cluster data using the DBSCAN algorithm to identify and remove outliers from training and prediction data. See details about how it works [here](freqai-feature-engineering.md#identifying-outliers-with-dbscan). <br> **Datatype:** Boolean. 
| `noise_standard_deviation` | If set, FreqAI adds noise to the training features with the aim of preventing overfitting. FreqAI generates random deviates from a gaussian distribution with a standard deviation of `noise_standard_deviation` and adds them to all data points. `noise_standard_deviation` should be kept relative to the normalized space, i.e., between -1 and 1. In other words, since data in FreqAI is always normalized to be between -1 and 1, `noise_standard_deviation: 0.05` would result in 32% of the data being randomly increased/decreased by more than 2.5% (i.e., the percent of data falling within the first standard deviation). <br> **Datatype:** Integer. <br> Default: `0`.
| `outlier_protection_percentage` | Enable to prevent outlier detection methods from discarding too much data. If more than `outlier_protection_percentage` % of points are detected as outliers by the SVM or DBSCAN, FreqAI will log a warning message and ignore outlier detection, i.e., the original dataset will be kept intact. If the outlier protection is triggered, no predictions will be made based on the training dataset. <br> **Datatype:** Float. <br> Default: `30`.
| `reverse_train_test_order` | Split the feature dataset (see below) and use the latest data split for training and test on historical split of the data. This allows the model to be trained up to the most recent data point, while avoiding overfitting. However, you should be careful to understand the unorthodox nature of this parameter before employing it. <br> **Datatype:** Boolean. <br> Default: `False` (no reversal).
| `shuffle_after_split` | Split the data into train and test sets, and then shuffle both sets individually. <br> **Datatype:** Boolean. <br> Default: `False`.
| `buffer_train_data_candles` | Cut `buffer_train_data_candles` off the beginning and end of the training data *after* the indicators were populated. The main example use is when predicting maxima and minima, the argrelextrema function  cannot know the maxima/minima at the edges of the timerange. To improve model accuracy, it is best to compute argrelextrema on the full timerange and then use this function to cut off the edges (buffer) by the kernel. In another case, if the targets are set to a shifted price movement, this buffer is unnecessary because the shifted candles at the end of the timerange will be NaN and FreqAI will automatically cut those off of the training dataset.<br> **Datatype:** Integer. <br> Default: `0`.

### Data split parameters

|  Parameter | Description |
|------------|-------------|
|  |  **Data split parameters within the `freqai.data_split_parameters` sub dictionary**
| `data_split_parameters` | Include any additional parameters available from scikit-learn `test_train_split()`, which are shown [here](https://scikit-learn.org/stable/modules/generated/sklearn.model_selection.train_test_split.html) (external website). <br> **Datatype:** Dictionary.
| `test_size` | The fraction of data that should be used for testing instead of training. <br> **Datatype:** Positive float < 1.
| `shuffle` | Shuffle the training data points during training. Typically, to not remove the chronological order of data in time-series forecasting, this is set to `False`. <br> **Datatype:** Boolean. <br> Default: `False`.

### Model training parameters

|  Parameter | Description |
|------------|-------------|
|  |  **Model training parameters within the `freqai.model_training_parameters` sub dictionary**
| `model_training_parameters` | A flexible dictionary that includes all parameters available by the selected model library. For example, if you use `LightGBMRegressor`, this dictionary can contain any parameter available by the `LightGBMRegressor` [here](https://lightgbm.readthedocs.io/en/latest/pythonapi/lightgbm.LGBMRegressor.html) (external website). If you select a different model, this dictionary can contain any parameter from that model. A list of the currently available models can be found [here](freqai-configuration.md#using-different-prediction-models).  <br> **Datatype:** Dictionary.
| `n_estimators` | The number of boosted trees to fit in the training of the model. <br> **Datatype:** Integer.
| `learning_rate` | Boosting learning rate during training of the model. <br> **Datatype:** Float.
| `n_jobs`, `thread_count`, `task_type` | Set the number of threads for parallel processing and the `task_type` (`gpu` or `cpu`). Different model libraries use different parameter names. <br> **Datatype:** Float.

### Reinforcement Learning parameters

|  Parameter | Description |
|------------|-------------|
|  |  **Reinforcement Learning Parameters within the `freqai.rl_config` sub dictionary**
| `rl_config` | A dictionary containing the control parameters for a Reinforcement Learning model. <br> **Datatype:** Dictionary.
| `train_cycles` | Training time steps will be set based on the `train_cycles * number of training data points. <br> **Datatype:** Integer.
| `max_trade_duration_candles`| Guides the agent training to keep trades below desired length. Example usage shown in `prediction_models/ReinforcementLearner.py` within the customizable `calculate_reward()` function. <br> **Datatype:** int.
| `model_type` | Model string from stable_baselines3 or SBcontrib. Available strings include: `'TRPO', 'ARS', 'RecurrentPPO', 'MaskablePPO', 'PPO', 'A2C', 'DQN'`. User should ensure that `model_training_parameters` match those available to the corresponding stable_baselines3 model by visiting their documentation. [PPO doc](https://stable-baselines3.readthedocs.io/en/master/modules/ppo.html) (external website) <br> **Datatype:** string.
| `policy_type` | One of the available policy types from stable_baselines3 <br> **Datatype:** string.
| `max_training_drawdown_pct` | The maximum drawdown that the agent is allowed to experience during training. <br> **Datatype:** float. <br> Default: 0.8
| `cpu_count` | Number of threads/cpus to dedicate to the Reinforcement Learning training process (depending on if `ReinforcementLearning_multiproc` is selected or not). Recommended to leave this untouched, by default, this value is set to the total number of physical cores minus 1. <br> **Datatype:** int. 
| `model_reward_parameters` | Parameters used inside the customizable `calculate_reward()` function in `ReinforcementLearner.py` <br> **Datatype:** int.
| `add_state_info` | Tell FreqAI to include state information in the feature set for training and inferencing. The current state variables include trade duration, current profit, trade position. This is only available in dry/live runs, and is automatically switched to false for backtesting. <br> **Datatype:** bool. <br> Default: `False`.
| `net_arch` | Network architecture which is well described in [`stable_baselines3` doc](https://stable-baselines3.readthedocs.io/en/master/guide/custom_policy.html#examples). In summary: `[<shared layers>, dict(vf=[<non-shared value network layers>], pi=[<non-shared policy network layers>])]`. By default this is set to `[128, 128]`, which defines 2 shared hidden layers with 128 units each.
| `randomize_starting_position` | Randomize the starting point of each episode to avoid overfitting. <br> **Datatype:** bool. <br> Default: `False`.
| `drop_ohlc_from_features` | Do not include the normalized ohlc data in the feature set passed to the agent during training (ohlc will still be used for driving the environment in all cases) <br> **Datatype:** Boolean. <br> **Default:** `False`
| `progress_bar` | Display a progress bar with the current progress, elapsed time and estimated remaining time. <br> **Datatype:** Boolean. <br> Default: `False`.

### PyTorch parameters

#### general

|  Parameter | Description |
|------------|-------------|
|  |  **Model training parameters within the `freqai.model_training_parameters` sub dictionary**
| `learning_rate` | Learning rate to be passed to the optimizer. <br> **Datatype:** float. <br> Default: `3e-4`.
| `model_kwargs` | Parameters to be passed to the model class. <br> **Datatype:** dict. <br> Default: `{}`.
| `trainer_kwargs` | Parameters to be passed to the trainer class. <br> **Datatype:** dict. <br> Default: `{}`.

#### trainer_kwargs

| Parameter    | Description |
|--------------|-------------|
|              |  **Model training parameters within the `freqai.model_training_parameters.model_kwargs` sub dictionary**
| `n_epochs`   | The `n_epochs` parameter is a crucial setting in the PyTorch training loop that determines the number of times the entire training dataset will be used to update the model's parameters. An epoch represents one full pass through the entire training dataset. Overrides `n_steps`. Either `n_epochs` or `n_steps` must be set. <br><br> **Datatype:** int. optional. <br> Default: `10`.
| `n_steps`    | An alternative way of setting `n_epochs` -  the number of training iterations to run. Iteration here refer to the number of times we call `optimizer.step()`. Ignored if `n_epochs` is set. A simplified version of the function: <br><br> n_epochs = n_steps / (n_obs / batch_size) <br><br> The motivation here is that `n_steps` is easier to optimize and keep stable across different n_obs - the number of data points.  <br> <br> **Datatype:** int. optional. <br> Default: `None`.
| `batch_size` | The size of the batches to use during training. <br><br> **Datatype:** int. <br> Default: `64`.


### Additional parameters

|  Parameter | Description |
|------------|-------------|
|  |  **Extraneous parameters**
| `freqai.keras` | If the selected model makes use of Keras (typical for TensorFlow-based prediction models), this flag needs to be activated so that the model save/loading follows Keras standards. <br> **Datatype:** Boolean. <br> Default: `False`.
| `freqai.conv_width` | The width of a neural network input tensor. This replaces the need for shifting candles (`include_shifted_candles`) by feeding in historical data points as the second dimension of the tensor. Technically, this parameter can also be used for regressors, but it only adds computational overhead and does not change the model training/prediction. <br> **Datatype:** Integer. <br> Default: `2`.
| `freqai.reduce_df_footprint` | Recast all numeric columns to float32/int32, with the objective of reducing ram/disk usage and decreasing train/inference timing. This parameter is set in the main level of the Freqtrade configuration file (not inside FreqAI). <br> **Datatype:** Boolean. <br> Default: `False`.

---

## File: freqai-reinforcement-learning.md

# Reinforcement Learning

!!! Note "Installation size"
    Reinforcement learning dependencies include large packages such as `torch`, which should be explicitly requested during `./setup.sh -i` by answering "y" to the question "Do you also want dependencies for freqai-rl (~700mb additional space required) [y/N]?".
    Users who prefer docker should ensure they use the docker image appended with `_freqairl`.

## Background and terminology

### What is RL and why does FreqAI need it?

Reinforcement learning involves two important components, the *agent* and the training *environment*. During agent training, the agent moves through historical data candle by candle, always making 1 of a set of actions: Long entry, long exit, short entry, short exit, neutral). During this training process, the environment tracks the performance of these actions and rewards the agent according to a custom user made `calculate_reward()` (here we offer a default reward for users to build on if they wish [details here](#creating-a-custom-reward-function)). The reward is used to train weights in a neural network.

A second important component of the FreqAI RL implementation is the use of *state* information. State information is fed into the network at each step, including current profit, current position, and current trade duration. These are used to train the agent in the training environment, and to reinforce the agent in dry/live (this functionality is not available in backtesting). *FreqAI + Freqtrade is a perfect match for this reinforcing mechanism since this information is readily available in live deployments.*

Reinforcement learning is a natural progression for FreqAI, since it adds a new layer of adaptivity and market reactivity that Classifiers and Regressors cannot match. However, Classifiers and Regressors have strengths that RL does not have such as robust predictions. Improperly trained RL agents may find "cheats" and "tricks" to maximize reward without actually winning any trades. For this reason, RL is more complex and demands a higher level of understanding than typical Classifiers and Regressors.

### The RL interface

With the current framework, we aim to expose the training environment via the common "prediction model" file, which is a user inherited `BaseReinforcementLearner` object (e.g. `freqai/prediction_models/ReinforcementLearner`). Inside this user class, the RL environment is available and customized via `MyRLEnv` as [shown below](#creating-a-custom-reward-function).

We envision the majority of users focusing their effort on creative design of the `calculate_reward()` function [details here](#creating-a-custom-reward-function), while leaving the rest of the environment untouched. Other users may not touch the environment at all, and they will only play with the configuration settings and the powerful feature engineering that already exists in FreqAI. Meanwhile, we enable advanced users to create their own model classes entirely.

The framework is built on stable_baselines3 (torch) and OpenAI gym for the base environment class. But generally speaking, the model class is well isolated. Thus, the addition of competing libraries can be easily integrated into the existing framework. For the environment, it is inheriting from `gym.Env` which means that it is necessary to write an entirely new environment in order to switch to a different library.

### Important considerations

As explained above, the agent is "trained" in an artificial trading "environment". In our case, that environment may seem quite similar to a real Freqtrade backtesting environment, but it is *NOT*. In fact, the RL training environment is much more simplified. It does not incorporate any of the complicated strategy logic, such as callbacks like `custom_exit`, `custom_stoploss`, leverage controls, etc. The RL environment is instead a very "raw" representation of the true market, where the agent has free will to learn the policy (read: stoploss, take profit, etc.) which is enforced by the `calculate_reward()`. Thus, it is important to consider that the agent training environment is not identical to the real world.

## Running Reinforcement Learning

Setting up and running a Reinforcement Learning model is the same as running a Regressor or Classifier. The same two flags, `--freqaimodel` and `--strategy`, must be defined on the command line:

```bash
freqtrade trade --freqaimodel ReinforcementLearner --strategy MyRLStrategy --config config.json
```

where `ReinforcementLearner` will use the templated `ReinforcementLearner` from `freqai/prediction_models/ReinforcementLearner` (or a custom user defined one located in `user_data/freqaimodels`). The strategy, on the other hand, follows the same base [feature engineering](freqai-feature-engineering.md) with `feature_engineering_*` as a typical Regressor. The difference lies in the creation of the targets, Reinforcement Learning doesn't require them. However, FreqAI requires a default (neutral) value to be set in the action column:

```python
    def set_freqai_targets(self, dataframe, **kwargs) -> DataFrame:
        """
        *Only functional with FreqAI enabled strategies*
        Required function to set the targets for the model.
        All targets must be prepended with `&` to be recognized by the FreqAI internals.

        More details about feature engineering available:

        https://www.freqtrade.io/en/latest/freqai-feature-engineering

        :param df: strategy dataframe which will receive the targets
        usage example: dataframe["&-target"] = dataframe["close"].shift(-1) / dataframe["close"]
        """
        # For RL, there are no direct targets to set. This is filler (neutral)
        # until the agent sends an action.
        dataframe["&-action"] = 0
        return dataframe
```

Most of the function remains the same as for typical Regressors, however, the function below shows how the strategy must pass the raw price data to the agent so that it has access to raw OHLCV in the training environment:

```python
    def feature_engineering_standard(self, dataframe: DataFrame, **kwargs) -> DataFrame:
        # The following features are necessary for RL models
        dataframe[f"%-raw_close"] = dataframe["close"]
        dataframe[f"%-raw_open"] = dataframe["open"]
        dataframe[f"%-raw_high"] = dataframe["high"]
        dataframe[f"%-raw_low"] = dataframe["low"]
    return dataframe
```

Finally, there is no explicit "label" to make - instead it is necessary to assign the `&-action` column which will contain the agent's actions when accessed in `populate_entry/exit_trends()`. In the present example, the neutral action to 0. This value should align with the environment used. FreqAI provides two environments, both use 0 as the neutral action.

After users realize there are no labels to set, they will soon understand that the agent is making its "own" entry and exit decisions. This makes strategy construction rather simple. The entry and exit signals come from the agent in the form of an integer - which are used directly to decide entries and exits in the strategy:

```python
    def populate_entry_trend(self, df: DataFrame, metadata: dict) -> DataFrame:

        enter_long_conditions = [df["do_predict"] == 1, df["&-action"] == 1]

        if enter_long_conditions:
            df.loc[
                reduce(lambda x, y: x & y, enter_long_conditions), ["enter_long", "enter_tag"]
            ] = (1, "long")

        enter_short_conditions = [df["do_predict"] == 1, df["&-action"] == 3]

        if enter_short_conditions:
            df.loc[
                reduce(lambda x, y: x & y, enter_short_conditions), ["enter_short", "enter_tag"]
            ] = (1, "short")

        return df

    def populate_exit_trend(self, df: DataFrame, metadata: dict) -> DataFrame:
        exit_long_conditions = [df["do_predict"] == 1, df["&-action"] == 2]
        if exit_long_conditions:
            df.loc[reduce(lambda x, y: x & y, exit_long_conditions), "exit_long"] = 1

        exit_short_conditions = [df["do_predict"] == 1, df["&-action"] == 4]
        if exit_short_conditions:
            df.loc[reduce(lambda x, y: x & y, exit_short_conditions), "exit_short"] = 1

        return df
```

It is important to consider that `&-action` depends on which environment they choose to use. The example above shows 5 actions, where 0 is neutral, 1 is enter long, 2 is exit long, 3 is enter short and 4 is exit short.

## Configuring the Reinforcement Learner

In order to configure the `Reinforcement Learner` the following dictionary must exist in the `freqai` config:

```json
        "rl_config": {
            "train_cycles": 25,
            "add_state_info": true,
            "max_trade_duration_candles": 300,
            "max_training_drawdown_pct": 0.02,
            "cpu_count": 8,
            "model_type": "PPO",
            "policy_type": "MlpPolicy",
            "model_reward_parameters": {
                "rr": 1,
                "profit_aim": 0.025
            }
        }
```

Parameter details can be found [here](freqai-parameter-table.md), but in general the `train_cycles` decides how many times the agent should cycle through the candle data in its artificial environment to train weights in the model. `model_type` is a string which selects one of the available models in [stable_baselines](https://stable-baselines3.readthedocs.io/en/master/)(external link).

!!! Note
    If you would like to experiment with `continual_learning`, then you should set that value to `true` in the main `freqai` configuration dictionary. This will tell the Reinforcement Learning library to continue training new models from the final state of previous models, instead of retraining new models from scratch each time a retrain is initiated.

!!! Note
    Remember that the general `model_training_parameters` dictionary should contain all the model hyperparameter customizations for the particular `model_type`. For example, `PPO` parameters can be found [here](https://stable-baselines3.readthedocs.io/en/master/modules/ppo.html).

## Creating a custom reward function

!!! danger "Not for production"
    Warning!
    The reward function provided with the Freqtrade source code is a showcase of functionality designed to show/test as many possible environment control features as possible. It is also designed to run quickly on small computers. This is a benchmark, it is *not* for live production. Please beware that you will need to create your own custom_reward() function or use a template built by other users outside of the Freqtrade source code.

As you begin to modify the strategy and the prediction model, you will quickly realize some important differences between the Reinforcement Learner and the Regressors/Classifiers. Firstly, the strategy does not set a target value (no labels!). Instead, you set the `calculate_reward()` function inside the `MyRLEnv` class (see below). A default `calculate_reward()` is provided inside `prediction_models/ReinforcementLearner.py` to demonstrate the necessary building blocks for creating rewards, but this is *not* designed for production. Users *must* create their own custom reinforcement learning model class or use a pre-built one from outside the Freqtrade source code and save it to `user_data/freqaimodels`. It is inside the `calculate_reward()` where creative theories about the market can be expressed. For example, you can reward your agent when it makes a winning trade, and penalize the agent when it makes a losing trade. Or perhaps, you wish to reward the agent for entering trades, and penalize the agent for sitting in trades too long. Below we show examples of how these rewards are all calculated:

!!! note "Hint"
    The best reward functions are ones that are continuously differentiable, and well scaled. In other words, adding a single large negative penalty to a rare event is not a good idea, and the neural net will not be able to learn that function. Instead, it is better to add a small negative penalty to a common event. This will help the agent learn faster. Not only this, but you can help improve the continuity of your rewards/penalties by having them scale with severity according to some linear/exponential functions. In other words, you'd slowly scale the penalty as the duration of the trade increases. This is better than a single large penalty occurring at a single point in time.

```python
from freqtrade.freqai.prediction_models.ReinforcementLearner import ReinforcementLearner
from freqtrade.freqai.RL.Base5ActionRLEnv import Actions, Base5ActionRLEnv, Positions


class MyCoolRLModel(ReinforcementLearner):
    """
    User created RL prediction model.

    Save this file to `freqtrade/user_data/freqaimodels`

    then use it with:

    freqtrade trade --freqaimodel MyCoolRLModel --config config.json --strategy SomeCoolStrat

    Here the users can override any of the functions
    available in the `IFreqaiModel` inheritance tree. Most importantly for RL, this
    is where the user overrides `MyRLEnv` (see below), to define custom
    `calculate_reward()` function, or to override any other parts of the environment.

    This class also allows users to override any other part of the IFreqaiModel tree.
    For example, the user can override `def fit()` or `def train()` or `def predict()`
    to take fine-tuned control over these processes.

    Another common override may be `def data_cleaning_predict()` where the user can
    take fine-tuned control over the data handling pipeline.
    """
    class MyRLEnv(Base5ActionRLEnv):
        """
        User made custom environment. This class inherits from BaseEnvironment and gym.Env.
        Users can override any functions from those parent classes. Here is an example
        of a user customized `calculate_reward()` function.

        Warning!
        This is function is a showcase of functionality designed to show as many possible
        environment control features as possible. It is also designed to run quickly
        on small computers. This is a benchmark, it is *not* for live production.
        """
        def calculate_reward(self, action: int) -> float:
            # first, penalize if the action is not valid
            if not self._is_valid(action):
                return -2
            pnl = self.get_unrealized_profit()

            factor = 100

            pair = self.pair.replace(':', '')

            # you can use feature values from dataframe
            # Assumes the shifted RSI indicator has been generated in the strategy.
            rsi_now = self.raw_features[f"%-rsi-period_10_shift-1_{pair}_"
                            f"{self.config['timeframe']}"].iloc[self._current_tick]

            # reward agent for entering trades
            if (action in (Actions.Long_enter.value, Actions.Short_enter.value)
                    and self._position == Positions.Neutral):
                if rsi_now < 40:
                    factor = 40 / rsi_now
                else:
                    factor = 1
                return 25 * factor

            # discourage agent from not entering trades
            if action == Actions.Neutral.value and self._position == Positions.Neutral:
                return -1
            max_trade_duration = self.rl_config.get('max_trade_duration_candles', 300)
            trade_duration = self._current_tick - self._last_trade_tick
            if trade_duration <= max_trade_duration:
                factor *= 1.5
            elif trade_duration > max_trade_duration:
                factor *= 0.5
            # discourage sitting in position
            if self._position in (Positions.Short, Positions.Long) and \
            action == Actions.Neutral.value:
                return -1 * trade_duration / max_trade_duration
            # close long
            if action == Actions.Long_exit.value and self._position == Positions.Long:
                if pnl > self.profit_aim * self.rr:
                    factor *= self.rl_config['model_reward_parameters'].get('win_reward_factor', 2)
                return float(pnl * factor)
            # close short
            if action == Actions.Short_exit.value and self._position == Positions.Short:
                if pnl > self.profit_aim * self.rr:
                    factor *= self.rl_config['model_reward_parameters'].get('win_reward_factor', 2)
                return float(pnl * factor)
            return 0.
```

## Using Tensorboard

Reinforcement Learning models benefit from tracking training metrics. FreqAI has integrated Tensorboard to allow users to track training and evaluation performance across all coins and across all retrainings. Tensorboard is activated via the following command:

```bash
tensorboard --logdir user_data/models/unique-id
```

where `unique-id` is the `identifier` set in the `freqai` configuration file. This command must be run in a separate shell to view the output in the browser at 127.0.0.1:6006 (6006 is the default port used by Tensorboard).

![tensorboard](assets/tensorboard.jpg)

## Custom logging

FreqAI also provides a built in episodic summary logger called `self.tensorboard_log` for adding custom information to the Tensorboard log. By default, this function is already called once per step inside the environment to record the agent actions. All values accumulated for all steps in a single episode are reported at the conclusion of each episode, followed by a full reset of all metrics to 0 in preparation for the subsequent episode.

`self.tensorboard_log` can also be used anywhere inside the environment, for example, it can be added to the `calculate_reward` function to collect more detailed information about how often various parts of the reward were called:

```python
    class MyRLEnv(Base5ActionRLEnv):
        """
        User made custom environment. This class inherits from BaseEnvironment and gym.Env.
        Users can override any functions from those parent classes. Here is an example
        of a user customized `calculate_reward()` function.
        """
        def calculate_reward(self, action: int) -> float:
            if not self._is_valid(action):
                self.tensorboard_log("invalid")
                return -2

```

!!! Note
    The `self.tensorboard_log()` function is designed for tracking incremented objects only i.e. events, actions inside the training environment. If the event of interest is a float, the float can be passed as the second argument e.g. `self.tensorboard_log("float_metric1", 0.23)`. In this case the metric values are not incremented.

## Choosing a base environment

FreqAI provides three base environments, `Base3ActionRLEnvironment`, `Base4ActionEnvironment` and `Base5ActionEnvironment`. As the names imply, the environments are customized for agents that can select from 3, 4 or 5 actions. The `Base3ActionEnvironment` is the simplest, the agent can select from hold, long, or short. This environment can also be used for long-only bots (it automatically follows the `can_short` flag from the strategy), where long is the enter condition and short is the exit condition. Meanwhile, in the `Base4ActionEnvironment`, the agent can enter long, enter short, hold neutral, or exit position. Finally, in the `Base5ActionEnvironment`, the agent has the same actions as Base4, but instead of a single exit action, it separates exit long and exit short. The main changes stemming from the environment selection include:

* the actions available in the `calculate_reward`
* the actions consumed by the user strategy

All of the FreqAI provided environments inherit from an action/position agnostic environment object called the `BaseEnvironment`, which contains all shared logic. The architecture is designed to be easily customized. The simplest customization is the `calculate_reward()` (see details [here](#creating-a-custom-reward-function)). However, the customizations can be further extended into any of the functions inside the environment. You can do this by simply overriding those functions inside your `MyRLEnv` in the prediction model file. Or for more advanced customizations, it is encouraged to create an entirely new environment inherited from `BaseEnvironment`.

!!! Note
    Only the `Base3ActionRLEnv` can do long-only training/trading (set the user strategy attribute `can_short = False`).

---

## File: freqai-running.md

# Running FreqAI

There are two ways to train and deploy an adaptive machine learning model - live deployment and historical backtesting. In both cases, FreqAI runs/simulates periodic retraining of models as shown in the following figure:

![freqai-window](assets/freqai_moving-window.jpg)

## Live deployments

FreqAI can be run dry/live using the following command:

```bash
freqtrade trade --strategy FreqaiExampleStrategy --config config_freqai.example.json --freqaimodel LightGBMRegressor
```

When launched, FreqAI will start training a new model, with a new `identifier`, based on the config settings. Following training, the model will be used to make predictions on incoming candles until a new model is available. New models are typically generated as often as possible, with FreqAI managing an internal queue of the coin pairs to try to keep all models equally up to date. FreqAI will always use the most recently trained model to make predictions on incoming live data. If you do not want FreqAI to retrain new models as often as possible, you can set `live_retrain_hours` to tell FreqAI to wait at least that number of hours before training a new model. Additionally, you can set `expired_hours` to tell FreqAI to avoid making predictions on models that are older than that number of hours.

Trained models are by default saved to disk to allow for reuse during backtesting or after a crash. You can opt to [purge old models](#purging-old-model-data) to save disk space by setting `"purge_old_models": true` in the config.

To start a dry/live run from a saved backtest model (or from a previously crashed dry/live session), you only need to specify the `identifier` of the specific model:

```json
    "freqai": {
        "identifier": "example",
        "live_retrain_hours": 0.5
    }
```

In this case, although FreqAI will initiate with a pre-trained model, it will still check to see how much time has elapsed since the model was trained. If a full `live_retrain_hours` has elapsed since the end of the loaded model, FreqAI will start training a new model.

### Automatic data download

FreqAI automatically downloads the proper amount of data needed to ensure training of a model through the defined `train_period_days` and `startup_candle_count` (see the [parameter table](freqai-parameter-table.md) for detailed descriptions of these parameters). 

### Saving prediction data

All predictions made during the lifetime of a specific `identifier` model are stored in `historic_predictions.pkl` to allow for reloading after a crash or changes made to the config.

### Purging old model data

FreqAI stores new model files after each successful training. These files become obsolete as new models are generated to adapt to new market conditions. If you are planning to leave FreqAI running for extended periods of time with high frequency retraining, you should enable `purge_old_models` in the config:

```json
    "freqai": {
        "purge_old_models": 4,
    }
```

This will automatically purge all models older than the four most recently trained ones to save disk space. Inputing "0" will never purge any models.

## Backtesting

The FreqAI backtesting module can be executed with the following command:

```bash
freqtrade backtesting --strategy FreqaiExampleStrategy --strategy-path freqtrade/templates --config config_examples/config_freqai.example.json --freqaimodel LightGBMRegressor --timerange 20210501-20210701
```

If this command has never been executed with the existing config file, FreqAI will train a new model
for each pair, for each backtesting window within the expanded `--timerange`.

Backtesting mode requires [downloading the necessary data](#downloading-data-to-cover-the-full-backtest-period) before deployment (unlike in dry/live mode where FreqAI handles the data downloading automatically). You should be careful to consider that the time range of the downloaded data is more than the backtesting time range. This is because FreqAI needs data prior to the desired backtesting time range in order to train a model to be ready to make predictions on the first candle of the set backtesting time range. More details on how to calculate the data to download can be found [here](#deciding-the-size-of-the-sliding-training-window-and-backtesting-duration).

!!! Note "Model reuse"
    Once the training is completed, you can execute the backtesting again with the same config file and
    FreqAI will find the trained models and load them instead of spending time training. This is useful
    if you want to tweak (or even hyperopt) buy and sell criteria inside the strategy. If you
    *want* to retrain a new model with the same config file, you should simply change the `identifier`.
    This way, you can return to using any model you wish by simply specifying the `identifier`.

!!! Note
    Backtesting calls `set_freqai_targets()` one time for each backtest window (where the number of windows is the full backtest timerange divided by the `backtest_period_days` parameter). Doing this means that the targets simulate dry/live behavior without look ahead bias. However, the definition of the features in `feature_engineering_*()` is performed once on the entire training timerange. This means that you should be sure that features do not look-ahead into the future.
    More details about look-ahead bias can be found in [Common Mistakes](strategy-customization.md#common-mistakes-when-developing-strategies).

---

### Saving backtesting prediction data

To allow for tweaking your strategy (**not** the features!), FreqAI will automatically save the predictions during backtesting so that they can be reused for future backtests and live runs using the same `identifier` model. This provides a performance enhancement geared towards enabling **high-level hyperopting** of entry/exit criteria.

An additional directory called `backtesting_predictions`, which contains all the predictions stored in `feather` format, will be created in the `unique-id` folder.

To change your **features**, you **must** set a new `identifier` in the config to signal to FreqAI to train new models.

To save the models generated during a particular backtest so that you can start a live deployment from one of them instead of training a new model, you must set `save_backtest_models` to `True` in the config.

!!! Note
    To ensure that the model can be reused, freqAI will call your strategy with a dataframe of length 1. 
    If your strategy requires more data than this to generate the same features, you can't reuse backtest predictions for live deployment and need to update your `identifier` for each new backtest.

### Backtest live collected predictions

FreqAI allow you to reuse live historic predictions through the backtest parameter `--freqai-backtest-live-models`. This can be useful when you want to reuse predictions generated in dry/run for comparison or other study.

The `--timerange` parameter must not be informed, as it will be automatically calculated through the data in the historic predictions file.

### Downloading data to cover the full backtest period

For live/dry deployments, FreqAI will download the necessary data automatically. However, to use backtesting functionality, you need to download the necessary data using `download-data` (details [here](data-download.md#data-downloading)). You need to pay careful attention to understanding how much *additional* data needs to be downloaded to ensure that there is a sufficient amount of training data *before* the start of the backtesting time range. The amount of additional data can be roughly estimated by moving the start date of the time range backwards by `train_period_days` and the `startup_candle_count` (see the [parameter table](freqai-parameter-table.md) for detailed descriptions of these parameters) from the beginning of the desired backtesting time range. 

As an example, to backtest the `--timerange 20210501-20210701` using the [example config](freqai-configuration.md#setting-up-the-configuration-file) which sets `train_period_days` to 30, together with `startup_candle_count: 40` on a maximum `include_timeframes` of 1h, the start date for the downloaded data needs to be `20210501` - 30 days - 40 * 1h / 24 hours = 20210330 (31.7 days earlier than the start of the desired training time range).

### Deciding the size of the sliding training window and backtesting duration

The backtesting time range is defined with the typical `--timerange` parameter in the configuration file. The duration of the sliding training window is set by `train_period_days`, whilst `backtest_period_days` is the sliding backtesting window, both in number of days (`backtest_period_days` can be
a float to indicate sub-daily retraining in live/dry mode). In the presented [example config](freqai-configuration.md#setting-up-the-configuration-file) (found in `config_examples/config_freqai.example.json`), the user is asking FreqAI to use a training period of 30 days and backtest on the subsequent 7 days. After the training of the model, FreqAI will backtest the subsequent 7 days. The "sliding window" then moves one week forward (emulating FreqAI retraining once per week in live mode) and the new model uses the previous 30 days (including the 7 days used for backtesting by the previous model) to train. This is repeated until the end of `--timerange`.  This means that if you set `--timerange 20210501-20210701`, FreqAI will have trained 8 separate models at the end of `--timerange` (because the full range comprises 8 weeks).

!!! Note
    Although fractional `backtest_period_days` is allowed, you should be aware that the `--timerange` is divided by this value to determine the number of models that FreqAI will need to train in order to backtest the full range. For example, by setting a `--timerange` of 10 days, and a `backtest_period_days` of 0.1, FreqAI will need to train 100 models per pair to complete the full backtest. Because of this, a true backtest of FreqAI adaptive training would take a *very* long time. The best way to fully test a model is to run it dry and let it train constantly. In this case, backtesting would take the exact same amount of time as a dry run.

## Defining model expirations

During dry/live mode, FreqAI trains each coin pair sequentially (on separate threads/GPU from the main Freqtrade bot). This means that there is always an age discrepancy between models. If you are training on 50 pairs, and each pair requires 5 minutes to train, the oldest model will be over 4 hours old. This may be undesirable if the characteristic time scale (the trade duration target) for a strategy is less than 4 hours. You can decide to only make trade entries if the model is less than a certain number of hours old by setting the `expiration_hours` in the config file:

```json
    "freqai": {
        "expiration_hours": 0.5,
    }
```

In the presented example config, the user will only allow predictions on models that are less than 1/2 hours old.

## Controlling the model learning process

Model training parameters are unique to the selected machine learning library. FreqAI allows you to set any parameter for any library using the `model_training_parameters` dictionary in the config. The example config (found in `config_examples/config_freqai.example.json`) shows some of the example parameters associated with `Catboost` and `LightGBM`, but you can add any parameters available in those libraries or any other machine learning library you choose to implement.

Data split parameters are defined in `data_split_parameters` which can be any parameters associated with scikit-learn's `train_test_split()` function. `train_test_split()` has a parameters called `shuffle` which allows to shuffle the data or keep it unshuffled. This is particularly useful to avoid biasing training with temporally auto-correlated data. More details about these parameters can be found the [scikit-learn website](https://scikit-learn.org/stable/modules/generated/sklearn.model_selection.train_test_split.html) (external website).

The FreqAI specific parameter `label_period_candles` defines the offset (number of candles into the future) used for the `labels`. In the presented [example config](freqai-configuration.md#setting-up-the-configuration-file), the user is asking for `labels` that are 24 candles in the future.

## Continual learning

You can choose to adopt a continual learning scheme by setting `"continual_learning": true` in the config. By enabling `continual_learning`, after training an initial model from scratch, subsequent trainings will start from the final model state of the preceding training. This gives the new model a "memory" of the previous state. By default, this is set to `False` which means that all new models are trained from scratch, without input from previous models.

???+ danger "Continual learning enforces a constant parameter space"
    Since `continual_learning` means that the model parameter space *cannot* change between trainings, `principal_component_analysis` is automatically disabled when `continual_learning` is enabled. Hint: PCA changes the parameter space and the number of features, learn more about PCA [here](freqai-feature-engineering.md#data-dimensionality-reduction-with-principal-component-analysis).

???+ danger "Experimental functionality"
    Beware that this is currently a naive approach to incremental learning, and it has a high probability of overfitting/getting stuck in local minima while the market moves away from your model. We have the mechanics available in FreqAI primarily for experimental purposes and so that it is ready for more mature approaches to continual learning in chaotic systems like the crypto market.

## Hyperopt

You can hyperopt using the same command as for [typical Freqtrade hyperopt](hyperopt.md):

```bash
freqtrade hyperopt --hyperopt-loss SharpeHyperOptLoss --strategy FreqaiExampleStrategy --freqaimodel LightGBMRegressor --strategy-path freqtrade/templates --config config_examples/config_freqai.example.json --timerange 20220428-20220507
```

`hyperopt` requires you to have the data pre-downloaded in the same fashion as if you were doing [backtesting](#backtesting). In addition, you must consider some restrictions when trying to hyperopt FreqAI strategies:

- The `--analyze-per-epoch` hyperopt parameter is not compatible with FreqAI.
- It's not possible to hyperopt indicators in the `feature_engineering_*()` and `set_freqai_targets()` functions. This means that you cannot optimize model parameters using hyperopt. Apart from this exception, it is possible to optimize all other [spaces](hyperopt.md#running-hyperopt-with-smaller-search-space).
- The backtesting instructions also apply to hyperopt.

The best method for combining hyperopt and FreqAI is to focus on hyperopting entry/exit thresholds/criteria. You need to focus on hyperopting parameters that are not used in your features. For example, you should not try to hyperopt rolling window lengths in the feature creation, or any part of the FreqAI config which changes predictions. In order to efficiently hyperopt the FreqAI strategy, FreqAI stores predictions as dataframes and reuses them. Hence the requirement to hyperopt entry/exit thresholds/criteria only.

A good example of a hyperoptable parameter in FreqAI is a threshold for the [Dissimilarity Index (DI)](freqai-feature-engineering.md#identifying-outliers-with-the-dissimilarity-index-di) `DI_values` beyond which we consider data points as outliers:

```python
di_max = IntParameter(low=1, high=20, default=10, space='buy', optimize=True, load=True)
dataframe['outlier'] = np.where(dataframe['DI_values'] > self.di_max.value/10, 1, 0)
```

This specific hyperopt would help you understand the appropriate `DI_values` for your particular parameter space.

## Using Tensorboard

!!! note "Availability"
    FreqAI includes tensorboard for a variety of models, including XGBoost, all PyTorch models, Reinforcement Learning, and Catboost. If you would like to see Tensorboard integrated into another model type, please open an issue on the [Freqtrade GitHub](https://github.com/freqtrade/freqtrade/issues)

!!! danger "Requirements"
    Tensorboard logging requires the FreqAI torch installation/docker image.


The easiest way to use tensorboard is to ensure `freqai.activate_tensorboard` is set to `True` (default setting) in your configuration file, run FreqAI, then open a separate shell and run:

```bash
cd freqtrade
tensorboard --logdir user_data/models/unique-id
```

where `unique-id` is the `identifier` set in the `freqai` configuration file. This command must be run in a separate shell if you wish to view the output in your browser at 127.0.0.1:6060 (6060 is the default port used by Tensorboard).

![tensorboard](assets/tensorboard.jpg)


!!! note "Deactivate for improved performance"
    Tensorboard logging can slow down training and should be deactivated for production use.

---

## File: freq-ui.md

# FreqUI

Freqtrade provides a builtin webserver, which can serve [FreqUI](https://github.com/freqtrade/frequi), the freqtrade frontend.

By default, the UI is automatically installed as part of the installation (script, docker).
freqUI can also be manually installed by using the `freqtrade install-ui` command.
This same command can also be used to update freqUI to new new releases.

Once the bot is started in trade / dry-run mode (with `freqtrade trade`) - the UI will be available under the configured API port (by default `http://127.0.0.1:8080`).

??? Note "Looking to contribute to freqUI?"
    Developers should not use this method, but instead clone the corresponding use the method described in the [freqUI repository](https://github.com/freqtrade/frequi) to get the source-code of freqUI. A working installation of node will be required to build the frontend.

!!! tip "freqUI is not required to run freqtrade"
    freqUI is an optional component of freqtrade, and is not required to run the bot.
    It is a frontend that can be used to monitor the bot and to interact with it - but freqtrade itself will work perfectly fine without it.

## Configuration

FreqUI does not have it's own configuration file - but assumes a working setup for the [rest-api](rest-api.md) is available.
Please refer to the corresponding documentation page to get setup with freqUI

## UI

FreqUI is a modern, responsive web application that can be used to monitor and interact with your bot.

FreqUI provides a light, as well as a dark theme.
Themes can be easily switched via a prominent button at the top of the page.
The theme of the screenshots on this page will adapt to the selected documentation Theme, so to see the dark (or light) version, please switch the theme of the Documentation.

### Login

The below screenshot shows the login screen of freqUI.

![FreqUI - login](assets/frequi-login-CORS.png#only-dark)
![FreqUI - login](assets/frequi-login-CORS-light.png#only-light)

!!! Hint "CORS"
    The Cors error shown in this screenshot is due to the fact that the UI is running on a different port than the API, and [CORS](#cors) has not been setup correctly yet.

### Trade view

The trade view allows you to visualize the trades that the bot is making and to interact with the bot.
On this page, you can also interact with the bot by starting and stopping it and - if configured - force trade entries and exits.

![FreqUI - trade view](assets/freqUI-trade-pane-dark.png#only-dark)
![FreqUI - trade view](assets/freqUI-trade-pane-light.png#only-light)

### Plot Configurator

FreqUI Plots can be configured either via a `plot_config` configuration object in the strategy (which can be loaded via "from strategy" button) or via the UI.
Multiple plot configurations can be created and switched at will - allowing for flexible, different views into your charts.

The plot configuration can be accessed via the "Plot Configurator" (Cog icon) button in the top right corner of the trade view.

![FreqUI - plot configuration](assets/freqUI-plot-configurator-dark.png#only-dark)
![FreqUI - plot configuration](assets/freqUI-plot-configurator-light.png#only-light)

### Settings

Several UI related settings can be changed by accessing the settings page.

Things you can change (among others):

* Timezone of the UI
* Visualization of open trades as part of the favicon (browser tab)
* Candle colors (up/down -> red/green)
* Enable / disable in-app notification types

![FreqUI - Settings view](assets/frequi-settings-dark.png#only-dark)
![FreqUI - Settings view](assets/frequi-settings-light.png#only-light)

## Backtesting

When freqtrade is started in [webserver mode](utils.md#webserver-mode) (freqtrade started with `freqtrade webserver`), the backtesting view becomes available.
This view allows you to backtest strategies and visualize the results.

You can also load and visualize previous backtest results, as well as compare the results with each other.

![FreqUI - Backtesting](assets/freqUI-backtesting-dark.png#only-dark)
![FreqUI - Backtesting](assets/freqUI-backtesting-light.png#only-light)


--8<-- "includes/cors.md"

---

## File: hyperopt.md

# Hyperopt

This page explains how to tune your strategy by finding the optimal
parameters, a process called hyperparameter optimization. The bot uses algorithms included in the `scikit-optimize` package to accomplish this.
The search will burn all your CPU cores, make your laptop sound like a fighter jet and still take a long time.

In general, the search for best parameters starts with a few random combinations (see [below](#reproducible-results) for more details) and then uses Bayesian search with a ML regressor algorithm (currently ExtraTreesRegressor) to quickly find a combination of parameters in the search hyperspace that minimizes the value of the [loss function](#loss-functions).

Hyperopt requires historic data to be available, just as backtesting does (hyperopt runs backtesting many times with different parameters).
To learn how to get data for the pairs and exchange you're interested in, head over to the [Data Downloading](data-download.md) section of the documentation.

!!! Bug
    Hyperopt can crash when used with only 1 CPU Core as found out in [Issue #1133](https://github.com/freqtrade/freqtrade/issues/1133)

!!! Note
    Since 2021.4 release you no longer have to write a separate hyperopt class, but can configure the parameters directly in the strategy.
    The legacy method was supported up to 2021.8 and has been removed in 2021.9.

## Install hyperopt dependencies

Since Hyperopt dependencies are not needed to run the bot itself, are heavy, can not be easily built on some platforms (like Raspberry PI), they are not installed by default. Before you run Hyperopt, you need to install the corresponding dependencies, as described in this section below.

!!! Note
    Since Hyperopt is a resource intensive process, running it on a Raspberry Pi is not recommended nor supported.

### Docker

The docker-image includes hyperopt dependencies, no further action needed.

### Easy installation script (setup.sh) / Manual installation

```bash
source .venv/bin/activate
pip install -r requirements-hyperopt.txt
```

## Hyperopt command reference

--8<-- "commands/hyperopt.md"

### Hyperopt checklist

Checklist on all tasks / possibilities in hyperopt

Depending on the space you want to optimize, only some of the below are required:

* define parameters with `space='buy'` - for entry signal optimization
* define parameters with `space='sell'` - for exit signal optimization

!!! Note
    `populate_indicators` needs to create all indicators any of the spaces may use, otherwise hyperopt will not work.

Rarely you may also need to create a [nested class](advanced-hyperopt.md#overriding-pre-defined-spaces) named `HyperOpt` and implement

* `roi_space` - for custom ROI optimization (if you need the ranges for the ROI parameters in the optimization hyperspace that differ from default)
* `generate_roi_table` - for custom ROI optimization (if you need the ranges for the values in the ROI table that differ from default or the number of entries (steps) in the ROI table which differs from the default 4 steps)
* `stoploss_space` - for custom stoploss optimization (if you need the range for the stoploss parameter in the optimization hyperspace that differs from default)
* `trailing_space` - for custom trailing stop optimization (if you need the ranges for the trailing stop parameters in the optimization hyperspace that differ from default)
* `max_open_trades_space` - for custom max_open_trades optimization (if you need the ranges for the max_open_trades parameter in the optimization hyperspace that differ from default)

!!! Tip "Quickly optimize ROI, stoploss and trailing stoploss"
    You can quickly optimize the spaces `roi`, `stoploss` and `trailing` without changing anything in your strategy.

    ``` bash
    # Have a working strategy at hand.
    freqtrade hyperopt --hyperopt-loss SharpeHyperOptLossDaily --spaces roi stoploss trailing --strategy MyWorkingStrategy --config config.json -e 100
    ```

### Hyperopt execution logic

Hyperopt will first load your data into memory and will then run `populate_indicators()` once per Pair to generate all indicators, unless `--analyze-per-epoch` is specified.

Hyperopt will then spawn into different processes (number of processors, or `-j <n>`), and run backtesting over and over again, changing the parameters that are part of the `--spaces` defined.

For every new set of parameters, freqtrade will run first `populate_entry_trend()` followed by `populate_exit_trend()`, and then run the regular backtesting process to simulate trades.

After backtesting, the results are passed into the [loss function](#loss-functions), which will evaluate if this result was better or worse than previous results.  
Based on the loss function result, hyperopt will determine the next set of parameters to try in the next round of backtesting.

### Configure your Guards and Triggers

There are two places you need to change in your strategy file to add a new buy hyperopt for testing:

* Define the parameters at the class level hyperopt shall be optimizing.
* Within `populate_entry_trend()` - use defined parameter values instead of raw constants.

There you have two different types of indicators: 1. `guards` and 2. `triggers`.

1. Guards are conditions like "never buy if ADX < 10", or never buy if current price is over EMA10.
2. Triggers are ones that actually trigger buy in specific moment, like "buy when EMA5 crosses over EMA10" or "buy when close price touches lower Bollinger band".

!!! Hint "Guards and Triggers"
    Technically, there is no difference between Guards and Triggers.  
    However, this guide will make this distinction to make it clear that signals should not be "sticking".
    Sticking signals are signals that are active for multiple candles. This can lead into entering a signal late (right before the signal disappears - which means that the chance of success is a lot lower than right at the beginning).

Hyper-optimization will, for each epoch round, pick one trigger and possibly multiple guards.

#### Exit signal optimization

Similar to the entry-signal above, exit-signals can also be optimized.
Place the corresponding settings into the following methods

* Define the parameters at the class level hyperopt shall be optimizing, either naming them `sell_*`, or by explicitly defining `space='sell'`.
* Within `populate_exit_trend()` - use defined parameter values instead of raw constants.

The configuration and rules are the same than for buy signals.

## Solving a Mystery

Let's say you are curious: should you use MACD crossings or lower Bollinger Bands to trigger your long entries.
And you also wonder should you use RSI or ADX to help with those decisions.
If you decide to use RSI or ADX, which values should I use for them?

So let's use hyperparameter optimization to solve this mystery.

### Defining indicators to be used

We start by calculating the indicators our strategy is going to use.

``` python
class MyAwesomeStrategy(IStrategy):

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        """
        Generate all indicators used by the strategy
        """
        dataframe['adx'] = ta.ADX(dataframe)
        dataframe['rsi'] = ta.RSI(dataframe)
        macd = ta.MACD(dataframe)
        dataframe['macd'] = macd['macd']
        dataframe['macdsignal'] = macd['macdsignal']
        dataframe['macdhist'] = macd['macdhist']

        bollinger = ta.BBANDS(dataframe, timeperiod=20, nbdevup=2.0, nbdevdn=2.0)
        dataframe['bb_lowerband'] = bollinger['lowerband']
        dataframe['bb_middleband'] = bollinger['middleband']
        dataframe['bb_upperband'] = bollinger['upperband']
        return dataframe
```

### Hyperoptable parameters

We continue to define hyperoptable parameters:

```python
class MyAwesomeStrategy(IStrategy):
    buy_adx = DecimalParameter(20, 40, decimals=1, default=30.1, space="buy")
    buy_rsi = IntParameter(20, 40, default=30, space="buy")
    buy_adx_enabled = BooleanParameter(default=True, space="buy")
    buy_rsi_enabled = CategoricalParameter([True, False], default=False, space="buy")
    buy_trigger = CategoricalParameter(["bb_lower", "macd_cross_signal"], default="bb_lower", space="buy")
```

The above definition says: I have five parameters I want to randomly combine to find the best combination.  
`buy_rsi` is an integer parameter, which will be tested between 20 and 40. This space has a size of 20.  
`buy_adx` is a decimal parameter, which will be evaluated between 20 and 40 with 1 decimal place (so values are 20.1, 20.2, ...). This space has a size of 200.  
Then we have three category variables. First two are either `True` or `False`.
We use these to either enable or disable the ADX and RSI guards.
The last one we call `trigger` and use it to decide which buy trigger we want to use.

!!! Note "Parameter space assignment"
    Parameters must either be assigned to a variable named `buy_*` or `sell_*` - or contain `space='buy'` | `space='sell'` to be assigned to a space correctly.
    If no parameter is available for a space, you'll receive the error that no space was found when running hyperopt.  
    Parameters with unclear space (e.g. `adx_period = IntParameter(4, 24, default=14)` - no explicit nor implicit space) will not be detected and will therefore be ignored.

So let's write the buy strategy using these values:

```python
    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        conditions = []
        # GUARDS AND TRENDS
        if self.buy_adx_enabled.value:
            conditions.append(dataframe['adx'] > self.buy_adx.value)
        if self.buy_rsi_enabled.value:
            conditions.append(dataframe['rsi'] < self.buy_rsi.value)

        # TRIGGERS
        if self.buy_trigger.value == 'bb_lower':
            conditions.append(dataframe['close'] < dataframe['bb_lowerband'])
        if self.buy_trigger.value == 'macd_cross_signal':
            conditions.append(qtpylib.crossed_above(
                dataframe['macd'], dataframe['macdsignal']
            ))

        # Check that volume is not 0
        conditions.append(dataframe['volume'] > 0)

        if conditions:
            dataframe.loc[
                reduce(lambda x, y: x & y, conditions),
                'enter_long'] = 1

        return dataframe
```

Hyperopt will now call `populate_entry_trend()` many times (`epochs`) with different value combinations.  
It will use the given historical data and simulate buys based on the buy signals generated with the above function.  
Based on the results, hyperopt will tell you which parameter combination produced the best results (based on the configured [loss function](#loss-functions)).

!!! Note
    The above setup expects to find ADX, RSI and Bollinger Bands in the populated indicators.
    When you want to test an indicator that isn't used by the bot currently, remember to
    add it to the `populate_indicators()` method in your strategy or hyperopt file.

## Parameter types

There are four parameter types each suited for different purposes.

* `IntParameter` - defines an integral parameter with upper and lower boundaries of search space.
* `DecimalParameter` - defines a floating point parameter with a limited number of decimals (default 3). Should be preferred instead of `RealParameter` in most cases.
* `RealParameter` - defines a floating point parameter with upper and lower boundaries and no precision limit. Rarely used as it creates a space with a near infinite number of possibilities.
* `CategoricalParameter` - defines a parameter with a predetermined number of choices.
* `BooleanParameter` - Shorthand for `CategoricalParameter([True, False])` - great for "enable" parameters.

### Parameter options

There are two parameter options that can help you to quickly test various ideas:

* `optimize` - when set to `False`, the parameter will not be included in optimization process. (Default: True)
* `load` - when set to `False`, results of a previous hyperopt run (in `buy_params` and `sell_params` either in your strategy or the JSON output file) will not be used as the starting value for subsequent hyperopts. The default value specified in the parameter will be used instead. (Default: True)

!!! Tip "Effects of `load=False` on backtesting"
    Be aware that setting the `load` option to `False` will mean backtesting will also use the default value specified in the parameter and *not* the value found through hyperoptimisation.

!!! Warning
    Hyperoptable parameters cannot be used in `populate_indicators` - as hyperopt does not recalculate indicators for each epoch, so the starting value would be used in this case.

## Optimizing an indicator parameter

Assuming you have a simple strategy in mind - a EMA cross strategy (2 Moving averages crossing) - and you'd like to find the ideal parameters for this strategy.
By default, we assume a stoploss of 5% - and a take-profit (`minimal_roi`) of 10% - which means freqtrade will sell the trade once 10% profit has been reached.

``` python
from pandas import DataFrame
from functools import reduce

import talib.abstract as ta

from freqtrade.strategy import (BooleanParameter, CategoricalParameter, DecimalParameter, 
                                IStrategy, IntParameter)
import freqtrade.vendor.qtpylib.indicators as qtpylib

class MyAwesomeStrategy(IStrategy):
    stoploss = -0.05
    timeframe = '15m'
    minimal_roi = {
        "0":  0.10
    }
    # Define the parameter spaces
    buy_ema_short = IntParameter(3, 50, default=5)
    buy_ema_long = IntParameter(15, 200, default=50)


    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        """Generate all indicators used by the strategy"""
        
        # Calculate all ema_short values
        for val in self.buy_ema_short.range:
            dataframe[f'ema_short_{val}'] = ta.EMA(dataframe, timeperiod=val)
        
        # Calculate all ema_long values
        for val in self.buy_ema_long.range:
            dataframe[f'ema_long_{val}'] = ta.EMA(dataframe, timeperiod=val)
        
        return dataframe

    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        conditions = []
        conditions.append(qtpylib.crossed_above(
                dataframe[f'ema_short_{self.buy_ema_short.value}'], dataframe[f'ema_long_{self.buy_ema_long.value}']
            ))

        # Check that volume is not 0
        conditions.append(dataframe['volume'] > 0)

        if conditions:
            dataframe.loc[
                reduce(lambda x, y: x & y, conditions),
                'enter_long'] = 1
        return dataframe

    def populate_exit_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        conditions = []
        conditions.append(qtpylib.crossed_above(
                dataframe[f'ema_long_{self.buy_ema_long.value}'], dataframe[f'ema_short_{self.buy_ema_short.value}']
            ))

        # Check that volume is not 0
        conditions.append(dataframe['volume'] > 0)

        if conditions:
            dataframe.loc[
                reduce(lambda x, y: x & y, conditions),
                'exit_long'] = 1
        return dataframe
```

Breaking it down:

Using `self.buy_ema_short.range` will return a range object containing all entries between the Parameters low and high value.
In this case (`IntParameter(3, 50, default=5)`), the loop would run for all numbers between 3 and 50 (`[3, 4, 5, ... 49, 50]`).
By using this in a loop, hyperopt will generate 48 new columns (`['buy_ema_3', 'buy_ema_4', ... , 'buy_ema_50']`).

Hyperopt itself will then use the selected value to create the buy and sell signals.

While this strategy is most likely too simple to provide consistent profit, it should serve as an example how optimize indicator parameters.

!!! Note
    `self.buy_ema_short.range` will act differently between hyperopt and other modes. For hyperopt, the above example may generate 48 new columns, however for all other modes (backtesting, dry/live), it will only generate the column for the selected value. You should therefore avoid using the resulting column with explicit values (values other than `self.buy_ema_short.value`).

!!! Note
    `range` property may also be used with `DecimalParameter` and `CategoricalParameter`. `RealParameter` does not provide this property due to infinite search space.

??? Hint "Performance tip"
    During normal hyperopting, indicators are calculated once and supplied to each epoch, linearly increasing RAM usage as a factor of increasing cores. As this also has performance implications, there are two alternatives to reduce RAM usage

    * Move `ema_short` and `ema_long` calculations from `populate_indicators()` to `populate_entry_trend()`. Since `populate_entry_trend()` will be calculated every epoch, you don't need to use `.range` functionality.
    * hyperopt provides `--analyze-per-epoch` which will move the execution of `populate_indicators()` to the epoch process, calculating a single value per parameter per epoch instead of using the `.range` functionality. In this case, `.range` functionality will only return the actually used value.

    These alternatives will reduce RAM usage, but increase CPU usage. However, your hyperopting run will be less likely to fail due to Out Of Memory (OOM) issues.

    Whether you are using `.range` functionality or the alternatives above, you should try to use space ranges as small as possible since this will improve CPU/RAM usage.

## Optimizing protections

Freqtrade can also optimize protections. How you optimize protections is up to you, and the following should be considered as example only.

The strategy will simply need to define the "protections" entry as property returning a list of protection configurations.

``` python
from pandas import DataFrame
from functools import reduce

import talib.abstract as ta

from freqtrade.strategy import (BooleanParameter, CategoricalParameter, DecimalParameter, 
                                IStrategy, IntParameter)
import freqtrade.vendor.qtpylib.indicators as qtpylib

class MyAwesomeStrategy(IStrategy):
    stoploss = -0.05
    timeframe = '15m'
    # Define the parameter spaces
    cooldown_lookback = IntParameter(2, 48, default=5, space="protection", optimize=True)
    stop_duration = IntParameter(12, 200, default=5, space="protection", optimize=True)
    use_stop_protection = BooleanParameter(default=True, space="protection", optimize=True)


    @property
    def protections(self):
        prot = []

        prot.append({
            "method": "CooldownPeriod",
            "stop_duration_candles": self.cooldown_lookback.value
        })
        if self.use_stop_protection.value:
            prot.append({
                "method": "StoplossGuard",
                "lookback_period_candles": 24 * 3,
                "trade_limit": 4,
                "stop_duration_candles": self.stop_duration.value,
                "only_per_pair": False
            })

        return prot

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        # ...
        
```

You can then run hyperopt as follows:
`freqtrade hyperopt --hyperopt-loss SharpeHyperOptLossDaily --strategy MyAwesomeStrategy --spaces protection`

!!! Note
    The protection space is not part of the default space, and is only available with the Parameters Hyperopt interface, not with the legacy hyperopt interface (which required separate hyperopt files).
    Freqtrade will also automatically change the "--enable-protections" flag if the protection space is selected.

!!! Warning
    If protections are defined as property, entries from the configuration will be ignored.
    It is therefore recommended to not define protections in the configuration.

### Migrating from previous property setups

A migration from a previous setup is pretty simple, and can be accomplished by converting the protections entry to a property.
In simple terms, the following configuration will be converted to the below.

``` python
class MyAwesomeStrategy(IStrategy):
    protections = [
        {
            "method": "CooldownPeriod",
            "stop_duration_candles": 4
        }
    ]
```

Result

``` python
class MyAwesomeStrategy(IStrategy):
    
    @property
    def protections(self):
        return [
            {
                "method": "CooldownPeriod",
                "stop_duration_candles": 4
            }
        ]
```

You will then obviously also change potential interesting entries to parameters to allow hyper-optimization.

### Optimizing `max_entry_position_adjustment`

While `max_entry_position_adjustment` is not a separate space, it can still be used in hyperopt by using the property approach shown above.

``` python
from pandas import DataFrame
from functools import reduce

import talib.abstract as ta

from freqtrade.strategy import (BooleanParameter, CategoricalParameter, DecimalParameter, 
                                IStrategy, IntParameter)
import freqtrade.vendor.qtpylib.indicators as qtpylib

class MyAwesomeStrategy(IStrategy):
    stoploss = -0.05
    timeframe = '15m'

    # Define the parameter spaces
    max_epa = CategoricalParameter([-1, 0, 1, 3, 5, 10], default=1, space="buy", optimize=True)

    @property
    def max_entry_position_adjustment(self):
        return self.max_epa.value
        

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        # ...
```

??? Tip "Using `IntParameter`"
    You can also use the `IntParameter` for this optimization, but you must explicitly return an integer:
    ``` python
    max_epa = IntParameter(-1, 10, default=1, space="buy", optimize=True)

    @property
    def max_entry_position_adjustment(self):
        return int(self.max_epa.value)
    ```

## Loss-functions

Each hyperparameter tuning requires a target. This is usually defined as a loss function (sometimes also called objective function), which should decrease for more desirable results, and increase for bad results.

A loss function must be specified via the `--hyperopt-loss <Class-name>` argument (or optionally via the configuration under the `"hyperopt_loss"` key).
This class should be in its own file within the `user_data/hyperopts/` directory.

Currently, the following loss functions are builtin:

* `ShortTradeDurHyperOptLoss` - (default legacy Freqtrade hyperoptimization loss function) - Mostly for short trade duration and avoiding losses.
* `OnlyProfitHyperOptLoss` - takes only amount of profit into consideration.
* `SharpeHyperOptLoss` - Optimizes Sharpe Ratio calculated on trade returns relative to standard deviation.
* `SharpeHyperOptLossDaily` - Optimizes Sharpe Ratio calculated on **daily** trade returns relative to standard deviation.
* `SortinoHyperOptLoss` - Optimizes Sortino Ratio calculated on trade returns relative to **downside** standard deviation.
* `SortinoHyperOptLossDaily` - optimizes Sortino Ratio calculated on **daily** trade returns relative to **downside** standard deviation.
* `MaxDrawDownHyperOptLoss` - Optimizes Maximum absolute drawdown.
* `MaxDrawDownRelativeHyperOptLoss` -  Optimizes both maximum absolute drawdown while also adjusting for maximum relative drawdown.
* `CalmarHyperOptLoss` - Optimizes Calmar Ratio calculated on trade returns relative to max drawdown.
* `ProfitDrawDownHyperOptLoss` - Optimizes by max Profit & min Drawdown objective. `DRAWDOWN_MULT` variable within the hyperoptloss file can be adjusted to be stricter or more flexible on drawdown purposes.
* `MultiMetricHyperOptLoss` -  Optimizes by several key metrics to achieve balanced performance. The primary focus is on maximizing Profit and minimizing Drawdown, while also considering additional metrics such as Profit Factor, Expectancy Ratio and Winrate. Moreover, it applies a penalty for epochs with a low number of trades, encouraging strategies with adequate trade frequency.

Creation of a custom loss function is covered in the [Advanced Hyperopt](advanced-hyperopt.md) part of the documentation.

## Execute Hyperopt

Once you have updated your hyperopt configuration you can run it.
Because hyperopt tries a lot of combinations to find the best parameters it will take time to get a good result.

We strongly recommend to use `screen` or `tmux` to prevent any connection loss.

```bash
freqtrade hyperopt --config config.json --hyperopt-loss <hyperoptlossname> --strategy <strategyname> -e 500 --spaces all
```

The `-e` option will set how many evaluations hyperopt will do. Since hyperopt uses Bayesian search, running too many epochs at once may not produce greater results. Experience has shown that best results are usually not improving much after 500-1000 epochs.  
Doing multiple runs (executions) with a few 1000 epochs and different random state will most likely produce different results.

The `--spaces all` option determines that all possible parameters should be optimized. Possibilities are listed below.

!!! Note
    Hyperopt will store hyperopt results with the timestamp of the hyperopt start time.
    Reading commands (`hyperopt-list`, `hyperopt-show`) can use `--hyperopt-filename <filename>` to read and display older hyperopt results.
    You can find a list of filenames with `ls -l user_data/hyperopt_results/`.

### Execute Hyperopt with different historical data source

If you would like to hyperopt parameters using an alternate historical data set that
you have on-disk, use the `--datadir PATH` option. By default, hyperopt uses data from directory `user_data/data`.

### Running Hyperopt with a smaller test-set

Use the `--timerange` argument to change how much of the test-set you want to use.
For example, to use one month of data, pass `--timerange 20210101-20210201` (from january 2021 - february 2021) to the hyperopt call.

Full command:

```bash
freqtrade hyperopt --strategy <strategyname> --timerange 20210101-20210201
```

### Running Hyperopt with Smaller Search Space

Use the `--spaces` option to limit the search space used by hyperopt.
Letting Hyperopt optimize everything is a huuuuge search space.
Often it might make more sense to start by just searching for initial buy algorithm.
Or maybe you just want to optimize your stoploss or roi table for that awesome new buy strategy you have.

Legal values are:

* `all`: optimize everything
* `buy`: just search for a new buy strategy
* `sell`: just search for a new sell strategy
* `roi`: just optimize the minimal profit table for your strategy
* `stoploss`: search for the best stoploss value
* `trailing`: search for the best trailing stop values
* `trades`: search for the best max open trades values
* `protection`: search for the best protection parameters (read the [protections section](#optimizing-protections) on how to properly define these)
* `default`: `all` except `trailing` and `protection`
* space-separated list of any of the above values for example `--spaces roi stoploss`

The default Hyperopt Search Space, used when no `--space` command line option is specified, does not include the `trailing` hyperspace. We recommend you to run optimization for the `trailing` hyperspace separately, when the best parameters for other hyperspaces were found, validated and pasted into your custom strategy.

## Understand the Hyperopt Result

Once Hyperopt is completed you can use the result to update your strategy.
Given the following result from hyperopt:

```
Best result:

    44/100:    135 trades. Avg profit  0.57%. Total profit  0.03871918 BTC (0.7722%). Avg duration 180.4 mins. Objective: 1.94367

    # Buy hyperspace params:
    buy_params = {
        'buy_adx': 44,
        'buy_rsi': 29,
        'buy_adx_enabled': False,
        'buy_rsi_enabled': True,
        'buy_trigger': 'bb_lower'
    }
```

You should understand this result like:

* The buy trigger that worked best was `bb_lower`.
* You should not use ADX because `'buy_adx_enabled': False`.
* You should **consider** using the RSI indicator (`'buy_rsi_enabled': True`) and the best value is `29.0` (`'buy_rsi': 29.0`)

### Automatic parameter application to the strategy

When using Hyperoptable parameters, the result of your hyperopt-run will be written to a json file next to your strategy (so for `MyAwesomeStrategy.py`, the file would be `MyAwesomeStrategy.json`).  
This file is also updated when using the `hyperopt-show` sub-command, unless `--disable-param-export` is provided to either of the 2 commands.


Your strategy class can also contain these results explicitly. Simply copy hyperopt results block and paste them at class level, replacing old parameters (if any). New parameters will automatically be loaded next time strategy is executed.

Transferring your whole hyperopt result to your strategy would then look like:

```python
class MyAwesomeStrategy(IStrategy):
    # Buy hyperspace params:
    buy_params = {
        'buy_adx': 44,
        'buy_rsi': 29,
        'buy_adx_enabled': False,
        'buy_rsi_enabled': True,
        'buy_trigger': 'bb_lower'
    }
```

!!! Note
    Values in the configuration file will overwrite Parameter-file level parameters - and both will overwrite parameters within the strategy.
    The prevalence is therefore: config > parameter file > strategy `*_params` > parameter default

### Understand Hyperopt ROI results

If you are optimizing ROI (i.e. if optimization search-space contains 'all', 'default' or 'roi'), your result will look as follows and include a ROI table:

```
Best result:

    44/100:    135 trades. Avg profit  0.57%. Total profit  0.03871918 BTC (0.7722%). Avg duration 180.4 mins. Objective: 1.94367

    # ROI table:
    minimal_roi = {
        0: 0.10674,
        21: 0.09158,
        78: 0.03634,
        118: 0
    }
```

In order to use this best ROI table found by Hyperopt in backtesting and for live trades/dry-run, copy-paste it as the value of the `minimal_roi` attribute of your custom strategy:

```
    # Minimal ROI designed for the strategy.
    # This attribute will be overridden if the config file contains "minimal_roi"
    minimal_roi = {
        0: 0.10674,
        21: 0.09158,
        78: 0.03634,
        118: 0
    }
```

As stated in the comment, you can also use it as the value of the `minimal_roi` setting in the configuration file.

#### Default ROI Search Space

If you are optimizing ROI, Freqtrade creates the 'roi' optimization hyperspace for you -- it's the hyperspace of components for the ROI tables. By default, each ROI table generated by the Freqtrade consists of 4 rows (steps). Hyperopt implements adaptive ranges for ROI tables with ranges for values in the ROI steps that depend on the timeframe used. By default the values vary in the following ranges (for some of the most used timeframes, values are rounded to 3 digits after the decimal point):

| # step | 1m     |               | 5m       |             | 1h         |               | 1d           |               |
| ------ | ------ | ------------- | -------- | ----------- | ---------- | ------------- | ------------ | ------------- |
| 1      | 0      | 0.011...0.119 | 0        | 0.03...0.31 | 0          | 0.068...0.711 | 0            | 0.121...1.258 |
| 2      | 2...8  | 0.007...0.042 | 10...40  | 0.02...0.11 | 120...480  | 0.045...0.252 | 2880...11520 | 0.081...0.446 |
| 3      | 4...20 | 0.003...0.015 | 20...100 | 0.01...0.04 | 240...1200 | 0.022...0.091 | 5760...28800 | 0.040...0.162 |
| 4      | 6...44 | 0.0           | 30...220 | 0.0         | 360...2640 | 0.0           | 8640...63360 | 0.0           |

These ranges should be sufficient in most cases. The minutes in the steps (ROI dict keys) are scaled linearly depending on the timeframe used. The ROI values in the steps (ROI dict values) are scaled logarithmically depending on the timeframe used.

If you have the `generate_roi_table()` and `roi_space()` methods in your custom hyperopt, remove them in order to utilize these adaptive ROI tables and the ROI hyperoptimization space generated by Freqtrade by default.

Override the `roi_space()` method if you need components of the ROI tables to vary in other ranges. Override the `generate_roi_table()` and `roi_space()` methods and implement your own custom approach for generation of the ROI tables during hyperoptimization if you need a different structure of the ROI tables or other amount of rows (steps).

A sample for these methods can be found in the [overriding pre-defined spaces section](advanced-hyperopt.md#overriding-pre-defined-spaces).

!!! Note "Reduced search space"
    To limit the search space further, Decimals are limited to 3 decimal places (a precision of 0.001). This is usually sufficient, every value more precise than this will usually result in overfitted results. You can however [overriding pre-defined spaces](advanced-hyperopt.md#overriding-pre-defined-spaces) to change this to your needs.

### Understand Hyperopt Stoploss results

If you are optimizing stoploss values (i.e. if optimization search-space contains 'all', 'default' or 'stoploss'), your result will look as follows and include stoploss:

```
Best result:

    44/100:    135 trades. Avg profit  0.57%. Total profit  0.03871918 BTC (0.7722%). Avg duration 180.4 mins. Objective: 1.94367

    # Buy hyperspace params:
    buy_params = {
        'buy_adx': 44,
        'buy_rsi': 29,
        'buy_adx_enabled': False,
        'buy_rsi_enabled': True,
        'buy_trigger': 'bb_lower'
    }

    stoploss: -0.27996
```

In order to use this best stoploss value found by Hyperopt in backtesting and for live trades/dry-run, copy-paste it as the value of the `stoploss` attribute of your custom strategy:

``` python
    # Optimal stoploss designed for the strategy
    # This attribute will be overridden if the config file contains "stoploss"
    stoploss = -0.27996
```

As stated in the comment, you can also use it as the value of the `stoploss` setting in the configuration file.

#### Default Stoploss Search Space

If you are optimizing stoploss values, Freqtrade creates the 'stoploss' optimization hyperspace for you. By default, the stoploss values in that hyperspace vary in the range -0.35...-0.02, which is sufficient in most cases.

If you have the `stoploss_space()` method in your custom hyperopt file, remove it in order to utilize Stoploss hyperoptimization space generated by Freqtrade by default.

Override the `stoploss_space()` method and define the desired range in it if you need stoploss values to vary in other range during hyperoptimization. A sample for this method can be found in the [overriding pre-defined spaces section](advanced-hyperopt.md#overriding-pre-defined-spaces).

!!! Note "Reduced search space"
    To limit the search space further, Decimals are limited to 3 decimal places (a precision of 0.001). This is usually sufficient, every value more precise than this will usually result in overfitted results. You can however [overriding pre-defined spaces](advanced-hyperopt.md#overriding-pre-defined-spaces) to change this to your needs.

### Understand Hyperopt Trailing Stop results

If you are optimizing trailing stop values (i.e. if optimization search-space contains 'all' or 'trailing'), your result will look as follows and include trailing stop parameters:

```
Best result:

    45/100:    606 trades. Avg profit  1.04%. Total profit  0.31555614 BTC ( 630.48%). Avg duration 150.3 mins. Objective: -1.10161

    # Trailing stop:
    trailing_stop = True
    trailing_stop_positive = 0.02001
    trailing_stop_positive_offset = 0.06038
    trailing_only_offset_is_reached = True
```

In order to use these best trailing stop parameters found by Hyperopt in backtesting and for live trades/dry-run, copy-paste them as the values of the corresponding attributes of your custom strategy:

``` python
    # Trailing stop
    # These attributes will be overridden if the config file contains corresponding values.
    trailing_stop = True
    trailing_stop_positive = 0.02001
    trailing_stop_positive_offset = 0.06038
    trailing_only_offset_is_reached = True
```

As stated in the comment, you can also use it as the values of the corresponding settings in the configuration file.

#### Default Trailing Stop Search Space

If you are optimizing trailing stop values, Freqtrade creates the 'trailing' optimization hyperspace for you. By default, the `trailing_stop` parameter is always set to True in that hyperspace, the value of the `trailing_only_offset_is_reached` vary between True and False, the values of the `trailing_stop_positive` and `trailing_stop_positive_offset` parameters vary in the ranges 0.02...0.35 and 0.01...0.1 correspondingly, which is sufficient in most cases.

Override the `trailing_space()` method and define the desired range in it if you need values of the trailing stop parameters to vary in other ranges during hyperoptimization. A sample for this method can be found in the [overriding pre-defined spaces section](advanced-hyperopt.md#overriding-pre-defined-spaces).

!!! Note "Reduced search space"
    To limit the search space further, Decimals are limited to 3 decimal places (a precision of 0.001). This is usually sufficient, every value more precise than this will usually result in overfitted results. You can however [overriding pre-defined spaces](advanced-hyperopt.md#overriding-pre-defined-spaces) to change this to your needs.

### Reproducible results

The search for optimal parameters starts with a few (currently 30) random combinations in the hyperspace of parameters, random Hyperopt epochs. These random epochs are marked with an asterisk character (`*`) in the first column in the Hyperopt output.

The initial state for generation of these random values (random state) is controlled by the value of the `--random-state` command line option. You can set it to some arbitrary value of your choice to obtain reproducible results.

If you have not set this value explicitly in the command line options, Hyperopt seeds the random state with some random value for you. The random state value for each Hyperopt run is shown in the log, so you can copy and paste it into the `--random-state` command line option to repeat the set of the initial random epochs used.

If you have not changed anything in the command line options, configuration, timerange, Strategy and Hyperopt classes, historical data and the Loss Function -- you should obtain same hyper-optimization results with same random state value used.

## Output formatting

By default, hyperopt prints colorized results -- epochs with positive profit are printed in the green color. This highlighting helps you find epochs that can be interesting for later analysis. Epochs with zero total profit or with negative profits (losses) are printed in the normal color. If you do not need colorization of results (for instance, when you are redirecting hyperopt output to a file) you can switch colorization off by specifying the `--no-color` option in the command line.

You can use the `--print-all` command line option if you would like to see all results in the hyperopt output, not only the best ones. When `--print-all` is used, current best results are also colorized by default -- they are printed in bold (bright) style. This can also be switched off with the `--no-color` command line option.

!!! Note "Windows and color output"
    Windows does not support color-output natively, therefore it is automatically disabled. To have color-output for hyperopt running under windows, please consider using WSL.

## Position stacking and disabling max market positions

In some situations, you may need to run Hyperopt (and Backtesting) with the `--eps`/`--enable-position-staking` argument, or you may need to set `max_open_trades` to a very high number to disable the limit on the number of open trades.

By default, hyperopt emulates the behavior of the Freqtrade Live Run/Dry Run, where only one
open trade per pair is allowed. The total number of trades open for all pairs
is also limited by the `max_open_trades` setting. During Hyperopt/Backtesting this may lead to
potential trades being hidden (or masked) by already open trades.

The `--eps`/`--enable-position-stacking` argument allows emulation of buying the same pair multiple times.
Using `--max-open-trades` with a very high number will disable the limit on the number of open trades.

!!! Note
    Dry/live runs will **NOT** use position stacking - therefore it does make sense to also validate the strategy without this as it's closer to reality.

You can also enable position stacking in the configuration file by explicitly setting
`"position_stacking"=true`.

## Out of Memory errors

As hyperopt consumes a lot of memory (the complete data needs to be in memory once per parallel backtesting process), it's likely that you run into "out of memory" errors.
To combat these, you have multiple options:

* Reduce the amount of pairs.
* Reduce the timerange used (`--timerange <timerange>`).
* Avoid using `--timeframe-detail` (this loads a lot of additional data into memory).
* Reduce the number of parallel processes (`-j <n>`).
* Increase the memory of your machine.
* Use `--analyze-per-epoch` if you're using a lot of parameters with `.range` functionality.


## The objective has been evaluated at this point before.

If you see `The objective has been evaluated at this point before.` - then this is a sign that your space has been exhausted, or is close to that.
Basically all points in your space have been hit (or a local minima has been hit) - and hyperopt does no longer find points in the multi-dimensional space it did not try yet.
Freqtrade tries to counter the "local minima" problem by using new, randomized points in this case.

Example:

``` python
buy_ema_short = IntParameter(5, 20, default=10, space="buy", optimize=True)
# This is the only parameter in the buy space
```

The `buy_ema_short` space has 15 possible values (`5, 6, ... 19, 20`). If you now run hyperopt for the buy space, hyperopt will only have 15 values to try before running out of options.
Your epochs should therefore be aligned to the possible values - or you should be ready to interrupt a run if you norice a lot of `The objective has been evaluated at this point before.` warnings.

## Show details of Hyperopt results

After you run Hyperopt for the desired amount of epochs, you can later list all results for analysis, select only best or profitable once, and show the details for any of the epochs previously evaluated. This can be done with the `hyperopt-list` and `hyperopt-show` sub-commands. The usage of these sub-commands is described in the [Utils](utils.md#list-hyperopt-results) chapter.

## Output debug messages from your strategy

If you want to output debug messages from your strategy, you can use the `logging` module. By default, Freqtrade will output all messages with a level of `INFO` or higher. 


``` python
import logging


logger = logging.getLogger(__name__)


class MyAwesomeStrategy(IStrategy):
    ...

    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        logger.info("This is a debug message")
        ...

```

!!! Note "using print"
    Messages printed via `print()` will not be shown in the hyperopt output unless parallelism is disabled (`-j 1`). 
    It is recommended to use the `logging` module instead.

## Validate backtesting results

Once the optimized strategy has been implemented into your strategy, you should backtest this strategy to make sure everything is working as expected.

To achieve same the results (number of trades, their durations, profit, etc.) as during Hyperopt, please use the same configuration and parameters (timerange, timeframe, ...) used for hyperopt for Backtesting.

### Why do my backtest results not match my hyperopt results?

Should results not match, check the following factors:

* You may have added parameters to hyperopt in `populate_indicators()` where they will be calculated only once **for all epochs**. If you are, for example, trying to optimise multiple SMA timeperiod values, the hyperoptable timeperiod parameter should be placed in `populate_entry_trend()` which is calculated every epoch. See [Optimizing an indicator parameter](https://www.freqtrade.io/en/stable/hyperopt/#optimizing-an-indicator-parameter).
* If you have disabled the auto-export of hyperopt parameters into the JSON parameters file, double-check to make sure you transferred all hyperopted values into your strategy correctly.
* Check the logs to verify what parameters are being set and what values are being used.
* Pay special care to the stoploss, max_open_trades and trailing stoploss parameters, as these are often set in configuration files, which override changes to the strategy. Check the logs of your backtest to ensure that there were no parameters inadvertently set by the configuration (like `stoploss`, `max_open_trades` or `trailing_stop`).
* Verify that you do not have an unexpected parameters JSON file overriding the parameters or the default hyperopt settings in your strategy.
* Verify that any protections that are enabled in backtesting are also enabled when hyperopting, and vice versa. When using `--space protection`, protections are auto-enabled for hyperopting.

---

## File: index.md

![freqtrade](assets/freqtrade_poweredby.svg)

[![Freqtrade CI](https://github.com/freqtrade/freqtrade/actions/workflows/ci.yml/badge.svg?branch=develop)](https://github.com/freqtrade/freqtrade/actions/)
[![DOI](https://joss.theoj.org/papers/10.21105/joss.04864/status.svg)](https://doi.org/10.21105/joss.04864)
[![Coverage Status](https://coveralls.io/repos/github/freqtrade/freqtrade/badge.svg?branch=develop&service=github)](https://coveralls.io/github/freqtrade/freqtrade?branch=develop)
[![Maintainability](https://api.codeclimate.com/v1/badges/5737e6d668200b7518ff/maintainability)](https://codeclimate.com/github/freqtrade/freqtrade/maintainability)

<!-- Place this tag where you want the button to render. -->
<a class="github-button" href="https://github.com/freqtrade/freqtrade" data-icon="octicon-star" data-size="large" aria-label="Star freqtrade/freqtrade on GitHub">Star</a>
<a class="github-button" href="https://github.com/freqtrade/freqtrade/fork" data-icon="octicon-repo-forked" data-size="large" aria-label="Fork freqtrade/freqtrade on GitHub">Fork</a>
<a class="github-button" href="https://github.com/freqtrade/freqtrade/archive/stable.zip" data-icon="octicon-cloud-download" data-size="large" aria-label="Download freqtrade/freqtrade on GitHub">Download</a>

## Introduction

Freqtrade is a free and open source crypto trading bot written in Python. It is designed to support all major exchanges and be controlled via Telegram or webUI. It contains backtesting, plotting and money management tools as well as strategy optimization by machine learning.

!!! Danger "DISCLAIMER"
    This software is for educational purposes only. Do not risk money which you are afraid to lose. USE THE SOFTWARE AT YOUR OWN RISK. THE AUTHORS AND ALL AFFILIATES ASSUME NO RESPONSIBILITY FOR YOUR TRADING RESULTS.

    Always start by running a trading bot in Dry-run and do not engage money before you understand how it works and what profit/loss you should expect.

    We strongly recommend you to have basic coding skills and Python knowledge. Do not hesitate to read the source code and understand the mechanisms of this bot, algorithms and techniques implemented in it.

![freqtrade screenshot](assets/freqtrade-screenshot.png)

## Features

- Develop your Strategy: Write your strategy in python, using [pandas](https://pandas.pydata.org/). Example strategies to inspire you are available in the [strategy repository](https://github.com/freqtrade/freqtrade-strategies).
- Download market data: Download historical data of the exchange and the markets your may want to trade with.
- Backtest: Test your strategy on downloaded historical data.
- Optimize: Find the best parameters for your strategy using hyperoptimization which employs machine learning methods. You can optimize buy, sell, take profit (ROI), stop-loss and trailing stop-loss parameters for your strategy.
- Select markets: Create your static list or use an automatic one based on top traded volumes and/or prices (not available during backtesting). You can also explicitly blacklist markets you don't want to trade.
- Run: Test your strategy with simulated money (Dry-Run mode) or deploy it with real money (Live-Trade mode).
- Run using Edge (optional module): The concept is to find the best historical [trade expectancy](edge.md#expectancy) by markets based on variation of the stop-loss and then allow/reject markets to trade. The sizing of the trade is based on a risk of a percentage of your capital.
- Control/Monitor: Use Telegram or a WebUI (start/stop the bot, show profit/loss, daily summary, current open trades results, etc.).
- Analyze: Further analysis can be performed on either Backtesting data or Freqtrade trading history (SQL database), including automated standard plots, and methods to load the data into [interactive environments](data-analysis.md).

## Supported exchange marketplaces

Please read the [exchange specific notes](exchanges.md) to learn about eventual, special configurations needed for each exchange.

- [X] [Binance](https://www.binance.com/)
- [X] [BingX](https://bingx.com/invite/0EM9RX)
- [X] [Bitmart](https://bitmart.com/)
- [X] [Bybit](https://bybit.com/)
- [X] [Gate.io](https://www.gate.io/ref/6266643)
- [X] [HTX](https://www.htx.com/)
- [X] [Hyperliquid](https://hyperliquid.xyz/) (A decentralized exchange, or DEX)
- [X] [Kraken](https://kraken.com/)
- [X] [OKX](https://okx.com/)
- [X] [MyOKX](https://okx.com/) (OKX EEA)
- [ ] [potentially many others through <img alt="ccxt" width="30px" src="assets/ccxt-logo.svg" />](https://github.com/ccxt/ccxt/). _(We cannot guarantee they will work)_

### Supported Futures Exchanges (experimental)

- [X] [Binance](https://www.binance.com/)
- [X] [Bybit](https://bybit.com/)
- [X] [Gate.io](https://www.gate.io/ref/6266643)
- [X] [Hyperliquid](https://hyperliquid.xyz/) (A decentralized exchange, or DEX)
- [X] [OKX](https://okx.com/)

Please make sure to read the [exchange specific notes](exchanges.md), as well as the [trading with leverage](leverage.md) documentation before diving in.

### Community tested

Exchanges confirmed working by the community:

- [X] [Bitvavo](https://bitvavo.com/)
- [X] [Kucoin](https://www.kucoin.com/)

## Community showcase

--8<-- "includes/showcase.md"

## Requirements

### Hardware requirements

To run this bot we recommend you a linux cloud instance with a minimum of:

- 2GB RAM
- 1GB disk space
- 2vCPU

### Software requirements

- Docker (Recommended)

Alternatively

- Python 3.10+
- pip (pip3)
- git
- TA-Lib
- virtualenv (Recommended)

## Support

### Help / Discord

For any questions not covered by the documentation or for further information about the bot, or to simply engage with like-minded individuals, we encourage you to join the Freqtrade [discord server](https://discord.gg/p7nuUNVfP7).

## Ready to try?

Begin by reading the installation guide [for docker](docker_quickstart.md) (recommended), or for [installation without docker](installation.md).

---

## File: installation.md

# Installation

This page explains how to prepare your environment for running the bot.

The freqtrade documentation describes various ways to install freqtrade

* [Docker images](docker_quickstart.md) (separate page)
* [Script Installation](#script-installation)
* [Manual Installation](#manual-installation)
* [Installation with Conda](#installation-with-conda)

Please consider using the prebuilt [docker images](docker_quickstart.md) to get started quickly while evaluating how freqtrade works.

------

## Information

For Windows installation, please use the [windows installation guide](windows_installation.md).

The easiest way to install and run Freqtrade is to clone the bot Github repository and then run the `./setup.sh` script, if it's available for your platform.

!!! Note "Version considerations"
    When cloning the repository the default working branch has the name `develop`. This branch contains all last features (can be considered as relatively stable, thanks to automated tests).
    The `stable` branch contains the code of the last release (done usually once per month on an approximately one week old snapshot of the `develop` branch to prevent packaging bugs, so potentially it's more stable).

!!! Note
    Python3.10 or higher and the corresponding `pip` are assumed to be available. The install-script will warn you and stop if that's not the case. `git` is also needed to clone the Freqtrade repository.  
    Also, python headers (`python<yourversion>-dev` / `python<yourversion>-devel`) must be available for the installation to complete successfully.

!!! Warning "Up-to-date clock"
    The clock on the system running the bot must be accurate, synchronized to a NTP server frequently enough to avoid problems with communication to the exchanges.

------

## Requirements

These requirements apply to both [Script Installation](#script-installation) and [Manual Installation](#manual-installation).

!!! Note "ARM64 systems"
    If you are running an ARM64 system (like a MacOS M1 or an Oracle VM), please use [docker](docker_quickstart.md) to run freqtrade.
    While native installation is possible with some manual effort, this is not supported at the moment.

### Install guide

* [Python >= 3.10](http://docs.python-guide.org/en/latest/starting/installation/)
* [pip](https://pip.pypa.io/en/stable/installing/)
* [git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
* [virtualenv](https://virtualenv.pypa.io/en/stable/installation.html) (Recommended)
* [TA-Lib](https://ta-lib.github.io/ta-lib-python/) (install instructions [below](#install-ta-lib))

### Install code

We've included/collected install instructions for Ubuntu, MacOS, and Windows. These are guidelines and your success may vary with other distros.
OS Specific steps are listed first, the common section below is necessary for all systems.

!!! Note
    Python3.10 or higher and the corresponding pip are assumed to be available.

=== "Debian/Ubuntu"
    #### Install necessary dependencies

    ```bash
    # update repository
    sudo apt-get update

    # install packages
    sudo apt install -y python3-pip python3-venv python3-dev python3-pandas git curl
    ```

=== "MacOS"
    #### Install necessary dependencies

    Install [Homebrew](https://brew.sh/) if you don't have it already.

    ```bash
    # install packages
    brew install gettext libomp
    ```
    !!! Note
        The `setup.sh` script will install these dependencies for you - assuming brew is installed on your system.

=== "RaspberryPi/Raspbian"
    The following assumes the latest [Raspbian Buster lite image](https://www.raspberrypi.org/downloads/raspbian/).
    This image comes with python3.11 preinstalled, making it easy to get freqtrade up and running.

    Tested using a Raspberry Pi 3 with the Raspbian Buster lite image, all updates applied.


    ```bash
    sudo apt-get install python3-venv libatlas-base-dev cmake curl
    # Use piwheels.org to speed up installation
    sudo echo "[global]\nextra-index-url=https://www.piwheels.org/simple" > tee /etc/pip.conf

    git clone https://github.com/freqtrade/freqtrade.git
    cd freqtrade

    bash setup.sh -i
    ```

    !!! Note "Installation duration"
        Depending on your internet speed and the Raspberry Pi version, installation can take multiple hours to complete.
        Due to this, we recommend to use the pre-build docker-image for Raspberry, by following the [Docker quickstart documentation](docker_quickstart.md)

    !!! Note
        The above does not install hyperopt dependencies. To install these, please use `python3 -m pip install -e .[hyperopt]`.
        We do not advise to run hyperopt on a Raspberry Pi, since this is a very resource-heavy operation, which should be done on powerful machine.

------

## Freqtrade repository

Freqtrade is an open source crypto-currency trading bot, whose code is hosted on `github.com`

```bash
# Download `develop` branch of freqtrade repository
git clone https://github.com/freqtrade/freqtrade.git

# Enter downloaded directory
cd freqtrade

# your choice (1): novice user
git checkout stable

# your choice (2): advanced user
git checkout develop
```

(1) This command switches the cloned repository to the use of the `stable` branch. It's not needed, if you wish to stay on the (2) `develop` branch.

You may later switch between branches at any time with the `git checkout stable`/`git checkout develop` commands.

??? Note "Install from pypi"
    An alternative way to install Freqtrade is from [pypi](https://pypi.org/project/freqtrade/). The downside is that this method requires ta-lib to be correctly installed beforehand, and is therefore currently not the recommended way to install Freqtrade.

    ``` bash
    pip install freqtrade
    ```

------

## Script Installation

First of the ways to install Freqtrade, is to use provided the Linux/MacOS `./setup.sh` script, which install all dependencies and help you configure the bot.

Make sure you fulfill the [Requirements](#requirements) and have downloaded the [Freqtrade repository](#freqtrade-repository).

### Use /setup.sh -install (Linux/MacOS)

If you are on Debian, Ubuntu or MacOS, freqtrade provides the script to install freqtrade.

```bash
# --install, Install freqtrade from scratch
./setup.sh -i
```

### Activate your virtual environment

Each time you open a new terminal, you must run `source .venv/bin/activate` to activate your virtual environment.

```bash
# activate virtual environment
source ./.venv/bin/activate
```

[You are now ready](#you-are-ready) to run the bot.

### Other options of /setup.sh script

You can as well update, configure and reset the codebase of your bot with `./script.sh`

```bash
# --update, Command git pull to update.
./setup.sh -u
# --reset, Hard reset your develop/stable branch.
./setup.sh -r
```

```
** --install **

With this option, the script will install the bot and most dependencies:
You will need to have git and python3.10+ installed beforehand for this to work.

* Mandatory software as: `ta-lib`
* Setup your virtualenv under `.venv/`

This option is a combination of installation tasks and `--reset`

** --update **

This option will pull the last version of your current branch and update your virtualenv. Run the script with this option periodically to update your bot.

** --reset **

This option will hard reset your branch (only if you are on either `stable` or `develop`) and recreate your virtualenv.
```

-----

## Manual Installation

Make sure you fulfill the [Requirements](#requirements) and have downloaded the [Freqtrade repository](#freqtrade-repository).

### Install TA-Lib

#### TA-Lib script installation

```bash
sudo ./build_helpers/install_ta-lib.sh
```

!!! Note
    This will use the ta-lib tar.gz included in this repository.

##### TA-Lib manual installation

[Official installation guide](https://ta-lib.github.io/ta-lib-python/install.html)

```bash
wget http://prdownloads.sourceforge.net/ta-lib/ta-lib-0.4.0-src.tar.gz
tar xvzf ta-lib-0.4.0-src.tar.gz
cd ta-lib
sed -i.bak "s|0.00000001|0.000000000000000001 |g" src/ta_func/ta_utility.h
./configure --prefix=/usr/local
make
sudo make install
# On debian based systems (debian, ubuntu, ...) - updating ldconfig might be necessary.
sudo ldconfig  
cd ..
rm -rf ./ta-lib*
```

### Setup Python virtual environment (virtualenv)

You will run freqtrade in separated `virtual environment`

```bash
# create virtualenv in directory /freqtrade/.venv
python3 -m venv .venv

# run virtualenv
source .venv/bin/activate
```

### Install python dependencies

```bash
python3 -m pip install --upgrade pip
python3 -m pip install -r requirements.txt
# install freqtrade
python3 -m pip install -e .
```

[You are now ready](#you-are-ready) to run the bot.

### (Optional) Post-installation Tasks

!!! Note 
    If you run the bot on a server, you should consider using [Docker](docker_quickstart.md) or a terminal multiplexer like `screen` or [`tmux`](https://en.wikipedia.org/wiki/Tmux) to avoid that the bot is stopped on logout.

On Linux with software suite `systemd`, as an optional post-installation task, you may wish to setup the bot to run as a `systemd service` or configure it to send the log messages to the `syslog`/`rsyslog` or `journald` daemons. See [Advanced Logging](advanced-setup.md#advanced-logging) for details.

------

## Installation with Conda

Freqtrade can also be installed with Miniconda or Anaconda. We recommend using Miniconda as it's installation footprint is smaller. Conda will automatically prepare and manage the extensive library-dependencies of the Freqtrade program.

### What is Conda?

Conda is a package, dependency and environment manager for multiple programming languages: [conda docs](https://docs.conda.io/projects/conda/en/latest/index.html)

### Installation with conda

#### Install Conda

[Installing on linux](https://conda.io/projects/conda/en/latest/user-guide/install/linux.html#install-linux-silent)

[Installing on windows](https://conda.io/projects/conda/en/latest/user-guide/install/windows.html)

Answer all questions. After installation, it is mandatory to turn your terminal OFF and ON again.

#### Freqtrade download

Download and install freqtrade.

```bash
# download freqtrade
git clone https://github.com/freqtrade/freqtrade.git

# enter downloaded directory 'freqtrade'
cd freqtrade      
```

#### Freqtrade install: Conda Environment

```bash
conda create --name freqtrade python=3.12
```

!!! Note "Creating Conda Environment"
    The conda command `create -n` automatically installs all nested dependencies for the selected libraries, general structure of installation command is:

    ```bash
    # choose your own packages
    conda env create -n [name of the environment] [python version] [packages]
    ```

#### Enter/exit freqtrade environment

To check available environments, type

```bash
conda env list
```

Enter installed environment

```bash
# enter conda environment
conda activate freqtrade

# exit conda environment - don't do it now
conda deactivate
```

Install last python dependencies with pip

```bash
python3 -m pip install --upgrade pip
python3 -m pip install -r requirements.txt
python3 -m pip install -e .
```

Patch conda libta-lib (Linux only)

```bash
# Ensure that the environment is active!
conda activate freqtrade

cd build_helpers
bash install_ta-lib.sh ${CONDA_PREFIX} nosudo
```

[You are now ready](#you-are-ready) to run the bot.

### Important shortcuts

```bash
# list installed conda environments
conda env list

# activate base environment
conda activate

# activate freqtrade environment
conda activate freqtrade

#deactivate any conda environments
conda deactivate                              
```

### Further info on anaconda

!!! Info "New heavy packages"
    It may happen that creating a new Conda environment, populated with selected packages at the moment of creation takes less time than installing a large, heavy library or application, into previously set environment.

!!! Warning "pip install within conda"
    The documentation of conda says that pip should NOT be used within conda, because internal problems can occur.
    However, they are rare. [Anaconda Blogpost](https://www.anaconda.com/blog/using-pip-in-a-conda-environment)

    Nevertheless, that is why, the `conda-forge` channel is preferred:

    * more libraries are available (less need for `pip`)
    * `conda-forge` works better with `pip`
    * the libraries are newer

Happy trading!

-----

## You are ready

You've made it this far, so you have successfully installed freqtrade.

### Initialize the configuration

```bash
# Step 1 - Initialize user folder
freqtrade create-userdir --userdir user_data

# Step 2 - Create a new configuration file
freqtrade new-config --config user_data/config.json
```

You are ready to run, read [Bot Configuration](configuration.md), remember to start with `dry_run: True` and verify that everything is working.

To learn how to setup your configuration, please refer to the [Bot Configuration](configuration.md) documentation page.

### Start the Bot

```bash
freqtrade trade --config user_data/config.json --strategy SampleStrategy
```

!!! Warning
    You should read through the rest of the documentation, backtest the strategy you're going to use, and use dry-run before enabling trading with real money.

-----

## Troubleshooting

### Common problem: "command not found"

If you used (1)`Script` or (2)`Manual` installation, you need to run the bot in virtual environment. If you get error as below, make sure venv is active.

```bash
# if:
bash: freqtrade: command not found

# then activate your virtual environment
source ./.venv/bin/activate
```

### MacOS installation error

Newer versions of MacOS may have installation failed with errors like `error: command 'g++' failed with exit status 1`.

This error will require explicit installation of the SDK Headers, which are not installed by default in this version of MacOS.
For MacOS 10.14, this can be accomplished with the below command.

```bash
open /Library/Developer/CommandLineTools/Packages/macOS_SDK_headers_for_macOS_10.14.pkg
```

If this file is inexistent, then you're probably on a different version of MacOS, so you may need to consult the internet for specific resolution details.

---

## File: leverage.md

# Trading with Leverage

!!! Warning "Beta feature"
    This feature is still in it's testing phase. Should you notice something you think is wrong please let us know via Discord or via Github Issue.

!!! Note "Multiple bots on one account"
    You can't run 2 bots on the same account with leverage. For leveraged / margin trading, freqtrade assumes it's the only user of the account, and all liquidation levels are calculated based on this assumption.

!!! Danger "Trading with leverage is very risky"
    Do not trade with a leverage > 1 using a strategy that hasn't shown positive results in a live run using the spot market. Check the stoploss of your strategy. With a leverage of 2, a stoploss of 0.5 (50%) would be too low, and these trades would be liquidated before reaching that stoploss.
    We do not assume any responsibility for eventual losses that occur from using this software or this mode.

    Please only use advanced trading modes when you know how freqtrade (and your strategy) works.
    Also, never risk more than what you can afford to lose.

If you already have an existing strategy, please read the [strategy migration guide](strategy_migration.md#strategy-migration-between-v2-and-v3) to migrate your strategy from a freqtrade v2 strategy, to strategy of version 3 which can short and trade futures.

## Shorting

Shorting is not possible when trading with [`trading_mode`](#leverage-trading-modes) set to `spot`. To short trade, `trading_mode` must be set to `margin`(currently unavailable) or [`futures`](#futures), with [`margin_mode`](#margin-mode) set to `cross`(currently unavailable) or [`isolated`](#isolated-margin-mode)

For a strategy to short, the strategy class must set the class variable `can_short = True`

Please read [strategy customization](strategy-customization.md#entry-signal-rules) for instructions on how to set signals to enter and exit short trades.

## Understand `trading_mode`

The possible values are: `spot` (default), `margin`(*Currently unavailable*) or `futures`.

### Spot

Regular trading mode (low risk)

- Long trades only (No short trades).
- No leverage.
- No Liquidation.
- Profits gained/lost are equal to the change in value of the assets (minus trading fees).

### Leverage trading modes

With leverage, a trader borrows capital from the exchange. The capital must be re-payed fully to the exchange (potentially with interest), and the trader keeps any profits, or pays any losses, from any trades made using the borrowed capital.

Because the capital must always be re-payed, exchanges will **liquidate** (forcefully sell the traders assets) a trade made using borrowed capital when the total value of assets in the leverage account drops to a certain point (a point where the total value of losses is less than the value of the collateral that the trader actually owns in the leverage account), in order to ensure that the trader has enough capital to pay the borrowed assets back to the exchange. The exchange will also charge a **liquidation fee**, adding to the traders losses.

For this reason, **DO NOT TRADE WITH LEVERAGE IF YOU DON'T KNOW EXACTLY WHAT YOUR DOING. LEVERAGE TRADING IS HIGH RISK, AND CAN RESULT IN THE VALUE OF YOUR ASSETS DROPPING TO 0 VERY QUICKLY, WITH NO CHANCE OF INCREASING IN VALUE AGAIN.**

#### Margin (currently unavailable)

Trading occurs on the spot market, but the exchange lends currency to you in an amount equal to the chosen leverage. You pay the amount lent to you back to the exchange with interest, and your profits/losses are multiplied by the leverage specified.

#### Futures

Perpetual swaps (also known as Perpetual Futures) are contracts traded at a price that is closely tied to the underlying asset they are based off of (ex.). You are not trading the actual asset but instead are trading a derivative contract. Perpetual swap contracts can last indefinitely, in contrast to futures or option contracts.

In addition to the gains/losses from the change in price of the futures contract, traders also exchange _funding fees_, which are gains/losses worth an amount that is derived from the difference in price between the futures contract and the underlying asset. The difference in price between a futures contract and the underlying asset varies between exchanges.

To trade in futures markets, you'll have to set `trading_mode` to "futures".
You will also have to pick a "margin mode" (explanation below) - with freqtrade currently only supporting isolated margin.

``` json
"trading_mode": "futures",
"margin_mode": "isolated"
```

##### Pair namings

Freqtrade follows the [ccxt naming conventions for futures](https://docs.ccxt.com/#/README?id=perpetual-swap-perpetual-future).
A futures pair will therefore have the naming of `base/quote:settle` (e.g. `ETH/USDT:USDT`).

### Margin mode

On top of `trading_mode` - you will also have to configure your `margin_mode`.
While freqtrade currently only supports one margin mode, this will change, and by configuring it now you're all set for future updates.

The possible values are: `isolated`, or `cross`(*currently unavailable*).

#### Isolated margin mode

Each market(trading pair), keeps collateral in a separate account

``` json
"margin_mode": "isolated"
```

#### Cross margin mode

One account is used to share collateral between markets (trading pairs). Margin is taken from total account balance to avoid liquidation when needed.

``` json
"margin_mode": "cross"
```

Please read the [exchange specific notes](exchanges.md) for exchanges that support this mode and how they differ.

## Set leverage to use

Different strategies and risk profiles will require different levels of leverage.
While you could configure one static leverage value - freqtrade offers you the flexibility to adjust this via [strategy leverage callback](strategy-callbacks.md#leverage-callback) - which allows you to use different leverages by pair, or based on some other factor benefitting your strategy result.

If not implemented, leverage defaults to 1x (no leverage).

!!! Warning
    Higher leverage also equals higher risk - be sure you fully understand the implications of using leverage!

## Understand `liquidation_buffer`

*Defaults to `0.05`*

A ratio specifying how large of a safety net to place between the liquidation price and the stoploss to prevent a position from reaching the liquidation price.
This artificial liquidation price is calculated as:

`freqtrade_liquidation_price = liquidation_price ± (abs(open_rate - liquidation_price) * liquidation_buffer)`

- `±` = `+` for long trades
- `±` = `-` for short trades

Possible values are any floats between 0.0 and 0.99

**ex:** If a trade is entered at a price of 10 coin/USDT, and the liquidation price of this trade is 8 coin/USDT, then with `liquidation_buffer` set to `0.05` the minimum stoploss for this trade would be $8 + ((10 - 8) * 0.05) = 8 + 0.1 = 8.1$

!!! Danger "A `liquidation_buffer` of 0.0, or a low `liquidation_buffer` is likely to result in liquidations, and liquidation fees"
    Currently Freqtrade is able to calculate liquidation prices, but does not calculate liquidation fees. Setting your `liquidation_buffer` to 0.0, or using a low `liquidation_buffer` could result in your positions being liquidated. Freqtrade does not track liquidation fees, so liquidations will result in inaccurate profit/loss results for your bot. If you use a low `liquidation_buffer`, it is recommended to use `stoploss_on_exchange` if your exchange supports this.

## Unavailable funding rates

For futures data, exchanges commonly provide the futures candles, the marks, and the funding rates. However, it is common that whilst candles and marks might be available, the funding rates are not. This can affect backtesting timeranges, i.e. you may only be able to test recent timeranges and not earlier, experiencing the `No data found. Terminating.` error. To get around this, add the `futures_funding_rate` config option as listed in [configuration.md](configuration.md), and it is recommended that you set this to `0`, unless you know a given specific funding rate for your pair, exchange and timerange. Setting this to anything other than `0` can have drastic effects on your profit calculations within strategy, e.g. within the `custom_exit`, `custom_stoploss`, etc functions.

!!! Warning "This will mean your backtests are inaccurate."
    This will not overwrite funding rates that are available from the exchange, but bear in mind that setting a false funding rate will mean backtesting results will be inaccurate for historical timeranges where funding rates are not available.

### Developer

#### Margin mode

For shorts, the currency which pays the interest fee for the `borrowed` currency is purchased at the same time of the closing trade (This means that the amount purchased in short closing trades is greater than the amount sold in short opening trades).

For longs, the currency which pays the interest fee for the `borrowed` will already be owned by the user and does not need to be purchased. The interest is subtracted from the `close_value` of the trade.

All Fees are included in `current_profit` calculations during the trade.

#### Futures mode

Funding fees are either added or subtracted from the total amount of a trade

---

## File: lookahead-analysis.md

# Lookahead analysis

This page explains how to validate your strategy in terms of look ahead bias.

Checking look ahead bias is the bane of any strategy since it is sometimes very easy to introduce backtest bias -
but very hard to detect.

Backtesting initializes all timestamps at once and calculates all indicators in the beginning.
This means that if your indicators or entry/exit signals could look into future candles and falsify your backtest.

Lookahead-analysis requires historic data to be available.
To learn how to get data for the pairs and exchange you're interested in,
head over to the [Data Downloading](data-download.md) section of the documentation.

This command is built upon backtesting since it internally chains backtests and pokes at the strategy to provoke it to show look ahead bias.
This is done by not looking at the strategy itself - but at the results it returned.
The results are things like changed indicator-values and moved entries/exits compared to the full backtest.

You can use commands of [Backtesting](backtesting.md).
It also supports the lookahead-analysis of freqai strategies.

- `--cache` is forced to "none".
- `--max-open-trades` is forced to be at least equal to the number of pairs.
- `--dry-run-wallet` is forced to be basically infinite (1 billion).
- `--stake-amount` is forced to be a static 10000 (10k).
- `--enable-protections` is forced to be off.

Those are set to avoid users accidentally generating false positives.

## Lookahead-analysis command reference

--8<-- "commands/lookahead-analysis.md"

!!! Note ""
    The above Output was reduced to options `lookahead-analysis` adds on top of regular backtesting commands.

### Summary

Checks a given strategy for look ahead bias via lookahead-analysis
Look ahead bias means that the backtest uses data from future candles thereby not making it viable beyond backtesting
and producing false hopes for the one backtesting.

### Introduction

Many strategies - without the programmer knowing - have fallen prey to look ahead bias.

Any backtest will populate the full dataframe including all time stamps at the beginning.
If the programmer is not careful or oblivious how things work internally
(which sometimes can be really hard to find out) then it will just look into the future making the strategy amazing
but not realistic.

This command is made to try to verify the validity in the form of the aforementioned look ahead bias.

### How does the command work?

It will start with a backtest of all pairs to generate a baseline for indicators and entries/exits.
After the backtest ran, it will look if the `minimum-trade-amount` is met
and if not cancel the lookahead-analysis for this strategy.

After setting the baseline it will then do additional runs for every entry and exit separately.
When a verification-backtest is done, it will compare the indicators as the signal (either entry or exit) and report the bias.
After all signals have been verified or falsified a result-table will be generated for the user to see.

### Caveats

- `lookahead-analysis` can only verify / falsify the trades it calculated and verified.
If the strategy has many different signals / signal types, it's up to you to select appropriate parameters to ensure that all signals have triggered at least once. Not triggered signals will not have been verified.
This could lead to a false-negative (the strategy will then be reported as non-biased).
- `lookahead-analysis` has access to everything that backtesting has too.
Please don't provoke any configs like enabling position stacking.
If you decide to do so, then make doubly sure that you won't ever run out of `max_open_trades` amount and neither leftover money in your wallet.
- In the results table, the `biased_indicators` column will falsely flag FreqAI target indicators defined in `set_freqai_targets()` as biased. These are not biased and can safely be ignored.

---

## File: plotting.md

# Plotting

This page explains how to plot prices, indicators and profits.

!!! Warning "Deprecated"
    The commands described in this page (`plot-dataframe`, `plot-profit`) should be considered deprecated and are in maintenance mode.
    This is mostly for the performance problems even medium sized plots can cause, but also because "store a file and open it in a browser" isn't very intuitive from a UI perspective.

    While there are no immediate plans to remove them, they are not actively maintained - and may be removed short-term should major changes be required to keep them working.
    
    Please use [FreqUI](freq-ui.md) for plotting needs, which doesn't struggle with the same performance problems.

## Installation / Setup

Plotting modules use the Plotly library. You can install / upgrade this by running the following command:

``` bash
pip install -U -r requirements-plot.txt
```

## Plot price and indicators

The `freqtrade plot-dataframe` subcommand shows an interactive graph with three subplots:

* Main plot with candlesticks and indicators following price (sma/ema)
* Volume bars
* Additional indicators as specified by `--indicators2`

![plot-dataframe](assets/plot-dataframe.png)

Possible arguments:

--8<-- "commands/plot-dataframe.md"

Example:

``` bash
freqtrade plot-dataframe -p BTC/ETH --strategy AwesomeStrategy
```

The `-p/--pairs` argument can be used to specify pairs you would like to plot.

!!! Note
    The `freqtrade plot-dataframe` subcommand generates one plot-file per pair.

Specify custom indicators.
Use `--indicators1` for the main plot and `--indicators2` for the subplot below (if values are in a different range than prices).

``` bash
freqtrade plot-dataframe --strategy AwesomeStrategy -p BTC/ETH --indicators1 sma ema --indicators2 macd
```

### Further usage examples

To plot multiple pairs, separate them with a space:

``` bash
freqtrade plot-dataframe --strategy AwesomeStrategy -p BTC/ETH XRP/ETH
```

To plot a timerange (to zoom in)

``` bash
freqtrade plot-dataframe --strategy AwesomeStrategy -p BTC/ETH --timerange=20180801-20180805
```

To plot trades stored in a database use `--db-url` in combination with `--trade-source DB`:

``` bash
freqtrade plot-dataframe --strategy AwesomeStrategy --db-url sqlite:///tradesv3.dry_run.sqlite -p BTC/ETH --trade-source DB
```

To plot trades from a backtesting result, use `--export-filename <filename>`

``` bash
freqtrade plot-dataframe --strategy AwesomeStrategy --export-filename user_data/backtest_results/backtest-result.json -p BTC/ETH
```

### Plot dataframe basics

![plot-dataframe2](assets/plot-dataframe2.png)

The `plot-dataframe` subcommand requires backtesting data, a strategy and either a backtesting-results file or a database, containing trades corresponding to the strategy.

The resulting plot will have the following elements:

* Green triangles: Buy signals from the strategy. (Note: not every buy signal generates a trade, compare to cyan circles.)
* Red triangles: Sell signals from the strategy. (Also, not every sell signal terminates a trade, compare to red and green squares.)
* Cyan circles: Trade entry points.
* Red squares: Trade exit points for trades with loss or 0% profit.
* Green squares: Trade exit points for profitable trades.
* Indicators with values corresponding to the candle scale (e.g. SMA/EMA), as specified with `--indicators1`.
* Volume (bar chart at the bottom of the main chart).
* Indicators with values in different scales (e.g. MACD, RSI) below the volume bars, as specified with `--indicators2`.

!!! Note "Bollinger Bands"
    Bollinger bands are automatically added to the plot if the columns `bb_lowerband` and `bb_upperband` exist, and are painted as a light blue area spanning from the lower band to the upper band.

#### Advanced plot configuration

An advanced plot configuration can be specified in the strategy in the `plot_config` parameter.

Additional features when using `plot_config` include:

* Specify colors per indicator
* Specify additional subplots
* Specify indicator pairs to fill area in between

The sample plot configuration below specifies fixed colors for the indicators. Otherwise, consecutive plots may produce different color schemes each time, making comparisons difficult.
It also allows multiple subplots to display both MACD and RSI at the same time.

Plot type can be configured using `type` key. Possible types are:

* `scatter` corresponding to `plotly.graph_objects.Scatter` class (default).
* `bar` corresponding to `plotly.graph_objects.Bar` class.

Extra parameters to `plotly.graph_objects.*` constructor can be specified in `plotly` dict.

Sample configuration with inline comments explaining the process:

``` python
@property
def plot_config(self):
    """
        There are a lot of solutions how to build the return dictionary.
        The only important point is the return value.
        Example:
            plot_config = {'main_plot': {}, 'subplots': {}}

    """
    plot_config = {}
    plot_config['main_plot'] = {
        # Configuration for main plot indicators.
        # Assumes 2 parameters, emashort and emalong to be specified.
        f'ema_{self.emashort.value}': {'color': 'red'},
        f'ema_{self.emalong.value}': {'color': '#CCCCCC'},
        # By omitting color, a random color is selected.
        'sar': {},
        # fill area between senkou_a and senkou_b
        'senkou_a': {
            'color': 'green', #optional
            'fill_to': 'senkou_b',
            'fill_label': 'Ichimoku Cloud', #optional
            'fill_color': 'rgba(255,76,46,0.2)', #optional
        },
        # plot senkou_b, too. Not only the area to it.
        'senkou_b': {}
    }
    plot_config['subplots'] = {
         # Create subplot MACD
        "MACD": {
            'macd': {'color': 'blue', 'fill_to': 'macdhist'},
            'macdsignal': {'color': 'orange'},
            'macdhist': {'type': 'bar', 'plotly': {'opacity': 0.9}}
        },
        # Additional subplot RSI
        "RSI": {
            'rsi': {'color': 'red'}
        }
    }

    return plot_config
```

??? Note "As attribute (former method)"
    Assigning plot_config is also possible as Attribute (this used to be the default way).
    This has the disadvantage that strategy parameters are not available, preventing certain configurations from working.

    ``` python
        plot_config = {
            'main_plot': {
                # Configuration for main plot indicators.
                # Specifies `ema10` to be red, and `ema50` to be a shade of gray
                'ema10': {'color': 'red'},
                'ema50': {'color': '#CCCCCC'},
                # By omitting color, a random color is selected.
                'sar': {},
            # fill area between senkou_a and senkou_b
            'senkou_a': {
                'color': 'green', #optional
                'fill_to': 'senkou_b',
                'fill_label': 'Ichimoku Cloud', #optional
                'fill_color': 'rgba(255,76,46,0.2)', #optional
            },
            # plot senkou_b, too. Not only the area to it.
            'senkou_b': {}
            },
            'subplots': {
                # Create subplot MACD
                "MACD": {
                    'macd': {'color': 'blue', 'fill_to': 'macdhist'},
                    'macdsignal': {'color': 'orange'},
                    'macdhist': {'type': 'bar', 'plotly': {'opacity': 0.9}}
                },
                # Additional subplot RSI
                "RSI": {
                    'rsi': {'color': 'red'}
                }
            }
        }

    ```


!!! Note
    The above configuration assumes that `ema10`, `ema50`, `senkou_a`, `senkou_b`,
    `macd`, `macdsignal`, `macdhist` and `rsi` are columns in the DataFrame created by the strategy.

!!! Warning
    `plotly` arguments are only supported with plotly library and will not work with freq-ui.

!!! Note "Trade position adjustments"
    If `position_adjustment_enable` / `adjust_trade_position()` is used, the trade initial buy price is averaged over multiple orders and the trade start price will most likely appear outside the candle range.

## Plot profit

![plot-profit](assets/plot-profit.png)

The `plot-profit` subcommand shows an interactive graph with three plots:

* Average closing price for all pairs.
* The summarized profit made by backtesting.
Note that this is not the real-world profit, but more of an estimate.
* Profit for each individual pair.
* Parallelism of trades.
* Underwater (Periods of drawdown).

The first graph is good to get a grip of how the overall market progresses.

The second graph will show if your algorithm works or doesn't.
Perhaps you want an algorithm that steadily makes small profits, or one that acts less often, but makes big swings.
This graph will also highlight the start (and end) of the Max drawdown period.

The third graph can be useful to spot outliers, events in pairs that cause profit spikes.

The forth graph can help you analyze trade parallelism, showing how often max_open_trades have been maxed out.

Possible options for the `freqtrade plot-profit` subcommand:

--8<-- "commands/plot-profit.md"

The `-p/--pairs`  argument, can be used to limit the pairs that are considered for this calculation.

Examples:

Use custom backtest-export file

``` bash
freqtrade plot-profit  -p LTC/BTC --export-filename user_data/backtest_results/backtest-result.json
```

Use custom database

``` bash
freqtrade plot-profit  -p LTC/BTC --db-url sqlite:///tradesv3.sqlite --trade-source DB
```

``` bash
freqtrade --datadir user_data/data/binance_save/ plot-profit -p LTC/BTC
```

---

## File: plugins.md

# Plugins
--8<-- "includes/pairlists.md"
--8<-- "includes/protections.md"

---

## File: producer-consumer.md

# Producer / Consumer mode

freqtrade provides a mechanism whereby an instance (also called `consumer`) may listen to messages from an upstream freqtrade instance (also called `producer`) using the message websocket. Mainly, `analyzed_df` and `whitelist` messages. This allows the reuse of computed indicators (and signals) for pairs in multiple bots without needing to compute them multiple times.

See [Message Websocket](rest-api.md#message-websocket) in the Rest API docs for setting up the `api_server` configuration for your message websocket (this will be your producer).

!!! Note
    We strongly recommend to set `ws_token` to something random and known only to yourself to avoid unauthorized access to your bot.

## Configuration

Enable subscribing to an instance by adding the `external_message_consumer` section to the consumer's config file.

```json
{
    //...
   "external_message_consumer": {
        "enabled": true,
        "producers": [
            {
                "name": "default", // This can be any name you'd like, default is "default"
                "host": "127.0.0.1", // The host from your producer's api_server config
                "port": 8080, // The port from your producer's api_server config
                "secure": false, // Use a secure websockets connection, default false
                "ws_token": "sercet_Ws_t0ken" // The ws_token from your producer's api_server config
            }
        ],
        // The following configurations are optional, and usually not required
        // "wait_timeout": 300,
        // "ping_timeout": 10,
        // "sleep_time": 10,
        // "remove_entry_exit_signals": false,
        // "message_size_limit": 8
    }
    //...
}
```

|  Parameter | Description |
|------------|-------------|
| `enabled` | **Required.** Enable consumer mode. If set to false, all other settings in this section are ignored.<br>*Defaults to `false`.*<br> **Datatype:** boolean .
| `producers` | **Required.** List of producers <br> **Datatype:** Array.
| `producers.name` | **Required.** Name of this producer. This name must be used in calls to `get_producer_pairs()` and `get_producer_df()` if more than one producer is used.<br> **Datatype:** string
| `producers.host` | **Required.** The hostname or IP address from your producer.<br> **Datatype:** string
| `producers.port` | **Required.** The port matching the above host.<br>*Defaults to `8080`.*<br> **Datatype:** Integer
| `producers.secure` | **Optional.**  Use ssl in websockets connection. Default False.<br> **Datatype:** string
| `producers.ws_token` | **Required.**  `ws_token` as configured on the producer.<br> **Datatype:** string
| | **Optional settings**
| `wait_timeout` | Timeout until we ping again if no message is received. <br>*Defaults to `300`.*<br> **Datatype:** Integer - in seconds.
| `ping_timeout` | Ping timeout <br>*Defaults to `10`.*<br> **Datatype:** Integer - in seconds.
| `sleep_time` | Sleep time before retrying to connect.<br>*Defaults to `10`.*<br> **Datatype:** Integer - in seconds.
| `remove_entry_exit_signals` | Remove signal columns from the dataframe (set them to 0) on dataframe receipt.<br>*Defaults to `false`.*<br> **Datatype:** Boolean.
| `message_size_limit` | Size limit per message<br>*Defaults to `8`.*<br> **Datatype:** Integer - Megabytes.

Instead of (or as well as) calculating indicators in `populate_indicators()` the follower instance listens on the connection to a producer instance's messages (or multiple producer instances in advanced configurations) and requests the producer's most recently analyzed dataframes for each pair in the active whitelist.

A consumer instance will then have a full copy of the analyzed dataframes without the need to calculate them itself.

## Examples

### Example - Producer Strategy

A simple strategy with multiple indicators. No special considerations are required in the strategy itself.

```py
class ProducerStrategy(IStrategy):
    #...
    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        """
        Calculate indicators in the standard freqtrade way which can then be broadcast to other instances
        """
        dataframe['rsi'] = ta.RSI(dataframe)
        bollinger = qtpylib.bollinger_bands(qtpylib.typical_price(dataframe), window=20, stds=2)
        dataframe['bb_lowerband'] = bollinger['lower']
        dataframe['bb_middleband'] = bollinger['mid']
        dataframe['bb_upperband'] = bollinger['upper']
        dataframe['tema'] = ta.TEMA(dataframe, timeperiod=9)

        return dataframe

    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        """
        Populates the entry signal for the given dataframe
        """
        dataframe.loc[
            (
                (qtpylib.crossed_above(dataframe['rsi'], self.buy_rsi.value)) &
                (dataframe['tema'] <= dataframe['bb_middleband']) &
                (dataframe['tema'] > dataframe['tema'].shift(1)) &
                (dataframe['volume'] > 0)
            ),
            'enter_long'] = 1

        return dataframe
```

!!! Tip "FreqAI"
    You can use this to setup [FreqAI](freqai.md) on a powerful machine, while you run consumers on simple machines like raspberries, which can interpret the signals generated from the producer in different ways.


### Example - Consumer Strategy

A logically equivalent strategy which calculates no indicators itself, but will have the same analyzed dataframes available to make trading decisions based on the indicators calculated in the producer. In this example the consumer has the same entry criteria, however this is not necessary. The consumer may use different logic to enter/exit trades, and only use the indicators as specified.

```py
class ConsumerStrategy(IStrategy):
    #...
    process_only_new_candles = False # required for consumers

    _columns_to_expect = ['rsi_default', 'tema_default', 'bb_middleband_default']

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        """
        Use the websocket api to get pre-populated indicators from another freqtrade instance.
        Use `self.dp.get_producer_df(pair)` to get the dataframe
        """
        pair = metadata['pair']
        timeframe = self.timeframe

        producer_pairs = self.dp.get_producer_pairs()
        # You can specify which producer to get pairs from via:
        # self.dp.get_producer_pairs("my_other_producer")

        # This func returns the analyzed dataframe, and when it was analyzed
        producer_dataframe, _ = self.dp.get_producer_df(pair)
        # You can get other data if the producer makes it available:
        # self.dp.get_producer_df(
        #   pair,
        #   timeframe="1h",
        #   candle_type=CandleType.SPOT,
        #   producer_name="my_other_producer"
        # )

        if not producer_dataframe.empty:
            # If you plan on passing the producer's entry/exit signal directly,
            # specify ffill=False or it will have unintended results
            merged_dataframe = merge_informative_pair(dataframe, producer_dataframe,
                                                      timeframe, timeframe,
                                                      append_timeframe=False,
                                                      suffix="default")
            return merged_dataframe
        else:
            dataframe[self._columns_to_expect] = 0

        return dataframe

    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        """
        Populates the entry signal for the given dataframe
        """
        # Use the dataframe columns as if we calculated them ourselves
        dataframe.loc[
            (
                (qtpylib.crossed_above(dataframe['rsi_default'], self.buy_rsi.value)) &
                (dataframe['tema_default'] <= dataframe['bb_middleband_default']) &
                (dataframe['tema_default'] > dataframe['tema_default'].shift(1)) &
                (dataframe['volume'] > 0)
            ),
            'enter_long'] = 1

        return dataframe
```

!!! Tip "Using upstream signals"
    By setting `remove_entry_exit_signals=false`, you can also use the producer's signals directly. They should be available as `enter_long_default` (assuming `suffix="default"` was used) - and can be used as either signal directly, or as additional indicator.

---

## File: recursive-analysis.md

# Recursive analysis

This page explains how to validate your strategy for inaccuracies due to recursive issues with certain indicators.

A recursive formula defines any term of a sequence relative to its preceding term(s). An example of a recursive formula is a<sub>n</sub> = a<sub>n-1</sub> + b.

Why does this matter for Freqtrade? In backtesting, the bot will get full data of the pairs according to the timerange specified. But in a dry/live run, the bot will be limited by the amount of data each exchanges gives.

For example, to calculate a very basic indicator called `steps`, the first row's value is always 0, while the following rows' values are equal to the value of the previous row plus 1. If I were to calculate it using the latest 1000 candles, then the `steps` value of the first row is 0, and the `steps` value at the last closed candle is 999.

What happens if the calculation is using only the latest 500 candles? Then instead of 999, the `steps` value at last closed candle is 499. The difference of the value means your backtest result can differ from your dry/live run result.

The `recursive-analysis` command requires historic data to be available. To learn how to get data for the pairs and exchange you're interested in,
head over to the [Data Downloading](data-download.md) section of the documentation.

This command is built upon preparing different lengths of data and calculates indicators based on them.
This does not backtest the strategy itself, but rather only calculates the indicators. After calculating the indicators of different startup candle values (`startup_candle_count`) are done, the values of last rows across all specified `startup_candle_count` are compared to see how much variance they show compared to the base calculation.

Command settings:

- Use the `-p` option to set your desired pair to analyze. Since we are only looking at indicator values, using more than one pair is redundant. Preferably use a pair with a relatively high price and at least moderate volatility, such as BTC or ETH, to avoid rounding issues that can make the results inaccurate. If no pair is set on the command, the pair used for this analysis is the first pair in the whitelist.
- It is recommended to set a long timerange (at least 5000 candles) so that the initial indicators' calculation that is going to be used as a benchmark has very small or no recursive issues itself. For example, for a 5m timeframe, a timerange of 5000 candles would be equal to 18 days.
- `--cache` is forced to "none" to avoid loading previous indicators calculation automatically.

In addition to the recursive formula check, this command also carries out a simple lookahead bias check on the indicator values only. For a full lookahead check, use [Lookahead-analysis](lookahead-analysis.md).

## Recursive-analysis command reference

--8<-- "commands/recursive-analysis.md"

### Why are odd-numbered default startup candles used?

The default value for startup candles are odd numbers. When the bot fetches candle data from the exchange's API, the last candle is the one being checked by the bot and the rest of the data are the "startup candles".

For example, Binance allows 1000 candles per API call. When the bot receives 1000 candles, the last candle is the "current candle", and the preceding 999 candles are the "startup candles". By setting the startup candle count as 1000 instead of 999, the bot will try to fetch 1001 candles instead. The exchange API will then send candle data in a paginated form, i.e. in case of the Binance API, this will be two groups- one of length 1000 and another of length 1. This results in the bot thinking the strategy needs 1001 candles of data, and so it will download 2000 candles worth of data instead, which means there will be 1 "current candle" and 1999 "startup candles".

Furthermore, exchanges limit the number of consecutive bulk API calls, e.g. Binance allows 5 calls. In this case, only 5000 candles can be downloaded from Binance API without hitting the API rate limit, which means the max `startup_candle_count` you can have is 4999.

Please note that this candle limit may be changed in the future by the exchanges without any prior notice.

### How does the command work?

- Firstly an initial indicator calculation is carried out using the supplied timerange to generate a benchmark for indicator values.
- After setting the benchmark it will then carry out additional runs for each of the different startup candle count values.
- The command will then compare the indicator values at the last candle rows and report the differences in a table.

## Understanding the recursive-analysis output

This is an example of an output results table where at least one indicator has a recursive formula issue:

```
| indicators   | 20      | 40      | 80     | 100    | 150     | 300     | 999    |
|--------------+---------+---------+--------+--------+---------+---------+--------|
| rsi_30       | nan%    | -6.025% | 0.612% | 0.828% | -0.140% | 0.000%  | 0.000% |
| rsi_14       | 24.141% | -0.876% | 0.070% | 0.007% | -0.000% | -0.000% | -      |
```

The column headers indicate the different `startup_candle_count` used in the analysis. The values in the table indicate the variance of the calculated indicators compared to the benchmark value.

`nan%` means the value of that indicator cannot be calculated due to lack of data. In this example, you cannot calculate RSI with length 30 with just 21 candles (1 current candle + 20 startup candles).

Users should assess the table per indicator to decide if the specified `startup_candle_count` results in a sufficiently small variance so that the indicator does not have any effect on entries and/or exits.

As such, aiming for absolute zero variance (shown by `-` value) might not be the best option, because some indicators might require you to use such a long `startup_candle_count` to have zero variance.

## Caveats

- `recursive-analysis` will only calculate and compare the indicator values at the last row. The output table reports the percentage differences between the different startup candle count calculations and the original benchmark calculation. Whether it has any actual impact on your entries and exits is not included.
- The ideal scenario is that indicators will have no variance (or at least very close to 0%) despite the startup candle being varied. In reality, indicators such as EMA are using a recursive formula to calculate indicator values, so the goal is not necessarily to have zero percentage variance, but to have the variance low enough (and therefore `startup_candle_count` high enough) that the recursion inherent in the indicator will not have any real impact on trading decisions.
- `recursive-analysis` will only run calculations on `populate_indicators` and `@informative` decorator(s). If you put any indicator calculation on `populate_entry_trend` or `populate_exit_trend`, it won't be calculated.

---

## File: rest-api.md

# REST API

## FreqUI

FreqUI now has it's own dedicated [documentation section](freq-ui.md) - please refer to that section for all information regarding the FreqUI.

## Configuration

Enable the rest API by adding the api_server section to your configuration and setting `api_server.enabled` to `true`.

Sample configuration:

``` json
    "api_server": {
        "enabled": true,
        "listen_ip_address": "127.0.0.1",
        "listen_port": 8080,
        "verbosity": "error",
        "enable_openapi": false,
        "jwt_secret_key": "somethingrandom",
        "CORS_origins": [],
        "username": "Freqtrader",
        "password": "SuperSecret1!",
        "ws_token": "sercet_Ws_t0ken"
    },
```

!!! Danger "Security warning"
    By default, the configuration listens on localhost only (so it's not reachable from other systems). We strongly recommend to not expose this API to the internet and choose a strong, unique password, since others will potentially be able to control your bot.

??? Note "API/UI Access on a remote servers"
    If you're running on a VPS, you should consider using either a ssh tunnel, or setup a VPN (openVPN, wireguard) to connect to your bot.
    This will ensure that freqUI is not directly exposed to the internet, which is not recommended for security reasons (freqUI does not support https out of the box).
    Setup of these tools is not part of this tutorial, however many good tutorials can be found on the internet.

You can then access the API by going to `http://127.0.0.1:8080/api/v1/ping` in a browser to check if the API is running correctly.
This should return the response:

``` output
{"status":"pong"}
```

All other endpoints return sensitive info and require authentication and are therefore not available through a web browser.

### Security

To generate a secure password, best use a password manager, or use the below code.

``` python
import secrets
secrets.token_hex()
```

!!! Hint "JWT token"
    Use the same method to also generate a JWT secret key (`jwt_secret_key`).

!!! Danger "Password selection"
    Please make sure to select a very strong, unique password to protect your bot from unauthorized access.
    Also change `jwt_secret_key` to something random (no need to remember this, but it'll be used to encrypt your session, so it better be something unique!).

### Configuration with docker

If you run your bot using docker, you'll need to have the bot listen to incoming connections. The security is then handled by docker.

``` json
    "api_server": {
        "enabled": true,
        "listen_ip_address": "0.0.0.0",
        "listen_port": 8080,
        "username": "Freqtrader",
        "password": "SuperSecret1!",
        //...
    },
```

Make sure that the following 2 lines are available in your docker-compose file:

```yml
    ports:
      - "127.0.0.1:8080:8080"
```

!!! Danger "Security warning"
    By using `"8080:8080"` (or `"0.0.0.0:8080:8080"`) in the docker port mapping, the API will be available to everyone connecting to the server under the correct port, so others may be able to control your bot.
    This **may** be safe if you're running the bot in a secure environment (like your home network), but it's not recommended to expose the API to the internet.

## Rest API

### Consuming the API

We advise consuming the API by using the supported `freqtrade-client` package (also available as `scripts/rest_client.py`).

This command can be installed independent of any running freqtrade bot by using `pip install freqtrade-client`.

This module is designed to be lightweight, and only depends on the `requests` and `python-rapidjson` modules, skipping all heavy dependencies freqtrade otherwise needs.

``` bash
freqtrade-client <command> [optional parameters]
```

By default, the script assumes `127.0.0.1` (localhost) and port `8080` to be used, however you can specify a configuration file to override this behaviour.

#### Minimalistic client config

``` json
{
    "api_server": {
        "enabled": true,
        "listen_ip_address": "0.0.0.0",
        "listen_port": 8080,
        "username": "Freqtrader",
        "password": "SuperSecret1!",
        //...
    }
}
```

``` bash
freqtrade-client --config rest_config.json <command> [optional parameters]
```

Commands with many arguments may require keyword arguments (for clarity) - which can be provided as follows:

``` bash
freqtrade-client --config rest_config.json forceenter BTC/USDT long enter_tag=GutFeeling
```

This method will work for all arguments - check the "show" command for a list of available parameters.

??? Note "Programmatic use"
    The `freqtrade-client` package (installable independent of freqtrade) can be used in your own scripts to interact with the freqtrade API.
    to do so, please use the following:

    ``` python
    from freqtrade_client import FtRestClient
    

    client = FtRestClient(server_url, username, password)

    # Get the status of the bot
    ping = client.ping()
    print(ping)
    # ... 
    ```

    For a full list of available commands, please refer to the list below.

Possible commands can be listed from the rest-client script using the `help` command.

``` bash
freqtrade-client help
```

``` output
Possible commands:

available_pairs
	Return available pair (backtest data) based on timeframe / stake_currency selection

        :param timeframe: Only pairs with this timeframe available.
        :param stake_currency: Only pairs that include this timeframe

balance
	Get the account balance.

blacklist
	Show the current blacklist.

        :param add: List of coins to add (example: "BNB/BTC")

cancel_open_order
	Cancel open order for trade.

        :param trade_id: Cancels open orders for this trade.

count
	Return the amount of open trades.

daily
	Return the profits for each day, and amount of trades.

delete_lock
	Delete (disable) lock from the database.

        :param lock_id: ID for the lock to delete

delete_trade
	Delete trade from the database.
        Tries to close open orders. Requires manual handling of this asset on the exchange.

        :param trade_id: Deletes the trade with this ID from the database.

edge
	Return information about edge.

forcebuy
	Buy an asset.

        :param pair: Pair to buy (ETH/BTC)
        :param price: Optional - price to buy

forceenter
	Force entering a trade

        :param pair: Pair to buy (ETH/BTC)
        :param side: 'long' or 'short'
        :param price: Optional - price to buy

forceexit
	Force-exit a trade.

        :param tradeid: Id of the trade (can be received via status command)
        :param ordertype: Order type to use (must be market or limit)
        :param amount: Amount to sell. Full sell if not given

health
	Provides a quick health check of the running bot.

lock_add
    Manually lock a specific pair

        :param pair: Pair to lock
        :param until: Lock until this date (format "2024-03-30 16:00:00Z")
        :param side: Side to lock (long, short, *)
        :param reason: Reason for the lock        

locks
	Return current locks

logs
	Show latest logs.

        :param limit: Limits log messages to the last <limit> logs. No limit to get the entire log.

pair_candles
	Return live dataframe for <pair><timeframe>.

        :param pair: Pair to get data for
        :param timeframe: Only pairs with this timeframe available.
        :param limit: Limit result to the last n candles.

pair_history
	Return historic, analyzed dataframe

        :param pair: Pair to get data for
        :param timeframe: Only pairs with this timeframe available.
        :param strategy: Strategy to analyze and get values for
        :param timerange: Timerange to get data for (same format than --timerange endpoints)

performance
	Return the performance of the different coins.

ping
	simple ping

plot_config
	Return plot configuration if the strategy defines one.

profit
	Return the profit summary.

reload_config
	Reload configuration.

show_config
        Returns part of the configuration, relevant for trading operations.

start
	Start the bot if it's in the stopped state.

stats
	Return the stats report (durations, sell-reasons).

status
	Get the status of open trades.

stop
	Stop the bot. Use `start` to restart.

stopbuy
	Stop buying (but handle sells gracefully). Use `reload_config` to reset.

strategies
	Lists available strategies

strategy
	Get strategy details

        :param strategy: Strategy class name

sysinfo
	Provides system information (CPU, RAM usage)

trade
	Return specific trade

        :param trade_id: Specify which trade to get.

trades
	Return trades history, sorted by id

        :param limit: Limits trades to the X last trades. Max 500 trades.
        :param offset: Offset by this amount of trades.

list_open_trades_custom_data
    Return a dict containing open trades custom-datas

        :param key: str, optional - Key of the custom-data
        :param limit: Limits trades to X trades.
        :param offset: Offset by this amount of trades.

list_custom_data
    Return a dict containing custom-datas of a specified trade

        :param trade_id: int - ID of the trade
        :param key: str, optional - Key of the custom-data

version
	Return the version of the bot.

whitelist
	Show the current whitelist.


```

### Available endpoints

If you wish to call the REST API manually via another route, e.g. directly via `curl`, the table below shows the relevant URL endpoints and parameters.
All endpoints in the below table need to be prefixed with the base URL of the API, e.g. `http://127.0.0.1:8080/api/v1/` - so the command becomes `http://127.0.0.1:8080/api/v1/<command>`.

|  Endpoint | Method | Description / Parameters |
|-----------|--------|--------------------------|
| `/ping` | GET | Simple command testing the API Readiness - requires no authentication.
| `/start` | POST | Starts the trader.
| `/stop` | POST | Stops the trader.
| `/stopbuy` | POST | Stops the trader from opening new trades. Gracefully closes open trades according to their rules.
| `/reload_config` | POST | Reloads the configuration file.
| `/trades` | GET | List last trades. Limited to 500 trades per call.
| `/trade/<tradeid>` | GET | Get specific trade.<br/>*Params:*<br/>- `tradeid` (`int`)
| `/trades/<tradeid>` | DELETE | Remove trade from the database. Tries to close open orders. Requires manual handling of this trade on the exchange.<br/>*Params:*<br/>- `tradeid` (`int`) 
| `/trades/<tradeid>/open-order` | DELETE | Cancel open order for this trade.<br/>*Params:*<br/>- `tradeid` (`int`) 
| `/trades/<tradeid>/reload` | POST | Reload a trade from the Exchange. Only works in live, and can potentially help recover a trade that was manually sold on the exchange.<br/>*Params:*<br/>- `tradeid` (`int`) 
| `/show_config` | GET | Shows part of the current configuration with relevant settings to operation.
| `/logs` | GET | Shows last log messages.
| `/status` | GET | Lists all open trades.
| `/count` | GET | Displays number of trades used and available.
| `/entries` | GET | Shows profit statistics for each enter tags for given pair (or all pairs if pair isn't given). Pair is optional.<br/>*Params:*<br/>- `pair` (`str`) 
| `/exits` | GET | Shows profit statistics for each exit reasons for given pair (or all pairs if pair isn't given). Pair is optional.<br/>*Params:*<br/>- `pair` (`str`) 
| `/mix_tags` | GET | Shows profit statistics for each combinations of enter tag + exit reasons for given pair (or all pairs if pair isn't given). Pair is optional.<br/>*Params:*<br/>- `pair` (`str`) 
| `/locks` | GET | Displays currently locked pairs.
| `/locks` | POST | Locks a pair until "until". (Until will be rounded up to the nearest timeframe). Side is optional and is either `long` or `short` (default is `long`). Reason is optional.<br/>*Params:*<br/>- `<pair>` (`str`)<br/>- `<until>` (`datetime`)<br/>- `[side]` (`str`)<br/>- `[reason]` (`str`) 
| `/locks/<lockid>` | DELETE | Deletes (disables) the lock by id.<br/>*Params:*<br/>- `lockid` (`int`) 
| `/profit` | GET | Display a summary of your profit/loss from close trades and some stats about your performance.
| `/forceexit` | POST | Instantly exits the given trade (ignoring `minimum_roi`), using the given order type ("market" or "limit", uses your config setting if not specified), and the chosen amount (full sell if not specified). If `all` is supplied as the `tradeid`, then all currently open trades will be forced to exit.<br/>*Params:*<br/>- `<tradeid>` (`int` or `str`)<br/>- `<ordertype>` (`str`)<br/>- `[amount]` (`float`)
| `/forceenter` | POST | Instantly enters the given pair. Side is optional and is either `long` or `short` (default is `long`). Rate is optional. (`force_entry_enable` must be set to True)<br/>*Params:*<br/>- `<pair>` (`str`)<br/>- `<side>` (`str`)<br/>- `[rate]` (`float`)
| `/performance` | GET | Show performance of each finished trade grouped by pair.
| `/balance` | GET | Show account balance per currency.
| `/daily` | GET | Shows profit or loss per day, over the last n days (n defaults to 7).<br/>*Params:*<br/>- `<n>` (`int`)
| `/weekly` | GET | Shows profit or loss per week, over the last n days (n defaults to 4).<br/>*Params:*<br/>- `<n>` (`int`)
| `/monthly` | GET | Shows profit or loss per month, over the last n days (n defaults to 3).<br/>*Params:*<br/>- `<n>` (`int`)
| `/stats` | GET | Display a summary of profit / loss reasons as well as average holding times.
| `/whitelist` | GET | Show the current whitelist.
| `/blacklist` | GET | Show the current blacklist.
| `/blacklist` | POST | Adds the specified pair to the blacklist.<br/>*Params:*<br/>- `pair` (`str`)
| `/blacklist` | DELETE | Deletes the specified list of pairs from the blacklist.<br/>*Params:*<br/>- `[pair,pair]` (`list[str]`) 
| `/edge` | GET | Show validated pairs by Edge if it is enabled.
| `/pair_candles` | GET | Returns dataframe for a pair / timeframe combination while the bot is running. **Alpha**
| `/pair_candles` | POST | Returns dataframe for a pair / timeframe combination while the bot is running, filtered by a provided list of columns to return. **Alpha**<br/>*Params:*<br/>- `<column_list>` (`list[str]`)
| `/pair_history` | GET | Returns an analyzed dataframe for a given timerange, analyzed by a given strategy. **Alpha**
| `/pair_history` | POST | Returns an analyzed dataframe for a given timerange, analyzed by a given strategy, filtered by a provided list of columns to return. **Alpha**<br/>*Params:*<br/>- `<column_list>` (`list[str]`)
| `/plot_config` | GET | Get plot config from the strategy (or nothing if not configured). **Alpha**
| `/strategies` | GET | List strategies in strategy directory. **Alpha**
| `/strategy/<strategy>` | GET | Get specific Strategy content by strategy class name. **Alpha**<br/>*Params:*<br/>- `<strategy>` (`str`)
| `/available_pairs` | GET | List available backtest data. **Alpha**
| `/version` | GET | Show version.
| `/sysinfo` | GET | Show information about the system load.
| `/health` | GET | Show bot health (last bot loop).

!!! Warning "Alpha status"
    Endpoints labeled with *Alpha status* above may change at any time without notice.

### Message WebSocket

The API Server includes a websocket endpoint for subscribing to RPC messages from the freqtrade Bot.
This can be used to consume real-time data from your bot, such as entry/exit fill messages, whitelist changes, populated indicators for pairs, and more.

This is also used to setup [Producer/Consumer mode](producer-consumer.md) in Freqtrade.

Assuming your rest API is set to `127.0.0.1` on port `8080`, the endpoint is available at `http://localhost:8080/api/v1/message/ws`.

To access the websocket endpoint, the `ws_token` is required as a query parameter in the endpoint URL.

To generate a safe `ws_token` you can run the following code:

``` python
>>> import secrets
>>> secrets.token_urlsafe(25)
'hZ-y58LXyX_HZ8O1cJzVyN6ePWrLpNQv4Q'
```

You would then add that token under `ws_token` in your `api_server` config. Like so:

``` json
"api_server": {
    "enabled": true,
    "listen_ip_address": "127.0.0.1",
    "listen_port": 8080,
    "verbosity": "error",
    "enable_openapi": false,
    "jwt_secret_key": "somethingrandom",
    "CORS_origins": [],
    "username": "Freqtrader",
    "password": "SuperSecret1!",
    "ws_token": "hZ-y58LXyX_HZ8O1cJzVyN6ePWrLpNQv4Q" // <-----
},
```

You can now connect to the endpoint at `http://localhost:8080/api/v1/message/ws?token=hZ-y58LXyX_HZ8O1cJzVyN6ePWrLpNQv4Q`.

!!! Danger "Reuse of example tokens"
    Please do not use the above example token. To make sure you are secure, generate a completely new token.

#### Using the WebSocket

Once connected to the WebSocket, the bot will broadcast RPC messages to anyone who is subscribed to them. To subscribe to a list of messages, you must send a JSON request through the WebSocket like the one below. The `data` key must be a list of message type strings.

``` json
{
  "type": "subscribe",
  "data": ["whitelist", "analyzed_df"] // A list of string message types
}
```

For a list of message types, please refer to the RPCMessageType enum in `freqtrade/enums/rpcmessagetype.py`

Now anytime those types of RPC messages are sent in the bot, you will receive them through the WebSocket as long as the connection is active. They typically take the same form as the request:

``` json
{
  "type": "analyzed_df",
  "data": {
      "key": ["NEO/BTC", "5m", "spot"],
      "df": {}, // The dataframe
      "la": "2022-09-08 22:14:41.457786+00:00"
  }
}
```

#### Reverse Proxy setup

When using [Nginx](https://nginx.org/en/docs/), the following configuration is required for WebSockets to work (Note this configuration is incomplete, it's missing some information and can not be used as is):

Please make sure to replace `<freqtrade_listen_ip>` (and the subsequent port) with the IP and Port matching your configuration/setup.

```
http {
    map $http_upgrade $connection_upgrade {
        default upgrade;
        '' close;
    }

    #...

    server {
        #...

        location / {
            proxy_http_version 1.1;
            proxy_pass http://<freqtrade_listen_ip>:8080;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection $connection_upgrade;
            proxy_set_header Host $host;
        }
    }
}
```

To properly configure your reverse proxy (securely), please consult it's documentation for proxying websockets.

- **Traefik**: Traefik supports websockets out of the box, see the [documentation](https://doc.traefik.io/traefik/)
- **Caddy**: Caddy v2 supports websockets out of the box, see the [documentation](https://caddyserver.com/docs/v2-upgrade#proxy)

!!! Tip "SSL certificates"
    You can use tools like certbot to setup ssl certificates to access your bot's UI through encrypted connection by using any of the above reverse proxies.
    While this will protect your data in transit, we do not recommend to run the freqtrade API outside of your private network (VPN, SSH tunnel).

### OpenAPI interface

To enable the builtin openAPI interface (Swagger UI), specify `"enable_openapi": true` in the api_server configuration.
This will enable the Swagger UI at the `/docs` endpoint. By default, that's running at http://localhost:8080/docs - but it'll depend on your settings.

### Advanced API usage using JWT tokens

!!! Note
    The below should be done in an application (a Freqtrade REST API client, which fetches info via API), and is not intended to be used on a regular basis.

Freqtrade's REST API also offers JWT (JSON Web Tokens).
You can login using the following command, and subsequently use the resulting access_token.

``` bash
> curl -X POST --user Freqtrader http://localhost:8080/api/v1/token/login
{"access_token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE1ODkxMTk2ODEsIm5iZiI6MTU4OTExOTY4MSwianRpIjoiMmEwYmY0NWUtMjhmOS00YTUzLTlmNzItMmM5ZWVlYThkNzc2IiwiZXhwIjoxNTg5MTIwNTgxLCJpZGVudGl0eSI6eyJ1IjoiRnJlcXRyYWRlciJ9LCJmcmVzaCI6ZmFsc2UsInR5cGUiOiJhY2Nlc3MifQ.qt6MAXYIa-l556OM7arBvYJ0SDI9J8bIk3_glDujF5g","refresh_token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE1ODkxMTk2ODEsIm5iZiI6MTU4OTExOTY4MSwianRpIjoiZWQ1ZWI3YjAtYjMwMy00YzAyLTg2N2MtNWViMjIxNWQ2YTMxIiwiZXhwIjoxNTkxNzExNjgxLCJpZGVudGl0eSI6eyJ1IjoiRnJlcXRyYWRlciJ9LCJ0eXBlIjoicmVmcmVzaCJ9.d1AT_jYICyTAjD0fiQAr52rkRqtxCjUGEMwlNuuzgNQ"}

> access_token="eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE1ODkxMTk2ODEsIm5iZiI6MTU4OTExOTY4MSwianRpIjoiMmEwYmY0NWUtMjhmOS00YTUzLTlmNzItMmM5ZWVlYThkNzc2IiwiZXhwIjoxNTg5MTIwNTgxLCJpZGVudGl0eSI6eyJ1IjoiRnJlcXRyYWRlciJ9LCJmcmVzaCI6ZmFsc2UsInR5cGUiOiJhY2Nlc3MifQ.qt6MAXYIa-l556OM7arBvYJ0SDI9J8bIk3_glDujF5g"
# Use access_token for authentication
> curl -X GET --header "Authorization: Bearer ${access_token}" http://localhost:8080/api/v1/count

```

Since the access token has a short timeout (15 min) - the `token/refresh` request should be used periodically to get a fresh access token:

``` bash
> curl -X POST --header "Authorization: Bearer ${refresh_token}"http://localhost:8080/api/v1/token/refresh
{"access_token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE1ODkxMTk5NzQsIm5iZiI6MTU4OTExOTk3NCwianRpIjoiMDBjNTlhMWUtMjBmYS00ZTk0LTliZjAtNWQwNTg2MTdiZDIyIiwiZXhwIjoxNTg5MTIwODc0LCJpZGVudGl0eSI6eyJ1IjoiRnJlcXRyYWRlciJ9LCJmcmVzaCI6ZmFsc2UsInR5cGUiOiJhY2Nlc3MifQ.1seHlII3WprjjclY6DpRhen0rqdF4j6jbvxIhUFaSbs"}
```

--8<-- "includes/cors.md"

---

## File: sql_cheatsheet.md

# SQL Helper

This page contains some help if you want to query your sqlite db.

!!! Tip "Other Database systems"
    To use other Database Systems like PostgreSQL or MariaDB, you can use the same queries, but you need to use the respective client for the database system. [Click here](advanced-setup.md#use-a-different-database-system) to learn how to setup a different database system with freqtrade.

!!! Warning
    If you are not familiar with SQL, you should be very careful when running queries on your database.  
    Always make sure to have a backup of your database before running any queries.

## Install sqlite3

Sqlite3 is a terminal based sqlite application.
Feel free to use a visual Database editor like SqliteBrowser if you feel more comfortable with that.

### Ubuntu/Debian installation

```bash
sudo apt-get install sqlite3
```

### Using sqlite3 via docker

The freqtrade docker image does contain sqlite3, so you can edit the database without having to install anything on the host system.

``` bash
docker compose exec freqtrade /bin/bash
sqlite3 <database-file>.sqlite
```

## Open the DB

```bash
sqlite3
.open <filepath>
```

## Table structure

### List tables

```bash
.tables
```

### Display table structure

```bash
.schema <table_name>
```

### Get all trades in the table

```sql
SELECT * FROM trades;
```

## Destructive queries

Queries that write to the database.
These queries should usually not be necessary as freqtrade tries to handle all database operations itself - or exposes them via API or telegram commands.

!!! Warning
    Please make sure you have a backup of your database before running any of the below queries.

!!! Danger
    You should also **never** run any writing query (`update`, `insert`, `delete`) while a bot is connected to the database.
    This can and will lead to data corruption - most likely, without the possibility of recovery.

### Fix trade still open after a manual exit on the exchange

!!! Warning
    Manually selling a pair on the exchange will not be detected by the bot and it will try to sell anyway. Whenever possible, /forceexit <tradeid> should be used to accomplish the same thing.  
    It is strongly advised to backup your database file before making any manual changes.

!!! Note
    This should not be necessary after /forceexit, as force_exit orders are closed automatically by the bot on the next iteration.

```sql
UPDATE trades
SET is_open=0,
  close_date=<close_date>,
  close_rate=<close_rate>,
  close_profit = close_rate / open_rate - 1,
  close_profit_abs = (amount * <close_rate> * (1 - fee_close) - (amount * (open_rate * (1 - fee_open)))),
  exit_reason=<exit_reason>
WHERE id=<trade_ID_to_update>;
```

#### Example

```sql
UPDATE trades
SET is_open=0,
  close_date='2020-06-20 03:08:45.103418',
  close_rate=0.19638016,
  close_profit=0.0496,
  close_profit_abs = (amount * 0.19638016 * (1 - fee_close) - (amount * (open_rate * (1 - fee_open)))),
  exit_reason='force_exit'  
WHERE id=31;
```

### Remove trade from the database

!!! Tip "Use RPC Methods to delete trades"
    Consider using `/delete <tradeid>` via telegram or rest API. That's the recommended way to deleting trades.

If you'd still like to remove a trade from the database directly, you can use the below query.

!!! Danger
    Some systems (Ubuntu) disable foreign keys in their sqlite3 packaging. When using sqlite - please ensure that foreign keys are on by running `PRAGMA foreign_keys = ON` before the above query.

```sql
DELETE FROM trades WHERE id = <tradeid>;

DELETE FROM trades WHERE id = 31;
```

!!! Warning
    This will remove this trade from the database. Please make sure you got the correct id and **NEVER** run this query without the `where` clause.

---

## File: stoploss.md

# Stop Loss

The `stoploss` configuration parameter is loss as ratio that should trigger a sale.
For example, value `-0.10` will cause immediate sell if the profit dips below -10% for a given trade. This parameter is optional.
Stoploss calculations do include fees, so a stoploss of -10% is placed exactly 10% below the entry point.

Most of the strategy files already include the optimal `stoploss` value.

!!! Info
    All stoploss properties mentioned in this file can be set in the Strategy, or in the configuration.  
    <ins>Configuration values will override the strategy values.</ins>

## Stop Loss On-Exchange/Freqtrade

Those stoploss modes can be *on exchange* or *off exchange*.

These modes can be configured with these values:

``` python
    'emergency_exit': 'market',
    'stoploss_on_exchange': False
    'stoploss_on_exchange_interval': 60,
    'stoploss_on_exchange_limit_ratio': 0.99
```

Stoploss on exchange is only supported for the following exchanges, and not all exchanges support both stop-limit and stop-market.
The Order-type will be ignored if only one mode is available.

| Exchange | stop-loss type |
|----------|-------------|
| Binance  | limit |
| Binance Futures  | market, limit |
| Bingx    | market, limit |
| HTX      | limit |
| kraken   | market, limit |
| Gate     | limit |
| Okx      | limit |
| Kucoin   | stop-limit, stop-market|
| Hyperliquid (futures only)   | limit |

!!! Note "Tight stoploss"
    <ins>Do not set too low/tight stoploss value when using stop loss on exchange!</ins>  
    If set to low/tight you will have greater risk of missing fill on the order and stoploss will not work.

### stoploss_on_exchange and stoploss_on_exchange_limit_ratio

Enable or Disable stop loss on exchange.
If the stoploss is *on exchange* it means a stoploss limit order is placed on the exchange immediately after buy order fills. This will protect you against sudden crashes in market, as the order execution happens purely within the exchange, and has no potential network overhead.

If `stoploss_on_exchange` uses limit orders, the exchange needs 2 prices, the stoploss_price and the Limit price.  
`stoploss` defines the stop-price where the limit order is placed - and limit should be slightly below this.  
If an exchange supports both limit and market stoploss orders, then the value of `stoploss` will be used to determine the stoploss type.  

Calculation example: we bought the asset at 100\$.  
Stop-price is 95\$, then limit would be `95 * 0.99 = 94.05$` - so the limit order fill can happen between 95$ and 94.05$.  

For example, assuming the stoploss is on exchange, and trailing stoploss is enabled, and the market is going up, then the bot automatically cancels the previous stoploss order and puts a new one with a stop value higher than the previous stoploss order.

!!! Note
    If `stoploss_on_exchange` is enabled and the stoploss is cancelled manually on the exchange, then the bot will create a new stoploss order.

### stoploss_on_exchange_interval

In case of stoploss on exchange there is another parameter called `stoploss_on_exchange_interval`. This configures the interval in seconds at which the bot will check the stoploss and update it if necessary.  
The bot cannot do these every 5 seconds (at each iteration), otherwise it would get banned by the exchange.
So this parameter will tell the bot how often it should update the stoploss order. The default value is 60 (1 minute).
This same logic will reapply a stoploss order on the exchange should you cancel it accidentally.

### stoploss_price_type

!!! Warning "Only applies to futures"
    `stoploss_price_type` only applies to futures markets (on exchanges where it's available).
    Freqtrade will perform a validation of this setting on startup, failing to start if an invalid setting for your exchange has been selected.
    Supported price types are gonna differs between each exchanges. Please check with your exchange on which price types it supports.

Stoploss on exchange on futures markets can trigger on different price types.
The naming for these prices in exchange terminology often varies, but is usually something around "last" (or "contract price" ), "mark" and "index".

Acceptable values for this setting are `"last"`, `"mark"` and `"index"` - which freqtrade will transfer automatically to the corresponding API type, and place the [stoploss on exchange](#stoploss_on_exchange-and-stoploss_on_exchange_limit_ratio) order correspondingly.

### force_exit

`force_exit` is an optional value, which defaults to the same value as `exit` and is used when sending a `/forceexit` command from Telegram or from the Rest API.

### force_entry

`force_entry` is an optional value, which defaults to the same value as `entry` and is used when sending a `/forceentry` command from Telegram or from the Rest API.

### emergency_exit

`emergency_exit` is an optional value, which defaults to `market` and is used when creating stop loss on exchange orders fails.
The below is the default which is used if not changed in strategy or configuration file.

Example from strategy file:

``` python
order_types = {
    "entry": "limit",
    "exit": "limit",
    "emergency_exit": "market",
    "stoploss": "market",
    "stoploss_on_exchange": True,
    "stoploss_on_exchange_interval": 60,
    "stoploss_on_exchange_limit_ratio": 0.99
}
```

## Stop Loss Types

At this stage the bot contains the following stoploss support modes:

1. Static stop loss.
2. Trailing stop loss.
3. Trailing stop loss, custom positive loss.
4. Trailing stop loss only once the trade has reached a certain offset.
5. [Custom stoploss function](strategy-callbacks.md#custom-stoploss)

### Static Stop Loss

This is very simple, you define a stop loss of x (as a ratio of price, i.e. x * 100% of price). This will try to sell the asset once the loss exceeds the defined loss.

Example of stop loss:

``` python
    stoploss = -0.10
```

For example, simplified math:

* the bot buys an asset at a price of 100$
* the stop loss is defined at -10%
* the stop loss would get triggered once the asset drops below 90$

### Trailing Stop Loss

The initial value for this is `stoploss`, just as you would define your static Stop loss.
To enable trailing stoploss:

``` python
    stoploss = -0.10
    trailing_stop = True
```

This will now activate an algorithm, which automatically moves the stop loss up every time the price of your asset increases.

For example, simplified math:

* the bot buys an asset at a price of 100$
* the stop loss is defined at -10%
* the stop loss would get triggered once the asset drops below 90$
* assuming the asset now increases to 102$
* the stop loss will now be -10% of 102$ = 91.8$
* now the asset drops in value to 101\$, the stop loss will still be 91.8$ and would trigger at 91.8$.

In summary: The stoploss will be adjusted to be always be -10% of the highest observed price.

### Trailing stop loss, different positive loss

You could also have a default stop loss when you are in the red with your buy (buy - fee), but once you hit a positive result (or an offset you define) the system will utilize a new stop loss, with a different value.
For example, your default stop loss is -10%, but once you have reached profitability (example 0.1%) a different trailing stoploss will be used.

!!! Note
    If you want the stoploss to only be changed when you break even of making a profit (what most users want) please refer to next section with [offset enabled](#trailing-stop-loss-only-once-the-trade-has-reached-a-certain-offset).

Both values require `trailing_stop` to be set to true and `trailing_stop_positive` with a value.

``` python
    stoploss = -0.10
    trailing_stop = True
    trailing_stop_positive = 0.02
    trailing_stop_positive_offset = 0.0
    trailing_only_offset_is_reached = False  # Default - not necessary for this example
```

For example, simplified math:

* the bot buys an asset at a price of 100$
* the stop loss is defined at -10%
* the stop loss would get triggered once the asset drops below 90$
* assuming the asset now increases to 102$
* the stop loss will now be -2% of 102$ = 99.96$ (99.96$ stop loss will be locked in and will follow asset price increments with -2%)
* now the asset drops in value to 101\$, the stop loss will still be 99.96$ and would trigger at 99.96$

The 0.02 would translate to a -2% stop loss.
Before this, `stoploss` is used for the trailing stoploss.

!!! Tip "Use an offset to change your stoploss"
    Use `trailing_stop_positive_offset` to ensure that your new trailing stoploss will be in profit by setting `trailing_stop_positive_offset` higher than `trailing_stop_positive`. Your first new stoploss value will then already have locked in profits.

    Example with simplified math:

    ``` python
        stoploss = -0.10
        trailing_stop = True
        trailing_stop_positive = 0.02
        trailing_stop_positive_offset = 0.03
    ```

    * the bot buys an asset at a price of 100$
    * the stop loss is defined at -10%, so the stop loss would get triggered once the asset drops below 90$
    * assuming the asset now increases to 102$
    * the stoploss will now be at 91.8$ - 10% below the highest observed rate
    * assuming the asset now increases to 103.5$ (above the offset configured)
    * the stop loss will now be -2% of 103.5$ = 101.43$
    * now the asset drops in value to 102\$, the stop loss will still be 101.43$ and would trigger once price breaks below 101.43$

### Trailing stop loss only once the trade has reached a certain offset

You can also keep a static stoploss until the offset is reached, and then trail the trade to take profits once the market turns.

If `trailing_only_offset_is_reached = True` then the trailing stoploss is only activated once the offset is reached. Until then, the stoploss remains at the configured `stoploss` and is not trailing.
Leaving this value as `trailing_only_offset_is_reached=False` will allow the trailing stoploss to start trailing as soon as the asset price increases above the initial entry price.

This option can be used with or without `trailing_stop_positive`, but uses `trailing_stop_positive_offset` as offset.

Configuration (offset is buy-price + 3%):

``` python
    stoploss = -0.10
    trailing_stop = True
    trailing_stop_positive = 0.02
    trailing_stop_positive_offset = 0.03
    trailing_only_offset_is_reached = True
```

For example, simplified math:

* the bot buys an asset at a price of 100$
* the stop loss is defined at -10%
* the stop loss would get triggered once the asset drops below 90$
* stoploss will remain at 90$ unless asset increases to or above the configured offset
* assuming the asset now increases to 103$ (where we have the offset configured)
* the stop loss will now be -2% of 103$ = 100.94$
* now the asset drops in value to 101\$, the stop loss will still be 100.94$ and would trigger at 100.94$

!!! Tip
    Make sure to have this value (`trailing_stop_positive_offset`) lower than minimal ROI, otherwise minimal ROI will apply first and sell the trade.

## Stoploss and Leverage

Stoploss should be thought of as "risk on this trade" - so a stoploss of 10% on a 100$ trade means you are willing to lose 10$ (10%) on this trade - which would trigger if the price moves 10% to the downside.

When using leverage, the same principle is applied - with stoploss defining the risk on the trade (the amount you are willing to lose).

Therefore, a stoploss of 10% on a 10x trade would trigger on a 1% price move.
If your stake amount (own capital) was 100$ - this trade would be 1000$ at 10x (after leverage).
If price moves 1% - you've lost 10$ of your own capital - therefore stoploss will trigger in this case.

Make sure to be aware of this, and avoid using too tight stoploss (at 10x leverage, 10% risk may be too little to allow the trade to "breath" a little).

## Changing stoploss on open trades

A stoploss on an open trade can be changed by changing the value in the configuration or strategy and use the `/reload_config` command (alternatively, completely stopping and restarting the bot also works).

The new stoploss value will be applied to open trades (and corresponding log-messages will be generated).

### Limitations

Stoploss values cannot be changed if `trailing_stop` is enabled and the stoploss has already been adjusted, or if [Edge](edge.md) is enabled (since Edge would recalculate stoploss based on the current market situation).

---

## File: strategy-101.md

# Freqtrade Strategies 101: A Quick Start for Strategy Development

For the purposes of this quick start, we are assuming you are familiar with the basics of trading, and have read the 
[Freqtrade basics](bot-basics.md) page.

## Required Knowledge

A strategy in Freqtrade is a Python class that defines the logic for buying and selling cryptocurrency `assets`.

Assets are defined as `pairs`, which represent the `coin` and the `stake`. The coin is the asset you are trading using another currency as the stake.

Data is supplied by the exchange in the form of `candles`, which are made up of a six values: `date`, `open`, `high`, `low`, `close` and `volume`.

`Technical analysis` functions analyse the candle data using various computational and statistical formulae, and produce secondary values called `indicators`.

Indicators are analysed on the asset pair candles to generate `signals`.

Signals are turned into `orders` on a cryptocurrency `exchange`, i.e. `trades`.

We use the terms `entry` and `exit` instead of `buying` and `selling` because Freqtrade supports both `long` and `short` trades.

- **long**: You buy the coin based on a stake, e.g. buying the coin BTC using USDT as your stake, and you make a profit by selling the coin at a higher rate than you paid for. In long trades, profits are made by the coin value going up versus the stake.
- **short**: You borrow capital from the exchange in the form of the coin, and you pay back the stake value of the coin later. In short trades profits are made by the coin value going down versus the stake (you pay the loan off at a lower rate).

Whilst Freqtrade supports spot and futures markets for certain exchanges, for simplicity we will focus on spot (long) trades only.

## Structure of a Basic Strategy

### Main dataframe

Freqtrade strategies use a tabular data structure with rows and columns known as a `dataframe` to generate signals to enter and exit trades.

Each pair in your configured pairlist has its own dataframe. Dataframes are indexed by the `date` column, e.g. `2024-06-31 12:00`.

The next 5 columns represent the `open`, `high`, `low`, `close` and `volume` (OHLCV) data.

### Populate indicator values

The `populate_indicators` function adds columns to the dataframe that represent the technical analysis indicator values.

Examples of common indicators include Relative Strength Index, Bollinger Bands, Money Flow Index, Moving Average, and Average True Range.

Columns are added to the dataframe by calling technical analysis functions, e.g. ta-lib's RSI function `ta.RSI()`, and assigning them to a column name, e.g. `rsi`

```python
dataframe['rsi'] = ta.RSI(dataframe)
```

??? Hint "Technical Analysis libraries"
    Different libraries work in different ways to generate indicator values. Please check the documentation of each library to understand
    how to integrate it into your strategy. You can also check the [Freqtrade example strategies](https://github.com/freqtrade/freqtrade-strategies) to give you ideas.

### Populate entry signals

The `populate_entry_trend` function defines conditions for an entry signal.

The dataframe column `enter_long` is added to the dataframe, and when a value of `1` is in this column, Freqtrade sees an entry signal.

??? Hint "Shorting"
    To enter short trades, use the `enter_short` column.

### Populate exit signals

The `populate_exit_trend` function defines conditions for an exit signal.

The dataframe column `exit_long` is added to the dataframe, and when a value of `1` is in this column, Freqtrade sees an exit signal.

??? Hint "Shorting"
    To exit short trades, use the `exit_short` column.

## A simple strategy

Here is a minimal example of a Freqtrade strategy:

```python
from freqtrade.strategy import IStrategy
from pandas import DataFrame
import talib.abstract as ta

class MyStrategy(IStrategy):

    timeframe = '15m'

    # set the initial stoploss to -10%
    stoploss = -0.10

    # exit profitable positions at any time when the profit is greater than 1%
    minimal_roi = {"0": 0.01}

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        # generate values for technical analysis indicators
        dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)

        return dataframe

    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        # generate entry signals based on indicator values
        dataframe.loc[
            (dataframe['rsi'] < 30),
            'enter_long'] = 1

        return dataframe

    def populate_exit_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        # generate exit signals based on indicator values
        dataframe.loc[
            (dataframe['rsi'] > 70),
            'exit_long'] = 1

        return dataframe
```

## Making trades

When a signal is found (a `1` in an entry or exit column), Freqtrade will attempt to make an order, i.e. a `trade` or `position`.

Each new trade position takes up a `slot`. Slots represent the maximum number of concurrent new trades that can be opened.

The number of slots is defined by the `max_open_trades` [configuration](configuration.md) option.

However, there can be a range of scenarios where generating a signal does not always create a trade order. These include:

- not enough remaining stake to buy an asset, or funds in your wallet to sell an asset (including any fees)
- not enough remaining free slots for a new trade to be opened (the number of positions you have open equals the `max_open_trades` option)
- there is already an open trade for a pair (Freqtrade cannot stack positions - however it can [adjust existing positions](strategy-callbacks.md#adjust-trade-position))
- if an entry and exit signal is present on the same candle, they are considered as [colliding](strategy-customization.md#colliding-signals), and no order will be raised
- the strategy actively rejects the trade order due to logic you specify by using one of the relevant [entry](strategy-callbacks.md#trade-entry-buy-order-confirmation) or [exit](strategy-callbacks.md#trade-exit-sell-order-confirmation) callbacks

Read through the [strategy customization](strategy-customization.md) documentation for more details.

## Backtesting and forward testing

Strategy development can be a long and frustrating process, as turning our human "gut instincts" into a working computer-controlled
("algo") strategy is not always straightforward.

Therefore a strategy should be tested to verify that it is going to work as intended.

Freqtrade has two testing modes:

- **backtesting**: using historical data that you [download from an exchange](data-download.md), backtesting is a quick way to assess performance of a strategy. However, it can be very easy to distort results so a strategy will look a lot more profitable than it really is. Check the [backtesting documentation](backtesting.md) for more information.
- **dry run**: often referred to as _forward testing_, dry runs use real time data from the exchange. However, any signals that would result in trades are tracked as normal by Freqtrade, but do not have any trades opened on the exchange itself. Forward testing runs in real time, so whilst it takes longer to get results it is a much more reliable indicator of **potential** performance than backtesting.

Dry runs are enabled by setting `dry_run` to true in your [configuration](configuration.md#using-dry-run-mode).

!!! Warning "Backtests can be very inaccurate"
    There are many reasons why backtest results may not match reality. Please check the [backtesting assumptions](backtesting.md#assumptions-made-by-backtesting) and [common strategy mistakes](strategy-customization.md#common-mistakes-when-developing-strategies) documentation.
    Some websites that list and rank Freqtrade strategies show impressive backtest results. Do not assume these results are achieveable or realistic.

??? Hint "Useful commands"
    Freqtrade includes two useful commands to check for basic flaws in strategies: [lookahead-analysis](lookahead-analysis.md) and [recursive-analysis](recursive-analysis.md).

### Assessing backtesting and dry run results

Always dry run your strategy after backtesting it to see if backtesting and dry run results are sufficiently similar.

If there is any significant difference, verify that your entry and exit signals are consistent and appear on the same candles between the two modes. However, there will always be differences between dry runs and backtests:

- Backtesting assumes all orders fill. In dry runs this might not be the case if using limit orders or there is no volume on the exchange.
- Following an entry signal on candle close, backtesting assumes trades enter at the next candle's open price (unless you have custom pricing callbacks in your strategy). In dry runs, there is often a delay between signals and trades opening.
  This is because when new candles come in on your main timeframe, e.g. every 5 minutes, it takes time for Freqtrade to analyse all pair dataframes. Therefore, Freqtrade will attempt to open trades a few seconds (ideally a small a delay as possible)
  after candle open.
- As entry rates in dry runs might not match backtesting, this means profit calculations will also differ. Therefore, it is normal if ROI, stoploss, trailing stoploss and callback exits are not identical.
- The more computational "lag" you have between new candles coming in and your signals being raised and trades being opened will result in greater price unpredictability. Make sure your computer is powerful enough to process the data for the number 
  of pairs you have in your pairlist within a reasonable time. Freqtrade will warn you in the logs if there are significant data processing delays.

## Controlling or monitoring a running bot

Once your bot is running in dry or live mode, Freqtrade has five mechanisms to control or monitor a running bot:

- **[FreqUI](freq-ui.md)**: The easiest to get started with, FreqUI is a web interface to see and control current activity of your bot.
- **[Telegram](telegram-usage.md)**: On mobile devices, Telegram integration is available to get alerts about your bot activity and to control certain aspects.
- **[FTUI](https://github.com/freqtrade/ftui)**: FTUI is a terminal (command line) interface to Freqtrade, and allows monitoring of a running bot only.
- **[REST API](rest-api.md)**: The REST API allows programmers to develop their own tools to interact with a Freqtrade bot.
- **[Webhooks](webhook-config.md)**: Freqtrade can send information to other services, e.g. discord, by webhooks.

### Logs

Freqtrade generates extensive debugging logs to help you understand what's happening. Please familiarise yourself with the information and error messages you might see in your bot logs.

## Final Thoughts

Algo trading is difficult, and most public strategies are not good performers due to the time and effort to make a strategy work profitably in multiple scenarios.

Therefore, taking public strategies and using backtests as a way to assess performance is often problematic. However, Freqtrade provides useful ways to help you make decisions and do your due diligence.

There are many different ways to achieve profitability, and there is no one single tip, trick or config option that will fix a poorly performing strategy.

Freqtrade is an open source platform with a large and helpful community - make sure to visit our [discord channel](https://discord.gg/p7nuUNVfP7) to discuss your strategy with others!

As always, only invest what you are willing to lose.

## Conclusion

Developing a strategy in Freqtrade involves defining entry and exit signals based on technical indicators. By following the structure and methods outlined above, you can create and test your own trading strategies.

Common questions and answers are available on our [FAQ](faq.md).

To continue, refer to the more in-depth [Freqtrade strategy customization documentation](strategy-customization.md).

---

## File: strategy-advanced.md

# Advanced Strategies

This page explains some advanced concepts available for strategies.
If you're just getting started, please familiarize yourself with the [Freqtrade basics](bot-basics.md) and methods described in [Strategy Customization](strategy-customization.md) first.

The call sequence of the methods described here is covered under [bot execution logic](bot-basics.md#bot-execution-logic). Those docs are also helpful in deciding which method is most suitable for your customisation needs.

!!! Note
    Callback methods should *only* be implemented if a strategy uses them.

!!! Tip
    Start off with a strategy template containing all available callback methods by running `freqtrade new-strategy --strategy MyAwesomeStrategy --template advanced`

## Storing information (Persistent)

Freqtrade allows storing/retrieving user custom information associated with a specific trade in the database.

Using a trade object, information can be stored using `trade.set_custom_data(key='my_key', value=my_value)` and retrieved using `trade.get_custom_data(key='my_key')`. Each data entry is associated with a trade and a user supplied key (of type `string`). This means that this can only be used in callbacks that also provide a trade object.

For the data to be able to be stored within the database, freqtrade must serialized the data. This is done by converting the data to a JSON formatted string.
Freqtrade will attempt to reverse this action on retrieval, so from a strategy perspective, this should not be relevant.

```python
from freqtrade.persistence import Trade
from datetime import timedelta

class AwesomeStrategy(IStrategy):

    def bot_loop_start(self, **kwargs) -> None:
        for trade in Trade.get_open_order_trades():
            fills = trade.select_filled_orders(trade.entry_side)
            if trade.pair == 'ETH/USDT':
                trade_entry_type = trade.get_custom_data(key='entry_type')
                if trade_entry_type is None:
                    trade_entry_type = 'breakout' if 'entry_1' in trade.enter_tag else 'dip'
                elif fills > 1:
                    trade_entry_type = 'buy_up'
                trade.set_custom_data(key='entry_type', value=trade_entry_type)
        return super().bot_loop_start(**kwargs)

    def adjust_entry_price(self, trade: Trade, order: Order | None, pair: str,
                           current_time: datetime, proposed_rate: float, current_order_rate: float,
                           entry_tag: str | None, side: str, **kwargs) -> float:
        # Limit orders to use and follow SMA200 as price target for the first 10 minutes since entry trigger for BTC/USDT pair.
        if (
            pair == 'BTC/USDT' 
            and entry_tag == 'long_sma200' 
            and side == 'long' 
            and (current_time - timedelta(minutes=10)) > trade.open_date_utc 
            and order.filled == 0.0
        ):
            dataframe, _ = self.dp.get_analyzed_dataframe(pair=pair, timeframe=self.timeframe)
            current_candle = dataframe.iloc[-1].squeeze()
            # store information about entry adjustment
            existing_count = trade.get_custom_data('num_entry_adjustments', default=0)
            if not existing_count:
                existing_count = 1
            else:
                existing_count += 1
            trade.set_custom_data(key='num_entry_adjustments', value=existing_count)

            # adjust order price
            return current_candle['sma_200']

        # default: maintain existing order
        return current_order_rate

    def custom_exit(self, pair: str, trade: Trade, current_time: datetime, current_rate: float, current_profit: float, **kwargs):

        entry_adjustment_count = trade.get_custom_data(key='num_entry_adjustments')
        trade_entry_type = trade.get_custom_data(key='entry_type')
        if entry_adjustment_count is None:
            if current_profit > 0.01 and (current_time - timedelta(minutes=100) > trade.open_date_utc):
                return True, 'exit_1'
        else
            if entry_adjustment_count > 0 and if current_profit > 0.05:
                return True, 'exit_2'
            if trade_entry_type == 'breakout' and current_profit > 0.1:
                return True, 'exit_3

        return False, None
```

The above is a simple example - there are simpler ways to retrieve trade data like entry-adjustments.

!!! Note
    It is recommended that simple data types are used `[bool, int, float, str]` to ensure no issues when serializing the data that needs to be stored.
    Storing big junks of data may lead to unintended side-effects, like a database becoming big (and as a consequence, also slow).

!!! Warning "Non-serializable data"
    If supplied data cannot be serialized a warning is logged and the entry for the specified `key` will contain `None` as data.

??? Note "All attributes"
    custom-data has the following accessors through the Trade object (assumed as `trade` below):

    * `trade.get_custom_data(key='something', default=0)` - Returns the actual value given in the type provided.
    * `trade.get_custom_data_entry(key='something')` - Returns the entry - including metadata. The value is accessible via `.value` property.
    * `trade.set_custom_data(key='something', value={'some': 'value'})` - set or update the corresponding key for this trade. Value must be serializable - and we recommend to keep the stored data relatively small.

    "value" can be any type (both in setting and receiving) - but must be json serializable.

## Storing information (Non-Persistent)

!!! Warning "Deprecated"
    This method of storing information is deprecated and we do advise against using non-persistent storage.  
    Please use [Persistent Storage](#storing-information-persistent) instead.

    It's content has therefore been collapsed.

??? Abstract "Storing information"
    Storing information can be accomplished by creating a new dictionary within the strategy class.

    The name of the variable can be chosen at will, but should be prefixed with `custom_` to avoid naming collisions with predefined strategy variables.

    ```python
    class AwesomeStrategy(IStrategy):
        # Create custom dictionary
        custom_info = {}

        def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
            # Check if the entry already exists
            if not metadata["pair"] in self.custom_info:
                # Create empty entry for this pair
                self.custom_info[metadata["pair"]] = {}

            if "crosstime" in self.custom_info[metadata["pair"]]:
                self.custom_info[metadata["pair"]]["crosstime"] += 1
            else:
                self.custom_info[metadata["pair"]]["crosstime"] = 1
    ```

    !!! Warning
        The data is not persisted after a bot-restart (or config-reload). Also, the amount of data should be kept smallish (no DataFrames and such), otherwise the bot will start to consume a lot of memory and eventually run out of memory and crash.

    !!! Note
        If the data is pair-specific, make sure to use pair as one of the keys in the dictionary.

## Dataframe access

You may access dataframe in various strategy functions by querying it from dataprovider.

``` python
from freqtrade.exchange import timeframe_to_prev_date

class AwesomeStrategy(IStrategy):
    def confirm_trade_exit(self, pair: str, trade: 'Trade', order_type: str, amount: float,
                           rate: float, time_in_force: str, exit_reason: str,
                           current_time: 'datetime', **kwargs) -> bool:
        # Obtain pair dataframe.
        dataframe, _ = self.dp.get_analyzed_dataframe(pair, self.timeframe)

        # Obtain last available candle. Do not use current_time to look up latest candle, because 
        # current_time points to current incomplete candle whose data is not available.
        last_candle = dataframe.iloc[-1].squeeze()
        # <...>

        # In dry/live runs trade open date will not match candle open date therefore it must be 
        # rounded.
        trade_date = timeframe_to_prev_date(self.timeframe, trade.open_date_utc)
        # Look up trade candle.
        trade_candle = dataframe.loc[dataframe['date'] == trade_date]
        # trade_candle may be empty for trades that just opened as it is still incomplete.
        if not trade_candle.empty:
            trade_candle = trade_candle.squeeze()
            # <...>
```

!!! Warning "Using .iloc[-1]"
    You can use `.iloc[-1]` here because `get_analyzed_dataframe()` only returns candles that backtesting is allowed to see.
    This will not work in `populate_*` methods, so make sure to not use `.iloc[]` in that area.
    Also, this will only work starting with version 2021.5.

***

## Enter Tag

When your strategy has multiple buy signals, you can name the signal that triggered.
Then you can access your buy signal on `custom_exit`

```python
def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    dataframe.loc[
        (
            (dataframe['rsi'] < 35) &
            (dataframe['volume'] > 0)
        ),
        ['enter_long', 'enter_tag']] = (1, 'buy_signal_rsi')

    return dataframe

def custom_exit(self, pair: str, trade: Trade, current_time: datetime, current_rate: float,
                current_profit: float, **kwargs):
    dataframe, _ = self.dp.get_analyzed_dataframe(pair, self.timeframe)
    last_candle = dataframe.iloc[-1].squeeze()
    if trade.enter_tag == 'buy_signal_rsi' and last_candle['rsi'] > 80:
        return 'sell_signal_rsi'
    return None

```

!!! Note
    `enter_tag` is limited to 100 characters, remaining data will be truncated.

!!! Warning
    There is only one `enter_tag` column, which is used for both long and short trades.
    As a consequence, this column must be treated as "last write wins" (it's just a dataframe column after all).
    In fancy situations, where multiple signals collide (or if signals are deactivated again based on different conditions), this can lead to odd results with the wrong tag applied to an entry signal.
    These results are a consequence of the strategy overwriting prior tags - where the last tag will "stick" and will be the one freqtrade will use.

## Exit tag

Similar to [Entry Tagging](#enter-tag), you can also specify an exit tag.

``` python
def populate_exit_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    dataframe.loc[
        (
            (dataframe['rsi'] > 70) &
            (dataframe['volume'] > 0)
        ),
        ['exit_long', 'exit_tag']] = (1, 'exit_rsi')

    return dataframe
```

The provided exit-tag is then used as sell-reason - and shown as such in backtest results.

!!! Note
    `exit_reason` is limited to 100 characters, remaining data will be truncated.

## Strategy version

You can implement custom strategy versioning by using the "version" method, and returning the version you would like this strategy to have.

``` python
def version(self) -> str:
    """
    Returns version of the strategy.
    """
    return "1.1"
```

!!! Note
    You should make sure to implement proper version control (like a git repository) alongside this, as freqtrade will not keep historic versions of your strategy, so it's up to the user to be able to eventually roll back to a prior version of the strategy.

## Derived strategies

The strategies can be derived from other strategies. This avoids duplication of your custom strategy code. You can use this technique to override small parts of your main strategy, leaving the rest untouched:

``` python title="user_data/strategies/myawesomestrategy.py"
class MyAwesomeStrategy(IStrategy):
    ...
    stoploss = 0.13
    trailing_stop = False
    # All other attributes and methods are here as they
    # should be in any custom strategy...
    ...

```

``` python title="user_data/strategies/MyAwesomeStrategy2.py"
from myawesomestrategy import MyAwesomeStrategy
class MyAwesomeStrategy2(MyAwesomeStrategy):
    # Override something
    stoploss = 0.08
    trailing_stop = True
```

Both attributes and methods may be overridden, altering behavior of the original strategy in a way you need.

While keeping the subclass in the same file is technically possible, it can lead to some problems with hyperopt parameter files, we therefore recommend to use separate strategy files, and import the parent strategy as shown above.

## Embedding Strategies

Freqtrade provides you with an easy way to embed the strategy into your configuration file.
This is done by utilizing BASE64 encoding and providing this string at the strategy configuration field,
in your chosen config file.

### Encoding a string as BASE64

This is a quick example, how to generate the BASE64 string in python

```python
from base64 import urlsafe_b64encode

with open(file, 'r') as f:
    content = f.read()
content = urlsafe_b64encode(content.encode('utf-8'))
```

The variable 'content', will contain the strategy file in a BASE64 encoded form. Which can now be set in your configurations file as following

```json
"strategy": "NameOfStrategy:BASE64String"
```

Please ensure that 'NameOfStrategy' is identical to the strategy name!

## Performance warning

When executing a strategy, one can sometimes be greeted by the following in the logs

> PerformanceWarning: DataFrame is highly fragmented.

This is a warning from [`pandas`](https://github.com/pandas-dev/pandas) and as the warning continues to say:
use `pd.concat(axis=1)`.
This can have slight performance implications, which are usually only visible during hyperopt (when optimizing an indicator).

For example:

```python
for val in self.buy_ema_short.range:
    dataframe[f'ema_short_{val}'] = ta.EMA(dataframe, timeperiod=val)
```

should be rewritten to

```python
frames = [dataframe]
for val in self.buy_ema_short.range:
    frames.append(DataFrame({
        f'ema_short_{val}': ta.EMA(dataframe, timeperiod=val)
    }))

# Combine all dataframes, and reassign the original dataframe column
dataframe = pd.concat(frames, axis=1)
```

Freqtrade does however also counter this by running `dataframe.copy()` on the dataframe right after the `populate_indicators()` method - so performance implications of this should be low to non-existent.

---

## File: strategy_analysis_example.md

# Strategy analysis example

Debugging a strategy can be time-consuming. Freqtrade offers helper functions to visualize raw data.
The following assumes you work with SampleStrategy, data for 5m timeframe from Binance and have downloaded them into the data directory in the default location.
Please follow the [documentation](https://www.freqtrade.io/en/stable/data-download/) for more details.

## Setup

### Change Working directory to repository root


```python
import os
from pathlib import Path


# Change directory
# Modify this cell to insure that the output shows the correct path.
# Define all paths relative to the project root shown in the cell output
project_root = "somedir/freqtrade"
i = 0
try:
    os.chdir(project_root)
    if not Path("LICENSE").is_file():
        i = 0
        while i < 4 and (not Path("LICENSE").is_file()):
            os.chdir(Path(Path.cwd(), "../"))
            i += 1
        project_root = Path.cwd()
except FileNotFoundError:
    print("Please define the project root relative to the current directory")
print(Path.cwd())
```

### Configure Freqtrade environment


```python
from freqtrade.configuration import Configuration


# Customize these according to your needs.

# Initialize empty configuration object
config = Configuration.from_files([])
# Optionally (recommended), use existing configuration file
# config = Configuration.from_files(["user_data/config.json"])

# Define some constants
config["timeframe"] = "5m"
# Name of the strategy class
config["strategy"] = "SampleStrategy"
# Location of the data
data_location = config["datadir"]
# Pair to analyze - Only use one pair here
pair = "BTC/USDT"
```


```python
# Load data using values set above
from freqtrade.data.history import load_pair_history
from freqtrade.enums import CandleType


candles = load_pair_history(
    datadir=data_location,
    timeframe=config["timeframe"],
    pair=pair,
    data_format="json",  # Make sure to update this to your data
    candle_type=CandleType.SPOT,
)

# Confirm success
print(f"Loaded {len(candles)} rows of data for {pair} from {data_location}")
candles.head()
```

## Load and run strategy
* Rerun each time the strategy file is changed


```python
# Load strategy using values set above
from freqtrade.data.dataprovider import DataProvider
from freqtrade.resolvers import StrategyResolver


strategy = StrategyResolver.load_strategy(config)
strategy.dp = DataProvider(config, None, None)
strategy.ft_bot_start()

# Generate buy/sell signals using strategy
df = strategy.analyze_ticker(candles, {"pair": pair})
df.tail()
```

### Display the trade details

* Note that using `data.head()` would also work, however most indicators have some "startup" data at the top of the dataframe.
* Some possible problems
    * Columns with NaN values at the end of the dataframe
    * Columns used in `crossed*()` functions with completely different units
* Comparison with full backtest
    * having 200 buy signals as output for one pair from `analyze_ticker()` does not necessarily mean that 200 trades will be made during backtesting.
    * Assuming you use only one condition such as, `df['rsi'] < 30` as buy condition, this will generate multiple "buy" signals for each pair in sequence (until rsi returns > 29). The bot will only buy on the first of these signals (and also only if a trade-slot ("max_open_trades") is still available), or on one of the middle signals, as soon as a "slot" becomes available.  



```python
# Report results
print(f"Generated {df['enter_long'].sum()} entry signals")
data = df.set_index("date", drop=False)
data.tail()
```

## Load existing objects into a Jupyter notebook

The following cells assume that you have already generated data using the cli.  
They will allow you to drill deeper into your results, and perform analysis which otherwise would make the output very difficult to digest due to information overload.

### Load backtest results to pandas dataframe

Analyze a trades dataframe (also used below for plotting)


```python
from freqtrade.data.btanalysis import load_backtest_data, load_backtest_stats


# if backtest_dir points to a directory, it'll automatically load the last backtest file.
backtest_dir = config["user_data_dir"] / "backtest_results"
# backtest_dir can also point to a specific file
# backtest_dir = (
#   config["user_data_dir"] / "backtest_results/backtest-result-2020-07-01_20-04-22.json"
# )
```


```python
# You can get the full backtest statistics by using the following command.
# This contains all information used to generate the backtest result.
stats = load_backtest_stats(backtest_dir)

strategy = "SampleStrategy"
# All statistics are available per strategy, so if `--strategy-list` was used during backtest,
# this will be reflected here as well.
# Example usages:
print(stats["strategy"][strategy]["results_per_pair"])
# Get pairlist used for this backtest
print(stats["strategy"][strategy]["pairlist"])
# Get market change (average change of all pairs from start to end of the backtest period)
print(stats["strategy"][strategy]["market_change"])
# Maximum drawdown ()
print(stats["strategy"][strategy]["max_drawdown_abs"])
# Maximum drawdown start and end
print(stats["strategy"][strategy]["drawdown_start"])
print(stats["strategy"][strategy]["drawdown_end"])


# Get strategy comparison (only relevant if multiple strategies were compared)
print(stats["strategy_comparison"])
```


```python
# Load backtested trades as dataframe
trades = load_backtest_data(backtest_dir)

# Show value-counts per pair
trades.groupby("pair")["exit_reason"].value_counts()
```

## Plotting daily profit / equity line


```python
# Plotting equity line (starting with 0 on day 1 and adding daily profit for each backtested day)

import pandas as pd
import plotly.express as px

from freqtrade.configuration import Configuration
from freqtrade.data.btanalysis import load_backtest_stats


# strategy = 'SampleStrategy'
# config = Configuration.from_files(["user_data/config.json"])
# backtest_dir = config["user_data_dir"] / "backtest_results"

stats = load_backtest_stats(backtest_dir)
strategy_stats = stats["strategy"][strategy]

df = pd.DataFrame(columns=["dates", "equity"], data=strategy_stats["daily_profit"])
df["equity_daily"] = df["equity"].cumsum()

fig = px.line(df, x="dates", y="equity_daily")
fig.show()
```

### Load live trading results into a pandas dataframe

In case you did already some trading and want to analyze your performance


```python
from freqtrade.data.btanalysis import load_trades_from_db


# Fetch trades from database
trades = load_trades_from_db("sqlite:///tradesv3.sqlite")

# Display results
trades.groupby("pair")["exit_reason"].value_counts()
```

## Analyze the loaded trades for trade parallelism
This can be useful to find the best `max_open_trades` parameter, when used with backtesting in conjunction with a very high `max_open_trades` setting.

`analyze_trade_parallelism()` returns a timeseries dataframe with an "open_trades" column, specifying the number of open trades for each candle.


```python
from freqtrade.data.btanalysis import analyze_trade_parallelism


# Analyze the above
parallel_trades = analyze_trade_parallelism(trades, "5m")

parallel_trades.plot()
```

## Plot results

Freqtrade offers interactive plotting capabilities based on plotly.


```python
from freqtrade.plot.plotting import generate_candlestick_graph


# Limit graph period to keep plotly quick and reactive

# Filter trades to one pair
trades_red = trades.loc[trades["pair"] == pair]

data_red = data["2019-06-01":"2019-06-10"]
# Generate candlestick graph
graph = generate_candlestick_graph(
    pair=pair,
    data=data_red,
    trades=trades_red,
    indicators1=["sma20", "ema50", "ema55"],
    indicators2=["rsi", "macd", "macdsignal", "macdhist"],
)
```


```python
# Show graph inline
# graph.show()

# Render graph in a separate window
graph.show(renderer="browser")
```

## Plot average profit per trade as distribution graph


```python
import plotly.figure_factory as ff


hist_data = [trades.profit_ratio]
group_labels = ["profit_ratio"]  # name of the dataset

fig = ff.create_distplot(hist_data, group_labels, bin_size=0.01)
fig.show()
```

Feel free to submit an issue or Pull Request enhancing this document if you would like to share ideas on how to best analyze the data.

---

## File: strategy-callbacks.md

# Strategy Callbacks

While the main strategy functions (`populate_indicators()`, `populate_entry_trend()`, `populate_exit_trend()`) should be used in a vectorized way, and are only called [once during backtesting](bot-basics.md#backtesting-hyperopt-execution-logic), callbacks are called "whenever needed".

As such, you should avoid doing heavy calculations in callbacks to avoid delays during operations.
Depending on the callback used, they may be called when entering / exiting a trade, or throughout the duration of a trade.

Currently available callbacks:

* [`bot_start()`](#bot-start)
* [`bot_loop_start()`](#bot-loop-start)
* [`custom_stake_amount()`](#stake-size-management)
* [`custom_exit()`](#custom-exit-signal)
* [`custom_stoploss()`](#custom-stoploss)
* [`custom_entry_price()` and `custom_exit_price()`](#custom-order-price-rules)
* [`check_entry_timeout()` and `check_exit_timeout()`](#custom-order-timeout-rules)
* [`confirm_trade_entry()`](#trade-entry-buy-order-confirmation)
* [`confirm_trade_exit()`](#trade-exit-sell-order-confirmation)
* [`adjust_trade_position()`](#adjust-trade-position)
* [`adjust_entry_price()`](#adjust-entry-price)
* [`leverage()`](#leverage-callback)
* [`order_filled()`](#order-filled-callback)

!!! Tip "Callback calling sequence"
    You can find the callback calling sequence in [bot-basics](bot-basics.md#bot-execution-logic)

--8<-- "includes/strategy-imports.md"

## Bot start

A simple callback which is called once when the strategy is loaded.
This can be used to perform actions that must only be performed once and runs after dataprovider and wallet are set

``` python
import requests

class AwesomeStrategy(IStrategy):

    # ... populate_* methods

    def bot_start(self, **kwargs) -> None:
        """
        Called only once after bot instantiation.
        :param **kwargs: Ensure to keep this here so updates to this won't break your strategy.
        """
        if self.config["runmode"].value in ("live", "dry_run"):
            # Assign this to the class by using self.*
            # can then be used by populate_* methods
            self.custom_remote_data = requests.get("https://some_remote_source.example.com")

```

During hyperopt, this runs only once at startup.

## Bot loop start

A simple callback which is called once at the start of every bot throttling iteration in dry/live mode (roughly every 5
seconds, unless configured differently) or once per candle in backtest/hyperopt mode.
This can be used to perform calculations which are pair independent (apply to all pairs), loading of external data, etc.

``` python
# Default imports
import requests

class AwesomeStrategy(IStrategy):

    # ... populate_* methods

    def bot_loop_start(self, current_time: datetime, **kwargs) -> None:
        """
        Called at the start of the bot iteration (one loop).
        Might be used to perform pair-independent tasks
        (e.g. gather some remote resource for comparison)
        :param current_time: datetime object, containing the current datetime
        :param **kwargs: Ensure to keep this here so updates to this won't break your strategy.
        """
        if self.config["runmode"].value in ("live", "dry_run"):
            # Assign this to the class by using self.*
            # can then be used by populate_* methods
            self.remote_data = requests.get("https://some_remote_source.example.com")

```

## Stake size management

Called before entering a trade, makes it possible to manage your position size when placing a new trade.

```python
# Default imports

class AwesomeStrategy(IStrategy):
    def custom_stake_amount(self, pair: str, current_time: datetime, current_rate: float,
                            proposed_stake: float, min_stake: float | None, max_stake: float,
                            leverage: float, entry_tag: str | None, side: str,
                            **kwargs) -> float:

        dataframe, _ = self.dp.get_analyzed_dataframe(pair=pair, timeframe=self.timeframe)
        current_candle = dataframe.iloc[-1].squeeze()

        if current_candle["fastk_rsi_1h"] > current_candle["fastd_rsi_1h"]:
            if self.config["stake_amount"] == "unlimited":
                # Use entire available wallet during favorable conditions when in compounding mode.
                return max_stake
            else:
                # Compound profits during favorable conditions instead of using a static stake.
                return self.wallets.get_total_stake_amount() / self.config["max_open_trades"]

        # Use default stake amount.
        return proposed_stake
```

Freqtrade will fall back to the `proposed_stake` value should your code raise an exception. The exception itself will be logged.

!!! Tip
    You do not _have_ to ensure that `min_stake <= returned_value <= max_stake`. Trades will succeed as the returned value will be clamped to supported range and this action will be logged.

!!! Tip
    Returning `0` or `None` will prevent trades from being placed.

## Custom exit signal

Called for open trade every throttling iteration (roughly every 5 seconds) until a trade is closed.

Allows to define custom exit signals, indicating that specified position should be sold. This is very useful when we need to customize exit conditions for each individual trade, or if you need trade data to make an exit decision.

For example you could implement a 1:2 risk-reward ROI with `custom_exit()`.

Using `custom_exit()` signals in place of stoploss though *is not recommended*. It is a inferior method to using `custom_stoploss()` in this regard - which also allows you to keep the stoploss on exchange.

!!! Note
    Returning a (none-empty) `string` or `True` from this method is equal to setting exit signal on a candle at specified time. This method is not called when exit signal is set already, or if exit signals are disabled (`use_exit_signal=False`). `string` max length is 64 characters. Exceeding this limit will cause the message to be truncated to 64 characters.
    `custom_exit()` will ignore `exit_profit_only`, and will always be called unless `use_exit_signal=False`, even if there is a new enter signal.

An example of how we can use different indicators depending on the current profit and also exit trades that were open longer than one day:

``` python
# Default imports

class AwesomeStrategy(IStrategy):
    def custom_exit(self, pair: str, trade: Trade, current_time: datetime, current_rate: float,
                    current_profit: float, **kwargs):
        dataframe, _ = self.dp.get_analyzed_dataframe(pair, self.timeframe)
        last_candle = dataframe.iloc[-1].squeeze()

        # Above 20% profit, sell when rsi < 80
        if current_profit > 0.2:
            if last_candle["rsi"] < 80:
                return "rsi_below_80"

        # Between 2% and 10%, sell if EMA-long above EMA-short
        if 0.02 < current_profit < 0.1:
            if last_candle["emalong"] > last_candle["emashort"]:
                return "ema_long_below_80"

        # Sell any positions at a loss if they are held for more than one day.
        if current_profit < 0.0 and (current_time - trade.open_date_utc).days >= 1:
            return "unclog"
```

See [Dataframe access](strategy-advanced.md#dataframe-access) for more information about dataframe use in strategy callbacks.

## Custom stoploss

Called for open trade every iteration (roughly every 5 seconds) until a trade is closed.

The usage of the custom stoploss method must be enabled by setting `use_custom_stoploss=True` on the strategy object.

The stoploss price can only ever move upwards - if the stoploss value returned from `custom_stoploss` would result in a lower stoploss price than was previously set, it will be ignored. The traditional `stoploss` value serves as an absolute lower level and will be instated as the initial stoploss (before this method is called for the first time for a trade), and is still mandatory.  
As custom stoploss acts as regular, changing stoploss, it will behave similar to `trailing_stop` - and trades exiting due to this will have the exit_reason of `"trailing_stop_loss"`.

The method must return a stoploss value (float / number) as a percentage of the current price.
E.g. If the `current_rate` is 200 USD, then returning `0.02` will set the stoploss price 2% lower, at 196 USD.
During backtesting, `current_rate` (and `current_profit`) are provided against the candle's high (or low for short trades) - while the resulting stoploss is evaluated against the candle's low (or high for short trades).

The absolute value of the return value is used (the sign is ignored), so returning `0.05` or `-0.05` have the same result, a stoploss 5% below the current price.
Returning `None` will be interpreted as "no desire to change", and is the only safe way to return when you'd like to not modify the stoploss.
`NaN` and `inf` values are considered invalid and will be ignored (identical to `None`).

Stoploss on exchange works similar to `trailing_stop`, and the stoploss on exchange is updated as configured in `stoploss_on_exchange_interval` ([More details about stoploss on exchange](stoploss.md#stop-loss-on-exchangefreqtrade)).

!!! Note "Use of dates"
    All time-based calculations should be done based on `current_time` - using `datetime.now()` or `datetime.utcnow()` is discouraged, as this will break backtesting support.

!!! Tip "Trailing stoploss"
    It's recommended to disable `trailing_stop` when using custom stoploss values. Both can work in tandem, but you might encounter the trailing stop to move the price higher while your custom function would not want this, causing conflicting behavior.

### Adjust stoploss after position adjustments

Depending on your strategy, you may encounter the need to adjust the stoploss in both directions after a [position adjustment](#adjust-trade-position).
For this, freqtrade will make an additional call with `after_fill=True` after an order fills, which will allow the strategy to move the stoploss in any direction (also widening the gap between stoploss and current price, which is otherwise forbidden).

!!! Note "backwards compatibility"
    This call will only be made if the `after_fill` parameter is part of the function definition of your `custom_stoploss` function.
    As such, this will not impact (and with that, surprise) existing, running strategies.

### Custom stoploss examples

The next section will show some examples on what's possible with the custom stoploss function.
Of course, many more things are possible, and all examples can be combined at will.

#### Trailing stop via custom stoploss

To simulate a regular trailing stoploss of 4% (trailing 4% behind the maximum reached price) you would use the following very simple method:

``` python
# Default imports

class AwesomeStrategy(IStrategy):

    # ... populate_* methods

    use_custom_stoploss = True

    def custom_stoploss(self, pair: str, trade: Trade, current_time: datetime,
                        current_rate: float, current_profit: float, after_fill: bool, 
                        **kwargs) -> float | None:
        """
        Custom stoploss logic, returning the new distance relative to current_rate (as ratio).
        e.g. returning -0.05 would create a stoploss 5% below current_rate.
        The custom stoploss can never be below self.stoploss, which serves as a hard maximum loss.

        For full documentation please go to https://www.freqtrade.io/en/latest/strategy-advanced/

        When not implemented by a strategy, returns the initial stoploss value.
        Only called when use_custom_stoploss is set to True.

        :param pair: Pair that's currently analyzed
        :param trade: trade object.
        :param current_time: datetime object, containing the current datetime
        :param current_rate: Rate, calculated based on pricing settings in exit_pricing.
        :param current_profit: Current profit (as ratio), calculated based on current_rate.
        :param after_fill: True if the stoploss is called after the order was filled.
        :param **kwargs: Ensure to keep this here so updates to this won't break your strategy.
        :return float: New stoploss value, relative to the current_rate
        """
        return -0.04
```

#### Time based trailing stop

Use the initial stoploss for the first 60 minutes, after this change to 10% trailing stoploss, and after 2 hours (120 minutes) we use a 5% trailing stoploss.

``` python
# Default imports

class AwesomeStrategy(IStrategy):

    # ... populate_* methods

    use_custom_stoploss = True

    def custom_stoploss(self, pair: str, trade: Trade, current_time: datetime,
                        current_rate: float, current_profit: float, after_fill: bool, 
                        **kwargs) -> float | None:

        # Make sure you have the longest interval first - these conditions are evaluated from top to bottom.
        if current_time - timedelta(minutes=120) > trade.open_date_utc:
            return -0.05
        elif current_time - timedelta(minutes=60) > trade.open_date_utc:
            return -0.10
        return None
```

#### Time based trailing stop with after-fill adjustments

Use the initial stoploss for the first 60 minutes, after this change to 10% trailing stoploss, and after 2 hours (120 minutes) we use a 5% trailing stoploss.
If an additional order fills, set stoploss to -10% below the new `open_rate` ([Averaged across all entries](#position-adjust-calculations)).

``` python
# Default imports

class AwesomeStrategy(IStrategy):

    # ... populate_* methods

    use_custom_stoploss = True

    def custom_stoploss(self, pair: str, trade: Trade, current_time: datetime,
                        current_rate: float, current_profit: float, after_fill: bool, 
                        **kwargs) -> float | None:

        if after_fill: 
            # After an additional order, start with a stoploss of 10% below the new open rate
            return stoploss_from_open(0.10, current_profit, is_short=trade.is_short, leverage=trade.leverage)
        # Make sure you have the longest interval first - these conditions are evaluated from top to bottom.
        if current_time - timedelta(minutes=120) > trade.open_date_utc:
            return -0.05
        elif current_time - timedelta(minutes=60) > trade.open_date_utc:
            return -0.10
        return None
```

#### Different stoploss per pair

Use a different stoploss depending on the pair.
In this example, we'll trail the highest price with 10% trailing stoploss for `ETH/BTC` and `XRP/BTC`, with 5% trailing stoploss for `LTC/BTC` and with 15% for all other pairs.

``` python
# Default imports

class AwesomeStrategy(IStrategy):

    # ... populate_* methods

    use_custom_stoploss = True

    def custom_stoploss(self, pair: str, trade: Trade, current_time: datetime,
                        current_rate: float, current_profit: float, after_fill: bool,
                        **kwargs) -> float | None:

        if pair in ("ETH/BTC", "XRP/BTC"):
            return -0.10
        elif pair in ("LTC/BTC"):
            return -0.05
        return -0.15
```

#### Trailing stoploss with positive offset

Use the initial stoploss until the profit is above 4%, then use a trailing stoploss of 50% of the current profit with a minimum of 2.5% and a maximum of 5%.

Please note that the stoploss can only increase, values lower than the current stoploss are ignored.

``` python
# Default imports

class AwesomeStrategy(IStrategy):

    # ... populate_* methods

    use_custom_stoploss = True

    def custom_stoploss(self, pair: str, trade: Trade, current_time: datetime,
                        current_rate: float, current_profit: float, after_fill: bool,
                        **kwargs) -> float | None:

        if current_profit < 0.04:
            return None # return None to keep using the initial stoploss

        # After reaching the desired offset, allow the stoploss to trail by half the profit
        desired_stoploss = current_profit / 2

        # Use a minimum of 2.5% and a maximum of 5%
        return max(min(desired_stoploss, 0.05), 0.025)
```

#### Stepped stoploss

Instead of continuously trailing behind the current price, this example sets fixed stoploss price levels based on the current profit.

* Use the regular stoploss until 20% profit is reached
* Once profit is > 20% - set stoploss to 7% above open price.
* Once profit is > 25% - set stoploss to 15% above open price.
* Once profit is > 40% - set stoploss to 25% above open price.

``` python
# Default imports

class AwesomeStrategy(IStrategy):

    # ... populate_* methods

    use_custom_stoploss = True

    def custom_stoploss(self, pair: str, trade: Trade, current_time: datetime,
                        current_rate: float, current_profit: float, after_fill: bool,
                        **kwargs) -> float | None:

        # evaluate highest to lowest, so that highest possible stop is used
        if current_profit > 0.40:
            return stoploss_from_open(0.25, current_profit, is_short=trade.is_short, leverage=trade.leverage)
        elif current_profit > 0.25:
            return stoploss_from_open(0.15, current_profit, is_short=trade.is_short, leverage=trade.leverage)
        elif current_profit > 0.20:
            return stoploss_from_open(0.07, current_profit, is_short=trade.is_short, leverage=trade.leverage)

        # return maximum stoploss value, keeping current stoploss price unchanged
        return None
```

#### Custom stoploss using an indicator from dataframe example

Absolute stoploss value may be derived from indicators stored in dataframe. Example uses parabolic SAR below the price as stoploss.

``` python
# Default imports

class AwesomeStrategy(IStrategy):

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        # <...>
        dataframe["sar"] = ta.SAR(dataframe)

    use_custom_stoploss = True

    def custom_stoploss(self, pair: str, trade: Trade, current_time: datetime,
                        current_rate: float, current_profit: float, after_fill: bool,
                        **kwargs) -> float | None:

        dataframe, _ = self.dp.get_analyzed_dataframe(pair, self.timeframe)
        last_candle = dataframe.iloc[-1].squeeze()

        # Use parabolic sar as absolute stoploss price
        stoploss_price = last_candle["sar"]

        # Convert absolute price to percentage relative to current_rate
        if stoploss_price < current_rate:
            return stoploss_from_absolute(stoploss_price, current_rate, is_short=trade.is_short)

        # return maximum stoploss value, keeping current stoploss price unchanged
        return None
```

See [Dataframe access](strategy-advanced.md#dataframe-access) for more information about dataframe use in strategy callbacks.

### Common helpers for stoploss calculations

#### Stoploss relative to open price

Stoploss values returned from `custom_stoploss()` must specify a percentage relative to `current_rate`, but sometimes you may want to specify a stoploss relative to the _entry_ price instead.
`stoploss_from_open()` is a helper function to calculate a stoploss value that can be returned from `custom_stoploss` which will be equivalent to the desired trade profit above the entry point.

??? Example "Returning a stoploss relative to the open price from the custom stoploss function"

    Say the open price was $100, and `current_price` is $121 (`current_profit` will be `0.21`).  

    If we want a stop price at 7% above the open price we can call `stoploss_from_open(0.07, current_profit, False)` which will return `0.1157024793`.  11.57% below $121 is $107, which is the same as 7% above $100.

    This function will consider leverage - so at 10x leverage, the actual stoploss would be 0.7% above $100 (0.7% * 10x = 7%).


    ``` python
    # Default imports

    class AwesomeStrategy(IStrategy):

        # ... populate_* methods

        use_custom_stoploss = True

        def custom_stoploss(self, pair: str, trade: Trade, current_time: datetime,
                            current_rate: float, current_profit: float, after_fill: bool,
                            **kwargs) -> float | None:

            # once the profit has risen above 10%, keep the stoploss at 7% above the open price
            if current_profit > 0.10:
                return stoploss_from_open(0.07, current_profit, is_short=trade.is_short, leverage=trade.leverage)

            return 1

    ```

    Full examples can be found in the [Custom stoploss](strategy-callbacks.md#custom-stoploss) section of the Documentation.

!!! Note
    Providing invalid input to `stoploss_from_open()` may produce "CustomStoploss function did not return valid stoploss" warnings.
    This may happen if `current_profit` parameter is below specified `open_relative_stop`. Such situations may arise when closing trade
    is blocked by `confirm_trade_exit()` method. Warnings can be solved by never blocking stop loss sells by checking `exit_reason` in
    `confirm_trade_exit()`, or by using `return stoploss_from_open(...) or 1` idiom, which will request to not change stop loss when
    `current_profit < open_relative_stop`.

#### Stoploss percentage from absolute price

Stoploss values returned from `custom_stoploss()` always specify a percentage relative to `current_rate`. In order to set a stoploss at specified absolute price level, we need to use `stop_rate` to calculate what percentage relative to the `current_rate` will give you the same result as if the percentage was specified from the open price.

The helper function `stoploss_from_absolute()` can be used to convert from an absolute price, to a current price relative stop which can be returned from `custom_stoploss()`.

??? Example "Returning a stoploss using absolute price from the custom stoploss function"

    If we want to trail a stop price at 2xATR below current price we can call `stoploss_from_absolute(current_rate + (side * candle["atr"] * 2), current_rate=current_rate, is_short=trade.is_short, leverage=trade.leverage)`.
    For futures, we need to adjust the direction (up or down), as well as adjust for leverage, since the [`custom_stoploss`](strategy-callbacks.md#custom-stoploss) callback  returns the ["risk for this trade"](stoploss.md#stoploss-and-leverage) - not the relative price movement.

    ``` python
    # Default imports

    class AwesomeStrategy(IStrategy):

        use_custom_stoploss = True

        def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
            dataframe["atr"] = ta.ATR(dataframe, timeperiod=14)
            return dataframe

        def custom_stoploss(self, pair: str, trade: Trade, current_time: datetime,
                            current_rate: float, current_profit: float, after_fill: bool,
                            **kwargs) -> float | None:
            dataframe, _ = self.dp.get_analyzed_dataframe(pair, self.timeframe)
            trade_date = timeframe_to_prev_date(self.timeframe, trade.open_date_utc)
            candle = dataframe.iloc[-1].squeeze()
            side = 1 if trade.is_short else -1
            return stoploss_from_absolute(current_rate + (side * candle["atr"] * 2), 
                                          current_rate=current_rate, 
                                          is_short=trade.is_short,
                                          leverage=trade.leverage)

    ```

---

## Custom order price rules

By default, freqtrade use the orderbook to automatically set an order price([Relevant documentation](configuration.md#prices-used-for-orders)), you also have the option to create custom order prices based on your strategy.

You can use this feature by creating a `custom_entry_price()` function in your strategy file to customize entry prices and `custom_exit_price()` for exits.

Each of these methods are called right before placing an order on the exchange.

!!! Note
    If your custom pricing function return None or an invalid value, price will fall back to `proposed_rate`, which is based on the regular pricing configuration.

!!! Note
    Using custom_entry_price, the Trade object will be available as soon as the first entry order associated with the trade is created, for the first entry, `trade` parameter value will be `None`.

### Custom order entry and exit price example

``` python
# Default imports

class AwesomeStrategy(IStrategy):

    # ... populate_* methods

    def custom_entry_price(self, pair: str, trade: Trade | None, current_time: datetime, proposed_rate: float,
                           entry_tag: str | None, side: str, **kwargs) -> float:

        dataframe, last_updated = self.dp.get_analyzed_dataframe(pair=pair,
                                                                timeframe=self.timeframe)
        new_entryprice = dataframe["bollinger_10_lowerband"].iat[-1]

        return new_entryprice

    def custom_exit_price(self, pair: str, trade: Trade,
                          current_time: datetime, proposed_rate: float,
                          current_profit: float, exit_tag: str | None, **kwargs) -> float:

        dataframe, last_updated = self.dp.get_analyzed_dataframe(pair=pair,
                                                                timeframe=self.timeframe)
        new_exitprice = dataframe["bollinger_10_upperband"].iat[-1]

        return new_exitprice

```

!!! Warning
    Modifying entry and exit prices will only work for limit orders. Depending on the price chosen, this can result in a lot of unfilled orders. By default the maximum allowed distance between the current price and the custom price is 2%, this value can be changed in config with the `custom_price_max_distance_ratio` parameter.
    **Example**:
    If the new_entryprice is 97, the proposed_rate is 100 and the `custom_price_max_distance_ratio` is set to 2%, The retained valid custom entry price will be 98, which is 2% below the current (proposed) rate.

!!! Warning "Backtesting"
    Custom prices are supported in backtesting (starting with 2021.12), and orders will fill if the price falls within the candle's low/high range.
    Orders that don't fill immediately are subject to regular timeout handling, which happens once per (detail) candle.
    `custom_exit_price()` is only called for sells of type exit_signal, Custom exit and partial exits. All other exit-types will use regular backtesting prices.

## Custom order timeout rules

Simple, time-based order-timeouts can be configured either via strategy or in the configuration in the `unfilledtimeout` section.

However, freqtrade also offers a custom callback for both order types, which allows you to decide based on custom criteria if an order did time out or not.

!!! Note
    Backtesting fills orders if their price falls within the candle's low/high range.
    The below callbacks will be called once per (detail) candle for orders that don't fill immediately (which use custom pricing).

### Custom order timeout example

Called for every open order until that order is either filled or cancelled.
`check_entry_timeout()` is called for trade entries, while `check_exit_timeout()` is called for trade exit orders.

A simple example, which applies different unfilled-timeouts depending on the price of the asset can be seen below.
It applies a tight timeout for higher priced assets, while allowing more time to fill on cheap coins.

The function must return either `True` (cancel order) or `False` (keep order alive).

``` python
    # Default imports

class AwesomeStrategy(IStrategy):

    # ... populate_* methods

    # Set unfilledtimeout to 25 hours, since the maximum timeout from below is 24 hours.
    unfilledtimeout = {
        "entry": 60 * 25,
        "exit": 60 * 25
    }

    def check_entry_timeout(self, pair: str, trade: Trade, order: Order,
                            current_time: datetime, **kwargs) -> bool:
        if trade.open_rate > 100 and trade.open_date_utc < current_time - timedelta(minutes=5):
            return True
        elif trade.open_rate > 10 and trade.open_date_utc < current_time - timedelta(minutes=3):
            return True
        elif trade.open_rate < 1 and trade.open_date_utc < current_time - timedelta(hours=24):
           return True
        return False


    def check_exit_timeout(self, pair: str, trade: Trade, order: Order,
                           current_time: datetime, **kwargs) -> bool:
        if trade.open_rate > 100 and trade.open_date_utc < current_time - timedelta(minutes=5):
            return True
        elif trade.open_rate > 10 and trade.open_date_utc < current_time - timedelta(minutes=3):
            return True
        elif trade.open_rate < 1 and trade.open_date_utc < current_time - timedelta(hours=24):
           return True
        return False
```

!!! Note
    For the above example, `unfilledtimeout` must be set to something bigger than 24h, otherwise that type of timeout will apply first.

### Custom order timeout example (using additional data)

``` python
    # Default imports

class AwesomeStrategy(IStrategy):

    # ... populate_* methods

    # Set unfilledtimeout to 25 hours, since the maximum timeout from below is 24 hours.
    unfilledtimeout = {
        "entry": 60 * 25,
        "exit": 60 * 25
    }

    def check_entry_timeout(self, pair: str, trade: Trade, order: Order,
                            current_time: datetime, **kwargs) -> bool:
        ob = self.dp.orderbook(pair, 1)
        current_price = ob["bids"][0][0]
        # Cancel buy order if price is more than 2% above the order.
        if current_price > order.price * 1.02:
            return True
        return False


    def check_exit_timeout(self, pair: str, trade: Trade, order: Order,
                           current_time: datetime, **kwargs) -> bool:
        ob = self.dp.orderbook(pair, 1)
        current_price = ob["asks"][0][0]
        # Cancel sell order if price is more than 2% below the order.
        if current_price < order.price * 0.98:
            return True
        return False
```

---

## Bot order confirmation

Confirm trade entry / exits.
This are the last methods that will be called before an order is placed.

### Trade entry (buy order) confirmation

`confirm_trade_entry()` can be used to abort a trade entry at the latest second (maybe because the price is not what we expect).

``` python
# Default imports

class AwesomeStrategy(IStrategy):

    # ... populate_* methods

    def confirm_trade_entry(self, pair: str, order_type: str, amount: float, rate: float,
                            time_in_force: str, current_time: datetime, entry_tag: str | None,
                            side: str, **kwargs) -> bool:
        """
        Called right before placing a entry order.
        Timing for this function is critical, so avoid doing heavy computations or
        network requests in this method.

        For full documentation please go to https://www.freqtrade.io/en/latest/strategy-advanced/

        When not implemented by a strategy, returns True (always confirming).

        :param pair: Pair that's about to be bought/shorted.
        :param order_type: Order type (as configured in order_types). usually limit or market.
        :param amount: Amount in target (base) currency that's going to be traded.
        :param rate: Rate that's going to be used when using limit orders 
                     or current rate for market orders.
        :param time_in_force: Time in force. Defaults to GTC (Good-til-cancelled).
        :param current_time: datetime object, containing the current datetime
        :param entry_tag: Optional entry_tag (buy_tag) if provided with the buy signal.
        :param side: "long" or "short" - indicating the direction of the proposed trade
        :param **kwargs: Ensure to keep this here so updates to this won't break your strategy.
        :return bool: When True is returned, then the buy-order is placed on the exchange.
            False aborts the process
        """
        return True

```

### Trade exit (sell order) confirmation

`confirm_trade_exit()` can be used to abort a trade exit (sell) at the latest second (maybe because the price is not what we expect).

`confirm_trade_exit()` may be called multiple times within one iteration for the same trade if different exit-reasons apply.
The exit-reasons (if applicable) will be in the following sequence:

* `exit_signal` / `custom_exit`
* `stop_loss`
* `roi`
* `trailing_stop_loss`

``` python
# Default imports

class AwesomeStrategy(IStrategy):

    # ... populate_* methods

    def confirm_trade_exit(self, pair: str, trade: Trade, order_type: str, amount: float,
                           rate: float, time_in_force: str, exit_reason: str,
                           current_time: datetime, **kwargs) -> bool:
        """
        Called right before placing a regular exit order.
        Timing for this function is critical, so avoid doing heavy computations or
        network requests in this method.

        For full documentation please go to https://www.freqtrade.io/en/latest/strategy-advanced/

        When not implemented by a strategy, returns True (always confirming).

        :param pair: Pair for trade that's about to be exited.
        :param trade: trade object.
        :param order_type: Order type (as configured in order_types). usually limit or market.
        :param amount: Amount in base currency.
        :param rate: Rate that's going to be used when using limit orders
                     or current rate for market orders.
        :param time_in_force: Time in force. Defaults to GTC (Good-til-cancelled).
        :param exit_reason: Exit reason.
            Can be any of ["roi", "stop_loss", "stoploss_on_exchange", "trailing_stop_loss",
                           "exit_signal", "force_exit", "emergency_exit"]
        :param current_time: datetime object, containing the current datetime
        :param **kwargs: Ensure to keep this here so updates to this won't break your strategy.
        :return bool: When True, then the exit-order is placed on the exchange.
            False aborts the process
        """
        if exit_reason == "force_exit" and trade.calc_profit_ratio(rate) < 0:
            # Reject force-sells with negative profit
            # This is just a sample, please adjust to your needs
            # (this does not necessarily make sense, assuming you know when you're force-selling)
            return False
        return True

```

!!! Warning
    `confirm_trade_exit()` can prevent stoploss exits, causing significant losses as this would ignore stoploss exits.
    `confirm_trade_exit()` will not be called for Liquidations - as liquidations are forced by the exchange, and therefore cannot be rejected.

## Adjust trade position

The `position_adjustment_enable` strategy property enables the usage of `adjust_trade_position()` callback in the strategy.
For performance reasons, it's disabled by default and freqtrade will show a warning message on startup if enabled.
`adjust_trade_position()` can be used to perform additional orders, for example to manage risk with DCA (Dollar Cost Averaging) or to increase or decrease positions.

Additional orders also result in additional fees and those orders don't count towards `max_open_trades`.

This callback is also called when there is an open order (either buy or sell) waiting for execution - and will cancel the existing open order to place a new order if the amount, price or direction is different. Also partially filled orders will be canceled, and will be replaced with the new amount as returned by the callback.

`adjust_trade_position()` is called very frequently for the duration of a trade, so you must keep your implementation as performant as possible.

Position adjustments will always be applied in the direction of the trade, so a positive value will always increase your position (negative values will decrease your position), no matter if it's a long or short trade.
Adjustment orders can be assigned with a tag by returning a 2 element Tuple, with the first element being the adjustment amount, and the 2nd element the tag (e.g. `return 250, "increase_favorable_conditions"`).

Modifications to leverage are not possible, and the stake-amount returned is assumed to be before applying leverage.

The combined stake currently allocated to the position is held in `trade.stake_amount`. Therefore `trade.stake_amount` will always be updated on every additional entry and partial exit made through `adjust_trade_position()`.

!!! Danger "Loose Logic"
    On dry and live run, this function will be called every `throttle_process_secs` (default to 5s). If you have a loose logic, (e.g. increase position if RSI of the last candle is below 30), your bot will do extra re-entry every 5 secs until you either it run out of money, hit the `max_position_adjustment` limit, or a new candle with RSI more than 30 arrived.

    Same thing also can happen with partial exit.  
    So be sure to have a strict logic and/or check for the last filled order and if an order is already open.

!!! Warning "Performance with many position adjustments"
    Position adjustments can be a good approach to increase a strategy's output - but it can also have drawbacks if using this feature extensively.  
    Each of the orders will be attached to the trade object for the duration of the trade - hence increasing memory usage.
    Trades with long duration and 10s or even 100ds of position adjustments are therefore not recommended, and should be closed at regular intervals to not affect performance.

!!! Warning "Backtesting"
    During backtesting this callback is called for each candle in `timeframe` or `timeframe_detail`, so run-time performance will be affected.
    This can also cause deviating results between live and backtesting, since backtesting can adjust the trade only once per candle, whereas live could adjust the trade multiple times per candle.

### Increase position

The strategy is expected to return a positive **stake_amount** (in stake currency) between `min_stake` and `max_stake` if and when an additional entry order should be made (position is increased -> buy order for long trades, sell order for short trades).

If there are not enough funds in the wallet (the return value is above `max_stake`) then the signal will be ignored.
`max_entry_position_adjustment` property is used to limit the number of additional entries per trade (on top of the first entry order) that the bot can execute. By default, the value is -1 which means the bot have no limit on number of adjustment entries.

Additional entries are ignored once you have reached the maximum amount of extra entries that you have set on `max_entry_position_adjustment`, but the callback is called anyway looking for partial exits.

!!! Note "About stake size"
    Using fixed stake size means it will be the amount used for the first order, just like without position adjustment.
    If you wish to buy additional orders with DCA, then make sure to leave enough funds in the wallet for that.
    Using `"unlimited"` stake amount with DCA orders requires you to also implement the `custom_stake_amount()` callback to avoid allocating all funds to the initial order.

### Decrease position

The strategy is expected to return a negative stake_amount (in stake currency) for a partial exit.
Returning the full owned stake at that point (`-trade.stake_amount`) results in a full exit.  
Returning a value more than the above (so remaining stake_amount would become negative) will result in the bot ignoring the signal.

For a partial exit, it's important to know that the formula used to calculate the amount of the coin for the partial exit order is `amount to be exited partially = negative_stake_amount * trade.amount / trade.stake_amount`, where `negative_stake_amount` is the value returned from the `adjust_trade_position` function. As seen in the formula, the formula doesn't care about current profit/loss of the position. It only cares about `trade.amount` and `trade.stake_amount` which aren't affected by the price movement at all.

For example, let's say you buy 2 SHITCOIN/USDT at open rate of 50, which means the trade's stake amount is 100 USDT. Now the price raises to 200 and you want to sell half of it. In that case, you have to return -50% of `trade.stake_amount` (0.5 * 100 USDT) which equals to -50. The bot will calculate the amount it needed to sell, which is `50 * 2 / 100` which equals 1 SHITCOIN/USDT. If you return -200 (50% of 2 * 200), the bot will ignore it since `trade.stake_amount` is only 100 USDT but you asked to sell 200 USDT which means you are asking to sell 4 SHITCOIN/USDT.

Back to the example above, since current rate is 200, the current USDT value of your trade is now 400 USDT. Let's say you want to partially sell 100 USDT to take out the initial investment and leave the profit in the trade hoping that the price keeps rising. In that case, you have to do a different approach. First, you need to calculate the exact amount you needed to sell. In this case, since you want to sell 100 USDT worth based of current rate, the exact amount you need to partially sell is `100 * 2 / 400` which equals 0.5 SHITCOIN/USDT. Since we know now the exact amount we want to sell (0.5), the value you need to return in the `adjust_trade_position` function is `-amount to be exited partially * trade.stake_amount / trade.amount`, which equals -25. The bot will sell 0.5 SHITCOIN/USDT, keeping 1.5 in trade. You will receive 100 USDT from the partial exit.

!!! Warning "Stoploss calculation"
    Stoploss is still calculated from the initial opening price, not averaged price.
    Regular stoploss rules still apply (cannot move down).

    While `/stopentry` command stops the bot from entering new trades, the position adjustment feature will continue buying new orders on existing trades.

``` python
# Default imports

class DigDeeperStrategy(IStrategy):

    position_adjustment_enable = True

    # Attempts to handle large drops with DCA. High stoploss is required.
    stoploss = -0.30

    # ... populate_* methods

    # Example specific variables
    max_entry_position_adjustment = 3
    # This number is explained a bit further down
    max_dca_multiplier = 5.5

    # This is called when placing the initial order (opening trade)
    def custom_stake_amount(self, pair: str, current_time: datetime, current_rate: float,
                            proposed_stake: float, min_stake: float | None, max_stake: float,
                            leverage: float, entry_tag: str | None, side: str,
                            **kwargs) -> float:

        # We need to leave most of the funds for possible further DCA orders
        # This also applies to fixed stakes
        return proposed_stake / self.max_dca_multiplier

    def adjust_trade_position(self, trade: Trade, current_time: datetime,
                              current_rate: float, current_profit: float,
                              min_stake: float | None, max_stake: float,
                              current_entry_rate: float, current_exit_rate: float,
                              current_entry_profit: float, current_exit_profit: float,
                              **kwargs
                              ) -> float | None | tuple[float | None, str | None]:
        """
        Custom trade adjustment logic, returning the stake amount that a trade should be
        increased or decreased.
        This means extra entry or exit orders with additional fees.
        Only called when `position_adjustment_enable` is set to True.

        For full documentation please go to https://www.freqtrade.io/en/latest/strategy-advanced/

        When not implemented by a strategy, returns None

        :param trade: trade object.
        :param current_time: datetime object, containing the current datetime
        :param current_rate: Current entry rate (same as current_entry_profit)
        :param current_profit: Current profit (as ratio), calculated based on current_rate 
                               (same as current_entry_profit).
        :param min_stake: Minimal stake size allowed by exchange (for both entries and exits)
        :param max_stake: Maximum stake allowed (either through balance, or by exchange limits).
        :param current_entry_rate: Current rate using entry pricing.
        :param current_exit_rate: Current rate using exit pricing.
        :param current_entry_profit: Current profit using entry pricing.
        :param current_exit_profit: Current profit using exit pricing.
        :param **kwargs: Ensure to keep this here so updates to this won't break your strategy.
        :return float: Stake amount to adjust your trade,
                       Positive values to increase position, Negative values to decrease position.
                       Return None for no action.
                       Optionally, return a tuple with a 2nd element with an order reason
        """
        if trade.has_open_orders:
            # Only act if no orders are open
            return

        if current_profit > 0.05 and trade.nr_of_successful_exits == 0:
            # Take half of the profit at +5%
            return -(trade.stake_amount / 2), "half_profit_5%"

        if current_profit > -0.05:
            return None

        # Obtain pair dataframe (just to show how to access it)
        dataframe, _ = self.dp.get_analyzed_dataframe(trade.pair, self.timeframe)
        # Only buy when not actively falling price.
        last_candle = dataframe.iloc[-1].squeeze()
        previous_candle = dataframe.iloc[-2].squeeze()
        if last_candle["close"] < previous_candle["close"]:
            return None

        filled_entries = trade.select_filled_orders(trade.entry_side)
        count_of_entries = trade.nr_of_successful_entries
        # Allow up to 3 additional increasingly larger buys (4 in total)
        # Initial buy is 1x
        # If that falls to -5% profit, we buy 1.25x more, average profit should increase to roughly -2.2%
        # If that falls down to -5% again, we buy 1.5x more
        # If that falls once again down to -5%, we buy 1.75x more
        # Total stake for this trade would be 1 + 1.25 + 1.5 + 1.75 = 5.5x of the initial allowed stake.
        # That is why max_dca_multiplier is 5.5
        # Hope you have a deep wallet!
        try:
            # This returns first order stake size
            stake_amount = filled_entries[0].stake_amount_filled
            # This then calculates current safety order size
            stake_amount = stake_amount * (1 + (count_of_entries * 0.25))
            return stake_amount, "1/3rd_increase"
        except Exception as exception:
            return None

        return None

```

### Position adjust calculations

* Entry rates are calculated using weighted averages.
* Exits will not influence the average entry rate.
* Partial exit relative profit is relative to the average entry price at this point.
* Final exit relative profit is calculated based on the total invested capital. (See example below)

??? example "Calculation example"
    *This example assumes 0 fees for simplicity, and a long position on an imaginary coin.*  
    
    * Buy 100@8\$ 
    * Buy 100@9\$ -> Avg price: 8.5\$
    * Sell 100@10\$ -> Avg price: 8.5\$, realized profit 150\$, 17.65%
    * Buy 150@11\$ -> Avg price: 10\$, realized profit 150\$, 17.65%
    * Sell 100@12\$ -> Avg price: 10\$, total realized profit 350\$, 20%
    * Sell 150@14\$ -> Avg price: 10\$, total realized profit 950\$, 40%  <- *This will be the last "Exit" message*

    The total profit for this trade was 950$ on a 3350$ investment (`100@8$ + 100@9$ + 150@11$`). As such - the final relative profit is 28.35% (`950 / 3350`).

## Adjust order Price

The `adjust_order_price()` callback may be used by strategy developer to refresh/replace limit orders upon arrival of new candles.  
This callback is called once every iteration unless the order has been (re)placed within the current candle - limiting the maximum (re)placement of each order to once per candle.
This also means that the first call will be at the start of the next candle after the initial order was placed.

Be aware that `custom_entry_price()`/`custom_exit_price()` is still the one dictating initial limit order price target at the time of the signal.

Orders can be cancelled out of this callback by returning `None`.

Returning `current_order_rate` will keep the order on the exchange "as is".
Returning any other price will cancel the existing order, and replace it with a new order.

If the cancellation of the original order fails, then the order will not be replaced - though the order will most likely have been canceled on exchange. Having this happen on initial entries will result in the deletion of the order, while on position adjustment orders, it'll result in the trade size remaining as is.  
If the order has been partially filled, the order will not be replaced. You can however use [`adjust_trade_position()`](#adjust-trade-position) to adjust the trade size to the expected position size, should this be necessary / desired.

!!! Warning "Regular timeout"
    Entry `unfilledtimeout` mechanism (as well as `check_entry_timeout()`/`check_exit_timeout()`) takes precedence over this callback.
    Orders that are cancelled via the above methods will not have this callback called. Be sure to update timeout values to match your expectations.

```python
# Default imports

class AwesomeStrategy(IStrategy):

    # ... populate_* methods

    def adjust_order_price(
        self,
        trade: Trade,
        order: Order | None,
        pair: str,
        current_time: datetime,
        proposed_rate: float,
        current_order_rate: float,
        entry_tag: str | None,
        side: str,
        is_entry: bool,
        **kwargs,
    ) -> float | None:
        """
        Exit and entry order price re-adjustment logic, returning the user desired limit price.
        This only executes when a order was already placed, still open (unfilled fully or partially)
        and not timed out on subsequent candles after entry trigger.

        For full documentation please go to https://www.freqtrade.io/en/latest/strategy-callbacks/

        When not implemented by a strategy, returns current_order_rate as default.
        If current_order_rate is returned then the existing order is maintained.
        If None is returned then order gets canceled but not replaced by a new one.

        :param pair: Pair that's currently analyzed
        :param trade: Trade object.
        :param order: Order object
        :param current_time: datetime object, containing the current datetime
        :param proposed_rate: Rate, calculated based on pricing settings in entry_pricing.
        :param current_order_rate: Rate of the existing order in place.
        :param entry_tag: Optional entry_tag (buy_tag) if provided with the buy signal.
        :param side: 'long' or 'short' - indicating the direction of the proposed trade
        :param is_entry: True if the order is an entry order, False if it's an exit order.
        :param **kwargs: Ensure to keep this here so updates to this won't break your strategy.
        :return float or None: New entry price value if provided
        """

        # Limit entry orders to use and follow SMA200 as price target for the first 10 minutes since entry trigger for BTC/USDT pair.
        if (
            is_entry
            and pair == "BTC/USDT" 
            and entry_tag == "long_sma200" 
            and side == "long" 
            and (current_time - timedelta(minutes=10)) <= trade.open_date_utc
        ):
            # just cancel the order if it has been filled more than half of the amount
            if order.filled > order.remaining:
                return None
            else:
                dataframe, _ = self.dp.get_analyzed_dataframe(pair=pair, timeframe=self.timeframe)
                current_candle = dataframe.iloc[-1].squeeze()
                # desired price
                return current_candle["sma_200"]
        # default: maintain existing order
        return current_order_rate
```

!!! danger "Incompatibility with `adjust_*_price()`"
    If you have both `adjust_order_price()` and `adjust_entry_price()`/`adjust_exit_price()` implemented, only `adjust_order_price()` will be used.
    If you need to adjust entry/exit prices, you can either implement the logic in `adjust_order_price()`, or use the split `adjust_entry_price()` / `adjust_exit_price()` callbacks, but not both.
    Mixing these is not supported and will raise an error during bot startup.

### Adjust Entry Price

The `adjust_entry_price()` callback may be used by strategy developer to refresh/replace entry limit orders upon arrival.
It's a sub-set of `adjust_order_price()` and is called only for entry orders.
All remaining behavior is identical to `adjust_order_price()`.

The trade open-date (`trade.open_date_utc`) will remain at the time of the very first order placed.
Please make sure to be aware of this - and eventually adjust your logic in other callbacks to account for this, and use the date of the first filled order instead.

### Adjust Exit Price

The `adjust_exit_price()` callback may be used by strategy developer to refresh/replace exit limit orders upon arrival.
It's a sub-set of `adjust_order_price()` and is called only for exit orders.
All remaining behavior is identical to `adjust_order_price()`.

## Leverage Callback

When trading in markets that allow leverage, this method must return the desired Leverage (Defaults to 1 -> No leverage).

Assuming a capital of 500USDT, a trade with leverage=3 would result in a position with 500 x 3 = 1500 USDT.

Values that are above `max_leverage` will be adjusted to `max_leverage`.
For markets / exchanges that don't support leverage, this method is ignored.

``` python
# Default imports

class AwesomeStrategy(IStrategy):
    def leverage(self, pair: str, current_time: datetime, current_rate: float,
                 proposed_leverage: float, max_leverage: float, entry_tag: str | None, side: str,
                 **kwargs) -> float:
        """
        Customize leverage for each new trade. This method is only called in futures mode.

        :param pair: Pair that's currently analyzed
        :param current_time: datetime object, containing the current datetime
        :param current_rate: Rate, calculated based on pricing settings in exit_pricing.
        :param proposed_leverage: A leverage proposed by the bot.
        :param max_leverage: Max leverage allowed on this pair
        :param entry_tag: Optional entry_tag (buy_tag) if provided with the buy signal.
        :param side: "long" or "short" - indicating the direction of the proposed trade
        :return: A leverage amount, which is between 1.0 and max_leverage.
        """
        return 1.0
```

All profit calculations include leverage. Stoploss / ROI also include leverage in their calculation.
Defining a stoploss of 10% at 10x leverage would trigger the stoploss with a 1% move to the downside.

## Order filled Callback

The `order_filled()` callback may be used to perform specific actions based on the current trade state after an order is filled.
It will be called independent of the order type (entry, exit, stoploss or position adjustment).

Assuming that your strategy needs to store the high value of the candle at trade entry, this is possible with this callback as the following example show.

``` python
# Default imports

class AwesomeStrategy(IStrategy):
    def order_filled(self, pair: str, trade: Trade, order: Order, current_time: datetime, **kwargs) -> None:
        """
        Called right after an order fills. 
        Will be called for all order types (entry, exit, stoploss, position adjustment).
        :param pair: Pair for trade
        :param trade: trade object.
        :param order: Order object.
        :param current_time: datetime object, containing the current datetime
        :param **kwargs: Ensure to keep this here so updates to this won't break your strategy.
        """
        # Obtain pair dataframe (just to show how to access it)
        dataframe, _ = self.dp.get_analyzed_dataframe(trade.pair, self.timeframe)
        last_candle = dataframe.iloc[-1].squeeze()
        
        if (trade.nr_of_successful_entries == 1) and (order.ft_order_side == trade.entry_side):
            trade.set_custom_data(key="entry_candle_high", value=last_candle["high"])

        return None

```

---

## File: strategy-customization.md

# Strategy Customization

This page explains how to customize your strategies, add new indicators and set up trading rules.

If you haven't already, please familiarize yourself with:

- the [Freqtrade strategy 101](strategy-101.md), which provides a quick start to strategy development
- the [Freqtrade bot basics](bot-basics.md), which provides overall info on how the bot operates

## Develop your own strategy

The bot includes a default strategy file.

Also, several other strategies are available in the [strategy repository](https://github.com/freqtrade/freqtrade-strategies).

You will however most likely have your own idea for a strategy.

This document intends to help you convert your ideas into a working strategy.

### Generating a strategy template

To get started, you can use the command:

```bash
freqtrade new-strategy --strategy AwesomeStrategy
```

This will create a new strategy called `AwesomeStrategy` from a template, which will be located using the filename `user_data/strategies/AwesomeStrategy.py`.

!!! Note
    There is a difference between the *name* of the strategy and the filename. In most commands, Freqtrade uses the *name* of the strategy, *not the filename*.

!!! Note
    The `new-strategy` command generates starting examples which will not be profitable out of the box.

??? Hint "Different template levels"
    `freqtrade new-strategy` has an additional parameter, `--template`, which controls the amount of pre-build information you get in the created strategy. Use `--template minimal` to get an empty strategy without any indicator examples, or `--template advanced` to get a template with more complicated features defined.

### Anatomy of a strategy

A strategy file contains all the information needed to build the strategy logic:

- Candle data in OHLCV format
- Indicators
- Entry logic
  - Signals
- Exit logic
  - Signals
  - Minimal ROI
  - Callbacks ("custom functions")
- Stoploss
  - Fixed/absolute
  - Trailing
  - Callbacks ("custom functions")
- Pricing [optional]
- Position adjustment [optional]

The bot includes a sample strategy called `SampleStrategy` that you can use as a basis: `user_data/strategies/sample_strategy.py`.
You can test it with the parameter: `--strategy SampleStrategy`. Remember that you use the strategy class name, not the filename.

Additionally, there is an attribute called `INTERFACE_VERSION`, which defines the version of the strategy interface the bot should use.
The current version is 3 - which is also the default when it's not set explicitly in the strategy.

You may see older strategies set to interface version 2, and these will need to be updated to v3 terminology as future versions will require this to be set.

Starting the bot in dry or live mode is accomplished using the `trade` command:

```bash
freqtrade trade --strategy AwesomeStrategy
```

### Bot modes

Freqtrade strategies can be processed by the Freqtrade bot in 5 main modes:

- backtesting
- hyperopting
- dry ("forward testing")
- live
- FreqAI (not covered here)

Check the [configuration documentation](configuration.md) about how to set the bot to dry or live mode.

**Always use dry mode when testing as this gives you an idea of how your strategy will work in reality without risking capital.**

## Diving in deeper
**For the following section we will use the [user_data/strategies/sample_strategy.py](https://github.com/freqtrade/freqtrade/blob/develop/freqtrade/templates/sample_strategy.py)
file as reference.**

!!! Note "Strategies and Backtesting"
    To avoid problems and unexpected differences between backtesting and dry/live modes, please be aware
    that during backtesting the full time range is passed to the `populate_*()` methods at once.
    It is therefore best to use vectorized operations (across the whole dataframe, not loops) and
    avoid index referencing (`df.iloc[-1]`), but instead use `df.shift()` to get to the previous candle.

!!! Warning "Warning: Using future data"
    Since backtesting passes the full time range to the `populate_*()` methods, the strategy author
    needs to take care to avoid having the strategy utilize data from the future.
    Some common patterns for this are listed in the [Common Mistakes](#common-mistakes-when-developing-strategies) section of this document.

??? Hint "Lookahead and recursive analysis"
    Freqtrade includes two helpful commands to help assess common lookahead (using future data) and 
    recursive bias (variance in indicator values) issues. Before running a strategy in dry or live more, 
    you should always use these commands first. Please check the relevant documentation for 
    [lookahead](lookahead-analysis.md) and [recursive](recursive-analysis.md) analysis.

### Dataframe

Freqtrade uses [pandas](https://pandas.pydata.org/) to store/provide the candlestick (OHLCV) data.
Pandas is a great library developed for processing large amounts of data in tabular format.

Each row in a dataframe corresponds to one candle on a chart, with the latest complete candle always being the last in the dataframe (sorted by date).

If we were to look at the first few rows of the main dataframe using the pandas `head()` function, we would see:

```output
> dataframe.head()
                       date      open      high       low     close     volume
0 2021-11-09 23:25:00+00:00  67279.67  67321.84  67255.01  67300.97   44.62253
1 2021-11-09 23:30:00+00:00  67300.97  67301.34  67183.03  67187.01   61.38076
2 2021-11-09 23:35:00+00:00  67187.02  67187.02  67031.93  67123.81  113.42728
3 2021-11-09 23:40:00+00:00  67123.80  67222.40  67080.33  67160.48   78.96008
4 2021-11-09 23:45:00+00:00  67160.48  67160.48  66901.26  66943.37  111.39292
```

A dataframe is a table where columns are not single values, but a series of data values. As such, simple python comparisons like the following will not work:

``` python
    if dataframe['rsi'] > 30:
        dataframe['enter_long'] = 1
```

The above section will fail with `The truth value of a Series is ambiguous [...]`.

This must instead be written in a pandas-compatible way, so the operation is performed across the whole dataframe, i.e. `vectorisation`.

``` python
    dataframe.loc[
        (dataframe['rsi'] > 30)
    , 'enter_long'] = 1
```

With this section, you have a new column in your dataframe, which has `1` assigned whenever RSI is above 30.

Freqtrade uses this new column as an entry signal, where it is assumed that a trade will subsequently open on the next open candle.

Pandas provides fast ways to calculate metrics, i.e. "vectorisation". To benefit from this speed, it is advised to not use loops, but use vectorized methods instead.

Vectorized operations perform calculations across the whole range of data and are therefore, compared to looping through each row, a lot faster when calculating indicators.

??? Hint "Signals vs Trades"
    - Signals are generated from indicators at candle close, and are intentions to enter a trade.
    - Trades are orders that are executed (on the exchange in live mode) where a trade will then open as close to next candle open as possible.

!!! Warning "Trade order assumptions"
    In backtesting, signals are generated on candle close. Trades are then initiated immeditely on next candle open.
    
    In dry and live, this may be delayed due to all pair dataframes needing to be analysed first, then trade processing 
    for each of those pairs happens. This means that in dry/live you need to be mindful of having as low a computation 
    delay as possible, usually by running a low number of pairs and having a CPU with a good clock speed.

#### Why can't I see "real time" candle data?

Freqtrade does not store incomplete/unfinished candles in the dataframe.

The use of incomplete data for making strategy decisions is called "repainting" and you might see other platforms allow this.

Freqtrade does not. Only complete/finished candle data is available in the dataframe.

### Customize Indicators

Entry and exit signals need indicators. You can add more indicators by extending the list contained in the method `populate_indicators()` from your strategy file.

You should only add the indicators used in either `populate_entry_trend()`, `populate_exit_trend()`, or to populate another indicator, otherwise performance may suffer.

It's important to always return the dataframe from these three functions without removing/modifying the columns `"open", "high", "low", "close", "volume"`, otherwise these fields would contain something unexpected.

Sample:

```python
def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    """
    Adds several different TA indicators to the given DataFrame

    Performance Note: For the best performance be frugal on the number of indicators
    you are using. Let uncomment only the indicator you are using in your strategies
    or your hyperopt configuration, otherwise you will waste your memory and CPU usage.
    :param dataframe: Dataframe with data from the exchange
    :param metadata: Additional information, like the currently traded pair
    :return: a Dataframe with all mandatory indicators for the strategies
    """
    dataframe['sar'] = ta.SAR(dataframe)
    dataframe['adx'] = ta.ADX(dataframe)
    stoch = ta.STOCHF(dataframe)
    dataframe['fastd'] = stoch['fastd']
    dataframe['fastk'] = stoch['fastk']
    dataframe['bb_lower'] = ta.BBANDS(dataframe, nbdevup=2, nbdevdn=2)['lowerband']
    dataframe['sma'] = ta.SMA(dataframe, timeperiod=40)
    dataframe['tema'] = ta.TEMA(dataframe, timeperiod=9)
    dataframe['mfi'] = ta.MFI(dataframe)
    dataframe['rsi'] = ta.RSI(dataframe)
    dataframe['ema5'] = ta.EMA(dataframe, timeperiod=5)
    dataframe['ema10'] = ta.EMA(dataframe, timeperiod=10)
    dataframe['ema50'] = ta.EMA(dataframe, timeperiod=50)
    dataframe['ema100'] = ta.EMA(dataframe, timeperiod=100)
    dataframe['ao'] = awesome_oscillator(dataframe)
    macd = ta.MACD(dataframe)
    dataframe['macd'] = macd['macd']
    dataframe['macdsignal'] = macd['macdsignal']
    dataframe['macdhist'] = macd['macdhist']
    hilbert = ta.HT_SINE(dataframe)
    dataframe['htsine'] = hilbert['sine']
    dataframe['htleadsine'] = hilbert['leadsine']
    dataframe['plus_dm'] = ta.PLUS_DM(dataframe)
    dataframe['plus_di'] = ta.PLUS_DI(dataframe)
    dataframe['minus_dm'] = ta.MINUS_DM(dataframe)
    dataframe['minus_di'] = ta.MINUS_DI(dataframe)

    # remember to always return the dataframe
    return dataframe
```

!!! Note "Want more indicator examples?"
    Look into the [user_data/strategies/sample_strategy.py](https://github.com/freqtrade/freqtrade/blob/develop/freqtrade/templates/sample_strategy.py).
    Then uncomment indicators you need.

#### Indicator libraries

Out of the box, freqtrade installs the following technical libraries:

- [ta-lib](https://ta-lib.github.io/ta-lib-python/)
- [pandas-ta](https://twopirllc.github.io/pandas-ta/)
- [technical](https://technical.freqtrade.io)

Additional technical libraries can be installed as necessary, or custom indicators may be written / invented by the strategy author.

### Strategy startup period

Some indicators have an unstable startup period in which there isn't enough candle data to calculate any values (NaN), or the calculation is incorrect. This can lead to inconsistencies, since Freqtrade does not know how long this unstable period is and uses whatever indicator values are in the dataframe.

To account for this, the strategy can be assigned the `startup_candle_count` attribute.

This should be set to the maximum number of candles that the strategy requires to calculate stable indicators. In the case where a user includes higher timeframes with informative pairs, the `startup_candle_count` does not necessarily change. The value is the maximum period (in candles) that any of the informatives timeframes need to compute stable indicators.

You can use [recursive-analysis](recursive-analysis.md) to check and find the correct `startup_candle_count` to be used. When recursive analysis shows a variance of 0%, then you can be sure that you have enough startup candle data.

In this example strategy, this should be set to 400 (`startup_candle_count = 400`), since the minimum needed history for ema100 calculation to make sure the value is correct is 400 candles.

``` python
    dataframe['ema100'] = ta.EMA(dataframe, timeperiod=100)
```

By letting the bot know how much history is needed, backtest trades can start at the specified timerange during backtesting and hyperopt.

!!! Warning "Using x calls to get OHLCV"
    If you receive a warning like `WARNING - Using 3 calls to get OHLCV. This can result in slower operations for the bot. Please check if you really need 1500 candles for your strategy` - you should consider if you really need this much historic data for your signals.
    Having this will cause Freqtrade to make multiple calls for the same pair, which will obviously be slower than one network request.
    As a consequence, Freqtrade will take longer to refresh candles - and should therefore be avoided if possible.
    This is capped to 5 total calls to avoid overloading the exchange, or make freqtrade too slow.

!!! Warning
    `startup_candle_count` should be below `ohlcv_candle_limit * 5` (which is 500 * 5 for most exchanges) - since only this amount of candles will be available during Dry-Run/Live Trade operations.

#### Example

Let's try to backtest 1 month (January 2019) of 5m candles using an example strategy with EMA100, as above.

``` bash
freqtrade backtesting --timerange 20190101-20190201 --timeframe 5m
```

Assuming `startup_candle_count` is set to 400, backtesting knows it needs 400 candles to generate valid entry signals. It will load data from `20190101 - (400 * 5m)` - which is ~2018-12-30 11:40:00.

If this data is available, indicators will be calculated with this extended timerange. The unstable startup period (up to 2019-01-01 00:00:00) will then be removed before backtesting is carried out.

!!! Note "Unavailable startup candle data"
    If data for the startup period is not available, then the timerange will be adjusted to account for this startup period. In our example, backtesting would then start from 2019-01-02 09:20:00.

### Entry signal rules

Edit the method `populate_entry_trend()` in your strategy file to update your entry strategy.

It's important to always return the dataframe without removing/modifying the columns `"open", "high", "low", "close", "volume"`, otherwise these fields would contain something unexpected. The strategy may then produce invalid values, or cease to work entirely.

This method will also define a new column, `"enter_long"` (`"enter_short"` for shorts), which needs to contain `1` for entries, and `0` for "no action". `enter_long` is a mandatory column that must be set even if the strategy is shorting only.

You can name your entry signals by using the `"enter_tag"` column, which can help debug and assess your strategy later. 

Sample from `user_data/strategies/sample_strategy.py`:

```python
def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    """
    Based on TA indicators, populates the buy signal for the given dataframe
    :param dataframe: DataFrame populated with indicators
    :param metadata: Additional information, like the currently traded pair
    :return: DataFrame with buy column
    """
    dataframe.loc[
        (
            (qtpylib.crossed_above(dataframe['rsi'], 30)) &  # Signal: RSI crosses above 30
            (dataframe['tema'] <= dataframe['bb_middleband']) &  # Guard
            (dataframe['tema'] > dataframe['tema'].shift(1)) &  # Guard
            (dataframe['volume'] > 0)  # Make sure Volume is not 0
        ),
        ['enter_long', 'enter_tag']] = (1, 'rsi_cross')

    return dataframe
```

??? Note "Enter short trades"
    Short entries can be created by setting `enter_short` (corresponds to `enter_long` for long trades).
    The `enter_tag` column remains identical.
    Shorting needs to be supported by your exchange and market configuration!
    Also, make sure you set [`can_short`](#can-short) appropriately on your strategy if you intend to short.

    ```python
    # allow both long and short trades
    can_short = True

    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        dataframe.loc[
            (
                (qtpylib.crossed_above(dataframe['rsi'], 30)) &  # Signal: RSI crosses above 30
                (dataframe['tema'] <= dataframe['bb_middleband']) &  # Guard
                (dataframe['tema'] > dataframe['tema'].shift(1)) &  # Guard
                (dataframe['volume'] > 0)  # Make sure Volume is not 0
            ),
            ['enter_long', 'enter_tag']] = (1, 'rsi_cross')

        dataframe.loc[
            (
                (qtpylib.crossed_below(dataframe['rsi'], 70)) &  # Signal: RSI crosses below 70
                (dataframe['tema'] > dataframe['bb_middleband']) &  # Guard
                (dataframe['tema'] < dataframe['tema'].shift(1)) &  # Guard
                (dataframe['volume'] > 0)  # Make sure Volume is not 0
            ),
            ['enter_short', 'enter_tag']] = (1, 'rsi_cross')

        return dataframe
    ```

!!! Note
    Buying requires sellers to buy from. Therefore volume needs to be > 0 (`dataframe['volume'] > 0`) to make sure that the bot does not buy/sell in no-activity periods.

### Exit signal rules

Edit the method `populate_exit_trend()` into your strategy file to update your exit strategy.

The exit-signal can be suppressed by setting `use_exit_signal` to false in the configuration or strategy.

`use_exit_signal` will not influence [signal collision rules](#colliding-signals) - which will still apply and can prevent entries.

It's important to always return the dataframe without removing/modifying the columns `"open", "high", "low", "close", "volume"`, otherwise these fields would contain something unexpected. The strategy may then produce invalid values, or cease to work entirely.

This method will also define a new column, `"exit_long"` (`"exit_short"` for shorts), which needs to contain `1` for exits, and `0` for "no action".

You can name your exit signals by using the `"exit_tag"` column, which can help debug and assess your strategy later.

Sample from `user_data/strategies/sample_strategy.py`:

```python
def populate_exit_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    """
    Based on TA indicators, populates the exit signal for the given dataframe
    :param dataframe: DataFrame populated with indicators
    :param metadata: Additional information, like the currently traded pair
    :return: DataFrame with buy column
    """
    dataframe.loc[
        (
            (qtpylib.crossed_above(dataframe['rsi'], 70)) &  # Signal: RSI crosses above 70
            (dataframe['tema'] > dataframe['bb_middleband']) &  # Guard
            (dataframe['tema'] < dataframe['tema'].shift(1)) &  # Guard
            (dataframe['volume'] > 0)  # Make sure Volume is not 0
        ),
        ['exit_long', 'exit_tag']] = (1, 'rsi_too_high')
    return dataframe
```

??? Note "Exit short trades"
    Short exits can be created by setting `exit_short` (corresponds to `exit_long`).
    The `exit_tag` column remains identical.
    Shorting needs to be supported by your exchange and market configuration!
    Also, make sure you set [`can_short`](#can-short) appropriately on your strategy if you intend to short.

    ```python
    # allow both long and short trades
    can_short = True

    def populate_exit_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        dataframe.loc[
            (
                (qtpylib.crossed_above(dataframe['rsi'], 70)) &  # Signal: RSI crosses above 70
                (dataframe['tema'] > dataframe['bb_middleband']) &  # Guard
                (dataframe['tema'] < dataframe['tema'].shift(1)) &  # Guard
                (dataframe['volume'] > 0)  # Make sure Volume is not 0
            ),
            ['exit_long', 'exit_tag']] = (1, 'rsi_too_high')
        dataframe.loc[
            (
                (qtpylib.crossed_below(dataframe['rsi'], 30)) &  # Signal: RSI crosses below 30
                (dataframe['tema'] < dataframe['bb_middleband']) &  # Guard
                (dataframe['tema'] > dataframe['tema'].shift(1)) &  # Guard
                (dataframe['volume'] > 0)  # Make sure Volume is not 0
            ),
            ['exit_short', 'exit_tag']] = (1, 'rsi_too_low')
        return dataframe
    ```

### Minimal ROI

The `minimal_roi` strategy variable defines the minimal Return On Investment (ROI) a trade should reach before exiting, independent from the exit signal.

It is of the following format, i.e. a python `dict`, with the dict key (left side of the colon) being the minutes passed since the trade opened, and the value (right side of the colon) being the percentage.

```python
minimal_roi = {
    "40": 0.0,
    "30": 0.01,
    "20": 0.02,
    "0": 0.04
}
```

The above configuration would therefore mean:

- Exit whenever 4% profit was reached
- Exit when 2% profit was reached (in effect after 20 minutes)
- Exit when 1% profit was reached (in effect after 30 minutes)
- Exit when trade is non-loosing (in effect after 40 minutes)

The calculation does include fees.

#### Disabling minimal ROI

To disable ROI completely, set it to an empty dictionary:

```python
minimal_roi = {}
```

#### Using calculations in minimal ROI

To use times based on candle duration (timeframe), the following snippet can be handy.

This will allow you to change the timeframe for the strategy, but the minimal ROI times will still be set as candles, e.g. after 3 candles.

``` python
from freqtrade.exchange import timeframe_to_minutes

class AwesomeStrategy(IStrategy):

    timeframe = "1d"
    timeframe_mins = timeframe_to_minutes(timeframe)
    minimal_roi = {
        "0": 0.05,                      # 5% for the first 3 candles
        str(timeframe_mins * 3): 0.02,  # 2% after 3 candles
        str(timeframe_mins * 6): 0.01,  # 1% After 6 candles
    }
```

??? info "Orders that don't fill immediately"
    `minimal_roi` will take the `trade.open_date` as reference, which is the time the trade was initialized, i.e. when the first order for this trade was placed.
    This will also hold true for limit orders that don't fill immediately (usually in combination with "off-spot" prices through `custom_entry_price()`), as well as for cases where the initial order price is replaced through `adjust_entry_price()`.
    The time used will still be from the initial `trade.open_date` (when the initial order was first placed), not from the newly placed or adjusted order date.

### Stoploss

Setting a stoploss is highly recommended to protect your capital from strong moves against you.

Sample of setting a 10% stoploss:

``` python
stoploss = -0.10
```

For the full documentation on stoploss features, look at the dedicated [stoploss page](stoploss.md).

### Timeframe

This is the periodicity of candles the bot should use in the strategy.

Common values are `"1m"`, `"5m"`, `"15m"`, `"1h"`, however all values supported by your exchange should work.

Please note that the same entry/exit signals may work well with one timeframe, but not with others.

This setting is accessible within the strategy methods as the `self.timeframe` attribute.

### Can short

To use short signals in futures markets, you will have to set `can_short = True`.

Strategies which enable this will fail to load on spot markets.

If you have `1` values in the `enter_short` column to raise short signals, setting `can_short = False` (which is the default) will mean that these short signals are ignored, even if you have specified futures markets in your configuration.

### Metadata dict

The `metadata` dict (available for `populate_entry_trend`, `populate_exit_trend`, `populate_indicators`) contains additional information.
Currently this is `pair`, which can be accessed using `metadata['pair']`, and will return a pair in the format `XRP/BTC` (or `XRP/BTC:BTC` for futures markets).

The metadata dict should not be modified and does not persist information across multiple functions in your strategy.

Instead, please check the [Storing information](strategy-advanced.md#storing-information-persistent) section.

--8<-- "includes/strategy-imports.md"

## Strategy file loading

By default, freqtrade will attempt to load strategies from all `.py` files within the `userdir` (default `user_data/strategies`).

Assuming your strategy is called `AwesomeStrategy`, stored in the file `user_data/strategies/AwesomeStrategy.py`, then you can start freqtrade in dry (or live, depending on your configuration) mode with:

```bash
freqtrade trade --strategy AwesomeStrategy
```

Note that we're using the class name, not the file name.

You can use `freqtrade list-strategies` to see a list of all strategies Freqtrade is able to load (all strategies in the correct folder).
It will also include a "status" field, highlighting potential problems.

??? Hint "Customize strategy directory"
    You can use a different directory by using `--strategy-path user_data/otherPath`. This parameter is available to all commands that require a strategy.

## Informative Pairs

### Get data for non-tradeable pairs

Data for additional, informative pairs (reference pairs) can be beneficial for some strategies to see data on a wider timeframe.

OHLCV data for these pairs will be downloaded as part of the regular whitelist refresh process and is available via `DataProvider` just as other pairs (see below).

These pairs will **not** be traded unless they are also specified in the pair whitelist, or have been selected by Dynamic Whitelisting, e.g. `VolumePairlist`.

The pairs need to be specified as tuples in the format `("pair", "timeframe")`, with pair as the first and timeframe as the second argument.

Sample:

``` python
def informative_pairs(self):
    return [("ETH/USDT", "5m"),
            ("BTC/TUSD", "15m"),
            ]
```

A full sample can be found [in the DataProvider section](#complete-dataprovider-sample).

!!! Warning
    As these pairs will be refreshed as part of the regular whitelist refresh, it's best to keep this list short.
    All timeframes and all pairs can be specified as long as they are available (and active) on the used exchange.
    It is however better to use resampling to longer timeframes whenever possible
    to avoid hammering the exchange with too many requests and risk being blocked.

??? Note "Alternative candle types"
    Informative_pairs can also provide a 3rd tuple element defining the candle type explicitly.
    Availability of alternative candle-types will depend on the trading-mode and the exchange. 
    In general, spot pairs cannot be used in futures markets, and futures candles can't be used as informative pairs for spot bots.
    Details about this may vary, if they do, this can be found in the exchange documentation.

    ``` python
    def informative_pairs(self):
        return [
            ("ETH/USDT", "5m", ""),   # Uses default candletype, depends on trading_mode (recommended)
            ("ETH/USDT", "5m", "spot"),   # Forces usage of spot candles (only valid for bots running on spot markets).
            ("BTC/TUSD", "15m", "futures"),  # Uses futures candles (only bots with `trading_mode=futures`)
            ("BTC/TUSD", "15m", "mark"),  # Uses mark candles (only bots with `trading_mode=futures`)
        ]
    ```
***

### Informative pairs decorator (`@informative()`)

To easily define informative pairs, use the `@informative` decorator. All decorated `populate_indicators_*` methods run in isolation,
and do not have access to data from other informative pairs. However, all informative dataframes for each pair are merged and passed to main `populate_indicators()` method.

!!! Note
    Do not use the `@informative` decorator if you need to use data from one informative pair when generating another informative pair. Instead, define informative pairs manually as described [in the DataProvider section](#complete-dataprovider-sample).

When hyperopting, use of the hyperoptable parameter `.value` attribute is not supported. Please use the `.range` attribute. See [optimizing an indicator parameter](hyperopt.md#optimizing-an-indicator-parameter) for more information.

??? info "Full documentation"
    ``` python
    def informative(
        timeframe: str,
        asset: str = "",
        fmt: str | Callable[[Any], str] | None = None,
        *,
        candle_type: CandleType | str | None = None,
        ffill: bool = True,
    ) -> Callable[[PopulateIndicators], PopulateIndicators]:
        """
        A decorator for populate_indicators_Nn(self, dataframe, metadata), allowing these functions to
        define informative indicators.

        Example usage:

            @informative('1h')
            def populate_indicators_1h(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
                dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)
                return dataframe

        :param timeframe: Informative timeframe. Must always be equal or higher than strategy timeframe.
        :param asset: Informative asset, for example BTC, BTC/USDT, ETH/BTC. Do not specify to use
                    current pair. Also supports limited pair format strings (see below)
        :param fmt: Column format (str) or column formatter (callable(name, asset, timeframe)). When not
        specified, defaults to:
        * {base}_{quote}_{column}_{timeframe} if asset is specified.
        * {column}_{timeframe} if asset is not specified.
        Pair format supports these format variables:
        * {base} - base currency in lower case, for example 'eth'.
        * {BASE} - same as {base}, except in upper case.
        * {quote} - quote currency in lower case, for example 'usdt'.
        * {QUOTE} - same as {quote}, except in upper case.
        Format string additionally supports this variables.
        * {asset} - full name of the asset, for example 'BTC/USDT'.
        * {column} - name of dataframe column.
        * {timeframe} - timeframe of informative dataframe.
        :param ffill: ffill dataframe after merging informative pair.
        :param candle_type: '', mark, index, premiumIndex, or funding_rate
        """
    ```

??? Example "Fast and easy way to define informative pairs"

    Most of the time we do not need power and flexibility offered by `merge_informative_pair()`, therefore we can use a decorator to quickly define informative pairs.

    ``` python

    from datetime import datetime
    from freqtrade.persistence import Trade
    from freqtrade.strategy import IStrategy, informative

    class AwesomeStrategy(IStrategy):
        
        # This method is not required. 
        # def informative_pairs(self): ...

        # Define informative upper timeframe for each pair. Decorators can be stacked on same 
        # method. Available in populate_indicators as 'rsi_30m' and 'rsi_1h'.
        @informative('30m')
        @informative('1h')
        def populate_indicators_1h(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
            dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)
            return dataframe

        # Define BTC/STAKE informative pair. Available in populate_indicators and other methods as
        # 'btc_rsi_1h'. Current stake currency should be specified as {stake} format variable 
        # instead of hard-coding actual stake currency. Available in populate_indicators and other 
        # methods as 'btc_usdt_rsi_1h' (when stake currency is USDT).
        @informative('1h', 'BTC/{stake}')
        def populate_indicators_btc_1h(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
            dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)
            return dataframe

        # Define BTC/ETH informative pair. You must specify quote currency if it is different from
        # stake currency. Available in populate_indicators and other methods as 'eth_btc_rsi_1h'.
        @informative('1h', 'ETH/BTC')
        def populate_indicators_eth_btc_1h(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
            dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)
            return dataframe
    
        # Define BTC/STAKE informative pair. A custom formatter may be specified for formatting
        # column names. A callable `fmt(**kwargs) -> str` may be specified, to implement custom
        # formatting. Available in populate_indicators and other methods as 'rsi_upper_1h'.
        @informative('1h', 'BTC/{stake}', '{column}_{timeframe}')
        def populate_indicators_btc_1h_2(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
            dataframe['rsi_upper'] = ta.RSI(dataframe, timeperiod=14)
            return dataframe
    
        def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
            # Strategy timeframe indicators for current pair.
            dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)
            # Informative pairs are available in this method.
            dataframe['rsi_less'] = dataframe['rsi'] < dataframe['rsi_1h']
            return dataframe

    ```

!!! Note
    Use string formatting when accessing informative dataframes of other pairs. This will allow easily changing stake currency in config without having to adjust strategy code.

    ``` python
    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        stake = self.config['stake_currency']
        dataframe.loc[
            (
                (dataframe[f'btc_{stake}_rsi_1h'] < 35)
                &
                (dataframe['volume'] > 0)
            ),
            ['enter_long', 'enter_tag']] = (1, 'buy_signal_rsi')
    
        return dataframe
    ```

    Alternatively column renaming may be used to remove stake currency from column names: `@informative('1h', 'BTC/{stake}', fmt='{base}_{column}_{timeframe}')`.

!!! Warning "Duplicate method names"
    Methods tagged with the `@informative()` decorator must always have unique names! Reusing the same name (for example when copy-pasting already defined informative methods) will overwrite previously defined methods and not produce any errors due to limitations of Python programming language. In such cases you will find that indicators created in methods higher up in the strategy file are not available in the dataframe. Carefully review method names and make sure they are unique!

### *merge_informative_pair()*

This method helps you merge an informative pair to the regular main dataframe safely and consistently, without lookahead bias.

Options:

- Rename the columns to create unique columns
- Merge the dataframe without lookahead bias
- Forward-fill (optional)

For a full sample, please refer to the [complete data provider example](#complete-dataprovider-sample) below.

All columns of the informative dataframe will be available on the returning dataframe in a renamed fashion:

!!! Example "Column renaming"
    Assuming `inf_tf = '1d'` the resulting columns will be:

    ``` python
    'date', 'open', 'high', 'low', 'close', 'rsi'                     # from the original dataframe
    'date_1d', 'open_1d', 'high_1d', 'low_1d', 'close_1d', 'rsi_1d'   # from the informative dataframe
    ```

??? Example "Column renaming - 1h"
    Assuming `inf_tf = '1h'` the resulting columns will be:

    ``` python
    'date', 'open', 'high', 'low', 'close', 'rsi'                     # from the original dataframe
    'date_1h', 'open_1h', 'high_1h', 'low_1h', 'close_1h', 'rsi_1h'   # from the informative dataframe
    ```

??? Example "Custom implementation"
    A custom implementation for this is possible, and can be done as follows:

    ``` python

    # Shift date by 1 candle
    # This is necessary since the data is always the "open date"
    # and a 15m candle starting at 12:15 should not know the close of the 1h candle from 12:00 to 13:00
    minutes = timeframe_to_minutes(inf_tf)
    # Only do this if the timeframes are different:
    informative['date_merge'] = informative["date"] + pd.to_timedelta(minutes, 'm')

    # Rename columns to be unique
    informative.columns = [f"{col}_{inf_tf}" for col in informative.columns]
    # Assuming inf_tf = '1d' - then the columns will now be:
    # date_1d, open_1d, high_1d, low_1d, close_1d, rsi_1d

    # Combine the 2 dataframes
    # all indicators on the informative sample MUST be calculated before this point
    dataframe = pd.merge(dataframe, informative, left_on='date', right_on=f'date_merge_{inf_tf}', how='left')
    # FFill to have the 1d value available in every row throughout the day.
    # Without this, comparisons would only work once per day.
    dataframe = dataframe.ffill()

    ```

!!! Warning "Informative timeframe < timeframe"
    Using informative timeframes smaller than the main dataframe timeframe is not recommended with this method, as it will not use any of the additional information this would provide.
    To use the more detailed information properly, more advanced methods should be applied (which are out of scope for this documentation).

## Additional data (DataProvider)

The strategy provides access to the `DataProvider`. This allows you to get additional data to use in your strategy.

All methods return `None` in case of failure, i.e. failures do not raise an exception.

Please always check the mode of operation to select the correct method to get data (see below for examples).

!!! Warning "Hyperopt Limitations"
    The DataProvider is available during hyperopt, however it can only be used in `populate_indicators()` **within a strategy**, not within a hyperopt class file.
    It is also not available in `populate_entry_trend()` and `populate_exit_trend()` methods.

### Possible options for DataProvider

- [`available_pairs`](#available_pairs) - Property with tuples listing cached pairs with their timeframe (pair, timeframe).
- [`current_whitelist()`](#current_whitelist) - Returns a current list of whitelisted pairs. Useful for accessing dynamic whitelists (i.e. VolumePairlist)
- [`get_pair_dataframe(pair, timeframe)`](#get_pair_dataframepair-timeframe) - This is a universal method, which returns either historical data (for backtesting) or cached live data (for the Dry-Run and Live-Run modes).
- [`get_analyzed_dataframe(pair, timeframe)`](#get_analyzed_dataframepair-timeframe) - Returns the analyzed dataframe (after calling `populate_indicators()`, `populate_buy()`, `populate_sell()`) and the time of the latest analysis.
- `historic_ohlcv(pair, timeframe)` - Returns historical data stored on disk.
- `market(pair)` - Returns market data for the pair: fees, limits, precisions, activity flag, etc. See [ccxt documentation](https://github.com/ccxt/ccxt/wiki/Manual#markets) for more details on the Market data structure.
- `ohlcv(pair, timeframe)` - Currently cached candle (OHLCV) data for the pair, returns DataFrame or empty DataFrame.
- [`orderbook(pair, maximum)`](#orderbookpair-maximum) - Returns latest orderbook data for the pair, a dict with bids/asks with a total of `maximum` entries.
- [`ticker(pair)`](#tickerpair) - Returns current ticker data for the pair. See [ccxt documentation](https://github.com/ccxt/ccxt/wiki/Manual#price-tickers) for more details on the Ticker data structure.
- `runmode` - Property containing the current runmode.

### Example Usages

### *available_pairs*

``` python
for pair, timeframe in self.dp.available_pairs:
    print(f"available {pair}, {timeframe}")
```

### *current_whitelist()*

Imagine you've developed a strategy that trades the `5m` timeframe using signals generated from a `1d` timeframe on the top 10 exchange pairs by volume.

The strategy logic might look something like this:

*Scan through the top 10 pairs by volume using the `VolumePairList` every 5 minutes and use a 14 day RSI to enter and exit.*

Due to the limited available data, it's very difficult to resample `5m` candles into daily candles for use in a 14 day RSI. Most exchanges limit users to just 500-1000 candles which effectively gives us around 1.74 daily candles. We need 14 days at least!

Since we can't resample the data we will have to use an informative pair, and since the whitelist will be dynamic we don't know which pair(s) to use! We have a problem!

This is where calling `self.dp.current_whitelist()` comes in handy to retrieve only those pairs in the whitelist.

```python
    def informative_pairs(self):

        # get access to all pairs available in whitelist.
        pairs = self.dp.current_whitelist()
        # Assign timeframe to each pair so they can be downloaded and cached for strategy.
        informative_pairs = [(pair, '1d') for pair in pairs]
        return informative_pairs
```

??? Note "Plotting with current_whitelist"
    Current whitelist is not supported for `plot-dataframe`, as this command is usually used by providing an explicit pairlist and would therefore make the return values of this method misleading.
    It's also not supported for FreqUI visualization in [webserver mode](utils.md#webserver-mode), as the configuration for webserver mode doesn't require a pairlist to be set.

### *get_pair_dataframe(pair, timeframe)*

``` python
# fetch live / historical candle (OHLCV) data for the first informative pair
inf_pair, inf_timeframe = self.informative_pairs()[0]
informative = self.dp.get_pair_dataframe(pair=inf_pair,
                                         timeframe=inf_timeframe)
```

!!! Warning "Warning about backtesting"
    In backtesting, `dp.get_pair_dataframe()` behavior differs depending on where it's called.
    Within `populate_*()` methods, `dp.get_pair_dataframe()` returns the full timerange. Please make sure to not "look into the future" to avoid surprises when running in dry/live mode.
    Within [callbacks](strategy-callbacks.md), you'll get the full timerange up to the current (simulated) candle.

### *get_analyzed_dataframe(pair, timeframe)*

This method is used by freqtrade internally to determine the last signal.
It can also be used in specific callbacks to get the signal that caused the action (see [Advanced Strategy Documentation](strategy-advanced.md) for more details on available callbacks).

``` python
# fetch current dataframe
dataframe, last_updated = self.dp.get_analyzed_dataframe(pair=metadata['pair'],
                                                         timeframe=self.timeframe)
```

!!! Note "No data available"
    Returns an empty dataframe if the requested pair was not cached.
    You can check for this with `if dataframe.empty:` and handle this case accordingly.
    This should not happen when using whitelisted pairs.

### *orderbook(pair, maximum)*

``` python
if self.dp.runmode.value in ('live', 'dry_run'):
    ob = self.dp.orderbook(metadata['pair'], 1)
    dataframe['best_bid'] = ob['bids'][0][0]
    dataframe['best_ask'] = ob['asks'][0][0]
```

The orderbook structure is aligned with the order structure from [ccxt](https://github.com/ccxt/ccxt/wiki/Manual#order-book-structure), so the result will be formatted as follows:

``` js
{
    'bids': [
        [ price, amount ], // [ float, float ]
        [ price, amount ],
        ...
    ],
    'asks': [
        [ price, amount ],
        [ price, amount ],
        //...
    ],
    //...
}
```

Therefore, using `ob['bids'][0][0]` as demonstrated above will use the best bid price. `ob['bids'][0][1]` would look at the amount at this orderbook position.

!!! Warning "Warning about backtesting"
    The order book is not part of the historic data which means backtesting and hyperopt will not work correctly if this method is used, as the method will return up-to-date values.

### *ticker(pair)*

``` python
if self.dp.runmode.value in ('live', 'dry_run'):
    ticker = self.dp.ticker(metadata['pair'])
    dataframe['last_price'] = ticker['last']
    dataframe['volume24h'] = ticker['quoteVolume']
    dataframe['vwap'] = ticker['vwap']
```

!!! Warning
    Although the ticker data structure is a part of the ccxt Unified Interface, the values returned by this method can
    vary for different exchanges. For instance, many exchanges do not return `vwap` values, and some exchanges
    do not always fill in the `last` field (so it can be None), etc. So you need to carefully verify the ticker
    data returned from the exchange and add appropriate error handling / defaults.

!!! Warning "Warning about backtesting"
    This method will always return up-to-date / real-time values. As such, usage during backtesting / hyperopt without runmode checks will lead to wrong results, e.g. your whole dataframe will contain the same single value in all rows.

### Send Notification

The dataprovider `.send_msg()` function allows you to send custom notifications from your strategy.
Identical notifications will only be sent once per candle, unless the 2nd argument (`always_send`) is set to True.

``` python
    self.dp.send_msg(f"{metadata['pair']} just got hot!")

    # Force send this notification, avoid caching (Please read warning below!)
    self.dp.send_msg(f"{metadata['pair']} just got hot!", always_send=True)
```

Notifications will only be sent in trading modes (Live/Dry-run) - so this method can be called without conditions for backtesting.

!!! Warning "Spamming"
    You can spam yourself pretty good by setting `always_send=True` in this method. Use this with great care and only in conditions you know will not happen throughout a candle to avoid a message every 5 seconds.

### Complete DataProvider sample

```python
from freqtrade.strategy import IStrategy, merge_informative_pair
from pandas import DataFrame

class SampleStrategy(IStrategy):
    # strategy init stuff...

    timeframe = '5m'

    # more strategy init stuff..

    def informative_pairs(self):

        # get access to all pairs available in whitelist.
        pairs = self.dp.current_whitelist()
        # Assign tf to each pair so they can be downloaded and cached for strategy.
        informative_pairs = [(pair, '1d') for pair in pairs]
        # Optionally Add additional "static" pairs
        informative_pairs += [("ETH/USDT", "5m"),
                              ("BTC/TUSD", "15m"),
                            ]
        return informative_pairs

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        if not self.dp:
            # Don't do anything if DataProvider is not available.
            return dataframe

        inf_tf = '1d'
        # Get the informative pair
        informative = self.dp.get_pair_dataframe(pair=metadata['pair'], timeframe=inf_tf)
        # Get the 14 day rsi
        informative['rsi'] = ta.RSI(informative, timeperiod=14)

        # Use the helper function merge_informative_pair to safely merge the pair
        # Automatically renames the columns and merges a shorter timeframe dataframe and a longer timeframe informative pair
        # use ffill to have the 1d value available in every row throughout the day.
        # Without this, comparisons between columns of the original and the informative pair would only work once per day.
        # Full documentation of this method, see below
        dataframe = merge_informative_pair(dataframe, informative, self.timeframe, inf_tf, ffill=True)

        # Calculate rsi of the original dataframe (5m timeframe)
        dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)

        # Do other stuff
        # ...

        return dataframe

    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:

        dataframe.loc[
            (
                (qtpylib.crossed_above(dataframe['rsi'], 30)) &  # Signal: RSI crosses above 30
                (dataframe['rsi_1d'] < 30) &                     # Ensure daily RSI is < 30
                (dataframe['volume'] > 0)                        # Ensure this candle had volume (important for backtesting)
            ),
            ['enter_long', 'enter_tag']] = (1, 'rsi_cross')

```

***

## Additional data (Wallets)

The strategy provides access to the `wallets` object. This contains the current balances of your wallets/accounts on the exchange.

!!! Note "Backtesting / Hyperopt"
    Wallets behaves differently depending on the function from which it is called.
    Within `populate_*()` methods, it'll return the full wallet as configured.
    Within [callbacks](strategy-callbacks.md), you'll get the wallet state corresponding to the actual simulated wallet at that point in the simulation process.

Always check if `wallets` is available to avoid failures during backtesting.

``` python
if self.wallets:
    free_eth = self.wallets.get_free('ETH')
    used_eth = self.wallets.get_used('ETH')
    total_eth = self.wallets.get_total('ETH')
```

### Possible options for Wallets

- `get_free(asset)` - currently available balance to trade
- `get_used(asset)` - currently tied up balance (open orders)
- `get_total(asset)` - total available balance - sum of the 2 above

***

## Additional data (Trades)

A history of trades can be retrieved in the strategy by querying the database.

At the top of the file, import the required object:

```python
from freqtrade.persistence import Trade
```

The following example queries trades from today for the current pair (`metadata['pair']`). Other filters can easily be added.

``` python
trades = Trade.get_trades_proxy(pair=metadata['pair'],
                                open_date=datetime.now(timezone.utc) - timedelta(days=1),
                                is_open=False,
            ]).order_by(Trade.close_date).all()
# Summarize profit for this pair.
curdayprofit = sum(trade.close_profit for trade in trades)
```

For a full list of available methods, please consult the [Trade object](trade-object.md) documentation.

!!! Warning
    Trade history is not available in `populate_*` methods during backtesting or hyperopt, and will result in empty results.

## Prevent trades from happening for a specific pair

Freqtrade locks pairs automatically for the current candle (until that candle is over) when a pair exits, preventing an immediate re-entry of that pair.

This is to prevent "waterfalls" of many and frequent trades within a single candle.

Locked pairs will show the message `Pair <pair> is currently locked.`.

### Locking pairs from within the strategy

Sometimes it may be desired to lock a pair after certain events happen (e.g. multiple losing trades in a row).

Freqtrade has an easy method to do this from within the strategy, by calling `self.lock_pair(pair, until, [reason])`.
`until` must be a datetime object in the future, after which trading will be re-enabled for that pair, while `reason` is an optional string detailing why the pair was locked.

Locks can also be lifted manually, by calling `self.unlock_pair(pair)` or `self.unlock_reason(<reason>)`, providing the reason the pair was unlocked.
`self.unlock_reason(<reason>)` will unlock all pairs currently locked with the provided reason.

To verify if a pair is currently locked, use `self.is_pair_locked(pair)`.

!!! Note
    Locked pairs will always be rounded up to the next candle. So assuming a `5m` timeframe, a lock with `until` set to 10:18 will lock the pair until the candle from 10:15-10:20 will be finished.

!!! Warning
    Manually locking pairs is not available during backtesting. Only locks via Protections are allowed.

#### Pair locking example

``` python
from freqtrade.persistence import Trade
from datetime import timedelta, datetime, timezone
# Put the above lines a the top of the strategy file, next to all the other imports
# --------

# Within populate indicators (or populate_entry_trend):
if self.config['runmode'].value in ('live', 'dry_run'):
    # fetch closed trades for the last 2 days
    trades = Trade.get_trades_proxy(
        pair=metadata['pair'], is_open=False, 
        open_date=datetime.now(timezone.utc) - timedelta(days=2))
    # Analyze the conditions you'd like to lock the pair .... will probably be different for every strategy
    sumprofit = sum(trade.close_profit for trade in trades)
    if sumprofit < 0:
        # Lock pair for 12 hours
        self.lock_pair(metadata['pair'], until=datetime.now(timezone.utc) + timedelta(hours=12))
```

## Print the main dataframe

To inspect the current main dataframe, you can issue a print-statement in either `populate_entry_trend()` or `populate_exit_trend()`.
You may also want to print the pair so it's clear what data is currently shown.

``` python
def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    dataframe.loc[
        (
            #>> whatever condition<<<
        ),
        ['enter_long', 'enter_tag']] = (1, 'somestring')

    # Print the Analyzed pair
    print(f"result for {metadata['pair']}")

    # Inspect the last 5 rows
    print(dataframe.tail())

    return dataframe
```

Printing more than a few rows is also possible by using `print(dataframe)` instead of `print(dataframe.tail())`. However this is not recommended, as can results in a lot of output (~500 lines per pair every 5 seconds).

## Common mistakes when developing strategies

### Looking into the future while backtesting

Backtesting analyzes the whole dataframe timerange at once for performance reasons. Because of this, strategy authors need to make sure that strategies do not lookahead into the future, i.e. using data that would not be available in dry or live mode.

This is a common pain-point, which can cause huge differences between backtesting and dry/live run methods. Strategies that look into the future will perform well during backtesting, often with incredible profits or winrates, but will fail or perform badly in real conditions.

The following list contains some common patterns which should be avoided to prevent frustration:

- don't use `shift(-1)` or other negative values. This uses data from the future in backtesting, which is not available in dry or live modes.
- don't use `.iloc[-1]` or any other absolute position in the dataframe within `populate_` functions, as this will be different between dry-run and backtesting. Absolute `iloc` indexing is safe to use in callbacks however - see [Strategy Callbacks](strategy-callbacks.md).
- don't use functions that use all dataframe or column values, e.g. `dataframe['mean_volume'] = dataframe['volume'].mean()`. As backtesting uses the full dataframe, at any point in the dataframe, the `'mean_volume'` series would include data from the future. Use rolling() calculations instead, e.g. `dataframe['volume'].rolling(<window>).mean()`.
- don't use `.resample('1h')`. This uses the left border of the period interval, so moves data from an hour boundary to the start of the hour. Use `.resample('1h', label='right')` instead.
- don't use `.merge()` to combine longer timeframes onto shorter ones. Instead, use the [informative pair](#informative-pairs) helpers. (A plain merge can implicitly cause a lookahead bias as date refers to open date, not close date).

!!! Tip "Identifying problems"
    You should always use the two helper commands [lookahead-analysis](lookahead-analysis.md) and [recursive-analysis](recursive-analysis.md), which can each help you figure out problems with your strategy in different ways.
    Please treat them as what they are - helpers to identify most common problems. A negative result of each does not guarantee that there are none of the above errors included.

### Colliding signals

When conflicting signals collide (e.g. both `'enter_long'` and `'exit_long'` are set to `1`), freqtrade will do nothing and ignore the entry signal. This will avoid trades that enter, and exit immediately. Obviously, this can potentially lead to missed entries.

The following rules apply, and entry signals will be ignored if more than one of the 3 signals is set:

- `enter_long` -> `exit_long`, `enter_short`
- `enter_short` -> `exit_short`, `enter_long`

## Further strategy ideas

To get additional ideas for strategies, head over to the [strategy repository](https://github.com/freqtrade/freqtrade-strategies). Feel free to use them as examples, but results will depend on the current market situation, pairs used, etc. Therefore, these strategies should be considered only for learning purposes, not real world trading. Please backtest the strategy for your exchange/desired pairs first, then dry run to evaluate carefully, and use at your own risk.

Feel free to use any of them as inspiration for your own strategies. We're happy to accept Pull Requests containing new strategies to the repository.

## Next steps

Now you have a perfect strategy you probably want to backtest it.
Your next step is to learn [how to use backtesting](backtesting.md).

---

## File: strategy_migration.md

# Strategy Migration between V2 and V3

To support new markets and trade-types (namely short trades / trades with leverage), some things had to change in the interface.
If you intend on using markets other than spot markets, please migrate your strategy to the new format.

We have put a great effort into keeping compatibility with existing strategies, so if you just want to continue using freqtrade in __spot markets__, there should be no changes necessary for now.

You can use the quick summary as checklist. Please refer to the detailed sections below for full migration details.

## Quick summary / migration checklist

Note : `forcesell`, `forcebuy`, `emergencysell` are changed to `force_exit`, `force_enter`, `emergency_exit` respectively.

* Strategy methods:
  * [`populate_buy_trend()` -> `populate_entry_trend()`](#populate_buy_trend)
  * [`populate_sell_trend()` -> `populate_exit_trend()`](#populate_sell_trend)
  * [`custom_sell()` -> `custom_exit()`](#custom_sell)
  * [`check_buy_timeout()` -> `check_entry_timeout()`](#custom_entry_timeout)
  * [`check_sell_timeout()` -> `check_exit_timeout()`](#custom_entry_timeout)
  * New `side` argument to callbacks without trade object
    * [`custom_stake_amount`](#custom_stake_amount)
    * [`confirm_trade_entry`](#confirm_trade_entry)
    * [`custom_entry_price`](#custom_entry_price)
  * [Changed argument name in `confirm_trade_exit`](#confirm_trade_exit)
* Dataframe columns:
  * [`buy` -> `enter_long`](#populate_buy_trend)
  * [`sell` -> `exit_long`](#populate_sell_trend)
  * [`buy_tag` -> `enter_tag` (used for both long and short trades)](#populate_buy_trend)
  * [New column `enter_short` and corresponding new column `exit_short`](#populate_sell_trend)
* trade-object now has the following new properties:
  * `is_short`
  * `entry_side`
  * `exit_side`
  * `trade_direction`
  * renamed: `sell_reason` -> `exit_reason`
* [Renamed `trade.nr_of_successful_buys` to `trade.nr_of_successful_entries` (mostly relevant for `adjust_trade_position()`)](#adjust-trade-position-changes)
* Introduced new [`leverage` callback](strategy-callbacks.md#leverage-callback).
* Informative pairs can now pass a 3rd element in the Tuple, defining the candle type.
* `@informative` decorator now takes an optional `candle_type` argument.
* [helper methods](#helper-methods) `stoploss_from_open` and `stoploss_from_absolute` now take `is_short` as additional argument.
* `INTERFACE_VERSION` should be set to 3.
* [Strategy/Configuration settings](#strategyconfiguration-settings).
  * `order_time_in_force` buy -> entry, sell -> exit.
  * `order_types` buy -> entry, sell -> exit.
  * `unfilledtimeout` buy -> entry, sell -> exit.
  * `ignore_buying_expired_candle_after` -> moved to root level instead of "ask_strategy/exit_pricing"
* Terminology changes
  * Sell reasons changed to reflect the new naming of "exit" instead of sells. Be careful in your strategy if you're using `exit_reason` checks and eventually update your strategy.
    * `sell_signal` -> `exit_signal`
    * `custom_sell` -> `custom_exit`
    * `force_sell` -> `force_exit`
    * `emergency_sell` -> `emergency_exit`
  * Order pricing
    * `bid_strategy` -> `entry_pricing`
    * `ask_strategy` -> `exit_pricing`
    * `ask_last_balance` -> `price_last_balance`
    * `bid_last_balance` -> `price_last_balance`
  * Webhook terminology changed from "sell" to "exit", and from "buy" to entry
    * `webhookbuy` -> `entry`
    * `webhookbuyfill` -> `entry_fill`
    * `webhookbuycancel` -> `entry_cancel`
    * `webhooksell` -> `exit`
    * `webhooksellfill` -> `exit_fill`
    * `webhooksellcancel` -> `exit_cancel`
  * Telegram notification settings
    * `buy` -> `entry`
    * `buy_fill` -> `entry_fill`
    * `buy_cancel` -> `entry_cancel`
    * `sell` -> `exit`
    * `sell_fill` -> `exit_fill`
    * `sell_cancel` -> `exit_cancel`
  * Strategy/config settings:
    * `use_sell_signal` -> `use_exit_signal`
    * `sell_profit_only` -> `exit_profit_only`
    * `sell_profit_offset` -> `exit_profit_offset`
    * `ignore_roi_if_buy_signal` -> `ignore_roi_if_entry_signal`
    * `forcebuy_enable` -> `force_entry_enable`

## Extensive explanation

### `populate_buy_trend`

In `populate_buy_trend()` - you will want to change the columns you assign from `'buy`' to `'enter_long'`, as well as the method name from `populate_buy_trend` to `populate_entry_trend`.

```python hl_lines="1 9"
def populate_buy_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    dataframe.loc[
        (
            (qtpylib.crossed_above(dataframe['rsi'], 30)) &  # Signal: RSI crosses above 30
            (dataframe['tema'] <= dataframe['bb_middleband']) &  # Guard
            (dataframe['tema'] > dataframe['tema'].shift(1)) &  # Guard
            (dataframe['volume'] > 0)  # Make sure Volume is not 0
        ),
        ['buy', 'buy_tag']] = (1, 'rsi_cross')

    return dataframe
```

After:

```python hl_lines="1 9"
def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    dataframe.loc[
        (
            (qtpylib.crossed_above(dataframe['rsi'], 30)) &  # Signal: RSI crosses above 30
            (dataframe['tema'] <= dataframe['bb_middleband']) &  # Guard
            (dataframe['tema'] > dataframe['tema'].shift(1)) &  # Guard
            (dataframe['volume'] > 0)  # Make sure Volume is not 0
        ),
        ['enter_long', 'enter_tag']] = (1, 'rsi_cross')

    return dataframe
```

Please refer to the [Strategy documentation](strategy-customization.md#entry-signal-rules) on how to enter and exit short trades.

### `populate_sell_trend`

Similar to `populate_buy_trend`, `populate_sell_trend()` will be renamed to `populate_exit_trend()`.
We'll also change the column from `'sell'` to `'exit_long'`.

``` python hl_lines="1 9"
def populate_sell_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    dataframe.loc[
        (
            (qtpylib.crossed_above(dataframe['rsi'], 70)) &  # Signal: RSI crosses above 70
            (dataframe['tema'] > dataframe['bb_middleband']) &  # Guard
            (dataframe['tema'] < dataframe['tema'].shift(1)) &  # Guard
            (dataframe['volume'] > 0)  # Make sure Volume is not 0
        ),
        ['sell', 'exit_tag']] = (1, 'some_exit_tag')
    return dataframe
```

After

``` python hl_lines="1 9"
def populate_exit_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    dataframe.loc[
        (
            (qtpylib.crossed_above(dataframe['rsi'], 70)) &  # Signal: RSI crosses above 70
            (dataframe['tema'] > dataframe['bb_middleband']) &  # Guard
            (dataframe['tema'] < dataframe['tema'].shift(1)) &  # Guard
            (dataframe['volume'] > 0)  # Make sure Volume is not 0
        ),
        ['exit_long', 'exit_tag']] = (1, 'some_exit_tag')
    return dataframe
```

Please refer to the [Strategy documentation](strategy-customization.md#exit-signal-rules) on how to enter and exit short trades.

### `custom_sell`

`custom_sell` has been renamed to `custom_exit`.
It's now also being called for every iteration, independent of current profit and `exit_profit_only` settings.

``` python hl_lines="2"
class AwesomeStrategy(IStrategy):
    def custom_sell(self, pair: str, trade: 'Trade', current_time: 'datetime', current_rate: float,
                    current_profit: float, **kwargs):
        dataframe, _ = self.dp.get_analyzed_dataframe(pair, self.timeframe)
        last_candle = dataframe.iloc[-1].squeeze()
        # ...
```

``` python hl_lines="2"
class AwesomeStrategy(IStrategy):
    def custom_exit(self, pair: str, trade: 'Trade', current_time: 'datetime', current_rate: float,
                    current_profit: float, **kwargs):
        dataframe, _ = self.dp.get_analyzed_dataframe(pair, self.timeframe)
        last_candle = dataframe.iloc[-1].squeeze()
        # ...
```

### `custom_entry_timeout`

`check_buy_timeout()` has been renamed to `check_entry_timeout()`, and `check_sell_timeout()` has been renamed to `check_exit_timeout()`.

``` python hl_lines="2 6"
class AwesomeStrategy(IStrategy):
    def check_buy_timeout(self, pair: str, trade: 'Trade', order: dict, 
                            current_time: datetime, **kwargs) -> bool:
        return False

    def check_sell_timeout(self, pair: str, trade: 'Trade', order: dict, 
                            current_time: datetime, **kwargs) -> bool:
        return False 
```

``` python hl_lines="2 6"
class AwesomeStrategy(IStrategy):
    def check_entry_timeout(self, pair: str, trade: 'Trade', order: 'Order', 
                            current_time: datetime, **kwargs) -> bool:
        return False

    def check_exit_timeout(self, pair: str, trade: 'Trade', order: 'Order', 
                            current_time: datetime, **kwargs) -> bool:
        return False 
```

### `custom_stake_amount`

New string argument `side` - which can be either `"long"` or `"short"`.

``` python hl_lines="4"
class AwesomeStrategy(IStrategy):
    def custom_stake_amount(self, pair: str, current_time: datetime, current_rate: float,
                            proposed_stake: float, min_stake: Optional[float], max_stake: float,
                            entry_tag: Optional[str], **kwargs) -> float:
        # ... 
        return proposed_stake
```

``` python hl_lines="4"
class AwesomeStrategy(IStrategy):
    def custom_stake_amount(self, pair: str, current_time: datetime, current_rate: float,
                            proposed_stake: float, min_stake: float | None, max_stake: float,
                            entry_tag: str | None, side: str, **kwargs) -> float:
        # ... 
        return proposed_stake
```

### `confirm_trade_entry`

New string argument `side` - which can be either `"long"` or `"short"`.

``` python hl_lines="4"
class AwesomeStrategy(IStrategy):
    def confirm_trade_entry(self, pair: str, order_type: str, amount: float, rate: float,
                            time_in_force: str, current_time: datetime, entry_tag: Optional[str], 
                            **kwargs) -> bool:
      return True
```

After: 

``` python hl_lines="4"
class AwesomeStrategy(IStrategy):
    def confirm_trade_entry(self, pair: str, order_type: str, amount: float, rate: float,
                            time_in_force: str, current_time: datetime, entry_tag: str | None, 
                            side: str, **kwargs) -> bool:
      return True
```

### `confirm_trade_exit`

Changed argument `sell_reason` to `exit_reason`.
For compatibility, `sell_reason` will still be provided for a limited time.

``` python hl_lines="3"
class AwesomeStrategy(IStrategy):
    def confirm_trade_exit(self, pair: str, trade: Trade, order_type: str, amount: float,
                           rate: float, time_in_force: str, sell_reason: str,
                           current_time: datetime, **kwargs) -> bool:
    return True
```

After:

``` python hl_lines="3"
class AwesomeStrategy(IStrategy):
    def confirm_trade_exit(self, pair: str, trade: Trade, order_type: str, amount: float,
                           rate: float, time_in_force: str, exit_reason: str,
                           current_time: datetime, **kwargs) -> bool:
    return True
```

### `custom_entry_price`

New string argument `side` - which can be either `"long"` or `"short"`.

``` python hl_lines="3"
class AwesomeStrategy(IStrategy):
    def custom_entry_price(self, pair: str, current_time: datetime, proposed_rate: float,
                           entry_tag: Optional[str], **kwargs) -> float:
      return proposed_rate
```

After:

``` python hl_lines="3"
class AwesomeStrategy(IStrategy):
    def custom_entry_price(self, pair: str, trade: Trade | None, current_time: datetime, proposed_rate: float,
                           entry_tag: str | None, side: str, **kwargs) -> float:
      return proposed_rate
```

### Adjust trade position changes

While adjust-trade-position itself did not change, you should no longer use `trade.nr_of_successful_buys` - and instead use `trade.nr_of_successful_entries`, which will also include short entries.

### Helper methods

Added argument "is_short" to `stoploss_from_open` and `stoploss_from_absolute`.
This should be given the value of `trade.is_short`.

``` python hl_lines="5 7"
    def custom_stoploss(self, pair: str, trade: 'Trade', current_time: datetime,
                        current_rate: float, current_profit: float, **kwargs) -> float:
        # once the profit has risen above 10%, keep the stoploss at 7% above the open price
        if current_profit > 0.10:
            return stoploss_from_open(0.07, current_profit)

        return stoploss_from_absolute(current_rate - (candle['atr'] * 2), current_rate)

        return 1

```

After:

``` python hl_lines="5 7"
    def custom_stoploss(self, pair: str, trade: 'Trade', current_time: datetime,
                        current_rate: float, current_profit: float, after_fill: bool, 
                        **kwargs) -> float | None:
        # once the profit has risen above 10%, keep the stoploss at 7% above the open price
        if current_profit > 0.10:
            return stoploss_from_open(0.07, current_profit, is_short=trade.is_short)

        return stoploss_from_absolute(current_rate - (candle['atr'] * 2), current_rate, is_short=trade.is_short, leverage=trade.leverage)


```

### Strategy/Configuration settings

#### `order_time_in_force`

`order_time_in_force` attributes changed from `"buy"` to `"entry"` and `"sell"` to `"exit"`.

``` python
    order_time_in_force: dict = {
        "buy": "gtc",
        "sell": "gtc",
    }
```

After:

``` python hl_lines="2 3"
    order_time_in_force: dict = {
        "entry": "GTC",
        "exit": "GTC",
    }
```

#### `order_types`

`order_types` have changed all wordings from `buy` to `entry` - and `sell` to `exit`.
And two words are joined with `_`. 

``` python hl_lines="2-6"
    order_types = {
        "buy": "limit",
        "sell": "limit",
        "emergencysell": "market",
        "forcesell": "market",
        "forcebuy": "market",
        "stoploss": "market",
        "stoploss_on_exchange": false,
        "stoploss_on_exchange_interval": 60
    }
```

After:

``` python hl_lines="2-6"
    order_types = {
        "entry": "limit",
        "exit": "limit",
        "emergency_exit": "market",
        "force_exit": "market",
        "force_entry": "market",
        "stoploss": "market",
        "stoploss_on_exchange": false,
        "stoploss_on_exchange_interval": 60
    }
```

#### Strategy level settings

* `use_sell_signal` -> `use_exit_signal`
* `sell_profit_only` -> `exit_profit_only`
* `sell_profit_offset` -> `exit_profit_offset`
* `ignore_roi_if_buy_signal` -> `ignore_roi_if_entry_signal`

``` python hl_lines="2-5"
    # These values can be overridden in the config.
    use_sell_signal = True
    sell_profit_only = True
    sell_profit_offset: 0.01
    ignore_roi_if_buy_signal = False
```

After:

``` python hl_lines="2-5"
    # These values can be overridden in the config.
    use_exit_signal = True
    exit_profit_only = True
    exit_profit_offset: 0.01
    ignore_roi_if_entry_signal = False
```

#### `unfilledtimeout`

`unfilledtimeout` have changed all wordings from `buy` to `entry` - and `sell` to `exit`.

``` python hl_lines="2-3"
unfilledtimeout = {
        "buy": 10,
        "sell": 10,
        "exit_timeout_count": 0,
        "unit": "minutes"
    }
```

After:

``` python hl_lines="2-3"
unfilledtimeout = {
        "entry": 10,
        "exit": 10,
        "exit_timeout_count": 0,
        "unit": "minutes"
    }
```

#### `order pricing`

Order pricing changed in 2 ways. `bid_strategy` was renamed to `entry_pricing` and `ask_strategy` was renamed to `exit_pricing`.
The attributes `ask_last_balance` -> `price_last_balance` and `bid_last_balance` -> `price_last_balance` were renamed as well.
Also, price-side can now be defined as `ask`, `bid`, `same` or `other`.
Please refer to the [pricing documentation](configuration.md#prices-used-for-orders) for more information.

``` json hl_lines="2-3 6 12-13 16"
{
    "bid_strategy": {
        "price_side": "bid",
        "use_order_book": true,
        "order_book_top": 1,
        "ask_last_balance": 0.0,
        "check_depth_of_market": {
            "enabled": false,
            "bids_to_ask_delta": 1
        }
    },
    "ask_strategy":{
        "price_side": "ask",
        "use_order_book": true,
        "order_book_top": 1,
        "bid_last_balance": 0.0
        "ignore_buying_expired_candle_after": 120
    }
}
```

after:

``` json  hl_lines="2-3 6 12-13 16"
{
    "entry_pricing": {
        "price_side": "same",
        "use_order_book": true,
        "order_book_top": 1,
        "price_last_balance": 0.0,
        "check_depth_of_market": {
            "enabled": false,
            "bids_to_ask_delta": 1
        }
    },
    "exit_pricing":{
        "price_side": "same",
        "use_order_book": true,
        "order_book_top": 1,
        "price_last_balance": 0.0
    },
    "ignore_buying_expired_candle_after": 120
}
```

## FreqAI strategy

The `populate_any_indicators()` method has been split into `feature_engineering_expand_all()`, `feature_engineering_expand_basic()`, `feature_engineering_standard()` and`set_freqai_targets()`.

For each new function, the pair (and timeframe where necessary) will be automatically added to the column.
As such, the definition of features becomes much simpler with the new logic.

For a full explanation of each method, please go to the corresponding [freqAI documentation page](freqai-feature-engineering.md#defining-the-features)

``` python linenums="1" hl_lines="12-37 39-42 63-65 67-75"

def populate_any_indicators(
        self, pair, df, tf, informative=None, set_generalized_indicators=False
    ):

        if informative is None:
            informative = self.dp.get_pair_dataframe(pair, tf)

        # first loop is automatically duplicating indicators for time periods
        for t in self.freqai_info["feature_parameters"]["indicator_periods_candles"]:

            t = int(t)
            informative[f"%-{pair}rsi-period_{t}"] = ta.RSI(informative, timeperiod=t)
            informative[f"%-{pair}mfi-period_{t}"] = ta.MFI(informative, timeperiod=t)
            informative[f"%-{pair}adx-period_{t}"] = ta.ADX(informative, timeperiod=t)
            informative[f"%-{pair}sma-period_{t}"] = ta.SMA(informative, timeperiod=t)
            informative[f"%-{pair}ema-period_{t}"] = ta.EMA(informative, timeperiod=t)

            bollinger = qtpylib.bollinger_bands(
                qtpylib.typical_price(informative), window=t, stds=2.2
            )
            informative[f"{pair}bb_lowerband-period_{t}"] = bollinger["lower"]
            informative[f"{pair}bb_middleband-period_{t}"] = bollinger["mid"]
            informative[f"{pair}bb_upperband-period_{t}"] = bollinger["upper"]

            informative[f"%-{pair}bb_width-period_{t}"] = (
                informative[f"{pair}bb_upperband-period_{t}"]
                - informative[f"{pair}bb_lowerband-period_{t}"]
            ) / informative[f"{pair}bb_middleband-period_{t}"]
            informative[f"%-{pair}close-bb_lower-period_{t}"] = (
                informative["close"] / informative[f"{pair}bb_lowerband-period_{t}"]
            )

            informative[f"%-{pair}roc-period_{t}"] = ta.ROC(informative, timeperiod=t)

            informative[f"%-{pair}relative_volume-period_{t}"] = (
                informative["volume"] / informative["volume"].rolling(t).mean()
            ) # (1)

        informative[f"%-{pair}pct-change"] = informative["close"].pct_change()
        informative[f"%-{pair}raw_volume"] = informative["volume"]
        informative[f"%-{pair}raw_price"] = informative["close"]
        # (2)

        indicators = [col for col in informative if col.startswith("%")]
        # This loop duplicates and shifts all indicators to add a sense of recency to data
        for n in range(self.freqai_info["feature_parameters"]["include_shifted_candles"] + 1):
            if n == 0:
                continue
            informative_shift = informative[indicators].shift(n)
            informative_shift = informative_shift.add_suffix("_shift-" + str(n))
            informative = pd.concat((informative, informative_shift), axis=1)

        df = merge_informative_pair(df, informative, self.config["timeframe"], tf, ffill=True)
        skip_columns = [
            (s + "_" + tf) for s in ["date", "open", "high", "low", "close", "volume"]
        ]
        df = df.drop(columns=skip_columns)

        # Add generalized indicators here (because in live, it will call this
        # function to populate indicators during training). Notice how we ensure not to
        # add them multiple times
        if set_generalized_indicators:
            df["%-day_of_week"] = (df["date"].dt.dayofweek + 1) / 7
            df["%-hour_of_day"] = (df["date"].dt.hour + 1) / 25
            # (3)

            # user adds targets here by prepending them with &- (see convention below)
            df["&-s_close"] = (
                df["close"]
                .shift(-self.freqai_info["feature_parameters"]["label_period_candles"])
                .rolling(self.freqai_info["feature_parameters"]["label_period_candles"])
                .mean()
                / df["close"]
                - 1
            )  # (4)

        return df
```

1. Features - Move to `feature_engineering_expand_all`
2. Basic features, not expanded across `indicator_periods_candles` - move to`feature_engineering_expand_basic()`.
3. Standard features which should not be expanded - move to `feature_engineering_standard()`.
4. Targets - Move this part to `set_freqai_targets()`.

### freqai - feature engineering expand all

Features will now expand automatically. As such, the expansion loops, as well as the `{pair}` / `{timeframe}` parts will need to be removed.

``` python linenums="1"
    def feature_engineering_expand_all(self, dataframe, period, **kwargs) -> DataFrame::
        """
        *Only functional with FreqAI enabled strategies*
        This function will automatically expand the defined features on the config defined
        `indicator_periods_candles`, `include_timeframes`, `include_shifted_candles`, and
        `include_corr_pairs`. In other words, a single feature defined in this function
        will automatically expand to a total of
        `indicator_periods_candles` * `include_timeframes` * `include_shifted_candles` *
        `include_corr_pairs` numbers of features added to the model.

        All features must be prepended with `%` to be recognized by FreqAI internals.

        More details on how these config defined parameters accelerate feature engineering
        in the documentation at:

        https://www.freqtrade.io/en/latest/freqai-parameter-table/#feature-parameters

        https://www.freqtrade.io/en/latest/freqai-feature-engineering/#defining-the-features

        :param df: strategy dataframe which will receive the features
        :param period: period of the indicator - usage example:
        dataframe["%-ema-period"] = ta.EMA(dataframe, timeperiod=period)
        """

        dataframe["%-rsi-period"] = ta.RSI(dataframe, timeperiod=period)
        dataframe["%-mfi-period"] = ta.MFI(dataframe, timeperiod=period)
        dataframe["%-adx-period"] = ta.ADX(dataframe, timeperiod=period)
        dataframe["%-sma-period"] = ta.SMA(dataframe, timeperiod=period)
        dataframe["%-ema-period"] = ta.EMA(dataframe, timeperiod=period)

        bollinger = qtpylib.bollinger_bands(
            qtpylib.typical_price(dataframe), window=period, stds=2.2
        )
        dataframe["bb_lowerband-period"] = bollinger["lower"]
        dataframe["bb_middleband-period"] = bollinger["mid"]
        dataframe["bb_upperband-period"] = bollinger["upper"]

        dataframe["%-bb_width-period"] = (
            dataframe["bb_upperband-period"]
            - dataframe["bb_lowerband-period"]
        ) / dataframe["bb_middleband-period"]
        dataframe["%-close-bb_lower-period"] = (
            dataframe["close"] / dataframe["bb_lowerband-period"]
        )

        dataframe["%-roc-period"] = ta.ROC(dataframe, timeperiod=period)

        dataframe["%-relative_volume-period"] = (
            dataframe["volume"] / dataframe["volume"].rolling(period).mean()
        )

        return dataframe

```

### Freqai - feature engineering basic

Basic features. Make sure to remove the `{pair}` part from your features.

``` python linenums="1"
    def feature_engineering_expand_basic(self, dataframe: DataFrame, **kwargs) -> DataFrame::
        """
        *Only functional with FreqAI enabled strategies*
        This function will automatically expand the defined features on the config defined
        `include_timeframes`, `include_shifted_candles`, and `include_corr_pairs`.
        In other words, a single feature defined in this function
        will automatically expand to a total of
        `include_timeframes` * `include_shifted_candles` * `include_corr_pairs`
        numbers of features added to the model.

        Features defined here will *not* be automatically duplicated on user defined
        `indicator_periods_candles`

        All features must be prepended with `%` to be recognized by FreqAI internals.

        More details on how these config defined parameters accelerate feature engineering
        in the documentation at:

        https://www.freqtrade.io/en/latest/freqai-parameter-table/#feature-parameters

        https://www.freqtrade.io/en/latest/freqai-feature-engineering/#defining-the-features

        :param df: strategy dataframe which will receive the features
        dataframe["%-pct-change"] = dataframe["close"].pct_change()
        dataframe["%-ema-200"] = ta.EMA(dataframe, timeperiod=200)
        """
        dataframe["%-pct-change"] = dataframe["close"].pct_change()
        dataframe["%-raw_volume"] = dataframe["volume"]
        dataframe["%-raw_price"] = dataframe["close"]
        return dataframe
```

### FreqAI - feature engineering standard

``` python linenums="1"
    def feature_engineering_standard(self, dataframe: DataFrame, **kwargs) -> DataFrame:
        """
        *Only functional with FreqAI enabled strategies*
        This optional function will be called once with the dataframe of the base timeframe.
        This is the final function to be called, which means that the dataframe entering this
        function will contain all the features and columns created by all other
        freqai_feature_engineering_* functions.

        This function is a good place to do custom exotic feature extractions (e.g. tsfresh).
        This function is a good place for any feature that should not be auto-expanded upon
        (e.g. day of the week).

        All features must be prepended with `%` to be recognized by FreqAI internals.

        More details about feature engineering available:

        https://www.freqtrade.io/en/latest/freqai-feature-engineering

        :param df: strategy dataframe which will receive the features
        usage example: dataframe["%-day_of_week"] = (dataframe["date"].dt.dayofweek + 1) / 7
        """
        dataframe["%-day_of_week"] = dataframe["date"].dt.dayofweek
        dataframe["%-hour_of_day"] = dataframe["date"].dt.hour
        return dataframe
```

### FreqAI - set Targets

Targets now get their own, dedicated method.

``` python linenums="1"
    def set_freqai_targets(self, dataframe: DataFrame, **kwargs) -> DataFrame:
        """
        *Only functional with FreqAI enabled strategies*
        Required function to set the targets for the model.
        All targets must be prepended with `&` to be recognized by the FreqAI internals.

        More details about feature engineering available:

        https://www.freqtrade.io/en/latest/freqai-feature-engineering

        :param df: strategy dataframe which will receive the targets
        usage example: dataframe["&-target"] = dataframe["close"].shift(-1) / dataframe["close"]
        """
        dataframe["&-s_close"] = (
            dataframe["close"]
            .shift(-self.freqai_info["feature_parameters"]["label_period_candles"])
            .rolling(self.freqai_info["feature_parameters"]["label_period_candles"])
            .mean()
            / dataframe["close"]
            - 1
            )

        return dataframe
```


### FreqAI - New data Pipeline

If you have created your own custom `IFreqaiModel` with a custom `train()`/`predict()` function, *and* you still rely on `data_cleaning_train/predict()`, then you will need to migrate to the new pipeline. If your model does *not* rely on `data_cleaning_train/predict()`, then you do not need to worry about this migration. That means that this migration guide is relevant for a very small percentage of power-users. If you stumbled upon this guide by mistake, feel free to inquire in depth about your problem in the Freqtrade discord server.

The conversion involves first removing `data_cleaning_train/predict()` and replacing them with a `define_data_pipeline()` and `define_label_pipeline()` function to your `IFreqaiModel` class:

```python  linenums="1" hl_lines="11-14 47-49 55-57"
class MyCoolFreqaiModel(BaseRegressionModel):
    """
    Some cool custom IFreqaiModel you made before Freqtrade version 2023.6
    """
    def train(
        self, unfiltered_df: DataFrame, pair: str, dk: FreqaiDataKitchen, **kwargs
    ) -> Any:

        # ... your custom stuff

        # Remove these lines
        # data_dictionary = dk.make_train_test_datasets(features_filtered, labels_filtered)
        # self.data_cleaning_train(dk)
        # data_dictionary = dk.normalize_data(data_dictionary)
        # (1)

        # Add these lines. Now we control the pipeline fit/transform ourselves
        dd = dk.make_train_test_datasets(features_filtered, labels_filtered)
        dk.feature_pipeline = self.define_data_pipeline(threads=dk.thread_count)
        dk.label_pipeline = self.define_label_pipeline(threads=dk.thread_count)

        (dd["train_features"],
         dd["train_labels"],
         dd["train_weights"]) = dk.feature_pipeline.fit_transform(dd["train_features"],
                                                                  dd["train_labels"],
                                                                  dd["train_weights"])

        (dd["test_features"],
         dd["test_labels"],
         dd["test_weights"]) = dk.feature_pipeline.transform(dd["test_features"],
                                                             dd["test_labels"],
                                                             dd["test_weights"])

        dd["train_labels"], _, _ = dk.label_pipeline.fit_transform(dd["train_labels"])
        dd["test_labels"], _, _ = dk.label_pipeline.transform(dd["test_labels"])

        # ... your custom code

        return model

    def predict(
        self, unfiltered_df: DataFrame, dk: FreqaiDataKitchen, **kwargs
    ) -> tuple[DataFrame, npt.NDArray[np.int_]]:

        # ... your custom stuff

        # Remove these lines:
        # self.data_cleaning_predict(dk)
        # (2)

        # Add these lines:
        dk.data_dictionary["prediction_features"], outliers, _ = dk.feature_pipeline.transform(
            dk.data_dictionary["prediction_features"], outlier_check=True)

        # Remove this line
        # pred_df = dk.denormalize_labels_from_metadata(pred_df)
        # (3)

        # Replace with these lines
        pred_df, _, _ = dk.label_pipeline.inverse_transform(pred_df)
        if self.freqai_info.get("DI_threshold", 0) > 0:
            dk.DI_values = dk.feature_pipeline["di"].di_values
        else:
            dk.DI_values = np.zeros(outliers.shape[0])
        dk.do_predict = outliers

        # ... your custom code
        return (pred_df, dk.do_predict)
```


1. Data normalization and cleaning is now homogenized with the new pipeline definition. This is created in the new `define_data_pipeline()` and `define_label_pipeline()` functions. The `data_cleaning_train()` and `data_cleaning_predict()` functions are no longer used. You can override `define_data_pipeline()` to create your own custom pipeline if you wish.
2. Data normalization and cleaning is now homogenized with the new pipeline definition. This is created in the new `define_data_pipeline()` and `define_label_pipeline()` functions. The `data_cleaning_train()` and `data_cleaning_predict()` functions are no longer used. You can override `define_data_pipeline()` to create your own custom pipeline if you wish.
3. Data denormalization is done with the new pipeline. Replace this with the lines below.

---

## File: telegram-usage.md

# Telegram usage

## Setup your Telegram bot

Below we explain how to create your Telegram Bot, and how to get your
Telegram user id.

### 1. Create your Telegram bot

Start a chat with the [Telegram BotFather](https://telegram.me/BotFather)

Send the message `/newbot`.

*BotFather response:*

> Alright, a new bot. How are we going to call it? Please choose a name for your bot.

Choose the public name of your bot (e.x. `Freqtrade bot`)

*BotFather response:*

> Good. Now let's choose a username for your bot. It must end in `bot`. Like this, for example: TetrisBot or tetris_bot.

Choose the name id of your bot and send it to the BotFather (e.g. "`My_own_freqtrade_bot`")

*BotFather response:*

> Done! Congratulations on your new bot. You will find it at `t.me/yourbots_name_bot`. You can now add a description, about section and profile picture for your bot, see /help for a list of commands. By the way, when you've finished creating your cool bot, ping our Bot Support if you want a better username for it. Just make sure the bot is fully operational before you do this.

> Use this token to access the HTTP API: `22222222:APITOKEN`

> For a description of the Bot API, see this page: https://core.telegram.org/bots/api Father bot will return you the token (API key)

Copy the API Token (`22222222:APITOKEN` in the above example) and keep use it for the config parameter `token`.

Don't forget to start the conversation with your bot, by clicking `/START` button

### 2. Telegram user_id

#### Get your user id

Talk to the [userinfobot](https://telegram.me/userinfobot)

Get your "Id", you will use it for the config parameter `chat_id`.

#### Use Group id

To get the group ID, you can add the bot to the group, start freqtrade, and issue a `/tg_info` command.
This will return the group id to you, without having to use some random bot.
While "chat_id" is still required, it doesn't need to be set to this particular group id for this command.

The response will also contain the "topic_id" if necessary - both in a format ready to copy/paste into your configuration.

``` json
 {
    "enabled": true,
    "token": "********",
    "chat_id": "-1001332619709",
    "topic_id": "122"
}
```

For the Freqtrade configuration, you can then use the full value (including `-` ) as string:

```json
   "chat_id": "-1001332619709"
```

!!! Warning "Using telegram groups"
    When using telegram groups, you're giving every member of the telegram group access to your freqtrade bot and to all commands possible via telegram. Please make sure that you can trust everyone in the telegram group to avoid unpleasant surprises.

##### Group Topic ID

To use a specific topic in a group, you can use the `topic_id` parameter in the configuration. This will allow you to use the bot in a specific topic in a group.  
Without this, the bot will always respond to the general channel in the group if topics are enabled for a group chat.

```json
   "chat_id": "-1001332619709",
   "topic_id": "3"
```

Similar to the group-id - you can use `/tg_info` from the topic/thread to get the correct topic-id.

#### Authorized users

For groups, it can be useful to limit who can send commands to the bot.

If `"authorized_users": []` is present and empty, no user will be allowed to control the bot.
In the below example, only the user with the id "1234567" is allowed to control the bot - all other users will only be able to receive messages.

```json
   "chat_id": "-1001332619709",
   "topic_id": "3",
   "authorized_users": ["1234567"]
```

## Control telegram noise

Freqtrade provides means to control the verbosity of your telegram bot.
Each setting has the following possible values:

* `on` - Messages will be sent, and user will be notified.
* `silent` - Message will be sent, Notification will be without sound / vibration.
* `off` - Skip sending a message-type all together.

Example configuration showing the different settings:

``` json
"telegram": {
    "enabled": true,
    "token": "your_telegram_token",
    "chat_id": "your_telegram_chat_id",
    "allow_custom_messages": true,
    "notification_settings": {
        "status": "silent",
        "warning": "on",
        "startup": "off",
        "entry": "silent",
        "entry_fill": "on",
        "entry_cancel": "silent",
        "exit": {
            "roi": "silent",
            "emergency_exit": "on",
            "force_exit": "on",
            "exit_signal": "silent",
            "trailing_stop_loss": "on",
            "stop_loss": "on",
            "stoploss_on_exchange": "on",
            "custom_exit": "silent",  // custom_exit without specifying an exit reason
            "partial_exit": "on",
            // "custom_exit_message": "silent",  // Disable individual custom exit reasons
            "*": "off"  // Disable all other exit reasons
        },
        // "exit": "off",  // Simplistic configuration to disable all exit messages
        "exit_cancel": "on",
        "exit_fill": "off",
        "protection_trigger": "off",
        "protection_trigger_global": "on",
        "strategy_msg": "off",
        "show_candle": "off"
    },
    "reload": true,
    "balance_dust_level": 0.01
},
```

* `entry` notifications are sent when the order is placed, while `entry_fill` notifications are sent when the order is filled on the exchange.  
* `exit` notifications are sent when the order is placed, while `exit_fill` notifications are sent when the order is filled on the exchange.  
    Exit messages (`exit` and `exit_fill`) can be further controlled at individual exit reasons level, with the specific exit reason as the key. the default for all exit reasons is `on` - but can be configured via special `*` key - which will act as a wildcard for all exit reasons that are not explicitly defined.
* `*_fill` notifications are off by default and must be explicitly enabled.  
* `protection_trigger` notifications are sent when a protection triggers and `protection_trigger_global` notifications trigger when global protections are triggered.  
* `strategy_msg` - Receive notifications from the strategy, sent via `self.dp.send_msg()` from the strategy [more details](strategy-customization.md#send-notification).  
* `show_candle` - show candle values as part of entry/exit messages. Only possible values are `"ohlc"` or `"off"`.  
* `balance_dust_level` will define what the `/balance` command takes as "dust" - Currencies with a balance below this will be shown.  
* `allow_custom_messages` completely disable strategy messages.  
* `reload` allows you to disable reload-buttons on selected messages.  

## Create a custom keyboard (command shortcut buttons)

Telegram allows us to create a custom keyboard with buttons for commands.
The default custom keyboard looks like this.

```python
[
    ["/daily", "/profit", "/balance"], # row 1, 3 commands
    ["/status", "/status table", "/performance"], # row 2, 3 commands
    ["/count", "/start", "/stop", "/help"] # row 3, 4 commands
]
```

### Usage

You can create your own keyboard in `config.json`:

``` json
"telegram": {
      "enabled": true,
      "token": "your_telegram_token",
      "chat_id": "your_telegram_chat_id",
      "keyboard": [
          ["/daily", "/stats", "/balance", "/profit"],
          ["/status table", "/performance"],
          ["/reload_config", "/count", "/logs"]
      ]
   },
```

!!! Note "Supported Commands"
    Only the following commands are allowed. Command arguments are not supported!

    `/start`, `/stop`, `/status`, `/status table`, `/trades`, `/profit`, `/performance`, `/daily`, `/stats`, `/count`, `/locks`, `/balance`, `/stopentry`, `/reload_config`, `/show_config`, `/logs`, `/whitelist`, `/blacklist`, `/edge`, `/help`, `/version`, `/marketdir`

## Telegram commands

Per default, the Telegram bot shows predefined commands. Some commands
are only available by sending them to the bot. The table below list the
official commands. You can ask at any moment for help with `/help`.

|  Command | Description |
|----------|-------------|
| **System commands**
| `/start` | Starts the trader
| `/stop` | Stops the trader
| `/stopbuy | /stopentry` | Stops the trader from opening new trades. Gracefully closes open trades according to their rules.
| `/reload_config` | Reloads the configuration file
| `/show_config` | Shows part of the current configuration with relevant settings to operation
| `/logs [limit]` | Show last log messages.
| `/help` | Show help message
| `/version` | Show version
| **Status** |
| `/status` | Lists all open trades
| `/status <trade_id>` | Lists one or more specific trade. Separate multiple <trade_id> with a blank space.
| `/status table` | List all open trades in a table format. Pending buy orders are marked with an asterisk (*) Pending sell orders are marked with a double asterisk (**)
| `/order <trade_id>` | Lists orders of one or more specific trade. Separate multiple <trade_id> with a blank space.
| `/trades [limit]` | List all recently closed trades in a table format.
| `/count` | Displays number of trades used and available
| `/locks` | Show currently locked pairs.
| `/unlock <pair or lock_id>` | Remove the lock for this pair (or for this lock id).
| `/marketdir [long | short | even | none]` | Updates the user managed variable that represents the current market direction. If no direction is provided, the currently set direction will be displayed.
| `/list_custom_data <trade_id> [key]` | List custom_data for Trade ID & Key combination. If no Key is supplied it will list all key-value pairs found for that Trade ID.
| **Modify Trade states** |
| `/forceexit <trade_id> | /fx <tradeid>` | Instantly exits the given trade  (Ignoring `minimum_roi`).
| `/forceexit all | /fx all` | Instantly exits all open trades (Ignoring `minimum_roi`).
| `/fx` | alias for `/forceexit`
| `/forcelong <pair> [rate]` | Instantly buys the given pair. Rate is optional and only applies to limit orders. (`force_entry_enable` must be set to True)
| `/forceshort <pair> [rate]` | Instantly shorts the given pair. Rate is optional and only applies to limit orders. This will only work on non-spot markets. (`force_entry_enable` must be set to True)
| `/delete <trade_id>` | Delete a specific trade from the Database. Tries to close open orders. Requires manual handling of this trade on the exchange.
| `/reload_trade <trade_id>` | Reload a trade from the Exchange. Only works in live, and can potentially help recover a trade that was manually sold on the exchange.
| `/cancel_open_order <trade_id> | /coo <trade_id>` | Cancel an open order for a trade.
| **Metrics** |
| `/profit [<n>]` | Display a summary of your profit/loss from close trades and some stats about your performance, over the last n days (all trades by default)
| `/performance` | Show performance of each finished trade grouped by pair
| `/balance` | Show bot managed balance per currency
| `/balance full` | Show account balance per currency
| `/daily <n>` | Shows profit or loss per day, over the last n days (n defaults to 7)
| `/weekly <n>` | Shows profit or loss per week, over the last n weeks (n defaults to 8)
| `/monthly <n>` | Shows profit or loss per month, over the last n months (n defaults to 6)
| `/stats` | Shows Wins / losses by Exit reason as well as Avg. holding durations for buys and sells
| `/exits` | Shows Wins / losses by Exit reason as well as Avg. holding durations for buys and sells
| `/entries` | Shows Wins / losses by Exit reason as well as Avg. holding durations for buys and sells
| `/whitelist [sorted] [baseonly]` | Show the current whitelist. Optionally display in alphabetical order and/or with just the base currency of each pairing.
| `/blacklist [pair]` | Show the current blacklist, or adds a pair to the blacklist.
| `/edge` | Show validated pairs by Edge if it is enabled.

## Telegram commands in action

Below, example of Telegram message you will receive for each command.

### /start

> **Status:** `running`

### /stop

> `Stopping trader ...`
> **Status:** `stopped`

### /stopbuy

> **status:** `Setting max_open_trades to 0. Run /reload_config to reset.`

Prevents the bot from opening new trades by temporarily setting "max_open_trades" to 0. Open trades will be handled via their regular rules (ROI / Sell-signal, stoploss, ...).

After this, give the bot time to close off open trades (can be checked via `/status table`).
Once all positions are sold, run `/stop` to completely stop the bot.

`/reload_config` resets "max_open_trades" to the value set in the configuration and resets this command.

!!! Warning
    The stop-buy signal is ONLY active while the bot is running, and is not persisted anyway, so restarting the bot will cause this to reset.

### /status

For each open trade, the bot will send you the following message.
Enter Tag is configurable via Strategy.

> **Trade ID:** `123` `(since 1 days ago)`  
> **Current Pair:** CVC/BTC  
> **Direction:** Long  
> **Leverage:** 1.0  
> **Amount:** `26.64180098`  
> **Enter Tag:** Awesome Long Signal  
> **Open Rate:** `0.00007489`  
> **Current Rate:** `0.00007489`  
> **Unrealized Profit:** `12.95%`  
> **Stoploss:** `0.00007389 (-0.02%)`  

### /status table

Return the status of all open trades in a table format.

```
ID L/S    Pair     Since   Profit
----    --------  -------  --------
  67 L   SC/BTC    1 d      13.33%
 123 S   CVC/BTC   1 h      12.95%
```

### /count

Return the number of trades used and available.

```
current    max
---------  -----
     2     10
```

### /profit

Return a summary of your profit/loss and performance.

> **ROI:** Close trades  
>   ∙ `0.00485701 BTC (2.2%) (15.2 Σ%)`  
>   ∙ `62.968 USD`  
> **ROI:** All trades  
>   ∙ `0.00255280 BTC (1.5%) (6.43 Σ%)`  
>   ∙ `33.095 EUR`  
>  
> **Total Trade Count:** `138`  
> **Bot started:** `2022-07-11 18:40:44`  
> **First Trade opened:** `3 days ago`  
> **Latest Trade opened:** `2 minutes ago`  
> **Avg. Duration:** `2:33:45`  
> **Best Performing:** `PAY/BTC: 50.23%`  
> **Trading volume:** `0.5 BTC`  
> **Profit factor:** `1.04`  
> **Win / Loss:** `102 / 36`  
> **Winrate:** `73.91%`  
> **Expectancy (Ratio):** `4.87 (1.66)`  
> **Max Drawdown:** `9.23% (0.01255 BTC)`  

The relative profit of `1.2%` is the average profit per trade.  
The relative profit of `15.2 Σ%` is be based on the starting capital - so in this case, the starting capital was `0.00485701 * 1.152 = 0.00738 BTC`.  
**Starting capital(**) is either taken from the `available_capital` setting, or calculated by using current wallet size - profits.  
**Profit Factor** is calculated as gross profits / gross losses - and should serve as an overall metric for the strategy.  
**Expectancy** corresponds to the average return per currency unit at risk, i.e. the winrate and the risk-reward ratio (the average gain of winning trades compared to the average loss of losing trades).  
**Expectancy Ratio** is expected profit or loss of a subsequent trade based on the performance of all past trades.  
**Max drawdown** corresponds to the backtesting metric `Absolute Drawdown (Account)` - calculated as `(Absolute Drawdown) / (DrawdownHigh + startingBalance)`.  
**Bot started date** will refer to the date the bot was first started. For older bots, this will default to the first trade's open date.  

### /forceexit <trade_id>

> **BINANCE:** Exiting BTC/LTC with limit `0.01650000 (profit: ~-4.07%, -0.00008168)`

!!! Tip
    You can get a list of all open trades by calling `/forceexit` without parameter, which will show a list of buttons to simply exit a trade.
    This command has an alias in `/fx` - which has the same capabilities, but is faster to type in "emergency" situations.

### /forcelong <pair> [rate] | /forceshort <pair> [rate]

`/forcebuy <pair> [rate]` is also supported for longs but should be considered deprecated.

> **BINANCE:** Long ETH/BTC with limit `0.03400000` (`1.000000 ETH`, `225.290 USD`)

Omitting the pair will open a query asking for the pair to trade (based on the current whitelist).
Trades created through `/forcelong` will have the buy-tag of `force_entry`.

![Telegram force-buy screenshot](assets/telegram_forcebuy.png)

Note that for this to work, `force_entry_enable` needs to be set to true.

[More details](configuration.md#understand-force_entry_enable)

### /performance

Return the performance of each crypto-currency the bot has sold.
> Performance:  
> 1. `RCN/BTC 0.003 BTC (57.77%) (1)`  
> 2. `PAY/BTC 0.0012 BTC (56.91%) (1)`  
> 3. `VIB/BTC 0.0011 BTC (47.07%) (1)`  
> 4. `SALT/BTC 0.0010 BTC (30.24%) (1)`  
> 5. `STORJ/BTC 0.0009 BTC (27.24%) (1)`  
> ...  

The relative performance is calculated against the total investment in the currency, aggregating all filled entries for the currency.

### /balance

Return the balance of all crypto-currency your have on the exchange.

> **Currency:** BTC  
> **Available:** 3.05890234  
> **Balance:** 3.05890234  
> **Pending:** 0.0  
>
> **Currency:** CVC  
> **Available:** 86.64180098  
> **Balance:** 86.64180098  
> **Pending:** 0.0  

### /daily <n>

Per default `/daily` will return the 7 last days. The example below if for `/daily 3`:

> **Daily Profit over the last 3 days:**

```
Day (count)     USDT          USD         Profit %
--------------  ------------  ----------  ----------
2022-06-11 (1)  -0.746 USDT   -0.75 USD   -0.08%
2022-06-10 (0)  0 USDT        0.00 USD    0.00%
2022-06-09 (5)  20 USDT       20.10 USD   5.00%
```

### /weekly <n>

Per default `/weekly` will return the 8 last weeks, including the current week. Each week starts
from Monday. The example below if for `/weekly 3`:

> **Weekly Profit over the last 3 weeks (starting from Monday):**

```
Monday (count)  Profit BTC      Profit USD   Profit %
-------------  --------------  ------------    ----------
2018-01-03 (5)  0.00224175 BTC  29,142 USD   4.98%
2017-12-27 (1)  0.00033131 BTC   4,307 USD   0.00%
2017-12-20 (4)  0.00269130 BTC  34.986 USD   5.12%
```

### /monthly <n>

Per default `/monthly` will return the 6 last months, including the current month. The example below
if for `/monthly 3`:

> **Monthly Profit over the last 3 months:**
```
Month (count)  Profit BTC      Profit USD    Profit %
-------------  --------------  ------------    ----------
2018-01 (20)    0.00224175 BTC  29,142 USD  4.98%
2017-12 (5)    0.00033131 BTC   4,307 USD   0.00%
2017-11 (10)    0.00269130 BTC  34.986 USD  5.10%
```

### /whitelist

Shows the current whitelist

> Using whitelist `StaticPairList` with 22 pairs  
> `IOTA/BTC, NEO/BTC, TRX/BTC, VET/BTC, ADA/BTC, ETC/BTC, NCASH/BTC, DASH/BTC, XRP/BTC, XVG/BTC, EOS/BTC, LTC/BTC, OMG/BTC, BTG/BTC, LSK/BTC, ZEC/BTC, HOT/BTC, IOTX/BTC, XMR/BTC, AST/BTC, XLM/BTC, NANO/BTC`

### /blacklist [pair]

Shows the current blacklist.
If Pair is set, then this pair will be added to the pairlist.
Also supports multiple pairs, separated by a space.  
Use `/reload_config` to reset the blacklist.

> Using blacklist `StaticPairList` with 2 pairs  
>`DODGE/BTC`, `HOT/BTC`.  

### /edge

Shows pairs validated by Edge along with their corresponding win-rate, expectancy and stoploss values.

> **Edge only validated following pairs:**
```
Pair        Winrate    Expectancy    Stoploss
--------  ---------  ------------  ----------
DOCK/ETH   0.522727      0.881821       -0.03
PHX/ETH    0.677419      0.560488       -0.03
HOT/ETH    0.733333      0.490492       -0.03
HC/ETH     0.588235      0.280988       -0.02
ARDR/ETH   0.366667      0.143059       -0.01
```

### /version

> **Version:** `0.14.3`

### /marketdir

If a market direction is provided the command updates the user managed variable that represents the current market direction.
This variable is not set to any valid market direction on bot startup and must be set by the user. The example below is for `/marketdir long`:

```
Successfully updated marketdirection from none to long.
```

If no market direction is provided the command outputs the currently set market directions. The example below is for `/marketdir`:

```
Currently set marketdirection: even
```

You can use the market direction in your strategy via `self.market_direction`.

!!! Warning "Bot restarts"
    Please note that the market direction is not persisted, and will be reset after a bot restart/reload.

!!! Danger "Backtesting"
    As this value/variable is intended to be changed manually in dry/live trading.
    Strategies using `market_direction` will probably not produce reliable, reproducible results (changes to this variable will not be reflected for backtesting). Use at your own risk.

---

## File: trade-object.md

# Trade Object

## Trade

A position freqtrade enters is stored in a `Trade` object - which is persisted to the database.
It's a core concept of freqtrade - and something you'll come across in many sections of the documentation, which will most likely point you to this location.

It will be passed to the strategy in many [strategy callbacks](strategy-callbacks.md). The object passed to the strategy cannot be modified directly. Indirect modifications may occur based on callback results.

## Trade - Available attributes

The following attributes / properties are available for each individual trade - and can be used with `trade.<property>` (e.g. `trade.pair`).

|  Attribute | DataType | Description |
|------------|-------------|-------------|
| `pair` | string | Pair of this trade. |
| `is_open` | boolean | Is the trade currently open, or has it been concluded. |
| `open_rate` | float | Rate this trade was entered at (Avg. entry rate in case of trade-adjustments). |
| `close_rate` | float | Close rate - only set when is_open = False. |
| `stake_amount` | float | Amount in Stake (or Quote) currency. |
| `amount` | float | Amount in Asset / Base currency that is currently owned. Will be 0.0 until the initial order fills. |
| `open_date` | datetime | Timestamp when trade was opened **use `open_date_utc` instead** |
| `open_date_utc` | datetime | Timestamp when trade was opened - in UTC. |
| `close_date` | datetime | Timestamp when trade was closed **use `close_date_utc` instead** |
| `close_date_utc` | datetime | Timestamp when trade was closed - in UTC. |
| `close_profit` | float | Relative profit at the time of trade closure. `0.01` == 1% |
| `close_profit_abs` | float | Absolute profit (in stake currency) at the time of trade closure. |
| `leverage` | float | Leverage used for this trade - defaults to 1.0 in spot markets. |
| `enter_tag` | string | Tag provided on entry via the `enter_tag` column in the dataframe. |
| `is_short` | boolean | True for short trades, False otherwise. |
| `orders` | Order[] | List of order objects attached to this trade (includes both filled and cancelled orders). |
| `date_last_filled_utc` | datetime | Time of the last filled order. |
| `entry_side` | "buy" / "sell" | Order Side the trade was entered. |
| `exit_side` | "buy" / "sell" | Order Side that will result in a trade exit / position reduction. |
| `trade_direction` | "long" / "short" | Trade direction in text - long or short. |
| `nr_of_successful_entries` | int | Number of successful (filled) entry orders. |
| `nr_of_successful_exits` | int | Number of successful (filled) exit orders. |
| `has_open_orders` | boolean | Has the trade open orders (excluding stoploss orders). |

## Class methods

The following are class methods - which return generic information, and usually result in an explicit query against the database.
They can be used as `Trade.<method>` - e.g. `open_trades = Trade.get_open_trade_count()`

!!! Warning "Backtesting/hyperopt"
    Most methods will work in both backtesting / hyperopt and live/dry modes.
    During backtesting, it's limited to usage in [strategy callbacks](strategy-callbacks.md). Usage in `populate_*()` methods is not supported and will result in wrong results.

### get_trades_proxy

When your strategy needs some information on existing (open or close) trades - it's best to use `Trade.get_trades_proxy()`.

Usage:

``` python
from freqtrade.persistence import Trade
from datetime import timedelta

# ...
trade_hist = Trade.get_trades_proxy(pair='ETH/USDT', is_open=False, open_date=current_date - timedelta(days=2))

```

`get_trades_proxy()` supports the following keyword arguments. All arguments are optional - calling `get_trades_proxy()` without arguments will return a list of all trades in the database.

* `pair` e.g. `pair='ETH/USDT'`
* `is_open` e.g. `is_open=False`
* `open_date` e.g. `open_date=current_date - timedelta(days=2)`
* `close_date` e.g. `close_date=current_date - timedelta(days=5)`

### get_open_trade_count

Get the number of currently open trades

``` python
from freqtrade.persistence import Trade
# ...
open_trades = Trade.get_open_trade_count()
```

### get_total_closed_profit

Retrieve the total profit the bot has generated so far.
Aggregates `close_profit_abs` for all closed trades.

``` python
from freqtrade.persistence import Trade

# ...
profit = Trade.get_total_closed_profit()
```

### total_open_trades_stakes

Retrieve the total stake_amount that's currently in trades.

``` python
from freqtrade.persistence import Trade

# ...
profit = Trade.total_open_trades_stakes()
```

### get_overall_performance

Retrieve the overall performance - similar to the `/performance` telegram command.

``` python
from freqtrade.persistence import Trade

# ...
if self.config['runmode'].value in ('live', 'dry_run'):
    performance = Trade.get_overall_performance()
```

Sample return value: ETH/BTC had 5 trades, with a total profit of 1.5% (ratio of 0.015).

``` json
{"pair": "ETH/BTC", "profit": 0.015, "count": 5}
```

## Order Object

An `Order` object represents an order on the exchange (or a simulated order in dry-run mode).
An `Order` object will always be tied to it's corresponding [`Trade`](#trade-object), and only really makes sense in the context of a trade.

### Order - Available attributes

an Order object is typically attached to a trade.
Most properties here can be None as they are dependent on the exchange response.

|  Attribute | DataType | Description |
|------------|-------------|-------------|
| `trade` | Trade | Trade object this order is attached to |
| `ft_pair` | string | Pair this order is for |
| `ft_is_open` | boolean | is the order filled? |
| `order_type` | string | Order type as defined on the exchange - usually market, limit or stoploss |
| `status` | string | Status as defined by ccxt. Usually open, closed, expired or canceled |
| `side` | string | Buy or Sell |
| `price` | float | Price the order was placed at |
| `average` | float | Average price the order filled at |
| `amount` | float | Amount in base currency |
| `filled` | float | Filled amount (in base currency) |
| `remaining` | float | Remaining amount |
| `cost` | float | Cost of the order - usually average * filled (*Exchange dependent on futures, may contain the cost with or without leverage and may be in contracts.*) |
| `stake_amount` | float | Stake amount used for this order. *Added in 2023.7.* |
| `stake_amount_filled` | float | Filled Stake amount used for this order. *Added in 2024.11.* |
| `order_date` | datetime | Order creation date **use `order_date_utc` instead** |
| `order_date_utc` | datetime | Order creation date (in UTC) |
| `order_fill_date` | datetime |  Order fill date **use `order_fill_utc` instead** |
| `order_fill_date_utc` | datetime | Order fill date |

---

## File: updating.md

# How to update

To update your freqtrade installation, please use one of the below methods, corresponding to your installation method.

!!! Note "Tracking changes"
    Breaking changes / changed behavior will be documented in the changelog that is posted alongside every release.
    For the develop branch, please follow PR's to avoid being surprised by changes.

## Docker

!!! Note "Legacy installations using the `master` image"
    We're switching from master to stable for the release Images - please adjust your docker-file and replace `freqtradeorg/freqtrade:master` with `freqtradeorg/freqtrade:stable`

``` bash
docker compose pull
docker compose up -d
```

## Installation via setup script

``` bash
./setup.sh --update
```

!!! Note
    Make sure to run this command with your virtual environment disabled!

## Plain native installation

Please ensure that you're also updating dependencies - otherwise things might break without you noticing.

``` bash
git pull
pip install -U -r requirements.txt
pip install -e .

# Ensure freqUI is at the latest version
freqtrade install-ui 
```

### Problems updating

Update-problems usually come missing dependencies (you didn't follow the above instructions) - or from updated dependencies, which fail to install (for example TA-lib).
Please refer to the corresponding installation sections (common problems linked below)

Common problems and their solutions:

* [ta-lib update on windows](windows_installation.md#2-install-ta-lib)

---

## File: utils.md

# Utility Subcommands

Besides the Live-Trade and Dry-Run run modes, the `backtesting`, `edge` and `hyperopt` optimization subcommands, and the `download-data` subcommand which prepares historical data, the bot contains a number of utility subcommands. They are described in this section.

## Create userdir

Creates the directory structure to hold your files for freqtrade.
Will also create strategy and hyperopt examples for you to get started.
Can be used multiple times - using `--reset` will reset the sample strategy and hyperopt files to their default state.

--8<-- "commands/create-userdir.md"

!!! Warning
    Using `--reset` may result in loss of data, since this will overwrite all sample files without asking again.

```
├── backtest_results
├── data
├── hyperopt_results
├── hyperopts
│   ├── sample_hyperopt_loss.py
├── notebooks
│   └── strategy_analysis_example.ipynb
├── plot
└── strategies
    └── sample_strategy.py
```

## Create new config

Creates a new configuration file, asking some questions which are important selections for a configuration.

--8<-- "commands/new-config.md"

!!! Warning
    Only vital questions are asked. Freqtrade offers a lot more configuration possibilities, which are listed in the [Configuration documentation](configuration.md#configuration-parameters)

### Create config examples

```
$ freqtrade new-config --config user_data/config_binance.json

? Do you want to enable Dry-run (simulated trades)?  Yes
? Please insert your stake currency: BTC
? Please insert your stake amount: 0.05
? Please insert max_open_trades (Integer or -1 for unlimited open trades): 3
? Please insert your desired timeframe (e.g. 5m): 5m
? Please insert your display Currency (for reporting): USD
? Select exchange  binance
? Do you want to enable Telegram?  No
```

## Show config

Show configuration file (with sensitive values redacted by default).
Especially useful with [split configuration files](configuration.md#multiple-configuration-files) or [environment variables](configuration.md#environment-variables), where this command will show the merged configuration.

![Show config output](assets/show-config-output.png)

--8<-- "commands/show-config.md"

``` output
Your combined configuration is:
{
  "exit_pricing": {
    "price_side": "other",
    "use_order_book": true,
    "order_book_top": 1
  },
  "stake_currency": "USDT",
  "exchange": {
    "name": "binance",
    "key": "REDACTED",
    "secret": "REDACTED",
    "ccxt_config": {},
    "ccxt_async_config": {},
  }
  // ...
}
```

!!! Warning "Sharing information provided by this command"
    We try to remove all known sensitive information from the default output (without `--show-sensitive`). 
    Yet, please do double-check for sensitive values in your output to make sure you're not accidentally exposing some private info.

## Create new strategy

Creates a new strategy from a template similar to SampleStrategy.
The file will be named inline with your class name, and will not overwrite existing files.

Results will be located in `user_data/strategies/<strategyclassname>.py`.

--8<-- "commands/new-strategy.md"

### Sample usage of new-strategy

```bash
freqtrade new-strategy --strategy AwesomeStrategy
```

With custom user directory

```bash
freqtrade new-strategy --userdir ~/.freqtrade/ --strategy AwesomeStrategy
```

Using the advanced template (populates all optional functions and methods)

```bash
freqtrade new-strategy --strategy AwesomeStrategy --template advanced
```

## List Strategies

Use the `list-strategies` subcommand to see all strategies in one particular directory.

This subcommand is useful for finding problems in your environment with loading strategies: modules with strategies that contain errors and failed to load are printed in red (LOAD FAILED), while strategies with duplicate names are printed in yellow (DUPLICATE NAME).

--8<-- "commands/list-strategies.md"

!!! Warning
    Using these commands will try to load all python files from a directory. This can be a security risk if untrusted files reside in this directory, since all module-level code is executed.

Example: Search default strategies directories (within the default userdir).

``` bash
freqtrade list-strategies
```

Example: Search strategies  directory within the userdir.

``` bash
freqtrade list-strategies --userdir ~/.freqtrade/
```

Example: Search dedicated strategy path.

``` bash
freqtrade list-strategies --strategy-path ~/.freqtrade/strategies/
```

## List Hyperopt-Loss functions

Use the `list-hyperoptloss` subcommand to see all hyperopt loss functions available.

It provides a quick list of all available loss functions in your environment.

This subcommand can be useful for finding problems in your environment with loading loss functions: modules with Hyperopt-Loss functions that contain errors and failed to load are printed in red (LOAD FAILED), while hyperopt-Loss functions with duplicate names are printed in yellow (DUPLICATE NAME).

--8<-- "commands/list-hyperoptloss.md"

## List freqAI models

Use the `list-freqaimodels` subcommand to see all freqAI models available.

This subcommand is useful for finding problems in your environment with loading freqAI models: modules with models that contain errors and failed to load are printed in red (LOAD FAILED), while models with duplicate names are printed in yellow (DUPLICATE NAME).

--8<-- "commands/list-freqaimodels.md"

## List Exchanges

Use the `list-exchanges` subcommand to see the exchanges available for the bot.

--8<-- "commands/list-exchanges.md"

Example: see exchanges available for the bot:

```
$ freqtrade list-exchanges
Exchanges available for Freqtrade:
Exchange name       Supported    Markets                 Reason
------------------  -----------  ----------------------  ------------------------------------------------------------------------
binance             Official     spot, isolated futures
bitmart             Official     spot
bybit                            spot, isolated futures
gate                Official     spot, isolated futures
htx                 Official     spot
huobi                            spot
kraken              Official     spot
okx                 Official     spot, isolated futures
```

!!! info ""
    Output reduced for clarity - supported and available exchanges may change over time.

!!! Note "missing opt exchanges"
    Values with "missing opt:" might need special configuration (e.g. using orderbook if `fetchTickers` is missing) - but should in theory work (although we cannot guarantee they will).

Example: see all exchanges supported by the ccxt library (including 'bad' ones, i.e. those that are known to not work with Freqtrade)

```
$ freqtrade list-exchanges -a
All exchanges supported by the ccxt library:
Exchange name       Valid    Supported    Markets                 Reason
------------------  -------  -----------  ----------------------  ---------------------------------------------------------------------------------
binance             True     Official     spot, isolated futures
bitflyer            False                 spot                    missing: fetchOrder. missing opt: fetchTickers.
bitmart             True     Official     spot
bybit               True                  spot, isolated futures
gate                True     Official     spot, isolated futures
htx                 True     Official     spot
kraken              True     Official     spot
okx                 True     Official     spot, isolated futures
```

!!! info ""
    Reduced output - supported and available exchanges may change over time.

## List Timeframes

Use the `list-timeframes` subcommand to see the list of timeframes available for the exchange.

--8<-- "commands/list-timeframes.md"

* Example: see the timeframes for the 'binance' exchange, set in the configuration file:

```
$ freqtrade list-timeframes -c config_binance.json
...
Timeframes available for the exchange `binance`: 1m, 3m, 5m, 15m, 30m, 1h, 2h, 4h, 6h, 8h, 12h, 1d, 3d, 1w, 1M
```

* Example: enumerate exchanges available for Freqtrade and print timeframes supported by each of them:
```
$ for i in `freqtrade list-exchanges -1`; do freqtrade list-timeframes --exchange $i; done
```

## List pairs/list markets

The `list-pairs` and `list-markets` subcommands allow to see the pairs/markets available on exchange.

Pairs are markets with the '/' character between the base currency part and the quote currency part in the market symbol.
For example, in the 'ETH/BTC' pair 'ETH' is the base currency, while 'BTC' is the quote currency.

For pairs traded by Freqtrade the pair quote currency is defined by the value of the `stake_currency` configuration setting.

You can print info about any pair/market with these subcommands - and you can filter output by quote-currency using `--quote BTC`, or by base-currency using `--base ETH` options correspondingly.

These subcommands have same usage and same set of available options:

--8<-- "commands/list-pairs.md"

By default, only active pairs/markets are shown. Active pairs/markets are those that can currently be traded on the exchange.
You can use the `-a`/`-all` option to see the list of all pairs/markets, including the inactive ones.
Pairs may be listed as untradeable if the smallest tradeable price for the market is very small, i.e. less than `1e-11` (`0.00000000001`)

Pairs/markets are sorted by its symbol string in the printed output.

### Examples

* Print the list of active pairs with quote currency USD on exchange, specified in the default
configuration file (i.e. pairs on the "Binance" exchange) in JSON format:

```
$ freqtrade list-pairs --quote USD --print-json
```

* Print the list of all pairs on the exchange, specified in the `config_binance.json` configuration file
(i.e. on the "Binance" exchange) with base currencies BTC or ETH and quote currencies USDT or USD, as the
human-readable list with summary:

```
$ freqtrade list-pairs -c config_binance.json --all --base BTC ETH --quote USDT USD --print-list
```

* Print all markets on exchange "Kraken", in the tabular format:

```
$ freqtrade list-markets --exchange kraken --all
```

## Test pairlist

Use the `test-pairlist` subcommand to test the configuration of [dynamic pairlists](plugins.md#pairlists).

Requires a configuration with specified `pairlists` attribute.
Can be used to generate static pairlists to be used during backtesting / hyperopt.

--8<-- "commands/test-pairlist.md"

### Examples

Show whitelist when using a [dynamic pairlist](plugins.md#pairlists).

```
freqtrade test-pairlist --config config.json --quote USDT BTC
```

## Convert database

`freqtrade convert-db` can be used to convert your database from one system to another (sqlite -> postgres, postgres -> other postgres), migrating all trades, orders and Pairlocks.

Please refer to the [corresponding documentation](advanced-setup.md#use-a-different-database-system) to learn about requirements for different database systems.

--8<-- "commands/convert-db.md"

!!! Warning
    Please ensure to only use this on an empty target database. Freqtrade will perform a regular migration, but may fail if entries already existed.

## Webserver mode

!!! Warning "Experimental"
    Webserver mode is an experimental mode to increase backesting and strategy development productivity.
    There may still be bugs - so if you happen to stumble across these, please report them as github issues, thanks.

Run freqtrade in webserver mode.
Freqtrade will start the webserver and allow FreqUI to start and control backtesting processes.
This has the advantage that data will not be reloaded between backtesting runs (as long as timeframe and timerange remain identical).
FreqUI will also show the backtesting results.

--8<-- "commands/webserver.md"

### Webserver mode - docker

You can also use webserver mode via docker.
Starting a one-off container requires the configuration of the port explicitly, as ports are not exposed by default.
You can use `docker compose run --rm -p 127.0.0.1:8080:8080 freqtrade webserver` to start a one-off container that'll be removed once you stop it. This assumes that port 8080 is still available and no other bot is running on that port.

Alternatively, you can reconfigure the docker-compose file to have the command updated:

``` yml
    command: >
      webserver
      --config /freqtrade/user_data/config.json
```

You can now use `docker compose up` to start the webserver.
This assumes that the configuration has a webserver enabled and configured for docker (listening port = `0.0.0.0`).

!!! Tip
    Don't forget to reset the command back to the trade command if you want to start a live or dry-run bot. 

## Show previous Backtest results

Allows you to show previous backtest results.
Adding `--show-pair-list` outputs a sorted pair list you can easily copy/paste into your configuration (omitting bad pairs).

??? Warning "Strategy overfitting"
    Only using winning pairs can lead to an overfitted strategy, which will not work well on future data. Make sure to extensively test your strategy in dry-run before risking real money.

--8<-- "commands/backtesting-show.md"

## Detailed backtest analysis

Advanced backtest result analysis.

More details in the [Backtesting analysis](advanced-backtesting.md#analyze-the-buyentry-and-sellexit-tags) Section.

--8<-- "commands/backtesting-analysis.md"

## List Hyperopt results

You can list the hyperoptimization epochs the Hyperopt module evaluated previously with the `hyperopt-list` sub-command.

--8<-- "commands/hyperopt-list.md"

!!! Note
    `hyperopt-list` will automatically use the latest available hyperopt results file.
    You can override this using the `--hyperopt-filename` argument, and specify another, available filename (without path!).

### Examples

List all results, print details of the best result at the end:
```
freqtrade hyperopt-list
```

List only epochs with positive profit. Do not print the details of the best epoch, so that the list can be iterated in a script:
```
freqtrade hyperopt-list --profitable --no-details
```

## Show details of Hyperopt results

You can show the details of any hyperoptimization epoch previously evaluated by the Hyperopt module with the `hyperopt-show` subcommand.

--8<-- "commands/hyperopt-show.md"

!!! Note
    `hyperopt-show` will automatically use the latest available hyperopt results file.
    You can override this using the `--hyperopt-filename` argument, and specify another, available filename (without path!).

### Examples

Print details for the epoch 168 (the number of the epoch is shown by the `hyperopt-list` subcommand or by Hyperopt itself during hyperoptimization run):

```
freqtrade hyperopt-show -n 168
```

Prints JSON data with details for the last best epoch (i.e., the best of all epochs):

```
freqtrade hyperopt-show --best -n -1 --print-json --no-header
```

## Show trades

Print selected (or all) trades from database to screen.

--8<-- "commands/show-trades.md"

### Examples

Print trades with id 2 and 3 as json

``` bash
freqtrade show-trades --db-url sqlite:///tradesv3.sqlite --trade-ids 2 3 --print-json
```

## Strategy-Updater

Updates listed strategies or all strategies within the strategies folder to be v3 compliant.
If the command runs without --strategy-list then all strategies inside the strategies folder will be converted.
Your original strategy will remain available in the `user_data/strategies_orig_updater/` directory.

!!! Warning "Conversion results"
    Strategy updater will work on a "best effort" approach. Please do your due diligence and verify the results of the conversion.
    We also recommend to run a python formatter (e.g. `black`) to format results in a sane manner.

--8<-- "commands/strategy-updater.md"

---

## File: webhook-config.md

# Webhook usage

## Configuration

Enable webhooks by adding a webhook-section to your configuration file, and setting `webhook.enabled` to `true`.

Sample configuration (tested using IFTTT).

```json
  "webhook": {
        "enabled": true,
        "url": "https://maker.ifttt.com/trigger/<YOUREVENT>/with/key/<YOURKEY>/",
        "entry": {
            "value1": "Buying {pair}",
            "value2": "limit {limit:8f}",
            "value3": "{stake_amount:8f} {stake_currency}"
        },
        "entry_cancel": {
            "value1": "Cancelling Open Buy Order for {pair}",
            "value2": "limit {limit:8f}",
            "value3": "{stake_amount:8f} {stake_currency}"
        },
         "entry_fill": {
            "value1": "Buy Order for {pair} filled",
            "value2": "at {open_rate:8f}",
            "value3": ""
        },
        "exit": {
            "value1": "Exiting {pair}",
            "value2": "limit {limit:8f}",
            "value3": "profit: {profit_amount:8f} {stake_currency} ({profit_ratio})"
        },
        "exit_cancel": {
            "value1": "Cancelling Open Exit Order for {pair}",
            "value2": "limit {limit:8f}",
            "value3": "profit: {profit_amount:8f} {stake_currency} ({profit_ratio})"
        },
        "exit_fill": {
            "value1": "Exit Order for {pair} filled",
            "value2": "at {close_rate:8f}.",
            "value3": ""
        },
        "status": {
            "value1": "Status: {status}",
            "value2": "",
            "value3": ""
        }
    },
```

The url in `webhook.url` should point to the correct url for your webhook. If you're using [IFTTT](https://ifttt.com) (as shown in the sample above) please insert your event and key to the url.

You can set the POST body format to Form-Encoded (default), JSON-Encoded, or raw data. Use `"format": "form"`, `"format": "json"`, or `"format": "raw"` respectively. Example configuration for Mattermost Cloud integration:

```json
  "webhook": {
        "enabled": true,
        "url": "https://<YOURSUBDOMAIN>.cloud.mattermost.com/hooks/<YOURHOOK>",
        "format": "json",
        "status": {
            "text": "Status: {status}"
        }
    },
```

The result would be a POST request with e.g. `{"text":"Status: running"}` body and `Content-Type: application/json` header which results `Status: running` message in the Mattermost channel.

When using the Form-Encoded or JSON-Encoded configuration you can configure any number of payload values, and both the key and value will be output in the POST request. However, when using the raw data format you can only configure one value and it **must** be named `"data"`. In this instance the data key will not be output in the POST request, only the value. For example:

```json
  "webhook": {
        "enabled": true,
        "url": "https://<YOURHOOKURL>",
        "format": "raw",
        "webhookstatus": {
            "data": "Status: {status}"
        }
    },
```

The result would be a POST request with e.g. `Status: running` body and `Content-Type: text/plain` header.

## Additional configurations

The `webhook.retries` parameter can be set for the maximum number of retries the webhook request should attempt if it is unsuccessful (i.e. HTTP response status is not 200). By default this is set to `0` which is disabled. An additional `webhook.retry_delay` parameter can be set to specify the time in seconds between retry attempts. By default this is set to `0.1` (i.e. 100ms). Note that increasing the number of retries or retry delay may slow down the trader if there are connectivity issues with the webhook.
You can also specify `webhook.timeout` - which defines how long the bot will wait until it assumes the other host as unresponsive (defaults to 10s).

Example configuration for retries:

```json
  "webhook": {
        "enabled": true,
        "url": "https://<YOURHOOKURL>",
        "timeout": 10,
        "retries": 3,
        "retry_delay": 0.2,
        "status": {
            "status": "Status: {status}"
        }
    },
```

Custom messages can be sent to Webhook endpoints via the `self.dp.send_msg()` function from within the strategy. To enable this, set the `allow_custom_messages` option to `true`:

```json
  "webhook": {
        "enabled": true,
        "url": "https://<YOURHOOKURL>",
        "allow_custom_messages": true,
        "strategy_msg": {
            "status": "StrategyMessage: {msg}"
        }
    },
```

Different payloads can be configured for different events. Not all fields are necessary, but you should configure at least one of the dicts, otherwise the webhook will never be called.

## Webhook Message types

### Entry

The fields in `webhook.entry` are filled when the bot executes a long/short. Parameters are filled using string.format.
Possible parameters are:

* `trade_id`
* `exchange`
* `pair`
* `direction`
* `leverage`
* ~~`limit` # Deprecated - should no longer be used.~~
* `open_rate`
* `amount`
* `open_date`
* `stake_amount`
* `stake_currency`
* `base_currency`
* `quote_currency`
* `fiat_currency`
* `order_type`
* `current_rate`
* `enter_tag`

### Entry cancel

The fields in `webhook.entry_cancel` are filled when the bot cancels a long/short order. Parameters are filled using string.format.
Possible parameters are:

* `trade_id`
* `exchange`
* `pair`
* `direction`
* `leverage`
* `limit`
* `amount`
* `open_date`
* `stake_amount`
* `stake_currency`
* `base_currency`
* `quote_currency`
* `fiat_currency`
* `order_type`
* `current_rate`
* `enter_tag`

### Entry fill

The fields in `webhook.entry_fill` are filled when the bot filled a long/short order. Parameters are filled using string.format.
Possible parameters are:

* `trade_id`
* `exchange`
* `pair`
* `direction`
* `leverage`
* `open_rate`
* `amount`
* `open_date`
* `stake_amount`
* `stake_currency`
* `base_currency`
* `quote_currency`
* `fiat_currency`
* `order_type`
* `current_rate`
* `enter_tag`

### Exit

The fields in `webhook.exit` are filled when the bot exits a trade. Parameters are filled using string.format.
Possible parameters are:

* `trade_id`
* `exchange`
* `pair`
* `direction`
* `leverage`
* `gain`
* `limit`
* `amount`
* `open_rate`
* `profit_amount`
* `profit_ratio`
* `stake_currency`
* `base_currency`
* `quote_currency`
* `fiat_currency`
* `exit_reason`
* `order_type`
* `open_date`
* `close_date`

### Exit fill

The fields in `webhook.exit_fill` are filled when the bot fills a exit order (closes a Trade). Parameters are filled using string.format.
Possible parameters are:

* `trade_id`
* `exchange`
* `pair`
* `direction`
* `leverage`
* `gain`
* `close_rate`
* `amount`
* `open_rate`
* `current_rate`
* `profit_amount`
* `profit_ratio`
* `stake_currency`
* `base_currency`
* `quote_currency`
* `fiat_currency`
* `exit_reason`
* `order_type`
* `open_date`
* `close_date`

### Exit cancel

The fields in `webhook.exit_cancel` are filled when the bot cancels a exit order. Parameters are filled using string.format.
Possible parameters are:

* `trade_id`
* `exchange`
* `pair`
* `direction`
* `leverage`
* `gain`
* `limit`
* `amount`
* `open_rate`
* `current_rate`
* `profit_amount`
* `profit_ratio`
* `stake_currency`
* `base_currency`
* `quote_currency`
* `fiat_currency`
* `exit_reason`
* `order_type`
* `open_date`
* `close_date`

### Status

The fields in `webhook.status` are used for regular status messages (Started / Stopped / ...). Parameters are filled using string.format.

The only possible value here is `{status}`.

## Discord

A special form of webhooks is available for discord.
You can configure this as follows:

```json
"discord": {
    "enabled": true,
    "webhook_url": "https://discord.com/api/webhooks/<Your webhook URL ...>",
    "exit_fill": [
        {"Trade ID": "{trade_id}"},
        {"Exchange": "{exchange}"},
        {"Pair": "{pair}"},
        {"Direction": "{direction}"},
        {"Open rate": "{open_rate}"},
        {"Close rate": "{close_rate}"},
        {"Amount": "{amount}"},
        {"Open date": "{open_date:%Y-%m-%d %H:%M:%S}"},
        {"Close date": "{close_date:%Y-%m-%d %H:%M:%S}"},
        {"Profit": "{profit_amount} {stake_currency}"},
        {"Profitability": "{profit_ratio:.2%}"},
        {"Enter tag": "{enter_tag}"},
        {"Exit Reason": "{exit_reason}"},
        {"Strategy": "{strategy}"},
        {"Timeframe": "{timeframe}"},
    ],
    "entry_fill": [
        {"Trade ID": "{trade_id}"},
        {"Exchange": "{exchange}"},
        {"Pair": "{pair}"},
        {"Direction": "{direction}"},
        {"Open rate": "{open_rate}"},
        {"Amount": "{amount}"},
        {"Open date": "{open_date:%Y-%m-%d %H:%M:%S}"},
        {"Enter tag": "{enter_tag}"},
        {"Strategy": "{strategy} {timeframe}"},
    ]
}
```

The above represents the default (`exit_fill` and `entry_fill` are optional and will default to the above configuration) - modifications are obviously possible.
To disable either of the two default values (`entry_fill` / `exit_fill`), you can assign them an empty array (`exit_fill: []`).

Available fields correspond to the fields for webhooks and are documented in the corresponding webhook sections.

The notifications will look as follows by default.

![discord-notification](assets/discord_notification.png)

Custom messages can be sent from a strategy to Discord endpoints via the dataprovider.send_msg() function. To enable this, set the `allow_custom_messages` option to `true`:

```json
  "discord": {
        "enabled": true,
        "webhook_url": "https://discord.com/api/webhooks/<Your webhook URL ...>",
        "allow_custom_messages": true,
    },
```

---

## File: windows_installation.md

# Windows installation

We **strongly** recommend that Windows users use [Docker](docker_quickstart.md) as this will work much easier and smoother (also more secure).

If that is not possible, try using the Windows Linux subsystem (WSL) - for which the Ubuntu instructions should work.
Otherwise, please follow the instructions below.

All instructions assume that python 3.10+ is installed and available.

## Clone the git repository

First of all clone the repository by running:

``` powershell
git clone https://github.com/freqtrade/freqtrade.git
```

Now, choose your installation method, either automatically via script (recommended) or manually following the corresponding instructions.

## Install freqtrade automatically

### Run the installation script

The script will ask you a few questions to determine which parts should be installed.

```powershell
Set-ExecutionPolicy -ExecutionPolicy Bypass
cd freqtrade
. .\setup.ps1
```

## Install freqtrade manually

!!! Note "64bit Python version"
    Please make sure to use 64bit Windows and 64bit Python to avoid problems with backtesting or hyperopt due to the memory constraints 32bit applications have under Windows.
    32bit python versions are no longer supported under Windows.

!!! Hint
    Using the [Anaconda Distribution](https://www.anaconda.com/distribution/) under Windows can greatly help with installation problems. Check out the [Anaconda installation section](installation.md#installation-with-conda) in the documentation for more information.

### Install ta-lib

Install ta-lib according to the [ta-lib documentation](https://github.com/TA-Lib/ta-lib-python#windows).

As compiling from source on windows has heavy dependencies (requires a partial visual studio installation), Freqtrade provides these dependencies (in the binary wheel format) for the latest 3 Python versions (3.10, 3.11 and 3.12) and for 64bit Windows.
These Wheels are also used by CI running on windows, and are therefore tested together with freqtrade.

Other versions must be downloaded from the above link.

``` powershell
cd \path\freqtrade
python -m venv .venv
.venv\Scripts\activate.ps1
# optionally install ta-lib from wheel
# Eventually adjust the below filename to match the downloaded wheel
pip install --find-links build_helpers\ TA-Lib -U
pip install -r requirements.txt
pip install -e .
freqtrade
```

!!! Note "Use Powershell"
    The above installation script assumes you're using powershell on a 64bit windows.
    Commands for the legacy CMD windows console may differ.

### Error during installation on Windows

``` bash
error: Microsoft Visual C++ 14.0 is required. Get it with "Microsoft Visual C++ Build Tools": http://landinghub.visualstudio.com/visual-cpp-build-tools
```

Unfortunately, many packages requiring compilation don't provide a pre-built wheel. It is therefore mandatory to have a C/C++ compiler installed and available for your python environment to use.

You can download the Visual C++ build tools from [here](https://visualstudio.microsoft.com/visual-cpp-build-tools/) and install "Desktop development with C++" in it's default configuration. Unfortunately, this is a heavy download / dependency so you might want to consider WSL2 or [docker compose](docker_quickstart.md) first.

![Windows installation](assets/windows_install.png)

---

---

