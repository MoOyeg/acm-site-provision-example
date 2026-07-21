# Remote site fleet

Git source of truth for remote OpenShift sites provisioned by
[Red Hat Advanced Cluster Management](https://www.redhat.com/en/technologies/management/advanced-cluster-management)
using the **SiteConfig operator** and **OpenShift GitOps (Argo CD)**.

Each folder under `sites/` describes one remote site as a single `ClusterInstance`
resource. Argo CD syncs it to the ACM hub; the SiteConfig operator merges it with
the referenced installation templates and renders the install manifests
(`ClusterDeployment`, `AgentClusterInstall`, `InfraEnv`, `BareMetalHost`,
`NMStateConfig`, `ManagedCluster`, `KlusterletAddonConfig`). The Assisted
Installer then provisions the hardware over Redfish virtual media and the finished
cluster registers itself back into ACM.

```
argocd/applicationset.yaml   Git generator — one Argo CD Application per sites/* folder
sites/<cluster>/
├─ namespace.yaml            must match the cluster name; the templates hardcode it
├─ clusterinstance.yaml      the site definition
└─ extra-manifests.yaml      MachineConfigs applied to this site during install
```

## Install-time customisation

`extra-manifests.yaml` holds a ConfigMap of manifests referenced from the
ClusterInstance via `spec.extraManifestsRefs`. The SiteConfig operator passes it
to the Assisted Installer (as `AgentClusterInstall.spec.manifestsConfigMapRefs`)
and its contents are applied to the site **while the cluster is installing** — so
each location comes up already configured, with no post-install drift to chase.

| Site | Profile | MachineConfigs |
|---|---|---|
| `sno1-cluster2` | Telco far-edge (RAN DU) | `02-master-workload-partitioning` — reserve CPUs 0-1 for platform workloads; site chrony |
| `sno2-cluster2` | Retail / industrial edge | `99-master-journald-limits` — cap journal growth on constrained hardware; site chrony |

Two things to know before copying this pattern:

- **Put MachineConfigs in `extraManifestsRefs`, not `templateRefs`.** Cluster-level
  templates render resources onto the *hub*; a MachineConfig placed there would be
  created on the hub and applied to the hub's own nodes.
- **`extraManifestsRefs` is immutable after the ClusterInstance is created.** The
  admission webhook rejects adding or changing it on an installed site
  (`unauthorized changes in immutable fields: /extraManifestsRefs`). These
  manifests take effect on install or reinstall only — which is the intent, since
  it is what guarantees every site boots identically.

## The separation of concerns

`clusterinstance.yaml` is the **what** — cluster name, base domain, networks, and
each node's BMC address, boot MAC and static network config.

The **how** lives in the templates it references via `templateRefs`, which ship
with the SiteConfig operator:

| templateRefs | Install method |
|---|---|
| `ai-cluster-templates-v1` / `ai-node-templates-v1` | Assisted Installer |
| `ibi-cluster-templates-v1` / `ibi-node-templates-v1` | Image-Based Install |

Pointing a site at the `ibi-*` templates changes how it is installed without
changing the site definition itself.

## Adding a site

Commit a new folder under `sites/`. The ApplicationSet's Git generator picks it
up and creates an Argo CD Application for it — no Argo CD configuration change
and no pipeline change.

## Adding a node to an existing site

Append one entry to `spec.nodes[]` in that site's `clusterinstance.yaml` and
push. The operator renders install resources for the new node only; nodes already
installed are untouched. Removing a node's block drains and deprovisions it.

## Secrets

Deliberately absent. Each site namespace needs a `pull-secret` and a
`<node>-bmc-credentials` secret, created out of band. In a production fleet these
would come from Sealed Secrets or the External Secrets Operator rather than
living here.
