---
title: "Workshop"
date: 2024-01-01
weight: 5
chapter: false
pre: " <b> 5. </b> "
---

# Smart Document Assistant

#### Overview

**Smart Document Assistant** is a serverless document processing system built on AWS. Users upload files (PDF, Word, PowerPoint, images) through an Angular frontend, and the system automatically extracts text via OCR and analyzes content with AI to generate summaries and classify documents by topic.

The architecture uses two Lambda functions connected by an SNS topic, with DynamoDB for metadata storage, AppSync for the GraphQL API, and Cognito for user authentication.

#### Architecture Diagram

![Architecture Diagram](/images/2-Proposal/architecture.png)

#### Content

1. [Workshop Overview](5.1-Workshop-overview/)
2. [Prerequisites & Infrastructure Setup](5.2-Prerequiste/)
3. [Upload Pipeline — Lambda A & Textract](5.3-Upload-pipeline/)
