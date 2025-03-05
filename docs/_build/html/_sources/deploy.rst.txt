Deploy Jobs
===========

.. contents:: Table of Contents
   :local:

Overview
--------

Two core containers are available for researchers to run jobs on the cloud:
* Features jobs
* Strategies jobs

Deployment Steps
----------------

1. Add your module to the appropriate directory:

    - Features modules: ``src/pipelines/features``
    - Strategy modules: ``src/pipelines/strategies``

2. Update ``requirements.txt`` and ``Dockerfile`` in the directory if needed

3. Run cloud deployment command while pointing to the appropriate docker file

    .. code-block:: bash

        python3 -m src.deploy_cloud_run_job \
        --job-name [JOB-NAME] \
        --docker-path [DOCKER-FILE-PATH]

    example

    .. code-block:: bash

        python3 -m src.deploy_cloud_run_job \
        --job-name example-job \
        --docker-path src/pipelines/features/Dockerfile

4. The container should be built and the job is available in GCP console: https://console.cloud.google.com/run/jobs

5. edit job args in GCP and execute

    .. image:: _static/job_args.png


