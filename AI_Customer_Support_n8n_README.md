# AI Customer Support Automation Platform

### n8n · Telegram · Supabase · Pinecone · AI · Analytics

> An advanced, modular customer-support automation system that accepts
> text and voice messages, retrieves trusted answers from structured
> data and a knowledge base, escalates technical issues, controls usage,
> tracks AI costs, and measures support performance.

------------------------------------------------------------------------

## 1. Project Overview

This project is an AI-powered customer support platform built with
**n8n**. It is designed to do more than generate chatbot replies: it
manages the full customer-support lifecycle, from receiving a message to
storing the conversation, retrieving an answer, escalating unresolved
issues, and measuring operational performance.

The initial customer-facing channel is **Telegram**. The architecture
should remain channel-agnostic so that additional channels or a
website/API interface can be connected later without redesigning the
core processing logic.

### Main objectives

-   Provide fast, consistent answers to customer questions.
-   Support both text and voice messages.
-   Convert voice messages into text before processing.
-   Check structured database answers before using RAG.
-   Retrieve relevant information from a document knowledge base.
-   Route technical problems to a dedicated support process.
-   Give customers interactive choices and clear next steps.
-   Enforce configurable per-customer usage limits.
-   Track AI token usage and estimated operating cost.
-   Provide a dashboard to evaluate chatbot quality and efficiency.
-   Keep a reliable history of customers, conversations, messages, and
    system events.

------------------------------------------------------------------------

## 2. Core Features

  -----------------------------------------------------------------------
  Feature                             Description
  ----------------------------------- -----------------------------------
  Text input                          Receive and process customer text
                                      messages.

  Voice input                         Receive voice notes/audio files and
                                      convert them to text.

  Input processing                    Normalize, validate, classify, and
                                      prepare incoming data.

  Customer identification             Find an existing customer or create
                                      a new customer record.

  Conversation management             Create or reuse the customer's
                                      active conversation.

  Database-first answers              Search approved answers in the
                                      database before invoking RAG.

  RAG search                          Retrieve relevant chunks from the
                                      knowledge base using vector search.

  AI response generation              Generate a grounded response using
                                      retrieved context and conversation
                                      history.

  Technical support                   Detect technical issues and route
                                      them through a dedicated workflow.

  Interactive buttons                 Let customers choose actions such
                                      as FAQ, technical support, human
                                      agent, or restart.

  Rate limiting                       Apply configurable message limits
                                      per customer and time window.

  AI cost tracking                    Record model, token usage, and
                                      estimated cost per request.

  Analytics dashboard                 Measure answer quality, response
                                      speed, usage, cost, and escalation
                                      rates.

  Reliability                         Handle errors, retries, duplicate
                                      updates, and failed integrations.
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 3. High-Level Architecture

``` mermaid
flowchart TD
    A[Telegram Trigger / Webhook] --> B[Normalize Input]
    B --> C[Validate Input]
    C --> D[Idempotency Check]
    D --> E[Rate Limit Check]
    E --> F[Identify or Create Customer]
    F --> G[Get or Create Conversation]
    G --> H[Save Customer Message]
    H --> I{Input Type}
    I -->|Text| J[Prepare Text]
    I -->|Voice| K[Download Audio]
    K --> L[Speech-to-Text]
    L --> J
    J --> M[Intent Classification]
    M --> N{Technical Issue?}
    N -->|Yes| O[Technical Support Workflow]
    N -->|No| P[Search Approved Answers]
    P --> Q{Answer Found?}
    Q -->|Yes| R[Prepare Database Answer]
    Q -->|No| S[RAG Retrieval]
    S --> T[Build AI Context]
    T --> U[AI Response]
    R --> V[Validate Response]
    O --> V
    U --> V
    V --> W[Save Assistant Message]
    W --> X[Send Telegram Response]
    X --> Y[Record Metrics and Cost]
```

