# Hướng dẫn chi tiết & Bộ câu hỏi vấn đáp: Lab 2.1 & Lab 2.2

Tài liệu này tổng hợp toàn bộ thông tin chi tiết về mục đích, các bước cài đặt, giải thích chi tiết từng file cấu hình (dòng qua dòng, block qua block) và bộ câu hỏi ôn tập vấn đáp cho hai bài Lab:
* **Lab 2.1:** Quản lý và xoay vòng Secret tự động không cần restart Pod (External Secrets Operator - ESO).
* **Lab 2.2:** Bảo mật chuỗi cung ứng phần mềm (Trivy scan + Cosign sign + Sigstore Policy Controller admission webhook).

---

## PHẦN 1: TỔNG QUAN HAI BÀI LAB

### 1. Lab 2.1: Rotate Secret không restart Pod (ESO)
* **Mục đích:** Giải quyết vấn đề lộ thông tin nhạy cảm (như mật khẩu cơ sở dữ liệu) trong mã nguồn/Git (GitOps) và loại bỏ việc phải khởi động lại ứng dụng (downtime) khi thay đổi thông tin kết nối Database.
* **Các thành phần chính:**
  * **AWS Secrets Manager:** Đóng vai trò là "Source of Truth" lưu trữ mật khẩu DB thật dạng plaintext ở ngoài cluster.
  * **External Secrets Operator (ESO):** Bộ điều khiển tự động kết nối và kéo mật khẩu từ AWS Secrets Manager về đồng bộ thành một Kubernetes Secret thông thường trong Cluster định kỳ (ví dụ mỗi 10s).
  * **Volume Mount:** Pod đọc mật khẩu trực tiếp dưới dạng file thông qua cơ chế mount ổ đĩa từ Kubernetes Secret, giúp nội dung file tự động cập nhật mà không làm Pod bị restart.

### 2. Lab 2.2: Scan + Ký + Xác thực container image (Trivy + Cosign)
* **Mục đích:** Đảm bảo chỉ những container image an toàn (đã quét sạch lỗ hổng bảo mật nghiêm trọng) và chính chủ (đã được ký số bởi Platform Team) mới được phép chạy trên Cluster, tránh bị tấn công chuỗi cung ứng (Supply Chain Attack).
* **Các thành phần chính:**
  * **Trivy (CI):** Công cụ quét mã độc và lỗ hổng bảo mật tích hợp vào GitHub Actions. Nếu phát hiện lỗ hổng mức `HIGH` hoặc `CRITICAL`, pipeline sẽ tự động báo đỏ và dừng lại.
  * **Cosign (CI):** Công cụ ký số cho Container Image. Image sau khi build xong và quét sạch CVE sẽ được ký bằng private key lưu trong GitHub Secrets.
  * **Sigstore Policy Controller (Admission Webhook):** Một webhook kiểm duyệt trong Kubernetes. Khi có yêu cầu tạo Pod, webhook này sẽ kiểm tra xem ảnh đó có chữ ký hợp lệ khớp với public key đã cấu hình hay không. Nếu không, nó sẽ chặn đứng yêu cầu tạo Pod.
  * **CVE Exception Process (Exception ADR):** Quy trình xử lý ngoại lệ cho các lỗ hổng bảo mật của bên thứ ba mà chưa có bản vá từ vendor (sử dụng file `.trivyignore` kèm thời hạn 14 ngày).

---

## PHẦN 2: TỪNG BƯỚC TRIỂN KHAI CHI TIẾT (STEP-BY-STEP)

### Bước 1: Khởi tạo thông tin xác thực AWS cục bộ (Không commit lên Git)
Chạy lệnh sau trên máy trạm để tạo secret chứa credentials kết nối đến AWS:
```bash
kubectl create secret generic aws-creds -n demo \
  --from-literal=access-key="<YOUR_AWS_ACCESS_KEY_ID>" \
  --from-literal=secret-key="<YOUR_AWS_SECRET_ACCESS_KEY>"
```

