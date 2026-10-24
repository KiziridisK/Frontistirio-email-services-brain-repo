# Frontistirio — Email Services Brain

> **Κατάσταση τεκμηρίωσης** (σφραγίδα 2026-10-24) · τελευταία γραφή **2026-07-30**.
> **Δεν φέρει επαλήθευση έναντι κώδικα** — κανόνας #4 του ευρετηρίου. Από εκείνη την ημερομηνία ο κώδικας
> έχει προχωρήσει **114 commits** στο frontend και **32** στο API (όλο το repo, όχι μόνο αυτό το feature).
> Σημερινά HEAD: frontend `0f466e0` / API `cabaa2c`.
> Τι είναι **γνωστό** ότι λείπει: §5 και §5.1 του ευρετηρίου (`Frontistirio-brain-repos-index`).

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

### 2026-07-30 — SES production rollout + Brevo fully removed
All three "blocking real-world use" items are done, and email now sends for real (out of sandbox, deliverable, tracked). **All SES config is in region `eu-central-1`** (sandbox/prod + domain verification are per-region — NOT the us-east-1 of the CloudFront cert).

- **Verified sending domain `eduflow.gr`** with **Easy DKIM** (RSA-2048, `Verified:SUCCESS`) + custom **MAIL FROM** `mail.eduflow.gr` (SPF alignment) + **DMARC** `_dmarc.eduflow.gr` (now `p=quarantine`). DNS records live in the eduflow.gr Route 53 zone (3 DKIM CNAMEs, MAIL FROM MX+SPF-TXT, DMARC TXT — **zone id + exact records in the private custom-domain brain**). `SES_FROM` is now `no-reply@eduflow.gr`. **Confirmed:** a live send landed in the Gmail **Inbox** (not spam), fixing BUG-EML-003.
- **Delivery-tracking pipeline wired**: Configuration Set **`frontistirio-events`** → SNS topic `frontistirio-ses-events` (topic policy grants `ses.amazonaws.com` publish) → HTTPS subscription **CONFIRMED** to `POST /emails/ses-webhook`. The REST API got the `/emails/ses-webhook` resource (auth NONE, HTTP_PROXY→EC2) + Prod deploy — **gateway/resource ids in the private aws-infrastructure brain**. `SES_CONFIGURATION_SET=frontistirio-events`. **SNS signature verification** (`sns-validator`) added to `handleSesWebhook` + AWS-host guard on SubscribeURL (fixes BUG-EML-004). Verified end-to-end: SNS metrics showed `MessagesPublished=2 / NotificationsDelivered=2 / Failed=0`.
- **Production access GRANTED** (out of sandbox): quota **50,000/day, 14/s**. The `put-account-details` request first showed `ReviewDetails.Status:DENIED` while its support case stayed open; approval flipped `ProductionAccessEnabled:true` ~3 days later via the case reply (a CLI re-submit while the case was open returned `ConflictException` — the DENIED flag was stale, the case was still live). The Support API needs a Business plan, so case replies go through the web console.
- **Brevo removed entirely**: the **student** and **parent** invites now go through the SES transactional core (`services/inviteMailer.js`), like the teacher invite. `inviteMailer` is now role-agnostic via an **`accountKind`** param ("μαθητή"/"γονέα"/"καθηγητή") + generic "Όνομα χρήστη:" label. Deleted `utils/mailService.js` + `models/mailLog.js`; dropped the dead `SES_FROM || SMTP_FROM` fallbacks; removed `SMTP_*` from the EC2 `.env`. **`nodemailer` stays** (sesMailer's MIME composer for attachments).

**Gotchas:** `aws sesv2 send-email --content file://…` with a Greek UTF-8 body fails "could not be decoded" → set `AWS_CLI_FILE_ENCODING=utf-8`. The EC2 box has no GitHub creds (HTTPS remote, `git pull` fails) so deploys go via a **git bundle over SSH** (`git bundle create … <old>..test` → scp → `git fetch <bundle> test` + `git merge --ff-only`) — see custom-domain brain.

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
SES_FROM=no-reply@eduflow.gr      # verified domain identity (was @gmail.com → spam; fixed 2026-07-30)
SES_FROM_NAME=Frontistirio        # fallback display name (store name overrides at send time)
SES_CONFIGURATION_SET=frontistirio-events   # SES config set → SNS → webhook (delivery tracking)
# reuses AWS_REGION / AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY (region eu-central-1)
# NOTE: SMTP_* (old Brevo) removed 2026-07-30 — everything sends through SES now.
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

## AWS setup (manual, NOT in code) — ✅ DONE 2026-07-30 (region eu-central-1)

1. ✅ **Sending identity**: domain `eduflow.gr` verified + Easy DKIM + MAIL FROM `mail.eduflow.gr`. `SES_FROM=no-reply@eduflow.gr`.
2. ✅ **Configuration Set** `frontistirio-events` with an SNS event destination (topic `frontistirio-ses-events`). `SES_CONFIGURATION_SET=frontistirio-events`.
3. ✅ **Subscribed** the webhook (the API Gateway `Prod` URL + `/emails/ses-webhook`) to the SNS topic — auto-confirmed by the handler. (Local dev still needs an ngrok tunnel since SNS can't reach `localhost`.)
4. ✅ **Production access GRANTED** — out of sandbox, quota 50,000/day, 14/s.

> Redo-from-scratch commands + the API Gateway wiring for the webhook resource are in the change-log entry above and the aws-infrastructure brain.

---

## TODO / Εκκρεμότητες

### ✅ Blocking real-world use — ALL DONE 2026-07-30
- [x] **Verified sending domain + Easy DKIM.** `eduflow.gr` verified + Easy DKIM + MAIL FROM `mail.eduflow.gr` + DMARC `p=quarantine`; `SES_FROM=no-reply@eduflow.gr`. Live send confirmed in Gmail Inbox (not spam).
- [x] **SES production access.** Granted — out of sandbox, quota 50,000/day, 14/s.
- [x] **SNS delivery webhook.** Config Set `frontistirio-events` → SNS `frontistirio-ses-events` → subscribed `/emails/ses-webhook` (wired in API Gateway + deployed) with `sns-validator` signature check; `SES_CONFIGURATION_SET` set. Pills now go 🟠→🟢/🔴.

### 🟠 Robustness / scale
- [ ] **Move scheduling/sending to a job queue (BullMQ)** for retries/backoff and multi-instance safety. Current `setInterval` worker is in-process only; immediate send loops recipients **synchronously in the request** (fine for school-size lists, not for large blasts).
- [ ] **Resume stranded sends.** A hard crash mid-`dispatchEmail` can leave an email in `sending`. Add a recovery sweep (re-claim `sending` emails with `pending` recipients).
- [ ] **Garbage-collect ad-hoc S3 attachments.** `email-attachments/...` objects are never deleted — add an S3 lifecycle policy or cleanup job.

### 🟢 Enhancements
- [ ] **Live email preview in compose** (show the branded template before sending).
- [ ] **Per-store branding** in the template (logo + brand colour from store settings; currently hardcoded `#3b5bdb`).
- [x] **Unify the invitation flow** onto this SES core — **DONE 2026-07-30** (Brevo removed). Teacher (2026-07-06), then **student + parent** (2026-07-30) invites all go through `services/inviteMailer.js` → `sesMailer.sendOneEmail` (transactional; temp password never persisted — no Email doc, no MailLog). `sendInviteEmail({to, recipientName, storeName, username, password, loginUrl, accountKind, fromAddress, fromName, configurationSetName})` — `accountKind` ("μαθητή"/"γονέα"/"καθηγητή") makes the copy role-agnostic. Called from `users.createStoreStudentUser` (before-commit, rollback on fail), `createStoreParentUser` (best-effort — parents may be phone-only), `createStoreTeacherUser`/`resendStoreTeacherInvite`. `utils/mailService.js` + `models/mailLog.js` **deleted**; `SMTP_*` env removed. Trade-off: transactional invites get NO per-recipient delivery tracking (fine for a one-off credential email).
- [ ] **Attachments display: file sizes / download** on the detail page (currently filename + source icon only).
- [ ] **Search+modal for ad-hoc files** too (if needed) — materials already got this treatment.
