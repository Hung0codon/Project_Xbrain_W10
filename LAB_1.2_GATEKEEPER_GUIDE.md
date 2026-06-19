# HƯỚNG DẪN TRIỂN KHAI & NGHIỆM THU: OPA GATEKEEPER (LAB 1.2)
> **Tài liệu hướng dẫn triển khai bộ kiểm soát chính sách OPA Gatekeeper qua GitOps, cấu hình 4 luật cơ bản bảo mật + 1 luật nâng cao giới hạn Replicas, kèm theo các kịch bản kiểm thử nghiệm thu.**

---

## 1. TẠI SAO CẦN THỰC HIỆN BƯỚC NÀY?
Trong môi trường Kubernetes chạy trên Production, lập trình viên có thể vô tình hoặc cố ý đẩy lên các manifest cấu hình không an toàn (misconfiguration). Ví dụ:
* Sử dụng ảnh Docker gắn tag `:latest` khiến cụm kéo về các phiên bản không đồng nhất, khó kiểm soát phiên bản.
* Chạy Pod bằng quyền root (`runAsUser: 0`), nếu container bị hack, tin tặc sẽ chiếm quyền điều khiển toàn bộ Node vật lý.
* Không thiết lập giới hạn tài nguyên (`resources.limits`), dẫn đến việc một Pod lỗi chiếm dụng hết CPU/RAM của Node và làm sập các Pod khác.
* Sử dụng mạng host (`hostNetwork: true`), cho phép container sniffing các gói tin mạng của các Pod khác trên cùng Node.

**OPA Gatekeeper** đóng vai trò là một **Admission Webhook**, chặn đứng các hành vi vi phạm chính sách bảo mật này ngay tại cổng API Server trước khi tài nguyên được lưu vào cơ sở dữ liệu etcd.

---

## 🗺️ 2. SƠ ĐỒ HOẠT ĐỘNG KIỂM DUYỆT (ADMISSION FLOW)

```mermaid
flowchart TD
    User["Lập trình viên / CI-CD (Apply Manifest)"] -->|HTTP POST| APIServer["Kubernetes API Server"]
    APIServer -->|Mutating Admission| Mutating["Mutating Webhooks"]
    Mutating -->|Validating Admission| Validating{"Validating Webhook <br> (OPA Gatekeeper)"}
    
    subgraph Gatekeeper ["OPA Gatekeeper Engine"]
        Validating -->|Đọc luật từ| Templates["ConstraintTemplates <br> (Quy định logic Rego)"]
        Validating -->|Đối chiếu với| Constraints["Constraints <br> (Phạm vi áp dụng - Namespace: demo)"]
    end
    
    Validating -- "Vi phạm luật bất kỳ" -->|Trả về BadRequest| Denied["❌ REJECT <br> (Từ chối tạo tài nguyên)"]
    Validating -- "Thỏa mãn tất cả luật" -->|Ghi vào etcd| Allowed["✅ ALLOW <br> (Khởi tạo tài nguyên trên cụm)"]
```

### ⚙️ 2.1. CƠ CHẾ HOẠT ĐỘNG DƯỚI GÓC ĐỘ MÃ NGUỒN (UNDER THE HOOD)

Khi có một sự kiện tạo/cập nhật tài nguyên trên Kubernetes (ví dụ: chạy lệnh `kubectl apply`):

1. **Serialization**: API Server chuyển đổi file YAML của người dùng thành đối tượng JSON và gói vào một payload có cấu trúc tên là `AdmissionReview`.
2. **Webhook Request**: API Server gửi payload này qua giao thức HTTPS POST tới Endpoint của OPA Gatekeeper.
3. **Rego Evaluation**: OPA Engine tự động nạp JSON này vào biến môi trường toàn cục `input` trong Rego:
   * Bản thân đặc tả cấu hình của Pod/Deployment sẽ nằm tại đường dẫn: `input.review.object`.
4. **Xử lý logic**: Gatekeeper chạy qua toàn bộ các luật Rego đã biên dịch trong các `ConstraintTemplate`:
   * Nếu điều kiện trong mệnh đề `violation[...]` được thỏa mãn (tức là phát hiện vi phạm chính sách), Gatekeeper sẽ tự động thu thập thông điệp lỗi `msg`.
5. **Webhook Response**: Gatekeeper trả về JSON `AdmissionResponse` cho API Server:
   * Nếu có vi phạm: `allowed: false` kèm theo chuỗi `msg` cảnh báo.
   * Nếu sạch lỗi: `allowed: true`, cho phép API Server tiếp tục ghi tài nguyên vào database `etcd`.

