# Data Sync Architecture - OnPrem to GCP

As part of the migration to GCP, despite a few pipeline may still be running in the On Premise cluster, consumers may have already migrated to GCP, then, data retrieved by Run Replicator from Teradata to On Premise HDFS needs to be also retrieved to Google Cloud Storage for the customer to be able to fully migrate to GCP. 

1. On Premise cluster will push a message to a Pub/Sub topic to notify a Run Replicator process has been executed.

2. A Cloud Function will be triggered by the message published to the Cloud Pub/Sub Topic.

3. The Cloud Function will write into BigQuery to register the execution of the On Premise Run Replicator and will call Cloud Composer to trigger a DAG run which will execute Run Replicator in GCP. The Cloud Function will also generate an entry in the table to register the future execution of Run Replicator in GCP.

4. A Dataproc cluster will execute the Run Replicator process and copy the data from Teradata to Google Cloud Storage.

5. The last task of the Cloud Composer DAG will register in the Cloud SQL the GCP Run Replicator execution and update the entry generated in point 3.

## Google Cloud Function Event and Trigger Type

Events are things that happen within the cloud environment that you might want / need to take action on. In this case the Google Cloud Function is going to be triggered by a Pub/Sub message publish event.

## Google Cloud Function Code

This URL contains the Cloud Function Python code.

## Format of the message published to the Pub/Sub topic

Example message published to the Pub/Sub topic:
```bash
--
--
--
--
--
```

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
query = f'gsutil cp gs:/{ETL_CODE_BASE}/cerebro/run_replicator.sh . && chmod 777 run_replicator.sh && ./run_replicator.sh  dev1_groupondw ref_attr_class_dev',
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


