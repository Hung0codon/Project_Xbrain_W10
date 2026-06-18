# HƯỚNG DẪN CHI TIẾT TRIỂN KHAI LAB 2.1: EXTERNAL SECRETS OPERATOR & OPA GATEKEEPER
> **Tài liệu hướng dẫn toàn diện từ A-Z về quá trình gỡ lỗi chính sách bảo mật, tích hợp AWS Secrets Manager qua External Secrets Operator (ESO) và cấu hình GitOps với ArgoCD.**

---

## 🗺️ Sơ đồ Kiến trúc Hoạt động (Mermaid)

```mermaid
graph TD
    subgraph AWS Cloud ["AWS Cloud"]
        ASM["AWS Secrets Manager <br> (Secret: prod/db/credentials)"]
    end

    subgraph Minikube Cluster ["Minikube Local Cluster"]
        direction TB
        subgraph Namespace: argocd ["Namespace: argocd"]
            ArgoCD["ArgoCD Controller"]
        end

        subgraph Namespace: external-secrets ["Namespace: external-secrets"]
            ESO["External Secrets Operator (Controller)"]
        end

        subgraph Namespace: demo ["Namespace: demo"]
            SecretCreds["K8s Secret: aws-secret-creds <br> (AWS Keys - Local Only)"]
            Store["SecretStore: aws-secretsmanager-store"]
            ExtSecret["ExternalSecret: db-external-secret"]
            LocalSecret["K8s Secret: db-secret-local <br> (Mật khẩu tự động kéo về)"]
            Pod["Flask API Pod <br> (Mounts db-secret-local as Volume)"]
        end
        
        subgraph Namespace: gatekeeper-system ["Namespace: gatekeeper-system"]
            GK["OPA Gatekeeper <br> (Chỉ kiểm duyệt Namespace: demo)"]
        end
    end

    %% Cấp quyền và cấu hình
    SecretCreds -->|Cung cấp API Keys| Store
    Store -->|Đăng nhập & Xác thực| ASM
    ExtSecret -->|Tham chiếu| Store
    ExtSecret -->|Đọc Key: prod/db/credentials| ASM
    ExtSecret -->|Tự động sinh ra & Cập nhật| LocalSecret
    LocalSecret -->|Mount dưới dạng File| Pod
    GK -.->|Kiểm duyệt giới hạn tài nguyên| Pod
    ArgoCD -->|Đồng bộ cấu hình từ Git| Store
    ArgoCD -->|Đồng bộ cấu hình từ Git| ExtSecret
    ArgoCD -->|Đồng bộ cấu hình từ Git| Pod
```

---

## 📋 BẢNG TỔNG HỢP CÁC BƯỚC THỰC HIỆN

| Bước | Hành động | Nội dung chính | Tại sao phải thực hiện? |
| :--- | :--- | :--- | :--- |
| **1** | **Git Clone & Init** | Khởi tạo repo GitOps local. | Lấy bộ khung cấu hình khai báo (Declarative Manifests). |
| **2** | **Cấu hình OPA Gatekeeper** | Giới hạn phạm vi Constraints chỉ áp dụng cho namespace `demo`. | Tránh việc OPA chặn đứng các Pod hệ thống (như Prometheus Admission Job), gây treo cụm. |
| **3** | **Tạo Secret trên AWS** | Tạo Secret `prod/db/credentials` trên AWS Secrets Manager. | Lưu trữ bảo mật tập trung mật khẩu trên Cloud, không đưa mật khẩu lên Git. |
| **4** | **Tạo AWS Credentials trong cụm** | Tạo Secret `aws-secret-creds` trực tiếp bằng CLI trong namespace `demo`. | Cung cấp khóa API để cụm K8s kết nối lên AWS mà không bị lộ keys trên Git. |
| **5** | **Chuẩn hóa tệp tin ESO** | Tách biệt tệp `Application` của ArgoCD và `SecretStore` tài nguyên. | Sửa lỗi hoán đổi file giúp ArgoCD nhận diện đúng tài nguyên đồng bộ. |
| **6** | **Khôi phục AWS Secret bị xóa** | Sử dụng AWS CLI để khôi phục Secret bị "Marked for deletion". | Sửa lỗi `SecretSyncedError` của ESO khi không tìm thấy Secret đang hoạt động trên AWS. |
| **7** | **Mount Secret vào App API** | Cấu hình Volume và VolumeMounts trong file `rollout.yaml`. | Gắn mật khẩu vào container dưới dạng file để bảo mật tối đa và tự động reload. |

---

## 🛠️ CHI TIẾT TỪNG BƯỚC THỰC HIỆN

### Bước 1: Clone Git Repository và khởi tạo môi trường
Bạn thực hiện clone repository dự án từ GitHub về máy cá nhân:
```powershell
git clone https://github.com/Hung0codon/Project_Xbrain_W10.git
```
> [!NOTE]
> **Tại sao cần bước này?**
> Mẫu thiết kế GitOps yêu cầu toàn bộ trạng thái của hệ thống phải được mô tả thông qua code (Infrastructure as Code). Clone repository giúp bạn có được môi trường khai báo cục bộ trước khi đồng bộ lên ArgoCD.

