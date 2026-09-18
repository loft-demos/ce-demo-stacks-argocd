# ce-demo-stacks-argocd

A self-contained example of vCluster Platform **Stacks**: a dependency-ordered
graph where each task hands captured values to the ones after it.

Three tasks, two types:

```text
credentials ──────► backend ──────────► frontend
  Platform App        Argo CD App         Argo CD App
  generates the       syncs               syncs
  shared API token    charts/backend      charts/frontend
       │                    │                   ▲
       │ token              │ endpoint          │
       └────────────────────┴───────────────────┘
         frontend reads the token through its
         transitive path to credentials
```

Nothing here just waits for the thing before it. Each task is *configured from*
its predecessors: the generated API token and the backend's Service cluster IP
are captured after the task that produced them is healthy, then rendered into
the next task's values. Neither value can be known when the stack is written,
which is the thing Stacks do that sync waves and `dependsOn` in other tools do
not.

The graph deliberately mixes task types. Ordering and dataflow work identically
across the two, so an `app` task feeding an Argo CD task is the same handoff as
any other. What differs is the health gate, who creates the target namespace,
and whether a task can be retried by annotation; the notes below call out each
where it matters.

Requires vCluster Platform 4.12 or later. The `deploy.stacks` install path also
requires vCluster 0.37 or later.

## What is in here

```text
platform/                              applied to the Platform
  app-templates.yaml                   the App behind the credentials task
  argocd-application-templates.yaml    two ArgoCDApplicationTemplate blueprints
  stack-template.yaml                  the StackTemplate: the task DAG
  install/
    stack-instance.yaml                path A: run it on an existing tenant cluster
    virtual-cluster-template.yaml      path B: run it on every new tenant cluster
charts/                                synced by Argo CD, never read by the Stack
  backend/
  frontend/
```

`platform/` is applied with `kubectl` against the Platform's management API.
`charts/` is what Argo CD pulls from this repository. They are separate on
purpose: the Stack points Argo CD at a path and then addresses the result by
name, so nothing in `platform/` parses a chart.

## Prerequisites

- A vCluster Platform 4.12+ instance, and a project you can create objects in.
- An Argo CD connector configured on the Platform, and both the
  `argo-integration` and `apps` license features, since the graph uses one task
  of each type. The Platform registers the destination cluster with the Argo CD
  backend itself; there is no cluster to add by hand.
- A tenant cluster to deploy into, or a template that creates one.
- This repository pushed somewhere the Argo CD connector can read.

Optional, for the Gateway API exposure described below: Gateway API installed on
the **control plane cluster** with a controller and a Gateway to share, plus one
health-check override on your Argo CD. Nothing extra is needed inside the tenant
cluster as vCluster installs the Gateway API CRDs there itself.

## Quick start

**1. Push this repository** and note its URL. Argo CD reads `charts/backend` and
`charts/frontend` from it.

**2. Connect to the management API.**

```bash
vcluster platform connect management
```

**3. Apply the blueprints and the stack template.** All of them are
cluster-scoped.

```bash
kubectl apply -f platform/app-templates.yaml
kubectl apply -f platform/argocd-application-templates.yaml
kubectl apply -f platform/stack-template.yaml
```

Apply them in that order. `stack-template.yaml` references all three blueprints
by name, and a `templateRef` is resolved on every reconcile, so a stack applied
first simply reports `TemplateNotFound` until they arrive.

**4. Run it.** Edit `platform/install/stack-instance.yaml` and set the project
namespace, the tenant cluster name, the owner, and your `repoURL`:

```bash
kubectl apply -f platform/install/stack-instance.yaml
```

`spec.owner` matters. The stack's children deploy as that identity, and its
access to the destination and to referenced project secrets is what governs. The
UI and the management API fill it in from the creating identity; a hand-applied
StackInstance has to set it itself.

For the per-tenant path instead, use
`platform/install/virtual-cluster-template.yaml`, which declares the same stack
under `deploy.stacks` in a VirtualClusterTemplate. Every tenant cluster created
from that template gets its own StackInstance, owned by the tenant cluster and
deleted with it. That path needs `integrations.argoCD` enabled with a connector
in the **same** `vcluster.yaml`.

## Exposing the frontend through a shared Gateway

Off by default. Turned on, the frontend gets an HTTPRoute served by a Gateway
that lives in the **control plane cluster** while the the tenant runs no Gateway, no
controller, and no ingress of its own.

