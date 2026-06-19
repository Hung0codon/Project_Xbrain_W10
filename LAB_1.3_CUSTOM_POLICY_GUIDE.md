# HƯỚNG DẪN TRIỂN KHAI & NGHIỆM THU: CHÍNH SÁCH CUSTOM (LAB 1.3)
> **Tài liệu hướng dẫn tự phát triển luật kiểm duyệt tài nguyên (Custom ConstraintTemplate) sử dụng ngôn ngữ Rego để giới hạn số lượng Replicas tối đa là 5, kèm theo hướng dẫn kiểm thử và đồng bộ hóa qua GitOps.**

---

## 1. TẠI SAO CẦN THỰC HIỆN BƯỚC NÀY?
Mặc dù thư viện chính thức của OPA Gatekeeper cung cấp sẵn nhiều luật bảo mật phổ biến, mỗi doanh nghiệp/dự án thường có các quy chuẩn vận hành riêng biệt (Custom Policy). 
* **Ví dụ**: Giới hạn tài nguyên máy chủ trên môi trường Local Minikube. Nếu lập trình viên vô tình scale một Deployment lên 20 hay 50 bản sao, RAM/CPU của máy local sẽ lập tức bị cạn kiệt, làm sập toàn bộ hệ thống Minikube.
* Do đó, chúng ta cần tự tay viết một luật **ConstraintTemplate** sử dụng ngôn ngữ lập trình khai báo **Rego** để chủ động bắt lỗi và chặn đứng các yêu cầu deploy có số lượng Replicas vượt quá mức cho phép (ở đây cấu hình là 5).

---

## 2. GIẢI THÍCH CHI TIẾT LOGIC REGO (CONSTRAINTTEMPLATE)

Dưới đây là đoạn code Rego tự viết trong tệp tin `gatekeeper/templates/k8smaxreplicas.yaml`:

```rego
package k8smaxreplicas

violation[{"msg": msg}] {
  # 1. Tìm thuộc tính replicas trong file Deployment gửi lên
  replicas := input.review.object.spec.replicas
  
  # 2. Lấy số lượng tối đa được phép từ tham số cấu hình (Constraint)
  max_allowed := input.parameters.maxReplicas
  
  # 3. So sánh: Nếu số replicas gửi lên LỚN HƠN số cho phép -> Báo vi phạm!
  replicas > max_allowed
  
  # 4. Trả về câu thông báo lỗi cho người dùng
  msg := sprintf("Số lượng bản sao (%v) vượt quá mức tối đa cho phép (%v) của hệ thống!", [replicas, max_allowed])
}
```

### 🔍 Giải thích cú pháp:
* `input.review.object`: Đại diện cho toàn bộ nội dung của tệp Manifest YAML mà người dùng gửi lên Kubernetes API Server.
* `input.review.object.spec.replicas`: Lấy giá trị khai báo tại trường `spec.replicas` của Deployment.
* `input.parameters.maxReplicas`: Lấy giá trị tham số đầu vào được cấu hình động tại file Constraint (ở đây là số `5`).
* `replicas > max_allowed`: Điều kiện kích hoạt trạng thái Vi phạm (Violation). Trong Rego, nếu tất cả các biểu thức trong khối `{}` đều đúng (true), khối đó sẽ trả về kết quả vi phạm.
* `sprintf(...)`: Sinh ra câu thông báo lỗi thân thiện với người dùng để hiển thị trực tiếp trên terminal của họ khi deploy bị từ chối.

---

## 📂 3. CÁC TỆP TIN CẤU HÌNH GITOPS

### A. Tệp ConstraintTemplate (`gatekeeper/templates/k8smaxreplicas.yaml`)
Định nghĩa cấu trúc dữ liệu đầu vào và chứa mã nguồn Rego để kiểm tra:
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8smaxreplicas
spec:
  crd:
    spec:
      names:
        kind: K8sMaxReplicas
      validation:
        openAPIV3Schema:
          type: object
          properties:
            maxReplicas:
              type: integer
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8smaxreplicas

        violation[{"msg": msg}] {
          replicas := input.review.object.spec.replicas
          max_allowed := input.parameters.maxReplicas
          replicas > max_allowed
          msg := sprintf("Số lượng bản sao (%v) vượt quá mức tối đa cho phép (%v) của hệ thống!", [replicas, max_allowed])
        }
```

### B. Tệp Constraint áp dụng luật (`gatekeeper/constraints/max-replicas-5.yaml`)
Áp dụng luật vừa định nghĩa lên đối tượng `Deployment` trong namespace `demo` với tham số giới hạn tối đa là 5:
```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sMaxReplicas
metadata:
  name: limit-deployment-replicas
spec:
  enforcementAction: deny # Chặn đứng ngay lập tức nếu vi phạm
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment"]
    namespaces:
      - demo
  parameters:
    maxReplicas: 5
```

---

## 🧪 4. KỊCH BẢN KIỂM THỬ VÀ NGHIỆM THU (TEST SCRIPTS)

### 🔬 Thử nghiệm 1: Tạo Deployment vi phạm chính sách (Replicas = 6)
* **Hành động**: Thử tạo một Deployment bất kỳ với số lượng bản sao là 6 trong namespace `demo`:
  ```bash
  kubectl apply -f - <<EOF
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: test-bad-deployment
    namespace: demo
  spec:
    replicas: 6
    selector:
      matchLabels:
        app: bad-app
    template:
      metadata:
        labels:
          app: bad-app
      spec:
        containers:
        - name: nginx
          image: nginx:alpine
          resources:
            limits:
              cpu: "100m"
              memory: "128Mi"
            requests:
              cpu: "50m"
              memory: "64Mi"
          securityContext:
            runAsNonRoot: true
            runAsUser: 1000
  EOF
  ```
* **Kết quả kỳ vọng (REJECT)**: API Server từ chối tiếp nhận và in ra lỗi:
  ```bash
  Error from server (Forbidden): admission webhook "validation.gatekeeper.sh" denied the request: [limit-deployment-replicas] Số lượng bản sao (6) vượt quá mức tối đa cho phép (5) của hệ thống!
  ```
  ➡️ **Thành công**: Hệ thống đã chặn đứng deployment xấu và in ra đúng câu thông báo lỗi cấu hình bằng Rego!

---

### 🔬 Thử nghiệm 2: Tạo Deployment hợp lệ (Replicas = 3)
* **Hành động**: Tạo thử deployment với số lượng bản sao hợp lệ là 3:
  ```bash
  kubectl apply -f - <<EOF
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: test-good-deployment
    namespace: demo
  spec:
    replicas: 3
    selector:
      matchLabels:
        app: good-app
    template:
      metadata:
        labels:
          app: good-app
      spec:
        containers:
        - name: nginx
          image: nginx:alpine
          resources:
            limits:
              cpu: "100m"
              memory: "128Mi"
            requests:
              cpu: "50m"
              memory: "64Mi"
          securityContext:
            runAsNonRoot: true
            runAsUser: 1000
  EOF
  ```
* **Kết quả kỳ vọng (PASS)**: Deployment được khởi tạo thành công:
  ```bash
  deployment.apps/test-good-deployment created
  ```
  *(Và sau đó bạn dọn dẹp bằng lệnh: `kubectl delete deployment test-good-deployment -n demo`)*.
  ➡️ **Thành công**: Hệ thống cho phép triển khai khi tài nguyên tuân thủ đúng luật.
