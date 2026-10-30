# Resilient Claims API: Error Contract & Retry Design

> A Forward Deployed Engineering (FDE) proof of concept. This repository contains both the **pre-transformation** (before) and **post-transformation** (after) states of the solution, so the impact of the work can be seen side by side.

---

## 1. Overview

| Item | Details |
|------|---------|
| **POC Name** | Resilient Claims API: Error Contract & Retry Design |
| **Scenario / Customer** | Illustrative: an insurance motor-claims assistant, backed by generative AI, that is called as an API by a separate consuming engineering team |
| **Owner** | Aditya Jagtap |
| **Status** | Completed |
| **Program** | FDE Program, Sprint 1 |

**One-line summary:** Designing an API error taxonomy (HTTP status plus a mandatory `retryable` flag) for an AI-backed claims service proves that a client retrying on status code alone fails in both directions at once, while a client that reads `retryable` wastes nothing and loses nothing.

---

## 2. Problem Statement

Once the claims pipeline (document extraction, reconciliation, generation) works, another team's application calls it as an API. Every unexpected input, timeout, retry or partial response then becomes the API designer's problem, and generic REST advice is subtly wrong for an AI-backed service.

- **Misleading rules of thumb:** "5xx means retry", "return 500 when an upstream call fails", "return 409 for duplicates" and "validate in the handler" each break something in this service.
- **AI-specific failure modes:** a model can return HTTP 200 with invalid or truncated JSON, and a streamed response commits its status code before the content that decides success or failure exists.
- **Cost and privacy exposure:** unsolicited fields (for example `applicant_ssn`) are processed, logged and stored with no lawful basis if they are not rejected early, and rejection after the expensive pipeline wastes time and money.

---

## 3. Pre-Transformation (Before)

The starting point is the generic, CRUD-style approach to API errors, plus the reference material and exercise brief used to question it.

| Aspect | Current State |
|--------|---------------|
| Process | Error handling decided case by case using generic REST heuristics; validation placed in the handler |
| Model / Tooling | Starter notebook, exercise brief and two reference case studies (status codes; retries, backoff and jitter) |
| Cost | Unexpected fields caught only after the full pipeline runs (about $0.014 per request) |
| Latency / Performance | Rejection after an ~11.7 second pipeline run instead of a ~2 ms gateway check |
| Accuracy / Quality | Status codes chosen by habit; no explicit retry instruction for clients |
| Pain points | Wrong 4xx/5xx choices, 409 returned for legitimate retries, no way to report a mid-stream failure, retry logic that guesses from the status code |

**Location in repo:** [Pre-Transformation/](Pre-Transformation/)

---

## 4. Post-Transformation (After)

A complete, executed error contract for the claims API, run against a real Azure OpenAI deployment (`gpt-4.1-mini`, live mode).

| Aspect | New State |
|--------|-----------|
| Process | Two-layer API: a cheap **gateway** (auth, idempotency, strict schema) in front of an expensive **handler**. Every error is an RFC 9457 problem-details body with a required `retryable` field |
| Model / Tooling | Python, `openai` (`AzureOpenAI`), `jsonschema`, `python-dotenv`; real calls for the invalid-JSON and streaming scenarios |
| Cost | Bad requests rejected at the gateway for $0 instead of $0.014 |
| Latency / Performance | Gateway rejection in 2 ms instead of 11,722 ms |
| Accuracy / Quality | A consistent test ("can the caller do something differently and succeed?") decides 4xx vs 5xx across all scenarios |
| Benefits | Idempotent retries, no PII silently accepted, a rule against streaming structured output, and a retry contract clients can follow |

**Five scenarios covered**

1. **Unexpected field:** rejected at the gateway (400), not after the pipeline runs.
2. **Unreadable document:** 422, non-retryable, because the caller can send a clearer scan.
3. **Invalid model JSON:** a real truncated completion is classed as 500 with `retryable: true`, and treated as an incident to page on.
4. **Client retry:** the same key and body returns the original `assessment_id`; the same key with a different body returns 409; a missing key returns 400.
5. **Mid-stream failure:** a real stream shows `200 OK` committed at t=0 and a failure at about 1,083 ms, so 422 is no longer possible. Rule: never stream the structured decision profile.

**Location in repo:** [Post-Trasformation/](Post-Trasformation/)

---

## 5. Before vs. After

| Metric | Pre-Transformation | Post-Transformation | Change |
|--------|--------------------|---------------------|--------|
| Cost to reject an unexpected field | $0.014 (after full pipeline) | $0.00 (at gateway) | -100% |
| Time to reject an unexpected field | 11,722 ms | 2 ms | ~99.98% faster |
| Retry client "retry on 5xx": wasted calls | 78 per 1,000 requests | 0 (client reads `retryable`) | -78 calls |
| Retry client "retry on 5xx": lost cases | 35 per 1,000 requests | 0 (client reads `retryable`) | -35 cases |

