# ce-demo-stacks-argocd

A self-contained example of vCluster Platform **Stacks**: a dependency-ordered
bundle of Argo CD Applications where one task hands captured values to the next.

Stacks can mix task types; this repository uses the Argo CD one throughout. The
other is an `app` task, which deploys a Platform App instead of an Argo CD
Application. The dependency and output mechanics are identical either way; what
differs is the health gate, who creates the target namespace, and whether a task
can be retried by annotation. The notes below call out each of those where it
matters.

Two tasks, one edge:

```text
backend ──────────────────────────────► frontend
   │  Argo CD Application                  Argo CD Application
   │  syncs charts/backend                 syncs charts/frontend
   │
   ├─ captures Service cluster IP  ─┐
   └─ captures API token           ─┴──► supplied as template parameters
```

The frontend does not just wait for the backend. It is *configured from* it: the
backend's Service cluster IP and API token are captured after it syncs and
rendered into the frontend's Helm values. Neither value can be known when the
stack is written, which is the thing Stacks do that sync waves and `dependsOn`
in other tools do not.

Requires vCluster Platform 4.12 or later. The `deploy.stacks` install path also
requires vCluster 0.37 or later.

## What is in here

```text
platform/                              applied to the Platform
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
- An Argo CD connector configured on the Platform, and the `argo-integration`
  license feature. The Platform registers the destination cluster with the Argo
  CD backend itself; there is no cluster to add by hand.
- A tenant cluster to deploy into, or a template that creates one.
- This repository pushed somewhere the Argo CD connector can read.

## Quick start

**1. Push this repository** and note its URL. Argo CD reads `charts/backend` and
`charts/frontend` from it.

**2. Connect to the management API.**

```bash
vcluster platform connect management
```

**3. Apply the blueprints and the stack template.** Both are cluster-scoped.

```bash
kubectl apply -f platform/argocd-application-templates.yaml
kubectl apply -f platform/stack-template.yaml
```

Apply them in that order. `stack-template.yaml` references the blueprints by
name, and a `templateRef` is resolved on every reconcile, so a stack applied
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

## Watching it run

Aggregate phase plus one line per task:

```bash
kubectl get stackinstance demo-app -n p-my-project -o jsonpath='{.status.phase}{"\n"}{range .status.tasks[*]}{.name}{"\t"}{.phase}{"\t"}{.reason}{"\t"}{.message}{"\n"}{end}'
```

Expect `backend` to reach `Healthy` before `frontend` leaves `Waiting`, with a
short `CapturingOutputs` in between while the two values are read.

The published output, through its own subresource rather than status:

```bash
kubectl get --raw /apis/management.loft.sh/v1/namespaces/p-my-project/stackinstances/demo-app/outputs
```

`backendEndpoint` is published; the token is captured but deliberately not, which
is what `publishedOutputs` is for.

Check the result end to end:

```bash
vcluster connect my-tenant-cluster
kubectl -n demo-frontend run curl --rm -it --image=curlimages/curl --restart=Never -- \
  curl -s http://frontend:8080/api/
```

That should return the backend's JSON, proxied through the frontend using the
address and token the stack handed over.

## How the handoff actually works

A task payload passes through more than one render, and each layer owns its own
`{{ }}`. This is why the example uses `templateRef` rather than an inline Argo CD
template: nothing has to be escaped.

| Stage | Who runs it | What happens |
| --- | --- | --- |
| 1 | Stack parameter render | `{{ .Values.backendNamespace }}` is filled into the task's `parameters` and its output sources |
| 2 | Stack output render | `{{ .Outputs.backend.endpoint }}` becomes `10.96.0.42` in the frontend task's `parameters`, once the backend is Synced and Healthy and the value is captured |
| 3 | ArgoCDApplication controller | the blueprint's `{{ .Values.backendEndpoint }}` is filled, producing `backend.url: "http://10.96.0.42:8080"` in `source.helm.values` |
| 4 | Argo CD and Helm | the chart renders that into the frontend's nginx config |

Stages 1 and 2 use a restricted function set: no `lookup`, no `include`, no
`tpl`, no `randAlphaNum`. Anything a real chart needs belongs in the chart, which
is rendered at stage 4.

A consuming task needs a `dependsOn` path to the producer. The edge may be
transitive, but with none at all the reference is rejected outright rather than
resolving to nothing.

## The contract between the Stack and the charts

The Stack addresses the charts' output by name and never reads them, so these
have to agree. `charts/README.md` has the full version.

| Stack | Chart |
| --- | --- |
| `backendServiceName` parameter, default `backend` | Service named after the release, with `helm.releaseName: backend` in the Argo CD source |
| `backendSecretName` parameter, default `backend-api` | `tokenSecret.name` |
| `fromSecret.key: token` | the key Kubernetes' token controller writes |
| `backendNamespace` parameter | Argo CD `destination.namespace` |

Two details in the backend chart exist specifically so the token is capturable
under Argo CD, and both are worth copying into real charts:

- The token Secret is a `kubernetes.io/service-account-token` declared with **no
  `data` block**, so the cluster fills it. A chart that generated it with
  `randAlphaNum` would rotate it on every sync, because Argo CD renders
  manifests client-side and cannot look up what it already created. With
  `selfHeal: true` that is a permanent diff.
- The ServiceAccount carries `argocd.argoproj.io/sync-wave: "-1"`. The token
  controller deletes an SA-token Secret whose ServiceAccount does not exist, and
  Argo CD sorts Secret before ServiceAccount. This is the in-Application version
  of the ordering problem the Stack solves between Applications.

## Trying the charts without a Platform

Both charts render standalone:

```bash
helm template backend charts/backend --namespace demo-backend
helm template frontend charts/frontend --namespace demo-frontend \
  --set backend.url=http://10.96.0.42:8080 \
  --set backend.token=example-token
```

Installed by hand, the handoff the Stack automates is done manually:

```bash
helm install backend charts/backend -n demo-backend --create-namespace
kubectl -n demo-backend get svc backend -o jsonpath='{.spec.clusterIP}'
kubectl -n demo-backend get secret backend-api -o jsonpath='{.data.token}' | base64 -d
```

Reading those two values and passing them to the frontend release is exactly what
the `backend` task's outputs and the `frontend` task's parameters do.

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

**The frontend failed to render.** The chart calls `fail` when `backend.url` or
`backend.token` is empty, on purpose: a frontend that silently installed pointing
nowhere would be worse than a failed sync. If the values are empty, the capture
did not happen, so look at the backend task first.

**Retrying.** Argo CD tasks are retryable by annotation; app tasks are not.

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
