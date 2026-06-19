# GIẢI THÍCH CHI TIẾT LAB 1 & LAB 2

> **RBAC + OPA Gatekeeper + ESO + Trivy + Cosign + Sigstore**
> Dựa trên code thực tế repo: `https://github.com/bearh141/Lab1806`
> Phiên bản: 2025 | Dùng để ôn thi vấn đáp

---

## MỤC LỤC

1. [Tổng quan hệ thống và sơ đồ ASCII](#1-tổng-quan-hệ-thống)
2. [Thứ tự tạo file và Sync-Wave](#2-thứ-tự-tạo-file-và-sync-wave)
3. [Lab 1.1 – RBAC (Role-Based Access Control)](#3-lab-11--rbac)
4. [Lab 1.2 – Gatekeeper 4 Constraints từ Library](#4-lab-12--gatekeeper-4-constraints)
5. [Lab 1.3 – Custom Policy: Owner Label](#5-lab-13--custom-policy-owner-label)
6. [Lab 2.1 – ESO (External Secrets Operator)](#6-lab-21--eso-external-secrets-operator)
7. [Lab 2.2 – Trivy + Cosign + Sigstore (Supply Chain Security)](#7-lab-22--trivy--cosign--sigstore)
8. [Bộ câu hỏi vấn đáp (15 câu)](#8-bộ-câu-hỏi-vấn-đáp)

---

## 1. TỔNG QUAN HỆ THỐNG

### 1.1 Kiến trúc tổng thể

Hệ thống Lab1806 được xây dựng trên **App-of-Apps pattern** của ArgoCD. Một Application "root" quản lý tất cả child applications, mỗi child application quản lý một tầng của platform.

### 1.2 Sơ đồ ASCII – Luồng hoạt động tổng thể

```
┌─────────────────────────────────────────────────────────────────┐
│                    GITHUB REPOSITORY                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │   rbac/      │  │  gatekeeper/ │  │  eso/ policies/      │  │
│  │  roles.yaml  │  │  templates/  │  │  secret-store.yaml   │  │
│  │  bindings.yaml│  │  constraints/│  │  external-secret.yaml│  │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘  │
│         │                  │                      │             │
│         └──────────────────┼──────────────────────┘             │
│                            │                                    │
│                   argocd/apps/*.yaml                            │
│                   argocd/root.yaml                              │
└─────────────────────────────┬───────────────────────────────────┘
                              │  Git Pull (every 3 min)
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     ARGOCD (sync-wave)                          │
│                                                                 │
│  Wave -1: rbac.yaml ──────────────────► K8s RBAC API           │
│           ↓ Tạo Role/ClusterRole/Binding trước tiên             │
│                                                                 │
│  Wave 0:  gatekeeper.yaml ────────────► Helm Install           │
│           eso.yaml ───────────────────► Helm Install           │
│           policy-controller.yaml ─────► Helm Install           │
│           ↓ Cài Admission Webhooks                              │
│                                                                 │
│  Wave 1:  gatekeeper-policies.yaml ───► CRD Templates+Constraints│
│           eso-config.yaml ────────────► SecretStore+ExternalSecret│
│           policies.yaml ──────────────► ClusterImagePolicy     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              KUBERNETES CLUSTER (runtime)                       │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  ADMISSION CONTROL PIPELINE (mọi kubectl apply đều qua) │  │
│  │                                                          │  │
│  │  kubectl apply ──► API Server ──► AuthN ──► AuthZ(RBAC) │  │
│  │                                       │                  │  │
│  │                    Mutating Webhooks ◄─┘                  │  │
│  │                    (cosign policy-controller inject)      │  │
│  │                           │                              │  │
│  │                    Validating Webhooks                   │  │
│  │                    ┌──────┴──────────────┐               │  │
│  │                    │  OPA Gatekeeper     │               │  │
│  │                    │  - disallow-latest  │               │  │
│  │                    │  - require-limits   │               │  │
│  │                    │  - no-root          │               │  │
│  │                    │  - no-hostnet       │               │  │
│  │                    │  - require-owner    │               │  │
│  │                    └──────┬──────────────┘               │  │
│  │                    ┌──────┴──────────────┐               │  │
│  │                    │  Sigstore Policy    │               │  │
│  │                    │  Controller         │               │  │
│  │                    │  - verify cosign    │               │  │
│  │                    │  signature          │               │  │
│  │                    └─────────────────────┘               │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌─────────────────┐    ┌──────────────────────────────────┐   │
│  │  namespace: demo│    │  AWS Secrets Manager             │   │
│  │  ┌───────────┐  │    │  key: w10/demo/db-password       │   │
│  │  │ ESO       │◄─┼────┤  ↓ (pull every 10s)             │   │
│  │  │ Controller│  │    └──────────────────────────────────┘   │
│  │  └─────┬─────┘  │                                           │
│  │        │        │                                           │
│  │  ┌─────▼──────┐ │                                           │
│  │  │ db-secret  │ │  ← K8s Secret (managed by ESO)           │
│  │  │ (K8s Secret│ │                                           │
│  │  └────────────┘ │                                           │
│  └─────────────────┘                                           │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│               GITHUB ACTIONS CI/CD PIPELINE                     │
│                                                                 │
│  git push src/api/ ──► Workflow trigger                         │
│         │                                                       │
│         ├─1─► docker build + push ──► ghcr.io/bearh141/w10-api  │
│         │                                                       │
│         ├─2─► Trivy scan (HIGH/CRITICAL CVE)                    │
│         │     ↓ Nếu tìm thấy lỗ hổng → exit code 1 → FAIL     │
│         │                                                       │
│         ├─3─► Cosign sign (private key từ GitHub Secrets)       │
│         │     ↓ Ghi chữ ký vào OCI registry (cosign.sig tag)   │
│         │                                                       │
│         └─4─► Update rollout.yaml với version mới               │
│               ↓ ArgoCD pick up + deploy                         │
│               ↓ Sigstore Policy Controller verify chữ ký        │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3 Giải thích luồng end-to-end

**Bước 1 – Developer push code:**
Developer push code vào branch `main`, GitHub Actions workflow `build-push.yml` được kích hoạt.

**Bước 2 – Build & Scan:**
Docker image được build từ `src/api/Dockerfile` (Flask app Python), push lên `ghcr.io/bearh141/w10-api-141`. Trivy quét CVE mức HIGH/CRITICAL, nếu tìm thấy lỗ hổng chưa có patch → pipeline fail, image không được deploy.

**Bước 3 – Sign:**
Cosign ký image bằng private key lưu trong GitHub Secrets. Chữ ký được lưu vào cùng OCI registry dưới dạng tag đặc biệt (`.sig`).

**Bước 4 – ArgoCD sync:**
ArgoCD (theo dõi repo mỗi 3 phút hoặc webhook) phát hiện thay đổi trong `rollout.yaml`, bắt đầu sync theo sync-wave order.

**Bước 5 – Admission Control:**
Khi workload được apply vào cluster, API Server gọi các Admission Webhook theo thứ tự: Mutating (cosign policy-controller) → Validating (Gatekeeper + policy-controller). RBAC được kiểm tra trước đó (AuthZ layer).

**Bước 6 – ESO pull secrets:**
ESO Controller trong namespace `demo` định kỳ (mỗi 10 giây) pull secret từ AWS Secrets Manager và tạo/cập nhật K8s Secret `db-secret`. Pod mount secret này qua volume hoặc env var.

---

## 2. THỨ TỰ TẠO FILE VÀ SYNC-WAVE

### 2.1 Khái niệm Sync-Wave

**Sync-wave** là cơ chế của ArgoCD cho phép kiểm soát thứ tự sync các resource. Resource với wave thấp hơn được deploy trước, ArgoCD đợi cho đến khi chúng `Healthy` rồi mới chuyển sang wave tiếp theo.

Annotation sử dụng:
```yaml
annotations:
  argocd.argoproj.io/sync-wave: "-1"   # âm = chạy đầu tiên
  argocd.argoproj.io/sync-wave: "0"    # mặc định nếu không có annotation
  argocd.argoproj.io/sync-wave: "1"    # chạy cuối
```

### 2.2 Thứ tự sync trong repo này

```
Wave -1  │  rbac.yaml
         │  ↳ Tạo Role, ClusterRole, RoleBinding TRƯỚC
         │  ↳ Lý do: RBAC là nền tảng authn/authz, phải có trước khi
         │    bất kỳ service account hay user nào tương tác với cluster
         │
Wave 0   │  gatekeeper.yaml       (Helm chart OPA Gatekeeper)
         │  eso.yaml              (Helm chart External Secrets Operator)
         │  policy-controller.yaml(Helm chart Sigstore Policy Controller)
         │  ↳ Tất cả là Helm chart, cài đặt CRDs + Admission Webhooks
         │  ↳ Phải cài trước khi tạo các custom resource của chúng
         │    (ConstraintTemplate, SecretStore, ClusterImagePolicy)
         │
Wave 1   │  gatekeeper-policies.yaml  (ConstraintTemplates + Constraints)
         │  eso-config.yaml           (SecretStore + ExternalSecret)
         │  policies.yaml             (ClusterImagePolicy)
         │  ↳ Phụ thuộc vào CRDs đã được tạo ở wave 0
```

### 2.3 Điều gì xảy ra nếu sai thứ tự?

| Sai thứ tự | Lỗi xảy ra |
|---|---|
| Deploy `gatekeeper-policies` trước khi cài Gatekeeper | `no matches for kind "ConstraintTemplate"` – CRD chưa tồn tại |
| Deploy `eso-config` trước khi cài ESO | `no matches for kind "SecretStore"` – CRD chưa tồn tại |
| Deploy `ClusterImagePolicy` trước khi cài policy-controller | `no matches for kind "ClusterImagePolicy"` |
| Bỏ `SkipDryRunOnMissingResource=true` trong gatekeeper-policies | ArgoCD dry-run fail vì CRD chưa registered trong API server |

**Chi tiết lỗi thực tế khi CRD chưa tồn tại:**
```
error: unable to recognize "gatekeeper/constraints/disallow-latest-tag.yaml":
no matches for kind "K8sDisallowedTags" in version "constraints.gatekeeper.sh/v1beta1"
```

### 2.4 Tại sao `SkipDryRunOnMissingResource=true` quan trọng?

ArgoCD mặc định thực hiện dry-run trước khi apply. Nếu CRD của Gatekeeper (ConstraintTemplate, Constraint) chưa được tạo trong cluster (vì Gatekeeper đang cài ở wave 0), dry-run sẽ fail vì API server không biết resource đó là gì.

`SkipDryRunOnMissingResource=true` nói với ArgoCD: "Nếu resource kind không tồn tại trong cluster, bỏ qua dry-run và apply thẳng" – điều này an toàn vì Gatekeeper đã được cài ở wave 0.

### 2.5 Tại sao `ServerSideApply=true` trong rbac.yaml và gatekeeper-policies.yaml?

`ServerSideApply` chuyển logic merge từ client (kubectl) sang API Server. Lợi ích:
- Tránh conflict khi nhiều controller cùng manage một field
- ArgoCD có thể track field ownership chính xác hơn
- Cần thiết khi apply CRD có nhiều field phức tạp (như Gatekeeper ConstraintTemplate)

---

## 3. LAB 1.1 – RBAC (Role-Based Access Control)

### 3.1 Khái niệm cốt lõi

RBAC (Role-Based Access Control) là hệ thống phân quyền của Kubernetes. Nó kiểm soát **ai** (subject) được phép làm **gì** (verb) với **resource** nào trong **namespace** nào.

4 thành phần chính:
- **Role** – tập hợp quyền, giới hạn trong 1 namespace cụ thể
- **ClusterRole** – tập hợp quyền, áp dụng toàn cluster hoặc non-namespaced resources
- **RoleBinding** – gán Role hoặc ClusterRole cho subject trong 1 namespace
- **ClusterRoleBinding** – gán ClusterRole cho subject ở cấp cluster

### 3.2 File: `rbac/roles.yaml` – Phân tích từng dòng

#### Block 1: Role `developer-workload-manager` (cho Alice)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
# ^ API group của RBAC trong K8s. Luôn dùng v1 (stable)

kind: Role
# ^ "Role" = chỉ có hiệu lực trong 1 namespace cụ thể (demo)
# ^ Khác với ClusterRole: Role không thể cấp quyền trên toàn cluster

metadata:
  name: developer-workload-manager
  # ^ Tên role, được tham chiếu trong RoleBinding
  namespace: demo
  # ^ Role này CHỈ tồn tại và có hiệu lực trong namespace "demo"
  # ^ Nếu alice cố làm gì đó trong namespace khác → DENIED

rules:
  - apiGroups: ["", "apps", "argoproj.io"]
    # ^ "" (empty string) = core API group: pods, services, configmaps
    # ^ "apps" = deployments, replicasets, statefulsets
    # ^ "argoproj.io" = argo rollouts (custom resource)
    resources:
      - pods          # Pod object chính
      - pods/log      # Subresource: kubectl logs
      - services      # K8s Service
      - configmaps    # ConfigMap
      - deployments   # apps/v1 Deployment
      - replicasets   # apps/v1 ReplicaSet (tự động tạo bởi Deployment)
      - rollouts      # argoproj.io/v1alpha1 Rollout (Argo Rollouts)
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
    # ^ get    = đọc 1 object cụ thể (kubectl get pod nginx)
    # ^ list   = liệt kê tất cả (kubectl get pods)
    # ^ watch  = theo dõi thay đổi real-time (kubectl get pods -w)
    # ^ create = tạo mới (kubectl apply, kubectl run)
    # ^ update = cập nhật toàn bộ (PUT request)
    # ^ patch  = cập nhật một phần (PATCH request, kubectl patch)
    # ^ delete = xóa (kubectl delete)
```

**Ý nghĩa thực tế:** Alice là developer, cần full CRUD quyền với workloads trong namespace `demo` để deploy và debug ứng dụng. Không cần xem nodes, namespaces, hay resources ở cluster level.

---

#### Block 2: ClusterRole `sre-pod-operator` (cho Bob)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
# ^ "ClusterRole" = có thể áp dụng ở bất kỳ namespace nào
# ^ Hoặc có thể truy cập cluster-level resources (nodes, namespaces)
# ^ Khi bind bằng ClusterRoleBinding → hiệu lực trên toàn cluster
# ^ Khi bind bằng RoleBinding (trong 1 namespace) → chỉ trong namespace đó

metadata:
  name: sre-pod-operator
  # ^ Không có "namespace" field → đây là cluster-level resource

rules:
  - apiGroups: [""]
    resources:
      - pods        # Quản lý pods
      - pods/log    # Xem logs
      - pods/exec   # Exec vào pod (kubectl exec -it) - quan trọng cho SRE!
      - events      # Xem events để debug
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
    # ^ Full access với pods trên toàn cluster (vì ClusterRoleBinding)

  - apiGroups: [""]
    resources:
      - namespaces  # Xem danh sách namespaces
      - nodes       # Xem thông tin nodes (memory, CPU, status)
    verbs: ["get", "list", "watch"]
    # ^ CHỈ đọc (read-only) với namespaces và nodes
    # ^ SRE cần xem để diagnose nhưng không cần modify

  - apiGroups: ["apps", "argoproj.io"]
    resources:
      - deployments
      - replicasets
      - rollouts
    verbs: ["get", "list", "watch"]
    # ^ CHỈ đọc với deployments/rollouts
    # ^ SRE cần xem để hiểu app state nhưng không deploy
```

**Ý nghĩa thực tế:** Bob là SRE, cần troubleshoot pods ở bất kỳ namespace nào (kubectl exec, logs, events), xem nodes/namespaces để diagnose infrastructure issues, nhưng không cần tạo/sửa deployments.

---

#### Block 3: ClusterRole `platform-viewer` (cho Carol)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: platform-viewer

rules:
  - apiGroups: ["*"]
    # ^ "*" = tất cả API groups (core, apps, networking, argoproj.io, v.v.)
    resources: ["*"]
    # ^ "*" = tất cả resource types
    verbs: ["get", "list", "watch"]
    # ^ CHỈ đọc - không có create/update/delete
    # ^ Đây là "read-only admin" pattern
```

**Ý nghĩa thực tế:** Carol là platform engineer/quản lý, cần visibility toàn bộ cluster để monitoring, audit, planning. Không cần modify bất cứ gì.

---

### 3.3 File: `rbac/rolebindings.yaml` – Phân tích từng dòng

#### Binding 1: Alice – RoleBinding (namespace-scoped)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
# ^ RoleBinding → áp dụng trong 1 namespace cụ thể

metadata:
  name: alice-developer-workload-manager
  namespace: demo
  # ^ Binding này chỉ có hiệu lực trong namespace "demo"

subjects:
  - kind: User
    # ^ Loại subject: User | Group | ServiceAccount
    # ^ "User" = người dùng xác thực qua certificate, OIDC, token
    name: alice
    # ^ Tên user (phải khớp với CN trong client certificate)
    apiGroup: rbac.authorization.k8s.io
    # ^ Luôn phải có field này cho User và Group

roleRef:
  kind: Role
  # ^ Tham chiếu đến Role (namespace-scoped)
  # ^ Lưu ý: RoleBinding cũng có thể ref đến ClusterRole
  #   nhưng khi đó ClusterRole chỉ có hiệu lực trong namespace của Binding
  name: developer-workload-manager
  # ^ Phải khớp chính xác với metadata.name của Role
  apiGroup: rbac.authorization.k8s.io
```

**Tại sao Alice dùng Role thay vì ClusterRole?**
Alice là developer chỉ làm việc với namespace `demo`. Cấp ClusterRole cho developer vi phạm nguyên tắc **Principle of Least Privilege** – chỉ cấp đúng quyền cần thiết. Nếu Alice bị compromise, kẻ tấn công chỉ ảnh hưởng được namespace `demo`.

---

#### Binding 2: Bob – ClusterRoleBinding (cluster-wide)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
# ^ ClusterRoleBinding → áp dụng trên TOÀN cluster (không có namespace field)

metadata:
  name: bob-sre-pod-operator
  # ^ Không có "namespace" field

subjects:
  - kind: User
    name: bob
    apiGroup: rbac.authorization.k8s.io

roleRef:
  kind: ClusterRole
  name: sre-pod-operator
  apiGroup: rbac.authorization.k8s.io
```

**Tại sao Bob dùng ClusterRole + ClusterRoleBinding?**
SRE cần troubleshoot pods ở **bất kỳ namespace nào** (production, staging, demo...). Khi incident xảy ra, không thể giới hạn SRE chỉ xem 1 namespace. ClusterRoleBinding cho phép Bob `exec` vào pod ở mọi namespace.

---

#### Binding 3: Carol – ClusterRoleBinding (cluster-wide read-only)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: carol-platform-viewer

subjects:
  - kind: User
    name: carol
    apiGroup: rbac.authorization.k8s.io

roleRef:
  kind: ClusterRole
  name: platform-viewer
  apiGroup: rbac.authorization.k8s.io
```

**Tại sao Carol dùng ClusterRoleBinding?**
Carol cần xem toàn bộ platform: tất cả namespaces, tất cả resources, tất cả nodes. Chỉ read-only nên risk thấp, nhưng phạm vi phải cluster-wide.

---

### 3.4 Bảng so sánh Role vs ClusterRole

| Tiêu chí | Role | ClusterRole |
|---|---|---|
| Phạm vi | 1 namespace | Toàn cluster |
| Có `namespace` field | Có | Không |
| Bind bằng | RoleBinding | ClusterRoleBinding hoặc RoleBinding |
| Truy cập nodes/namespaces | Không | Có (nếu được define) |
| Use case | Developer, app-specific | SRE, admin, cross-namespace |

### 3.5 Bảng so sánh RoleBinding vs ClusterRoleBinding

| Tiêu chí | RoleBinding | ClusterRoleBinding |
|---|---|---|
| Phạm vi binding | 1 namespace | Toàn cluster |
| Ref đến Role | Có | Không |
| Ref đến ClusterRole | Có (nhưng giới hạn namespace) | Có (full cluster) |
| Kết quả | Quyền chỉ trong 1 namespace | Quyền trên toàn cluster |

### 3.6 File ArgoCD App: `argocd/apps/rbac.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: rbac
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
    # ^ Wave -1: deploy TRƯỚC TẤT CẢ các wave khác
    # ^ RBAC phải tồn tại trước khi Gatekeeper, ESO cần ServiceAccount permissions

spec:
  project: default
  source:
    repoURL: https://github.com/bearh141/Lab1806.git
    path: rbac
    # ^ Chỉ sync folder "rbac/" (roles.yaml + rolebindings.yaml)
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: demo
    # ^ Destination namespace = demo (nơi Role và RoleBinding được tạo)
    # ^ ClusterRole và ClusterRoleBinding vẫn cluster-scoped dù destination là demo

  syncPolicy:
    automated:
      prune: true
      # ^ Nếu xóa file khỏi repo → ArgoCD xóa resource trong cluster
      selfHeal: true
      # ^ Nếu ai đó manual edit cluster → ArgoCD revert về trạng thái repo
    syncOptions:
      - CreateNamespace=true
      # ^ Tự tạo namespace "demo" nếu chưa tồn tại
      - ServerSideApply=true
      # ^ Dùng Server-Side Apply thay vì client-side apply
      # ^ Tránh conflict khi ArgoCD và kubectl cùng manage RBAC resources
```

---

## 4. LAB 1.2 – GATEKEEPER 4 CONSTRAINTS

### 4.1 Kiến trúc Admission Webhook

Trước khi giải thích từng constraint, cần hiểu **Admission Webhook** hoạt động như thế nào:

```
kubectl apply -f pod.yaml
        │
        ▼
┌─────────────────┐
│   API Server    │
│                 │
│  1. AuthN       │  ← Xác thực: user này là ai?
│     (cert/token)│
│                 │
│  2. AuthZ(RBAC) │  ← Phân quyền: user này có quyền tạo Pod không?
│                 │
│  3. Mutating    │  ← Sửa đổi object (inject sidecar, add labels)
│     Webhooks    │
│                 │
│  4. Object      │  ← Schema validation (OpenAPI)
│     Validation  │
│                 │
│  5. Validating  │  ← GATEKEEPER chạy ở đây
│     Webhooks    │     Kiểm tra policy: có vi phạm không?
│                 │
│  6. Persist     │  ← Lưu vào etcd nếu pass hết
│     to etcd     │
└─────────────────┘
```

**Điểm quan trọng:** Gatekeeper là **Validating Admission Webhook**. Nó chỉ có thể **ALLOW** hoặc **DENY**, không sửa đổi object. Nếu deny → API Server trả về HTTP 403 với message từ Rego policy.

### 4.2 ConstraintTemplate vs Constraint – Sự khác biệt

```
ConstraintTemplate               Constraint
─────────────────                ──────────
"Khuôn" của policy               "Thực thể" policy cụ thể
Định nghĩa CRD mới               Instance của CRD đó
Chứa logic Rego                  Chứa params + match conditions
Tương tự: class definition        Tương tự: object instance
Tạo trước (wave 0 trong template) Tạo sau (wave 1 trong constraint)

Ví dụ:
  ConstraintTemplate: k8sdisallowedtags     → tạo CRD "K8sDisallowedTags"
  Constraint: disallow-latest-tag           → instance K8sDisallowedTags
                                              với params: tags: ["latest"]
```

### 4.3 Constraint 1: Disallow Latest Tag

#### File: `gatekeeper/constraints/disallow-latest-tag.yaml`

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
# ^ API group tự động tạo bởi ConstraintTemplate khi define CRD

kind: K8sDisallowedTags
# ^ Kind này KHÔNG có sẵn trong K8s
# ^ Được tạo động bởi ConstraintTemplate "k8sdisallowedtags"
# ^ Nếu ConstraintTemplate chưa được apply → lỗi "unknown kind"

metadata:
  name: disallow-latest-tag
  annotations:
    argocd.argoproj.io/sync-wave: "1"
    # ^ Wave 1: sau khi ConstraintTemplate (wave 0) đã được tạo

spec:
  enforcementAction: deny
  # ^ "deny" = hard block, request bị từ chối hoàn toàn
  # ^ Các option khác:
  #   "warn"  = cho phép qua nhưng hiện warning (mềm hơn)
  #   "dryrun"= chỉ log violation, không block (test mode)

  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
        # ^ "" = core API group, Pod là core resource
      - apiGroups: ["apps"]
        kinds: ["Deployment"]
        # ^ apps group, Deployment
      - apiGroups: ["argoproj.io"]
        kinds: ["Rollout"]
        # ^ Argo Rollouts custom resource
    excludedNamespaces:
      - kube-system
      # ^ System components (kube-proxy, coredns) dùng internal images
      - gatekeeper-system
      # ^ Chính Gatekeeper cần image không kiểm soát được tag
      - argocd
      # ^ ArgoCD components cũng vậy
      - external-secrets
      # ^ ESO operator
      - monitoring
      # ^ Prometheus, Grafana stack
      - cosign-system
      # ^ Sigstore policy controller
      # ^ TẠI SAO phải exclude? Vì các system namespace dùng images
      # ^ từ upstream (quay.io, ghcr.io) có thể dùng :latest hoặc
      # ^ không có tag. Nếu không exclude → block toàn bộ system

  parameters:
    tags: ["latest"]
    # ^ Danh sách tag bị cấm. Ở đây chỉ cấm "latest"
    # ^ Template Rego sẽ check: nếu image có suffix ":latest" → deny
    # ^ Cũng cấm image không có tag (ví dụ "nginx" không có ":")
```

**Tại sao cấm tag `:latest`?**
1. **Không reproducible:** `nginx:latest` hôm nay khác `nginx:latest` tuần sau
2. **Không biết version:** debug khó vì không biết đang chạy version nào
3. **Security risk:** latest có thể pull về image mới có CVE
4. **GitOps bị phá vỡ:** Git có version cụ thể nhưng cluster pull random version

**Logic Rego trong ConstraintTemplate (k8sdisallowlatesttag.yaml):**
```rego
violation[{"msg": msg}] {
    container := input_containers[_]          # Lấy từng container
    not is_exempt(container)                  # Không trong exemptImages
    tags := [tag_with_prefix | tag := input.parameters.tags[_];
             tag_with_prefix := concat(":", ["", tag])]  # Build ":latest"
    strings.any_suffix_match(container.image, tags)      # Image kết thúc bằng ":latest"
    msg := sprintf("container <%v> uses a disallowed tag <%v>...", [...])
}

violation[{"msg": msg}] {
    container := input_containers[_]
    not is_exempt(container)
    parts := split(container.image, "/")
    not contains(parts[count(parts) - 1], ":")  # Không có ":" → không có tag
    msg := sprintf("container <%v> didn't specify an image tag <%v>", [...])
}
```

---

### 4.4 Constraint 2: Require Resource Limits

#### File: `gatekeeper/constraints/require-resource-limits.yaml`

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredResources
# ^ Tạo bởi ConstraintTemplate "k8srequiredresources"

metadata:
  name: require-resource-limits
  annotations:
    argocd.argoproj.io/sync-wave: "1"

spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
      - apiGroups: ["apps"]
        kinds: ["Deployment"]
      - apiGroups: ["argoproj.io"]
        kinds: ["Rollout"]
    excludedNamespaces:
      # ^ Cùng danh sách exclude với các constraint khác
      - kube-system
      - gatekeeper-system
      - argocd
      - external-secrets
      - monitoring
      - cosign-system

  parameters:
    limits: ["cpu", "memory"]
    # ^ Bắt buộc phải có CẢ cpu VÀ memory limits
    # ^ Nếu chỉ có cpu.limit mà không có memory.limit → vẫn DENY
```

**Tại sao bắt buộc resource limits?**
1. **Noisy neighbor problem:** Pod không có limit có thể consume hết CPU/memory của node, làm chết các pod khác
2. **OOM Killer:** Linux sẽ kill process random khi hết memory; limit giúp container bị kill trước khi ảnh hưởng node
3. **Cost control:** Không limit → 1 bug có thể spin up vô hạn resources → bill AWS tăng vọt
4. **Kubernetes Scheduler:** Scheduler cần `requests` để quyết định pod đặt lên node nào; không có limit thì QoS class = BestEffort (thấp nhất)

**Logic Rego:**
```rego
general_violation[{"msg": msg, "field": field}] {
    container := input.review.object.spec[field][_]  # Từng container
    not is_exempt(container)
    provided := {resource_type | container.resources.limits[resource_type]}
    # ^ Set of resource types đã có limit: {"cpu", "memory"} hoặc subset
    required := {resource_type | resource_type := input.parameters.limits[_]}
    # ^ Set of required: {"cpu", "memory"}
    missing := required - provided
    # ^ Set difference: cái cần có nhưng chưa có
    count(missing) > 0
    # ^ Nếu missing không rỗng → violation
    msg := sprintf("container <%v> does not have <%v> limits defined", ...)
}
```

---

### 4.5 Constraint 3: Disallow Root User

#### File: `gatekeeper/constraints/disallow-root-user.yaml`

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sPSPAllowedUsers
# ^ PSP = Pod Security Policy (deprecated nhưng template vẫn dùng)
# ^ Template này từ Gatekeeper library, handle runAsUser logic

metadata:
  name: disallow-root-user
  annotations:
    argocd.argoproj.io/sync-wave: "1"

spec:
  enforcementAction: deny
  match:
    # ^ Cùng kinds và excludedNamespaces như các constraint khác
    kinds: [Pod, Deployment, Rollout...]
    excludedNamespaces: [kube-system, ...]

  parameters:
    runAsUser:
      rule: MustRunAsNonRoot
      # ^ 3 rule options:
      # ^ "MustRunAs"       = phải chạy trong range UID cụ thể
      # ^ "MustRunAsNonRoot"= UID != 0 (root), không cần specify UID cụ thể
      # ^ "RunAsAny"        = không restrict (default của K8s)
      # ^
      # ^ "MustRunAsNonRoot" có nghĩa: container phải có 1 trong 2:
      # ^   securityContext.runAsNonRoot: true  (K8s tự enforce != 0)
      # ^   securityContext.runAsUser: <non-zero number>
```

**Tại sao cấm root?**
1. **Privilege escalation:** Process root trong container có thể escape sang host nếu có lỗ hổng container runtime
2. **File permission:** Root có thể đọc/ghi file nhạy cảm
3. **Best practice:** Theo Docker security best practices và CIS Kubernetes Benchmark
4. **Kubernetes PSA:** K8s 1.25+ cũng enforce điều này qua Pod Security Admission (restricted profile)

**Ví dụ pod hợp lệ:**
```yaml
securityContext:
  runAsUser: 1000        # UID 1000, không phải 0
  runAsNonRoot: true     # Enforced ở runtime
```

---

### 4.6 Constraint 4: Disallow Host Network

#### File: `gatekeeper/constraints/disallow-host-network.yaml`

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sPSPHostNetworkingPorts
metadata:
  name: disallow-host-network
  annotations:
    argocd.argoproj.io/sync-wave: "1"

spec:
  enforcementAction: deny
  match:
    kinds: [Pod, Deployment, Rollout...]
    excludedNamespaces: [kube-system, ...]

  parameters:
    hostNetwork: false
    # ^ hostNetwork: false = KHÔNG cho phép pod dùng host network namespace
    # ^ Pod dùng hostNetwork: true sẽ thấy và bind vào network interfaces
    # ^ của node (eth0, lo) thay vì virtual network interface của pod
    min: 0
    max: 65535
    # ^ Dải port được phép dùng làm hostPort
    # ^ min: 0, max: 65535 = cho phép toàn bộ port range
    # ^ (Nhưng hostNetwork: false đã block rồi nên min/max ít quan trọng)
```

**Tại sao cấm hostNetwork?**
1. **Network isolation:** K8s network model có container network namespace riêng biệt; hostNetwork phá vỡ isolation này
2. **Port conflict:** Pod dùng hostNetwork có thể conflict với services đang chạy trên node
3. **Security:** Pod có thể nghe traffic của node, sniff packets từ các pod khác trên cùng node
4. **Privilege:** hostNetwork là privilege tương đương với quyền admin mạng trên node

**Cờ `hostNetwork: true` trong pod spec:**
```yaml
spec:
  hostNetwork: true   # BTHỊ ĐÂY → bị Gatekeeper deny
  containers:
  - name: nginx
    image: nginx:1.25
```

---

### 4.7 File ArgoCD: `argocd/apps/gatekeeper.yaml` và `gatekeeper-policies.yaml`

```yaml
# gatekeeper.yaml – Cài OPA Gatekeeper bằng Helm
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gatekeeper
  annotations:
    argocd.argoproj.io/sync-wave: "0"   # Wave 0: cài Gatekeeper trước

spec:
  source:
    chart: gatekeeper
    repoURL: https://open-policy-agent.github.io/gatekeeper/charts
    targetRevision: 3.17.1              # Version cố định
    helm:
      parameters:
        # Tăng timeout cho probe để tránh restart loop trong cluster chậm
        - name: controllerManager.livenessProbe.timeoutSeconds
          value: "5"
        - name: controllerManager.readinessProbe.timeoutSeconds
          value: "5"
        - name: audit.livenessProbe.timeoutSeconds
          value: "5"
        - name: audit.readinessProbe.timeoutSeconds
          value: "5"

  destination:
    namespace: gatekeeper-system        # Namespace riêng cho Gatekeeper

  ignoreDifferences:
    # Gatekeeper tự update CRD status/labels → ArgoCD sẽ liên tục "out of sync"
    # ignoreDifferences nói ArgoCD bỏ qua các field này
    - group: apiextensions.k8s.io
      kind: CustomResourceDefinition
      jsonPointers:
        - /status                        # CRD status tự update
        - /metadata/labels               # Gatekeeper inject labels
        - /metadata/annotations          # Gatekeeper inject annotations
        - /spec/conversion               # Conversion webhook tự config
```

```yaml
# gatekeeper-policies.yaml – Deploy Templates + Constraints
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "1"   # Wave 1: sau Gatekeeper đã cài

spec:
  source:
    path: gatekeeper                     # Sync cả folder gatekeeper/
    # bao gồm: templates/ + constraints/

  syncPolicy:
    syncOptions:
      - SkipDryRunOnMissingResource=true
      # ^ Quan trọng! ConstraintTemplate CRD chưa tồn tại trước wave 0
      # ^ Nếu không có flag này, ArgoCD dry-run sẽ fail
      - ServerSideApply=true
      # ^ Gatekeeper CRD có validation schema phức tạp
      # ^ Server-Side Apply xử lý tốt hơn client-side
```

---

## 5. LAB 1.3 – CUSTOM POLICY: OWNER LABEL

### 5.1 Tổng quan Custom Template

Trong khi 4 constraints ở Lab 1.2 đều dùng template từ **Gatekeeper Policy Library** (open-source, community-maintained), constraint `require-owner-label` dùng **template tự viết** vì không có template nào sẵn có kiểm tra business label tùy chỉnh.

```
Gatekeeper Library Templates (tải về)    Custom Template (tự viết)
──────────────────────────────────────   ─────────────────────────
k8sdisallowedtags.yaml                   k8srequiredownerlabel.yaml
k8srequiredresources.yaml
k8spspallowedusers.yaml
k8spsphostnetworkingports.yaml
```

### 5.2 File: `gatekeeper/templates/k8srequiredownerlabel.yaml`

```yaml
apiVersion: templates.gatekeeper.sh/v1
# ^ API group cho ConstraintTemplate

kind: ConstraintTemplate
# ^ Đây là "khuôn", định nghĩa CRD mới và logic Rego

metadata:
  name: k8srequiredownerlabel
  # ^ Tên của template. Convention: tất cả lowercase, không có dash
  # ^ Tên này sẽ trở thành prefix của CRD (K8sRequiredOwnerLabel)
  annotations:
    argocd.argoproj.io/sync-wave: "0"
    # ^ Deploy TRƯỚC constraint (wave 0 trong folder templates/)

spec:
  crd:
    spec:
      names:
        kind: K8sRequiredOwnerLabel
        # ^ Đây là Kind mới sẽ được tạo trong K8s API
        # ^ Convention: PascalCase, thường bắt đầu bằng "K8s"
        # ^ Sau khi apply template → kubectl get K8sRequiredOwnerLabel sẽ work
        # ^ Không có "validation" schema ở đây vì constraint này
        # ^ không cần parameters (không cần customize gì cả)

  targets:
    - target: admission.k8s.gatekeeper.sh
      # ^ Target: admission webhook của Gatekeeper
      # ^ Có thể có targets khác (audit) nhưng admission là phổ biến nhất
      
      rego: |
        package k8srequiredownerlabel
        # ^ Package declaration - tên package (convention: tên template)
        # ^ Rego là ngôn ngữ policy của OPA (Open Policy Agent)

        violation[{"msg": msg}] {
          # ^ "violation" là rule đặc biệt của Gatekeeper
          # ^ Nếu rule này evaluate thành TRUE → có violation → DENY
          # ^ Format: violation[{"msg": msg}] { ... body ... }
          # ^ "msg" là message trả về cho user khi bị deny
          
          not input.review.object.metadata.labels.owner
          # ^ "input" là object đặc biệt trong Rego chứa toàn bộ context
          # ^ input.review = admission review object từ K8s API Server
          # ^ input.review.object = resource đang được submit (Pod/Deployment)
          # ^ input.review.object.metadata = metadata của resource
          # ^ input.review.object.metadata.labels = labels map
          # ^ input.review.object.metadata.labels.owner = label "owner"
          # ^
          # ^ "not X" trong Rego = negation as failure
          # ^ Nghĩa là: nếu labels.owner KHÔNG TỒN TẠI (undefined/null)
          # ^ → condition này TRUE → tiếp tục evaluate
          # ^ Nếu labels.owner có giá trị → "not" = false → rule không fire

          msg := "workload must have metadata.labels.owner"
          # ^ Assign message. Đây là string trả về cho user khi bị deny
          # ^ kubectl apply sẽ hiển thị: admission webhook denied the request:
          # ^   workload must have metadata.labels.owner
        }
```

### 5.3 Giải thích chi tiết Rego Language

#### Khái niệm cơ bản

**Rego** là ngôn ngữ khai báo (declarative) được dùng trong OPA. Khác với ngôn ngữ imperative (Java, Python), Rego không nói "làm gì" mà nói "điều gì là đúng".

```
Ngôn ngữ imperative (Python):     Rego (declarative):
─────────────────────────────      ───────────────────
if "owner" not in labels:          violation[{"msg": msg}] {
    deny = True                      not input.review.object.
    msg = "need owner label"               metadata.labels.owner
                                     msg := "need owner label"
                                   }
```

#### `input` object – Cấu trúc đầy đủ

```
input {
  parameters {         ← Từ spec.parameters của Constraint
    tags: ["latest"]   ← Ví dụ với disallow-latest
  }
  review {             ← AdmissionReview từ K8s API Server
    operation: "CREATE"  ← CREATE | UPDATE | DELETE | CONNECT
    kind {
      group: "apps"
      kind: "Deployment"
      version: "v1"
    }
    namespace: "demo"
    object {           ← Resource đang được submit
      metadata {
        name: "my-app"
        namespace: "demo"
        labels {
          app: "my-app"
          # owner: ???  ← Nếu thiếu field này → not input.review.object.metadata.labels.owner = TRUE
        }
      }
      spec {
        containers: [...]
      }
    }
    oldObject {        ← Object cũ (chỉ có khi UPDATE)
      ...
    }
  }
}
```

#### Negation as Failure (`not`)

Đây là concept quan trọng nhất trong Rego:

```rego
# Case 1: labels.owner = "team-a"
not input.review.object.metadata.labels.owner
# → "owner" tồn tại và có giá trị → not TRUE = FALSE
# → Condition FALSE → rule không fire → NO violation → ALLOW

# Case 2: labels không có "owner"
not input.review.object.metadata.labels.owner
# → "owner" không tồn tại → evaluates to undefined
# → not undefined = TRUE
# → Condition TRUE → rule fires → VIOLATION → DENY

# Case 3: labels.owner = "" (empty string)
not input.review.object.metadata.labels.owner
# → "" là falsy trong Rego → not "" = TRUE
# → VIOLATION → DENY
# ^ Nghĩa là: label phải tồn tại VÀ có giá trị non-empty
```

#### So sánh với logic imperative

```
Gatekeeper Rego (logic đảo ngược):
- Rule "violation" fire khi điều kiện SAI
- Nếu không có violation → ALLOW
- Nếu có ít nhất 1 violation → DENY

KHÔNG viết: "if owner exists → allow"
MÀ viết:    "if owner NOT exists → violation (deny)"
```

### 5.4 File: `gatekeeper/constraints/require-owner-label.yaml`

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredOwnerLabel
# ^ Kind này vừa được tạo bởi ConstraintTemplate "k8srequiredownerlabel"
# ^ Đây là instance của CRD đó

metadata:
  name: require-owner-label
  annotations:
    argocd.argoproj.io/sync-wave: "1"   # Sau template (wave 0)

spec:
  enforcementAction: deny
  # ^ Hard block

  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment"]
        # ^ Chỉ check Deployment, không check Pod
        # ^ Lý do: Pod thường được tạo bởi Deployment/ReplicaSet
        # ^ Yêu cầu label ở cấp Deployment là đủ
      - apiGroups: ["argoproj.io"]
        kinds: ["Rollout"]
        # ^ Argo Rollouts cũng là workload level → check luôn
    excludedNamespaces:
      - kube-system
      - gatekeeper-system
      - argocd
      - external-secrets
      - monitoring
      - cosign-system
      # ^ System namespaces không cần owner label
      # ^ Chỉ workload user-defined cần label này

  # Không có "parameters" field
  # ^ Constraint này không cần tham số vì Rego logic đã hardcode
  # ^ check "owner" label cụ thể
  # ^ Nếu muốn flexible (check bất kỳ label nào), cần thêm
  # ^ parameters schema vào ConstraintTemplate và đọc từ input.parameters
```

### 5.5 Test cases cho Owner Label

```bash
# Test 1: Deploy KHÔNG có owner label → BỊ DENY
kubectl apply -f tests/deploy-no-owner.yaml
# deploy-no-owner.yaml:
#   metadata:
#     name: test-deploy-no-owner
#     namespace: demo
#     # KHÔNG CÓ labels.owner → BỊ GATEKEEPER BLOCK
# Output:
# Error from server (Forbidden): admission webhook "validation.gatekeeper.sh" denied the request:
# [require-owner-label] workload must have metadata.labels.owner

# Test 2: Deploy CÓ owner label → ĐƯỢC PASS
kubectl apply -f tests/deploy-with-owner.yaml
# deploy-with-owner.yaml:
#   metadata:
#     name: test-deploy-with-owner
#     namespace: demo
#     labels:
#       owner: platform-team   # ← CÓ LABEL NÀY → PASS
# Output: deployment.apps/test-deploy-with-owner created
```

### 5.6 Tại sao đây là Custom Template chứ không dùng từ Library?

Gatekeeper Policy Library có sẵn nhiều template nhưng không có template "require specific custom label" vì:
1. **Tên label là business-specific:** `owner` là tên label của project này, project khác có thể dùng `team`, `squad`, `cost-center`
2. **Logic quá đơn giản:** Chỉ cần 3 dòng Rego, không cần complex library template
3. **Không có `parameters`:** Library templates thường có schema validation phức tạp; custom template đơn giản hơn nhiều
4. **Flexibility:** Tự viết cho phép customize logic (ví dụ: check label value hợp lệ, check nhiều labels cùng lúc)

Nếu muốn dùng approach "required labels" từ library, có thể dùng template `k8srequiredlabels` từ Gatekeeper library nhưng sẽ cần thêm parameters schema phức tạp hơn.

---

## 6. LAB 2.1 – ESO (External Secrets Operator)

### 6.1 Vấn đề mà ESO giải quyết

Kubernetes Secret thông thường có nhược điểm lớn:
1. **Lưu trong etcd dưới dạng base64** (không phải mã hóa thật sự)
2. **Ai có quyền đọc Secret trong namespace đó đều đọc được**
3. **Phải commit secret vào Git** hoặc manual apply → không phù hợp GitOps
4. **Rotation khó:** Update secret → phải restart pods thủ công

ESO (External Secrets Operator) giải quyết bằng cách:
- **Lưu secret thật sự ở AWS Secrets Manager** (mã hóa bằng KMS)
- **ESO tự động sync** từ AWS vào K8s Secret
- **Rotation tự động:** Update trong AWS → ESO detect → K8s Secret update trong 10s

### 6.2 Sơ đồ flow ESO

```
┌─────────────────────────────────────────────────────────────────┐
│                    AWS SECRETS MANAGER                          │
│  Secret name: "w10/demo/db-password"                           │
│  Value: {"password": "SuperSecret123!"}                         │
│  KMS encrypted, IAM controlled                                  │
└───────────────────┬─────────────────────────────────────────────┘
                    │  AWS SDK API call
                    │  (using IAM credentials)
                    ▼
┌─────────────────────────────────────────────────────────────────┐
│              ESO CONTROLLER (namespace: external-secrets)        │
│                                                                 │
│  1. Đọc ExternalSecret "db-creds" trong namespace "demo"        │
│  2. Xem secretStoreRef → trỏ đến SecretStore "aws-store"       │
│  3. Đọc SecretStore → lấy AWS credentials từ Secret "aws-creds" │
│  4. Gọi AWS SecretsManager.GetSecretValue()                     │
│  5. Tạo/Update K8s Secret "db-secret" trong namespace "demo"    │
│  6. Lặp lại mỗi 10 giây (refreshInterval)                       │
└───────────────────┬─────────────────────────────────────────────┘
                    │  Create/Update K8s Secret
                    ▼
┌─────────────────────────────────────────────────────────────────┐
│  NAMESPACE: demo                                                 │
│                                                                 │
│  Secret "aws-creds"           Secret "db-secret"               │
│  ┌──────────────────┐         ┌──────────────────────────┐     │
│  │ access-key: xxx  │──ESO──► │ password: SuperSecret123!│     │
│  │ secret-key: yyy  │ reads   └────────────┬─────────────┘     │
│  └──────────────────┘                      │ mount              │
│                                            ▼                   │
│                               Pod "api-server"                 │
│                               ┌────────────────────┐           │
│                               │ env:               │           │
│                               │  DB_PASSWORD=      │           │
│                               │  SuperSecret123!   │           │
│                               └────────────────────┘           │
└─────────────────────────────────────────────────────────────────┘
```

### 6.3 File: `eso/secret-store.yaml` – Phân tích chi tiết

```yaml
apiVersion: external-secrets.io/v1beta1
# ^ API group của ESO (sau khi cài Helm chart với CRDs)

kind: SecretStore
# ^ SecretStore: cấu hình KẾT NỐI đến external secret provider
# ^ Khác với ClusterSecretStore: SecretStore chỉ trong 1 namespace
# ^ ClusterSecretStore: dùng được từ nhiều namespaces

metadata:
  name: aws-store
  # ^ Tên SecretStore, được tham chiếu trong ExternalSecret
  namespace: demo
  # ^ Chỉ ExternalSecret trong namespace "demo" mới dùng được store này
  # ^ Nếu cần cross-namespace → dùng ClusterSecretStore

spec:
  provider:
    aws:
      service: SecretsManager
      # ^ AWS service: SecretsManager | ParameterStore | SSM
      # ^ SecretsManager: lưu JSON secrets, versioning, rotation
      # ^ ParameterStore: lưu key-value, phân cấp path, rẻ hơn
      
      region: ap-southeast-1
      # ^ AWS region của Secrets Manager
      # ^ Phải đúng region nơi secret được lưu
      
      auth:
        secretRef:
          # ^ Cách auth: dùng Access Key/Secret Key từ K8s Secret
          # ^ Alternative: IRSA (IAM Roles for Service Accounts) - tốt hơn cho production
          # ^ IRSA không cần lưu credentials trong cluster, dùng OIDC
          
          accessKeyIDSecretRef:
            name: aws-creds
            # ^ Tên K8s Secret chứa AWS credentials
            # ^ Secret này phải tồn tại TRƯỚC khi ESO chạy
            key: access-key
            # ^ Key trong Secret data: {"access-key": "AKIA...", "secret-key": "..."}
          
          secretAccessKeySecretRef:
            name: aws-creds
            # ^ Cùng Secret
            key: secret-key
            # ^ Key khác trong Secret
```

**Lưu ý quan trọng về `aws-creds` Secret:**
Secret `aws-creds` phải được tạo thủ công (không lưu trong Git vì lý do bảo mật):
```bash
kubectl create secret generic aws-creds \
  --namespace demo \
  --from-literal=access-key="AKIAIOSFODNN7EXAMPLE" \
  --from-literal=secret-key="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
```

**Production best practice – IRSA (không cần lưu credentials):**
```yaml
auth:
  jwt:
    serviceAccountRef:
      name: eso-service-account  # SA với IRSA annotation
# Cần annotation trên ServiceAccount:
# eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT:role/ESO-Role
```

### 6.4 File: `eso/external-secret.yaml` – Phân tích chi tiết

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
# ^ ExternalSecret: cấu hình SECRET CỤ THỂ cần pull
# ^ Mỗi ExternalSecret = 1 K8s Secret được tạo ra

metadata:
  name: db-creds
  # ^ Tên ExternalSecret (metadata name, không phải tên Secret tạo ra)
  namespace: demo
  # ^ Namespace nơi K8s Secret sẽ được tạo

spec:
  refreshInterval: 10s
  # ^ ESO sẽ check AWS Secrets Manager mỗi 10 giây
  # ^ Nếu secret thay đổi → K8s Secret tự động update
  # ^ Production nên dùng giá trị cao hơn (1h, 24h) để tránh throttle
  # ^ AWS SM API có rate limit: 10,000 requests/region/account/second
  
  secretStoreRef:
    name: aws-store
    # ^ Tham chiếu đến SecretStore "aws-store" đã tạo ở trên
    kind: SecretStore
    # ^ SecretStore (namespace-scoped) hoặc ClusterSecretStore

  target:
    name: db-secret
    # ^ TÊN K8s Secret sẽ được tạo trong cluster
    # ^ kubectl get secret db-secret -n demo → thấy secret này
    creationPolicy: Owner
    # ^ Owner: ESO "owns" secret, nếu ExternalSecret bị xóa → secret cũng xóa
    # ^ Orphan: secret tồn tại ngay cả khi ExternalSecret bị xóa
    # ^ Merge: merge vào secret đã tồn tại (không xóa existing keys)

  data:
    - secretKey: password
      # ^ Tên key trong K8s Secret data
      # ^ kubectl get secret db-secret -n demo -o jsonpath='{.data.password}' | base64 -d
      remoteRef:
        key: w10/demo/db-password
        # ^ Tên secret trong AWS Secrets Manager
        # ^ Convention: {project}/{namespace}/{secret-name}
        # ^ Không có "property" field → lấy entire secret value
        # ^ Nếu AWS secret là JSON: {"password": "..."}, thêm:
        #   property: password   # Lấy chỉ field "password"
```

**Kết quả sau khi ESO sync:**
```bash
$ kubectl get secret db-secret -n demo
NAME        TYPE     DATA   AGE
db-secret   Opaque   1      5m

$ kubectl get secret db-secret -n demo -o jsonpath='{.data.password}' | base64 -d
SuperSecret123!
```

### 6.5 SecretStore vs ExternalSecret – So sánh chi tiết

| Tiêu chí | SecretStore | ExternalSecret |
|---|---|---|
| Mục đích | Cấu hình kết nối (connection) | Cấu hình secret cụ thể (mapping) |
| Tương tự | Database connection string | Database query |
| Tái sử dụng | 1 Store cho nhiều ExternalSecret | 1 ExternalSecret cho 1 K8s Secret |
| Chứa credentials | Có (AWS creds) | Không |
| Output | Không tạo Secret | Tạo K8s Secret |
| Scope | Namespace-specific | Namespace-specific |

### 6.6 Tại sao không cần restart pod khi rotate secret?

Đây là câu hỏi quan trọng. Có 2 cách mount secret vào pod:

**Cách 1: Environment Variable (KHÔNG auto-update)**
```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: password
# ^ Env var được inject lúc pod START
# ^ Nếu K8s Secret update → env var KHÔNG thay đổi
# ^ Cần restart pod để lấy giá trị mới
```

**Cách 2: Volume Mount (TỰ ĐỘNG update, với delay ~1 phút)**
```yaml
volumes:
  - name: db-creds
    secret:
      secretName: db-secret
volumeMounts:
  - name: db-creds
    mountPath: /etc/secrets
    readOnly: true
# ^ File /etc/secrets/password được sync từ K8s Secret
# ^ Khi K8s Secret update → kubelet tự update file trong ~1 phút
# ^ Không cần restart pod!
# ^ Nhưng: app phải đọc file mỗi lần dùng (không cache), hoặc
# ^        có cơ chế watch file changes (inotify)
```

**Cơ chế kubelet sync:**
```
AWS SM update → ESO detect (10s) → K8s Secret update
                                         ↓
                                   kubelet detect secret change
                                         ↓ (~60s)
                                   Update file trong pod volume
                                         ↓
                                   App đọc file mới
```

### 6.7 Files ArgoCD: `eso.yaml` và `eso-config.yaml`

```yaml
# eso.yaml – Cài ESO Operator bằng Helm (wave 0)
source:
  chart: external-secrets
  repoURL: https://charts.external-secrets.io
  targetRevision: 0.9.18
  helm:
    parameters:
      - name: installCRDs
        value: "true"
        # ^ Tự động cài CRDs (SecretStore, ExternalSecret, etc.)
        # ^ Nếu false, phải cài CRD thủ công trước
```

```yaml
# eso-config.yaml – Deploy SecretStore + ExternalSecret (wave 1)
annotations:
  argocd.argoproj.io/sync-wave: "1"
  # ^ Sau khi ESO đã cài (wave 0) và CRDs đã tồn tại
source:
  path: eso
  # ^ Sync folder eso/ chứa secret-store.yaml + external-secret.yaml
destination:
  namespace: demo
  # ^ SecretStore và ExternalSecret tạo trong namespace "demo"
```

---

## 7. LAB 2.2 – TRIVY + COSIGN + SIGSTORE (SUPPLY CHAIN SECURITY)

### 7.1 Vấn đề Supply Chain Attack

Supply chain attack xảy ra khi kẻ tấn công inject code độc hại vào **chuỗi build/deploy** thay vì tấn công trực tiếp vào production:

```
Kẻ tấn công inject malware vào:
  - Base image (alpine, ubuntu, python)      ← Image pull attack
  - Build dependencies (npm, pip packages)   ← Dependency confusion
  - CI/CD pipeline                           ← Pipeline hijack
  - Registry (man-in-the-middle)             ← Image tampering

Kết quả: Image "hợp lệ" nhưng chứa backdoor chạy trong production
```

**Lab này giải quyết bằng 3 lớp bảo vệ:**
1. **Trivy:** Scan CVE trong image dependencies trước khi push
2. **Cosign:** Ký image để chứng minh xuất xứ từ pipeline hợp lệ
3. **Sigstore Policy Controller:** Verify chữ ký khi pod được deploy vào cluster

### 7.2 Sơ đồ Supply Chain đầy đủ

```
                    Developer push code
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│                 GITHUB ACTIONS WORKFLOW                       │
│                                                              │
│  Step 1: Calculate semantic version                          │
│  ─────────────────────────────────────────                   │
│  paulhatch/semantic-version@v5.4.0                           │
│  Pattern: feat → minor bump, BREAKING CHANGE → major bump    │
│  Result: version = "1.2.3"                                   │
│                                                              │
│  Step 2: Build & Push Docker image                           │
│  ─────────────────────────────────────────                   │
│  docker build ./src/api → ghcr.io/bearh141/w10-api-141:1.2.3│
│  Also tags: latest, v1.2.3-sha-abc123                        │
│  Dockerfile: python:3.13-alpine (nhỏ, ít CVE hơn ubuntu)     │
│                                                              │
│  Step 3: Trivy vulnerability scan ← GATE 1                   │
│  ─────────────────────────────────────────                   │
│  trivy image ghcr.io/bearh141/w10-api-141:1.2.3             │
│  --severity HIGH,CRITICAL                                    │
│  --exit-code 1                    ← Nếu tìm thấy → FAIL     │
│  --ignore-unfixed                 ← Bỏ qua CVE chưa có patch│
│  --vuln-type os,library           ← Scan cả OS và deps       │
│                                                              │
│  Nếu Trivy fail → Pipeline dừng lại, image KHÔNG được sign  │
│  Nếu Trivy pass ↓                                            │
│                                                              │
│  Step 4: Cosign sign image ← GATE 2                          │
│  ─────────────────────────────────────────                   │
│  cosign sign --yes \                                         │
│    --key env://COSIGN_PRIVATE_KEY \                          │
│    "ghcr.io/bearh141/w10-api-141:1.2.3"                      │
│                                                              │
│  Private key: lưu trong GitHub Secrets (COSIGN_PRIVATE_KEY)  │
│  Kết quả: chữ ký lưu vào ghcr.io/bearh141/w10-api-141:sha256-xxx.sig │
│                                                              │
│  Step 5: Update rollout.yaml với version mới                 │
│  Step 6: Commit + push → ArgoCD trigger                      │
└──────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│                  ARGOCD SYNC                                  │
│                                                              │
│  ArgoCD detect rollout.yaml thay đổi                         │
│  Apply Rollout resource với image mới                         │
└──────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│              K8S ADMISSION CONTROL                            │
│                                                              │
│  kubectl apply Rollout ──► API Server                        │
│                                │                             │
│                     Validating Webhook                       │
│                     (Sigstore Policy Controller)              │
│                                │                             │
│    ClusterImagePolicy match?   │                             │
│    glob: "ghcr.io/bearh141/*" ─┘                             │
│                   │                                          │
│        YES: Verify cosign signature                          │
│                   │                                          │
│    Fetch .sig tag từ registry                                 │
│    Verify với public key trong ClusterImagePolicy            │
│                   │                                          │
│    VALID?  → ALLOW deployment                                │
│    INVALID? → DENY deployment (403)                          │
└──────────────────────────────────────────────────────────────┘
```

### 7.3 Giải thích chi tiết GitHub Actions Workflow

#### `build-push.yml` – Từng step

```yaml
name: Build and Push Image

on:
  push:
    branches: [main]
    paths:
      - 'src/api/**'               # Chỉ trigger khi code Python thay đổi
      - '.github/workflows/build-push.yml'  # Hoặc workflow thay đổi
  workflow_dispatch:               # Cho phép trigger manual từ GitHub UI

env:
  REGISTRY: ghcr.io                # GitHub Container Registry
  IMAGE_NAME: ${{ github.repository_owner }}/w10-api-141
  # ^ = bearh141/w10-api-141
  # ^ Full image: ghcr.io/bearh141/w10-api-141

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: write              # Cần để push commit (update rollout.yaml)
      packages: write              # Cần để push vào GitHub Container Registry
```

**Step: Calculate Semantic Version**
```yaml
- name: Calculate semantic version
  id: semver
  uses: paulhatch/semantic-version@v5.4.0
  with:
    tag_prefix: "v"
    major_pattern: "(BREAKING CHANGE:|!:)"  # feat!: hoặc BREAKING CHANGE: → major
    minor_pattern: "^feat"                   # feat: → minor
    version_format: "${major}.${minor}.${patch}"
    bump_each_commit: false
# Kết quả: steps.semver.outputs.version = "1.2.3"
# Conventional commits: feat: → 0.1.0, fix: → 0.0.1, feat!: → 1.0.0
```

**Step: Build & Push với multi-tag**
```yaml
- name: Extract metadata
  id: meta
  uses: docker/metadata-action@v5
  with:
    images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
    tags: |
      type=raw,value=latest            # tag :latest (luôn có)
      type=raw,value=${{ steps.semver.outputs.version }}  # tag :1.2.3
      type=sha,prefix=v${{ steps.semver.outputs.version }}-  # tag :v1.2.3-sha-abc1234

# Kết quả: image được push với 3 tags
# - ghcr.io/bearh141/w10-api-141:latest
# - ghcr.io/bearh141/w10-api-141:1.2.3
# - ghcr.io/bearh141/w10-api-141:v1.2.3-sha-abc1234
```

**Step: Trivy Scan**
```yaml
- name: Run Trivy vulnerability scanner
  uses: aquasecurity/trivy-action@v0.22.0
  with:
    image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.semver.outputs.version }}
    # ^ Scan image vừa push lên (version tag, không phải latest)
    
    format: 'table'
    # ^ Output format: table | json | sarif | cyclonedx
    # ^ "table" dễ đọc trong CI log
    
    exit-code: '1'
    # ^ '1' = fail pipeline nếu tìm thấy CVE theo severity
    # ^ '0' = chỉ report, không fail (dùng cho audit)
    
    ignore-unfixed: true
    # ^ Bỏ qua CVE chưa có patch upstream
    # ^ Lý do: không thể fix CVE chưa có patch → false positive
    # ^ Chỉ block những CVE đã có fix nhưng chưa update
    
    vuln-type: 'os,library'
    # ^ Scan cả OS packages (alpine apk) và library dependencies (pip)
    
    severity: 'HIGH,CRITICAL'
    # ^ Chỉ fail khi có HIGH hoặc CRITICAL
    # ^ LOW, MEDIUM: chấp nhận được (fix dần)
    # ^ HIGH: phải fix trước khi deploy
    # ^ CRITICAL: block ngay lập tức
```

**Trivy CVE severity levels:**
```
CRITICAL  (CVSS >= 9.0): Remote code execution, privilege escalation
HIGH      (CVSS 7.0-8.9): SQL injection, authentication bypass
MEDIUM    (CVSS 4.0-6.9): XSS, minor info disclosure
LOW       (CVSS < 4.0):   Minor issues
UNKNOWN:                  CVE chưa được CVSS score
```

**Step: Cosign Sign**
```yaml
- name: Sign the published Docker image
  env:
    COSIGN_PRIVATE_KEY: ${{ secrets.COSIGN_PRIVATE_KEY }}
    # ^ Private key được load từ GitHub Secrets (không hardcode trong code)
    # ^ Format: PEM-encoded ECDSA P-256 private key
    COSIGN_PASSWORD: ${{ secrets.COSIGN_PASSWORD }}
    # ^ Password để decrypt private key (nếu key được encrypt)
  run: |
    cosign sign --yes \
      --key env://COSIGN_PRIVATE_KEY \
      "${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.semver.outputs.version }}"
    # ^ --yes: không interactive prompt
    # ^ --key env://COSIGN_PRIVATE_KEY: đọc key từ env var (không từ file)
    # ^ Sign chỉ version tag, không sign :latest
    # ^ Tại sao? :latest thay đổi, không stable để verify
```

**Cosign signature storage:**
```
Sau khi sign, Cosign tạo thêm 1 entry trong registry:

ghcr.io/bearh141/w10-api-141:1.2.3           ← Image gốc
ghcr.io/bearh141/w10-api-141:sha256-abc...   ← Digest reference
ghcr.io/bearh141/w10-api-141:sha256-abc....sig ← Cosign signature!

Signature chứa:
- Payload: {"critical": {"image": {"docker-manifest-digest": "sha256:..."}}}
- Signature: ECDSA signature bằng private key
```

### 7.4 File: `signing/cosign.key` và `signing/cosign.pub`

```bash
# Tạo keypair Cosign
cosign generate-key-pair

# Output:
# cosign.key  ← Private key (lưu vào GitHub Secrets, KHÔNG commit)
# cosign.pub  ← Public key (có thể commit vào repo, an toàn)

# cosign.pub được dùng trong ClusterImagePolicy
```

**Lưu ý về file `signing/cosign.key` trong repo:**
Repo này có `signing/cosign.key` - đây thường là **key đã được encrypt bằng passphrase** hoặc là key cho lab/demo. Trong production, private key **không bao giờ** được commit vào Git.

### 7.5 File: `policies/cluster-image-policy.yaml` – Phân tích chi tiết

```yaml
apiVersion: policy.sigstore.dev/v1beta1
# ^ API group của Sigstore Policy Controller
# ^ Khác với OPA Gatekeeper (constraints.gatekeeper.sh)

kind: ClusterImagePolicy
# ^ Cluster-scoped policy (không có namespace)
# ^ Áp dụng cho tất cả namespaces được label phù hợp

metadata:
  name: image-signature-policy
  # ^ Tên policy, được log trong audit events khi verify

spec:
  images:
    - glob: "ghcr.io/bearh141/*"
      # ^ Glob pattern: match TẤT CẢ images từ ghcr.io/bearh141/
      # ^ "ghcr.io/bearh141/*" match:
      #     ghcr.io/bearh141/w10-api-141:1.2.3     ✓
      #     ghcr.io/bearh141/anything:tag            ✓
      # ^ KHÔNG match:
      #     docker.io/library/nginx:1.25             ✗
      #     ghcr.io/otheruser/image:tag              ✗
      #
      # ^ Ý nghĩa: chỉ enforce với images của project này
      # ^ Images từ nguồn khác (nginx, redis) không bị check
      
  authorities:
    - key:
        data: |
          -----BEGIN PUBLIC KEY-----
          MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEXSynXtPCQdDgDOjO7zdxDQCgw36D
          ioHMaKQ78DBPLDVtlr7jZTiSh1gQgfv4Ye+HSwifbPRf9Kn5R5c23Hef2A==
          -----END PUBLIC KEY-----
          # ^ Public key tương ứng với private key dùng để ký
          # ^ ECDSA P-256 public key (secp256r1)
          # ^ Ai có private key tương ứng mới tạo được chữ ký hợp lệ
          # ^ Public key an toàn để commit vào repo (chỉ dùng để verify)
          # ^ Nếu private key bị lộ → rotate key + update policy này
```

### 7.6 Cơ chế verify của Policy Controller

```
Pod được submit vào namespace "demo" (có label cosign.sigstore.dev/include: "true")
                │
                ▼
    MutatingWebhook của policy-controller intercept
                │
                ▼
    Policy controller check: image có match glob không?
    "ghcr.io/bearh141/w10-api-141:1.2.3" matches "ghcr.io/bearh141/*"?
                │
               YES
                │
                ▼
    Fetch signature từ OCI registry:
    Tag: sha256-<digest>.sig của image
                │
                ▼
    Verify signature với public key trong ClusterImagePolicy:
    ECDSA.Verify(publicKey, payload, signature)
                │
            ┌───┴───┐
           YES      NO
            │        │
           ALLOW    DENY (403)
            │
    Pod được tạo
```

### 7.7 Tại sao phải label namespace?

Sigstore Policy Controller KHÔNG áp dụng cho tất cả namespace mặc định. Phải opt-in bằng cách label namespace:

```bash
# Label namespace để enable policy enforcement
kubectl label namespace demo cosign.sigstore.dev/include=true

# Namespace không có label → policy controller BỎ QUA
# Namespace có label → policy controller VERIFY tất cả images
```

**Lý do thiết kế này:**
1. **Tránh break system namespaces:** `kube-system`, `argocd` dùng images không có chữ ký → nếu enforce toàn cluster sẽ crash
2. **Gradual rollout:** Có thể bật dần từng namespace, không phải tất cả cùng lúc
3. **Per-namespace control:** Môi trường dev có thể không require signing, production thì require

**Test signed vs unsigned image:**
```bash
# Pod dùng image đã ký (ghcr.io/bearh141/w10-api-141:0.0.2)
kubectl apply -f tests/pod-signed.yaml
# → PASS: chữ ký hợp lệ

# Pod dùng image không có chữ ký (nginx từ docker.io)
kubectl apply -f tests/pod-unsigned.yaml
# → Nginx không match glob "ghcr.io/bearh141/*" → policy không apply
# → PASS (nhưng Gatekeeper policies khác vẫn apply!)
# ^ Đây là điểm cần chú ý: policy chỉ enforce với images trong glob
```

### 7.8 Keypair và cryptography

**ECDSA P-256 (Elliptic Curve Digital Signature Algorithm):**
```
Private key:   32 bytes, giữ bí mật, dùng để KÝ
Public key:    65 bytes, chia sẻ công khai, dùng để VERIFY

Ký (signing):
  input = image digest (sha256:abc...)
  signature = ECDSA.Sign(private_key, SHA256(input))
  → Signature là proof rằng người có private_key đã sign input này

Verify:
  ECDSA.Verify(public_key, input, signature)
  → TRUE nếu signature được tạo bởi private_key tương ứng với public_key
  → FALSE nếu signature giả mạo hoặc image bị tamper
```

**Key format trong repo:**
- `signing/cosign.pub` → Public key (PEM format, ECDSA P-256)
- `signing/cosign.key` → Encrypted private key (không lưu trong ClusterImagePolicy)
- GitHub Secret `COSIGN_PRIVATE_KEY` → Nội dung của `cosign.key`

### 7.9 Tổng kết Supply Chain Security Flow

```
Điều kiện để một image được deploy vào cluster:

1. ✅ Source code được push lên GitHub (không bypass CI)
2. ✅ Trivy không tìm thấy HIGH/CRITICAL CVE
3. ✅ Pipeline đã ký image với cosign private key
4. ✅ Chữ ký verify được bằng public key trong ClusterImagePolicy
5. ✅ Image tag không phải :latest (Gatekeeper policy)
6. ✅ Pod có resource limits (Gatekeeper policy)
7. ✅ Pod không chạy as root (Gatekeeper policy)
8. ✅ Workload có owner label (Custom Gatekeeper policy)

Nếu fail BẤT KỲ điều kiện nào → Deployment bị block
```

---

## 8. BỘ CÂU HỎI VẤN ĐÁP (15 CÂU)

---

### Câu 1: Giải thích sự khác nhau giữa Role và ClusterRole trong Kubernetes RBAC. Trong lab này, tại sao alice dùng Role còn bob dùng ClusterRole?

**Trả lời:**

**Role vs ClusterRole:**
- `Role` là namespace-scoped: chỉ có hiệu lực trong 1 namespace cụ thể, được định nghĩa với `namespace` field trong metadata
- `ClusterRole` là cluster-scoped: áp dụng trên toàn cluster, có thể cấp quyền với cluster-level resources như nodes, namespaces, và các non-namespaced resources

**Trong lab:**
- **Alice** (`developer-workload-manager`) dùng **Role** vì:
  - Alice là developer chỉ làm việc trong namespace `demo`
  - Quyền của Alice giới hạn: quản lý pods, deployments, services trong `demo`
  - Principle of Least Privilege: không cần quyền cross-namespace
  - Nếu account bị compromise, blast radius giới hạn trong `demo`

- **Bob** (`sre-pod-operator`) dùng **ClusterRole** vì:
  - SRE cần troubleshoot ở **mọi namespace** khi có incident
  - Cần `kubectl exec` vào pod ở bất kỳ namespace nào
  - Cần xem nodes (cluster-level resource) để diagnose infrastructure
  - Không thể biết trước khi incident sẽ xảy ra ở namespace nào

- **Carol** (`platform-viewer`) dùng **ClusterRole** vì:
  - Cần visibility toàn bộ cluster cho monitoring và audit
  - Nhưng read-only (`get`, `list`, `watch` only) → risk thấp

---

### Câu 2: Admission Webhook là gì? Gatekeeper hook vào điểm nào trong pipeline?

**Trả lời:**

Admission Webhook là cơ chế mở rộng của Kubernetes cho phép code bên ngoài (external webhook) intercept các API request trước khi chúng được persist vào etcd.

**Pipeline xử lý request:**
```
kubectl apply → API Server → AuthN → AuthZ (RBAC) → Mutating Webhooks → Object Validation → Validating Webhooks → etcd
```

Có 2 loại:
1. **MutatingAdmissionWebhook:** Có thể sửa đổi object trước khi lưu (inject sidecar, add default labels)
2. **ValidatingAdmissionWebhook:** Chỉ có thể ALLOW hoặc DENY, không sửa đổi

**Gatekeeper là Validating Admission Webhook:**
- Hook vào bước cuối cùng (Validating Webhooks) trước khi persist
- Khi Gatekeeper deny → API Server trả về HTTP 403 với error message từ Rego
- Message hiển thị với user khi `kubectl apply`

**Sigstore Policy Controller là cả Mutating và Validating:**
- Mutating: inject annotation về verified signature
- Validating: verify signature, deny nếu không hợp lệ

---

### Câu 3: Giải thích ConstraintTemplate và Constraint trong Gatekeeper. Phân biệt vai trò của từng loại.

**Trả lời:**

**ConstraintTemplate:**
- Định nghĩa một loại policy mới
- Tạo ra CRD (Custom Resource Definition) mới trong K8s
- Chứa logic Rego – ngôn ngữ policy của OPA
- Tương tự như class definition trong OOP
- Ví dụ: `ConstraintTemplate` tên `k8sdisallowedtags` → tạo CRD `K8sDisallowedTags`

**Constraint:**
- Instance cụ thể của CRD được tạo bởi ConstraintTemplate
- Xác định: match conditions (áp dụng cho resource nào), parameters (cấu hình cụ thể), enforcementAction
- Tương tự như object instance trong OOP
- Ví dụ: `K8sDisallowedTags` constraint `disallow-latest-tag` với `parameters.tags: ["latest"]`

**Thứ tự bắt buộc:**
1. Deploy ConstraintTemplate trước (sync-wave 0) → tạo CRD
2. Deploy Constraint sau (sync-wave 1) → tạo instance

Nếu deploy Constraint trước khi có ConstraintTemplate → lỗi `no matches for kind "K8sDisallowedTags"` vì CRD chưa tồn tại.

---

### Câu 4: Giải thích logic Rego trong custom template `k8srequiredownerlabel`. Tại sao dùng `not` thay vì check điều kiện dương?

**Trả lời:**

```rego
package k8srequiredownerlabel

violation[{"msg": msg}] {
  not input.review.object.metadata.labels.owner
  msg := "workload must have metadata.labels.owner"
}
```

**Rego sử dụng "Negation as Failure" (NAF):**
- Rego là ngôn ngữ khai báo (declarative), không phải imperative
- Rule `violation` được thiết kế để fire khi có vi phạm
- `not X` trong Rego = "X không tồn tại hoặc là falsy"

**Logic flow:**
- Nếu `labels.owner` = "team-a" → `not "team-a"` = false → rule không fire → NO violation → ALLOW
- Nếu `labels.owner` không tồn tại → evaluates to `undefined` → `not undefined` = true → rule fires → VIOLATION → DENY
- Nếu `labels.owner` = "" (empty) → `not ""` = true → VIOLATION → DENY

**Tại sao không viết ngược:**
```rego
# Sai (logic không đúng trong Rego):
allow {
  input.review.object.metadata.labels.owner
}
# Rego không có "allow" rule theo mặc định
# Gatekeeper chỉ check "violation" rule
# Không có violation = allow
```

**Kết luận:** Trong Gatekeeper, bạn định nghĩa khi nào thì vi phạm (`violation`), không phải khi nào thì được phép. Absence of violation = allowed.

---

### Câu 5: ESO SecretStore vs ExternalSecret khác nhau như thế nào? Và creationPolicy: Owner có nghĩa gì?

**Trả lời:**

**SecretStore:**
- Định nghĩa **kết nối** đến external secret provider (AWS, Vault, GCP...)
- Chứa credentials để authenticate với provider
- Reusable: 1 SecretStore có thể dùng cho nhiều ExternalSecret
- Namespace-scoped (hoặc ClusterSecretStore cho cross-namespace)
- Tương tự: database connection string

**ExternalSecret:**
- Định nghĩa **mapping** từ external secret sang K8s Secret
- Chỉ định: key nào trong AWS SM, map sang field nào trong K8s Secret
- Output: tạo ra 1 K8s Secret thực sự trong cluster
- Tương tự: database query

**creationPolicy: Owner:**
- `Owner`: ESO "owns" K8s Secret tạo ra. Nếu ExternalSecret bị xóa → K8s Secret cũng bị xóa (garbage collection)
- `Orphan`: K8s Secret không bị xóa khi ExternalSecret bị xóa
- `Merge`: Merge data vào K8s Secret đã tồn tại (giữ existing keys)
- `None`: Không tạo secret, chỉ update nếu đã tồn tại

**`Owner` phù hợp cho lab này vì:**
Lifecycle của Secret phải gắn với ExternalSecret. Nếu ExternalSecret bị remove khỏi Git và ArgoCD prune nó, Secret cũng phải bị xóa để tránh orphaned secrets chứa credentials.

---

### Câu 6: Tại sao refreshInterval của ESO là 10 giây? Vấn đề gì khi đặt quá ngắn?

**Trả lời:**

**Tại sao 10 giây trong lab:**
- Demo purposes: muốn thấy rotation ngay lập tức
- Trong lab test: sau khi update AWS SM → 10s sau pod đã có secret mới

**Vấn đề khi đặt quá ngắn (production):**
1. **AWS API throttling:** AWS Secrets Manager có rate limit (~10,000 requests/region/account/second). Với 100 pods × 10s refresh = 10 req/s, còn OK. Nhưng 10,000 pods × 10s = 1000 req/s, có thể gặp `ThrottlingException`
2. **Cost:** AWS SM tính phí per API call. 10s × 3600s × 24h × 365 days = 3.15 triệu calls/secret/năm
3. **ESO operator load:** Controller phải handle nhiều concurrent refresh

**Production recommendation:**
- `refreshInterval: 1h` – đủ nhanh cho rotation
- Dùng **IRSA** thay vì static credentials cho scalability
- Monitor AWS SM API call count qua CloudWatch

---

### Câu 7: Trivy là gì? Tại sao dùng `--ignore-unfixed` và `--exit-code 1`?

**Trả lời:**

**Trivy** là open-source vulnerability scanner của Aqua Security. Nó quét:
- OS packages (alpine apk, debian dpkg, ubuntu apt)
- Language dependencies (Python pip, Node npm, Java maven, Go modules)
- Infrastructure as Code (Dockerfile, Terraform, K8s manifests)
- Secrets (hardcoded API keys, passwords trong code)

**CVE (Common Vulnerabilities and Exposures):** Danh sách các lỗ hổng bảo mật đã biết, mỗi CVE có ID dạng `CVE-2024-12345` và CVSS score.

**`--ignore-unfixed`:**
- Bỏ qua CVE chưa có upstream patch
- Lý do: developer không thể fix CVE nếu library chưa release patch
- Nếu không có flag này → pipeline liên tục fail dù không có gì có thể làm
- Triết lý: chỉ fail khi có actionable item (có fix mà chưa update)

**`--exit-code 1`:**
- Khi tìm thấy CVE với severity trong `--severity` → exit code = 1 → GitHub Actions job fail
- Fail early trong pipeline tốt hơn là deploy image có CVE lên production
- Nếu đặt `--exit-code 0` → Trivy chỉ report, không block pipeline

**Tại sao `python:3.13-alpine` trong Dockerfile:**
- Alpine Linux có ít packages hơn → ít CVE hơn so với Ubuntu/Debian
- Image nhỏ hơn → build nhanh hơn, ít attack surface hơn

---

### Câu 8: Cosign hoạt động như thế nào? Private key lưu ở đâu, public key dùng để làm gì?

**Trả lời:**

**Cosign** là tool của Sigstore project để ký và verify container images.

**Cơ chế:**
1. **Generate keypair:** `cosign generate-key-pair` → `cosign.key` (private) + `cosign.pub` (public)
2. **Sign:** `cosign sign --key cosign.key image:tag`
   - Tính digest của image manifest (sha256:...)
   - Tạo ECDSA signature từ digest + private key
   - Push signature vào OCI registry dưới dạng `image:sha256-<digest>.sig`
3. **Verify:** `cosign verify --key cosign.pub image:tag`
   - Fetch signature từ registry
   - Verify signature dùng public key
   - Nếu match → image không bị tamper và đến từ người có private key

**Private key lưu ở đâu:**
- **KHÔNG** commit vào Git
- Lưu trong **GitHub Secrets** với tên `COSIGN_PRIVATE_KEY`
- Trong CI, đọc từ env var: `--key env://COSIGN_PRIVATE_KEY`
- Encrypted bằng passphrase (trong `COSIGN_PASSWORD` secret)

**Public key dùng để làm gì:**
- Commit vào repo (an toàn vì không thể tạo chữ ký từ public key)
- Nhúng vào `ClusterImagePolicy` trong field `authorities[].key.data`
- Sigstore Policy Controller dùng public key này để verify chữ ký runtime

**Trust model:**
```
Chỉ pipeline CI mới có private key
→ Chỉ image được build bởi pipeline mới có chữ ký hợp lệ
→ Cluster chỉ chạy image từ pipeline (đã qua Trivy scan)
→ Không ai có thể push image giả mạo lên cluster
```

---

### Câu 9: ClusterImagePolicy glob pattern hoạt động như thế nào? Điều gì xảy ra với image không match glob?

**Trả lời:**

**Glob pattern `ghcr.io/bearh141/*`:**
```
ghcr.io/bearh141/w10-api-141:1.2.3    → MATCH (phải verify signature)
ghcr.io/bearh141/any-image:tag         → MATCH
ghcr.io/bearh141/                      → MATCH (bất kỳ gì sau /)
docker.io/library/nginx:1.25           → NO MATCH
quay.io/prometheus/prometheus:latest   → NO MATCH
ghcr.io/otherowner/image:tag           → NO MATCH
```

**Điều gì xảy ra với image KHÔNG match glob:**
- Policy Controller **BỎ QUA** (không verify, không block)
- Image được phép chạy bình thường (không cần có chữ ký)
- Ví dụ: `nginx:1.25.1` trong test files không match → PASS

**Ý nghĩa thiết kế:**
- Chỉ enforce signing với images của project này (`bearh141/`)
- Third-party images (nginx, redis, postgres) không có cosign signature → nếu enforce toàn bộ sẽ break mọi deployment
- Gradual adoption: bắt đầu với internal images, sau mở rộng ra

**Test case `pod-unsigned.yaml`:**
```yaml
image: docker.io/library/nginx:1.25.1
# → Không match glob "ghcr.io/bearh141/*"
# → Policy controller không check
# → PASS (nhưng Gatekeeper vẫn check resource limits, non-root, etc.)
```

---

### Câu 10: Giải thích sync-wave trong ArgoCD. Nếu deploy Gatekeeper constraints trước khi cài Gatekeeper, lỗi gì xảy ra và tại sao?

**Trả lời:**

**Sync-wave là gì:**
ArgoCD sử dụng annotation `argocd.argoproj.io/sync-wave` để ordering resource deployment. Số wave thấp hơn → deploy trước. ArgoCD đợi resource của wave trước `Healthy` trước khi chuyển sang wave tiếp theo.

**Trong repo:**
- Wave -1: RBAC resources (roles, bindings)
- Wave 0: Helm charts (Gatekeeper, ESO, policy-controller)
- Wave 1: Custom resources (ConstraintTemplates, Constraints, SecretStore, ClusterImagePolicy)

**Nếu deploy constraints trước Gatekeeper:**

Lỗi cụ thể:
```
Error from server (NotFound): error when creating "gatekeeper/constraints/disallow-latest-tag.yaml":
no matches for kind "K8sDisallowedTags" in version "constraints.gatekeeper.sh/v1beta1"
```

**Tại sao:**
1. Gatekeeper khi cài sẽ đăng ký CRD `K8sDisallowedTags`, `K8sRequiredResources`, etc. vào K8s API Server
2. Nếu Gatekeeper chưa được cài → những CRD này chưa tồn tại trong cluster
3. Khi `kubectl apply` Constraint → K8s API Server không biết `K8sDisallowedTags` là gì
4. `SkipDryRunOnMissingResource=true` trong ArgoCD giải quyết vấn đề dry-run nhưng không giải quyết race condition: nếu Gatekeeper chưa register CRD kịp, apply sẽ vẫn fail

---

### Câu 11: Giải thích `enforcementAction: deny` vs `warn` vs `dryrun`. Khi nào dùng loại nào?

**Trả lời:**

| enforcementAction | Behavior | Use case |
|---|---|---|
| `deny` | Block request hoàn toàn, trả về 403 | Production, khi policy đã được test kỹ |
| `warn` | Cho phép qua, nhưng ghi warning vào response | Rollout policy mới, thông báo developer |
| `dryrun` | Chỉ ghi vào audit log, không block, không warn | Test policy mới, đánh giá impact |

**Best practice rollout policy mới:**
```
Phase 1: enforcementAction: dryrun
  → Monitor audit log: bao nhiêu violation?
  → Không ảnh hưởng production
  
Phase 2: enforcementAction: warn
  → Developer nhận warning khi deploy
  → Thời gian để fix existing deployments
  
Phase 3: enforcementAction: deny
  → Hard enforcement
  → Chỉ sau khi tất cả existing deployments đã compliant
```

**Trong lab này:** Tất cả constraints dùng `deny` vì đây là lab setup từ đầu, không có legacy workloads cần migration.

---

### Câu 12: Tại sao ESO cần `aws-creds` Secret nhưng Secret đó không được lưu trong Git? Giải thích chicken-and-egg problem này.

**Trả lời:**

**Chicken-and-egg problem:**
- ESO cần `aws-creds` Secret để pull secrets từ AWS
- Nhưng `aws-creds` chứa AWS credentials → không thể commit vào Git
- ArgoCD sync từ Git → không có `aws-creds` trong Git → ArgoCD không biết secret này

**Giải pháp trong lab:**
`aws-creds` Secret phải được tạo **thủ công** trước khi ESO được cài:
```bash
kubectl create secret generic aws-creds \
  --namespace demo \
  --from-literal=access-key="AKIAIOSFODNN7EXAMPLE" \
  --from-literal=secret-key="wJalrXUtnFEMI/K7MDENG..."
```

**Giải pháp production (tốt hơn):**
1. **IRSA (IAM Roles for Service Accounts):** ESO ServiceAccount có annotation với IAM Role ARN. Pod nhận JWT token từ K8s OIDC provider, AWS STS exchange token lấy credentials tạm thời. Không cần lưu credentials trong cluster.

2. **HashiCorp Vault:** Lưu bootstrap secret trong Vault, ESO đọc từ Vault. Vault seal/unseal được quản lý riêng.

3. **AWS SSM Parameter Store với IAM:** EC2 node có IAM role → ESO inherit qua IRSA.

**Trong file `eso/secret-store.yaml`:** Phần `auth.secretRef` tham chiếu đến Secret `aws-creds`. Nếu Secret này không tồn tại → ESO controller báo lỗi `secret "aws-creds" not found` và không thể sync.

---

### Câu 13: Mô tả toàn bộ flow từ khi developer push code đến khi image chạy trong cluster, nhấn mạnh các "gate" bảo mật.

**Trả lời:**

```
1. Developer: git push origin main (thay đổi src/api/app.py)

2. GitHub Actions trigger:
   Gate 1 → Trivy scan: tìm CVE HIGH/CRITICAL với --ignore-unfixed
            ↓ PASS → tiếp tục
            ↓ FAIL → pipeline stop, image không được deploy

3. Cosign sign:
   Gate 2 → Ký image với private key (chỉ pipeline mới có)
            Signature lưu vào registry
            
4. Update rollout.yaml với version mới, commit + push

5. ArgoCD detect thay đổi (3 phút hoặc webhook):
   Sync theo wave: -1 → 0 → 1
   Apply Rollout với image mới

6. K8s API Server nhận Rollout:
   Gate 3 → RBAC check: ArgoCD ServiceAccount có quyền tạo Rollout không?
   
   Gate 4 → Gatekeeper (Validating Webhook):
            - Image tag không phải :latest? ✓
            - Resource limits có không? ✓  
            - Không chạy as root? ✓
            - Không dùng hostNetwork? ✓
            - Có owner label? ✓
            ↓ Tất cả pass → tiếp tục
            ↓ Bất kỳ fail → 403, Rollout không được tạo

   Gate 5 → Sigstore Policy Controller (Validating Webhook):
            - Image match glob "ghcr.io/bearh141/*"?
            - Cosign signature hợp lệ với public key?
            ↓ VALID → ALLOW
            ↓ INVALID → 403

7. Rollout được tạo, Argo Rollouts controller deploy pod mới
   Pod mount db-secret từ ESO (pull từ AWS SM)
```

**Tổng cộng: 5 security gates** từ code đến running container.

---

### Câu 14: Giải thích `argocd/root.yaml` và App-of-Apps pattern. Tại sao dùng pattern này thay vì apply manifest trực tiếp?

**Trả lời:**

**App-of-Apps pattern:**
Thay vì quản lý từng resource riêng lẻ, tạo một ArgoCD Application "root" quản lý các child Applications. Mỗi child Application lại quản lý các K8s resources.

```yaml
# root.yaml – Application "root"
spec:
  source:
    path: argocd/apps    # ← Folder chứa tất cả child app definitions
  destination:
    namespace: argocd    # ← Deploy child apps vào namespace argocd
  syncPolicy:
    automated:
      prune: true        # Xóa child app nếu xóa file khỏi argocd/apps/
      selfHeal: true     # Revert nếu ai manual sửa child app
```

**Lợi ích:**
1. **Single source of truth:** Chỉ cần apply 1 file `root.yaml` ban đầu. Sau đó mọi thứ tự động sync từ Git
2. **Bootstrap dễ:** Cluster mới chỉ cần `kubectl apply -f argocd/root.yaml`
3. **Dependency management:** Sync-wave trong child apps kiểm soát thứ tự deploy
4. **Visibility:** ArgoCD UI hiển thị toàn bộ application tree
5. **Separation of concern:** Mỗi team quản lý file yaml của app của họ, root app tự detect

**So sánh với alternative (kustomize/helm trực tiếp):**
- Kustomize: không có dependency management, không có sync-wave
- Helm: mỗi chart deploy độc lập, khó kiểm soát thứ tự
- App-of-Apps: declarative, self-healing, dependency-aware

---

### Câu 15: Nếu ESO SecretStore bị misconfigure (sai region hoặc sai credentials), điều gì xảy ra với pod đang chạy và pod mới?

**Trả lời:**

**Kịch bản: ESO không thể connect AWS Secrets Manager**

**Pod đang chạy (existing pods):**
- **Không bị ảnh hưởng ngay lập tức**
- K8s Secret `db-secret` vẫn tồn tại trong etcd với giá trị cũ
- Pod mount secret qua volume → file vẫn có giá trị cũ
- Pod hoạt động bình thường cho đến khi secret rotate và giá trị cũ expire

**ExternalSecret status:**
```bash
kubectl get externalsecret db-creds -n demo
# STATUS: SecretSyncedError
# Condition: Ready = False
# Message: "could not get secret from SecretStore: ... AccessDeniedException"
```

**ESO behavior:**
- ESO sẽ retry theo `refreshInterval`
- Log error trong ESO controller pod
- K8s Event được tạo: `Warning SecretSyncedError ...`

**Pod mới (chưa được tạo):**
- Nếu Secret `db-secret` đã tồn tại (từ trước) → pod mới vẫn mount được, start OK
- Nếu Secret chưa tồn tại (cluster mới) → `creationPolicy: Owner` → Secret không được tạo
- Pod start fail với: `Error: secret "db-secret" not found`
- Argo Rollouts sẽ không promote sang new revision nếu pod health check fail

**Cách debug:**
```bash
# Xem ExternalSecret status
kubectl describe externalsecret db-creds -n demo

# Xem ESO controller logs
kubectl logs -n external-secrets deploy/external-secrets

# Xem K8s Events
kubectl get events -n demo --sort-by=.lastTimestamp
```

**Fix:**
1. Kiểm tra `aws-creds` Secret có đúng credentials không
2. Kiểm tra IAM policy của user/role có permission `secretsmanager:GetSecretValue` không
3. Kiểm tra VPC endpoint hoặc network connectivity tới AWS SM endpoint
4. Kiểm tra secret name `w10/demo/db-password` có tồn tại trong đúng region không

---

## TỔNG KẾT

| Lab | Component | Mục đích | Phụ thuộc |
|---|---|---|---|
| 1.1 | RBAC | Phân quyền user | Không |
| 1.2 | Gatekeeper | Enforce cluster policies | Gatekeeper Helm (wave 0) |
| 1.3 | Custom Rego | Business-specific policy | Gatekeeper (wave 0) |
| 2.1 | ESO | Sync secrets từ AWS | ESO Helm (wave 0), aws-creds Secret |
| 2.2 | Trivy | CVE scan trong CI | GitHub Actions |
| 2.2 | Cosign | Sign image trong CI | cosign keypair |
| 2.2 | Policy Controller | Verify signature runtime | policy-controller Helm (wave 0) |

**Các nguyên tắc bảo mật được áp dụng:**
1. **Principle of Least Privilege** – RBAC, Resource Limits
2. **Defense in Depth** – Multiple security layers (RBAC + Gatekeeper + Cosign)
3. **Shift Left Security** – Trivy scan trong CI, fail fast
4. **Supply Chain Security** – Cosign signature verification
5. **Secrets Management** – ESO + AWS Secrets Manager, không lưu secret trong Git
6. **GitOps** – ArgoCD, self-healing, declarative config

---

> *Tài liệu được tạo dựa trên code thực tế của repo Lab1806*
> *Repo: https://github.com/bearh141/Lab1806*
