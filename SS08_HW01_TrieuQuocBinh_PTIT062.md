# BÀI 1: Thực hành Tích hợp & Sinh Code - Import giao dịch JSON

## 1. Phân tích các trường hợp biên (Edge Cases) khi đọc chuỗi JSON từ hệ thống cũ

Khi tích hợp với hệ thống cũ (Legacy System) qua chuỗi JSON thô, dữ liệu thường không đồng nhất và dễ phát sinh lỗi. Dưới đây là các trường hợp biên cần kiểm soát chặt chẽ:

1. **Chuỗi JSON tổng thể không hợp lệ:**
   - Chuỗi JSON bị `null` hoặc hoàn toàn rỗng (`""`).
   - Lỗi cú pháp JSON (ví dụ: thiếu dấu ngoặc `{}` hoặc `[]`, thiếu dấu phẩy `,`, hoặc sai định dạng chuỗi).
   - JSON không phải là một mảng (JSON Array) mà chỉ là một đối tượng đơn lẻ hoặc một giá trị nguyên thủy.

2. **Lỗi biên liên quan đến mã giao dịch (`id`):**
   - Trường `id` bị thiếu hoàn toàn trong đối tượng JSON.
   - Giá trị `id` bị `null` hoặc là chuỗi rỗng (`""`), chuỗi chỉ chứa khoảng trắng (`"   "`).

3. **Lỗi biên liên quan đến số tiền (`amount`):**
   - Trường `amount` bị thiếu hoặc mang giá trị `null`.
   - Giá trị `amount` không phải là số mà là chuỗi không hợp lệ (ví dụ: `"abc"`, `"12.3a"`).
   - Số tiền mang giá trị âm hoặc bằng không (`amount <= 0`), không phù hợp với nghiệp vụ giao dịch ngân hàng.
   - Số tiền chứa ký tự phân tách đặc biệt (ví dụ: `"1,000.00"` hoặc ký hiệu tiền tệ `"$100"`).

4. **Lỗi biên liên quan đến trạng thái giao dịch (`status`):**
   - Trường `status` bị thiếu hoặc mang giá trị `null`.
   - Trạng thái viết thường hoặc sai chính tả (ví dụ: `"success"`, `"failed"`, `"PENDING"`), trong khi hệ thống chỉ chấp nhận `"SUCCESS"` hoặc `"FAILED"`.

5. **Lỗi biên liên quan đến thời gian giao dịch (`transactionDate`):**
   - Trường `transactionDate` bị thiếu hoặc mang giá trị `null`.
   - Chuỗi ngày tháng sai định dạng ISO-8601 (ví dụ: `"25/10/2023 14:30:00"`, `"2023-13-45"`, hoặc chuỗi không liên quan).

6. **Lỗi cục bộ làm sập tiến trình toàn cục:**
   - Nếu parse trực tiếp cả danh sách bằng cách map trực tiếp vào Object (Data Binding toàn bộ danh sách), chỉ cần một bản ghi bị lỗi kiểu dữ liệu (như `amount = "abc"`), thư viện Jackson sẽ ném ra ngoại lệ và hủy bỏ toàn bộ tiến trình. Cần phải parse từng bản ghi độc lập để bỏ qua dòng lỗi mà không làm ảnh hưởng đến các dòng hợp lệ.

---

## 2. Nội dung Prompt tối ưu thiết kế cho AI

