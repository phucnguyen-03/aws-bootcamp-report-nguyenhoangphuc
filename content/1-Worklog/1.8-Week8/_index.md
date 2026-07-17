---
title: "Worklog Week 8"
date: 2026-06-06
weight: 8
chapter: false
pre: " <b> 1.8. </b> "
---

{{% notice warning %}}
⚠️ **Note:** The information below is for reference purposes only. Please **do not copy it word for word** into your internship report, including this warning.
{{% /notice %}}

### Weekly Objectives:

* Develop the first Minimum Viable Product (MVP) of the system.
* Implement user authentication using Amazon Cognito.
* Develop document management features with Amazon S3.
* Complete the basic user interface and integrate the frontend with the backend.

### Weekly Tasks:

| Day | Tasks | Start Date | Completion Date | Reference |
| --- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| Saturday | - Implement user registration and sign-in using Amazon Cognito.<br>- Configure the User Pool, Authentication Flow, and Email Verification for user authentication. | 06/06 | 06/06 | AWS Cognito Documentation |
| Sunday | - Configure the Forgot Password feature and Multi-Factor Authentication (MFA).<br>- Set up Email OTP for account verification. | 07/06 | 07/06 | AWS Cognito Documentation |
| Monday | - Integrate Amazon S3 into the system.<br>- Implement document uploads using Presigned URLs.<br>- Validate file types and file size limits before uploading. | 08/06 | 08/06 | Amazon S3 Documentation |
| Tuesday | - Develop GraphQL APIs using AWS AppSync.<br>- Design GraphQL Queries and Mutations for document management.<br>- Store document metadata in Amazon DynamoDB. | 09/06 | 09/06 | AWS AppSync Documentation |
| Wednesday | - Develop the document management interface.<br>- Implement document preview, download, and delete features.<br>- Configure document access permissions using Amazon S3 Access Levels. | 10/06 | 10/06 | AWS Amplify Storage Documentation |
| Thursday | - Improve the Angular user interface with responsive design.<br>- Implement Dark Mode and Light Mode support.<br>- Test the application's responsiveness across multiple screen sizes. | 11/06 | 11/06 | Angular Documentation |
| Friday | - Perform end-to-end testing of all MVP features.<br>- Present the project progress to the mentor.<br>- Collect feedback and prepare the implementation plan for AI integration in the next phase. | 12/06 | 12/06 | Internal Project Documentation |

### Results Achieved in Week 8:

* Successfully completed the first **Minimum Viable Product (MVP)** of the **AI-Powered Smart Document Assistant**.

* Successfully implemented user authentication using **Amazon Cognito**, including:
  * User Registration
  * User Login
  * Email Verification
  * Forgot Password
  * Multi-Factor Authentication (MFA)

* Successfully integrated **Amazon S3** for secure document storage using Presigned URLs, improving upload performance while maintaining data security.

* Developed **GraphQL APIs** with **AWS AppSync** and stored document metadata in **Amazon DynamoDB**.

* Successfully implemented the core document management features, including:
  * Document Upload
  * Document List
  * File Preview
  * File Download
  * File Deletion

* Configured document access permissions using **Amazon S3 Access Levels**, ensuring that documents can only be accessed by authorized users.

* Completed a responsive user interface using **Angular**, with support for both **Dark Mode** and **Light Mode**.

* Successfully tested all MVP functionalities and completed the first functional version of the system, providing a solid foundation for integrating **Amazon Textract** and **Amazon Bedrock** in the next development phase.