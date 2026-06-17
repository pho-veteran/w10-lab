# W10 Lab: Secure & Operate

`w10-lab` là lab demo end-to-end của **Tuần 10**, chủ đề **Secure & Operate**. Nối tiếp
W8 (foundation) và W9 (delivery), tuần này tập trung hardening ở cluster level: chặn vi
phạm ngay tại admission thay vì "tin developer hứa". Cuối tuần, repo dựng lại được một mini
platform end-to-end (GitOps + observability + canary + security) trên fresh cluster trong
dưới 2h, sẵn sàng cho capstone W11-W12.

> Đây là dự án W9 đã rename sang W10. Nó tái dùng toàn bộ nền tảng tuần 9 (minikube trên
> EC2, ArgoCD App-of-Apps, canary, observability) và bồi thêm lớp bảo mật, vận hành của
> tuần 10.

## Mục tiêu cuối tuần

- 3 RBAC role rõ ràng: `developer`, `sre`, `viewer`.
- 4 Gatekeeper constraint enforce ở namespace `demo`.
- ESO rotate secret trong dưới 60s, không cần restart pod.
- Admission từ chối image chưa ký (Cosign + Kyverno verifyImages).
- Resource guardrails (Quota/LimitRange/QoS), NetworkPolicy, runbook, Cost Guard.

## Lịch tuần (15-19/06/2026)

| Ngày | Nội dung | Trạng thái |
|---|---|---|
| **T2 15/06** | D1: RBAC + Admission Policy (OPA/Gatekeeper) | ✅ Đã tích hợp |
| **T3 16/06** | D2: Secrets Rotation (ESO) + Supply Chain (Trivy/Cosign/Kyverno) | 📋 Kế hoạch |
| **T4 17/06** | D3: Platform Integration + Runbook + Cost Guard; Live AWS Security; Test 1 | 📋 Kế hoạch |
| **T5 18/06** | Onsite: Lab "6-risk cluster cleanup + cluster-level enforcement" (full day) | 📋 Kế hoạch |
| **T6 19/06** | Onsite: hoàn thiện Lab, show-and-tell, Test 2 | 📋 Kế hoạch |

---

## Bối cảnh dự án

- **Cluster:** minikube trên 1 EC2 host (dựng bằng Terraform `w10-app/terraform/`).
- **GitOps:** ArgoCD App-of-Apps. AppProject `w10-lab`, repo `pho-veteran/w10-gitops`,
  sync theo wave, `automated: {prune, selfHeal}`.
- **Workload `web`:** Node.js/Express ở namespace `demo`, port 3000, probe `/api/health`,
  chạy bằng Argo Rollout (canary), image `vihn/w10-web`.

> ⚠️ **Khác với tài liệu D1-D3:** các doc giả định `web` là `Deployment` `replicas: 1`
> chưa hardening. Repo này dùng Argo Rollout (canary) và đã để `replicas: 2`, nên các bước
> được chỉnh lại cho khớp: constraint min-replicas match `argoproj.io/Rollout`, còn
> remediation áp vào `rollout.yaml`.

---

## Kế hoạch theo ngày

### ✅ D1: RBAC + Admission Policy (đã tích hợp)

- **RBAC 3 persona** trong ns `demo` (`w10-gitops/platform/rbac/`, App `platform-rbac`):
  - `developer`: CRUD workload, không có secrets/exec/delete-namespace.
  - `sre`: toàn quyền trong ns (secrets, `pods/exec`, RBAC ns-scope), không có cluster-scope.
  - `viewer`: read-only, bind ClusterRole `view`.
- **OPA Gatekeeper** (Helm 3.21.0, App `gatekeeper`) cùng 4 constraint
  (`w10-gitops/platform/gatekeeper/`, App `gatekeeper-policies`), khởi đầu ở `dryrun`:
  `deny-privileged`, `require-requests-limits`, `require-run-as-nonroot`,
  `require-min-2-replicas`.
- **Remediate** rollout `web`: securityContext (runAsNonRoot, drop caps,
  readOnlyRootFilesystem cùng `/tmp` emptyDir) và resource requests/limits.

Nghiệm thu: `kubectl auth can-i --as=...` trả đúng ranh giới từng role; ở `dryrun` audit
phát hiện vi phạm; remediate qua Git thì audit về 0; đổi `dryrun` sang `deny` thì manifest
vi phạm bị chặn tại admission. Runbook chi tiết: [`w10-gitops/platform/README.md`](w10-gitops/platform/README.md).

### 📋 D2: Secrets Rotation + Supply Chain (kế hoạch)

- **ESO (External Secrets Operator)** đọc AWS Secrets Manager rồi ghi K8s Secret
  `web-app-secret` trong `demo`, `refreshInterval: 30s`. Manifest dự kiến ở
  `w10-gitops/platform/eso/` (SecretStore + ExternalSecret), kèm một App ArgoCD.
- **Trivy gate** trong CI (`w10-app/.github/workflows/ci.yml`): fail-on `HIGH,CRITICAL`;
  mỗi ngoại lệ trong `.trivyignore` ghi rõ lý do và ngày hết hạn (ADR có thời hạn). Bonus: secrets scan.
