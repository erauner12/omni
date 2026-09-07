# Substrate certificate prerequisite

Apply only after all home nodes have completed Kubernetes 1.35.8. Talos stays
1.13.10 and Cilium stays 1.19.7. This is a separate PR from version upgrades
and from replacing the old kagent/Substrate installation.

The home-only patch enables `ClusterTrustBundle`, `ClusterTrustBundleProjection`
and `PodCertificateRequest` on control-plane components and kubelets, matching
upstream's feature-gate configuration. The API server additionally serves
`certificates.k8s.io/v1beta1`. The three gates default to false in Kubernetes
1.35; `PodCertificateRequest`'s `AuthorizeNodeWithSelectors` dependency is
already GA and enabled. No signing keys or service-account issuer settings
are changed. Substrate will supply its own certificate controllers and CAs.

## Reconcile and validate

1. Capture a fresh private etcd snapshot; check all four nodes and workloads.
2. Validate and diff the Omni template. Only the new home config patch should
   be added; do not sync a version upgrade at the same time.
3. Merge the PR and sync the exact merged template. Expect API-server and
   other control-plane component restarts plus kubelet restarts. Do not reboot
   nodes manually or replace existing configurations.
4. Verify all four nodes Ready, Omni machine-config reconciliation complete,
   and API discovery includes `clustertrustbundles` and
   `podcertificaterequests` under `certificates.k8s.io/v1beta1`.
5. Read only `/etc/kubernetes/kubelet.yaml` through Talos on each node; confirm
   the three gates and the existing `LocalStorageCapacityIsolationFSQuotaMonitoring`
   gate remain enabled. Confirm the existing image-GC thresholds remain 40/50.
6. Validate API admission of projected certificate/trust-bundle volumes with
   server-side dry runs. Full issuance/projection is validated after installing
   Substrate's signers; API discovery alone is not an end-to-end certificate test.
7. Recheck Cilium/BGP, exact route sets, Service VIPs, Plex, Envoy, in-pod DNS
   and attached storage before replacing kagent/Substrate.

Rollback must be a Git PR. Before disabling these gates, remove workloads
that depend on the projections while their controllers/signers still run.
Do not disable the APIs under an active Substrate installation.

Sources:
- [Kagent's pinned integration configuration](https://github.com/kagent-dev/kagent/blob/d5326715d44aa0ed505f7ea681d1b6f9e1869017/scripts/kind/kind-config.yaml)
- [Kubernetes 1.35 projected volumes](https://v1-35.docs.kubernetes.io/docs/concepts/storage/projected-volumes/)
- [Kubernetes 1.35.8 feature defaults](https://github.com/kubernetes/kubernetes/blob/v1.35.8/pkg/features/kube_features.go)
