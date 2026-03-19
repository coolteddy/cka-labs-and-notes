# CKA Networking Cheat Sheet

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
