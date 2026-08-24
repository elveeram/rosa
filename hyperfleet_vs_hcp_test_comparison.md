# Comparison: hyperfleet_sanity_test.go vs hcp_cluster_test.go

## Executive Summary

While both tests deal with HCP (Hosted Control Plane) clusters, they serve fundamentally different purposes and operate through different APIs. The `hyperfleet_sanity_test.go` is a **complete end-to-end lifecycle test** for the Platform API (Hyperfleet), while `hcp_cluster_test.go` contains **Day2 operational tests** for existing HCP clusters via the OCM API.

## Key Architectural Differences

### API Layer

**hyperfleet_sanity_test.go:**
- Uses the **Platform API (Hyperfleet API)** directly
- Imports `github.com/openshift-online/rosa-hyperfleet-api/clientset`
- Authenticates via `rosa login --hyperfleet-url`
- Tests the new Platform API path for cluster management
- Bypasses OCM entirely

**hcp_cluster_test.go:**
- Uses the **OCM (OpenShift Cluster Manager) API** 
- Works through standard rosa CLI commands
- Expects `CLUSTER_ID` environment variable (pre-existing cluster)
- Tests traditional OCM-based cluster operations
- No direct API client imports (uses rosa CLI wrapper)

### Test Scope

**hyperfleet_sanity_test.go:**
- **Full lifecycle test**: Creates cluster → validates → deletes cluster
- **Infrastructure-from-scratch**: Provisions all AWS resources (VPC, subnets, NAT gateway, route tables, security groups, Route53 hosted zone)
- **Manual IAM setup**: Creates OIDC provider, operator roles, worker instance profiles
- **Node pool lifecycle**: Creates, waits for ready state, and deletes node pools
- **Comprehensive cleanup**: Tears down all AWS resources in reverse dependency order
- Single focused test case: "creates and deletes an HCP cluster via the Platform API"

**hcp_cluster_test.go:**
- **Day2 operations testing**: Assumes cluster already exists
- **Feature validation**: Tests log forwarding, audit logs, KMS encryption, external auth, registry config, network types, additional principals, autonode configuration
- **No infrastructure management**: Works with pre-provisioned clusters
- **Multiple test cases**: 10+ separate test scenarios for different cluster features
- **Limited cleanup**: Only cleans up test-specific resources (e.g., temp config files)
- One exception: "create cluster withskip_inflight property" test creates a cluster but still uses OCM path

## Detailed Comparison Matrix

| Aspect | hyperfleet_sanity_test.go | hcp_cluster_test.go |
|--------|---------------------------|---------------------|
| **Primary Purpose** | Validate Platform API cluster lifecycle | Test HCP cluster Day2 features |
| **API Used** | Platform API (Hyperfleet) | OCM API |
| **Authentication** | `rosa login --hyperfleet-url <URL>` | Standard OCM login (implicit) |
| **Cluster Creation** | Yes, via Platform API SDK | No (except one test via rosa CLI) |
| **Cluster Deletion** | Yes, with full AWS cleanup | No |
| **AWS Setup** | Complete (VPC, subnets, NAT, IGW, security groups, Route53) | None (assumes existing infrastructure) |
| **IAM/OIDC Setup** | Manual (creates OIDC provider, operator roles, worker roles) | None (assumes pre-configured) |
| **Test Duration** | Long (90+ min cluster ready timeout, 30 min node pool timeout) | Variable (mostly describe/edit operations) |
| **Dependencies** | Hyperfleet SDK, AWS SDK v2, rosa CLI | rosa CLI, test utilities/helpers |
| **Environment Variables** | `HYPERFLEET_URL`, `CLUSTER_NAME`, `OPERATOR_ROLES_PREFIX`, `HYPERFLEET_VERSION`, `HYPERFLEET_INSTANCE_TYPE` | `CLUSTER_ID` (required), various feature flags |
| **Cleanup Strategy** | DeferCleanup with bounded grace period (90+ min), resource-by-resource teardown | Basic cleanup via `rosaClient.CleanResources(clusterID)` |
| **Node Pool Testing** | Full lifecycle (create → wait ready → delete) for 2 node pools | None (uses machinePoolService for listing only) |
| **Validation Strategy** | State-based (wait for phases: Ready, deleted) | Feature-based (describe/edit/assert values) |
| **AWS Resource Management** | Extensive (VPC, subnets, NAT, ELB, ENI, security groups, Route53, IAM) | None |

