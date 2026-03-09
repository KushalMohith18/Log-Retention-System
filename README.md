# Log Retention System using Elasticsearch, Kubernetes and S3

## Overview

This project implements a **Log Retention System** using **Elasticsearch 8.5.1 running on Kubernetes**, with automated **snapshot backups to AWS S3** and lifecycle-based log management.

The system stores logs in Elasticsearch, automatically manages log retention using **Index Lifecycle Management (ILM)**, and periodically backs up indices to **Amazon S3 snapshots** for durability and recovery.

This setup simulates how production observability stacks manage large volumes of logs efficiently.

---

## Architecture

```
Applications → Log Ingestion → Elasticsearch → ILM Lifecycle → S3 Snapshots
```

### Components

* **Kubernetes cluster** (Minikube)
* **Elasticsearch 8.5.1** (Single-node cluster)
* **Kibana 8.5.1** (Visualization and management)
* **AWS S3** (Snapshot repository, ap-south-1 region)
* **Elasticsearch Snapshot Lifecycle Management (SLM)**
* **Elasticsearch Index Lifecycle Management (ILM)**

---

## Features

* ✅ Logs stored in Elasticsearch indices
* ✅ Hot storage for 7 days (active indexing)
* ✅ Warm storage after 7 days (read-only)
* ✅ Automatic deletion after 30 days
* ✅ Automatic index rollover (30GB OR 1 day)
* ✅ Daily snapshots to S3
* ✅ Snapshot retention policy (30 days)
* ✅ Persistent storage for data and snapshots (5Gi each)

---

## Lifecycle Policy

| Phase  | Duration      | Action          |
| ------ | ------------- | --------------- |
| Hot    | 0-7 days      | Active indexing |
| Warm   | After 7 days  | Read-only       |
| Delete | After 30 days | Index deleted   |

---

## Snapshot Policy

* **Snapshot Frequency**: Daily at 2:00 AM
* **Storage**: AWS S3 (ap-south-1 region)
* **Retention**: 30 days
* **Min Count**: 5 snapshots
* **Max Count**: 50 snapshots

---

## Project Structure

```
Log-Retention-System/
│
├── ElasticSearch/
│   ├── configmap.yaml          # Elasticsearch configuration
│   ├── statefulset.yaml        # StatefulSet with persistent volumes
│   └── svc.yaml                # Service for Elasticsearch
│
├── Kibana/
│   └── deployment.yaml         # Kibana deployment and service
│
└── README.md
```

---

## Infrastructure Details

### Elasticsearch Configuration
- **Version**: 8.5.1
- **Cluster Name**: prod-logs
- **Discovery Type**: single-node
- **Security**: Disabled (xpack.security.enabled: false)
- **Network**: 0.0.0.0 (all interfaces)
- **Resources**: 1Gi memory, 500m-1 CPU
- **Storage**: 5Gi data volume + 5Gi snapshot volume
- **S3 Endpoint**: s3.ap-south-1.amazonaws.com

### Kibana Configuration
- **Version**: 8.5.1
- **Service Type**: NodePort
- **Elasticsearch Connection**: http://elasticsearch-client:9200

---

## Prerequisites

Before running the project, ensure the following tools are installed:

* **Docker** (for container runtime)
* **Kubernetes** (kubectl CLI)
* **Minikube** (local Kubernetes cluster)
* **AWS CLI** (configured with credentials)
* **AWS Account** with S3 access
* **S3 Bucket** (in ap-south-1 region)

---

## Deployment Steps

### Step 1: Start Kubernetes Cluster

```bash
minikube start
```

Create the logging namespace:

```bash
kubectl create namespace logging
```

---

### Step 2: Deploy Elasticsearch

Apply the Elasticsearch configuration, StatefulSet, and Service:

```bash
kubectl apply -f ElasticSearch/configmap.yaml
kubectl apply -f ElasticSearch/statefulset.yaml
kubectl apply -f ElasticSearch/svc.yaml
```

Wait for Elasticsearch to be ready:

```bash
kubectl get pods -n logging -w
```

Verify the pod is running:

```bash
kubectl get statefulsets -n logging
```

---

### Step 3: Deploy Kibana

Apply the Kibana deployment and service:

```bash
kubectl apply -f Kibana/deployment.yaml
```

Wait for Kibana to be ready:

```bash
kubectl get pods -n logging
```

Access Kibana using port-forward:

```bash
kubectl port-forward service/kibana 5601:5601 -n logging
```

Open your browser and navigate to:

```
http://localhost:5601
```

---

### Step 4: Configure S3 Snapshot Repository

#### Add AWS Credentials to Elasticsearch Keystore

Enter the Elasticsearch pod:

```bash
kubectl exec -it elasticsearch-0 -n logging -- bash
```

Add your AWS credentials to the keystore:

```bash
bin/elasticsearch-keystore add s3.client.default.access_key
# Enter your AWS Access Key ID when prompted

bin/elasticsearch-keystore add s3.client.default.secret_key
# Enter your AWS Secret Access Key when prompted
```

Reload the secure settings without restarting:

```bash
curl -X POST "localhost:9200/_nodes/reload_secure_settings?pretty" -H 'Content-Type: application/json'
```

Exit the pod:

```bash
exit
```

---

### Step 5: Create S3 Snapshot Repository

