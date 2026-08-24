# zt-showroom-deployer

A reusable Helm chart that deploys a Red Hat **showroom** onto an already-provisioned
OpenShift cluster. It renders the full showroom runtime: a Deployment that clones and
builds your Antora/AsciiDoc content, serves it, and exposes an in-browser web terminal —
fronted by a Service and an edge Route.

This chart is the deploy target of the OODA workshop factory's `workshop-do` phase. Each
workshop's `zt-<slug>-automation` repo is a thin wrapper that supplies a
`values-<slug>.yaml` and calls `helm upgrade --install` against this published chart.

## What it deploys

A single `showroom` Deployment (`Recreate` strategy, 1 replica) with:

**Init containers**
- `cluster-setup` *(optional, gated by `showroom.clusterSetup.enabled`)* — patches the
  default IngressController to strip `X-Frame-Options` and `Content-Security-Policy`
  response headers so the showroom can be embedded in an iframe.
- `git-cloner` — clones `showroom.gitRepoUrl` @ `showroom.gitRepoRef` into a shared volume.
- `antora-builder` — builds the Antora site from the cloned content.

**Runtime containers**
- `content` (:8000) — serves the built showroom content.
- `nginx` (:8080) — reverse proxy: `/` → content (:8000), `/wetty` → wetty (:8001,
  websocket upgrade).
- `wetty` (:8001) — browser SSH terminal into the lab environment.

**Supporting objects**
- `Namespace`, `ServiceAccount` (`showroom`)
- RBAC: a namespace-suffixed cluster-setup `ClusterRole`/`ClusterRoleBinding` (ingress
  header patch, gated by `showroom.clusterSetup.enabled`) and a namespace `edit`
  `RoleBinding` for the SA (gated by `showroom.rbac.namespaceEdit`).
- ConfigMaps: `showroom-proxy-config` (nginx.conf) and `showroom-userdata`
  (`user_data.yml` — login command + cluster URLs surfaced to the content).
- `Service` (ClusterIP :8080) and `Route` (edge/Redirect, 1h haproxy timeouts).

## Install

```bash
helm upgrade --install showroom . \
  -n showroom --create-namespace \
  -f values.yaml
```

Or from the published Helm repo:

```bash
helm repo add eformat https://eformat.github.io/helm-charts
helm repo update eformat
helm upgrade --install showroom eformat/showroom-deployer \
  -n showroom-<slug> --create-namespace \
  -f values-<slug>.yaml
```

Render without applying:

```bash
helm template showroom . -f values.yaml
```

## Configuration

See `values.yaml` for the full contract. The keys you almost always override per workshop:

| Key | Meaning |
|-----|---------|
| `deployer.domain` | Cluster apps wildcard domain (e.g. `apps.cluster.example.com`) |
| `deployer.apiUrl` | Cluster API URL (e.g. `https://api.cluster.example.com:6443`) |
| `showroom.namespace` | Target namespace for this showroom |
| `showroom.gitRepoUrl` / `gitRepoRef` | The `zt-<slug>-showroom` content repo + ref |
| `showroom.guid` / `user` / `password` | Lab identity surfaced in `user_data.yml` |
| `showroom.projectName` | Project name shown to the learner |
| `showroom.wetty.ssh*` | SSH coordinates for the web terminal |
| `showroom.images.*` | Pin the runtime image versions |

### Security note

`showroom.wetty.sshPass` is sensitive. This chart renders it as a container arg to match
the live reference deployment. A future hardening could source it from a `Secret` instead —
out of scope for now.

## Publishing

Publishing to a Helm chart repo is a manual step:

```bash
helm package .
# then push the resulting .tgz + updated index.yaml to your chart repo
```
