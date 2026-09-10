# Deploying a Persistent Application for Zidd3.0

## Project Overview

This repository contains my Kubernetes practice and implementation work for the **Zidd3.0 Persistent Application assignment**.

The objective was to redesign a small web application so that it:

- Retains important application data when Pods are deleted or recreated.
- Stores sensitive credentials securely.
- Uses centralized, non-sensitive configuration.
- Runs on a specific Kubernetes node.
- Is reachable internally within the cluster.
- Is accessible from outside the cluster.

For practice, I used **Nginx** as the application image, as permitted by the assignment.

## Assignment Requirements

| Requirement | Kubernetes resource or feature used |
|---|---|
| Persistent application data | PersistentVolume, PersistentVolumeClaim, and a volume mount |
| Secure credentials | Secret |
| Centralized configuration | ConfigMap |
| Scheduling on a specific node | Node label and `nodeSelector` |
| Internal cluster access | ClusterIP Service |
| External access | NodePort Service |
| Web application | Nginx Deployment |

## Repository Structure

```text
.
├── namespace.yaml
├── persistent-volume.yaml
├── persistent-volume-claim.yaml
├── secret.yaml
├── configmap.yaml
├── deployment.yaml
├── service-clusterip.yaml
├── service-nodeport.yaml
└── README.md
```

> File names may differ if the manifests were organized differently during practice. The resource definitions and commands in this README describe the implementation approach.

## Prerequisites

- A working Kubernetes cluster.
- `kubectl` installed and configured.
- Permission to create namespaces and workload resources.
- At least one worker node available for scheduling.
- A storage path or storage class suitable for the chosen cluster.

Check the cluster before beginning:

```bash
kubectl cluster-info
kubectl get nodes -o wide
kubectl get namespaces
```

## Implementation Steps

### 1. Create the namespace

I created a separate namespace to keep the project resources organized and isolated.

```bash
kubectl apply -f namespace.yaml
kubectl get namespace zidd3
```

If a namespace manifest was not used, the namespace can be created directly:

```bash
kubectl create namespace zidd3
```

### 2. Label the target node

The application had to run on a specific Kubernetes node. I labeled the selected node and referenced that label in the Deployment.

First, I identified the node name:

```bash
kubectl get nodes
```

Then I added the label:

```bash
kubectl label node <target-node-name> workload=zidd3
```

I verified the label with:

```bash
kubectl get nodes --show-labels
```

The Deployment uses the following scheduling rule:

```yaml
spec:
  template:
    spec:
      nodeSelector:
        workload: zidd3
```

This ensures that the Pod is scheduled only on a node having the `workload=zidd3` label.

### 3. Configure persistent storage

The original problem was data loss after Pod recreation. To solve this, I created a PersistentVolume and a PersistentVolumeClaim.

The claim is mounted inside the Nginx container at:

```text
/usr/share/nginx/html
```

The important relationship is:

```text
PersistentVolume -> PersistentVolumeClaim -> Deployment volumeMount
```

I applied and verified the storage resources:

```bash
kubectl apply -f persistent-volume.yaml -n zidd3
kubectl apply -f persistent-volume-claim.yaml -n zidd3
kubectl get pv
kubectl get pvc -n zidd3
```

The PVC should reach the `Bound` state before the application is started.

> In a production cluster, a dynamic StorageClass is generally preferable to a manually configured host path. For a local practice cluster, a static PV can be useful for demonstrating persistence.

### 4. Create centralized configuration

I used a ConfigMap for non-sensitive application configuration. This keeps configuration separate from the container image and Deployment code.

Example configuration values include:

```yaml
data:
  APP_NAME: zidd3-web
  APP_ENV: practice
```

I applied the ConfigMap with:

```bash
kubectl apply -f configmap.yaml -n zidd3
kubectl get configmap -n zidd3
kubectl describe configmap zidd3-config -n zidd3
```

The values can be exposed to the container as environment variables using `envFrom` or individual `configMapKeyRef` entries.

### 5. Store credentials in a Secret

Sensitive values must not be stored in a ConfigMap or committed as plain text. I created a Kubernetes Secret for credentials such as a username and password.

Example command:

```bash
kubectl create secret generic zidd3-secret \\
  --from-literal=APP_USERNAME=admin \\
  --from-literal=APP_PASSWORD='change-me' \\
  --namespace zidd3
```

I verified only the Secret metadata rather than exposing its decoded values:

```bash
kubectl get secret zidd3-secret -n zidd3
kubectl describe secret zidd3-secret -n zidd3
```

The Deployment consumes the Secret through environment variables:

```yaml
envFrom:
  - secretRef:
      name: zidd3-secret
```

In a real environment, I would use an external secret manager or another protected secret-delivery process and would never commit real credentials to Git.

### 6. Deploy the Nginx application

I created an Nginx Deployment with the following characteristics:

- Nginx container image.
- Replica count suitable for the practice cluster.
- ConfigMap-based configuration.
- Secret-based credentials.
- Persistent storage mounted into the web root.
- Node selection using `nodeSelector`.

I deployed it with:

