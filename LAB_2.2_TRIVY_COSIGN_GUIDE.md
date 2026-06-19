# HƯỚNG DẪN CHI TIẾT TỪ A-Z: TRIỂN KHAI TRIVY + COSIGN (LAB 2.2)
> **Tài liệu hướng dẫn triển khai hệ thống quét lỗ hổng bảo mật (Trivy) và xác thực chữ ký số Image (Cosign) trên hạ tầng Kubernetes GitOps.**

---

## 🗺️ Sơ đồ Luồng hoạt động Bảo mật (Trivy + Cosign)

```mermaid
flowchart TD
    subgraph CI_Pipeline ["1. GitHub Actions (CI Pipeline)"]
        direction TB
        Code["Code Thay Đổi (src/api)"] --> Build["Build Docker Image"]
        Build --> TrivyScan{"Quét Trivy <br> (Lọc High/Critical)"}
        
        TrivyScan -- "Phát hiện CVE nặng" --> FailCI["❌ Fail Pipeline <br> (Chặn không cho Sign/Push)"]
        TrivyScan -- "Clean CVE" --> Push["Push Image lên GHCR"]
        
        Push --> CosignSign["Ký Image bằng Cosign <br> (Dùng Private Key)"]
        CosignSign --> UpdateRollout["Cập nhật tag mới vào rollout.yaml"]
    end

    subgraph K8s_Cluster ["2. Minikube Cluster (Admission Control)"]
        direction TB
        ArgoCD["ArgoCD Engine"] -->|Sync Rollout| K8sAPI["Kubernetes API Server"]
        
        subgraph Admission_Webhook ["Lớp Kiểm Duyệt Admission Webhook"]
            Sigstore["Sigstore Policy Controller"]
            Policy["ClusterImagePolicy <br> (Chứa Public Key)"]
            
            Sigstore -->|Đọc luật| Policy
        end

        K8sAPI -->|Yêu cầu chạy Pod| WebhookCheck{"Kiểm tra Chữ ký"}
        WebhookCheck -- "Chưa ký / Ký sai Key" --> Reject["❌ Từ chối chạy Pod <br> (Block Deployment)"]
        WebhookCheck -- "Chữ ký hợp lệ" --> RunPod["✅ Cho phép chạy Pod"]
    end

    %% Mối liên kết
    UpdateRollout -->|Commit Git| ArgoCD
    CosignSign -->|Đẩy Chữ Ký Số| GHCR["GitHub Registry (GHCR)"]
    K8sAPI -->|Kéo Image & Chữ Ký| GHCR
```

---

## 🔒 1. TẠI SAO CẦN BỘ ĐÔI TRIVY + COSIGN?

Nếu không có hai công cụ này, chuỗi cung ứng phần mềm (Software Supply Chain) của bạn sẽ đối mặt với 2 rủi ro nghiêm trọng:

### ⚠️ Rủi ro 1: Sử dụng Image nền hoặc thư viện chứa mã độc (CVE)
* **Vấn đề**: Dev viết code rất sạch và an toàn, nhưng Docker Base Image (như `python:3.9`) hoặc các thư viện cài qua `pip` chứa các lỗ hổng bảo mật nghiêm trọng (`HIGH` / `CRITICAL`). Hacker có thể lợi dụng lỗi này để chiếm quyền kiểm soát Container.
* **Giải pháp**: **Trivy** tự động quét mã lỗi bảo mật của Image ngay khi vừa build xong. Nếu phát hiện lỗi lớn, Trivy trả về `exit-code 1` để dừng toàn bộ quy trình CI, ngăn chặn đẩy Image lỗi lên máy chủ.

### ⚠️ Rủi ro 2: Bị tráo đổi Image trên đường truyền (Man-in-the-middle / Registry Hack)
* **Vấn đề**: Hacker có thể xâm nhập vào Docker Registry (Docker Hub, GHCR) và đẩy đè lên một bản Image độc hại có trùng tên, trùng tag với Image của bạn. Cụm Kubernetes sẽ kéo Image độc hại đó về chạy mà hoàn toàn không hề hay biết nội dung bên trong đã bị thay đổi.
* **Giải pháp**: **Cosign** ký xác nhận (Digital Signature) lên Image hợp pháp bằng Private Key. Khi cụm Kubernetes chuẩn bị khởi chạy Pod, **Sigstore Policy Controller** (ở tầng Admission Webhook) sử dụng Public Key để xác thực chữ ký. Image không có chữ ký hợp lệ sẽ lập tức bị **từ chối (Block)**.

---

## 🛠️ 2. QUY TRÌNH TRIỂN KHAI CHI TIẾT TỪNG BƯỚC

### 🟩 BƯỚC 1: TẢI COSIGN VÀ SINH CẶP KHÓA (CHẠY TRÊN MÁY LOCAL)