### How it works

Two halves of vCluster's Gateway API sync meet in the middle:

```text
control plane cluster                         tenant cluster
─────────────────────                         ──────────────
gateway-system/shared-gateway  ──import──►    gateway-system/shared-gateway
  (real Gateway + controller)                   (read-only mirror)
                                                        ▲
                                                        │ parentRefs
loft-default-v-<tenant>/                                │
  frontend-x-demo-frontend-x-…  ◄──sync──     demo-frontend/frontend
  (HTTPRoute, parentRefs                        (HTTPRoute the chart creates)
   rewritten to the real Gateway)
```

- `sync.fromHost.gateways` mirrors the control plane Gateway into the tenant,
  read-only, at whatever namespace and name the mapping gives it. The tenant can
  see it and attach to it, but cannot edit it.
- `sync.toHost.gatewayApi.httpRoutes` syncs the tenant's HTTPRoute outward into
  the tenant's host namespace, rewriting `parentRefs` back to the real Gateway
  and `backendRefs` to the synced Services.

So the chart writes an ordinary HTTPRoute against an ordinary Gateway. The sync
config is what makes that Gateway a shared one.

vCluster installs the Gateway API CRDs into the tenant cluster itself when
HTTPRoute sync is on, so there is nothing to pre-install there.

### What the control plane cluster needs

A Gateway whose listener accepts routes from the tenant's host namespace. The
tenant's routes land in `loft-default-v-<tenant>`, not in a namespace you named,
so a listener restricted to `Same` will not serve them:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shared-gateway
  namespace: gateway-system
spec:
  gatewayClassName: <your class>
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All      # or a Selector matching the vCluster host namespaces
```

### The tenant side

`platform/install/virtual-cluster-template.yaml` carries the sync config:

```yaml
sync:
  fromHost:
    gatewayClasses:
      enabled: true
    gateways:
      enabled: true
      mappings:
        byName:
          "gateway-system/shared-gateway": "gateway-system/shared-gateway"
      allowedRoutes:
        defaultVirtualNamespacePolicy:
          from: All
      status:
        exposeAddresses: true
  toHost:
    services:
      enabled: true
    gatewayApi:
      httpRoutes:
        enabled: true
```

Four of those are load-bearing and easy to get wrong:

| Setting | Why |
| --- | --- |
| `mappings.byName` | Required whenever `fromHost.gateways` is on, and the mapping is what decides the namespace and name the tenant sees. A wildcard key must map to a wildcard target, and the target namespace may not be the vCluster's own host namespace. |
| `allowedRoutes.defaultVirtualNamespacePolicy.from: All` | The default is `Same`. Without this, a route in `demo-frontend` cannot attach to a Gateway mirrored into `gateway-system`. |
| `toHost.services.enabled` | The Gateway routes to the Service on the control plane cluster, so the backendRef only resolves if the Service is synced out. |
| `toHost.gatewayApi.gateways` left off | Routes travel outward; tenants do not create Gateways. Leaving Gateway sync off keeps it that way. |

`status.exposeAddresses: true` is optional but makes the demo easier and it lets
the tenant see where to point DNS:

```bash
vcluster connect stacks-demo
kubectl get gateway -n gateway-system shared-gateway
```

### What your Argo CD needs

One health-check override, or the frontend Application sits in Progressing
forever and the stack task eventually times out.

This is a vCluster bug, not a configuration mistake. `statusToVirtual` in the
HTTPRoute syncer translates the parent reference from host to tenant, but copies
`observedGeneration` across verbatim. That field is defined by Kubernetes API
convention as the generation of *the object it appears on*, and the host and
tenant copies have independent generations. They track each other only until
something writes the host copy on its own, which the Platform's own sleep-mode
agent does: it adds a RequestMirror filter so route traffic can refresh the
tenant's last-activity timestamp. vCluster then deliberately keeps that filter
(`preserveRequestMirrorFilters`, driven by the
`vcluster.loft.sh/preserve-request-mirror-filters` annotation the agent sets), so
the host spec is a permanent superset of the tenant spec and the host generation
stays permanently ahead. The tenant route ends up reporting `generation: 1`
alongside `observedGeneration: 2`, and nothing ever writes generation 2 to the
tenant object.

Argo CD's built-in HTTPRoute health check implements the Gateway API freshness
convention faithfully: `isParentGenerationObserved` skips any parent whose
conditions carry an `observedGeneration` that differs from `metadata.generation`.
Every parent gets skipped, so the check falls through to its last branch,
`Progressing` with "Waiting for HTTPRoute status", and `Accepted=True` and
`ResolvedRefs=True` are never read.

This bites only clients that read the per-parent conditions. kstatus looks for
`status.observedGeneration` at the object root, which HTTPRoute does not have, so
Flux is not affected by this path, and `kubectl wait --for=condition=` cannot
reach conditions nested under `status.parents[]` at all.

The same gap is in the TLSRoute, BackendTLSPolicy and imported-Gateway syncers.
Argo CD gates TLSRoute and BackendTLSPolicy on `observedGeneration` the same way;
its Gateway check has no generation guard, so an imported Gateway carries the
same wrong field without tripping Argo CD today.

Until that is fixed upstream, judge the route by its conditions instead. In the
`argo-cd` Helm chart this goes under `configs.cm`, which passes values through
verbatim:

```yaml
configs:
  cm:
    resource.customizations.health.gateway.networking.k8s.io_HTTPRoute: |
      local hs = {}
      if obj.status ~= nil and obj.status.parents ~= nil and #obj.status.parents > 0 then
        for _, parent in ipairs(obj.status.parents) do
          if parent.conditions ~= nil then
            for _, condition in ipairs(parent.conditions) do
              if (condition.type == "Accepted" or condition.type == "ResolvedRefs")
                 and condition.status ~= "True" then
                hs.status = "Degraded"
                hs.message = condition.message
                return hs
              end
            end
          end
        end
        hs.status = "Healthy"
        hs.message = "Route accepted"
        return hs
      end
      hs.status = "Progressing"
      hs.message = "Waiting for HTTPRoute status"
      return hs
