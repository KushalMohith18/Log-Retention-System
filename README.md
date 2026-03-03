# Production-Ready Log Retention System

A Kubernetes-based log retention system using Elasticsearch and Kibana with automated lifecycle management, snapshot backups, and optimized storage tiering.

## 📋 Overview

This repository contains a production-ready log retention solution designed to manage logs efficiently through their lifecycle—from active querying (hot) to archival (warm) and eventual deletion. The system uses Elasticsearch's Index Lifecycle Management (ILM) policies to automate log retention and AWS S3 for snapshot backups.

## 🏗️ Architecture

### Components

- **Elasticsearch 8.5.1**: Core search and analytics engine with ILM policies
- **Kibana 8.5.1**: Visualization and management interface
- **Kubernetes**: Container orchestration platform
- **AWS S3**: External snapshot repository for backups

### Deployment Structure

```
Log-Retention System/
├── ElasticSearch/
│   ├── configmap.yaml       # Elasticsearch configuration
│   ├── statefulset.yaml     # StatefulSet with persistent storage
│   └── svc.yaml             # Service definition
└── Kibana/
    └── deployment.yaml      # Kibana deployment and service
```

## ⚙️ Log Retention Requirements

The system implements the following production requirements:

| Requirement | Configuration |
|-------------|---------------|
| **Hot Phase Duration** | 7 days |
| **Warm Phase** | After 7 days (read-only, force merge) |
| **Snapshot Frequency** | Daily to S3 |
| **Data Retention** | 30 days (delete after) |
| **Primary Shards** | 3 per index |
| **Replica Shards** | 1 per index |
| **Rollover Triggers** | 30GB OR 1 day (whichever first) |

## 🚀 Quick Start

### Prerequisites

- Kubernetes cluster (v1.20+)
- kubectl configured
- AWS S3 bucket for snapshots (optional for initial setup)
- Minimum 2GB RAM available

### Deployment Steps

1. **Create the logging namespace:**
   ```bash
   kubectl create namespace logging
   ```

2. **Deploy Elasticsearch:**
   ```bash
   kubectl apply -f ElasticSearch/configmap.yaml
   kubectl apply -f ElasticSearch/statefulset.yaml
   kubectl apply -f ElasticSearch/svc.yaml
   ```

3. **Wait for Elasticsearch to be ready:**
   ```bash
   kubectl wait --for=condition=ready pod -l app=elasticsearch -n logging --timeout=300s
   ```

4. **Deploy Kibana:**
   ```bash
   kubectl apply -f Kibana/deployment.yaml
   ```

5. **Access Kibana:**
   ```bash
   kubectl port-forward svc/kibana 5601:5601 -n logging
   ```
   Then open http://localhost:5601 in your browser.

## 📊 Index Lifecycle Management (ILM) Configuration

### ILM Policy Setup

After deployment, configure the ILM policy via Elasticsearch API:

```bash
# Port forward to Elasticsearch
kubectl port-forward svc/elasticsearch-client 9200:9200 -n logging

# Create the ILM policy
curl -X PUT "localhost:9200/_ilm/policy/logs-policy" -H 'Content-Type: application/json' -d'
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_size": "30gb",
            "max_age": "1d"
          },
          "set_priority": {
            "priority": 100
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "readonly": {},
          "forcemerge": {
            "max_num_segments": 1
          },
          "set_priority": {
            "priority": 50
          }
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
'
```

### Index Template Setup

Create an index template to apply the ILM policy and shard configuration:

```bash
curl -X PUT "localhost:9200/_index_template/logs-template" -H 'Content-Type: application/json' -d'
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 3,
      "number_of_replicas": 1,
      "index.lifecycle.name": "logs-policy",
      "index.lifecycle.rollover_alias": "logs"
    }
  }
}
'
```

### Bootstrap the First Index

```bash
curl -X PUT "localhost:9200/logs-000001" -H 'Content-Type: application/json' -d'
{
  "aliases": {
    "logs": {
      "is_write_index": true
    }
  }
}
'
```

## 💾 Snapshot Configuration

### Configure S3 Repository

1. **Install S3 Repository Plugin (if needed):**
   ```bash
   kubectl exec -it elasticsearch-0 -n logging -- bin/elasticsearch-plugin install repository-s3
   kubectl rollout restart statefulset/elasticsearch -n logging
   ```

