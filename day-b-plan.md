# W10 Day-B implementation plan — Secrets Rotation + Supply Chain

## Context

Day-A is verified working: RBAC, Gatekeeper, pod hardening, rollout, monitoring. Day-B should extend the same lab into the W10 D2 target from mentor docs:

- AWS Secrets Manager + External Secrets Operator (ESO)
- secret rotation visible in the running app in <60s without pod restart
- Trivy vulnerability + secret scanning in CI
- Cosign signing in CI
- admission-level signature verification with unsigned image rejection
- GitOps-managed platform components and workload config

Chosen decisions:

- ESO auth: **EC2 instance profile**, not static AWS access keys.
- AWS network path: **VPC interface endpoint / PrivateLink for Secrets Manager**.
- Image signature admission engine: **Sigstore policy-controller**, not Kyverno.
- End-state: **audit/prove first, then enforce reject unsigned images**.
- Deliverable: **implement + guide + live apply/verify via SSM**.
- Day-B secret demo target: **real RDS PostgreSQL credentials**, not only a toy `APP_MESSAGE` secret.

## RDS PostgreSQL amendment (added after planning discussion)

Reason: the mentor announcement explicitly frames D2 around DB secrets/rotation. A real private RDS PostgreSQL instance makes the ESO demo concrete: Terraform creates the DB + Secrets Manager source secret; ESO syncs that secret into K8s; the app reads DB credentials from a mounted Secret volume and proves it can connect without committing credentials to Git.

Recommended RDS shape: cheapest practical lab configuration, private-only:

- Engine: PostgreSQL.
- Instance class: `db.t4g.micro` if available in `ap-southeast-1`; fallback `db.t3.micro` if needed.
- Storage: 20 GiB `gp3`, no autoscaling unless needed.
- Multi-AZ: false.
- Public accessibility: false.
- Backup retention: 0 or 1 day; choose 0 for lowest lab cost, 1 if wanting point-in-time recovery demo.
- Deletion protection: false.
- Skip final snapshot: true for disposable lab.
- Performance Insights / enhanced monitoring: off.
- Security group: ingress `5432/tcp` only from the EC2 minikube SG.
- Subnets: private DB subnet group only.

Important Terraform networking change: RDS DB subnet groups require subnets in at least two AZs. The current network module has only one private subnet. Update `modules/network` to create **two private subnets** from `private_subnet_cidrs`, associate both with the private route table, and keep EC2 in the first private subnet. RDS uses both private subnet IDs.

Secret model:

- Create a Secrets Manager secret named `prod/w10-lab/rds` containing JSON:

```json
{
  "host": "<rds-endpoint>",
  "port": "5432",
  "database": "w10app",
  "username": "appuser",
  "password": "<generated-password>"
}
```

- For lab simplicity, Terraform can generate the password via `random_password`, set it as the RDS master/app password, and write the same value into Secrets Manager. Caveat: the password will exist in Terraform state; acceptable for a controlled lab if state is private/encrypted, but document it clearly.
- More production-like alternative: use RDS-managed master password in Secrets Manager (`manage_master_user_password = true`) and output its secret ARN, but GitOps then needs that generated ARN/name wired into the ExternalSecret. More secure, more moving parts. For this lab, fixed secret name `prod/w10-lab/rds` is easier to teach and verify.

App change:

- Add Node `pg` dependency.
- Add a DB credential reader that reads mounted files on every `/api/db/health` request instead of using env vars or a long-lived cached config. This preserves the Day-B no-restart rotation story.
- Mount synced K8s Secret at `/etc/db-secret` with keys: `host`, `port`, `database`, `username`, `password`.
- Add endpoint:

```http
GET /api/db/health
```

Expected JSON:

```json
{
  "ok": true,
  "dbConnected": true,
  "database": "w10app",
  "user": "appuser",
  "host": "<rds-endpoint>",
  "serverTime": "...",
  "secretSource": "/etc/db-secret"
}
```

