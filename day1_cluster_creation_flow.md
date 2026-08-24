# ROSA CLI Day 1 Cluster Creation Flow

This document traces the complete flow of cluster creation and AWS resource provisioning for Day 1 tests in the `openshift/rosa` repository.

## Overview

Day 1 testing creates a full ROSA cluster with all required AWS infrastructure. The test flow is split into two phases:
1. **Cluster Creation** - Creates the cluster and all AWS resources
2. **Wait for Ready** - Waits for cluster to reach ready state and performs post-configuration

---

## Entry Point: `tests/e2e/e2e_setup_test.go`

### Test 1: "by profile" (Lines 23-42) - CREATES CLUSTER

```go
It("by profile", labels.Runtime.Day1, labels.Critical, func() {
    // 1. Initialize clients and load profile
    client := rosacli.NewClient()
    profile := handler.LoadProfileYamlFileByENV()  // Loads from TEST_PROFILE env var
    clusterHandler, err := handler.NewClusterHandler(client, profile)
    
    // 2. Prepare IAM roles
    _, err = clusterHandler.GetResourcesHandler().PrepareOCMRole(constants.OCMRolePreifx, true, "/test/")
    _, err = clusterHandler.GetResourcesHandler().PrepareUserRole(constants.UserRolePreifx, "/test/")
    
    // 3. Create cluster (triggers ALL AWS resource creation)
    err = clusterHandler.CreateCluster(config.Test.GlobalENV.WaitSetupClusterReady)
    
    clusterID := clusterHandler.GetClusterDetail().ClusterID
})
```

### Test 2: "to wait for cluster ready" (Lines 44-134) - WAITS FOR CLUSTER

```go
It("to wait for cluster ready", labels.Runtime.Day1Readiness, func() {
    // Load existing cluster from filesystem (created by Test 1)
    clusterHandler, err := handler.NewClusterHandlerFromFilesystem(client, profile)
    
    // Wait for cluster to be ready
    err = clusterHandler.WaitForClusterReady(config.Test.GlobalENV.ClusterWaitingTime)
    
    // Post-configuration: proxy settings, Cilium, etc.
})
```

---

## Resource Creation Flow

### Phase 1: IAM Roles Preparation

Located in: `tests/utils/handler/resources_handler_prepare.go`

#### 1.1 PrepareOCMRole (Line 488)
**Purpose**: Creates AWS IAM role for OpenShift Cluster Manager

**Process**:
1. Get account info via `rosa whoami`
2. Determine environment (stage/int/prod) for role naming
3. Check for existing linked OCM roles
4. Unlink stale roles that no longer exist in AWS
5. Create OCM role via `rosa create ocm-role --mode auto`

**AWS Resources Created**:
- IAM Role: `{prefix}-{env}-Admin-Role` (e.g., `rosacli-ocm-role-stage-Admin-Role`)
- IAM Policies attached to the role
- Trust relationship with OCM service

#### 1.2 PrepareUserRole (Line 583)
**Purpose**: Creates AWS IAM role for the user account

**Process**:
1. Get account info via `rosa whoami`
2. Determine environment for role naming
3. Check for existing linked user roles
4. Unlink stale roles
5. Create user role via `rosa create user-role --mode auto`

**AWS Resources Created**:
- IAM Role: `{prefix}-{env}-User-{username}-Role` (e.g., `rosacli-user-role-stage-User-elveeram-Role`)
- IAM Policies attached to the role
- Trust relationship for user access

---

### Phase 2: Cluster Creation

Entry: `clusterHandler.CreateCluster()` → `tests/utils/handler/cluster_handler.go:1189`

Flow:
```
CreateCluster()
  └─> createClusterByProfileWithoutWaiting() [Line 1135]
       ├─> GenerateClusterCreateFlags() [Line 197]
       │    └─> Prepares ALL AWS resources before cluster creation
       └─> clusterService.Create() [Line 1143]
            └─> Executes: rosa create cluster {flags}
```

---

### Phase 3: GenerateClusterCreateFlags - Detailed Resource Creation

Located in: `tests/utils/handler/cluster_handler.go:197`

This is where **ALL AWS resources** are prepared before the `rosa create cluster` command runs.

#### 3.1 Cluster Naming (Lines 199-206)
- Generates unique cluster name from `NAME_PREFIX` environment variable
- Default prefix: `rosacli-ci` (if NAME_PREFIX not set)
- Name length: Configurable via profile (default: 15 chars)

#### 3.2 Version Preparation (Lines 227-246)
- Resolves OpenShift version based on profile
- Uses `PrepareVersion()` to find matching version
- Supports: `latest`, `4.x.y`, `y-1`, `z-1` version patterns

