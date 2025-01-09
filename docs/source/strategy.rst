Strategy Class
==============

.. contents:: Table of Contents
   :local:

Overview
--------
The Strategy class provides three core methods:

* ``train_all``
* ``generate_live_predictions``
* ``back_test``

Class Reference
---------------

.. autoclass:: src.utils.BaseStrategy
    :undoc-members:
    :members:

Example Strategy Implementation
-------------------------------

Below is an example of a simple linear regression strategy using features from the ``log_ret_ohlc`` feature view.

* ``train_all``: saves models in: ``models/example_strategy/``
* ``back_test``: saves models in: ``backtests/example_strategy/`` and saves
* Predictions are saved in BigQuery with format: ``{strategy_name}_{backtest_name}``

* ``train_all`` saves trained models in storage bucket dir ``models/example_strategy/``
* ``back_test`` saves trained models in storage bucket dir ``backtests/example_strategy/``
* ``back_test`` saves predictions in bigquery backtest table in a column named ``example_strategy_example_backtest`` which follows the format ``{strategy_name}_{backtest_name}``
* ``generate_live_predictions`` saves predictions in bigquery predictions table

.. code-block:: python

    import io
    import joblib
    import pandas as pd
    import numpy as np
    import datetime
    import ccxt

    from google.cloud import storage

    from sklearn.linear_model import LinearRegression

    from src.config import FEAST_STORE, PREDICTIONS_TABLE_REF, BACKTEST_TABLE_REF, BUCKET_ID
    from src.utils import BaseStrategy, load_to_bq_table, fetch_from_feature_store


    class dummy_strategy(BaseStrategy):
        def __init__(self, strategy_name):
            self.strategy_name = strategy_name
            self.df = None
            self.features = ['log_ret_open','log_ret_high','log_ret_low','log_ret_close']

        def train_all(self):

            #fetch all features from feature view name log_ret_ohlc
            feature_view = FEAST_STORE.get_feature_view("log_ret_ohlc")
            feature_references = [f"{feature_view.name}:{feature.name}" for feature in feature_view.features]

            #get ccxt exchange name handle kucoin -> KuCoin
            exchange_ccxt_name = getattr(ccxt, 'kucoin')().__str__()
            self.df = fetch_from_feature_store(from_datetime='2025-01-01 00:00:00',
                                               to_datetime=None,
                                               feature_refs=feature_references,
                                               entities={"exchange":exchange_ccxt_name}
                                               )

            self.build_target()

            self.df.dropna(inplace=True)

            model = LinearRegression()
            model.fit(self.df[self.features], self.df['target'])

            self.save_model(model, model_dir=f'models/{self.strategy_name}')

        def generate_live_preds(self):

            feature_view = FEAST_STORE.get_feature_view("log_ret_ohlc")
            feature_references = [f"{feature_view.name}:{feature.name}" for feature in feature_view.features]

            exchange_ccxt_name = getattr(ccxt, 'kucoin')().__str__()

            #get previous day date_time in string format
            from_datetime = pd.to_datetime(datetime.datetime.now().date()-datetime.timedelta(days=1), utc=True).__str__()
            self.df = fetch_from_feature_store(from_datetime=from_datetime,
                                               to_datetime=None,
                                               feature_refs=feature_references,
                                               entities={"exchange":exchange_ccxt_name}
                                               )

            model = self.load_latest_model(f'models/{self.strategy_name}/')
            self.df[self.strategy_name] = model.predict(self.df[self.features].fillna(0))

            errors = load_to_bq_table(df=self.df[['event_timestamp', 'exchange', 'symbol', self.strategy_name]],
                     table_ref=PREDICTIONS_TABLE_REF,
                     keys=['event_timestamp', 'exchange', 'symbol'])

            return errors

        def back_test(self,
                      backtest_name,
                      backtest_from_datetime,
                      backtest_to_datetime,
                      train_every_n_tuesday=4,
                      lookback_warmup_days=0,
                      lookback_days=None):

            from_datetime = pd.to_datetime(backtest_from_datetime, utc=True)
            to_datetime = pd.to_datetime(backtest_to_datetime, utc=True)

            #train_every_n_tuesday = 4, train every 4th tuesday ~ (every month)
            #self.backtest_dates: timeseries walkforward backtest training dates
            self.backtest_dates = list(
                pd.date_range(start=from_datetime, end=to_datetime, freq=f'{train_every_n_tuesday}W-TUE', tz='UTC'))

            #append backtest end date if not included
            if self.backtest_dates[-1] < to_datetime:
                self.backtest_dates.append(to_datetime)

            #backtest name, unique identifier for backtest models dir and backtest results bigquery column
            self.backtest_id = f"{self.strategy_name}_{backtest_name}"
            self.lookback_warmup_days = lookback_warmup_days

            feature_view = FEAST_STORE.get_feature_view("log_ret_ohlc")
            feature_references = [f"{feature_view.name}:{feature.name}" for feature in feature_view.features]

            exchange_ccxt_name = getattr(ccxt, 'kucoin')().__str__()
            self.df = fetch_from_feature_store(from_datetime=from_datetime,
                                               to_datetime=self.backtest_dates[-1],
                                               feature_refs=feature_references,
                                               entities={"exchange":exchange_ccxt_name}
                                               )

            self.build_target()

            self.backtest_predictions = pd.DataFrame([])

            #_generate_train_test: generator for train and test data,
            for counter, (date, train, test) in enumerate(self._generate_train_test()):
                model = LinearRegression()
                train.dropna(inplace=True)
                test.fillna(0,inplace=True)
                model.fit(train[self.features], train['target'])
                self.save_model(model, model_dir=f'backtests/{self.backtest_id}')

                model = self.load_latest_model(f'backtests/{self.backtest_id}/')

                pred = model.predict(test[self.features])

                test[self.backtest_id] = pred

                #in case warm_up_days > 0, we need to ensure test doesnt include warm up days
                test = test[(test.event_timestamp >= date) & (test.event_timestamp < self.backtest_dates[counter + 1])]
                self.backtest_predictions = pd.concat([self.backtest_predictions, test], axis=0)


            load_to_bq_table(df=self.backtest_predictions[['event_timestamp','exchange', 'symbol', self.backtest_id]],
                             table_ref=BACKTEST_TABLE_REF,
                             keys=['event_timestamp', 'exchange', 'symbol'])



        def build_target(self):
            self.df['target'] = self.df.groupby('symbol').log_ret_close.transform(lambda x: x.shift(-1))
            self.df.dropna(subset=['target'], inplace=True)

        def save_model(self, model, model_dir):
            storage_client = storage.Client()
            bucket = storage_client.bucket(BUCKET_ID)
            timestamp = datetime.datetime.now().strftime('%Y%m%d_%H%M%S')
            blob = bucket.blob(f'{model_dir}/model_{timestamp}')
            buffer = io.BytesIO()
            joblib.dump(model, buffer)
            buffer.seek(0)
            blob.upload_from_file(buffer)

        def load_latest_model(self, model_dir):

            storage_client = storage.Client()
            bucket = storage_client.bucket(BUCKET_ID)
            blobs = list(bucket.list_blobs(prefix=f'{model_dir}'))

            #get latest created blob
            blob = sorted(blobs, key=lambda blob: blob.time_created)[-1]

            buffer = io.BytesIO()
            blob.download_to_file(buffer)
            buffer.seek(0)
            model = joblib.load(buffer)

            return model

        def _generate_train_test(self):
            for i, date in enumerate(self.backtest_dates[:-1]):
                train_end = date - datetime.timedelta(days=1)

                test_start = date - datetime.timedelta(days=self.lookback_warmup_days)
                yield (date,
                       self.df[self.df.event_timestamp <= train_end],
                       self.df[(self.df.event_timestamp >= test_start) & (
                                   self.df.event_timestamp < self.backtest_dates[i + 1])])

    if __name__ == '__main__':
        strat = dummy_strategy(strategy_name='example_strategy')

        strat.back_test(backtest_name='example_backtest',
                        backtest_from_datetime='2025-01-05 00:00:00',
                        backtest_to_datetime='2025-01-09 00:00:00',
                        train_every_n_tuesday=1,
                        lookback_warmup_days=0)