Implementation detail: create a short-lived `pg.Client` per check (`connect -> SELECT now(), current_database(), current_user -> end`) so password file changes are picked up without pod restart. Do not create a process-wide pool for the lab demo unless also adding file-watch/reload logic.

ESO change:

- Replace/supplement the old `web-app-secret` / `APP_MESSAGE` plan with `web-db-secret` sourced from `prod/w10-lab/rds`.
- ExternalSecret writes a Kubernetes Secret in `demo` with keys:
  - `host`
  - `port`
  - `database`
  - `username`
  - `password`
- `refreshInterval: 30s`.

Rotation demo options:

1. Basic ESO sync proof: update only a non-auth field in `prod/w10-lab/rds` and prove K8s Secret/app sees it in <60s.
2. Real DB credential rotation proof: generate a new password, run `aws rds modify-db-instance --master-user-password <new> --apply-immediately`, then update Secrets Manager `prod/w10-lab/rds` with the same new password. ESO syncs; `/api/db/health` reconnects successfully without pod restart.
3. Full production rotation Lambda: out of scope for first Day-B implementation unless time permits; mention in guide as next step.

Acceptance additions:

- RDS PostgreSQL exists in private subnets only and is not publicly accessible.
- RDS SG allows 5432 only from EC2/minikube SG.
- Secrets Manager secret `prod/w10-lab/rds` contains DB connection JSON.
- ESO syncs `prod/w10-lab/rds` into `demo/web-db-secret`.
- Web Rollout mounts `web-db-secret` read-only at `/etc/db-secret`.
- `/api/db/health` returns `dbConnected: true`.
- Updating the Secrets Manager value propagates to the mounted secret/app check in <60s without pod restart.
- If doing real password rotation, old pod name/restart count stays unchanged and `/api/db/health` still passes after RDS password update + secret sync.

Cost warning: RDS adds ongoing hourly + storage cost. Destroy after lab, or add `enable_rds = false` variable defaulting false if we want the stack to be cheap by default and only enable RDS for Day-B.

## Current repo facts

### App repo: `w10-app`

Key files:

- `w10-app/.github/workflows/ci.yml`
  - Current jobs: `changes`, `test`, `plan`, `publish`, `apply`.
  - Current publish flow: build/push Docker image to Docker Hub, then update `pho-veteran/w10-gitops` image tag via kustomize.
  - Current publish job only has `permissions.contents: read`; Cosign keyless will need `id-token: write`.
  - Current build step uses `docker/build-push-action@v6`, tags only, no digest output currently consumed.

- `w10-app/app/server.js`
  - Express app.
  - Health: `/api/health`.
  - Metrics: `/metrics`.
  - Debug infra: `/api/debug/infra`.
  - Good place to add `/api/debug/secret` and runtime secret-file read helper.

- `w10-app/app/Dockerfile`
  - `node:20-alpine`.
  - Ends with `USER node`.
  - Day-A k8s pod uses numeric `runAsUser: 1000`, already verified.

- `w10-app/terraform/modules/ec2/main.tf`
  - EC2 is private: `associate_public_ip_address = false`.
  - Uses instance profile from `aws_iam_instance_profile.ssm`.
  - Existing IAM role only has `AmazonSSMManagedInstanceCore`.
  - Need Secrets Manager scoped policy.
  - Need `metadata_options` for IMDSv2 and pod access:
    - `http_tokens = "required"`
    - `http_put_response_hop_limit = 2`

- `w10-app/terraform/modules/network/main.tf`
  - VPC has public and private subnet.
  - Private route currently egresses via NAT Gateway.
  - Need Interface VPC Endpoint for Secrets Manager in private subnet, private DNS enabled.

- `w10-app/terraform/modules/security/main.tf`
  - EC2 SG allows all outbound currently.
  - Need endpoint SG allowing TCP/443 from EC2 SG.

### GitOps repo: `w10-gitops`

Key files:

- `w10-gitops/apps/children/kustomization.yaml`
  - App-of-apps children list.
  - Need add ESO and Sigstore policy-controller applications.

