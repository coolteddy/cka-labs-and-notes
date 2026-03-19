# CKA Networking Drills

This folder contains the CKA-focused networking drills completed in the kubeadm + Calico lab.

Layout:

- `01-services/`: basic `ClusterIP` Service creation
- `02-dns/`: Service discovery and DNS checks
- `03-networkpolicy/`: allow-list ingress policy
- `04-ingress/`: `ingress-nginx` routing
- `05-gateway-api/`: NGINX Gateway Fabric routing
- `06-gateway-troubleshooting/`: Gateway API troubleshooting
- `07-service-troubleshooting/`: Service port troubleshooting
- `08-networkpolicy-troubleshooting/`: label-based `NetworkPolicy` troubleshooting

Notes:

- Early drill files are cleaned copies of the manifests created in the lab.
- Troubleshooting drills include both broken and fixed versions for re-use.
- These drills are separate from the older `exercises/networking/` content that was built for a different lab environment.