### Bước 2: Tạo cặp khóa Cosign (Key Pair) để ký ảnh
Chạy lệnh sau cục bộ để tạo ra file khóa công khai (`cosign.pub`) và khóa bí mật (`cosign.key`):
```bash
cosign generate-key-pair
```
* File `cosign.pub` sẽ được commit vào thư mục `signing/` trong Repo.
* Nội dung file `cosign.key` và mật khẩu khóa sẽ được lưu vào **GitHub Secrets** của Repository với tên:
  * `COSIGN_PRIVATE_KEY`: Copy toàn bộ nội dung file private key.
  * `COSIGN_PASSWORD`: Mật khẩu bạn đặt lúc tạo key.

### Bước 3: Đẩy cấu hình GitOps lên GitHub
Đẩy toàn bộ mã nguồn của bạn lên Github. Khi có commit mới đẩy lên nhánh `main`, GitHub Actions sẽ chạy và:
1. Build mã nguồn API tại `src/api/`.
2. Dùng Trivy để quét lỗ hổng.
3. Dùng Cosign để ký ảnh và đẩy lên GitHub Container Registry (GHCR).
4. Tự động cập nhật thẻ tag mới của image vào file `app-api/rollout.yaml` và push ngược lại repo.

### Bước 4: Deploy thông qua Argo CD
Argo CD sẽ tự động đồng bộ file `argocd/root.yaml` để kéo các ứng dụng con:
* Ứng dụng `eso` cài đặt Helm chart của External Secrets Operator.
* Ứng dụng `eso-config` đồng bộ cấu hình SecretStore và ExternalSecret.
* Ứng dụng `policy-controller` cài đặt Sigstore Webhook.
* Ứng dụng `policies` đồng bộ chính sách xác thực chữ ký ảnh ClusterImagePolicy.

### Bước 5: Kích hoạt xác thực chữ ký trên Namespace mong muốn
Sau khi ảnh đầu tiên đã được ký thành công từ GitHub Actions, ta gán label kích hoạt xác thực lên namespace `demo`:
```bash
kubectl label namespace demo policy.sigstore.dev/include=true --overwrite
```

---

## PHẦN 3: GIẢI THÍCH CHI TIẾT DÒNG QUA DÒNG CÁC FILE CẤU HÌNH

### 1. File [eso/secret-store.yaml](file:///d:/Download/AWS/Lab1806/eso/secret-store.yaml) (Kết nối AWS)
```yaml
1: apiVersion: external-secrets.io/v1beta1     # Sử dụng API version v1beta1 của ESO
2: kind: SecretStore                          # Loại tài nguyên cấu hình nơi lưu trữ secret
3: metadata:
4:   name: aws-store                          # Tên của SecretStore
5:   namespace: demo                          # Đặt trong namespace "demo" cùng với ứng dụng API
6: spec:
7:   provider:
8:     aws:                                   # Khai báo nhà cung cấp là AWS
9:       service: SecretsManager              # Dịch vụ sử dụng là AWS Secrets Manager
10:       region: ap-southeast-1               # Khu vực đặt Secret (Singapore)
11:       auth:
12:         secretRef:                        # Phương thức xác thực lấy từ một Kubernetes Secret
13:           accessKeyIDSecretRef:
14:             name: aws-creds                # Tên Kubernetes Secret chứa AWS credentials
15:             key: access-key                # Key chứa AWS Access Key ID trong secret
16:           secretAccessKeySecretRef:
17:             name: aws-creds                # Tên Kubernetes Secret chứa AWS credentials
18:             key: secret-key                # Key chứa AWS Secret Access Key trong secret
```

