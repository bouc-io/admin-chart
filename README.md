# admin-chart

Helm chart for the **admin UI** (`monochrome-admin-ui`), the browser front end for managing LLM
providers and assignments, MCP servers, organizations, and global or org-level instructions.

Despite the bare `-chart` name, this is not a legacy or secondary chart: it is the chart that
actually serves the admin UI, and it is reconciled continuously. The companion `admin-api-chart`
deploys the API this UI talks to.

## What it deploys

A single stateless Deployment serving static assets, plus a ClusterIP Service. **No database, no
cache, and no subchart dependencies**, which is why this chart has no `dependencies:` in
`Chart.yaml`.

| Object | Name |
|---|---|
| Deployment | `<release>-admin-chart` |
| Service | `<release>-admin-chart` (port 80 to `service.targetPort`) |
| PodDisruptionBudget | `<release>-admin-chart-pdb` (when `pdb.create`) |
| HorizontalPodAutoscaler | `<release>-admin-chart` (when `autoscaling.enabled`) |
| Ingress | `<release>-admin-chart` (when `ingress.enabled`) |

The container name inside the pod is `{{ .Chart.Name }}`, i.e. `admin-chart`, so use
`-c admin-chart` with `kubectl exec`.

Note that `serviceAccount.create` defaults to **false** here, unlike the API charts. The pods run
under the namespace default ServiceAccount unless you set a name.

## Values

Values live in three files, matching the convention used across bouc.io charts. There is no plain
`values.yaml`.

| File | Purpose |
|---|---|
| `base.values.yaml` | Values that never change between environments |
| `lcl.values.yaml` | Local cluster overrides |
| `snbx.values.yaml` | Sandbox overrides |

> In the cluster, FluxCD supplies values from generated ConfigMaps via `valuesFrom:`, not from these
> files directly. They are the source the ConfigMaps are generated from.

Both environment files are short: they carry the `environment` and `auth` blocks and nothing else,
because everything structural is already correct in `base.values.yaml`.

### Image

`image.registry` and `image.repository` are separate values, joined by the `admin-chart.image`
helper. The registry half is what a relocating operator overrides, either per release or through
`global.imageRegistry`. A `repository` whose first path segment already looks like a host is used
verbatim and `registry` is ignored, which keeps older full-string values rendering correctly.

`image.tag` is empty in `base.values.yaml` on purpose. An empty tag falls back to `Chart.AppVersion`
for a manual install, and in the cluster FluxCD's image automation writes a concrete tag in.

The repository path ends in `admin-mirror-ui` rather than `monochrome-admin-ui`. That mismatch
between the local directory name and the published artifact name is deliberate and load-bearing, so
do not "correct" it.

### Front-end configuration

This is a Vite single-page app, so its configuration arrives as `VITE_*` environment variables read
at container start:

| Value | Meaning |
|---|---|
| `environment.VITE_IDENTITY_PROVIDER` | Which auth provider the UI drives, e.g. `keycloak` |
| `environment.VITE_SSO_SERVER_URL` | Identity provider base URL |
| `environment.VITE_OAUTH_REALM` | Realm holding the platform's users |
| `environment.VITE_OAUTH_CLIENT_ID` | OAuth client this UI authenticates as |
| `environment.VITE_API_URL` | Base URL of the platform API the UI calls |
| `environment.VITE_OLLAMA_API_URL` | LLM endpoint exposed through the same API host |
| `environment.VITE_AVAILABLE_MODELS` | List of `{id, name, description}` entries offered in the UI |

`VITE_AVAILABLE_MODELS` is a list in values but a **single environment variable** in the pod: the
template JSON-encodes it and escapes the quotes. Keep it a list in values and let the template do
the encoding.

The `${CLUSTER_DOMAIN}` placeholders in the environment files are substituted by FluxCD at the
Kustomization step, not by Helm.

All three values files also carry an `auth` block (`auth.issuer`, `auth.jwksUri`). **No template
reads it.** Token validation happens at the edge, so those values are currently inert and changing
them has no effect on the rendered manifests.

### Disruption budget

`pdb.create` is true by default with `minAvailable: "0%"`, which permits full voluntary eviction.
That is the intended posture for a stateless UI: the object exists so the field is explicit and easy
to tighten, not to block node drains.

## Probes

`livenessProbe` and `readinessProbe` render only when set, and the shipped values leave both empty
(`{}`). Pods count as ready as soon as the container starts. Supply probes per environment if you
want real health gating.

## Local usage

```bash
helm lint . -f base.values.yaml -f lcl.values.yaml
helm template test . -f base.values.yaml -f lcl.values.yaml
helm install admin . -f base.values.yaml -f lcl.values.yaml
```

The values files layer: `base` first, then exactly one environment file. Pass them to `helm lint`
too: a bare `helm lint .` fails on a nil pointer, because there is no `values.yaml` for it to fall
back on. There are no chart dependencies, so no `helm dependency build` step is needed.
`templates/tests/test-connection.yaml` provides a `helm test` connectivity check.

> The chart must be published to the chart registry by CI before FluxCD can reconcile it. Pushing
> chart source to git is not enough.
