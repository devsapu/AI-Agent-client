# PFEC Global — AI Calling Solution: Technical Implementation Guide

**Prepared by:** AI Technology Consulting Engagement
**Companion to:** `proposal/PFEC_AI_Calling_Proposal.pptx` / `.docx`
**Purpose:** Engineering-level detail behind the proposal — exact technologies, what gets built, and in what order. Where the deck answers "what and why," this answers "how."

---

## 1. Scope of this document

The proposal recommends a four-layer composable architecture (Zoho CRM → Custom Middleware → Voice Orchestration → Speech/LLM). This guide specifies:

- The concrete technology choice for each layer and the reasoning
- The middleware's internal design: modules, data model, APIs in and out
- The exact Zoho CRM configuration tasks (fields, Workflow Rules, Deluge functions)
- Voice platform and speech-layer setup steps
- Security, environments, and a testing plan mapped to the BRD's acceptance criteria (AC-01–AC-07)

Everything here is planning-level — a starting spec for the Phase 1 discovery week, not a fixed build contract.

---

## 2. Recommended technology stack

| Layer | Technology | Why |
|---|---|---|
| CRM system of record | Zoho CRM (existing) | Already in place; owns lead intake, the 11-stage model, and downstream Email/WhatsApp automation |
| Integration middleware | Node.js/TypeScript (or Python/FastAPI) on a small cloud VM or serverless container | Thin, stateless-where-possible service; team's existing language preference decides this — no strong technical reason to prefer one over the other here |
| Middleware state store | PostgreSQL (attempt history, retry schedule) + Redis (short-lived job queue for retry scheduling) | Attempt/stage history needs durable relational storage for reporting (BRD §12); retry scheduling benefits from a lightweight queue rather than cron-polling a database table |
| Voice orchestration | Vapi (primary) or Retell AI | BYO-SIP, mid-call function-calling, configurable webhook — see proposal §4.1 for the comparison |
| Speech (TTS/STT) | Azure AI Speech (bn-BD neural) primary, Sarvam AI / Deepgram Nova-3 as A/B candidates | Bangla dialect accuracy is the single highest-risk decision — validated empirically in Phase 1 Week 5, not committed up front |
| Conversation LLM | GPT-4o / GPT-4.1-class (OpenAI or Azure OpenAI) | Multilingual reasoning, native function-calling for Phase 2 booking |
| Migration knowledge base | Small RAG store (e.g. pgvector on the same Postgres instance, or a managed vector store) inside the middleware | Keeps visa/migration content PFEC-governed per FR-07, not hard-coded into a script |
| Telephony | PFEC's own SIP trunk (BYO-SIP), bridged via Twilio Elastic SIP Trunking if needed | Confirmed to support Bangladesh-destined outbound calling; local caller ID improves answer rates |
| WhatsApp (Phase 2) | Direct Meta WhatsApp Cloud API, or a BSP (WATI/Interakt) | Zoho CRM has no native WhatsApp channel; decided in Phase 2 discovery based on PFEC's dev capacity |

---

## 3. Middleware — detailed design

The middleware is the one component with no off-the-shelf equivalent — it owns the Zoho CRM contract, the retry logic, and the migration knowledge base.

### 3.1 Core modules

| Module | Responsibility |
|---|---|
| **Zoho webhook receiver** | Accepts the Instant Webhook / Deluge function POST when a lead becomes call-eligible. Validates the shared secret, parses Lead ID + context fields. |
| **Script/assistant selector** | Maps campaign name, UTM parameters, destination, branch, enquiry type, and region type to the correct Offshore/Onshore, Education/Migration script variant (BRD §5, §6). |
| **Outbound call trigger** | Calls the voice platform's Create Outbound Call API with `{phone, leadId, assistantId, contextVars}`. |
| **End-of-call webhook receiver** | Accepts the voice platform's callback: `{leadId, outcome, transcript, summary, capturedFields, recordingUrl, duration}`. |
| **Stage mapper** | Maps the outcome to the correct one of the 11 Zoho CRM Lead Stage values per Business Rules BR-01–BR-10. |
| **Zoho write-back client** | Calls the Zoho CRM REST API to update Lead Stage, attempt number, timestamps, call summary, appointment/disqualification detail (BRD §7.2). |
| **Retry scheduler** | Tracks attempt count per lead; schedules the next attempt within the 7-attempt / 5-day window (BR-04), respecting configurable calling-hour windows (NFR-04). Auto-disqualifies on the 7th unanswered attempt (BR-08). |
| **Migration knowledge base (RAG)** | Small vector store of PFEC-approved visa categories and talking points; queried via function-calling during Onshore Migration calls only (FR-07). Content is PFEC-owned and versioned, not hard-coded. |
| **Structured logger** | Emits attempt-level events for the BRD §12 reporting requirements: lead volume, stage distribution, attempt-wise conversion, disqualification reasons. |