### 2. File [eso/external-secret.yaml](file:///d:/Download/AWS/Lab1806/eso/external-secret.yaml) (Đồng bộ Secret)
```yaml
1: apiVersion: external-secrets.io/v1beta1     # API version của ESO
2: kind: ExternalSecret                       # Loại tài nguyên định nghĩa ánh xạ đồng bộ secret
3: metadata:
4:   name: db-creds                           # Tên tài nguyên đồng bộ
5:   namespace: demo                          # Đặt trong namespace "demo"
6: spec:
7:   refreshInterval: 10s                     # Tần suất tự động đồng bộ lại từ AWS (mỗi 10 giây)
8:   secretStoreRef:
9:     name: aws-store                        # Liên kết tới SecretStore "aws-store" khai báo ở trên
10:     kind: SecretStore
11:   target:
12:     name: db-secret                        # Tên Kubernetes Secret đích sẽ được ESO tự tạo ra
13:     creationPolicy: Owner                  # ESO đóng vai trò là chủ sở hữu quản lý secret này
14:   data:
15:     - secretKey: password                  # Key trong Kubernetes Secret đích sẽ chứa giá trị mật khẩu
16:       remoteRef:
17:         key: w10/demo/db-password          # Tên khóa lưu trên AWS Secrets Manager cần lấy về
```

### 3. Cấu hình [app-api/rollout.yaml](file:///d:/Download/AWS/Lab1806/app-api/rollout.yaml) (Chỉ các dòng Mount Secret)
```yaml
28:         image: ghcr.io/bearh141/w10-api-141:0.0.2  # Image API chính chủ
...
33:         volumeMounts:                      # Cấu hình gắn ổ đĩa vào container
34:         - name: db-secret-volume           # Tên của volume khai báo bên dưới
35:           mountPath: /etc/db-secret        # Thư mục trong container sẽ chứa file mật khẩu
36:           readOnly: true                   # Chỉ cho phép đọc mật khẩu, tránh bị ghi đè từ container
...
58:       volumes:                             # Khai báo danh sách ổ đĩa của Pod
59:       - name: db-secret-volume             # Tên định danh cho volume
60:         secret:                            # Nguồn của volume lấy từ một Kubernetes Secret
61:           secretName: db-secret            # Mount secret có tên "db-secret" được tạo tự động bởi ESO
```
* **Tại sao không restart Pod?** Khi K8s Secret `db-secret` thay đổi, kubelet trên worker node sẽ phát hiện và cập nhật trực tiếp nội dung file tại `/etc/db-secret/password` (thực chất là cập nhật lại liên kết symlink trỏ đến file dữ liệu thật). Tiến trình trong container chỉ cần đọc lại file này khi cần mà không cần container phải khởi động lại.

### 4. File [policies/cluster-image-policy.yaml](file:///d:/Download/AWS/Lab1806/policies/cluster-image-policy.yaml) (Xác thực chữ ký)
```yaml
1: apiVersion: policy.sigstore.dev/v1beta1      # API version của Sigstore Policy Controller
2: kind: ClusterImagePolicy                    # Chính sách kiểm soát container image mức Cluster
3: metadata:
4:   name: image-signature-policy              # Tên chính sách
5: spec:
6:   images:
7:     - glob: "ghcr.io/bearh141/*"            # Áp dụng chính sách này cho tất cả image thuộc repo của bạn
8:   authorities:                              # Danh sách các bên có thẩm quyền ký
9:     - key:                                  # Sử dụng phương thức xác thực bằng khóa công khai (Public Key)
10:         data: |                            # Nội dung khóa công khai cosign.pub
11:           -----BEGIN PUBLIC KEY-----
12:           MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEXS+Hvf2hPKuX6+KrDl9rGvX9ZEzj
13:           ac38u+CXBR81AmD9BhnMNYFTo5hPtxHr5EA8jR1AcPRh8nroXu4oia7N6A==
14:           -----END PUBLIC KEY-----
```

