# istio-service-graph-data-collection
Tool to collect so sample dataset (metrics), for graph rendering optimization 

# Introduction 
Solo.io is working closely with customers to build a new service graph capability for advanced monitoring as part of Solo Enterprise for Istio. The goal of this feature is to visualize service dependencies in a clear and practical way.

To properly design, validate, and performance test the service graph, both internally and against expected customer production scale, the team needs a representative sample of real world data.

This guide explains how to collect and share the required data sample.

# ClickHouse Backup to S3 (AWS/EKS)

Step-by-step instructions for backing up ClickHouse data to S3 using [Altinity clickhouse-backup](https://github.com/Altinity/clickhouse-backup) on an EKS cluster with IRSA authentication.

---

## Prerequisites

- AWS CLI, kubectl, helm, eksctl installed
- AWS credentials with EKS + IAM + S3 permissions

## 1. Set Environment Variables

```bash
export SOLO_ENT_VERSION="0.3.5"   # adjust to your version
export CLUSTER_NAME="<your-eks-cluster>"
export AWS_REGION="us-east-1"
export NAMESPACE="solo-enterprise"
export RELEASE_NAME="management"
export CH_STS="${RELEASE_NAME}-clickhouse-shard0"
export CH_POD="${CH_STS}-0"
export CH_SA="${RELEASE_NAME}-clickhouse"
export S3_BUCKET="<your-s3-bucket>"
export MGMT_KUBE_CTX="${CLUSTER_NAME}"
```

## 2. (Optional) Create EKS Cluster

Skip this step if you already have an EKS cluster. Set `CLUSTER_NAME` and `MGMT_KUBE_CTX` to match your existing cluster.

```bash
eksctl create cluster \
  --name "${CLUSTER_NAME}" \
  --region "${AWS_REGION}" \
  --version 1.31 \
  --nodegroup-name workers \
  --node-type m5.xlarge \
  --nodes 2 \
  --managed
```

This takes ~15-20 minutes. Then set kubeconfig:

```bash
aws eks update-kubeconfig --name "${CLUSTER_NAME}" --region "${AWS_REGION}" --alias "${CLUSTER_NAME}"
```

Install the EBS CSI driver addon (required for persistent volumes on EKS):

```bash
eksctl create addon --name aws-ebs-csi-driver \
  --cluster "${CLUSTER_NAME}" --region "${AWS_REGION}" --force
```

Set `gp2` as the default StorageClass (eksctl doesn't do this automatically):

```bash
kubectl --context "${MGMT_KUBE_CTX}" annotate storageclass gp2 \
  storageclass.kubernetes.io/is-default-class=true
```

## 3. (Optional) Install Management Plane

Skip if the management Helm chart is already installed. **Important**: ClickHouse must have persistent storage enabled with an explicit `storageClass`.

```bash
helm upgrade -i "${RELEASE_NAME}" \
  oci://us-docker.pkg.dev/solo-public/solo-enterprise-helm/charts/management \
  --version "${SOLO_ENT_VERSION}" \
  --namespace "${NAMESPACE}" \
  --create-namespace \
  --kube-context "${MGMT_KUBE_CTX}" \
  -f - <<EOF
products:
  mesh:
    enabled: true
cluster: mgmt
clickhouse:
  persistentVolume:
    enabled: true
    size: 20Gi
    storageClass: gp2
EOF
```

Wait for ClickHouse:

```bash
kubectl --context "${MGMT_KUBE_CTX}" -n "${NAMESPACE}" \
  wait --for=condition=ready pod/${CH_POD} --timeout=300s
```

## 4. Create S3 Bucket

```bash
aws s3api create-bucket --bucket "${S3_BUCKET}" --region "${AWS_REGION}"
```

> For regions other than `us-east-1`, add `--create-bucket-configuration LocationConstraint=${AWS_REGION}`.

## 5. Set Up IRSA (IAM Roles for Service Accounts)

### 5a. Associate OIDC provider

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster "${CLUSTER_NAME}" --region "${AWS_REGION}" --approve
```

### 5b. Get OIDC and account info

```bash
export OIDC_ISSUER=$(aws eks describe-cluster --name "${CLUSTER_NAME}" \
  --region "${AWS_REGION}" --query "cluster.identity.oidc.issuer" --output text)
export OIDC_ID=$(echo "${OIDC_ISSUER}" | sed 's|https://||')
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)
```

### 5c. Create IAM policy

```bash
aws iam create-policy --policy-name "ch-backup-s3-policy" \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": ["s3:PutObject","s3:GetObject","s3:DeleteObject","s3:ListBucket","s3:GetBucketLocation"],
      "Resource": ["arn:aws:s3:::'${S3_BUCKET}'","arn:aws:s3:::'${S3_BUCKET}'/*"]
    }]
  }'