## Code Structure Differences

### hyperfleet_sanity_test.go

**Structure:**
1. Environment setup and region derivation
2. AWS configuration and identity resolution
3. Hyperfleet client construction
4. Network infrastructure provisioning (200+ lines)
5. IAM role and OIDC provider creation
6. Cluster creation via rosa CLI (`--hyperfleet-url`)
7. Cluster readiness validation via SDK
8. Node pool lifecycle testing
9. Comprehensive cleanup with dependency ordering

**Key patterns:**
- Uses `deferTeardown` wrapper for all cleanup with extended grace period (`teardownGracePeriod = hfClusterReadyTimeout + 30*time.Minute`)
- Direct AWS SDK calls (EC2, IAM, Route53, ELB)
- Manual OIDC trust policy construction
- Resource tagging with cluster name for tracking
- Explicit wait loops with polling intervals

### hcp_cluster_test.go

**Structure:**
1. BeforeEach: Load cluster config, verify HCP type
2. Multiple independent test cases (It blocks)
3. AfterEach: Basic cleanup via `rosaClient.CleanResources()`

**Key patterns:**
- Uses test helper frameworks (`handler`, `config` packages)
- Service-oriented approach (`clusterService`, `logForwarderService`, `machinePoolService`)
- JSON parsing and reflection for validation
- Profile-based test configuration
- Label-based test categorization (`labels.Critical`, `labels.High`, `labels.Runtime.Day2`)

## Use Cases

### When to use hyperfleet_sanity_test.go pattern:
- Testing Platform API functionality
- Validating complete cluster lifecycle via Hyperfleet
- Verifying Platform API regional deployments
- Testing Platform API authentication and authorization
- Validating Hyperfleet SDK integration
- CI/CD pipelines for Platform API releases

### When to use hcp_cluster_test.go pattern:
- Testing cluster features and Day2 operations
- Validating rosa CLI commands on existing clusters
- Feature flag validation
- Regression testing for cluster configuration changes
- Customer scenario validation (e.g., audit log, external auth)
- Integration testing of cluster services (log forwarding, autoscaling)

## Technical Deep Dive

### Authentication Flow

**hyperfleet_sanity_test.go:**
```go
// Logs into Platform API directly
rosacli.NewClient().Runner.
    Cmd("login").
    CmdFlags("--hyperfleet-url", hfURL).
    Run()

// Verifies Platform API URL stored in config
whoamiMap["Platform API"] == hfURL  // Assertion
```

**hcp_cluster_test.go:**
```go
// Assumes standard OCM login already completed
// No explicit login in test code
// Uses CLUSTER_ID from environment
clusterID = config.GetClusterID()
```

### Cluster Creation

**hyperfleet_sanity_test.go:**
```go
// Creates cluster via CLI but expects Hyperfleet backend
rosacli.NewClient().Runner.
    Cmd("create", "cluster").
    CmdFlags(
        "--cluster-name", clusterName,
        "--subnet-ids", subnetID,
        "--operator-roles-prefix", rolesPrefix,
        "--version", version,  // optional
    ).
    Run()

// Then uses Hyperfleet SDK to wait for ready state
hfClient.HyperfleetV1alpha1().Clusters().WaitUntil(
    ctx, clusterID,
    func(c *v1alpha1.Cluster) bool {
        return c.Status.Phase == v1alpha1.ClusterPhaseReady
    },
    hfClusterReadyInterval, hfClusterReadyTimeout,
)
```

