# Frontistirio — Email Services Brain

> **Full-stack reference** for the email feature: composing & sending email through **Amazon SES**, immediate or **scheduled** delivery, **per-recipient delivery-status tracking** (green/red/orange), attachments (ad-hoc uploads + educational-material files), and a branded, personalized HTML template. Spans **Frontistirio-API** (Node/Express/MongoDB/SES) and **Frontistirio** (Angular/Ionic/NgRx).

---

## Change log

### 2026-06-26 — Initial build of the email service
Built the whole feature end-to-end on top of the existing (empty) `/emails` UI scaffold.

- **Provider:** Amazon SES (chosen over SendGrid — user prefers AWS; backend already used AWS SDK v3 for S3). Reuses the existing `AWS_*` creds; the IAM user just needs `ses:SendEmail`.
- Send **now** or **schedule** for later; **cancel** a still-scheduled email.
- **Per-recipient** status with live updates over the existing Socket.IO + NgRx pipeline.
- **Attachments** from two sources: ad-hoc uploads (→ S3) and selected educational materials (reuse their S3 key). Sent as raw MIME via SES.
- **Branded HTML template** with greeting (recipient name) + sign-off (store name), rendered per recipient at send time.

**Verification:** backend `node --check` + require smoke-tests clean; frontend `ng build` (dev) exit 0; template render-test (Greek) OK.

