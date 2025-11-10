# Data Processing System on Kubernetes 

## Introduction

This project build an data processing system on kubernetes

This is my resume: [TDD CV]([https://www.google.com](https://www.topcv.vn/xem-cv/BlFSVwEEXVlRV1YAWwQFAwEAUgEBBghSBwEGBA1840) "My CV")

## Overall System Architecture

<div style="text-align: center;"> <img src="images/overall-architecture-data-processing.png" style="width: 1188px; height: auto;"></div>

# Table of Contents
[Overall System Architecture](#overall-system-architecture)
1.[Data-Processing](#data-processing)

   1.1 [Batch Processing](#batch-processing)  
   2.2 [Stream Processing](#stream-processing)  

## Project Structure
```txt
├── helm-charts               - Directory for Helm chart to deploy the application
├── app                       - Python script for the application
├── config                    - Config file for chart deloy
├── dockerfile                - Dockerfile for build custom image
├── dags                      - Directory for image files
├── Jenkinsfile               - Jenkins pipeline script to describe the CI/CD process
├── docker-compose.yaml       - Docker Compose configuration file
├── Dockerfile                - Dockerfile for building the image
├── requirements.txt          - Python requirements file
├── images                    - Directory for image files
└── README.md                 - This README file
```
# DATA PROCESSING

[Request edit host](#edit-host-on-your-computer-to-access-service-by-domain)

## Batch Processing

- Apply ingress

```bash
k create namespace storage &&
kubectl apply -f ./helm-charts/ingress 
```

### Source Systems

#### Postgresql
- Install Postgresql
```bash
helm upgrade --install postgresql ./helm-charts/postgresql -f ./helm-charts/postgresql/auth-values.yaml --namespace storage --create-namespace
```
### Storage:
#### MinIO
- Install MinIO
```bash
helm upgrade --install minio-operator ./helm-charts/minio-operator -n storage 

```
```bash
helm upgrade --install minio-tenant ./helm-charts/minio-tenant  -f ./helm-charts/minio-tenant/override-values.yaml -n storage
```
- Login minio.tsc.vn with user name: minio and password: minio123

<div style="text-align: center;"> <img src="images/minio-homepage.png" style="width: 888px; height: auto;"></div>

<div style="text-align: center;"> <img src="images/minio-bronze-data.png" style="width: 888px; height: auto;"></div>

#### Trino & Hive Metastore

- Create minio secret for Hive Metastore access

```bash
kubectl create secret generic minio-credentials \
  --from-file=access-key=config/s3/access-key.properties \
  --from-file=secret-key=config/s3/secret-key.properties \
  -n storage
```
- Create database for Hive Metastore
=> Access postgresql pod with password in /helm-chart/postgresql/auth-values.yaml (pg123)

```bash
kubectl exec -it postgresql-0 -n storage -- psql -U pgadmin -d postgres
```
- Run SQL Command to create Database
```sql
CREATE DATABASE hive;
CREATE DATABASE crm_db;
```
<div style="text-align: center;"> <img src="images/dbeaver-postgresql.png" style="width: 888px; height: auto;"></div>

- Install Hive Metastore & Trino
```bash
helm upgrade --install olap ./helm-charts/olap -n storage
```
- Access trino.tsc.vn with user name is admin
<div style="text-align: center;"> <img src="images/trino-homepage.png" style="width: 888px; height: auto;"></div>

#### Initialize data

- Forward port postgresql and minio to local
```bash
kubectl port-forward svc/minio-tenant-hl 9000:9000 -n storage
kubectl port-forward svc/postgresql 5432:5432 -n storage
```
- Init data:
```bash
cd ./helm-charts/postgresql/initdata && python copydata.py
```
<div style="text-align: center;"> <img src="images/init-data.png" style="width: 888px; height: auto;"></div>

### Pipeline Orchestration: 
#### Airflow on GKE
- Install Airflow
```bash
helm upgrade --install airflow ./helm-charts/airflow -f ./helm-charts/airflow/override-values.yaml --namespace orchestration --create-namespace
```
- Access airflow.tsc.vn with user name:admin and password: admin (in ./helm-charts/airflow/values.yaml > webserver/defaultUser/username + password)
<div style="text-align: center;"> <img src="images/airflow-homepage.png" style="width: 888px; height: auto;"></div>

#### Schedule Job Script

- Python file ".dags/airflow_dag.py" is sync with Airflow over gitSync (config in ./helm-charts/airflow/override-values.yaml)
<div style="text-align: center;"> <img src="images/airflow-dags.png" style="width: 888px; height: auto;"></div>

- Docker image dongtd6/airflow-job-scripts is use by airflow_dag.py for job script (batch-processing/Dockerfile)
<div style="text-align: center;"> <img src="images/bronze-job-py.png" style="width: 888px; height: auto;"></div>
<div style="text-align: center;"> <img src="images/silver-job-py.png" style="width: 888px; height: auto;"></div>
<div style="text-align: center;"> <img src="images/gold-job-py.png" style="width: 888px; height: auto;"></div>
- Access trino.tsc.vn (user name is admin) over Dbeaver
<div style="text-align: center;"> <img src="images/dbeaver-trino.png" style="width: 888px; height: auto;"></div>

## Stream Processing

- Create secret
```shell

kubectl create namespace infrastructure

kubectl create secret generic postgres-credentials \
  --from-file=config/postgres/postgres-credentials.properties \
  -n infrastructure &&

kubectl create secret generic minio-credentials \
  --from-file=access-key=config/s3/access-key.properties \
  --from-file=secret-key=config/s3/secret-key.properties \
  -n infrastructure &&

kubectl create secret generic minio-credentials \
  --from-file=access-key=config/s3/access-key.properties \
  --from-file=secret-key=config/s3/secret-key.properties \
  -n processor

kubectl create secret generic telegram-secrets \
  --from-literal=bot-token=<your-telegram-bot-token> \
  --from-literal=chat-id=<your-telegram-chat-id> \
  -n infrastructure
```

### Postgresql

- Postgresql Config

```SQL
CREATE PUBLICATION debezium FOR TABLE product_reviews;
ALTER ROLE pgadmin WITH REPLICATION;
ALTER SYSTEM SET wal_level = logical;
ALTER SYSTEM SET max_replication_slots = 4 ;
ALTER SYSTEM SET max_wal_senders = 4;
```
- Check & test config

```sql 
SHOW wal_level; -- must be 'logical'
SELECT pg_current_wal_lsn();
SELECT * FROM pg_publication;
SELECT slot_name, active, active_pid FROM pg_replication_slots;
SELECT * FROM pg_publication_tables WHERE pubname = 'debezium';
SELECT * FROM pg_stat_replication;

INSERT INTO product_reviews (review_id, review, created_at) VALUES ('test-123', 'Test review 123', CURRENT_TIMESTAMP);
INSERT INTO product_reviews (review_id, created_at, updated_at, product_id, user_id, review, source) VALUES ('13546f11-1070-1d1b-a080-d6b901062ff9',CURRENT_TIMESTAMP, CURRENT_TIMESTAMP,'PRD368','USR368','Sản phẩm dùng không được', 'ZALO');

```
<div style="text-align: center;"> <img src="images/postgresql-config.png" style="width: 888px; height: auto;"></div>

- Restart Postgresql
```bash
kubectl exec -it postgresql-0 -n storage -- pg_ctl restart
```

### Kafka

- Install Strimzi Kafka Operator in the `operators` namespace:
=> use custom image in ./helm-charts/strimzi-kafka-operator/values.yaml > kafkaConnect: image: dongtd6/kafka-connectors-customize:v1.1
=> image build by ./dockerfile/Dockerfile-kafka-conect

```bash
curl -sL https://github.com/operator-framework/operator-lifecycle-manager/releases/download/v0.20.0/install.sh | bash -s v0.20.0
kubectl apply -f ./helm-charts/strimzi-kafka-operator/strimzi-kafka-operator.yaml 
helm upgrade --install strimzi-kafka-operator ./helm-charts/strimzi-kafka-operator --namespace infrastructure
```
<div style="text-align: center;"> <img src="images/kafka-dashboard.png" style="width: 888px; height: auto;"></div>

- Create Debezium Kafka connector
```bash
kubectl apply -f ./helm-charts/strimzi-kafka-operator/postgres-connector.yaml
```

<div style="text-align: center;"> <img src="images/cdc-product-review-topic.png" style="width: 888px; height: auto;"></div>

### Flink

- Install Flink Operator
```bash
helm repo add flink-operator-repo https://downloads.apache.org/flink/flink-kubernetes-operator-1.12.1/
kubectl apply -f ../helm-charts/flink-kubernetes-operator/cert-manager.yaml
helm install flink-kubernetes-operator ./helm-charts/flink-kubernetes-operator -n infrastructure
```
- Run jobs
```bash
kubectl apply -f ./helm-charts/flink-kubernetes-operator/flink-sentiment-job.yaml
kubectl apply -f ./helm-charts/flink-kubernetes-operator/flink-telegram-job.yaml
```
<div style="text-align: center;"> <img src="images/message-queue-topic.png" style="width: 888px; height: auto;"></div>

- Message will send to Telegram after new review received
<div style="text-align: center;"> <img src="images/telegram-alert.png" style="width: 888px; height: auto;"></div>
