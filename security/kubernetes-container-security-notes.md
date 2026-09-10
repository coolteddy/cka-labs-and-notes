# Kubernetes Container Security Notes

## ServiceAccounts and mounted tokens

- If a Pod does not specify `spec.serviceAccountName`, it uses the namespace's `default` ServiceAccount.
- `kubectl get serviceaccount` may show `SECRETS: 0` even when a Pod has a token mounted.
- Modern Kubernetes normally uses the TokenRequest API to mount a short-lived, projected token under:

  ```text
  /var/run/secrets/kubernetes.io/serviceaccount
  ```

- A volume named like `kube-api-access-*` is evidence of this projected identity bundle.
- Reduce risk by using a dedicated ServiceAccount, least-privilege RBAC, and `automountServiceAccountToken: false` when the workload does not need the Kubernetes API.

## Compromised Pod attack path

```text
Vulnerable application
        ↓
Attacker gains code execution inside the container
        ↓
Steals the mounted ServiceAccount token
        ↓
Uses permitted Kubernetes API access for reconnaissance
        ↓
Retrieves accessible credentials or sensitive configuration
        ↓
Uses those credentials for lateral movement
        ↓
Reaches sensitive workloads, databases, or infrastructure
```

### Reconnaissance

Reconnaissance means learning about the environment and identifying useful targets. Depending on RBAC permissions, an attacker might discover:

- Workload names, labels, images, Pod IPs, and nodes
- Services and application components
- ServiceAccounts used by workloads
- Environment-variable and Secret references
- Databases, administrative services, and other sensitive targets

### Credential theft

A Pod normally sees only credentials mounted into that Pod. However, an attacker can use its mounted ServiceAccount token to call the Kubernetes API and retrieve any Secrets permitted by that identity's RBAC.

Stolen credentials might include:

- Database usernames and passwords
- API tokens
- TLS private keys
- Registry credentials
- Static cloud credentials

### Lateral movement

Lateral movement means using the initial compromised workload to reach other systems.

Examples include:

- Connecting to another internal Service
- Authenticating to a database with a stolen password
- Calling cloud APIs with stolen credentials
- Exploiting another vulnerable workload
- Using excessive Kubernetes RBAC to inspect or modify other resources

Being inside the cluster does not automatically grant access to every workload. The attacker still needs network reachability, credentials, excessive RBAC, or another vulnerability. NetworkPolicy, authentication, and authorization should continue enforcing boundaries inside the cluster.

### Blast radius

Blast radius is the maximum damage possible from one compromised component.

Reduce it with:

- Dedicated ServiceAccounts and least-privilege RBAC
- `automountServiceAccountToken: false` when API access is unnecessary
- Default-deny NetworkPolicies with explicit allow rules
- Short-lived workload credentials
- Namespace, node, or cluster isolation according to tenant sensitivity
- Minimal Secrets exposure
- Restricted security contexts and Pod Security enforcement
- Kubernetes audit logs and runtime monitoring

### Why disable token automount for a “powerless” ServiceAccount?

Every Pod uses a ServiceAccount. If `serviceAccountName` is omitted, it uses the namespace's `default` ServiceAccount.

Even if that identity currently has minimal RBAC, its token is still a valid Kubernetes credential. Its permissions may expand later through RBAC drift, bindings to ServiceAccount groups, or bindings to all authenticated identities.

If the application does not need the Kubernetes API:

```yaml
spec:
  automountServiceAccountToken: false
```

Interview-ready answer:

> Every Pod uses a ServiceAccount, even when one isn't specified—it gets the namespace's default ServiceAccount. If the application doesn't need the Kubernetes API, I disable token automount so a compromised container has no Kubernetes credential to steal. This also protects against future RBAC mistakes that might give the default ServiceAccount more access.

### Interview-ready threat explanation

