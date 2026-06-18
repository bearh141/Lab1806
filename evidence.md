# BÁO CÁO MINH CHỨNG (EVIDENCE) - LAB BUỔI SÁNG

Tài liệu này hướng dẫn chi tiết các bước chạy lệnh kiểm tra và cung cấp sẵn cấu trúc tên file hình ảnh để bạn đặt tên và chèn vào báo cáo nghiệm thu dễ dàng nhất.

---

## PHẦN 1: LAB 1.1 - THIẾT LẬP RBAC
Chứng minh việc phân quyền thành công cho các tài khoản `alice`, `bob`, và `carol`.

### Lệnh chạy kiểm tra:
Chạy lần lượt 4 lệnh dưới đây trong terminal:
```bash
# 1. Kiểm tra alice có tạo được deployment trong namespace demo (Kỳ vọng: yes)
kubectl auth can-i create deployments -n demo --as alice

# 2. Kiểm tra alice có tạo được deployment trong namespace kube-system (Kỳ vọng: no)
kubectl auth can-i create deployments -n kube-system --as alice

# 3. Kiểm tra bob có đọc được pod ở mọi namespace (Kỳ vọng: yes)
kubectl auth can-i get pods -A --as bob

# 4. Kiểm tra carol có xóa được node không (Kỳ vọng: no)
kubectl auth can-i delete nodes --as carol
```

### Minh chứng cần chụp:
* **Tên file ảnh đặt là:** `1_1_rbac_verify.png`
* **Nội dung cần chụp:** Toàn bộ terminal hiển thị kết quả chạy 4 lệnh trên (kết quả trả về lần lượt là `yes`, `no`, `yes`, `no`).
* **Hiển thị hình ảnh:**
  ![Minh chứng RBAC](./images/1_1_rbac_verify.png)

---

## PHẦN 2: LAB 1.2 - OPA GATEKEEPER SECURE POLICIES
Chứng minh OPA Gatekeeper đã cấu hình và chặn thành công các tài nguyên vi phạm chính sách an toàn.

### Lệnh chạy kiểm tra:
Chạy lần lượt các lệnh apply file test trong thư mục `tests/`:

```bash
# 1. Test cấm tag :latest (Kỳ vọng: Bị reject)
kubectl apply -f tests/pod-latest.yaml

# 2. Test bắt buộc khai báo resource limits (Kỳ vọng: Bị reject)
kubectl apply -f tests/pod-no-limits.yaml

# 3. Test cấm chạy quyền root (Kỳ vọng: Bị reject)
kubectl apply -f tests/pod-root-user.yaml

# 4. Test cấm sử dụng host network (Kỳ vọng: Bị reject)
kubectl apply -f tests/pod-hostnetwork.yaml

# 5. Test Pod hợp lệ (Kỳ vọng: Tạo thành công)
kubectl apply -f tests/pod-valid.yaml
kubectl delete -f tests/pod-valid.yaml
```

### Minh chứng cần chụp:
Bạn có thể chụp gom chung hoặc chia nhỏ các ảnh như sau:

* **Tên file ảnh tag latest bị chặn:** `1_2_gatekeeper_latest.png`
  ![latest tag blocked](./images/1_2_gatekeeper_latest.png)

* **Tên file ảnh thiếu resource limits bị chặn:** `1_2_gatekeeper_no_limits.png`
  ![no limits blocked](./images/1_2_gatekeeper_no_limits.png)

* **Tên file ảnh quyền root bị chặn:** `1_2_gatekeeper_root_user.png`
  ![root user blocked](./images/1_2_gatekeeper_root_user.png)

* **Tên file ảnh hostNetwork bị chặn:** `1_2_gatekeeper_hostnetwork.png`
  ![hostNetwork blocked](./images/1_2_gatekeeper_hostnetwork.png)

* **Tên file ảnh Pod hợp lệ được tạo thành công:** `1_2_gatekeeper_valid.png`
  ![valid pod created](./images/1_2_gatekeeper_valid.png)

---

## PHẦN 3: LAB 1.3 - CUSTOM CONSTRAINTTEMPLATE (OWNER LABEL)
Chứng minh chính sách tùy chỉnh (Custom Policy) bắt buộc mọi Deployment/Rollout phải có label `owner` hoạt động chính xác.

### Lệnh chạy kiểm tra:
```bash
# 1. Test deploy Deployment thiếu label owner (Kỳ vọng: Bị reject)
kubectl apply -f tests/deploy-no-owner.yaml

# 2. Test deploy Deployment có label owner (Kỳ vọng: Tạo thành công)
kubectl apply -f tests/deploy-with-owner.yaml
kubectl delete -f tests/deploy-with-owner.yaml
```

### Minh chứng cần chụp:
* **Tên file ảnh khi Deployment bị chặn do thiếu label:** `1_3_custom_policy_no_owner.png`
  ![no owner label blocked](./images/1_3_custom_policy_no_owner.png)

* **Tên file ảnh khi Deployment tạo thành công nhờ có label:** `1_3_custom_policy_with_owner.png`
  ![with owner label created](./images/1_3_custom_policy_with_owner.png)

---

## PHẦN 4: LAB 2 - ARGO ROLLOUTS & CANARY PROGRESSIVE DELIVERY
Chứng minh việc chạy canary rollout, phân tích tự động (Automated Analysis) và tự động Rollback hoạt động đúng đắn.

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

* **Tên file ảnh Rollout thành công (ArgoCD UI hoặc terminal):** `2_canary_success.png`
  ![Canary Success](./images/2_canary_success.png)

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

* **Tên file ảnh Rollback thành công do Analysis fail:** `2_canary_rollback.png`
  ![Canary Rollback](./images/2_canary_rollback.png)

---

### Kịch bản 3: Kích hoạt Email cảnh báo SLO (Error Rate = 10%)
1. Sửa `ERROR_RATE` trong `app-api/rollout.yaml` thành `"0.10"`.
2. Commit và push lên GitHub.
3. Chờ 2-3 phút, sau đó kiểm tra hòm thư email của bạn.

* **Tên file ảnh chụp Email cảnh báo SLO từ AlertManager:** `2_slo_email.png`
  ![SLO Alert Email](./images/2_slo_email.png)
