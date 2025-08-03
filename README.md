# Argo

* Udemy (6, 4)
* Labs (4)
* Lessons (4)
* Templates (3 + 3 = 6)
* Articles (4)

## CD

### Deploy ArgoCD to Kubernetes

```sh
kubectl create namespace argocd

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Login via UI

```sh
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### Port forward traffic

```sh
kubectl port-forward svc/argocd-example-app-service 9090:3000
```

### Show password

```sh
kubectl -nargocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

argocd login localhost:8080 # admin + pass above
```

### Add user

```sh
argocd login localhost:8080

kubectl edit cm argocd-cm -n argocd
argocd account update-password --account developer --new-password Semafor4!
```

### Modify permissions

```sh
kubectl edit cm argocd-rbac-cm -n argocd

apiVersion: v1
kind: ConfigMap
metadata:
  name: my-extra-rbac
  namespace: argocd
data:
  policy.csv: |
    p, role:synconly, applications, sync, */*, allow
    g, developer, role:synconly

data:
  policy.csv: |
    p, role:synconly, applications, sync, */*, allow
    g, developer, role:synconly
    policy.default: role:readonly
```

## Workflows

### Deploy Argo Workflows

```sh
kubectl create namespace argo
kubectl apply -n argo -f https://github.com/argoproj/argo-workflows/releases/latest/download/install.yaml
```

### Access the UI

```sh
# Omit authentication for now
kubectl patch deployment argo-server -n argo --type=json -p '[{"op":"replace","path":"/spec/template/spec/containers/0/args","value":["server","--auth-mode=server"]}]'

kubectl -n argo port-forward deployment/argo-server 2746:2746
# https://localhost:2746/workflows/argo
```

### Create role binding

```sh
kubectl create rolebinding default-admin --clusterrole=admin --serviceaccount=argo:default -n argo
```

### Create workflow template

```sh
kubectl apply -f dag-workflow-template.yaml
```

### Create workflow

```sh
kubectl create -f dag-workflow.yaml

# alternatively
kubectl apply -f dag-workflow.yaml
```

### Deploy CI CD pipeline

```sh
kubectl -n argo apply -f workflow-ci.yaml
```

## Rollouts

### Deploy Rollouts to Kubernetes

```sh
kubectl create namespace argo-rollouts

kubectl apply -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```

### Access Dashboard

```sh
kubectl argo rollouts dashboard
```

### Create rollout

```sh
kubectl apply -f rollout.yaml
```

### Create active service

```sh
kubectl apply -f active-service.yaml
```

### Create preview service

```sh
kubectl apply -f preview-service.yaml
```

### Perform a promotion

```sh
kubectl argo rollouts promote rollout-bluegreen

# Requires at least 2 revisions to make undo
kubectl argo rollouts undo rollout-bluegreen
kubectl argo rollouts undo rollout-bluegreen --to-revision=1
```

## Events

### Make steps: Deploy Argo Workflows, Access the UI from Argo Workflows

### Deploy Argo Events

```sh
kubectl create namespace argo-events

kubectl apply -n argo-events -f https://raw.githubusercontent.com/argoproj/argo-events/stable/manifests/install.yaml
```

### Install the Validating Admission Webhook

A Validating Admission Webhook checks every Argo Events CR before it’s stored in Kubernetes, catching mistakes early.

```sh
kubectl apply -n argo-events -f https://raw.githubusercontent.com/argoproj/argo-events/stable/manifests/install-validating-webhook.yaml
```

### Create the native EventBus

The EventBus is the messaging backbone (in-memory or NATS) that carries events between sources and sensors.

```sh
kubectl apply -n argo-events -f https://raw.githubusercontent.com/argoproj/argo-events/stable/examples/eventbus/native.yaml
```

### Deploy the Webhook EventSource

This spins up an HTTP server in Kubernetes that listens for external POSTs and converts them into Argo Events.

```sh
kubectl apply -n argo-events -f https://raw.githubusercontent.com/argoproj/argo-events/stable/examples/event-sources/webhook.yaml
```

### Grant Sensor permissions (RBAC)

Sensors need rights to read EventSources and write status.

```sh
kubectl apply -n argo-events -f https://raw.githubusercontent.com/argoproj/argo-events/master/examples/rbac/sensor-rbac.yaml
```

### Grant Workflow-triggering permissions (RBAC)

When a Sensor fires, it will submit Argo Workflows—so it needs create/get on Workflow objects.

```sh
kubectl apply -n argo-events -f https://raw.githubusercontent.com/argoproj/argo-events/master/examples/rbac/workflow-rbac.yaml
```

### Deploy the Sensor

The Sensor ties the EventSource to an Argo Workflow trigger (in this case, on any POST to /example).

```sh
kubectl apply -n argo-events -f https://raw.githubusercontent.com/argoproj/argo-events/stable/examples/sensors/webhook.yaml
```

### Expose the EventSource pod

Make the webhook HTTP server reachable on your localhost by port-forwarding it:

```sh
kubectl -n argo-events port-forward $(kubectl -n argo-events get pod -l eventsource-name=webhook -o name) 12000:12000 &
```

### Send a test webhook

Push a JSON payload to the EventSource; this generates an Argo Event that your Sensor will catch.

```sh
curl -X POST http://localhost:12000/example -H "Content-Type: application/json" -d '{"message":"this is my first webhook"}'
```
