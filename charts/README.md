# Charts

What Argo CD actually syncs. The Stack in `platform/` never reads these charts;
it points Argo CD at them by path and then addresses their output by name.

```text
charts/
  backend/     synced by the stack's `backend` task, from path charts/backend
  frontend/    synced by the stack's `frontend` task, from path charts/frontend
```

Neither chart generates the API token. That is the job of the `credentials`
task, which is a Platform App rather than an Argo CD Application precisely
because these charts cannot do it; see the repository README for why. Both
charts *consume* the token.

Both charts render under `helm template` on their own, so they can be checked
without a Platform, an Argo CD instance, or a cluster.

## The contract between the Stack and this repository

The Stack does not read the charts. It addresses their output by name, so these
have to agree:

| Stack | Where it is set | Value |
| --- | --- | --- |
| `backendServiceName` parameter | backend chart Service name | `backend` |
| `backendSecretName` parameter | `secretName` parameter on the `demo-api-token` App | `backend-api` |
| `fromSecret.key` on the `token` output | key the App's manifest writes | `token` |
| `backendNamespace` parameter | `defaultNamespace` on the App, and the backend Argo CD `destination.namespace` | `demo-backend` |
| `frontendNamespace` parameter | frontend Argo CD `destination.namespace` | `demo-frontend` |

Both charts name their resources after the release, and the
`ArgoCDApplicationTemplate` sets `helm.releaseName` to `backend` and `frontend`,
which is what produces the names above.

The `backendNamespace` row is the one to watch. An App's `defaultNamespace` is
not templated, so it is a literal in `platform/app-templates.yaml`. It is also
the release namespace, and therefore the only namespace the `credentials` task's
outputs may be read from. Change one side without the other and the capture
fails rather than reading the wrong thing.

## backend

Serves a fixed JSON body on `/` and `ok` on `/healthz`.

- **Requires `backend.token`** and fails the render without it. The value is the
  shared token the `credentials` task generated; nginx returns `401` on `/`
  unless the request carries it as `X-Api-Token`. `/healthz` is deliberately
  left open so the probes do not need it.
- **Produces the Service the Stack captures.** `backend` is a ClusterIP Service,
  and the stack reads `{.spec.clusterIP}` off it with a `fromResource` output.
  The cluster assigns that address, so it is the clearest case of a value that
  has to be captured rather than configured.

The served config is a Secret rather than a ConfigMap because it carries the
token, and the Deployment has a `checksum/config` annotation over it so the pods
roll when the token or the config changes.

## frontend

An nginx reverse proxy in front of the backend. `/api/` is proxied to the backend
and carries the token as `X-Api-Token`; `/` returns a static body.

It requires two values and fails the render without them:

```yaml
backend:
  url: http://10.96.0.42:8080    # from the backend task's endpoint output
  token: <captured>              # from the credentials task's token output
```

Neither has a default, on purpose. They arrive from the Stack, and a frontend
that silently installed pointing nowhere, or sending an empty token, would be
worse than a failed sync.

It also renders an optional HTTPRoute, off unless `httpRoute.enabled` is set:

```yaml
httpRoute:
  enabled: true
  gateway:
    name: shared-gateway        # as the tenant sees it after the import mapping
    namespace: gateway-system
    sectionName: ""             # optional listener name
  hostnames:
    - demo.example.com          # optional
```

The route is ordinary Gateway API. What makes it reach a Gateway in the control
plane cluster is the tenant's sync config, which the repository README covers.
`gateway.name` is required when the route is enabled, and the chart fails the
render without it.

Two things about the route are load-bearing rather than decorative.

The rule carries an explicit `name`. On a tenant cluster that can sleep, the Platform agent adds a
RequestMirror filter to the host copy of the route and needs a named rule to
attach it to; vCluster correlates host and tenant rules by name when deciding
which filters to preserve. Leave the rule unnamed and the agent stamps a name on
the host while vCluster strips it again on the next sync, which never converges.

And the route needs a health-check override on the Argo CD side, because
vCluster copies `status.observedGeneration` verbatim from the host copy of the
route. The repository README has the override and the reasoning; without it the
Application never leaves Progressing even though the route is being served.

Note that the two values come from *different* tasks. The frontend lists only
`backend` in `dependsOn`, and reaches the `credentials` output through the
transitive path `frontend -> backend -> credentials`.

## Trying the charts without the Platform

```bash
helm template backend charts/backend --namespace demo-backend \
  --set backend.token=example-token

helm template frontend charts/frontend --namespace demo-frontend \
  --set backend.url=http://10.96.0.42:8080 \
  --set backend.token=example-token
```

Installed for real, the handoff the Stack automates is done by hand:

```bash
TOKEN=$(head -c 24 /dev/urandom | base64 | tr -dc 'A-Za-z0-9' | head -c 32)
kubectl create namespace demo-backend
kubectl -n demo-backend create secret generic backend-api --from-literal=token="$TOKEN"

helm install backend charts/backend -n demo-backend --set backend.token="$TOKEN"
ENDPOINT=$(kubectl -n demo-backend get svc backend -o jsonpath='{.spec.clusterIP}')

helm install frontend charts/frontend -n demo-frontend --create-namespace \
  --set backend.url="http://$ENDPOINT:8080" \
  --set backend.token="$TOKEN"
```

Generating that token, reading the cluster IP back out, and passing both to the
releases that need them is exactly what the `credentials` task, the `backend`
task's output, and the two sets of task parameters do.