The diagram is a conceptual overview. The implementation should be split
into smaller workflows so that each module can be tested, reused, and
maintained independently.

------------------------------------------------------------------------

## 4. Message Processing Lifecycle

### 4.1 Receive and normalize

Accept updates from Telegram or a test webhook. Convert every incoming
event into one consistent internal structure.

Suggested canonical request:

``` json
{
  "request_id": "uuid",
  "channel": "telegram",
  "source_message_id": "telegram-message-id",
  "customer": {
    "customer_id": "telegram-user-id",
    "name": "Customer Name"
  },
  "message": {
    "type": "text",
    "text": "Customer message",
    "file_id": null,
    "transcript": null
  },
  "conversation_id": null,
  "received_at": "ISO-8601 timestamp"
}
```

For voice messages, `message.type` is `voice`; the audio file is
downloaded and transcribed, and the resulting text is stored in
`message.transcript` or a normalized text field. Keep the original
message metadata for traceability.

### 4.2 Validate and protect

Validation should include:

-   Reject empty messages and unsupported message types.
-   Enforce a configurable maximum input length.
-   Validate required Telegram/customer identifiers.
-   Check file type and file-size limits for voice messages.
-   Detect duplicate updates using the channel and source message ID.
-   Sanitize data before passing it to downstream nodes.
-   Treat customer content and retrieved documents as untrusted input.

### 4.3 Identify the customer and conversation

Use a stable, channel-aware customer identifier. Search for the customer
first; create a record only when no matching customer exists. Then
retrieve or create the appropriate active conversation.

Do not lose the original message, customer ID, request ID, or
conversation ID when database nodes replace their input with query
results. Merge the required context back into the workflow explicitly.

### 4.4 Store messages

Store both customer messages and assistant replies. Keep direction,
channel, source message ID, timestamps, and useful metadata. Use
idempotency controls to prevent the same Telegram update from being
saved or answered twice.

------------------------------------------------------------------------

## 5. Voice Message Processing

Voice support should be a separate, testable sub-workflow.

1.  Detect that the incoming Telegram update contains a voice message or
    audio attachment.
2.  Retrieve the Telegram file metadata.
3.  Download the audio binary.
4.  Validate file type, size, and duration against configured limits.
5.  Send the audio to a speech-to-text service.
6.  Store the transcript and transcription metadata.
7.  Pass the transcript through the same validation and
    intent-classification path used for text.
8.  Continue through database lookup, RAG, or technical support.
9.  Optionally send a short confirmation or transcript to the customer.

**Important:** Speech-to-text and AI response generation are separate
operations. Track their usage and costs independently where provider
pricing allows it. Handle transcription failures with a clear customer
message and a text-input fallback.

------------------------------------------------------------------------

## 6. Answer Retrieval: Database First, RAG Second

Use a retrieval sequence that avoids unnecessary AI calls.

### Stage 1 --- Structured database / approved answer lookup

Search the approved FAQ or answer table first.

Recommended order:

1.  Exact match using a normalized question or intent key.
2.  Similarity match against approved questions, if configured.
3.  Apply a confidence threshold.
4.  Return the approved answer when the match is reliable.

A database/cache hit should bypass vector retrieval and answer
generation when the stored answer is complete and approved.

### Stage 2 --- RAG search

If no reliable approved answer is found:

1.  Create an embedding for the normalized question.
2.  Search the Pinecone index.
3.  Apply metadata filters where appropriate, such as document,
    category, language, or tenant.
4.  Retrieve the most relevant chunks.
5.  Optionally rerank the retrieved chunks.
6.  Check the relevance score and minimum evidence threshold.
7.  Pass only relevant context to the AI response workflow.

### Stage 3 --- Grounded AI response

The AI must:

-   Answer using the supplied knowledge-base context and approved
    conversation data.
-   Avoid inventing policies, prices, specifications, or troubleshooting
    steps.
