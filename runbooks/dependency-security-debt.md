# Dependency security debt — backend

- **Status:** Open
- **Recorded:** 2026-09-22
- **Source:** `pip-audit -r requirements.txt` against `SaaradhiGo-backend@669ce7a`
- **Scope:** Python dependencies of `SaaradhiGo-backend` only. No dependency
  has been modified; this file exists so the debt is tracked rather than
  rediscovered.

## Why this is not already failing CI

The `pip-audit` step in `.github/workflows/deploy.yml` is
`continue-on-error: true`, so the `lint` job passes green while advisories
accumulate. The report still prints on every PR.

The comment on that step gives the reason as *"cashfree-pg pins pydantic <2,
which drags in transitive advisories we cannot resolve without dropping the
SDK."* **That rationale is now inaccurate and is understating the debt.**
`pydantic 1.10.26` is itself clean — it carries no advisories. The real
constraints from `cashfree-pg==3.2.12` are:

```
urllib3    <2.1.0,>=1.25.3      blocks urllib3 2.7.0 (fix needs >=2.1)
sentry-sdk <1.33.0,>=1.32.0     blocks sentry-sdk 1.45.1
pydantic   <2,>=1.10.5          clean — not a security constraint
```

So of the 13 flagged packages, **only 2 are genuinely blocked by the Cashfree
SDK.** The other 11 are freely upgradable today, and that set includes an
end-of-life web framework and the JWT library that backs authentication.
The blanket `continue-on-error` is hiding them.

## Current findings (13 packages)

| Package | Installed | Advisories | Lowest fix | Direct? | Blocked by cashfree-pg? |
|---|---|---|---|---|---|
| **django** | 5.0.3 | **26** | 4.2.14 / 5.2 LTS | direct | no |
| **pyjwt** | 2.10.1 | 7 | 2.12.0 | direct | no |
| cryptography | 44.0.0 | 7 | 44.0.1 | direct | no |
| urllib3 | 2.0.7 | 6 | 2.7.0 | direct | **yes** |
| sqlparse | 0.5.4 | 5 | 0.6.0 | direct | no |
| daphne | 4.1.2 | 2 | 4.2.2 | direct | no |
| djangorestframework | 3.16.1 | 2 | 3.17.2 | direct | no |
| pyopenssl | 25.1.0 | 2 | 26.0.0 | transitive (twisted TLS) | no |
| twisted | 25.5.0 | 1 | 26.4.0 | transitive (daphne) | no |
| sentry-sdk | 1.32.0 | 1 | 1.45.1 | direct | **yes** |
| requests | 2.32.5 | 1 | 2.33.0 | direct | no |
| click | 8.3.1 | 1 | 8.3.3 | direct | no |
| python-dotenv | 1.2.1 | 1 | 1.2.2 | direct | no |

### The headline item

**`django==5.0.3` carries 26 advisories and the 5.0.x series is
end-of-life.** Some of its fixes landed as far back as 4.2.14, meaning this
pin has been unpatched for a long time. Nothing about the Cashfree SDK
blocks a Django upgrade. Target is **5.2 LTS**.

Note `STATICFILES_STORAGE` is already deprecated in favour of `STORAGES`
(the deprecation warning shows on every test run), so a Django upgrade
should absorb that change at the same time.

## Suggested remediation, in three PRs

Deliberately not one PR — the risk profiles are completely different.

**Tier 1 — low-risk patch bumps, no API changes expected.** Should be a
same-day PR.

```
cryptography        44.0.0  -> 44.0.1     patch
sqlparse            0.5.4   -> 0.6.0
djangorestframework 3.16.1  -> 3.17.2
requests            2.32.5  -> 2.33.0
click               8.3.1   -> 8.3.3
python-dotenv       1.2.1   -> 1.2.2
```

**Tier 2 — needs a real test pass, one PR each.**

- `django 5.0.3 -> 5.2 LTS` — biggest item. Absorb the `STORAGES`
  migration. Exercise admin dashboard, DRF serializers, Channels.
- `PyJWT 2.10.1 -> 2.12.0+` — authentication-critical. Verify
  `rest_framework_simplejwt` compatibility and the `JWT_SIGNING_KEY` path
  before merging.
- `daphne 4.1.2 -> 4.2.2` together with `twisted -> 26.4.0` and
  `pyopenssl -> 26.0.0`. Daphne is the ASGI server, so exercise the
  WebSocket trip and chat paths.

**Tier 3 — requires moving off or forward on the Cashfree SDK.**

`urllib3` and `sentry-sdk` cannot move while `cashfree-pg==3.2.12` holds its
pins. Options, in order of preference:

1. Check whether a newer `cashfree-pg` relaxes the `urllib3` and
   `sentry-sdk` ranges. This is the cheap path if it exists.
2. Call the Cashfree REST API directly through the existing
   `PaymentGateway` interface
   (`servers/payments/payment_gateways/base_gateway.py`) and drop the SDK.
   The interface was built for exactly this kind of swap, so the change
   would be contained to the `payments` app.
3. Accept the two pins, and narrow the CI exemption to only them (below).

## Recommended CI change

Replace the blanket `continue-on-error: true` with an explicit ignore list
for just the advisories that are genuinely blocked, so the gate goes red on
anything else:

```yaml
- name: pip-audit (known CVEs in dependencies)
  run: pip-audit -r requirements.txt --ignore-vuln <urllib3-ids> --ignore-vuln <sentry-sdk-ids>
```

That keeps the two known-blocked packages from failing the build while
making any new advisory — or any of the 11 currently-upgradable ones —
break CI as it should.

## Related, not dependency debt

The `deploy` job has never succeeded: `Setup SSH` fails because the
`EC2_HOST_KEY` GitHub secret is unset and the `ssh-keyscan` fallback cannot
reach `EC2_HOST`. This is already on the Phase-0 launch-readiness list under
Infrastructure. The QA environment currently deploys via Railway, not this
workflow, so the two deployment paths have drifted apart.