> After compromising an application, an attacker may steal its mounted ServiceAccount token and use whatever RBAC permissions it has for reconnaissance. If the identity can retrieve Secrets, those credentials may support lateral movement into databases, services, or infrastructure. I reduce the blast radius with dedicated least-privilege ServiceAccounts, disabling unnecessary token mounts, NetworkPolicy, short-lived workload credentials, and monitoring unusual API activity through audit logs.

## What `securityContext: {}` means

An empty security context does **not** mean secure defaults. It means Kubernetes has not requested additional restrictions.

Typical implications include:

```text
runAsNonRoot              not enforced
runAsUser                 image/runtime default
allowPrivilegeEscalation  normally true
readOnlyRootFilesystem    false
privileged                false
capabilities.drop         none explicitly dropped
```

The container is not automatically privileged, but important hardening controls are absent.

A stronger baseline is:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 10001
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL
```

Applications may require explicitly mounted writable volumes for legitimate temporary files, caches, or application data.

## Privilege escalation and setuid

Linux processes have a real UID and an effective UID. Simplifying slightly:

```text
Application process
Real UID:      10001
Effective UID: 10001
```

A root-owned executable can have the setuid permission bit:

```text
-rwsr-xr-x root root /usr/bin/example
```

When UID `10001` executes it, Linux can set the new process's effective UID to the file owner:

```text
UID 10001 executes setuid-root program
                ↓
Real UID:      10001
Effective UID: 0
```

Setuid is not inherently a vulnerability. The risk arises when such a program is vulnerable, unnecessary, or attacker-controlled.

With:

```yaml
allowPrivilegeEscalation: false
```

the runtime applies Linux `no_new_privs`. The kernel then refuses to grant additional privilege through setuid executables or file capabilities.

Important distinctions:

- `runAsNonRoot: true` does not automatically prevent later privilege escalation.
- Running as root does not itself prove that `allowPrivilegeEscalation` is enabled.
- Enforce both `runAsNonRoot: true` and `allowPrivilegeEscalation: false` where possible.

## Container root versus host root

Root inside a container is not automatically root on the Kubernetes node. Container namespaces and other isolation mechanisms still form a boundary.

An attacker generally needs another weakness to escape or affect the host, such as:

- A kernel or container-runtime vulnerability
- A privileged container
- Dangerous Linux capabilities
- A sensitive or writable `hostPath`
- Access to a container-runtime socket
- Host namespaces such as `hostPID` or `hostNetwork`

Use layered controls so one application compromise does not immediately become a node or cluster compromise.

## Linux capabilities

Linux capabilities split traditional root authority into smaller privileges.

Examples:

| Capability | Purpose or risk |
|---|---|
| `NET_BIND_SERVICE` | Bind ports below 1024 |
| `NET_RAW` | Create raw sockets; useful in some network attacks |
| `CHOWN` | Change file ownership |
| `SYS_PTRACE` | Inspect or manipulate other processes where isolation permits |
| `NET_ADMIN` | Change network configuration |
| `SYS_ADMIN` | Extremely broad and dangerous; associated with many escape paths |

An omitted capabilities section does not mean the process has no capabilities. The container runtime normally supplies a default subset.

Prefer dropping everything and adding back only demonstrated requirements:

```yaml
securityContext:
  capabilities:
    drop:
      - ALL
    add:
      - NET_BIND_SERVICE