- **Cosign keyless** (OIDC GitHub Actions) ký image theo digest và ghi vào Rekor.
- **Kyverno `verifyImages`** (`w10-gitops/platform/kyverno/`): chạy `Audit` trước rồi
  `Enforce`, từ chối image chưa ký trong `demo`.

Nghiệm thu: secret rotate dưới 60s mà không sửa Git; trình bày được tradeoff giữa env và
mounted volume (no-restart); CI fail khi có CVE HIGH/CRITICAL; image ký keyless có bản ghi
Rekor; admission ở `Enforce` từ chối image chưa ký.

### 📋 D3: Platform Integration + Operations (kế hoạch)

- **ResourceQuota + LimitRange** cho ns `demo`, và nâng `web` lên QoS Guaranteed
  (requests bằng limits). Manifest dự kiến ở `w10-gitops/platform/quotas/`, App
  `platform-quotas` (sync-wave 1).
- **NetworkPolicy** `default-deny-ingress` cùng `allow-web` (cần `minikube start --cni=calico`).
- **Chaos test** pod-delete để kiểm chứng self-heal của ArgoCD (tùy chọn Litmus).
- **Runbook** sự cố `docs/runbooks/web-pod-compromise.md`, 8 mục từ Trigger đến Post-mortem.
- **Cost Guard** (Terraform `w10-app/terraform/modules/cost-guard/`): SNS cùng AWS Cost
  Anomaly Detection, ngưỡng $50/ngày, gửi email.

Nghiệm thu: pod `web` ở `QoS Class: Guaranteed`; deploy vượt trần bị `exceeded quota`;
NetworkPolicy chặn đúng luồng; xóa pod thì self-heal giữ Healthy (replicas=2 nên không
downtime); runbook đủ mục; Cost Guard active.

### 📋 Lab tổng hợp (T5-T6)

Lab "6-risk cluster cleanup + cluster-level enforcement": quét sạch vi phạm và bật
enforcement toàn cluster. Chốt bằng bài kiểm tra mini platform end-to-end dưới 2h: từ fresh
cluster, chạy `infra:apply` rồi `gitops:bootstrap`, đến khi `web` Synced/Healthy với mọi
policy D1+D2+D3 bật đúng thứ tự wave (namespace, project, platform, web).

---

## Cấu trúc thư mục

```text
w10-lab/
├── w10-app/                      # App Node.js/Express + Terraform (infra + bootstrap)
│   ├── app/                      #   source app + Dockerfile (USER node, non-root)
│   ├── bootstrap/terraform/      #   GitHub OIDC + IAM role + S3 state backend
│   ├── terraform/                #   network / security / ec2 / alb  (cost-guard thêm ở D3)
│   ├── .github/workflows/ci.yml  #   CI (Trivy + Cosign thêm ở D2)
│   └── task.sh
└── w10-gitops/                   # Source of truth cho ArgoCD (App-of-Apps)
    ├── apps/
    │   ├── root.yaml             #   root Application
    │   └── children/             #   namespace, project, rbac, gatekeeper, web, monitoring…
    ├── platform/                 # Cluster guardrails
    │   ├── rbac/                 #   ✅ D1: 3 persona
    │   ├── gatekeeper/           #   ✅ D1: ConstraintTemplates + Constraints
    │   ├── eso/                  #   📋 D2: External Secrets
    │   ├── kyverno/              #   📋 D2: verifyImages
    │   └── quotas/               #   📋 D3: Quota/LimitRange/NetworkPolicy
    └── workloads/
        ├── web/                  #   Argo Rollout (canary) + services
        └── web-monitoring/       #   ServiceMonitor + PrometheusRule + Grafana
```

### `task.sh` (wrapper gọi `w10-app/task.sh`)

`bootstrap:validate` · `bootstrap:apply` · `infra:apply` · `infra:destroy` ·
`infra:fmt` · `infra:validate` · `output` · `ssh` · `ssm` ·
`gitops:bootstrap` · `gitops:status` · `gitops:sync` · `argocd:port-forward`

---

## Tài liệu nguồn

- **Yêu cầu tuần:** `xbrain-learners/W10/W10_phase2_announcement_cloud.md`
- **D1:** `cloud/w10/w10-d1-rbac-admission-doc.html` (RBAC + Admission)
- **D2:** `cloud/w10/w10-d2-secrets-supplychain-doc.html` (Secrets + Supply Chain)
- **D3:** `cloud/w10/w10-d3-platform-operations-doc.html` (Platform + Cost Guard)

Tham khảo chính thống: [Kubernetes RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac) ·
[OPA Gatekeeper](https://open-policy-agent.github.io/gatekeeper) ·
[External Secrets Operator](https://external-secrets.io/latest) ·
[Trivy](https://aquasecurity.github.io/trivy) ·
[Cosign / Sigstore](https://docs.sigstore.dev/cosign/overview) ·
[Kyverno](https://kyverno.io/docs) ·
[ResourceQuota](https://kubernetes.io/docs/concepts/policy/resource-quotas) ·
[AWS Cost Anomaly Detection](https://docs.aws.amazon.com/cost-management/latest/userguide/manage-ad.html)
