# QA test accounts — setup without relying on SMS

Written for: QA, and the operator who creates the accounts.

Two independent ways in. Use whichever is available; neither requires a real
SMS to arrive, and neither needs a password to be written down or pasted
anywhere.

---

## Route A — phone + OTP bypass (works today, no configuration needed)

Already live on QA. `TEST_PHONE_NUMBERS` maps a phone to a fixed OTP and the
backend **skips the SNS call entirely** — confirmed in the service log:

```
OTP test-phone bypass for +9199*** — no SMS sent
```

| Account | Phone | OTP |
|---|---|---|
| Rider | `+919999000001` | `100001` |
| Driver | `+919999000002` | `100002` |

These are not secrets: they are operator-chosen OTP-bypass pairs, they only
work on an environment whose `TEST_PHONE_NUMBERS` lists them, and they are the
documented example in `base/settings.py`. **Never add a real customer's number
to that variable.**

Verified working against deployed QA during this batch — both accounts
returned a 256-character token.

In the apps: open the **Mobile** tab (the default), enter the number, then the
OTP.

### What Route A cannot test

- Email + password login, obviously.
- Password reset, change, or email verification.
- Anything about an operator account, which needs a password and MFA.

---

## Route B — email + password (needs an operator, and email for reset)

### B1. An operator creates the account

`POST /api/v1/auth/admin/invite/` as an operator:

```json
{ "email": "qa-rider@yourdomain.test",
  "role": "rider",
  "full_name": "QA Rider" }
```

What it does and does not do:

- Creates the account with the role, and a rider/driver profile row so it
  behaves like an OTP-created one.
- Sets an **unusable password**. Nothing is generated, returned, logged or
  stored — the response contains no password field and no password value.
- Emails a one-time set-password link.
- Writes an `AdminAuditLog` row (`account_invited`).
- Assigns a **placeholder phone number** (`invite-<random>`) when none is
  given. It is deliberately not dialable, so the account cannot request an
  OTP. Supply `phone_number` if you want the account usable on the Mobile tab
  too.

Who may invite whom:

| Caller's `ops_role` | May create |
|---|---|
| `admin` | rider, driver, **operator** (and must have cleared MFA on the session) |
| `support`, `driver_ops`, `finance` | rider, driver only |

A non-operator gets 403. An address or phone that already belongs to an
account gets 409 — it is never re-pointed, because silently changing an
existing account's role is the escalation this endpoint must not have.

### B2. The invitee sets their own password

From the emailed link, or directly:

```
POST /api/v1/auth/password/reset/
{ "uid": "<from the link>", "token": "<from the link>",
  "new_password": "..." }
```

Using the link both sets the password **and verifies the address**, because
receiving it there is the proof of control. The token is single-use and
expires in 2 hours (`PASSWORD_RESET_TIMEOUT`).

### B3. Sign in

In the apps: **Email** tab → address + password. No OTP step; the password was
the proof.

```
POST /api/v1/auth/email/login/
{ "email": "...", "password": "..." }
```

Operators must also send `mfa_code`. The app shows that field only after the
server has asked for it, which it does only once the password is accepted.

---

## ⚠ Route B is blocked until email delivery is configured

**No email will arrive.** There is no `EMAIL_BACKEND`, `EMAIL_HOST` or
`DEFAULT_FROM_EMAIL` in settings or in the QA environment, so Django falls back
to SMTP on `localhost:25`, which does not exist on Railway. Receipt email has
the same gap and has been failing silently.

The code does **not** pretend otherwise: `base/auth_email.py` logs
`auth_email_undeliverable` at ERROR and the invite response says
`"invite_sent": false` with an instruction.

### Owner action required

Set on the `backend` service (and later production):

| Variable | Value |
|---|---|
| `EMAIL_HOST` | e.g. `email-smtp.ap-south-1.amazonaws.com` |
| `EMAIL_PORT` | `587` |
| `EMAIL_HOST_USER` | SES SMTP username |
| `EMAIL_HOST_PASSWORD` | SES SMTP password |
| `EMAIL_USE_TLS` | `True` |
| `DEFAULT_FROM_EMAIL` | e.g. `no-reply@saaradhigo.in` |

SES is the natural choice — the stack already uses SES for SMS — and it needs
a **verified sender domain**, which only the owner can do. Note that applying
Railway variables is a deploy, so it waits for the deploy gate.

### Until then, two workarounds for QA only

1. **Set the password from the server**, without email:
   ```
   railway run python manage.py shell -c "
   from django.contrib.auth import get_user_model
   u = get_user_model().objects.get(email='qa-rider@yourdomain.test')
   u.set_password(input('password: ')); u.email_verified_at=__import__('django.utils.timezone',fromlist=['timezone']).timezone.now(); u.save()"
   ```
   Type the password at the prompt so it is not in shell history or in this
   file. Do this only for QA accounts, never for a real user.

2. **Mint the uid/token directly** and complete B2 over HTTP:
   ```
   railway run python manage.py shell -c "
   from django.contrib.auth import get_user_model
   from servers.auth_user.password_views import reset_tokens, _uid
   u = get_user_model().objects.get(email='qa-rider@yourdomain.test')
   print(_uid(u), reset_tokens.make_token(u))"
   ```

Both run against the deployed image and need no code change — but both are a
**shell on QA**, so they need the same approval as any other QA action.

---

## What to check once QA accounts exist

Full list in `android-device-test-checklist.md`. The auth-specific ones:

- [ ] Rider: Mobile tab OTP login still works (regression — it is the primary
      route).
- [ ] Rider: Email tab login works, and shows **one** generic message for a
      wrong password.
- [ ] Rider: "Forgot password?" shows the same message for a known and an
      unknown address.
- [ ] Driver: both tabs, same checks. **Note:** driver OTP login was broken on
      the rebuild branch until this batch — it read `access`/`refresh` while
      the API sends `token`/`refresh_token` — so this is a first-time
      verification, not a regression check.
- [ ] Operator: console login still requires password **and** TOTP.
- [ ] Operator: an email login without `mfa_code` is refused with
      `AUTH_MFA_REQUIRED`.
- [ ] Support-level operator cannot create an operator account (403).
- [ ] Token refresh keeps a session alive past 15 minutes — **also a
      first-time check on the driver app**, for the same reason.
