---
title: "Worklog Week 11"
date: 2026-06-27
weight: 11
chapter: false
pre: " <b> 1.11. </b> "
---

{{% notice warning %}}
⚠️ **Note:** The information below is for reference purposes only. Please **do not copy it word for word** into your internship report, including this warning.
{{% /notice %}}

### Weekly Objectives:

* Develop intelligent document retrieval using Semantic Search.
* Build an AI-powered chat system based on document content.
* Integrate Amazon Titan Embeddings and the Retrieval-Augmented Generation (RAG) architecture.
* Improve document retrieval accuracy and AI-assisted information extraction.

### Weekly Tasks:

| Day | Tasks | Start Date | Completion Date | Reference |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| Saturday | - Design document search functionality based on document name, document type, and upload date.<br>- Configure Global Secondary Indexes (GSIs) in Amazon DynamoDB to optimize query performance. | 27/06 | 27/06 | Amazon DynamoDB Documentation |
| Sunday | - Integrate Amazon Titan Embeddings to generate vector embeddings from document content.<br>- Store vector embeddings together with document metadata in Amazon DynamoDB. | 28/06 | 28/06 | Amazon Bedrock Documentation |
| Monday | - Develop the Semantic Search feature.<br>- Implement a Cosine Similarity algorithm to retrieve semantically related documents based on vector embeddings. | 29/06 | 29/06 | Amazon Bedrock Documentation |
| Tuesday | - Develop an AI-powered chat feature using the Retrieval-Augmented Generation (RAG) architecture.<br>- Retrieve the most relevant document content before sending requests to Amazon Bedrock Claude Haiku. | 30/06 | 30/06 | Amazon Bedrock Documentation |
| Wednesday | - Integrate the AI Chat interface into the application.<br>- Display AI-generated responses together with the referenced document excerpts used to generate the answers. | 01/07 | 01/07 | Angular Documentation |
| Thursday | - Test the Semantic Search and AI Chat features using different document types.<br>- Evaluate retrieval accuracy and AI-generated responses.<br>- Refine prompts and retrieval algorithms to improve answer quality. | 02/07 | 02/07 | AWS Best Practices |
| Friday | - Present the project progress to the mentor.<br>- Summarize the completed development tasks.<br>- Prepare the system for the final optimization phase and project demonstration. | 03/07 | 03/07 | Internal Project Documentation |

### Results Achieved in Week 11:

* Successfully implemented metadata-based document search using **Amazon DynamoDB Global Secondary Indexes (GSIs)**.

* Successfully integrated **Amazon Titan Embeddings** to generate vector embeddings from document content.

* Developed a **Semantic Search** mechanism that retrieves documents based on semantic similarity rather than simple keyword matching.

* Successfully implemented an **AI-powered chat system** using the **Retrieval-Augmented Generation (RAG)** architecture.

* Developed the complete RAG workflow, including:
  * Generating vector embeddings from user queries.
  * Retrieving the most relevant document segments from the vector database.
  * Providing the retrieved context to **Amazon Bedrock Claude Haiku** to generate accurate responses.

* Successfully integrated the AI Chat interface and displayed AI-generated responses together with the relevant document references used during answer generation.

* Completed functional testing of both Semantic Search and AI Chat using multiple document formats, while optimizing retrieval accuracy and AI response quality.

* Successfully completed the development of the core AI capabilities of the **AI-Powered Smart Document Assistant**, establishing a strong foundation for final system optimization, production readiness, and the final project demonstration.