- `w10-gitops/apps/children/project.yaml`
  - Current source repos include GitOps repo, Argo Rollouts, prometheus, Gatekeeper.
  - Need add Helm/release source(s) for ESO and Sigstore policy-controller.
  - Need add destination namespace(s): `external-secrets`, `cosign-system` or actual Sigstore namespace.

- Existing sync-wave pattern:
  - `project`: wave 0
  - platform controllers (Gatekeeper, Rollouts, monitoring/RBAC): wave 1
  - web workload: wave 2
  - Gatekeeper policies: wave 4

- `w10-gitops/workloads/web/base/rollout.yaml`
  - Need mount secret volume from ESO-created Secret, probably `web-app-secret`.
  - Add env path variable e.g. `APP_MESSAGE_PATH=/etc/web-secret/APP_MESSAGE`.
  - Preserve readOnlyRootFilesystem and existing `/tmp` emptyDir.
  - Add new `app-secret` readOnly secret volume.

## Recommended implementation

### 1. Terraform: private Secrets Manager access + instance-profile auth

Modify Terraform so ESO pods can use the EC2 instance profile and reach Secrets Manager privately.

Files likely modified:

- `w10-app/terraform/modules/ec2/main.tf`
- `w10-app/terraform/modules/ec2/variables.tf`
- `w10-app/terraform/modules/security/main.tf`
- `w10-app/terraform/modules/security/outputs.tf` if needed
- `w10-app/terraform/modules/network/main.tf`
- `w10-app/terraform/modules/network/variables.tf` if endpoint config added as vars
- `w10-app/terraform/main.tf`
- `w10-app/terraform/outputs.tf` optional

Concrete changes:

1. Add `metadata_options` to EC2:

```hcl
metadata_options {
  http_endpoint               = "enabled"
  http_tokens                 = "required"
  http_put_response_hop_limit = 2
}
```

2. Attach inline IAM policy to the EC2 role allowing only required Secrets Manager reads:

```hcl
action = [
  "secretsmanager:GetSecretValue",
  "secretsmanager:DescribeSecret"
]
resource = "arn:aws:secretsmanager:${region}:${account_id}:secret:prod/w10-lab/app-*"
```

Need region/account from variables/data sources. Could use `data.aws_caller_identity.current` and `data.aws_region.current` in module/root, or pass region/account into module.

3. Add Interface VPC Endpoint for Secrets Manager:

```hcl
resource "aws_vpc_endpoint" "secretsmanager" {
  vpc_id              = aws_vpc.this.id
  service_name        = "com.amazonaws.${var.aws_region}.secretsmanager"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = [aws_subnet.private.id]
  private_dns_enabled = true
  security_group_ids  = [var.secretsmanager_endpoint_security_group_id]
}
```

But avoid module cycle. Better approach:

- Put endpoint SG inside `modules/security` because it needs `vpc_id` and `ec2` SG reference.
- Output `secretsmanager_endpoint_security_group_id` from security.
- Pass it to `modules/network`, where the endpoint has access to private subnet ID and VPC ID.

Security rule:

```hcl
resource "aws_security_group" "secretsmanager_endpoint" { ... }

resource "aws_security_group_rule" "secretsmanager_endpoint_from_ec2" {
  type                     = "ingress"
  security_group_id        = aws_security_group.secretsmanager_endpoint.id
  from_port                = 443
  to_port                  = 443
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.ec2.id
}
```

4. Keep NAT Gateway for now because Docker pulls/GitHub/Helm still need outbound internet. Endpoint only removes SM dependency on NAT.

### 2. App: runtime secret read endpoint

Modify `w10-app/app/server.js`.

Add a small helper:

```js
const fs = require("fs");

function readRuntimeSecret(filePath) {
  try {
    return { found: true, value: fs.readFileSync(filePath, "utf8").trim() };
  } catch (error) {
    if (error.code === "ENOENT") return { found: false };
    return { found: false, error: error.message };
  }
}
```

Add env/path near existing env vars:

