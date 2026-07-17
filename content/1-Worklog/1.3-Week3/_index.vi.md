---
title: "Worklog Tuần 3"
date: 2026-05-02
weight: 3
chapter: false
pre: " <b> 1.3. </b> "
---

{{% notice warning %}}
⚠️ **Lưu ý:** Các thông tin dưới đây chỉ nhằm mục đích tham khảo, vui lòng **không sao chép nguyên văn** cho bài báo cáo của bạn kể cả warning này.
{{% /notice %}}

### Mục tiêu tuần 3:

* Tìm hiểu kiến trúc mạng trên AWS thông qua dịch vụ Amazon VPC.
* Thực hành xây dựng hệ thống mạng cơ bản trên AWS.
* Hiểu cơ chế kết nối giữa Internet và EC2 Instance.
* Nắm được các thành phần bảo mật trong môi trường mạng AWS.

### Các công việc cần triển khai trong tuần này:

| Thứ | Công việc | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| 6 | - Tìm hiểu tổng quan về Amazon Virtual Private Cloud (VPC).<br>- Hiểu khái niệm CIDR Block, VPC và phạm vi địa chỉ IP. | 02/05 | 02/05 | https://cloudjourney.awsstudygroup.com/ |
| 7 | - Thực hành tạo VPC.<br>- Tạo Public Subnet và Private Subnet.<br>- Phân chia địa chỉ mạng cho từng Subnet. | 03/05 | 03/05 | https://cloudjourney.awsstudygroup.com/ |
| CN | - Tìm hiểu Internet Gateway (IGW).<br>- Tạo và gắn Internet Gateway vào VPC.<br>- Thiết lập Route Table để EC2 có thể truy cập Internet. | 04/05 | 04/05 | https://cloudjourney.awsstudygroup.com/ |
| 2 | - Tìm hiểu NAT Gateway.<br>- Cấu hình NAT Gateway cho Private Subnet.<br>- Kiểm tra khả năng truy cập Internet từ Private EC2 Instance. | 05/05 | 05/05 | https://cloudjourney.awsstudygroup.com/ |
| 3 | - Tìm hiểu Security Group và Network ACL.<br>- Cấu hình các luật Inbound và Outbound.<br>- So sánh sự khác nhau giữa Security Group và Network ACL. | 06/05 | 06/05 | https://cloudjourney.awsstudygroup.com/ |
| 4 | - Thực hành triển khai EC2 trong VPC vừa tạo.<br>- Kiểm tra kết nối giữa các Instance.<br>- Kiểm thử truy cập SSH và HTTP trong môi trường mạng AWS. | 07/05 | 07/05 | https://cloudjourney.awsstudygroup.com/ |
| 5 | - Ôn tập toàn bộ kiến thức về VPC.<br>- Thực hành lại các bài Lab đã học.<br>- Trao đổi với mentor về các nội dung đã thực hiện và ghi nhận góp ý. | 08/05 | 08/05 | https://cloudjourney.awsstudygroup.com/ |

### Kết quả đạt được tuần 3:

* Hiểu được kiến trúc mạng trên AWS thông qua dịch vụ Amazon Virtual Private Cloud (VPC).

* Tạo thành công môi trường mạng riêng trên AWS bao gồm:
  * Virtual Private Cloud (VPC)
  * Public Subnet
  * Private Subnet

* Cấu hình Internet Gateway và Route Table để cho phép EC2 Instance trong Public Subnet truy cập Internet.

* Triển khai NAT Gateway giúp các EC2 Instance trong Private Subnet có thể truy cập Internet mà vẫn đảm bảo tính bảo mật.

* Hiểu được chức năng của:
  * Security Group
  * Network ACL

* Biết cách cấu hình các luật Inbound và Outbound nhằm kiểm soát lưu lượng truy cập vào tài nguyên AWS.

* Thực hành triển khai EC2 trong VPC và kiểm tra kết nối giữa các máy chủ thông qua SSH và HTTP.

* Hoàn thành các bài Lab về Amazon VPC và nắm được quy trình xây dựng một hệ thống mạng cơ bản trên AWS phục vụ cho việc triển khai ứng dụng trong các giai đoạn tiếp theo.