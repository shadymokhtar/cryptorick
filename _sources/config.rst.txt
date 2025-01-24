Config
======

.. contents:: Table of Contents
   :local:

Overview
--------
The config module provides essential variables and objects for interacting with data and infrastructure.
These can be imported from ``src.config`` as shown below:

.. code-block:: python

    from src.config import PROD_DATASET_ID
    print(PROD_DATASET_ID)
    # Output: bigquery_data

Variables
---------

PROJECT_ID
    *(str)* GCP project ID.

PROD_DATASET_ID
    *(str)* Production dataset ID.

DATASET_ID
    *(str)* Researcher's BigQuery dataset ID.

BUCKET_ID
    *(str)* Researcher's storage bucket ID.

PREDICTIONS_TABLE_REF
    *(str)* Reference to the predictions table in research dataset.

BACKTEST_TABLE_REF
    *(str)* Reference to the backtest table in research dataset.

SERVICE_ACCOUNT
    *(str)* GCP service account.

Objects
-------

BQ_CLIENT
    *(google.cloud.bigquery.client.Client)* BigQuery client object connected to the project.

FEAST_STORE
    *(feast.feature_store.FeatureStore)* Feast feature store client.

Usage Example
-------------
.. code-block:: python

    from src.config import BQ_CLIENT, DATASET_ID, PROJECT_ID

    # Query example
    query = f"""
        SELECT *
        FROM `{PROJECT_ID}.{DATASET_ID}.ohlcv_daily`
        LIMIT 5
    """
    df = BQ_CLIENT.query(query).to_dataframe()