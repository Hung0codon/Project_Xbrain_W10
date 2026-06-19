# Architecture Decision Record (ADR): Ngoại lệ quét lỗ hổng bảo mật Trivy đối với CVE chưa có bản vá (Unfixed CVEs)

## 1. Trạng thái (Status)
* **Trạng thái**: Đã phê duyệt (Approved)
* **Ngày quyết định**: 2026-06-19
* **Người phê duyệt**: DevSecOps Team / Hung0codon

## 2. Bối cảnh (Context)
Trong quy trình CI/CD bảo mật của dự án W10, công cụ Trivy được sử dụng để quét container image trước khi ký số bằng Cosign. 
Nếu phát hiện bất kỳ lỗ hổng bảo mật nào ở mức độ `HIGH` hoặc `CRITICAL`, Trivy sẽ dừng pipeline và báo lỗi (`exit-code: 1`).
Tuy nhiên, có nhiều lỗ hổng bảo mật do upstream (nhà cung cấp hệ điều hành hoặc thư viện nền tảng như Alpine, Python...) chưa phát hành bản vá chính thức (Unfixed CVEs).
Nếu chặn tuyệt đối tất cả các lỗ hổng này, pipeline sẽ bị kẹt vĩnh viễn (block mãi), ngăn cản đội ngũ phát triển đẩy các bản vá nghiệp vụ (business hotfix) quan trọng lên hệ thống.

## 3. Quyết định (Decision)
Chúng tôi quyết định áp dụng chính sách **Ngoại lệ tạm thời (Exception ADR)** đối với các CVE chưa có bản vá từ nhà cung cấp:
1. Cấu hình tham số `ignore-unfixed: true` trong bước quét Trivy tại `.github/workflows/build-push.yml` để bỏ qua các lỗ hổng chưa có bản vá chính thức.
2. Thiết lập thời hạn áp dụng ngoại lệ này tối đa là **90 ngày** kể từ ngày phát hiện lỗi.
3. Trong thời gian này, đội ngũ bảo mật và vận hành phải theo dõi các bản vá từ upstream để nâng cấp Base Image ngay khi bản vá được phát hành.
4. Nếu quá thời hạn 90 ngày mà chưa có bản vá chính thức, nhóm phát triển phải xem xét thay thế thư viện dính lỗi bằng giải pháp thay thế an toàn hơn (ví dụ: chuyển đổi base image hoặc thư viện tương đương).

## 4. Hệ quả (Consequences)
* **Tích cực**: Pipeline CI/CD không bị nghẽn vô thời hạn bởi các lỗi nằm ngoài tầm kiểm soát của dự án. Đội phát triển có thể liên tục phân phối sản phẩm.
* **Tiêu cực**: Hệ thống có khả năng chạy các container chứa một số CVE đã biết nhưng chưa có bản vá. Rủi ro này được giảm thiểu bằng cách giám sát mạng chặt chẽ và giới hạn phạm vi truy cập mạng của các pod thông qua chính sách bảo mật NetworkPolicy trên Kubernetes.
