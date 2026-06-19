# Runbook: Kiểm tra và vận hành đồng bộ xoay vòng Secret (ESO)
> **Mục tiêu**: Đảm bảo secret từ AWS Secrets Manager được tự động đồng bộ xuống cụm Kubernetes dưới 60 giây và Pod Flask API cập nhật mật khẩu mà không bị restart.

## 1. Quy trình kiểm tra tự động xoay vòng mật khẩu (Secret Rotation)
Để xác thực tiêu chí "ESO rotate < 60s, pod không restart", thực hiện các bước sau:

1. **Bước 1**: Xem trạng thái và thời gian chạy hiện tại của Pod trong namespace `demo`:
   ```bash
   kubectl get pods -n demo
   ```
   *Ghi lại cột `AGE` và `RESTARTS`.*

2. **Bước 2**: Thực hiện thay đổi giá trị mật khẩu trên AWS Secrets Manager:
   ```bash
   aws secretsmanager put-secret-value --secret-id prod/db/credentials --secret-string '{"username":"postgres","password":"NewPassword123"}'
   ```

3. **Bước 3**: Chờ tối đa 60 giây (Khoảng thời gian `refreshInterval: 10s` của ExternalSecret) để ESO đồng bộ dữ liệu.

4. **Bước 4**: Kiểm tra giá trị mật khẩu mới đã được cập nhật thành công xuống cụm K8s:
   ```bash
   kubectl get secret db-secret-local -n demo -o jsonpath='{.data.password}' | base64 --decode
   ```
   *Kỳ vọng: Kết quả trả về là `NewPassword123`.*

5. **Bước 5**: Kiểm tra lại Pod Flask API:
   ```bash
   kubectl get pods -n demo
   ```
   *Kỳ vọng: Cột `RESTARTS` vẫn giữ nguyên bằng `0` và cột `AGE` tăng tiến bình thường (Pod không hề bị restart).*

## 2. Hướng dẫn khắc phục sự cố (Troubleshooting) khi ESO không đồng bộ
* **Lỗi Webhook Certificate**: Nếu ExternalSecret bị kẹt trạng thái `Ready: False`, kiểm tra logs của webhook pod:
  ```bash
  kubectl rollout restart deployment external-secrets-webhook -n external-secrets
  ```
* **Lỗi AWS Credentials**: Kiểm tra xem Pod `external-secrets` có thể kết nối Internet hoặc AWS API không bằng cách kiểm tra log operator:
  ```bash
  kubectl logs -l app.kubernetes.io/name=external-secrets -n external-secrets
  ```
