# OpenShift GitOps Lab

# OpenShift Virtualization via GitOps (Argo CD)

Fully reproducible OpenShift Virtualization (CNV) deployment using OpenShift GitOps
(Argo CD) with the App-of-Apps pattern. Rebuild a complete CNV + VM environment on
any fresh OpenShift cluster with two `oc apply` commands.

## Branch

Active work lives on `lab/ocp-virt`, not `main`.

## Architecture

bootstrap/
├── argocd-app-of-apps.yaml     # the ONE manifest you apply manually
└── rbac/
└── argocd-cnv-permissions.yaml   # extra RBAC Argo CD needs for CNV CRDs
apps/
├── ocp-virt-operator-app.yaml  # Application -> virt/operator
└── ocp-virt-vms-app.yaml       # Application -> virt/vms
virt/
├── operator/                   # CNV operator install (namespace, OperatorGroup,
│                                # Subscription, HyperConverged CR)
└── vms/                        # VirtualMachine definitions

**Pattern:** `root-app` (App-of-Apps) watches `apps/`. Each file there is itself an
Argo CD `Application` pointing at a subfolder of `virt/`. This keeps the operator
install and the VM workloads as two independent, separately-syncing units — a
failure in one never blocks the other.

## Rebuild runbook (fresh cluster)

```bash
# 1. Login as cluster-admin
oc login <cluster>

# 2. Install OpenShift GitOps operator (OperatorHub or subscription YAML),
#    wait for openshift-gitops pods to be Running
oc get pods -n openshift-gitops

# 3. Clone and checkout this branch
git clone https://github.com/KadirSoft/gitops-ocp.git
cd gitops-ocp
git checkout lab/ocp-virt

# 4. Bootstrap: RBAC first, then root-app
oc apply -f bootstrap/rbac/argocd-cnv-permissions.yaml
oc apply -f bootstrap/argocd-app-of-apps.yaml

# 5. Watch it cascade
oc get application.argoproj.io -n openshift-gitops
```

### Expected timeline

| Time      | What's happening                                                    |
|-----------|-----------------------------------------------------------------------|
| 0-1 min   | `root-app` syncs, spawns `ocp-virt-operator` + `ocp-virt-vms`         |
| 1-5 min   | Namespace/OperatorGroup/Subscription created; OLM installs CNV (CSV `Installing` → `Succeeded`). `ocp-virt-vms` shows failed/retrying — **normal**, `VirtualMachine` CRD doesn't exist yet |
| 5-10 min  | `HyperConverged` created; CNV rolls out ~15-20 pods                   |
| 10-15 min | `VirtualMachine` CRD + cluster instancetypes appear; VMs provision and boot |

If `ocp-virt-vms` shows `Failed` because it exhausted its 5 retries before CNV
finished installing, it needs a manual nudge — see Troubleshooting below.

## Key lessons learned (why this took a while to get right)

### 1. Operator + CR ordering breaks naive sync waves
Sync waves control *order*, but Argo CD validates the **entire** manifest batch
against live CRDs before running any wave. If a later-wave resource's CRD doesn't
exist yet (e.g. `HyperConverged`, `VirtualMachine`), the whole sync is rejected
up front — waves never even get a chance to run in order.

**Fix:** `syncOptions: [SkipDryRunOnMissingResource=true]` on any Application whose
manifests reference a CRD that another resource in the same batch is responsible
for installing.

**Better fix (what we landed on):** split "install the operator" and "use the
CRD it provides" into two **separate Applications**. Cleaner, and a stuck VM
sync can never block the operator install.

### 2. Argo CD's default RBAC doesn't cover every operator's CRDs
OpenShift GitOps' `argocd-application-controller` ClusterRole grants read access to
everything but only write access to a specific allowlist of API groups
(`operators.coreos.com`, `config.openshift.io`, etc). Custom operator CRDs like
`hco.kubevirt.io` (HyperConverged) and `kubevirt.io` (VirtualMachine) are **not**
on that list by default.

**Fix:** `bootstrap/rbac/argocd-cnv-permissions.yaml` — an additive ClusterRole +
ClusterRoleBinding granting the application-controller SA full access to
`hco.kubevirt.io`, `kubevirt.io`, `cdi.kubevirt.io`, `ssp.kubevirt.io`. Applied
before `root-app` in the bootstrap sequence. Symptom if missing: sync fails with
`... is forbidden: User "system:serviceaccount:openshift-gitops:..." cannot patch
resource ...`.

### 3. Never hardcode cluster-generated snapshot names
CNV auto-imports "golden" boot images as VolumeSnapshots with a random hash
suffix (e.g. `rhel8-004e24cfacec`) — regenerated fresh on every cluster. A VM
manifest referencing that exact name will never resolve on a different cluster.

**Fix:** reference the stable `DataSource` object instead (fixed name like `rhel8`,
always points at whatever the current snapshot is):

```yaml
dataVolumeTemplates:
  - spec:
      sourceRef:
        kind: DataSource
        name: rhel8
        namespace: openshift-virtualization-os-images
```

### 4. KubeVirt VMs need a specific `ignoreDifferences` set to ever show `Synced`
KubeVirt's admission webhooks and controllers mutate the live VM object after
creation — none of this is "real" drift, but Argo CD flags it as `OutOfSync`
unless told otherwise. The full set needed:

```yaml
ignoreDifferences:
  - group: kubevirt.io
    kind: VirtualMachine
    jqPathExpressions:
      - '.spec.template.spec.domain'                          # expanded from instancetype/preference
      - '.spec.template.spec.architecture'                      # auto-detected
      - '.spec.instancetype.revisionName'                       # ControllerRevision pin, auto-added
      - '.spec.preference.revisionName'                         # same, for preference
      - '.spec.dataVolumeTemplates[0].metadata.creationTimestamp'  # null-defaulted nested field
      - '.metadata.annotations'                                 # kubemacpool timestamp + others, changes every reconcile
      - '.metadata.finalizers'
      - '.metadata.generation'
      - '.metadata.resourceVersion'
```

Found by reading Argo CD's own **DIFF tab** in the UI — the only reliable source
of truth for "what does Argo CD actually think is different." `oc diff` is
**not** useful here since it has no knowledge of `ignoreDifferences`.

### 5. A failed sync blocks auto-sync on that revision — refresh alone won't retry it
Once a sync operation fails, Argo CD (even with `selfHeal: true`) will not
automatically retry the *same* Git revision. `argocd.argoproj.io/refresh=hard`
only recomputes the diff — it does not trigger a new sync attempt.

**Reliable fix, works every time:**
```bash
oc delete application.argoproj.io <app-name> -n openshift-gitops
oc annotate application.argoproj.io root-app -n openshift-gitops \
  argocd.argoproj.io/refresh=hard --overwrite
```
Deleting the Application only removes Argo CD's tracking object — it does **not**
delete the resources it manages (no cascade), so this is safe to do on a stuck app.
`root-app`'s `selfHeal` recreates it fresh against the current commit, with no
failed-operation history to block auto-sync.

### 6. Watch out for duplicate/wrongly-named CRDs on OpenShift
`Application`, `Subscription`, and similar names exist under **multiple** CRDs
on an OpenShift cluster (e.g. ACM's `subscriptions.apps.open-cluster-management.io`
vs OLM's `subscriptions.operators.coreos.com`). Plain `oc get application` /
`oc get subscription` can silently hit the wrong one. Always use the fully
qualified name: `oc get application.argoproj.io`, `oc get subscription.operators.coreos.com`.
