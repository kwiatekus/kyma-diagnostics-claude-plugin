---
name: kyma-diagnose
description: Diagnose problems on SAP BTP Kyma instances. Runs kyma alpha diagnose commands via dev-toolbox Docker image and performs root cause analysis for administrators and support engineers.
argument-hint: "optional: module name or area to focus on (e.g. api-gateway, eventing)"
---

# Kyma Instance Diagnostics

You are a Kyma/Kubernetes expert helping SAP BTP Kyma administrators and support engineers diagnose problems on Kyma instances.

Follow the steps below exactly and in order.

---

## Step 1 — Check if kyma CLI is installed on the host

Run:
```bash
which kyma
```

- If `kyma` is found: set **MODE=local**. All diagnostic commands run directly on the host using the host's `KUBECONFIG`. Skip Step 1b and Step 2.
- If `kyma` is not found: set **MODE=docker**. Proceed through all steps including 1b and 2.

Record the mode and carry it through all subsequent steps.

---

## Step 1b — Verify KUBECONFIG (both modes)

Read the `KUBECONFIG` environment variable and confirm with the user before proceeding.

Run:
```bash
echo ${KUBECONFIG:-$HOME/.kube/config}
```

Print the path and ask:
> "I will use the KUBECONFIG at `<path>`. Is this the correct cluster? (yes/no)"

If the user says no, ask them to set `KUBECONFIG` correctly and stop.

If `KUBECONFIG` is empty, check whether `~/.kube/config` exists and ask the user to confirm that default config should be used.

---

## Step 1c — Check for interactive auth (MODE=docker only)

**Skip this step entirely if MODE=local.** The host kyma CLI can handle OIDC auth natively via the user's session.

When MODE=docker, inspect the KUBECONFIG for any `exec:` credential plugin that triggers browser-based OIDC login — this is incompatible with the headless Docker container environment.

Run:
```bash
grep -E "command:|oidc|exec" "${KUBECONFIG:-$HOME/.kube/config}"
yq '.users[].user.exec.command' "${KUBECONFIG:-$HOME/.kube/config}" 2>/dev/null
```

If an interactive plugin is detected (`kubelogin`, `kubectl-oidc_login`, `oidc-login`, `aws`, `gke-gcloud-auth-plugin`, or anything that opens a browser), **stop immediately** and inform the user:

> "Your KUBECONFIG uses an interactive credential plugin (`<command>`) that requires browser-based OIDC login. This will not work inside the headless Docker container used by this skill.
>
> **Option A — Install kyma CLI on your host** (recommended): if `kyma` is installed locally, re-run `/kyma-diagnose` and it will use your active OIDC session directly without Docker.
>
> **Option B — Generate a temporary headless kubeconfig** using the Kyma CLI (requires your current session to still be valid on the host):
> ```bash
> kyma alpha kubeconfig generate \
>   --serviceaccount temp-diagnoser \
>   --clusterrole cluster-admin \
>   --cluster-wide \
>   --output /tmp/diagnostics-kubeconfig-headless.yaml
> ```
> By default valid for **1 hour**. Use `--time` to adjust (e.g. `--time 30m`, `--time 2h`).
>
> Then set `export KUBECONFIG=/tmp/diagnostics-kubeconfig-headless.yaml` and re-run `/kyma-diagnose`."

If no interactive plugin is found, continue to the next step.

---

## Step 2 — Pull the latest dev-toolbox image (MODE=docker only)

**Skip this step if MODE=local.**

```bash
docker pull europe-docker.pkg.dev/kyma-project/prod/dev-toolbox:main
```

If the pull fails, report the error and stop — do not proceed with a potentially stale image.

Also resolve the kubeconfig path for volume mounting:
```bash
echo ${KUBECONFIG:-$HOME/.kube/config}
```
Use this as `<kubeconfig-path>` in all subsequent `docker run` commands.

---

## Step 3 — Run cluster diagnostics (single pass, save to file)

Run `kyma alpha diagnose cluster` **once** and save the full output to a temp file. Apply all yq filters against that file — never call the diagnose command more than once.

**MODE=local:**
```bash
DIAG_FILE="/tmp/kyma-diag-cluster-$$.yaml"
kyma alpha diagnose cluster --format yaml > "$DIAG_FILE" 2>&1
echo "exit:$? size:$(wc -c < "$DIAG_FILE") file:$DIAG_FILE"
```

**MODE=docker:**
```bash
DIAG_FILE="/tmp/kyma-diag-cluster-$$.yaml"
docker run --rm \
  -v <kubeconfig-path>:/root/.kube/config:ro \
  europe-docker.pkg.dev/kyma-project/prod/dev-toolbox:main \
  kyma alpha diagnose cluster --format yaml > "$DIAG_FILE" 2>&1
echo "exit:$? size:$(wc -c < "$DIAG_FILE") file:$DIAG_FILE"
```

If the command fails or the file is empty, report the error and stop.

Once the file is saved, extract all four sections in parallel:

```bash
yq '.metadata'          "$DIAG_FILE"
yq '.kymaModulesErrors' "$DIAG_FILE"
yq '.warnings'          "$DIAG_FILE"
yq '.nodes'             "$DIAG_FILE"
```

**From `.metadata`**, note:
- Cloud provider, region, Kubernetes version, Kyma version, Gardener extensions

**From `.kymaModulesErrors`**, for each module:
- Note `state` and failing `conditions` (`status: False`)
- `Error` or `Warning` state = degraded

**From `.warnings`**, scan for:
- Pod crash loops (`BackOff`, `OOMKilled`, `Error`)
- Failed scheduling (`Insufficient`, `Unschedulable`)
- Volume mount or image pull failures
- Node pressure (`MemoryPressure`, `DiskPressure`, `PIDPressure`)