-   Say when the available information is insufficient.
-   Ask a focused clarification question when required.
-   Provide source/document references when available.
-   Escalate unresolved or low-confidence cases instead of guessing.

If retrieval fails or returns weak evidence, use a safe fallback or
technical/human escalation path.

------------------------------------------------------------------------

## 7. Technical Support Workflow

Technical support should be a dedicated route, not just a different
prompt.

### Responsibilities

-   Detect technical-support intent.
-   Collect the relevant product/service, error message, and
    troubleshooting details.
-   Ask for missing information through guided questions.
-   Search technical documentation and known-issue records.
-   Provide safe, documented troubleshooting steps.
-   Create a support ticket when the issue remains unresolved.
-   Assign priority and status.
-   Notify a human support agent when required.
-   Link the ticket to the customer and conversation.

### Suggested ticket fields

  Field               Purpose
  ------------------- -------------------------------------------------------
  `ticket_id`         Unique support ticket identifier
  `customer_id`       Customer who reported the issue
  `conversation_id`   Related conversation
  `category`          Issue category
  `description`       Problem summary
  `priority`          Low, normal, high, or urgent
  `status`            Open, in_progress, waiting_customer, resolved, closed
  `assigned_to`       Responsible support agent
  `created_at`        Ticket creation time
  `resolved_at`       Resolution time, when applicable

------------------------------------------------------------------------

## 8. Interactive Customer Experience

Use Telegram inline keyboards or equivalent channel-specific interactive
controls.

Suggested main menu:

-   **Ask a question**
-   **Technical support**
-   **Track my ticket**
-   **Talk to a human**
-   **Main menu**

Suggested technical-support actions:

-   **Problem solved**
-   **Still having the problem**
-   **Create a support ticket**
-   **Back to main menu**

Every button callback must be validated and linked to the correct
customer and conversation. Do not trust callback data as authorization.
Handle expired, repeated, and out-of-order button clicks gracefully.

------------------------------------------------------------------------

## 9. Customer Message Limits and Abuse Protection

Implement configurable usage limits per customer. The limits should be
enforced before expensive AI operations.

Example policy (values are configuration placeholders, not final
business rules):

-   Maximum messages per customer per minute.
-   Maximum messages per customer per day.
-   Maximum voice messages per day.
-   Maximum audio duration or file size.
-   Optional separate limits for free and paid customer tiers.

When a customer reaches a limit:

1.  Do not execute embedding, RAG, or AI generation unnecessarily.
2.  Return a clear message explaining that the limit has been reached.
3.  Provide the reset time or next available usage window when known.
4.  Record the rate-limit event.
5.  Allow authorized support/admin users to bypass limits only through
    explicit rules.

Use atomic database operations or a reliable counter mechanism to avoid
race conditions when several messages arrive simultaneously.

------------------------------------------------------------------------

## 10. AI Usage and Cost Tracking

Record usage for every billable AI operation.

### Suggested usage fields

  Field               Description
  ------------------- ----------------------------------------------------
  `request_id`        Request associated with the operation
  `customer_id`       Customer, when applicable
  `conversation_id`   Related conversation
  `provider`          AI provider
  `model`             Model name/version
  `operation`         Transcription, embedding, reranking, or generation
  `input_tokens`      Input token count, when supplied
  `output_tokens`     Output token count, when supplied
  `total_tokens`      Total tokens, when supplied
  `audio_seconds`     Audio duration, when relevant
  `estimated_cost`    Cost calculated from the applicable pricing
  `currency`          Currency used for the estimate
  `created_at`        Operation timestamp

### Cost calculation

For token-priced generation:

`Estimated cost = (Input tokens × Input price + Output tokens × Output price) / 1,000,000`

Use the provider's current price per million tokens and the correct
model-specific rates. Do not hardcode a price permanently in workflow
nodes. Keep pricing in configuration or a database table with an
effective date.

For operations priced by audio duration, requests, or other units, use
the provider's billing unit instead of forcing a token formula.

