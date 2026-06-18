# BÁO CÁO TOÀN DIỆN TIẾN TRÌNH TRIỂN KHAI DỰ ÁN W10
> **Tài liệu hướng dẫn và báo cáo chi tiết từ A-Z toàn bộ các phần việc đã thực hiện: RBAC, OPA Gatekeeper, External Secrets Operator (ESO) và ứng dụng Argo Rollout.**

---

## 🗺️ Sơ đồ Tổng quan Hệ thống (Mermaid)

```mermaid
graph TD
    subgraph AWS Cloud ["AWS Cloud"]
        ASM["AWS Secrets Manager <br> (prod/db/credentials)"]
    end

    subgraph Minikube Cluster ["Minikube Local Cluster"]
        direction TB
        
        subgraph Namespace: argocd ["Namespace: argocd"]
            ArgoCD["ArgoCD Engine"]
        end

        subgraph Namespace: external-secrets ["Namespace: external-secrets"]
            ESO["External Secrets Operator"]
        end

        subgraph Namespace: gatekeeper-system ["Namespace: gatekeeper-system"]
            GK["OPA Gatekeeper <br> (Chỉ kiểm duyệt Namespace: demo)"]
        end

        subgraph Namespace: demo ["Namespace: demo"]
            direction TB
            SecretCreds["K8s Secret: aws-secret-creds <br> (Access Keys - Local Only)"]
            Store["SecretStore"]
            ExtSecret["ExternalSecret"]
            LocalSecret["K8s Secret: db-secret-local <br> (Tự động kéo từ AWS)"]
            Pod["Flask API Pod <br> (Mounts db-secret-local)"]
            
            %% Phân quyền RBAC
            Alice["User: Alice <br> (Role: developer-role)"]
            Bob["User: Bob <br> (ClusterRoleBinding: sre-cluster-role)"]
            Carol["User: Carol <br> (ClusterRoleBinding: viewer-cluster-role)"]
        end
    end

    %% Mối liên kết
    SecretCreds -->|Cung cấp API Keys| Store
    Store -->|Xác thực kết nối| ASM
    ExtSecret -->|Tham chiếu| Store
    ExtSecret -->|Đọc mật khẩu| ASM
    ExtSecret -->|Sinh tự động| LocalSecret
    LocalSecret -->|Volume Mount| Pod
    GK -.->|Kiểm soát chính sách bảo mật| Pod
    ArgoCD -->|Đồng bộ hóa GitOps| Minikube Cluster
```

---

## 📋 TỔNG HỢP TIẾN TRÌNH CÁC BƯỚC THỰC HIỆN

| Phần | Hành động | Nội dung chính | Tại sao phải thực hiện? |
| :--- | :--- | :--- | :--- |
| **Phần 1** | **Cấu hình RBAC** | Thêm dấu ngăn cách YAML, sửa tên Carol, mở rộng quyền xem toàn cụm. | Thiết lập phân quyền đúng vai trò (Alice - Dev, Bob - SRE, Carol - Auditor). |
| **Phần 2** | **Cài đặt OPA Gatekeeper** | Triển khai Operator, viết ConstraintTemplate (Rego) và Constraint cho 4 luật cơ bản + 1 luật custom (Replicas <= 5). | Kiểm duyệt chính sách an toàn thông tin và quy chuẩn tài nguyên khi tạo Pod. |
| **Phần 3** | **Tối ưu hóa Namespace OPA** | Giới hạn OPA chỉ chạy trên namespace `demo`. Xóa các parameter thừa. | Tránh việc OPA chặn đứng các Pod hệ thống (Prometheus), giải phóng cụm khỏi trạng thái treo. |
| **Phần 4** | **Tích hợp AWS Secrets & ESO** | Cài đặt ESO, chuẩn hóa cấu trúc file, tạo API Keys cục bộ, khôi phục AWS Secret bị xóa. | Bảo mật mật khẩu ứng dụng bằng cách lưu tập trung trên Cloud và tự động sync về cụm. |
| **Phần 5** | **Mount Secret & Rollout Canary** | Mount Secret `db-secret-local` vào volume của API Rollout. | Cho phép Pod nhận mật khẩu dạng file tĩnh an toàn, tự động cập nhật mà không cần restart container. |