#### 3.3 STS Account Roles (Lines 282-393)
**Function**: `PrepareAccountRoles()` - Line 675 in `resources_handler_prepare.go`

**AWS Resources Created**:
- **Installer Role**: `{prefix}-Installer-Role`
- **Support Role**: `{prefix}-Support-Role`
- **Worker Role**: `{prefix}-Worker-Role`
- **Control Plane Role**: `{prefix}-ControlPlane-Role` (Classic only)

**Command**: `rosa create account-roles --mode auto`

**For HCP with Shared VPC**:
- Also creates shared VPC IAM roles with Route53 and VPC Endpoint permissions

#### 3.4 OIDC Configuration (Lines 361-393)
**When**: If `oidc_config: "managed"` or `"unmanaged"` in profile

**Process**:
1. `PrepareOIDCConfig()` - Creates OIDC config
2. `PrepareOIDCProvider()` - Creates OIDC provider in AWS
3. `PrepareOperatorRolesByOIDCConfig()` - Creates operator roles

**AWS Resources Created**:
- **OIDC Provider**: IAM OIDC identity provider
- **Operator Roles** (per OpenShift operator):
  - `{prefix}-openshift-ingress-operator-cloud-credentials`
  - `{prefix}-openshift-cluster-csi-drivers-ebs-cloud-credentials`
  - `{prefix}-openshift-machine-api-aws-cloud-credentials`
  - `{prefix}-openshift-cloud-credential-operator-cloud-credential-operator-iam-ro-creds`
  - `{prefix}-openshift-image-registry-installer-cloud-credentials`
  - And more (varies by cluster type)

#### 3.5 Shared VPC Resources (Lines 398-430)
**When**: If `shared_vpc: true` in profile

**AWS Resources Created**:
- **Shared VPC Role**: For cross-account VPC access
- **Route53 Role** (HCP): `{prefix}-shared-route53-role`
- **VPC Endpoint Role** (HCP): `{prefix}-shared-vpcendpoint-role`

#### 3.6 Audit Log Forwarding (Lines 432-441)
**When**: If `auditlog_forward: true` in profile

**AWS Resources Created**:
- **IAM Role**: `{prefix}-AuditLog-Role` with permissions to write to CloudWatch

#### 3.7 Additional Principals (Lines 446-467)
**When**: If `additional_principals: true` in profile

**AWS Resources Created**:
- **IAM Role**: `{prefix}-additional-principal-role` for additional AWS principals

#### 3.8 VPC and Networking (Lines 603-760)
**When**: If `byo_vpc: true` in profile

**Function**: `PrepareVPC()` - Line 227 in `resources_handler_prepare.go`

**AWS Resources Created**:

**VPC Resources**:
- **VPC**: CIDR block (default: 10.0.0.0/16)
- **Internet Gateway**
- **NAT Gateways**: One per AZ for multi-AZ clusters
- **Route Tables**: Public and private route tables
- **Subnets**:
  - Public subnets (one per AZ)
  - Private subnets (one per AZ)
- **VPC Tags**: Includes cluster ownership tags

**Function**: `PrepareSubnets()` - Prepares subnet IDs for cluster

**Security Groups** (Lines 658-686):
**Function**: `PrepareAdditionalSecurityGroups()`

**AWS Resources Created**:
- **Security Groups**: Number specified by `additional_sg_number` (e.g., 3)
  - Compute security groups
  - Infrastructure security groups (Classic)
  - Control plane security groups (Classic)

**Shared VPC Resources** (Lines 688-760):
When `shared_vpc: true`:
- **Resource Share**: AWS RAM (Resource Access Manager) share
- **DNS Domain**: Route53 hosted zone
- **Hosted Zones**:
  - Ingress private hosted zone
  - HCP internal communication hosted zone (HCP only)
- **Subnet ARNs**: For resource sharing
- **Security Group ARNs**: For resource sharing

#### 3.9 Proxy Configuration (Lines 762-803)
**When**: If `proxy_enabled: true` in profile

**Function**: `PrepareProxy()` or `PrepareProxyWithAuth()`

**AWS Resources Created**:
- **EC2 Instance**: Squid proxy server
- **Security Groups**: For proxy access
- **CA Bundle**: Proxy certificate authority bundle file
- Optionally: **Secrets Manager Secret** for proxy auth credentials

#### 3.10 KMS Encryption (Lines 819-874)
**When**: If `kms_key: true` or `etcd_kms: true` in profile

**Function**: `PrepareKMSKey()` - Line 425 in `resources_handler_prepare.go`

