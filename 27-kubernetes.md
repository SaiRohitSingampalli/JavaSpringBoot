# 27 — Kubernetes for Spring Boot Microservices

## 1. What Kubernetes does
Container **orchestrator**: you declare desired state (YAML), K8s continuously reconciles actual state to match.
- Scheduling containers onto nodes
- Self-healing (restart crashed containers, replace dead pods/nodes)
- Horizontal scaling (manual & autoscaling)
- Service discovery & load balancing
- Rolling updates & rollbacks
- Config & secret management
- Storage orchestration
- Resource isolation (requests/limits)

## 2. Architecture
```
               ┌─────────────── Control Plane ───────────────┐
 kubectl ────► │ kube-apiserver ◄─► etcd (cluster state)      │
               │ kube-scheduler (assign pods to nodes)       │
               │ kube-controller-manager (Deployment, RS,    │
               │   Node, Job controllers... reconcile loops) │
               │ cloud-controller-manager (LBs, volumes)     │
               └─────────────────────────────────────────────┘
                       │
       ┌───────────────┼────────────────┐
   Worker Node 1   Worker Node 2   Worker Node 3
   ├ kubelet (runs pods, reports status, probes)
   ├ kube-proxy (Service networking: iptables/IPVS) or eBPF CNI (Cilium)
   ├ container runtime (containerd / CRI-O)
   └ Pods
```
Managed offerings: **EKS** (AWS), **GKE** (Google), **AKS** (Azure), OpenShift. Local: Minikube, kind, k3d, Docker Desktop, Rancher Desktop.

## 3. Core objects
| Object | Purpose |
|--------|---------|
| **Pod** | Smallest deployable unit: one or more containers sharing network namespace (same IP, localhost) and volumes. Ephemeral — never create directly in production |
| **ReplicaSet** | Ensures N identical pods run (selector + template) |
| **Deployment** | Manages ReplicaSets; declarative updates, rolling update, rollback history — **use for stateless services** |
| **StatefulSet** | Stable network identity (`pod-0`, `pod-1`), ordered startup, per-pod persistent volumes — databases, Kafka, ZooKeeper |
| **DaemonSet** | One pod per node — log shippers, monitoring agents, CNI |
| **Job / CronJob** | Run-to-completion tasks / scheduled tasks (replaces app-level cron in many cases) |
| **Service** | Stable virtual IP + DNS name load-balancing across pods selected by labels |
| **Ingress** / **Gateway API** | HTTP(S) routing from outside to Services (host/path rules, TLS) via an ingress controller (NGINX, Traefik, cloud LB) |
| **ConfigMap** | Non-secret configuration |
| **Secret** | Sensitive data (base64-encoded — not encrypted unless etcd encryption/external secrets) |
| **PersistentVolume (PV)** / **PersistentVolumeClaim (PVC)** / **StorageClass** | Storage abstraction & dynamic provisioning |
| **Namespace** | Logical isolation (dev/qa/prod, team), resource quotas |
| **HorizontalPodAutoscaler** | Scale replicas by CPU/memory/custom metrics |
| **ServiceAccount**, **Role/ClusterRole**, **RoleBinding** | RBAC identities & permissions |
| **NetworkPolicy** | Firewall rules between pods |
| **PodDisruptionBudget** | Min available pods during voluntary disruptions (node drains) |

**Labels & selectors** connect everything: Deployment's pod template labels `app: orders` ← Service selector `app: orders`.

