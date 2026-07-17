---
title: "Worklog Tuần 10"
date: 2026-06-20
weight: 10
chapter: false
pre: " <b> 1.10. </b> "
---

{{% notice warning %}}
⚠️ **Lưu ý:** Các thông tin dưới đây chỉ nhằm mục đích tham khảo, vui lòng **không sao chép nguyên văn** cho bài báo cáo của bạn kể cả warning này.
{{% /notice %}}

### Mục tiêu tuần 10:

* Hoàn thiện quy trình xử lý AI cho hệ thống.
* Triển khai cơ chế thông báo thời gian thực (Realtime Notification).
* Tối ưu hiệu suất xử lý của Lambda và Amazon Bedrock.
* Kiểm thử và đánh giá toàn bộ AI Processing Pipeline.

### Các công việc cần triển khai trong tuần này:

| Thứ | Công việc | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| 6 | - Tích hợp AWS AppSync Subscription để cập nhật trạng thái xử lý tài liệu theo thời gian thực.<br>- Kiểm tra kết nối giữa Backend và Frontend. | 20/06 | 20/06 | AWS AppSync Documentation |
| 7 | - Hiển thị tiến trình xử lý tài liệu trên giao diện người dùng.<br>- Đồng bộ trạng thái Upload, OCR, AI Processing và Completed theo thời gian thực. | 21/06 | 21/06 | AWS Amplify Documentation |
| CN | - Tối ưu Lambda Function.<br>- Cải thiện tốc độ xử lý và giảm thời gian thực thi.<br>- Kiểm tra khả năng xử lý đồng thời của hệ thống. | 22/06 | 22/06 | AWS Lambda Best Practices |
| 2 | - Tối ưu Prompt cho Amazon Bedrock Claude Haiku.<br>- Kiểm tra chất lượng kết quả tóm tắt và phân loại tài liệu.<br>- Điều chỉnh Prompt để tăng độ chính xác của AI. | 23/06 | 23/06 | Amazon Bedrock Documentation |
| 3 | - Kiểm tra luồng xử lý tài liệu với nhiều định dạng như PDF, DOCX, PPTX và hình ảnh.<br>- Đánh giá kết quả OCR và AI Processing trên từng loại tài liệu. | 24/06 | 24/06 | Amazon Textract Documentation |
| 4 | - Thực hiện kiểm thử tích hợp (Integration Testing).<br>- Kiểm tra khả năng xử lý lỗi và xử lý ngoại lệ của hệ thống.<br>- Khắc phục các lỗi phát sinh trong quá trình kiểm thử. | 25/06 | 25/06 | AWS Well-Architected Framework |
| 5 | - Báo cáo tiến độ với Mentor.<br>- Tổng hợp kết quả AI Processing.<br>- Chuẩn bị kế hoạch triển khai chức năng Semantic Search và AI Chat cho giai đoạn tiếp theo. | 26/06 | 26/06 | Nội bộ Project |

### Kết quả đạt được tuần 10:

* Hoàn thiện quy trình xử lý AI của hệ thống theo kiến trúc Serverless trên AWS.

* Triển khai thành công AWS AppSync Subscription giúp cập nhật trạng thái xử lý tài liệu theo thời gian thực mà không cần tải lại trang.

* Xây dựng giao diện theo dõi tiến trình xử lý tài liệu với các trạng thái:
  * Uploading
  * Processing
  * OCR Completed
  * AI Processing
  * Completed

* Tối ưu hiệu suất của AWS Lambda, giúp giảm thời gian xử lý và cải thiện khả năng phản hồi của hệ thống.

* Điều chỉnh Prompt cho Amazon Bedrock Claude Haiku nhằm nâng cao chất lượng:
  * AI Summary
  * Document Classification
  * Metadata Generation

* Kiểm thử thành công quá trình xử lý đối với nhiều định dạng tài liệu như PDF, DOCX, PPTX và Image.

* Hoàn thành kiểm thử tích hợp giữa Amazon S3, AWS Lambda, Amazon Textract, Amazon Bedrock, AWS AppSync và Amazon DynamoDB.

* Hoàn thiện AI Processing Pipeline, tạo nền tảng cho việc phát triển chức năng Semantic Search và AI-powered Document Chat trong giai đoạn tiếp theo.