# CKA Notes: Networking

## Cluster Reality

- This k3d cluster is using Flannel with `vxlan`.
- CoreDNS is running as `kube-dns` in `kube-system`.
- Available ingress classes are `traefik` and `nginx`.
- `ingress-nginx` is installed as a `NodePort` Service.
- Default k3s `traefik` is also running as a `LoadBalancer` Service.
- Calico is not installed on this cluster.
- NetworkPolicy YAML can be created here, but traffic enforcement needs a policy-capable CNI such as Calico or Cilium.

## Fast Commands

- Services, endpoints, and endpoint slices:
  - `kubectl get svc,endpoints,endpointslices -A`
- Describe a Service:
  - `kubectl describe svc <name> -n <ns>`
- Check Service selectors:
  - `kubectl get svc <name> -n <ns> -o yaml`
- Check Pod labels:
  - `kubectl get pod -n <ns> --show-labels`
- DNS test from a Pod:
  - `kubectl exec -it -n <ns> <pod> -- nslookup <svc>`
- HTTP test from a Pod:
  - `kubectl exec -it -n <ns> <pod> -- wget -qO- http://<svc>:<port>`
- Ingresses and classes:
  - `kubectl get ingress,ingressclass -A`
- Describe an Ingress:
  - `kubectl describe ingress <name> -n <ns>`
- Network policies:
  - `kubectl get netpol -A`
- Controller logs:
  - `kubectl logs -n ingress-nginx deploy/ingress-nginx-controller`

## Services

- A Service provides stable virtual IPs and DNS names for a changing set of Pods.
- Service selectors match Pod labels and build `Endpoints` or `EndpointSlices`.
- If a Service has no endpoints, traffic will fail even if the Service object exists.

Common types:

- `ClusterIP`: reachable only inside the cluster.
- `NodePort`: exposes the Service on each node IP and port.
- `LoadBalancer`: asks a controller to expose the Service externally.
- `ExternalName`: returns a DNS CNAME and does not create endpoints.

Important fields:

- `port`: the port exposed by the Service.
- `targetPort`: the Pod container port the Service forwards to.
- `nodePort`: fixed external port for `NodePort` or `LoadBalancer`.
- `selector`: labels used to find backend Pods.

Trouble patterns:

- wrong `selector`
- wrong `targetPort`
- Pods not `Ready`, so endpoints are empty
- app listens on a different container port than expected

## DNS

- CoreDNS resolves Service names inside the cluster.
- Short names work inside the same namespace: `web`
- Cross-namespace form: `web.demo`
- Full form: `web.demo.svc.cluster.local`

Useful rules:

- Pods get `/etc/resolv.conf` search domains for their namespace and cluster domain.
- Service DNS points to the Service IP, not directly to Pod IPs.
- Headless Services return Pod IPs instead of a virtual Service IP.

Fast checks:

- `nslookup web`
- `nslookup web.demo.svc.cluster.local`
- `wget -qO- http://web`
- `kubectl get svc -n demo`

## NetworkPolicy

- NetworkPolicy controls allowed traffic for selected Pods.
- Policies are additive allow rules, not ordered deny rules.
- Once a Pod is selected by an ingress or egress policy, that direction becomes restricted to what is explicitly allowed.

Key selectors:

- `podSelector`: select Pods in the same namespace
- `namespaceSelector`: select namespaces by label
- `ipBlock`: allow CIDRs

Important behavior:

- Empty `podSelector: {}` means "all Pods in this namespace".
- `policyTypes` can include `Ingress`, `Egress`, or both.
- DNS often breaks after egress lockdown unless you explicitly allow access to DNS.

Cluster caveat:

- On this Flannel-based k3d cluster, NetworkPolicy manifests can be created and inspected.
- Actual traffic blocking will not be enforced unless you install a policy-capable CNI.

## Ingress

- Ingress routes HTTP and HTTPS traffic to Services.
- An Ingress resource does nothing by itself without an Ingress controller.
- This cluster has two controllers, so set `ingressClassName` explicitly.

Important fields:

- `ingressClassName`: choose `nginx` or `traefik`
- `rules[].host`: optional host match
- `rules[].http.paths[].path`: URL path match
- backend `service.name` and `service.port`

Common mistakes:

- missing or wrong `ingressClassName`
- backend Service name typo
- wrong backend Service port
- host header does not match
- controller is healthy, but Service has no endpoints

## Troubleshooting Flow

When traffic fails, check in this order:

1. Is the Pod running and ready
2. Does the Service selector match the Pod labels
3. Does the Service have endpoints
4. Does DNS resolve the Service name
5. Is the app listening on the expected port
6. Is an Ingress pointing to the right Service and port
7. Is the correct Ingress controller watching that Ingress
8. Is a NetworkPolicy expected to matter, and does the cluster CNI actually enforce it

High-value commands:

- `kubectl get pod,svc,endpoints -n <ns> -o wide`
- `kubectl describe svc <name> -n <ns>`
- `kubectl describe ingress <name> -n <ns>`
- `kubectl exec -it -n <ns> <pod> -- nslookup <svc>`
- `kubectl exec -it -n <ns> <pod> -- wget -S -O- http://<svc>:<port>`
- `kubectl logs -n ingress-nginx deploy/ingress-nginx-controller`

Common failure signals:

- `Endpoints: <none>`: selector mismatch or Pods not ready
- `connection refused`: app is not listening on `targetPort`
- `no such host`: DNS name wrong or object missing
- Ingress exists but no routing: wrong class, host, path, or Service backend

## Cheat Sheet

- Service failing: check `selector`, `targetPort`, endpoints, Pod readiness
- DNS failing: check `kube-dns`, Service name, namespace, and FQDN
- Use `kubectl get svc,endpoints,endpointslices -A`
- Use `kubectl exec ... -- nslookup <svc>` for proof
- Ingress needs a controller and the right `ingressClassName`
- This cluster has both `traefik` and `nginx`, so always specify the class
- NetworkPolicy syntax is still exam-relevant even if this cluster does not enforce it
- For real NetworkPolicy blocking tests, install Calico or Cilium

## Exam Habits

- Scope every command with `-n <namespace>`.
- Verify Services with both `get` and `describe`.
- Always inspect endpoints when a Service does not work.
- Test from inside the cluster before testing from outside.
- On clusters with multiple ingress controllers, never rely on defaults.
- For NetworkPolicy, first confirm whether the cluster CNI actually enforces policies.
