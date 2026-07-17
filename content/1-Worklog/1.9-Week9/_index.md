---
title: "Worklog Week 9"
date: 2026-06-13
weight: 9
chapter: false
pre: " <b> 1.9. </b> "
---

{{% notice warning %}}
⚠️ **Note:** The information below is for reference purposes only. Please **do not copy it word for word** into your internship report, including this warning.
{{% /notice %}}

### Weekly Objectives:

* Integrate an automated document processing workflow using AWS Serverless services.
* Develop the OCR pipeline for extracting text from uploaded documents.
* Integrate Amazon Bedrock for document summarization and classification.
* Store AI processing results in the database for further retrieval.

### Weekly Tasks:

| Day | Tasks | Start Date | Completion Date | Reference |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| Saturday | - Configure Amazon S3 Event Triggers.<br>- Develop an AWS Lambda function to automatically process documents uploaded to Amazon S3.<br>- Verify the automatic event-driven workflow. | 13/06 | 13/06 | AWS Lambda Documentation |
| Sunday | - Integrate Amazon Textract for Optical Character Recognition (OCR) of PDF documents and images.<br>- Configure asynchronous document processing using Amazon SNS callbacks. | 14/06 | 14/06 | Amazon Textract Documentation |
| Monday | - Develop Lambda Layers for processing DOCX and PPTX files.<br>- Integrate document parsing libraries to extract and normalize document content before AI processing. | 15/06 | 15/06 | AWS Lambda Documentation |
| Tuesday | - Integrate Amazon Bedrock Claude Haiku.<br>- Design prompts for document summarization and document classification.<br>- Validate the quality of AI-generated responses. | 16/06 | 16/06 | Amazon Bedrock Documentation |
| Wednesday | - Design the metadata storage structure in Amazon DynamoDB.<br>- Store OCR results, AI-generated summaries, document classifications, and processing metadata. | 17/06 | 17/06 | Amazon DynamoDB Documentation |
| Thursday | - Implement an AI quota management mechanism to validate user limits before invoking Amazon Bedrock.<br>- Optimize the AI processing workflow to reduce operational costs. | 18/06 | 18/06 | AWS Best Practices |
| Friday | - Perform end-to-end testing of the document processing workflow from document upload to AI processing.<br>- Present the project progress to the mentor.<br>- Collect feedback and refine the AI processing architecture where necessary. | 19/06 | 19/06 | Internal Project Documentation |

### Results Achieved in Week 9:

* Successfully implemented an automated document processing workflow based on an AWS Serverless architecture.

* Configured Amazon S3 Event Triggers to automatically invoke AWS Lambda whenever a user uploads a document.

* Successfully integrated Amazon Textract to perform asynchronous Optical Character Recognition (OCR) for PDF documents and image files.

* Developed Lambda Layers capable of extracting content from DOCX and PPTX files before submitting the extracted text to the AI processing pipeline.

* Successfully integrated Amazon Bedrock Claude Haiku to perform:
  * Document Summarization
  * Document Classification
  * AI-based Content Processing

* Designed and implemented a metadata storage structure in Amazon DynamoDB, including:
  * OCR Results
  * AI-generated Summaries
  * Document Classifications
  * Processing Status

* Implemented an AI quota control mechanism to prevent excessive AI usage and optimize Amazon Bedrock operational costs.

* Successfully completed end-to-end testing of the AI processing pipeline, from document upload through OCR, AI processing, and metadata storage, providing a solid foundation for implementing real-time notifications and AI-powered document chat in the next development phase.