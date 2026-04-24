# CKA Notes: Networking

## Cluster Reality

- This lab cluster is kubeadm-based with Calico, so `NetworkPolicy` is enforced.
- CoreDNS is exposed by the `kube-dns` Service in `kube-system`.
- `ingress-nginx` is installed as a `NodePort` Service.
- Gateway API CRDs are installed.
- NGINX Gateway Fabric is installed and provides `GatewayClass` `nginx`.

## Fast Commands

- Services, endpoints, and endpoint slices:
  - `kubectl get svc,endpoints,endpointslices -A`
- Describe a Service:
  - `kubectl describe svc <name> -n <ns>`
- Check Pod labels:
  - `kubectl get pod -n <ns> --show-labels`
- Check Service selectors:
  - `kubectl get svc <name> -n <ns> -o yaml`
- DNS test from a Pod:
  - `kubectl exec <pod> -n <ns> -- nslookup <svc>`
- HTTP test from a Pod:
  - `kubectl exec <pod> -n <ns> -- wget -qO- http://<svc>:<port>`
- HTTP test with timeout:
  - `kubectl exec <pod> -n <ns> -- wget -qO- --timeout=3 http://<svc>:<port>`
- TCP test with timeout:
  - `kubectl exec <pod> -n <ns> -- nc -vz -w 3 <svc> <port>`
- Ingress and class:
  - `kubectl get ing,ingressclass -A`
- Describe an Ingress:
  - `kubectl describe ing <name> -n <ns>`
- Gateway API objects:
  - `kubectl get gatewayclass,gateway,httproute -A`
- Describe a Gateway:
  - `kubectl describe gateway <name> -n <ns>`
- Describe an HTTPRoute:
  - `kubectl describe httproute <name> -n <ns>`
- Network policies:
  - `kubectl get netpol -A`

## Services

- A Service gives a stable virtual IP and DNS name to a set of Pods.
- Service selectors match Pod labels and create endpoints.
- If the Service exists but has no endpoints, traffic will fail.

Important fields:

- `port`: port exposed by the Service
- `targetPort`: port on the Pod the Service forwards to
- `selector`: labels used to find backend Pods
- `type`: `ClusterIP`, `NodePort`, or `LoadBalancer`

Common mistakes:

- wrong `selector`
- wrong `targetPort`
- Pods not ready, so endpoints are empty
- app listens on a different port than expected

## DNS

- CoreDNS resolves Service names inside the cluster.
- Same-namespace short name: `web-svc`
- Cross-namespace short form: `web-svc.demo`
- Full name: `web-svc.demo.svc.cluster.local`

Useful checks:

- `kubectl exec <pod> -n <ns> -- nslookup <svc>`
- `kubectl exec <pod> -n <ns> -- nslookup <svc>.<ns>.svc.cluster.local`
- `kubectl get svc -n kube-system -l k8s-app=kube-dns`
- `kubectl get po -n kube-system -l k8s-app=kube-dns`

BusyBox note:

- BusyBox `nslookup` can print the correct answer and still show extra `NXDOMAIN` lines for additional search-domain attempts.
- If the correct FQDN and ClusterIP are shown, DNS resolution is usually fine.

## NetworkPolicy

- A NetworkPolicy controls allowed traffic for selected Pods.
- Once a Pod is selected for ingress or egress, that direction becomes restricted to what is explicitly allowed.
- Policies are additive allow rules.

Selector behavior:

- `podSelector` matches Pods in the same namespace
- `namespaceSelector` matches namespaces by label
- `ipBlock` matches CIDRs

Cross-namespace trap:

- `podSelector` alone never means "any namespace"
- to match Pods in another namespace, combine:
  - `namespaceSelector`
  - and `podSelector`

Logic:

- `from` and `ports` in the same ingress rule are `AND`
- multiple entries under `from` are `OR`
- multiple entries under `ports` are `OR`
- multiple ingress rules are `OR`

AND vs OR trap:

- same `from` item with both `namespaceSelector` and `podSelector` = AND
- separate `from` items = OR

Ports note:

- ingress `ports` means destination ports on the selected Pod
- source/client port can be any ephemeral port

Useful checks:

- `kubectl get pod -n <ns> --show-labels`
- `kubectl get netpol -n <ns>`
- `kubectl describe netpol <name> -n <ns>`
- `kubectl exec <pod> -n <ns> -- wget -qO- --timeout=3 http://<svc>:<port>`
- `kubectl exec <pod> -n <ns> -- nc -vz -w 3 <svc> <port>`

Failure hints:

- timeout often suggests `NetworkPolicy`, routing, or unreachable backend
- `connection refused` often suggests wrong `targetPort` or app not listening

Lab reality note:

- NetworkPolicy YAML can be correct even if traffic is not blocked in a lab cluster.
- If the cluster does not have a policy-capable CNI (for example Calico, Cilium, or kube-router), NetworkPolicy objects may not be enforced.
- In that case, still practice:
  - YAML shape
  - least-privilege reasoning
  - selector logic

## Ingress

- An Ingress resource needs an Ingress controller.
- In this lab, use `ingressClassName: nginx`.
- An Ingress points to a Service, not directly to Pods.

Important fields:

