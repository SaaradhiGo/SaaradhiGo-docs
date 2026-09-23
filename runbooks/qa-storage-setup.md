# QA document storage — what was set up

- **Status:** live in QA. Production untouched.
- **Date:** 2026-09-23

## What was provisioned

A **Railway bucket**, not an AWS S3 bucket. This avoids your AWS account entirely
and was chosen against the requirements as follows:

| Requirement | How it is met |
|---|---|
| separate QA bucket | `saaradhigo-qa-docs` (`saaradhigo-qa-docs-0j5vhc`), provisioned in the **QA environment only**, region `sjc` |
| no production documents | brand-new empty bucket; production has no storage configured at all |
| no production credentials | **no AWS credentials exist anywhere in QA.** The bucket has its own key pair, issued by Railway |
| least privilege | the key pair is scoped to this one bucket by construction — it cannot address any other |
| KYC upload supported | verified end to end: presign → PUT → key accepted by the API |
| receipt PDF generation supported | same storage (`private_document_storage`); not yet exercised because no QA trip has completed |
| no public-read KYC | `PrivateDocumentStorage` keeps `querystring_auth=True` and `default_acl=None`; access is via time-limited signed URLs (`AWS_QUERYSTRING_EXPIRE`, 900 s) |
| lifecycle/cleanup | below |

## The code change it required

`base/s3.py::build_s3_client` hardcoded AWS. It now honours an optional
`AWS_S3_ENDPOINT_URL`; django-storages already read the same setting itself, so
only the presign client needed telling. Unset — production's state — still resolves
to AWS S3, asserted by a test, along with the empty-string case.

Merged as `feat/s3-endpoint-support` (`0bb2142`).

## QA variables (backend and celery)

```
AWS_S3_ENDPOINT_URL = https://t3.storageapi.dev
AWS_S3_BUCKET_NAME  = saaradhigo-qa-docs-0j5vhc
AWS_S3_REGION       = auto
AWS_ACCESS_KEY_ID   = <bucket key, from Railway>
AWS_SECRET_ACCESS_KEY = <bucket secret, from Railway>
```

Retrievable any time via the Railway API (`bucketS3Credentials`) or the dashboard.
`region = auto` is what Railway reports; for a custom endpoint the region is only a
signing scope, which is why the client must not validate it against AWS's list.

Production has **zero** `AWS_*` variables — verified by listing them.

## Verified before use

Against the live bucket, before any code was written:

- `put_object` — OK in both virtual-host and path addressing
- **presigned PUT** — HTTP 200 in both, and `head_object` confirmed the bytes

Then through the running QA API:

```
license_doc: presign HTTP 200, PUT HTTP 200, 15738b, key=license_docs/97f7277b…png
rc_doc:      presign HTTP 200, PUT HTTP 200, 15486b, key=rc_docs/91931d92…png
```

The API verifies the object exists (`base.s3.verify_uploaded_key`) before accepting
a key, so those 200s prove the round trip, not just the signature.

## What is in the bucket

Only QA test artifacts:

| Prefix | Contents |
|---|---|
| `license_docs/` | one placeholder PNG |
| `rc_docs/` | one placeholder PNG |
| `qa-smoke/` | three tiny files from the compatibility check — deletable |

The placeholder documents are generated images reading **"QA TEST ARTIFACT — NOT A
REAL DOCUMENT"** in red, repeated `QA-TEST` watermarks, and the text "Contains no
personal data. Not valid for any legal or regulatory purpose." They cannot be
mistaken for real KYC.

## Lifecycle and cleanup

Railway buckets have no lifecycle-rule UI, so cleanup is deliberate rather than
automatic:

- **Routine:** the bucket only accumulates what QA rehearsals upload — a few files
  per driver plus receipt PDFs per completed trip. Expected to stay in the low
  megabytes.
- **Reset:** delete every object, or delete and re-provision the bucket and re-set
  the five variables. Nothing else depends on it.
- **Cost:** billed on stored bytes and egress. Negligible at this size, but it is
  not free, and the bucket should be deleted if QA is retired.
- **Recommended hygiene:** clear `qa-smoke/` now, and clear the whole bucket
  whenever the QA database is reset, so orphaned documents do not outlive the rows
  that referenced them.

## If you would rather use AWS S3 instead

The code supports both. To switch QA (or to configure production) to real AWS:

1. Create a bucket, e.g. `saaradhigo-qa-documents`, **block all public access**.
2. Create an IAM user with an inline policy limited to that bucket:
   `s3:PutObject`, `s3:GetObject`, `s3:DeleteObject` on `arn:aws:s3:::<bucket>/*`
   and `s3:ListBucket` on `arn:aws:s3:::<bucket>`. Nothing else.
3. Set `AWS_S3_BUCKET_NAME`, `AWS_S3_REGION`, `AWS_ACCESS_KEY_ID`,
   `AWS_SECRET_ACCESS_KEY` — and **unset `AWS_S3_ENDPOINT_URL`**, which is what
   returns the client to AWS.
4. Add an S3 lifecycle rule if document retention is wanted; note that a retention
   policy for `TripLocationPoint` is still an open decision.

Production needs this, or an equivalent, before receipts or KYC can work there at
all — it currently has no storage configured.