**From `.nodes`**, for each node assess:
- **Conditions**: flag any condition other than `Ready=True` — especially `MemoryPressure`, `DiskPressure`, `PIDPressure`, or `NetworkUnavailable` set to `True`
- **Resource consumption**: compare `allocatable` vs actual usage; flag nodes where CPU or memory consumption exceeds ~80% of allocatable
- **Topology**: note the `zone` / availability zone of each node to identify AZ imbalance or a full AZ outage
- **Node roles**: distinguish control-plane vs worker nodes; a not-ready control-plane node is critical
- **Cordoned / unschedulable nodes**: a node with `unschedulable: true` reduces effective cluster capacity silently

---

## Step 4 — Collect Istio mesh diagnostics

**MODE=local:**
```bash
kyma alpha diagnose istio --format yaml
```

**MODE=docker:**
```bash
docker run --rm \
  -v <kubeconfig-path>:/root/.kube/config:ro \
  europe-docker.pkg.dev/kyma-project/prod/dev-toolbox:main \
  kyma alpha diagnose istio --format yaml
```

Analyze for:
- Sidecar injection failures or missing sidecars
- mTLS policy conflicts
- Invalid `VirtualService`, `DestinationRule`, or `Gateway` configurations
- Certificate expiry warnings
- Envoy proxy errors

---

## Step 5 — Extract module logs (targeted)

If Step 3 revealed degraded modules, extract their logs.

If `$ARGUMENTS` specifies a module name, use `--module <name>`. Otherwise run without `--module` for all modules.

**MODE=local:**
```bash
kyma alpha diagnose logs [--module <module-name>]
```

**MODE=docker:**
```bash
docker run --rm \
  -v <kubeconfig-path>:/root/.kube/config:ro \
  europe-docker.pkg.dev/kyma-project/prod/dev-toolbox:main \
  kyma alpha diagnose logs [--module <module-name>]
```

Focus on:
- Error-level log lines
- Stack traces or panics
- Reconciliation failures
- Repeated error patterns (high frequency = likely root cause)

---

## Step 6 — Root cause analysis

Synthesize all collected data and produce a structured report:

### Report Format

```
## Kyma Cluster Diagnostic Report

### Cluster Info
- Provider: <provider>
- Region: <region>
- Kyma version: <version>
- Kubernetes version: <k8s-version>
- Gardener extensions: <list>

### Overall Health: <HEALTHY | DEGRADED | CRITICAL>

### Degraded Components
For each degraded module/component:
- **Component**: <name>
- **State**: <state>
- **Failing conditions**: <list>
- **Relevant events**: <summary>
- **Log evidence**: <key log lines>

### Node Health
For each node (or summarise if all healthy):
- **Node**: <name> | **Zone**: <az> | **Role**: <worker|control-plane>
- **Status**: <Ready / NotReady / SchedulingDisabled>
- **Pressure conditions**: <MemoryPressure | DiskPressure | PIDPressure | none>
- **Resource usage**: CPU <used>/<allocatable> | Memory <used>/<allocatable>
- **Notable**: <cordoned, taint, AZ imbalance, or "none">

### Istio Mesh Issues
<summary of Istio findings, or "None detected">

### Root Cause Assessment
<Primary hypothesis — one paragraph. Be specific: name the component, the failure mode, and why the evidence points there.>

### Contributing Factors
<Secondary issues that may not be root cause but aggravate the situation>

### Recommended Actions
1. <Immediate action — highest priority>
2. <Follow-up investigation or fix>
3. <Preventive measure if applicable>

### Confidence
<HIGH / MEDIUM / LOW> — explain briefly why (e.g., "HIGH: clear error in operator logs matching known issue", "LOW: symptoms are ambiguous, further investigation needed")
```

---

## Kyma-specific knowledge to apply during analysis

- **Kyma modules** are managed by the Lifecycle Manager (`kyma-system` namespace). If Lifecycle Manager itself is degraded, all modules may appear broken — check it first.
- **Module state** flows from the module's operator CR (e.g., `APIGateway`, `Eventing`, `BTP Operator`) — a module in `Error` state means its operator is reporting a reconciliation failure.
- **Gardener** manages node pools, infrastructure, and certificates on SAP BTP. Gardener-related issues (e.g., shoot reconciliation failures) show in cluster metadata or node events.
- **Istio** is a core Kyma dependency. Missing sidecars or broken mTLS often cascade into application-level 503/connection refused errors.
- **BTP Operator** failure affects service binding — applications that depend on bound BTP services will fail silently if BTP Operator is degraded.
- **Eventing** uses NATS or EventMesh. If eventing shows errors, check whether the backend (NATS cluster health or EventMesh subscription status) is the source.
- **Resource exhaustion** (OOM, CPU throttling, PVC full) is common on trial or small SKU clusters — always cross-check node capacity when pods are crashing.
- **Node topology**: Gardener provisions worker nodes across availability zones. If all nodes in one AZ show `NotReady` or pressure conditions, it is likely an infrastructure-level AZ issue rather than an application bug. AZ imbalance (e.g. 2 nodes in zone A, 0 in zone B after a scale event) can cause `Unschedulable` errors for workloads with hard anti-affinity rules.
- **Node pressure cascade**: `MemoryPressure` on a node triggers pod eviction (starting with `BestEffort` pods). If evicted pods are Kyma module operators, modules will appear degraded even though the root cause is node-level. Always check node conditions before concluding a module is broken.
- **Cordoned nodes**: A node with `unschedulable: true` (cordoned) silently reduces cluster capacity. Pending pods and `Insufficient CPU/memory` scheduling failures may be caused by fewer schedulable nodes than expected, not by actual resource shortage.