2. **Add AWS Credentials to Keystore:**
   ```bash
   kubectl exec -it elasticsearch-0 -n logging -- bash
   bin/elasticsearch-keystore add s3.client.default.access_key
   bin/elasticsearch-keystore add s3.client.default.secret_key
   exit
   kubectl rollout restart statefulset/elasticsearch -n logging
   ```

3. **Register the S3 Repository:**
   ```bash
   curl -X PUT "localhost:9200/_snapshot/s3-backup" -H 'Content-Type: application/json' -d'
   {
     "type": "s3",
     "settings": {
       "bucket": "your-backup-bucket-name",
       "region": "us-east-1",
       "base_path": "elasticsearch-snapshots"
     }
   }
   '
   ```

### Daily Snapshot Policy

Configure automatic daily snapshots:

```bash
curl -X PUT "localhost:9200/_slm/policy/daily-snapshots" -H 'Content-Type: application/json' -d'
{
  "schedule": "0 0 1 * * ?",
  "name": "<daily-snap-{now/d}>",
  "repository": "s3-backup",
  "config": {
    "indices": ["logs-*"],
    "ignore_unavailable": false,
    "include_global_state": false
  },
  "retention": {
    "expire_after": "30d",
    "min_count": 5,
    "max_count": 50
  }
}
'
```

## 📈 Monitoring and Management

### Check ILM Policy Status

```bash
curl -X GET "localhost:9200/_ilm/status"
```

### View Index Lifecycle

```bash
curl -X GET "localhost:9200/logs-*/_ilm/explain"
```

### Check Snapshot Status

```bash
curl -X GET "localhost:9200/_snapshot/s3-backup/_all"
```

### Monitor Cluster Health

```bash
curl -X GET "localhost:9200/_cluster/health?pretty"
```

## 🔧 Configuration Details

### Elasticsearch Resources

- **Memory**: 1Gi (request and limit)
- **CPU**: 500m request, 1 CPU limit
- **Storage**: 5Gi persistent volume per pod
- **Snapshots**: 5Gi persistent volume for local snapshots

### Shard Configuration

- **Primary Shards**: 3 (for parallel processing and distribution)
- **Replica Shards**: 1 (for high availability)
- **Total Shards per Index**: 6 (3 primary + 3 replica)

### Rollover Conditions

Indices automatically rollover when **either** condition is met:
- Size exceeds **30GB**
- Age exceeds **1 day**

## 🛠️ Troubleshooting

### Check Elasticsearch Logs

```bash
kubectl logs -f elasticsearch-0 -n logging
```

### Check Kibana Logs

```bash
kubectl logs -f deployment/kibana -n logging
```

### Verify ILM is Working

```bash
# Check if ILM is running
curl -X GET "localhost:9200/_ilm/status"

# If stopped, start it
curl -X POST "localhost:9200/_ilm/start"
```

### Test Snapshot Repository

```bash
curl -X POST "localhost:9200/_snapshot/s3-backup/_verify"
```

## 📝 Sending Logs

You can send logs to the system using various methods:

### Via curl

```bash
curl -X POST "localhost:9200/logs/_doc" -H 'Content-Type: application/json' -d'
{
  "timestamp": "'$(date -u +%Y-%m-%dT%H:%M:%S.%3NZ)'",
  "level": "INFO",
  "message": "Application started successfully",
  "service": "my-app",
  "environment": "production"
}
'
```

### Via Filebeat

Configure Filebeat to send logs to `elasticsearch-client.logging.svc.cluster.local:9200` with index name `logs`.

### Via Logstash

Configure Logstash output to use the `logs` alias for automatic rollover.

## 🔐 Production Considerations

### Security (To Be Implemented)

For production environments, consider enabling:
- X-Pack Security (authentication and authorization)
- TLS/SSL for communication
- Network policies
- RBAC for Kubernetes resources

### Scaling

To scale Elasticsearch:
```bash
kubectl scale statefulset elasticsearch --replicas=3 -n logging
```

Update the `discovery.type` in configmap.yaml from `single-node` to `zen` for multi-node clusters.

### Resource Tuning

Adjust memory settings based on your log volume:
- Each shard requires ~1MB of heap
- Recommended heap: 50% of pod memory (max 32GB)
- Consider using memory-mapped files for better performance

## 📚 Additional Resources

- [Elasticsearch ILM Documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-lifecycle-management.html)
- [Snapshot and Restore](https://www.elastic.co/guide/en/elasticsearch/reference/current/snapshot-restore.html)
- [Elasticsearch on Kubernetes Best Practices](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-overview.html)

## 📄 License

This project is provided as-is for educational and production use.

---

**Last Updated**: March 2026
