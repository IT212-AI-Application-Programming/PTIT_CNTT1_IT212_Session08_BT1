# BÀI 1: Thực hành Tích hợp & Sinh Code - Import giao dịch JSON

## 1. Phân tích các trường hợp biên (Edge Cases) khi đọc chuỗi JSON từ hệ thống cũ

Khi tích hợp với một hệ thống Core Banking cũ bằng tệp JSON thô, dữ liệu trả về thường "bẩn" và thiếu tính đồng nhất. Dưới đây là phân tích các luồng dị thường (Edge Cases) chắc chắn sẽ xảy ra và có thể làm sập tiến trình đọc nếu không xử lý tốt:

1.  **Dữ liệu thiếu trường (Missing Fields):** Đối tượng JSON thiếu hẳn node `id`, hoặc node `amount` không tồn tại.
2.  **Lỗi sai kiểu dữ liệu (Type Mismatch):** Trường `amount` thay vì là số (number) lại bị hệ thống cũ gửi dưới dạng chuỗi chứa chữ cái (ví dụ: `"abc"` hoặc chứa ký tự tiền tệ `"1,000$"`), gây ra `NumberFormatException`.
3.  **Lỗi vi phạm miền giá trị nghiệp vụ (Domain Constraint Violation):** Số tiền (`amount`) bị âm hoặc bằng 0 do lỗi đối soát, trạng thái (`status`) gửi về các mã rác như `"PENDING_1"`, `"CANCEL"` thay vì `"SUCCESS"` hay `"FAILED"`.
4.  **Sai định dạng thời gian (Date-time Parsing Error):** Trường `transactionDate` không tuân theo chuẩn ISO-8601 (có thể bị gửi dạng timestamp hoặc định dạng custom), gây ra `DateTimeParseException`.
5.  **Cấu trúc JSON bị vỡ (Malformed JSON):** Bản thân chuỗi JSON thô bị mất ngoặc `[`, `{` ở mức root, khiến toàn bộ tiến trình parse văng `JsonParseException`.

---

## 2. Thiết kế Prompt tối ưu (Gửi cho AI)

Dưới đây là Prompt sử dụng cấu trúc 4 phần kết hợp yêu cầu Dry-run CoT để ép AI tạo ra đoạn mã xử lý tốt nhất:

> **[Vai Trò]**
> Hãy đóng vai trò là một Senior Java Developer chuyên về tích hợp hệ thống.
> 
> **[Nhiệm Vụ]**
> Hãy viết lớp `TransactionParser` bằng Java 17 để phân tích và chuẩn hóa danh sách giao dịch từ hệ thống Core Banking cũ chuyển sang.
> 
> **[Đầu Vào & Đầu Ra]**
> - **Đầu vào (Input):** Một chuỗi JSON thô (Raw JSON string) đại diện cho một mảng (Array) các đối tượng giao dịch. Mỗi đối tượng chứa các trường: `id` (String), `amount` (dạng số hoặc chuỗi), `status` (String), `transactionDate` (chuỗi ISO-8601). Dữ liệu này chứa nhiều phần tử bị lỗi biên như: thiếu `id`, `amount` chứa chữ cái, sai định dạng ngày.
> - **Đầu ra (Output):** Phương thức trả về `List<TransactionDTO>`. Trong đó, định nghĩa `TransactionDTO` là một Java 17 record gồm: `String id, double amount, String status, LocalDateTime transactionDate`.
> 
> **[Ràng Buộc Kỹ Thuật]**
> 1. Chỉ sử dụng thư viện `Jackson ObjectMapper` của Spring Boot. Gợi ý: Hãy parse JSON thành dạng `JsonNode` (Cây cấu trúc) để duyệt và bắt lỗi từng phần tử thay vì tự động mapping thẳng ra Object (vì một object map lỗi sẽ làm sập nguyên list).
> 2. Kiểm duyệt dữ liệu:
>    - `amount` > 0.
>    - `status` CHỈ cho phép `"SUCCESS"` hoặc `"FAILED"`.
>    - `id` không được null hay rỗng.
> 3. Tuyệt đối không để ứng dụng bị crash khi gặp lỗi. Nếu một phần tử JSON không hợp lệ, phải bẫy lỗi (bắt các ngoại lệ như `NumberFormatException`, `DateTimeParseException`), ghi log cảnh báo bằng `@Slf4j` và **bỏ qua dòng đó, tiếp tục đọc dòng tiếp theo**.
> 
> **[Quy Trình Dry-run CoT]**
> Đừng viết code ngay. Hãy trình bày theo các bước sau ra màn hình:
> 1. Trình bày luồng suy luận (Chain-of-Thought) và phác thảo thuật toán bằng mã giả (Pseudocode) giải thích cách bạn xử lý JSON từng phần tử một cách an toàn.
> 2. Sinh ra mã nguồn Java hoàn chỉnh.

---

## 3. Minh chứng chạy thực tế (Log phản hồi từ AI)

*(Dưới đây là nội dung AI sinh ra khi chạy Prompt trên)*

**[AI RESPONSE BEGIN]**

Chào bạn, với tư cách là Senior Java Developer, tôi hoàn toàn hiểu vấn đề đau đầu khi phải xử lý dữ liệu bẩn từ hệ thống cũ. Việc dùng `ObjectMapper.readValue()` trực tiếp map thành `List<TransactionDTO>` là "tự sát" vì chỉ cần 1 node sai kiểu (như amount là chữ), Jackson sẽ quăng exception và làm mất toàn bộ 99 giao dịch hợp lệ còn lại. 