- `ingressClassName`
- `rules[].host`
- `rules[].http.paths[].path`
- `pathType`
- backend Service name and port

Useful checks:

- `kubectl get ing -n <ns>`
- `kubectl describe ing <name> -n <ns>`
- `kubectl get svc -n ingress-nginx`
- `kubectl get po -n ingress-nginx`
- `kubectl logs -n ingress-nginx deploy/ingress-nginx-controller`

Route tests:

- `curl -H 'Host: <host>' http://127.0.0.1:<nodeport>`
- `wget -qO- --header='Host: <host>' http://127.0.0.1:<nodeport>`

Good signs in `describe ing`:

- correct `Ingress Class`
- controller sync events from `nginx-ingress-controller`
- backend Service and endpoint IPs are listed

## Gateway API

- Gateway API needs both CRDs and a real controller implementation.
- In this lab, NGINX Gateway Fabric provides `GatewayClass` `nginx`.
- `GatewayClass` selects the controller.
- `Gateway` creates the listener and data plane.
- `HTTPRoute` attaches rules to the Gateway.

Useful checks:

- `kubectl get gatewayclass`
- `kubectl get gateway,httproute -n <ns>`
- `kubectl describe gateway <name> -n <ns>`
- `kubectl describe httproute <name> -n <ns>`
- `kubectl get svc -n <ns>`
- `curl -H 'Host: <host>' http://127.0.0.1:<nodeport>`

Good signs:

- `GatewayClass` `Accepted=True`
- `Gateway` `Accepted=True`
- `Gateway` `Programmed=True`
- `Attached Routes` is non-zero
- `HTTPRoute` `Accepted=True`
- `HTTPRoute` `ResolvedRefs=True`

Controller-specific note:

- NGINX Gateway Fabric creates a data-plane Deployment and Service for the Gateway in the same namespace.

## Troubleshooting Flow

When traffic fails, check in this order:

1. Is the Pod running and ready
2. Does the Service selector match the Pod labels
3. Does the Service have endpoints
4. Does DNS resolve the Service name
5. Is the Service forwarding to the right `targetPort`
6. Is a `NetworkPolicy` blocking traffic
7. If using Ingress, is the correct host, class, path, and backend set
8. If using Gateway API, is the `Gateway` programmed and the `HTTPRoute` attached

High-value commands:

- `kubectl get deploy,po,svc,endpoints -n <ns>`
- `kubectl describe svc <name> -n <ns>`
- `kubectl get pod -n <ns> --show-labels`
- `kubectl exec <pod> -n <ns> -- nslookup <svc>`
- `kubectl exec <pod> -n <ns> -- wget -qO- --timeout=3 http://<svc>:<port>`
- `kubectl exec <pod> -n <ns> -- nc -vz -w 3 <svc> <port>`
- `kubectl describe netpol <name> -n <ns>`
- `kubectl describe ing <name> -n <ns>`
- `kubectl describe gateway <name> -n <ns>`
- `kubectl describe httproute <name> -n <ns>`

## Cheat Sheet

- Service failure:
  - `kubectl get svc,endpoints -n <ns>`
  - `kubectl describe svc <name> -n <ns>`
  - `kubectl get pod -n <ns> --show-labels`
- DNS proof:
  - `kubectl exec <pod> -n <ns> -- nslookup <svc>`
  - `kubectl exec <pod> -n <ns> -- nslookup <svc>.<ns>.svc.cluster.local`
  - `kubectl get svc -n kube-system -l k8s-app=kube-dns`
  - `kubectl get po -n kube-system -l k8s-app=kube-dns`
- HTTP proof:
  - `kubectl exec <pod> -n <ns> -- wget -qO- http://<svc>:<port>`
  - `kubectl exec <pod> -n <ns> -- wget -qO- --timeout=3 http://<svc>:<port>`
  - `kubectl exec <pod> -n <ns> -- nc -vz -w 3 <svc> <port>`
- NetworkPolicy proof:
  - `kubectl get netpol -n <ns>`
  - `kubectl describe netpol <name> -n <ns>`
  - `kubectl get pod -n <ns> --show-labels`
- Ingress proof:
  - `kubectl describe ing <name> -n <ns>`
  - `kubectl get svc -n ingress-nginx`
  - `curl -H 'Host: <host>' http://127.0.0.1:<nodeport>`
- Gateway API proof:
  - `kubectl get gatewayclass`
  - `kubectl describe gateway <name> -n <ns>`
  - `kubectl describe httproute <name> -n <ns>`
  - `curl -H 'Host: <host>' http://127.0.0.1:<nodeport>`
- Failure hints:
  - `Endpoints: <none>`: selector mismatch or Pods not ready
  - `connection refused`: wrong `targetPort` or app not listening
  - timeout: policy, routing, or unreachable backend
  - route object exists but no traffic: wrong host, class, path, or backend

## Exam Habits

- Set the namespace at the start of each task:
  - `kubectl config set-context --current --namespace=<ns>`
- Verify with `get` and `describe`, not just one of them.
- Prefer explicit resource lists over `kubectl get all`.
- Test from inside the cluster before testing from outside when possible.
- In troubleshooting, keep a short pattern:
  1. one failing command
  2. two or three inspection commands
  3. one minimal fix
  4. one success command
