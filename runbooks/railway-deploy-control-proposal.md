# Railway deployment control — proposal (NOT APPLIED)

Written for: the owner, who has to approve each change before anyone makes it.

**Read from Railway on 2026-10-17, read-only.** Nothing in this document has
been applied. No Railway setting, variable, service or deployment was changed.

---

## 1. What actually triggers a deploy today

Deploys are **not** driven by GitHub Actions. Railway's own GitHub watcher is.
The GitHub `deploy` job in `SaaradhiGo-backend/.github/workflows/deploy.yml`
targets **EC2** and is currently failing on a missing `EC2_HOST_KEY` secret.

Verified per service with `get-service-config`:

| # | Service | Env | Repo | Branch | Waits for CI? | State |
|---|---|---|---|---|---|---|
| 1 | `backend` | QA | SaaradhiGo-backend | `dev` | **No** (`checkSuites: false`) | Live, serving `backend-qa-811d.up.railway.app` |
| 2 | `celery` | QA | SaaradhiGo-backend | `dev` | **No** | Live worker + beat |
| 3 | `alluring-happiness` | QA | SaaradhiGo-backend | `dev` | **No** | **Abandoned.** Zero variables. Deletion **staged but never applied** |
| 4 | `SaaradhiGo-backend` | production | SaaradhiGo-backend | `dev` | **No** | **Empty shell.** Zero variables, no domain. Deletion **staged but never applied** |
| 5 | `web` | QA | SaaradhiGo-web | `develop` | **No** | Next.js ops console, live at `web-qa-800c.up.railway.app` |

### Three things that follow from that table

1. **One push to `dev` triggers four builds, not one.** Two of them are
   abandoned shells that nobody reads. My own previous session pushed five
   commits to `dev` — that is up to twenty service builds for five commits.

2. **`checkSuites: false` everywhere.** Railway does not wait for GitHub
   checks, so a push whose tests fail still deploys to QA. The test suite is
   not currently a gate on anything.

3. **The production shell deploys from `dev`.** It has no variables so it
   cannot boot, which is the only reason this has been harmless. A production
   service pointed at a development branch is not a configuration anyone
   should keep, in either direction.

Also worth knowing: the deletions staged on services 3 and 4 mean someone
already decided they should go and the change was never applied. Applying a
staged change is a Railway action I have deliberately not taken.

---

## 2. Proposed changes, in priority order

Each is independent. Each needs your approval. None has been applied.

### P0 — Stop `dev` from deploying the two dead services

**Change:** apply the already-staged deletion of `alluring-happiness` (QA) and
`SaaradhiGo-backend` (production).

**Why:** they are zero-variable shells that cannot run, and each still consumes
a build on every push to `dev`. This is pure waste with no rollback cost —
though it is a deletion, so it is yours to confirm, and a staged patch already
proposes exactly this.

**If you would rather not delete them:** disconnect the repo from each service
instead. That stops the builds and keeps the shells.

### P0 — Make CI a gate

**Change:** set `checkSuites: true` on `backend` and `celery` (QA).

**Why:** one toggle per service, and the only change here that improves
correctness as well as cost — a red build stops instead of shipping. The
backend suite is ~1,200 tests and already runs on every push.

**Caveat worth stating:** the GitHub workflow's `deploy` job currently fails on
the missing `EC2_HOST_KEY`. If Railway waits for *all* check suites, that
failing job will block QA deploys. So this change should be made **together
with** either fixing `EC2_HOST_KEY` or removing/disabling the EC2 deploy job —
otherwise QA stops deploying and the toggle gets reverted in frustration.

### P1 — Freeze the Next.js console's deploys

**Change:** disconnect `SaaradhiGo-web` from the `web` service, or point it at
a branch nobody pushes.

**Why:** `runbooks/operations-console-decision.md` already ruled the Django
console canonical for the pilot and this console not pilot-ready. It has no
tests, and it has received none of the multi-country work, so it still renders
money without a currency. Leaving it auto-deploying means an operator can find
it and use it.

### P1 — Branch discipline, so `dev` stops being the working branch

**Change (process, not Railway):**

- Feature work on `feat/*` branches. PR CI runs lint, tests,
  postgres-concurrency, the debug APK and the new `release-config` job — all at
  **zero** Railway spend, because no Railway service watches a feature branch.
- Merge to `dev` only when a batch is ready to be exercised on QA. One deploy
  per batch.
- Production deploys on written sign-off, never automatically.

This batch is already on `feat/android-pilot-batch1` in all three repos and has
not been pushed.

### P2 — Watch the other two cost drivers

- **Google Maps Distance Matrix** is called on every fare quote and on the
  fallback path. A quota alarm before the pilot is cheaper than a surprise.
- **AWS SNS** per OTP. The test-phone bypass sends no SMS and is now covered by
  a test, which keeps QA runs free.

---

## 3. What I did not do, and why

| Not done | Why |
|---|---|
| Changed any Railway setting | You said not to without approval |
| Applied the staged deletions | Same, and a deletion is irreversible |
| Pushed any branch | Pushing `dev` deploys four services; pushing a feature branch is safe but still your call |
| Deployed anything | Out of scope for this batch |
| Enabled live payments or production | Explicitly held |
| Read any secret's value | The variable *names* are listed by the API; I did not fetch values, including the QA bootstrap credentials |

---

## 4. One consolidated QA deployment plan

When you want this batch on QA, in this order:

1. **Decide the two P0 Railway items above.** Doing them first means step 4
   deploys two services instead of four.
2. **Push the three feature branches.** PR CI runs everywhere; no Railway
   deploy. Confirm green: backend lint + tests + postgres-concurrency, both
   apps' analyze + test + debug APK + `release-config`.
3. **Add the Android signing secrets** (`ANDROID_KEYSTORE_BASE64`,
   `ANDROID_KEYSTORE_PATH`, `ANDROID_KEY_ALIAS`, `ANDROID_KEY_PASSWORD`,
   `ANDROID_STORE_PASSWORD`) if you want a signed APK out of this round. The
   batch works without them — the release build simply refuses.
4. **Merge all three branches to `dev` together**, then let the single QA
   deploy happen. One deploy, not five.
5. **Run the device checklist** in `runbooks/android-device-test-checklist.md`.
   Everything marked UNKNOWN in this batch is in there, and none of it can be
   cleared from this machine.
6. **Re-run `qa/acceptance_ride.py`** once the QA OTP behaviour is confirmed
   stable (see the diagnosis note in the batch report). It needs
   `ACCEPTANCE_RIDER` / `ACCEPTANCE_DRIVER` from `TEST_PHONE_NUMBERS`.
7. **Production stays off** until the checklist is clear and you sign off.
