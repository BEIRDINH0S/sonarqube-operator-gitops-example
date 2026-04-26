# sonarqube-operator-gitops-example

End-to-end GitOps example for [sonarqube-operator](https://github.com/BEIRDINH0S/sonarqube-operator):
Argo CD pulls this repo and reconciles a complete SonarQube setup — instance, plugins,
projects, quality gates, users, groups, permission templates, webhooks, branch rules,
and backups — from declarative YAML.

```
You commit YAML  ──►  Argo CD syncs  ──►  Operator reconciles  ──►  SonarQube state
```

## Prerequisites

- A Kubernetes cluster with an ingress controller. For local testing on kind,
  use the included [`kind-config.yaml`](kind-config.yaml):
  ```bash
  kind create cluster --name sonarqube --config kind-config.yaml --image kindest/node:v1.31.0
  kubectl apply -f https://kind.sigs.k8s.io/examples/ingress/deploy-ingress-nginx.yaml
  ```
- [`sonarqube-operator`](https://github.com/BEIRDINH0S/sonarqube-operator) installed
  (e.g. `helm install sonarqube-operator ./charts/sonarqube-operator -n sonarqube-operator-system --create-namespace`)
- [Argo CD](https://argo-cd.readthedocs.io/en/stable/getting_started/) installed in
  the `argocd` namespace

## Bootstrap

One-time, apply the contents of [`argocd/`](argocd/) — the Application that points
Argo CD at this repo, plus the Ingress for the Argo CD UI itself:

```bash
kubectl apply -f argocd/
```

From now on, every commit to this repo flows to your cluster.

## Accessing the services

If you used the bundled `kind-config.yaml`, the cluster's ingress controller is
exposed on host ports `30080` (HTTP) and `30443` (HTTPS). On a real cluster,
adjust the URLs to your ingress's address.

| Service     | URL                                            | Credentials |
|-------------|------------------------------------------------|-------------|
| SonarQube   | http://sonarqube.127.0.0.1.nip.io:30080        | `admin` / value of `sonar-admin` Secret |
| Argo CD UI  | https://argocd.127.0.0.1.nip.io:30443          | `admin` / `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' \| base64 -d` |

> [`nip.io`](https://nip.io) is a wildcard DNS that resolves
> `<anything>.<ip>.nip.io` to `<ip>`, **IPv4 only**. We use it instead of
> `localtest.me` (which also returns `::1` AAAA records) because Docker Desktop
> on Windows only publishes ports on IPv4, and some browsers fail to fall back
> from IPv6 to IPv4 fast enough.

> **Homelab / production.** Replace the `127.0.0.1.nip.io` host in
> [`apps/sonarqube/instance.yaml`](apps/sonarqube/instance.yaml) and
> [`argocd/ingress.yaml`](argocd/ingress.yaml) with whatever DNS your cluster
> exposes (e.g. `sonarqube.lab.example.com`). The Argo CD `Application` will
> auto-sync the change.

## What's in here

```
apps/sonarqube/
├── 00-namespace.yaml                # the sonarqube namespace
├── 01-postgres.yaml                 # demo PostgreSQL backend (replace in prod)
├── 02-secrets.yaml                  # placeholder secrets — see "Secrets" below
├── instance.yaml                    # SonarQubeInstance — the SonarQube install
├── plugin.yaml                      # SonarQubePlugin — Mercurial SCM plugin
├── quality-gate.yaml                # SonarQubeQualityGate — coverage > 80%, etc.
├── project.yaml                     # SonarQubeProject — demo project
├── sonarqube-operator-project.yaml  # SonarQubeProject — dogfooding target (see below)
├── user.yaml                        # SonarQubeUser — a CI bot account
├── group.yaml                       # SonarQubeGroup — developers group
├── permission-template.yaml         # SonarQubePermissionTemplate — default template
├── webhook.yaml                     # SonarQubeWebhook — analysis-finished hook
├── branch-rule.yaml                 # SonarQubeBranchRule — main branch new-code
└── backup.yaml                      # SonarQubeBackup — nightly pg_dump
```

Every CRD shipped by the operator has at least one example here. Read top to
bottom to learn how they fit together.

> **Status of the scaffold CRDs.** `SonarQubeBranchRule` and `SonarQubeBackup`
> are accepted by the API but their reconcile pipelines are not wired up yet
> (tracked as sonarqube-operator issues #58 and #59). The CRs in this repo are
> already shaped so they will reconcile automatically once those land — no
> spec change required.

## Dogfooding: scanning the operator's own code

The `sonarqube-operator-project.yaml` manifest creates a SonarQube project
specifically for the GitHub Actions workflow on
[`BEIRDINH0S/sonarqube-operator`](https://github.com/BEIRDINH0S/sonarqube-operator)
to scan against. Wiring it up:

1. Apply this repo (Argo CD will create the project + a CI token Secret).
2. Wait for the Project CR to reach `Ready`:
   ```bash
   kubectl get sonarqubeproject sonarqube-operator -n sonarqube -w
   ```
3. Extract the CI token from the Secret the operator just created:
   ```bash
   kubectl get secret sonarqube-operator-ci-token -n sonarqube \
     -o jsonpath='{.data.token}' | base64 -d
   ```
4. On the operator's GitHub repo, add two repository secrets/variables:
   - **Secret** `SONAR_TOKEN` — the value from step 3
   - **Variable** `SONAR_HOST_URL` — the public URL of your SonarQube
     (e.g. `https://sonarqube.<your-homelab-domain>`)
5. The workflow `.github/workflows/sonar-analysis.yml` on the operator repo
   picks both up and runs `sonar-scanner` on every PR + push to `main`.

To rotate the token (recommended periodically), annotate the Project CR — the
operator revokes the old token and creates a new Secret value:

```bash
kubectl annotate sonarqubeproject sonarqube-operator -n sonarqube \
  sonarqube.io/rotate-token=true --overwrite
```

## Secrets

The `02-secrets.yaml` file ships **placeholder credentials** so the example runs out of
the box. **Do not use these values in production.** For real deployments use one of:

- [Sealed Secrets](https://sealed-secrets.netlify.app/) — encrypt secrets in git
- [External Secrets Operator](https://external-secrets.io/) — pull from Vault, AWS SM, etc.
- [SOPS](https://github.com/getsops/sops) — encrypt YAML files in git

The operator only reads `Secret` objects in the cluster — it does not care how they got
there. So any of the above tools slot in cleanly.

## How the SonarQube Ingress is exposed

`SonarQubeInstance` has a built-in `ingress` block — the operator creates the
Ingress resource for you, no separate manifest needed. See the `ingress:` section
in [`instance.yaml`](apps/sonarqube/instance.yaml).