```js
const appMessagePath = process.env.APP_MESSAGE_PATH || "/etc/web-secret/APP_MESSAGE";
```

Add endpoint:

```js
app.get("/api/debug/secret", (_req, res) => {
  const secret = readRuntimeSecret(appMessagePath);
  res.json({
    path: appMessagePath,
    found: secret.found,
    preview: secret.found ? `${secret.value.slice(0, 8)}...` : null,
    value: process.env.EXPOSE_SECRET_DEBUG === "true" && secret.found ? secret.value : undefined,
    timestamp: new Date().toISOString()
  });
});
```

Better for lab verification: include full value only when `EXPOSE_SECRET_DEBUG=true`; otherwise preview only. In lab overlay can set `EXPOSE_SECRET_DEBUG=true` to prove rotation.

Update tests in `w10-app/app/server.test.js` if needed to cover missing secret path and debug route behavior.

### 3. CI: Trivy + Cosign keyless

Modify `w10-app/.github/workflows/ci.yml`.

Changes:

1. Add security scan in `test` job:

```yaml
- name: Scan repo for secrets
  uses: aquasecurity/trivy-action@0.24.0
  with:
    scan-type: fs
    scanners: secret
    exit-code: '1'
```

Maybe restrict to app path or repo root. Beware false positives from Terraform state or generated private key. Existing repo contains `terraform/.generated/p2-w10-lab-lab.pem` and tfstate files. Trivy secret scan over repo root could fail. Prefer scan `app/` and workflow files, or add ignore/skip paths. Need avoid scanning committed local state/private key if present.

Suggested:

```yaml
with:
  scan-type: fs
  scan-ref: .
  scanners: secret
  skip-dirs: w10-app/terraform/.generated,w10-app/terraform/.terraform,w10-app/bootstrap/terraform/.terraform
```

But workflow runs in `w10-app` repo, so paths are `terraform/.generated`, `terraform/.terraform`, `bootstrap/terraform/.terraform`. Also tfstate may include sensitive values; ideally repo should not contain tfstate in remote, but local workspace has it. CI only scans committed files.

2. In `publish` job permissions, add:

```yaml
permissions:
  contents: read
  id-token: write
```

3. Ensure build step has `id: build` and output digest:

```yaml
- name: Build and push image
  id: build
  uses: docker/build-push-action@v6
  with:
    context: ./app
    push: true
    tags: ${{ steps.meta.outputs.image }}
```

4. Add Trivy image scan gate after build/push, before GitOps update/sign? Mentor doc says build/push → Trivy → sign → update GitOps. More secure would scan before push using local image; current `build-push-action` pushes immediately. Minimal change: build/push then scan remote image, fail before GitOps promotion/sign. Since image may exist unsigned if scan fails, but not deployed.

```yaml
- name: Scan image with Trivy
  uses: aquasecurity/trivy-action@0.24.0
  with:
    image-ref: ${{ steps.meta.outputs.image }}
    format: table
    exit-code: '1'
    severity: HIGH,CRITICAL
    ignore-unfixed: true
    trivyignores: .trivyignore
```

5. Install Cosign and sign by digest:

```yaml
- name: Install cosign
  uses: sigstore/cosign-installer@v3

- name: Sign image (keyless)
  env:
    IMAGE: ${{ secrets.DOCKERHUB_USERNAME }}/${{ env.IMAGE_REPOSITORY }}
    DIGEST: ${{ steps.build.outputs.digest }}
  run: cosign sign --yes "${IMAGE}@${DIGEST}"
```

Need Docker Hub registry path for Sigstore policy-controller later: use digest and image `docker.io/vihn/w10-web@sha256:...`.

6. Optionally update GitOps image to digest rather than tag for immutable deploy:

- Current kustomize edit sets tag only: `w10-web=vihn/w10-web:main-xxxx`.
- Signature verification works best with digest. Sigstore admission can resolve tag to digest, but digest pinning is more deterministic.
- Recommended: keep tag for readability but sign digest; policy-controller can verify resolved digest. If tooling issues appear, switch GitOps to digest pinning.

