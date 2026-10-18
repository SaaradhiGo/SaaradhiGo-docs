# QA test-driver approval: what exists, and the smallest safe thing that would work

**Status:** design only. Nothing in this document is implemented.
**Decision required from the owner.**

## The question

Certifying a ride end to end needs exactly one dispatchable driver in QA.
Today there are none, and `Driver #8` (the designated QA driver) carries
placeholder KYC documents that must not be approved as a genuine verification.

## 1. Does an isolated QA-only approval mechanism already exist?

**No.** Three code paths set `approved = True`, and none of them is usable
here:

| Path | What it is | Why it does not apply |
|---|---|---|
| `admin_dashboard` console → `KYCApprovalSerializer` | The real operator approval. Enforces the MVA-2020 document checks. | This is the control we must not weaken. Using it would record a real approval over placeholder documents. |
| `seed_load_fixtures` | Creates approved drivers for load runs. | Refuses unless `ENVIRONMENT == 'load'`. Its docstring names QA explicitly as somewhere it must never run. |
| `DriverProfileSerializer` | The driver's own profile endpoint. | `approved` and `status` are in `read_only_fields` precisely to stop self-approval. |

So the answer to "if yes, use it after proving production cannot reach it" is
moot. The remainder of this document is the "if not" branch: the smallest safe
QA-only simulation.

## 2. Why this cannot simply be done by hand

Three requirements interact:

1. `Driver.approved` must be true, and `active_vehicle` must be set, or the
   driver app refuses to go online (*"You can go online once approved"*,
   *"Add an active vehicle before going online"*).
2. The approval must never be mistakable for a real KYC review — not in the
   console, not in an audit, not by a future reader of the database.
3. Whatever opens this door must be provably shut in production.

A direct database edit fails (2) and (3). The console fails (2).

## 3. The smallest safe mechanism

A single management command, `qa_approve_test_driver`, with no HTTP surface —
so it cannot be reached by any request, authenticated or not, in any
environment. It would:

**Refuse unless all three hold**, each sufficient on its own to stop it:

1. `settings.ENVIRONMENT == 'qa'` exactly.
   This is the load-bearing guard, and it fails safe: `base/settings.py:574`
   defines
   `ENVIRONMENT = os.environ.get('ENVIRONMENT', 'production' if not DEBUG else 'development')`,
   so a deployment that simply does not set the variable is treated as
   production and refused. The proof that production cannot reach this
   mechanism is that production would have to *opt in* by setting
   `ENVIRONMENT=qa`, which would also change the behaviour of every other
   environment-gated control in the codebase.
2. `settings.IS_PRODUCTION` is false — a second, independent read, so a future
   third value of `ENVIRONMENT` cannot land in an unguarded state.
3. The target driver's phone number is inside a QA test-account prefix range.
   Even inside QA, the command cannot touch a driver who was not provisioned
   as a test account.

No `--force`, no environment-variable override, no flag that relaxes any of
the three. The only path to approving a real driver stays the console.

**What it would set:** `approved = True`, `doc_status = 'approved'`, and
nothing else. Deliberately *not* `status = 'online'`: going online is the
driver app's job because the app owns the location socket, and a driver marked
online in the database with no socket is one dispatch will offer trips to and
who will never answer.

**What it would record:** an `AdminAuditLog` row with
`action = 'qa_test_driver_made_dispatchable'`, a null actor, and a reason
stating in words that no document was verified and that the documents on the
account are QA placeholders. The point is that the row survives: anyone later
asking "was this driver actually vetted?" finds the answer attached to the
driver, not in a chat log.

**Reversible:** a `--revert` flag returning the driver to pending and offline,
so QA can put the environment back.

**Tests that would accompany it:** that it refuses when `ENVIRONMENT` is
`production`, `load`, empty, or absent; that it refuses a driver outside the
test prefix range even in QA; that it refuses a driver with no active vehicle;
that it writes the audit row; and that `--revert` restores the prior state.

## 4. Honest assessment of the residual risk

This mechanism still produces a driver whose `doc_status` reads `approved`
over placeholder documents. The audit row and the phone-prefix restriction
make that visible and bounded, but they do not make it untrue. Anyone reading
`Driver.doc_status` alone, without the audit log, would see an approved
driver.

If that is unacceptable, the alternative is to add a distinct state — e.g. a
`is_qa_test_account` boolean on `Driver`, set by this command and surfaced as
a badge in the console — so the test driver is visibly different everywhere a
driver is displayed. That is a schema change and a migration, which is why it
is offered as an option rather than assumed.

## 5. What is being asked

One of:

- **A.** Approve building `qa_approve_test_driver` as described in §3.
- **B.** Approve §3 plus the `is_qa_test_account` flag in §4, accepting a
  migration.
- **C.** Supply genuine documents for the QA driver and approve through the
  console as a real driver.
- **D.** Leave QA with no dispatchable driver; the ride stays uncertified.

Under A or B, the ride certification can proceed immediately afterwards: the
driver app now reaches QA, the vehicle is registered and active, and browser
booking already reaches `trip_created`.
