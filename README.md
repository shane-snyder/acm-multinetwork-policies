# ACM MultiNetworkPolicy Governance

Uses Red Hat Advanced Cluster Management (ACM) governance to dynamically create and manage `MultiNetworkPolicy` resources based on namespace and workload labels/annotations.

## Architecture

Three ACM policies work together to provide a flexible, label-driven multi-network security model:

| Policy | Scope | Behavior |
|--------|-------|----------|
| `policy-deny-all-multinet` | Namespaces with `multinetwork-policy.custom.io/enabled: "true"` | Creates a deny-all MultiNetworkPolicy per NAD (blocks all ingress + egress) |
| `policy-allow-ns-ingress` | Namespaces with `multinetwork-policy.custom.io/scope: namespace` | Reads `ingress-ports` and `egress-ports` annotations from the namespace, creates allow MultiNetworkPolicies for all pods |
| `policy-allow-vm-ingress` | Namespaces with `multinetwork-policy.custom.io/scope: workload` | Reads `ingress-ports` and `egress-ports` annotations from individual VMs, creates per-VM allow MultiNetworkPolicies with `podSelector` |

## Labels and Annotations

### Namespace Labels

| Label | Values | Purpose |
|-------|--------|---------|
| `env` | `dev`, `prod` | Determines which localnet network applies (`localnet-dev` or `localnet-prod`) |
| `multinetwork-policy.custom.io/enabled` | `"true"` | Enables deny-all policy for this namespace |
| `multinetwork-policy.custom.io/scope` | `namespace` or `workload` | Selects which allow policy strategy applies |

### Namespace Annotations (scope: namespace)

| Annotation | Example | Purpose |
|------------|---------|---------|
| `network.custom.io/ingress-ports` | `"22,443,8443"` | Comma-separated TCP ports to allow inbound traffic for all pods |
| `network.custom.io/egress-ports` | `"22,443,8443"` | Comma-separated TCP ports to allow outbound traffic for all pods |

### VM Annotations (scope: workload)

| Annotation | Example | Purpose |
|------------|---------|---------|
| `network.custom.io/ingress-ports` | `"22,443,8443"` | Comma-separated TCP ports to allow inbound traffic, scoped to this VM |
| `network.custom.io/egress-ports` | `"22,443,8443"` | Comma-separated TCP ports to allow outbound traffic, scoped to this VM |

A VM can have both annotations (server that sends and receives), only ingress (locked-down server), only egress (client-only), or neither (fully denied by the default deny policy).

## Adding New Namespaces

To add a new namespace to this system, apply the appropriate labels and annotations:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-new-namespace
  labels:
    env: dev
    multinetwork-policy.custom.io/enabled: "true"
    multinetwork-policy.custom.io/scope: namespace  # or "workload"
  annotations:
    network.custom.io/ingress-ports: "22,443,8443"  # only for scope: namespace
    network.custom.io/egress-ports: "22,443,8443"   # only for scope: namespace
```

Then create a `NetworkAttachmentDefinition` for the appropriate localnet in that namespace.

## Directory Structure

```
acm/                    - ACM operator (namespace, operator-group, subscription)
acm-instance/           - MultiClusterHub instance
namespaces/             - Demo namespace definitions with labels/annotations
networks/               - NNCPs (OVS bridges) and NADs (localnet per namespace)
virtual-machines/       - Test VMs (Fedora container disk, secondary network)
policies/               - ACM policy infrastructure and all three policies
```

## Deployment Order

1. `oc apply -f acm/` - Install ACM operator
2. Wait for operator CSV to succeed
3. `oc apply -f acm-instance/` - Create MultiClusterHub
4. Wait for MCH phase `Running` and `local-cluster` ManagedCluster
5. `oc apply -f namespaces/` - Create demo namespaces
6. `oc apply -f networks/` - Create bridges and NADs
7. `oc apply -f virtual-machines/` - Create test VMs
8. `oc apply -f policies/` - Deploy policy infrastructure and all policies

**Note:** `MultiNetworkPolicy` support must be enabled on the cluster:
```bash
oc patch network.operator.openshift.io cluster --type=merge \
  -p '{"spec":{"useMultiNetworkPolicy":true}}'
```