```

The same key with `_TLSRoute` or `_BackendTLSPolicy` covers those kinds if you
sync them.

The tradeoff is real but small: without the generation check, a route whose spec
was just edited reads Healthy from the previous generation's conditions until the
controller catches up. `argocd-cm` is re-read live, so no restart is needed, but
a Hard Refresh on the Application forces re-evaluation immediately.

### Turning it on

Via the tenant cluster template, it is already wired: supply the `hostname`
parameter and the stack sets `exposeThroughGateway: "true"` for you.

For a StackInstance against an existing tenant cluster, that cluster's
`vcluster.yaml` needs the sync block above, and the stack needs the parameters:

```yaml
spec:
  parameters:
    exposeThroughGateway: "true"
    hostname: demo.example.com
    # gatewayName / gatewayNamespace default to shared-gateway / gateway-system,
    # named as the tenant sees them after the import mapping.
```

Point DNS for that hostname at the Gateway's address, and the demo ends on a URL
rather than a kubectl pod.

## Watching it run

Aggregate phase plus one line per task:

```bash
kubectl get stackinstance demo-app -n p-my-project -o jsonpath='{.status.phase}{"\n"}{range .status.tasks[*]}{.name}{"\t"}{.phase}{"\t"}{.reason}{"\t"}{.message}{"\n"}{end}'
```

Expect `credentials` to reach `Healthy` first, then `backend`, then `frontend`,
each with a short `CapturingOutputs` in between while its value is read.

The published output, through its own subresource rather than status:

```bash
kubectl get --raw /apis/management.loft.sh/v1/namespaces/p-my-project/stackinstances/demo-app/outputs
```

`backendEndpoint` is published; the token is captured but deliberately not, which
is what `publishedOutputs` is for.

Check the result end to end. The frontend proxies to the backend using the
address and token the stack handed over, so this returns the backend's JSON:

```bash
vcluster connect my-tenant-cluster
kubectl -n demo-frontend run curl --rm -it --image=curlimages/curl --restart=Never -- \
  curl -s http://frontend:8080/api/
```

With the Gateway exposure on, the same check is just a browser tab, or:

```bash
curl -s https://demo.example.com/api/
```

The backend rejects anything that does not carry the generated token, which is
what makes the handoff observable rather than merely plausible:

```bash
kubectl -n demo-backend run curl --rm -it --image=curlimages/curl --restart=Never -- \
  curl -s -o /dev/null -w '%{http_code}\n' http://backend:8080/