Vì máy của bạn chưa cài đặt Chocolatey (`choco`) hoặc Cosign, bạn có thể tải bản binary trực tiếp về máy thông qua PowerShell:

#### 1. Tải Cosign.exe bằng PowerShell:
Mở PowerShell tại thư mục `W10_Project/temp` và chạy lệnh:
```powershell
Invoke-WebRequest -Uri "https://github.com/sigstore/cosign/releases/download/v2.2.3/cosign-windows-amd64.exe" -OutFile "cosign.exe"
```

#### 2. Sinh cặp khóa chữ ký:
Chạy lệnh sau để tạo cặp khóa (lưu ý: Cosign yêu cầu nhập mật khẩu bảo vệ khóa):
```powershell
# Thiết lập mật khẩu bảo vệ (Passphrase) cho khóa
$env:COSIGN_PASSWORD="MatKhauBaoVeKhoaCosign123"

# Sinh cặp khóa
.\cosign.exe generate-key-pair
```
* **Kết quả**: Bạn sẽ nhận được 2 tệp tin trong thư mục:
  * `cosign.key`: Khóa bí mật (Private Key) - **Tuyệt đối không commit lên Git**.
  * `cosign.pub`: Khóa công khai (Public Key) - Dùng để cấu hình trên cụm K8s.

---

### 🟩 BƯỚC 2: CẤU HÌNH GITHUB SECRETS (TRÊN REPO GITHUB)

Để GitHub Actions có thể tự động ký tên vào Image sau khi build, bạn cần đưa khóa bí mật vào GitHub Secrets:

1. Vào Repository của bạn trên GitHub ➡️ Chọn **Settings** > **Secrets and variables** > **Actions**.
2. Tạo 2 Secrets mới:
   * **`COSIGN_PRIVATE_KEY`**: Mở file `cosign.key` vừa tạo ở Bước 1, copy toàn bộ nội dung (bao gồm cả dòng đầu `-----BEGIN COSIGN PRIVATE KEY-----` và dòng cuối) và dán vào đây.
   * **`COSIGN_PASSWORD`**: Điền mật khẩu bảo vệ bạn đã tạo ở Bước 1 (`MatKhauBaoVeKhoaCosign123`).

---

### 🟩 BƯỚC 3: CẤU HÌNH PIPELINE GITHUB ACTIONS (`build-push.yml`)

Mở file `.github/workflows/build-push.yml` và cấu hình các bước quét Trivy và ký Cosign ngay sau khi build Image thành công. 

#### Nội dung cấu hình cập nhật (phần các step build-push):
```yaml
      - name: Build and push Docker image
        uses: docker/build-push-action@v6
        with:
          context: ./src/api
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}

      # 1. Quét lỗ hổng bảo mật bằng Trivy
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@0.28.0
        with:
          image-ref: '${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.semver.outputs.version }}'
          format: 'table'
          exit-code: '1' # Chặn pipeline và báo lỗi nếu phát hiện CVE High/Critical
          ignore-unfixed: true
          vuln-type: 'os,library'
          severity: 'CRITICAL,HIGH'
        env:
          TRIVY_USERNAME: ${{ github.actor }}
          TRIVY_PASSWORD: ${{ secrets.GITHUB_TOKEN }} # Dùng để xác thực đọc image private trên GHCR

      # 2. Cài đặt môi trường Cosign
      - name: Install Cosign
        uses: sigstore/cosign-installer@v3.5.0

      # 3. Ký tên lên Docker Image
      - name: Sign the published Docker image
        env:
          COSIGN_PRIVATE_KEY: ${{ secrets.COSIGN_PRIVATE_KEY }}
          COSIGN_PASSWORD: ${{ secrets.COSIGN_PASSWORD }}
        run: |
          cosign sign --yes --key env://COSIGN_PRIVATE_KEY "${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.semver.outputs.version }}"

      - name: Update rollout.yaml with new version
        run: |
          sed -i "s|image: ghcr.io/.*/w10-api:.*|image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.semver.outputs.version }}|g" app-api/rollout.yaml
          sed -i "s|value: \"v.*\"|value: \"v${{ steps.semver.outputs.version }}\"|g" app-api/rollout.yaml
```
> **Tại sao bước Trivy lại đặt trước bước Cosign?**
> Nếu đặt Trivy sau, ta sẽ mất công ký lên một image không an toàn. Đặt Trivy trước đảm bảo chỉ những image thực sự sạch CVE mới được ký số và cấp phép chạy trên hệ thống.

---

### 🟩 BƯỚC 4: TRIỂN KHAI CHÍNH SÁCH BẢO MẬT TRÊN KUBERNETES GITOPS