```markdown
Hãy đóng vai trò là một Senior Java Developer.
Nhiệm vụ của bạn là thiết kế và viết một class Java 17 tên là `TransactionParser` để phân tích và chuẩn hóa danh sách giao dịch từ một chuỗi JSON thô (raw JSON string).

[Đầu vào (Input)]
- Một chuỗi JSON thô (raw JSON string) biểu diễn một danh sách (JSON Array) chứa các đối tượng giao dịch thô. Mỗi đối tượng chứa các trường: `id` (String), `amount` (có thể là số hoặc chuỗi), `status` (String), `transactionDate` (String).
- Dữ liệu đầu vào có cài cắm các lỗi biên như: thiếu trường `id`, trường `amount` chứa chuỗi chữ "abc", trường `status` không hợp lệ, ngày tháng sai định dạng, chuỗi JSON rỗng hoặc lỗi cú pháp JSON.
- Ví dụ chuỗi JSON đầu vào:
  """
  [
    {"id": "TXN001", "amount": 150000.0, "status": "SUCCESS", "transactionDate": "2023-10-25T14:30:00"},
    {"id": "", "amount": 50000.0, "status": "FAILED", "transactionDate": "2023-10-25T14:35:00"},
    {"id": "TXN002", "amount": "abc", "status": "SUCCESS", "transactionDate": "2023-10-25T14:40:00"},
    {"id": "TXN003", "amount": -100.0, "status": "SUCCESS", "transactionDate": "2023-10-25T14:45:00"},
    {"id": "TXN004", "amount": 200000.0, "status": "INVALID_STATUS", "transactionDate": "2023-10-25T14:50:00"},
    {"id": "TXN005", "amount": 350000.0, "status": "SUCCESS", "transactionDate": "invalid-date-format"},
    {"id": "TXN006", "amount": 500000.0, "status": "FAILED", "transactionDate": "2023-10-25T15:00:00"}
  ]
  """

[Đầu ra (Output)]
- Trả về đối tượng `List<TransactionDTO>`.
- `TransactionDTO` là một Java record gồm các trường:
  - `String id`
  - `double amount`
  - `String status`
  - `LocalDateTime localDateTime`

[Ràng buộc kỹ thuật & Xử lý logic]
- Sử dụng phiên bản Java 17 và thư viện Jackson `ObjectMapper` của Spring Boot (thư viện `com.fasterxml.jackson.databind.ObjectMapper` và các class liên quan như `JsonNode`) để parse JSON.
- Phải kiểm duyệt dữ liệu biên cho từng giao dịch:
  - Bỏ qua giao dịch và ghi log lỗi (sử dụng slf4j Logger hoặc System.err.println) nếu:
    - ID bị null hoặc rỗng (blank).
    - Amount bị null, rỗng, không phải định dạng số hợp lệ, hoặc có giá trị <= 0.
    - Status không thuộc một trong hai giá trị "SUCCESS" hoặc "FAILED" (phân biệt chữ hoa chữ thường).
    - TransactionDate bị null, rỗng, hoặc không thể parse thành `LocalDateTime` (hệ thống cũ sử dụng định dạng ISO-8601, ví dụ: "2023-10-25T14:30:00").
  - Nếu gặp lỗi cú pháp JSON toàn cục (JsonProcessingException), ghi log lỗi và trả về danh sách rỗng, không làm sập toàn bộ tiến trình.
  - Quá trình duyệt lỗi của một dòng/đối tượng không được làm gián đoạn hay dừng toàn bộ tiến trình đọc của các giao dịch khác.

[Quy trình Dry-run CoT]
Trước khi viết mã nguồn Java, bạn bắt buộc phải thực hiện các bước sau:
1. Giải thích ngắn gọn cách bạn sẽ xử lý và bẫy các lỗi dữ liệu biên cụ thể: dòng thiếu ID, amount là "abc", status không hợp lệ, ngày tháng sai định dạng.
2. Phác thảo thuật toán xử lý dữ liệu và bẫy lỗi bằng mã giả (Pseudocode) hoặc sơ đồ từng bước.
3. Cuối cùng, sinh mã nguồn Java 17 hoàn chỉnh, sạch, tuân thủ Clean Code bao gồm định nghĩa record `TransactionDTO` và class `TransactionParser`.
```

---

## 3. Minh chứng chạy thực tế với AI

Dưới đây là toàn bộ phản hồi bằng văn bản của AI khi nhận được prompt trên:

```markdown
Chào bạn, tôi là Senior Java Developer. Dưới đây là giải pháp chi tiết cho yêu cầu thiết kế class `TransactionParser` xử lý và chuẩn hóa dữ liệu giao dịch từ chuỗi JSON thô.

### 1. Giải thích cách bẫy lỗi dữ liệu biên (Dry-run CoT)

Để tránh trường hợp một giao dịch lỗi làm hỏng toàn bộ quá trình đọc dữ liệu (ví dụ: lỗi parse kiểu dữ liệu của `amount` hay `transactionDate`), chúng ta không nên thực hiện map trực tiếp chuỗi JSON sang `List<RawTransaction>` bằng `readValue`. Thay vào đó, chúng ta sẽ thực hiện theo phương pháp **Tree Model**:
- Sử dụng `objectMapper.readTree(jsonStr)` để đọc chuỗi JSON thành một cấu trúc cây `JsonNode`.
- Duyệt qua từng nút con trong mảng JSON. Bằng cách này, nếu có lỗi xảy ra ở bất kỳ thuộc tính nào của một giao dịch cụ thể, chúng ta có thể dễ dàng bắt ngoại lệ cục bộ của giao dịch đó, ghi log lỗi và tiếp tục xử lý các giao dịch tiếp theo.

**Cách xử lý cụ thể cho từng lỗi biên:**
1. **Thiếu hoặc trống ID:** Kiểm tra nếu thuộc tính `id` không tồn tại, hoặc `id.asText()` trả về chuỗi rỗng sau khi trim. Nếu lỗi, log cảnh báo và `continue`.
2. **Amount là "abc" hoặc <= 0:** Đọc giá trị dưới dạng chuỗi và thử parse sang kiểu `double`. Nếu ném ra `NumberFormatException`, log cảnh báo và bỏ qua. Nếu parse thành công nhưng giá trị `<= 0`, tiếp tục log và bỏ qua.
3. **Status không hợp lệ:** Kiểm tra nếu chuỗi trạng thái khác `"SUCCESS"` và `"FAILED"`. Sử dụng câu lệnh so sánh chính xác để loại bỏ các trạng thái không hợp lệ hoặc viết thường.
4. **Ngày tháng sai định dạng:** Sử dụng `LocalDateTime.parse()` kết hợp với định dạng chuẩn `DateTimeFormatter.ISO_LOCAL_DATE_TIME`. Bọc trong khối `try-catch` để bắt `DateTimeParseException` nhằm phát hiện ngày tháng sai định dạng và bỏ qua một cách an toàn.

---

### 2. Phác thảo thuật toán (Pseudocode)

```text
Hàm parseTransactions(rawJsonString):
    Khởi tạo listResult = new ArrayList()
    Nếu rawJsonString là null hoặc trống:
        Trả về listResult
    
    Thử:
        jsonNode = objectMapper.readTree(rawJsonString)
        Nếu jsonNode không phải là một JSON Array:
            Ghi log lỗi: "Dữ liệu không phải là một mảng JSON"
            Trả về listResult
    Bắt lỗi JsonProcessingException:
        Ghi log lỗi: "Lỗi cú pháp JSON toàn cục"
        Trả về listResult

    Duyệt qua từng node trong jsonNode:
        Thử:
            // 1. Kiểm tra và parse ID
            Nếu node không chứa "id" hoặc node.get("id") là null:
                Ném ngoại lệ: "Thiếu ID giao dịch"
            id = node.get("id").asText().trim()
            Nếu id rỗng:
                Ném ngoại lệ: "ID giao dịch không hợp lệ (rỗng)"

            // 2. Kiểm tra và parse Amount
            Nếu node không chứa "amount" hoặc node.get("amount") là null:
                Ném ngoại lệ: "Thiếu trường amount"
            amountStr = node.get("amount").asText().trim()
            amount = parseDouble(amountStr)
            Nếu amount <= 0:
                Ném ngoại lệ: "Số tiền giao dịch phải > 0"

            // 3. Kiểm tra và parse Status
            Nếu node không chứa "status" hoặc node.get("status") là null:
                Ném ngoại lệ: "Thiếu trường status"
            status = node.get("status").asText().trim()
            Nếu status không phải "SUCCESS" và không phải "FAILED":
                Ném ngoại lệ: "Trạng thái giao dịch không hợp lệ: " + status

            // 4. Kiểm tra và parse Date
            Nếu node không chứa "transactionDate" hoặc node.get("transactionDate") là null:
                Ném ngoại lệ: "Thiếu trường transactionDate"
            dateStr = node.get("transactionDate").asText().trim()
            localDateTime = LocalDateTime.parse(dateStr, ISO_LOCAL_DATE_TIME)

            // 5. Thêm giao dịch hợp lệ vào danh sách đầu ra
            listResult.add(new TransactionDTO(id, amount, status, localDateTime))
            
        Bắt các ngoại lệ (IllegalArgumentException, NumberFormatException, DateTimeParseException) cho từng dòng:
            Ghi log warning: "Bỏ qua dòng giao dịch lỗi: " + node.toString() + " - Nguyên nhân: " + e.getMessage()
            
    Trả về listResult
```

---

### 3. Mã nguồn Java 17 hoàn chỉnh

Dưới đây là mã nguồn Java 17 chuẩn doanh nghiệp, sử dụng thư viện Jackson và ghi log chi tiết.

#### Lớp chứa DTO Record: `TransactionDTO.java`
```java
package com.securebank.dto;

import java.time.LocalDateTime;

