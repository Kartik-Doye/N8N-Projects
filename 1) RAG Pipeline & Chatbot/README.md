# RAG Pipeline & Chatbot — n8n Workflow

A two-part n8n automation that combines a **document ingestion pipeline** with a **retrieval-augmented generation (RAG) chatbot**. Files uploaded to Google Drive are automatically chunked, embedded, and stored in Pinecone. An AI agent then uses that vector store as a live knowledge base to answer chat queries.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Workflow Breakdown](#workflow-breakdown)
  - [Part 1 — Document Ingestion Pipeline](#part-1--document-ingestion-pipeline)
  - [Part 2 — RAG Chatbot](#part-2--rag-chatbot)
- [Prerequisites](#prerequisites)
- [Credentials Required](#credentials-required)
- [Setup Instructions](#setup-instructions)
- [Configuration Reference](#configuration-reference)
- [Workflow Variants](#workflow-variants)
- [Troubleshooting](#troubleshooting)

---

## Overview

This workflow solves a common use case: **keeping a chatbot's knowledge base automatically up to date** without manual reindexing. Drop a file into a designated Google Drive folder and the pipeline handles the rest — parsing, splitting, embedding, and storing — so the chatbot can immediately query the new content.

```
[Google Drive Folder]
       │  (new file uploaded)
       ▼
 ┌─────────────────────────────────────┐
 │   INGESTION PIPELINE                │
 │  Download → Split → Embed → Store  │
 └─────────────────┬───────────────────┘
                   │ (vectors in Pinecone)
                   ▼
 ┌─────────────────────────────────────┐
 │   RAG CHATBOT                       │
 │  Chat → AI Agent → Retrieve → Reply │
 └─────────────────────────────────────┘
```

---

## Architecture

```
INGESTION PIPELINE
──────────────────
Google Drive Trigger
    │  (polls every minute; fires on fileCreated)
    ▼
Download File (Google Drive node)
    │  (binary file data)
    ▼
Pinecone Vector Store  [INSERT mode]
    ├──◄── Embeddings Model  (OpenAI / Google Gemini)
    └──◄── Default Data Loader
               └──◄── Recursive Character Text Splitter

RAG CHATBOT
───────────
Chat Trigger  (webhook — receives user message)
    │
    ▼
AI Agent
    ├──◄── LLM  (OpenRouter → Claude)
    └──◄── Pinecone Vector Store  [RETRIEVE-AS-TOOL mode]
               └──◄── Embeddings Model  (same model as ingestion)
```

---

## Workflow Breakdown

### Part 1 — Document Ingestion Pipeline

| Step | Node | What it does |
|------|------|-------------|
| 1 | **Google Drive Trigger** | Polls the target folder every minute; fires when a new file is created |
| 2 | **Download File** | Downloads the newly created file as binary data using its Drive ID (`$json.id`) |
| 3 | **Default Data Loader** | Reads binary file content and prepares it for splitting (supports PDF, DOCX, TXT, and more) |
| 4 | **Recursive Character Text Splitter** | Breaks the document into overlapping chunks to preserve context across splits |
| 5 | **Embeddings Model** | Converts each text chunk into a vector using the configured embedding model |
| 6 | **Pinecone Vector Store** *(insert)* | Upserts all vectors into the designated Pinecone index and namespace |

### Part 2 — RAG Chatbot

| Step | Node | What it does |
|------|------|-------------|
| 1 | **Chat Trigger** | Exposes a webhook endpoint that receives incoming chat messages |
| 2 | **AI Agent** | Orchestrates the conversation; decides when to call the knowledge base tool |
| 3 | **OpenRouter Chat Model** | The LLM powering the agent (Claude via OpenRouter) |
| 4 | **Pinecone Vector Store** *(retrieve-as-tool)* | Registered as a tool the agent can call to search the knowledge base |
| 5 | **Embeddings Model** | Embeds the user's query so it can be matched against stored vectors |

The agent autonomously decides when the knowledge base is relevant and retrieves the most semantically similar chunks before formulating its answer.

---

## Prerequisites

- **n8n** instance (self-hosted or cloud), version compatible with `@n8n/n8n-nodes-langchain`
- A **Google Drive** account with OAuth2 access
- A **Pinecone** account with an index created (see [Configuration Reference](#configuration-reference))
- An **OpenRouter** account for Claude LLM access
- An **OpenAI** API key **or** a **Google Gemini** API key for embeddings (depending on which variant you use — see [Workflow Variants](#workflow-variants))

---

## Credentials Required

| Credential | Used By | Notes |
|------------|---------|-------|
| Google Drive OAuth2 | Drive Trigger, Download File | Requires Drive read permissions on the watched folder |
| Pinecone API Key | Both Pinecone Vector Store nodes | Index must already exist before first run |
| OpenRouter API Key | OpenRouter Chat Model | Used to call Claude via OpenRouter |
| OpenAI API Key *(Variant A)* | Embeddings OpenAI nodes | Must use the **same model** in both insert and retrieve nodes |
| Google Gemini (PaLM) API Key *(Variant B)* | Embeddings Google Gemini nodes | Must use compatible model in both insert and retrieve nodes |

> ⚠️ **Important:** The embedding model used during **ingestion** and during **retrieval** must be identical. Mismatched models will produce vectors in different spaces and break semantic search.

---

## Setup Instructions

### 1. Import the Workflow
- In n8n, go to **Workflows → Import from File**
- Select the `.json` file for your preferred variant

### 2. Configure Google Drive
- Connect your Google Drive OAuth2 credential
- In the **Google Drive Trigger** node, select the folder you want to watch for new files
- Set the event to `fileCreated`

### 3. Set Up Pinecone
- Log in to [Pinecone](https://www.pinecone.io/) and create an index:
  - **Dimensions** must match your embedding model output (e.g. 1536 for `text-embedding-ada-002`, 768 for Gemini Embedding)
  - **Metric:** Cosine
- Create a namespace (e.g. `FAQ` or `n8n learn`)
- Enter the index name and namespace in **both** Pinecone Vector Store nodes

### 4. Configure the Embedding Model
- Add your OpenAI or Google Gemini API credential
- Ensure **both** Embeddings nodes (one in the ingestion pipeline, one in the chatbot pipeline) use the **same model**

### 5. Configure OpenRouter
- Add your OpenRouter API credential to the **OpenRouter Chat Model** node
- Choose a Claude model (e.g. `anthropic/claude-sonnet-4.5`)

### 6. Activate the Workflow
- Click **Activate** (top-right toggle)
- The ingestion pipeline will begin polling Google Drive immediately
- The chatbot webhook is live once activated — find the URL in the **Chat Trigger** node

### 7. Test
- Upload a document to your watched Google Drive folder
- Wait ~1 minute for the ingestion pipeline to process it
- Open the Chat Trigger's test URL and ask a question about the document's content

---

## Configuration Reference

### Google Drive Trigger

| Parameter | Description |
|-----------|-------------|
| `pollTimes` | How often to check for new files (default: every minute) |
| `triggerOn` | Set to `specificFolder` to watch a single folder |
| `folderToWatch` | The Drive folder ID or selected folder |
| `event` | `fileCreated` — fires only when a new file appears |
| `fileType` | Optionally filter by file type (e.g. PDF only, or `all`) |

### Pinecone Vector Store (Insert)

| Parameter | Description |
|-----------|-------------|
| `mode` | `insert` — writes new vectors |
| `pineconeIndex` | Name of your Pinecone index |
| `pineconeNamespace` | Logical partition within the index (e.g. `FAQ`) |

### Pinecone Vector Store (Retrieve-as-Tool)

| Parameter | Description |
|-----------|-------------|
| `mode` | `retrieve-as-tool` — exposes retrieval as an agent tool |
| `toolName` | Name the agent uses to reference this tool (e.g. `knowledgeBase`) |
| `toolDescription` | Natural language description telling the agent when to call this tool |
| `pineconeIndex` | Must match the insert node's index |
| `pineconeNamespace` | Must match the insert node's namespace |

### Recursive Character Text Splitter

| Parameter | Default | Description |
|-----------|---------|-------------|
| `chunkSize` | 1000 | Max characters per chunk |
| `chunkOverlap` | 200 | Characters of overlap between chunks (preserves context) |

---

## Workflow Variants

Two versions of this workflow are included, differing mainly in the embedding model and Pinecone index:

| Feature | **Variant A** (`1__RAG_Pipeline___Chatbot.json`) | **Variant B** (`1-RAG_Pipeline___Chatbot.json`) |
|---------|--------------------------------------------------|--------------------------------------------------|
| **Embedding Model** | OpenAI (`text-embedding-ada-002`) | Google Gemini (`gemini-embedding-2`) |
| **Pinecone Index** | `sample` | `n8n-learning` |
| **Namespace** | `FAQ` | `n8n learn` |
| **Drive Folder** | FAQ document folder | n8n resources folder |
| **LLM** | `anthropic/claude-3.5-sonnet` | `anthropic/claude-sonnet-4.5` |
| **Agent Version** | `1.8` | `3.1` |
| **Knowledge Base Tool Description** | "Call this tool to access the policy and FAQ database" | "call this tool for the access and for learn" |

> Choose **Variant A** if you already have an OpenAI key and want a battle-tested embedding model.  
> Choose **Variant B** if you prefer Google Gemini for embeddings and want the newer Claude Sonnet model.

---

## Troubleshooting

**Files are uploaded but not appearing in Pinecone**
- Check that the Google Drive Trigger credential has read permissions on the watched folder
- Confirm the Pinecone index exists and the API key is valid
- Verify the embedding model dimensions match the Pinecone index dimensions

**Chatbot gives generic answers, not using the knowledge base**
- Ensure the Pinecone namespace in the retrieve node matches the one used during insert
- Confirm the embedding model is identical in both the ingestion and retrieval nodes
- Improve the `toolDescription` in the retrieve node to be more specific — the agent uses this to decide when to call the tool

**Embedding model mismatch error**
- This occurs when inserting with one model (e.g. OpenAI) and retrieving with another (e.g. Gemini)
- Delete the existing vectors, fix the model in both nodes to be the same, and re-ingest

**Workflow polls but doesn't trigger**
- n8n polls Drive on its schedule, but the trigger only fires on `fileCreated` — existing files won't re-trigger
- To re-process a file, delete it and re-upload it to the folder

**Chat webhook is not responding**
- The workflow must be **activated** (not just saved) for the webhook to be live
- Find the webhook URL in the **When chat message received** node's settings
