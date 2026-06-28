# Customer Support Workflow — n8n

An automated customer support pipeline that monitors a Gmail inbox, classifies incoming emails, and uses an AI agent to draft and send personalised replies — all without human intervention.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Workflow Breakdown](#workflow-breakdown)
- [Prerequisites](#prerequisites)
- [Credentials Required](#credentials-required)
- [Setup Instructions](#setup-instructions)
- [Configuration Reference](#configuration-reference)
- [Agent Persona](#agent-persona)
- [Troubleshooting](#troubleshooting)

---

## Overview

This workflow automates the first-line customer support email process for **Tech Haven Solutions**. It continuously monitors a Gmail inbox, filters out non-support emails, and uses a RAG-powered AI agent to generate accurate, friendly replies grounded in a Pinecone knowledge base (FAQ & Policies).

```
[Gmail Inbox]
     │  (new email arrives)
     ▼
[Text Classifier]
     │
     ├── "Other"           → No Operation (ignored)
     │
     └── "Customer Support"
               │
               ▼
         [AI Agent]  ←── Pinecone Knowledge Base (FAQ)
               │
               ▼
         [Label Email]  →  [Send Reply]
```

---

## Architecture

```
Gmail Trigger  (polls every minute)
    │
    ▼
Text Classifier  ◄── OpenRouter Chat Model  (LLM for classification)
    │
    ├── [Other]  ──► No Operation (stop)
    │
    └── [Customer Support]
              │
              ▼
         AI Agent  ◄── OpenAI GPT-4o-mini  (LLM for reply generation)
              └──◄── Pinecone Vector Store  [retrieve-as-tool]
                          └──◄── Embeddings OpenAI
              │
              ▼
         Label  (adds Gmail label to original email)
              │
              ▼
         Send  (replies to the original email thread)
```

---

## Workflow Breakdown

| Step | Node | What it does |
|------|------|-------------|
| 1 | **Gmail Trigger** | Polls the inbox every minute; fires when a new email is detected; fetches full email data (`simple: false`) |
| 2 | **Text Classifier** | Uses an LLM to classify the email as either `Customer Support` or `Other` |
| 3a | **No Operation** | If classified as `Other`, the workflow stops — no reply is sent |
| 3b | **AI Agent** | If classified as `Customer Support`, the agent drafts a reply using the knowledge base tool |
| 4 | **Pinecone Vector Store** | Registered as a tool (`knowledgeBase`) the agent calls to look up relevant FAQs and policies |
| 5 | **Embeddings OpenAI** | Embeds the agent's query so it can be semantically matched against stored vectors |
| 6 | **Label** | Applies a Gmail label to the original email to mark it as handled |
| 7 | **Send** | Replies to the original email thread with the AI-generated response |

---

## Prerequisites

- **n8n** instance with `@n8n/n8n-nodes-langchain` support
- A **Gmail** account with OAuth2 access (the inbox to monitor)
- A **Pinecone** index pre-populated with FAQ and policy documents (see the RAG Pipeline workflow to ingest documents)
- An **OpenAI** API key (for GPT-4o-mini and embeddings)
- An **OpenRouter** API key (for the text classification LLM)

> This workflow is designed to work alongside the **RAG Pipeline & Chatbot** workflow. That workflow ingests documents into Pinecone; this workflow queries them.

---

## Credentials Required

| Credential | Used By | Notes |
|------------|---------|-------|
| Gmail OAuth2 | Gmail Trigger, Label, Send | Requires read + send + label permissions |
| OpenRouter API Key | OpenRouter Chat Model (Text Classifier) | Used for email classification only |
| OpenAI API Key | OpenAI Chat Model (GPT-4o-mini), Embeddings OpenAI | Used for reply generation and vector search |
| Pinecone API Key | Pinecone Vector Store | Index must already contain embedded FAQ/policy documents |

---

## Setup Instructions

### 1. Import the Workflow
- In n8n, go to **Workflows → Import from File**
- Select `2__Customer_Support_Workflow.json`

### 2. Configure Gmail
- Connect your Gmail OAuth2 credential to both the **Gmail Trigger** and the **Label** and **Send** nodes
- Ensure the OAuth2 scope includes: `gmail.readonly`, `gmail.send`, `gmail.labels`

### 3. Create a Gmail Label
- In Gmail, create a label (e.g. `Auto-Replied`) to mark handled emails
- Find the label's ID (visible in Gmail settings URL or via the Gmail API)
- Paste the label ID into the **Label** node's `labelIds` field

### 4. Connect Pinecone
- Add your Pinecone API credential
- Set the index name to `sample` (or your own index name)
- Set the namespace to `FAQ` (must match what was used during ingestion)
- Confirm documents have been ingested via the RAG Pipeline workflow first

### 5. Configure OpenAI
- Add your OpenAI API credential to both:
  - **OpenAI Chat Model** node (GPT-4o-mini for reply generation)
  - **Embeddings OpenAI** node (for vector search)

### 6. Configure OpenRouter
- Add your OpenRouter API credential to the **OpenRouter Chat Model** node
- This powers the Text Classifier step only

### 7. Activate the Workflow
- Toggle **Active** on
- The Gmail Trigger will begin polling every minute
- Send a test customer support email to the monitored inbox to verify end-to-end

---

## Configuration Reference

### Gmail Trigger

| Parameter | Value | Description |
|-----------|-------|-------------|
| `pollTimes` | Every minute | How often to check for new emails |
| `simple` | `false` | Returns full email data including headers, body, thread ID |
| `filters` | *(empty)* | No filters — all incoming emails are checked |

### Text Classifier

| Category | Description used for classification |
|----------|-------------------------------------|
| `Customer Support` | Emails asking about policies, products, or services |
| `Other` | Any email not related to customer support |

### AI Agent

| Parameter | Value |
|-----------|-------|
| `promptType` | `define` — passes the email body directly as the prompt |
| `text` | `$('Gmail Trigger').item.json.text` — the full email body |
| LLM | OpenAI GPT-4o-mini |

### Pinecone Vector Store (retrieve-as-tool)

| Parameter | Value |
|-----------|-------|
| `mode` | `retrieve-as-tool` |
| `toolName` | `knowledgeBase` |
| `toolDescription` | "Call this tool to access information about Policies and FAQ" |
| `pineconeIndex` | `sample` |
| `pineconeNamespace` | `FAQ` |

### Label Node

| Parameter | Description |
|-----------|-------------|
| `messageId` | Taken from the original Gmail Trigger email ID |
| `labelIds` | The Gmail label ID to apply to the processed email |

### Send Node

| Parameter | Value |
|-----------|-------|
| `operation` | `reply` — replies in the same thread |
| `message` | `$json.output` — the AI Agent's generated reply |
| `appendAttribution` | `false` — no "sent via n8n" footer |

---

## Agent Persona

The AI Agent is configured with the following system prompt:

**Role:** Customer support agent for *Tech Haven Solutions*

**Behaviour:**
- Searches the knowledge base before responding
- Writes friendly, emoji-inclusive email replies
- Signs off as **Mr. Helpful from Tech Haven Solutions**
- Outputs the email body only (no subject line or metadata)

To change the persona, update the `systemMessage` in the **AI Agent** node's options.

---

## Troubleshooting

**Emails are detected but no reply is sent**
- Check that the Text Classifier is correctly routing to the `Customer Support` branch (index 0), not the `Other` branch (index 1)
- Review the classifier's output in the execution log to see which category was assigned

**AI Agent replies are generic / not using the knowledge base**
- Confirm the Pinecone index and namespace match what was used during document ingestion
- Verify the Embeddings OpenAI model is the same one used to ingest the documents
- Check Pinecone's dashboard to confirm vectors exist in the `FAQ` namespace

**Gmail label is not being applied**
- Ensure the label ID in the **Label** node is correct (Gmail label IDs are long numeric strings, not the label name)
- Verify the Gmail OAuth2 scope includes label management permissions

**Workflow triggers but does nothing for new emails**
- n8n Gmail Trigger uses polling — it may miss emails if they arrive and are read before the next poll
- Consider switching to a Gmail Push Notification trigger for real-time processing

**Reply is sent but the formatting looks off**
- The Send node is configured for `text` email type; switch to `html` if you want formatted output
- Adjust the system prompt to guide the agent on desired formatting
