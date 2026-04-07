# CKA Networking Cheat Sheet

> **Domain weight: 20%.** Covers Services, DNS, NetworkPolicy, Ingress, Gateway API.

## Fast Setup

- Set namespace:
  - `kubectl config set-context --current --namespace=<ns>`
- Check current namespace:
  - `kubectl config view --minify | grep namespace:`
- Prefer explicit resources over `kubectl get all`

## Core Checks

- `kubectl get deploy,po,svc,endpoints -n <ns>`
- `kubectl describe svc <name> -n <ns>`
- `kubectl get pod -n <ns> --show-labels`
- `kubectl get svc,endpoints,endpointslices -A`

## DNS

- Resolve short name:
  - `kubectl exec <pod> -n <ns> -- nslookup <svc>`
- Resolve FQDN:
  - `kubectl exec <pod> -n <ns> -- nslookup <svc>.<ns>.svc.cluster.local`
- Check cluster DNS:
  - `kubectl get svc -n kube-system -l k8s-app=kube-dns`
  - `kubectl get po -n kube-system -l k8s-app=kube-dns`

BusyBox note:

- BusyBox `nslookup` may print the correct answer and still show extra `NXDOMAIN` lines.

## HTTP and TCP Tests

- HTTP:
  - `kubectl exec <pod> -n <ns> -- wget -qO- http://<svc>:<port>`
- HTTP with timeout:
  - `kubectl exec <pod> -n <ns> -- wget -qO- --timeout=3 http://<svc>:<port>`
- TCP with timeout:
  - `kubectl exec <pod> -n <ns> -- nc -vz -w 3 <svc> <port>`

Failure hints:

- `connection refused`: wrong `targetPort` or app not listening
- timeout: `NetworkPolicy`, routing, or unreachable backend

## NetworkPolicy

- List:
  - `kubectl get netpol -n <ns>`
- Describe:
  - `kubectl describe netpol <name> -n <ns>`
- Check source labels:
  - `kubectl get pod -n <ns> --show-labels`

Logic:

- `from` and `ports` in one rule are `AND`
- multiple `from` entries are `OR`
- multiple `ports` entries are `OR`
- multiple ingress rules are `OR`

## Ingress

- List:
  - `kubectl get ing,ingressclass -A`
- Describe:
  - `kubectl describe ing <name> -n <ns>`
- Controller health:
  - `kubectl get svc -n ingress-nginx`
  - `kubectl get po -n ingress-nginx`
  - `kubectl logs -n ingress-nginx deploy/ingress-nginx-controller`
- Route test:
  - `curl -H 'Host: <host>' http://127.0.0.1:<nodeport>`
  - `wget -qO- --header='Host: <host>' http://127.0.0.1:<nodeport>`

## Gateway API

- List:
  - `kubectl get gatewayclass`
  - `kubectl get gateway,httproute -n <ns>`
- Describe:
  - `kubectl describe gateway <name> -n <ns>`
  - `kubectl describe httproute <name> -n <ns>`
- Route test:
  - `curl -H 'Host: <host>' http://127.0.0.1:<nodeport>`

Good signs:

- `GatewayClass` `Accepted=True`
- `Gateway` `Accepted=True`
- `Gateway` `Programmed=True`
- `Attached Routes` is non-zero
- `HTTPRoute` `Accepted=True`
- `HTTPRoute` `ResolvedRefs=True`

## Troubleshooting Order

1. `kubectl get deploy,po,svc,endpoints -n <ns>`
2. `kubectl describe svc <name> -n <ns>`
3. `kubectl get pod -n <ns> --show-labels`
4. `kubectl exec <pod> -n <ns> -- nslookup <svc>`
5. `kubectl exec <pod> -n <ns> -- wget -qO- --timeout=3 http://<svc>:<port>`
6. `kubectl exec <pod> -n <ns> -- nc -vz -w 3 <svc> <port>`
7. `kubectl describe netpol <name> -n <ns>`
8. `kubectl describe ing <name> -n <ns>`
9. `kubectl describe gateway <name> -n <ns>`
10. `kubectl describe httproute <name> -n <ns>`

## Exam Habits

- Use one failing command, a few inspection commands, one minimal fix, then one success command.
- Test from inside the cluster before outside when possible.
- Use `get` plus `describe`.
- Switch namespace at the start of every task.

---

## Service Types — YAML Reference