### 3.2 Data model (minimum viable schema)

```
leads
  id, zoho_lead_id, phone, region_type, enquiry_type, campaign,
  utm_source, utm_medium, utm_campaign, utm_content,
  destination, branch, current_stage, attempt_count,
  created_at, updated_at

call_attempts
  id, lead_id (fk), attempt_number, attempt_at, outcome,
  transcript_summary, recording_url, duration_seconds,
  next_follow_up_at

disqualifications
  lead_id (fk), reason, disqualified_at

appointments
  lead_id (fk), meeting_mode, booked_at, branch
```

### 3.3 APIs — in and out

**Inbound (middleware exposes):**
- `POST /webhooks/zoho` — Zoho Workflow Rule / Deluge function calls this when a lead becomes eligible
- `POST /webhooks/voice-platform` — Vapi/Retell calls this at end-of-call

**Outbound (middleware calls):**
- Voice platform "Create Outbound Call" API
- Zoho CRM REST API (`PATCH` on the Lead record for stage/attempt/appointment write-back)
- Speech/LLM APIs — only if the middleware itself needs to query the knowledge base for function-calling responses (Phase 2); the voice platform calls these directly for the conversation loop itself

### 3.4 Retry scheduler logic (BR-01–BR-09)

```
on lead eligible:
  stage = "Not Yet Contacted"
  trigger first call attempt

on call outcome = not_connected:
  attempt_count += 1
  if attempt_count >= 7:
    stage = "Disqualified"; reason = "No Response After 7 Attempts"
    stop scheduling further attempts
  else:
    stage = f"{ordinal(attempt_count)} Contact Attempted"
    schedule next attempt within configured window, before day 5 elapses

on call outcome = connected, interested, not booked:
  stage = "In Contact Interested"
  stop unanswered-retry counting; lead stays active for follow-up

on call outcome = connected, booked:
  stage = "Appointment Booked"
  stop all further AI attempts (BR-09)

on call outcome = connected, not interested:
  stage = "Disqualified"; reason = "Not Interested"
  stop all further AI attempts (BR-09)
```

---

## 4. Zoho CRM configuration tasks

1. **Custom fields / picklists** — Lead Stage (11 values, exact names per BRD §8), attempt number, next follow-up date/time, call summary, recording URL, appointment status, meeting mode, disqualification reason, plus the read-side context fields already in BRD §7.1 (campaign, UTM set, destination, branch, enquiry type, region type).
2. **Workflow Rule** — triggers on "lead becomes call-eligible" (however that's defined for PFEC — e.g. a specific lead source + stage combination).
3. **Instant Webhook or Deluge custom function** — fires from the Workflow Rule, POSTs Lead ID + context fields to the middleware's `/webhooks/zoho` endpoint. No Zapier or other iPaaS anywhere in this path (BRD §3.2, explicit constraint).
4. **Downstream Workflow Rules** — separate rules keyed off Lead Stage changes, to trigger Zoho's own Email/WhatsApp automation on `Appointment Booked` (confirmation) and `In Contact Interested` (nurture nudge), per BR-10.
5. **API / OAuth setup** — a Zoho CRM API client (self-client or server-based) with scopes sufficient for reading lead fields and updating Lead Stage/attempt fields; confirm current Zoho CRM edition supports Workflow Rules + custom functions + the API call volume the retry model implies (NFR-01, flagged as a Week 1 discovery item in the proposal).

---

## 5. Voice platform setup (Vapi / Retell)

1. Create platform account; confirm BYO-SIP trunk connection (PFEC supplies SIP credentials for a local Bangladeshi/Australian caller ID).
2. Build the assistant(s): one script tree per Offshore/Onshore × Education/Migration combination, parameterized by the context variables passed in the outbound call request.
3. Configure the end-of-call webhook URL (pointing at the middleware) and the shared secret/token used to authenticate it (NFR-01/NFR-02).
4. Configure custom metadata pass-through so `leadId` round-trips unchanged through the whole call.
5. Phase 2 only: wire mid-call function-calling to the middleware's booking/availability endpoints.

---

## 6. Speech layer setup

1. Provision Azure AI Speech (bn-BD neural voices) as the primary TTS candidate; provision Sarvam AI and Deepgram Nova-3 as A/B candidates for STT.
2. Build a small A/B test harness: same script, three speech configurations, a batch of test calls reviewed by a native Bangladeshi Bangla speaker.
3. Score against PFEC's own Bangla-quality bar before locking in the production configuration (this is the pilot's go/no-go gate, not a build task with a fixed answer today).

