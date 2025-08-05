CKA Notes after the Retake

My plan for preparing will be as the following:

1. Review the CKA 2025 Github repo and master all the questions.
2. Redo all Kodecloud mock exams especially number 5.
3. Review killershell questions and practice.

------------------------------------------------------------

🔹 Task:

1. List all cert-manager-related Custom Resource Definitions (CRDs) in the cluster.
   Save the filtered list to: /home/student/cert-manager-crds.txt

2. Extract the definition of the subject field from the Certificate CRD schema and save it as YAML.
   The output should show the structure of the .spec.subject field defined in the CRD.
   Save this output to: /home/student/subject-schema.yaml

------------------------------------------------------------

# 1. List cert-manager CRDs and extract subject field from Certificate CRD

Commands:

    # List cert-manager CRDs and save to file
    k get crd | grep cert-manager | cut -d ' ' -f 1 > cert-manager-crds.txt

    # Extract subject field schema from Certificate CRD
    kubectl get crd certificates.cert-manager.io -o json | jq '.spec.versions[].schema.openAPIV3Schema.properties.spec.properties.subject'

------------------------------------------------------------

# Step 3: Install cri-dockerd and configure system

Commands:

    # Install cri-dockerd .deb package
    sudo dpkg -i cri-dockerd_0.3.1.3-0.ubuntu-jammy_amd64.deb

    # Enable and start cri-dockerd service
    sudo systemctl enable cri-docker.service
    sudo systemctl start cri-docker.service

------------------------------------------------------------

# 5. Gateway API Migration (Ingress ➝ Gateway + HTTPRoute)

gateway.yaml

    apiVersion: gateway.networking.k8s.io/v1
    kind: Gateway
    metadata:
      name: tls-basic
    spec:
      gatewayClassName: nginx
      listeners:
      - name: foo-https
        protocol: HTTPS
        port: 443
        hostname: example.com
        tls:
          certificateRefs:
          - kind: Secret
            group: ""
            name: example-tls

------------------------------------------------------------

httproute.yaml

    apiVersion: gateway.networking.k8s.io/v1
    kind: HTTPRoute
    metadata:
      name: example-httproute
    spec:
      parentRefs:
      - name: tls-basic
      hostnames:
      - "example.com"
      rules:
      - matches:
        - path:
            type: PathPrefix
            value: /
        backendRefs:
        - name: my-service
          port: 80

------------------------------------------------------------

# 6. Add Sidecar and Shared Volume

    volumes:
    - name: shared-data
      emptyDir: {}

    containers:
    - name: main
      image: myapp
      volumeMounts:
      - name: shared-data
        mountPath: /app/shared

    - name: sidecar
      image: alpine
      command: ["/bin/sh", "-c", "tail -f /dev/null"]
      volumeMounts:
      - name: shared-data
        mountPath: /sidecar/shared

------------------------------------------------------------

# 7. PVC + Mount to Deployment

```md
# Kubernetes PVC and Deployment Example

## 1. PersistentVolumeClaim (PVC)

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
  namespace: example
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 8Gi
  storageClassName: slow
  selector:
    matchLabels:
      release: stable

## 2. Deployment that mounts the PVC

apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  namespace: example
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp-container
        image: nginx
        volumeMounts:
        - name: my-storage
          mountPath: /usr/share/nginx/html
      volumes:
      - name: my-storage
        persistentVolumeClaim:
          claimName: myclaim

---
```

### Notes:

- The PVC and Deployment **must be in the same namespace**.
- The PVC will bind to a PV with matching labels and specs.
- You can change the mount path inside the container as needed.







# Installing ArgoCD with Helm

## Step 1: Render ArgoCD manifests to a file (skip CRDs)

Run this command in your terminal to generate Kubernetes manifests locally and save them to a file. This skips CRDs because you usually manage CRDs separately.

```
helm template argocd argo/argo-cd \
  --version 5.51.6 \
  --namespace argocd \
  --skip-crds > argocd.yaml
  ```

- Generates YAML manifests for ArgoCD without CRDs.
- Output saved in `argocd.yaml` for review or manual apply.

---

## Step 2: Install ArgoCD using Helm (including CRDs)

Use this command to install ArgoCD fully, including CRDs, directly into the `argocd` namespace. The `--create-namespace` flag ensures the namespace is created if missing.

```
helm install argocd argo/argo-cd \
  --version 5.51.6 \
  --namespace argocd \
  --create-namespace \
  --set installCRDs=true
  ```

- Helm installs ArgoCD and its CRDs automatically.
- Resources will be namespaced under `argocd`.

---

#  9. Divide Node Resources (Include Init Containers)

## Scenario: One Pod Pending Due to Insufficient Node Resources

### Context:
You have a Deployment with **3 replicas**. Each pod contains:
- An **init container** with a CPU and memory request.
- A **main container** with identical CPU and memory request.

The total request for each pod = `initContainer` + `mainContainer`.

Since init containers run sequentially, only one init container runs at a time per pod, but Kubernetes still ensures the **max(resource of init container, resource of all app containers)** is available on the node before scheduling.

### YAML: Deployment with Init and Main Containers
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resource-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      initContainers:
      - name: init-container
        image: busybox
        command: ["sh", "-c", "sleep 5"]
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
      containers:
      - name: main-container
        image: nginx
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
```

### Node Resource Summary (`kubectl describe node <node-name>`)
```
Capacity:
  CPU:                1000m
  Memory:             1Gi

Allocated resources:
  CPU Requests:       750m (75%)
  Memory Requests:    768Mi (75%)

Non-terminated Pods:  (2)
  Namespace   Name           CPU Requests   Memory Requests
  ---------   ----           ------------   ----------------
  default     pod-xxxxx      250m           256Mi
  default     pod-yyyyy      250m           256Mi

Events:
  Type     Reason            Message
  ----     ------            -------
  Warning  FailedScheduling  0/1 nodes are available: 1 Insufficient cpu, 1 Insufficient memory.
```

### Explanation:
- Each pod requires 250m CPU + 256Mi memory (init container) and the same for main container.
- Kubernetes schedules using the **maximum** request (not sum) between init and main containers. So each pod requires:
  - **250m CPU**
  - **256Mi Memory**
- With 3 replicas → total request = 750m CPU, 768Mi memory.
- If the node only has 1000m CPU and 1Gi memory:
  - Two pods are scheduled successfully.
  - The **third pod remains pending** due to **Insufficient CPU/Memory**.

### Key Notes:
- `kubectl describe node` shows allocated vs. capacity.
- Pod pending events indicate exact reason (Insufficient cpu/memory).
- Always plan resource requests carefully to match available capacity.

------------------------------------------------------------


# 10. Least Permissive NetworkPolicy

Select one out of the three NetworkPolicy manifests which matches the scenario described in the question (e.g., backend pods should only have ingress traffic from the frontend namespace).


------------------------------------------------------------

https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml

This is the all-in-one Calico manifest (non-operator install), and here’s how to handle it step-by-step including configuring the PodCIDR:
Step 1: Get your PodCIDR

```
kubectl get node <node-name> -o jsonpath='{.spec.podCIDR}'
```
10.244.0.0/16

```
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml

Search for this block inside the file (under env: in the calico-node container):

# - name: CALICO_IPV4POOL_CIDR
#   value: "192.168.0.0/16"

kubectl apply -f calico.yaml

 Verify Calico is running:

 kubectl get pods -n kube-system
```

# 12. HPA - Horizontal Pod Autoscaler

```
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-deployment
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 2
  maxReplicas: 6
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
```

