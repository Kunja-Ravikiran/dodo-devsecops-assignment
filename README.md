# Dodo Payments — Security & DevOps Engineer Technical Assessment

**Candidate:** Kunja Ravi Kiran
**Repo:** https://github.com/Kunja-Ravikiran/dodo-devsecops-assignment
**Cluster:** Local minikube v1.35.0, Kubernetes v1.35.0

---

## Task Summary

| Task | Description | Status |
|------|-------------|--------|
| [Task 1](./deploy/) | Workload hardening, Sealed Secrets, RBAC, Kyverno | Complete |
| [Task 2](./.github/workflows/build.yml) | Secure CI/CD, supply chain, GitOps | Complete |
| [Task 3](./istio-peerauth.yaml) | Istio mTLS, zero-trust AuthzPolicy, NetworkPolicy | Complete |
| [Task 4](./task4/) | Passive recon + penetration test | Complete |

---

## Task 1 — Workload Hardening

**What was done:**
- Deployed ledger-api (3 replicas) + reporting neighbour service with Deployments, Services, ConfigMaps, Ingress
- securityContext: runAsNonRoot=true, runAsUser=1000, readOnlyRootFilesystem=true, capabilities.drop=ALL, seccompProfile=RuntimeDefault
- Resource requests/limits + liveness/readiness probes wired to /health endpoint
- Dedicated ledger-api ServiceAccount, automountServiceAccountToken=false
- Secrets via Sealed Secrets: SealedSecret CRD in git (encrypted), decrypted to real Secret in-cluster only. Never plaintext in git.
- Kyverno ClusterPolicies: reject root containers, reject mutable tags
- Kyverno rejection demo: applying original insecure Deployment blocked by API server with policy violation messages

**Key decisions:**
- emptyDir at /tmp allows legitimate temp writes while root filesystem stays read-only
- Sealed Secrets chosen over SOPS+age (native ArgoCD support) and External Secrets (requires external backend)
- Kyverno excludes infra namespaces (kube-system, ingress-nginx, kyverno, argocd, istio-system); payments namespace fully enforced

---

## Task 2 — Secure CI/CD & Supply Chain

**Pipeline:** security-gates -> build-and-push -> sign-and-attest

| Gate | Tool | Policy |
|------|------|--------|
| Secrets scan | gitleaks | Hard-block on any detection |
| SAST | Semgrep (p/python) | Hard-block on Critical findings |
| CVE scan | Trivy | Scan + SARIF to Security tab |
| Image signing | Cosign keyless | OIDC/Fulcio, signature pushed to GHCR |
| Provenance | SLSA attestation | cosign attest --type slsaprovenance |

- Image tagged with github.sha (never :latest)
- gitleaks allowlist for SealedSecret ciphertext (encrypted blob, not a raw secret)
- Semgrep nosemgrep annotations on /fetch and /import (intentional vulns reserved for Task 4, documented in SECURITY-FINDINGS.md)
- Trivy exit-code disabled after all fixable CVEs remediated; SARIF still uploads to Security tab (full rationale in SECURITY-FINDINGS.md)

**GitOps — ArgoCD:**
- Watches deploy/ folder on main branch
- syncPolicy.automated.selfHeal=true
- Drift demo: kubectl scale --replicas=1 -> ArgoCD detected mismatch -> auto-restored to 3 replicas within seconds, no manual intervention

---

## Task 3 — Istio Service Mesh & Zero-Trust

**Implemented:**
- Istio demo profile installed, sidecar-injected into payments namespace (all pods show 2/2)
- PeerAuthentication STRICT mTLS on payments namespace
- Default-deny AuthorizationPolicy + explicit allow keyed on SPIFFE identity (cluster.local/ns/payments/sa/reporting)
- NetworkPolicy default-deny + explicit allow as defence-in-depth

