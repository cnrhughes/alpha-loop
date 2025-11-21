# **alpha-loop**

An end-to-end MLOps trading system designed to retrain, validate, and deploy machine-learning models on a fixed schedule. The project focuses on engineering discipline rather than market prediction, using financial data as a convenient stress-test for automated ML pipelines.



## **Contents**
- [Project Structure](#project-structure)
- [Architecture Overview](#architecture-overview)
- [Training Factory — Loop A](#training-factory--loop-a)
- [Trading Bot — Loop B](#trading-bot--loop-b)
- [Key Components](#key-components)
- [Backtesting as a Gatekeeper](#backtesting-as-a-gatekeeper)
- [Roadmap](#roadmap)



## **Project Structure**
```text
alpha-loop
├── LICENSE
├── README.md
├── alpha_loop
│   ├── __init__.py
│   ├── backtest
│   │   ├── run_backtest.py
│   │   └── strategy.py
│   ├── bot
│   │   ├── alpaca_client.py
│   │   ├── features_live.py
│   │   └── inference.py
│   ├── data
│   │   ├── fetch_yfinance.py
│   │   └── loaders.py
│   ├── features
│   │   ├── indicators.py
│   │   └── pipeline.py
│   ├── labels
│   │   └── triple_barrier.py
│   ├── models
│   │   ├── optuna_search.py
│   │   ├── train.py
│   │   └── walk_forward.py
│   ├── registry
│   │   ├── mlflow_utils.py
│   │   └── promotion.py
│   └── utils
│       ├── logging.py
│       └── time_windows.py
├── configs
│   ├── backtest.yaml
│   ├── bot.yaml
│   ├── data.yaml
│   └── training.yaml
├── notebooks
│   ├── 01_exploration.ipynb
│   ├── 02_label_sanity.ipynb
│   └── 03_backtest_sanity.ipynb
├── pyproject.toml
├── scripts
│   ├── run_trading_bot.py
│   └── run_training_factory.py
├── setup.sh
└── tests
    ├── test_backtest_strategy.py
    ├── test_live_features.py
    ├── test_triple_barrier.py
    └── test_walk_forward.py
```



## **Architecture Overview**

The system is organised around two coordinated loops:

- A **weekly training factory** that ingests data, labels it, trains models, runs Optuna optimisation, backtests the best candidate, and promotes the result through an MLflow registry.
- An **hourly trading bot** that loads the current production model and executes a simple long/flat strategy in Alpaca’s paper environment.

This design demonstrates a complete ML lifecycle in a noisy and adversarial setting.



## **Training Factory — Loop A**

Runs weekly and handles all offline model work:

1. **Ingestion:** Download hourly price data via `yfinance`.  
2. **Labelling:** Apply the Triple Barrier Method for directional labels.  
3. **Feature Engineering:** Generate technical indicators using `pandas-ta`.  
4. **Training:** Train an XGBoost classifier using walk-forward validation.  
5. **Hyperparameter Search:** Use `optuna` to run Bayesian optimisation over model parameters and barrier configurations.  
6. **Validation:** Backtest the best model on withheld data with `Backtesting.py` to measure realistic returns, Sharpe ratio, and drawdowns.  
7. **Registry:** Log everything in MLflow and promote the model to “Production” only if it exceeds the configured performance threshold.



## **Trading Bot — Loop B**

Runs every hour during market hours:

1. **Wake:** Triggered on a schedule.  
2. **Inference:** Pull the latest Production model from MLflow and build real-time features from the most recent market data.  
3. **Execution:** Submit long/flat orders to Alpaca Paper Trading when the model’s signal exceeds the chosen confidence threshold.



## **Key Components**

| Component | Tool | Purpose |
|----------|------|----------|
| **Data** | `yfinance` | Hourly historical price data |
| **Indicators** | `pandas-ta` | Technical feature generation |
| **Model** | `xgboost` | Tabular classification |
| **Tuning** | `optuna` | Bayesian hyperparameter search |
| **Simulation** | `Backtesting.py` | Realistic trading simulation with costs |
| **Ops** | `mlflow` | Experiment tracking and model registry |
| **Broker** | `alpaca-trade-api` | Paper-trading execution |



## **Backtesting as a Gatekeeper**

Traditional classification metrics do not reliably reflect trading performance. A model may predict direction correctly yet produce poor equity curves.

Each candidate model is therefore evaluated through a backtest:

- **Input:** Historical prices and the model’s predicted signals  
- **Logic:**  
  - Enter long when the signal is 1 (with defined stop-loss and take-profit).  
  - Exit positions when the signal returns to 0.  
- **Outputs:** Sharpe ratio, returns, drawdowns  
- **Decision rule:** Only models that pass the minimum performance threshold (e.g. Sharpe > 1.0) are promoted to Production.

All experiments, parameters, metrics, and artifacts are logged in MLflow.



## **Roadmap**

### **Phase 1 — Foundation**
Build the ingestion layer, triple-barrier labelling, and feature pipelines.

### **Phase 2 — The Lab**
Implement walk-forward training, model evaluation, and backtesting integration.

### **Phase 3 — The Factory**
Automate experiments with Optuna and wrap training into a fully orchestrated weekly workflow backed by MLflow.

### **Phase 4 — The Bot**
Develop the hourly inference agent and Alpaca execution module, linking Production models to live paper-trading decisions.