/**
 * Định nghĩa Java record đại diện cho Transaction DTO sau khi được parse và chuẩn hóa.
 */
public record TransactionDTO(
    String id,
    double amount,
    String status,
    LocalDateTime localDateTime
) {}
```

#### Lớp Parser: `TransactionParser.java`
```java
package com.securebank.parser;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.securebank.dto.TransactionDTO;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.time.format.DateTimeParseException;
import java.util.ArrayList;
import java.util.List;

public class TransactionParser {

    private static final Logger log = LoggerFactory.getLogger(TransactionParser.class);
    private static final ObjectMapper objectMapper = new ObjectMapper();
    private static final DateTimeFormatter DATE_TIME_FORMATTER = DateTimeFormatter.ISO_LOCAL_DATE_TIME;

    /**
     * Phân tích cú pháp chuỗi JSON thô thành danh sách TransactionDTO hợp lệ.
     * Các bản ghi không hợp lệ sẽ bị bỏ qua và ghi log cảnh báo.
     *
     * @param jsonRaw Chuỗi JSON thô
     * @return Danh sách TransactionDTO đã chuẩn hóa
     */
    public static List<TransactionDTO> parseTransactions(String jsonRaw) {
        List<TransactionDTO> validTransactions = new ArrayList<>();

        if (jsonRaw == null || jsonRaw.isBlank()) {
            log.warn("Dữ liệu JSON đầu vào bị null hoặc rỗng.");
            return validTransactions;
        }

        JsonNode rootNode;
        try {
            rootNode = objectMapper.readTree(jsonRaw);
            if (!rootNode.isArray()) {
                log.error("Dữ liệu đầu vào không phải là một JSON Array.");
                return validTransactions;
            }
        } catch (JsonProcessingException e) {
            log.error("Lỗi cú pháp JSON toàn cục. Không thể phân tích dữ liệu đầu vào. Chi tiết: {}", e.getMessage());
            return validTransactions;
        }

        for (JsonNode node : rootNode) {
            try {
                // 1. Phân tích và kiểm tra 'id'
                if (!node.hasNonNull("id")) {
                    throw new IllegalArgumentException("Trường 'id' bị thiếu hoặc null");
                }
                String id = node.get("id").asText().trim();
                if (id.isEmpty()) {
                    throw new IllegalArgumentException("Trường 'id' không được để trống");
                }

                // 2. Phân tích và kiểm tra 'amount'
                if (!node.hasNonNull("amount")) {
                    throw new IllegalArgumentException("Trường 'amount' bị thiếu hoặc null");
                }
                String amountStr = node.get("amount").asText().trim();
                double amount;
                try {
                    amount = Double.parseDouble(amountStr);
                } catch (NumberFormatException e) {
                    throw new IllegalArgumentException("Trường 'amount' không đúng định dạng số: " + amountStr);
                }
                if (amount <= 0) {
                    throw new IllegalArgumentException("Số tiền giao dịch phải lớn hơn 0: " + amount);
                }

                // 3. Phân tích và kiểm tra 'status'
                if (!node.hasNonNull("status")) {
                    throw new IllegalArgumentException("Trường 'status' bị thiếu hoặc null");
                }
                String status = node.get("status").asText().trim();
                if (!"SUCCESS".equals(status) && !"FAILED".equals(status)) {
                    throw new IllegalArgumentException("Trạng thái 'status' không hợp lệ: " + status);
                }

                // 4. Phân tích và kiểm tra 'transactionDate'
                if (!node.hasNonNull("transactionDate")) {
                    throw new IllegalArgumentException("Trường 'transactionDate' bị thiếu hoặc null");
                }
                String dateStr = node.get("transactionDate").asText().trim();
                LocalDateTime localDateTime;
                try {
                    localDateTime = LocalDateTime.parse(dateStr, DATE_TIME_FORMATTER);
                } catch (DateTimeParseException e) {
                    throw new IllegalArgumentException("Định dạng 'transactionDate' không hợp lệ (yêu cầu ISO-8601): " + dateStr);
                }

                // Nếu tất cả các trường hợp đều hợp lệ, thêm vào danh sách kết quả
                validTransactions.add(new TransactionDTO(id, amount, status, localDateTime));

            } catch (IllegalArgumentException e) {
                // Bắt lỗi kiểm duyệt cho từng dòng giao dịch
                log.warn("Bỏ qua giao dịch không hợp lệ: {} - Lý do: {}", node.toString(), e.getMessage());
            }
        }

        return validTransactions;
    }
}
```
```