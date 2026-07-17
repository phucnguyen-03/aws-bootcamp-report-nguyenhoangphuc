---
title: "Worklog Tuần 9"
date: 2026-06-13
weight: 9
chapter: false
pre: " <b> 1.9. </b> "
---

{{% notice warning %}}
⚠️ **Lưu ý:** Các thông tin dưới đây chỉ nhằm mục đích tham khảo, vui lòng **không sao chép nguyên văn** cho bài báo cáo của bạn kể cả warning này.
{{% /notice %}}

### Mục tiêu tuần 9:

* Tích hợp quy trình xử lý tài liệu tự động bằng AWS Serverless.
* Xây dựng luồng OCR và trích xuất nội dung tài liệu.
* Tích hợp Amazon Bedrock để tóm tắt và phân loại tài liệu.
* Lưu trữ kết quả xử lý AI vào cơ sở dữ liệu.

### Các công việc cần triển khai trong tuần này:

| Thứ | Công việc | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| 6 | - Cấu hình Amazon S3 Event Trigger.<br>- Xây dựng AWS Lambda để tự động xử lý khi người dùng tải tài liệu lên Amazon S3.<br>- Kiểm tra luồng kích hoạt tự động của hệ thống. | 13/06 | 13/06 | AWS Lambda Documentation |
| 7 | - Tích hợp Amazon Textract để nhận dạng văn bản (OCR) đối với PDF và hình ảnh.<br>- Cấu hình quy trình xử lý bất đồng bộ (Asynchronous Processing) sử dụng Amazon SNS Callback. | 14/06 | 14/06 | Amazon Textract Documentation |
| CN | - Phát triển Lambda Layer để xử lý các tệp DOCX và PPTX.<br>- Tích hợp các thư viện hỗ trợ trích xuất nội dung tài liệu và chuẩn hóa dữ liệu trước khi xử lý AI. | 15/06 | 15/06 | AWS Lambda Documentation |
| 2 | - Tích hợp Amazon Bedrock Claude Haiku.<br>- Xây dựng Prompt phục vụ chức năng tóm tắt nội dung và phân loại tài liệu.<br>- Kiểm tra kết quả phản hồi từ mô hình AI. | 16/06 | 16/06 | Amazon Bedrock Documentation |
| 3 | - Thiết kế cấu trúc lưu trữ Metadata trên Amazon DynamoDB.<br>- Lưu nội dung OCR, kết quả tóm tắt và thông tin phân loại của tài liệu sau khi xử lý AI. | 17/06 | 17/06 | Amazon DynamoDB Documentation |
| 4 | - Xây dựng cơ chế kiểm tra Quota AI theo từng người dùng trước khi gửi yêu cầu đến Amazon Bedrock.<br>- Tối ưu luồng xử lý nhằm giảm chi phí sử dụng dịch vụ AI. | 18/06 | 18/06 | AWS Best Practices |
| 5 | - Kiểm thử toàn bộ quy trình xử lý tài liệu từ Upload đến AI.<br>- Báo cáo tiến độ với Mentor.<br>- Tiếp nhận góp ý và điều chỉnh kiến trúc xử lý AI khi cần thiết. | 19/06 | 19/06 | Nội bộ Project |

### Kết quả đạt được tuần 9:

* Xây dựng thành công quy trình xử lý tài liệu tự động theo kiến trúc Serverless trên AWS.

* Hoàn thành cơ chế kích hoạt AWS Lambda thông qua Amazon S3 Event mỗi khi người dùng tải tài liệu lên hệ thống.

* Tích hợp thành công Amazon Textract để nhận dạng văn bản (OCR) đối với tài liệu PDF và hình ảnh bằng phương thức xử lý bất đồng bộ.

* Phát triển Lambda Layer để trích xuất nội dung từ các tệp DOCX và PPTX trước khi gửi đến hệ thống AI.

* Tích hợp Amazon Bedrock Claude Haiku nhằm:
  * Tóm tắt nội dung tài liệu
  * Phân loại tài liệu
  * Chuẩn hóa kết quả xử lý

* Thiết kế và lưu trữ thành công Metadata trên Amazon DynamoDB, bao gồm:
  * Nội dung OCR
  * Kết quả tóm tắt
  * Thông tin phân loại
  * Trạng thái xử lý

* Áp dụng cơ chế kiểm soát AI Quota nhằm hạn chế việc sử dụng tài nguyên vượt mức và tối ưu chi phí triển khai.

* Hoàn thành kiểm thử toàn bộ quy trình xử lý AI từ khi người dùng tải tài liệu lên cho đến khi kết quả được lưu vào hệ thống, tạo nền tảng cho việc xây dựng chức năng thông báo thời gian thực và AI Chat trong giai đoạn tiếp theo.