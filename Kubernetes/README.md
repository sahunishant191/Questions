> All questions are **scenario-based, senior/advanced level** — covering real-world situations, debugging, architecture decisions, and trade-offs.

---

## Table of Contents

- [Q1 — Pod Stuck in CrashLoopBackOff — Debugging](#q1-pod-stuck-in-crashloopbackoff--debugging)
- [Q2 — Pod Scheduling Failures — Node Affinity, Taints, Tolerations](#q2-pod-scheduling-failures--node-affinity-taints-tolerations)
- [Q3 — Service Networking — ClusterIP vs NodePort vs LoadBalancer vs Ingress](#q3-service-networking--clusterip-vs-nodeport-vs-loadbalancer-vs-ingress)
- [Q4 — Rolling Updates, Rollbacks, and Deployment Strategies](#q4-rolling-updates-rollbacks-and-deployment-strategies)
- [Q5 — Resource Requests, Limits, and OOMKilled Pods](#q5-resource-requests-limits-and-oomkilled-pods)
- [Q6 — RBAC — Securing Multi-Tenant Clusters](#q6-rbac--securing-multi-tenant-clusters)
- [Q7 — Persistent Volumes, PVCs, and StatefulSets](#q7-persistent-volumes-pvcs-and-statefulsets)
- [Q8 — ConfigMaps, Secrets, and Configuration Management](#q8-configmaps-secrets-and-configuration-management)
- [Q9 — Horizontal Pod Autoscaler and Cluster Autoscaler](#q9-horizontal-pod-autoscaler-and-cluster-autoscaler)
- [Q10 — Network Policies — Microsegmentation in Kubernetes](#q10-network-policies--microsegmentation-in-kubernetes)

---

# Kubernetes

## Q1. Pod Stuck in CrashLoopBackOff — Debugging

### Question

> It's 2 AM. PagerDuty fires. Your payment microservice in production has all 5 pods in `CrashLoopBackOff`. The service was working fine 2 hours ago. Walk me through your complete debugging process — from `kubectl` commands to identifying the root cause and fixing it. Cover at least 5 different causes of CrashLoopBackOff.

### Real-Life Scenario

A developer pushed a config change at midnight via ArgoCD. The new deployment rolled out and all pods started crash-looping. Customers can't check out. The old ReplicaSet was scaled down to 0. You're the on-call engineer.

### Answer

#### Step 1: Quick Status Assessment

```bash
# What's happening?
kubectl get pods -n payments -o wide
# NAME                          READY   STATUS             RESTARTS   AGE     NODE
# payment-svc-7f8d9c-abc12     0/1     CrashLoopBackOff   8          15m     ip-10-0-1-42
# payment-svc-7f8d9c-def34     0/1     CrashLoopBackOff   8          15m     ip-10-0-2-18
# payment-svc-7f8d9c-ghi56     0/1     CrashLoopBackOff   7          15m     ip-10-0-1-42
# payment-svc-7f8d9c-jkl78     0/1     CrashLoopBackOff   8          15m     ip-10-0-3-55
# payment-svc-7f8d9c-mno90     0/1     CrashLoopBackOff   6          15m     ip-10-0-2-18

# Check events for the namespace
kubectl get events -n payments --sort-by='.lastTimestamp' | tail -20
```

#### Step 2: Inspect the Pod

```bash
# Describe one of the failing pods
kubectl describe pod payment-svc-7f8d9c-abc12 -n payments

# Key sections to look at:
# - State:          Waiting (CrashLoopBackOff)
# - Last State:     Terminated (Exit Code: 1, Reason: Error)
# - Restart Count:  8
# - Events:
#   Warning  BackOff   kubelet  Back-off restarting failed container
```

**Exit code interpretation:**

| Exit Code | Meaning |
|-----------|---------|
| 0 | Success (shouldn't crash-loop) |
| 1 | Application error (unhandled exception, missing config) |
| 126 | Permission denied (can't execute entrypoint) |
| 127 | Command not found (wrong entrypoint/image) |
| 137 | OOMKilled (killed by kernel, out of memory) |
| 139 | Segfault (segmentation fault) |
| 143 | SIGTERM (graceful shutdown, shouldn't loop) |

#### Step 3: Check Container Logs

```bash
# Current container logs (might be empty if it crashes instantly)
kubectl logs payment-svc-7f8d9c-abc12 -n payments

# Previous container logs (the one that crashed)
kubectl logs payment-svc-7f8d9c-abc12 -n payments --previous

# If multi-container pod, specify container:
kubectl logs payment-svc-7f8d9c-abc12 -n payments -c payment-app --previous

# Stream logs (useful if the container runs for a few seconds before crashing)
kubectl logs -f payment-svc-7f8d9c-abc12 -n payments
```

#### Step 4: Common Root Causes and Fixes

**Cause 1: Missing or Wrong Environment Variable / ConfigMap**

```bash
# Logs show:
# Error: DATABASE_URL environment variable is not set
# or
# Error: Cannot connect to redis://redis-prod:6379 — connection refused

# Check env vars
kubectl get pod payment-svc-7f8d9c-abc12 -n payments -o jsonpath='{.spec.containers[0].env}' | jq .

# Check ConfigMap
kubectl get configmap payment-config -n payments -o yaml

# Check if ConfigMap/Secret referenced exists
kubectl get configmap payment-config -n payments
# Error: configmaps "payment-config" not found  ← ROOT CAUSE

# Fix: Create the missing ConfigMap
kubectl create configmap payment-config -n payments \
  --from-literal=DATABASE_URL="postgresql://prod-db:5432/payments" \
  --from-literal=REDIS_URL="redis://redis-prod:6379"
```

**Cause 2: Wrong Image Tag or Image Pull Failure**

```bash
# Describe shows:
# Events:
#   Warning  Failed   kubelet  Error: ImagePullBackOff
#   Warning  Failed   kubelet  Failed to pull image "mycompany/payment:v2.5.1": not found

# Check the image
kubectl get pod payment-svc-7f8d9c-abc12 -n payments \
  -o jsonpath='{.spec.containers[0].image}'
# mycompany/payment:v2.5.1  ← tag doesn't exist!

# Fix: Correct the image tag
kubectl set image deployment/payment-svc \
  payment-app=mycompany/payment:v2.5.0 -n payments
```

**Cause 3: Liveness Probe Failure**

```bash
# Describe shows:
# Events:
#   Warning  Unhealthy  kubelet  Liveness probe failed: HTTP probe failed with statuscode: 503
#   Normal   Killing    kubelet  Container payment-app failed liveness probe, will be restarted

# The app starts slowly and the liveness probe kills it before it's ready
# Check probe config:
kubectl get deployment payment-svc -n payments -o yaml | grep -A10 livenessProbe

# Fix: Increase initialDelaySeconds or add startupProbe
kubectl edit deployment payment-svc -n payments
```

```yaml
# Add startupProbe (Kubernetes 1.20+) — gives the app time to start
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30       # 30 × 10s = 5 minutes to start
  periodSeconds: 10

livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 5     # After startupProbe passes
  periodSeconds: 10
  failureThreshold: 3
```

**Cause 4: OOMKilled (Exit Code 137)**

```bash
# Describe shows:
# Last State: Terminated
#   Reason:   OOMKilled
#   Exit Code: 137

# The container exceeded its memory limit
kubectl get pod payment-svc-7f8d9c-abc12 -n payments \
  -o jsonpath='{.spec.containers[0].resources}'
# {"limits":{"memory":"256Mi"},"requests":{"memory":"128Mi"}}

# 256Mi is too low for a Java app!

# Check actual usage before crash:
kubectl top pod payment-svc-7f8d9c-abc12 -n payments
# (If metrics-server is installed)

# Fix: Increase memory limit
kubectl patch deployment payment-svc -n payments -p \
  '{"spec":{"template":{"spec":{"containers":[{"name":"payment-app","resources":{"limits":{"memory":"512Mi"},"requests":{"memory":"256Mi"}}}]}}}}'
```

**Cause 5: File System or Permission Issue**

```bash
# Logs show:
# Error: EACCES: permission denied, open '/app/data/cache.db'
# or
# Error: read-only file system

# Check security context
kubectl get pod payment-svc-7f8d9c-abc12 -n payments \
  -o jsonpath='{.spec.containers[0].securityContext}'
# {"readOnlyRootFilesystem":true,"runAsNonRoot":true,"runAsUser":1000}

# Fix: Add an emptyDir volume for writable paths
```

```yaml
spec:
  containers:
    - name: payment-app
      volumeMounts:
        - name: cache-dir
          mountPath: /app/data
        - name: tmp-dir
          mountPath: /tmp
  volumes:
    - name: cache-dir
      emptyDir: {}
    - name: tmp-dir
      emptyDir: {}
```

#### Step 5: Immediate Recovery — Rollback

```bash
# While debugging, ROLLBACK to the last working version immediately
kubectl rollout undo deployment/payment-svc -n payments

# Verify rollback
kubectl rollout status deployment/payment-svc -n payments

# Check which revision we rolled back to
kubectl rollout history deployment/payment-svc -n payments

# Roll back to a specific revision
kubectl rollout undo deployment/payment-svc -n payments --to-revision=5
```

#### Step 6: Debug a Crashing Container Interactively

```bash
# If the container crashes too fast to exec into it:

# Option 1: Override the entrypoint to keep it alive
kubectl run debug-payment --image=mycompany/payment:v2.5.0 -n payments \
  --command -- sleep 3600

kubectl exec -it debug-payment -n payments -- /bin/sh
# Now you can inspect files, test connectivity, etc.

# Option 2: Ephemeral debug container (Kubernetes 1.25+)
kubectl debug payment-svc-7f8d9c-abc12 -n payments \
  --image=busybox \
  --target=payment-app \
  -it

# Option 3: Copy the pod with modified command
kubectl debug payment-svc-7f8d9c-abc12 -n payments \
  --copy-to=debug-pod \
  --container=payment-app \
  -- sleep 3600
```

---

## Q2. Pod Scheduling Failures — Node Affinity, Taints, Tolerations

### Question

> Your new deployment has 10 pods stuck in `Pending` state. The cluster has 20 nodes with plenty of CPU and memory available. Why aren't the pods being scheduled? Walk me through all possible reasons and how to use taints, tolerations, node affinity, and pod anti-affinity to control scheduling.

### Real-Life Scenario

You deployed a GPU-based ML inference service. The pods need to run on GPU nodes (`p3.2xlarge`), but they're stuck `Pending`. Meanwhile, regular application pods keep getting scheduled on GPU nodes, wasting expensive resources. You need to reserve GPU nodes for GPU workloads only.

### Answer

#### Step 1: Why is the Pod Pending?

```bash
# Check pod status
kubectl get pods -n ml-inference
# NAME                        READY   STATUS    RESTARTS   AGE
# inference-7f8d9c-abc12      0/1     Pending   0          10m

# Describe the pod — Events section tells you WHY
kubectl describe pod inference-7f8d9c-abc12 -n ml-inference

# Common events:
# "0/20 nodes are available: 20 Insufficient nvidia.com/gpu"
# "0/20 nodes are available: 10 node(s) had taint {gpu=true: NoSchedule}"
# "0/20 nodes are available: 5 node(s) didn't match Pod's node affinity"
# "0/20 nodes are available: 3 node(s) had volume node affinity conflict"
# "0/20 nodes are available: 20 Insufficient cpu"
```

#### All Reasons a Pod Can Be Pending

| Reason | Event Message | Fix |
|--------|--------------|-----|
| Not enough CPU/Memory | `Insufficient cpu` / `Insufficient memory` | Scale nodes or reduce requests |
| Taints without tolerations | `node(s) had taint that the pod didn't tolerate` | Add tolerations to pod |
| Node affinity mismatch | `node(s) didn't match Pod's node affinity` | Fix node labels or affinity rules |
| Pod anti-affinity | `node(s) didn't match pod anti-affinity rules` | Relax anti-affinity or add nodes |
| PVC not bound | `persistentvolumeclaim not found` or `unbound` | Create PV or fix StorageClass |
| Node selector | `node(s) didn't match node selector` | Label nodes correctly |
| Resource quota exceeded | `exceeded quota` | Increase quota or free resources |
| Too many pods on node | `Too many pods` | Increase `maxPods` on node |

#### Step 2: Taints and Tolerations — Reserve GPU Nodes

```bash
# Taint the GPU nodes — ONLY GPU workloads can run here
kubectl taint nodes gpu-node-1 gpu-node-2 gpu-node-3 \
  nvidia.com/gpu=true:NoSchedule

# Verify taints
kubectl describe node gpu-node-1 | grep -A3 Taints
# Taints: nvidia.com/gpu=true:NoSchedule
```

```yaml
# GPU workload — add toleration to run on tainted nodes
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inference-service
  namespace: ml-inference
spec:
  replicas: 5
  selector:
    matchLabels:
      app: inference
  template:
    metadata:
      labels:
        app: inference
    spec:
      tolerations:
        - key: "nvidia.com/gpu"
          operator: "Equal"
          value: "true"
          effect: "NoSchedule"
      containers:
        - name: inference
          image: mycompany/ml-inference:v3
          resources:
            limits:
              nvidia.com/gpu: 1    # Request 1 GPU
              memory: "8Gi"
            requests:
              cpu: "2"
              memory: "4Gi"
```

**Taint effects explained:**

```
NoSchedule      → New pods without toleration won't be scheduled
                  Existing pods stay
PreferNoSchedule → Kubernetes TRIES to avoid scheduling here
                   but will if no other option
NoExecute       → Existing pods without toleration are EVICTED
                  New pods without toleration won't be scheduled
```

```bash
# NoExecute example — drain GPU nodes of non-GPU workloads immediately
kubectl taint nodes gpu-node-1 \
  nvidia.com/gpu=true:NoExecute

# All pods without the toleration are evicted within tolerationSeconds
```

#### Step 3: Node Affinity — Schedule Pods on Specific Nodes

```yaml
# requiredDuringSchedulingIgnoredDuringExecution = HARD requirement (must match)
# preferredDuringSchedulingIgnoredDuringExecution = SOFT preference (try to match)

apiVersion: apps/v1
kind: Deployment
metadata:
  name: inference-service
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          # HARD: Must be on a GPU node
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: node.kubernetes.io/instance-type
                    operator: In
                    values:
                      - p3.2xlarge
                      - p3.8xlarge
                      - g4dn.xlarge
          # SOFT: Prefer ap-south-1a zone
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 70
              preference:
                matchExpressions:
                  - key: topology.kubernetes.io/zone
                    operator: In
                    values:
                      - ap-south-1a
      containers:
        - name: inference
          image: mycompany/ml-inference:v3
```

```bash
# Label nodes for affinity matching
kubectl label nodes ip-10-0-1-42 node.kubernetes.io/instance-type=p3.2xlarge
kubectl label nodes ip-10-0-1-42 workload-type=gpu

# Verify labels
kubectl get nodes --show-labels | grep gpu
```

#### Step 4: Pod Anti-Affinity — Spread Pods Across Nodes

```yaml
# Ensure no two inference pods land on the same node
spec:
  affinity:
    podAntiAffinity:
      # HARD: Never put 2 inference pods on the same node
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - inference
          topologyKey: kubernetes.io/hostname

      # SOFT: Try to spread across AZs
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchExpressions:
                - key: app
                  operator: In
                  values:
                    - inference
            topologyKey: topology.kubernetes.io/zone
```

#### Step 5: Pod Topology Spread Constraints (Kubernetes 1.19+)

```yaml
# More granular than anti-affinity — control max skew
spec:
  topologySpreadConstraints:
    - maxSkew: 1                             # Max difference between zones
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule       # Hard constraint
      labelSelector:
        matchLabels:
          app: inference
    - maxSkew: 1
      topologyKey: kubernetes.io/hostname    # Also spread across nodes
      whenUnsatisfiable: ScheduleAnyway      # Soft constraint
      labelSelector:
        matchLabels:
          app: inference
```

**Result:** If you have 3 AZs and 6 pods, each AZ gets exactly 2 pods.

#### Troubleshooting

```bash
# Debug scheduling decisions
kubectl get events --field-selector reason=FailedScheduling -n ml-inference

# Check node resources available
kubectl describe node gpu-node-1 | grep -A5 "Allocated resources"
# CPU Requests: 3800m (95%)  ← Node is almost full!
# Memory Requests: 7Gi (87%)

# Check which pods are on which nodes
kubectl get pods -n ml-inference -o wide

# Simulate scheduling (dry-run)
kubectl run test-schedule --image=busybox --dry-run=client \
  --overrides='{"spec":{"nodeSelector":{"workload-type":"gpu"}}}' -o yaml
```

---

## Q3. Service Networking — ClusterIP vs NodePort vs LoadBalancer vs Ingress

### Question

> Explain the 4 ways to expose a Kubernetes service. A new team member asks: "When do I use ClusterIP vs LoadBalancer vs Ingress?" Design a production networking setup for 15 microservices where some are internal-only, some need external access, and some need WebSocket support.

### Real-Life Scenario

Your company has 15 microservices:
- 3 are public-facing APIs (need HTTPS from the internet)
- 5 are internal APIs (only other services call them)
- 2 are gRPC services (need HTTP/2)
- 3 are background workers (no networking needed)
- 1 is a WebSocket server (long-lived connections)
- 1 is an admin dashboard (restricted to office IP)

### Answer

#### The 4 Service Types Compared

```
                    Internet
                       │
               ┌───────┴───────┐
               │   Ingress     │    ← L7 (HTTP/HTTPS), path-based routing
               │  Controller   │       SSL termination, single LB for all
               └───────┬───────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
   ┌────┴────┐   ┌────┴────┐   ┌────┴────┐
   │ Service │   │ Service │   │ Service │   ← ClusterIP (internal)
   │ (API-1) │   │ (API-2) │   │ (Admin) │
   └────┬────┘   └────┬────┘   └────┬────┘
        │              │              │
   ┌────┴────┐   ┌────┴────┐   ┌────┴────┐
   │  Pods   │   │  Pods   │   │  Pods   │
   └─────────┘   └─────────┘   └─────────┘
```

| Type | Layer | Use Case | Creates AWS LB? | Cost |
|------|-------|----------|-----------------|------|
| ClusterIP | L4 | Internal services | No | Free |
| NodePort | L4 | Dev/debugging (avoid in prod) | No | Free |
| LoadBalancer | L4 | Single service needing external access | Yes (1 per service!) | ~$18/month each |
| Ingress | L7 | Multiple services behind one LB | Yes (1 for all) | ~$18/month total |

#### ClusterIP — Internal Services

```yaml
# For the 5 internal APIs — only reachable within the cluster
apiVersion: v1
kind: Service
metadata:
  name: user-service
  namespace: backend
spec:
  type: ClusterIP          # Default — internal only
  selector:
    app: user-service
  ports:
    - port: 80             # Service port (what other pods use)
      targetPort: 8080     # Container port
      protocol: TCP

# Other services access it via:
#   http://user-service.backend.svc.cluster.local
#   or simply: http://user-service (within same namespace)
```

#### NodePort — Dev/Debugging Only

```yaml
# Exposes on every node's IP at a high port (30000-32767)
apiVersion: v1
kind: Service
metadata:
  name: debug-service
spec:
  type: NodePort
  selector:
    app: debug-app
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 31080      # Access via <any-node-ip>:31080

# ❌ Don't use in production:
# - Exposes ports on ALL nodes (security risk)
# - Port range limited to 30000-32767
# - No SSL termination
# - Client needs to know node IPs
```

#### LoadBalancer — Single Service External Access

```yaml
# Creates an AWS NLB/ALB per service — expensive for many services
apiVersion: v1
kind: Service
metadata:
  name: payment-api
  namespace: backend
  annotations:
    # AWS Load Balancer Controller annotations
    service.beta.kubernetes.io/aws-load-balancer-type: "external"
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
spec:
  type: LoadBalancer
  selector:
    app: payment-api
  ports:
    - port: 443
      targetPort: 8080
      protocol: TCP

# ⚠️ Problem: 15 services = 15 load balancers = $270/month just for LBs!
```

#### Ingress — Production Setup (Recommended)

```yaml
# ONE Ingress Controller (NGINX/ALB) handles ALL external services
# Install NGINX Ingress Controller:
# helm install ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx

# Ingress resource — routes traffic by hostname/path
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  namespace: backend
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/use-regex: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.mycompany.com
        - admin.mycompany.com
      secretName: mycompany-tls
  rules:
    # Public API
    - host: api.mycompany.com
      http:
        paths:
          - path: /v1/payments
            pathType: Prefix
            backend:
              service:
                name: payment-api
                port:
                  number: 80
          - path: /v1/orders
            pathType: Prefix
            backend:
              service:
                name: order-api
                port:
                  number: 80
          - path: /v1/users
            pathType: Prefix
            backend:
              service:
                name: user-api
                port:
                  number: 80

    # Admin dashboard — IP restricted
    - host: admin.mycompany.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: admin-dashboard
                port:
                  number: 80
```

```yaml
# IP whitelist for admin dashboard
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: admin-ingress
  annotations:
    nginx.ingress.kubernetes.io/whitelist-source-range: "203.0.113.0/24,10.0.0.0/8"
spec:
  rules:
    - host: admin.mycompany.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: admin-dashboard
                port:
                  number: 80
```

#### WebSocket Support

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: websocket-ingress
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    nginx.ingress.kubernetes.io/connection-proxy-header: "keep-alive"
    nginx.ingress.kubernetes.io/upstream-hash-by: "$remote_addr"   # Sticky sessions
spec:
  rules:
    - host: ws.mycompany.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: websocket-server
                port:
                  number: 80
```

#### gRPC Services

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grpc-ingress
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
    - hosts:
        - grpc.mycompany.com
      secretName: grpc-tls
  rules:
    - host: grpc.mycompany.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: grpc-service
                port:
                  number: 50051
```

#### Complete Architecture for 15 Microservices

| Service | Type | Exposure | How |
|---------|------|----------|-----|
| payment-api | ClusterIP + Ingress | External HTTPS | Ingress path `/v1/payments` |
| order-api | ClusterIP + Ingress | External HTTPS | Ingress path `/v1/orders` |
| catalog-api | ClusterIP + Ingress | External HTTPS | Ingress path `/v1/catalog` |
| user-service | ClusterIP | Internal only | `user-service.backend.svc` |
| auth-service | ClusterIP | Internal only | `auth-service.backend.svc` |
| notification-svc | ClusterIP | Internal only | `notification-svc.backend.svc` |
| inventory-svc | ClusterIP | Internal only | `inventory-svc.backend.svc` |
| search-svc | ClusterIP | Internal only | `search-svc.backend.svc` |
| ml-inference (gRPC) | ClusterIP + Ingress | External gRPC | Ingress with `GRPC` backend |
| recommendation (gRPC) | ClusterIP | Internal gRPC | Direct pod-to-pod |
| order-worker | None (headless) | No networking | Background job |
| email-worker | None | No networking | Background job |
| report-worker | None | No networking | Background job |
| websocket-server | ClusterIP + Ingress | External WS | Ingress with timeout annotations |
| admin-dashboard | ClusterIP + Ingress | External (IP restricted) | Ingress with whitelist |

#### Troubleshooting

```bash
# Service not reachable
kubectl get svc -n backend
kubectl get endpoints payment-api -n backend
# If ENDPOINTS is <none> → selector doesn't match any pods

# Check pod labels match service selector
kubectl get pods -n backend --show-labels
kubectl get svc payment-api -n backend -o jsonpath='{.spec.selector}'

# DNS not resolving
kubectl run dns-test --image=busybox --rm -it -- nslookup payment-api.backend.svc.cluster.local

# Ingress not working
kubectl get ingress -n backend
kubectl describe ingress main-ingress -n backend
# Check the ingress controller logs:
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx
```

---

## Q4. Rolling Updates, Rollbacks, and Deployment Strategies

### Question

> Your team deploys to production 5 times a day. Last week, a bad deployment caused 3 minutes of errors before anyone noticed. Design a zero-downtime deployment strategy. Explain rolling updates, blue/green, canary, and how `maxSurge` / `maxUnavailable` work. Show how to rollback instantly.

### Real-Life Scenario

You deploy a new version of the order service. The new version has a bug where orders above $10,000 fail silently. Rolling update replaces all pods within 2 minutes. By the time the team notices (3 minutes later), 47 high-value orders were lost. You need a safer deployment strategy.

### Answer

#### Strategy 1: Rolling Update (Default)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2          # Create 2 extra pods during update (12 total max)
      maxUnavailable: 1    # Allow 1 pod to be down during update (9 minimum)
  template:
    spec:
      containers:
        - name: order-app
          image: mycompany/order-service:v3.2.0
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 10
```

**How `maxSurge` and `maxUnavailable` interact:**

```
Replicas: 10, maxSurge: 2, maxUnavailable: 1

Step 1: Create 2 new pods (v3.2.1)     → Total: 12 (10 old + 2 new)
Step 2: 2 new pods pass readiness      → Terminate 2 old pods
Step 3: Create 2 more new pods         → Total: 12 (8 old + 4 new)
Step 4: Repeat until all 10 are new    → Total: 10 (0 old + 10 new)

At NO point are fewer than 9 pods serving traffic.
```

```bash
# Watch the rollout
kubectl rollout status deployment/order-service -n backend

# Check rollout history
kubectl rollout history deployment/order-service -n backend
# REVISION  CHANGE-CAUSE
# 1         Initial deployment
# 2         Update to v3.1.0
# 3         Update to v3.2.0
# 4         Update to v3.2.1
```

#### Instant Rollback

```bash
# ROLLBACK to previous version (takes ~30 seconds)
kubectl rollout undo deployment/order-service -n backend

# Rollback to a specific revision
kubectl rollout undo deployment/order-service -n backend --to-revision=2

# Pause a bad rollout in progress
kubectl rollout pause deployment/order-service -n backend

# Resume after investigation
kubectl rollout resume deployment/order-service -n backend
```

#### Strategy 2: Blue/Green Deployment

```yaml
# Blue deployment (current production — v3.2.0)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-blue
  labels:
    app: order-service
    version: blue
spec:
  replicas: 10
  selector:
    matchLabels:
      app: order-service
      version: blue
  template:
    metadata:
      labels:
        app: order-service
        version: blue
    spec:
      containers:
        - name: order-app
          image: mycompany/order-service:v3.2.0

---
# Green deployment (new version — v3.2.1)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-green
  labels:
    app: order-service
    version: green
spec:
  replicas: 10
  selector:
    matchLabels:
      app: order-service
      version: green
  template:
    metadata:
      labels:
        app: order-service
        version: green
    spec:
      containers:
        - name: order-app
          image: mycompany/order-service:v3.2.1
```

```yaml
# Service — switch traffic by changing selector
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service
    version: blue        # ← Change to "green" to switch traffic
  ports:
    - port: 80
      targetPort: 8080
```

```bash
# Switch traffic from blue to green (instant, zero-downtime)
kubectl patch svc order-service -n backend \
  -p '{"spec":{"selector":{"version":"green"}}}'

# Rollback — switch back to blue
kubectl patch svc order-service -n backend \
  -p '{"spec":{"selector":{"version":"blue"}}}'

# After verifying green works, delete blue
kubectl delete deployment order-service-blue -n backend
```

#### Strategy 3: Canary Deployment

```yaml
# Production deployment (90% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-stable
spec:
  replicas: 9           # 9 out of 10 = 90% traffic
  selector:
    matchLabels:
      app: order-service
      track: stable
  template:
    metadata:
      labels:
        app: order-service
        track: stable
    spec:
      containers:
        - name: order-app
          image: mycompany/order-service:v3.2.0

---
# Canary deployment (10% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-canary
spec:
  replicas: 1           # 1 out of 10 = 10% traffic
  selector:
    matchLabels:
      app: order-service
      track: canary
  template:
    metadata:
      labels:
        app: order-service
        track: canary
    spec:
      containers:
        - name: order-app
          image: mycompany/order-service:v3.2.1

---
# Service selects BOTH — traffic split by pod count
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service      # Matches both stable AND canary pods
  ports:
    - port: 80
      targetPort: 8080
```

```bash
# Monitor canary vs stable error rates
# Check canary logs for errors
kubectl logs -l track=canary -n backend --tail=100

# If canary is healthy, gradually increase:
kubectl scale deployment order-service-canary --replicas=3 -n backend
kubectl scale deployment order-service-stable --replicas=7 -n backend
# Now 30% canary, 70% stable

# If canary fails, kill it:
kubectl scale deployment order-service-canary --replicas=0 -n backend
kubectl scale deployment order-service-stable --replicas=10 -n backend
```

#### Strategy 4: Argo Rollouts (Advanced — Weighted Canary)

```yaml
# More sophisticated canary with automatic promotion
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: order-service
spec:
  replicas: 10
  strategy:
    canary:
      steps:
        - setWeight: 5         # 5% traffic to canary
        - pause: { duration: 5m }
        - setWeight: 20        # 20%
        - pause: { duration: 5m }
        - setWeight: 50        # 50%
        - pause: { duration: 10m }
        - setWeight: 100       # Full rollout
      canaryService: order-service-canary
      stableService: order-service-stable
      trafficRouting:
        nginx:
          stableIngress: order-service-ingress
```

#### Comparison

| Strategy | Downtime | Rollback Speed | Resource Usage | Risk Level |
|----------|----------|---------------|----------------|------------|
| Rolling Update | Zero | ~30 sec | 1.2x (during update) | Medium |
| Blue/Green | Zero | Instant (~1 sec) | 2x (both running) | Low |
| Canary | Zero | Instant | 1.1x | Very Low |
| Recreate | YES | Slow | 1x | High |

---

## Q5. Resource Requests, Limits, and OOMKilled Pods

### Question

> Your Java application pods keep getting OOMKilled in production. The developers say "it works fine locally with 2GB RAM." The pod memory limit is set to 512Mi. Meanwhile, other pods are starving for CPU because one pod with no limits is consuming 4 full CPU cores. Explain requests, limits, QoS classes, and how to properly size resources.

### Real-Life Scenario

Monday morning:
- 3 pods are `OOMKilled` (Java app with 512Mi limit, but JVM heap alone needs 256Mi + metaspace + native memory)
- Cluster nodes are at 95% CPU because one data-processing pod has no CPU limits
- The scheduler can't place new pods because requested resources are fully allocated, but actual usage is only 30%

### Answer

#### Requests vs Limits — The Fundamental Difference

```
requests = "Minimum guaranteed resources for scheduling"
           → Scheduler uses this to place pods on nodes
           → Pod is GUARANTEED this much

limits   = "Maximum allowed resources"
           → If pod exceeds memory limit → OOMKilled
           → If pod exceeds CPU limit → throttled (NOT killed)
```

```yaml
spec:
  containers:
    - name: order-app
      resources:
        requests:
          cpu: "500m"        # 0.5 CPU cores — guaranteed minimum
          memory: "256Mi"    # 256 MB — guaranteed minimum
        limits:
          cpu: "2000m"       # 2 CPU cores — maximum allowed (throttled beyond)
          memory: "1Gi"      # 1 GB — maximum allowed (OOMKilled beyond)
```

#### QoS Classes (Quality of Service)

```yaml
# 1. Guaranteed — requests == limits (highest priority, last to be evicted)
resources:
  requests:
    cpu: "1"
    memory: "512Mi"
  limits:
    cpu: "1"           # Same as request
    memory: "512Mi"    # Same as request

# 2. Burstable — requests < limits (medium priority)
resources:
  requests:
    cpu: "500m"
    memory: "256Mi"
  limits:
    cpu: "2"           # Higher than request — can burst
    memory: "1Gi"      # Higher than request — can burst

# 3. BestEffort — NO requests, NO limits (lowest priority, first to be evicted)
# (No resources block at all)
```

```bash
# Check pod QoS class
kubectl get pod order-app-abc12 -n backend -o jsonpath='{.status.qosClass}'
# "Burstable"
```

**Eviction order when node is under pressure:**
```
BestEffort → evicted first (no guarantees)
Burstable  → evicted second (pods exceeding requests evicted first)
Guaranteed → evicted last (only if node is critically low)
```

#### Why Java Apps Get OOMKilled at 512Mi

```
Java Memory Breakdown:
┌─────────────────────────┐
│ JVM Heap (Xmx)          │  256 MB (-Xmx256m)
│ Metaspace               │  ~64 MB (class metadata)
│ Thread Stacks           │  ~30 MB (30 threads × 1MB each)
│ JIT Compiled Code       │  ~48 MB
│ Native Memory (NIO)     │  ~50 MB (direct buffers)
│ GC Overhead             │  ~40 MB
│ OS/Container overhead   │  ~24 MB
├─────────────────────────┤
│ TOTAL                   │  ~512 MB  ← RIGHT at the limit!
└─────────────────────────┘

Any spike → OOMKilled (exit code 137)
```

**Fix for Java applications:**

```yaml
spec:
  containers:
    - name: order-app
      image: mycompany/order-service:v3
      resources:
        requests:
          cpu: "500m"
          memory: "768Mi"      # Headroom above JVM total
        limits:
          memory: "1Gi"        # 2x the heap to be safe
          cpu: "2"
      env:
        # Set JVM heap to ~50-60% of the memory limit
        - name: JAVA_OPTS
          value: "-Xms256m -Xmx512m -XX:MaxMetaspaceSize=128m -XX:+UseContainerSupport"
        # UseContainerSupport makes JVM aware of container memory limits (JDK 10+)
```

```bash
# JDK 8u191+ / JDK 11+ respects container limits automatically:
# -XX:MaxRAMPercentage=60.0   ← Use 60% of container memory for heap
```

```yaml
env:
  - name: JAVA_OPTS
    value: "-XX:MaxRAMPercentage=60.0 -XX:+UseContainerSupport"
```

#### Sizing Resources Properly

```bash
# Step 1: Check actual usage with metrics-server
kubectl top pods -n backend --sort-by=memory
# NAME                    CPU(cores)   MEMORY(bytes)
# order-app-abc12         145m         423Mi
# order-app-def34         132m         418Mi
# payment-app-ghi56       890m         1.2Gi    ← CPU hog!
# data-processor-jkl78    3800m        256Mi    ← 3.8 CPU cores!

kubectl top nodes
# NAME            CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# ip-10-0-1-42    3900m        97%    12Gi            78%     ← Almost full!

# Step 2: Use VPA (Vertical Pod Autoscaler) recommendations
kubectl get vpa -n backend
kubectl describe vpa order-service-vpa -n backend
# Recommendation:
#   Container: order-app
#     Target:  cpu=200m, memory=512Mi
#     Lower:   cpu=100m, memory=384Mi
#     Upper:   cpu=500m, memory=768Mi
```

#### LimitRange — Enforce Defaults Namespace-Wide

```yaml
# Prevent pods with no limits from consuming everything
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: backend
spec:
  limits:
    - type: Container
      default:           # Default limits (if not specified)
        cpu: "1"
        memory: "512Mi"
      defaultRequest:    # Default requests (if not specified)
        cpu: "100m"
        memory: "128Mi"
      max:               # Maximum allowed
        cpu: "4"
        memory: "4Gi"
      min:               # Minimum allowed
        cpu: "50m"
        memory: "64Mi"
```

#### ResourceQuota — Prevent One Team from Hogging the Cluster

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-backend-quota
  namespace: backend
spec:
  hard:
    requests.cpu: "20"        # Team can request up to 20 CPU cores total
    requests.memory: "40Gi"   # Team can request up to 40Gi total
    limits.cpu: "40"
    limits.memory: "80Gi"
    pods: "100"               # Max 100 pods in this namespace
```

```bash
# Check quota usage
kubectl describe resourcequota team-backend-quota -n backend
# Name:            team-backend-quota
# Resource         Used    Hard
# --------         ----    ----
# limits.cpu       12      40
# limits.memory    24Gi    80Gi
# pods             35      100
# requests.cpu     6       20
# requests.memory  18Gi    40Gi
```

#### Troubleshooting OOMKilled

```bash
# Check if it's the container or the pod that's OOMKilled
kubectl describe pod order-app-abc12 -n backend | grep -A5 "Last State"
#   Last State:     Terminated
#     Reason:       OOMKilled
#     Exit Code:    137

# Check node-level OOM events
kubectl get events -n backend --field-selector reason=OOMKilling

# Check dmesg on the node (SSH or SSM)
dmesg | grep -i "oom\|killed" | tail -20

# Real-time memory monitoring
kubectl top pod order-app-abc12 -n backend --containers
# POD              NAME        CPU(cores)  MEMORY(bytes)
# order-app-abc12  order-app   145m        489Mi        ← Close to 512Mi limit!
```

---

## Q6. RBAC — Securing Multi-Tenant Clusters

### Question

> Your Kubernetes cluster is shared by 4 teams: Frontend, Backend, Data, and Platform. The frontend team should only access the `frontend` namespace, the data team needs read-only access to all namespaces but write access to `data-pipeline`, and the platform team needs cluster-admin. Design a complete RBAC model with Roles, ClusterRoles, RoleBindings, and ServiceAccounts.

### Real-Life Scenario

A developer from the frontend team accidentally deleted a deployment in the `backend` namespace using `kubectl delete deployment order-service`. The backend API went down for 5 minutes. Investigation reveals that all developers have cluster-admin access because "it was easier to set up."

### Answer

#### RBAC Building Blocks

```
Role              → Permissions within ONE namespace
ClusterRole       → Permissions cluster-wide (or reusable across namespaces)
RoleBinding       → Binds a Role/ClusterRole to a user/group WITHIN a namespace
ClusterRoleBinding → Binds a ClusterRole to a user/group CLUSTER-WIDE
ServiceAccount    → Identity for pods (not humans)
```

#### Step 1: Create Namespaces

```bash
kubectl create namespace frontend
kubectl create namespace backend
kubectl create namespace data-pipeline
kubectl create namespace monitoring
```

#### Step 2: Define Roles

```yaml
# Role: Full access within a namespace (for team members)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: namespace-admin
  namespace: frontend
rules:
  - apiGroups: ["", "apps", "batch", "networking.k8s.io"]
    resources: ["deployments", "services", "pods", "configmaps",
                "secrets", "ingresses", "jobs", "cronjobs",
                "persistentvolumeclaims", "horizontalpodautoscalers"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["pods/log", "pods/exec", "pods/portforward"]
    verbs: ["get", "list", "create"]
  # CANNOT delete namespaces, nodes, or cluster-level resources

---
# ClusterRole: Read-only access to all namespaces
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-reader
rules:
  - apiGroups: ["", "apps", "batch"]
    resources: ["pods", "deployments", "services", "configmaps",
                "jobs", "cronjobs", "replicasets", "statefulsets",
                "daemonsets", "events", "nodes"]
    verbs: ["get", "list", "watch"]
  # NO create, update, delete, exec

---
# ClusterRole: Deployer — can update deployments and rollback
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: deployer
rules:
  - apiGroups: ["apps"]
    resources: ["deployments", "deployments/rollback"]
    verbs: ["get", "list", "watch", "update", "patch"]
  - apiGroups: ["apps"]
    resources: ["replicasets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

#### Step 3: Bind Roles to Teams

```yaml
# Frontend team → admin in frontend namespace ONLY
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: frontend-team-admin
  namespace: frontend            # Only applies in frontend namespace
subjects:
  - kind: Group
    name: "frontend-team"        # AWS IAM group or OIDC group
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: namespace-admin
  apiGroup: rbac.authorization.k8s.io

---
# Backend team → admin in backend namespace ONLY
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: backend-team-admin
  namespace: backend
subjects:
  - kind: Group
    name: "backend-team"
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: namespace-admin
  apiGroup: rbac.authorization.k8s.io

---
# Data team → read-only CLUSTER-WIDE
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: data-team-reader
subjects:
  - kind: Group
    name: "data-team"
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-reader
  apiGroup: rbac.authorization.k8s.io

---
# Data team → admin in data-pipeline namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: data-team-admin
  namespace: data-pipeline
subjects:
  - kind: Group
    name: "data-team"
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: namespace-admin
  apiGroup: rbac.authorization.k8s.io

---
# Platform team → cluster-admin (full access)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: platform-team-admin
subjects:
  - kind: Group
    name: "platform-team"
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin           # Built-in cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

#### Step 4: ServiceAccount for CI/CD

```yaml
# ServiceAccount for GitHub Actions deployer
apiVersion: v1
kind: ServiceAccount
metadata:
  name: github-deployer
  namespace: backend

---
# Only allow deploying in backend namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: github-deployer-binding
  namespace: backend
subjects:
  - kind: ServiceAccount
    name: github-deployer
    namespace: backend
roleRef:
  kind: ClusterRole
  name: deployer
  apiGroup: rbac.authorization.k8s.io
```

#### Testing RBAC — Verify Permissions

```bash
# Check what YOU can do
kubectl auth can-i create deployments -n frontend
# yes

kubectl auth can-i delete deployments -n backend
# no

# Check what a specific user/group can do
kubectl auth can-i create deployments -n backend \
  --as=john@mycompany.com \
  --as-group=frontend-team
# no

# List all permissions for a user
kubectl auth can-i --list --as=john@mycompany.com --as-group=frontend-team -n frontend

# Test ServiceAccount permissions
kubectl auth can-i update deployments -n backend \
  --as=system:serviceaccount:backend:github-deployer
# yes
```

#### RBAC Access Matrix

| Team | frontend NS | backend NS | data-pipeline NS | monitoring NS | Cluster |
|------|-------------|------------|-------------------|---------------|---------|
| Frontend | **Admin** | No access | No access | No access | No access |
| Backend | No access | **Admin** | No access | No access | No access |
| Data | Read-only | Read-only | **Admin** | Read-only | Read-only |
| Platform | **Admin** | **Admin** | **Admin** | **Admin** | **cluster-admin** |
| CI/CD (SA) | No access | Deploy only | No access | No access | No access |

#### Troubleshooting

```bash
# Problem: "forbidden: User cannot list resource"
kubectl get pods -n backend
# Error: pods is forbidden: User "john" cannot list resource "pods" in namespace "backend"

# Debug: Check what bindings exist
kubectl get rolebindings -n backend
kubectl get clusterrolebindings | grep team

# Check the role details
kubectl describe role namespace-admin -n backend
kubectl describe clusterrole cluster-reader

# Problem: ServiceAccount can't access resources
kubectl describe sa github-deployer -n backend
kubectl get rolebindings -n backend -o yaml | grep -A10 github-deployer
```

---

## Q7. Persistent Volumes, PVCs, and StatefulSets

### Question

> You need to deploy a PostgreSQL database and a Kafka cluster on Kubernetes. Both need persistent storage that survives pod restarts. Explain PV, PVC, StorageClass, and StatefulSets. What happens when a pod with a PVC is deleted? What about node failure? Show a complete StatefulSet example with volume expansion.

### Real-Life Scenario

Your Kafka cluster runs as a StatefulSet with 3 brokers. Each broker has a 100GB EBS volume. Broker-2's pod gets killed and rescheduled to a different node. The question is: does it get the same data back? Meanwhile, all 3 brokers are running out of disk space and you need to expand from 100GB to 500GB without downtime.

### Answer

#### PV, PVC, StorageClass — How They Fit Together

```
StorageClass → "What KIND of storage" (gp3, io2, efs)
       │
       ▼
PersistentVolume (PV) → "Actual storage block" (aws://vol-abc123)
       │
       ▼
PersistentVolumeClaim (PVC) → "Pod's request for storage"
       │
       ▼
Pod → Mounts the PVC as a volume
```

#### StorageClass for AWS EBS

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-encrypted
provisioner: ebs.csi.aws.com       # AWS EBS CSI Driver
parameters:
  type: gp3
  encrypted: "true"
  fsType: ext4
  iops: "3000"
  throughput: "125"
reclaimPolicy: Retain              # Keep the EBS volume when PVC is deleted
allowVolumeExpansion: true         # Allow online volume expansion
volumeBindingMode: WaitForFirstConsumer  # Don't create volume until pod is scheduled
```

**Reclaim policies:**

```
Delete  → EBS volume is DELETED when PVC is deleted (default, dangerous for data!)
Retain  → EBS volume is KEPT when PVC is deleted (manual cleanup, safe)
```

#### StatefulSet for Kafka

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kafka
  namespace: data-pipeline
spec:
  serviceName: kafka-headless     # Required — creates DNS records
  replicas: 3
  podManagementPolicy: Parallel   # Start/stop pods simultaneously
  selector:
    matchLabels:
      app: kafka
  template:
    metadata:
      labels:
        app: kafka
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - name: kafka
          image: confluentinc/cp-kafka:7.5.0
          ports:
            - containerPort: 9092
              name: kafka
          env:
            - name: KAFKA_BROKER_ID
              valueFrom:
                fieldRef:
                  fieldPath: metadata.labels['apps.kubernetes.io/pod-index']
            - name: KAFKA_ZOOKEEPER_CONNECT
              value: "zookeeper-0.zookeeper:2181,zookeeper-1.zookeeper:2181,zookeeper-2.zookeeper:2181"
          resources:
            requests:
              cpu: "1"
              memory: "2Gi"
            limits:
              cpu: "2"
              memory: "4Gi"
          volumeMounts:
            - name: kafka-data
              mountPath: /var/lib/kafka/data
  volumeClaimTemplates:
    - metadata:
        name: kafka-data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: gp3-encrypted
        resources:
          requests:
            storage: 100Gi

---
# Headless Service — Required for StatefulSet DNS
apiVersion: v1
kind: Service
metadata:
  name: kafka-headless
  namespace: data-pipeline
spec:
  clusterIP: None              # Headless — no load balancing
  selector:
    app: kafka
  ports:
    - port: 9092
      name: kafka
```

**StatefulSet creates predictable pod names and DNS:**

```
Pod names:    kafka-0, kafka-1, kafka-2
DNS:          kafka-0.kafka-headless.data-pipeline.svc.cluster.local
              kafka-1.kafka-headless.data-pipeline.svc.cluster.local
              kafka-2.kafka-headless.data-pipeline.svc.cluster.local
PVCs:         kafka-data-kafka-0, kafka-data-kafka-1, kafka-data-kafka-2
```

#### What Happens When a Pod Dies?

```
kafka-2 pod dies (OOMKilled or node failure)
    │
    ▼
StatefulSet controller creates new kafka-2 pod
    │
    ▼
New kafka-2 gets the SAME PVC (kafka-data-kafka-2)
    │
    ▼
PVC is bound to the SAME PV (EBS volume)
    │
    ▼
Data is intact! Kafka-2 resumes with all its partitions.
```

```bash
# If pod is on a different node, EBS volume detaches from old node
# and attaches to the new node (takes ~30 seconds for EBS)

# Verify
kubectl get pvc -n data-pipeline
# NAME                STATUS   VOLUME          CAPACITY   STORAGECLASS
# kafka-data-kafka-0  Bound    pvc-aaa111...   100Gi      gp3-encrypted
# kafka-data-kafka-1  Bound    pvc-bbb222...   100Gi      gp3-encrypted
# kafka-data-kafka-2  Bound    pvc-ccc333...   100Gi      gp3-encrypted
```

#### Volume Expansion (100GB → 500GB Without Downtime)

```bash
# Step 1: Verify StorageClass allows expansion
kubectl get storageclass gp3-encrypted -o jsonpath='{.allowVolumeExpansion}'
# true

# Step 2: Edit each PVC
kubectl patch pvc kafka-data-kafka-0 -n data-pipeline \
  -p '{"spec":{"resources":{"requests":{"storage":"500Gi"}}}}'

kubectl patch pvc kafka-data-kafka-1 -n data-pipeline \
  -p '{"spec":{"resources":{"requests":{"storage":"500Gi"}}}}'

kubectl patch pvc kafka-data-kafka-2 -n data-pipeline \
  -p '{"spec":{"resources":{"requests":{"storage":"500Gi"}}}}'

# Step 3: Check expansion status
kubectl get pvc -n data-pipeline
# STATUS might show "FileSystemResizePending"

# The pod needs to be restarted for filesystem resize on some CSI drivers:
kubectl delete pod kafka-0 -n data-pipeline
# StatefulSet recreates it, and the filesystem is expanded on mount

# Step 4: Verify inside the pod
kubectl exec kafka-0 -n data-pipeline -- df -h /var/lib/kafka/data
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/nvme1n1    492G   85G  407G  17% /var/lib/kafka/data
```

#### Troubleshooting

```bash
# PVC stuck in Pending
kubectl describe pvc kafka-data-kafka-0 -n data-pipeline
# Events:
#   Warning  ProvisioningFailed  no matching storage class  ← Wrong StorageClass name
#   Warning  ProvisioningFailed  volume node affinity conflict  ← AZ mismatch

# Pod stuck in ContainerCreating (waiting for volume)
kubectl describe pod kafka-0 -n data-pipeline
# Events:
#   Warning  FailedAttachVolume  Multi-Attach error for volume "pvc-xxx"
# ← EBS volume still attached to old node (takes up to 6 minutes to force-detach)

# Force detach an EBS volume
aws ec2 detach-volume --volume-id vol-abc123 --force

# PVC won't delete (finalizer protection)
kubectl get pvc kafka-data-kafka-0 -n data-pipeline -o jsonpath='{.metadata.finalizers}'
# Remove finalizer if stuck:
kubectl patch pvc kafka-data-kafka-0 -n data-pipeline \
  -p '{"metadata":{"finalizers":null}}'
```

---

## Q8. ConfigMaps, Secrets, and Configuration Management

### Question

> Your microservices need different configurations per environment (dev, staging, prod) — database URLs, API keys, feature flags, and TLS certificates. How do you manage this in Kubernetes? Compare ConfigMaps, Secrets, External Secrets Operator, and Sealed Secrets. What happens when you update a ConfigMap — do pods automatically pick up the change?

### Real-Life Scenario

A developer updates the database connection string in a ConfigMap. They expect the application to use the new database immediately. But the pods keep connecting to the old database. After investigation, they realize pods need to be restarted to pick up ConfigMap changes (when used as environment variables).

### Answer

#### ConfigMap — Non-Sensitive Configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-config
  namespace: backend
data:
  # Simple key-value pairs
  DATABASE_HOST: "prod-db.cluster-xyz.ap-south-1.rds.amazonaws.com"
  DATABASE_PORT: "5432"
  DATABASE_NAME: "orders"
  LOG_LEVEL: "info"
  CACHE_TTL: "300"
  FEATURE_FLAG_NEW_CHECKOUT: "true"

  # File-based config
  application.yaml: |
    server:
      port: 8080
    spring:
      datasource:
        url: jdbc:postgresql://prod-db:5432/orders
        hikari:
          maximum-pool-size: 20
    management:
      endpoints:
        web:
          exposure:
            include: health,metrics,prometheus
```

**Using ConfigMap as Environment Variables:**

```yaml
spec:
  containers:
    - name: order-app
      envFrom:
        - configMapRef:
            name: order-service-config    # ALL keys become env vars

      # Or pick specific keys:
      env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: order-service-config
              key: DATABASE_HOST
```

**Using ConfigMap as a Mounted File:**

```yaml
spec:
  containers:
    - name: order-app
      volumeMounts:
        - name: config-volume
          mountPath: /app/config
          readOnly: true
  volumes:
    - name: config-volume
      configMap:
        name: order-service-config
        items:
          - key: application.yaml
            path: application.yaml
# File appears at: /app/config/application.yaml
```

#### Do Pods Auto-Reload ConfigMap Changes?

```
Method                          Auto-Update?    Delay
─────────────────────────────────────────────────────────
env vars (envFrom/valueFrom)    NO              Must restart pod
Volume mount                    YES             ~60-120 seconds (kubelet sync)
Projected volume                YES             ~60-120 seconds
```

```bash
# Force pods to restart after ConfigMap change:
kubectl rollout restart deployment/order-service -n backend

# Or use a hash annotation trick (auto-restart on config change):
```

```yaml
spec:
  template:
    metadata:
      annotations:
        # Change this hash when ConfigMap changes → triggers rolling update
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    spec:
      containers:
        - name: order-app
```

#### Kubernetes Secrets

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: order-service-secrets
  namespace: backend
type: Opaque
data:
  # Values MUST be base64 encoded
  DATABASE_PASSWORD: UHIwZHVjdGlvbiFQQHNz    # base64 of "Production!P@ss"
  API_KEY: c2stbGl2ZS1hYmMxMjM=              # base64 of "sk-live-abc123"

# Or use stringData (plaintext, Kubernetes encodes it for you):
stringData:
  DATABASE_PASSWORD: "Production!P@ss"
  API_KEY: "sk-live-abc123"
```

```bash
# Create secret from CLI
kubectl create secret generic order-service-secrets -n backend \
  --from-literal=DATABASE_PASSWORD='Production!P@ss' \
  --from-literal=API_KEY='sk-live-abc123'

# Create from file (TLS cert)
kubectl create secret tls mycompany-tls -n backend \
  --cert=fullchain.pem \
  --key=privkey.pem
```

> **WARNING:** Kubernetes Secrets are **base64-encoded, NOT encrypted**. Anyone with access to the namespace can decode them:

```bash
kubectl get secret order-service-secrets -n backend -o jsonpath='{.data.DATABASE_PASSWORD}' | base64 -d
# Production!P@ss  ← plaintext!
```

#### External Secrets Operator (Production-Grade)

```yaml
# Install: helm install external-secrets external-secrets/external-secrets

# Connect to AWS Secrets Manager
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-manager
  namespace: backend
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-south-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa    # IRSA — IAM Role for Service Account

---
# Pull secrets from AWS Secrets Manager into Kubernetes
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: order-service-secrets
  namespace: backend
spec:
  refreshInterval: 5m              # Check for changes every 5 minutes
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: order-service-secrets    # Kubernetes Secret name to create
    creationPolicy: Owner
  data:
    - secretKey: DATABASE_PASSWORD
      remoteRef:
        key: prod/order-service    # AWS Secrets Manager secret name
        property: db_password      # JSON key within the secret
    - secretKey: API_KEY
      remoteRef:
        key: prod/order-service
        property: api_key
```

```bash
# Verify the ExternalSecret is syncing
kubectl get externalsecret -n backend
# NAME                    STORE                  REFRESH   STATUS
# order-service-secrets   aws-secrets-manager    5m        SecretSynced

# The Kubernetes Secret is auto-created and auto-updated
kubectl get secret order-service-secrets -n backend
```

#### Sealed Secrets (GitOps-Friendly)

```bash
# Install kubeseal CLI
brew install kubeseal

# Encrypt a secret (can be committed to Git safely)
kubectl create secret generic order-service-secrets \
  --from-literal=DATABASE_PASSWORD='Production!P@ss' \
  --dry-run=client -o yaml | \
  kubeseal --controller-name=sealed-secrets \
  --controller-namespace=kube-system \
  --format yaml > sealed-secret.yaml
```

```yaml
# sealed-secret.yaml — SAFE to commit to Git
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: order-service-secrets
  namespace: backend
spec:
  encryptedData:
    DATABASE_PASSWORD: AgBghJ7k9f...long-encrypted-string...
    API_KEY: AgCdfK2m5g...long-encrypted-string...
```

#### Comparison

| Feature | ConfigMap | K8s Secret | External Secrets | Sealed Secrets |
|---------|-----------|------------|------------------|----------------|
| Encryption at rest | No | Optional (etcd encryption) | Yes (source vault) | Yes (asymmetric) |
| Safe for Git | Yes | **NO** | Yes (references only) | Yes (encrypted) |
| Auto-rotation | No | No | Yes (refreshInterval) | No |
| Audit trail | K8s audit logs | K8s audit logs | AWS CloudTrail | Git history |
| Complexity | Low | Low | Medium | Medium |
| Best for | App config | Dev/simple setups | **Production** | GitOps workflows |

---

## Q9. Horizontal Pod Autoscaler and Cluster Autoscaler

### Question

> Your e-commerce platform has traffic spikes during flash sales — 10x normal traffic in 5 minutes. How do you configure HPA to scale pods based on CPU, memory, and custom metrics? How does the Cluster Autoscaler add nodes when pods can't be scheduled? Walk me through the complete autoscaling chain from traffic spike to new pods running.

### Real-Life Scenario

Flash sale starts at 12:00 PM. By 12:02, the order service pods are at 95% CPU. HPA kicks in at 12:03, creating 20 new pods. But only 8 can be scheduled — the cluster doesn't have enough nodes. Cluster Autoscaler starts adding nodes at 12:04, but new EC2 instances take 3-4 minutes to join. By 12:07, the new nodes are ready. Total scaling time: 7 minutes. Customers were seeing errors from 12:02 to 12:07.

### Answer

#### Horizontal Pod Autoscaler (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
  namespace: backend
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 5
  maxReplicas: 50
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30     # Don't flap — wait 30s before scaling up again
      policies:
        - type: Percent
          value: 100                      # Can double pods in one step
          periodSeconds: 60
        - type: Pods
          value: 10                       # Or add up to 10 pods
          periodSeconds: 60
      selectPolicy: Max                   # Use whichever adds more pods
    scaleDown:
      stabilizationWindowSeconds: 300    # Wait 5 min before scaling down
      policies:
        - type: Percent
          value: 25                       # Remove at most 25% at a time
          periodSeconds: 60
  metrics:
    # Scale on CPU
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60          # Scale when avg CPU > 60%

    # Scale on Memory
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 70          # Scale when avg Memory > 70%

    # Scale on custom metric (requests per second via Prometheus)
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"             # Scale when avg RPS > 100 per pod
```

```bash
# Check HPA status
kubectl get hpa -n backend
# NAME               REFERENCE              TARGETS           MINPODS  MAXPODS  REPLICAS  AGE
# order-service-hpa  Deployment/order-svc   82%/60%, 45%/70%  5        50       15        2d

# Describe HPA for detailed status
kubectl describe hpa order-service-hpa -n backend
# Events:
#   Normal  SuccessfulRescale  New size: 15; reason: cpu resource utilization above target
```

#### Cluster Autoscaler

```yaml
# Cluster Autoscaler deployment (runs on the cluster)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  template:
    spec:
      containers:
        - name: cluster-autoscaler
          image: registry.k8s.io/autoscaling/cluster-autoscaler:v1.28.0
          command:
            - ./cluster-autoscaler
            - --v=4
            - --cloud-provider=aws
            - --skip-nodes-with-local-storage=false
            - --expander=least-waste            # Choose ASG that wastes least resources
            - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/production
            - --balance-similar-node-groups      # Keep ASGs balanced across AZs
            - --scale-down-delay-after-add=5m    # Don't scale down within 5 min of adding
            - --scale-down-unneeded-time=5m      # Node must be idle 5 min before removal
            - --max-node-provision-time=15m       # Give up after 15 min
```

**How Cluster Autoscaler decides to add nodes:**

```
1. HPA creates new pods
2. Scheduler can't place them → pods are "Pending"
3. Cluster Autoscaler sees Pending pods
4. Checks which ASG can accommodate them
5. Increases ASG desired count
6. EC2 launches new instance
7. kubelet starts on new node, registers with API server
8. Scheduler places pending pods on new node
9. Pods start running

Timeline: ~3-5 minutes for EC2 + ~30 seconds for pod startup
```

#### Complete Autoscaling Chain During Flash Sale

```
12:00 PM — Flash sale starts
           Pods: 5, CPU: 30%, Nodes: 3

12:01 PM — Traffic 5x, CPU: 75%
           HPA starts evaluation (checks every 15s by default)

12:02 PM — HPA scales to 15 pods (CPU > 60% target)
           8 pods scheduled, 7 pods Pending (insufficient resources)

12:02 PM — Cluster Autoscaler sees 7 Pending pods
           Requests 3 new nodes from ASG

12:03 PM — EC2 instances launching...
           Pods: 8 running, 7 pending

12:05 PM — New nodes join cluster
           7 pending pods get scheduled
           Pods: 15 running, CPU: 55%

12:06 PM — Traffic still rising, CPU: 70%
           HPA scales to 25 pods

12:07 PM — All pods running on existing + new nodes
           Total: 25 pods, 6 nodes, CPU: 50%
```

#### Reducing Scale-Up Time — Pre-Warming

```yaml
# Strategy 1: Overprovisioning with Pause Pods
# Low-priority pods that get evicted when real pods need space
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: overprovisioning
value: -1                     # Lower than default (0)
globalDefault: false

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: overprovisioning
  namespace: kube-system
spec:
  replicas: 3                 # Keep 3 "buffer" pods running
  selector:
    matchLabels:
      app: overprovisioning
  template:
    metadata:
      labels:
        app: overprovisioning
    spec:
      priorityClassName: overprovisioning
      containers:
        - name: pause
          image: registry.k8s.io/pause:3.9
          resources:
            requests:
              cpu: "2"          # Reserve 2 CPU per buffer pod
              memory: "4Gi"     # Reserve 4Gi per buffer pod
```

**How it works:**
```
Normal state: 3 buffer pods occupy 6 CPU + 12Gi across nodes
Flash sale: HPA creates new order-service pods
Scheduler evicts low-priority buffer pods to make room
Real pods start IMMEDIATELY — no waiting for new nodes
Meanwhile, Cluster Autoscaler adds nodes for the evicted buffer pods
Buffer pods come back on new nodes → pre-warmed for next spike
```

#### Karpenter (Modern Alternative to Cluster Autoscaler)

```yaml
# Karpenter — faster and smarter than Cluster Autoscaler
# Provisions RIGHT-SIZED nodes in ~60 seconds
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand", "spot"]        # Mix spot and on-demand
        - key: node.kubernetes.io/instance-type
          operator: In
          values: ["m5.large", "m5.xlarge", "m5.2xlarge", "c5.large", "c5.xlarge"]
      nodeClassRef:
        name: default
  limits:
    cpu: 100                   # Max 100 CPU cores total
    memory: 400Gi              # Max 400Gi total
  disruption:
    consolidationPolicy: WhenUnderutilized    # Auto-consolidate nodes
    expireAfter: 720h          # Replace nodes after 30 days
```

**Karpenter vs Cluster Autoscaler:**

| Feature | Cluster Autoscaler | Karpenter |
|---------|-------------------|-----------|
| Scale-up time | 3-5 minutes | ~60 seconds |
| Node sizing | Fixed ASG instance types | Dynamic right-sizing |
| Spot support | Via ASG mixed instances | Native, with fallback |
| Bin-packing | Basic | Advanced |
| Consolidation | Scale-down only | Active consolidation |
| Works with | Any Kubernetes | EKS only (for now) |

---

## Q10. Network Policies — Microsegmentation in Kubernetes

### Question

> By default, all pods in a Kubernetes cluster can communicate with every other pod — this is a security nightmare. Your security team mandates microsegmentation: the payment service should only accept traffic from the order service, the database should only accept traffic from the backend namespace, and no pod should be able to access the internet directly except through an egress proxy. Design Network Policies for this setup.

### Real-Life Scenario

A compromised pod in the `frontend` namespace was used for cryptocurrency mining. The attacker exploited a vulnerable npm package, got code execution in the pod, and then:
1. Connected to the Redis instance in the `backend` namespace (no network restriction)
2. Scanned the internal network and found the RDS endpoint
3. Exfiltrated data to an external server (no egress restrictions)

If Network Policies had been in place, the blast radius would have been limited to the compromised pod only.

### Answer

#### Default Deny — Start by Blocking Everything

```yaml
# Block ALL ingress traffic in the backend namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: backend
spec:
  podSelector: {}              # Applies to ALL pods in namespace
  policyTypes:
    - Ingress
  # No ingress rules = deny all incoming traffic

---
# Block ALL egress traffic in the backend namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: backend
spec:
  podSelector: {}
  policyTypes:
    - Egress
  # No egress rules = deny all outgoing traffic
  # ⚠️ This also blocks DNS! Add DNS exception below.

---
# Allow DNS resolution (required for service discovery)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: backend
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

#### Policy 1: Payment Service — Only Accept Traffic from Order Service

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: payment-service-ingress
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: payment-service
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: order-service       # ONLY order-service pods
      ports:
        - protocol: TCP
          port: 8080
    # Payment service is NOT reachable from:
    # - frontend namespace
    # - other backend services
    # - internet
```

#### Policy 2: Database — Only Accept from Backend Namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-ingress
  namespace: data
spec:
  podSelector:
    matchLabels:
      app: postgresql
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              team: backend              # Only pods from backend namespace
      ports:
        - protocol: TCP
          port: 5432
    # Database NOT reachable from frontend, monitoring, or other namespaces
```

```bash
# Label the namespace for namespaceSelector to work
kubectl label namespace backend team=backend
```

#### Policy 3: Allow Internet Only Through Egress Proxy

```yaml
# Backend pods can ONLY talk to egress proxy (not directly to internet)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-egress
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: order-service
  policyTypes:
    - Egress
  egress:
    # Allow to other backend services
    - to:
        - podSelector: {}              # Any pod in same namespace
      ports:
        - protocol: TCP
          port: 8080

    # Allow to database namespace
    - to:
        - namespaceSelector:
            matchLabels:
              team: data
      ports:
        - protocol: TCP
          port: 5432

    # Allow to egress proxy ONLY (not directly to internet)
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: egress-proxy
          podSelector:
            matchLabels:
              app: squid-proxy
      ports:
        - protocol: TCP
          port: 3128

    # Allow DNS
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
```

#### Policy 4: Frontend — Restricted Access

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-egress
  namespace: frontend
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    # Frontend can only talk to backend API gateway
    - to:
        - namespaceSelector:
            matchLabels:
              team: backend
          podSelector:
            matchLabels:
              app: api-gateway
      ports:
        - protocol: TCP
          port: 8080

    # DNS
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53

    # ❌ NO direct database access
    # ❌ NO direct internet access
    # ❌ NO access to payment service
```

#### Policy 5: Allow Prometheus to Scrape All Pods

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus-scrape
  namespace: backend
spec:
  podSelector: {}             # All pods in backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
          podSelector:
            matchLabels:
              app: prometheus
      ports:
        - protocol: TCP
          port: 9090          # Metrics port
```

#### Complete Network Policy Map

```
┌─────────────────────────────────────────────────┐
│                   CLUSTER                        │
│                                                  │
│  ┌──────────┐     ┌───────────┐    ┌─────────┐  │
│  │ Frontend │────→│ API       │    │ Monitrng│  │
│  │ (React)  │     │ Gateway   │    │ (Prom)  │  │
│  └──────────┘     └─────┬─────┘    └────┬────┘  │
│       ❌ ─ ─ ─ ─ ─ ❌   │   ← only     │scrape │
│      no direct DB       ▼    via GW     ▼ all   │
│               ┌─────────────────┐                │
│               │ Backend NS      │                │
│               │ ┌─────┐ ┌─────┐│                │
│               │ │Order│→│Pay- ││                │
│               │ │ Svc │ │ment ││                │
│               │ └──┬──┘ └─────┘│                │
│               └────┼───────────┘                 │
│                    │ only backend                 │
│                    ▼                              │
│               ┌──────────┐                       │
│               │ Database │                       │
│               │ (PG)     │                       │
│               └──────────┘                       │
│                                                  │
│  All egress → Squid Proxy → Internet             │
└─────────────────────────────────────────────────┘
```

#### Important: CNI Plugin Must Support Network Policies

```bash
# Default EKS VPC CNI supports Network Policies (since EKS 1.25 with network policy agent)
# Calico also supports Network Policies

# Enable on EKS:
aws eks create-addon \
  --cluster-name production \
  --addon-name vpc-cni \
  --addon-version v1.15.0-eksbuild.1 \
  --configuration-values '{"enableNetworkPolicy": "true"}'

# Verify Network Policies are enforced
kubectl get daemonset aws-node -n kube-system \
  -o jsonpath='{.spec.template.spec.containers[0].env}' | jq '.[] | select(.name=="ENABLE_NETWORK_POLICY")'
```

#### Testing Network Policies

```bash
# Test: Can frontend reach payment-service? (should FAIL)
kubectl run test-curl --image=curlimages/curl --rm -it -n frontend \
  -- curl -s --max-time 3 http://payment-service.backend.svc:8080/health
# curl: (28) Connection timed out  ← Network Policy working!

# Test: Can order-service reach payment-service? (should SUCCEED)
kubectl run test-curl --image=curlimages/curl --rm -it -n backend \
  -l app=order-service \
  -- curl -s --max-time 3 http://payment-service.backend.svc:8080/health
# {"status":"UP"}  ← Allowed by policy

# Test: Can backend reach internet directly? (should FAIL)
kubectl run test-curl --image=curlimages/curl --rm -it -n backend \
  -- curl -s --max-time 3 https://ifconfig.me
# curl: (28) Connection timed out  ← Blocked!

# Visualize Network Policies
# Use tools like: kubectl np-viewer, Cilium Hubble, or Calico UI
```

#### Troubleshooting

```bash
# Problem: "Everything stopped working after applying deny-all"
# You forgot to allow DNS!
# Without DNS, pods can't resolve service names

# Problem: "Policy seems to have no effect"
# Check if your CNI supports Network Policies
kubectl get pods -n kube-system | grep -E "calico|cilium|aws-node"

# Problem: "Can't figure out which policy is blocking"
# List all policies in namespace
kubectl get networkpolicies -n backend -o wide

# Describe specific policy
kubectl describe networkpolicy payment-service-ingress -n backend

# Debug with Cilium Hubble (if using Cilium CNI)
hubble observe --namespace backend --verdict DROPPED
```

---

## Q11. Init Containers, Sidecar Containers, and Multi-Container Pod Patterns

### Question

> Your application pod needs to: (1) wait for the database to be ready before starting, (2) run a log shipper alongside the main app, and (3) inject Vault secrets into a shared volume before the app starts. Design a multi-container pod using init containers and sidecars. Explain the execution order and what happens when an init container fails.

### Real-Life Scenario

Your order-service pod crashes because it tries to connect to PostgreSQL before the database has finished its migration job. Meanwhile, the operations team wants Fluentd to ship logs from every pod, and the security team wants HashiCorp Vault Agent to inject secrets as files. You need all three patterns in one pod.

### Answer

#### Execution Order in a Multi-Container Pod

```
Pod starts
  │
  ├── Init Container 1: wait-for-db          (runs first, must succeed)
  │     └── Checks if PostgreSQL is accepting connections
  │
  ├── Init Container 2: vault-agent-init     (runs second, must succeed)
  │     └── Fetches secrets from Vault, writes to shared volume
  │
  ├── [All init containers complete]
  │
  ├── Main Container: order-app              (starts after all inits)
  │     └── Reads secrets from shared volume, connects to DB
  │
  └── Sidecar Container: fluentd             (starts with main container)
        └── Reads logs from shared volume, ships to CloudWatch
```

**Rules:**
- Init containers run **sequentially** (one at a time)
- If an init container **fails**, Kubernetes restarts the **entire pod**
- Main + sidecar containers start **simultaneously** after all inits pass
- Init containers don't count toward pod resource limits (they run and exit)

#### Complete Multi-Container Pod Manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      # ─── Init Containers ───────────────────────────────
      initContainers:
        # Init 1: Wait for database to be ready
        - name: wait-for-db
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              echo "Waiting for PostgreSQL..."
              until nc -z prod-db.backend.svc.cluster.local 5432; do
                echo "DB not ready, retrying in 2s..."
                sleep 2
              done
              echo "DB is ready!"
          resources:
            requests:
              cpu: "50m"
              memory: "32Mi"

        # Init 2: Fetch secrets from Vault
        - name: vault-agent-init
          image: hashicorp/vault:1.15
          command:
            - vault
            - agent
            - -config=/vault/config/agent.hcl
            - -exit-after-auth
          env:
            - name: VAULT_ADDR
              value: "https://vault.mycompany.com:8200"
          volumeMounts:
            - name: vault-secrets
              mountPath: /vault/secrets
            - name: vault-config
              mountPath: /vault/config
          resources:
            requests:
              cpu: "100m"
              memory: "64Mi"

        # Init 3: Run database migrations
        - name: db-migrate
          image: mycompany/order-service:v3.2.0
          command: ["./migrate", "--direction=up"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: order-db-credentials
                  key: url
          resources:
            requests:
              cpu: "200m"
              memory: "128Mi"

      # ─── Main + Sidecar Containers ─────────────────────
      containers:
        # Main application container
        - name: order-app
          image: mycompany/order-service:v3.2.0
          ports:
            - containerPort: 8080
          env:
            - name: DB_PASSWORD_FILE
              value: /vault/secrets/db-password
          volumeMounts:
            - name: vault-secrets
              mountPath: /vault/secrets
              readOnly: true
            - name: app-logs
              mountPath: /var/log/app
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 10
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "2"
              memory: "1Gi"

        # Sidecar: Log shipper
        - name: fluentd
          image: fluent/fluentd:v1.16
          volumeMounts:
            - name: app-logs
              mountPath: /var/log/app
              readOnly: true
            - name: fluentd-config
              mountPath: /fluentd/etc
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "200m"
              memory: "256Mi"

        # Sidecar: Vault agent (keeps secrets refreshed)
        - name: vault-agent-sidecar
          image: hashicorp/vault:1.15
          command:
            - vault
            - agent
            - -config=/vault/config/agent-sidecar.hcl
          volumeMounts:
            - name: vault-secrets
              mountPath: /vault/secrets
            - name: vault-config
              mountPath: /vault/config
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"

      # ─── Shared Volumes ────────────────────────────────
      volumes:
        - name: vault-secrets
          emptyDir:
            medium: Memory     # tmpfs — secrets never hit disk
        - name: app-logs
          emptyDir: {}
        - name: vault-config
          configMap:
            name: vault-agent-config
        - name: fluentd-config
          configMap:
            name: fluentd-config
```

#### Native Sidecar Containers (Kubernetes 1.28+)

```yaml
# New restartPolicy: Always for init containers makes them true sidecars
initContainers:
  - name: log-shipper
    image: fluent/fluentd:v1.16
    restartPolicy: Always        # ← Runs as a sidecar, doesn't block main container
    volumeMounts:
      - name: app-logs
        mountPath: /var/log/app
```

**Benefits of native sidecars:**
- Start before main containers (guaranteed log capture from the beginning)
- Shut down after main containers (capture shutdown logs)
- Proper lifecycle ordering

#### What Happens When Init Containers Fail?

```bash
# Scenario: wait-for-db fails because DB is actually down
kubectl get pods -n backend
# NAME                     READY   STATUS     RESTARTS   AGE
# order-svc-7f8d-abc12     0/3     Init:0/3   5          8m
#                                  ^^^^^^^^
#                                  Stuck on first init container

# Check init container logs
kubectl logs order-svc-7f8d-abc12 -n backend -c wait-for-db
# "DB not ready, retrying in 2s..."
# "DB not ready, retrying in 2s..."

# Pod restart policy applies:
# Always (default) → Kubernetes keeps restarting with backoff (10s, 20s, 40s, ..., 5min max)
# Never → Pod goes to Failed state
```

#### Troubleshooting

```bash
# Which init container is running/failing?
kubectl describe pod order-svc-7f8d-abc12 -n backend | grep -A20 "Init Containers"

# Logs of a specific init container
kubectl logs order-svc-7f8d-abc12 -c vault-agent-init -n backend

# Exec into an init container (while it's running — tricky timing)
kubectl exec -it order-svc-7f8d-abc12 -c wait-for-db -n backend -- sh

# Check shared volume contents
kubectl exec -it order-svc-7f8d-abc12 -c order-app -n backend -- ls -la /vault/secrets
```

---

## Q12. Kubernetes DNS, Service Discovery, and CoreDNS Troubleshooting

### Question

> Your pods can't resolve service names — `curl http://payment-service:8080` returns `Could not resolve host`. The application worked yesterday. Walk me through Kubernetes DNS architecture, how CoreDNS works, and how to troubleshoot DNS failures. Cover internal and external DNS resolution.

### Real-Life Scenario

After a cluster upgrade, intermittent DNS failures start occurring. 10% of DNS queries time out. Pods randomly can't reach other services. The issue is that CoreDNS pods are OOMKilled because their memory limit (170Mi) is too low for a cluster with 200+ services and 50+ namespaces.

### Answer

#### How Kubernetes DNS Works

```
Pod sends DNS query for "payment-service"
    │
    ▼
Pod's /etc/resolv.conf points to CoreDNS ClusterIP (10.96.0.10)
    │
    ▼
CoreDNS resolves using search domains:
  1. payment-service.backend.svc.cluster.local  ← tries this first
  2. payment-service.svc.cluster.local
  3. payment-service.cluster.local
  4. payment-service.ap-south-1.compute.internal  ← external fallback
    │
    ▼
CoreDNS returns the ClusterIP (e.g., 10.100.42.15)
    │
    ▼
kube-proxy / iptables / IPVS routes to actual pod IP
```

```bash
# Check pod's DNS config
kubectl exec order-app-abc12 -n backend -- cat /etc/resolv.conf
# nameserver 10.96.0.10          ← CoreDNS ClusterIP
# search backend.svc.cluster.local svc.cluster.local cluster.local
# options ndots:5                 ← Important! Explained below
```

**`ndots:5` explained:**
```
If your query has fewer than 5 dots, Kubernetes appends search domains first.

"payment-service" has 0 dots (< 5) → tries search domains first:
  1. payment-service.backend.svc.cluster.local ← found!

"api.external.com" has 2 dots (< 5) → tries search domains first:
  1. api.external.com.backend.svc.cluster.local ← not found
  2. api.external.com.svc.cluster.local ← not found
  3. api.external.com.cluster.local ← not found
  4. api.external.com ← finally tries the actual domain!

This means EXTERNAL DNS lookups generate 4 unnecessary queries!
```

#### DNS Record Types in Kubernetes

```bash
# ClusterIP Service → A record
payment-service.backend.svc.cluster.local → 10.100.42.15

# Headless Service (clusterIP: None) → A records for EACH pod
kafka-0.kafka-headless.data.svc.cluster.local → 10.0.1.42
kafka-1.kafka-headless.data.svc.cluster.local → 10.0.2.18
kafka-2.kafka-headless.data.svc.cluster.local → 10.0.3.55

# SRV records (port discovery)
_kafka._tcp.kafka-headless.data.svc.cluster.local → kafka-0:9092, kafka-1:9092, kafka-2:9092

# ExternalName Service → CNAME
database.backend.svc.cluster.local → prod-db.cluster-xyz.rds.amazonaws.com
```

#### Troubleshooting DNS Failures

```bash
# Step 1: Is CoreDNS running?
kubectl get pods -n kube-system -l k8s-app=kube-dns
# NAME                      READY   STATUS    RESTARTS   AGE
# coredns-7f89b7bc75-abc12  1/1     Running   0          5d
# coredns-7f89b7bc75-def34  1/1     Running   0          5d

# Step 2: Test DNS from inside a pod
kubectl run dns-test --image=busybox:1.36 --rm -it -n backend -- sh

# Inside the pod:
nslookup payment-service
# Server:    10.96.0.10
# Address:   10.96.0.10:53
# Name:      payment-service.backend.svc.cluster.local
# Address:   10.100.42.15    ← Working!

nslookup google.com
# Server:    10.96.0.10
# Address:   10.96.0.10:53
# Name:      google.com
# Address:   142.250.80.46   ← External DNS also working

# Step 3: If DNS fails, test CoreDNS directly
kubectl exec dns-test -- nslookup payment-service 10.96.0.10
# If this fails → CoreDNS issue
# If this works → Pod's resolv.conf issue

# Step 4: Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50
# Look for: "REFUSED", "SERVFAIL", "i/o timeout"

# Step 5: Check CoreDNS resources (OOM is common!)
kubectl top pods -n kube-system -l k8s-app=kube-dns
# NAME                      CPU    MEMORY
# coredns-abc12             45m    168Mi    ← Close to 170Mi limit!
# coredns-def34             42m    165Mi    ← OOM incoming!

# Fix: Increase CoreDNS memory
kubectl edit deployment coredns -n kube-system
# Change memory limit from 170Mi to 512Mi
```

#### Optimize DNS Performance

```yaml
# Option 1: Reduce ndots for pods that make many external calls
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      dnsConfig:
        options:
          - name: ndots
            value: "2"          # Reduces unnecessary search domain lookups
          - name: single-request-reopen
            value: ""           # Fixes race condition with parallel A/AAAA queries
      containers:
        - name: order-app
```

```yaml
# Option 2: Use FQDN in code to avoid search domain overhead
# Instead of:
#   http://payment-service:8080
# Use:
#   http://payment-service.backend.svc.cluster.local:8080
# This has 5+ dots, so it skips search domains entirely
```

```yaml
# Option 3: NodeLocal DNSCache (reduces CoreDNS load)
# Creates a DNS cache daemonset on every node
# Pods query local cache first → CoreDNS only for cache misses
# Reduces DNS latency from ~5ms to ~0.5ms
```

#### CoreDNS Corefile Configuration

```bash
kubectl get configmap coredns -n kube-system -o yaml
```

```
.:53 {
    errors
    health {
        lameduck 5s
    }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure
        fallthrough in-addr.arpa ip6.arpa
        ttl 30
    }
    prometheus :9153                    # Metrics endpoint
    forward . /etc/resolv.conf {       # External DNS forwarding
        max_concurrent 1000
    }
    cache 30                           # Cache TTL: 30 seconds
    loop
    reload
    loadbalance
}
```

---

## Q13. Jobs and CronJobs — Batch Processing in Kubernetes

### Question

> You need to run a database backup every night at 2 AM, a report generation job that processes 10 million records in parallel, and a one-time data migration that must complete successfully. How do you use Kubernetes Jobs and CronJobs? What happens if a job fails? How do you handle parallelism, retries, and completion guarantees?

### Real-Life Scenario

Your nightly CronJob runs a database backup at 2 AM. Last Tuesday, the backup job took 4 hours (usually 30 minutes) because the database was under heavy load. At 2 AM Wednesday, a new CronJob instance started while the previous one was still running. Two backup processes competed for resources and both failed. You lost 2 nights of backups.

### Answer

#### Basic Job — One-Time Execution

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-migration-v3
  namespace: backend
spec:
  backoffLimit: 3              # Retry up to 3 times on failure
  activeDeadlineSeconds: 3600  # Timeout after 1 hour
  ttlSecondsAfterFinished: 86400  # Auto-delete job 24h after completion
  template:
    spec:
      restartPolicy: Never     # Don't restart — let Job controller handle retries
      containers:
        - name: migrate
          image: mycompany/data-migration:v3
          command: ["./migrate", "--batch-size=1000"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: url
          resources:
            requests:
              cpu: "1"
              memory: "2Gi"
            limits:
              cpu: "2"
              memory: "4Gi"
```

```bash
# Run and monitor
kubectl apply -f data-migration-v3.yaml
kubectl get jobs -n backend
# NAME               COMPLETIONS   DURATION   AGE
# data-migration-v3  0/1           45s        45s

# Watch progress
kubectl logs -f job/data-migration-v3 -n backend

# Check job status
kubectl describe job data-migration-v3 -n backend
```

#### Parallel Job — Process 10M Records

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: report-generator
  namespace: backend
spec:
  completions: 10          # Need 10 successful completions total
  parallelism: 5           # Run 5 pods at a time
  backoffLimit: 6
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: report
          image: mycompany/report-generator:v2
          env:
            - name: JOB_COMPLETION_INDEX
              valueFrom:
                fieldRef:
                  fieldPath: metadata.annotations['batch.kubernetes.io/job-completion-index']
          command:
            - python
            - generate_report.py
            - --partition=$(JOB_COMPLETION_INDEX)
            - --total-partitions=10
```

```
Execution flow:
  T+0s:   Pods 0,1,2,3,4 start (parallelism=5)
  T+60s:  Pod 0 completes → Pod 5 starts
  T+65s:  Pod 1 completes → Pod 6 starts
  T+120s: Pod 2 fails → retried as new pod (backoffLimit allows)
  ...
  T+300s: All 10 completions done ✓
```

#### CronJob — Nightly Database Backup

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: db-backup
  namespace: backend
spec:
  schedule: "0 2 * * *"           # Every day at 2:00 AM
  timeZone: "Asia/Kolkata"        # IST (Kubernetes 1.27+)
  concurrencyPolicy: Forbid       # Don't start new if previous still running!
  successfulJobsHistoryLimit: 7   # Keep last 7 successful job records
  failedJobsHistoryLimit: 3       # Keep last 3 failed job records
  startingDeadlineSeconds: 600    # If missed by 10 min, skip this run
  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 7200   # Kill if running > 2 hours
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: mycompany/db-backup:v2
              command:
                - /bin/sh
                - -c
                - |
                  echo "Starting backup at $(date)"
                  pg_dump -h $DB_HOST -U $DB_USER -d orders | \
                    gzip | \
                    aws s3 cp - s3://mycompany-backups/db/orders-$(date +%Y%m%d-%H%M).sql.gz
                  echo "Backup completed at $(date)"
              env:
                - name: DB_HOST
                  value: "prod-db.cluster-xyz.rds.amazonaws.com"
                - name: DB_USER
                  valueFrom:
                    secretKeyRef:
                      name: db-credentials
                      key: username
                - name: PGPASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: db-credentials
                      key: password
              resources:
                requests:
                  cpu: "500m"
                  memory: "1Gi"
```

**Concurrency Policies:**

```
Allow    → Multiple job instances can run simultaneously (default, dangerous!)
Forbid   → Skip new run if previous is still running (safe for backups)
Replace  → Kill previous run and start new one (safe for report generation)
```

#### Monitoring and Troubleshooting

```bash
# List CronJob runs
kubectl get cronjobs -n backend
# NAME        SCHEDULE     SUSPEND   ACTIVE   LAST SCHEDULE   AGE
# db-backup   0 2 * * *    False     0        8h              30d

# List job history
kubectl get jobs -n backend --sort-by=.metadata.creationTimestamp | tail -10

# Check failed job
kubectl describe job db-backup-28456789 -n backend
kubectl logs job/db-backup-28456789 -n backend

# Manually trigger a CronJob
kubectl create job db-backup-manual --from=cronjob/db-backup -n backend

# Suspend a CronJob (stop future runs without deleting)
kubectl patch cronjob db-backup -n backend -p '{"spec":{"suspend":true}}'

# Resume
kubectl patch cronjob db-backup -n backend -p '{"spec":{"suspend":false}}'
```

---

## Q14. Kubernetes Probes — Liveness, Readiness, and Startup

### Question

> Your application takes 90 seconds to start (loading ML models into memory). The liveness probe kills it after 30 seconds because it thinks it's dead. Meanwhile, another service accepts traffic via the readiness probe too early, before its database connection pool is ready, causing 500 errors for the first 10 seconds. Design proper probe configurations. When should you use each type?

### Real-Life Scenario

Your ML inference service loads a 2GB model into GPU memory on startup (takes 90 seconds). The liveness probe has `initialDelaySeconds: 30`. Kubernetes kills the pod at 30 seconds, starts a new one, kills that at 30 seconds — infinite CrashLoopBackOff. Nobody realizes the probes are misconfigured because "the image works fine in Docker."

### Answer

#### The Three Probe Types

```
Startup Probe    → "Has the application FINISHED STARTING?"
                   Only runs during startup. Once it passes, it's never checked again.
                   If it fails, the container is killed and restarted.

Readiness Probe  → "Can this pod handle TRAFFIC right now?"
                   Runs throughout the pod's lifetime.
                   If it fails, pod is removed from Service endpoints (no traffic).
                   Pod is NOT killed.

Liveness Probe   → "Is this pod ALIVE and healthy?"
                   Runs throughout the pod's lifetime.
                   If it fails, the container is KILLED and restarted.
```

```
                    Pod Lifecycle
                    ─────────────────────────────────────────
Startup Probe:      ████████████████ ← checks until app is ready
                                    ✓ passes → never checked again
Readiness Probe:    ─ ─ ─ ─ ─ ─ ─ ─ ███████████████████████
                                      runs continuously
Liveness Probe:     ─ ─ ─ ─ ─ ─ ─ ─ ███████████████████████
                                      runs continuously

Time:  0s─────────────90s───────────────────────────────→
       ^               ^
       App starting     App ready for traffic
```

#### Complete Probe Configuration for ML Inference

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ml-inference
spec:
  template:
    spec:
      containers:
        - name: inference
          image: mycompany/ml-inference:v3
          ports:
            - containerPort: 8080

          # STARTUP PROBE — Allows up to 5 minutes for model loading
          startupProbe:
            httpGet:
              path: /health/startup    # Returns 200 only after model is loaded
              port: 8080
            initialDelaySeconds: 10    # Wait 10s before first check
            periodSeconds: 10          # Check every 10s
            failureThreshold: 30       # 30 failures × 10s = 300s (5 min) to start
            successThreshold: 1        # Pass once → startup complete
            timeoutSeconds: 5

          # READINESS PROBE — Is it ready for traffic?
          readinessProbe:
            httpGet:
              path: /health/ready      # Returns 200 when model AND GPU are ready
              port: 8080
            initialDelaySeconds: 5     # Start checking 5s after startup probe passes
            periodSeconds: 10
            failureThreshold: 3        # Remove from service after 3 failures
            successThreshold: 2        # Need 2 consecutive successes to be "ready"
            timeoutSeconds: 3

          # LIVENESS PROBE — Is it still alive?
          livenessProbe:
            httpGet:
              path: /health/live       # Returns 200 if process is responsive
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 15          # Check every 15s (less frequent than readiness)
            failureThreshold: 5        # Kill after 5 failures (75s of being dead)
            timeoutSeconds: 5
```

#### Application Health Endpoints

```python
# Python Flask example — different health endpoints for different probes

model_loaded = False
gpu_warmed_up = False
db_connected = False

@app.route('/health/startup')
def startup():
    """Startup probe — is the model loaded?"""
    if model_loaded:
        return jsonify({"status": "started", "model": "loaded"}), 200
    return jsonify({"status": "starting", "model": "loading"}), 503

@app.route('/health/ready')
def readiness():
    """Readiness probe — can we handle traffic?"""
    if model_loaded and gpu_warmed_up and db_connected:
        return jsonify({"status": "ready"}), 200
    return jsonify({
        "status": "not ready",
        "model": model_loaded,
        "gpu": gpu_warmed_up,
        "db": db_connected
    }), 503

@app.route('/health/live')
def liveness():
    """Liveness probe — is the process alive?"""
    try:
        # Simple check — can we respond at all?
        # Don't check external dependencies here!
        return jsonify({"status": "alive"}), 200
    except Exception:
        return jsonify({"status": "dead"}), 503
```

**Key Design Rule:**
```
Liveness  → Check ONLY internal process health (can I respond?)
             ❌ DON'T check database, Redis, external APIs
             (If DB is down, killing and restarting pods won't fix it)

Readiness → Check EVERYTHING needed to serve traffic
             ✅ Check database connection pool
             ✅ Check cache connection
             ✅ Check model loaded
```

#### Probe Types — HTTP, TCP, Command

```yaml
# HTTP probe (most common for web services)
livenessProbe:
  httpGet:
    path: /health
    port: 8080
    httpHeaders:
      - name: Authorization
        value: "Bearer probe-token"

# TCP probe (for databases, Kafka, non-HTTP services)
livenessProbe:
  tcpSocket:
    port: 5432            # Just checks if the port is open

# Exec/Command probe (for custom checks)
livenessProbe:
  exec:
    command:
      - /bin/sh
      - -c
      - "pg_isready -U postgres -h localhost"

# gRPC probe (Kubernetes 1.27+)
livenessProbe:
  grpc:
    port: 50051
    service: "health"     # gRPC health check service
```

#### Troubleshooting Probe Failures

```bash
# Pod keeps restarting — is it liveness probe?
kubectl describe pod ml-inference-abc12 -n backend | grep -A10 "Liveness\|Readiness\|Startup"
# Events:
#   Warning  Unhealthy  Liveness probe failed: HTTP probe failed with statuscode: 503
#   Normal   Killing    Container inference failed liveness probe, will be restarted

# Test the probe endpoint manually
kubectl exec ml-inference-abc12 -n backend -- curl -s localhost:8080/health/live
# {"status": "alive"}

# If timeout issues — check probe timeout vs app response time
kubectl exec ml-inference-abc12 -n backend -- \
  curl -s -o /dev/null -w "Time: %{time_total}s\n" localhost:8080/health/live
# Time: 4.2s  ← If timeoutSeconds is 3, this probe FAILS!
```

---

## Q15. Kubernetes Debugging — kubectl debug, ephemeral containers, and common issues

### Question

> A production pod is behaving strangely — it's running but returning wrong data intermittently. You can't SSH into the node, the container doesn't have shell utilities (distroless image), and you need to inspect the filesystem, network, and process state. How do you debug it?

### Real-Life Scenario

Your payment service returns incorrect currency conversions for 5% of requests. The container image is Google's distroless — there's no shell, no `curl`, no `tcpdump`. You can't reproduce the issue locally. You need to inspect the running container's environment, network connections, and file state without restarting it.

### Answer

#### Method 1: kubectl debug — Ephemeral Debug Container

```bash
# Attach a debug container to the running pod (doesn't restart it!)
kubectl debug payment-svc-abc12 -n backend \
  --image=nicolaka/netshoot \
  --target=payment-app \
  -it

# Inside the debug container:
# You share the same network namespace as the target container

# Check network connections
ss -tlnp
netstat -an | grep 8080

# DNS resolution
nslookup payment-service.backend.svc.cluster.local

# curl the application
curl localhost:8080/health
curl localhost:8080/api/v1/convert?from=USD&to=INR&amount=100

# Check processes (shared PID namespace if configured)
ps aux

# Capture network traffic
tcpdump -i eth0 -n port 8080 -c 50 -w /tmp/capture.pcap

# Check environment variables of the target container
cat /proc/1/environ | tr '\0' '\n'
```

#### Method 2: Copy Pod with Modified Command

```bash
# Create a copy of the pod but override the command
kubectl debug payment-svc-abc12 -n backend \
  --copy-to=debug-payment \
  --container=payment-app \
  --image=mycompany/payment-service:v3.2.0 \
  -- sleep 3600

# Now exec into the copy
kubectl exec -it debug-payment -n backend -- /bin/sh

# You have the same image, same volumes, same configs
# But the app isn't running — you can inspect everything
cat /app/config/application.yaml
env | sort
ls -la /vault/secrets/
```

#### Method 3: Debug on the Node

```bash
# Create a debug pod on the same node
kubectl debug node/ip-10-0-1-42 -it --image=ubuntu

# Inside the node debugger:
# Host filesystem is at /host
chroot /host

# Check container runtime
crictl ps | grep payment
crictl inspect <container-id>
crictl logs <container-id>

# Check node resources
top
df -h
free -m
dmesg | tail -50
```

#### Method 4: kubectl exec (when container has shell)

```bash
# Execute command in running container
kubectl exec -it payment-svc-abc12 -n backend -- /bin/sh

# Common debugging commands
env | sort                        # Check environment
cat /etc/resolv.conf              # DNS config
wget -qO- http://localhost:8080/health  # Test endpoint (no curl? use wget)
ls -la /app/                      # Check files
df -h                             # Disk space
cat /proc/1/status                # Process info

# For Java apps
jps                               # List Java processes
jstack <pid>                      # Thread dump
jmap -heap <pid>                  # Heap info
```

#### Method 5: Port-Forward for Local Testing

```bash
# Forward pod port to your laptop
kubectl port-forward payment-svc-abc12 8080:8080 -n backend

# Now test from your machine with full debugging tools
curl -v http://localhost:8080/api/v1/convert?from=USD&to=INR&amount=100
# Check response headers, body, status code
```

#### Common Issues Quick Reference

```bash
# Pod OOMKilled → check memory
kubectl describe pod <pod> | grep -A3 "Last State"
kubectl top pod <pod>

# Pod CrashLoopBackOff → check logs
kubectl logs <pod> --previous

# Pod Pending → check scheduling
kubectl describe pod <pod> | grep -A10 Events

# Pod ImagePullBackOff → check image name/registry creds
kubectl describe pod <pod> | grep -A5 "Failed"
kubectl get secret regcred -o yaml

# Service not reachable → check endpoints
kubectl get endpoints <service-name>
# If empty → pod labels don't match service selector

# Intermittent failures → check resource limits
kubectl top pods --sort-by=cpu
kubectl top pods --sort-by=memory
```

---

## Q16. Kubernetes Namespaces — Multi-Tenancy and Resource Isolation

### Question

> Your organization has 5 teams sharing one cluster. How do you use namespaces to isolate teams, enforce resource quotas, set default limits, and prevent one team from consuming all cluster resources? What are the limitations of namespace-based multi-tenancy?

### Real-Life Scenario

The data team runs a Spark job that consumes 80% of the cluster's CPU. The backend team's pods can't scale because there are no resources left. The frontend team accidentally deploys to the backend namespace and breaks the order service. You need proper isolation.

### Answer

#### Namespace Architecture

```yaml
# Create team namespaces
apiVersion: v1
kind: Namespace
metadata:
  name: backend
  labels:
    team: backend
    environment: production
    cost-center: CC-1001

---
apiVersion: v1
kind: Namespace
metadata:
  name: data-pipeline
  labels:
    team: data
    environment: production
    cost-center: CC-1002
```

#### ResourceQuota — Cap Resources Per Namespace

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: backend-quota
  namespace: backend
spec:
  hard:
    # Compute
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"

    # Object counts
    pods: "100"
    services: "30"
    secrets: "50"
    configmaps: "50"
    persistentvolumeclaims: "20"
    replicationcontrollers: "0"    # Force use of Deployments

    # LoadBalancers (expensive — limit per team)
    services.loadbalancers: "2"

    # NodePort range
    services.nodeports: "0"        # Disallow NodePort services
```

```bash
# Check quota usage
kubectl describe resourcequota backend-quota -n backend
```

#### LimitRange — Default Limits for Every Pod

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: backend
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "4"
        memory: "8Gi"
      min:
        cpu: "50m"
        memory: "64Mi"
    - type: Pod
      max:
        cpu: "8"
        memory: "16Gi"
    - type: PersistentVolumeClaim
      max:
        storage: "100Gi"
      min:
        storage: "1Gi"
```

#### Combine RBAC + NetworkPolicy + ResourceQuota

```
Namespace: backend
├── RBAC:           Backend team = admin, others = no access
├── NetworkPolicy:  Default deny, allow only specific ingress/egress
├── ResourceQuota:  Max 20 CPU, 40Gi memory, 100 pods
├── LimitRange:     Default 500m/256Mi, max 4CPU/8Gi per container
└── PodSecurity:    Restricted (no root, no host network)
```

#### Namespace Limitations — What Namespaces DON'T Isolate

```
❌ Node-level isolation     → Pods from different namespaces share nodes
                             Fix: Use taints/tolerations or node affinity

❌ Kernel-level isolation   → Pods share the same Linux kernel
                             Fix: Use gVisor, Kata Containers, or separate clusters

❌ Network isolation        → By default, all pods can reach all pods
                             Fix: Network Policies (mandatory!)

❌ Storage isolation        → PVs are cluster-scoped
                             Fix: StorageClass per team with quotas

❌ Cluster-scoped resources → ClusterRoles, Nodes, PVs are shared
                             Fix: RBAC + admission webhooks
```

---

## Q17. DaemonSets — Running Agents on Every Node

### Question

> You need to run a monitoring agent (Prometheus Node Exporter), a log collector (Fluentd), and a security scanner on every node in your cluster. How do you use DaemonSets? What happens when you add a new node? How do you exclude certain nodes? How do you update a DaemonSet without downtime?

### Real-Life Scenario

Your cluster has 50 nodes. You deploy Fluentd as a DaemonSet. A new node joins via Cluster Autoscaler. Five minutes later, you realize Fluentd isn't running on the new node because the DaemonSet has a node selector that doesn't match. Logs from pods on that node are lost.

### Answer

#### Basic DaemonSet — Runs on Every Node

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1       # Update one node at a time
      maxSurge: 0              # Don't create extra pods (1 per node max)
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostNetwork: true        # Use node's network namespace
      hostPID: true            # Access node's process list
      tolerations:
        # Run on ALL nodes, including tainted ones (master, GPU, etc.)
        - operator: Exists
      containers:
        - name: node-exporter
          image: prom/node-exporter:v1.7.0
          ports:
            - containerPort: 9100
              hostPort: 9100
          volumeMounts:
            - name: proc
              mountPath: /host/proc
              readOnly: true
            - name: sys
              mountPath: /host/sys
              readOnly: true
            - name: root
              mountPath: /host/root
              readOnly: true
              mountPropagation: HostToContainer
          resources:
            requests:
              cpu: "100m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "128Mi"
      volumes:
        - name: proc
          hostPath:
            path: /proc
        - name: sys
          hostPath:
            path: /sys
        - name: root
          hostPath:
            path: /
```

#### Exclude Specific Nodes Using nodeSelector or Affinity

```yaml
# Only run on worker nodes (not master/control plane)
spec:
  template:
    spec:
      nodeSelector:
        node-role.kubernetes.io/worker: ""

# Or using affinity — exclude GPU nodes from fluentd
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: node-type
                    operator: NotIn
                    values:
                      - gpu
```

#### DaemonSet Auto-Scheduling on New Nodes

```bash
# When a new node joins the cluster:
# 1. Kubelet registers the node
# 2. DaemonSet controller detects the new node
# 3. Automatically creates a pod on the new node
# 4. No manual intervention needed (if tolerations/selectors match)

# Check DaemonSet status
kubectl get daemonset -n monitoring
# NAME             DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR
# node-exporter    50        50        50      50           50          <none>
# fluentd          48        48        48      48           48          role=worker

# If DESIRED != CURRENT, some nodes don't match the selector
kubectl get daemonset fluentd -n monitoring -o yaml | grep -A5 nodeSelector
```

#### Rolling Update Strategy

```bash
# Update DaemonSet image
kubectl set image daemonset/fluentd fluentd=fluent/fluentd:v1.17 -n monitoring

# Watch the rollout (updates one node at a time)
kubectl rollout status daemonset/fluentd -n monitoring

# Rollback if needed
kubectl rollout undo daemonset/fluentd -n monitoring
```

---

## Q18. Kubernetes Security — PodSecurity, SecurityContext, and OPA Gatekeeper

### Question

> Your security team's audit found pods running as root, containers with privilege escalation, and pods mounting the host filesystem. Design a security policy that enforces: no root containers, no privilege escalation, read-only root filesystem, and no host networking. Compare PodSecurity Standards, OPA Gatekeeper, and Kyverno.

### Real-Life Scenario

A penetration test reveals that a compromised container running as root was able to escape the container and access the host node by mounting `/var/run/docker.sock`. From there, the attacker could launch containers on any node in the cluster.

### Answer

#### PodSecurity Standards (Built-in, Kubernetes 1.25+)

```yaml
# Apply security standard at the namespace level
apiVersion: v1
kind: Namespace
metadata:
  name: backend
  labels:
    # Enforce = reject pods that violate
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest

    # Warn = allow but show warning
    pod-security.kubernetes.io/warn: restricted

    # Audit = allow but log
    pod-security.kubernetes.io/audit: restricted
```

**Three built-in levels:**

```
privileged  → No restrictions (for system pods like kube-proxy)
baseline    → Prevents known privilege escalations
restricted  → Heavily restricted (recommended for application pods)
```

**What "restricted" blocks:**
- Running as root
- Privilege escalation
- Host networking, host PID, host IPC
- Host path volumes
- Privileged containers
- Non-default capabilities

#### SecurityContext — Per-Pod and Per-Container

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: backend
spec:
  template:
    spec:
      # Pod-level security
      securityContext:
        runAsNonRoot: true             # Pod MUST run as non-root
        runAsUser: 1000                # Run as UID 1000
        runAsGroup: 1000               # Primary group
        fsGroup: 1000                  # Volume files owned by this group
        seccompProfile:
          type: RuntimeDefault         # Use container runtime's default seccomp

      containers:
        - name: order-app
          image: mycompany/order-service:v3
          securityContext:
            # Container-level security
            allowPrivilegeEscalation: false  # Block setuid/setgid
            readOnlyRootFilesystem: true     # No writes to container filesystem
            capabilities:
              drop:
                - ALL                   # Drop ALL Linux capabilities
              # add:
              #   - NET_BIND_SERVICE   # Only add back what you need

          volumeMounts:
            - name: tmp
              mountPath: /tmp          # Writable temp dir
            - name: app-cache
              mountPath: /app/cache    # Writable cache

      volumes:
        - name: tmp
          emptyDir: {}
        - name: app-cache
          emptyDir: {}
```

#### OPA Gatekeeper — Custom Policy Enforcement

```yaml
# Install Gatekeeper
# helm install gatekeeper gatekeeper/gatekeeper -n gatekeeper-system

# Define a constraint template — "what to check"
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels
        violation[{"msg": msg}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("Missing required labels: %v", [missing])
        }

---
# Apply the constraint — "where to enforce"
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-team-label
spec:
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment"]
    namespaces: ["backend", "frontend", "data-pipeline"]
  parameters:
    labels:
      - "team"
      - "cost-center"
```

```yaml
# Block privileged containers
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8spsprivilegedcontainer
spec:
  crd:
    spec:
      names:
        kind: K8sPSPPrivilegedContainer
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8spsprivileged
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          container.securityContext.privileged == true
          msg := sprintf("Privileged container not allowed: %v", [container.name])
        }
        violation[{"msg": msg}] {
          container := input.review.object.spec.initContainers[_]
          container.securityContext.privileged == true
          msg := sprintf("Privileged init container not allowed: %v", [container.name])
        }
```

#### Comparison

| Feature | PodSecurity Standards | OPA Gatekeeper | Kyverno |
|---------|----------------------|----------------|---------|
| Complexity | Low | High (needs Rego) | Medium (YAML-based) |
| Customization | 3 fixed levels | Unlimited | Unlimited |
| Mutating support | No | Yes | Yes (can modify resources) |
| Audit mode | Yes | Yes | Yes |
| Language | Labels | Rego | YAML policies |
| Best for | Simple enforcement | Complex enterprise rules | Kubernetes-native teams |

---

## Q19. Kubernetes Upgrades — Cluster Upgrade Strategy

### Question

> You need to upgrade your production Kubernetes cluster from 1.28 to 1.30. The cluster runs 200+ pods across 15 nodes. How do you plan and execute this upgrade with zero downtime? What are the risks? How do you handle deprecated APIs?

### Real-Life Scenario

During a cluster upgrade from 1.27 to 1.28, the platform team didn't check for deprecated APIs. After upgrading the control plane, 5 CronJobs stopped being scheduled because they used `batch/v1beta1` which was removed in 1.25 (they'd been running since before 1.25 with an old manifest). The team also forgot to upgrade the AWS VPC CNI plugin, causing all new pods to fail to get IP addresses.

### Answer

#### Pre-Upgrade Checklist

```bash
# 1. Check current version
kubectl version --short
# Client: v1.28.4
# Server: v1.28.4

# 2. Check for deprecated APIs
kubectl get --raw /metrics | grep apiserver_requested_deprecated_apis
# OR use pluto (popular tool):
# brew install pluto
pluto detect-all-in-cluster
# NAME            KIND      VERSION         REPLACEMENT    REMOVED   DEPRECATED
# my-cronjob      CronJob   batch/v1beta1   batch/v1       v1.25     v1.21

# 3. Check if all manifests are compatible
pluto detect-files -d ./k8s-manifests/
# Scans all YAML files for deprecated APIs

# 4. Check addon compatibility
kubectl get pods -n kube-system
# Verify versions of:
# - CoreDNS
# - kube-proxy
# - AWS VPC CNI / Calico
# - EBS CSI Driver
# - Cluster Autoscaler / Karpenter
# - Metrics Server

# 5. Check node readiness
kubectl get nodes
kubectl describe nodes | grep -E "Kubelet Version|Taints|Conditions"

# 6. Review changelog for breaking changes
# https://kubernetes.io/releases/
# Check: Removed APIs, changed defaults, deprecated features
```

#### Upgrade Strategy — Control Plane First, Then Nodes

```
Step 1: Upgrade Control Plane (1.28 → 1.29)
    │   ← Can only skip one minor version at a time!
    ▼
Step 2: Upgrade Add-ons (CoreDNS, kube-proxy, CNI, CSI)
    │
    ▼
Step 3: Upgrade Worker Nodes (rolling — one at a time)
    │
    ▼
Step 4: Verify everything works
    │
    ▼
Step 5: Upgrade Control Plane (1.29 → 1.30)
    │
    ▼
Step 6: Repeat steps 2-4
```

#### Node Upgrade — Cordon, Drain, Upgrade, Uncordon

```bash
# For each node (one at a time):

# Step 1: Cordon — prevent new pods from being scheduled
kubectl cordon ip-10-0-1-42
# node/ip-10-0-1-42 cordoned

# Step 2: Drain — safely evict all pods
kubectl drain ip-10-0-1-42 \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=60 \
  --timeout=300s
# Pods are gracefully terminated and rescheduled to other nodes

# Step 3: Upgrade the node (EKS: replace with new ASG instance)
# For EKS managed node groups:
aws eks update-nodegroup-version \
  --cluster-name production \
  --nodegroup-name workers \
  --kubernetes-version 1.29

# Step 4: Uncordon — allow scheduling again
kubectl uncordon ip-10-0-1-42

# Step 5: Verify node is ready
kubectl get node ip-10-0-1-42
# NAME            STATUS   ROLES    AGE   VERSION
# ip-10-0-1-42    Ready    <none>   1m    v1.29.0
```

#### Post-Upgrade Validation

```bash
# Verify all pods are running
kubectl get pods --all-namespaces | grep -v Running | grep -v Completed

# Verify all services have endpoints
kubectl get endpoints --all-namespaces | awk '$2 == "<none>"'

# Run smoke tests
kubectl run smoke-test --image=busybox --rm -it -- sh -c "
  nslookup kubernetes.default &&
  wget -qO- --timeout=5 http://order-service.backend.svc:8080/health &&
  echo 'ALL CHECKS PASSED'
"

# Check for warning events
kubectl get events --all-namespaces --field-selector type=Warning --sort-by='.lastTimestamp' | tail -20
```

---

## Q20. Kubernetes Monitoring — Metrics Server, kubectl top, and Resource Observability

### Question

> Your cluster has 200+ pods across 20 nodes. How do you monitor resource usage, detect pods that are about to OOMKill, find nodes under pressure, and identify wasted resources (over-provisioned pods)? Walk me through Metrics Server, kubectl top, VPA recommendations, and how to build a resource optimization workflow.

### Real-Life Scenario

Your monthly AWS bill for EKS is $15,000. Investigation reveals that pods are requesting 4x more CPU than they actually use — total requests are 80 CPU cores but actual usage is only 20 cores. You're paying for 25 nodes but only need 8. The CFO wants a 50% cost reduction.

### Answer

#### Metrics Server — Foundation for Resource Monitoring

```bash
# Install Metrics Server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verify it's running
kubectl get pods -n kube-system -l k8s-app=metrics-server
kubectl get apiservice v1beta1.metrics.k8s.io

# Node resource usage
kubectl top nodes
# NAME             CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# ip-10-0-1-42     1200m        30%    8Gi             50%
# ip-10-0-1-43     3800m        95%    14Gi            87%    ← HOT NODE!
# ip-10-0-2-18     400m         10%    2Gi             12%    ← WASTED!

# Pod resource usage
kubectl top pods -n backend --sort-by=cpu
# NAME                    CPU(cores)   MEMORY(bytes)
# data-proc-abc12         3200m        2.1Gi      ← CPU HOG
# order-svc-def34         450m         512Mi
# payment-svc-ghi56       120m         256Mi

# Pod resource usage by container
kubectl top pods -n backend --containers
```

#### Identify Over-Provisioned Pods

```bash
# Compare requests vs actual usage
# Step 1: Get requests
kubectl get pods -n backend -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].resources.requests.cpu}{"\t"}{.spec.containers[0].resources.requests.memory}{"\n"}{end}'
# order-svc-def34    2         2Gi
# payment-svc-ghi56  1         1Gi

# Step 2: Get actual usage
kubectl top pods -n backend
# order-svc-def34     450m      512Mi
# payment-svc-ghi56   120m      256Mi

# Analysis:
# order-svc: Requests 2 CPU, uses 450m → 77.5% wasted!
# payment-svc: Requests 1 CPU, uses 120m → 88% wasted!
```

#### Vertical Pod Autoscaler (VPA) — Automatic Right-Sizing

```yaml
# Install VPA
# kubectl apply -f https://github.com/kubernetes/autoscaler/releases/.../vpa.yaml

apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: order-service-vpa
  namespace: backend
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  updatePolicy:
    updateMode: "Off"         # "Off" = just recommend, don't auto-update
                               # "Auto" = automatically resize pods
  resourcePolicy:
    containerPolicies:
      - containerName: order-app
        minAllowed:
          cpu: "100m"
          memory: "128Mi"
        maxAllowed:
          cpu: "4"
          memory: "4Gi"
```

```bash
# Check VPA recommendations
kubectl describe vpa order-service-vpa -n backend
# Recommendation:
#   Container Recommendations:
#     Container Name: order-app
#     Lower Bound:    cpu: 200m,  memory: 300Mi
#     Target:         cpu: 500m,  memory: 512Mi    ← Right-sized values!
#     Upper Bound:    cpu: 1200m, memory: 900Mi
#     Uncapped Target: cpu: 500m, memory: 512Mi
```

#### Cost Optimization Workflow

```bash
# Step 1: Deploy VPA in recommendation mode for ALL deployments
for deploy in $(kubectl get deployments -n backend -o name); do
  name=$(echo $deploy | cut -d/ -f2)
  cat <<EOF | kubectl apply -f -
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: ${name}-vpa
  namespace: backend
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ${name}
  updatePolicy:
    updateMode: "Off"
EOF
done

# Step 2: Wait 1 week for recommendations to stabilize

# Step 3: Apply recommendations
# Update deployment resource requests based on VPA Target values
kubectl patch deployment order-service -n backend -p \
  '{"spec":{"template":{"spec":{"containers":[{"name":"order-app","resources":{"requests":{"cpu":"500m","memory":"512Mi"},"limits":{"cpu":"1500m","memory":"1Gi"}}}]}}}}'
```

#### Node-Level Optimization

```bash
# Find under-utilized nodes
kubectl top nodes | awk '$3+0 < 20 {print "UNDERUTILIZED:", $0}'
# Nodes with < 20% CPU usage can be consolidated

# Check pod distribution across nodes
kubectl get pods --all-namespaces -o wide | awk '{print $8}' | sort | uniq -c | sort -rn
# 45 ip-10-0-1-42    ← Overloaded
# 42 ip-10-0-1-43
#  5 ip-10-0-2-18    ← Nearly empty → candidate for removal
```

#### Cost Savings Summary

```
BEFORE optimization:
  Nodes: 25 × m5.xlarge ($0.192/hr) = $3,456/month
  Total CPU requested: 80 cores
  Total CPU used: 20 cores
  Utilization: 25%

AFTER optimization (VPA + right-sizing + node consolidation):
  Nodes: 10 × m5.xlarge = $1,382/month
  Total CPU requested: 30 cores (right-sized)
  Total CPU used: 20 cores
  Utilization: 67%

  Monthly savings: $2,074 (60% reduction)
  Annual savings: $24,888
```

---

## Q21. Helm — Package Manager for Kubernetes

### Question

> Your team deploys 15 microservices, each with Deployments, Services, ConfigMaps, Secrets, Ingress, HPA, and ServiceAccounts. Managing 100+ YAML files is becoming unmanageable. How does Helm solve this? Walk me through chart structure, values files, templating, hooks, and release management. How do you rollback a bad Helm release?

### Real-Life Scenario

A developer copies an existing service's YAML folder to create a new service. They forget to change the service name in one of the 7 files. The new service overwrites the existing service's endpoints. Helm charts with templating would have prevented this.

### Answer

#### Helm Chart Structure

```
order-service/
├── Chart.yaml               # Chart metadata (name, version, appVersion)
├── values.yaml               # Default configuration values
├── values-dev.yaml            # Dev-specific overrides
├── values-prod.yaml           # Prod-specific overrides
├── templates/
│   ├── _helpers.tpl           # Reusable template snippets
│   ├── deployment.yaml        # Deployment manifest template
│   ├── service.yaml           # Service manifest template
│   ├── ingress.yaml           # Ingress manifest template
│   ├── configmap.yaml         # ConfigMap template
│   ├── secret.yaml            # Secret template
│   ├── hpa.yaml               # HPA template
│   ├── serviceaccount.yaml    # ServiceAccount template
│   ├── networkpolicy.yaml     # NetworkPolicy template
│   └── tests/
│       └── test-connection.yaml
└── charts/                    # Sub-chart dependencies
```

#### Chart.yaml

```yaml
apiVersion: v2
name: order-service
description: Order processing microservice
version: 1.5.0            # Chart version (changes when chart changes)
appVersion: "3.2.1"        # Application version (Docker image tag)
type: application
dependencies:
  - name: redis
    version: "18.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
```

#### values.yaml — Default Configuration

```yaml
# values.yaml
replicaCount: 3

image:
  repository: mycompany/order-service
  tag: "3.2.1"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  className: nginx
  host: api.mycompany.com
  path: /v1/orders
  tls:
    enabled: true
    secretName: mycompany-tls

resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "2"
    memory: "1Gi"

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilization: 60

env:
  DATABASE_HOST: "prod-db.cluster-xyz.rds.amazonaws.com"
  LOG_LEVEL: "info"
  CACHE_TTL: "300"

redis:
  enabled: true
```

#### Templated Deployment (templates/deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "order-service.fullname" . }}
  labels:
    {{- include "order-service.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "order-service.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "order-service.selectorLabels" . | nindent 8 }}
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: {{ .Values.service.targetPort }}
          {{- if .Values.env }}
          env:
            {{- range $key, $value := .Values.env }}
            - name: {{ $key }}
              value: {{ $value | quote }}
            {{- end }}
          {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

#### Helm Commands — Daily Operations

```bash
# Install a release
helm install order-service ./order-service \
  -n backend \
  -f values-prod.yaml

# Upgrade (update image, change config, etc.)
helm upgrade order-service ./order-service \
  -n backend \
  -f values-prod.yaml \
  --set image.tag=3.2.2

# Rollback to previous version
helm rollback order-service 1 -n backend
# "1" = revision number

# Check release history
helm history order-service -n backend
# REVISION  STATUS      CHART              APP VERSION  DESCRIPTION
# 1         superseded  order-service-1.4.0  3.2.0       Install complete
# 2         superseded  order-service-1.5.0  3.2.1       Upgrade complete
# 3         deployed    order-service-1.5.0  3.2.2       Upgrade complete

# Dry-run (see what would be deployed without applying)
helm upgrade order-service ./order-service \
  -n backend \
  -f values-prod.yaml \
  --set image.tag=3.2.3 \
  --dry-run --debug

# Template locally (see generated YAML without connecting to cluster)
helm template order-service ./order-service -f values-prod.yaml

# Uninstall
helm uninstall order-service -n backend

# List all releases
helm list -n backend
helm list --all-namespaces
```

#### Using One Chart for All 15 Microservices

```bash
# Same chart, different values files
helm install order-service ./microservice-chart -f services/order-service.yaml -n backend
helm install payment-service ./microservice-chart -f services/payment-service.yaml -n backend
helm install user-service ./microservice-chart -f services/user-service.yaml -n backend
```

---

## Q22. GitOps with ArgoCD — Continuous Delivery for Kubernetes

### Question

> Your team does `kubectl apply` from their laptops to deploy to production. This has led to configuration drift, no audit trail, and "it works on my machine" incidents. How would you implement GitOps using ArgoCD? Walk me through the architecture, application manifests, sync policies, and how to handle secrets in a GitOps workflow.

### Real-Life Scenario

Three engineers apply different versions of the same service from their laptops within an hour. The production cluster has a mix of configurations that doesn't match any Git commit. When a bug is reported, nobody knows which version is actually running.

### Answer

#### GitOps Principle

```
Git repository = Single source of truth
Any change = Git commit → PR → Review → Merge → Auto-deploy

Nobody touches the cluster directly.
ArgoCD watches Git and syncs the cluster to match.
```

#### ArgoCD Architecture

```
Developer → Git Push → GitHub → ArgoCD detects change
                                     │
                                     ▼
                              ArgoCD compares
                              Git state vs Cluster state
                                     │
                                     ▼
                              If different → Sync (apply changes)
                              If same → No action
```

#### ArgoCD Application Manifest

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-service
  namespace: argocd
spec:
  project: backend

  source:
    repoURL: https://github.com/mycompany/k8s-manifests.git
    targetRevision: main
    path: environments/prod/order-service

    # If using Helm:
    # helm:
    #   valueFiles:
    #     - values-prod.yaml

  destination:
    server: https://kubernetes.default.svc
    namespace: backend

  syncPolicy:
    automated:
      prune: true              # Delete resources removed from Git
      selfHeal: true           # Revert manual changes to match Git
    syncOptions:
      - CreateNamespace=true
      - ApplyOutOfSyncOnly=true
    retry:
      limit: 3
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 1m
```

#### Sync Policies Explained

```
Manual Sync:     ArgoCD detects drift, waits for you to click "Sync"
Auto Sync:       ArgoCD automatically applies changes from Git
Self-Heal:       If someone does kubectl edit, ArgoCD reverts it back to Git state
Prune:           If you delete a manifest from Git, ArgoCD deletes the resource from cluster
```

#### ArgoCD App-of-Apps Pattern (managing 15 microservices)

```yaml
# root-app.yaml — manages all other applications
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/mycompany/k8s-manifests.git
    targetRevision: main
    path: argocd-apps          # Directory containing all Application manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
```

```
argocd-apps/
├── order-service.yaml         # ArgoCD Application for order-service
├── payment-service.yaml       # ArgoCD Application for payment-service
├── user-service.yaml
├── ... (15 apps total)
└── monitoring.yaml            # ArgoCD Application for Prometheus/Grafana
```

#### Handling Secrets in GitOps

```bash
# Problem: You can't commit plaintext secrets to Git
# Solution: Use Sealed Secrets or External Secrets Operator

# Sealed Secrets workflow:
# 1. Create a regular secret
kubectl create secret generic db-creds \
  --from-literal=password='MyP@ss' \
  --dry-run=client -o yaml | \
  kubeseal --format yaml > sealed-secret.yaml

# 2. Commit sealed-secret.yaml to Git (safe — it's encrypted)
git add sealed-secret.yaml
git commit -m "Add database credentials"
git push

# 3. ArgoCD syncs → SealedSecret controller decrypts → creates K8s Secret
```

#### ArgoCD CLI Operations

```bash
# Login
argocd login argocd.mycompany.com

# List applications
argocd app list

# Sync an application
argocd app sync order-service

# Check sync status
argocd app get order-service
# Name:               order-service
# Sync Status:        Synced
# Health Status:      Healthy

# Rollback to a previous Git commit
argocd app rollback order-service <revision-number>

# Diff — see what would change
argocd app diff order-service
```

---

## Q23. Service Mesh — Istio Basics for Microservice Communication

### Question

> Your 15 microservices communicate over HTTP/gRPC. You need mutual TLS between all services, traffic splitting for canary deployments, circuit breaking, and distributed tracing — without changing application code. How does Istio solve this? Walk me through sidecar injection, VirtualServices, DestinationRules, and how mTLS works.

### Real-Life Scenario

Your payment service calls an external payment gateway. Occasionally, the gateway becomes slow (10-second responses instead of 200ms). Without circuit breaking, all 50 order-service pods stack up waiting, exhausting connection pools, and causing a cascading failure across the entire platform.

### Answer

#### How Istio Works — Sidecar Proxy Pattern

```
Without Istio:
  Pod A ──HTTP──→ Pod B

With Istio:
  Pod A → Envoy Proxy (sidecar) ──mTLS──→ Envoy Proxy (sidecar) → Pod B
          ↑                                    ↑
          Intercepts all traffic               Intercepts all traffic
          Adds mTLS, retries,                  Enforces policies,
          metrics, tracing                     rate limits, auth
```

```bash
# Enable sidecar injection for a namespace
kubectl label namespace backend istio-injection=enabled

# All new pods in backend namespace automatically get an Envoy sidecar
kubectl get pods -n backend
# NAME                    READY   STATUS    RESTARTS   AGE
# order-svc-abc12         2/2     Running   0          5m    ← 2/2 = app + sidecar
```

#### VirtualService — Traffic Routing (Canary Deployment)

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service
  namespace: backend
spec:
  hosts:
    - order-service                    # Kubernetes service name
  http:
    - match:
        - headers:
            x-canary:
              exact: "true"            # Header-based routing
      route:
        - destination:
            host: order-service
            subset: canary
    - route:
        - destination:
            host: order-service
            subset: stable
          weight: 90                   # 90% to stable
        - destination:
            host: order-service
            subset: canary
          weight: 10                   # 10% to canary
```

#### DestinationRule — Circuit Breaking

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-gateway
  namespace: backend
spec:
  host: payment-gateway
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: DEFAULT
        http1MaxPendingRequests: 50    # Queue max 50 requests
        http2MaxRequests: 100
        maxRequestsPerConnection: 10
        maxRetries: 3
    outlierDetection:
      consecutive5xxErrors: 3          # After 3 consecutive 5xx
      interval: 10s                    # Check every 10s
      baseEjectionTime: 30s           # Eject host for 30s
      maxEjectionPercent: 50          # Eject max 50% of hosts

  subsets:
    - name: stable
      labels:
        version: v3.2.0
    - name: canary
      labels:
        version: v3.2.1
```

**Circuit breaker in action:**
```
Normal:        Order → Payment Gateway (200ms response) ✅
Gateway slow:  Order → Payment Gateway (10s response)
               → After 3 timeouts, circuit OPENS
               → Subsequent requests fail FAST (no waiting)
               → After 30s, circuit HALF-OPENS (tries one request)
               → If success, circuit CLOSES (traffic resumes)
               → If fail, circuit OPENS again
```

#### Mutual TLS (mTLS) — Zero-Trust Networking

```yaml
# Enforce mTLS for all services in the mesh
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: backend
spec:
  mtls:
    mode: STRICT    # All traffic MUST be mTLS (unencrypted traffic rejected)
```

```
Without mTLS:
  Pod A ──plaintext HTTP──→ Pod B    (anyone can sniff traffic)

With mTLS:
  Pod A ──encrypted TLS──→ Pod B     (certificates auto-rotated by Istio)
  Envoy handles cert exchange — application doesn't need any code changes
```

---

## Q24. Kubernetes Troubleshooting — Complete Debugging Framework

### Question

> Give me a systematic framework for troubleshooting any Kubernetes issue. Cover pod-level, service-level, node-level, and cluster-level debugging. What are the 10 most common production issues and how do you resolve each?

### Real-Life Scenario

PagerDuty fires at 3 AM. The alert says "order-service 5xx errors > 10%." You need a systematic approach to find the root cause — is it the pod, the service, the node, the network, or an external dependency?

### Answer

#### The 4-Layer Debugging Framework

```
Layer 1: Pod Level
  └── Is the pod running? What's its status? Check logs.

Layer 2: Service Level
  └── Does the service have endpoints? Is traffic reaching pods?

Layer 3: Node Level
  └── Is the node healthy? Resource pressure? Disk full?

Layer 4: Cluster Level
  └── Is the control plane healthy? CoreDNS? CNI? API server?
```

#### Layer 1: Pod Debugging

```bash
# Quick status
kubectl get pods -n backend -o wide
kubectl describe pod <pod-name> -n backend
kubectl logs <pod-name> -n backend --previous    # Previous crash logs
kubectl logs <pod-name> -n backend --tail=100 -f # Stream live logs
kubectl top pod <pod-name> -n backend            # CPU/Memory usage
```

#### Layer 2: Service Debugging

```bash
# Does the service have endpoints?
kubectl get endpoints order-service -n backend
# If ENDPOINTS is <none>: pod labels don't match service selector

# Compare labels
kubectl get svc order-service -n backend -o jsonpath='{.spec.selector}'
kubectl get pods -n backend --show-labels

# Test service DNS
kubectl run curl-test --image=curlimages/curl --rm -it -n backend \
  -- curl -v http://order-service:80/health

# Test from outside the cluster
kubectl port-forward svc/order-service 8080:80 -n backend
curl http://localhost:8080/health
```

#### Layer 3: Node Debugging

```bash
# Node status
kubectl get nodes
kubectl describe node <node-name>
# Check: Conditions (MemoryPressure, DiskPressure, PIDPressure)

# Node resource usage
kubectl top nodes

# SSH/SSM into the node
aws ssm start-session --target <instance-id>
# Check:
top                    # CPU/processes
free -m                # Memory
df -h                  # Disk space
dmesg | tail -50       # Kernel messages
journalctl -u kubelet  # Kubelet logs
```

#### Layer 4: Cluster Debugging

```bash
# API server
kubectl cluster-info
kubectl get componentstatuses    # Deprecated but still useful

# CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns

# kube-proxy
kubectl get pods -n kube-system -l k8s-app=kube-proxy

# Events (cluster-wide)
kubectl get events --all-namespaces --sort-by='.lastTimestamp' | tail -30
```

#### Top 10 Production Issues Quick Reference

| # | Issue | Symptom | Quick Fix |
|---|-------|---------|-----------|
| 1 | OOMKilled | Pod restarts, exit code 137 | Increase memory limits |
| 2 | CrashLoopBackOff | Pod restart loop | Check logs `--previous`, fix app config |
| 3 | ImagePullBackOff | Can't start container | Check image name, tag, registry creds |
| 4 | Pending pods | Pod stuck Pending | Check resources, taints, PVC, node selector |
| 5 | Service no endpoints | Can't reach service | Fix pod labels to match service selector |
| 6 | DNS failures | Can't resolve service names | Check CoreDNS pods, memory, configmap |
| 7 | Node NotReady | Node goes offline | Check kubelet, disk space, network |
| 8 | Evicted pods | Pods terminated | Check node DiskPressure, clean up images |
| 9 | Slow performance | High latency | Check CPU throttling, HPA, resource limits |
| 10 | Ingress 502/504 | Gateway errors | Check ingress controller logs, backend health |

---

## Q25. Pod Disruption Budgets (PDBs) — Ensuring Availability During Maintenance

### Question

> During a node upgrade, all 3 pods of a critical service were drained simultaneously, causing a complete outage. How do you use PodDisruptionBudgets to prevent this? What's the difference between voluntary and involuntary disruptions?

### Real-Life Scenario

Your cluster has 20 nodes. The platform team starts a rolling node upgrade. `kubectl drain` evicts all pods from each node. Unfortunately, all 3 replicas of the payment service happen to be on the same 2 nodes being drained simultaneously. Payment processing goes down for 4 minutes.

### Answer

#### Types of Disruptions

```
Voluntary (PDB can protect):
  - kubectl drain (node maintenance)
  - Cluster autoscaler scaling down
  - Rolling deployments
  - Node upgrades

Involuntary (PDB CANNOT protect):
  - Node hardware failure
  - OOMKilled
  - Node crash
  - Kernel panic
```

#### PodDisruptionBudget Configuration

```yaml
# Option 1: Minimum available
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: payment-service-pdb
  namespace: backend
spec:
  minAvailable: 2              # At least 2 pods MUST always be running
  selector:
    matchLabels:
      app: payment-service

# With 3 replicas and minAvailable: 2
# → kubectl drain can only evict 1 pod at a time
# → Must wait for the evicted pod to be rescheduled and ready before draining next

---
# Option 2: Maximum unavailable
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: order-service-pdb
  namespace: backend
spec:
  maxUnavailable: 1            # At most 1 pod can be down
  selector:
    matchLabels:
      app: order-service

---
# Option 3: Percentage
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-frontend-pdb
  namespace: frontend
spec:
  maxUnavailable: "25%"        # Max 25% can be down at once
  selector:
    matchLabels:
      app: web-frontend

# With 8 replicas: max 2 can be disrupted at once
```

#### How PDB Works During Drain

```bash
# Without PDB:
kubectl drain node-1
# ALL pods on node-1 are evicted at once → possible outage

# With PDB (minAvailable: 2, replicas: 3):
kubectl drain node-1
# Step 1: Check PDB — can we evict payment-svc pod?
#   Running: 3, minAvailable: 2 → can evict 1 → YES
# Step 2: Evict 1 pod, wait for replacement to be Ready
#   Running: 2 (+ 1 starting) → can't evict more yet
# Step 3: Replacement is Ready → Running: 3
#   Can evict another one now

# If drain gets stuck:
kubectl drain node-1 --timeout=300s
# After 300s, drain gives up with remaining pods
# Use --disable-eviction as LAST RESORT (bypasses PDB!)
```

#### Best Practices

```yaml
# ALWAYS create PDBs for:
# - Stateful services (databases, Kafka)
# - Critical microservices (payment, auth)
# - Anything with < 5 replicas

# DON'T create PDBs that block all evictions:
# minAvailable: 3 with replicas: 3 → nothing can ever be drained!
# Always leave room: minAvailable should be < replicas
```

```bash
# Check PDB status
kubectl get pdb -n backend
# NAME                  MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
# payment-service-pdb   2               N/A                1                    30d
# order-service-pdb     N/A             1                  1                    30d

# ALLOWED DISRUPTIONS = 0 means drain will BLOCK!
# This happens when running pods = minAvailable
```

---

## Q26. Kubernetes Logging Architecture — EFK/ELK Stack on Kubernetes

### Question

> Your cluster runs 200+ pods generating 50GB of logs per day. You need centralized logging with search, filtering, and alerting. Design a logging architecture using Fluentd/Fluent Bit + Elasticsearch + Kibana (or CloudWatch). How do you handle log rotation, multi-line logs, and log levels per environment?

### Real-Life Scenario

A developer searches for a specific error across 200 pods using `kubectl logs` — it takes 45 minutes to find the right pod. Meanwhile, pods that crash have their logs lost because nobody checked before the pod was replaced. You need centralized logging.

### Answer

#### Logging Architecture

```
Pods (stdout/stderr)
    │
    ▼
Node filesystem (/var/log/containers/*.log)
    │
    ▼
Fluent Bit DaemonSet (lightweight, runs on every node)
    │
    ├──→ Amazon CloudWatch Logs (managed, simple)
    │
    ├──→ Elasticsearch (self-hosted or OpenSearch)
    │         │
    │         └──→ Kibana (visualization + search)
    │
    └──→ S3 (long-term archive, cheap)
```

#### Fluent Bit DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
spec:
  selector:
    matchLabels:
      app: fluent-bit
  template:
    metadata:
      labels:
        app: fluent-bit
    spec:
      serviceAccountName: fluent-bit
      tolerations:
        - operator: Exists           # Run on ALL nodes
      containers:
        - name: fluent-bit
          image: fluent/fluent-bit:2.2
          volumeMounts:
            - name: varlog
              mountPath: /var/log
              readOnly: true
            - name: containers
              mountPath: /var/log/containers
              readOnly: true
            - name: config
              mountPath: /fluent-bit/etc/
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: containers
          hostPath:
            path: /var/log/containers
        - name: config
          configMap:
            name: fluent-bit-config
```

#### Fluent Bit Configuration for CloudWatch

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: logging
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         5
        Log_Level     info
        Parsers_File  parsers.conf

    [INPUT]
        Name              tail
        Path              /var/log/containers/*.log
        Parser            cri
        Tag               kube.*
        Refresh_Interval  5
        Mem_Buf_Limit     50MB
        Skip_Long_Lines   On

    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Merge_Log           On           # Parse JSON logs
        K8S-Logging.Parser  On
        K8S-Logging.Exclude On

    [FILTER]
        Name    grep
        Match   kube.*
        Exclude log healthcheck          # Exclude health check noise

    [OUTPUT]
        Name                cloudwatch_logs
        Match               kube.*
        region              ap-south-1
        log_group_name      /eks/production
        log_stream_prefix   fluentbit-
        auto_create_group   true

  parsers.conf: |
    [PARSER]
        Name        cri
        Format      regex
        Regex       ^(?<time>[^ ]+) (?<stream>stdout|stderr) (?<logtag>[^ ]*) (?<log>.*)$
        Time_Key    time
        Time_Format %Y-%m-%dT%H:%M:%S.%L%z

    [PARSER]
        Name        json
        Format      json
        Time_Key    timestamp
        Time_Format %Y-%m-%dT%H:%M:%S.%L%z
```

#### Application Logging Best Practices

```json
// Structured JSON logging (recommended)
{
  "timestamp": "2026-05-11T14:32:00.123Z",
  "level": "ERROR",
  "service": "order-service",
  "traceId": "abc123def456",
  "message": "Payment failed",
  "orderId": "ORD-78901",
  "error": "Connection timeout to payment gateway",
  "duration_ms": 30000
}
```

```yaml
# Set log level per environment via ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-config
  namespace: backend
data:
  LOG_LEVEL: "info"       # dev: "debug", staging: "info", prod: "warn"
  LOG_FORMAT: "json"      # Always JSON in Kubernetes (parseable by Fluent Bit)
```

---

## Q27. Kubernetes etcd — Backup, Restore, and Disaster Recovery

### Question

> Your Kubernetes cluster's etcd database is the brain of the cluster — it stores ALL cluster state. How do you back up etcd, restore it after corruption, and plan for disaster recovery? What happens if etcd is unavailable?

### Real-Life Scenario

During a control plane upgrade, etcd data gets corrupted. The API server can't start. `kubectl` returns "connection refused." All deployments, services, configmaps, and secrets are stored in etcd — without it, the cluster is effectively dead. You need to restore from backup.

### Answer

#### What etcd Stores

```
Everything in Kubernetes:
├── Deployments, ReplicaSets, Pods (desired state)
├── Services, Endpoints
├── ConfigMaps, Secrets
├── RBAC (Roles, RoleBindings)
├── Namespaces
├── PersistentVolumes, PVCs
├── Custom Resources (CRDs)
└── Cluster state (node info, leases)

If etcd is lost without backup → ALL configuration is gone
(Running pods continue but can't be managed, scaled, or recovered)
```

#### Backing Up etcd

```bash
# For self-managed clusters (kubeadm-based):
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot-$(date +%Y%m%d).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify the backup
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot-20260511.db --write-out=table
# +----------+----------+------------+------------+
# |   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
# +----------+----------+------------+------------+
# | 2a6baf28 | 1285600  |       1245 |     5.8 MB |
# +----------+----------+------------+------------+

# Upload to S3
aws s3 cp /backup/etcd-snapshot-20260511.db \
  s3://mycompany-k8s-backups/etcd/etcd-snapshot-20260511.db

# For EKS: AWS manages etcd backups automatically
# You don't need to back up etcd separately
# But you SHOULD back up your manifests in Git (GitOps)
```

#### Automated CronJob for etcd Backup

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
  namespace: kube-system
spec:
  schedule: "0 */6 * * *"      # Every 6 hours
  jobTemplate:
    spec:
      template:
        spec:
          hostNetwork: true
          nodeSelector:
            node-role.kubernetes.io/control-plane: ""
          tolerations:
            - effect: NoSchedule
              operator: Exists
          containers:
            - name: backup
              image: bitnami/etcd:3.5
              command:
                - /bin/sh
                - -c
                - |
                  etcdctl snapshot save /backup/etcd-$(date +%Y%m%d-%H%M).db \
                    --endpoints=https://127.0.0.1:2379 \
                    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
                    --cert=/etc/kubernetes/pki/etcd/server.crt \
                    --key=/etc/kubernetes/pki/etcd/server.key
                  # Upload to S3
                  aws s3 cp /backup/etcd-$(date +%Y%m%d-%H%M).db \
                    s3://mycompany-k8s-backups/etcd/
                  # Cleanup old local backups
                  find /backup -name "etcd-*.db" -mtime +7 -delete
              volumeMounts:
                - name: etcd-certs
                  mountPath: /etc/kubernetes/pki/etcd
                  readOnly: true
                - name: backup-dir
                  mountPath: /backup
          volumes:
            - name: etcd-certs
              hostPath:
                path: /etc/kubernetes/pki/etcd
            - name: backup-dir
              hostPath:
                path: /var/lib/etcd-backups
          restartPolicy: OnFailure
```

#### Restoring etcd from Backup

```bash
# Stop the API server and etcd
sudo systemctl stop kube-apiserver
sudo systemctl stop etcd

# Restore from snapshot
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot-20260511.db \
  --data-dir=/var/lib/etcd-restored \
  --initial-cluster="master=https://10.0.1.10:2380" \
  --initial-advertise-peer-urls=https://10.0.1.10:2380 \
  --name=master

# Replace etcd data directory
sudo mv /var/lib/etcd /var/lib/etcd-corrupted
sudo mv /var/lib/etcd-restored /var/lib/etcd
sudo chown -R etcd:etcd /var/lib/etcd

# Restart etcd and API server
sudo systemctl start etcd
sudo systemctl start kube-apiserver

# Verify
kubectl get nodes
kubectl get pods --all-namespaces
```

---

## Q28. Kubernetes Admission Controllers and Webhooks

### Question

> You need to enforce custom rules that go beyond RBAC and PodSecurity — for example, all images must come from your private registry, all pods must have resource limits, and all deployments must have at least 2 replicas. How do you implement Validating and Mutating Admission Webhooks?

### Real-Life Scenario

A developer deploys a container image from Docker Hub (`nginx:latest`) instead of your approved private registry. The image contains a cryptocurrency miner. You need admission control to block unapproved images.

### Answer

#### Admission Controller Flow

```
kubectl apply → API Server → Authentication → Authorization (RBAC)
                                                    │
                                                    ▼
                                            Mutating Webhooks
                                            (modify the request)
                                                    │
                                                    ▼
                                            Validating Webhooks
                                            (accept/reject the request)
                                                    │
                                                    ▼
                                                  etcd
                                            (persist if accepted)
```

#### Kyverno — Policy-as-YAML (Simpler than OPA)

```yaml
# Policy: Block images not from approved registries
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-image-registries
spec:
  validationFailureAction: Enforce    # Block violating resources
  rules:
    - name: validate-registries
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Images must be from approved registries (ECR or mycompany registry)"
        pattern:
          spec:
            containers:
              - image: "*.dkr.ecr.*.amazonaws.com/* | mycompany/*"
            initContainers:
              - image: "*.dkr.ecr.*.amazonaws.com/* | mycompany/*"
```

```yaml
# Policy: Require resource limits on all containers
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-limits
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-limits
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "All containers must have CPU and memory limits"
        pattern:
          spec:
            containers:
              - resources:
                  limits:
                    memory: "?*"
                    cpu: "?*"
```

```yaml
# Mutating Policy: Auto-add labels if missing
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-default-labels
spec:
  rules:
    - name: add-team-label
      match:
        any:
          - resources:
              kinds:
                - Deployment
      mutate:
        patchStrategicMerge:
          metadata:
            labels:
              +(managed-by): "kubernetes"     # + means "add if not present"
              +(environment): "production"
```

```bash
# Test if a resource violates policies
kubectl apply -f bad-deployment.yaml
# Error: "Images must be from approved registries (ECR or mycompany registry)"

# Check policy reports
kubectl get policyreport -A
kubectl get clusterpolicyreport
```

---

## Q29. Kubernetes Horizontal Scaling with Custom and External Metrics

### Question

> The default CPU/memory HPA isn't sufficient. You need to scale based on: messages in an SQS queue, requests per second from Prometheus, active WebSocket connections, and response latency (P99). How do you configure HPA with custom and external metrics?

### Real-Life Scenario

Your order-processing service pulls messages from an SQS queue. During peak hours, 50,000 messages pile up but pods remain at 3 replicas because CPU is only at 20% — the bottleneck is I/O (waiting for SQS), not CPU. You need to scale based on queue depth.

### Answer

#### Custom Metrics Pipeline

```
Prometheus scrapes app metrics → Prometheus Adapter exposes them → HPA queries them

App (/metrics) → Prometheus → Prometheus Adapter → Kubernetes Metrics API → HPA
```

```bash
# Install Prometheus Adapter
helm install prometheus-adapter prometheus-community/prometheus-adapter \
  -n monitoring \
  -f adapter-values.yaml
```

#### HPA with Custom Prometheus Metric

```yaml
# Prometheus Adapter config — map Prometheus metric to K8s metric
rules:
  - seriesQuery: 'http_requests_total{namespace!="",pod!=""}'
    resources:
      overrides:
        namespace: {resource: "namespace"}
        pod: {resource: "pod"}
    name:
      matches: "^(.*)_total$"
      as: "${1}_per_second"
    metricsQuery: 'rate(<<.Series>>{<<.LabelMatchers>>}[2m])'
```

```yaml
# HPA using custom metric
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-processor-hpa
  namespace: backend
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-processor
  minReplicas: 3
  maxReplicas: 50
  metrics:
    # Scale on custom metric: requests per second per pod
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"        # Scale when avg RPS > 100 per pod

    # Scale on external metric: SQS queue depth
    - type: External
      external:
        metric:
          name: sqs_queue_messages
          selector:
            matchLabels:
              queue: "order-processing-queue"
        target:
          type: Value
          value: "1000"              # Scale when queue > 1000 messages

    # Scale on P99 latency
    - type: Pods
      pods:
        metric:
          name: http_request_duration_p99
        target:
          type: AverageValue
          averageValue: "500"        # Scale when P99 > 500ms
```

```bash
# Verify custom metrics are available
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1" | jq .
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/backend/pods/*/http_requests_per_second" | jq .

# Check external metrics
kubectl get --raw "/apis/external.metrics.k8s.io/v1beta1/namespaces/backend/sqs_queue_messages" | jq .
```

#### KEDA — Event-Driven Autoscaler (Simpler for External Metrics)

```yaml
# KEDA makes scaling on external events MUCH simpler
# Install: helm install keda kedacore/keda -n keda

apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor
  namespace: backend
spec:
  scaleTargetRef:
    name: order-processor
  minReplicaCount: 1              # Can scale to 0!
  maxReplicaCount: 50
  pollingInterval: 15
  cooldownPeriod: 300
  triggers:
    # Scale based on SQS queue
    - type: aws-sqs-queue
      metadata:
        queueURL: https://sqs.ap-south-1.amazonaws.com/123456789/order-processing
        queueLength: "5"          # 5 messages per pod
        awsRegion: "ap-south-1"
      authenticationRef:
        name: aws-credentials

    # Scale based on Prometheus metric
    - type: prometheus
      metadata:
        serverAddress: http://prometheus.monitoring.svc:9090
        metricName: http_requests_per_second
        query: sum(rate(http_requests_total{service="order-processor"}[2m]))
        threshold: "100"
```

---

## Q30. Kubernetes Cost Optimization — Spot Instances, Right-Sizing, and Idle Resources

### Question

> Your EKS cluster costs $25,000/month. The CFO wants a 50% reduction. Walk me through a complete cost optimization strategy including spot instances, right-sizing, idle resource cleanup, and Karpenter.

### Real-Life Scenario

Analysis of your cluster shows: 60% of pod CPU requests are unused, dev/staging environments run 24/7 but are only used during business hours, and you're using on-demand instances for all workloads including stateless batch processing.

### Answer

#### Cost Analysis — Where's the Money Going?

```bash
# Step 1: Node costs
kubectl get nodes -o custom-columns=NAME:.metadata.name,TYPE:.metadata.labels.node\\.kubernetes\\.io/instance-type,AZ:.metadata.labels.topology\\.kubernetes\\.io/zone

# Step 2: Pod resource utilization
kubectl top pods --all-namespaces --sort-by=cpu | head -20
kubectl top pods --all-namespaces --sort-by=memory | head -20

# Step 3: Over-provisioned pods (requests >> actual usage)
# Use kubecost, Datadog, or custom script
```

#### Strategy 1: Spot Instances for Non-Critical Workloads

```yaml
# Karpenter NodePool with spot instances
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: spot-workers
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot"]                # Spot instances (60-90% cheaper!)
        - key: node.kubernetes.io/instance-type
          operator: In
          values:
            - m5.large
            - m5.xlarge
            - m5a.large
            - m5a.xlarge
            - m6i.large
            - m6i.xlarge             # Multiple types for spot availability
      nodeClassRef:
        name: default
  limits:
    cpu: 100
  disruption:
    consolidationPolicy: WhenUnderutilized
```

```yaml
# Mark stateless workloads as spot-friendly
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-processor
spec:
  template:
    spec:
      nodeSelector:
        karpenter.sh/capacity-type: spot
      # Handle spot interruptions gracefully
      terminationGracePeriodSeconds: 60
      containers:
        - name: processor
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 30"]   # Finish current work
```

**What should go on spot vs on-demand?**

| Workload | Spot? | Why |
|----------|-------|-----|
| Stateless APIs | Yes (with PDB) | Can tolerate restarts |
| Batch/CronJobs | Yes | Checkpointable, retriable |
| CI/CD workers | Yes | Short-lived, retriable |
| Databases (StatefulSet) | **NO** | Data loss risk |
| Kafka/Redis | **NO** | Stateful, rebalancing is expensive |
| Control plane add-ons | **NO** | Critical infrastructure |

#### Strategy 2: Scale Down Non-Prod After Hours

```yaml
# CronJob: Scale dev to 0 at 8 PM, back up at 8 AM
apiVersion: batch/v1
kind: CronJob
metadata:
  name: scale-down-dev
  namespace: kube-system
spec:
  schedule: "0 20 * * 1-5"     # 8 PM Mon-Fri (IST)
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: scaler
          containers:
            - name: scaler
              image: bitnami/kubectl:1.29
              command:
                - /bin/sh
                - -c
                - |
                  for deploy in $(kubectl get deployments -n dev -o name); do
                    kubectl scale $deploy --replicas=0 -n dev
                  done
          restartPolicy: OnFailure

---
# Scale back up at 8 AM
apiVersion: batch/v1
kind: CronJob
metadata:
  name: scale-up-dev
  namespace: kube-system
spec:
  schedule: "0 8 * * 1-5"
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: scaler
          containers:
            - name: scaler
              image: bitnami/kubectl:1.29
              command:
                - /bin/sh
                - -c
                - |
                  kubectl scale deployment --all --replicas=2 -n dev
          restartPolicy: OnFailure
```

#### Strategy 3: Right-Size with VPA Recommendations

```bash
# Deploy VPA in recommendation mode
kubectl get vpa --all-namespaces -o custom-columns=\
  NAME:.metadata.name,\
  NAMESPACE:.metadata.namespace,\
  TARGET_CPU:.status.recommendation.containerRecommendations[0].target.cpu,\
  TARGET_MEM:.status.recommendation.containerRecommendations[0].target.memory,\
  CURRENT_CPU:.spec.targetRef.name

# Apply recommendations:
# Current request: 2 CPU, 4Gi → VPA recommends: 500m, 1Gi
# Savings per pod: 75% CPU, 75% memory
```

#### Strategy 4: Clean Up Idle Resources

```bash
# Find deployments with 0 replicas (forgotten scale-downs)
kubectl get deployments --all-namespaces -o json | \
  jq -r '.items[] | select(.spec.replicas == 0) | "\(.metadata.namespace)/\(.metadata.name)"'

# Find unattached PVCs (data volumes nobody uses)
kubectl get pvc --all-namespaces -o json | \
  jq -r '.items[] | select(.status.phase == "Bound") | "\(.metadata.namespace)/\(.metadata.name) - \(.spec.resources.requests.storage)"'

# Find orphaned ConfigMaps and Secrets (not referenced by any pod)
# This requires a custom script or tool like kubeclean

# Find LoadBalancers (each costs ~$18/month)
kubectl get svc --all-namespaces -o json | \
  jq -r '.items[] | select(.spec.type == "LoadBalancer") | "\(.metadata.namespace)/\(.metadata.name)"'
```

#### Cost Savings Summary

| Strategy | Before | After | Monthly Savings |
|----------|--------|-------|-----------------|
| Spot instances (stateless) | $10,000 | $3,000 | $7,000 |
| Right-sizing (VPA) | $8,000 | $3,000 | $5,000 |
| Dev/staging off-hours | $4,000 | $1,200 | $2,800 |
| Cleanup idle resources | $2,000 | $200 | $1,800 |
| Replace LoadBalancers with Ingress | $1,000 | $200 | $800 |
| **Total** | **$25,000** | **$7,600** | **$17,400 (70%)** |

---

*More Kubernetes questions coming in the next batch (Q31–Q50)...*

