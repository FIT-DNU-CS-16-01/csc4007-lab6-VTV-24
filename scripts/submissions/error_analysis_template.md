# Lab 6 – Báo cáo Phân tích Lỗi (Error Analysis Report)

## 1. Định nghĩa bài toán

* **Bài toán được chọn:** Phân loại cảm xúc (Sentiment Analysis) cho đánh giá phim.
* **Đầu vào:** Một đoạn review phim bằng tiếng Anh.
* **Đầu ra:** JSON chứa nhãn cảm xúc (`positive`, `negative`, `neutral`, `mixed`) cùng các thông tin giải thích.
* **LLM sử dụng:** Ollama (Llama 3.2).
* **Phương thức thực thi prompt:** Local model thông qua Ollama API.

---

## 2. Bộ dữ liệu kiểm thử

* Số lượng review: 29
* Số lượng review positive: 10
* Số lượng review negative: 9
* Số lượng review neutral: 2
* Số lượng review mixed: 8
* Number of easy reviews: 19
* Number of mixed reviews: 8
* Number of ambiguous reviews: 2
* Number of keyword-trap reviews: 0
* Number of long reviews: 0

---

## 3. Prompt v1 – Baseline Prompt

Prompt v1 yêu cầu mô hình:

* Xác định cảm xúc tổng thể của review.
* Trả về JSON gồm:

  * sentiment
  * short_explanation
  * evidence_phrases

Đây là phiên bản cơ bản nhất, không có hướng dẫn đặc biệt về các trường hợp mixed hoặc neutral.

---

## 4. Prompt v2 – Improved Prompt

Prompt v2 mở rộng thêm:

* Aspect-based sentiment analysis.
* Yêu cầu xác định từng khía cạnh như:

  * acting
  * story
  * visuals
  * music
  * pacing
  * dialogue
  * direction
* Bổ sung confidence score.

Mục tiêu của Prompt v2 là:

* Giảm lỗi đánh giá cảm xúc tổng quát.
* Tăng khả năng giải thích.
* Trích xuất bằng chứng rõ ràng hơn.

Tuy nhiên prompt phức tạp hơn khiến mô hình sinh nhiều JSON lỗi định dạng.

---

## 5. Prompt v3 – CoT-inspired Prompt

Prompt v3 khuyến khích mô hình suy luận theo từng bước:

1. Xác định positive clues.
2. Xác định negative clues.
3. Chọn dominant reason.
4. Kết luận sentiment cuối cùng.

Prompt không yêu cầu mô hình hiển thị toàn bộ chuỗi suy luận nội bộ, chỉ xuất JSON cuối cùng.

Mục tiêu:

* Tăng độ chính xác với review mơ hồ.
* Giảm lỗi bỏ sót thông tin.

---

## 6. So sánh định lượng

| Metric                  | Prompt v1 |  Prompt v2 | Prompt v3 CoT | Comment                                       |
| ----------------------- | --------: | ---------: | ------------: | --------------------------------------------- |
| Accuracy                |     72.4% |      55.2% |         62.1% | v1 tốt nhất                                   |
| Valid JSON rate         |      100% |      51.7% |         96.6% | v2 tạo nhiều JSON lỗi                         |
| Evidence exactness rate |       Cao | Trung bình |           Cao | v3 trích xuất clue khá tốt                    |
| Hallucination count     |         0 |          1 |             0 | Không đáng kể                                 |
| Outside knowledge count |         0 |          0 |             0 | Không sử dụng tri thức ngoài                  |
| Overconfidence count    |         2 |          7 |             6 | v2 và v3 thường confidence cao dù dự đoán sai |
| Error count             |         8 |         13 |            11 | v1 ít lỗi nhất                                |

---

## 7. Nhóm lỗi (Error Buckets)

| Error bucket           | Count v1 | Count v2 | Count v3 CoT | Example review_id | Comment                     |
| ---------------------- | -------: | -------: | -----------: | ----------------- | --------------------------- |
| wrong_sentiment        |        8 |       13 |           11 | R006              | Sai nhãn cảm xúc            |
| invalid_json           |        0 |       14 |            1 | R002              | JSON không parse được       |
| hallucinated_evidence  |        0 |        1 |            0 | R018              | Evidence không rõ ràng      |
| outside_knowledge      |        0 |        0 |            0 | -                 | Không xuất hiện             |
| missed_positive_aspect |        3 |        5 |            5 | R011              | Bỏ sót yếu tố tích cực      |
| missed_negative_aspect |        2 |        4 |            4 | R017              | Bỏ sót yếu tố tiêu cực      |
| keyword_trap           |        0 |        0 |            0 | -                 | Không có mẫu dữ liệu        |
| mixed_review_failure   |        6 |        8 |            8 | R006              | Chuyển mixed thành negative |
| overconfident          |        2 |        7 |            6 | R020              | Sai nhưng confidence cao    |
| cot_not_helpful        |        - |        - |            7 | R024              | CoT không cải thiện kết quả |

