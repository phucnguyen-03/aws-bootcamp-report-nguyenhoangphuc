---
title: "Worklog Tuần 7"
date: 2026-05-30
weight: 7
chapter: false
pre: " <b> 1.7. </b> "
---

{{% notice warning %}}
⚠️ **Lưu ý:** Các thông tin dưới đây chỉ nhằm mục đích tham khảo, vui lòng **không sao chép nguyên văn** cho bài báo cáo của bạn kể cả warning này.
{{% /notice %}}

### Mục tiêu tuần 7:

* Khởi tạo dự án AI-Powered Smart Document Assistant.
* Thiết lập môi trường phát triển với AWS Amplify Gen 2.
* Xây dựng kiến trúc ban đầu của hệ thống theo mô hình Serverless.
* Chuẩn bị hạ tầng và quy trình CI/CD phục vụ cho quá trình phát triển Project.

### Các công việc cần triển khai trong tuần này:

| Thứ | Công việc | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| 6 | - Khởi tạo repository của Project.<br>- Khởi tạo dự án AWS Amplify Gen 2.<br>- Thiết lập cấu trúc Backend-as-Code cho hệ thống. | 30/05 | 30/05 | AWS Amplify Documentation |
| 7 | - Thiết lập môi trường phát triển gồm **dev** và **prod**.<br>- Cấu hình Amplify Sandbox phục vụ quá trình phát triển và kiểm thử. | 31/05 | 31/05 | AWS Amplify Documentation |
| CN | - Thiết lập AWS Amplify Hosting.<br>- Kết nối Repository với Amplify.<br>- Cấu hình quy trình CI/CD tự động Build và Deploy khi có thay đổi trên Git Repository. | 01/06 | 01/06 | AWS Amplify Hosting |
| 2 | - Thiết lập AWS Budget Alert để theo dõi chi phí sử dụng dịch vụ.<br>- Cấu hình CloudWatch Billing Alarm nhằm kiểm soát ngân sách trong quá trình phát triển. | 02/06 | 02/06 | AWS Budgets Documentation |
| 3 | - Thiết lập IAM Roles theo nguyên tắc Least Privilege.<br>- Phân quyền riêng cho từng Lambda Function và các dịch vụ liên quan. | 03/06 | 03/06 | AWS IAM Documentation |
| 4 | - Thiết kế kiến trúc tổng thể của hệ thống AI-Powered Smart Document Assistant.<br>- Xác định các dịch vụ AWS sẽ sử dụng trong Project như Amplify, Cognito, AppSync, Lambda, S3, DynamoDB và Bedrock. | 04/06 | 04/06 | AWS Architecture Center |
| 5 | - Trao đổi với mentor về kiến trúc hệ thống và lộ trình phát triển Project.<br>- Tổng hợp các nội dung đã thực hiện và chuẩn bị triển khai các chức năng của MVP trong tuần tiếp theo. | 05/06 | 05/06 | Nội bộ Project |

### Kết quả đạt được tuần 7:

* Khởi tạo thành công dự án **AI-Powered Smart Document Assistant** bằng AWS Amplify Gen 2.

* Thiết lập hoàn chỉnh môi trường phát triển gồm:
  * Development Environment (dev)
  * Production Environment (prod)

* Cấu hình thành công AWS Amplify Hosting và tích hợp quy trình CI/CD giúp tự động Build và Deploy khi cập nhật mã nguồn.

* Thiết lập AWS Budget Alert và CloudWatch Billing Alarm để theo dõi chi phí sử dụng các dịch vụ AWS trong quá trình phát triển.

* Áp dụng nguyên tắc **Least Privilege** trong việc cấu hình IAM Roles nhằm tăng cường bảo mật cho hệ thống.

* Hoàn thành thiết kế kiến trúc tổng thể của hệ thống theo mô hình **Serverless Architecture** trên AWS.

* Xác định các dịch vụ chính sẽ sử dụng trong Project gồm:
  * AWS Amplify Hosting
  * Amazon Cognito
  * AWS AppSync
  * AWS Lambda
  * Amazon S3
  * Amazon DynamoDB
  * Amazon Bedrock
  * Amazon CloudWatch

* Hoàn thành giai đoạn chuẩn bị hạ tầng, môi trường phát triển và kế hoạch triển khai, sẵn sàng bước sang giai đoạn xây dựng MVP của hệ thống trong tuần tiếp theo.