**hcp_cluster_test.go (one test that creates):**
```go
// Uses standard OCM path
command = "rosa create cluster --cluster-name " + name + " " + flags
rosalCommand.AddFlags("--properties", "skip_inflight_tests:true")
rosaClient.Runner.RunCMD(strings.Split(rosalCommand.GetFullCommand(), " "))
```

### Cleanup Complexity

**hyperfleet_sanity_test.go cleanup order (LIFO via DeferCleanup):**
1. VPC deletion (waited)
2. Subnet deletions (public & private)
3. Internet Gateway detachment & deletion
4. NAT Gateway EIP release
5. NAT Gateway deletion (waited)
6. Route table disassociations & deletions
7. Security group deletions
8. Route53 hosted zone record purge & deletion
9. OIDC provider deletion
10. Operator role policy detachments & deletions
11. Worker instance profile & role deletion
12. **Cluster deletion** (waited for full termination)
13. Worker EC2 instances termination wait
14. Classic load balancers deletion (waited)
15. ALB/NLB deletion (waited)
16. Orphaned ENI cleanup
17. Non-default security groups cleanup

This ordering ensures:
- IAM resources remain available during cluster deletion (operators need OIDC)
- Network resources are released after all compute resources terminate
- Dependencies are respected (e.g., VPC deleted last)

**hcp_cluster_test.go cleanup:**
```go
// Single line cleanup
rosaClient.CleanResources(clusterID)
```

### AWS Infrastructure Provisioning

**hyperfleet_sanity_test.go creates:**
- VPC (10.0.0.0/16)
- Private subnet (10.0.0.0/19)
- Public subnet (10.0.101.0/24)
- Internet Gateway (attached to VPC)
- NAT Gateway (in public subnet with EIP)
- Public route table (IGW route)
- Private route table (NAT route)
- Worker security group (with VPC CIDR ingress)
- Route53 private hosted zone (`<cluster>.hypershift.local`)

**hcp_cluster_test.go:**
- No infrastructure provisioning

### Node Pool Testing

**hyperfleet_sanity_test.go:**
```go
// Creates 2 node pools
np1: 2 replicas, m5.xlarge (or env override)
np2: 1 replica, m5.xlarge (or env override)

// Waits for both to reach Ready phase
nodePools.WaitUntil(ctx, np1ID, readyCheck, 30s interval, 30min timeout)
nodePools.WaitUntil(ctx, np2ID, readyCheck, 30s interval, 30min timeout)

// Deletes np2 and waits for deletion
rosacli delete machinepool np2
nodePools.WaitUntil(ctx, np2ID, deletedCheck, ...)

// Initiates np1 deletion (PDB prevents completion until cluster delete)
```

**hcp_cluster_test.go:**
```go
// Only lists node pools for availability zone validation
npList, err := machinePoolService.ListAndReflectNodePools(clusterID)
// Checks if SingleAZ or MultiAZ based on count
```

## Error Handling and Resilience

### hyperfleet_sanity_test.go

**Interrupt handling:**
- Custom `teardownGracePeriod` (90+ minutes) to prevent orphaned resources on Ctrl-C
- Each cleanup receives fresh `SpecContext` that cancels when grace period expires
- Cleanup continues even if mid-run interrupted (Ginkgo default is 30s)