---

## 7. Security & compliance

| Concern | Approach |
|---|---|
| Zoho → middleware auth | Shared secret / signed payload on the Zoho webhook call |
| Voice platform → middleware auth | Webhook URL key + secret token (NFR-01) |
| Middleware → Zoho auth | OAuth client credentials, scoped to lead read/update only |
| Middleware → voice platform auth | Platform API key, stored in a secrets manager, never in source |
| Call recording / transcript PII | Encrypt at rest; confirm data residency with the chosen speech/voice vendor (most default to US/EU regions — Zoho already offers an India data center, which may drive vendor selection); define a retention policy in the eventual SOW |
| Migration knowledge base content | PFEC-approved only, versioned, reviewed before Phase 2 go-live (FR-07) |

---

## 8. Environments & deployment

- **Environments:** dev → staging → production, with the middleware's Zoho and voice-platform credentials scoped separately per environment (sandbox Zoho org / test voice-platform project for dev and staging).
- **Hosting:** a small cloud VM or serverless container is sufficient at pilot volume (~$50–150/month per the proposal's cost estimate); no need for a full Kubernetes setup at this scale.
- **CI/CD:** standard build → test → deploy pipeline; the middleware is a small enough service that a single-environment promotion flow (staging → production) is sufficient for Phase 1.
- **Monitoring/logging:** structured logs per call attempt (already required for BRD §12 reporting) double as the operational monitoring signal; alert on webhook failures and on retry-scheduler jobs falling behind.

---

## 9. Testing plan (mapped to BRD acceptance criteria)

| Test | Validates |
|---|---|
| Unit tests on the stage-mapper and retry-scheduler logic | BR-01–BR-09 business rules, independent of any live integration |
| Integration test: mock Zoho webhook → middleware → mock voice-platform call | AC-01 (lead triggers the workflow without Zapier) |
| Integration test: contextual field pass-through | AC-02 (campaign/UTM context correctly reaches the conversation) |
| Scripted attempt-sequence test (using fixed outcomes, not live calls) | AC-03 (unanswered leads progress through the 7-attempt sequence in order) |
| End-to-end test call, interested-but-unbooked outcome | AC-04 |
| End-to-end test call, booked outcome | AC-05 |
| End-to-end test call, disqualified outcomes (both reasons) | AC-06 |
| Downstream Zoho automation check | AC-07 (Zoho triggers follow-up communication off the returned stage) |
| Native-speaker review of recorded test calls | Pilot's Bangla-quality go/no-go gate |

The interactive demo mockup (`deliverables/demo-site/index.html`) exercises this exact same stage-mapping and write-back logic against sample leads, and can be used directly in the client meeting to walk through each test scenario above without needing a live call.

---

## 10. Open technical dependencies

These need a PFEC answer before implementation can start in earnest — carried over from the proposal's risk/assumptions section, restated here as engineering blockers:

- Current Zoho CRM edition and API rate limits (Week 1 discovery item)
- Final CRM field mapping and campaign naming conventions
- BYO-SIP trunk provider and credentials
- Migration/visa knowledge base content, approved and versioned by PFEC before Phase 2
- WhatsApp channel decision (direct Meta Cloud API vs. a BSP) — needed before Phase 2, not Phase 1
