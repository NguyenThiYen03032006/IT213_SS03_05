# Dò lỗi & Đóng gói — Cơ chế tự sửa lỗi (Error Feedback Loop)

## 1. Mục tiêu

Khi sử dụng `BeanOutputConverter.convert()`, LLM có thể trả về JSON không hợp lệ, ví dụ:

```json
{
  "fullName": "Nguyễn Văn An",
  "phone": "0901234567",
  "email": "an@gmail.com",
  "skills": ["Java", "Spring Boot"],
  "yearsExperience": 3
```

JSON trên bị thiếu dấu `}` đóng.

Khi gọi:

```java
converter.convert(llmResponse);
```

Jackson có thể phát sinh lỗi parse JSON.

Nếu không xử lý:

```text
LLM
 │
 ▼
JSON không hợp lệ
 │
 ▼
BeanOutputConverter.convert()
 │
 ▼
JsonProcessingException
 │
 ▼
ETL FAILED / HTTP 500
```

Giải pháp là xây dựng cơ chế **Self-Healing / Error Feedback Loop**:

```text
                    ┌─────────────────┐
                    │    Raw Text     │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │    ChatModel    │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │  LLM Response   │
                    └────────┬────────┘
                             │
                             ▼
                  ┌──────────────────────┐
                  │ BeanOutputConverter  │
                  │      convert()       │
                  └──────────┬───────────┘
                             │
                   ┌─────────┴─────────┐
                   │                   │
                 SUCCESS              ERROR
                   │                   │
                   ▼                   ▼
               Return Result      getMessage()
                                       │
                                       ▼
                                Error Feedback
                                       │
                                       ▼
                                  Call LLM
                                       │
                                       ▼
                                     Retry
                                       │
                              ┌────────┴────────┐
                              │                 │
                           SUCCESS          maxRetries
                              │                 │
                              ▼                 ▼
                         Return Result       Fallback
```

---

# 2. CandidateExtraction Record

Service sử dụng một Java Record làm kiểu dữ liệu đầu ra:

```java
package com.rikkei.academy.hr.etl;

import java.util.List;

public record CandidateExtraction(
        String fullName,
        String phone,
        String email,
        List<String> skills,
        int yearsExperience
) {
}
```

---

# 3. SelfHealingExtractionService

## File: `SelfHealingExtractionService.java`