7. Add `.trivyignore` with template comments, no active fake CVE unless needed.

### 4. GitOps: ESO controller and ExternalSecret

Add GitOps-managed ESO controller and app-specific secret sync.

Files likely added:

- `w10-gitops/apps/children/external-secrets.yaml`
- `w10-gitops/apps/children/eso-secrets.yaml` or app-specific `web-secrets.yaml`
- `w10-gitops/platform/eso/kustomization.yaml`
- `w10-gitops/platform/eso/secretstore.yaml`
- `w10-gitops/platform/eso/externalsecret.yaml`

Controller Application example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: external-secrets
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  project: w10-lab
  source:
    repoURL: https://charts.external-secrets.io
    chart: external-secrets
    targetRevision: <pinned-version>
    helm:
      releaseName: external-secrets
      valuesObject:
        installCRDs: true
  destination:
    server: https://kubernetes.default.svc
    namespace: external-secrets
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

Need choose stable ESO chart version. Check docs/latest before implementation if internet available.

SecretStore:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-sm
  namespace: demo
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-southeast-1
      # No auth block: use AWS SDK default credential chain -> EC2 instance profile.
```

ExternalSecret:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: web-app-secret
  namespace: demo
spec:
  refreshInterval: 30s
  secretStoreRef:
    name: aws-sm
    kind: SecretStore
  target:
    name: web-app-secret
    creationPolicy: Owner
  data:
    - secretKey: APP_MESSAGE
      remoteRef:
        key: prod/w10-lab/app
        property: APP_MESSAGE
```

Need add app-of-apps child app for `platform/eso` after ESO controller CRDs are installed. Use sync wave 2 or 3; web should depend on the secret existing. Current web is wave 2. To avoid race:

- external-secrets controller: wave 1
- platform eso secretstore/externalsecret: wave 2
- web workload: wave 3
- gatekeeper policies: wave 4 unchanged or web can stay wave 2 with ExternalSecret wave 2; safer to move web to wave 3.

### 5. GitOps: mount ESO secret into web Rollout

Modify `w10-gitops/workloads/web/base/rollout.yaml`.

Add env:

```yaml
- name: APP_MESSAGE_PATH
  value: /etc/web-secret/APP_MESSAGE
- name: EXPOSE_SECRET_DEBUG
  value: "true"
```

Add volumeMount:

```yaml
- name: app-secret
  mountPath: /etc/web-secret
  readOnly: true
```

Add volume:

```yaml
- name: app-secret
  secret:
    secretName: web-app-secret
    optional: false
```

Potential race: if the Secret does not exist yet, pod creation waits/fails until ESO syncs it. That's acceptable if sync order is correct; otherwise Argo may show transient error. Could set `optional: true` during first sync then later false, but acceptance wants real secret. Recommended: false with sync-wave ordering.

### 6. GitOps: Sigstore policy-controller + ClusterImagePolicy

Add Sigstore policy-controller as ArgoCD app.

Files likely added:

- `w10-gitops/apps/children/sigstore-policy-controller.yaml`
- `w10-gitops/apps/children/sigstore-policies.yaml` maybe
- `w10-gitops/platform/sigstore/kustomization.yaml`
- `w10-gitops/platform/sigstore/clusterimagepolicy.yaml`

Need exact install mechanism. Mentor slides show:

```bash
kubectl apply -f https://github.com/sigstore/policy-controller/releases/download/v0.8.0/release.yaml
```

For GitOps, better ArgoCD Application source may point directly to a GitHub release URL is not normal. Options:

1. Vendor the release YAML into `w10-gitops/platform/sigstore/controller/` and apply through Kustomize.
2. Use Helm chart if available.
3. Use remote Kustomize URL if release supports it.

Recommended for lab reliability: vendor a known release YAML or use official manifest under a directory, but this is many files. Alternative: create ArgoCD app with `repoURL: https://github.com/sigstore/policy-controller`, `targetRevision: <tag>`, `path: config/default` if repo supports it. Need verify during implementation.

