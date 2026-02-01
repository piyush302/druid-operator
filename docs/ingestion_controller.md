# Ingestion Controller

The Ingestion Controller manages Apache Druid ingestion tasks as Kubernetes-native resources using the `DruidIngestion` Custom Resource Definition (CRD).

## Overview

The `DruidIngestion` CRD allows you to:
- Create and manage Druid ingestion tasks declaratively
- Support multiple ingestion types (native batch, Kafka, Kinesis)
- Configure compaction and retention rules
- Automatically clean up tasks when the resource is deleted

## Supported Ingestion Types

| Type | Value | Description |
|------|-------|-------------|
| Native Batch | `native-batch` | Parallel batch indexing for one-time data loads |
| Kafka | `kafka` | Streaming ingestion from Apache Kafka |
| Kinesis | `kinesis` | Streaming ingestion from AWS Kinesis |
| SQL | `sql` | Query controller SQL-based ingestion |
| Hadoop | `index-hadoop` | Hadoop-based batch indexing |

## API Reference

### DruidIngestion Spec

```yaml
apiVersion: druid.apache.org/v1alpha1
kind: DruidIngestion
metadata:
  name: <ingestion-name>
spec:
  suspend: false                    # Optional: Suspend the ingestion task
  druidCluster: <cluster-name>      # Required: Name of the Druid cluster CR
  auth:                             # Optional: Authentication configuration
    secretRef:
      name: <secret-name>
    type: basic-auth
  ingestion:
    type: <ingestion-type>          # Required: kafka, native-batch, kinesis, sql, index-hadoop
    spec: |                         # Option 1: JSON string format
      { ... }
    nativeSpec:                     # Option 2: Native YAML format (recommended)
      ...
    compaction:                     # Optional: Compaction settings
      ...
    rules:                          # Optional: Retention rules
      - ...
```

### Status Fields

| Field | Description |
|-------|-------------|
| `taskId` | The Druid task/supervisor ID |
| `status` | Current condition status (True/False) |
| `message` | Human-readable status message |
| `lastUpdateTime` | Timestamp of last status update |
| `currentIngestionSpec` | The currently applied ingestion spec |

## Examples

### Native Batch Ingestion

Load data from a local file using parallel batch indexing:

```yaml
apiVersion: druid.apache.org/v1alpha1
kind: DruidIngestion
metadata:
  name: wikipedia-batch
spec:
  druidCluster: tiny-cluster
  ingestion:
    type: native-batch
    nativeSpec:
      type: index_parallel
      spec:
        dataSchema:
          dataSource: wikipedia
          timestampSpec:
            column: time
            format: iso
          dimensionsSpec:
            dimensions:
              - channel
              - page
              - user
              - name: added
                type: long
          granularitySpec:
            type: uniform
            segmentGranularity: day
            queryGranularity: none
            rollup: false
        ioConfig:
          type: index_parallel
          inputSource:
            type: local
            baseDir: quickstart/tutorial/
            filter: "*.json.gz"
          inputFormat:
            type: json
        tuningConfig:
          type: index_parallel
          partitionsSpec:
            type: dynamic
```

### Kafka Streaming Ingestion

Create a Kafka supervisor for real-time streaming:

```yaml
apiVersion: druid.apache.org/v1alpha1
kind: DruidIngestion
metadata:
  name: metrics-kafka
spec:
  druidCluster: tiny-cluster
  ingestion:
    type: kafka
    nativeSpec:
      type: kafka
      spec:
        dataSchema:
          dataSource: metrics-kafka
          timestampSpec:
            column: timestamp
            format: auto
          dimensionsSpec:
            dimensions: []
            dimensionExclusions:
              - timestamp
              - value
          metricsSpec:
            - name: count
              type: count
            - name: value_sum
              fieldName: value
              type: doubleSum
          granularitySpec:
            type: uniform
            segmentGranularity: HOUR
            queryGranularity: NONE
        ioConfig:
          topic: metrics
          inputFormat:
            type: json
          consumerProperties:
            bootstrap.servers: kafka:9092
          taskCount: 1
          replicas: 1
          taskDuration: PT1H
        tuningConfig:
          type: kafka
          maxRowsPerSegment: 5000000
```

### With Compaction and Retention Rules

Configure automatic compaction and data retention:

```yaml
apiVersion: druid.apache.org/v1alpha1
kind: DruidIngestion
metadata:
  name: kafka-with-rules
spec:
  druidCluster: tiny-cluster
  ingestion:
    type: kafka
    compaction:
      tuningConfig:
        type: kafka
        partitionsSpec:
          type: dynamic
      skipOffsetFromLatest: PT0S
      granularitySpec:
        segmentGranularity: DAY
    rules:
      - type: dropByPeriod
        period: P30D
        includeFuture: true
      - type: loadByPeriod
        period: P7D
        includeFuture: true
        tieredReplicants:
          _default_tier: 2
    nativeSpec:
      type: kafka
      spec:
        # ... ingestion spec
```

## Authentication

To use authentication with the Druid API, create a secret and reference it:

```yaml
# Create the secret
apiVersion: v1
kind: Secret
metadata:
  name: druid-auth
type: Opaque
stringData:
  username: admin
  password: your-password
---
# Reference in DruidIngestion
apiVersion: druid.apache.org/v1alpha1
kind: DruidIngestion
metadata:
  name: secure-ingestion
spec:
  druidCluster: tiny-cluster
  auth:
    secretRef:
      name: druid-auth
    type: basic-auth
  ingestion:
    type: native-batch
    nativeSpec:
      # ... spec
```

## Spec Format: JSON vs Native YAML

You can define the ingestion spec in two formats:

### JSON String (Legacy)

```yaml
spec:
  ingestion:
    spec: |-
      {
        "type": "index_parallel",
        "spec": { ... }
      }
```

### Native YAML (Recommended)

```yaml
spec:
  ingestion:
    nativeSpec:
      type: index_parallel
      spec:
        # ... native YAML structure
```

The `nativeSpec` format is recommended because:
- Better readability and maintainability
- Native Kubernetes YAML validation
- Easier to use with Helm templating and environment variables
- If both are provided, `nativeSpec` takes precedence

## Lifecycle Management

### Creation
When a `DruidIngestion` resource is created, the controller:
1. Connects to the Druid cluster's router service
2. Submits the ingestion task/supervisor via Druid API
3. Updates the status with the task ID

### Updates
When the spec changes:
1. Controller detects the change
2. Submits updated spec to Druid
3. For supervisors (Kafka/Kinesis), the existing supervisor is updated
4. For batch tasks, a new task is created

### Deletion
When the resource is deleted:
1. Finalizer ensures cleanup
2. Controller sends shutdown request to Druid
3. Task/supervisor is terminated
4. Resource is removed

## Troubleshooting

### Check Ingestion Status

```bash
kubectl get druidingestion
kubectl describe druidingestion <name>
```

### View Controller Events

```bash
kubectl get events --field-selector involvedObject.kind=DruidIngestion
```

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Task not created | Druid cluster not ready | Ensure Druid cluster is running |
| Authentication failed | Invalid credentials | Verify secret exists and has correct keys |
| Spec validation error | Invalid ingestion spec | Check Druid documentation for spec format |

## Related Resources

- [Apache Druid Ingestion Documentation](https://druid.apache.org/docs/latest/ingestion/index.html)
- [Kafka Ingestion](https://druid.apache.org/docs/latest/development/extensions-core/kafka-ingestion.html)
- [Native Batch Ingestion](https://druid.apache.org/docs/latest/ingestion/native-batch.html)
