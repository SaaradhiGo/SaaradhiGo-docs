# Standard Operating Procedure — Driver Onboarding & Verification

**Compliance basis:** Motor Vehicles Aggregator Guidelines, 2020 (MVA 2020)
issued by the Ministry of Road Transport & Highways, the Motor Vehicles Act,
1988, Telangana state motor vehicle rules, and SaaradhiGo's Driver
Agreement.

- **Audience:** Operations team (KYC reviewers, ops manager)
- **Document version:** {{POLICY_VERSION}}
- **Effective from:** {{LAUNCH_DATE}}
- **Owner:** Operations Manager
- **Reviewed by:** [Legal counsel + Compliance — sign and date]

> **DRAFT — REQUIRES OPERATIONAL REVIEW**
>
> This SOP describes the process Ops staff must run for every prospective
> Driver Partner before they can accept rides on the Platform. It is
> deliberately strict to ensure MVA 2020 compliance.

---

## 1. Purpose

To ensure that every Driver Partner SaaradhiGo onboards has the legal right
and operational ability to provide commercial passenger transport, in
compliance with MVA 2020 and Telangana motor vehicle rules, before they are
granted access to accept ride requests.

## 2. Outcomes a successful onboarding must achieve

1. The driver is **identifiable** beyond reasonable doubt.
2. The driver is **legally allowed to drive commercially** in Telangana.
3. The vehicle is **legally allowed to operate as a contract carriage** in
   Telangana.
4. The driver has **no disqualifying criminal record** within the period
   covered by police verification.
5. The driver has **agreed to** the Driver Agreement, Terms of Service, and
   Privacy Policy.
6. The driver has working **payout details** on file.
7. SaaradhiGo has retained **auditable proof** of all the above.

## 3. Process overview

```
1. Application
   ↓
2. Document collection
   ↓
3. Document verification (manual + automated)
   ↓
4. Background check (third party)
   ↓
5. In-person verification at HQ (Phase-0)
   ↓
6. Vehicle inspection
   ↓
7. Orientation + Agreement signing
   ↓
8. Activation in driver app
```

Target SLA from Application → Activation: **5 working days** (Phase-0).

## 4. Steps in detail

### Step 1 — Application

- The prospective driver applies via the driver app (Apply mode) or via a
  recruitment lead-form on the marketing site.
- Captured at intake: full legal name, mobile number, email (optional),
  date of birth, the city they intend to operate in, the vehicle type
  (auto / hatchback / sedan / SUV), and a primary device platform
  (Android / iOS).
- The application creates a pending Driver record in the database with
  `approved=False` and `status='off'`.

### Step 2 — Document collection

The driver uploads all of the following via the driver app, or brings
physical copies to the HQ session (preferred for Phase-0):

| Document | Required content | Acceptable format |
|---|---|---|
| Driving Licence | Both sides; commercial endorsement visible | PDF / JPG |
| RC of vehicle | Yellow board / commercial endorsement visible | PDF / JPG |
| Permit | Telangana taxi/auto permit OR all-India tourist permit | PDF / JPG |
| Insurance | Comprehensive policy, current; passenger cover present | PDF |
| Fitness Certificate | RTO-issued, current | PDF / JPG |
| PUC Certificate | Current | PDF / JPG |
| Police Verification | Issued by an accredited agency; not older than 12 months at intake; not older than 24 months at the time of operation | PDF |
| Self-declaration of medical fitness | Signed by driver | PDF / JPG (signed) |
| Aadhaar (front, with name & DOB visible; last 4 digits captured only) | For identity match | PDF / JPG |
| PAN | For TDS | PDF / JPG |
| Photograph | Recent passport-style colour photo | JPG |
| Bank passbook OR UPI VPA confirmation | Name on file must match the driver's legal name | PDF / Screenshot |

### Step 3 — Document verification

For each document, the reviewer:

1. Visually inspects clarity and legibility. Reject if any field is
   illegible.
2. Cross-checks names and dates across documents (driving licence vs RC
   vs Aadhaar vs PAN vs bank).
3. Validates issuing authority — only state RTOs, Government of India, and
   recognised authorities are acceptable.
4. Checks expiry dates. All operationally-relevant dates must be at least
   90 days in the future at the time of onboarding.
5. Records the document hash in SaaradhiGo's KYC system (via the driver
   app upload); the file itself is stored in encrypted private storage.

**Automated checks** (where available):

- RTO licence number verification (via the Vahan / Sarathi APIs or
  partnered KYC provider) — should return matching name + DOB.
- PAN verification via NSDL e-KYC (returns name only — must match).
- Bank account verification via NPCI / penny-drop (₹1 token credit).

Where an automated check fails or is unavailable, the manual cross-check
is the decision basis.

### Step 4 — Background check

Engage the contracted background-check partner (TBD pre-launch) to run:

- Criminal record check across nationwide databases
- Court records check
- Drug screen (recommended; optional in MVA 2020 but considered best
  practice)
- Address verification (residence)

Disqualifying findings under MVA 2020:

| Finding | Disqualifying? |
|---|---|
| Conviction for a crime involving violence, fraud, or sexual offence within the last 7 years | Yes — automatic rejection |
| Conviction for a traffic-related offence resulting in death or grievous hurt within the last 5 years | Yes |
| Pending criminal proceeding for any of the above | Yes — postpone decision pending resolution |
| Drug-screen positive | Yes |
| Minor traffic violations only | No |

