# Home service-account issuer discovery

## Problem and change

The home cluster advertises its Omni/SideroLink control-plane endpoint,
`https://[fdae:41e4:649b:9303::1]:10000`, as the Kubernetes service-account token
issuer. Pods cannot route to that IPv6 address. Substrate 0.0.9 discovers signing
keys by fetching the token issuer URL, so authentication fails even though pods
can reach the Kubernetes API through its Service.

The home-only control-plane patch changes the issuer and advertised JWKS URL to
`https://kubernetes.default.svc.cluster.local`. Substrate recognizes that issuer,
uses the mounted Kubernetes CA, and authenticates discovery requests with its
own service-account token. Its chart includes RBAC for discovery/JWKS reads.
The API endpoint, certificate/signing keys, and API audiences are not changed.
Talos continues to set the API audience to the original control-plane endpoint;
new default-audience tokens remain valid for that API audience.

This patch is appended to the home ControlPlane patch list to avoid renumbering
existing Omni patch resources. It does not change the shared controlplane patch
used by other clusters.

## Maintenance-window requirement

**Keep this PR in draft until the token-transition plan is accepted.**

Talos 1.11.5 represents `extraArgs` as a string map and renders one flag per key.
Kubernetes 1.31 accepts multiple issuers only through repeated
`--service-account-issuer` flags (a StringArray flag, not a comma-separated
StringSlice). A comma-separated string or YAML list is not a dual-issuer fix
on this Talos version.

Consequently, replacing the issuer makes already-issued projected tokens fail
API authentication until their consumers receive new tokens. Kubelet normally
rotates these, but waiting alone is not a zero-downtime rollout plan. Inventory
critical service-account consumers first (CNI, DNS, storage, Argo CD, workloads)
and arrange sequential pod replacement/rollout where needed, through the normal
GitOps maintenance workflow. Verify recovery per component before proceeding.
Some clients cache tokens and require restart even after the volume updates.

There is one control-plane node. Expect a brief API interruption when Talos
restarts kube-apiserver, plus possible controller interruptions during token
refresh. Existing application data is unrelated to this authentication change.
Keep working Omni administrative access available to revert the patch.

If that interruption is unacceptable, use a Substrate fix that separates the
expected token issuer from its discovery/JWKS fetch address instead; that avoids
changing cluster-wide token identity but requires a tested custom runtime build.

## Review and deployment

The repository's Jenkins pipeline validates templates; it does not sync Omni.
PR merge alone does not apply this change. Before an authorized sync:

```bash
omnictl cluster template validate -f cluster-template-home.yaml
omnictl cluster template diff -f cluster-template-home.yaml
```

The diff should contain only the new home control-plane ConfigPatch. Stop if it
also changes node membership, extensions, network settings, or other patches.
During the agreed maintenance window, sync the reviewed home template using the
normal Omni workflow. Do not run a direct `talosctl patch` that bypasses Git.

After the API returns:

1. Inspect the discovery document from inside a pod using its mounted CA/token.
   Expect the new issuer and the in-cluster `/openid/v1/jwks` URL.
2. Verify a newly issued projected token's issuer, successful API access, and
   authenticated JWKS retrieval. Do not print or commit token contents.
3. Verify CNI, DNS, storage and Argo CD controllers have recovered; refresh
   remaining consumers through reviewed, sequential rollouts as needed.
4. In homelab-k8s PR #1643, set `auth.jwt.issuer` to
   `https://kubernetes.default.svc.cluster.local`, then deploy/verify Substrate
   before the canary in PR #1644. Do not mark the prerequisite verified solely
   because the template renders.

Rollback is the inverse Git change followed by the reviewed Omni sync. It
restores the old issuer but invalidates tokens minted with the new issuer, so
the same consumer-refresh considerations apply in reverse.

Sources:

- [Talos 1.11.5 API-server flag construction](https://github.com/siderolabs/talos/blob/v1.11.5/internal/app/machined/pkg/controllers/k8s/control_plane_static_pod.go)
- [Talos 1.11.5 argument builder](https://github.com/siderolabs/talos/blob/v1.11.5/pkg/argsbuilder/argsbuilder_args.go)
- [Kubernetes 1.31 issuer/audience flags](https://github.com/kubernetes/kubernetes/blob/v1.31.5/pkg/kubeapiserver/options/authentication.go)
- [Substrate 0.0.9 discovery client](https://github.com/kagent-dev/substrate/blob/v0.0.9/cmd/ateapi/main.go)
