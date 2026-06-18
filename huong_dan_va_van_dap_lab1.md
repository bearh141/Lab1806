# Hướng dẫn chi tiết & Bộ câu hỏi vấn đáp: Lab 1.1, Lab 1.2, & Lab 1.3

Tài liệu này tổng hợp toàn bộ thông tin chi tiết về mục đích, các bước cài đặt, giải thích chi tiết từng file cấu hình (dòng qua dòng, block qua block) và bộ câu hỏi ôn tập vấn đáp cho các bài Lab buổi sáng:
* **Lab 1.1 - RBAC:** Phân quyền 3 vai trò qua GitOps (Alice, Bob, Carol).
* **Lab 1.2 - OPA Gatekeeper:** Triển khai 4 luật chặn manifest xấu tại Admission Controller.
* **Lab 1.3 - Custom Policy:** Viết custom ConstraintTemplate (Rego) bắt buộc khai báo label `owner`.

---

## PHẦN 1: TỔNG QUAN CÁC BÀI LAB

### 1. Lab 1.1: Phân quyền 3 vai trò qua GitOps (RBAC)
* **Mục đích:** Thay thế trạng thái "ai cũng là admin" bằng nguyên tắc đặc quyền tối thiểu (Least Privilege). Triển khai phân quyền tự động hoàn toàn qua GitOps (Argo CD), không chạy lệnh apply thủ công bằng tay.
* **Chi tiết phân quyền:**
  * **Alice (Developer):** CRUD (Create/Read/Update/Delete) các tài nguyên ứng dụng (Deployments, Pods, Services, Rollouts) chỉ duy nhất trong namespace `demo`.
  * **Bob (SRE):** Xem và thao tác (Create/Read/Update/Delete/Exec/Log) với Pod ở mọi namespace trên toàn cụm. Chỉ xem được các tài nguyên khác (Deployments, Rollouts, Namespaces, Nodes) nhưng không được sửa đổi.
  * **Carol (Viewer/Platform Team):** Chỉ đọc (get/list/watch) toàn bộ tài nguyên trên toàn cụm (Cluster-wide), không có quyền ghi hay sửa đổi bất kỳ tài nguyên nào.

### 2. Lab 1.2: Triển khai OPA Gatekeeper và 4 luật chặn Admission
* **Mục đích:** Thiết lập chốt chặn an toàn (Guardrails) ngăn chặn các lập trình viên vô tình hay cố ý triển khai các manifest cấu hình sai cấu trúc bảo mật lên Cluster.
* **4 Luật bảo mật cần enforce:**
  1. **Cấm Image Tag `:latest`:** Bắt buộc ứng dụng phải pin phiên bản cụ thể (ví dụ: `0.0.2`), tránh việc tự động cập nhật ảnh lỗi khi Pod restart.
  2. **Bắt buộc khai báo CPU/Memory Limits:** Đảm bảo tài nguyên cluster được kiểm soát, tránh hiện tượng một pod lỗi tiêu thụ hết tài nguyên của toàn bộ Node.
  3. **Cấm chạy Container bằng quyền Root (`runAsUser: 0`):** Ngăn chặn đặc quyền root bên trong container để tránh tấn công thoát khỏi container (Container Escape).
  4. **Cấm HostNetwork (`hostNetwork: true`):** Không cho phép Pod sử dụng trực tiếp card mạng của máy chủ vật lý, cô lập mạng của ứng dụng.

### 3. Lab 1.3: Viết Custom ConstraintTemplate (Rego)
* **Mục đích:** Tự thiết kế một chính sách tùy biến cho dự án mà thư viện mẫu của OPA Gatekeeper không có sẵn.
* **Yêu cầu triển khai:** Viết một Template sử dụng ngôn ngữ Rego bắt buộc mọi Deployment/Rollout được tạo ra trên cluster phải khai báo label `owner` (ví dụ: `owner: platform-team`), giúp quản lý và quy trách nhiệm rõ ràng khi hệ thống gặp sự cố.

---

## PHẦN 2: TỪNG BƯỚC TRIỂN KHAI CHI TIẾT (STEP-BY-STEP)

### Bước 1: Deploy cấu hình RBAC qua Argo CD
Cấu hình rbac được lưu trong thư mục `rbac/`. Argo CD sẽ tự động phát hiện và đồng bộ file `argocd/apps/rbac.yaml` trỏ tới đường dẫn này, tự động khởi tạo các Role và RoleBinding trong cluster.