Open **Kibana Dev Tools** (`http://localhost:5601/app/dev_tools#/console`) and run:

```json
PUT _snapshot/s3_repo
{
  "type": "s3",
  "settings": {
    "bucket": "your-s3-bucket-name",
    "region": "ap-south-1"
  }
}
```

Verify the repository:

```json
GET _snapshot/s3_repo
```

---

### Step 6: Configure Index Lifecycle Management (ILM)

Create the ILM policy for log retention:

```json
PUT _ilm/policy/log-retention-policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_size": "30gb",
            "max_age": "1d"
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "readonly": {}
        }
      },
      "delete": {
        "min_age": "30d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

Verify the policy:

```json
GET _ilm/policy/log-retention-policy
```

---

### Step 7: Create Index Template

Create an index template that applies the ILM policy to all `logs-*` indices:

```json
PUT _index_template/log-template
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 3,
      "number_of_replicas": 1,
      "index.lifecycle.name": "log-retention-policy",
      "index.lifecycle.rollover_alias": "logs"
    }
  }
}
```

---

### Step 8: Create Initial Index

Create the first index with a write alias:

```json
PUT logs-000001
{
  "aliases": {
    "logs": {
      "is_write_index": true
    }
  }
}
```

Verify the index:

```json
GET logs-000001
```

---

### Step 9: Configure Snapshot Lifecycle Management (SLM)

Create a daily snapshot policy:

```json
PUT _slm/policy/daily-s3-snapshot
{
  "schedule": "0 0 2 * * ?",
  "name": "<daily-snapshot-{now/d}>",
  "repository": "s3_repo",
  "config": {
    "indices": ["logs-*"]
  },
  "retention": {
    "expire_after": "30d",
    "min_count": 5,
    "max_count": 50
  }
}
```

**Schedule Explanation**: `0 0 2 * * ?` = Daily at 2:00 AM

Verify the policy:

```json
GET _slm/policy/daily-s3-snapshot
```

---

## Testing the System

### Insert Sample Logs

Insert test log documents:

```json
POST logs/_doc
{
  "timestamp": "2026-03-09T10:30:00Z",
  "service": "api-gateway",
  "level": "INFO",
  "message": "Request processed successfully"
}

POST logs/_doc
{
  "timestamp": "2026-03-09T10:35:00Z",
  "service": "auth-service",
  "level": "ERROR",
  "message": "Failed authentication attempt"
}
```

Search logs:

```json
GET logs/_search
{
  "query": {
    "match_all": {}
  }
}
```

---

### Test Snapshot Creation

Create a manual snapshot:

```json
PUT _snapshot/s3_repo/test_snapshot
{
  "indices": "logs-*",
  "include_global_state": false
}
```

Check snapshot status:

```json
GET _snapshot/s3_repo/test_snapshot
```

List all snapshots:

```json
GET _snapshot/s3_repo/_all
```

---

### Verify Snapshots in S3

1. Open your AWS Console
2. Navigate to S3 → Your bucket
3. Confirm the following files exist:

```
indices/
snap-<snapshot-id>.dat
meta-<snapshot-id>.dat
index-<number>
```

---

### Test Snapshot Restore

Delete test index (be careful!):

```json
DELETE logs-000001
```

Restore from snapshot:

```json
POST _snapshot/s3_repo/test_snapshot/_restore
{
  "indices": "logs-*"
}
```

Verify restoration:

```json
GET logs/_search
```

---

## Monitoring and Management

### View ILM Status

```json
GET _ilm/status
GET logs-000001/_ilm/explain
```

### View SLM Status

```json
GET _slm/status
GET _slm/policy
```

### Check Elasticsearch Cluster Health

```json
GET _cluster/health
GET _cat/indices?v
GET _cat/nodes?v
```

---

## Troubleshooting

### Elasticsearch Pod Not Starting

```bash
kubectl describe pod elasticsearch-0 -n logging
kubectl logs elasticsearch-0 -n logging
```

### Kibana Cannot Connect

Check service endpoints:

```bash
kubectl get svc -n logging
kubectl describe svc elasticsearch-client -n logging
```

### S3 Snapshot Failures

Check repository settings:

```json
POST _snapshot/s3_repo/_verify
```

Check AWS credentials:

```bash
kubectl exec -it elasticsearch-0 -n logging -- bash
bin/elasticsearch-keystore list
```

---

## Cleanup

To remove all resources:

```bash
kubectl delete namespace logging
```

To delete S3 snapshots, use AWS CLI or Console.

---

## Technologies Used

* **Kubernetes** - Container orchestration
* **Elasticsearch 8.5.1** - Search and analytics engine
* **Kibana 8.5.1** - Data visualization and management
* **AWS S3** - Object storage for snapshots
* **Docker** - Container runtime
* **Minikube** - Local Kubernetes cluster

---

## Future Enhancements

- [ ] Add Logstash/Fluentd for log ingestion
- [ ] Implement Elasticsearch cluster with multiple nodes
- [ ] Enable X-Pack security features
- [ ] Add monitoring with Prometheus and Grafana
- [ ] Implement hot-warm-cold architecture
- [ ] Add alerting for snapshot failures
- [ ] Configure cross-region replication

---

## Author

**Kushal Mohith**

DevOps | Cloud | Observability

---

## License

This project is open source and available for educational purposes.
