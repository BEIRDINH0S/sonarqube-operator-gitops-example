# sonarqube-operator-gitops-example

End-to-end GitOps example for [sonarqube-operator](https://github.com/BEIRDINH0S/sonarqube-operator):
Argo CD pulls this repo and reconciles a complete SonarQube setup — instance, plugins,
projects, quality gates, users — from declarative YAML.

```
You commit YAML  ──►  Argo CD syncs  ──►  Operator reconciles  ──►  SonarQube state
```

## Prerequisites

- A Kubernetes cluster (kind, minikube, or real)
- [`sonarqube-operator`](https://github.com/BEIRDINH0S/sonarqube-operator) installed
  (e.g. `helm install sonarqube-operator ./charts/sonarqube-operator -n sonarqube-operator-system --create-namespace`)
- [Argo CD](https://argo-cd.readthedocs.io/en/stable/getting_started/) installed in
  the `argocd` namespace

## Bootstrap

One-time, point Argo CD at this repo:

```bash
kubectl apply -f argocd/application.yaml
```

That's it. From now on, every commit to this repo flows to your cluster.

## What's in here

```
apps/sonarqube/
├── 00-namespace.yaml      # the sonarqube namespace
├── 01-postgres.yaml       # demo PostgreSQL backend (replace in prod)
├── 02-secrets.yaml        # placeholder secrets — see "Secrets" below
├── instance.yaml          # SonarQubeInstance — the SonarQube install
├── plugin.yaml            # SonarQubePlugin — install the Git SCM plugin
├── quality-gate.yaml      # SonarQubeQualityGate — coverage > 80%, etc.
├── project.yaml           # SonarQubeProject — uses the quality gate
└── user.yaml              # SonarQubeUser — a CI bot account
```

Each file is intentionally simple and self-contained. Read top to bottom to learn
how the operator's CRDs fit together.

## Secrets

The `02-secrets.yaml` file ships **placeholder credentials** so the example runs out of
the box. **Do not use these values in production.** For real deployments use one of:

- [Sealed Secrets](https://sealed-secrets.netlify.app/) — encrypt secrets in git
- [External Secrets Operator](https://external-secrets.io/) — pull from Vault, AWS SM, etc.
- [SOPS](https://github.com/getsops/sops) — encrypt YAML files in git

The operator only reads `Secret` objects in the cluster — it does not care how they got
there. So any of the above tools slot in cleanly.

## Exposing SonarQube via Ingress

The `SonarQubeInstance` CRD has a built-in `ingress` block (see commented section in
[`instance.yaml`](apps/sonarqube/instance.yaml)). Set `enabled: true`, give it a host
and an `ingressClassName`, and the operator generates the Ingress resource for you —
no separate manifest needed.

For local clusters (kind/minikube), install an ingress controller first:
```bash
kubectl apply -f https://kind.sigs.k8s.io/examples/ingress/deploy-ingress-nginx.yaml
```
