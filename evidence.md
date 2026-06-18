# BÁO CÁO MINH CHỨNG (EVIDENCE) - LAB BUỔI SÁNG

Tài liệu này hướng dẫn chi tiết các bước chạy lệnh kiểm tra và cung cấp sẵn cấu trúc tên file hình ảnh để bạn đặt tên và chèn vào báo cáo nghiệm thu cho 3 phần Lab buổi sáng (RBAC, Gatekeeper, và Custom Policy).

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
* **Tên file ảnh đặt là:** `1_2_gatekeeper.png`
* **Nội dung cần chụp:** Chụp toàn bộ terminal hiển thị kết quả chạy cả 5 lệnh test trên (cho thấy 4 lệnh đầu bị chặn/Forbidden bởi admission webhook, lệnh thứ 5 tạo pod thành công, và lệnh dọn dẹp pod).
* **Hiển thị hình ảnh:**
  ![Minh chứng OPA Gatekeeper](./images/1_2_gatekeeper.png)

---

## PHẦN 3: LAB 1.3 - CUSTOM CONSTRAINTTEMPLATE (OWNER LABEL)
Chứng minh chính sách tùy chỉnh (Custom Policy) bắt buộc mọi Deployment/Rollout phải có label `owner` hoạt động chính xác.

### Lệnh chạy kiểm tra:
Chạy lần lượt các lệnh test trong terminal:
```bash
# 1. Test deploy Deployment thiếu label owner (Kỳ vọng: Bị reject)
kubectl apply -f tests/deploy-no-owner.yaml

# 2. Test deploy Deployment có label owner (Kỳ vọng: Tạo thành công)
kubectl apply -f tests/deploy-with-owner.yaml

# 3. Dọn dẹp sau khi kiểm tra xong:
kubectl delete -f tests/deploy-with-owner.yaml
```

### Minh chứng cần chụp:
* **Tên file ảnh đặt là:** `1_3_custom_policy.png`
* **Nội dung cần chụp:** Chụp toàn bộ terminal hiển thị kết quả chạy cả 3 lệnh trên (cho thấy lệnh đầu bị chặn/Forbidden, lệnh thứ 2 tạo thành công, và lệnh thứ 3 dọn dẹp deployment).
* **Hiển thị hình ảnh:**
  ![Minh chứng Custom Owner Label Policy](./images/1_3_custom_policy.png)
