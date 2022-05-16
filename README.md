# Data Sync Architecture - OnPrem to GCP

As part of the migration to GCP, despite a few pipeline may still be running in the On Premise cluster, consumers may have already migrated to GCP, then, data retrieved by Run Replicator from Teradata to On Premise HDFS needs to be also retrieved to Google Cloud Storage for the customer to be able to fully migrate to GCP. 

1. On Premise cluster will push a message to a Pub/Sub topic to notify a Run Replicator process has been executed.

2. A Cloud Function will be triggered by the message published to the Cloud Pub/Sub Topic.

3. The Cloud Function will write into BigQuery to register the execution of the On Premise Run Replicator and will call Cloud Composer to trigger a DAG run which will execute Run Replicator in GCP. The Cloud Function will also generate an entry in the table to register the future execution of Run Replicator in GCP.

4. A Dataproc cluster will execute the Run Replicator process and copy the data from Teradata to Google Cloud Storage.

5. The last task of the Cloud Composer DAG will register in the Cloud SQL the GCP Run Replicator execution and update the entry generated in point 3.

## Format of the message published to the Pub/Sub topic

Example message published to the Pub/Sub topic:
```bash
${RUN_SCRIPT} "sh ${ETL_CODE_BASE}/cerebro/run_replicator.sh groupon_production im_brand_list"
```

## Google Cloud Function Deployment

Execute the following command to deploy the cloud function that will be in charge of triggering the Cloud Composer DAG that will copy the data from Teradata to Google Cloud Storage.

```bash
gcloud functions deploy my-python-function --entry-point helloworld --runtime python37 --trigger-http --allow-unauthenticated --serviceaccount=aaa@google.com
```

## Google Cloud Function Code

The following URL contains the Cloud Function Python code.

## BigQuery Table Entries & Schema
<br />

### Table Entries:
There will be two table entries generated. One for the OnPrem execution and another for the GCP execution. Cloud Composer DAG execution will update the GCP execution entry once the process is completed.  
<br />

### Table Schema:
id: integer   
command: string   
timestamp: datetime   
source: string   

## Dataproc Cluster Ephemeral Creation, Image and linked Hive Metastore

The Dataproc Cluster is ephemeraly created and destroyed by the Cloud Composer DAG, which code you can find in the following URL. As part of the Cluster creation command, it's specified the Hive Metastore that will be attached to the Ephemeral Cluster.

## Dataproc Cluster Command




