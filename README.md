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
policies/               - Reference copy of the raw Policy YAMLs (not synced)
policy-generator/       - PolicyGenerator config that emits the same Policies via the ACM kustomize plugin
```

`namespaces/`, `networks/`, `virtual-machines/`, and `policy-generator/` each carry a `kustomization.yaml` so ArgoCD can sync them. `policies/` is intentionally not GitOps-managed — `policy-generator/` is the source of truth.

## Deployment

This repo is reconciled by OpenShift GitOps (ArgoCD). The bootstrap (operators, network patch, ArgoCD Applications) lives outside this repo.

### Bootstrap (one-shot, out-of-repo)

1. Install **OpenShift GitOps operator**, **Kubernetes NMState operator**, **OpenShift Virtualization (HyperConverged)**, and ensure ACM is installed.
2. Enable MultiNetworkPolicy:
   ```bash
   oc patch network.operator.openshift.io cluster --type=merge \
     -p '{"spec":{"useMultiNetworkPolicy":true}}'
   ```
3. Patch the `openshift-gitops` ArgoCD CR to add the ACM **PolicyGenerator** kustomize-plugin sidecar (init container that drops the `PolicyGenerator` binary into `KUSTOMIZE_PLUGIN_HOME`). Without this, ArgoCD's repo-server cannot evaluate `policy-generator/kustomization.yaml`.
4. Create an ArgoCD `Application` per synced directory (`namespaces/`, `networks/`, `virtual-machines/`, `policy-generator/`) pointing at this repo.

### GitOps reconciles the rest

ArgoCD applies, in order via sync waves:

1. `namespaces/` — demo namespaces with the labels/annotations described above
2. `networks/` — NNCPs (OVS bridges) and NADs
3. `virtual-machines/` — KubeVirt VMs on the secondary network
4. `policy-generator/` — runs PolicyGenerator, which emits `Policy`, `Placement`, and `PlacementBinding` resources in `open-cluster-management-global-set`

Once placed, ACM evaluates the policies and writes `MultiNetworkPolicy` resources into each demo namespace.
