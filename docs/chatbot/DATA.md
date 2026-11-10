# Chatbot — Data Documentation

> **Status:** Draft · **Owner:** Data lead · **Last updated:** 2026-11-10
>
> This document defines what data the chatbot on the Cerion Consulting website uses, and how that data is collected, processed, stored and protected. Items marked "proposed" are not decisions until the team approves them.

---

## Contents

1. [Purpose and scope](#1-purpose-and-scope)
2. [Data sources](#2-data-sources)
3. [Knowledge base preparation](#3-knowledge-base-preparation)
4. [Conversation data](#4-conversation-data)
5. [Data model (schema)](#5-data-model-schema)
6. [Privacy, KVKK and GDPR](#6-privacy-kvkk-and-gdpr)
7. [Retention and deletion](#7-retention-and-deletion)
8. [Security and access](#8-security-and-access)
9. [Data quality and updates](#9-data-quality-and-updates)
10. [Metrics and reporting](#10-metrics-and-reporting)
11. [Roles and responsibilities](#11-roles-and-responsibilities)
12. [Open questions](#12-open-questions)

---

## 1. Purpose and scope

**What the chatbot is for:** Answering website visitors' questions about our services, how we work and how to get in touch, and directing interested visitors to the contact form or a meeting request.

**This document covers:**

- The content the chatbot relies on to answer (the knowledge base)
- The data created by conversations with visitors
- How this data is stored, protected and analyzed

**Out of scope:** UI design, model/provider choice and infrastructure setup (covered in their own documents).

---

## 2. Data sources

| # | Source | Content | Format | Owner | Update frequency |
|---|--------|---------|--------|-------|------------------|
| 1 | Website pages | Services, about us, contact | HTML / Markdown | Web team | Whenever a page changes |
| 2 | FAQ list | Frequently asked questions with approved answers | Markdown / YAML | Data lead | Monthly review |
| 3 | Blog / case studies | Published content | HTML / Markdown | Content team | When a new post is published |
| 4 | Fixed facts | Office hours, contact channels, office details | YAML / JSON | Data lead | Whenever they change |
| 5 | Conversation logs | Visitor messages and bot replies | Database | Data lead | Continuous |

**Rules:**

- Only **public, approved** content goes into the knowledge base.
- Client contracts, proposals, price lists, internal correspondence and client personal data **never** go into the knowledge base.
- Every source must have an owner; content without an owner is not added.

---

## 3. Knowledge base preparation

For the chatbot to give correct, up-to-date answers, content goes through the following steps (proposed flow — RAG, retrieval-augmented generation: before answering, the bot looks up the most relevant pieces of our content and bases its answer on them):

```
Sources ──► Collect ──► Clean ──► Chunk ──► Embed ──► Vector database
                                                          │
Visitor question ──────────────► Find most relevant chunks ◄┘
                                          │
                                          ▼
                          Model answers based on those chunks
```

### 3.1 Collection

- Web pages are pulled from the sitemap or from the content repository (Markdown).
- FAQ and fixed facts live in one folder under version control (proposed: `content/chatbot/`).

### 3.2 Cleaning

- Repeated elements such as menus, footers and cookie banners are removed.
- Empty, duplicate or draft pages are filtered out.
- Text is normalized to UTF-8; Turkish characters are preserved.

### 3.3 Chunking

| Parameter | Proposed value | Note |
|-----------|----------------|------|
| Chunk size | ~300–500 words | Splitting by heading is preferred |
| Overlap | ~10–15% | Reduces lost context between chunks |
| FAQ | Each question-answer pair is one chunk | Never split |

### 3.4 Metadata

Every chunk has at least these fields:

- `source_url` or `source_file` — to show where the answer came from
- `title` — page / section title
- `language` — `en` / `tr`
- `last_updated` — when the content was last updated
- `category` — e.g. `services`, `contact`, `faq`

### 3.5 Language

- The knowledge base is kept in every language the site is published in.
- Chunks in the visitor's language are retrieved first.

---

## 4. Conversation data

### 4.1 Data we collect

| Data | Why | Required? |
|------|-----|-----------|
| Session ID (anonymous) | To group messages into one conversation | Yes |
| Message text (user + bot) | To measure and improve answer quality | Yes |
| Timestamp | Analysis, retention tracking | Yes |
| Page URL (where the chat started) | To see which pages raise questions | Yes |
| Language | Reporting | Yes |
| Feedback (👍 / 👎) | Quality measurement | No |
| Contact details (name, email) | Only if the visitor **chooses** to request contact | No |

### 4.2 Data we do not collect

- Full IP addresses (if needed, only a truncated / anonymized form)
- Browser fingerprints
- Any personal data without the visitor's consent
- Identity details automatically extracted from conversation text

### 4.3 Personal data masking

Visitors may type personal data into messages on their own. Before storage, these patterns are masked (proposed):

- Email addresses → `[EMAIL]`
- Phone numbers → `[PHONE]`
- National ID numbers / IBANs / card numbers → `[REDACTED]`

> Note: Contact details shared **deliberately** through the contact form are not masked; they are kept in a separate table with separate access rights (see the `lead` table).

---

## 5. Data model (schema)

Proposed tables (database type not yet decided):

### `conversation`

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID | Primary key |
| `started_at` | timestamp | Start of the conversation |
| `ended_at` | timestamp, null | Time of the last message |
| `page_url` | text | Page where the chat started |
| `language` | text | `en` / `tr` |
| `handoff` | boolean | Whether it was handed off to a person / the contact form |

### `message`

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID | Primary key |
| `conversation_id` | UUID | `conversation.id` |
| `role` | enum | `user` / `assistant` |
| `content` | text | Masked message text |
| `sources` | JSON, null | Knowledge base chunks used in the answer |
| `created_at` | timestamp | Message time |

### `feedback`

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID | Primary key |
| `message_id` | UUID | `message.id` |
| `rating` | enum | `up` / `down` |
| `comment` | text, null | Optional comment |
| `created_at` | timestamp | |

### `lead` (separate access rights)

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID | Primary key |
| `conversation_id` | UUID | `conversation.id` |
| `name` | text | Name given by the visitor |
| `email` | text | Email given by the visitor |
| `consent_at` | timestamp | Time of explicit consent |
| `consent_text_version` | text | Version of the privacy notice shown |

---

## 6. Privacy, KVKK and GDPR

- **Notice:** The chat window shows a short notice and a link to the Privacy Policy ("This chat is recorded to improve our service").
- **Explicit consent:** Contact details are collected only after the visitor ticks a consent checkbox.
- **Data minimization:** No data is collected beyond what the purpose needs (see §4.2).
- **Third-party providers:** The data processing agreement (DPA) of the model / hosting provider is reviewed, and we confirm that messages are not used to train the provider's models.
- **International transfers:** The provider's server location is identified and, where KVKK/GDPR require it, added to the Privacy Policy.
- **Data subject requests:** Access / deletion requests are answered within 30 days at most; records must be findable and deletable by session ID or email.

---

## 7. Retention and deletion

| Data | Retention (proposed) | At the end of the period |
|------|----------------------|--------------------------|
| Conversations and messages | 12 months | Deleted automatically |
| Feedback | 12 months | Deleted automatically |
| Leads (contact details) | Until the contact process ends, max 24 months | Deleted or moved to the CRM |
| Anonymous aggregate statistics | Indefinite | — (no personal data) |
| Knowledge base | As long as the source is published | Removed from the index when the source is removed |

- Deletion runs as a scheduled job (e.g. daily) and is logged.
- Data in backups follows the same retention policy.

---

## 8. Security and access

- API keys and database passwords are **never** written to the repository; use environment variables or a secret manager.
- Data is encrypted in transit (TLS) and at rest.
- Access roles:

| Role | `conversation` / `message` | `feedback` | `lead` |
|------|----------------------------|------------|--------|
| Data lead | Read / delete | Read | Read / delete |
| Developer | Test environment only | Test environment only | No access |
| Sales / contact | No access | No access | Read |

- Test and development environments **never use real conversation data**; use synthetic sample data instead.

---

## 9. Data quality and updates

- **Single source of truth:** Each fact (e.g. the contact email) is kept in exactly one place.
- **Re-indexing:** When source content changes, the knowledge base is rebuilt (proposed: automatically after every merge to the content repository).
- **Monthly review:**
  - Review answers that got a 👎
  - List unanswered questions → add them to the FAQ or update the content
  - Check stale content (`last_updated` older than 6 months)
- **Test set:** Keep an evaluation set of at least 30–50 questions with expected answers; run it whenever the knowledge base or settings change.

---

## 10. Metrics and reporting

| Metric | Definition | Target (proposed) |
|--------|------------|-------------------|
| Conversations | Conversations started per week / month | Track |
| Answer rate | Share of questions the bot could answer from the knowledge base | ≥ 85% |
| Positive feedback rate | 👍 / (👍 + 👎) | ≥ 80% |
| Handoff rate | Share of conversations handed to the contact form / a person | Track |
| Lead conversion | Share of conversations where the visitor left contact details | Track |
| Top topics | Question distribution by category | Monthly report |

Reports use only **anonymous, aggregated** data.

---

## 11. Roles and responsibilities

| Task | Owner |
|------|-------|
| Maintaining this document | Data lead |
| Accuracy of knowledge base content | The relevant content owner (see §2) |
| Keeping the FAQ list up to date | Data lead |
| Privacy notice and KVKK compliance | Data lead + legal advisor |
| Infrastructure, encryption and access settings | Developer |
| Monthly quality report | Data lead |

---

## 12. Open questions

- [ ] Which model / provider will we use? Where are its servers, and what is the DPA status?
- [ ] Which database and vector database will we use?
- [ ] Which languages will the site be published in (English only, or English + Turkish)?
- [ ] Are the retention periods in §7 approved?
- [ ] Will lead data be sent to a CRM? If so, which one?
- [ ] How should the chatbot handle pricing / proposal questions (redirect, or give general information)?
- [ ] Who writes / approves the chatbot section of the Privacy Policy?
