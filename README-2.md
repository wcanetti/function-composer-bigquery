# Data Sync Architecture - GCP to OnPrem
1. GCP Run Replicator running in a Dataproc Cluster will copy the data from Teradata to Google Cloud Storage.
2. An entry for the GCP Run Replicator is going to be created in a BigQuery table for logging and monitoring purposes.
3. Cloud Composer will SSH into the CEREBRO cluster and trigger Run Replicator execution to perform the datasets’ copies from Teradata to CEREBRO HDFS.
4. An entry for the CEREBRO Run Replicator is going to be created in a Cloud SQL table for logging and monitoring purposes.

## BigQuery Table Entries & Schema

### Table Entries:
There will be two table entries generated. One for the OnPrem execution and another for the GCP execution. Cloud Composer DAG execution will update the GCP execution entry once the process is completed.  

### Table Schema:
Below is the BigQuery table schema that is going to be used to track the different Run Replicator executions.

```json
[
  {
    "name": "dbName",
    "type": "STRING"
  },
  {
    "name": "tableName",
    "type": "STRING"
  },
  {
    "name": "td_timestamp",
    "type": "TIMESTAMP",
    "mode": "NULLABLE"
  },
  {
    "name": "last_modified_at",
    "type": "TIMESTAMP",
    "mode": "NULLABLE"
  },
  {
    "name": "environment",
    "type": "STRING",
    "mode": "NULLABLE"
  },
  {
    "name": "status",
    "type": "STRING",
    "mode": "NULLABLE"
  },
  {
    "name": "dataproc_job_id",
    "type": "STRING",
    "mode": "NULLABLE"
  }
]
```

## Cloud Composer DAG and the integration with Dataproc through DataprocSubmitPigJob Operator

This URL contains the Cloud Composer DAG code.  

To integrate with Cloud Dataproc the DataprocSubmitPigJob Airflow Operator is used to submit shell scripts to the Dataproc Cluster in order to execute the Run Replicator application.

```python
DataprocSubmitPigJobOperator(
task_id = re.sub(r'\W+', '', 'CEREBRO_COPY_ACTIVE_DEALS_${varRUNDATE}'),
job_name= re.sub(r'\W+', '', 'CEREBRO_COPY_ACTIVE_DEALS_${varRUNDATE}'),
query = f'sh export HOME=/root && export ODBCINI=/root/.odbc.ini && export ZOMBIERC=/root/.zrc2 && gsutil cp gs:/{ETL_CODE_BASE}/cerebro/run_replicator.sh . && chmod 777 run_replicator.sh && ./run_replicator.sh  dev1_groupondw ref_attr_class_dev',
retries=0,
retry_delay=60,
priority_weight=21-10,
cluster_name = 'zr-1652997564-cerebro-copy-active-deals-varrundate',
region='us-central1',
)
```

## Dataproc Cluster Ephemeral Creation, Image and linked Hive Metastore

The Dataproc Cluster is ephemeraly created and destroyed by the Cloud Composer DAG, which code you can find in this URL. As part of the Cluster creation command, it's specified the Hive Metastore that will be attached to the Ephemeral Cluster.

### Dataproc Cluster Command

The Dataproc Cluster Command is embebbed in the Cloud Composer (Airflow) DAG. Find below the Dataproc Cluster command being executed:

```bash
gcloud dataproc clusters create run-replicator-airflow \  
--region=<region> \  
--no-address \  
--subnet=<subnet> \  
--service-account=<service-account> \  
--single-node \  
--tags 'allow-iap-ssh','dataproc-vm' \  
--dataproc-metastore=<dataproc-metastore> \  
--image=<dataproc-image> \  
--max-idle=12h \  
--scopes cloud-platform
```
