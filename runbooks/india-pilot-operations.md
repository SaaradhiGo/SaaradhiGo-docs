# India pilot operations runbook

- **Date:** 2026-09-24
- **Audience:** whoever is on duty while real riders and real drivers are using
  SaaradhiGo.
- **Scope:** one city, small driver cohort, cash-first.

## Rule zero

**Never edit a financial table by hand.** Not `wallet_transaction`, not
`trip_settlement`, not `payment`, not `receipt`. Every one of them is either a ledger
or immutable evidence, and a manual edit destroys the only record of what actually
happened. If money looks wrong, collect evidence and escalate to engineering — a
correction is a new row written by code, never an `UPDATE` typed by a person.

Reading is always fine. Changing is not.

## Before the first ride each day

Work down the list. Anything that fails stops driver onboarding until it is fixed.

| Check | How | Healthy looks like |
|---|---|---|
| Backend up | `GET /healthz` | HTTP 200 |
| Database | `/healthz` covers it | 200 (health fails if the DB is unreachable) |
| Redis | open the ops console; live driver map populates | drivers appear when they go online |
| Celery worker | a completed test ride produces a receipt | receipt arrives within a few minutes |
| Celery Beat | GPS trail rows appear for a finished ride | `persist_location_trail` runs every 60s |
| S3 | open one driver's KYC document | presigned URL opens |
| Cashfree | **do not test in production.** Confirm the dashboard shows the account live | account active |
| Notifications | request a test ride; the driver's phone buzzes | push arrives |
| App versions | confirm the APK in drivers' hands is the intended build | the environment badge is ABSENT (a badge means it is NOT a production build) |

That last row matters. A non-production build wears a visible environment label in
the top-right corner. If you can see one on a driver's phone, that phone is talking to
QA and none of its rides are real.

## Driver onboarding

1. Driver installs the app, signs in with OTP.
2. Driver submits KYC documents and vehicle details in-app.
3. Ops reviews in the console: licence, RC, permit, insurance, fitness, PUC.
4. Check expiry dates. An expired document is a refusal, not a warning.
5. Approve through the console's approval action. **Never** set `approved` directly
   in the database — the audited endpoint writes the audit trail, and a manual flip
   does not.
6. Driver goes online and should appear on the live map within a few seconds.

If they approve but never appear online: their geo-index entry is written in the
background after they go on duty. Wait ten seconds, then have them toggle offline and
online again before escalating.

## During an incident

### Ride stuck in `requested` (no driver found)

1. Open the trip in the console. Look at the dispatch history.
2. Were any drivers notified? If zero, the problem is supply or the geo index, not
   dispatch.
3. Check the live map for online drivers near the pickup.
4. The trip auto-cancels on its own deadline; you do not need to force it.
5. Tell the rider plainly that no driver was available. Do not offer a fee waiver —
   there is no cancellation fee to waive.

### Ride stuck in `accepted` — driver not moving

1. Call the driver. Most of the time this is a driver problem, not a software one.
2. If unreachable, cancel as the driver from the console and tell the rider.
3. Note the driver for follow-up. Repeat non-arrival is a cohort issue.

### Driver says the app lost the ride

1. Have them force-close and reopen. The app reconstructs trip state from the server
   on reconnect — the socket greeting carries the durable status.
2. Confirm in the console what state the trip is actually in. **The server is right
   and the app is wrong**, always, in this situation.
3. If the app still disagrees after a restart, that is an engineering escalation with
   the trip id.

### Completion acknowledgement missing — driver says "it won't complete"

1. Check the trip's status in the console first. If it says `completed`, the ride is
   finished and only the driver's screen is behind; have them reopen the app.
2. If it says `in_progress`, have them tap complete again. **Retrying is safe** —
   the server answers `already_done` for a command that already landed and will not
   settle twice.
3. If it still will not complete, check whether their socket is connected (the app
   shows connection state). Escalate with the trip id.

### GPS trail missing for a finished ride

1. This does not affect the fare. Fares are quoted, not metered — `final_fare` is
   NULL by design.
2. It does affect dispute evidence. Note the trip id.
3. Check whether Celery Beat is running; the trail is drained on a 60-second
   schedule, not in real time.

### Payment shows pending

1. Cash ride: the driver has not confirmed collection. Ask them to tap the cash
   confirmation. That step is what settles the ride.
2. Online ride: check the payment record in the console. Do not mark it paid by hand.
3. Never tell a rider a payment succeeded because the app said so. Read the server.

### Payout unresolved

1. Look up the withdrawal in the console.
2. `processed` means we dispatched it to the provider. It does **not** mean the
   driver has the money.
3. **Do not retry it.** The provider contract is unverified and a retry could
   plausibly pay twice. Escalate to engineering with the withdrawal id.
4. Tell the driver it is being checked with the bank. Do not promise a timeline.

### SOS

1. **Treat it as real until proven otherwise.** Never assume a mis-tap.
2. Open the SOS queue. Note the trip, who raised it, and the timestamp.
3. Call the person who raised it. If no answer, call the other party on the trip.
4. Escalate to emergency services per the business's stated policy. That policy is a
   business decision and must exist before the pilot opens — engineering cannot
   supply it.
5. Acknowledge the event in the console so the queue reflects reality.
6. Repeated presses of one panic button collapse into a single event by design. One
   event does not mean one press, and it does not mean the situation is smaller.
7. Record what happened. The SOS record is safety evidence.

### Receipt failed

1. Completion is never rolled back by a receipt failure — the ride is finished and
   the money is settled regardless.
2. Use the resend action in the console. Retries are idempotent and will not change
   the amounts.
3. If it fails repeatedly, S3 or email is the problem. Escalate; the rider can be
   sent the fare details manually in the meantime.

## Deployment

1. Deployments happen from `dev` to QA automatically. Production deployment does not
   exist yet.
2. **Freeze deployments during any live rehearsal or pilot window.** A deployment
   restarts the backend and kills every open WebSocket, so every in-flight ride
   loses its socket. This was observed directly: a 20-minute test ride was
   invalidated by an unrelated documentation commit.
3. A migration that fails blocks the rollout and the previous healthy revision keeps
   serving. That is proven, not assumed.
4. Never deploy a destructive migration in the same release as the code that stops
   using the old schema. Additive first, then the code change, then cleanup.

## Rollback

1. Roll back the deployment through the platform's own rollback, not by reverting
   commits under time pressure.
2. **A rollback does not undo a migration.** Every migration shipped so far is
   additive, which is what makes rollback safe; keep it that way.
3. After a rollback, verify `/healthz`, then complete one test ride end to end before
   letting drivers back on.

## What to collect before escalating

Always, and never more than this:

- trip id, and withdrawal or payment id if relevant
- what the console says the state is
- what the app said instead
- rough timestamp
- driver id and rider id — **ids, not names or phone numbers**

Never paste a phone number, an OTP, a JWT, a document URL or GPS coordinates into a
ticket or a chat. If you need the location of an incident, reference the trip id and
let someone with the right access read it.