**AWS Resources Created**:
- **KMS Key**: For EBS volume encryption
- **KMS Key Alias**: `rosacli-{random}`
- **KMS Key Policy**: Grants for cluster roles
- If `etcd_kms: true`:
  - Additional KMS key for etcd encryption

#### 3.11 Log Forwarding Resources (Lines 961-1004)
**When**: If `log_forward: true` in profile

**Functions**:
- `PrepareS3ForLogForward()`
- `PrepareCWLogGroup()`
- `PrepareLogForwardRole()`

**AWS Resources Created**:
- **S3 Bucket**: `rosaclis3bucket-{random}` for log storage
- **S3 Bucket Policy**: Allowing cluster to write logs
- **CloudWatch Log Group**: `rosaclicloudwatch{random}`
- **IAM Role**: `log_forward` role with permissions for both S3 and CloudWatch

---

### Phase 4: Cluster Creation Command

**Location**: `tests/utils/handler/cluster_handler.go:1143`

```go
clusterService.Create(ch.profile.ClusterConfig.Name, flags...)
```

This executes: `rosa create cluster {cluster-name} {all-flags}`

**What Happens**:
The `rosa create cluster` command triggers the OpenShift installer and OCM (OpenShift Cluster Manager) to create:

1. **Control Plane Resources**:
   - Control plane EC2 instances (Classic) or managed control plane (HCP)
   - Control plane security groups
   - Control plane load balancers

2. **Worker Node Resources**:
   - Worker EC2 instances based on `instance_type` and `replicas`
   - Auto Scaling Groups (if autoscaling enabled)
   - Worker node security groups

3. **Load Balancers**:
   - API load balancer (internal/external based on private flag)
   - Default ingress load balancer
   - PrivateLink endpoints (if `private_link: true`)

4. **Storage**:
   - EBS volumes for worker nodes
   - Encrypted with KMS key (if configured)

5. **Networking**:
   - ENIs (Elastic Network Interfaces)
   - Security group rules
   - Route53 DNS records

6. **OpenShift Operator Resources**:
   - Via operator roles created earlier
   - Each operator provisions its AWS resources as needed

---

### Phase 5: Post-Creation Steps

**Location**: `tests/utils/handler/cluster_handler.go:1164-1186`

After cluster creation, if certain conditions are met:

#### 5.1 OIDC Provider by Cluster (Lines 1164-1173)
**When**: STS enabled but no OIDC config specified in profile

**Resources**:
- Creates OIDC provider using cluster's OIDC endpoint
- Creates operator roles tied to the cluster

#### 5.2 KMS Key Elaboration (Lines 1174-1186)
**When**: KMS enabled with STS

**Action**:
- Updates KMS key policy to grant cluster operator roles access
- Function: `elaborateKMSKeyForSTSCluster()`

---

## Complete AWS Resources Summary

For a **typical HCP cluster with advanced features** (like `rosa-hcp-advanced` profile), the following AWS resources are created:

### IAM Resources
- 1 OCM Role
- 1 User Role
- 4 Account Roles (Installer, Support, Worker, ControlPlane for Classic)
- 10-15 Operator Roles (varies by cluster type)
- 1 Audit Log Role (if enabled)
- 1 Additional Principal Role (if enabled)
- 1 Log Forward Role (if enabled)
- 1-2 Shared VPC Roles (if Shared VPC enabled)
- 1 OIDC Provider

### Networking Resources
- 1 VPC
- 1 Internet Gateway
- 2-3 NAT Gateways (one per AZ)
- 4-6 Route Tables (public and private per AZ)
- 4-6 Subnets (public and private per AZ)
- 3-10 Security Groups (cluster + additional SGs)
- 1-3 Route53 Hosted Zones (if Shared VPC)
- 1 Resource Share (if Shared VPC)

### Encryption Resources
- 1-2 KMS Keys (one for EBS, optionally one for etcd)
- KMS Key Aliases
- KMS Key Policies

### Compute Resources
- 1 Proxy EC2 Instance (if proxy enabled)
- Control Plane Instances/Managed Service (HCP)
- 3-6 Worker EC2 Instances
- Auto Scaling Groups (if autoscaling)

### Storage Resources
- EBS Volumes (for worker nodes)
- 1 S3 Bucket (if log forwarding)

### Logging Resources
- 1 CloudWatch Log Group (if log forwarding)

### Load Balancers
- 1-2 Network Load Balancers (API and Ingress)

### DNS Resources
- Route53 DNS Records
- Route53 Hosted Zones (if Shared VPC)

---

## Configuration Sources

### Profile Files
Located in: `tests/ci/data/profiles/`

- `rosa-hcp.yaml` - HCP cluster profiles
- `rosa-classic.yaml` - Classic cluster profiles
- `external.yaml` - External team profiles