---

## 🛠️ CHI TIẾT TỪNG PHẦN VIỆC ĐÃ THỰC HIỆN

### PHẦN 1: CẤU HÌNH & TỐI ƯU HÓA RBAC (Role-Based Access Control)
#### 1. Các chỉnh sửa đã thực hiện:
* **Thêm dấu ngăn cách YAML (`---`)**: Thêm dấu ngăn cách hợp lệ giữa các tài nguyên trong file `rbac/roles.yaml` và `rbac/rolebindings.yaml`.
* **Sửa tên đối tượng**: Trong `rolebindings.yaml`, sửa tên tài khoản từ `carl` thành `carol` cho khớp với logic phân quyền.
* **Mở rộng quyền cho Carol**: Carol là Auditor cần xem mọi tài nguyên trên toàn cụm. Cập nhật `viewer-cluster-role` để bao quát toàn bộ tài nguyên:
  ```yaml
  rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["get", "list", "watch"]
  ```

#### 2. Tại sao cần thực hiện?
* **Tính toàn vẹn cú pháp**: Kubernetes không chấp nhận gộp nhiều tài nguyên vào một file mà không có dấu phân cách `---`, gây lỗi cú pháp khi ArgoCD deploy.
* **Đảm bảo tính chính xác trong IAM**: Sửa lỗi chính tả tên của Carol giúp binding hoạt động đúng người. Cấp quyền xem toàn cụm cho Carol giúp Auditor kiểm toán hệ thống mà không thể can thiệp phá hỏng tài nguyên (Read-only access).

---

### PHẦN 2: TRIỂN KHAI OPA GATEKEEPER & VIẾT LUẬT CUSTOM
#### 1. Các chỉnh sửa đã thực hiện:
* Triển khai OPA Gatekeeper Operator thông qua ArgoCD.
* Khai báo 4 luật cơ bản: Cấm sử dụng tag `:latest` (`block-latest-tag`), cấm dùng HostNetwork (`block-host-network`), bắt buộc chạy non-root (`enforce-non-root`), và bắt buộc khai báo CPU/Memory limits (`require-limits`).
* Sửa lỗi logic Rego trong các template (đảm bảo việc quét qua các container và check các trường tồn tại đúng cú pháp).
* **Tạo Luật Custom - Giới hạn Replicas <= 5**: 
  * Định nghĩa `k8smaxreplicas.yaml` (ConstraintTemplate) quét qua `spec.replicas` của Deployment/Rollout.
  * Định nghĩa `max-replicas-5.yaml` (Constraint) để giới hạn tối đa là 5 bản sao.

#### 2. Tại sao cần thực hiện?
* **Đảm bảo tính tuân thủ (Compliance) & Bảo mật**: Tránh Pod chạy quyền Admin (Root), tránh rò rỉ mạng qua HostNetwork, tránh việc kéo image không rõ phiên bản (latest).
* **Quản trị tài nguyên & Chi phí**: Bắt buộc giới hạn Limits để tránh rò rỉ bộ nhớ (OOM) làm sập Node. Giới hạn số lượng Replicas tối đa là 5 để tránh việc lập trình viên scale bừa bãi gây cạn kiệt tài nguyên của cụm máy chủ.

---

### PHẦN 3: GIỚI HẠN PHẠM VI OPA GATEKEEPER (GỠ LỖI TREO CỤM)
#### 1. Các chỉnh sửa đã thực hiện:
* Sửa đổi cấu hình `match.namespaces` của cả 5 luật Constraint, chỉ định duy nhất namespace **`demo`**.
* Xóa phần `parameters` thừa trong `require-limits.yaml` để khớp với Rego Template không có schema parameters.