# 401
```

## How the handoff actually works

A task payload passes through more than one render, and each layer owns its own
`{{ }}`. This is why the example uses `templateRef` rather than an inline Argo CD
template: nothing has to be escaped.

| Stage | Who runs it | What happens |
| --- | --- | --- |
| 1 | Stack parameter render | `{{ .Values.backendNamespace }}` is filled into each task's `parameters` and its output sources |
| 2 | Stack output render | `{{ .Outputs.credentials.token }}` and `{{ .Outputs.backend.endpoint }}` become real values in the consuming task's `parameters`, each once its producer is healthy and the value is captured |
| 3 | The child's own controller | the blueprint's `{{ .Values.backendEndpoint }}` is filled, producing `backend.url: "http://10.96.0.42:8080"` in `source.helm.values` |
| 4 | Helm | the chart, or the App's manifests, renders that into the final Kubernetes objects |

Stages 1 and 2 use a restricted function set: no `lookup`, no `include`, no
`tpl`, no `randAlphaNum`. Anything needing those belongs at stage 4.

A consuming task needs a `dependsOn` path to the producer. The edge may be
transitive, which is how the frontend reads a `credentials` output while listing
only `backend` in `dependsOn`. With no path at all the reference is rejected
outright rather than resolving to nothing.

### Why the token is an app task

Stage 4 is the only one with cluster access, and only for an App. An App's
`config.manifests` is handed to Helm as written and rendered against the real
cluster, so `lookup` and `randAlphaNum` both work:

```yaml
{{- $existing := lookup "v1" "Secret" .Release.Namespace $name -}}
{{- if and $existing (index ($existing.data | default dict) "token") -}}
{{- $token = index $existing.data "token" | b64dec -}}
{{- else -}}
{{- $token = randAlphaNum 32 -}}
{{- end }}
```

Argo CD renders manifests client-side with no cluster access. A chart it syncs
cannot look up whether it already generated a secret, so an inline
`randAlphaNum` produces a new value on every render, and with `selfHeal: true`
that is a permanent diff. Generating the token in an App and capturing it is the
way to have it both generated and stable.

This is also why the `{{ ... }}` above is safe to write in the App: the Stack
renders a task's *own payload*, and an app task using `templateRef` carries
nothing but the reference and its parameter values. The App object is never
touched by stages 1 and 2.

## The contract between the Stack and the charts

The Stack addresses the charts' output by name and never reads them, so these
have to agree. `charts/README.md` has the full version.

| Stack | Where it is set |
| --- | --- |
| `backendServiceName` parameter, default `backend` | backend chart Service, named after the release, with `helm.releaseName: backend` in the Argo CD source |
| `backendSecretName` parameter, default `backend-api` | `secretName` parameter on the `demo-api-token` App |
| `fromSecret.key: token` | the key the App's manifest writes |
| `backendNamespace` parameter | `defaultNamespace` on the App, and the backend Argo CD `destination.namespace` |

The `backendNamespace` row is the one to watch. An App's `defaultNamespace` is
not templated, so it is a literal in `platform/app-templates.yaml`. It is also
the release namespace, and therefore the only namespace the `credentials` task's
outputs may be read from. Change one side without the other and the capture
fails rather than reading the wrong thing.

## Trying the charts without a Platform

Both charts render standalone:

```bash
helm template backend charts/backend --namespace demo-backend \
  --set backend.token=example-token

helm template frontend charts/frontend --namespace demo-frontend \
  --set backend.url=http://10.96.0.42:8080 \
  --set backend.token=example-token
```

`charts/README.md` has the by-hand install, where generating the token, reading
the cluster IP back out, and passing both to the releases that need them is done
manually. That is exactly the work the stack's three tasks automate.

## Troubleshooting

**The stack is Pending or Degraded.** Read the per-task reason from the status
command above. `WaitingForDependencies` is normal early on. `OwnerRequired` means
`spec.owner` is unset. `TemplateNotFound` means the blueprints were not applied,
or the name does not match.

**A `deploy.stacks` entry produced no StackInstance at all.** An entry that is
blocked or invalid is never written, so there is nothing for `kubectl get
stackinstances` to show. The signal is on the VirtualClusterInstance:

```bash
kubectl get virtualclusterinstance my-tenant-cluster -n p-my-project \
  -o jsonpath='{range .status.conditions[?(@.type=="StacksSynced")]}{.status}{"\t"}{.reason}{"\t"}{.message}{"\n"}{end}'
