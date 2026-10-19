# ECK and Longhorn deployment

This repository deploys Longhorn storage, the ECK operator, and an Elasticsearch/Kibana stack. Install them in that order. The workload Helm chart is in `charts/eck-stack`; Longhorn and ECK have separate releases and values files.

## Prerequisites

- A reachable Kubernetes cluster and working `kubectl` and Helm 3 installations.
- Run these commands from the repository root in PowerShell.
- Longhorn disks mounted at the paths you intend to use. In the Longhorn UI, add the `hot`, `warm`, and `public` disk tags to the intended disks on each eligible node. The directories must be backed by the intended storage; tags alone do not create or resize disks.
- The hot and warm StorageClasses below request one Longhorn replica each. This is suitable for a small lab, but does not provide replica redundancy.

Check cluster access before installing:

```powershell
kubectl config current-context
kubectl get nodes
```

## 1. Install Longhorn

If the repository is not already configured, add it once and update the chart index:

```powershell
helm repo add longhorn https://charts.longhorn.io
helm repo update
```

Install or upgrade the pinned Longhorn chart version used by this project:

```powershell
helm upgrade --install longhorn longhorn/longhorn `
  --version 1.12.1 `
  --namespace longhorn-system `
  --create-namespace `
  --values .\deploy\longhorn\longhorn-values.yaml `
  --timeout 10m
```

Wait for Longhorn to start:

```powershell
kubectl rollout status daemonset/longhorn-manager -n longhorn-system --timeout=10m
kubectl rollout status deployment/longhorn-ui -n longhorn-system --timeout=10m
```

Configure the tagged disks in the Longhorn UI before deploying Elasticsearch. Apply the hot and warm StorageClasses:

```powershell
kubectl apply -f .\deploy\longhorn\storageclasses-longhorn.yaml
kubectl get storageclass
```

The default `longhorn` class stores master data on disks tagged `public`. `longhorn-hot` and `longhorn-warm` select disks tagged `hot` and `warm` respectively.

The Longhorn UI NodePort service is optional. For a lab cluster, you can apply it with:

```powershell
kubectl apply -f .\deploy\longhorn\longhorn-svc.yaml
```

It exposes the UI over HTTP on port `30080` of each node. Alternatively, use `kubectl port-forward` to access the UI locally.

## 2. Install ECK operator and CRDs

If the repository is not already configured, add it once and update the chart index:

```powershell
helm repo add elastic https://helm.elastic.co
helm repo update
```

Install the operator chart version matching the image tag in its values file:

```powershell
helm upgrade --install elastic-operator elastic/eck-operator `
  --version 3.4.1 `
  --namespace elastic-system `
  --create-namespace `
  --values .\deploy\eck-operator\eck-operator-values.yml `
  --timeout 10m
```

`installCRDs: true` in the operator values installs the ECK CRDs. `managedNamespaces: []` makes this operator watch all namespaces; the Elasticsearch and Kibana resources below are created in `elastic`.

Wait for the operator and verify the CRDs:

```powershell
kubectl rollout status statefulset/elastic-operator -n elastic-system --timeout=5m
kubectl get crd elasticsearches.elasticsearch.k8s.elastic.co kibanas.kibana.k8s.elastic.co
```

## 3. Deploy Elasticsearch and Kibana

The current workload values request `1Gi` for master, `0.4Gi` for hot, and `0.1Gi` for warm. These small sizes are for a lab or startup check; `0.1Gi` gives warm only about 102 MiB for data. Set the sizes you need in `charts/eck-stack/values.yaml` before the first install.

Install or upgrade the workload chart:

```powershell
helm upgrade --install eck-elk .\charts\eck-stack `
  --namespace elastic `
  --create-namespace `
  --values .\charts\eck-stack\values.yaml `
  --timeout 30m
```

Watch ECK create the Elasticsearch and Kibana pods and their PVCs:

```powershell
kubectl get elasticsearch,kibana,pods,pvc -n elastic --watch
```

In another terminal, inspect status and recent events if a pod or PVC remains Pending:

```powershell
helm status eck-elk -n elastic
kubectl get events -n elastic --sort-by=.metadata.creationTimestamp
```

The services use HTTPS with self-signed certificates. Once the pods are ready, access Elasticsearch at `https://<node-ip>:32000` and Kibana at `https://<node-ip>:32601`. The `elastic` user's generated password is in the `elasticsearch-es-elastic-user` Secret in namespace `elastic`.

## Existing installs and project files

`helm upgrade --install` can be rerun to apply changes. ECK does not allow shrinking an existing Elasticsearch volume claim template. Changing a storage size in `values.yaml` does not shrink an already-created PVC; plan any size reduction as a data migration or a fresh deployment where deleting the old data is acceptable.

The files `deploy/longhorn/storageclasses.yaml`, `deploy/longhorn/nodes-reference.yaml`, and `deploy/longhorn/live-settings.txt` are cluster snapshots/reference material. They contain live metadata or status and are not part of the install sequence. Use `storageclasses-longhorn.yaml` for the clean hot/warm StorageClass definitions.
