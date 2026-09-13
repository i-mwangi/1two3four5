# Recoverly

Recoverly is a Python prototype for business invoice recovery. Specialized Band agents analyze cases, draft reminders, review tone, request operator approval in Slack, and track payments. The web app exposes webhook routes and a settlement page.

The local collection/payment workflow is tested with simulated external services. It is **not yet a verified live, unattended product**. See [TEST_REPORT.md](TEST_REPORT.md) for the exact working paths and remaining gaps.

## Setup

Use Python **3.11 or newer**. This workspace was verified on Windows with Python 3.14.6 and Band SDK 1.6.0. The system Python 3.10 is too old for the current SDK and has an unrelated broken `pyreadline` installation.

```powershell
py -3.14 -m venv .venv
.venv/Scripts/python.exe -m pip install -r requirements.txt
Copy-Item .env.example .env
```

If this workspace already has `.venv`, use it. Do not overwrite an existing `.env`. Replace example credentials with real service configuration before connecting services. `.env` loads before settings are constructed; explicit process environment variables take precedence.

```powershell
.venv/Scripts/python.exe -m tools.config_check
.venv/Scripts/python.exe -m src.webapp
```

The server binds to `127.0.0.1:8400`; `/health` returns service health. `/pay/<case_id>` requires an existing case in `data/cases`. Amounts and invoice numbers come from saved case state, not URL parameters. This Flask development server is for local use.

## Verify without contacting services

The test suite is maintained locally and excluded from this repository. The commands below require that local suite; TEST_REPORT.md records the previous local verification.

```powershell
.venv/Scripts/python.exe -m pytest -q --cov=src --cov-report=term
.venv/Scripts/ruff.exe check src tests --select E9,F63,F7,F82
.venv/Scripts/python.exe -m pip check
```

Tests use temporary case/audit files and block external socket connections. The SDK is real; AI, email and payment transports are replaced with test doubles. `tests/test_end_to_end.py` contains the main collection-to-settlement integration test.

## Running connected components

The web server does not start agents or schedulers. Each agent needs its own configured `BAND_<ROLE>_AGENT_ID` and `BAND_<ROLE>_API_KEY` and runs as a separate process, for example:

```powershell
.venv/Scripts/python.exe -m src.agents.preflight_agent
.venv/Scripts/python.exe -m src.agents.investigator_agent
.venv/Scripts/python.exe -m src.agents.diplomat_agent
.venv/Scripts/python.exe -m src.agents.tone_coach_agent
.venv/Scripts/python.exe -m src.agents.concierge_agent
.venv/Scripts/python.exe -m src.agents.payment_agent
```

Other roles are `voice_agent`, `escalator_agent`, and `aaa_specialist_agent`. Configure the Band room and participant handles, Qwen credentials, Slack signing secret/token/channel, and the selected email backend. Set `RECOVERLY_PAYLINK_BASE` to the reachable web app URL ending in `/pay`.

The `.env.example` enables payment and voice dry runs with their matching allow flags. **DEMO_MODE alone does not suppress outbound email.** To exercise actual provider sandbox flows, deliberately disable the relevant dry-run flag and use provider test credentials and controlled recipients.

| Integration | Endpoint / process | Configuration |
|---|---|---|
| Slack events | `/slack/events` | Slack signing secret and bot token |
| Slack approvals | `/slack/interactivity` | Slack interactivity URL and signing secret |
| Resend events/replies | `/webhooks/resend` | Resend webhook signing secret in `whsec_...` format; receiving API key for fetching bodies |
| Paystack settlement | `/webhooks/paystack` | `PAYSTACK_SECRET_KEY`; USD-denominated case ledger |
| Card initialization | `POST /pay/<case_id>/start`, `method=card` | Existing case with buyer/customer email |
| Transfer instructions | Same route, `method=wire` or `method=usdc` | Wire beneficiary fields or Hedera receiving account/token |
| Hedera observation | `python -m src.payments.wallet_poller` | Receiving account and network; run separately |
| Voice calls | `/voice-webhook/twiml`, `/voice-webhook/fish-audio`, `/voice-webhook/call-status` | Twilio and Fish Audio credentials |

Manual bank payments use the Slack payment-received command with `case=RC-... amount=3000 reference=BANK-TRANSACTION-ID`. Reuse the same reference when retrying; partial payments keep the remaining balance open. The full-settlement approval button means the operator has verified the entire remaining amount.

The optional outgoing Hedera transfer helper additionally requires `hedera-sdk-python`; it is not needed to display USDC instructions or observe incoming payments. Outgoing transfers were not exercised. The helper now refuses to debit an operator account belonging to a different buyer.

Case data is currently local JSON/JSONL. The former PostgreSQL, Redis, Chroma, Stripe and other unused dependencies were removed from the required installation; listing those packages did not implement those integrations. Deployment, access control, durable jobs, and complete automated intake remain work described in the report.