### Environment Variables

Override profile settings:
- `TEST_PROFILE` - Profile name (required)
- `NAME_PREFIX` - Cluster name prefix (default: `rosacli-ci`)
- `REGION` - AWS region (default: from profile)
- `VERSION` - OpenShift version (default: from profile)
- `CHANNEL_GROUP` - Channel group (default: from profile)
- `PROVISION_SHARD` - Provision shard ID
- `USE_LOCAL_CREDENTIALS` - Use local AWS credentials
- `CLUSTER_TIMEOUT` - Cluster creation timeout
- `BYOVPC` - Override BYO VPC setting

---

## Execution Flow Diagram

```
e2e_setup_test.go
    │
    ├─> Test 1: "by profile"
    │   │
    │   ├─> LoadProfileYamlFileByENV()
    │   │   └─> Loads profile from TEST_PROFILE env var
    │   │
    │   ├─> PrepareOCMRole()
    │   │   └─> Creates IAM role for OCM
    │   │
    │   ├─> PrepareUserRole()
    │   │   └─> Creates IAM role for user
    │   │
    │   └─> CreateCluster()
    │       │
    │       └─> createClusterByProfileWithoutWaiting()
    │           │
    │           ├─> GenerateClusterCreateFlags()
    │           │   │
    │           │   ├─> PrepareVersion()
    │           │   ├─> PrepareAccountRoles()
    │           │   │   └─> Creates 4 account IAM roles
    │           │   │
    │           │   ├─> PrepareOIDCConfig()
    │           │   │   └─> Creates OIDC configuration
    │           │   │
    │           │   ├─> PrepareOIDCProvider()
    │           │   │   └─> Creates IAM OIDC provider
    │           │   │
    │           │   ├─> PrepareOperatorRolesByOIDCConfig()
    │           │   │   └─> Creates 10-15 operator IAM roles
    │           │   │
    │           │   ├─> PrepareVPC()
    │           │   │   └─> Creates VPC, IGW, NAT, Route Tables, Subnets
    │           │   │
    │           │   ├─> PrepareSubnets()
    │           │   │   └─> Prepares subnet IDs
    │           │   │
    │           │   ├─> PrepareAdditionalSecurityGroups()
    │           │   │   └─> Creates additional security groups
    │           │   │
    │           │   ├─> PrepareProxy()
    │           │   │   └─> Creates proxy EC2 instance
    │           │   │
    │           │   ├─> PrepareKMSKey()
    │           │   │   └─> Creates KMS keys for encryption
    │           │   │
    │           │   ├─> PrepareS3ForLogForward()
    │           │   │   └─> Creates S3 bucket
    │           │   │
    │           │   ├─> PrepareCWLogGroup()
    │           │   │   └─> Creates CloudWatch log group
    │           │   │
    │           │   └─> PrepareLogForwardRole()
    │           │       └─> Creates log forwarding IAM role
    │           │
    │           └─> clusterService.Create()
    │               └─> Executes: rosa create cluster {flags}
    │                   └─> Creates cluster, workers, LBs, etc.
    │
    └─> Test 2: "to wait for cluster ready"
        │
        ├─> NewClusterHandlerFromFilesystem()
        │   └─> Loads cluster info from file
        │
        └─> WaitForClusterReady()
            └─> Polls cluster status until ready
```

---

## Key Files Reference

- **Test Entry**: `tests/e2e/e2e_setup_test.go`
- **Cluster Handler**: `tests/utils/handler/cluster_handler.go`
- **Resource Preparation**: `tests/utils/handler/resources_handler_prepare.go`
- **Profile Interface**: `tests/utils/handler/interface.go`
- **Profile Loading**: `tests/utils/handler/profile.go`
- **Profiles**: `tests/ci/data/profiles/*.yaml`
- **Constants**: `tests/utils/constants/*.go`

---

## Notes

1. **Auto Mode**: Most AWS resource creation uses `--mode auto` flag, which automatically creates resources without prompting.

2. **Resource Cleanup**: All resources are tracked in a resources file and cleaned up during test teardown.

3. **Idempotency**: The code checks for existing resources and reuses them when possible (e.g., linked OCM roles).

4. **Multi-AZ Support**: When `multi_az: true`, resources are created across multiple availability zones.

5. **HCP vs Classic**: Different resources are created based on cluster type (e.g., HCP doesn't create control plane IAM role).

6. **Shared VPC**: Additional cross-account resources and RAM shares are created when `shared_vpc: true`.

7. **Error Handling**: If any resource creation fails, the test panics and cleanup is triggered.

8. **Resource Naming**: All resources use the cluster name prefix to ensure uniqueness and traceability.