#### 1. Tạo file cấu hình chính sách xác thực (`policies/cluster-image-policy.yaml`):
Tạo thư mục `policies/` trong mã nguồn và tạo tệp tin `cluster-image-policy.yaml`. Dán nội dung của file public key `cosign.pub` vào phần `key.data`:

```yaml
apiVersion: policy.sigstore.dev/v1alpha1
kind: ClusterImagePolicy
metadata:
  name: image-signature-policy
spec:
  images:
    - glob: "ghcr.io/hung0codon/*" # Chỉ quét và xác thực đối với các image của tài khoản của bạn
  authorities:
    - key:
        data: |
          -----BEGIN PUBLIC KEY-----
          <DÁN TOÀN BỘ NỘI DUNG TRONG FILE COSIGN.PUB CỦA BẠN VÀO ĐÂY>
          -----END PUBLIC KEY-----
```
* **Tại sao cần bước này?** File này hướng dẫn Sigstore Admission Webhook biết được rằng: Khi thấy bất kỳ image nào thuộc namespace registry `ghcr.io/hung0codon/`, hãy dùng Public Key này để kiểm chứng chữ ký.

#### 2. Cài đặt Sigstore Policy Controller thông qua ArgoCD (`argocd/apps/policy-controller.yaml`):
Tạo file để ArgoCD tự động triển khai bộ khung Sigstore:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sigstore-policy-controller
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "0" # Cài đặt hạ tầng Sigstore trước
spec:
  project: default
  source:
    repoURL: 'https://sigstore.github.io/helm-charts'
    chart: policy-controller
    targetRevision: 0.9.0
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: cosign-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

#### 3. Đồng bộ hóa thư mục chính sách (`argocd/apps/policies.yaml`):
Tạo file Application để sync thư mục `policies` chứa luật xác thực:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-img-policies
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "1" # Áp dụng luật sau khi controller đã sẵn sàng
spec:
  project: default
  source:
    repoURL: 'https://github.com/Hung0codon/Project_Xbrain_W10.git' # Thay đổi đúng URL repo của bạn
    targetRevision: HEAD
    path: policies
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: cosign-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

### 🟩 BƯỚC 5: KÍCH HOẠT LỚP XÁC THỰC (TRÁNH BẪY CRASH APP)

#### ⚠️ CẢNH BÁO BẪY THIẾT KẾ:
Sigstore mặc định sẽ **không quét toàn bộ các namespace** trong K8s để tránh làm treo các Pod hệ thống (tương tự lỗi OPA Gatekeeper trước đó). Nó chỉ quét các namespace có nhãn kích hoạt: `policy.sigstore.dev/include=true`.
* **Thứ tự thực hiện đúng**:
  1. Trigger GitHub Actions build và ký thành công ít nhất 1 phiên bản image mới.
  2. Đồng bộ các ứng dụng ArgoCD (`policy-controller` và `policies`).
  3. Khi image đã được ký an toàn trên Registry, tiến hành gán nhãn kích hoạt namespace `demo` bằng lệnh:
     ```bash
     kubectl label namespace demo policy.sigstore.dev/include=true
     ```
  *(Nếu gán nhãn trước khi image có chữ ký, Webhook sẽ chặn ngay lập tức Pod Flask API hiện tại và làm sập ứng dụng).*

---

## 🧪 3. HƯỚNG DẪN KIỂM THỬ XÁC THỰC (VERIFICATION)

Sau khi hoàn thành, bạn có thể thực hiện 2 bài test sau để nghiệm thu:

### 🔬 Test 1: Kiểm tra Pod hợp lệ (Có chữ ký) khởi chạy thành công
1. Đẩy một commit thay đổi code trong thư mục `src/api/`.
2. Pipeline GitHub Actions chạy: Build ➡️ Quét Trivy ➡️ Ký Cosign thành công ➡️ Bump tag `rollout.yaml`.
3. ArgoCD tự động kéo tag mới về deploy.
4. Chạy lệnh: `kubectl get pods -n demo`.
* **Kết quả mong đợi**: Pod mới khởi chạy ở trạng thái `Running` bình thường do chữ ký trùng khớp với Public Key cấu hình trên cụm.

### 🔬 Test 2: Thử nghiệm deploy Image không có chữ ký (Bị Reject)
1. Thử deploy một pod sử dụng image nginx chưa được ký bởi khóa của bạn vào namespace `demo`:
   ```bash
   kubectl run test-unsigned-pod --image=nginx:alpine -n demo
   ```
* **Kết quả mong đợi**: Kubernetes API Server trả về lỗi từ chối ngay lập tức:
   ```bash
   Error from server (BadRequest): admission webhook "policy.sigstore.dev" denied the request: validation failed: image nginx:alpine does not have a valid signature
   ```
   ➡️ Lớp bảo mật hoạt động hoàn hảo, bảo vệ cụm khỏi các image lạ/không an toàn!
