# PROJECT COMMANDS CHEATSHEET - W10
> **Tổng hợp toàn bộ các câu lệnh hữu ích sử dụng xuyên suốt dự án (Minikube, Kubectl, ArgoCD, Argo Rollouts, AWS Secrets Manager, OPA Gatekeeper).**

---

## 1. NHÓM LỆNH MINIKUBE (Quản lý Cluster)

* **Khởi động cụm Minikube (với driver Docker)**:
  ```bash
  minikube start -p w10 --driver=docker
  ```
* **Dừng cụm Minikube**:
  ```bash
  minikube stop -p w10
  ```
* **Xóa cụm Minikube (khi muốn làm sạch hoàn toàn)**:
  ```bash
  minikube delete -p w10
  ```
* **Chuyển ngữ cảnh (Context) kubectl về Minikube**:
  ```bash
  kubectl config use-context w10
  ```
* **Mở trang quản trị (Dashboard) của Minikube**:
  ```bash
  minikube dashboard -p w10
  ```

---

## 2. NHÓM LỆNH ARGOCD (Quản lý GitOps & Application)

* **Tải mật khẩu admin mặc định ban đầu**:
  ```powershell
  # Trên Windows PowerShell:
  kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | ForEach-Object { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }; echo ""
  
  # Trên Linux/macOS hoặc Git Bash:
  kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode; echo
  ```
* **Port-forward để truy cập ArgoCD UI (http://localhost:8080)**:
  ```bash
  kubectl port-forward svc/argocd-server -n argocd 8080:443
  ```
* **Liệt kê danh sách Application trong cụm**:
  ```bash
  kubectl get app -n argocd
  ```
* **Xem trạng thái chi tiết của một Application**:
  ```bash
  kubectl describe app app-eso-config -n argocd
  ```
* **Buộc ArgoCD đồng bộ lại ứng dụng (Manual Sync)**:
  ```bash
  kubectl patch app root -n argocd --type merge -p '{"operation":{"sync":{"syncStrategy":{"hook":{}}}}}'
  ```
* **Xóa thông tin trạng thái lỗi cũ để đồng bộ lại (Clear Operation State)**:
  ```bash
  kubectl patch app app-eso-config -n argocd --type merge -p '{"status":{"operationState":null}}'
  ```

---

## 3. NHÓM LỆNH KUBECTL (Thao tác tài nguyên K8s)

* **Xem danh sách Namespace**:
  ```bash
  kubectl get ns
  ```
* **Xem toàn bộ Pod trong tất cả namespace**:
  ```bash
  kubectl get pods -A
  ```
* **Xem Pod trong namespace cụ thể**:
  ```bash
  kubectl get pods -n demo
  kubectl get pods -n external-secrets
  ```
* **Mô tả chi tiết và xem lỗi của Pod/Service/Resource**:
  ```bash
  kubectl describe pod <ten-pod> -n demo
  kubectl describe secretstore aws-secretsmanager-store -n demo
  kubectl describe externalsecret db-external-secret -n demo
  ```
* **Xem log thời gian thực của Pod để debug**:
  ```bash
  kubectl logs <ten-pod> -n external-secrets --tail=100
  kubectl logs -n external-secrets -l app.kubernetes.io/name=external-secrets -f
  ```
* **Tạo Secret cục bộ chứa thông tin kết nối AWS (KHÔNG commit git)**:
  ```bash
  kubectl create secret generic aws-secret-creds -n demo --from-literal=access-key-id=YOUR_ACCESS_KEY --from-literal=secret-access-key=YOUR_SECRET_KEY
  ```
* **Xem thông tin giải mã của Secret cục bộ**:
  ```bash
  kubectl get secret db-secret-local -n demo -o jsonpath="{.data.local_db_password}" | base64 --decode
  ```

---

## 4. NHÓM LỆNH ARGO ROLLOUTS (Quản lý Canary & Progressive Delivery)

* **Xem danh sách các bản Rollout**:
  ```bash
  kubectl get rollout -n demo
  ```
* **Theo dõi tiến trình Rollout (Canary) trực quan**:
  ```bash
  kubectl argo rollouts get rollout api -n demo --watch
  ```
* **Thúc đẩy bước Canary kế tiếp (Promote/Skip Pause)**:
  ```bash
  kubectl argo rollouts promote api -n demo
  ```
* **Hủy bỏ đợt triển khai, quay về phiên bản ổn định trước đó (Abort/Rollback)**:
  ```bash
  kubectl argo rollouts abort api -n demo
  ```
* **Xem danh sách các đợt đo lường độ ổn định (AnalysisRun)**:
  ```bash
  kubectl get analysisrun -n demo
  ```

---

## 5. NHÓM LỆNH AWS CLI SECRETS MANAGER (Tương tác Cloud)

* **Khôi phục Secret bị xóa (đang trong thời gian chờ xóa)**:
  ```bash
  # Cần set biến môi trường AWS credentials trước khi chạy
  aws secretsmanager restore-secret --secret-id prod/db/credentials --region ap-southeast-1
  ```
* **Xem giá trị thực tế của Secret trên AWS**:
  ```bash
  aws secretsmanager get-secret-value --secret-id prod/db/credentials --region ap-southeast-1
  ```
* **Tạo Secret mới từ CLI**:
  ```bash
  aws secretsmanager create-secret --name prod/db/credentials --secret-string '{"db_password":"MatKhauGocCuaBan123"}' --region ap-southeast-1
  ```