Để giải quyết, tôi sẽ dùng `JsonNode` để duyệt cây JSON và validate thủ công từng đối tượng một.

### 1. Phác thảo thuật toán (Luồng suy luận CoT)

```text
1. Khởi tạo danh sách kết quả `validTransactions` rỗng.
2. Đọc chuỗi raw JSON chuyển thành cây `JsonNode` (rootNode). Nếu rootNode không phải là Array, báo lỗi và dừng.
3. Vòng lặp: Duyệt qua từng `JsonNode node` bên trong `rootNode`.
4. Mở khối `try-catch` BÊN TRONG vòng lặp để cách ly lỗi của từng node:
   a. Lấy `id`. Nếu null hoặc rỗng -> ném IllegalArgumentException.
   b. Lấy `amount` dưới dạng String, sau đó dùng Double.parseDouble(). Nếu không phải số -> ném NumberFormatException. Kiểm tra nếu <= 0 -> ném IllegalArgumentException.
   c. Lấy `status`. Nếu không bằng "SUCCESS" và "FAILED" -> ném IllegalArgumentException.
   d. Lấy `transactionDate` dạng String, dùng LocalDateTime.parse(). Nếu định dạng sai -> ném DateTimeParseException.
   e. Nếu qua hết các bước kiểm tra, khởi tạo TransactionDTO record và thêm vào danh sách kết quả.
5. Nếu bất kỳ Exception nào bị ném ra ở bước 4, khối catch tương ứng sẽ bắt lại, dùng log.error ghi lại lỗi và bỏ qua (continue) sang node tiếp theo.
6. Trả về `validTransactions`.
```

### 2. Mã nguồn Java hoàn chỉnh

```java
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.extern.slf4j.Slf4j;

import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.time.format.DateTimeParseException;
import java.util.ArrayList;
import java.util.List;

// Định nghĩa Java 17 Record cho đầu ra
record TransactionDTO(String id, double amount, String status, LocalDateTime transactionDate) {}

@Slf4j
public class TransactionParser {

    private final ObjectMapper objectMapper;
    private static final DateTimeFormatter DATE_FORMATTER = DateTimeFormatter.ISO_LOCAL_DATE_TIME;

    // Dependency Injection thông qua Constructor
    public TransactionParser(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    /**
     * Phân tích và lọc dữ liệu JSON thô chứa danh sách giao dịch
     */
    public List<TransactionDTO> parseTransactions(String rawJson) {
        List<TransactionDTO> validTransactions = new ArrayList<>();
        
        if (rawJson == null || rawJson.isBlank()) {
            log.warn("Input JSON string is null or empty");
            return validTransactions;
        }

        try {
            // Đọc JSON dưới dạng Tree để kiểm soát hoàn toàn từng phần tử
            JsonNode rootNode = objectMapper.readTree(rawJson);
            
            if (!rootNode.isArray()) {
                log.error("Invalid JSON structure: Expected a JSON array at root.");
                return validTransactions;
            }

            for (JsonNode node : rootNode) {
                // Khối try-catch bao bọc TỪNG phần tử để cách ly lỗi
                try {
                    // 1. Validate ID
                    String id = node.path("id").asText(null);
                    if (id == null || id.isBlank()) {
                        throw new IllegalArgumentException("Transaction ID is missing or empty.");
                    }

                    // 2. Validate Amount
                    String amountStr = node.path("amount").asText(null);
                    if (amountStr == null) {
                        throw new IllegalArgumentException("Amount is missing.");
                    }
                    double amount = Double.parseDouble(amountStr);
                    if (amount <= 0) {
                        throw new IllegalArgumentException("Amount must be strictly greater than 0.");
                    }

                    // 3. Validate Status
                    String status = node.path("status").asText(null);
                    if (!"SUCCESS".equals(status) && !"FAILED".equals(status)) {
                        throw new IllegalArgumentException("Invalid status: " + status + ". Allowed only SUCCESS/FAILED.");
                    }

                    // 4. Validate Date
                    String dateStr = node.path("transactionDate").asText(null);
                    if (dateStr == null) {
                        throw new IllegalArgumentException("Transaction date is missing.");
                    }
                    LocalDateTime transactionDate = LocalDateTime.parse(dateStr, DATE_FORMATTER);

                    // Map thành công, đưa vào danh sách
                    validTransactions.add(new TransactionDTO(id, amount, status, transactionDate));

                } catch (NumberFormatException e) {
                    log.error("Data type error (Amount) in node {}: {}", node.toString(), e.getMessage());
                } catch (DateTimeParseException e) {
                    log.error("Date format error in node {}: {}", node.toString(), e.getMessage());
                } catch (IllegalArgumentException e) {
                    log.error("Validation failed in node {}: {}", node.toString(), e.getMessage());
                } catch (Exception e) {
                    log.error("Unexpected error parsing node {}: {}", node.toString(), e.getMessage(), e);
                }
            }
        } catch (JsonProcessingException e) {
            log.error("Fatal error: Failed to parse raw JSON string as tree. JSON is severely malformed.", e);
        }

        log.info("Successfully parsed {} valid transactions out of total elements.", validTransactions.size());
        return validTransactions;
    }
}
```
