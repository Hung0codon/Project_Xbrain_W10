# Runbook: Vận hành và khắc phục sự cố chữ ký số Cosign
> **Mục tiêu**: Đảm bảo toàn bộ image chạy trong namespace `demo` được xác thực chữ ký số hợp lệ và hướng dẫn xử lý khi pod bị Admission Webhook từ chối.

## 1. Quy trình kiểm tra chữ ký số (Tự kiểm)
1. **Kiểm tra chặn image chưa ký (Admission Reject)**:
   Chạy lệnh để chạy thử một image từ Docker Hub không có chữ ký của bạn:
   ```bash
   kubectl run test-unsigned --image=nginx:alpine -n demo
   ```
   *Kết quả kỳ vọng*: Bị từ chối ngay lập tức kèm thông báo lỗi:
   ```bash
   Error from server (BadRequest): admission webhook "policy.sigstore.dev" denied the request
   ```

2. **Kiểm tra image đã ký khởi chạy thành công**:
   Kiểm tra Pod của Flask API:
   ```bash
   kubectl get pods -n demo -l app=api
   ```
   *Kết quả kỳ vọng*: Trạng thái `Running`, sẵn sàng nhận traffic.

## 2. Hướng dẫn xử lý sự cố khi Pod bị chặn (Admission Reject)
Nếu Pod chính thức của dự án bị Sigstore Admission Webhook chặn đứng, thực hiện theo các bước sau:

1. **Bước 1**: Xác định xem Image của Pod đã được ký hay chưa:
   ```bash
   cosign verify --key signing/cosign.pub <REGISTRY>/<IMAGE_NAME>:<TAG>
   ```
2. **Bước 2**: Nếu chưa được ký, tiến hành ký thủ công (trong trường hợp khẩn cấp cần hotfix):
   ```bash
   $env:COSIGN_PASSWORD="MatKhauBaoVeKhoaCosign123"
   cosign sign --key cosign.key <REGISTRY>/<IMAGE_NAME>:<TAG>
   ```
3. **Bước 3**: Nếu đã ký nhưng Webhook vẫn chặn, kiểm tra lại cấu hình Public Key trong `ClusterImagePolicy` trên cụm xem có trùng khớp với file `signing/cosign.pub` hay không:
   ```bash
   kubectl get clusterimagepolicy image-signature-policy -o yaml
   ```

## 3. Quy trình thay đổi khóa Cosign (Key Rotation)
Nếu khóa riêng tư `cosign.key` hoặc mật khẩu `COSIGN_PASSWORD` bị lộ:
1. Tạo cặp khóa mới: `cosign generate-key-pair`.
2. Cập nhật `COSIGN_PRIVATE_KEY` và `COSIGN_PASSWORD` trong GitHub Secrets của repository.
3. Cập nhật Public Key mới vào file `signing/cosign.pub` và `policies/cluster-image-policy.yaml`.
4. Commit và đẩy lên Git để ArgoCD đồng bộ lại chính sách mới xuống cụm Kubernetes.
