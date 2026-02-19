# istio-service-graph-data-collection
Tool to collect so sample dataset (metrics), for graph rendering optimization 

# Introduction 
Solo.io is working closely with customers to build a new service graph capability for advanced monitoring as part of Solo Enterprise for Istio. The goal of this feature is to visualize service dependencies in a clear and practical way.

To properly design, validate, and performance test the service graph, both internally and against expected customer production scale, the team needs a representative sample of real world data.

This guide explains how to collect and share the required data sample.

# Solo Enterprise Mesh  Setup

This document describes how to set up Solo Enterprise Mesh v3 in production environments and for local development.

Larger customers may need to configure a PVC to handle storage requirements, [PVC can be configured as part of the chart](https://github.com/solo-io/kagent-enterprise/blob/main/charts/management/values.yaml#L478-L480), which accepts these values

```
persistentVolume:
  # -- Enable persistent storage for ClickHouse
  # @section -- Storage
  enabled: false
  # -- Storage class to use for persistent volumes (uses default if empty)
  # @section -- Storage
  storageClass: ""
  # -- Size of persistent volume for ClickHouse data
  # @section -- Storage
  size: 20Gi

```

## Production environment

In a production environment, the transport between the management plane and agent components must be secured using Istio. This is especially critical for multi-cluster setups where the connectivity traverses the internet. Using Istio to secure the transport layer can be skipped for demo or testing purposes.

Set common environment variables:
```bash
# e.g. 0.3.3
export SOLO_ENT_VERSION=<solo-enterprise version>
export MGMT_CLUSTER=<management cluster name>
export MGMT_KUBE_CTX=<management kube context>
# if workload cluster is different from management cluster
export WORKLOAD_CLUSTER=<workload cluster name>
export WORKLOAD_KUBE_CTX=<workload kube context>
```

1. Install Istio in the management and workload clusters. Provision a shared root of trust across the clusters as per Istio's multi-cluster setup instructions.

2. Enable Ambient mode and multi-cluster peering in both clusters (skip for single cluster):
```bash
for ctx in ${MGMT_KUBE_CTX} ${WORKLOAD_KUBE_CTX}; do
  istioctl multicluster expose --namespace istio-system --context $ctx
  kubectl label namespace solo-enterprise istio.io/dataplane-mode=ambient --context $ctx
done

# Multi-cluster only (skip for single cluster):

# link clusters
istioctl multicluster link --namespace istio-system --contexts=${MGMT_KUBE_CTX},${WORKLOAD_KUBE_CTX}

# check status after linking
istioctl multicluster check --context ${MGMT_KUBE_CTX}
istioctl multicluster check --context ${WORKLOAD_KUBE_CTX}
```

3. Install the management plane:
```bash
helm upgrade -i management oci://us-docker.pkg.dev/solo-public/solo-enterprise-helm/charts/management --version ${SOLO_ENT_VERSION} -n solo-enterprise --create-namespace --kube-context ${MGMT_KUBE_CTX} -f - <<EOF
products:
  mesh:
    enabled: true
cluster: ${MGMT_CLUSTER}
EOF
```

4. Expose the management plane services to the agents (multi-cluster only):
```bash
kubectl --context ${MGMT_KUBE_CTX} label svc solo-enterprise-ui -n solo-enterprise solo.io/service-scope=global
kubectl --context ${MGMT_KUBE_CTX} label svc solo-enterprise-telemetry-gateway -n solo-enterprise solo.io/service-scope=global
```

5. Retrieve information from the management plane:
```bash
# kubectl --context ${MGMT_KUBE_CTX} get svc -n solo-enterprise solo-enterprise-ui -o=jsonpath='{.status.loadBalancer.ingress[0].ip}'
# or
# kubectl --context ${MGMT_KUBE_CTX} get svc -n solo-enterprise solo-enterprise-ui -o=jsonpath='{.status.loadBalancer.ingress[0].hostname}'
# or
# when using Istio multi-cluster: solo-enterprise-ui.solo-enterprise.mesh.internal
export TUNNEL_SERVER_FQDN=<external IP or hostname of solo-enterprise-ui service in mgmt cluster>
# kubectl --context ${MGMT_KUBE_CTX} get svc -n solo-enterprise solo-enterprise-telemetry-gateway -o=jsonpath='{.status.loadBalancer.ingress[0].ip}'
# or
# kubectl --context ${MGMT_KUBE_CTX} get svc -n solo-enterprise solo-enterprise-telemetry-gateway -o=jsonpath='{.status.loadBalancer.ingress[0].hostname}'
# or
# when using Istio multi-cluster: solo-enterprise-telemetry-gateway.solo-enterprise.mesh.internal
export TELEMETRY_GATEWAY_FQDN=<external IP or hostname of solo-enterprise-telemetry-gateway service in mgmt cluster>
```

6. Install the relay/agent component if the workloads are in a different cluster:
```bash
helm upgrade -i relay oci://us-docker.pkg.dev/solo-public/solo-enterprise-helm/charts/relay --version ${SOLO_ENT_VERSION} -n solo-enterprise --create-namespace --kube-context ${WORKLOAD_KUBE_CTX} -f - <<EOF
cluster: ${WORKLOAD_CLUSTER}
tunnel:
  fqdn: ${TUNNEL_SERVER_FQDN}
  port: 9000
telemetry:
  fqdn: ${TELEMETRY_GATEWAY_FQDN}
EOF
```

7. Attach the backup sidecar to ClickHouse. **This has to happen right after installation** as patching the StatefulSet recreates it:

```
kubectl --context ${MGMT_KUBE_CTX} patch statefulset solo-enterprise-management-clickhouse-shard0 -n solo-enterprise --type='strategic' -p '{
  "spec": {
    "template": {
      "spec": {
        "containers": [
          {
            "name": "clickhouse-backup",
            "image": "altinity/clickhouse-backup:latest",
            "command": ["clickhouse-backup", "server"],
            "env": [
              {"name": "CLICKHOUSE_HOST", "value": "127.0.0.1"},
              {"name": "CLICKHOUSE_PORT", "value": "9000"},
              {"name": "CLICKHOUSE_USER", "value": "default"},
              {"name": "CLICKHOUSE_PASSWORD", "value": "password"}
            ],
            "ports": [
              {"name": "backup-api", "containerPort": 7171}
            ],
            "volumeMounts": [
              {"name": "data", "mountPath": "/var/lib/clickhouse"}
            ]
          }
        ]
      }
    }
  }
}'
```

8. Wait for 6 hours to accumulate telemetry. If the data is too much, decrease this wait period.

9. Trigger backup creation:

```
kubectl --context ${MGMT_KUBE_CTX} exec -n solo-enterprise solo-enterprise-management-clickhouse-shard0-0 \
  -c clickhouse-backup -- clickhouse-backup create "backup-$(date +%Y%m%d-%H%M%S)"
```

10. Get the name of the previously created backup:

```
kubectl --context ${MGMT_KUBE_CTX} -n solo-enterprise solo-enterprise-management-clickhouse-shard0-0 \
  -c clickhouse-backup -- clickhouse-backup list local
```

11. Copy the backup with the name that you found in previous step:

```
mkdir -p ~/clickhouse-backups
kubectl --context ${MGMT_KUBE_CTX} cp \
  solo-enterprise/solo-enterprise-management-clickhouse-shard0-0:/var/lib/clickhouse/backup/<BACKUP_NAME> \
  ~/clickhouse-backups/<BACKUP_NAME> \
  -c clickhouse
```

12. Archive the backup:

```
kubectl --context ${MGMT_KUBE_CTX} exec -n solo-enterprise solo-enterprise-management-clickhouse-shard0-0 -c clickhouse -- \
  tar czf - -C /var/lib/clickhouse/backup backup-20260115-123053 \
  > ~/clickhouse-backups/backup-20260115-123053.tar.gz
```

## Backup to Cloud Storage
Altinity Clickhouse backup tool also includes configuration to persist backups to remote file storage (S3, GCS, SFTP, FTP). This is preferable to local backup.
https://github.com/Altinity/clickhouse-backup?tab=readme-ov-file#configurable-parameters