### ClusterIP (default — internal only)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-svc
  namespace: default
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
  - port: 80           # port clients connect to
    targetPort: 8080   # port the container listens on
    protocol: TCP
```

### NodePort (expose on every node)
```yaml
spec:
  type: NodePort
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080    # 30000–32767; auto-assigned if omitted
```

Access via: `http://<any-node-ip>:30080`

### LoadBalancer (cloud provider LB)
```yaml
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
```

### Headless (no load balancing — DNS returns pod IPs directly)
```yaml
spec:
  clusterIP: None      # This makes it headless
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
```

Used with StatefulSets. DNS returns individual pod IPs, not a single ClusterIP.

### ExternalName (CNAME redirect to external service)
```yaml
spec:
  type: ExternalName
  externalName: db.example.com   # must be an FQDN
```

> **Gotcha:** ExternalName returns a CNAME, not an A record. No `selector` or `ports` needed.

> **Gotcha:** NodePort and LoadBalancer **include** a ClusterIP. You can always test them from inside the cluster using the ClusterIP or DNS name.

---

## NetworkPolicy — YAML Reference

### Default Deny All Ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}       # applies to ALL pods in namespace
  policyTypes:
  - Ingress
  # No ingress: rules = deny all ingress
```

### Allow Ingress by Pod Label
```yaml
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

### Allow Ingress from Namespace
```yaml
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: monitoring
    ports:
    - protocol: TCP
      port: 8080
```

> **Gotcha:** `namespaceSelector` matches on namespace **labels**, not names. Use the auto-assigned label `kubernetes.io/metadata.name: <ns-name>` (available since k8s 1.21).

### Egress — Allow Specific Ports + DNS
```yaml
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 8080
  - ports:              # Allow DNS (always needed for name resolution)
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

> **Gotcha:** If you add an Egress policy, pods can no longer resolve DNS unless you explicitly allow UDP/TCP port 53.

### Combined Ingress + Egress
```yaml
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - port: 5432
  egress:
  - ports:
    - protocol: UDP
      port: 53
```

### NetworkPolicy Logic Rules
```
Within a rule (from/to + ports)         = AND  (must satisfy both)
Multiple from/to entries in one rule    = OR   (any match allows traffic)
Multiple ingress rules                  = OR   (any matching rule allows)
podSelector: {}                         = ALL pods in namespace
namespaceSelector: {}                   = ALL namespaces
ingress: []  or  no ingress: key        = DENY ALL ingress
```

---

## Ingress — YAML Reference

### Basic Ingress with Path Routing
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  namespace: default
spec:
  ingressClassName: nginx    # must match installed controller
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix     # matches /api, /api/users, /api/anything
        backend:
          service:
            name: api-svc
            port:
              number: 80
      - path: /api/v1/status
        pathType: Exact      # matches ONLY /api/v1/status
        backend:
          service:
            name: status-svc
            port:
              number: 8080
```

### Ingress with TLS
```bash
# Step 1: Create TLS secret (must exist before Ingress is applied)
kubectl create secret tls tls-secret \
  --cert=tls.crt \
  --key=tls.key \
  -n <ns>

# Or generate a self-signed cert in one step:
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=app.example.com/O=MyOrg"
kubectl create secret tls tls-secret --cert=tls.crt --key=tls.key
```

```yaml
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    secretName: tls-secret     # must match the secret name above
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-svc
            port:
              number: 80
```

> **Gotcha:** `pathType: Prefix` matches path elements (`/api` matches `/api/users`). `pathType: Exact` matches the exact string only.

> **Gotcha:** TLS secret must be of type `kubernetes.io/tls` and must be in the **same namespace** as the Ingress.

> **Gotcha:** Without `ingressClassName`, the Ingress may not be picked up by any controller.

### Test Ingress (from inside cluster)
```bash
curl -H 'Host: app.example.com' http://<node-ip>:<nodeport>
wget -qO- --header='Host: app.example.com' http://127.0.0.1:<nodeport>
```

---

## Gateway API — YAML Reference

The newer alternative to Ingress (added to CKA 2025+).

### GatewayClass (defines the controller — cluster-scoped)
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx
spec:
  controllerName: nginx.org/nginx-gateway-controller
```

### Gateway (listens for traffic — creates the entry point)
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: api-gateway
  namespace: ingress-system
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: All     # accept routes from any namespace
```

### HTTPRoute (routes requests to services)
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: api-route
  namespace: app
