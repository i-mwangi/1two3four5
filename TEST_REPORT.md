# Recoverly verification — 2026-09-12

## Result

**287 tests passed** on Windows / Python 3.14.6, using the actual Band SDK 1.6.0. Overall statement coverage is **55.83%** (3,508 of 6,283 statements). This is coverage of executed source statements, not a percentage of product completeness.

The original suite had 268 passing tests but replaced the agent SDK with empty classes. The expanded suite adds 19 tests/cases and exercises actual SDK imports. All 80 non-`__init__` source modules import, Flask registers 20 routes, and its in-process `/health` request returns HTTP 200. Fatal-error static checks and `pip check` pass. All packages in the revised requirements file are installed.

No buyer emails, Slack messages, phone calls, charges or wallet transfers were performed. External transports in tests are simulated, and tests block external network connections. Live service credentials are absent. No deployment was performed.

## Verified working locally

| Area | Evidence | Limit |
|---|---|---|
| Startup | Real SDK imports; all source modules import; Flask app/health smoke check | No live Band connection |
| Collection workflow | Structured invoice/case payload → preflight → investigator → diplomat → tone coach → operator approval → email dispatch | Model outputs and email delivery simulated; approval dispatched in process |
| Intake helpers | Existing PDF extraction, document-pairing and routing tests | Raw email/upload-to-live-agent handoff is not covered end to end |
| Approval discipline | No email before approval; repeating approval sends once; settled case blocks further dispatch | A crash after external delivery but before receipt persistence remains a recovery concern |
| Card checkout | Page → provider initialization → redirect; passes saved buyer email and server-side balance | Paystack transport simulated; merchant USD support must be confirmed with the actual account |
| Card settlement | Signed webhook → partial balance → retry deduplication → final payment → case closure | No real processor event received |
| Wire/USDC instructions | Configured beneficiary/account details render; unconfigured card method returns 503 | No transfer executed |
| Manual payments | Transaction-reference requirement; partial balance retained; repeated reference not reapplied | Operator is responsible for verifying bank receipt |
| Payment integrity | Unknown cases and invalid amounts rejected; duplicate references not counted; reconciliation replay preserves balances/reference IDs | Full audit reconstruction is not a complete backup of case metadata |
| Resend | Standard Svix signature headers, timestamp checks, body retrieval, invoice-subject matching | Live receiving domain/reply routing not configured; ambiguous subjects require manual resolution |
| Voice support | Existing dial-window, signal, speech and signed callback tests | Full voice-agent conversation and actual calls unverified |
| Agent lifecycle | Transport failure propagates instead of leaving the supervisor waiting forever | Live disconnect/reconnect behavior unverified |

## Changes made

1. Replaced the incorrect `thenvoi-sdk` dependency with the verified `band-sdk` version, added Flask and Windows timezone data, and reduced required packages to used runtime/test dependencies. Documented Python 3.11+.
2. Moved `.env` loading before immutable settings are constructed. Expanded runtime diagnostics so missing services and local JSON persistence are clear.
3. Added a real checkout action and transfer instructions. Removed the stale Stripe wording. Unknown case URLs return 404; caller-supplied amounts/invoice labels cannot override saved invoice data.
4. Added Paystack SHA-512 webhook authentication, payload/currency validation and reconciliation. The ledger accepts USD card settlement; it does not reinterpret other currencies as dollars.
5. Added transaction deduplication inside the case mutation lock. Partial payments retain the balance; invalid amounts and unknown cases do not settle cases. Updated payment-agent and manual-command behavior to use reconciliation.
6. Passed customer email to card initialization. Removed a misleading claim that failed card creation had already fallen back to wire instructions.
7. Persisted preflight case fields needed by subsequent agents, approvals and checkout.
8. Prevented the approval notification from instructing a second send. Added persisted, per-draft outbound receipts and serialization for repeated approvals. Closed/halted cases cannot dispatch email; stale cadence buttons cannot restart them.
9. Updated Resend verification to its actual signing format, added timestamp checks, fetched incoming message bodies when absent from webhook payloads, and matched invoice references in subjects.
10. Moved wallet processed-marking after reconciliation and prevented the outgoing settlement helper from debiting a different operator's account as if it were the buyer.
11. Added reconciliation-event replay support and a supervisor test for failed agent transports.
12. Removed SDK test stubs, isolated test queues/audits/cases, blocked external test connections, and added setup instructions and safe payment/voice dry-run examples.

## Still missing or unverified

These are substantive remaining gaps; **credentials alone will not make the whole product complete**.

| Priority | Gap | Required next work |
|---|---|---|
| High | Raw intake is not fully connected to the working structured-case workflow | Email intake currently writes an intake record; upload pairing writes pending/pairing artifacts. Implement a durable reviewed intake-to-case-to-preflight handoff, with retries and explicit missing-contract handling. Current buyer recognition relies heavily on predefined personas. |
| High | Live service configuration | Configure Band agents/room, Qwen, Slack, mail receiving/sending, Paystack and voice services; register publicly reachable webhook URLs. Run provider sandbox acceptance tests with controlled recipients. |
| High | Public access control | Checkout currently identifies cases by their IDs. Add scoped, expiring buyer links or authentication; enforce the intended Slack operator identity/permissions before internet exposure. |
| High | Durable orchestration and retries | Supervise agents, calendar ticks and wallet polling. Implement reliable queue draining/retry behavior, delivery idempotency across crashes, webhook replay recovery, and scheduled Slack-reminder cancellation. Closed-case guards stop dispatch; they do not delete every previously scheduled Slack reminder. |
| High | Complete escalation/voice behavior | The full Concierge, Escalator, AAA Specialist and Voice adapter execution paths still lack scenario tests. Their modules import, and some supporting functions are tested, but real coordination, approval, calls and arbitration handoffs are unverified. |
| Medium | Storage and recovery | Case state uses local files; PostgreSQL settings do not wire a database implementation. Define durable storage, backup/restore and concurrency strategy before multi-instance operation. Audit replay does not restore every invoice/contract/contact field. |
| Medium | Complete notification behavior | Paystack currently updates case state and audit records. Full parity with wallet supplier notifications, closed-case UI/card updates and durable retries remains to be completed. |
| Medium | Remaining prototype components | Invoice calendar execution, customer-history integration, transcript processing and x402 flows need targeted tests and product decisions. Some paths still use fixtures or partial adapters. |
| Medium | Deployment and product UI | No managed production server/process supervision, deployment configuration, or full operator dashboard is delivered. Slack remains the operator interface. |

## Reproduce

```powershell
.venv/Scripts/python.exe -m pytest -q --cov=src --cov-report=term
.venv/Scripts/ruff.exe check src tests --select E9,F63,F7,F82
.venv/Scripts/python.exe -m pip check
.venv/Scripts/python.exe -m tools.config_check
```

The configuration check is expected to report missing service credentials in this workspace. `coverage.json` holds the detailed coverage result. Windows sandbox permissions required running the fresh runtime's temporary-directory tests with elevated tool approval; no global Python packages were altered.

Provider protocol references used for the fixes: [Band SDK](https://docs-dev.band.ai/integrations/sdks/reference), [Paystack webhooks](https://paystack.com/docs/payments/webhooks/), [Resend signature verification](https://resend.com/docs/webhooks/verify-webhooks-requests), and [Resend received-email retrieval](https://resend.com/docs/api-reference/emails/retrieve-received-email).
