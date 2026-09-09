## Backend GitOps

This repository defines the production Kubernetes workload for the voting worker. ArgoCD discovers the Helm chart under `apps/*` and deploys it to the `backend-prod` namespace through the production ApplicationSet.

The [`worker-app`](https://github.com/YOUR_GITHUB_ORG/worker-app) builds the image and its CD workflow updates the image tag in this repository after a merge to `main`.

## Voting Worker

The `voting-worker` chart deploys:

- Image `${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/voting-worker:<git-sha>`.
- A non-root container running as UID/GID `999` with no exposed HTTP port.
- PostgreSQL and Redis connection ConfigMaps with passwords sourced from Kubernetes Secrets.
- Node affinity for the `backend` team capacity.
- One to five replicas through an HPA with 50 percent CPU and memory targets.
- A PodDisruptionBudget requiring one available replica.

The worker consumes vote messages from the Redis Sentinel `votes` list and writes them to the PostgreSQL service managed by the data GitOps repository. It is an internal background workload and does not define an HTTPRoute or Service for public traffic.

## Validation and Reconciliation

Pull requests run [`.github/workflows/ci-lint.yml`](.github/workflows/ci-lint.yml), which uses Gitleaks, Helm lint and Kubeconform for the `apps` and `bootstrap` directories:

```bash
helm lint -f=prod-values.yml apps/voting-worker
helm template apps/voting-worker -f prod-values.yml | kubeconform -ignore-missing-schemas -strict
```

After merge, ArgoCD watches the `main` branch and reconciles the `backend-prod` application. See the [worker application README](https://github.com/YOUR_GITHUB_ORG/worker-app), the [umbrella GitOps guide](https://github.com/YOUR_GITHUB_ORG/aws-eks-gitops-argocd-terraform/blob/main/GitOps/README.md) and the [data GitOps repository](https://github.com/YOUR_GITHUB_ORG/data-gitops) for the adjacent parts of the delivery path.
