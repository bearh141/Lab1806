# BÁO CÁO MINH CHỨNG (EVIDENCE) - LAB BUỔI SÁNG

Tài liệu này hướng dẫn chi tiết các bước chạy lệnh kiểm tra và vị trí cần chụp ảnh màn hình để hoàn thành báo cáo nghiệm thu cho phần Lab buổi sáng (RBAC, Gatekeeper, Custom Policy, và Argo Rollouts).

---

## PHẦN 1: LAB 1.1 - THIẾT LẬP RBAC
Phần này chứng minh việc phân quyền thành công cho ServiceAccount `api-sa` trong namespace `demo` chỉ có quyền read-only.

### Lệnh chạy kiểm tra:
Mở terminal và chạy các lệnh sau để kiểm tra quyền hạn của `api-sa`:
```bash
# 1. Kiểm tra xem api-sa có quyền xem (get/list) Pods không (Kỳ vọng: yes)
kubectl auth can-i get pods --as=system:serviceaccount:demo:api-sa -n demo

# 2. Kiểm tra xem api-sa có quyền tạo Pods không (Kỳ vọng: no)
kubectl auth can-i create pods --as=system:serviceaccount:demo:api-sa -n demo

# 3. Kiểm tra xem api-sa có quyền xóa Pods không (Kỳ vọng: no)
kubectl auth can-i delete pods --as=system:serviceaccount:demo:api-sa -n demo
```

### Hướng dẫn chụp ảnh minh chứng:
> **Hình ảnh 1: Xác thực phân quyền RBAC cho api-sa**
> * **Cách chụp:** Chụp toàn bộ cửa sổ terminal hiển thị kết quả của 3 lệnh `kubectl auth can-i` phía trên (kết quả trả về lần lượt phải là `yes`, `no`, `no`).
> * **Vị trí chèn ảnh:**
>   ![Minh chứng RBAC](./images/rbac-verify.png)

---

## PHẦN 2: LAB 1.2 - OPA GATEKEEPER SECURE POLICIES
Phần này chứng minh việc OPA Gatekeeper đã chặn thành công các cấu hình Pod không an toàn.

### Lệnh chạy kiểm tra:
Chạy lần lượt các file test đã tạo trong thư mục `tests/`:
```bash
# 1. Test cấm tag :latest
kubectl apply -f tests/pod-latest.yaml

# 2. Test bắt buộc khai báo resource limits
kubectl apply -f tests/pod-no-limits.yaml

# 3. Test cấm chạy quyền root
kubectl apply -f tests/pod-root-user.yaml

# 4. Test cấm sử dụng host network
kubectl apply -f tests/pod-hostnetwork.yaml
```

### Hướng dẫn chụp ảnh minh chứng:
> **Hình ảnh 2: Gatekeeper chặn các cấu hình không an toàn**
> * **Cách chụp:** Chụp màn hình terminal hiển thị thông báo lỗi `Error from server (Forbidden)` từ webhook `validation.gatekeeper.sh` khi bạn chạy 4 lệnh trên.
> * **Vị trí chèn ảnh:**
>   ![Minh chứng Gatekeeper Block](./images/gatekeeper-block.png)

---

## PHẦN 3: LAB 1.3 - CUSTOM CONSTRAINTTEMPLATE (OWNER LABEL)
Phần này chứng minh chính sách tùy chỉnh (Custom Policy) bắt buộc mọi Deployment/Rollout phải có label `owner` hoạt động chính xác.

### Lệnh chạy kiểm tra:
```bash
# 1. Test deploy Deployment thiếu label owner (Kỳ vọng: bị chặn/Forbidden)
kubectl apply -f tests/deploy-no-owner.yaml

# 2. Test deploy Deployment có label owner (Kỳ vọng: tạo thành công)
kubectl apply -f tests/deploy-with-owner.yaml

# Dọn dẹp sau khi test:
kubectl delete -f tests/deploy-with-owner.yaml
```

### Hướng dẫn chụp ảnh minh chứng:
> **Hình ảnh 3: Chặn thành công Deployment thiếu label owner**
> * **Cách chụp:** Chụp terminal hiển thị lỗi từ chối của webhook cho file `deploy-no-owner.yaml` và thông báo tạo thành công `deployment.apps/test-deploy-with-owner created` cho file `deploy-with-owner.yaml`.
> * **Vị trí chèn ảnh:**
>   ![Minh chứng Custom Owner Label Policy](./images/custom-policy.png)

---

## PHẦN 4: LAB 2 - ARGO ROLLOUTS & PROGRESSIVE DELIVERY
Phần này chứng minh việc phân tích tự động (Automated Analysis) và tự động Rollback hoạt động đúng đắn khi cập nhật ứng dụng.

### Kịch bản 1: Cập nhật thành công (Error Rate = 0%)
1. Mở file `app-api/rollout.yaml`, đảm bảo `ERROR_RATE` là `"0"`.
2. Thực hiện git commit và push:
   ```bash
   git add app-api/rollout.yaml
   git commit -m "test: rollout success"
   git push origin main
   ```
3. Theo dõi tiến trình Rollout:
   ```bash
   kubectl argo rollouts get rollout api -n demo
   ```

> **Hình ảnh 4: Rollout thành công và AnalysisRun ở trạng thái Successful**
> * **Cách chụp:** 
>   1. Chụp màn hình UI Argo CD hiển thị ứng dụng `api` có màu xanh lá (Healthy/Synced).
>   2. Chụp terminal chạy lệnh `kubectl argo rollouts get rollout api -n demo` hiển thị trạng thái chuyển đổi thành công sang phiên bản mới và các bước kiểm tra (AnalysisRun) đều màu xanh (Successful).
> * **Vị trí chèn ảnh:**
>   ![Minh chứng Rollout Success](./images/rollout-success.png)

---

### Kịch bản 2: Tự động Rollback do lỗi (Error Rate = 15%)
1. Sửa `ERROR_RATE` trong `app-api/rollout.yaml` thành `"0.15"`.
2. Commit và push:
   ```bash
   git add app-api/rollout.yaml
   git commit -m "test: rollout fail and rollback"
   git push origin main
   ```
3. Theo dõi tiến trình tự động hủy bỏ và hoàn tác phiên bản:
   ```bash
   kubectl argo rollouts get rollout api -n demo -w
   ```

> **Hình ảnh 5: Trạng thái Rollback thành công do Analysis fail**
> * **Cách chụp:** Chụp terminal hoặc giao diện Argo CD hiển thị AnalysisRun bị đỏ (Failed) và Rollout tự động quay đầu rollback trở lại phiên bản an toàn trước đó.
> * **Vị trí chèn ảnh:**
>   ![Minh chứng Rollback](./images/rollout-rollback.png)

---

### Kịch bản 3: Kích hoạt Email cảnh báo SLO (Error Rate = 10%)
1. Sửa `ERROR_RATE` trong `app-api/rollout.yaml` thành `"0.10"`.
2. Commit và push lên GitHub.
3. Chờ 2-3 phút để Prometheus phát hiện lỗi tỉ lệ cao và gửi cảnh báo về AlertManager.
4. Mở hòm thư email Gmail của bạn để kiểm tra thư cảnh báo nhận được từ AlertManager.

> **Hình ảnh 6: Email cảnh báo SLO từ AlertManager**
> * **Cách chụp:** Chụp lại email nhận được trong hộp thư của bạn có tiêu đề hoặc nội dung cảnh báo vi phạm ngưỡng SLO (>5% lỗi).
> * **Vị trí chèn ảnh:**
>   ![Minh chứng Alert Email](./images/alert-email.png)