export POLICY_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:policy/ch-backup-s3-policy"
```

### 5d. Create IAM role with trust policy

```bash
export ROLE_NAME="ch-backup-role"
aws iam create-role --role-name "${ROLE_NAME}" \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Federated": "arn:aws:iam::'${AWS_ACCOUNT_ID}':oidc-provider/'${OIDC_ID}'"},
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {"StringEquals": {
        "'${OIDC_ID}':sub": "system:serviceaccount:'${NAMESPACE}':'${CH_SA}'",
        "'${OIDC_ID}':aud": "sts.amazonaws.com"
      }}
    }]
  }'
aws iam attach-role-policy --role-name "${ROLE_NAME}" --policy-arn "${POLICY_ARN}"
export ROLE_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:role/${ROLE_NAME}"
```

### 5e. Annotate the ClickHouse ServiceAccount

```bash
kubectl --context "${MGMT_KUBE_CTX}" -n "${NAMESPACE}" annotate sa "${CH_SA}" \
  "eks.amazonaws.com/role-arn=${ROLE_ARN}"
```

## 6. Patch ClickHouse StatefulSet with clickhouse-backup Sidecar

```bash
kubectl --context "${MGMT_KUBE_CTX}" -n "${NAMESPACE}" \
  patch statefulset "${CH_STS}" --type='json' \
  -p='[{
    "op": "add",
    "path": "/spec/template/spec/containers/-",
    "value": {
      "name": "clickhouse-backup",
      "image": "altinity/clickhouse-backup:latest",
      "imagePullPolicy": "IfNotPresent",
      "command": ["clickhouse-backup", "server"],
      "env": [
        {"name": "CLICKHOUSE_HOST", "value": "127.0.0.1"},
        {"name": "CLICKHOUSE_PORT", "value": "9000"},
        {"name": "CLICKHOUSE_USERNAME", "value": "default"},
        {"name": "CLICKHOUSE_PASSWORD", "value": "password"},
        {"name": "REMOTE_STORAGE", "value": "s3"},
        {"name": "S3_BUCKET", "value": "'${S3_BUCKET}'"},
        {"name": "S3_REGION", "value": "'${AWS_REGION}'"},
        {"name": "S3_PATH", "value": "clickhouse-backups"},
        {"name": "S3_ACL", "value": ""},
        {"name": "AWS_REGION", "value": "'${AWS_REGION}'"}
      ],
      "ports": [{"name": "backup-api", "containerPort": 7171}],
      "volumeMounts": [{"name": "data", "mountPath": "/var/lib/clickhouse"}]
    }
  }]'
```

Wait for rollout:

```bash
kubectl --context "${MGMT_KUBE_CTX}" -n "${NAMESPACE}" \
  rollout status statefulset/${CH_STS} --timeout=300s
```

Verify sidecar (expect `2/2 Running`):

```bash
kubectl --context "${MGMT_KUBE_CTX}" -n "${NAMESPACE}" get pod ${CH_POD}
```

If collecting data in a real environment, at this point wait for the duration of the test before proceeding with the following steps.

## 7. Create Backup

```bash
export BACKUP_NAME="backup-$(date +%Y%m%d-%H%M%S)"
kubectl --context "${MGMT_KUBE_CTX}" exec -n "${NAMESPACE}" "${CH_POD}" \
  -c clickhouse-backup -- clickhouse-backup create "${BACKUP_NAME}"