```java
package com.rikkei.academy.hr.service;

import com.rikkei.academy.hr.etl.CandidateExtraction;
import org.springframework.ai.chat.model.ChatModel;
import org.springframework.ai.converter.BeanOutputConverter;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class SelfHealingExtractionService {

    private final ChatModel chatModel;
    private final BeanOutputConverter<CandidateExtraction> converter;

    public SelfHealingExtractionService(ChatModel chatModel) {
        this.chatModel = chatModel;
        this.converter =
                new BeanOutputConverter<>(CandidateExtraction.class);
    }

    /**
     * Extract candidate information from raw text.
     *
     * Self-Healing Flow:
     *
     * 1. Build prompt.
     * 2. Call LLM.
     * 3. Parse response using BeanOutputConverter.
     * 4. If parsing fails, capture exception.getMessage().
     * 5. Send the error back to the LLM.
     * 6. Retry until maxRetries is reached.
     * 7. Return fallback object if all attempts fail.
     */
    public CandidateExtraction extractWithRetry(
            String rawText,
            int maxRetries
    ) {

        // Input validation
        if (rawText == null || rawText.isBlank()) {
            return defaultExtraction();
        }

        if (maxRetries < 0) {
            maxRetries = 0;
        }

        String errorFeedback = null;

        /*
         * maxRetries represents the number of RETRIES
         * after the initial attempt.
         *
         * Example:
         * maxRetries = 2
         *
         * Attempt 1 -> Initial call
         * Attempt 2 -> Retry 1
         * Attempt 3 -> Retry 2
         */
        for (int attempt = 0;
             attempt <= maxRetries;
             attempt++) {

            try {

                // ----------------------------------------
                // 1. Build prompt
                // ----------------------------------------

                String prompt = buildPrompt(
                        rawText,
                        errorFeedback,
                        attempt
                );

                // ----------------------------------------
                // 2. Call LLM
                // ----------------------------------------

                String llmResponse =
                        chatModel.call(prompt);

                // ----------------------------------------
                // 3. Parse JSON
                // ----------------------------------------

                CandidateExtraction result =
                        converter.convert(llmResponse);

                // ----------------------------------------
                // 4. Parse success
                // ----------------------------------------

                if (result != null) {
                    return result;
                }

                /*
                 * Converter returned null.
                 * Treat this as an extraction failure
                 * and retry.
                 */
                errorFeedback =
                        "BeanOutputConverter returned null.";

            } catch (Exception ex) {

                /*
                 * Important:
                 * Capture Jackson/parser error and
                 * send it back to the LLM.
                 */
                errorFeedback = ex.getMessage();

                if (errorFeedback == null
                        || errorFeedback.isBlank()) {

                    errorFeedback =
                            "Unknown JSON parsing error.";
                }

                System.err.println(
                        "Self-healing attempt "
                                + (attempt + 1)
                                + " failed."
                );

                System.err.println(
                        "Parser error: "
                                + errorFeedback
                );
            }
        }

        // ----------------------------------------
        // 5. All attempts failed
        // ----------------------------------------

        return defaultExtraction();
    }

    /**
     * Builds the initial prompt or retry prompt.
     */
    private String buildPrompt(
            String rawText,
            String errorFeedback,
            int attempt
    ) {

        String basePrompt = """
                You are an AI resume extraction system.

                Your task is to extract candidate information
                from the resume below.

                STRICT RULES:

                1. Return ONLY valid JSON.
                2. Do NOT use Markdown code fences.
                3. Do NOT add explanations.
                4. Do NOT add fields outside the required structure.
                5. Make sure every { has a matching }.
                6. Make sure every [ has a matching ].
                7. Use valid JSON syntax.
                8. yearsExperience must be a number.
                9. Do not invent information.

                Required JSON structure:

                {
                  "fullName": "string",
                  "phone": "string",
                  "email": "string",
                  "skills": ["string"],
                  "yearsExperience": 0
                }

                Resume:

                %s
                """.formatted(rawText);

        /*
         * First attempt:
         * No previous error exists.
         */
        if (errorFeedback == null) {
            return basePrompt;
        }

        /*
         * Retry:
         * Add parser/Jackson error to prompt.
         */
        return basePrompt + """

                ==============================
                ERROR FEEDBACK
                ==============================

                Your previous response could not be parsed.

                Retry attempt:
                %d

                Parser/Jackson error:

                %s

                Analyze the error above and correct
                your previous JSON response.

                IMPORTANT:
                Return ONLY the corrected JSON.
                Do NOT explain the correction.
                Do NOT use Markdown.
                """.formatted(
                attempt + 1,
                errorFeedback
        );
    }

    /**
     * Default fallback record.
     */
    private CandidateExtraction defaultExtraction() {

        return new CandidateExtraction(
                "UNKNOWN",
                "",
                "",
                List.of(),
                0
        );
    }
}
```

---

# 4. Giải thích cơ chế Retry

Điểm quan trọng nhất của Service nằm ở vòng lặp:

```java
for (int attempt = 0;
     attempt <= maxRetries;
     attempt++) {
```

Nếu:

```text
maxRetries = 0
```

thì chỉ có một lần gọi:

```text
Attempt 1
```

Nếu:

```text
maxRetries = 1
```

thì:

```text
Attempt 1 → Initial attempt
Attempt 2 → Retry
```

Nếu:

```text
maxRetries = 2
```

thì:

```text
Attempt 1 → Initial attempt
Attempt 2 → Retry 1
Attempt 3 → Retry 2
```

Do đó:

> `maxRetries` là số lần **thử lại sau lần gọi đầu tiên**, không phải tổng số lần gọi API.

---

# 5. Error Feedback Loop

## Lần gọi đầu tiên

Prompt:

```text
You are an AI resume extraction system.

Your task is to extract candidate information
from the resume below.

Return ONLY valid JSON.

Resume:

Nguyễn Văn An
Java Backend Developer
Email: an@gmail.com
Phone: 0901234567
Experience: 3 years
```

LLM trả về JSON lỗi:

```json
{
  "fullName": "Nguyễn Văn An",
  "phone": "0901234567",
  "email": "an@gmail.com",
  "skills": ["Java", "Spring Boot"],
  "yearsExperience": 3
```

Thiếu dấu:

```text
}
```

---

# 6. BeanOutputConverter phát hiện lỗi

Service gọi:

```java
CandidateExtraction result =
        converter.convert(llmResponse);
```

Jackson phát sinh lỗi, ví dụ:

```text
Unexpected end-of-input:
expected close marker for Object
(for String starting at line 1, column 1)
```

Service bắt lỗi:

```java
catch (Exception ex) {

    errorFeedback = ex.getMessage();
}
```

Sau đó:

```text
errorFeedback
      │
      ▼
buildPrompt()
      │
      ▼
Retry Prompt
```

