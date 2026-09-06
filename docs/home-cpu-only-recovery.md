# Home worker CPU-only recovery

Workers 1 and 3 became unreachable during the Talos 1.12.12 rollout after
amdgpu SMU/gfxoff errors. The GPU driver is a suspected cause, not a confirmed
hardware diagnosis. Worker2 already runs without the GPU extension.

The recovery change keeps Talos and Kubernetes versions unchanged, removes
the shared workers' amdgpu extension and kernel module patch, and explicitly
sets `amd.com/gpu=false`. The GPU operator selects `amd.com/gpu=true`, so its
device plugin, labeller, and metrics exporter should no longer target them.
Keep AMD CPU microcode and all storage extensions. No disks or PVCs are reset.

## Application dependency check (2026-09-06)

- Live `media/media-stack-plex` StatefulSet and `apps/plex/values.yaml` have
  no GPU resource request, GPU device mount, or privileged container access.
  The older media-stack hardware-transcoding documentation is not the current
  deployment. No Plex configuration change is required for this recovery.
- Live `ai/qwen-llama` and its Git deployment have zero replicas. It still
  requires an AMD GPU; leave it suspended and preserve its cache PVC.
- No other application GPU requests or direct GPU device mounts were found.
- Plex was Pending due to worker availability and pod capacity. Disabling
  GPU support does not itself resolve capacity or storage availability.

## Apply and verify

1. Merge the reviewed recovery PR, check out the merged commit, validate the
   home cluster template, inspect its Omni diff, and sync that exact template.
2. Disconnected machines cannot receive an image/configuration update. If they
   remain unreachable, arrange local console access and a coordinated reboot.
   A reboot alone may boot the old GPU-enabled image and hang again; confirm
   the CPU-only image is installed rather than repeatedly power-cycling.
3. Verify workers 1 and 3 reconnect, run Talos 1.12.12 without the amdgpu
   extension, report Ready, and have `amd.com/gpu=false`. Confirm GPU operator
   node daemons are no longer scheduled on them and GPU allocatable resources
   are no longer advertised after device-plugin reconciliation.
4. Check Longhorn replicas/volumes, database readiness, and Plex scheduling.
   Do not delete or salvage storage to work around a disconnected node.
5. Keep Talos 1.13 and Substrate PRs paused until the cluster is healthy.
   Close the temporary database maintenance window through its cleanup PR
   once maintenance is safely finished.

## Revisit GPU support

Use a separate reviewed change after BIOS/thermal/memory checks and an isolated
GPU stability test. Restore the extension, module patch, and true label together
on one test node first. Verify it before enabling a GPU application. Do not
scale Qwen up unchanged: its existing affinity still targets CPU-only worker2.
Reverting this recovery immediately can reintroduce the suspected hang.