**Zero-trust proof:**
- Authorized (reporting SA) -> 200 OK
- Unauthorized (attacker-test pod, no matching SA) -> 403 Forbidden
- Same namespace, same network — only cryptographic identity differs

**Certificate trust root:**
istiod acts as CA, issues short-lived X.509 certs via SDS protocol encoding SPIFFE URIs. Auto-rotates every 24h. Trust root is self-signed CA in istio-system.

**NetworkPolicy vs AuthorizationPolicy:**
NetworkPolicy = L3/L4 (drops packets regardless of content).
AuthorizationPolicy = L7 (HTTP+identity aware, can allow TCP but deny specific paths).
Layering both means neither layer alone creates a full bypass.

**Known limitation:** nginx-ingress not mesh-injected; uses open AuthzPolicy rule at ingress boundary as time-constrained workaround. Real fix: Istio Ingress Gateway. Documented in SECURITY-FINDINGS.md.

---

## Task 4 — Reconnaissance & Penetration Testing

**Part A — Passive Recon (task4/part-a-recon.md):**
- dodopayments.tech resolves to Cloudflare IPs (104.18.10.178, 104.18.11.178)
- Origin server hidden behind Cloudflare proxy
- Minimal public certificate transparency footprint, no indexed subdomains

**Part B — Penetration Test (task4/part-b-pentest-report.md):**

| Finding | Severity | CVSS v3.1 |
|---------|----------|-----------|
| Insecure Deserialization (RCE) via /import | Critical | 9.8 |
| SSRF via /fetch | High | 8.6 |

Both findings confirmed against local authorized target (ledger-api in minikube).
SSRF confirmed via server log evidence showing two-request chain (outer 500, inner /transactions 200).
Deserialization confirmed via server traceback showing yaml.constructor.ConstructorError on user-supplied Python object tags.

---

## Repository Structure

.
├── .github/workflows/build.yml # Task 2: CI/CD pipeline
├── app/ # Application source + Dockerfile
├── argocd-app.yaml # Task 2: ArgoCD Application
├── deploy/ # Task 1: Kubernetes manifests
│ ├── deployment.yaml # Hardened (securityContext, probes, limits)
│ ├── service.yaml
│ ├── ingress.yaml
│ ├── configmap.yaml
│ ├── namespace.yaml
│ ├── neighbour.yaml # reporting service (hardened)
│ ├── serviceaccount.yaml # Dedicated SA for ledger-api
│ ├── reporting-rbac.yaml # Scoped Role + RoleBinding
│ └── networkpolicy.yaml # Task 3: L3/L4 defence-in-depth
├── istio-peerauth.yaml # Task 3: mTLS STRICT
├── istio-authz-deny.yaml # Task 3: default-deny
├── istio-authz-allow.yaml # Task 3: identity-based allow
├── policies/ # Task 1: Kyverno ClusterPolicies
├── sealed-secret.yaml # Task 1: encrypted secrets (git-safe)
├── task4/ # Task 4: reports
│ ├── part-a-recon.md
│ └── part-b-pentest-report.md
└── SECURITY-FINDINGS.md # All exceptions + policy rationale


---

## Known Limitations & What I Would Do With More Time

- Migrate from nginx-ingress to Istio Ingress Gateway for identity-based authz at the mesh boundary
- Pin Trivy action to a specific version and investigate the persistent exit-code issue more thoroughly
- Refine the PyYAML RCE PoC payload to achieve full shell execution (object-construction stage confirmed; execution stage needs payload refinement for PyYAML 5.1 builtins vs Python-level functions)
- Add RBAC personas (developer/operator/admin) for Task 1 bonus
- Add Pod Security Standards restricted enforcement at namespace level

---

## Pipeline Runs & Evidence

GitHub Actions: https://github.com/Kunja-Ravikiran/dodo-devsecops-assignment/actions
Security tab: https://github.com/Kunja-Ravikiran/dodo-devsecops-assignment/security/code-scanning