**Wait strategies:**
- Explicit AWS waiters (e.g., `ec2svc.NewVpcAvailableWaiter`)
- Custom polling loops for Hyperfleet resources
- Timeout handling with context cancellation
- Warning messages instead of failures for cleanup timeouts (don't mask test results)

### hcp_cluster_test.go

**Error handling:**
- Standard Gomega assertions (`Expect(err).To(BeNil())`)
- Skip patterns for optional features (`SkipNotHosted()`, `SkipTestOnFeature()`)
- Deferred cleanup with basic error ignoring (`defer func() { ... }()`)

## Environment Requirements

### hyperfleet_sanity_test.go

**Required:**
- `HYPERFLEET_URL`: Platform API endpoint (e.g., `https://api.hypershift.us-east-1.aws.redhat.com`)
- AWS credentials with full EC2, IAM, Route53, ELB permissions
- `AWS_DEFAULT_REGION` (derived from URL if not set)

**Optional:**
- `CLUSTER_NAME` (auto-generated if not provided)
- `OPERATOR_ROLES_PREFIX` (defaults to cluster name)
- `HYPERFLEET_VERSION` (server resolves default if not provided)
- `HYPERFLEET_INSTANCE_TYPE` (defaults to m5.xlarge)

### hcp_cluster_test.go

**Required:**
- `CLUSTER_ID`: Existing HCP cluster ID
- Various profile/config files in test infrastructure
- OCM API authentication (pre-configured)

**Optional (feature-dependent):**
- `AUDIT_LOG_ARN` for audit log tests
- `ENCRYPTION` config for KMS tests
- Various feature flags in cluster profile

## Performance Characteristics

### hyperfleet_sanity_test.go

**Estimated runtime:** 90-120 minutes
- VPC/network setup: ~5 minutes
- IAM/OIDC setup: ~2 minutes
- Cluster creation wait: 60-90 minutes (hfClusterReadyTimeout)
- Node pool 1 ready: up to 30 minutes
- Node pool 2 ready: up to 30 minutes  
- Node pool 2 deletion: ~10 minutes
- Cluster deletion wait: up to 90 minutes
- AWS resource cleanup: ~15 minutes

**Resource usage:**
- ~20 AWS resource types created
- 2 EC2 instances (worker nodes)
- 1 NAT Gateway (ongoing cost during test)
- Multiple API calls to AWS and Platform API

### hcp_cluster_test.go

**Estimated runtime:** Variable by test
- Log forwarder test: ~5 minutes
- Audit log test: ~2 minutes
- KMS/encryption test: <1 minute (describe only)
- Registry config edit: ~3 minutes
- Most tests: <5 minutes (describe/edit operations)

**Resource usage:**
- No infrastructure created
- Minimal AWS API calls (via rosa CLI)
- Temporary files for config (cleaned up)

## Testing Philosophy

### hyperfleet_sanity_test.go: **Smoke Test**
- Purpose: "Does the Platform API work at all?"
- Validates end-to-end happy path
- Not exhaustive feature testing
- Focus on core lifecycle: create → ready → delete
- Designed to catch Platform API regressions

### hcp_cluster_test.go: **Feature Matrix**
- Purpose: "Do all HCP features work correctly?"
- Validates specific cluster capabilities
- Comprehensive feature coverage
- Focus on configuration and Day2 ops
- Designed to catch feature regressions

## Conclusion

These tests are **complementary, not duplicative**. They test different aspects of the ROSA HCP ecosystem:

- **hyperfleet_sanity_test.go** validates the **Platform API infrastructure** and basic cluster lifecycle
- **hcp_cluster_test.go** validates **HCP cluster features** and OCM API operations

The Platform API (Hyperfleet) represents a new architectural path for ROSA, allowing cluster creation without going through OCM. The sanity test ensures this new path works correctly, while the HCP cluster test validates that regardless of creation method (OCM or Platform API), the cluster features behave correctly.

### Recommendation

If you need to:
- **Test Platform API functionality** → Use hyperfleet_sanity_test.go pattern
- **Test cluster features on existing clusters** → Use hcp_cluster_test.go pattern
- **Test full cluster lifecycle via OCM** → Need a different test (not covered by either)
- **Validate Hyperfleet regional deployment** → Use hyperfleet_sanity_test.go
- **Test rosa CLI Day2 operations** → Use hcp_cluster_test.go pattern