# Legal documents — Phase-0 working drafts

This directory contains the customer- and driver-facing legal documents that
SaaradhiGo / VahanGo needs in place before opening to paying users.

> **CRITICAL — these are working drafts, not law.**
>
> Every document here MUST be reviewed by an Indian-qualified lawyer who
> specialises in technology and transport before it is published, shown to a
> user, included in an app store listing, or signed by a driver. Sections that
> particularly need legal review are flagged inline.
>
> The drafts are written to give a competent reviewer a substantive starting
> point — they cover the structures, the India-specific clauses (MVA 2020,
> DPDP 2023, IT Act, RBI PPI posture, GST, TDS), and the operational realities
> of a Hyderabad ride-hailing platform. The reviewer's job is to verify
> jurisdiction, calibrate liability caps, refine indemnity, and replace
> placeholders.

## Contents

| File | What it is | Who signs / sees it |
|---|---|---|
| [terms-of-service.md](terms-of-service.md) | Rider Terms of Service | Every rider on signup (click-wrap) |
| [privacy-policy.md](privacy-policy.md) | Privacy Policy (DPDP Act 2023 compliant) | Every user; required URL for Play Store + App Store |
| [driver-agreement.md](driver-agreement.md) | Driver Partner Agreement | Every driver during onboarding (e-sign) |
| [mva-2020-driver-verification-sop.md](mva-2020-driver-verification-sop.md) | Driver onboarding standard operating procedure | Ops team internal |
| [mva-2020-driver-verification-checklist.md](mva-2020-driver-verification-checklist.md) | Per-driver KYC checklist | Ops team fills one per driver |
| [gst-tds-registration-guide.md](gst-tds-registration-guide.md) | What you have to file with the government | Finance / founder action items |
| [credits-policy.md](credits-policy.md) | VahanGo Credits Policy — closed-loop credit balance, non-cashable, non-transferable. Referenced from the Terms of Service (§5.5–5.6). See also [ADR-0003](../adr/0003-closed-loop-wallet.md) for the RBI PPI rationale. | Every rider; in-app link from Wallet screen |

## Required placeholders

Each document contains `{{PLACEHOLDER}}` tokens for company-specific facts.
Replace before publishing.

| Placeholder | Example value | Notes |
|---|---|---|
| `{{COMPANY_LEGAL_NAME}}` | "SaaradhiGo Mobility Private Limited" | Legal entity name on MCA |
| `{{COMPANY_CIN}}` | "U60000TS2026PTC123456" | Corporate Identification Number |
| `{{REGISTERED_ADDRESS}}` | Full address as on incorporation | Registered office |
| `{{PRINCIPAL_OFFICE_ADDRESS}}` | Operating office | If different from registered |
| `{{GSTIN}}` | "36ABCDE1234F1Z5" | After GST registration |
| `{{SUPPORT_EMAIL}}` | "support@saaradhigo.in" | |
| `{{SUPPORT_PHONE}}` | "+91-40-XXXXXXXX" | |
| `{{GRIEVANCE_OFFICER_NAME}}` | Founder or compliance hire | IT Act requirement |
| `{{GRIEVANCE_OFFICER_EMAIL}}` | "grievance@saaradhigo.in" | |
| `{{DPO_NAME}}` | Data Protection Officer | DPDP requirement |
| `{{DPO_EMAIL}}` | "dpo@saaradhigo.in" | |
| `{{JURISDICTION_CITY}}` | "Hyderabad" | Disputes jurisdiction |
| `{{LAUNCH_DATE}}` | YYYY-MM-DD | Effective date |
| `{{POLICY_VERSION}}` | "1.0" | Increment on every change |
