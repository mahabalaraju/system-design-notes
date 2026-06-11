# Kubernetes — Complete Notes

> Advanced reference covering Kubernetes architecture, workloads, networking, storage, configuration, security, and production patterns.

---

## Table of Contents

- [What is Kubernetes?](#what-is-kubernetes)
- [Architecture](#architecture)
- [Core Concepts](#core-concepts)
- [kubectl — CLI Reference](#kubectl--cli-reference)
- [Workloads](#workloads)
  - [Pod](#pod)
  - [Deployment](#deployment)
  - [StatefulSet](#statefulset)
  - [DaemonSet](#daemonset)
  - [Job & CronJob](#job--cronjob)
- [Services & Networking](#services--networking)
  - [Service Types](#service-types)
  - [Ingress](#ingress)
  - [DNS](#dns)
  - [NetworkPolicy](#networkpolicy)
- [Configuration](#configuration)
  - [ConfigMap](#configmap)
  - [Secret](#secret)
  - [Environment Variables](#environment-variables)
- [Storage](#storage)
  - [Volumes](#volumes)
  - [PersistentVolume & PersistentVolumeClaim](#persistentvolume--persistentvolumeclaim)
  - [StorageClass](#storageclass)
- [Resource Management](#resource-management)
  - [Requests & Limits](#requests--limits)
  - [LimitRange](#limitrange)
  - [ResourceQuota](#resourcequota)
  - [Horizontal Pod Autoscaler](#horizontal-pod-autoscaler)
  - [Vertical Pod Autoscaler](#vertical-pod-autoscaler)
- [Health Probes](#health-probes)
- [Scheduling](#scheduling)
  - [Node Selector & Affinity](#node-selector--affinity)
  - [Taints & Tolerations](#taints--tolerations)
  - [Pod Disruption Budget](#pod-disruption-budget)
- [Security](#security)
  - [RBAC](#rbac)
  - [ServiceAccount](#serviceaccount)
  - [SecurityContext](#securitycontext)
  - [Network Policies](#network-policies)
- [Namespaces](#namespaces)
- [Helm](#helm)
- [Java Spring Boot on Kubernetes](#java-spring-boot-on-kubernetes)
- [Observability](#observability)
- [Production Best Practices](#production-best-practices)
- [Debugging & Troubleshooting](#debugging--troubleshooting)
- [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## What is Kubernetes?

Kubernetes (K8s) is an open-source container orchestration platform that automates deployment, scaling, and management of containerized applications.

**What it solves:**
- Automatically restarts failed containers
- Scales apps up/down based on load
- Distributes load across multiple instances
- Rolls out updates with zero downtime
- Self-heals — replaces dead nodes/pods
- Service discovery and load balancing
- Secret and configuration management

---

## Architecture

```
┌─────────────────────── Control Plane ──────────────────────────┐
│                                                                  │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ API Server   │  │  Scheduler   │  │  Controller Manager  │  │
│  │ (kube-       │  │ (kube-       │  │  (kube-controller-   │  │
│  │  apiserver)  │  │  scheduler)  │  │   manager)           │  │
│  └──────┬──────┘  └──────────────┘  └──────────────────────┘  │
│         │                                                        │
│  ┌──────▼──────┐                                                 │
│  │    etcd      │  ← distributed key-value store (cluster state) │
│  └─────────────┘                                                 │
└─────────────────────────────────────────────────────────────────┘
          │ watches / updates via API
┌─────────▼──────────────────────────────────────────────────────┐
│                        Worker Nodes                              │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Node 1                                                    │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────────┐   │  │
│  │  │  kubelet  │  │kube-proxy│  │  Container Runtime   │   │  │
│  │  │(node agent│  │(iptables/│  │  (containerd/cri-o)  │   │  │
│  │  │ reports   │  │ ipvs)    │  │                      │   │  │
│  │  │ to API)   │  │          │  │  ┌─────┐  ┌─────┐   │   │  │
│  │  └──────────┘  └──────────┘  │  │Pod  │  │Pod  │   │   │  │
│  │                               │  └─────┘  └─────┘   │   │  │
│  │                               └──────────────────────┘   │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

**Control Plane Components:**

| Component | Role |
|---|---|
| **kube-apiserver** | Single entry point for all API calls. Validates and processes REST requests. Everything talks through it. |
| **etcd** | Distributed KV store — single source of truth for all cluster state. Only apiserver reads/writes etcd. |
| **kube-scheduler** | Watches for unscheduled Pods; assigns them to nodes based on resources, affinity, taints. |
| **kube-controller-manager** | Runs controllers: Deployment, ReplicaSet, Node, Endpoint controllers. Reconciles desired vs actual state. |
| **cloud-controller-manager** | Integrates with cloud provider APIs (AWS, GCP, Azure) for LBs, storage, nodes. |

**Worker Node Components:**

| Component | Role |
|---|---|
| **kubelet** | Node agent. Ensures containers in Pods are running per spec. Reports node/pod status to API server. |
| **kube-proxy** | Maintains network rules (iptables/ipvs). Implements Service abstraction — routes traffic to Pods. |
| **Container Runtime** | Actually runs containers. containerd (default), CRI-O. |

---

## Core Concepts

| Concept | Description |
|---|---|
| **Pod** | Smallest deployable unit. One or more containers sharing network + storage. |
| **ReplicaSet** | Ensures N replicas of a Pod are always running. Managed by Deployment. |
| **Deployment** | Declarative update management for Pods + ReplicaSets. Rolling updates, rollback. |
| **StatefulSet** | Like Deployment but for stateful apps. Stable pod names, ordered scaling, persistent storage. |
| **DaemonSet** | Runs one Pod per node (logging agents, monitoring). |
| **Job** | Runs a Pod to completion (batch tasks). |
| **CronJob** | Runs Jobs on a schedule. |
| **Service** | Stable network endpoint for a set of Pods. Load balances across them. |
| **Ingress** | HTTP/HTTPS routing rules into the cluster. |
| **ConfigMap** | Non-sensitive config data (key-value). |
| **Secret** | Sensitive data (passwords, tokens) — base64 encoded. |
| **PersistentVolume** | Cluster-level storage resource. |
| **PersistentVolumeClaim** | Pod's request for storage. |
| **Namespace** | Virtual cluster — logical isolation of resources. |
| **ServiceAccount** | Identity for Pods to interact with the API server. |
| **Node** | Worker machine (VM or physical). |

---

## kubectl — CLI Reference

```bash
# Context & cluster
kubectl config get-contexts               # list all contexts
kubectl config current-context            # current context
kubectl config use-context my-cluster     # switch context
kubectl config set-context --current --namespace=my-ns  # set default namespace

# Get resources
kubectl get pods
kubectl get pods -n my-namespace
kubectl get pods -A                       # all namespaces
kubectl get pods -o wide                  # extra info (node, IP)
kubectl get pods -o yaml                  # full YAML output
kubectl get pods -o json | jq
kubectl get pods --show-labels
kubectl get pods -l app=order-service     # filter by label
kubectl get pods --field-selector=status.phase=Running
kubectl get all                           # pods, services, deployments, etc.
kubectl get all -n my-namespace

# Describe (detailed info + events)
kubectl describe pod my-pod
kubectl describe deployment order-service
kubectl describe node my-node

# Apply / Create
kubectl apply -f deployment.yml           # create or update
kubectl apply -f ./k8s/                   # apply all files in directory
kubectl apply -f https://url/file.yml
kubectl create -f deployment.yml          # create only (fails if exists)

# Delete
kubectl delete -f deployment.yml
kubectl delete pod my-pod
kubectl delete pod my-pod --force --grace-period=0   # immediate
kubectl delete pods -l app=order-service
kubectl delete all -l app=order-service

# Logs
kubectl logs my-pod
kubectl logs my-pod -c container-name     # multi-container pod
kubectl logs -f my-pod                    # follow
kubectl logs --tail=100 my-pod
kubectl logs --previous my-pod            # previous container instance (after crash)
kubectl logs -l app=order-service --all-containers  # all matching pods

# Exec
kubectl exec -it my-pod -- bash
kubectl exec -it my-pod -c container -- sh
kubectl exec my-pod -- env
kubectl exec my-pod -- cat /app/config.yml

# Port forward (local debugging)
kubectl port-forward pod/my-pod 8080:8080
kubectl port-forward service/order-service 8080:8080
kubectl port-forward deployment/order-service 8080:8080

# Copy files
kubectl cp my-pod:/app/logs/app.log ./app.log
kubectl cp ./config.yml my-pod:/app/

# Scale
kubectl scale deployment order-service --replicas=5
kubectl scale --replicas=0 deployment order-service  # stop all pods

# Rollout
kubectl rollout status deployment order-service
kubectl rollout history deployment order-service
kubectl rollout undo deployment order-service         # rollback to previous
kubectl rollout undo deployment order-service --to-revision=3
kubectl rollout restart deployment order-service      # rolling restart

# Edit (opens in $EDITOR)
kubectl edit deployment order-service
kubectl edit configmap app-config

# Patch
kubectl patch deployment order-service -p '{"spec":{"replicas":3}}'

# Labels & annotations
kubectl label pod my-pod env=prod
kubectl annotate pod my-pod contact=team@example.com

# Top (resource usage — requires metrics-server)
kubectl top nodes
kubectl top pods
kubectl top pods -n my-namespace --containers

# Events
kubectl get events --sort-by='.lastTimestamp'
kubectl get events -n my-namespace

# API resources
kubectl api-resources                     # list all resource types
kubectl explain pod.spec.containers       # inline docs

# Dry run
kubectl apply -f deployment.yml --dry-run=client   # validate locally
kubectl apply -f deployment.yml --dry-run=server   # validate on server

# Diff
kubectl diff -f deployment.yml            # show what would change
```

---

## Workloads

### Pod

The smallest deployable unit. Containers in a Pod share the same network namespace (localhost) and can share volumes.

```yaml
# pod.yml
apiVersion: v1
kind: Pod
metadata:
  name: order-pod
  namespace: production
  labels:
    app: order-service
    version: "1.0.0"
  annotations:
    contact: "team@example.com"
spec:
  containers:
    - name: order-service
      image: myrepo/order-service:1.0.0
      ports:
        - containerPort: 8080
          name: http
      env:
        - name: SPRING_PROFILES_ACTIVE
          value: "prod"
      resources:
        requests:
          memory: "256Mi"
          cpu: "250m"
        limits:
          memory: "512Mi"
          cpu: "500m"
      livenessProbe:
        httpGet:
          path: /actuator/health/liveness
          port: 8080
        initialDelaySeconds: 45
        periodSeconds: 10
      readinessProbe:
        httpGet:
          path: /actuator/health/readiness
          port: 8080
        initialDelaySeconds: 30
        periodSeconds: 5
      volumeMounts:
        - name: config-volume
          mountPath: /app/config
          readOnly: true
        - name: logs
          mountPath: /app/logs

    # Sidecar container (e.g., log shipper)
    - name: log-shipper
      image: fluent/fluent-bit:latest
      volumeMounts:
        - name: logs
          mountPath: /logs
          readOnly: true

  # Init container — runs before app containers
  initContainers:
    - name: wait-for-db
      image: busybox:1.36
      command: ['sh', '-c',
        'until nc -z postgres 5432; do echo waiting for postgres; sleep 2; done']

  volumes:
    - name: config-volume
      configMap:
        name: order-config
    - name: logs
      emptyDir: {}

  restartPolicy: Always         # Always | OnFailure | Never
  serviceAccountName: order-sa
  terminationGracePeriodSeconds: 60
```

**Multi-container Pod patterns:**

| Pattern | Description | Example |
|---|---|---|
| **Sidecar** | Extends main container | Log shipper, proxy, secret injector |
| **Ambassador** | Proxy for external services | Local proxy to remote DB |
| **Adapter** | Transforms output | Metrics format converter |
| **Init Container** | Runs to completion before app starts | DB migration, wait-for-service |

---

### Deployment

Manages ReplicaSets for stateless applications. Handles rolling updates and rollbacks.

```yaml
# deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
  labels:
    app: order-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service            # must match template labels

  strategy:
    type: RollingUpdate             # RollingUpdate | Recreate
    rollingUpdate:
      maxSurge: 1                   # max pods above desired during update
      maxUnavailable: 0             # max pods unavailable during update
                                    # maxUnavailable=0 = zero-downtime

  # Recreate strategy: kill all old pods, then start new ones (downtime but clean)
  # strategy:
  #   type: Recreate

  minReadySeconds: 10               # pod must be ready N seconds before marked available
  revisionHistoryLimit: 5           # number of old ReplicaSets to keep (for rollback)
  progressDeadlineSeconds: 600      # fail if not progressed in 10 minutes

  template:
    metadata:
      labels:
        app: order-service          # must match selector.matchLabels
        version: "1.2.0"
    spec:
      containers:
        - name: order-service
          image: myrepo/order-service:1.2.0
          imagePullPolicy: Always   # Always | IfNotPresent | Never
          ports:
            - containerPort: 8080
              name: http
          envFrom:
            - configMapRef:
                name: order-config
            - secretRef:
                name: order-secrets
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "1000m"
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 45
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 20
            periodSeconds: 5
            failureThreshold: 3
          startupProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            failureThreshold: 30    # allows 30 * 10s = 300s for slow startup
            periodSeconds: 10
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]  # drain connections before SIGTERM

      terminationGracePeriodSeconds: 60
      affinity:
        podAntiAffinity:            # spread pods across nodes
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: order-service
                topologyKey: kubernetes.io/hostname
```

---

### StatefulSet

For stateful applications requiring stable identity, ordered deployment, and persistent storage (databases, Kafka, Elasticsearch).

```yaml
# statefulset.yml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: postgres-headless    # must match a headless Service
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0                  # only update pods with ordinal >= partition

  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data

  # PVC template — each Pod gets its own PVC
  volumeClaimTemplates:
    - metadata:
        name: postgres-data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 20Gi

---
# Headless service — gives each pod its own DNS
# postgres-0.postgres-headless.production.svc.cluster.local
# postgres-1.postgres-headless.production.svc.cluster.local
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: production
spec:
  clusterIP: None                   # headless — no virtual IP
  selector:
    app: postgres
  ports:
    - port: 5432
```

**StatefulSet vs Deployment:**

| | Deployment | StatefulSet |
|---|---|---|
| Pod names | Random (order-abc123) | Stable ordered (postgres-0, postgres-1) |
| Pod DNS | Not stable | Stable per-pod DNS |
| Storage | Shared or stateless | Dedicated PVC per pod |
| Scaling | Any order | Ordered (0→1→2 up, 2→1→0 down) |
| Use case | Stateless apps | Databases, Kafka, Elasticsearch |

---

### DaemonSet

Runs exactly one Pod on every node (or matching nodes). Used for node-level agents.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: fluentd
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          effect: NoSchedule        # run on control plane nodes too
      containers:
        - name: fluentd
          image: fluent/fluentd:v1.16
          resources:
            limits:
              memory: "200Mi"
              cpu: "100m"
          volumeMounts:
            - name: varlog
              mountPath: /var/log
              readOnly: true
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
```

**DaemonSet use cases:** Fluentd/Filebeat (log collection), Prometheus Node Exporter (metrics), kube-proxy (networking), CNI plugins, security agents.

---

### Job & CronJob

```yaml
# job.yml — run to completion
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  completions: 1                    # total successful completions needed
  parallelism: 1                    # pods running in parallel
  backoffLimit: 3                   # retry on failure
  activeDeadlineSeconds: 300        # max 5 minutes total
  ttlSecondsAfterFinished: 3600     # auto-delete job 1hr after completion
  template:
    spec:
      restartPolicy: OnFailure      # Jobs: OnFailure or Never (not Always)
      containers:
        - name: migration
          image: myrepo/order-service:1.2.0
          command: ["java", "-jar", "app.jar", "--migrate"]
          envFrom:
            - secretRef:
                name: db-secrets

---
# cronjob.yml — scheduled jobs
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-report
spec:
  schedule: "0 2 * * *"            # cron syntax — 2 AM daily
  timeZone: "Asia/Kolkata"         # timezone (K8s 1.27+)
  concurrencyPolicy: Forbid         # Allow | Forbid | Replace
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  startingDeadlineSeconds: 60      # skip if missed by > 60s
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: report
              image: myrepo/reports:1.0.0
              envFrom:
                - configMapRef:
                    name: report-config
```

---

## Services & Networking

### Service Types

A Service gives a stable IP and DNS name to a set of Pods (selected by labels).

```yaml
# ClusterIP — internal only (default)
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: production
spec:
  type: ClusterIP                   # only reachable within cluster
  selector:
    app: order-service              # routes to pods with this label
  ports:
    - name: http
      port: 8080                    # service port
      targetPort: 8080              # container port (or named port)
      protocol: TCP

---
# NodePort — exposes on each node's IP at a static port
apiVersion: v1
kind: Service
metadata:
  name: order-service-nodeport
spec:
  type: NodePort
  selector:
    app: order-service
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30080               # 30000–32767; auto-assigned if omitted

---
# LoadBalancer — provisions cloud load balancer (AWS ALB/NLB, GCP LB)
apiVersion: v1
kind: Service
metadata:
  name: order-service-lb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  selector:
    app: order-service
  ports:
    - port: 80
      targetPort: 8080

---
# ExternalName — maps service to external DNS
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: mydb.rds.amazonaws.com  # resolved externally

---
# Headless — no ClusterIP; DNS returns pod IPs directly (for StatefulSets)
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - port: 5432
```

**Service type comparison:**

| Type | Accessible from | Use case |
|---|---|---|
| `ClusterIP` | Within cluster only | Internal service communication |
| `NodePort` | Outside via node IP:port | Dev/test, simple external access |
| `LoadBalancer` | Internet via cloud LB | Production external traffic |
| `ExternalName` | Within cluster | Route to external service by DNS |
| Headless | Within cluster (pod IPs) | StatefulSet, direct pod access |

---

### Ingress

HTTP/HTTPS layer-7 routing into the cluster. Requires an Ingress Controller (Nginx, Traefik, AWS ALB).

```yaml
# ingress.yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    cert-manager.io/cluster-issuer: letsencrypt-prod   # auto TLS
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.example.com
      secretName: api-tls-cert      # cert-manager populates this

  rules:
    - host: api.example.com
      http:
        paths:
          - path: /orders
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 8080

          - path: /users
            pathType: Prefix
            backend:
              service:
                name: user-service
                port:
                  number: 8080

          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
```

---

### DNS

```
Kubernetes DNS format:
  <service-name>.<namespace>.svc.cluster.local
  <pod-ip>.<namespace>.pod.cluster.local (with dashes)

Examples:
  order-service.production.svc.cluster.local
  postgres-0.postgres-headless.production.svc.cluster.local  (StatefulSet pod)

Short forms (within same namespace):
  order-service                         → resolves to ClusterIP
  order-service.production              → from another namespace

DNS resolution:
  Pod search domains: production.svc.cluster.local, svc.cluster.local, cluster.local
  So within same namespace: just use service name
  Cross-namespace: service-name.namespace
```

---

### NetworkPolicy

Controls traffic between Pods. Default: all traffic allowed. With NetworkPolicy: only explicitly allowed traffic.

```yaml
# network-policy.yml — restrict order-service traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: order-service-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: order-service            # applies to these pods

  policyTypes:
    - Ingress
    - Egress

  ingress:
    # Allow traffic only from api-gateway and monitoring
    - from:
        - podSelector:
            matchLabels:
              app: api-gateway
        - podSelector:
            matchLabels:
              app: prometheus
      ports:
        - protocol: TCP
          port: 8080

  egress:
    # Allow to postgres
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
    # Allow to redis
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - protocol: TCP
          port: 6379
    # Allow DNS
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

---

## Configuration

### ConfigMap

```yaml
# configmap.yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-config
  namespace: production
data:
  # Key-value pairs
  SPRING_PROFILES_ACTIVE: "prod"
  SERVER_PORT: "8080"
  LOG_LEVEL: "INFO"
  DB_HOST: "postgres"
  DB_PORT: "5432"
  DB_NAME: "orderdb"

  # File content
  application.yml: |
    spring:
      datasource:
        url: jdbc:postgresql://postgres:5432/orderdb
        hikari:
          maximum-pool-size: 10
      jpa:
        hibernate:
          ddl-auto: validate

  log4j2.xml: |
    <?xml version="1.0" encoding="UTF-8"?>
    <Configuration status="WARN">
      ...
    </Configuration>
```

```bash
# Create from command line
kubectl create configmap app-config --from-literal=ENV=prod --from-literal=PORT=8080
kubectl create configmap app-config --from-file=application.yml
kubectl create configmap app-config --from-env-file=.env
```

---

### Secret

```yaml
# secret.yml — values must be base64 encoded
apiVersion: v1
kind: Secret
metadata:
  name: order-secrets
  namespace: production
type: Opaque
data:
  DB_PASSWORD: c2VjcmV0cGFzc3dvcmQ=    # echo -n 'secretpassword' | base64
  JWT_SECRET: bXlqd3RzZWNyZXQ=
  API_KEY: YXBpa2V5MTIz

# stringData — plain text (K8s encodes automatically)
stringData:
  DB_URL: "jdbc:postgresql://postgres:5432/orderdb"
```

```bash
# Create from command line (auto base64 encodes)
kubectl create secret generic order-secrets \
  --from-literal=DB_PASSWORD=secretpassword \
  --from-literal=JWT_SECRET=myjwtsecret

# TLS secret
kubectl create secret tls api-tls-cert \
  --cert=tls.crt \
  --key=tls.key

# Docker registry secret (for private registries)
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=password \
  --docker-email=user@example.com

# Decode a secret
kubectl get secret order-secrets -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
```

> ⚠️ **Secrets are only base64 encoded, NOT encrypted by default.** In production use:
> - **Sealed Secrets** (encrypt before committing to Git)
> - **External Secrets Operator** (sync from AWS Secrets Manager, Vault, GCP Secret Manager)
> - **Vault Agent Injector** (HashiCorp Vault sidecar)

---

### Environment Variables

```yaml
spec:
  containers:
    - name: order-service
      image: myrepo/order-service:1.0.0

      # Individual env vars
      env:
        - name: SPRING_PROFILES_ACTIVE
          value: "prod"

        # From ConfigMap key
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: order-config
              key: DB_HOST

        # From Secret key
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: order-secrets
              key: DB_PASSWORD

        # From Pod metadata (Downward API)
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: MEMORY_LIMIT
          valueFrom:
            resourceFieldRef:
              containerName: order-service
              resource: limits.memory

      # All keys from ConfigMap as env vars
      envFrom:
        - configMapRef:
            name: order-config
        - secretRef:
            name: order-secrets
          prefix: "SECRET_"         # optional prefix

      # Mount config file
      volumeMounts:
        - name: config-volume
          mountPath: /app/config    # mounts as files under this path

  volumes:
    - name: config-volume
      configMap:
        name: order-config
        items:
          - key: application.yml
            path: application.yml   # mount as /app/config/application.yml
```

---

## Storage

### Volumes

```yaml
spec:
  containers:
    - name: app
      volumeMounts:
        - name: shared-data
          mountPath: /app/data
        - name: host-logs
          mountPath: /var/log/app
        - name: config
          mountPath: /app/config
          readOnly: true
        - name: temp
          mountPath: /tmp/scratch

  volumes:
    # emptyDir — shared scratch space, deleted with pod
    - name: shared-data
      emptyDir: {}
    - name: temp
      emptyDir:
        medium: Memory              # tmpfs — in-memory
        sizeLimit: "128Mi"

    # hostPath — mount from node filesystem (avoid in prod)
    - name: host-logs
      hostPath:
        path: /var/log/myapp
        type: DirectoryOrCreate

    # ConfigMap as volume
    - name: config
      configMap:
        name: order-config

    # Secret as volume (mounted as files)
    - name: secrets
      secret:
        secretName: order-secrets
        defaultMode: 0400           # read-only by owner

    # Projected — combine multiple sources
    - name: combined
      projected:
        sources:
          - configMap:
              name: order-config
          - secret:
              name: order-secrets
```

---

### PersistentVolume & PersistentVolumeClaim

```yaml
# PersistentVolume — cluster-level storage resource (admin creates)
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-pv
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce                 # RWO: one node | RWX: many nodes | ROX: many nodes read-only
  persistentVolumeReclaimPolicy: Retain  # Retain | Recycle | Delete
  storageClassName: fast-ssd
  awsElasticBlockStore:             # provider-specific
    volumeID: vol-0abc123
    fsType: ext4

---
# PersistentVolumeClaim — pod's request for storage (developer creates)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-ssd        # matches StorageClass
  resources:
    requests:
      storage: 20Gi

---
# Use PVC in Pod
spec:
  volumes:
    - name: postgres-data
      persistentVolumeClaim:
        claimName: postgres-pvc
  containers:
    - name: postgres
      volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
```

**Access modes:**

| Mode | Short | Description |
|---|---|---|
| `ReadWriteOnce` | RWO | Read-write by a single node |
| `ReadOnlyMany` | ROX | Read-only by many nodes |
| `ReadWriteMany` | RWX | Read-write by many nodes |
| `ReadWriteOncePod` | RWOP | Read-write by a single pod (K8s 1.22+) |

---

### StorageClass

Dynamic provisioning — creates PVs automatically when PVCs are created.

```yaml
# storageclass.yml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: kubernetes.io/aws-ebs    # or ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  fsType: ext4
reclaimPolicy: Delete                 # Delete PV when PVC deleted
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer  # wait until pod is scheduled
```

```bash
kubectl get storageclass
kubectl get pv
kubectl get pvc -n production
kubectl describe pvc postgres-pvc -n production
```

---

## Resource Management

### Requests & Limits

```yaml
resources:
  requests:
    memory: "256Mi"      # guaranteed allocation; used for scheduling
    cpu: "250m"          # 250 millicores = 0.25 CPU
  limits:
    memory: "512Mi"      # hard limit — OOM killed if exceeded
    cpu: "1000m"         # throttled (not killed) if exceeded
```

**CPU units:** `1000m` = 1 CPU core = 1 vCPU. `500m` = 0.5 core.

**Memory units:** `256Mi` = 256 mebibytes. `1Gi` = 1 gibibyte.

**QoS Classes (determined by requests/limits):**

| Class | Condition | Eviction priority |
|---|---|---|
| `Guaranteed` | requests == limits for all containers | Last to be evicted |
| `Burstable` | requests < limits | Middle |
| `BestEffort` | No requests or limits set | First to be evicted |

```yaml
# Guaranteed QoS — set equal requests and limits
resources:
  requests:
    memory: "512Mi"
    cpu: "500m"
  limits:
    memory: "512Mi"    # same as request
    cpu: "500m"        # same as request
```

---

### LimitRange

Sets default and maximum resource limits per namespace.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:
        memory: "256Mi"
        cpu: "250m"
      defaultRequest:
        memory: "128Mi"
        cpu: "100m"
      max:
        memory: "2Gi"
        cpu: "2000m"
      min:
        memory: "64Mi"
        cpu: "50m"
    - type: Pod
      max:
        memory: "4Gi"
        cpu: "4000m"
    - type: PersistentVolumeClaim
      max:
        storage: "50Gi"
      min:
        storage: "1Gi"
```

---

### ResourceQuota

Limits total resource consumption per namespace.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    # Compute
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"

    # Object counts
    pods: "50"
    services: "20"
    persistentvolumeclaims: "20"
    secrets: "50"
    configmaps: "50"
    deployments.apps: "20"

    # Storage
    requests.storage: "500Gi"
    fast-ssd.storageclass.storage.k8s.io/requests.storage: "200Gi"
```

---

### Horizontal Pod Autoscaler

Automatically scales Deployment replicas based on metrics.

```yaml
# hpa.yml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 2
  maxReplicas: 20
  metrics:
    # CPU utilization
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70    # scale up when avg CPU > 70%

    # Memory utilization
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80

    # Custom metric (e.g., requests per second from Prometheus)
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"

  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60   # wait 60s before scaling up again
      policies:
        - type: Pods
          value: 4                      # max 4 pods added per period
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300  # wait 5 min before scaling down
      policies:
        - type: Percent
          value: 25                     # max 25% pods removed per period
          periodSeconds: 60
```

```bash
kubectl get hpa -n production
kubectl describe hpa order-service-hpa -n production
```

---

### Vertical Pod Autoscaler

Automatically adjusts resource requests/limits. Requires VPA addon.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: order-service-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  updatePolicy:
    updateMode: "Auto"              # Auto | Recreate | Initial | Off
  resourcePolicy:
    containerPolicies:
      - containerName: order-service
        minAllowed:
          memory: "128Mi"
          cpu: "100m"
        maxAllowed:
          memory: "2Gi"
          cpu: "2000m"
```

---

## Health Probes

```yaml
containers:
  - name: order-service
    image: myrepo/order-service:1.0.0

    # Startup probe — for slow-starting containers
    # Disables liveness/readiness until startup succeeds
    # Allows: failureThreshold × periodSeconds before giving up
    startupProbe:
      httpGet:
        path: /actuator/health
        port: 8080
      failureThreshold: 30          # 30 × 10s = 300s max startup time
      periodSeconds: 10

    # Liveness probe — is the container alive? Restart if fails.
    livenessProbe:
      httpGet:
        path: /actuator/health/liveness
        port: 8080
        httpHeaders:
          - name: Custom-Header
            value: probe
      initialDelaySeconds: 0        # with startupProbe, set to 0
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3           # restart after 3 consecutive failures
      successThreshold: 1

    # Readiness probe — is the container ready to receive traffic?
    # Removes pod from Service endpoints if fails (no restart)
    readinessProbe:
      httpGet:
        path: /actuator/health/readiness
        port: 8080
      initialDelaySeconds: 0
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 3
      successThreshold: 1

    # Probe types:
    # httpGet — HTTP GET request (200-399 = success)
    # tcpSocket — TCP connection check
    # exec — run command (exit 0 = success)
    # grpc — gRPC health check protocol

    # TCP probe example
    livenessProbe:
      tcpSocket:
        port: 5432
      initialDelaySeconds: 15
      periodSeconds: 20

    # Exec probe example
    livenessProbe:
      exec:
        command:
          - /bin/sh
          - -c
          - "pg_isready -U postgres"
      periodSeconds: 10
```

**Spring Boot Actuator endpoints:**
```yaml
# application.yml
management:
  endpoint:
    health:
      probes:
        enabled: true               # enables /actuator/health/liveness and /readiness
      show-details: always
  health:
    livenessState:
      enabled: true
    readinessState:
      enabled: true
```

**Probe decision guide:**

| Probe | Failure action | Use for |
|---|---|---|
| `startupProbe` | Kills and restarts pod | Slow-starting apps (JVM warmup) |
| `livenessProbe` | Kills and restarts pod | Deadlocked/stuck app detection |
| `readinessProbe` | Removes from Service endpoints | App not ready (DB connecting, warming up) |

---

## Scheduling

### Node Selector & Affinity

```yaml
spec:
  # Simple node selector (exact match)
  nodeSelector:
    disktype: ssd
    kubernetes.io/arch: amd64

  affinity:
    # Node affinity — which nodes pod can be scheduled on
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:   # hard requirement
        nodeSelectorTerms:
          - matchExpressions:
              - key: kubernetes.io/arch
                operator: In
                values: ["amd64"]
              - key: node.kubernetes.io/instance-type
                operator: In
                values: ["c5.xlarge", "c5.2xlarge"]

      preferredDuringSchedulingIgnoredDuringExecution:  # soft preference
        - weight: 80
          preference:
            matchExpressions:
              - key: disktype
                operator: In
                values: ["ssd"]

    # Pod affinity — schedule near other pods
    podAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                app: redis           # schedule near redis pods
            topologyKey: kubernetes.io/hostname

    # Pod anti-affinity — spread pods across nodes/zones
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: order-service
          topologyKey: kubernetes.io/hostname   # no two on same node
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                app: order-service
            topologyKey: topology.kubernetes.io/zone  # prefer different zones
```

---

### Taints & Tolerations

Taints repel pods from nodes. Tolerations allow pods to be scheduled on tainted nodes.

```bash
# Add taint to node
kubectl taint nodes node1 key=value:NoSchedule
kubectl taint nodes node1 dedicated=gpu:NoSchedule
kubectl taint nodes node1 disk=ssd:PreferNoSchedule

# Remove taint
kubectl taint nodes node1 dedicated=gpu:NoSchedule-

# Taint effects:
# NoSchedule       — don't schedule new pods (existing pods unaffected)
# PreferNoSchedule — prefer not to schedule (soft)
# NoExecute        — evict existing pods that don't tolerate it
```

```yaml
spec:
  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "gpu"
      effect: "NoSchedule"

    - key: "disk"
      operator: "Exists"           # tolerate any value for this key
      effect: "PreferNoSchedule"

    - operator: "Exists"           # tolerate ALL taints (use cautiously)

    # Tolerate node not-ready (useful for critical DaemonSets)
    - key: "node.kubernetes.io/not-ready"
      operator: "Exists"
      effect: "NoExecute"
      tolerationSeconds: 300
```

---

### Pod Disruption Budget

Limits voluntary disruptions during upgrades, node drains, etc.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: order-service-pdb
  namespace: production
spec:
  minAvailable: 2                   # at least 2 pods always available
  # OR
  # maxUnavailable: 1               # at most 1 pod unavailable at once
  selector:
    matchLabels:
      app: order-service
```

```bash
kubectl get pdb -n production
kubectl drain node1 --ignore-daemonsets --delete-emptydir-data  # respects PDB
```

---

## Security

### RBAC

Role-Based Access Control — who can do what on which resources.

```yaml
# Role — namespace-scoped permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
  - apiGroups: [""]                 # "" = core API group
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "update", "patch"]
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list"]

---
# RoleBinding — bind Role to user/group/serviceaccount
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: production
subjects:
  - kind: User
    name: mahabalaraju
    apiGroup: rbac.authorization.k8s.io
  - kind: Group
    name: developers
    apiGroup: rbac.authorization.k8s.io
  - kind: ServiceAccount
    name: order-sa
    namespace: production
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io

---
# ClusterRole — cluster-wide permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
  - apiGroups: [""]
    resources: ["nodes", "namespaces"]
    verbs: ["get", "list", "watch"]
  - nonResourceURLs: ["/healthz", "/metrics"]
    verbs: ["get"]

---
# ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-reader-binding
subjects:
  - kind: Group
    name: ops-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
# Check permissions
kubectl auth can-i create pods -n production
kubectl auth can-i create pods -n production --as=mahabalaraju
kubectl auth can-i '*' '*'                        # am I cluster-admin?

# List roles and bindings
kubectl get roles,rolebindings -n production
kubectl get clusterroles,clusterrolebindings
```

---

### ServiceAccount

Identity for pods when interacting with the Kubernetes API.

```yaml
# serviceaccount.yml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-sa
  namespace: production
  annotations:
    # AWS IRSA — bind to IAM role
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/order-service-role
automountServiceAccountToken: false   # disable auto-mount (security best practice)

---
# Grant permissions to ServiceAccount
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: order-sa-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: order-sa
    namespace: production
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io

---
# Use in pod
spec:
  serviceAccountName: order-sa
  automountServiceAccountToken: false  # don't mount token unless needed
```

---

### SecurityContext

```yaml
spec:
  # Pod-level security context
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000                   # volume ownership
    seccompProfile:
      type: RuntimeDefault          # enable default seccomp filter
    sysctls:
      - name: net.ipv4.tcp_tw_reuse
        value: "1"

  containers:
    - name: order-service
      # Container-level (overrides pod-level)
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        runAsNonRoot: true
        runAsUser: 1000
        capabilities:
          drop:
            - ALL                   # drop all capabilities
          add:
            - NET_BIND_SERVICE      # only add what's needed
```

---

## Namespaces

```yaml
# namespace.yml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production
    team: backend
```

```bash
# Create
kubectl create namespace staging

# List
kubectl get namespaces

# Set default namespace for context
kubectl config set-context --current --namespace=production

# Get all resources in namespace
kubectl get all -n production

# Delete namespace (deletes ALL resources in it)
kubectl delete namespace staging
```

**Common namespace strategy:**
```
default          — avoid using (no isolation)
production       — production workloads
staging          — staging environment
development      — dev environment
monitoring       — Prometheus, Grafana, alertmanager
ingress-nginx    — Ingress controller
cert-manager     — Certificate manager
kube-system      — System components (DNS, proxy)
```

---

## Helm

Helm is the package manager for Kubernetes. A **Chart** is a package of Kubernetes manifests with templating.

```bash
# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Add repos
helm repo add stable https://charts.helm.sh/stable
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Search
helm search repo postgres
helm search hub nginx

# Install
helm install my-postgres bitnami/postgresql \
  --namespace production \
  --create-namespace \
  --set auth.postgresPassword=secret \
  --set primary.persistence.size=20Gi

# Install with values file
helm install order-service ./order-chart \
  --namespace production \
  -f values.prod.yml

# Upgrade
helm upgrade order-service ./order-chart \
  --namespace production \
  -f values.prod.yml \
  --atomic \                        # rollback on failure
  --timeout 5m

# Upgrade or install
helm upgrade --install order-service ./order-chart -n production -f values.prod.yml

# Rollback
helm rollback order-service 1 -n production

# List releases
helm list -n production
helm list -A                        # all namespaces

# Status
helm status order-service -n production

# History
helm history order-service -n production

# Uninstall
helm uninstall order-service -n production

# Template (render without installing)
helm template order-service ./order-chart -f values.prod.yml

# Lint
helm lint ./order-chart

# Package
helm package ./order-chart
```

**Chart structure:**

```
order-chart/
├── Chart.yaml          # chart metadata
├── values.yaml         # default values
├── values.prod.yaml    # production overrides
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── hpa.yaml
│   ├── _helpers.tpl    # reusable template helpers
│   └── NOTES.txt       # post-install notes
└── charts/             # chart dependencies
```

```yaml
# values.yaml
replicaCount: 3
image:
  repository: myrepo/order-service
  tag: "1.0.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 8080

resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "1000m"

env:
  SPRING_PROFILES_ACTIVE: prod
  DB_HOST: postgres

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
```

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "order-chart.fullname" . }}
  labels:
    {{- include "order-chart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "order-chart.selectorLabels" . | nindent 6 }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          env:
            {{- range $key, $val := .Values.env }}
            - name: {{ $key }}
              value: {{ $val | quote }}
            {{- end }}
```

---

## Java Spring Boot on Kubernetes

```yaml
# Full production deployment for Spring Boot
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: order-service
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/path: "/actuator/prometheus"
        prometheus.io/port: "8080"
    spec:
      serviceAccountName: order-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000

      initContainers:
        - name: wait-for-db
          image: busybox:1.36
          command: ['sh', '-c',
            'until nc -z postgres 5432; do echo waiting for postgres; sleep 2; done']

      containers:
        - name: order-service
          image: myrepo/order-service:1.2.0
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
              name: http

          env:
            - name: JAVA_OPTS
              value: >-
                -XX:+UseContainerSupport
                -XX:MaxRAMPercentage=75.0
                -XX:InitialRAMPercentage=50.0
                -Djava.security.egd=file:/dev/./urandom
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace

          envFrom:
            - configMapRef:
                name: order-config
            - secretRef:
                name: order-secrets

          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "1000m"

          startupProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            failureThreshold: 30
            periodSeconds: 10

          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            periodSeconds: 10
            failureThreshold: 3

          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            periodSeconds: 5
            failureThreshold: 3

          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 10"]   # drain in-flight requests

          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]

          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: logs
              mountPath: /app/logs

      volumes:
        - name: tmp
          emptyDir: {}
        - name: logs
          emptyDir: {}

      terminationGracePeriodSeconds: 60

      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: order-service
                topologyKey: kubernetes.io/hostname
```

**Spring Boot `application.yml` for Kubernetes:**

```yaml
# application-kubernetes.yml
server:
  shutdown: graceful                  # drain requests on SIGTERM

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s   # wait up to 30s for in-flight requests

management:
  endpoint:
    health:
      probes:
        enabled: true
      show-details: always
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics
  metrics:
    export:
      prometheus:
        enabled: true

logging:
  structured:
    format:
      console: ecs                    # JSON logs (Spring Boot 3.4+)
```

---

## Observability

```yaml
# Prometheus ServiceMonitor (with Prometheus Operator)
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: order-service
  namespace: monitoring
  labels:
    release: prometheus               # matches Prometheus selector
spec:
  selector:
    matchLabels:
      app: order-service
  namespaceSelector:
    matchNames:
      - production
  endpoints:
    - port: http
      path: /actuator/prometheus
      interval: 15s

---
# PrometheusRule — alerting rules
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: order-service-alerts
  namespace: monitoring
spec:
  groups:
    - name: order-service
      rules:
        - alert: OrderServiceHighErrorRate
          expr: |
            rate(http_server_requests_seconds_count{app="order-service",status=~"5.."}[5m])
            / rate(http_server_requests_seconds_count{app="order-service"}[5m]) > 0.05
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "High error rate on order-service"
            description: "Error rate is {{ $value | humanizePercentage }}"
```

---

## Production Best Practices

```yaml
# Checklist in manifest form

# ✅ 1. Always set resource requests AND limits
resources:
  requests: { memory: "256Mi", cpu: "250m" }
  limits:   { memory: "512Mi", cpu: "1000m" }

# ✅ 2. All three probes for slow-starting apps
startupProbe: ...
livenessProbe: ...
readinessProbe: ...

# ✅ 3. Anti-affinity to spread across nodes
podAntiAffinity:
  preferredDuringSchedulingIgnoredDuringExecution:
    - topologyKey: kubernetes.io/hostname

# ✅ 4. PodDisruptionBudget for HA
minAvailable: 2  # or maxUnavailable: 1

# ✅ 5. Non-root user + drop capabilities
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  capabilities: { drop: ["ALL"] }

# ✅ 6. Read-only root filesystem + tmpfs for writable dirs
readOnlyRootFilesystem: true
volumes:
  - name: tmp
    emptyDir: {}

# ✅ 7. Graceful shutdown
terminationGracePeriodSeconds: 60
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 10"]

# ✅ 8. Pin image tags — never :latest in production
image: myrepo/order-service:1.2.0  # or SHA digest

# ✅ 9. RollingUpdate with maxUnavailable=0
rollingUpdate:
  maxSurge: 1
  maxUnavailable: 0

# ✅ 10. HPA for auto-scaling
minReplicas: 2
maxReplicas: 20

# ✅ 11. Dedicated ServiceAccount with minimal permissions
serviceAccountName: order-sa
automountServiceAccountToken: false

# ✅ 12. NetworkPolicy — deny all, allow only needed
```

---

## Debugging & Troubleshooting

```bash
# ── Pod won't start ─────────────────────────────────────
kubectl get pods -n production
kubectl describe pod my-pod -n production     # check Events section at bottom
kubectl logs my-pod -n production
kubectl logs --previous my-pod -n production  # previous crash logs

# Common Events to look for:
# "Insufficient memory/cpu"        → Node doesn't have capacity; check resources
# "ImagePullBackOff"               → Can't pull image; check image name, registry creds
# "CrashLoopBackOff"               → App keeps crashing; check logs --previous
# "OOMKilled"                      → Container exceeded memory limit; increase limits
# "Pending"                        → Scheduling failed; check node capacity, taints, affinity
# "ContainerCreating"              → Volume mount issue or image pull in progress

# ── Pending pod ─────────────────────────────────────────
kubectl describe pod my-pod | grep -A 10 Events
# "Insufficient cpu" → scale cluster or reduce requests
# "no nodes match taints" → check tolerations
# "unbound PVC" → check PVC/StorageClass

# ── Service not reachable ────────────────────────────────
# 1. Check service selector matches pod labels
kubectl get service order-service -o yaml | grep selector
kubectl get pods -l app=order-service         # should return pods

# 2. Check endpoints
kubectl get endpoints order-service
# Empty endpoints → label mismatch or no ready pods

# 3. Test from inside cluster
kubectl run test --rm -it --image=curlimages/curl -- sh
curl http://order-service.production.svc.cluster.local:8080/actuator/health

# 4. Check kube-proxy / DNS
kubectl exec -it my-pod -- nslookup order-service
kubectl exec -it my-pod -- curl http://order-service:8080

# ── Node issues ──────────────────────────────────────────
kubectl get nodes
kubectl describe node my-node              # check Conditions: MemoryPressure, DiskPressure
kubectl top nodes                          # resource usage
kubectl get events --field-selector involvedObject.kind=Node

# ── Resource usage ───────────────────────────────────────
kubectl top pods -n production
kubectl top pods --containers -n production
kubectl describe node | grep -A 10 "Allocated resources"

# ── Exec into pod for debugging ──────────────────────────
kubectl exec -it my-pod -- sh
kubectl exec -it my-pod -- curl localhost:8080/actuator/health
kubectl exec -it my-pod -- env | grep DB

# Debug ephemeral container (K8s 1.23+ — attach debug container to running pod)
kubectl debug -it my-pod --image=busybox --target=order-service

# ── ConfigMap / Secret issues ────────────────────────────
kubectl exec -it my-pod -- env | grep MY_VAR
kubectl exec -it my-pod -- cat /app/config/application.yml

# ── RBAC debugging ──────────────────────────────────────
kubectl auth can-i list pods -n production --as=system:serviceaccount:production:order-sa
kubectl describe clusterrolebinding cluster-admin

# ── Common exit codes ────────────────────────────────────
# Exit 0   → clean exit
# Exit 1   → application error
# Exit 137 → OOM killed (exceeded memory limit)
# Exit 143 → SIGTERM (graceful shutdown — normal)
# Exit 255 → container start failure
```

---

## Quick Reference Cheat Sheet

### Object selection guide

```
Stateless app, multiple replicas   → Deployment
Stateful app (DB, Kafka)           → StatefulSet
One pod per node (agents)          → DaemonSet
Run to completion (batch)          → Job
Scheduled batch                    → CronJob
Stable network endpoint            → Service (ClusterIP)
External HTTP routing              → Ingress
Non-sensitive config               → ConfigMap
Sensitive config                   → Secret
Persistent storage request         → PersistentVolumeClaim
Dynamic storage provisioning       → StorageClass
```

### Service type selection

```
Internal only                      → ClusterIP
Dev/test external                  → NodePort
Production external (cloud)        → LoadBalancer
External DNS mapping               → ExternalName
StatefulSet per-pod DNS            → Headless (clusterIP: None)
```

### Probe selection

```
Slow JVM startup (60-300s)         → startupProbe (buys time for liveness)
App deadlock detection             → livenessProbe (restart on failure)
App ready for traffic              → readinessProbe (remove from LB on failure)
```

### Resource QoS selection

```
Critical system pods               → Guaranteed (requests == limits)
Normal workloads                   → Burstable (requests < limits)
Non-critical / dev                 → BestEffort (no requests/limits)
```

### kubectl quick ref

```bash
apply -f file.yml              create or update
get pods -A                    all namespaces
describe pod name              events + details
logs -f pod --previous         live / crash logs
exec -it pod -- bash           shell
port-forward pod 8080:8080     local debug
rollout undo deployment name   rollback
scale deployment name --replicas=5
top pods                       CPU + memory
auth can-i verb resource       RBAC check
```

---

*References: Kubernetes Docs (kubernetes.io/docs) | Kubernetes Patterns — Ibryam & Huß | Production Kubernetes — Rosso, Lander, Brand, Harris*