Distinguish between:

-   Actual usage returned by the provider.
-   Estimated cost calculated by the workflow.
-   Final billed cost from the provider account.

If a provider does not return usage data, mark the cost as estimated or
unavailable rather than inventing token counts.

------------------------------------------------------------------------

## 11. Analytics Dashboard --- Chatbot Performance

Build a dashboard using **Metabase** connected to the reporting tables
in Supabase/PostgreSQL.

### Operational KPIs

-   Total incoming messages.
-   Unique active customers.
-   Total conversations.
-   Average and median response time.
-   Successful response rate.
-   Failed request rate.
-   Unanswered question count.
-   Human escalation count and rate.
-   Technical tickets created and resolved.

### Knowledge and AI KPIs

-   Database answer/cache hit rate.
-   RAG usage rate.
-   RAG retrieval success rate.
-   Low-confidence answer rate.
-   AI fallback rate.
-   Voice messages and transcription success rate.
-   Average input/output tokens per AI request.
-   AI cost per day, week, and month.
-   Average AI cost per conversation and per customer.

### Customer experience KPIs

-   Customer feedback score, if feedback collection is enabled.
-   Resolved-on-first-contact rate.
-   Repeated question rate.
-   Average messages per resolved conversation.
-   Ticket resolution time.

### Suggested dashboard sections

1.  **Overview:** total conversations, active customers, response time,
    resolution rate.
2.  **AI Performance:** database hits, RAG results, confidence,
    fallback, and errors.
3.  **Cost Monitoring:** spend by model, operation, customer, and date.
4.  **Voice Analytics:** audio volume, transcription success, and
    transcription cost.
5.  **Support Operations:** tickets, priorities, escalation, and
    resolution time.
6.  **Customer Activity:** usage distribution and rate-limit events.

Define each KPI precisely before building charts. For example, decide
whether response time is measured from message receipt to the first
assistant reply, and whether failed or escalated conversations count as
resolved.

------------------------------------------------------------------------

## 12. Suggested Database Design

Use Supabase/PostgreSQL as the operational database. Keep the schema
channel-agnostic.

  Table                 Responsibility
  --------------------- ----------------------------------------------
  `customers`           Customer identity and profile
  `conversations`       Conversation lifecycle and status
  `messages`            Incoming and outgoing messages
  `faq_answers`         Approved questions and answers
  `support_tickets`     Technical support cases
  `usage_limits`        Customer or plan-level limit configuration
  `usage_counters`      Message/audio counters by time window
  `ai_usage_logs`       Provider/model usage and cost records
  `retrieval_logs`      Database lookup and RAG retrieval outcomes
  `workflow_errors`     Workflow failures and diagnostic metadata
  `customer_feedback`   Optional satisfaction feedback
  `system_settings`     Configurable thresholds and feature settings

### Data design principles

-   Use UUID primary keys where appropriate.
-   Add foreign keys and indexes for frequently queried relationships.
-   Use unique constraints for channel identity and source-message
    idempotency.
-   Store timestamps consistently.
-   Keep customer data separate from message content.
-   Use Row Level Security (RLS) and least-privilege access.
-   Never expose service-role keys or provider secrets to customers.
-   Avoid storing unnecessary sensitive data in logs.
-   Define retention and deletion rules for audio files and transcripts.

The exact columns, constraints, indexes, and RLS policies should be
documented in the database migration files.

------------------------------------------------------------------------

## 13. n8n Workflow Structure