---

# 7. Prompt ở lần Retry

Prompt lần thứ hai sẽ chứa:

```text
==============================
ERROR FEEDBACK
==============================

Your previous response could not be parsed.

Retry attempt:
2

Parser/Jackson error:

Unexpected end-of-input:
expected close marker for Object
(for String starting at line 1, column 1)

Analyze the error above and correct
your previous JSON response.

IMPORTANT:
Return ONLY the corrected JSON.
Do NOT explain the correction.
Do NOT use Markdown.
```

LLM sẽ có thêm thông tin để sửa lỗi.

---

# 8. LLM trả về JSON đã sửa

Ví dụ:

```json
{
  "fullName": "Nguyễn Văn An",
  "phone": "0901234567",
  "email": "an@gmail.com",
  "skills": [
    "Java",
    "Spring Boot"
  ],
  "yearsExperience": 3
}
```

Lần này:

```java
converter.convert(llmResponse);
```

parse thành công.

Kết quả:

```text
CandidateExtraction[
    fullName=Nguyễn Văn An,
    phone=0901234567,
    email=an@gmail.com,
    skills=[Java, Spring Boot],
    yearsExperience=3
]
```

Service lập tức:

```java
return result;
```

---

# 9. Trường hợp tất cả Retry đều thất bại

Giả sử:

```java
extractWithRetry(rawText, 2);
```

Có tổng cộng 3 lần gọi:

```text
Attempt 1 → FAIL
Attempt 2 → FAIL
Attempt 3 → FAIL
```

Sau khi vòng lặp kết thúc:

```java
return defaultExtraction();
```

Fallback:

```java
new CandidateExtraction(
        "UNKNOWN",
        "",
        "",
        List.of(),
        0
);
```

Kết quả:

```json
{
  "fullName": "UNKNOWN",
  "phone": "",
  "email": "",
  "skills": [],
  "yearsExperience": 0
}
```

Nhờ vậy exception không tiếp tục lan ra ngoài:

```text
LLM
 ↓
Invalid JSON
 ↓
Exception
 ↓
Retry
 ↓
Retry
 ↓
Fallback
 ↓
ETL vẫn tiếp tục
```

---

# 10. Ví dụ Log chạy thực tế

```text
========================================
SELF-HEALING EXTRACTION
========================================

Raw resume:
Nguyễn Văn An
Java Backend Developer

Email: an@gmail.com
Phone: 0901234567

Skills:
Java, Spring Boot

Experience:
3 years

========================================
ATTEMPT 1
========================================

Calling ChatModel...

LLM RESPONSE:

{
  "fullName": "Nguyễn Văn An",
  "phone": "0901234567",
  "email": "an@gmail.com",
  "skills": ["Java", "Spring Boot"],
  "yearsExperience": 3

========================================
PARSER
========================================

FAILED

Jackson error:

Unexpected end-of-input:
expected close marker for Object
(for String starting at line 1, column 1)

========================================
ERROR FEEDBACK
========================================

Adding exception.getMessage()
to retry prompt...

========================================
ATTEMPT 2
========================================

Calling ChatModel...

LLM RESPONSE:

{
  "fullName": "Nguyễn Văn An",
  "phone": "0901234567",
  "email": "an@gmail.com",
  "skills": [
    "Java",
    "Spring Boot"
  ],
  "yearsExperience": 3
}

========================================
PARSER
========================================

SUCCESS

========================================
FINAL RESULT
========================================

CandidateExtraction[
    fullName=Nguyễn Văn An,
    phone=0901234567,
    email=an@gmail.com,
    skills=[Java, Spring Boot],
    yearsExperience=3
]

========================================
RESULT: SUCCESS AFTER RETRY
========================================
```

---

# 11. Log trường hợp Fallback

Nếu:

```text
maxRetries = 2
```

và tất cả các lần đều thất bại:

```text
========================================
ATTEMPT 1
========================================
FAILED

========================================
ATTEMPT 2
========================================
FAILED

========================================
ATTEMPT 3
========================================
FAILED

========================================
MAX RETRIES REACHED
========================================

Returning default CandidateExtraction.

========================================
FALLBACK RESULT
========================================

CandidateExtraction[
    fullName=UNKNOWN,
    phone=,
    email=,
    skills=[],
    yearsExperience=0
]

========================================
RESULT: FALLBACK
========================================
```

---

# 12. Phân tích Trade-off

## 12.1 Chi phí Token

Self-Healing làm tăng số lần gọi LLM.

### Không có Retry

```text
Raw Text
   ↓
LLM Call
   ↓
Result
```