spec:
  parentRefs:
  - name: api-gateway
    namespace: ingress-system   # reference the Gateway
  hostnames:
  - api.example.com
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /v1
    backendRefs:
    - name: api-v1-svc
      port: 8080
  - matches:
    - path:
        type: PathPrefix
        value: /v2
    backendRefs:
    - name: api-v2-svc
      port: 8080
```

### Status Conditions to Verify
```bash
kubectl describe gateway api-gateway -n ingress-system
# Look for:
#   Accepted:    True
#   Programmed:  True
#   Attached Routes: N  (should be non-zero)

kubectl describe httproute api-route -n app
# Look for:
#   Accepted:     True
#   ResolvedRefs: True
```

> **Gotcha:** If `ResolvedRefs: False`, the backend Service doesn't exist or the port is wrong.

> **Gotcha:** `allowedRoutes.namespaces.from: Same` (default) — HTTPRoute must be in the same namespace as Gateway. Use `All` to allow cross-namespace routes.

---

## CoreDNS Corefile Reference

```bash
# View the current Corefile
kubectl get cm coredns -n kube-system -o yaml

# Edit Corefile
kubectl edit cm coredns -n kube-system

# CRITICAL: must restart CoreDNS after any ConfigMap change
kubectl rollout restart deployment coredns -n kube-system
```

### Corefile Plugin Reference
```
.:53 {
    errors                          # log errors to stdout
    health { lameduck 5s }          # health check on :8080/health; 5s graceful shutdown
    ready                           # readiness on :8181/ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure               # resolve pod IP records
        fallthrough in-addr.arpa ip6.arpa
        ttl 30
    }
    prometheus :9153                # metrics endpoint
    forward . /etc/resolv.conf      # forward non-cluster queries to node's DNS
    cache 30                        # cache responses for 30s
    loop                            # detect + halt on forwarding loops
    reload                          # auto-reload Corefile on change
    loadbalance                     # round-robin DNS responses
}
```

| Plugin | What it does |
|--------|---|
| `forward . /etc/resolv.conf` | Forward external DNS queries to node resolver |
| `forward . 8.8.8.8` | Forward external DNS to Google DNS |
| `kubernetes` | Resolve in-cluster service/pod names |
| `cache 30` | Cache responses 30 seconds |
| `loop` | Detect forwarding loops and crash CoreDNS to prevent them |
| `health` | Liveness endpoint at `:8080/health` |
| `ready` | Readiness endpoint at `:8181/ready` |

> **Gotcha:** After editing the CoreDNS ConfigMap, you **must** restart the CoreDNS deployment. Without restart, the old config stays loaded.

> **Gotcha:** `forward` must come **after** the `kubernetes` block. Order matters in the Corefile.

---

## Port-Forward (Debugging Tool)

```bash
# Forward local port to pod port
kubectl port-forward pod/<name> 8080:8080 -n <ns>

# Forward to deployment (picks a pod)
kubectl port-forward deployment/<name> 8080:8080

# Forward to service
kubectl port-forward service/<name> 8080:80

# Listen on all interfaces (default is 127.0.0.1 only)
kubectl port-forward --address 0.0.0.0 pod/<name> 8080:8080
```

> **Gotcha:** `port-forward` creates a direct tunnel bypassing NetworkPolicy. It is useful for testing pod connectivity but **does not test service routing or NetworkPolicy**.

---

## Networking Exam Gotchas

| Gotcha | Correct Approach |
|--------|---|
| Selector mismatch → endpoints `<none>` | Compare `svc.spec.selector` vs `pod.metadata.labels` exactly |
| Port mismatch → connection refused | `svc.spec.ports.targetPort` must match container's actual listen port |
| NetworkPolicy with egress but no DNS | Always add egress rule for UDP/TCP port 53 |
| `namespaceSelector` by name | Use label `kubernetes.io/metadata.name: <ns>` not the name string |
| Ingress not routing | Check `ingressClassName`, controller pod health, and TLS secret exists |
| Gateway `ResolvedRefs: False` | Backend Service doesn't exist or wrong port |
| CoreDNS ConfigMap edited but DNS still broken | Must restart CoreDNS: `kubectl rollout restart deploy/coredns -n kube-system` |
| TLS secret wrong type | Secret must be type `kubernetes.io/tls`, created with `kubectl create secret tls` |
| ExternalName and ports | ExternalName services do CNAME redirect; no selector, no pod |
| Cross-namespace HTTPRoute fails | Check Gateway `allowedRoutes.namespaces.from: All` |
