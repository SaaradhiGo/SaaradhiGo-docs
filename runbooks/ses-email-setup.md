# SES email — what exists, what is needed, and how QA stays safe

Written for: the owner (AWS account actions) and QA.

---

## 1. Finding: there is no SES integration

Searched the whole backend for an SES client, an SES email backend, or
`django-ses`:

```
boto3.client("ses"  / "sesv2"      -> 0 matches
django_ses / SESBackend            -> 0 matches
EMAIL_HOST / EMAIL_HOST_USER (pre) -> 0 matches in settings
SES-related variables in QA        -> none
```

What **does** exist:

| AWS service | Where | Used for |
|---|---|---|
| **SNS** | `base/utils.py` → `boto3.client("sns")` | OTP SMS. Working |
| **S3** | `base/s3.py` | KYC documents, presigned PUT. Working |
| **SES** | — | **nothing** |

`servers/ride/receipts.py` says *"if SES is down, the receipt row still gets
written"*. That comment describes an integration that was never built. Django
fell back to its default — SMTP on `localhost:25` — which does not exist on
Railway, so **every receipt email has silently failed**: the row is written,
`send_failure_reason` is populated, and nobody reads that column.

That comment should be corrected when someone next touches that file.

---

## 2. Two ways to actually send mail

### Option A — SES SMTP (recommended)

Uses the Django SMTP backend the settings already read. No new dependency, no
new code.

| Variable | Value |
|---|---|
| `EMAIL_BACKEND` | `django.core.mail.backends.smtp.EmailBackend` (the non-DEBUG default) |
| `EMAIL_HOST` | `email-smtp.<region>.amazonaws.com` (e.g. `ap-south-1`) |
| `EMAIL_PORT` | `587` |
| `EMAIL_HOST_USER` | SES **SMTP** username |
| `EMAIL_HOST_PASSWORD` | SES **SMTP** password |
| `EMAIL_USE_TLS` | `True` |
| `DEFAULT_FROM_EMAIL` | `no-reply@saaradhigo.com` |

> SES SMTP credentials are **not** your IAM access keys. They are generated in
> the SES console ("Create SMTP credentials"), which creates an IAM user with
> `ses:SendRawEmail` behind the scenes.

### Option B — SESv2 API

Reuses the IAM credentials already in the environment, so there is no new
secret to manage — but needs `django-ses` or a small custom backend, plus an
`ses:SendEmail` policy on the existing IAM user.

**Recommendation: Option A for the pilot.** It is configuration only, it works
with the code as it stands today, and SMTP credentials can be revoked without
touching the keys that S3 and SNS depend on.

---

## 3. AWS-side prerequisites (owner only)

1. **Verify the sending domain** `saaradhigo.com` in SES, in the same region
   you will send from. Add the DKIM CNAMEs and the SPF/MAIL FROM records.
   A verified *domain* is better than a verified address: it lets you send as
   `no-reply@`, `support@` and `receipts@` without re-verifying each.
2. **Move out of the SES sandbox.** A new SES account can only send to
   *verified* recipients and is capped at 200 messages/day. Production needs
   the sandbox lifted; QA does **not** — see §4.
3. **Set a DMARC record** once DKIM and SPF pass. Reset links that land in
   spam are reset links nobody uses.
4. **Configure a bounce/complaint destination** (SNS topic). SES will suspend
   an account with a high bounce rate, and a pilot sending to typo'd addresses
   generates bounces.
5. **Decide the region.** `ap-south-1` (Mumbai) for an India pilot — lower
   latency and keeps delivery data in-region.

---

## 4. QA configuration, and how no real customer gets emailed

Three independent layers. The first is the one this repo owns.

### Layer 1 — `AUTH_EMAIL_ALLOWLIST` (implemented, tested)

`base/auth_email.py` refuses to send to any address not on the allowlist
whenever `ENVIRONMENT != 'production'` **and** the backend actually delivers.

```
AUTH_EMAIL_ALLOWLIST="qa@deeptrics.com,@saaradhigo.com"
```

- Entries starting with `@` match a whole domain.
- Matching is case-insensitive; `@saaradhigo.com` does **not** match
  `evilsaaradhigo.com` or `saaradhigo.com.example.com`.
- **An empty allowlist on a delivering non-production deployment blocks every
  message.** The safe default for "we do not yet know who we may write to" is
  nobody.
- Console and locmem backends are exempt — nothing leaves the machine.
- A blocked message is logged as `auth_email_blocked` with the reason, and the
  recipient is **not** logged (an auth log listing addresses it failed to
  reach is an account list).

Covered by 12 tests in `tests/test_auth_email_safety.py`, including the
lookalike-domain case and "an unset `ENVIRONMENT` is not production".

The accident this prevents is concrete: QA pointed at real SES, a real
customer address in QA's database (restored from a dump, or typed into a test
account), somebody exercises password reset, and a reset link for a live
account lands in that person's inbox.

### Layer 2 — keep QA in the SES sandbox

