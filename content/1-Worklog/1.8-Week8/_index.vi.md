---
title: "Worklog Tuần 8"
date: 2026-06-06
weight: 8
chapter: false
pre: " <b> 1.8. </b> "
---

{{% notice warning %}}
⚠️ **Lưu ý:** Các thông tin dưới đây chỉ nhằm mục đích tham khảo, vui lòng **không sao chép nguyên văn** cho bài báo cáo của bạn kể cả warning này.
{{% /notice %}}

### Mục tiêu tuần 8:

* Xây dựng phiên bản MVP đầu tiên của hệ thống.
* Triển khai chức năng xác thực người dùng bằng Amazon Cognito.
* Xây dựng chức năng quản lý tài liệu trên Amazon S3.
* Hoàn thiện giao diện cơ bản và kết nối Frontend với Backend.

### Các công việc cần triển khai trong tuần này:

| Thứ | Công việc | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu |
| --- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| 6 | - Triển khai chức năng đăng ký và đăng nhập bằng Amazon Cognito.<br>- Cấu hình User Pool, Authentication Flow và xác thực Email cho người dùng. | 06/06 | 06/06 | AWS Cognito Documentation |
| 7 | - Cấu hình chức năng Quên mật khẩu (Forgot Password) và xác thực đa yếu tố (MFA).<br>- Thiết lập Email OTP phục vụ quá trình xác thực tài khoản. | 07/06 | 07/06 | AWS Cognito Documentation |
| CN | - Tích hợp Amazon S3 vào hệ thống.<br>- Thực hiện Upload tài liệu bằng Presigned URL.<br>- Kiểm tra giới hạn loại tệp và kích thước tệp trước khi tải lên. | 08/06 | 08/06 | Amazon S3 Documentation |
| 2 | - Xây dựng API GraphQL bằng AWS AppSync.<br>- Thiết kế Mutation và Query phục vụ quản lý tài liệu.<br>- Lưu metadata tài liệu trên Amazon DynamoDB. | 09/06 | 09/06 | AWS AppSync Documentation |
| 3 | - Xây dựng giao diện hiển thị danh sách tài liệu.<br>- Thực hiện chức năng Preview, Download và Delete tài liệu.<br>- Cấu hình phân quyền truy cập thông qua Amazon S3 Access Level. | 10/06 | 10/06 | AWS Amplify Storage |
| 4 | - Hoàn thiện giao diện Responsive trên Angular.<br>- Bổ sung Dark Mode và Light Mode.<br>- Kiểm tra khả năng hoạt động trên nhiều kích thước màn hình. | 11/06 | 11/06 | Angular Documentation |
| 5 | - Kiểm thử toàn bộ các chức năng của phiên bản MVP.<br>- Báo cáo tiến độ với Mentor.<br>- Tiếp nhận góp ý và lập kế hoạch tích hợp AI cho giai đoạn tiếp theo. | 12/06 | 12/06 | Nội bộ Project |

### Kết quả đạt được tuần 8:

* Hoàn thành phiên bản MVP đầu tiên của hệ thống **AI-Powered Smart Document Assistant**.

* Triển khai thành công hệ thống xác thực người dùng bằng Amazon Cognito, bao gồm:
  * User Registration
  * User Login
  * Email Verification
  * Forgot Password
  * Multi-Factor Authentication (MFA)

* Tích hợp thành công Amazon S3 để lưu trữ tài liệu thông qua Presigned URL, giúp tăng hiệu suất upload và đảm bảo tính bảo mật.

* Xây dựng GraphQL API bằng AWS AppSync và lưu trữ metadata của tài liệu trên Amazon DynamoDB.

* Hoàn thiện các chức năng quản lý tài liệu gồm:
  * Upload Document
  * Document List
  * File Preview
  * Download File
  * Delete File

* Áp dụng cơ chế phân quyền truy cập tài liệu thông qua Amazon S3 Access Level nhằm đảm bảo dữ liệu chỉ được truy cập bởi người dùng có quyền.

* Hoàn thiện giao diện người dùng với khả năng Responsive và hỗ trợ chế độ Dark/Light Mode.

* Hoàn thành kiểm thử chức năng của phiên bản MVP và sẵn sàng chuyển sang giai đoạn tích hợp AI với Amazon Textract và Amazon Bedrock trong tuần tiếp theo.