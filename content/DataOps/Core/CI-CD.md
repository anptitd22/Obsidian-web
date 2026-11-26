**CI/CD** là viết tắt của **Continuous Integration** (Tích hợp liên tục) và **Continuous Delivery / Continuous Deployment** (Chuyển giao liên tục / Triển khai liên tục). Đây là một tập hợp các nguyên tắc và quy trình tự động hóa trong phát triển phần mềm (và cả Data Engineering/MLOps), giúp đội ngũ kỹ thuật đưa code từ máy cá nhân lên môi trường thực tế (Production) một cách nhanh chóng, an toàn và ít lỗi thủ công nhất.

![[Pasted image 20261116002310.png]]

## 1. Giải mã từng thành phần

### CI — Continuous Integration (Tích hợp liên tục)

Là thói quen lập trình viên thường xuyên đẩy (push/merge) code mới lên kho lưu trữ chung (như GitHub, GitLab) nhiều lần trong ngày. Mỗi lần có code mới, hệ thống tự động kích hoạt một chuỗi kiểm tra:

- **Build:** Biên dịch mã nguồn hoặc đóng gói ứng dụng (ví dụ: build Docker image).
    
- **Lint & Format:** Kiểm tra quy chuẩn viết code (ví dụ: chạy `ruff` hoặc `flake8` cho Python).
    
- **Automated Test:** Chạy các bài kiểm thử tự động (Unit test, Integration test) để đảm bảo code mới không làm hỏng các tính năng cũ.
    

### CD — Continuous Delivery & Continuous Deployment

CD nối tiếp ngay sau khi giai đoạn CI vượt qua toàn bộ bài kiểm tra:

|**Khái niệm**|**Cơ chế hoạt động**|**Mức độ tự động**|
|---|---|---|
|**Continuous Delivery** (Chuyển giao liên tục)|Code sau khi test xong được đóng gói và đẩy lên môi trường thử nghiệm (Staging). Tuy nhiên, việc đưa lên Production **cần một nút bấm phê duyệt thủ công** từ con người (Tech Lead/PM).|Bán tự động (Manual approval ở bước cuối)|
|**Continuous Deployment** (Triển khai liên tục)|Mọi thay đổi vượt qua tất cả các vòng kiểm thử tự động sẽ được **đẩy thẳng lên Production** mà không cần bất kỳ sự can thiệp thủ công nào.|Tự động hóa 100%|

## 2. Một CI/CD Pipeline thực tế diễn ra như thế nào?

Giả sử bạn vừa sửa xong một tính năng hoặc một pipeline xử lý dữ liệu và đẩy code lên nhánh `main`:

1. **Commit & Push:** Bạn đẩy code lên GitLab/GitHub.
    
2. **Trigger (Kích hoạt):** Hệ thống CI/CD nhận diện sự thay đổi thông qua Webhook.
    
3. **Test & Quality Check:** Hệ thống tự động tạo môi trường ảo, cài đặt thư viện, chạy các bài test logic và quét lỗ hổng bảo mật. Nếu lỗi ở bước này, quy trình dừng lại và báo ngay cho bạn.
    
4. **Build & Push Artifact:** Nếu test thành công, hệ thống đóng gói ứng dụng thành các sản phẩm hoàn chỉnh (như file thực thi, gói thư viện hoặc đẩy Docker image lên Docker Hub / Harbor).
    
5. **Deploy:** Hệ thống cập nhật phiên bản mới lên server hoặc cụm Kubernetes (thông qua các công cụ như ArgoCD, Helm hoặc SSH script) mà không làm gián đoạn dịch vụ của người dùng (Zero-downtime deployment).
    

## 3. Lợi ích cốt lõi và Các công cụ phổ biến

- **Phát hiện lỗi sớm:** Thay vì dồn code cả tháng rồi mới ghép lại và đối mặt với hàng trăm xung đột (merge conflict), lỗi được phát hiện ngay khi vừa viết xong vài chục dòng code.
    
- **Tiết kiệm thời gian vận hành:** Loại bỏ hoàn toàn việc thao tác tay (copy file, chạy lệnh thủ công trên server) vốn rất dễ gây nhầm lẫn.
    
- **Tăng tốc độ ra mắt sản phẩm:** Cho phép cập nhật tính năng mới hoặc vá lỗi khẩn cấp cho người dùng chỉ trong vài phút.
    

**Các công cụ phổ biến hiện nay:**

- **Tích hợp sẵn nền tảng Git:** GitHub Actions, GitLab CI/CD, Bitbucket Pipelines.
    
- **Máy chủ CI/CD độc lập:** Jenkins, CircleCI, Travis CI.
    
- **Chuyên biệt cho Cloud & Kubernetes (GitOps):** ArgoCD, FluxCD, AWS CodePipeline, Azure DevOps.