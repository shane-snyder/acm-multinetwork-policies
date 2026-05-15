# ACM MultiNetworkPolicy Governance

Uses Red Hat Advanced Cluster Management (ACM) governance to dynamically generate `MultiNetworkPolicy` resources from labels and annotations on namespaces and VirtualMachines.

## Policies

A namespace opts in by setting `multinetwork-policy.custom.io/enabled: "true"`. Inside an opted-in namespace, three policies run independently and emit MultiNetworkPolicies (MNPs) based on what they find:

| Policy | Fires when | Emits |
|--------|------------|-------|
| `policy-deny-all-multinet` | Always (in opted-in namespaces) | One deny-all MNP per NAD in the namespace (blocks all ingress + egress) |
| `policy-allow-ns-ingress` | The namespace has `network.custom.io/ingress-ports` and/or `egress-ports` annotations | One allow MNP per NAD in the namespace, applied to all pods, opening the listed TCP ports |
| `policy-allow-vm-ingress` | A VM in the namespace has `network.custom.io/ingress-ports` and/or `egress-ports` annotations | For that VM, iterates `spec.template.spec.networks` and emits a per-VM allow MNP (via `podSelector: vm.kubevirt.io/name=<vm>`) per attached NAD, opening the listed TCP ports |

Namespace-level and VM-level allow policies can coexist in the same namespace — they target different `podSelector`s, so they layer additively.

## Labels and annotations

### Namespace labels

| Label | Values | Purpose |
|-------|--------|---------|
| `multinetwork-policy.custom.io/enabled` | `"true"` | Master gate — when set, this namespace participates in all three policies |

### Namespace annotations

| Annotation | Example | Purpose |
|------------|---------|---------|
| `network.custom.io/ingress-ports` | `"22,53/udp,443"` | Ports to allow inbound for all pods, across every NAD in the namespace |
| `network.custom.io/egress-ports` | `"22,53/udp,443"` | Ports to allow outbound for all pods, across every NAD in the namespace |

### VM annotations

| Annotation | Example | Purpose |
|------------|---------|---------|
| `network.custom.io/ingress-ports` | `"22,53/udp,443"` | Ports to allow inbound to this VM, on each of its attached secondary networks |
| `network.custom.io/egress-ports` | `"22,53/udp,443"` | Ports to allow outbound from this VM, on each of its attached secondary networks |

A VM can have both annotations (server that sends and receives), only ingress (locked-down server), only egress (client-only), or neither (fully denied by the baseline deny-all).

### Port syntax

Each entry in an `*-ports` annotation is either `<port>` (defaults to TCP) or `<port>/<proto>`, where `<proto>` is `tcp`, `udp`, or `sctp` (case-insensitive). Examples:

| Annotation value | Generated rules |
|---|---|
| `"22,443"` | TCP 22, TCP 443 |
| `"22,53/udp"` | TCP 22, UDP 53 |
| `"123/udp,5060/sctp"` | UDP 123, SCTP 5060 |

## How MNPs bind to networks

Each generated MNP carries a `k8s.v1.cni.cncf.io/policy-for: <namespace>/<nad>` annotation that scopes it to one NetworkAttachmentDefinition. The templates discover the NAD automatically:

- The **namespace-level** allow ranges over every NAD in the namespace.
- The **VM-level** allow ranges over every `spec.template.spec.networks` entry on each VM that references a Multus `networkName`. A VM attached to multiple secondary networks gets one allow MNP per attached NAD.

## Adding a new namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-new-namespace
  labels:
    multinetwork-policy.custom.io/enabled: "true"
  annotations:
    # optional — drives the namespace-wide allow policy.
    # Omit to leave the namespace strictly deny-all (plus any per-VM allows).
    network.custom.io/ingress-ports: "22,53/udp,443"
    network.custom.io/egress-ports: "22,53/udp,443"
```

Then create one or more `NetworkAttachmentDefinition`s in that namespace, and (optionally) annotate individual VMs with their own `network.custom.io/ingress-ports` / `egress-ports`. ACM picks the new resources up on its next reconcile (~30s) and emits the corresponding MNPs.

## Adding a new VM

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: my-vm
  namespace: my-namespace
  annotations:
    # optional — drives the per-VM allow policy. Each entry opens that port
    # on every secondary NAD this VM attaches to (see networks below).
    # Defaults to TCP; append /udp or /sctp to override.
    # Omit either annotation to leave that direction strictly deny-all.
    network.custom.io/ingress-ports: "22,53/udp,443"
    network.custom.io/egress-ports: "22,53/udp,443"
spec:
  runStrategy: Always
  template:
    spec:
      domain:
        devices:
          interfaces:
            - name: default
              masquerade: {}
            - name: localnet
              bridge: {}
      networks:
        - name: default
          pod: {}
        - name: localnet
          multus:
            networkName: my-nad   # the NAD the allow policy will target
```

The VM-level allow policy reads the annotations and emits one MNP per `multus.networkName` entry under `spec.template.spec.networks` — so multi-homed VMs automatically get a policy per attached NAD.