Build the system as modular workflows. Suggested names:

  -----------------------------------------------------------------------
  Workflow                            Responsibility
  ----------------------------------- -----------------------------------
  `00 - Config`                       Shared settings, model names,
                                      thresholds, and constants

  `01 - Telegram Input`               Receive Telegram text, voice, and
                                      callbacks

  `02 - Normalize Input`              Build the canonical request
                                      structure

  `03 - Validate Input`               Validate message, identifiers, and
                                      attachments

  `04 - Idempotency`                  Prevent duplicate processing

  `05 - Rate Limit`                   Enforce customer usage limits

  `06 - Customer Management`          Find or create customer

  `07 - Conversation Management`      Find or create active conversation

  `08 - Message Persistence`          Store incoming and outgoing
                                      messages

  `09 - Voice Transcription`          Download audio and create
                                      transcript

  `10 - Intent Classification`        Detect FAQ, technical support,
                                      ticket, and other intents

  `11 - Database Answer Lookup`       Search approved answers

  `12 - RAG Retrieval`                Search Pinecone and optionally
                                      rerank results

  `13 - AI Response Generation`       Generate grounded responses

  `14 - Technical Support`            Troubleshoot and manage tickets

  `15 - Interactive Response`         Build Telegram messages and buttons

  `16 - AI Usage & Cost Tracking`     Record usage and calculate
                                      estimated costs

  `17 - Analytics Events`             Record performance events

  `18 - Error Handler`                Centralized error handling and
                                      alerts
  -----------------------------------------------------------------------

These are logical modules. Some can be implemented as sub-workflows,
while others may be nodes within a parent workflow. Keep workflow
boundaries practical and avoid splitting every single node into a
separate workflow.

------------------------------------------------------------------------

## 14. Reliability, Security, and Observability

### Reliability

-   Configure retries only for transient failures.
-   Use bounded retries and sensible delays.
-   Add timeouts to external API calls.
-   Handle provider rate limits and temporary outages.
-   Use idempotency for inbound events and ticket creation.
-   Keep failed requests traceable through `request_id`.
-   Add a dead-letter or failed-execution review process for events that
    cannot be recovered automatically.

### Security

-   Store credentials in n8n Credentials or a secure secrets manager.
-   Apply Supabase RLS and least-privilege database roles.
-   Validate Telegram callback data and webhook requests.
-   Limit uploaded file types and sizes.
-   Protect against prompt injection in customer messages and retrieved
    documents.
-   Treat retrieved text as reference data, never as system
    instructions.
-   Redact secrets and sensitive personal data from logs.

### Observability

Every request should be traceable across workflows using a shared
`request_id`. Log important stage outcomes, durations, errors, retrieval
scores, and usage metadata without logging secrets.

------------------------------------------------------------------------

## 15. Testing Strategy

Test each module independently before running end-to-end tests.

  -----------------------------------------------------------------------
  Test case                           Expected result
  ----------------------------------- -----------------------------------
  New customer sends text             Customer and conversation are
                                      created; message is stored and
                                      processed.

  Existing customer sends text        Existing customer is reused.

  Customer sends voice                Audio is transcribed and processed
                                      through the normal pipeline.

  Approved answer exists              Database answer is returned without
                                      unnecessary RAG/AI calls.

  No database answer exists           RAG is executed.

  Relevant RAG context exists         AI produces a grounded answer.

  No relevant context exists          Safe fallback or escalation is
                                      returned.

  Technical issue detected            Technical-support workflow is
                                      started.

  Customer clicks a button            Correct action is executed for the
                                      correct conversation.

  Customer exceeds limit              Request is blocked before expensive
                                      AI processing.

  Duplicate Telegram update arrives   It is not processed twice.

  AI provider fails                   Error is logged and a safe fallback
                                      is used.

  Cost usage is returned              Usage and estimated cost are
                                      recorded.

  Dashboard is refreshed              KPIs reflect the underlying event
                                      data.
  -----------------------------------------------------------------------

Also test concurrent messages from the same customer, expired callbacks,
malformed audio, empty transcripts, database outages, Pinecone outages,
and partial workflow failures.

------------------------------------------------------------------------

## 16. Implementation Roadmap

### Phase 1 --- Core messaging

-   Telegram text input.
-   Canonical data structure.
-   Validation and idempotency.
-   Customer and conversation management.
-   Message persistence.