```

Verify:

```bash
kubectl --context "${MGMT_KUBE_CTX}" exec -n "${NAMESPACE}" "${CH_POD}" \
  -c clickhouse-backup -- clickhouse-backup list local
```

## 8. Upload Backup to S3

```bash
kubectl --context "${MGMT_KUBE_CTX}" exec -n "${NAMESPACE}" "${CH_POD}" \
  -c clickhouse-backup -- clickhouse-backup upload "${BACKUP_NAME}"
```

Verify in S3:

```bash
aws s3 ls "s3://${S3_BUCKET}/clickhouse-backups/" --recursive | head -20
```

## 9. Download Backup from S3

```bash
mkdir -p ~/clickhouse-backups
aws s3 cp "s3://${S3_BUCKET}/clickhouse-backups/${BACKUP_NAME}/" \
  ~/clickhouse-backups/${BACKUP_NAME}/ --recursive
```

## 10. Alternative: Local Copy from Pod

### Direct copy

```bash
kubectl --context "${MGMT_KUBE_CTX}" cp \
  "${NAMESPACE}/${CH_POD}:/var/lib/clickhouse/backup/${BACKUP_NAME}" \
  ~/clickhouse-backups/${BACKUP_NAME}-local \
  -c clickhouse-backup
```

### Tar archive

```bash
kubectl --context "${MGMT_KUBE_CTX}" exec -n "${NAMESPACE}" "${CH_POD}" -c clickhouse -- \
  tar czf - -C /var/lib/clickhouse/backup "${BACKUP_NAME}" \
  > ~/clickhouse-backups/${BACKUP_NAME}.tar.gz
```

## 11. Restore from Backup

To restore to the **same** ClickHouse instance, use `--restore-database-mapping` to avoid UUID collisions with the Atomic database engine:

```bash
# Restore to a "restored" database (avoids UUID collision with existing tables)
kubectl --context "${MGMT_KUBE_CTX}" exec -n "${NAMESPACE}" "${CH_POD}" \
  -c clickhouse-backup -- clickhouse-backup restore \
  --restore-database-mapping "default:restored" "${BACKUP_NAME}"
```

To restore to a **fresh** ClickHouse instance (no existing tables), restore directly:

```bash
# Download from S3 first if needed
kubectl --context "${MGMT_KUBE_CTX}" exec -n "${NAMESPACE}" "${CH_POD}" \
  -c clickhouse-backup -- clickhouse-backup download "${BACKUP_NAME}"

# Restore all tables
kubectl --context "${MGMT_KUBE_CTX}" exec -n "${NAMESPACE}" "${CH_POD}" \
  -c clickhouse-backup -- clickhouse-backup restore "${BACKUP_NAME}"
```

## (Optional) Cleanup

```bash
# Delete local backup files
rm -rf ~/clickhouse-backups/${BACKUP_NAME}*

# Empty and delete S3 bucket
aws s3 rm "s3://${S3_BUCKET}" --recursive
aws s3api delete-bucket --bucket "${S3_BUCKET}" --region "${AWS_REGION}"

# Detach and delete IAM role/policy
aws iam detach-role-policy --role-name "${ROLE_NAME}" --policy-arn "${POLICY_ARN}"
aws iam delete-role --role-name "${ROLE_NAME}"
aws iam delete-policy --policy-arn "${POLICY_ARN}"

# Uninstall Helm release and delete PVCs
helm uninstall "${RELEASE_NAME}" --namespace "${NAMESPACE}" --kube-context "${MGMT_KUBE_CTX}"
kubectl --context "${MGMT_KUBE_CTX}" -n "${NAMESPACE}" delete pvc --all

# Delete EKS cluster (includes OIDC provider cleanup, ~10-15 min)
eksctl delete cluster --name "${CLUSTER_NAME}" --region "${AWS_REGION}"
```