Policy shape (approx; must verify API version/schema during implementation):

```yaml
apiVersion: policy.sigstore.dev/v1beta1
kind: ClusterImagePolicy
metadata:
  name: require-signed-w10-web
spec:
  images:
    - glob: "index.docker.io/vihn/w10-web**"
  authorities:
    - keyless:
        identities:
          - issuer: "https://token.actions.githubusercontent.com"
            subjectRegExp: "https://github.com/pho-veteran/w10-app/.github/workflows/ci.yml@refs/heads/main"
```

Need confirm exact keyless identity fields for selected policy-controller version. If `subjectRegExp` unsupported, use exact `subject` or `subjectAlternativeName` depending version.

Audit/prove then Enforce:

- Sigstore policy-controller's enforcement/audit semantics differ from Kyverno. Need verify whether it supports warn/audit mode via policy setting or namespace label/config.
- If no native audit mode, use staged rollout:
  1. Install controller with policy scoped narrowly or in non-enforcing namespace/test namespace.
  2. Verify signed image via `cosign verify` and test an unsigned image in a temporary namespace or after policy apply.
  3. Apply enforcing ClusterImagePolicy to `demo`.

Need research exact policy-controller dry-run/audit support before implementation.

### 7. ArgoCD project/source updates

Modify:

- `w10-gitops/apps/children/project.yaml`
- `w10-gitops/apps/children/kustomization.yaml`

Add source repos:

- `https://charts.external-secrets.io`
- Sigstore controller source repo/chart URL once selected.

Add destinations:

- `external-secrets`
- `cosign-system` or `sigstore-system` depending controller install namespace.

Add children resources:

- `external-secrets.yaml`
- `platform-eso.yaml`
- `sigstore-policy-controller.yaml`
- `sigstore-policies.yaml`

### 8. AWS secret creation

Need create/update source secret:

```bash
aws secretsmanager create-secret \
  --name prod/w10-lab/app \
  --secret-string '{"APP_MESSAGE":"hello from ESO v1"}' \
  --region ap-southeast-1
```

If exists, use update/put-secret-value:

```bash
aws secretsmanager put-secret-value \
  --secret-id prod/w10-lab/app \
  --secret-string '{"APP_MESSAGE":"hello from ESO v2"}' \
  --region ap-southeast-1
```

The IAM policy ARN wildcard must match Secrets Manager suffix, e.g. `secret:prod/w10-lab/app-*`.

### 9. Documentation guide

Add `day-b-guide.html` at repo root, similar role to `day-a-guide.html`.

Content sections:

1. Day-B goal and architecture
2. Why EC2 instance profile instead of static access keys
3. Why IMDSv2 hop-limit=2 is needed for pods in minikube
4. Why VPC endpoint for Secrets Manager
5. ESO SecretStore/ExternalSecret flow
6. Secret volume vs env var; no-restart condition
7. Trivy CI gate + `.trivyignore` with expiring ADR
8. Cosign keyless signing with GitHub OIDC
9. Sigstore policy-controller admission verification
10. Audit/prove then enforce workflow
11. Verification commands via SSM
12. Troubleshooting:
    - ESO cannot get credentials -> check IMDS hop limit, IAM policy, controller logs
    - Secret not syncing -> check SecretStore/ExternalSecret status
    - signature reject false negative -> verify subject/issuer/image glob/digest
    - app not seeing rotation -> ensure mounted volume + runtime file read, not env

### 10. Live apply/verify flow

After repo edits and tests:

1. Local checks:

```bash
cd w10-app/app && npm test && npm run build
cd ../terraform && terraform fmt -check -recursive && terraform validate
cd ../../w10-gitops && kubectl kustomize apps/children
```

2. Terraform apply for:

- Secrets Manager endpoint
- EC2 role policy
- IMDS metadata options

```bash
cd w10-app/terraform
terraform init ...
terraform plan
terraform apply -auto-approve
```

Note: changing metadata options should not replace EC2. Verify plan carefully.