### Phase 2 --- Retrieval

-   Approved-answer database lookup.
-   Pinecone RAG retrieval.
-   Context preparation.
-   Grounded AI response.
-   Safe fallback for low-confidence results.

### Phase 3 --- Voice

-   Telegram audio download.
-   Speech-to-text.
-   Transcript storage.
-   Voice error handling and usage tracking.

### Phase 4 --- Support operations

-   Intent classification.
-   Technical-support workflow.
-   Ticket creation and status tracking.
-   Interactive buttons and human escalation.

### Phase 5 --- Controls and cost

-   Per-customer rate limits.
-   Usage counters.
-   AI token/audio usage logging.
-   Model-specific cost calculation.

### Phase 6 --- Analytics and hardening

-   Metabase dashboard.
-   KPI definitions and SQL views.
-   Error monitoring and alerts.
-   Security/RLS review.
-   End-to-end and load testing.
-   Documentation and portfolio demo.

------------------------------------------------------------------------

## 17. Definition of Done

The project is ready for portfolio demonstration when:

-   Text and voice messages are both supported.
-   Voice is transcribed and follows the same processing pipeline as
    text.
-   Customer and conversation records are correctly reused.
-   Duplicate messages do not create duplicate replies.
-   Approved database answers are checked before RAG.
-   RAG answers are grounded in retrieved knowledge.
-   Unsupported questions are handled safely.
-   Technical issues can create and track support tickets.
-   Interactive buttons work and are validated.
-   Customer limits are enforced reliably.
-   AI usage and estimated cost are recorded.
-   Dashboard KPIs are calculated from stored events.
-   Errors are traceable and recoverable where possible.
-   Credentials, RLS, and sensitive data are handled securely.

------------------------------------------------------------------------

## 18. Technology Stack

  -----------------------------------------------------------------------
  Component                           Technology
  ----------------------------------- -----------------------------------
  Workflow automation                 n8n

  Initial messaging channel           Telegram Bot API

  Operational database                Supabase / PostgreSQL

  Vector database                     Pinecone

  Embeddings                          Configurable embedding provider

  Speech-to-text                      Configurable transcription provider

  AI response generation              Configurable LLM provider/model

  Optional reranking                  Configurable reranking provider

  Analytics dashboard                 Metabase

  Knowledge source                    Documents such as PDFs, with an
                                      ingestion workflow
  -----------------------------------------------------------------------

Provider and model choices should be configurable rather than hardwired
into business logic.

------------------------------------------------------------------------

## 19. Portfolio Description

**AI Customer Support Automation Platform** is a modular n8n-based
system that automates customer support across text and voice channels.
It combines database-first answer retrieval, Pinecone-powered RAG,
AI-generated grounded responses, technical ticket escalation,
interactive Telegram controls, per-customer rate limiting, AI usage/cost
monitoring, and a Metabase analytics dashboard.

The project demonstrates practical skills in workflow architecture, API
integration, database design, vector search, speech processing, AI
orchestration, reliability engineering, security, and operational
analytics.

------------------------------------------------------------------------

## 20. Project Decisions and Open Configuration

The following values should be finalized during implementation rather
than guessed:

-   Maximum messages per minute/day.
-   Maximum voice duration and file size.
-   Supported languages.
-   Approved-answer matching threshold.
-   RAG minimum relevance threshold and top-K.
-   Human escalation conditions and notification channel.
-   AI provider, model, and current pricing.
-   Audio transcription provider and pricing.
-   Data retention period.
-   Customer feedback method.
-   Dashboard refresh schedule.

Keep these settings centralized so they can be changed without editing
many workflow nodes.

------------------------------------------------------------------------

**Document status:** Initial project specification\
**Project type:** Advanced n8n automation / AI customer support\
**Initial channel:** Telegram\
**Core data and retrieval:** Supabase + Pinecone\
**Analytics:** Metabase
