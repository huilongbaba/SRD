# BACKEND -- Overview

> Generated: 2026-04-02

## 1. Objective

Provide the API gateway layer and microservice backend for the Memoket mobile application -- an AI-powered audio recording assistant. The backend orchestrates between mobile clients, an external AI service, cloud storage (AWS S3/CloudFront), and third-party integrations (Google Calendar, Firebase, Notion, Slack).

## 2. Scope

| Area | Included |
|------|----------|
| **User lifecycle** | Registration (email / Google / Apple), login, JWT auth, profile CRUD, settings, email verification, account deletion |
| **Audio recordings** | Metadata CRUD, S3 presigned URL upload pipeline, folder organisation, sharing |
| **AI proxy** | Transparent proxy to external AI service for transcription, summarisation, AI agents, chat (SSE streaming), insight analytics |
| **Task management** | AI-identified todos from recordings, manual tasks, Google Calendar two-way sync, webhook handling |
| **Export** | Async PDF/Word document generation via Redis Stream worker |
| **Notifications** | Real-time SSE push + Firebase Cloud Messaging (FCM) |
| **Integrations** | OAuth flows for Slack, Google Calendar, Notion; send-to-integration |
| **MCP** | Model Context Protocol server for tool integrations |
| **Template community** | Browsable/favorite-able summary templates with usage tracking |
| **Admin console** | System users, menus, dictionaries, firmware OTA, operation logs |
| **Device management** | IoT hardware device binding/unbinding, firmware check |

## 3. Non-Scope

| Area | Reason |
|------|--------|
| AI processing (transcription, LLM inference, embedding) | Handled by external AI Service (Python/FastAPI) |
| Mobile UI / Flutter application | Separate APP repository |
| Hardware firmware | Memoket BLE recorder firmware is a separate codebase |
| Admin frontend | Separate admin web application |
| Payment / billing | Not implemented in current scope |
| Analytics / BI dashboards | Not part of backend services |

## 4. Tech Stack

| Concern | Technology | Version/Notes |
|---------|-----------|---------------|
| Language | Go | 1.24 |
| Framework | go-zero | v1.9.3 |
| Inter-service communication | gRPC + etcd discovery | go-zero built-in |
| Database | PostgreSQL | Primary data store |
| Cache / Session / Stream | Redis | Standalone + cluster support |
| Object storage | AWS S3 + CloudFront CDN | Presigned URL pattern |
| Push notifications | Firebase Cloud Messaging (FCM) | Android + iOS |
| Email | Mailgun | Transactional email |
| Auth | JWT (HS256) + Redis token store | Single-device-per-type enforcement |
| Transport encryption | AES-CBC request/response body encryption | Additional app-level layer |
| Streaming | Server-Sent Events (SSE) | AI chat / insight streaming proxy |
| Async processing | Redis Streams (consumer groups) | Export worker, notification dispatcher |
| ID generation | Snowflake | Distributed unique IDs |
| Code generation | goctl (go-zero CLI) | API, RPC, and model boilerplate |
| Deployment | Docker Compose | Per-environment configuration |