```

An application listening on an unprivileged port such as `8080` may not need any capabilities added back.

## Image tags and immutable digests

### Mutable tags

This is an image tag:

```text
httpd:2.4
```

A tag is a human-friendly pointer. By default, it can be moved to different content:

```text
Today:     httpd:2.4 → sha256:AAAA
Next week: httpd:2.4 → sha256:BBBB
```

Consequences include:

- The same Kubernetes YAML can produce different containers at different times.
- Nodes with cached images and nodes pulling later may run different content.
- It is harder to prove that the deployed image is the exact artifact that CI scanned and approved.

A registry can enforce a policy preventing tag overwrites. This is commonly called **tag immutability**, but it comes from registry policy rather than the tag format.

### Immutable digest references

A digest reference identifies exact image content:

```text
registry.example.com/team/myapp@sha256:abc123...
```

Changing the content changes the digest. Kubernetes can also use a readable tag alongside the digest:

```text
registry.example.com/team/myapp:1.2.3@sha256:abc123...
```

The digest determines the content; the tag provides human-readable context.

### Typical build and deployment workflow

1. Build and assign a readable tag:

   ```bash
   docker build -t registry.example.com/platform/myapp:1.2.3 .
   ```

2. Push the image:

   ```bash
   docker push registry.example.com/platform/myapp:1.2.3
   ```

3. Capture the digest reported by the registry.
4. Scan and approve that digest.
5. Deploy the approved digest:

   ```yaml
   containers:
   - name: myapp
     image: registry.example.com/platform/myapp@sha256:abc123...
   ```

Docker build creates an image and can assign a tag. It does not make an ordinary tag inherently immutable. The registry calculates and stores the content digest when the image is pushed.

## Desired image, image ID, and container ID

These are different identifiers.

### Desired image

The Pod specification declares the requested image:

```yaml
spec:
  containers:
  - image: httpd:2.4