**Still pending — see [TODO](#todo--εκκρεμότητες) at the bottom.** The big ones: SNS webhook not yet wired (so status stops at "sent", never "delivered"); SES still in sandbox; `SES_FROM` is a `@gmail.com` address → lands in spam (must use a verified owned domain + DKIM).

---

## Architecture overview

```
COMPOSE (frontend) ──POST /emails/send (multipart)──► Express
   │                                                    │
   │                                          create Email doc (status=sending|scheduled)
   │                                                    │
   │                        immediate ─────► dispatchEmail()  ◄───── scheduled (jobs/emailScheduler.js, 60s tick)
   │                                                    │
   │                                  per recipient: render template + fetch S3 attachments
   │                                                    │
   │                                         SES SendEmail (one call PER recipient)
   │                                                    │
   │                                  recipient.status = sent | failed, store providerMessageId
   │                                                    ▼
   │                                          email.save() → MongoDB change stream
   │                                                    │
   └──────────── socket "emailAdded"/"emailUpdated" ◄──┘ (watchEmails)

DELIVERY EVENTS:  SES → Configuration Set → SNS → POST /emails/ses-webhook
                  → applyDeliveryEvent() → recipient.status = delivered|bounced
                  → save() → change stream → socket → UI pill recolours live
```

**Key design decision — one SES call PER recipient:** gives each recipient its own `MessageId` (the correlation key stored on the recipient) so webhook events map back precisely, and keeps recipients' addresses private.

---

## Status lifecycle

**Per-recipient** (`email.recipients[].status`) → drives the colored pills:

| status | colour | meaning |
|---|---|---|
| `pending` | 🟠 orange | created, not handed to SES yet (e.g. still scheduled) |
| `sent` | 🟠 orange | SES accepted it, awaiting delivery webhook |
| `delivered` | 🟢 green | SNS webhook confirmed delivery |
| `bounced` | 🔴 red | bounce / spam complaint |
| `failed` | 🔴 red | the SES API call itself failed |
| `canceled` | ⚪ grey | scheduled send canceled before it left |

**Message rollup** (`email.status`), recomputed from recipients by `recomputeRollup()`:
`scheduled` → `sending` (any still `pending`) → `sent` (all handed off) / `partially_failed` / `failed`; or `canceled`.

> ⚠️ **Gotcha fixed 2026-06-26:** `recomputeRollup` originally bucketed `sent` with `pending`, so a message got stuck on `sending` forever when no webhook was configured. Now `sent` counts as handed-off, so the message reaches `sent` immediately and individual recipients upgrade to `delivered` later (if the webhook is set up).

---

## Backend (`Frontistirio-API`)

### Model — `models/email.js` (`Email`)
```javascript
{
  subject: String, bodyHtml: String, bodyText: String,   // bodyText = raw composed text

  recipients: [{                       // one sub-doc per person → per-recipient status
    name, email (required), role: 'student'|'teacher'|'parent',
    status: 'pending'|'sent'|'delivered'|'bounced'|'failed'|'canceled',
    providerMessageId,                 // SES MessageId — webhook match key
    error, lastEventAt
  }],

  attachments: [{                      // REFERENCES only — bytes live in S3
    filename, contentType, size,
    source: 'upload'|'educational-material',
    materialId → EducationalMaterial,
    s3Key, bucket
  }],

  store → Store, createdBy → User,
  fromAddress,                         // verified SES identity (SES_FROM)
  fromName,                            // = store name (recipient sees the store as sender)
  storeName,                           // persisted for the template (scheduled sends too)

  mode: 'now'|'scheduled', sendAt: Date, sentAt: Date,
  status: 'scheduled'|'sending'|'sent'|'partially_failed'|'failed'|'canceled',
  counts: { total, pending, delivered, failed },  // denormalized for list badges

  isDeleted, deletedAt, updatedBy, timestamps: true
}
// method: recomputeRollup() → refresh counts + rollup status from recipients
```

### Routes — `routes/emails.js`
| Method | Path | Middleware | Handler |
|---|---|---|---|
| POST | `/emails/send` | auth, isStoreUser, `multer.memoryStorage` `upload.array("attachments",10)` | `sendEmail` |
| GET | `/emails/store-emails` | auth, isStoreUser | `getStoreEmails` |
| GET | `/emails/get-email/:emailId` | auth, isStoreUser | `getEmail` |
| POST | `/emails/cancel` | auth, isStoreUser | `cancelScheduled` |
| POST | `/emails/ses-webhook` | **public**, `express.text({type:"*/*"})` | `handleSesWebhook` |

> Multipart so files ride along. Array fields (`recipients`, `educationalMaterialIds`) arrive as **JSON strings** → `parseJsonField()` in the controller.

### Controller — `controllers/emails.js`
- `sendEmail` — parse multipart; build recipient sub-docs (pending); upload ad-hoc files to S3 + build material refs; look up store name (`Store.findById`); create the Email doc; if immediate → `dispatchEmail`, else leave `scheduled`.
- `getStoreEmails` / `getEmail` — filter `isDeleted:false`.
- `cancelScheduled` — 409 if already sending.
- `handleSesWebhook` — SNS: `SubscriptionConfirmation` (GET the SubscribeURL once) + `Notification` (Delivery→delivered, Bounce/Complaint→bounced) → `applyDeliveryEvent`. Always returns 200 so SNS doesn't hammer retries.

### Handler (DB + change stream) — `controllers/handlers/emails.js`
- `createEmail`, `getStoreEmails`, `getEmail`.
- `claimDueScheduledEmail()` — atomic `findOneAndUpdate` `scheduled→sending` (prevents double-send).
- `cancelScheduledEmail(id,userId)` — guarded `status:"scheduled"`; marks all recipients canceled.
- `applyDeliveryEvent(providerMessageId, email, newStatus, err)` — locate recipient by MessageId (fallback: email), update, `recomputeRollup`, save. Never downgrades a `delivered`.
- `watchEmails(io, userSockets)` — MongoDB change stream → socket `emailAdded`/`emailUpdated`(full doc as `changes`)/`emailDeleted`. **Uses `fullDocumentBeforeChange:"whenAvailable"`** (NOT "required" — the emails collection has no pre-images enabled; we soft-delete so we never need the before-image). Has an `on("error")` handler that logs + re-establishes after 5s (prevents the unhandled-error process crash).

### Services
- `services/sesClient.js` — shared `SESv2Client` (same region/creds as `s3Client.js`).
- `services/sesMailer.js` — `sendOneEmail({to,subject,html,text,fromAddress,fromName,configurationSetName,tags,attachments})`. No attachments → `Content.Simple`. With attachments → builds **raw MIME** via `nodemailer/lib/mail-composer` → `Content.Raw`. Enforces the **10 MB** SES cap (`attachment_too_large`).
- `services/emailDispatcher.js` — `dispatchEmail(emailDoc)`: fetch S3 attachment bytes once; per pending recipient render the template (`renderEmailHtml`/`renderEmailText`) + `sendOneEmail`; set `sent`+`providerMessageId` or `failed`+`error`; `recomputeRollup`; `save()`. Shared by immediate send + scheduler.
- `services/emailAttachments.js` — `uploadAdHocAttachments(files,group,store)` (→ S3 `email-attachments/{group}/{store}/...`, bucket `logeion-educational-material`), `buildMaterialAttachmentRefs(ids,store)` (store-scoped lookup, reuse material's s3Key), `fetchAttachmentBytes(refs)` (`GetObjectCommand` → `Body.transformToByteArray()`).
- `services/emailTemplate.js` — `renderEmailHtml`/`renderEmailText`. Responsive table-based, inline-styled email. Header band (store name), greeting `Αγαπητέ/ή {name},`, escaped body (blank lines→`<p>`, `\n`→`<br>`), sign-off `Με εκτίμηση, {store}`, footer. Brand colour `#3b5bdb`. **Rendered per recipient at send time — never stored.**

### Background worker — `jobs/emailScheduler.js`
`startEmailScheduler()` = dependency-free `setInterval` (60s). Each tick drains all due emails via `claimDueScheduledEmail()` (atomic) → `dispatchEmail`. Started from `index.js`. **In-process only — not safe for multiple server instances** (move to BullMQ then).

### `index.js` wiring
`require` + `app.use("/emails", EmailRoutes)`; `watchEmails(io, userSockets)` (after the other watchers); `startEmailScheduler()`.

### Env vars (`.env`)
```
SES_FROM=...                 # MUST be a verified SES identity (currently a @gmail.com → spam!)
SES_FROM_NAME=Frontistirio   # fallback display name
SES_CONFIGURATION_SET=       # SES config set that publishes events to SNS (EMPTY → no delivery tracking)
# reuses AWS_REGION / AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY (region eu-central-1)
```

### Maintenance scripts
- `scripts/recomputeEmails.js` — re-run `recomputeRollup()` on all emails (idempotent; fixes ones stuck on an old rollup).
- `scripts/inspectLatestEmail.js` — dump the latest email's per-recipient status / messageId / error (diagnose accepted-vs-rejected).

---

## Frontend (`Frontistirio`)

- **Model** `models/email.model.ts` — `Email`, `EmailRecipient`, `EmailAttachmentRef`, `EmailCounts`, status unions, `SendEmailPayload`.
- **Service** `services/emails.service.ts` — `sendEmail(payload, files)` builds **FormData** (multipart); `getStoreEmails`, `getEmail`, `cancelScheduled`.
- **NgRx** `state/emails/emails.{actions,reducer}.ts` — `setEmails`/`addEmail`(upsert)/`updateEmailFields`(merge full doc)/`deleteEmail`; registered in `app.reducer.ts`; selectors `selectAllEmails`, `selectEmailById(id)`.
- **Socket** `services/web-socket.service.ts` — `emailAdded`/`emailUpdated`/`emailDeleted` → dispatch NgRx (this is what makes status pills update live).
- **Status styling** `emails/email-status.util.ts` — `recipientStatus*` + `emailStatus*` → Ionic colour/icon/i18n-key.
- **Components**
  - `emails/emails.component.*` — list bound to store + status filter + search + cancel; pulls latest via service on init.
  - `emails/email-item/*` — card: subject, rollup badge, counts, scheduled-cancel button.
  - `emails/email-details/*` — route `emails/:id`; per-recipient pills + attachments list + cancel; live via `selectEmailById`.
  - `emails/add-email/*` — compose: subject/body, recipients (via `select-recipients` modal), ad-hoc file upload, educational-material picker, schedule now/later, send.
  - `emails/select-materials/*` — **modal** with searchbar (filter by name) + checkbox list to pick educational-material attachments (scales to many materials).
- **Routing** `app-routing.module.ts` — `emails`, `emails/add`, `emails/:id` (`:id` AFTER `add`). All `roles: ['store-user']`.
- **i18n** `assets/i18n/{el,en}.json` — `emails.*` (status, recipient_status, attach_from_material, cancel_*, …), `communication.emails`, `no.emails`, `general.all`, etc.

---

## AWS setup required (manual, NOT in code)

1. **Verify a sending identity** in SES (Frankfurt / eu-central-1) → set `SES_FROM`.
2. **Configuration Set** with an **SNS event destination** → set `SES_CONFIGURATION_SET`. Without it, mail sends but recipients stay on `sent` (never `delivered`).
3. **Subscribe** the webhook URL (`/emails/ses-webhook`) to the SNS topic. Local dev: SNS can't reach `localhost` → use an **ngrok** tunnel.
4. **Request production access** (sandbox = verified recipients only).

---

## TODO / Εκκρεμότητες

### 🔴 Blocking real-world use
- [ ] **Verified sending domain + Easy DKIM.** `SES_FROM` is a `@gmail.com` address → fails SPF/DKIM/DMARC alignment for gmail.com → **Gmail spam-folders it**. Verify an OWNED domain, enable Easy DKIM (3 CNAMEs), set `SES_FROM=no-reply@<domain>`. (Code unchanged.)
- [ ] **SES production access.** Still in sandbox — every recipient must be verified. Request production access in the SES console.
- [ ] **SNS delivery webhook.** Create the Configuration Set → SNS topic → subscribe `/emails/ses-webhook`; set `SES_CONFIGURATION_SET`. This is what turns recipient pills 🟠→🟢 (`delivered`) / 🔴 (`bounced`). Until then we're "blind" to the real outcome.

### 🟠 Robustness / scale
- [ ] **Move scheduling/sending to a job queue (BullMQ)** for retries/backoff and multi-instance safety. Current `setInterval` worker is in-process only; immediate send loops recipients **synchronously in the request** (fine for school-size lists, not for large blasts).
- [ ] **Resume stranded sends.** A hard crash mid-`dispatchEmail` can leave an email in `sending`. Add a recovery sweep (re-claim `sending` emails with `pending` recipients).
- [ ] **Garbage-collect ad-hoc S3 attachments.** `email-attachments/...` objects are never deleted — add an S3 lifecycle policy or cleanup job.

### 🟢 Enhancements
- [ ] **Live email preview in compose** (show the branded template before sending).
- [ ] **Per-store branding** in the template (logo + brand colour from store settings; currently hardcoded `#3b5bdb`).
- [ ] **Unify the invitation flow** onto this SES core. The old `utils/mailService.js` (Brevo SMTP) + `models/mailLog.js` still send invitations separately — candidate to merge so invitations also get delivery tracking.
- [ ] **Attachments display: file sizes / download** on the detail page (currently filename + source icon only).
- [ ] **Search+modal for ad-hoc files** too (if needed) — materials already got this treatment.
