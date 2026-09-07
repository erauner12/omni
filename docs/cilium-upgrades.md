# Home Cilium upgrades

Cilium is a Helm release in namespace `cilium`, originally installed by the
Talos bootstrap Job. Argo CD manages monitoring and BGP resources in
`homelab-k8s`, not the Helm release. Keep this ownership unchanged during the
Kubernetes/Substrate prerequisite upgrades.

The chart version and explicit Helm values are in
`patches/cilium-install-home.yaml`. Each version change gets a PR. After merge,
reconcile the exact merged configuration with Omni and Helm; do not rerun the
completed install Job or sync all bootstrap manifests (other manifests have
independent owners).

## Reconcile a merged version

Run from a clean checkout of merged `main`. Use the home-cluster kubeconfig,
never the local kind context. The commands below use the selected kubeconfig;
set `KUBECONFIG` explicitly and verify the four home nodes first.

```bash
kubectl get nodes
omnictl cluster template validate -f cluster-template-home.yaml
omnictl cluster template diff -f cluster-template-home.yaml
```

Extract only the Cilium manifest from the patch. The helper prints non-secret
Helm values; never substitute a dump of the whole Talos machine configuration.

```bash
cilium_home_manifest() {
  yq -r '.cluster.inlineManifests[] | select(.name == "cilium-install-home") | .contents' patches/cilium-install-home.yaml
}
cilium_home_values() {
  cilium_home_manifest | yq -r 'select(.kind == "ConfigMap") | .data."values.yaml"'
}
cilium_target_version=$(cilium_home_manifest | yq -r 'select(.kind == "Job") | .spec.template.spec.containers[0].command[-1]')
cilium_chart_dir=$(mktemp -d)
curl -fSL "https://helm.cilium.io/cilium-${cilium_target_version}.tgz" -o "${cilium_chart_dir}/cilium.tgz"
tar -xzf "${cilium_chart_dir}/cilium.tgz" -C "${cilium_chart_dir}"
cilium_home_values | helm template cilium "${cilium_chart_dir}/cilium" -n cilium -f - | kubectl apply --dry-run=server -f -
```

Run upstream's temporary preflight before upgrading. It validates existing
Cilium policies and pre-pulls images without replacing running agents. Stop if
either rollout fails; inspect the preflight logs. Only remove this exact
temporary release after recording its result.

```bash
helm install cilium-preflight "${cilium_chart_dir}/cilium" -n cilium \
  --set preflight.enabled=true --set agent=false --set operator.enabled=false \
  --set k8sServiceHost=192.168.1.193 --set k8sServicePort=6443
kubectl -n cilium rollout status daemonset/cilium-pre-flight-check --timeout=10m
kubectl -n cilium rollout status deployment/cilium-pre-flight-check --timeout=10m
helm uninstall cilium-preflight -n cilium

omnictl cluster template sync -f cluster-template-home.yaml
cilium_home_values | helm upgrade cilium "${cilium_chart_dir}/cilium" -n cilium -f - \
  --reset-values --atomic --timeout=15m
# Keep the bootstrap ConfigMap consistent without rerunning its install Job.
cilium_home_manifest | yq 'select(.kind == "ConfigMap")' | kubectl apply -f -
```

Do not use `--reuse-values` across versions. Explicit reviewed values preserve
the running host-only socket load-balancing scope, single operator, native
routing, Geneve DSR, IPAM pool, device selector, BGP and Hubble settings.
The old release had a failed revision 10 after a timeout; revision 9 was the
last successful release. Inspect Helm history before choosing any rollback.

After each upgrade, verify all four agents, the operator and Hubble, all three
worker BGP sessions to `192.168.1.1` (local AS 65000, peer AS 64512), advertised
routes, in-pod DNS, LoadBalancer reachability, attached Longhorn volumes, and
Plex. Record results on the PR before the next change. A Helm atomic rollback
does not revert Git: if triggered, stop and reconcile the desired version via
a follow-up PR before attempting another upgrade.

## Remaining sequence

1. Finish the 1.17 patch upgrade before moving to Cilium 1.18.
2. Upgrade consecutive Cilium minors only. Check each release's Kubernetes
   matrix and upgrade notes before advancing Kubernetes.
3. Convert `network-secrets`'s `home-bgp-policy` to BGP v2 in a separate
   `homelab-k8s` PR before upgrading to a release that removes the old policy.
   Preserve pod CIDR and all LoadBalancer advertisements; expect BGP sessions
   to restart during this conversion.
4. Upgrade Kubernetes one minor per Omni PR toward 1.35, then enable the
   Substrate certificate feature gates separately. Talos stays 1.13.10 unless
   a separately verified prerequisite requires otherwise.

Reference: [upstream upgrade guide](https://github.com/cilium/cilium/blob/v1.17.18/Documentation/operations/upgrade.rst).