---

### Bước 2: Tối ưu hóa OPA Gatekeeper (Khắc phục lỗi treo cụm)
#### Vấn đề phát sinh:
Khi OPA Gatekeeper được kích hoạt, mặc định các quy tắc bảo mật (như cấm tag `latest`, bắt buộc cấu hình `resources.limits`) được áp dụng **toàn cụm (Cluster-wide)**. Khi cài đặt Prometheus Stack, Job hệ thống `kube-prometheus-stack-admission-create` không khai báo limits tài nguyên ➡️ Bị Gatekeeper chặn đứng ➡️ Cụm bị kẹt trạng thái đồng bộ vô thời hạn.

#### Cách xử lý:
Chúng ta đã sửa đổi cả 5 file Constraint trong thư mục `gatekeeper/constraints/` để chỉ áp dụng cho namespace `demo` bằng cách thêm bộ lọc `namespaceSelector` hoặc chỉ định trực tiếp danh sách `namespaces`:

*Ví dụ chỉnh sửa trong `gatekeeper/constraints/require-limits.yaml`:*
```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLimits
metadata:
  name: require-cpu-memory-limits
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces:
      - demo # <--- CHỈ áp dụng luật lên namespace demo
```

> [!TIP]
> **Lưu ý tối ưu:** Tôi đã xóa bỏ phần cấu hình `parameters` thừa trong `require-limits.yaml` vì Rego template của nó không định nghĩa schema parameters này. Việc xóa đi giúp ArgoCD không bị báo lệch cấu hình `OutOfSync` do Kubernetes tự động cắt tỉa các trường không khai báo trong CRD.

> [!IMPORTANT]
> **Tại sao cần bước này?**
> Giúp cô lập môi trường kiểm thử (sandbox `demo`) độc lập với hệ thống. Các Pod hệ thống (Prometheus, Grafana, ArgoCD) sẽ không bị chặn khởi động, đảm bảo toàn cụm hoạt động bình thường.

---

### Bước 3: Tạo Secret trên AWS Secrets Manager
Đăng nhập vào AWS Console và tạo Secret:
* **Loại secret**: Other type of secret
* **Key/Value**: `db_password` / `MatKhauGocCuaBan123`
* **Tên Secret (Secret name)**: `prod/db/credentials`
* **Vùng (Region)**: Chọn vùng hoạt động (ví dụ: `ap-southeast-1` - Singapore).

> [!NOTE]
> **Tại sao cần bước này?**
> Đảm bảo nguyên tắc bảo mật thông tin (Secret Management): Mật khẩu gốc được quản lý tập trung và bảo vệ bởi AWS, tránh nguy cơ rò rỉ khi đẩy mã nguồn lên GitHub công khai.

---

### Bước 4: Cấp quyền kết nối AWS cho Minikube (Local Secret)
Vì thông tin Access Key ID và Secret Access Key là nhạy cảm, bạn tuyệt đối **không được commit lên Git**. Bạn tạo thủ công K8s Secret trực tiếp trong namespace `demo` bằng CLI:
```powershell
kubectl create secret generic aws-secret-creds -n demo --from-literal=access-key-id=YOUR_AWS_ACCESS_KEY_ID --from-literal=secret-access-key=YOUR_AWS_SECRET_ACCESS_KEY
```

> [!NOTE]
> **Tại sao cần bước này?**
> Cung cấp thông tin xác thực để External Secrets Operator (ESO) có quyền gọi các API của AWS Secrets Manager nhằm lấy mật khẩu về cụm một cách an toàn.

---

### Bước 5: Tổ chức lại cấu trúc thư mục cấu hình ESO
Để đảm bảo cấu trúc dự án chuẩn hóa theo mô hình App-of-Apps của ArgoCD, chúng ta đã tiến hành phân tách các file bị hoán đổi nhầm vị trí trước đó:

1. **[`argocd/apps/eso.yaml`](file:///d:/FIle_doc/W10/W10_Project/temp/argocd/apps/eso.yaml)** (Khai báo Application cho ArgoCD quản lý):
   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: app-eso-config
     namespace: argocd
     annotations:
       argocd.argoproj.io/sync-wave: "1" # Chạy ở Wave 1 sau khi Operator đã sẵn sàng ở Wave 0
   spec:
     project: default
     source:
       repoURL: 'https://github.com/Hung0codon/Project_Xbrain_W10.git'
       targetRevision: main
       path: eso # Chỉ tới thư mục cấu hình tài nguyên
     destination:
       server: 'https://kubernetes.default.svc'
       namespace: demo
     syncPolicy:
       automated:
         prune: true
         selfHeal: true
   ```

2. **[`eso/secret-store.yaml`](file:///d:/FIle_doc/W10/W10_Project/temp/eso/secret-store.yaml)** (Tài nguyên cấu hình cổng kết nối AWS):
   ```yaml
   apiVersion: external-secrets.io/v1beta1
   kind: SecretStore
   metadata:
     name: aws-secretsmanager-store
     namespace: demo
   spec:
     provider:
       aws:
         service: SecretsManager
         region: ap-southeast-1 # Sửa thành Singapore khớp với tài khoản AWS của bạn
         auth:
           secretRef:
             accessKeyIDSecretRef:
               name: aws-secret-creds
               key: access-key-id
             secretAccessKeySecretRef:
               name: aws-secret-creds
               key: secret-access-key
   ```

3. **[`eso/external-secret.yaml`](file:///d:/FIle_doc/W10/W10_Project/temp/eso/external-secret.yaml)** (Khai báo ánh xạ Secret):
   ```yaml
   apiVersion: external-secrets.io/v1beta1
   kind: ExternalSecret
   metadata:
     name: db-external-secret
     namespace: demo
   spec:
     refreshInterval: "30s" # Tự động quét cập nhật mật khẩu mới mỗi 30 giây
     secretStoreRef:
       name: aws-secretsmanager-store
       kind: SecretStore
     target:
       name: db-secret-local # Tên K8s Secret được ESO tự động sinh ra trong cụm
       creationPolicy: Owner
     data:
       - secretKey: local_db_password
         remoteRef:
           key: prod/db/credentials
           property: db_password
   ```

---

### Bước 6: Khôi phục AWS Secret bị xóa (Gỡ lỗi SecretSyncedError)
#### Vấn đề phát sinh:
Khi kiểm tra trạng thái ExternalSecret bằng lệnh `kubectl describe externalsecret db-external-secret -n demo`, chúng ta nhận được thông báo lỗi từ AWS:
`InvalidRequestException: You can't perform this operation on the secret because it was marked for deletion.`
Do Secret `prod/db/credentials` trước đó đã bị xóa trên AWS (vẫn trong thời gian chờ xóa thực tế), ESO không thể đọc được dữ liệu.

#### Cách xử lý:
Chúng ta sử dụng AWS CLI để gửi lệnh khôi phục Secret đó trực tiếp trên vùng Singapore (`ap-southeast-1`):
```powershell
$env:AWS_ACCESS_KEY_ID="AKIA5YDPDNKWDJOCFV7L"
$env:AWS_SECRET_ACCESS_KEY="YOUR_SECRET_ACCESS_KEY"
$env:AWS_DEFAULT_REGION="ap-southeast-1"
aws secretsmanager restore-secret --secret-id prod/db/credentials
```
> **Kết quả:** Secret được khôi phục thành công. ESO lập tức kết nối lại được và tự động sinh ra tệp mật khẩu cục bộ **`db-secret-local`** trong namespace `demo`.

---

### Bước 7: Cấu hình ứng dụng Rollout để Mount Secret
Sau khi kiểm tra thấy K8s Secret `db-secret-local` đã tồn tại thực tế trong cụm bằng lệnh `kubectl get secret db-secret-local -n demo`, ta tiến hành cập nhật tệp **[`app-api/rollout.yaml`](file:///d:/FIle_doc/W10/W10_Project/temp/app-api/rollout.yaml)**:

```yaml
# Sửa spec.template.spec trong rollout.yaml:
    spec:
      containers:
      - name: api
        image: ghcr.io/hung0codon/w10-api:0.0.1
        # ... giữ nguyên cấu hình cũ ...
        volumeMounts:
        - name: secret-volume
          mountPath: /etc/secrets # Thư mục ứng dụng sẽ đọc mật khẩu
          readOnly: true
      volumes:
      - name: secret-volume
        secret:
          secretName: db-secret-local # Trỏ đúng vào tên secret do ESO sinh ra
```
Sau đó, tiến hành đẩy thay đổi lên Git:
```bash
git add app-api/rollout.yaml
git commit -m "feat: mount db-secret-local as volume in API rollout"
git push origin main
```

> [!CAUTION]
> **Tại sao không cấu hình mount trước khi tạo Secret?**
> Nếu bạn cố gắng deploy Rollout này trước khi `db-secret-local` được tạo ra trong cụm, Kubelet sẽ không tìm thấy volume nguồn để gắn vào Pod. Pod ứng dụng sẽ bị lỗi khởi động (`MountVolume.SetUp failed`) và sập toàn bộ hệ thống API. Phải chạy theo đúng trình tự: **Tạo ESO/AWS Secret trước ➡️ Kiểm tra đã có secret local ➡️ Mới cập nhật file Rollout của ứng dụng.**

---

## 🏁 KẾT QUẢ ĐẠT ĐƯỢC
Sau khi hoàn thành tất cả các bước trên, hệ thống hoạt động chính xác theo đúng mô hình tự động hóa GitOps:
1. **ArgoCD**: Đồng bộ thành công, không còn trạng thái treo hay báo lệch cấu hình `OutOfSync`.
2. **External Secrets Operator**: Chạy ổn định, tự động cập nhật mật khẩu từ AWS Secrets Manager về local K8s Secret mỗi 30 giây.
3. **Flask API App**: Nhận mật khẩu thành công thông qua file `/etc/secrets/local_db_password` được mount trực tiếp dưới dạng Volume, sẵn sàng cho việc cập nhật mật khẩu động (Zero-Downtime Secret Rotation).
