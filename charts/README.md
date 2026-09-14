# Charts

What Argo CD actually syncs. The Stack in `platform/` never reads these charts;
it points Argo CD at them by path and then addresses their output by name.

```text
charts/
  backend/     synced by the stack's `backend` task, from path charts/backend
  frontend/    synced by the stack's `frontend` task, from path charts/frontend
```

Both charts render under `helm template` on their own, so they can be checked
without a Platform, an Argo CD instance, or a cluster. See the repository README
for the whole picture; this file covers the chart side.

## The contract between the Stack and the charts

The Stack does not read the charts. It addresses their output by name, so these
have to agree:

| Stack | Chart | Value |
| --- | --- | --- |
| `backendServiceName` parameter | backend Service name | `backend` |
| `backendSecretName` parameter | backend `tokenSecret.name` | `backend-api` |
| `fromSecret.key` on the `token` output | key in that Secret | `token` |
| `backendNamespace` parameter | Argo CD `destination.namespace` | `demo-backend` |
| `frontendNamespace` parameter | Argo CD `destination.namespace` | `demo-frontend` |

Both charts name their resources after the release, and the
`ArgoCDApplicationTemplate` sets `helm.releaseName` to `backend` and `frontend`,
which is what produces the names above.

## backend

Serves a fixed JSON body on `/` and `ok` on `/healthz`.

It produces the two values the Stack captures:

- **Service `backend`.** The Stack reads `{.spec.clusterIP}` off it with a
  `fromResource` output. The cluster assigns that address, so it is the clearest
  case of a value that has to be captured rather than configured.
- **Secret `backend-api`, key `token`.** Declared as a
  `kubernetes.io/service-account-token` Secret with no `data` block. Kubernetes'
  token controller fills `data.token` after the Secret exists, so the value is
  created by the cluster rather than by whoever installed the chart. The Stack
  reads it with a `fromSecret` output.

Two details worth copying into a real chart:

- **No `randAlphaNum`.** Argo CD renders manifests client-side with no cluster
  access, so a chart cannot look up whether it already generated a secret. A
  chart that generates one inline rotates it on every sync, and with
  `selfHeal: true` that is a permanent diff. Letting the cluster fill the value,
  and declaring no `data`, means Argo CD compares only the fields the manifest
  actually sets and leaves the generated token alone.
- **A sync wave on the ServiceAccount.** The token controller deletes an
  SA-token Secret whose ServiceAccount does not exist. Helm's install order
  already puts ServiceAccount before Secret; Argo CD sorts Secret first, so
  `argocd.argoproj.io/sync-wave: "-1"` on the ServiceAccount is what orders them
  there. That is the in-Application version of the same ordering problem the
  Stack solves between Applications with `dependsOn`.

## frontend

An nginx reverse proxy in front of the backend. `/api/` is proxied to the
backend and carries the API token as `X-Api-Token`; `/` returns a static body.

It requires two values and fails the render without them:

```yaml
backend:
  url: http://10.96.0.42:8080
  token: <captured>
```

Neither has a default, on purpose. They arrive from the Stack, and a frontend
that silently installed pointing nowhere would be worse than a failed sync.

The proxy config is a Secret rather than a ConfigMap because it carries the
token, and the Deployment has a `checksum/config` annotation over it so the pods
roll when a re-captured output changes the value.

## Trying the charts without the Platform

```bash
helm template backend charts/backend --namespace demo-backend
helm template frontend charts/frontend --namespace demo-frontend \
  --set backend.url=http://10.96.0.42:8080 \
  --set backend.token=example-token
```

Installed for real, the handoff the Stack automates is done by hand:

```bash
helm install backend charts/backend -n demo-backend --create-namespace
kubectl -n demo-backend get svc backend -o jsonpath='{.spec.clusterIP}'
kubectl -n demo-backend get secret backend-api -o jsonpath='{.data.token}' | base64 -d
```

Reading those two values and passing them to the frontend release is exactly
what the `backend` task's outputs and the `frontend` task's parameters do.