#### 2. Tại sao cần thực hiện?
* **Không làm gián đoạn tài nguyên hệ thống**: Prometheus admission webhook và các job khởi tạo hệ thống chạy trong namespace `monitoring` không cấu hình tài nguyên limits. Nếu áp dụng luật Gatekeeper toàn cụm, các job này sẽ bị chặn ➡️ ArgoCD bị treo cứng do Wave 0 không thể hoàn thành. Việc giới hạn OPA chỉ chạy trên namespace ứng dụng (`demo`) giúp giải phóng hoàn toàn cụm hệ thống.

---

### PHẦN 4: TÍCH HỢP AWS SECRETS MANAGER VÀ ESO
#### 1. Các chỉnh sửa đã thực hiện:
* Triển khai External Secrets Operator bằng Helm Chart.
* **Sửa lỗi hoán đổi file cấu hình**: Trả lại đúng vị trí cho file ArgoCD Application (`argocd/apps/eso.yaml`) và file cấu hình AWS SecretStore (`eso/secret-store.yaml`).
* **Tạo AWS Credentials cục bộ dưới máy**: Chạy lệnh tạo Secret chứa khóa API kết nối AWS trong cụm Kubernetes.
* **Khôi phục Secret bị xóa trên AWS**: Dùng AWS CLI khôi phục Secret `prod/db/credentials` ở Singapore đang ở trạng thái chờ xóa.

#### 2. Tại sao cần thực hiện?
* **GitOps Bảo mật (Secret Integration)**: Giải quyết bài toán bảo mật của GitOps: Không đưa bất kỳ thông tin nhạy cảm nào lên git. ESO đóng vai trò cầu nối, tự động đồng bộ mật khẩu từ đám mây (AWS Secrets Manager) xuống cụm local một cách hoàn toàn tự động.

---

### PHẦN 5: MOUNT SECRET VÀO ROLLOUT CANARY (FLASK API)
#### 1. Các chỉnh sửa đã thực hiện:
* Cập nhật file `app-api/rollout.yaml` để thêm Volume và VolumeMounts tham chiếu tới `db-secret-local`.
* Thực hiện đồng bộ qua Argo Rollout với chiến lược Canary (10% ➡️ 50% ➡️ 100%).

#### 2. Tại sao cần thực hiện?
* **Bảo mật dữ liệu khi Pod chạy**: Khi mount Secret dạng Volume, mật khẩu sẽ xuất hiện trong container dưới dạng một file tĩnh tại `/etc/secrets/local_db_password` thay vì biến môi trường (env). Việc này tránh việc mật khẩu bị rò rỉ qua các tiến trình in log hoặc dump env của hệ thống.
* **Trình tự đồng bộ an toàn**: Phải triển khai ESO thành công để tạo ra `db-secret-local` trước, sau đó mới deploy Rollout mount volume này. Nếu làm ngược lại, Pod ứng dụng sẽ lỗi do thiếu Volume và sập toàn bộ hệ thống API.

---

## 🏁 KẾT QUẢ ĐẠT ĐƯỢC
Hiện tại hệ thống đã được tối ưu hóa toàn diện và đạt trạng thái lý tưởng:
* **Hệ thống phân quyền (RBAC)** hoạt động chuẩn xác cho Alice, Bob, Carol.
* **Bộ lọc bảo mật (OPA Gatekeeper)** hoạt động tốt trên namespace `demo` mà không ảnh hưởng tới hệ thống.
* **Hệ thống đồng bộ mật khẩu (ESO)** kết nối thành công với AWS Secrets Manager, tự động đồng bộ mật khẩu và cấp phát cho ứng dụng Flask API thông qua volume mount.
* **Ứng dụng API** đang chạy mượt mà dưới dạng Progressive Delivery (Argo Rollout Canary 10%) không hề có lỗi.