Ví dụ:

```text
Input  = 1,000 tokens
Output =   200 tokens

Total = 1,200 tokens
```

### Có Retry

Nếu lần đầu thất bại:

```text
LLM Call #1
   ↓
ERROR
   ↓
LLM Call #2
   ↓
SUCCESS
```

Ví dụ:

```text
Call #1 = 1,200 tokens

Call #2:
Input  = 1,000 tokens
Error  =   100 tokens
Output =   200 tokens

Total ≈ 2,500 tokens
```

Như vậy, self-healing có thể làm chi phí token tăng đáng kể.

Đặc biệt nếu hệ thống xử lý:

```text
100 CV/ngày
1,000 CV/ngày
10,000 CV/ngày
```

thì chi phí retry phải được theo dõi.

---

# 13. Latency

Mỗi lần retry là một lần gọi API mạng.

Giả sử một LLM call mất:

```text
2 giây
```

### Không retry

```text
LLM #1
   │
   └── 2s
       ↓
     Result
```

Latency:

```text
≈ 2s
```

### Retry 1 lần

```text
LLM #1 → 2s → FAIL
                   ↓
                LLM #2
                   ↓
                  2s
                   ↓
                SUCCESS
```

Latency:

```text
≈ 4s
```

### Retry 2 lần

```text
LLM #1 → 2s → FAIL
LLM #2 → 2s → FAIL
LLM #3 → 2s → SUCCESS
```

Latency:

```text
≈ 6s
```

Chưa tính:

* Network latency.
* API queue.
* Model processing time.
* Timeout.
* Prompt serialization.

Do đó không nên cho phép retry vô hạn.

---

# 14. Độ tin cậy

Self-Healing giúp tăng khả năng phục hồi đối với các lỗi format.

### Không Self-Healing

```text
Invalid JSON
     ↓
Parser Exception
     ↓
ETL FAILED
```

### Có Self-Healing

```text
Invalid JSON
     ↓
Parser Exception
     ↓
Error Feedback
     ↓
LLM sửa JSON
     ↓
Valid JSON
     ↓
ETL SUCCESS
```

Các lỗi phù hợp với cơ chế này:

* Thiếu `}`.
* Thiếu `]`.
* Sai dấu phẩy.
* JSON bị truncate.
* Thêm Markdown code fence.
* JSON sai cấu trúc.
* Một số lỗi field format.

---

# 15. Self-Healing không đảm bảo dữ liệu đúng

Đây là điểm quan trọng.

JSON hợp lệ:

```json
{
  "fullName": "Nguyễn Văn An",
  "phone": "0901234567",
  "email": "an@gmail.com",
  "skills": ["Java"],
  "yearsExperience": -10
}
```

vẫn có thể là dữ liệu sai nghiệp vụ.

Do đó:

```text
BeanOutputConverter
```

chỉ giải quyết vấn đề:

```text
JSON structure / mapping
```

Không đảm bảo:

```text
Business correctness
```

Pipeline nên là:

```text
LLM
 ↓
BeanOutputConverter
 ↓
Business Validation
 ↓
Database
```

Ví dụ validation:

```java
if (extraction.yearsExperience() < 0) {
    throw new IllegalArgumentException(
            "Years experience cannot be negative"
    );
}
```

---

# 16. Ưu điểm của Self-Healing

## 16.1 Tăng khả năng phục hồi

Thay vì:

```text
1 lỗi JSON → ETL FAILED
```

có thể:

```text
1 lỗi JSON → Retry → SUCCESS
```

---

## 16.2 Giảm lỗi do format

LLM có thể tự sửa những lỗi đơn giản khi nhận được thông báo từ parser.

Ví dụ:

```text
Missing closing brace
```

LLM có thể hiểu:

```text
JSON thiếu }
```

và trả lại JSON hợp lệ.

---

## 16.3 Không cần sửa thủ công

Không cần:

```text
Developer
   ↓
Read error
   ↓
Modify prompt
   ↓
Run again
```

Mà hệ thống tự động:

```text
Error
 ↓
Feedback
 ↓
LLM
 ↓
Correct Output
```

---

# 17. Nhược điểm của Self-Healing

## 17.1 Tăng chi phí

Mỗi retry là một LLM request mới.

```text
1 CV
 ↓
1 request

1 CV lỗi 1 lần
 ↓
2 requests

1 CV lỗi 2 lần
 ↓
3 requests
```

---

## 17.2 Tăng latency

Retry xảy ra tuần tự:

```text
Call #1
 ↓
Wait
 ↓
Error
 ↓
Call #2
 ↓
Wait
 ↓
Success
```

