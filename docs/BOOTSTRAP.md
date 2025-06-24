# Omni Bootstrap Run-book (erauner-home cluster)

> **Scope:** steps from *empty NUCs in Maintenance mode* ➜ *Kubernetes nodes Ready with Cilium and CoreDNS*.
>
> **Day-2 GitOps** is documented in the [`homelab-k8s` BOOTSTRAP.md](https://github.com/erauner12/homelab-k8s/blob/master/docs/BOOTSTRAP.md).

---

## 0. CLI Configuration

**Before starting the bootstrap process**, ensure your CLI tools are properly configured.

If you're starting fresh or experiencing authentication issues, follow the complete setup process:

👉 **[CLI Setup: True-From-Zero Rebuild](../../homelab-k8s/docs/CLI-SETUP.md)**

This guide covers:
- Complete cleanup of existing configurations
- Fresh omnictl and talosctl setup
- Authentication troubleshooting
- OIDC vs break-glass configuration options

---

## 1. Prerequisites

| What                                                              | Why / Notes                                                      |
| ----------------------------------------------------------------- | ---------------------------------------------------------------  |
| **`omnictl` & `talosctl` CLIs**                                   | Install and configure both CLIs. Your `omniconfig.yaml` and `talosconfig.yaml` should be in place. See the [CLI Setup Guide](../../homelab-k8s/docs/CLI-SETUP.md) for complete setup instructions. |
| **Cluster Template**                                              | A complete `cluster-template-home.yaml` file, with correct machine UUIDs and patch paths.              |
| **Nodes in Maintenance Mode**                                     | All cluster nodes should be powered on and booted into Talos maintenance mode (e.g., via PXE boot). They should appear in the Omni UI.        |

---

## 2. Create the Cluster

With the prerequisites met, create the cluster by applying your template. This tells Omni to generate the machine configs and serve them to the nodes.

```bash
omnictl cluster template sync -f ./cluster-template-home.yaml
```

The nodes will reboot out of maintenance mode and begin the Talos installation process.

---

## 3. Approve the Initial Node Certificate

When the first control plane node comes up, it can't fully initialize its networking until a CNI (like Cilium) is installed. This creates a chicken-and-egg problem: the `kubelet-serving-cert-approver` (which automatically approves node certificates) is a pod that needs a running CNI to be scheduled and function correctly.

Therefore, the initial node(s) will get stuck waiting for their `kubelet-serving` certificate to be approved. You must manually intervene to break this deadlock.

First, get a working `kubeconfig`:

```bash
# This may take a minute or two to become available
omnictl cluster kubeconfig get erauner-home > ~/.kube/config-erauner-home
export KUBECONFIG=~/.kube/config-erauner-home
```

Check for pending Certificate Signing Requests (CSRs):

```bash
kubectl get csr
# NAME        AGE   SIGNERNAME                                    REQUESTOR                                                                   REQUESTEDDURATION   CONDITION
# csr-abcde   1m    kubernetes.io/kubelet-serving                 system:node:controlplane1                                                   8760h               Pending
```

You have two options to resolve this.

### Option 1: Manually Approve the CSR (Recommended)

This is the simplest approach. Approve the pending CSR for each node that is stuck.

```bash
# Get the name of the pending CSR and approve it
kubectl certificate approve <csr-name-from-above>

# Example:
kubectl certificate approve csr-abcde
```

The node should become `Ready` within a minute. Repeat for any other nodes that join before the CNI is active.

### Option 2: Patch the Cilium Install Job

Alternatively, you can patch the Cilium installation Job to run on the host's network. This allows the `cilium-cli` installer to connect to the local kube-apiserver proxy on the node, bypassing the need for cluster networking.

To do this, add `hostNetwork: true` to the Job's pod spec within your `patches/cilium-install-home.yaml` file before syncing the cluster template.

```yaml
# in patches/cilium-install-home.yaml

# ...
---
apiVersion: batch/v1
kind: Job
metadata:
  name: cilium-install
  namespace: cilium
spec:
  template:
    spec:
      serviceAccountName: cilium-install
      hostNetwork: true          # <-- ADD THIS LINE
      restartPolicy: OnFailure
# ...
```

This avoids the manual CSR approval step but requires modifying the installation manifest.

---

## 4. Install Cilium CNI

The cluster template (`cluster-template-home.yaml`) is configured to automatically install Cilium using an `inlineManifests` job defined in `patches/cilium-install-home.yaml`.

Once the first control plane node is `Ready` (after you've approved its CSR), Kubernetes will schedule this job, which installs the Cilium CNI across the cluster.

---

## 5. Monitor Bootstrap Progress

You can watch the cluster come online.

```bash
# Watch nodes join and become Ready
watch kubectl get nodes

# Watch for the Cilium install job to complete
watch kubectl -n cilium get job,pod

# Once the job is done, watch the main Cilium pods
watch kubectl -n cilium get pods
```

Once all Cilium pods are `Running` and all nodes are `Ready`, the cluster bootstrap is complete. You are now ready for the Day-2 GitOps setup.