3. Create/update AWS secret.

4. Push/trigger app CI to build, scan, sign, push image, update GitOps.

5. ArgoCD sync.

6. SSM checks:

Use established SSM pattern from Day-A:

- write JSON params file in terraform dir
- use relative `file://ssm-*.json`
- remote script starts with `export KUBECONFIG=/home/ubuntu/.kube/config`

Verify:

```bash
kubectl get pods -n external-secrets
kubectl get secretstore,externalsecret -n demo
kubectl get secret web-app-secret -n demo -o jsonpath='{.data.APP_MESSAGE}' | base64 -d
kubectl exec -n demo deploy/...? -- cat /etc/web-secret/APP_MESSAGE
curl http://localhost:30080/api/debug/secret
```

For Rollout pods use labels/selectors rather than Deployment because app is Argo Rollouts.

Rotation test:

1. Get current app pod name and restart count.
2. Put new secret version in AWS with `APP_MESSAGE=hello from ESO v2`.
3. Poll K8s Secret and app endpoint until value changes, target <60s.
4. Confirm pod name/restart count unchanged.

Signature test:

1. Verify signed image with cosign if cosign installed in runner/remote, or rely on policy-controller.
2. Try unsigned pod in `demo`:

```bash
kubectl -n demo run unsigned-test --image=nginx:latest --restart=Never
```

Expected: rejected by admission once enforce policy active.

3. Confirm signed web pods admitted/running.

4. Check policy-controller logs/events for proof.

### 11. Risks / known caveats

- Sigstore policy-controller exact CRD/schema and audit/enforce support must be verified during implementation; mentor slides use it, but examples vary by version.
- Docker Hub image path matching may require `index.docker.io/vihn/w10-web` vs `docker.io/vihn/w10-web` vs `vihn/w10-web`. Policy glob must match what admission sees.
- Keyless identity subject must match GitHub Actions workflow exactly. Likely:
  - issuer: `https://token.actions.githubusercontent.com`
  - subject: `https://github.com/pho-veteran/w10-app/.github/workflows/ci.yml@refs/heads/main`
- Existing local repo includes tfstate/private key artifacts. Secret scanning in CI should avoid false positives or ensure those are not committed remotely.
- Gatekeeper constraints currently require resource requests/limits and non-root; new controller pods from Helm/manifests may need resources/security contexts or namespace exclusions depending how Gatekeeper constraints match. Current Gatekeeper constraints only target namespace `demo`, so platform namespaces should be unaffected.
- `web-app-secret` missing can block web pods. Sync-wave ordering should install ESO and ExternalSecret before web.
- VPC endpoint adds hourly cost. NAT already exists; endpoint is for private SM path/security, not cost removal.

## Suggested commit sequence

No Claude co-author trailer per memory.

1. `feat(terraform): add private secrets manager access for day-b`
2. `feat(app): read rotated secret from mounted volume`
3. `ci: add trivy scan and cosign keyless signing`
4. `feat(gitops): add eso and sigstore policy controller`
5. `docs: add day-b secrets and supply-chain guide`

## Acceptance criteria

- Terraform creates/updates without replacing EC2 unless expected.
- EC2 role can read only `prod/w10-lab/app` secret.
- Secrets Manager VPC endpoint exists, private DNS enabled, SG allows 443 from EC2 SG.
- ESO controller is Healthy/Synced via ArgoCD.
- `SecretStore` and `ExternalSecret` show Ready/SecretSynced.
- `web-app-secret` appears in `demo`.
- Web pod mounts `/etc/web-secret/APP_MESSAGE`.
- App endpoint shows secret value/preview.
- Updating AWS Secrets Manager updates K8s Secret and app endpoint in <60s without pod restart.
- CI scans image with Trivy and signs image via Cosign keyless using GitHub OIDC.
- Sigstore policy-controller rejects an unsigned image in `demo`.
- Signed `vihn/w10-web` image from CI is admitted and rollout remains Healthy.
- All ArgoCD apps Synced/Healthy after Day-B.
