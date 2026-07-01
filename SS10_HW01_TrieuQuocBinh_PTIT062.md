# BÀI 1: PHÂN TÍCH & LỰA CHỌN CHIẾN LƯỢC TÀI LIỆU HÓA

### 1. Đáp án phương án được chọn
* **Phương án được chọn:** **Phương án B**
  *(Khởi tạo workspace trong Antigravity, index toàn bộ thư mục Guai-api và sử dụng prompt yêu cầu System Analyst đọc toàn bộ luồng xử lý từ endpoint API `/api/v1/checkout` đi qua các service, filter bảo mật để tổng hợp thành bản đặc tả Use Case theo định dạng SRS).*

---

### 2. Phân tích lý do lựa chọn và loại trừ

#### A. Lý do chọn Phương án B (Tối ưu nhất)
* **Khả năng nhận thức ngữ cảnh toàn cục (Global Context Awareness):**
  * Antigravity hoạt động dựa trên việc tạo chỉ mục (indexing) toàn bộ codebase của dự án. Nhờ vậy, khi nhận được yêu cầu phân tích một API endpoint như `/api/v1/checkout`, nó có khả năng tự động lần theo toàn bộ luồng đi của dữ liệu (control flow và data flow) từ `Controller`, đi ngược qua các `JWT Security Filter`, đến các lớp logic nghiệp vụ `Service`, và kết thúc tại các lớp giao tiếp cơ sở dữ liệu `Repository`.
  * Điều này loại bỏ hoàn toàn việc con người phải tự tìm kiếm xem các file nào liên quan đến luồng nghiệp vụ này, giúp tiết kiệm thời gian tối đa và đảm bảo tính chính xác cao.
* **Hạn chế tối đa hiện tượng thất thoát ngữ cảnh (Context Loss):**
  * Do toàn bộ workspace đã được index và AI phân tích trực tiếp trên cấu trúc dự án thực tế, các mối liên kết (dependencies, import, bean injection) giữa các thành phần Java được bảo toàn nguyên vẹn. AI sẽ không bị hiểu nhầm hoặc bỏ sót các đoạn code kiểm tra điều kiện ẩn (ví dụ: các điều kiện phân quyền trong JWT Security Filter).
* **Định vị vai trò rõ ràng và đầu ra chuẩn hóa (System Analyst Role & SRS Format):**
  * Việc áp dụng kỹ thuật Prompt Role-playing (*"Act as a System Analyst"*) giúp định hướng tư duy của AI. Thay vì giải thích code theo kiểu lập trình viên (kỹ thuật, cú pháp), AI sẽ chuyển dịch ngôn ngữ kỹ thuật của code Java sang ngôn ngữ nghiệp vụ của tài liệu đặc tả SRS (bao gồm đầy đủ: *Pre-conditions, Main flow, và Exceptions*).
* **Quy trình tự động hóa khép kín:**
  * Chỉ với một prompt duy nhất mang tính chỉ thị cấp cao, AI tự động thực hiện các bước: Đọc hiểu -> Phân tích luồng -> Tổng hợp -> Định dạng văn bản, giúp giảm thiểu sự can thiệp thủ công của kỹ sư hệ thống.

#### B. Lý do loại trừ các phương án còn lại (Phân tích rủi ro & lỗ hổng)

* **Loại trừ Phương án A (Dùng VSCode + Copilot Chat cục bộ từng file):**
  * **Thất thoát ngữ cảnh nghiêm trọng (Context Loss):** Copilot Chat truyền thống hoạt động chủ yếu dựa trên ngữ cảnh cục bộ của file đang mở hoặc đoạn code được bôi đen. Khi bôi đen từng file riêng biệt, AI không thể nhìn thấy bức tranh toàn cảnh (mối liên kết giữa Controller, Service, Repository và Security Filter). AI sẽ giải thích từng đoạn code rời rạc, làm mất đi tính liên tục của luồng nghiệp vụ.
  * **Tốn chi phí nhân lực tổng hợp:** Kỹ sư hệ thống (System Analyst) phải mất rất nhiều công sức để đọc từng phần giải thích nhỏ từ Copilot Chat, sau đó tự tay chắp vá, lắp ghép và cấu trúc lại thành tài liệu SRS. Quá trình thủ công này rất dễ dẫn đến sai sót logic hoặc bỏ sót các trường hợp ngoại lệ (Exceptions) xảy ra ở ranh giới giữa các file.
  * **Mất thời gian định vị thủ công:** Người dùng phải tự mò mẫm tìm kiếm tất cả các file liên quan đến Checkout flow trong một codebase lớn. Nếu codebase có hàng trăm file, việc bỏ sót 1-2 file service hoặc filter là rất dễ xảy ra.

* **Loại trừ Phương án C (Copy thủ công 5 file dán vào giao diện Web ChatGPT/Gemini):**
  * **Rủi ro vượt giới hạn Token (Context Window):** Mặc dù các mô hình web hiện nay có context window lớn, việc copy/paste thủ công code của nhiều file lớn dễ làm loãng ngữ cảnh. AI trên web có thể bị "quá tải" thông tin nhiễu từ các thư viện import hoặc code bổ trợ không liên quan, dẫn đến phản hồi chung chung hoặc bị "ảo tưởng" (hallucination).
  * **Rủi ro rò rỉ bảo mật dữ liệu (Data Privacy & Security Leak):** Đây là lỗ hổng nghiêm trọng nhất. Việc copy mã nguồn của dự án (đặc biệt là các logic thanh toán và bảo mật JWT Security Filter) dán lên các công cụ AI công cộng trên web có thể vi phạm điều khoản bảo mật của doanh nghiệp (IP Leakage). Các dữ liệu này có thể được sử dụng để huấn luyện mô hình tiếp theo, làm lộ bí mật kinh doanh hoặc lỗ hổng bảo mật của hệ thống.
  * **Đứt gãy liên kết cấu trúc dự án:** Khi copy code dán vào ô chat, cấu trúc thư mục, mối quan hệ package và các cấu hình Spring Boot (như `@Autowired`, `@Component`) bị biến thành văn bản phẳng (flat text). AI web sẽ mất đi khả năng định vị vật lý của các file trong codebase thực tế, gây khó khăn cho việc đối chiếu và cập nhật sau này.