```

JSONPath:

```text
.spec.containers[*].image
```

### Resolved image ID

The runtime records the exact image content it resolved:

```text
docker.io/library/httpd@sha256:979c...
```

JSONPath:

```text
.status.containerStatuses[*].imageID
```

### Container ID

The container ID identifies a particular running container instance:

```text
containerd://5d7bcca3...
```

JSONPath:

```text
.status.containerStatuses[*].containerID
```

Multiple Pods can run the same `imageID` while each has a different `containerID`.

Example comparison:

```bash
kubectl get pods -n interview-lab -l app=api \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].image}{"\t"}{.status.containerStatuses[*].imageID}{"\n"}{end}'
```

## `hostPath`, local PVs, and `local-path`

### `hostPath`

A `hostPath` volume directly mounts a directory from the Kubernetes node into the Pod. It does not use a PV or PVC.

```text
Pod /host → node /
```

It can create a direct node-compromise path, especially when writable or combined with privileged mode. Sensitive examples include `/`, `/var/lib/kubelet`, `/etc`, and container-runtime sockets.

### Kubernetes local PersistentVolume

A `local` PV represents node-local storage through the PV/PVC model. It uses node affinity so the scheduler knows which node owns the storage.

```text
Pod → PVC → local PV → disk/directory on a specific node
```

It provides better storage lifecycle and scheduling awareness than raw `hostPath`, but does not provide cross-node replication or automatic failover.

### k3s `local-path` provisioner

The `local-path` StorageClass dynamically provisions PVs backed by directories on Kubernetes nodes. It is convenient for k3s and development clusters but is still node-local storage, not distributed or inherently highly available storage.

Interview-ready answer:

> A `hostPath` directly exposes a node path to a Pod and may create a serious node-compromise path. A `local` PV exposes node-local storage through PV/PVC and node affinity, making scheduling storage-aware. The k3s `local-path` provisioner creates node-local PVs dynamically. None provides automatic cross-node replication.

## Prioritising controls for a dangerous Pod

Scenario:

```text
Privileged container
+ running as root
+ writable hostPath mounting node /
= high likelihood of node compromise
```

Prioritised controls:

1. Remove the `hostPath`.
   - Mitigates direct access to node files and credentials.
   - `readOnlyRootFilesystem` does not make mounted volumes read-only.
   - If a host mount is unavoidable, scope it narrowly and make it read-only.

2. Disable privileged mode.
   - Mitigates broad access to devices and kernel-facing functionality.
   - Restores important container-isolation boundaries.

3. Apply a restrictive security context.

```yaml
securityContext:
  privileged: false
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL
```

Control meanings:

- `runAsNonRoot`: prevents execution as UID 0. It does not prevent reading root-owned files when normal file permissions allow access.
- `allowPrivilegeEscalation: false`: prevents a process gaining privileges beyond its parent, including through setuid or file capabilities. It does not remove capabilities already granted to the process.
- `readOnlyRootFilesystem: true`: protects the container image filesystem. It does not automatically protect mounted volumes.
- `capabilities.drop: [ALL]`: removes the runtime's default Linux capability set. Add back only capabilities the application demonstrably requires.

Interview-ready answer:

> My first priority is removing the writable host mount because it directly exposes the node filesystem. Second, I disable privileged mode to restore container isolation. Third, I enforce non-root execution, disable privilege escalation, drop all capabilities, and use a read-only container filesystem. Each control addresses a different path, so I would apply them together.

## Admission control and Pod Security

Admission policy can block unsafe Pods, but strict security is not automatic.

```text
API request
→ authentication
→ RBAC authorization
→ admission checks
→ object stored
```

Pod Security Admission supports:

- `privileged`: unrestricted
- `baseline`: blocks common privilege-escalation risks
- `restricted`: stronger application-workload hardening

Example namespace enforcement:

```yaml
metadata:
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.31
```

Modes:

- `enforce`: reject violations
- `warn`: allow but warn the user
- `audit`: allow but annotate the audit event

Important limitations:

- Without enforcement configuration, unsafe Pods may be admitted.
- Existing Pods are not automatically terminated when policy changes.
- Exempt users, namespaces, or RuntimeClasses may bypass checks.
- Namespace security labels must be protected with RBAC.
- Pod Security does not enforce every organisational requirement.

Custom admission policies may additionally require:

- Approved registries
- Images pinned by digest
- Read-only root filesystems
- Approved ServiceAccounts
- Mandatory metadata
- Organisation-specific volume restrictions

Admission is preventive and does not replace RBAC, NetworkPolicy, image scanning, audit logs, or runtime detection.

Interview-ready answer:

> Admission can block dangerous Pod configurations, but only when policy is configured and enforced. I would use restricted Pod Security for application namespaces, add custom admission rules for image and organisational controls, tightly control exemptions, and retain runtime monitoring for activity that admission cannot detect.

## Interview summary

A concise answer could be:

> I would assume a compromised application can execute within its container and design controls to stop the attack progressing. I would run it as a non-root UID, disable privilege escalation, drop all capabilities, use a read-only root filesystem, avoid privileged mode and host mounts, and give it a dedicated least-privilege ServiceAccount without an API token unless required. I would deploy an approved image by digest so the running artifact is the one CI scanned and authorized. These preventive controls would be backed by audit logging, runtime detection, network policy, and an incident response process.

## Final Security recap

### Security assessment approach

Before choosing controls, clarify:

- Workloads and exposed entry points
- Data sensitivity
- Tenants and administrators
- Connectivity requirements
- Trust and compliance boundaries

Use this structure:

```text
Threat
→ Prevent
→ Detect
→ Respond
→ Explain blast radius
```

Assume that an individual container may eventually be compromised, and design the platform so that compromise remains contained.

### Workload controls

- Run containers as non-root and non-privileged.
- Disable privilege escalation.
- Drop all capabilities and add back only demonstrated requirements.
- Use a read-only root filesystem.
- Avoid `hostPath` and host namespaces.
- Enforce Pod Security plus additional organisational admission policies.

### Identity

```text
Human access      = SSO/MFA + group-based RBAC
Kubernetes access = ServiceAccount + RBAC
Cloud access      = workload identity + cloud IAM
```

- Prefer short-lived credentials and dedicated workload identities.
- Disable ServiceAccount token automount when Kubernetes API access is not needed.
- Use just-in-time elevation and audited break-glass access for administrators.

### TLS and cryptography

```text
TLS   = server identity + encrypted connection
mTLS  = both client and server authenticate
CA    = establishes which certificates are trusted
```

- Automate certificate issuance, expiry monitoring and rotation.
- Use overlapping trust during CA rotation to avoid outages.
- Separate trust roots or encryption keys when tenant risk requires it.

```text
Base64          = encoding only
API encryption  = API server encrypts data before etcd
etcd            = stores the ciphertext
KMS             = manages and protects the key-encryption key
HSM             = protects key material in hardware
```

### Multi-tenancy

```text
Namespace = logical boundary
Node pool = workload/host boundary
Cluster   = control-plane and failure boundary
```

- Namespaces require RBAC, NetworkPolicy, Pod Security, quotas and dedicated identities.
- Use dedicated nodes for host or compliance isolation.
- Use separate clusters for hostile tenants, different administrators, security classifications or stronger blast-radius requirements.

```text
Taint      = keep ordinary workloads out
Toleration = permit an approved workload onto the node
Affinity   = require that workload to remain on the intended nodes
Admission  = enforce the namespace-to-node rule
```

### Software supply chain

```text
Developer → Git → PR → CI → Build → Registry → Admission → Kubernetes
```

- Protect pipeline changes and separate untrusted PR tests from release builds.
- Use isolated disposable builders with temporary least-privilege credentials.
- Generate an SBOM, scan the image, record provenance and sign the immutable digest.
- Store artifacts in a trusted registry and reject unapproved images at admission.
- Rebuild, rescan and redeploy when vulnerabilities are discovered later.

```text
SBOM       = what is inside?
Provenance = where did it come from and how was it built?
Signature  = who vouches for this exact digest?
```

### Monitoring

```text
Application = how entry happened
Runtime     = what executed
Audit       = what the Kubernetes identity did
Network     = where the attacker moved
Node        = whether container isolation was crossed
SIEM        = correlates the evidence
```

Audit events should be assessed using identity, source IP, user agent, verb, resource, namespace, response code and timestamp.

Suspicious activity includes Secret reads, `pods/exec`, privileged Pod creation, RBAC changes, unusual cross-namespace access and repeated authorization failures.

### Incident response

```text
Validate
→ establish timeline and blast radius
→ contain workload, identity and network
→ preserve evidence
→ rotate exposed credentials
→ investigate lateral movement and node impact
→ rebuild from a trusted image
→ remediate and verify controls
```

- Out-of-band telemetry comes from trusted systems outside the compromised container.
- Cordoning prevents new scheduling; it does not isolate existing workloads.
- Quarantining restricts communication while preserving controlled forensic access.
- Do not clean a compromised container in place.
- Preserve Kubernetes state, logs, runtime evidence and storage snapshots as appropriate.
- Deployment replacement removes ephemeral state but not the original vulnerability.
- StatefulSet recreation may reattach a compromised PVC, so data integrity must be assessed.

### Important distinctions

- Root is not the same as privileged.
- `privileged: false` preserves the normal container sandbox.
- Dropping capabilities removes powers held now.
- Disabling privilege escalation prevents gaining powers later.
- A read-only root filesystem does not make mounted volumes read-only.
- Cordoning stops new scheduling; it does not quarantine a node.
- Namespaces do not automatically provide network or hard security isolation.
- Encryption does not replace RBAC.
- A signed image is not automatically vulnerability-free.
- A recreated StatefulSet Pod may reattach compromised persistent data.

### Final summary

> My goal is not to claim that compromise is impossible. I design the platform so one compromised container has minimal identity, network reach and host access; so suspicious behaviour is visible across application, runtime, audit, network and node telemetry; and so we can contain, investigate and recover through a trusted, repeatable process.

## Supply-chain incident scenario

### Situation

A production workload uses:

```text
public-registry.example/payment-api:latest
```

New Pods begin making unexpected outbound connections.

Possible causes:

- Source or dependency compromise
- Malicious pipeline change
- Compromised build worker
- Stolen registry credential
- Mutable tag replaced with different content
- Vulnerability discovered after the original scan

### Investigate

Trace the running artifact backwards:

```text
Running image digest
→ registry audit records
→ signature and provenance
→ CI build
→ source commit
→ SBOM
```

Collect:

- Declared image and resolved digest
- Registry push/pull events
- CI and deployment records
- Source and pipeline changes
- Signature and provenance verification
- Runtime processes and network activity
- Every workload running the affected digest

### Prevent

```text
Git:
MFA + protected branches + reviewed pipeline changes

