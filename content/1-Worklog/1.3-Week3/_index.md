---
title: "Worklog Week 3"
date: 2026-05-02
weight: 3
chapter: false
pre: " <b> 1.3. </b> "
---

{{% notice warning %}}
⚠️ **Note:** The information below is for reference purposes only. Please **do not copy it word for word** into your internship report, including this warning.
{{% /notice %}}

### Weekly Objectives:

* Learn about AWS networking architecture through Amazon Virtual Private Cloud (VPC).
* Practice building a basic network infrastructure on AWS.
* Understand the connectivity between the Internet and Amazon EC2 instances.
* Gain knowledge of network security components within the AWS environment.

### Weekly Tasks:

| Day | Tasks | Start Date | Completion Date | Reference |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| Saturday | - Study the fundamentals of Amazon Virtual Private Cloud (VPC).<br>- Learn about CIDR blocks, VPC architecture, and IP address ranges. | 02/05 | 02/05 | https://cloudjourney.awsstudygroup.com/ |
| Sunday | - Practice creating a Virtual Private Cloud (VPC).<br>- Create Public and Private Subnets.<br>- Configure IP address allocation for each subnet. | 03/05 | 03/05 | https://cloudjourney.awsstudygroup.com/ |
| Monday | - Learn about Internet Gateway (IGW).<br>- Create and attach an Internet Gateway to the VPC.<br>- Configure Route Tables to enable Internet access for EC2 instances. | 04/05 | 04/05 | https://cloudjourney.awsstudygroup.com/ |
| Tuesday | - Study NAT Gateway.<br>- Configure a NAT Gateway for the Private Subnet.<br>- Verify Internet connectivity from private EC2 instances. | 05/05 | 05/05 | https://cloudjourney.awsstudygroup.com/ |
| Wednesday | - Learn about Security Groups and Network ACLs.<br>- Configure inbound and outbound traffic rules.<br>- Compare the differences between Security Groups and Network ACLs. | 06/05 | 06/05 | https://cloudjourney.awsstudygroup.com/ |
| Thursday | - Deploy EC2 instances within the configured VPC.<br>- Test communication between EC2 instances.<br>- Verify SSH and HTTP connectivity in the AWS networking environment. | 07/05 | 07/05 | https://cloudjourney.awsstudygroup.com/ |
| Friday | - Review all VPC-related concepts.<br>- Repeat the completed hands-on labs.<br>- Discuss the completed tasks with the mentor and collect feedback for improvement. | 08/05 | 08/05 | https://cloudjourney.awsstudygroup.com/ |

### Results Achieved in Week 3:

* Gained a solid understanding of AWS networking architecture through Amazon Virtual Private Cloud (VPC).

* Successfully created a private network environment on AWS, including:
  * Virtual Private Cloud (VPC)
  * Public Subnets
  * Private Subnets

* Configured an Internet Gateway and Route Tables to allow EC2 instances in the Public Subnet to access the Internet.

* Deployed a NAT Gateway to provide Internet access for EC2 instances located in the Private Subnet while maintaining network security.

* Developed an understanding of the functions and purposes of:
  * Security Groups
  * Network Access Control Lists (Network ACLs)

* Learned how to configure inbound and outbound rules to control network traffic for AWS resources.

* Successfully deployed EC2 instances within the configured VPC and verified connectivity between instances using SSH and HTTP.

* Completed the Amazon VPC hands-on labs and gained practical experience in designing and implementing a basic AWS network infrastructure for future application deployment.