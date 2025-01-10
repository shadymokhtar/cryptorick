Datasets and Features
=====================

.. contents:: Table of Contents
   :local:

Overview
--------
Data (raw OHLCV data, features, backtests, predictions) is stored in BigQuery tables.
A **Feast feature store** is deployed for data retrieval and feature management.

.. note::
    It is recommended to retrieve data from Feast rather than directly querying BigQuery
    when developing features or strategy functions.

Feature definitions are managed in: :file:`src/feast_repo/features_definitions.yaml`

Common Table Structure
----------------------

- **event_timestamp** (mandatory)
    Timestamp column following the format 'YYYY-MM-DDTHH:MM:SS±hh:mm'

    Example: '2017-01-01 00:00:00' or '2017-01-01 00:00:00+00:00'

- **Entity Columns**
    Columns that uniquely identify rows within a specific ``event_timestamp``.
    Represent primary dimensions like ``exchange`` and ``symbol``.
    Used for joining, merging, and upserting new data.

- **Clustering Fields**
    Used for query optimization and efficient data retrieval.
    Currently set as the union of ``event_timestamp`` and entity columns.

- **Table References**
    Format: {PROJECT_ID}.{DATASET_ID}.{table_name}

Utility Functions
-----------------

.. autosummary::
   :toctree: generated

    src.utils.create_bq_table_if_not_exist
    src.utils.load_to_bq_table
    src.utils.fetch_from_feature_store
    src.utils.get_max_timestamp_from_bq

.. autofunction:: src.utils.create_bq_table_if_not_exist
.. autofunction:: src.utils.load_to_bq_table
.. autofunction:: src.utils.fetch_from_feature_store
.. autofunction:: src.utils.get_max_timestamp_from_bq

Example: Creating and Deploying Features
----------------------------------------

This example demonstrates:

1. Create new feature module:

    1. Get KuCoin raw price data from feature store
    2. Calculate 1 day log returns
    3. load new records / features to database

2. Add feature view definition

3. Deploy feature view


1. Create New Feature Module:
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: python

    import datetime

    import numpy as np

    from google.cloud import bigquery

    import ccxt

    from src.config import BQ_CLIENT, PROJECT_ID, DATASET_ID, PROD_DATASET_ID
    from src.utils import get_max_timestamp_from_bq, fetch_from_feature_store, load_to_bq_table

    def calculate_log_return_feature_set(from_datetime, exchange):
        #feature references follow the format feature_view_name:feature_name
        feature_refs = ['raw_ohlcv:open', 'raw_ohlcv:high', 'raw_ohlcv:low', 'raw_ohlcv:close']
        entities={"exchange": exchange}
        df = fetch_from_feature_store(
            from_datetime=from_datetime,
            to_datetime=None,
            feature_refs=feature_refs,
            entities=entities
        )

        #create 1 day log return price columns
        log_ret_columns = ['log_ret_open', 'log_ret_high', 'log_ret_low', 'log_ret_close']
        price_columns = ['open', 'high', 'low', 'close']
        df[log_ret_columns] = df.groupby('symbol')[price_columns].transform(lambda x: np.log(x).diff(1))
        df.drop(price_columns, axis=1, inplace=True)

        load_to_bq_table(df=df, table_ref=NEW_TABLE_REF, keys=['event_timestamp', 'exchange', 'symbol'])

    if __name__ == '__main__':
        NEW_TABLE_REF = f'{PROJECT_ID}.{DATASET_ID}.example_log_ret_features'

        #use below to get exchange name reference by ccxt
        exchange = getattr(ccxt, 'kucoin')().__str__() #kucoin -> KuCoin, kucoinfutures -> KuCoin Futures

        #Production ohlcv_daily table reference
        OHLCV_TABLE_REF = f'{PROJECT_ID}.{PROD_DATASET_ID}.ohlcv_daily'
        latest_timestamp = get_max_timestamp_from_bq(OHLCV_TABLE_REF, entities={"exchange":exchange})
        #>>datetime.datetime(2025, 1, 7, 0, 0, tzinfo=datetime.timezone.utc)
        from_datetime = (latest_timestamp - datetime.timedelta(days=1)).__str__()
        #>>'2025-01-07 00:00:00+00:00'
        calculate_log_return_feature_set('2025-01-05 00:00:00', 'KuCoin')



2. Add Feature View Definition
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Add new feature view to :file:`src/feast_repo/features_definitions.yaml`:

.. code-block:: yaml

    feature_views:
      log_ret_ohlc:
        description: description
        entities:
          - symbol
          - exchange
        ttl:
          days: 1
        source:
          type: bigquery
          table: example_log_ret_features
          production: false
        features:
          - name: log_ret_open
            dtype: Float64
          - name: log_ret_high
            dtype: Float64
          - name: log_ret_low
            dtype: Float64
          - name: log_ret_close
            dtype: Float64

3. Deploy Features
~~~~~~~~~~~~~~~~~~

Run the following command:

.. code-block:: bash

  python3 -m src.feast_repo.create_store



Now the deployed features and feature view can be retrieved from feast feature store

.. code-block:: python

    feature_refs = [
        'log_ret_ohlc:log_ret_open',
        'log_ret_ohlc:log_ret_high',
        'log_ret_ohlc:log_ret_low',
        'log_ret_ohlc:log_ret_close'
    ]

    # Example usage with fetch_from_feature_store
    df = fetch_from_feature_store(
        from_datetime='2025-01-05 00:00:00',
        feature_refs=feature_refs,
        entities={"exchange": "KuCoin"}
    )