Do **not** request production access for the QA identity. The sandbox already
restricts delivery to verified addresses. Two independent mechanisms that must
both fail before a customer is emailed.

### Layer 3 — a separate SES identity for QA

Ideally a different subdomain (`qa.saaradhigo.com`) so QA volume and bounces
never affect the production sending reputation.

### QA variable set

| Variable | QA value |
|---|---|
| `EMAIL_BACKEND` | `django.core.mail.backends.smtp.EmailBackend` |
| `EMAIL_HOST` | `email-smtp.ap-south-1.amazonaws.com` |
| `EMAIL_PORT` | `587` |
| `EMAIL_USE_TLS` | `True` |
| `EMAIL_HOST_USER` / `EMAIL_HOST_PASSWORD` | QA SMTP credentials |
| `DEFAULT_FROM_EMAIL` | `no-reply@qa.saaradhigo.com` |
| `AUTH_EMAIL_ALLOWLIST` | `@deeptrics.com` (or the specific QA addresses) |
| `ENVIRONMENT` | must **not** be `production` |

Applying Railway variables is a deploy, so this waits for the deploy gate.

#### Current state, verified 2026-10-17

The QA `backend` and `celery` services have **none** of these variables —
confirmed from each service's variable *names* in Railway (no values were
requested). So QA cannot send anything today, and every receipt and
password-reset has been failing into `Receipt.send_failure_reason` and the
log.

#### Apply them in this order. The order is the safety.

1. **`AUTH_EMAIL_ALLOWLIST` first**, before any credential exists. Setting it
   while nothing can send is free; setting it after the credentials means a
   window in which the deployment can reach anybody.
2. `ENVIRONMENT` — confirm it is not `production`.
3. `EMAIL_BACKEND`, `EMAIL_HOST`, `EMAIL_PORT`, `EMAIL_USE_TLS`,
   `DEFAULT_FROM_EMAIL`.
4. `EMAIL_HOST_USER` and `EMAIL_HOST_PASSWORD` **last**.

Between steps 3 and 4, and again after 4, run:

```
python manage.py email_safety_check --to <a QA address> --to <a non-QA address>
```

It sends nothing, prints no credential, and states plainly whether the
deployment can send and whether each address would be reached. An empty
allowlist reports "allowed to email nobody", which is the safe default and
not a misconfiguration.

> The one irreversible mistake in this whole runbook is setting the SMTP
> credentials before the allowlist. Nothing else here cannot be undone.

---

## 5. What QA can do before any of this exists

All three flows work end to end today with **no email at all** — see
`qa-test-accounts.md`:

| Flow | Without email |
|---|---|
| **Email verification** | A completed password reset sets `email_verified_at`, because receiving the link at that address is the proof. For a seeded QA account, set it directly in a shell. |
| **Password reset** | Mint `uid` + `token` with `manage.py shell` and POST them to `/auth/password/reset/`. The HTTP path is identical to the one a link would drive. |
| **Account invitation** | `POST /auth/admin/invite/` returns `invite_sent: false` and `email_delivery_configured: false`, then mint the set-password token the same way. |

The API is fully exercisable; only the delivery leg is missing.

---

## 6. Verification once configured

0. `manage.py email_safety_check --to qa@deeptrics.com --to
   someone@gmail.com` → reports that it can send, that the QA address is
   allowed and the other is refused. Do this first; it sends nothing.
1. `manage.py shell -c "from django.core.mail import send_mail;
   send_mail('t','t','no-reply@qa.saaradhigo.com',['qa@deeptrics.com'])"` →
   arrives.

   > **This call bypasses the allowlist.** `send_mail` is Django's own API;
   > the allowlist lives in `base/auth_email.send_message`, which every
   > application path goes through. Use it only to prove the SMTP credentials
   > work, and only ever with an address you own. Do **not** conclude
   > anything about the allowlist from it — step 2 is the test that does.
2. `POST /auth/password/forgot/` for an account whose address is **not** on
   the allowlist → nothing arrives and `auth_email_blocked` appears in the
   log. This goes through the real gate.
3. `POST /auth/password/forgot/` for an allowlisted QA account → link arrives,
   sets a password, works once.
4. Complete a trip → the receipt email arrives, with its PDF attached.
   *(This is the first time receipts will ever have been delivered.)* Receipts
   now go through the same gate as the auth emails, so a receipt for a rider
   whose address is not allowlisted is blocked on QA — which is the point, and
   something to expect rather than debug.
5. Check SES bounce/complaint metrics after the first batch.

---

## 7. Owner actions

| # | Action |
|---|---|
| 1 | Verify `saaradhigo.com` (and ideally `qa.saaradhigo.com`) in SES, with DKIM |
| 2 | Create SMTP credentials — **separate** ones for QA and production |
| 3 | Request production access for the **production** identity only |
| 4 | Set the QA variables in §4, including `AUTH_EMAIL_ALLOWLIST` |
| 5 | Add SPF / DMARC and a bounce destination |
| 6 | Confirm the region (`ap-south-1` recommended) |
