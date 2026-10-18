# QA test accounts, and the deployment that activates them

Written for: the owner, setting up accounts to validate SaaradhiGo by hand.

Two things are on this page: how to give yourself working Admin, Rider and
Driver logins without a password ever appearing in a chat message or in the
repository, and the single consolidated deployment that makes the new
sign-in journeys live.

---

## 1. Why there is a procedure rather than a list of credentials

A test account whose password was pasted into a chat is a working credential
in a chat log, a terminal history and a ticket. For a QA environment that
holds real-shaped data and talks to the same payment and storage services as
production, that is the wrong trade even when it is convenient.

So the password is **set by you, in a Railway variable, and read from the
environment**. Nothing prints it: not the command, not the logs, not masked,
not at DEBUG. If a variable is missing the command refuses rather than
inventing a default, because a built-in password is how a test account
becomes an incident.

---

## 2. Set the variables (Railway → QA → `backend`)

Nine variables, three per role. Pick the passwords yourself; 12 characters
minimum is enforced.

| Role | Phone | Email | Password |
|---|---|---|---|
| Admin | `QA_ADMIN_BOOTSTRAP_PHONE` | `QA_ADMIN_BOOTSTRAP_EMAIL` | `QA_ADMIN_BOOTSTRAP_PASSWORD` |
| Rider | `QA_RIDER_BOOTSTRAP_PHONE` | `QA_RIDER_BOOTSTRAP_EMAIL` | `QA_RIDER_BOOTSTRAP_PASSWORD` |
| Driver | `QA_DRIVER_BOOTSTRAP_PHONE` | `QA_DRIVER_BOOTSTRAP_EMAIL` | `QA_DRIVER_BOOTSTRAP_PASSWORD` |

`QA_ADMIN_BOOTSTRAP_PHONE` and `QA_ADMIN_BOOTSTRAP_PASSWORD` already exist —
keep them. Use addresses on a domain you control; `@deeptrics.com` is the
one already allowlisted for QA mail.

Use numbers that are **not** real customer numbers and not your own, so a
stray notification cannot reach a stranger.

---

## 3. Run the command

```bash
railway ssh --project <project> --environment QA --service backend \
  -- python manage.py bootstrap_qa_accounts
```

What it does:

* creates the three accounts, or converges them if they already exist;
* sets each password **only on creation**, so re-running it after a redeploy
  cannot silently rotate a credential you are using. To change one
  deliberately, update the variable and add `--reset-password`;
* marks each email **verified**, because shell access to the deployment is a
  stronger proof of control than an inbox — and without it `email_login`
  refuses the account and you could not sign in at all;
* gives the operator `is_staff`, and never gives it to the rider or driver;
* **refuses to run in production**, and treats an unlabelled non-debug
  deployment as production.

It will not re-role an account that already exists with a different role. It
reports that and stops, because a silent role change is privilege escalation.

Expected output is a short summary naming which accounts were created or
converged, and the line *"No password was printed by this command."*

---

## 4. Where to sign in

| | URL | Identifier |
|---|---|---|
| **Admin** | `https://backend-qa-811d.up.railway.app/login/` | Email **or** phone, plus password |
| **Rider** | `https://web-qa-800c.up.railway.app/sign-in` | Email + password |
| **Driver** | `https://web-qa-800c.up.railway.app/driver/sign-in` | Email + password |

The Admin console accepts either identifier now: `ModelBackend` still handles
the phone, and `EmailOrPhoneBackend` handles anything containing `@`.

---

## 5. What a new customer does, and what you should see

1. `/sign-up` — name, mobile, email, password, confirm.
2. The server creates the account **unverified** and emails a link to
   `/verify-email?uid=…&token=…`.
3. Until that link is opened, signing in returns `AUTH_EMAIL_UNVERIFIED` and
   the page says to confirm the address — not "invalid credentials", which
   would send people to reset a password that was never wrong.
4. After verifying, email + password signs in.

**On QA, step 2's email will not arrive**: `AUTH_EMAIL_ALLOWLIST` is empty and
no SMTP is configured, so the outbound gate blocks everything. That is the
intended safety posture, not a fault. The three accounts above work anyway
because the command marks them verified. To exercise the verification link
itself, either complete the SES setup in
`qa-external-integrations-checklist.md`, or mint a link from the shell.

---

## 6. The consolidated QA deployment

One deployment, two services, in this order. **It needs explicit approval
before it runs.**

### What it carries

| Change | Why it needs deploying |
|---|---|
| `POST /auth/email/register/` | The public sign-up form has nothing to call without it |
| Verification required at `email_login` | Makes the verification step real |
| `EmailOrPhoneBackend` | Admin console sign-in by email |
| `bootstrap_qa_accounts` | The command in §3 |
| Admin login template | SaaradhiGo Admin branding, email/phone field |
| Session revocation on credential change | From the previous batch, still undeployed |

### Migrations

**None.** `makemigrations --check` reports no model changes, so the
`preDeployCommand` migrate step is a no-op and rollback is a deployment
rollback only.

### Sequence

1. Pre-check: confirm the staged destructive patch `9dc1ab52-…` is still
   `STAGED` and **do not accept it** — it deletes `alluring-happiness`.
2. Deploy `backend` **first and alone** (it is the only service that
   migrates), pinned to the commit, using the per-service manual trigger in
   `android-pilot-overnight-readiness.md` §6. Never `accept-deploy`.
3. Wait for `/healthz` to report `status: ok`.
4. Deploy `celery` to the same commit.
5. Deploy `web` to the website commit.
6. Run §3 to create the accounts.
7. Sign in at all three URLs in §4.

### Rollback

Roll each service back to its previous deployment. No migration to reverse.
The three QA accounts are additive — if you roll back, they remain and simply
cannot use the email-login path until the backend is forward again.

---

## 7. What is still blocked, and by what

| Item | Blocked on |
|---|---|
| Verification and reset **emails** actually arriving | SES (allowlist first, credentials last) |
| Push notifications in either app | Firebase service account + `google-services.json` |
| Destination search, route, ETA | A restricted QA Maps **server** key |
| Any signed APK | Keystore backup verification |
| Public sign-up by phone (OTP) | Deliberate: `NEXT_PUBLIC_ALLOW_OTP_SEND` is off so a public form cannot spend money on SMS. Email sign-up works without it |