**Retry simulation (1,000 requests, same seed for every client)**

| Client | Calls | Wasted | Lost cases | Wasted cost |
|---|---|---|---|---|
| Naive (retry anything non-2xx) | 1,697 | 474 | 0 | $6.64 |
| Status code only (retry 5xx) | 1,246 | 78 | 35 | $1.09 |
| Reads `retryable` | 1,223 | 0 | 0 | $0.00 |

The status-only client fails in both directions because two error types disagree with their status code: `rate-limited` (429, retryable, never retried) and `unsupported-format` (500, not retryable, retried pointlessly).

Source of these numbers: [Post-Trasformation/api_taxonomy.json](Post-Trasformation/api_taxonomy.json).

---

## 6. Approach

1. **Discover:** understand the claims API, its consumers and where generic REST advice fails.
2. **Baseline:** capture the cost and latency of validating late, and the behaviour of naive retry logic.
3. **Evaluate:** test each status-code and retry decision against the "can the caller succeed differently?" criterion.
4. **Build:** implement the problem-details helper, gateway, handler, idempotency store and streaming demo.
5. **Measure:** run the live scenarios and a 1,000-request retry simulation.
6. **Hand off:** export the taxonomy and document the governance decisions.

### Decision Criteria

| Criterion | Weight | Why |
|---|---|---|
| Can the caller fix it? | High | Separates caller-fixable (4xx) from service-side (5xx) failures |
| Will a retry help? | High | Drives the `retryable` flag clients act on |
| Cost of late rejection | Medium | Cheap checks belong at the gateway, expensive work comes after |
| Data protection | Medium | Unsolicited fields must never enter logs or storage |

---

## 7. Repository Structure

```
Assignment10/
├── README.md
├── Pre-Transformation/        # "Before" state: starter notebook, exercise brief, reference case studies
└── Post-Trasformation/        # "After" state
    ├── *.ipynb                # Executed notebook with live outputs
    ├── api_taxonomy.json      # Exported error taxonomy, streaming rule, retry results
    ├── project_analysis.md    # Detailed analysis of the project
    ├── requirements.txt
    ├── .env                   # Local credentials (not committed)
    └── *.docx / *.md          # Reference case studies and exercise brief
```

---

## 8. Getting Started

### Prerequisites

- Python 3.12 (or any recent Python 3)
- An Azure OpenAI or Azure AI Foundry resource with a deployed chat model, and its endpoint, key and deployment name

### Setup

```powershell
cd Post-Trasformation
py -3.12 -m venv .venv
.venv\Scripts\pip install -r requirements.txt
```

Create a `.env` file in `Post-Trasformation` with these keys:

```
AOAI_ENDPOINT=<your endpoint>
AOAI_KEY=<your key>
AOAI_DEPLOYMENT=<your deployment name>
```

### Run

```powershell
.venv\Scripts\jupyter lab
```

Open the notebook in `Post-Trasformation` and run all cells. If the endpoint is left blank, the notebook falls back to simulated mode for the two scenarios that call the model; the rest run locally either way. A full live run makes roughly 12 model calls.

---

## 9. Results & Learnings

- **Key result:** retrying on status code alone wastes 78 calls and loses 35 cases per 1,000 requests; reading a dedicated `retryable` field wastes none and loses none.
- **What worked:** rejecting bad input at the gateway (2 ms, $0 against 11.7 s, $0.014); a keyword-only `retryable` argument with no default, so no error can ship without an explicit retry decision; replaying the original response for a repeated idempotency key.
- **What didn't:** a status code alone cannot carry retry intent, and a committed stream cannot be corrected with a later error status.
- **Recommendations:** emit `retryable` and `retry_after_seconds` on every error; ask the consuming team whether their client actually branches on `retryable`; never stream structured output; page on a rising rate of invalid-model-output 500s.

> Caveats: the error mix and cost figures are hand-specified for teaching, not production telemetry. The idempotency store is in-memory with no expiry policy. The declared `max_document_refs` limit is not enforced by the gateway. The mid-stream failure is triggered by the client stopping at a fixed chunk, not by a real content-filter rejection. Package versions in `requirements.txt` are unpinned.

---

## 10. Next Steps

- [ ] Enforce the `max_document_refs` limit in the gateway
- [ ] Add an expiry and eviction policy to the idempotency store
- [ ] Confirm with the consuming team whether their client reads `retryable`
- [ ] Replace the assumed error mix with measured production rates
- [ ] Pin dependency versions in `requirements.txt`

---

## 11. Contact

Aditya Jagtap · adiraj2503@gmail.com
