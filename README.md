# ArgoCD "App of Apps" Demo

A minimal, self-contained demo of the ArgoCD [App of Apps](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/) pattern.

One **root** Application watches the [`apps/`](apps) directory. Every child
`Application` manifest committed there is discovered and synced automatically —
so onboarding a new workload is just "drop a file in `apps/` and push".

```
root-app.yaml            # the root Application — points at apps/
apps/                    # child Applications (watched by the root, recurse: true)
  nginx-demo.yaml        #   → plain manifests in THIS repo
  podinfo.yaml           #   → external Helm chart (podinfo)
  guestbook.yaml         #   → external Git repo (argocd-example-apps)
manifests/
  nginx-demo/            # the raw k8s YAML that nginx-demo.yaml deploys
    configmap.yaml
    deployment.yaml
    service.yaml
```

The three child apps deliberately cover the three common ArgoCD source types:

| Child app                | Source type      | Where it lives                         |
|--------------------------|------------------|----------------------------------------|
| `lensagents-nginx-demo`  | Directory        | this repo (`manifests/nginx-demo`)     |
| `lensagents-podinfo`     | Helm chart       | `stefanprodan.github.io/podinfo`       |
| `lensagents-guestbook`   | Directory        | `argoproj/argocd-example-apps` repo    |

All child apps deploy into the `lademo` namespace.

## Prerequisites

- A Kubernetes cluster with ArgoCD installed in the `argocd` namespace.

```sh
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

## Deploy

Apply the root Application. ArgoCD does the rest.

```sh
kubectl apply -f root-app.yaml
```

Watch it fan out:

```sh
kubectl get applications -n argocd
# lensagents-demo-root, lensagents-nginx-demo, lensagents-podinfo, lensagents-guestbook should all appear and go Synced/Healthy
```

## Try it

```sh
# nginx-demo
kubectl -n lademo port-forward svc/lensagents-nginx-demo 8080:80
# open http://localhost:8080

# podinfo
kubectl -n lademo port-forward svc/podinfo 9898:9898
# open http://localhost:9898
```

## Add another app

Create `apps/my-app.yaml` with an `Application` spec, commit, and push.
The root app picks it up on its next sync — no `kubectl apply` needed.

## Tear down

```sh
kubectl delete -f root-app.yaml
```

The `resources-finalizer.argocd.argoproj.io` finalizer cascades the delete to
every child Application and their workloads.

> **Note:** the podinfo Helm chart names its own Service `podinfo` (unaffected by
> the `lensagents-` prefix, which only applies to the ArgoCD Application names and
> the in-repo nginx-demo manifests).
>
> **Note:** `repoURL` in `root-app.yaml` and `apps/nginx-demo.yaml` points at
> `github.com/flavius-dinu/k8s_manifests`. If you fork this, update those URLs.
