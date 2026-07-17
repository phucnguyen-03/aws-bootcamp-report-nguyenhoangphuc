---
title: "Worklog Week 10"
date: 2026-06-20
weight: 10
chapter: false
pre: " <b> 1.10. </b> "
---

{{% notice warning %}}
⚠️ **Note:** The information below is for reference purposes only. Please **do not copy it word for word** into your internship report, including this warning.
{{% /notice %}}

### Weekly Objectives:

* Complete the AI processing workflow of the system.
* Implement a real-time notification mechanism.
* Optimize the performance of AWS Lambda and Amazon Bedrock.
* Test and evaluate the entire AI Processing Pipeline.

### Weekly Tasks:

| Day | Tasks | Start Date | Completion Date | Reference |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| Saturday | - Integrate AWS AppSync Subscriptions to provide real-time updates on document processing status.<br>- Verify the communication between the backend and frontend services. | 20/06 | 20/06 | AWS AppSync Documentation |
| Sunday | - Implement a real-time document processing status interface.<br>- Synchronize document states, including Uploading, OCR Processing, AI Processing, and Completed, without requiring a page refresh. | 21/06 | 21/06 | AWS Amplify Documentation |
| Monday | - Optimize AWS Lambda functions.<br>- Improve execution performance and reduce processing latency.<br>- Evaluate the system's ability to handle concurrent requests. | 22/06 | 22/06 | AWS Lambda Best Practices |
| Tuesday | - Optimize prompts for Amazon Bedrock Claude Haiku.<br>- Evaluate the quality of document summaries and classifications.<br>- Refine prompts to improve AI response accuracy. | 23/06 | 23/06 | Amazon Bedrock Documentation |
| Wednesday | - Test the document processing workflow with multiple file formats, including PDF, DOCX, PPTX, and image files.<br>- Evaluate OCR accuracy and AI processing results for each document type. | 24/06 | 24/06 | Amazon Textract Documentation |
| Thursday | - Perform Integration Testing across all system components.<br>- Validate error handling and exception management.<br>- Identify and resolve issues discovered during testing. | 25/06 | 25/06 | AWS Well-Architected Framework |
| Friday | - Present the project progress to the mentor.<br>- Summarize the AI processing results.<br>- Prepare the implementation plan for Semantic Search and AI-powered Document Chat in the next development phase. | 26/06 | 26/06 | Internal Project Documentation |

### Results Achieved in Week 10:

* Successfully completed the AI processing workflow based on an AWS Serverless architecture.

* Successfully implemented **AWS AppSync Subscriptions**, enabling real-time updates of document processing status without requiring users to refresh the application.

* Developed a real-time monitoring interface displaying the following processing stages:
  * Uploading
  * Processing
  * OCR Completed
  * AI Processing
  * Completed

* Optimized AWS Lambda functions, reducing execution time and improving the overall responsiveness of the document processing workflow.

* Refined prompts for **Amazon Bedrock Claude Haiku**, resulting in improved quality for:
  * AI-generated Document Summaries
  * Document Classification
  * Metadata Generation

* Successfully validated the AI processing workflow using various document formats, including PDF, DOCX, PPTX, and image files.

* Completed integration testing across **Amazon S3**, **AWS Lambda**, **Amazon Textract**, **Amazon Bedrock**, **AWS AppSync**, and **Amazon DynamoDB**, ensuring seamless communication between all system components.

* Successfully finalized the AI Processing Pipeline, providing a solid foundation for implementing **Semantic Search** and **AI-powered Document Chat** in the following development phase.