### Bước 2: Cài đặt OPA Gatekeeper Operator
Để tránh lỗi thiếu CRD khi cài đặt đồng thời, chúng ta sử dụng cơ chế **Sync Waves** trong Argo CD:
1. File [argocd/apps/gatekeeper.yaml](file:///d:/Download/AWS/Lab1806/argocd/apps/gatekeeper.yaml) được gán wave nhỏ hơn (`sync-wave: "0"`) để cài đặt bộ điều khiển Gatekeeper trước.
2. File [argocd/apps/gatekeeper-policies.yaml](file:///d:/Download/AWS/Lab1806/argocd/apps/gatekeeper-policies.yaml) được gán wave lớn hơn (`sync-wave: "1"`) để cài đặt các ConstraintTemplate và Constraint sau khi các CRD của Gatekeeper đã sẵn sàng.

### Bước 3: Đẩy mã nguồn lên Git
Commit toàn bộ cấu hình vào Git. Argo CD sẽ tự động đồng bộ hóa trạng thái Cluster của bạn về trạng thái xanh.

### Bước 4: Kiểm tra phân quyền RBAC
Chạy các lệnh giả lập quyền (`--as`) để xác thực RBAC hoạt động chính xác:
```bash
# Alice có CRUD trong demo không? (Kỳ vọng: yes)
kubectl auth can-i create deployments -n demo --as alice

# Alice có CRUD ngoài demo không? (Kỳ vọng: no)
kubectl auth can-i create deployments -n kube-system --as alice

# Bob có đọc được pods toàn cụm không? (Kỳ vọng: yes)
kubectl auth can-i get pods -A --as bob

# Carol có xóa được node không? (Kỳ vọng: no)
kubectl auth can-i delete nodes --as carol
```

### Bước 5: Kiểm tra các luật chặn của Gatekeeper
Chạy thử các lệnh apply file test trong thư mục `tests/` để xác nhận các vi phạm bị chặn đứng:
```bash
# 1. Test cấm tag :latest (Kỳ vọng: Bị reject)
kubectl apply -f tests/pod-latest.yaml

# 2. Test bắt buộc resource limits (Kỳ vọng: Bị reject)
kubectl apply -f tests/pod-no-limits.yaml

# 3. Test cấm quyền root (Kỳ vọng: Bị reject)
kubectl apply -f tests/pod-root-user.yaml

# 4. Test cấm host network (Kỳ vọng: Bị reject)
kubectl apply -f tests/pod-hostnetwork.yaml

# 5. Test deploy thiếu label owner (Kỳ vọng: Bị reject)
kubectl apply -f tests/deploy-no-owner.yaml

# 6. Test Pod hợp lệ (Kỳ vọng: Tạo thành công)
kubectl apply -f tests/pod-valid.yaml
```

---

## PHẦN 3: GIẢI THÍCH CHI TIẾT DÒNG QUA DÒNG CÁC FILE CẤU HÌNH

### 1. File [rbac/roles.yaml](file:///d:/Download/AWS/Lab1806/rbac/roles.yaml) (Phần vai trò Developer của Alice)
```yaml
1: apiVersion: rbac.authorization.k8s.io/v1      # Phiên bản API của hệ thống RBAC K8s
2: kind: Role                                    # Định nghĩa vai trò có phạm vi Namespace
3: metadata:
4:   name: developer-workload-manager            # Tên vai trò
5:   namespace: demo                             # Chỉ có hiệu lực bên trong namespace "demo"
6: rules:
7:   - apiGroups: ["", "apps", "argoproj.io"]    # Các nhóm API mà vai trò này được phép truy cập
8:     resources:                                # Danh sách các tài nguyên được phép thao tác
9:       - pods
10:       - pods/log                              # Cho phép đọc log của pod
11:       - services                              # Cho phép quản lý service kết nối
12:       - configmaps
13:       - deployments                           # Cho phép quản lý Deployments
14:       - replicasets
15:       - rollouts                              # Cho phép chạy Rollouts (Argo Rollouts)
16:     verbs: ["get", "list", "watch", "create", "update", "patch", "delete"] # CRUD verbs
```

### 2. File [rbac/rolebindings.yaml](file:///d:/Download/AWS/Lab1806/rbac/rolebindings.yaml) (Liên kết Alice)
```yaml
1: apiVersion: rbac.authorization.k8s.io/v1      # Phiên bản API RBAC
2: kind: RoleBinding                             # Định nghĩa liên kết vai trò phạm vi Namespace
3: metadata:
4:   name: alice-developer-workload-manager      # Tên của liên kết
5:   namespace: demo                             # Đặt trong namespace "demo"
6: subjects:
7:   - kind: User                                # Áp dụng liên kết này cho một tài khoản User
8:     name: alice                               # Tên tài khoản nhận quyền là "alice"
9:     apiGroup: rbac.authorization.k8s.io
10: roleRef:
11:   kind: Role                                 # Loại liên kết đến là Role thông thường
12:   name: developer-workload-manager           # Tham chiếu đến Role "developer-workload-manager"
13:   apiGroup: rbac.authorization.k8s.io
```

### 3. File [gatekeeper/templates/k8srequiredownerlabel.yaml](file:///d:/Download/AWS/Lab1806/gatekeeper/templates/k8srequiredownerlabel.yaml) (Luật viết tay Lab 1.3)
```yaml
1: apiVersion: templates.gatekeeper.sh/v1        # API của OPA Gatekeeper Template
2: kind: ConstraintTemplate                      # Định nghĩa một khuôn mẫu chính sách (Rego)
3: metadata:
4:   name: k8srequiredownerlabel                 # Tên của template chính sách
5:   annotations:
6:     argocd.argoproj.io/sync-wave: "0"         # Cài đặt template trước ở wave 0
7: spec:
8:   crd:
9:     spec:
10:       names:
11:         kind: K8sRequiredOwnerLabel          # Tên Kind mới sẽ được đăng ký vào K8s API
12:   targets:
13:     - target: admission.k8s.gatekeeper.sh    # Đích tác động là bộ điều khiển admission của K8s
14:       rego: |                                # Khai báo mã ngôn ngữ Rego
15:         package k8srequiredownerlabel        # Khai báo package trùng tên template
16: 
17:         violation[{"msg": msg}] {            # Định nghĩa điều kiện vi phạm
18:           not input.review.object.metadata.labels.owner # Nếu đối tượng KHÔNG có label "owner"
19:           msg := "workload must have metadata.labels.owner" # Thông báo lỗi trả về cho client
20:         }
```

### 4. File [gatekeeper/constraints/require-owner-label.yaml](file:///d:/Download/AWS/Lab1806/gatekeeper/constraints/require-owner-label.yaml) (Enforce Luật Lab 1.3)
```yaml
1: apiVersion: constraints.gatekeeper.sh/v1beta1 # API của OPA Gatekeeper Constraint
2: kind: K8sRequiredOwnerLabel                   # Kind được sinh ra từ ConstraintTemplate ở trên
3: metadata:
4:   name: require-owner-label                   # Tên tài nguyên ràng buộc
5:   annotations:
6:     argocd.argoproj.io/sync-wave: "1"         # Áp dụng ràng buộc sau ở wave 1 (khi template đã có)
7: spec:
8:   match:
9:     kinds:
10:       - apiGroups: ["apps"]                   # Áp dụng cho các tài nguyên thuộc group apps
11:         kinds: ["Deployment"]                 # Cụ thể là Deployment
12:       - apiGroups: ["argoproj.io"]            # Và các tài nguyên thuộc group argoproj.io
13:         kinds: ["Rollout"]                    # Cụ thể là Rollout
14:     excludedNamespaces:                       # Loại trừ các namespace hệ thống để tránh lỗi tự sập
15:       - kube-system
16:       - argocd
17:       - external-secrets
18:       - cosign-system
```

---

## PHẦN 4: BỘ CÂU HỎI VẤN ĐÁP ÔN TẬP (VIVA Q&A)

### Câu 1: Sự khác nhau bản chất giữa `Role` / `RoleBinding` và `ClusterRole` / `ClusterRoleBinding` là gì?
**Trả lời:**
* **`Role` & `RoleBinding`:** Là tài nguyên có phạm vi **Namespace** (Namespace-scoped). `Role` khai báo quyền trong một namespace cụ thể, và `RoleBinding` gán các quyền đó cho User trong namespace đó.
* **`ClusterRole` & `ClusterRoleBinding`:** Là tài nguyên có phạm vi **toàn cụm** (Cluster-scoped). `ClusterRole` định nghĩa các quyền trên toàn cluster, và `ClusterRoleBinding` gán quyền đó cho User trên mọi namespace của cluster.
* **Trường hợp đặc biệt (RoleBinding tham chiếu đến ClusterRole):** Ta có thể dùng `RoleBinding` ở namespace `demo` tham chiếu đến một `ClusterRole` dùng chung (như `view`). Khi đó, User chỉ nhận được quyền xem bên trong namespace `demo` đó mà thôi.

### Câu 2: Tại sao vai trò của Bob (SRE) cần cấu hình là `ClusterRoleBinding` chứ không phải `RoleBinding`?
**Trả lời:**
Bob là kỹ sư SRE có nhiệm vụ kiểm tra, giám sát hệ thống và cần quyền xem thông tin Pod, logs cũng như chạy lệnh exec ở **mọi namespace trên toàn cụm** (như `kube-system`, `monitoring`, `demo`, v.v.). Nếu dùng `RoleBinding`, chúng ta sẽ phải tạo thủ công `RoleBinding` ở từng namespace một cho Bob, điều này gây khó khăn cho việc quản lý và bảo trì. Do đó, sử dụng `ClusterRoleBinding` là cách duy nhất để cấp quyền xem và tương tác Pod cho Bob trên toàn bộ cluster chỉ bằng một khai báo duy nhất.

### Câu 3: OPA Gatekeeper hoạt động dựa trên cơ chế nào trong Kubernetes? Phân biệt `ConstraintTemplate` và `Constraint`.
**Trả lời:**
OPA Gatekeeper hoạt động như một **Validating Admission Webhook**. Khi API Server nhận được yêu cầu tạo/sửa đổi tài nguyên, nó sẽ chuyển tiếp manifest của tài nguyên đó đến Gatekeeper để thẩm định.
* **`ConstraintTemplate`:** Là nơi định nghĩa **thuật toán logic** kiểm tra lỗi bảo mật bằng ngôn ngữ **Rego** và đăng ký một Custom Resource Definition (CRD) mới (ví dụ `K8sRequiredOwnerLabel`). Nó giống như việc viết ra một "hàm" hay "khuôn mẫu".
* **`Constraint`:** Là một **thực thể áp dụng** cụ thể được sinh ra từ template đó. Ràng buộc này chỉ ra phạm vi áp dụng (ví dụ: áp dụng cho tài nguyên `Deployment` ở các namespace, ngoại trừ `kube-system`). Nó giống như việc truyền "tham số" vào hàm để thực thi chính sách trong thực tế.

### Câu 4: Tại sao trong bài Lab 1.2, chúng ta không gộp chung cấu hình cài đặt Gatekeeper Operator và các file Constraints vào cùng một lượt đồng bộ (Argo CD sync)?
**Trả lời:**
Khi cài đặt Gatekeeper Operator, hệ thống sẽ đăng ký các CustomResourceDefinitions (CRDs) của OPA Gatekeeper (như `ConstraintTemplate`). Tại thời điểm bắt đầu đồng bộ, nếu CRD chưa tồn tại trên Kubernetes API Server, thì việc apply các manifest chứa loại tài nguyên ConstraintTemplate hay Constraint ngay lập tức sẽ bị lỗi vì API Server không hiểu được Kind đó (`no matches for kind...`). Bằng cách sử dụng **Sync Waves** (chỉ định cài operator trước ở wave 0 và chính sách sau ở wave 1), Argo CD sẽ chờ operator cài đặt thành công và đăng ký xong CRD rồi mới tiếp tục apply các Constraint, giúp quy trình tự động hóa không bị lỗi ngắt quãng.

### Câu 5: Lệnh `kubectl auth can-i` hoạt động dựa trên cơ chế nào? Làm thế nào để phân biệt giữa Authorization và Authentication thông qua lệnh này?
**Trả lời:**
* Lệnh `kubectl auth can-i <action> --as <user>` gửi một yêu cầu truy vấn đặc biệt (`SelfSubjectAccessReview` hoặc `SubjectAccessReview`) tới Kubernetes API Server. API Server sẽ chạy mô phỏng chính sách RBAC đã được định nghĩa trong cluster và trả về kết quả `yes` hoặc `no` xem User đó có quyền thực thi hành động đó không.
* Lệnh này chỉ phục vụ mục đích kiểm tra **Authorization** (Phân quyền - User được làm gì).
* Lệnh này **không thực hiện Authentication** (Xác thực - User là ai, có mật khẩu/token đúng không). Do đó, tham số `--as` chỉ là giả lập danh tính (Impersonation) của admin chứ không cần chứng chỉ bảo mật thật của Alice, Bob hay Carol để kiểm tra.