```bash
kubectl apply -f deployment.yaml -n zidd3
kubectl get deployment -n zidd3
kubectl get pods -n zidd3 -o wide
```

I checked the rollout status:

```bash
kubectl rollout status deployment/zidd3-nginx -n zidd3
```

The Pod should be in the `Running` state and should appear on the labeled target node.

### 7. Create internal access with ClusterIP

A ClusterIP Service provides stable access to the application from other workloads inside the cluster.

```bash
kubectl apply -f service-clusterip.yaml -n zidd3
kubectl get service -n zidd3
```

The application can be reached internally using the Service DNS name, for example:

```text
http://zidd3-nginx.zidd3.svc.cluster.local
```

The exact hostname depends on the Service name and namespace used in the manifests.

### 8. Create external access with NodePort

To access the application from outside the cluster, I created a NodePort Service.

```bash
kubectl apply -f service-nodeport.yaml -n zidd3
kubectl get service -n zidd3
```

The external URL follows this pattern:

```text
http://<node-ip>:<node-port>
```

For a Minikube cluster, the service can be opened with:

```bash
minikube service zidd3-nginx-nodeport -n zidd3
```

For other local clusters, I used the reachable node IP and the assigned NodePort.

## Validation and Testing

### Check all resources

```bash
kubectl get all -n zidd3
kubectl get pv
kubectl get pvc -n zidd3
kubectl get configmap -n zidd3
kubectl get secret -n zidd3
```

### Verify node placement

```bash
kubectl get pods -n zidd3 -o wide
```

The `NODE` column should show the target node.

### Verify the mounted data

I created a test file in the mounted Nginx web directory:

```bash
kubectl exec -n zidd3 deploy/zidd3-nginx -- \\
  sh -c 'echo "Zidd3.0 persistent data" > /usr/share/nginx/html/persistence-test.txt'
```

I confirmed that the file was served by Nginx:

```bash
kubectl exec -n zidd3 deploy/zidd3-nginx -- \\
  cat /usr/share/nginx/html/persistence-test.txt
```

### Test persistence after Pod recreation

I deleted the running Pod:

```bash
kubectl delete pod -n zidd3 -l app=zidd3-nginx
```

I waited for Kubernetes to create a replacement Pod:

```bash
kubectl get pods -n zidd3 -w
```

After the new Pod became ready, I checked the file again:

```bash
kubectl exec -n zidd3 deploy/zidd3-nginx -- \\
  cat /usr/share/nginx/html/persistence-test.txt
```

The file was still available because the data was stored on the persistent volume rather than only in the Pod's writable layer.

### Test internal connectivity

I created a temporary troubleshooting Pod:

```bash
kubectl run curl-test -n zidd3 --rm -it --restart=Never \\
  --image=curlimages/curl -- \\
  curl -I http://zidd3-nginx
```

A successful HTTP response confirmed that the application was reachable through the internal Service.

### Test external connectivity

```bash
kubectl get service zidd3-nginx-nodeport -n zidd3
curl http://<node-ip>:<node-port>
```

A successful response confirmed that the application was accessible from outside the cluster through NodePort.

## Useful Troubleshooting Commands

```bash
kubectl describe pod <pod-name> -n zidd3
kubectl logs deployment/zidd3-nginx -n zidd3
kubectl describe pvc <pvc-name> -n zidd3
kubectl describe service <service-name> -n zidd3
kubectl get events -n zidd3 --sort-by=.lastTimestamp
```

Common issues and checks:

| Symptom | Checks |
|---|---|
| Pod remains Pending | Confirm the node label, node availability, PVC status, and resource capacity |
| PVC remains Pending | Check the StorageClass, PV capacity, access mode, and matching selectors |
| Service has no endpoints | Confirm the Service selector matches the Pod labels |
| Nginx shows the default page | Confirm the volume is mounted at the intended web-root path |
| External access fails | Check the node IP, NodePort, firewall rules, and cluster networking |
| Configuration is missing | Confirm the ConfigMap name and referenced keys |

## What I Practiced and Learned

- Pods are ephemeral, so important data should not be stored only inside the container filesystem.
- A PVC provides the application with a stable request for persistent storage.
- ConfigMaps are appropriate for non-sensitive configuration.
- Secrets are intended for sensitive values and should be protected from accidental exposure.
- Labels and selectors connect Kubernetes objects and also support scheduling decisions.
- `nodeSelector` is a simple way to constrain a Pod to a labeled node.
- A ClusterIP Service supports internal communication and service discovery.
- A NodePort Service exposes an application through a port on each node.
- Kubernetes manifest files make deployments repeatable and easier to review.
- Validation should include both resource inspection and an actual persistence test after Pod deletion.

## Cleanup

To remove the practice resources:

```bash
kubectl delete namespace zidd3
```

If the PV was configured with a host path or retained policy, verify and clean up the underlying storage according to the cluster setup.

## Conclusion

This practice implementation addresses the Zidd3.0 assignment by combining persistent storage, Secrets, ConfigMaps, node-specific scheduling, and internal and external Services around an Nginx Deployment. The key validation was deleting and recreating the Pod while confirming that the test data remained available.