### 5. File [.github/workflows/build-push.yml](file:///d:/Download/AWS/Lab1806/.github/workflows/build-push.yml) (Phần Trivy và Cosign)
```yaml
65:       - name: Install Cosign                    # Tải và cài đặt công cụ Cosign CLI trên runner
66:         uses: actions/configure-pages@v5 # (hoặc sigstore/cosign-installer)
67: 
68:       - name: Run Trivy vulnerability scanner   # Step chạy quét bảo mật
69:         uses: aquasecurity/trivy-action@0.22.0
70:         with:
71:           image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.semver.outputs.version }}
72:           format: 'table'                       # Định dạng xuất kết quả dạng bảng
73:           exit-code: '1'                        # Trả về mã lỗi 1 nếu phát hiện vi phạm lỗ hổng (làm lỗi pipeline)
74:           ignore-unfixed: true                  # Bỏ qua lỗi không có bản vá từ vendor (phù hợp thực tế)
75:           vuln-type: 'os,library'               # Quét cả hệ điều hành gốc và các thư viện code đi kèm
76:           severity: 'HIGH,CRITICAL'             # Chỉ chặn khi phát hiện lỗ hổng mức độ cao hoặc chí mạng
77: 
78:       - name: Sign the published Docker image   # Step ký ảnh bằng khóa bí mật
79:         env:
80:           COSIGN_PRIVATE_KEY: ${{ secrets.COSIGN_PRIVATE_KEY }} # Lấy private key từ GitHub Secret
81:           COSIGN_PASSWORD: ${{ secrets.COSIGN_PASSWORD }}       # Lấy mật khẩu giải mã private key từ Secret
82:         run: |
83:           cosign sign --yes --key env://COSIGN_PRIVATE_KEY "${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.semver.outputs.version }}"
```

### 6. Cấu hình [argocd/apps/policy-controller.yaml](file:///d:/Download/AWS/Lab1806/argocd/apps/policy-controller.yaml) (Bỏ qua cấu hình tài nguyên và probe)
```yaml
23:   ignoreDifferences:                       # Bỏ qua sự sai khác giữa cấu hình Git và Cluster thực tế
24:     - group: apps
25:       kind: Deployment
26:       name: policy-controller-webhook
27:       jsonPointers:                         # Đường dẫn các trường cần bỏ qua để tránh Argo CD ghi đè lại
28:         - /spec/template/spec/containers/0/livenessProbe  # Bỏ qua liveness probe (đã cấu hình thủ công kéo dài timeout)
29:         - /spec/template/spec/containers/0/readinessProbe # Bỏ qua readiness probe
30:         - /spec/template/spec/containers/0/resources      # Bỏ qua giới hạn tài nguyên CPU/RAM đã tối ưu
```
* **Tại sao cần file này?** Môi trường thử nghiệm Minikube có tài nguyên CPU hạn chế, khiến webhook mặc định bị timeout khi APIServer gọi đến. Chúng ta đã nâng CPU limit lên `1000m` và giãn timeout probe lên `10s` trực tiếp trong cluster. Cấu hình `ignoreDifferences` ngăn Argo CD tự động khôi phục lại cấu hình mặc định (vốn sẽ làm pod bị lỗi crash/timeout).

---

## PHẦN 4: BỘ CÂU HỎI VẤN ĐÁP ÔN TẬP (VIVA Q&A)

### Câu 1: Tại sao việc thay đổi giá trị trong AWS Secrets Manager lại cập nhật được mật khẩu vào container mà Pod không cần restart? Cơ chế cập nhật file mount hoạt động ra sao?
**Trả lời:**
Do cơ chế mount mật khẩu dưới dạng **Volume Secret**. Kubelet trên mỗi worker node sẽ định kỳ kiểm tra (đồng bộ) trạng thái của các Secret đang được mount. Khi ESO cập nhật giá trị mới vào Kubernetes Secret `db-secret`, Kubelet sẽ tự động viết lại nội dung mới vào ổ đĩa ảo tương ứng tại đường dẫn `/etc/db-secret/password` (thực chất là cập nhật lại liên kết symlink trỏ đến file dữ liệu thật). Ứng dụng trong Container chỉ cần đọc trực tiếp từ file này khi cần truy vấn DB, do đó không cần restart lại Pod. Nếu chúng ta truyền mật khẩu qua biến môi trường (Env Variable), giá trị chỉ được đọc 1 lần duy nhất tại thời điểm khởi tạo Pod, bắt buộc phải restart Pod nếu muốn nhận giá trị mới.