## 4. kubectl essentials
```bash
kubectl config get-contexts / use-context dev-cluster
kubectl get nodes
kubectl get pods -n shop -o wide
kubectl get deploy,rs,svc,ing,cm,secret,hpa -n shop
kubectl describe pod orders-7d9f-abc -n shop        # events: scheduling, image pull, probe failures
kubectl logs -f orders-7d9f-abc -n shop [-c container] [--previous]
kubectl exec -it orders-7d9f-abc -- sh
kubectl port-forward svc/orders 8080:8080           # local access
kubectl apply -f k8s/                               # declarative create/update
kubectl delete -f k8s/deployment.yaml
kubectl scale deploy/orders --replicas=5
kubectl set image deploy/orders orders=acme/orders:1.1.0
kubectl rollout status deploy/orders
kubectl rollout history deploy/orders
kubectl rollout undo deploy/orders [--to-revision=2]
kubectl rollout restart deploy/orders               # restart pods (e.g., after config change)
kubectl top pods                                    # needs metrics-server
kubectl get events --sort-by=.lastTimestamp
kubectl create deployment orders --image=acme/orders:1.0.0 --dry-run=client -o yaml > deployment.yaml
kubectl explain deployment.spec.strategy
```
Imperative commands (`create`, `expose`) are good for learning; **declarative YAML in Git** (`apply`, Helm, Kustomize, GitOps) for real environments.

## 5. Full manifest for a Spring Boot service
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: orders-config
  namespace: shop
data:
  SPRING_PROFILES_ACTIVE: "prod"
  INVENTORY_URL: "http://inventory:8080"
  application.yml: |                       # can also mount whole files
    app:
      page-size: 50
---
apiVersion: v1
kind: Secret
metadata:
  name: orders-secrets
  namespace: shop
type: Opaque
stringData:                                # plain text here; stored base64
  SPRING_DATASOURCE_PASSWORD: "change-me"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders
  namespace: shop
  labels: { app: orders }
spec:
  replicas: 3
  revisionHistoryLimit: 5
  selector:
    matchLabels: { app: orders }
  strategy:
    type: RollingUpdate
    rollingUpdate: { maxSurge: 1, maxUnavailable: 0 }      # zero-downtime
  template:
    metadata:
      labels: { app: orders, version: "1.0.0" }
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/path: /actuator/prometheus
        prometheus.io/port: "8080"
    spec:
      serviceAccountName: orders
      terminationGracePeriodSeconds: 45
      securityContext: { runAsNonRoot: true, runAsUser: 1001 }
      containers:
        - name: orders
          image: registry.example.com/acme/orders:1.0.0     # immutable tag
          imagePullPolicy: IfNotPresent
          ports: [{ name: http, containerPort: 8080 }]
          envFrom:
            - configMapRef: { name: orders-config }
            - secretRef: { name: orders-secrets }
          env:
            - name: SPRING_DATASOURCE_URL
              value: jdbc:postgresql://postgres:5432/orders
            - name: JAVA_TOOL_OPTIONS
              value: "-XX:MaxRAMPercentage=75"
            - name: POD_NAME
              valueFrom: { fieldRef: { fieldPath: metadata.name } }
          resources:
            requests: { cpu: "500m", memory: "768Mi" }      # used for scheduling
            limits:   { memory: "1Gi" }                     # memory limit; CPU limit often omitted to avoid throttling
          startupProbe:                                      # protects slow-starting JVMs
            httpGet: { path: /actuator/health/liveness, port: http }
            periodSeconds: 5
            failureThreshold: 30                             # up to 150s to start
          livenessProbe:
            httpGet: { path: /actuator/health/liveness, port: http }
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet: { path: /actuator/health/readiness, port: http }
            periodSeconds: 5
            failureThreshold: 3
          lifecycle:
            preStop: { exec: { command: ["sh", "-c", "sleep 10"] } }   # let endpoints/LB deregister before shutdown
          volumeMounts:
            - { name: config, mountPath: /config }
      volumes:
        - name: config
          configMap: { name: orders-config, items: [{ key: application.yml, path: application.yml }] }
---
apiVersion: v1
kind: Service
metadata:
  name: orders
  namespace: shop
spec:
  type: ClusterIP
  selector: { app: orders }
  ports: [{ name: http, port: 8080, targetPort: http }]
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shop
  namespace: shop
  annotations: { cert-manager.io/cluster-issuer: letsencrypt }
spec:
  ingressClassName: nginx
  tls: [{ hosts: [api.shop.com], secretName: api-shop-tls }]
  rules:
    - host: api.shop.com
      http:
        paths:
          - { path: /api/orders, pathType: Prefix, backend: { service: { name: orders, port: { number: 8080 } } } }
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: orders
  namespace: shop
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: orders }
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 70 } }
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: orders, namespace: shop }
spec:
  minAvailable: 2
  selector: { matchLabels: { app: orders } }