```

`StackBlocked` with an Argo CD task usually means `integrations.argoCD` is not
enabled with a connector in the same `vcluster.yaml`.

**A chart failed to render with a `fail` message.** Both charts refuse an empty
`backend.token`, and the frontend also refuses an empty `backend.url`, on
purpose: silently installing a backend that trusts an empty token, or a frontend
pointing nowhere, would be worse than a failed sync. An empty value means the
capture did not happen, so look at the producing task rather than the chart.

**Requests to the frontend come back 401.** The backend and the frontend
disagree about the token. Both should be carrying the same value from the
`credentials` task, so re-check that the `credentials` output was captured, and
that a rotation did not reach only one of them:

```bash
kubectl -n demo-backend get secret backend-api -o jsonpath='{.data.token}' | base64 -d
```

**The Argo CD Application sits in Progressing and the syncer reports "the object
has been modified".** The route's rule needs an explicit `name`, which the chart
sets. Without one, two controllers rewrite each other forever: when the tenant
cluster can sleep, the Platform agent stamps a generated name onto the *host*
rule so it can attach a RequestMirror filter, and vCluster then rewrites the host
spec from the still-unnamed tenant rule and strips the name back off. The giveaway
is a tenant route stuck at `generation: 1` whose status reports a much higher
`observedGeneration` — the host copy is churning while the tenant copy is not.

**The Application is Synced but stuck in Progressing, health details "Waiting
for HTTPRoute status".** The route is fine; Argo CD's built-in health check is
reading the `observedGeneration` on `status.parents[].conditions[]` as stale,
because vCluster copies it from the host copy. Confirm by comparing the two numbers on the tenant route:

```bash
kubectl get httproute -n demo-frontend frontend \
  -o jsonpath='{.metadata.generation}{"\t"}{.status.parents[0].conditions[0].observedGeneration}{"\n"}'
```

Different numbers with `Accepted=True` means you need the health-check override
from **What your Argo CD needs** above. Note the task timeout: an Application that
never reports Healthy takes the stack task down with it.

**The HTTPRoute is not being served.** Check the three places it has to land.
In the tenant, the mirror must exist and the route must have attached:

```bash
vcluster connect stacks-demo
kubectl get gateway -n gateway-system shared-gateway
kubectl get httproute -n demo-frontend frontend -o jsonpath='{.status.parents[*].conditions[*]}{"\n"}'
```

A `NotAllowedByListeners` condition means a namespace policy rejected it: either
`allowedRoutes.defaultVirtualNamespacePolicy` on the tenant side, or the real
Gateway's own `allowedRoutes` on the control plane side, which has to accept
routes from the tenant's host namespace. Then confirm it synced outward:

```bash
kubectl get httproute -n loft-default-v-stacks-demo
```

Nothing there means `sync.toHost.gatewayApi.httpRoutes` is off, or the tenant
never accepted the route in the first place.

**Retrying.** The two task types differ here. An `argoCDApplication` task is
retried with a hard refresh plus a sync; an `app` task reports
`RetryNotApplicable`, because an AppInstance re-runs only when its spec changes.
A failed `credentials` task is fixed by editing the task, not by retrying it.

```bash
kubectl annotate stackinstance demo-app -n p-my-project \
  platform.vcluster.com/stack-retry=all --overwrite
```

The annotation is cleared once acted on, and the result shows up as a `Retried`
event. A retry of `all` also re-captures outputs from a healthy task, which is
the only way to force a re-capture without editing the task.

## Cleanup

```bash
kubectl delete -f platform/install/stack-instance.yaml
kubectl delete -f platform/stack-template.yaml
kubectl delete -f platform/argocd-application-templates.yaml
kubectl delete -f platform/app-templates.yaml
```

Deleting the StackInstance removes the Applications it owns. The example sets
`prunePolicy: Prune`, so a task removed from the stack takes its Application with
it rather than being reported as orphaned.

## Further reading

- [What are Stacks](https://vcluster.com/docs/platform/understand/what-are-stacks)
- [Create a Stack template](https://vcluster.com/docs/platform/administer/templates/create-stack-templates)
- [Use a Stack](https://vcluster.com/docs/platform/use-platform/apps/use-stacks)
- [Troubleshoot Stacks](https://vcluster.com/docs/platform/troubleshoot/stacks)
- [`deploy.stacks` in vcluster.yaml](https://vcluster.com/docs/vcluster/configure/vcluster-yaml/deploy#platform-stacks)