### Câu 2: Sự khác biệt giữa `SecretStore` và `ClusterSecretStore` trong External Secrets Operator là gì? Trong bài chúng ta dùng loại nào?
**Trả lời:**
* **`SecretStore`** là tài nguyên có phạm vi **Namespace** (Namespace-scoped). Nó chỉ cho phép các tài nguyên `ExternalSecret` trong cùng namespace đó tham chiếu đến để lấy credentials kết nối AWS.
* **`ClusterSecretStore`** là tài nguyên có phạm vi **toàn Cluster** (Cluster-scoped), cho phép mọi `ExternalSecret` ở mọi namespace sử dụng chung một cấu hình kết nối AWS.
* Trong bài lab này, chúng ta sử dụng **`SecretStore`** ở namespace `demo` để cô lập quyền truy cập thông tin xác thực AWS cục bộ cho riêng namespace này, đảm bảo tính bảo mật và nguyên tắc phân quyền tối giản.

### Câu 3: Sigstore Policy Controller hoạt động ở giai đoạn nào trong vòng đời của một Request gửi tới Kubernetes API Server?
**Trả lời:**
Sigstore Policy Controller hoạt động dưới dạng một **Admission Webhook** (gồm cả hai pha `MutatingAdmissionWebhook` và `ValidatingAdmissionWebhook`).
Khi người dùng chạy lệnh tạo Pod/Deployment, request sẽ đi qua các pha kiểm duyệt của API Server:
1. **Authentication & Authorization** (Xác thực người dùng & RBAC).
2. **Mutating Webhooks** (Pha thay đổi - Sigstore tự động tìm và thay thế tag image bằng SHA digest chính xác của nó để tránh bị đánh tráo tag sau này).
3. **Object Schema Validation** (Kiểm tra cú pháp tài nguyên).
4. **Validating Webhooks** (Pha xác thực - Sigstore kiểm tra chữ ký của SHA digest có khớp với khóa công khai trong `ClusterImagePolicy` hay không. Nếu không khớp hoặc không có chữ ký, Webhook sẽ từ chối và chặn đứng request, API Server trả về lỗi `Forbidden` / `BadRequest`).

### Câu 4: Tại sao trong bài Lab 2.2, chúng ta không gắn label xác thực chữ ký ảnh (`policy.sigstore.dev/include=true`) lên namespace `demo` ngay từ đầu mà phải đợi sau khi image đã ký?
**Trả lời:**
Nếu chúng ta gắn label kích hoạt xác thực lên namespace `demo` trước khi thực hiện ký ảnh đầu tiên trong CI/CD, chính sách sẽ có hiệu lực ngay lập tức. Khi đó, API Server/Policy Controller sẽ chặn mọi yêu cầu tạo Pod/Rollout mới của ứng dụng API vì hình ảnh hiện tại trên Container Registry chưa có chữ ký hợp lệ. Điều này sẽ khiến ứng dụng API bị lỗi triển khai ngay lập tức. Quy trình chuẩn là: Build image $\rightarrow$ Quét CVE $\rightarrow$ Ký ảnh $\rightarrow$ Đẩy ảnh lên Registry $\rightarrow$ Kích hoạt Label cho Namespace.

### Câu 5: Làm thế nào để bypass một lỗ hổng bảo mật mức HIGH hoặc CRITICAL mà vendor thư viện chưa phát hành bản vá (unfixed CVE) để không làm lỗi pipeline CI/CD?
**Trả lời:**
Chúng ta áp dụng quy trình ngoại lệ bảo mật **CVE Exception Process** được ghi lại bằng tài liệu ADR:
1. Tạo một file **`.trivyignore`** ở thư mục gốc của repository.
2. Thêm ID của lỗ hổng (ví dụ: `CVE-2026-99999`) vào file kèm ghi chú ngày hết hạn của ngoại lệ (tối đa 14 ngày).
3. Khai báo một file tài liệu ngoại lệ dạng ADR bảo mật (ví dụ: `adr-exception-CVE-2026-99999.md` trong thư mục `runbooks/`) giải trình lý do chưa thể sửa, các biện pháp giảm thiểu rủi ro tạm thời và cam kết ngày vá lỗi. Trivy Scanner trong pipeline sẽ đọc file `.trivyignore` và tạm thời bỏ qua CVE này, giúp pipeline xanh trở lại.