---

## 8. Ba lỗi đáng chú ý

### Error 1

* Review ID: R006
* Review type: mixed
* Gold sentiment: mixed
* Prompt version: v3
* What happened:

  * Review có cả ý tích cực và tiêu cực.
  * Mô hình xác định đúng positive clue và negative clue.
  * Tuy nhiên kết quả cuối cùng lại là negative.
* Why it matters:

  * Đây là lỗi phổ biến nhất trong bộ dữ liệu.
* How Prompt v2 or Prompt v3 tries to fix it:

  * Prompt v3 yêu cầu liệt kê clues nhưng vẫn thiên về chọn một cực cảm xúc duy nhất.

---

### Error 2

* Review ID: R020
* Review type: mixed
* Gold sentiment: mixed
* Prompt version: v2
* What happened:

  * Mô hình chỉ tập trung vào cụm "frustratingly bad".
  * Bỏ qua cụm "excellent".
* Why it matters:

  * Làm mất tính cân bằng của review.
* How Prompt v2 or Prompt v3 tries to fix it:

  * Prompt v3 có trích xuất cả positive và negative clues nhưng vẫn kết luận sai.

---

### Error 3

* Review ID: R002
* Review type: easy
* Gold sentiment: negative
* Prompt version: v2
* What happened:

  * JSON sinh ra bị lỗi dấu ngoặc kép.
  * Không parse được.
* Why it matters:

  * Hệ thống đánh giá tự động sẽ không đọc được kết quả.
* How Prompt v2 or Prompt v3 tries to fix it:

  * Prompt v3 đơn giản hơn nên gần như loại bỏ lỗi định dạng.

---

## 9. Nhận xét

### LLM làm tốt điều gì?

* Nhận diện rất tốt review positive và negative rõ ràng.
* Trích xuất evidence khá chính xác.
* Không sử dụng kiến thức ngoài review.

### Loại review nào khó nhất?

* Mixed review.
* Neutral review.
* Những review có cả lời khen và lời chê.

### Prompt v2 có cải thiện so với Prompt v1 không?

Không.

Mặc dù cung cấp thêm aspect analysis nhưng accuracy giảm và JSON lỗi nhiều hơn.

### Prompt v3 CoT có cải thiện so với Prompt v2 không?

Có.

* Accuracy tăng.
* JSON hợp lệ hơn.
* Giảm lỗi định dạng.

Tuy nhiên vẫn chưa vượt được Prompt v1.

### CoT có làm kết quả tệ hơn không?

Có.

Trong nhiều trường hợp mô hình suy luận quá mức và chuyển review mixed thành negative.

### Cần cải thiện gì tiếp theo?

* Thêm luật rõ ràng cho nhãn mixed.
* Ép mô hình trả về JSON hợp lệ.
* Bổ sung ví dụ few-shot cho mixed reviews.

---

## 10. Kết luận

Trong ba phiên bản prompt được thử nghiệm, Prompt v1 cho kết quả đáng tin cậy nhất với độ chính xác khoảng 72.4% và tỷ lệ JSON hợp lệ đạt 100%. Prompt v2 bổ sung phân tích theo khía cạnh nhưng làm tăng đáng kể số lượng lỗi định dạng JSON và giảm độ chính xác xuống khoảng 55.2%. Prompt v3 áp dụng phương pháp Chain-of-Thought giúp cải thiện so với v2 nhưng vẫn chỉ đạt khoảng 62.1% accuracy. Phần lớn lỗi của cả v2 và v3 xuất hiện ở các review mang cảm xúc hỗn hợp (mixed reviews), khi mô hình có xu hướng nghiêng về negative thay vì giữ nhãn mixed. Ngoài ra, v2 còn gặp nhiều lỗi cú pháp JSON khiến kết quả khó sử dụng trong hệ thống đánh giá tự động. Dựa trên các chỉ số định lượng và các ví dụ lỗi đã phân tích, Prompt v1 hiện là lựa chọn đáng tin cậy nhất cho hệ thống CineSense. Trong tương lai, việc bổ sung các quy tắc xử lý mixed sentiment và kiểm soát định dạng JSON sẽ là hướng cải thiện quan trọng.