Không thể kỳ vọng latency bằng một request duy nhất.

---

## 17.3 Có nguy cơ retry không hiệu quả

Nếu prompt ban đầu có vấn đề nghiêm trọng, gửi lại cùng prompt cộng thêm error có thể vẫn thất bại:

```text
Attempt 1 → FAIL
Attempt 2 → FAIL
Attempt 3 → FAIL
```

Do đó cần giới hạn:

```text
maxRetries = 1–3
```

thay vì retry vô hạn.

---

## 17.4 Không giải quyết lỗi API

Self-Healing không phải giải pháp cho:

```text
401 Unauthorized
403 Forbidden
429 Rate Limit
500 Provider Error
Network Timeout
```

Các lỗi này nên được xử lý bằng:

* Exponential backoff.
* Rate limiting.
* Circuit breaker.
* Timeout.
* API fallback.
* Provider failover.

---

# 18. Bảng Trade-off

| Tiêu chí             | Không Self-Healing | Có Self-Healing        |
| -------------------- | ------------------ | ---------------------- |
| Token cost           | Thấp               | Cao hơn                |
| Latency              | Thấp               | Cao hơn                |
| JSON recovery        | Thấp               | Cao                    |
| Reliability          | Thấp hơn           | Cao hơn                |
| API calls            | 1                  | 1 + retries            |
| Code complexity      | Thấp               | Cao hơn                |
| Xử lý lỗi format     | Kém                | Tốt                    |
| Nguy cơ tăng chi phí | Thấp               | Có                     |
| Phù hợp production   | Có                 | Có, nếu giới hạn retry |

---

# 19. Khuyến nghị kiến trúc Production

Không nên chỉ sử dụng:

```text
try
 ↓
catch
 ↓
retry
```

Một kiến trúc tốt hơn:

```text
                       RAW TEXT
                          │
                          ▼
                   ┌─────────────┐
                   │  ChatModel  │
                   └──────┬──────┘
                          │
                          ▼
              ┌─────────────────────┐
              │ BeanOutputConverter  │
              └──────────┬──────────┘
                         │
                   ┌─────┴─────┐
                   │           │
                 PASS         FAIL
                   │           │
                   ▼           ▼
              Validation   Error Feedback
                   │           │
                   ▼           ▼
               Database      Retry
                               │
                        ┌──────┴──────┐
                        │             │
                     SUCCESS       MAX RETRY
                        │             │
                        ▼             ▼
                     Result       Fallback
                                      │
                                      ▼
                                Error Monitoring
```

---

# 20. Kết luận

Cơ chế **Self-Healing / Error Feedback Loop** là một giải pháp phù hợp để xử lý lỗi JSON do LLM tạo ra.

Luồng chính:

```text
Raw Text
   ↓
ChatModel
   ↓
LLM Response
   ↓
BeanOutputConverter
   │
   ├── SUCCESS ──────────────> Return Result
   │
   └── ERROR
          ↓
    exception.getMessage()
          ↓
    Error Feedback Prompt
          ↓
       ChatModel
          ↓
        Retry
          │
          ├── SUCCESS ───────> Return Result
          │
          └── FAILED
                 ↓
          maxRetries reached
                 ↓
             Fallback
```

### Ưu điểm

* Tăng khả năng tự phục hồi.
* Giảm lỗi ETL do JSON không hợp lệ.
* Có thể tự sửa các lỗi syntax đơn giản.
* Giảm khả năng phải trả HTTP 500 chỉ vì lỗi format của LLM.

### Nhược điểm

* Tăng token usage.
* Tăng latency.
* Tăng số lần gọi API.
* Có thể tăng chi phí đáng kể khi xử lý số lượng CV lớn.
* Không đảm bảo dữ liệu nghiệp vụ chính xác.
* Không nên retry vô hạn.

### Kết luận kiến trúc

Self-Healing nên được sử dụng như một **cơ chế recovery có giới hạn**, không phải giải pháp thay thế validation.

Thiết kế phù hợp:

```text
LLM
 ↓
BeanOutputConverter
 ↓
Self-Healing Retry
 ↓
Business Validation
 ↓
Database
```

Trong đó:

> **Self-Healing xử lý lỗi format/parse, còn Business Validation xử lý tính đúng đắn của dữ liệu.**

Để tránh runaway cost và latency, nên giới hạn `maxRetries` ở mức nhỏ, thường khoảng **1–3 lần**, đồng thời log số lần retry, loại lỗi và thời gian phản hồi để theo dõi hiệu năng hệ thống.