---

### 🔍 2.2. PHÂN TÍCH LUỒNG CODE LOGIC CỦA 4 LUẬT CƠ BẢN

#### 1. Luật cấm tag `:latest` (`k8sallowedtags.yaml`)
* **Đoạn Code Rego**:
  ```rego
  container := input.review.object.spec.containers[_]
  endswith(container.image, ":latest")
  ```
* **Cách hoạt động**: 
  * Cú pháp `spec.containers[_]` duyệt qua tất cả các container khai báo trong Pod (kể cả InitContainers).
  * Hàm `endswith(..., ":latest")` kiểm tra xem đường dẫn ảnh của container có kết thúc bằng chuỗi `:latest` hay không. Nếu có, luật này được kích hoạt và chặn Deployment lại.

#### 2. Luật bắt buộc khai báo CPU/Memory limits (`k8srequiredlimits.yaml`)
* **Đoạn Code Rego**:
  ```rego
  container := input.review.object.spec.containers[_]
  not container.resources.limits.cpu
  ```
  *(Tương tự đối với `limits.memory`)*
* **Cách hoạt động**:
  * Từ khóa `not` trong Rego dùng để kiểm tra sự **vắng mặt** của một thuộc tính.
  * Nếu container không định nghĩa trường `resources.limits.cpu` hoặc `resources.limits.memory`, biểu thức `not` sẽ trả về `true` ➡️ Vi phạm chính sách ➡️ Chặn deploy.

#### 3. Luật cấm chạy bằng quyền Root (`k8scompulsorynonroot.yaml`)
* **Đoạn Code Rego**:
  ```rego
  container := input.review.object.spec.containers[_]
  container.securityContext.runAsUser == 0
  ```
* **Cách hoạt động**:
  * Duyệt vào sâu thuộc tính bảo mật của container `securityContext`.
  * So sánh giá trị `runAsUser` với số `0` (User ID của root). Nếu bằng `0`, pod sẽ bị từ chối khởi chạy.

#### 4. Luật cấm bật HostNetwork (`k8shostnetwork.yaml`)
* **Đoạn Code Rego**:
  ```rego
  input.review.object.spec.hostNetwork == true
  ```
* **Cách hoạt động**:
  * Khác với các luật trên duyệt sâu vào từng container, luật này kiểm tra trực tiếp cấu hình mạng ở cấp độ Pod (`spec.hostNetwork`).
  * Nếu được đặt thành `true`, toàn bộ Pod sẽ bị Admission Controller reject lập tức.

---


## 📂 3. CÁC LUẬT BẢO MẬT ĐÃ CẤU HÌNH (CONSTRAINTS)

Để tránh việc OPA Gatekeeper chặn nhầm các Pod hệ thống (ArgoCD, Prometheus, v.v.), tất cả các luật bên dưới đều được cấu hình giới hạn phạm vi áp dụng **chỉ duy nhất trong namespace `demo`**.

### 1. Cấm sử dụng tag `:latest` (`block-latest-tag.yaml`)
* **Logic**: Chỉ cho phép các tag có dạng phiên bản cụ thể hoặc mã băm sha256. Không chấp nhận tag `:latest`.
* **Constraint**:
  ```yaml
  spec:
    match:
      namespaces: ["demo"]
      kinds:
        - apiGroups: [""]
          kinds: ["Pod"]
  ```

### 2. Bắt buộc có giới hạn tài nguyên (`require-limits.yaml`)
* **Logic**: Yêu cầu tất cả các container chạy trên cụm phải khai báo rõ ràng các trường `limits.cpu` và `limits.memory`.
* **Constraint**:
  ```yaml
  spec:
    match:
      namespaces: ["demo"]
      kinds:
        - apiGroups: [""]
          kinds: ["Pod"]
  ```

### 3. Cấm chạy quyền Root (`enforce-non-root.yaml`)
* **Logic**: Bắt buộc trường `securityContext.runAsNonRoot` phải là `true`, hoặc `runAsUser` phải khác `0`.
* **Constraint**:
  ```yaml
  spec:
    match:
      namespaces: ["demo"]
      kinds:
        - apiGroups: [""]
          kinds: ["Pod"]
  ```

### 4. Cấm cấu hình mạng Host (`block-host-network.yaml`)
* **Logic**: Ngăn chặn cấu hình `hostNetwork: true` trong đặc tả Pod.
* **Constraint**:
  ```yaml
  spec:
    match:
      namespaces: ["demo"]
      kinds:
        - apiGroups: [""]
          kinds: ["Pod"]
  ```

