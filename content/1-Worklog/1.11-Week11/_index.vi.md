---
title: "Worklog Tuần 11"
date: 2026-06-27
weight: 11
chapter: false
pre: " <b> 1.11. </b> "
---

{{% notice warning %}}
⚠️ **Lưu ý:** Các thông tin dưới đây chỉ nhằm mục đích tham khảo, vui lòng **không sao chép nguyên văn** cho bài báo cáo của bạn kể cả warning này.
{{% /notice %}}

### Mục tiêu tuần 11:

* Phát triển chức năng tìm kiếm thông minh (Semantic Search).
* Xây dựng hệ thống AI Chat dựa trên nội dung tài liệu.
* Tích hợp Amazon Titan Embeddings và mô hình RAG.
* Tối ưu khả năng truy xuất và khai thác dữ liệu từ tài liệu.

### Các công việc cần triển khai trong tuần này:

| Thứ | Công việc | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| 6 | - Thiết kế chức năng tìm kiếm tài liệu theo tên, loại tài liệu và thời gian upload.<br>- Cấu hình Global Secondary Index (GSI) trên Amazon DynamoDB để tối ưu truy vấn. | 27/06 | 27/06 | Amazon DynamoDB Documentation |
| 7 | - Tích hợp Amazon Titan Embeddings để tạo Vector Embeddings từ nội dung tài liệu.<br>- Lưu trữ Vector Embeddings cùng Metadata trên Amazon DynamoDB. | 28/06 | 28/06 | Amazon Bedrock Documentation |
| CN | - Phát triển chức năng Semantic Search.<br>- Xây dựng thuật toán tính Cosine Similarity để tìm kiếm các tài liệu có nội dung liên quan. | 29/06 | 29/06 | AWS Bedrock Documentation |
| 2 | - Xây dựng AI Chat sử dụng mô hình Retrieval-Augmented Generation (RAG).<br>- Thực hiện truy xuất các đoạn nội dung phù hợp trước khi gửi yêu cầu đến Amazon Bedrock Claude Haiku. | 30/06 | 30/06 | Amazon Bedrock Documentation |
| 3 | - Tích hợp giao diện AI Chat trên hệ thống.<br>- Hiển thị câu trả lời và các đoạn tài liệu được AI tham chiếu trong quá trình phản hồi. | 01/07 | 01/07 | Angular Documentation |
| 4 | - Kiểm thử chức năng Semantic Search và AI Chat trên nhiều loại tài liệu.<br>- Đánh giá độ chính xác của kết quả truy xuất và phản hồi từ AI.<br>- Điều chỉnh Prompt và thuật toán truy xuất khi cần thiết. | 02/07 | 02/07 | AWS Best Practices |
| 5 | - Báo cáo kết quả với Mentor.<br>- Tổng hợp các nội dung đã hoàn thành.<br>- Chuẩn bị cho giai đoạn hoàn thiện hệ thống và triển khai bản Demo cuối cùng. | 03/07 | 03/07 | Nội bộ Project |

### Kết quả đạt được tuần 11:

* Hoàn thành chức năng tìm kiếm tài liệu dựa trên Metadata bằng Amazon DynamoDB Global Secondary Index (GSI).

* Tích hợp thành công Amazon Titan Embeddings để tạo Vector Embeddings từ nội dung tài liệu.

* Xây dựng cơ chế Semantic Search giúp tìm kiếm tài liệu theo mức độ tương đồng về ngữ nghĩa thay vì chỉ dựa trên từ khóa.

* Phát triển thành công hệ thống AI Chat theo mô hình Retrieval-Augmented Generation (RAG).

* Xây dựng quy trình xử lý gồm:
  * Sinh Vector Embeddings từ câu hỏi của người dùng.
  * Tìm kiếm các đoạn nội dung liên quan trong cơ sở dữ liệu.
  * Gửi ngữ cảnh truy xuất được đến Amazon Bedrock Claude Haiku để tạo câu trả lời.

* Hoàn thiện giao diện AI Chat và hiển thị kết quả phản hồi theo thời gian thực.

* Kiểm thử chức năng Semantic Search và AI Chat trên nhiều loại tài liệu, đồng thời tối ưu chất lượng truy xuất và phản hồi của hệ thống.

* Hoàn thiện giai đoạn phát triển các chức năng AI cốt lõi, tạo tiền đề cho việc tối ưu hệ thống, kiểm thử toàn diện và chuẩn bị Demo cuối kỳ.