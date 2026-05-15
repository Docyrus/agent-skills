# Integrations & Events

## Connector Framework

- **HTTP providers** — OAuth2-enabled external API integrations (Stripe, Slack, Microsoft, etc.) with custom transformers for request/response mapping
- **SQL providers** — Direct database connections to external databases
- **Microsoft integrations** — SharePoint and MS Graph connectors

## Provider Features

- OAuth2 credential management with automatic token refresh
- Webhook subscription management
- Configurable pagination
- Per-tenant and per-user credential isolation

## Webhooks & Events

- Create and manage webhooks with signed verification
- Event types: data operations, chat platform messages, collaboration events
- Public webhook keys for external service integration (AI agent, bot, email, portal, webform)
- File upload support via webhooks
- Delivery tracking with retry logic

## Content & Document Management

### Collaborative Document Editor

- Real-time collaborative editing
- Document versioning and history
- Fragment-based document access
- Batch document import and version management

### File Management

- File uploads with type validation and size limits
- Organized folder structures
- File metadata and association with records

## Messaging & Notifications

### In-App Messaging

- Thread-based conversations
- Message history with user mentions
- File sharing in messages
- Conversation channels embedded in data sources

### Notifications

- Notification center with read/unread tracking
- Per-type notification preferences
- Device-based push notifications
- Email notification delivery

### Email

- Transactional email templates (signup, password reset, invitations, magic links)
- Custom HTML email templates with dynamic content
- Email configuration management
- Delivery tracking and webhook integration
- Tenant email accounts with per-provider credentials, optional user-level identity override, and send-as-user enforcement
- Send endpoint `POST /v1/messaging/email/accounts/{accountId}/send` (scope `Messaging.Email.Send`) with multi-recipient `to`/`cc`/`bcc`/`replyTo`, HTML or text body, and up to 10 storage-backed attachments
- Account discovery via `GET /v1/messaging/email/accounts` (credentials are never returned; the response exposes provider, sender identity, and override flags)