### 5. Luật Custom nâng cao: Giới hạn Replicas <= 5 (`max-replicas-5.yaml`)
* **Logic**: Ngăn chặn lập trình viên scale số lượng bản sao (replicas) của Deployment hoặc Rollout vượt quá 5 để tối ưu hóa tài nguyên.
* **Constraint**:
  ```yaml
  spec:
    match:
      namespaces: ["demo"]
      kinds:
        - apiGroups: ["apps", "argoproj.io"]
          kinds: ["Deployment", "Rollout"]
    parameters:
      max: 5
  ```

---

## ⚠️ 4. BẪY THIẾT KẾ & GIẢI PHÁP GỠ TREO HỆ THỐNG
* **Bẫy 1 (Treo cụm hệ thống)**: Nếu áp dụng các luật này trên toàn cụm (`namespaces: ["*"]` hoặc không khai báo match namespace), các Pod hệ thống như Prometheus, Alertmanager hoặc ArgoCD (thường chạy bằng quyền root hoặc không set limits) sẽ bị OPA chặn đứng trong quá trình cài đặt. Khi đó, ArgoCD sẽ bị treo ở Wave 0 và sập toàn bộ hệ thống.
  * **👉 Giải pháp**: Chỉ định rõ ràng `match.namespaces: ["demo"]` trong mọi Constraint.
* **Bẫy 2 (App Flask API tự bị chặn)**: Trước khi kích hoạt luật, cần đảm bảo bản thân file `rollout.yaml` của Flask API đã khai báo đầy đủ limits tài nguyên, cấu hình non-root, và tag image cụ thể (không dùng latest).
  * **👉 Giải pháp**: Sửa đổi file `app-api/rollout.yaml` để khớp với 4 luật trên trước khi nạp Gatekeeper Constraints.

---

## 🧪 5. KỊCH BẢN KIỂM THỬ VÀ NGHIỆM THU (TEST SCRIPTS)

Thực hiện chạy các lệnh test sau trong namespace `demo` để nghiệm thu:

### 🔬 Kiểm thử 1: Thử nghiệm vi phạm tag `:latest`
* **Lệnh**:
  ```bash
  kubectl run test-latest --image=nginx:latest -n demo
  ```
* **Kết quả kỳ vọng**: Bị từ chối (REJECT) kèm thông báo:
  ```bash
  Error from server (Forbidden): admission webhook "validation.gatekeeper.sh" denied the request: [block-latest-tag] ...
  ```

### 🔬 Kiểm thử 2: Thử tạo Pod thiếu limits tài nguyên
* **Lệnh**:
  ```bash
  kubectl run test-no-limits --image=nginx:alpine -n demo
  ```
* **Kết quả kỳ vọng**: Bị từ chối do thiếu `cpu` hoặc `memory` limits.

### 🔬 Kiểm thử 3: Thử chạy Pod bằng quyền root
* **Lệnh chạy Pod cấu hình chạy root**:
  ```bash
  kubectl apply -f - <<EOF
  apiVersion: v1
  kind: Pod
  metadata:
    name: test-root-pod
    namespace: demo
  spec:
    containers:
    - name: test-root
      image: nginx:alpine
      securityContext:
        runAsUser: 0
  EOF
  ```
* **Kết quả kỳ vọng**: Bị từ chối: `runAsUser: 0 is not allowed / must run as non-root`.

### 🔬 Kiểm thử 4: Thử bật Host Network
* **Lệnh**:
  ```bash
  kubectl apply -f - <<EOF
  apiVersion: v1
  kind: Pod
  metadata:
    name: test-hostnetwork-pod
    namespace: demo
  spec:
    hostNetwork: true
    containers:
    - name: test-net
      image: nginx:alpine
  EOF
  ```
* **Kết quả kỳ vọng**: Bị từ chối: `hostNetwork: true is not allowed`.

### 🔬 Kiểm thử 5: Thử scale Replicas vượt quá 5
* **Lệnh tạo Deployment với 6 Replicas**:
  ```bash
  kubectl apply -f - <<EOF
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: test-replicas
    namespace: demo
  spec:
    replicas: 6
    selector:
      matchLabels:
        app: test
    template:
      metadata:
        labels:
          app: test
      spec:
        containers:
        - name: nginx
          image: nginx:alpine
  EOF
  ```
* **Kết quả kỳ vọng**: Bị từ chối: `Replicas 6 exceeds maximum allowed value of 5`.
