# Churn Brain: AI churn-risk triage with an eval gate

An AI workflow that scores SaaS customers for churn risk, routes them to the right follow-up, logs every decision, catches its own failures, and is protected by an automated eval gate in CI.

Built as the capstone for the AAOL course (Track 2: SaaS Churn Brain), using n8n, Google Gemini, Supabase, Promptfoo, GitHub Actions and Slack. Built with AI assistance; see "Honest notes" below.

## How it works

```
Intake Form (n8n Form Trigger)
        |
        v  POST (6 fields)
Churn_Qualifier webhook
        |
        v
Gemini scores churn risk 1-10  -->  Parse + validate JSON
        |                                   |
        |                                   +--> log decision to router_logs
        |                                   +--> log decision to system_logs
        v
Route by action
   risk 7-10  -> retention_call     -> POST to webhook + log outcome
   risk 4-6   -> automated_winback  -> POST to webhook + log outcome
   risk 1-3   -> monitor            -> POST to webhook + log outcome

Any failure -> Error_Handler workflow
   -> logs the error (router_logs + system_logs)
   -> sends an error alert (webhook)
   -> asks Gemini to diagnose the failure and sends the diagnosis
```

**Input fields:** `customer_name`, `email`, `plan`, `months_active`, `support_tickets_last_30d`, `last_login_days_ago`.

## Layers

| Layer | What it is |
|---|---|
| Intake | `Churn_Intake_Form`: public n8n form that posts to Churn |
| Decision | `Churn_Qualifier`: Gemini scoring, validation, 3-way routing |
| Audit trail | Supabase `system_logs` table, written in parallel with `router_logs` |
| Self-healing | `Error_Handler`: logs, alerts, and a Gemini diagnosis of the failure |
| Eval gate | 10-case Promptfoo suite run by GitHub Actions on every push |
| Alerting | Slack message to `#alerts` when the eval gate fails |

## Eval pack and CI gate

- `promptfooconfig.yaml`: 10 test cases for the churn prompt (obvious high risk, obvious safe, judgment calls, missing fields, a brand-new customer, contradictory signals, latency). Tests 7-9 are intentionally loose.
- `.github/workflows/eval.yml`: runs the suite on every push and PR to `main` (Node 22, `promptfoo@0.124.1` pinned). If the suite fails, a Slack message with the run link is sent.
- Secrets live in GitHub Secrets (`GOOGLE_API_KEY`, `SLACK_WEBHOOK_URL`), never in the repo.

**Gate verification (done on purpose, not assumed):**
1. Changed one assertion to an impossible value (`score >= 15`) and pushed. The run failed with 9 passed / 1 failed and exit code 100, and the Slack alert arrived in `#alerts`.
2. Restored the assertion and pushed. The run passed (10/10) and Slack stayed silent.

## The four core queries

Saved in the Supabase SQL editor against `system_logs`: full timeline (24h), errors only, borderline-risk review queue, platform health summary.

**Change from the course guide:** the guide's review queue filters `confidence < 0.6`. In this project `confidence` is just the risk score divided by 10, not a real model confidence, so that filter would surface *low-risk* customers. The review queue here targets borderline risk (`confidence` between 0.4 and 0.6) instead.

## Tested end to end

- Form submission for a high-risk customer produced a `retention_call` decision (0.9), and a low-risk customer produced `monitor` (0.1). Both were logged to `system_logs` with `outcome = success`.
- A deliberate failure (invalid Gemini model name) was submitted through the form. Error_Handler fired and wrote an error row to both `router_logs` and `system_logs`. The model was restored afterwards.
- Earlier tests (Oct 9): the Gemini diagnosis step returned a plain-language cause for a broken table name and for an invalid model name.

## Honest notes and limits

- **Built with AI assistance.** The Churn workflow and Error_Handler were generated with an AI assistant and then tested and debugged by me. The Solar lead-qualifier exercise follows the course material.
- **The "diagnosis" does not repair anything.** It explains the likely cause and rates urgency. It marked its own suggestions as not auto-fixable. It also sometimes suggests outdated model names, so its fix text should be read, not trusted.
- **`confidence` is not real confidence.** It is risk score / 10.
- **Form does not show the result.** The webhook responds before the AI finishes, so the form confirms submission only. Results appear in Supabase and the routed webhook.
- **Routing targets are mock webhooks** (webhook.site), not a real CRM or email system.
- **Not production.** Single Supabase table without row-level security, a free n8n trial, no real customer data, no throughput or time-saved metrics.
- **Solar workflow:** the `deep_research` route was tested with simulated input, because the model never chose it on real input.
- **Unverified in the last test run:** the webhook.site error and diagnosis payloads were not re-checked after the forced failure.

## Repo contents

- `promptfooconfig.yaml`: eval suite
- `.github/workflows/eval.yml`: CI gate
- n8n workflow exports (`Churn_Qualifier`, `Error_Handler`, `Churn_Intake_Form`, Solar)