CI:
isolated disposable builders + temporary least-privilege credentials

Build:
minimal image + SBOM + scanning + provenance + signed digest

Registry:
approved private registry + immutable tags + RBAC + audit logs

Admission:
verify registry + digest + signature + provenance + workload policy
```

### Detect

- Unexpected registry pushes or tag changes
- Pipeline changes and builds from unapproved branches
- Signature, provenance or admission failures
- New vulnerabilities matched against deployed SBOMs
- Unexpected runtime processes or network connections
- Differences between declared and running image digests

### Respond

```text
Block affected digest
→ stop further rollout
→ isolate affected workloads
→ identify every deployment using it
→ preserve CI, registry and runtime evidence
→ rotate affected credentials or signing keys
→ rebuild in a clean trusted builder
→ scan, sign and deploy a new digest
→ investigate data access and verify recovery
```

### Vulnerability discovered after deployment

```text
New CVE
→ search SBOM inventory
→ identify affected deployed digests
→ assess reachability and exploitability
→ patch and rebuild
→ rescan and sign
→ deploy new digest
→ retire or block old digest
```

### Air-gapped import

```text
External artifact
→ verify source, SBOM, signature and provenance
→ scan
→ controlled transfer across boundary
→ import into internal registry
→ scan and verify again
→ admission-controlled deployment
```

Vulnerability databases and security updates also require a controlled offline import process.

### Interview answer

> I would trace the running digest through the registry, build provenance and source commit rather than trust its tag. I would protect pipeline changes, build in isolated workers with temporary credentials, and produce an SBOM, scan result, provenance and signature for the immutable digest. Admission would allow only trusted, verified images. If trust were lost, I would block the digest, identify affected workloads, rotate credentials, rebuild through a clean pipeline and deploy a newly signed digest.

## Privileged-container demonstration

The same BusyBox image behaved differently depending on runtime privileges.

```text
Privileged Pod:
UID 0
CapEff: 000001ffffffffff
tmpfs mount succeeded

