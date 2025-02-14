.. cryptorick documentation master file, created by
   sphinx-quickstart on Mon Jan  6 11:54:00 2025.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

Overview
========

cryptorick is a Python framework for developing, researching, and running quantitative crypto trading strategies.
It simplifies the process of building, integrating, and orchestrating research and trading components.
This documentation is designed to help in-house researchers onboard, enabling them to explore the infrastructure, develop features, test, and research trading strategies.

Infrastructure
--------------

.. image:: _static/infrastructure.png



Core Functions
--------------


* **Data Collection**
   Dir: :file:`src/pipelines/fetch_ccxt_data_bq/`

   .. code-block:: bash

      python3 -m src.pipelines.fetch_ccxt_data_bq.fetch_ccxt_data_bq \
      --trading_exchange_data_name kucoin \
      --api_name kucoin \
      --asset_type spot \
      --from_datetime "2017-01-01 00:00:00"

* **Feature Engineering**
   Dir: :file:`src/pipelines/features/`

   .. code-block:: bash

      python3 -m src.pipelines.features.create_features \
      --trading_exchange_data_name kucoin

* **Strategy Operations**
   Dir: :file:`src/pipelines/strategies/`

   .. code-block:: bash

      python3 -m src.pipelines.strategies.template_strategy.template_lgbm \
      --infer \
      --trading_exchange_data_name kucoin \
      --trading_exchange_name kucoinfutures \
      --api_name kucoin \
      --strategy_name template_lgbm_strategy_binance

* **Rebalancing**
   Dir: :file:`src/pipelines/rebalance/`

   .. code-block:: bash

      python3 -m src.pipelines.rebalance.rebalance_daily \
      --execute \
      --trading_exchange_name kucoinfutures \
      --trading_exchange_data_name kucoin \
      --api_name kucoin_prod
      --strategies "strategy_1 strategy_2 strategy_3"


Quick Start
-----------

1. git clone https://github.com/shadymokhtar/cryptohouse.git

2. Edit config file if needed: :file:`src/config/__init__.py`

3. Set up feast feature store and deploy feature views:

   feature definitions: :file:`src/feast_repo/features_definitions.yaml`

   .. code-block:: bash

      python3 -m src.feast_repo.create_store


.. toctree::
   :maxdepth: 2
   :caption: Contents:

   self
   config
   dataset
   strategy
   deploy