### Step 5 — In-person verification

For Phase-0, every prospective driver attends a single 1-hour in-person
session at the SaaradhiGo HQ at {{PRINCIPAL_OFFICE_ADDRESS}}:

1. Selfie photograph captured against a reference background.
2. Original documents presented + sighted against the uploaded copies.
3. Brief operational interview: route familiarity, language, English /
   Telugu / Hindi proficiency, smartphone proficiency.
4. Driver's identity verified against Aadhaar / DL.

### Step 6 — Vehicle inspection

For Phase-0, every vehicle is physically inspected at HQ or at a
designated workshop:

| Checkpoint | Standard |
|---|---|
| Exterior condition | No major dents, no broken glass, working lights front + back |
| Interior condition | Clean, no offensive odour, working AC (for AC-category) |
| Seat belts | Functional for all seats |
| Mileage | Confirm odometer reading matches RC |
| Tyres | Adequate tread (>1.6mm) |
| First-aid kit | Present, in-date |
| Fire extinguisher | Present, in-date |
| Vehicle ID display | Permit number visible |
| Engine number + chassis number | Match the RC |
| Photographs taken | All four sides + interior |

Where the vehicle fails inspection, the driver is given a 15-day window to
rectify; re-inspection follows.

### Step 7 — Orientation + Agreement signing

A 30-minute orientation covers:

- How the driver app works (accept, navigate, OTP, complete, rate)
- SOS button behaviour and when to use it
- Behavioural code of conduct (Section 3 of Driver Agreement)
- Payment flows: cash vs online, payout cadence, weekly settlement
- TDS u/s 194O and what the driver sees on their settlement
- Cancellation policy
- Surge pricing
- Grievance and appeal process

After orientation, the driver e-signs:

- Driver Agreement (with Schedule A filled in)
- Terms of Service
- Privacy Policy

E-signatures are captured via the driver app, time-stamped, IP-logged, and
the signed PDFs are archived.

### Step 8 — Activation

Ops admin uses the admin web (when available) or Django admin to:

1. Mark `Driver.approved = True`.
2. Mark `Driver.status = 'off'` (driver chooses when to go online).
3. Confirm `active_vehicle` is set to the inspected vehicle.

The system's KYC document gate (PR #33) refuses the approval unless:

- `license_doc` is uploaded
- `license_expiry` is in the future
- `active_vehicle` is set
- The active vehicle has `rc_doc` uploaded

Every approval action writes an `AdminAuditLog` row with the approver's
identity. The daily expiry sweeper monitors all expiry dates from this
point forward and will auto-block the driver if any credential expires.

## 5. Post-onboarding monitoring

- **Daily**: the expiry sweeper (`driver.block_expired_driver_licenses`)
  blocks any driver whose licence or active-vehicle credentials have
  expired.
- **Monthly**: ops reviews the deactivated-by-expiry list, contacts those
  drivers, and walks them through re-uploading current documents (a
  shortened version of Steps 2-3).
- **Every 12 months**: police verification is renewed. The driver is
  reminded 60 days in advance.
- **Per-trip**: the driver's `approved` and `status != 'blocked'` are
  re-checked inside the ride-accept transaction (PR #15). No grace
  period.
- **Per-rating**: drivers below 4.0 average for 30 consecutive days
  trigger a coaching workflow; below 3.5 for 30 days triggers
  deactivation per Schedule B of the Driver Agreement.

## 6. Records and retention

| Record | Retention |
|---|---|
| Onboarding application | 7 years post-deactivation |
| Uploaded documents | While active + 7 years post-deactivation |
| Background check report | While active + 7 years (subject to background-check provider's retention policy) |
| In-person verification photo | While active + 7 years |
| Vehicle inspection report + photos | While active + 7 years |
| Signed Driver Agreement, ToS, Privacy Policy | While active + 7 years |
| AdminAuditLog rows | 10 years (immutable; not deletable by ops) |

## 7. Roles and responsibilities

| Role | Responsibility |
|---|---|
| Ops Reviewer (KYC) | Steps 2-4, document checks, automated check execution |
| Ops Manager | Step 5 (in-person), Step 7 (orientation), Step 8 (final approval) |
| Vehicle Inspector | Step 6 (vehicle inspection) |
| Compliance / Founder | Sign-off on disqualifying-finding decisions; periodic SOP review |
| Engineering | Maintain document gate + expiry sweeper; respond to audit-log queries |

## 8. Escalations

- **Document forgery suspected:** Reject application + reserve the right to
  report to police.
- **Background check ambiguous:** Refer to Compliance.
- **Driver disputes a rejection:** Driver may appeal in writing within 30
  days; Compliance reviews and issues a reasoned decision within 15
  working days.

## 9. Audit & compliance

- Quarterly: Compliance pulls a sample of 10 driver files at random and
  verifies each step was followed.
- Annually: External audit of the KYC pipeline against MVA 2020 + DPDP
  requirements.
- On regulatory request: SaaradhiGo provides driver records to the
  Telangana Transport Department within 7 days of a written request.

## 10. Revision history

| Date | Version | Change | Approved by |
|---|---|---|---|
| {{LAUNCH_DATE}} | {{POLICY_VERSION}} | Initial Phase-0 SOP | TBD |
