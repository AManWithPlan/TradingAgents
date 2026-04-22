
Got it — you want a concrete implementation plan for Czech energy price prediction plus a backtesting framework that fits this codebase.
 Identify reliable historical Czech energy data sources (and access constraints)
 Define target(s): what exactly to predict (day-ahead hourly, base/peak, spread)
 Extend current dataflow/tool routing architecture for energy data
 Add a reproducible, leakage-safe walk-forward backtesting engine
 Define evaluation metrics for both forecast quality and trading performance
Where to get historical Czech energy data
Best practical sources (in priority order):
ENTSO-E Transparency Platform (official EU electricity market data; includes Czech bidding zone prices, load, generation, cross-border flows)
OTE, a.s. (Czech market operator) for Czech market-specific day-ahead/intraday publications and market reports
CEPS (Czech TSO) for system load, generation mix, balancing/system conditions
PXE/EEX for Czech power futures (often useful features, frequently commercial/licensed)
CHMI (Czech Hydrometeorological Institute) for weather drivers (temperature, wind, solar proxies)
Optional enrichments: gas prices (TTF), EUA carbon, FX (EUR/CZK) from public market datasets
If you want mostly open/free + robust, start with ENTSO-E + weather + CEPS, then add PXE/EEX later if licensing allows.
How this fits your current architecture
Your repo already has a clean extension point:
Vendor routing lives in tradingagents/dataflows/interface.py
Config routing lives in tradingagents/default_config.py (data_vendors, tool_vendors)
Agent tools wrap dataflow calls in tradingagents/agents/utils/*.py
Graph orchestration is in tradingagents/graph/trading_graph.py and tradingagents/graph/setup.py
So the plan is to add an energy data category parallel to stock/news/fundamentals, not hack existing stock code.
Proposed implementation plan
Phase 1: Data layer (first)
Add tradingagents/dataflows/entsoe_cz.py and (optionally) tradingagents/dataflows/ote_cz.py
Normalize to one schema:
timestamp_utc, timestamp_cet, zone, product, price_eur_mwh, volume_mwh
Extend TOOLS_CATEGORIES + VENDOR_METHODS in tradingagents/dataflows/interface.py with methods like:
get_energy_spot_prices
get_energy_load_forecast
get_energy_generation_mix
get_weather_features
Add vendor config keys in tradingagents/default_config.py (e.g., energy_data)
Phase 2: Tooling + analyst integration
Add wrappers in tradingagents/agents/utils/energy_data_tools.py with @tool
Either:
Reuse market analyst with updated prompt context for power markets, or
Add new energy_market_analyst.py and include in graph selection flow in tradingagents/graph/setup.py
Keep current decision extraction (BUY/SELL/etc. in tradingagents/graph/signal_processing.py) only if you’ll trade products; otherwise add a numeric forecast output path
Phase 3: Forecast model layer (non-LLM baseline first)
Add tradingagents/models/energy_forecast/:
Baselines: seasonal naive, rolling mean
ML: LightGBM/XGBoost (lags, rolling stats, calendar, weather, load)
Predict horizon(s): e.g. t+1h, t+24h, day-ahead hourly profile
Keep LLM agents for reasoning/reporting, but rely on numeric model for core forecast
Backtesting framework design (leakage-safe)
Create new package tradingagents/backtesting/ with:
dataset.py: time-aligned feature/label assembly
split.py: expanding-window walk-forward splits
engine.py: run train -> predict -> score per fold
strategy.py: convert forecast to actions (threshold/spread strategy)
metrics.py: MAE/RMSE/sMAPE + PnL/Sharpe/MaxDD/hit rate
report.py: fold summary + aggregate CSV/plots
Use the same anti-lookahead principle already present in tradingagents/dataflows/stockstats_utils.py (filter future rows before inference).
Recommended first backtest spec (simple and strong)
Target: Czech day-ahead hourly price (CZ zone)
Horizon: next day hourly prices
Features: lags (1h/24h/168h), rolling volatility, hour/day/week, load forecast, temperature, wind/solar proxy, neighboring zone spread
Split: expanding window (e.g., train 2021-2023, validate monthly over 2024-2025)
Metrics:
Forecast: MAE, RMSE, sMAPE by hour and season
Strategy: net PnL after transaction costs, Sharpe, drawdown, turnover
Key design decisions to lock before coding
Exact product: spot hourly vs day-ahead block vs futures
Currency/timezone conventions (EUR/MWh, CET/CEST handling)
Trading rule from forecasts (long/short threshold, risk cap)
Data licensing constraints (especially PXE/EEX history)
If you want, next I can draft a file-by-file scaffold (exact modules/classes/functions) matching your repo conventions so implementation is mostly copy/paste.