Normal Pod:
UID 0
CapEff: 00000000a80425fb
tmpfs mount failed with permission denied
```

This demonstrates:

- Root inside a container does not automatically possess every capability.
- Privileged mode grants broad capabilities, including `SYS_ADMIN`.
- `SYS_ADMIN` permits powerful operations such as mounting filesystems.
- The tmpfs test used the container's mount namespace; it did not mount node data.

```text
privileged → broad kernel/device powers
hostPath   → node filesystem exposure
root       → broad filesystem permissions
```

The combination creates a serious node-compromise path.

## Built-in versus custom admission policy

Pod Security Admission with `baseline` or `restricted` can reject `hostPath` volumes during Pod creation.

```text
Pod with hostPath
→ API admission
→ Pod Security violation
→ Forbidden
→ Pod is not stored or scheduled
```

The built-in Pod Security Standards do not require:

```yaml
readOnlyRootFilesystem: true
```

Enforce that organisational requirement using:

- Kubernetes `ValidatingAdmissionPolicy` with CEL
- Kyverno
- OPA Gatekeeper
- A custom validating admission webhook

Admission validation is synchronous:

```text
create/update request
→ evaluate policy
→ allow or reject
→ store only if allowed
```

Adding a policy does not automatically delete existing non-compliant Pods. Adopt safely through:

```text
audit existing workloads
→ warn
→ remediate
→ enforce
```

Mental model:

```text
hostPath               → built-in Pod Security can reject
readOnlyRootFilesystem → custom admission policy required
existing violations    → background audit/reporting required
```