```
Spring Boot reads `/config/application.yml` automatically (`./config/` relative to working dir — set `WORKDIR` accordingly, or use `spring.config.additional-location`).

## 6. Services in detail
| Type | Reachable from | Use |
|------|---------------|-----|
| **ClusterIP** (default) | Inside cluster | Service-to-service |
| **NodePort** | `<NodeIP>:30000-32767` | Testing, bare metal |
| **LoadBalancer** | Cloud external LB | Expose a service directly |
| **ExternalName** | DNS CNAME to external host | Abstract external DBs |
| Headless (`clusterIP: None`) | DNS returns pod IPs | StatefulSets, client-side LB |

DNS: `orders` (same namespace), `orders.shop`, `orders.shop.svc.cluster.local`. Services load-balance at L4 (connection level) — HTTP/2/gRPC long-lived connections may need L7 balancing (service mesh/client-side).

## 7. Probes & Spring Boot
| Probe | Failure action | Spring endpoint |
|-------|---------------|-----------------|
| **startupProbe** | Restart if never starts; disables other probes until success | `/actuator/health/liveness` |
| **livenessProbe** | **Restart** container | `/actuator/health/liveness` (`LivenessState`) |
| **readinessProbe** | Remove from Service endpoints (no traffic) | `/actuator/health/readiness` (`ReadinessState` + chosen indicators) |
Boot auto-enables probe groups when it detects Kubernetes (`management.endpoint.health.probes.enabled=true` otherwise). During graceful shutdown readiness flips to REFUSING_TRAFFIC. Keep external dependencies out of liveness.

### Graceful shutdown sequence
1. Pod marked Terminating → removed from Endpoints (asynchronously) and `preStop` hook runs (sleep ~5–10s so load balancers stop sending traffic).
2. SIGTERM → Spring `server.shutdown=graceful` stops accepting new requests, finishes in-flight ones (up to `spring.lifecycle.timeout-per-shutdown-phase`).
3. After `terminationGracePeriodSeconds`, SIGKILL.

## 8. Resources, QoS & JVM
- **requests**: guaranteed share, used by the scheduler. **limits**: hard cap (memory over limit → OOMKilled; CPU over limit → throttled).
- QoS classes: Guaranteed (requests = limits), Burstable, BestEffort.
- JVM: set `MaxRAMPercentage`; memory limit = heap + metaspace + threads + direct buffers + overhead. CPU limits can starve JIT/GC during startup — many teams set CPU requests but no CPU limit.
- Startup time matters for autoscaling and rollouts → CDS/AOT, lazy init, native images.

## 9. Configuration & secrets
- ConfigMaps → env vars (`envFrom`) or mounted files (`configtree:` import for one-file-per-key mounts: `spring.config.import=optional:configtree:/etc/config/`).
- Changing a ConfigMap doesn't restart pods: mounted files update eventually, env vars don't → `kubectl rollout restart` or tools (Reloader, Spring Cloud Kubernetes config watcher).
- Secrets: enable etcd encryption; use **External Secrets Operator** / **Sealed Secrets** / CSI secrets store with Vault, AWS Secrets Manager, GCP Secret Manager; RBAC to restrict reads.

## 10. Deployment strategies
| Strategy | How | Notes |
|----------|-----|-------|
| **RollingUpdate** (default) | Replace pods gradually (`maxSurge`, `maxUnavailable`) | Both versions run simultaneously → API & DB must be backward compatible |
| **Recreate** | Kill all, then start new | Downtime; for incompatible versions |
| **Blue/Green** | Two full environments; switch Service selector/Ingress | Instant rollback; double resources |
| **Canary** | Small % of traffic to new version, increase while monitoring | Argo Rollouts, Flagger, Istio, Gateway API weights |
| **Feature flags** | Deploy dark, release by toggle | Decouple deploy from release |

## 11. Packaging & delivery
- **Helm**: templated charts (`values.yaml` per environment), releases, rollback (`helm upgrade --install orders ./chart -f values-prod.yaml`).
- **Kustomize**: base + overlays (dev/prod) without templates (`kubectl apply -k overlays/prod`).
- **GitOps**: Argo CD / Flux sync cluster state from Git — PR-based deployments, drift detection, easy rollback.
- CI builds & scans image → pushes to registry → updates image tag in Git → Argo CD deploys.

## 12. Observability & operations on K8s
- Metrics: Prometheus Operator (ServiceMonitor scraping `/actuator/prometheus`), kube-state-metrics, Grafana.
- Logs: stdout → Fluent Bit/Promtail/OTel Collector → Loki/Elasticsearch/cloud logging.
- Traces: OTel Collector → Tempo/Jaeger.
- Debugging pod states:

| Status | Likely cause |
|--------|--------------|
| `Pending` | Not enough resources / unschedulable / PVC not bound |
| `ImagePullBackOff` / `ErrImagePull` | Wrong image/tag, registry auth (imagePullSecrets) |
| `CrashLoopBackOff` | App exits repeatedly → `kubectl logs --previous` (bad config, can't reach DB, OOM) |
| `OOMKilled` | Memory limit too low / heap % too high |
| `Running` but `0/1 Ready` | Readiness probe failing |
| `Terminating` stuck | Finalizers, long grace period |

## 13. Service mesh (overview)
Istio / Linkerd inject sidecar proxies (Envoy) or use ambient/eBPF data planes → mTLS between services, retries, timeouts, circuit breaking (outlier detection), traffic splitting, telemetry — without app code. Trade-off: operational complexity and some latency. Avoid duplicating retries in both app and mesh.

## 14. Spring Boot on Kubernetes checklist
- [ ] Container image: non-root, JRE base, layered, immutable tag
- [ ] Probes: startup, liveness, readiness (Actuator)
- [ ] `server.shutdown=graceful` + preStop + terminationGracePeriod
- [ ] Resource requests/limits + `MaxRAMPercentage`
- [ ] Config via ConfigMaps, secrets via external secret store
- [ ] ≥2 replicas + PodDisruptionBudget + anti-affinity/topology spread
- [ ] HPA on CPU or custom metrics (RPS, Kafka lag via KEDA)
- [ ] Metrics/logs/traces wired
- [ ] NetworkPolicies, RBAC least privilege, ServiceAccount per app
- [ ] DB migrations run once (Flyway on startup with locks, or a Job/init container)
- [ ] Backward compatible deploys (rolling updates run old + new together)

## 15. Interview Q&A
**Q. Pod vs Deployment vs Service?** Pod runs containers; Deployment manages replicated pods and updates; Service provides a stable endpoint and load-balancing to pods.

**Q. Deployment vs StatefulSet?** Deployment for stateless interchangeable pods; StatefulSet for stable identities, ordered operations and per-pod storage.

**Q. How does service discovery work in K8s?** Services get DNS names via CoreDNS; kube-proxy routes the ClusterIP to healthy pod endpoints.

**Q. Liveness vs readiness vs startup probes?** Restart unhealthy containers; gate traffic; give slow apps time to start before liveness kicks in.

**Q. How does a rolling update achieve zero downtime?** New pods start and become ready before old ones are removed (maxUnavailable 0), readiness gates traffic, graceful shutdown drains old pods.

**Q. How do you roll back?** `kubectl rollout undo`, Helm rollback, or revert the Git commit with GitOps.

**Q. ConfigMap vs Secret?** Same mechanics; Secrets are meant for sensitive data (base64, RBAC-restricted, optionally encrypted at rest) — use external secret managers for real security.

**Q. requests vs limits?** Requests reserve capacity for scheduling; limits cap usage (memory → OOMKill, CPU → throttling).

**Q. What is HPA?** Horizontal Pod Autoscaler adjusting replica count based on observed metrics.

**Q. What does CrashLoopBackOff mean and how do you debug it?** Container keeps crashing; check `logs --previous`, `describe` events, config/secrets, resource limits, dependencies.
