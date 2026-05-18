# GST + TDS Registration & Compliance Guide

For {{COMPANY_LEGAL_NAME}} (CIN: {{COMPANY_CIN}}).

> **DRAFT — REQUIRES CA REVIEW BEFORE FILING**
>
> This is a structured starting point for the founder + finance lead. A
> chartered accountant (CA) must validate the specific filings, slabs,
> exemption claims, and timeline before any submission to tax authorities.
> Get-it-wrong penalties on GST and TDS are real and accumulate fast.
>
> A CA who has handled an aggregator-classification client before is ideal
> — the GST treatment for ride-hailing aggregators (Notification 13/2017
> reverse-charge mechanism for Section 9(5) services) is specialised.

---

## TL;DR — what you must complete before launch

| # | Item | Who does it | Approx. time |
|---|---|---|---|
| 1 | GST registration (GSTIN) for {{COMPANY_LEGAL_NAME}} | Founder + CA | 7-15 days |
| 2 | TAN (Tax Deduction Account Number) for TDS u/s 194O | Founder + CA | 7-15 days |
| 3 | Professional Tax registration (Telangana) | CA | 7-10 days |
| 4 | Shops & Establishments Act registration (Telangana) | CA | 7-10 days |
| 5 | GHMC trade licence | Founder | 15-30 days |
| 6 | Open a current account with a scheduled commercial bank | Founder | 7-15 days |
| 7 | Decide GST liability model for fares | CA | One call |
| 8 | Configure TDS deduction in the platform | Engineering | Already wired (PR #25); needs TAN to be filled into `.env.prod` |

After this list, file monthly GST returns (GSTR-1, GSTR-3B), quarterly TDS
return (24Q for salaries, 26Q for non-salary — section 194O TDS is reported
in 26Q), annual GSTR-9.

---

## Part 1 — GST registration

### 1.1 Why this is required from Day 1

For most service businesses, GST registration is required when annual
aggregate turnover crosses ₹20 lakh. BUT — under Section 24 of the CGST
Act, **persons required to collect tax at source under Section 52** must
register **regardless of turnover**. Ride-hailing aggregators clearly
fall under this.

Additionally, under Section 9(5) of the CGST Act read with Notification
17/2017-CT (Rate), as further extended/clarified, **the e-commerce
operator is required to discharge GST on certain services supplied
through it** — including (with the right interpretation) passenger
transport by radio taxi / motor cab. The result is that **SaaradhiGo, as
the aggregator, bears the GST liability on the fare** rather than the
individual driver. [CA REVIEW: confirm exact section 9(5) coverage for
your business model — auto vs taxi treatment historically diverged.]

The practical consequence: register before You make a single rupee of
revenue.

### 1.2 Documents needed

| Document | Source |
|---|---|
| PAN of {{COMPANY_LEGAL_NAME}} | Issued at incorporation |
| Certificate of Incorporation + MoA + AoA | MCA |
| Authorised signatory's PAN + Aadhaar + photograph | The director the company nominates |
| Board resolution authorising the signatory | Drafted internally |
| Registered office proof — rent agreement / property tax receipt / electricity bill | From the office at {{REGISTERED_ADDRESS}} |
| NOC from owner of the registered office (if rented) | Owner |
| Bank account details: cancelled cheque or first page of passbook | Bank |
| Digital Signature Certificate (DSC) of the authorised signatory | Class 3 DSC; ~₹2000 + Aadhaar OTP |

### 1.3 Step-by-step on gst.gov.in

1. Visit https://www.gst.gov.in/ → Services → Registration → New
   Registration.
2. Part A: Select "Taxpayer", state "Telangana", district appropriate to
   {{REGISTERED_ADDRESS}}, legal name of business as on PAN, PAN, email,
   mobile. Receive OTPs on each, get a Temporary Reference Number (TRN).
3. Part B: Log in with the TRN, fill in 10 sections:
   - **Business details** — trade name (use "SaaradhiGo" or "VahanGo"),
     constitution (Private Limited Company), commencement date.
   - **Promoters / partners** — director details.
   - **Authorised signatory** — usually the founder/CFO. Upload PAN,
     Aadhaar, photograph.
   - **Principal place of business** — {{REGISTERED_ADDRESS}}. Upload
     proof + NOC.
   - **Additional place(s)** — Phase-0 likely just the one office; add
     others if You have multiple operational locations.
   - **Goods & services** — for passenger transport aggregators, key SAC
     codes: **996412 (passenger road transport — taxi)**, **996413
     (passenger road transport — radio taxi / motor cab)**, **998599
     (other support services for transport)**. [CA REVIEW: pick all
     SAC/HSN codes you'll actually invoice under.]
   - **Bank account** — upload cancelled cheque / passbook page.
   - **State-specific information** — Telangana fields.
   - **Aadhaar authentication** — recommended; speeds up approval.
   - **Verification** — sign with DSC of the authorised signatory; submit.
4. Get an Application Reference Number (ARN). Track status; the GST
   officer may raise queries (you have 15 days to respond).
5. On approval: GSTIN is issued. Update `.env.prod`:
   ```
   GSTIN=36ABCDE1234F1Z5   # actual GSTIN from approval
   ```
6. Place the GSTIN on every invoice, on the website footer, and on the
   driver app's "Fee details" screen.

### 1.4 GST liability model — the key question

**Scenario A: Aggregator bears the GST (Section 9(5) treatment).**
SaaradhiGo invoices the rider for the fare inclusive of GST, files and
pays the GST itself. The driver is NOT liable for GST on the fare share.
SaaradhiGo's invoice to the rider is `fare + GST`. SaaradhiGo's
remittance to the driver is `fare share - SaaradhiGo commission - TDS`.

**Scenario B: Driver bears the GST.**
Used historically by some aggregators where the service was deemed not
to fall under Section 9(5). Each driver bills the rider; the driver
needs their own GST registration if their turnover requires it.
SaaradhiGo charges its commission separately.

**For Phase-0, Scenario A is almost certainly correct for taxi / motor-cab
services.** [CA REVIEW: confirm explicitly. Auto-rickshaw services may
have a different treatment in some periods — verify the active
notification.]

### 1.5 GST rates (as of mid-2026 — confirm at filing time)

| Service | Rate |
|---|---|
| Passenger transport by radio-taxi / motor-cab (with AC) | 5% (no ITC) or 12% (with ITC); choose at registration |
| Passenger transport by auto-rickshaw via app-based aggregator | Currently NIL on the fare (exempt), but the SaaradhiGo commission to the driver is still taxable [CA REVIEW] |
| SaaradhiGo's commission service to drivers (if billed) | 18% |
| Platform convenience fee to rider (if separately stated) | 18% |

The 5%-no-ITC vs 12%-with-ITC election affects whether You can claim input
tax credit on Your AWS / Google Cloud / Cashfree / office bills. Discuss
with the CA.

### 1.6 Ongoing GST compliance

| Return | Frequency | Due date |
|---|---|---|
| GSTR-1 (outward supplies) | Monthly (if turnover > ₹5 cr) or Quarterly QRMP | 11th of next month |
| GSTR-3B (summary + tax payment) | Monthly | 20th of next month |
| GSTR-9 (annual reconciliation) | Annual | 31 December for the previous FY |

Configure auto-reminders in the company calendar. Missed filings attract
late fees of ₹50/day + interest at 18% p.a.

---

## Part 2 — TAN + TDS u/s 194O

### 2.1 Why

Section 194O of the Income Tax Act, 1961, requires e-commerce operators
to deduct income tax at source on the **gross sale value** of services
sold through them on behalf of e-commerce participants (i.e. drivers).

- Current rate: **1%** (resident drivers with valid PAN). If PAN is
  missing, deduction is at 5% under Section 206AA.
- Triggered: on payment to the driver or credit, whichever is earlier.
- Threshold: ₹5 lakh per driver per financial year. Below ₹5 lakh, no
  TDS. Above, deduct on the gross.

The platform code (PR #25 + #33) is ready to deduct; what's missing is
the **TAN** (Tax Deduction Account Number) the deduction is recorded
against.

### 2.2 Getting a TAN

1. Apply on https://onlineservices.nsdl.com/paam/ →
   "Online TAN Application (Form 49B)".
2. Application type: "Allotment of New TAN" for {{COMPANY_LEGAL_NAME}}.
3. Pay fees online (~₹65).
4. e-Sign the application using Aadhaar OTP or DSC.
5. Receive TAN by email within 7–15 days.
6. Update `.env.prod`:
   ```
   COMPANY_TAN=HYDS01234A
   ```
   (Engineering: read this in the TDS calculation path; surface on driver
   payout receipts.)

### 2.3 What to set up before TDS goes live

- TAN ✅
- A separate TDS ledger in the accounting system
- Monthly TDS challan payment process (Challan 281, by the 7th of the
  following month — except for March which is by the 30th of April)
- Quarterly Form 26Q filing (15 working days after the quarter ends, via
  the TIN-NSDL portal)
- Form 16A issuance to drivers (15 days after the 26Q filing)
- Verify driver PANs in bulk via the TRACES portal before deducting;
  invalid PAN means the driver is treated as no-PAN and TDS is 5%

### 2.4 What the driver sees

| Item | Value (example for a ₹100 fare) |
|---|---|
| Gross fare | ₹100.00 |
| SaaradhiGo commission (18%) | ₹18.00 |
| Net to driver before TDS | ₹82.00 |
| TDS u/s 194O (1% on gross ₹100) | ₹1.00 |
| **Final payout to driver** | **₹81.00** |
| TDS deposited to Government against driver's PAN | ₹1.00 |

The driver can later claim the ₹1.00 as TDS credit in their own income
tax return; net cost to the driver is zero in the long run.

---

## Part 3 — Other registrations Phase-0 needs

### 3.1 Professional Tax (Telangana)

Required for the company (employer registration) and for each employee
earning above the slab.

- Apply via https://tgct.gov.in/
- Documents: PAN, incorporation, bank, registered office proof.
- Result: PTIN (Professional Tax Identification Number).
- Filing: monthly Form V if employees > 20; quarterly otherwise.

### 3.2 Shops & Establishments (Telangana)

Required for any establishment with employees in Telangana.

- Apply via https://labour.telangana.gov.in/.
- Documents: PAN, incorporation, registered office proof, employee list.
- Result: S&E registration certificate. Display at the office.

### 3.3 GHMC Trade Licence

Required to operate from the Hyderabad office.

- Apply via https://www.ghmc.gov.in/ → Citizen Services → Trade Licence.
- Documents: rent agreement, NOC, address proof, fire NoC (sometimes).
- Fee: based on category and office area; typically ₹2,000 – ₹15,000.
- Renew annually.

### 3.4 ESI + PF (when applicable)

- ESI: required if You have ≥10 employees earning ≤ ₹21,000/month gross.
- PF: required if You have ≥20 employees.
- For Phase-0 with a small core team You may be below thresholds, but
  apply for voluntary registration if attracting senior talent — both
  ESI and PF are seen as standard.

### 3.5 RBI / FEMA implications

If accepting investment from overseas, the company falls under
FEMA. Engage a CA before raising any foreign capital.

---

## Part 4 — Calendar (your first 12 months)

| Month | What's due |
|---|---|
| Month 0 (pre-launch) | GSTIN approved. TAN allotted. Bank current account active. PTIN. S&E. GHMC. |
| Month 1 | GSTR-3B for previous month by 20th. TDS challan by 7th. |
| Month 2 onward | Same monthly cadence. |
| Every quarter end + 15 working days | Form 26Q for TDS. |
| Every quarter end + 15 days + 15 days | Form 16A handed to drivers. |
| Year-end (Jan-Mar) | Reconcile books; prepare 26AS-vs-Form-16A; prepare GSTR-9. |
| 30 September | GSTR-9 for prior FY due. |
| 31 October | Income tax return (ITR-6) for prior FY due if not subject to audit. |
| 31 December | GSTR-9C (reconciliation, if turnover > ₹5 crore). |

A simple Google Sheet with these dates per quarter, owned by the founder
or finance lead, prevents almost all late-filing penalties.

---

## Part 5 — What this guide does NOT cover

- Equity capital and FDI / FEMA filings
- Trademark and IP registrations (separate exercise)
- Customs / import duty (not relevant for a software/services business)
- State commercial taxes outside Telangana (relevant only when You expand)
- TCS u/s 52 if/when SaaradhiGo's business model changes
- Audit requirements under Companies Act / Income Tax Act
- Transfer pricing if there are related-party transactions

For each of the above, engage the CA when they become relevant.

---

## Contact for clarifications

- Finance / Founder responsible: TBD
- Engaged CA firm: TBD — Selection pending
- Engaged Company Secretary (if any): TBD

*Last updated: {{LAUNCH_DATE}} — Version {{POLICY_VERSION}}*
