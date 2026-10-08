# lan-party-helmchart

One chart deploying the whole Mercurius LAN stack: backend, frontend and a
CloudNativePG database. Installed once per environment by ArgoCD from the
[gitops](https://github.com/MercuriusAalst/gitops) repo.

Published as an OCI chart on every release:

```
oci://ghcr.io/mercuriusaalst/charts/lan-party
```

## What it does not do

**No Ingress.** A Cloudflare Tunnel fronts the cluster and routes the public
hostnames to the two ClusterIP services. `hosts.frontend` / `hosts.backend`
exist only because the apps need to know their own public URLs (Auth0
callbacks, browser-facing image URLs) — nothing in this chart serves them.

**No secrets.** Every secret comes from external-secrets. The chart renders an
`ExternalSecret` per component from `externalSecrets.backend` /
`externalSecrets.frontend`; the values there are references, never values.
`externalSecrets.secretStore.name` is required.

## Required environment values

Supply these values in each environment's GitOps configuration before installing
or upgrading the chart:

```yaml
hosts:
  frontend: frontend.example.com
  backend: backend.example.com
backend:
  config:
    Mercurius.LAN.API_Auth0__Audience: https://api.example.com
frontend:
  config:
    Auth0__Audience: https://api.example.com
```

Use your public hostnames and configured Auth0 API audience. These four settings
default to empty strings; rendering fails with the missing setting's name.
Releases that relied on the previous defaults must add these overrides before
upgrading. `hosts.frontend` remains required as part of the environment contract,
although no current template consumes it.

## Backups

Off by default. Turn them on per environment:

```yaml
postgres:
  backup:
    enabled: true
    destinationPath: s3://mercurius-backups/prd
    endpointURL: https://<account>.r2.cloudflarestorage.com  # omit for AWS S3
externalSecrets:
  postgres:
    - envVar: ACCESS_KEY_ID
      key: S3_ACCESS_KEY_ID
    - envVar: ACCESS_SECRET_KEY
      key: S3_SECRET_ACCESS_KEY
```

That renders a Barman Cloud `ObjectStore` and attaches the plugin to the
database as a WAL archiver. The plugin itself is installed cluster-wide by the
gitops repo's `core/barman-cloud` Application — this chart only configures it.

`serverName` is the folder inside the bucket and defaults to `<release>-db`.
It lives on the Cluster's plugin parameters, not on the ObjectStore, which
rejects it. Changing it starts a fresh history and leaves the old one
unreachable, so leave it alone once backups exist.

Enabling backups does not by itself prove restores work. Nothing here tests
that; a restore drill is still a thing someone has to do.

## Things that will bite you

**The backend env var prefix.** `Program.cs` re-adds `appsettings.json` *after*
the default environment provider, then adds a second provider with the
`Mercurius.LAN.API_` prefix. So for any key that also exists in
`appsettings.json`, only the prefixed env var wins. Backend keys keep the
prefix; frontend keys don't (its `Program.cs` adds no prefixed provider).

**Both deployments are pinned to one replica.** The backend's media volume is
ReadWriteOnce and its SignalR hub keeps connection state in memory; the
frontend is Blazor Server and keeps a per-user circuit. Scaling either out
needs more than a bigger `replicas` — RWX plus a SignalR backplane for the
backend, sticky sessions at the tunnel for the frontend. The field is absent
rather than present-and-misleading.

**The images live on GHCR.** `ghcr.io/mercuriusaalst/mercurius-{backend,frontend}`,
pushed by each app repo with its own `GITHUB_TOKEN`. GHCR packages are
**private by default** even when the source repo is public, and a private
package means `ImagePullBackOff` with no useful message in the pod events.
Either set each package's visibility to public once, under the org's Packages
settings, or set `imagePullSecrets` to a `kubernetes.io/dockerconfigjson`
Secret. The chart creates no pull secret of its own.

**Probes are TCP, not HTTP.** Neither app exposes a health endpoint. A TCP
probe proves Kestrel is listening, not that the app works — it will happily
report ready with a dead database. Add `/health` to the backend and swap the
probes over.

**Two resources outlive the release.** The database `Cluster` and the backend
media PVC carry `helm.sh/resource-policy: keep`, so uninstalling the release
leaves the data behind. Deleting them is a deliberate, manual act.

## Local checks

```sh
helm lint . --values ci/lint-values.yaml
helm template lan-party . --values ci/lint-values.yaml
helm template lan-party . --values ci/lint-values.yaml --values ci/backup-values.yaml
```

`ci/lint-values.yaml` is the minimum that renders. The backup fixture layers over
it. CI also checks that missing, empty, and null environment values fail during
rendering, and that valid values appear in the deployments without placeholders.

## Releasing

Conventional commits on `main`. release-please opens the release PR; merging it
bumps `version.txt` and `Chart.yaml`, pushes the OCI chart, and tells `gitops`
to roll the new version into dev.
