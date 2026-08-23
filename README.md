# demo-app

Spring Boot service, packaged with Docker, deployed to Kubernetes (EKS) via Helm,
built and shipped through GitHub Actions.

## Layout

```
.
├── src/                          # Spring Boot source + tests
├── pom.xml
├── Dockerfile
├── helm/demo-app/                # Helm chart (Deployment + Service)
└── .github/workflows/ci-cd.yaml  # build -> test -> docker push (ECR) -> helm deploy (EKS)
```

## Local run

```bash
mvn spring-boot:run
curl http://localhost:8080/api/hello
```

## Build & run in Docker

```bash
docker build -t demo-app:local .
docker run -p 8080:8080 demo-app:local
```

## Deploy manually with Helm (once you have a kubeconfig pointed at your cluster)

```bash
helm upgrade --install demo-app helm/demo-app \
  --set image.repository=<account_id>.dkr.ecr.<region>.amazonaws.com/demo-app \
  --set image.tag=<tag>
```

## CI/CD pipeline (GitHub Actions)

`.github/workflows/ci-cd.yaml` runs on every push/PR to `main`:

1. **build-test** — compiles the app and runs `mvn test` (runs on PRs too, as a gate).
2. **docker-build-push** — (main branch only) builds the Docker image and pushes it to
   Amazon ECR, tagged with the short git SHA and `latest`.
3. **deploy** — (main branch only, gated behind a GitHub **Environment** called
   `production` so you can require manual approval) updates the kubeconfig for the
   target EKS cluster and runs `helm upgrade --install`, then waits for rollout.

### One-time AWS/GitHub setup

1. **OIDC trust, no static AWS keys in GitHub:**
   - Create an IAM OIDC identity provider for `token.actions.githubusercontent.com` in your AWS account (one-time per account).
   - Create an IAM role that trusts that provider, scoped to your repo (`repo:<org>/<repo>:*` or a specific branch), with permissions for `ecr:*` (push) and `eks:DescribeCluster` + whatever RBAC you map to it in the cluster.
   - Store the role ARN as a repo secret: `AWS_OIDC_ROLE_ARN`.
2. **ECR repository:** create it once (`aws ecr create-repository --repository-name demo-app`), or let your infra-as-code do it.
3. **EKS access for the role:** add the IAM role to the cluster's `aws-auth` ConfigMap (or EKS access entries) with enough RBAC to `helm upgrade`/`kubectl rollout status` in the target namespace.
4. **Environment protection (optional but recommended):** in the repo, go to Settings → Environments → create `production`, add required reviewers. The `deploy` job references `environment: production`, so it will pause for approval before touching the cluster.
5. Update the placeholders at the top of `ci-cd.yaml` (`AWS_REGION`, `ECR_REPOSITORY`, `EKS_CLUSTER_NAME`, namespace, etc).

## Can GitHub Actions control the deployment too?

Yes — that's what the `deploy` job above does: it authenticates to AWS via OIDC,
points `kubectl`/`helm` at the EKS cluster, and runs `helm upgrade --install`.
A few variations depending on how much control/safety you want:

- **Direct push (what's here):** Actions runs `helm upgrade` straight after the image
  push. Simple, but Actions needs cluster credentials.
- **Manual gate:** keep it as one workflow but put the deploy job behind a GitHub
  Environment with required reviewers (shown above) — build/push happens
  automatically, deploy waits for a human click.
- **Separate deploy workflow:** split into `build.yaml` (build/test/push) and
  `deploy.yaml` (`workflow_dispatch`, or triggered by the first workflow via
  `workflow_run`), so you can redeploy an existing image, or deploy to
  staging/prod separately, without rebuilding.
- **GitOps instead:** if you'd rather not give Actions cluster credentials at all,
  have the `docker-build-push` job only update the image tag in a Git repo that
  ArgoCD/Flux watches (a `values.yaml` bump), and let the GitOps controller inside
  the cluster pull and apply the change. More secure (no external cluster access),
  slightly more infrastructure to run.
