# Day 17 — The API Surface: Generate and Govern

**Birlasoft FORGE FDE Academy · Sprint 1 Day 17 · Learner Handout**

---

## How this works

You are not going to write this code. You are going to **specify it, read what
comes back, and find what is wrong with it.**

Today that matters more than usual, because a model has been trained on a great
deal of generic REST advice — and several pieces of that advice are **actively
wrong for an AI service**. It will hand them to you with complete confidence:

> *"5xx means retry."* · *"Return 500 when an upstream call fails."*
> *"Return 409 for duplicates."* · *"Validate in the handler."*

Every one of those is defensible in a CRUD API and breaks something here. You
are going to catch them.

Each exercise gives you:

| | |
|---|---|
| **The brief** | What has to be achieved. Read this first. |
| **The prompt** | Paste into Gemini. Improve it if you can. |
| **Accept it when** | Your checklist before moving on. |
| **It will probably get wrong** | Do not read until you have reviewed the output yourself. |

> **Work in that order.** Reading the failure list first turns a review exercise
> into spot-the-difference, and you learn nothing.

**In Colab:** the Gemini panel, or `Ctrl+Shift+Space` in a code cell.

---

## Setup — before Exercise 1

Run by hand. Not part of the exercise.

```python
%pip install -q openai jsonschema
import os, re, json, time, uuid, getpass
from dataclasses import dataclass

ENDPOINT   = input("Azure OpenAI endpoint: ").strip()
KEY        = getpass.getpass("Key (hidden): ").strip()
DEPLOYMENT = input("Deployment name: ").strip()

from openai import AzureOpenAI
client = AzureOpenAI(azure_endpoint=ENDPOINT, api_key=KEY,
                     api_version="2024-10-21", max_retries=0, timeout=60.0)

COST = {"gateway_ms": 2, "gateway_usd": 0.0,
        "doc_intel_ms": 9000, "doc_intel_usd": 0.010,
        "generation_ms": 2600, "generation_usd": 0.004}
print("ready")
```

---

# Exercise 1 · The contract

**15 minutes**

### The brief

Two things every error in this API carries: a **standard problem-details
shape**, and a **`retryable` flag**.

The second is the one that will go missing unless you insist on it, because
generic REST guidance does not include it.

### The prompt

```
Write a Python helper for a Colab notebook that builds RFC 9457 problem
details JSON for an API error response.

Signature:
  problem(status, title, detail, *, retryable, retry_after=None, **extra)

Requirements:
- Return a dict with: type, title, status, detail, retryable
- `retryable` is a REQUIRED keyword-only boolean — a caller must state it
  explicitly and must not be able to omit it
- Include retry_after_seconds only when retry_after is given
- **extra merges additional keys, e.g. a list of offending field names

Then write a short comment explaining the difference between what the HTTP
status code communicates and what `retryable` communicates, and why both are
needed.

Do not use HTTPException or any web framework — this is a plain function.
```

### Accept it when

- [ ] `retryable` is keyword-only and **has no default**
- [ ] `retry_after_seconds` appears only when supplied
- [ ] The comment distinguishes status from `retryable` correctly
- [ ] No framework dependency

### It will probably get wrong

> Read only after your own review.

- **Giving `retryable` a default of `True` or `False`.** A default means someone
  will forget to think about it, and the field silently lies. Making it required
  forces a decision at every call site — which is the entire point.
- **Explaining the difference vaguely.** A common generated answer is *"the
  status is standard and retryable is custom"*. True and useless. The correct
  framing: **the status is a category, `retryable` is an instruction.** A client
  library branches on the second.
- **Adding an `instance` or `traceId` field with a stack trace in it.** Never
  put internal detail in a body that crosses a trust boundary.

---

# Exercise 2 · The gateway, and the 4xx/5xx test

**25 minutes · covers Scenarios 1 and 2**

### The brief

Two layers: `gateway()` is API Management, `handler()` is the Function.

**Scenario 1** — a client sends `applicant_ssn`, a field nobody asked for. The
question is not whether to reject it. It is **where**.

**Scenario 2** — the document is unreadable. 422 or 500? Decide before you
prompt.

### The prompt

```
Write two Python functions modelling an API with a gateway layer and a
handler layer.

`gateway(req, *, validate_schema=True)` — API Management. Returns a Response
to reject the request, or None to pass it through. It must check, in order:
  - authentication (401)
  - idempotency key present (400)
  - JSON schema: required fields, types, regex patterns, and REJECT ANY FIELD
    NOT IN THE SCHEMA (400, listing the offending field names)

`handler(req, *, gateway_validated=True)` — the Function. It must:
  - accumulate cost and latency from a COST dict as it goes
  - if req["_doc_unreadable"]: the document could not be read. Choose the
    status code using this test and explain your choice in a comment:
      "can the caller do something differently and succeed?"
      yes -> 4xx, no -> 5xx
  - otherwise return 202 with an assessment id

Use the `problem()` helper from the previous cell for every error body.
Every error must set `retryable` explicitly.

A Response dataclass should carry: status, body, ms, usd, rejected_at.

Then demonstrate: send a request containing an unexpected field
`applicant_ssn` through BOTH designs — validate_schema=True and
validate_schema=False — and print the status, latency and cost of each.
```

### Accept it when

- [ ] Unknown fields are rejected, and the response **names them**
- [ ] The unreadable document returns **422**, not 500
- [ ] The comment applies the "can the caller fix it" test correctly
- [ ] Every error sets `retryable` explicitly
- [ ] The two designs are actually compared on cost and latency

### It will probably get wrong

- **Allowing unknown fields.** JSON Schema permits additional properties by
  default, and generated validators usually inherit that. Ask what happens to
  `applicant_ssn` — the honest answer is that it is accepted, processed, and
  now sits in your logs, your storage and your subject access response with no
  lawful basis. **Rejecting unknown fields is a data protection control that
  happens to also be a cost control.**
- **Returning 500 for the unreadable document.** This is the most likely single
  error today. The reasoning it will offer — *"an upstream service failed, so
  it is a server error"* — sounds right and is wrong. The caller can send a
  clearer scan. A 500 tells them it is our fault and to retry, and it will fail
  identically forever.
- **A generic detail line.** *"Internal server error"* is a support ticket.
  *"Supply a clearer scan or a text-based PDF"* is an instruction. The detail
  field is where you either help the caller or waste their afternoon.

### Then answer

The consuming team asks you to relax the schema so they can send an extra field
their system already has. **Is there a version of yes** — and what would it
require them to tell you first?

---

# Exercise 3 · When the model returns invalid JSON

**20 minutes · Scenario 3 · runs against your real endpoint**

### The brief

Ask your deployment for a strict JSON decision profile, then **truncate it**
with a low `max_tokens`. Truncated JSON is not valid JSON, and this is one of
the commonest ways real model output fails schema validation.

Then: 422 or 500?

### The prompt

```
Write a Colab cell that demonstrates model output failing schema validation
against a real Azure OpenAI deployment.

Requirements:
- A function `ask_for_profile(max_tokens)` that calls the deployment asking
  for ONLY a JSON object matching this shape:
    {"case_reference": str, "extracted_fields": {...},
     "policy_basis": [...], "needs_human": bool}
  Use this claim text: [paste the CASE string]
- A function `validate_profile(text)` returning (ok, parsed_or_None, reason).
  It must strip markdown code fences before parsing, then json.loads, then
  validate against a jsonschema schema. Distinguish "would not parse" from
  "parsed but failed schema" in the reason.
- Run it twice: max_tokens=400 and max_tokens=60
- Print, for each: character count, whether valid, the reason, and the last
  90 characters of what came back
- When validation fails, print the problem-details body your API would return

Choose the status code for a model that returned unparseable output, and
justify it in a comment using the "can the caller do something differently
and succeed" test.
```

### Accept it when

- [ ] Markdown code fences are stripped before parsing
- [ ] "Would not parse" and "parsed but failed schema" are distinguished
- [ ] The failure returns **500**, with `retryable: True`
- [ ] The last characters of the truncated output are shown

### It will probably get wrong

- **Returning 400 or 422.** The caller sent a well-formed request. *Our* model
  produced output that failed *our* schema. A 4xx blames them for something
  they did nothing to cause.
- **Setting `retryable: False`** on the grounds that it is a 500. A re-run may
  well succeed — this is the retryable 5xx that breaks the "5xx means retry"
  heuristic in the other direction. It should also **page somebody**: a rising
  rate of invalid model output is an incident, not a client integration issue.
- **No fence stripping.** Models return ```` ```json ```` fences constantly.
  Code without stripping fails on valid output and you spend an hour blaming
  the model.
- **Catching only `JSONDecodeError`.** Output can parse fine and still be the
  wrong shape — a missing `needs_human`, a `policy_basis` that came back as a
  string. Schema validation is a second, separate check.

### Then notice

The API call **succeeded**. HTTP 200 from the provider, no exception, every
layer reporting healthy.

**Only validating the content found the problem.** What else in your pipeline
is reporting success without anyone checking what it produced?

---

# Exercise 4 · The client times out and retries

**20 minutes · Scenario 4**

### The brief

The client POSTs, the connection drops before the response arrives, and it
retries — **exactly what you want it to do**. Stop that creating two
assessments.

The distinction that gets implemented wrongly is between a *replay* and a
*conflict*.

### The prompt

```
Add idempotency handling to the gateway function.

Requirements:
- The client supplies an Idempotency-Key. Missing key -> 400.
- Store, against each key, both the response that was returned AND a
  fingerprint of the request body (a stable hash or sorted JSON dump).
- Same key + same body     -> return the ORIGINAL response, unchanged,
                              with the original assessment id
- Same key + DIFFERENT body -> 409
- Explain the difference between those two cases in a comment.

Then demonstrate all four cases in order and print the status of each:
  1. first attempt succeeds (imagine the response is lost in transit)
  2. client retries with the same key and the same body
  3. same key, different body
  4. no key at all

Print the assessment_id from attempts 1 and 2 side by side.
```

### Accept it when

- [ ] A replay returns **202** with the **same assessment id**, not a 409
- [ ] Same key with different body returns **409**
- [ ] The body fingerprint is stable — sorted keys, not `str(dict)`
- [ ] Missing key is rejected at the gateway

### It will probably get wrong

- **Returning 409 for a straightforward replay.** This is the most likely error
  in this exercise, because "409 Conflict for duplicates" is standard CRUD
  advice. Here it is backwards. **The caller did nothing wrong** — their
  connection dropped and they retried. Punishing correct client behaviour with
  an error trains them to stop retrying, and then transient failures become
  lost cases.
- **Not storing the body fingerprint at all**, so any reuse of a key returns the
  original response — including a genuine mistake where they reused a key for a
  different claim. That silently returns the wrong assessment.
- **Fingerprinting with `str(dict)` or `hash()`.** Dict ordering and Python's
  per-process hash seed make that unstable across restarts. Use
  `json.dumps(body, sort_keys=True)` or a real digest.
- **Storing keys forever with no expiry.** Ask how long the store should retain
  a key, and what happens when it is full. There is no right answer, but there
  is a wrong one: not having thought about it.

---

# Exercise 5 · The content filter fails mid-stream

**25 minutes · Scenario 5 · the one that reshapes the architecture**

### The brief

Stream a **real** completion from your deployment. Interrupt it partway
through. Then answer: **can you still return 422?**

### The prompt

```
Write a Colab cell that streams a chat completion from Azure OpenAI and
simulates a content-filter failure partway through.

Requirements:
- Use stream=True against my deployment, asking for a ~120 word claim summary
- Record a timeline of events with millisecond timestamps, starting with the
  moment the HTTP status is committed
- Accumulate the text delivered so far
- At chunk index FAIL_AT_CHUNK (set it to 12), stop and record an error event,
  then close the stream cleanly
- Skip chunks with an empty `choices` list, and chunks whose delta has no
  content — only count chunks that actually delivered text
- Print the timeline showing: when the status was committed, when the failure
  happened, and HOW MANY CHARACTERS were already delivered
- Print the last 200 characters the reader can currently see

Then print an explanation of what status code the API can now return, and
what the available options are when a stream fails partway through.
```

### Accept it when

- [ ] The timeline shows the status committed at **t≈0**
- [ ] Only content-bearing chunks are counted
- [ ] Empty `choices` lists do not crash it
- [ ] It reports characters already delivered
- [ ] The explanation says a 422 is **no longer possible**

### It will probably get wrong

- **Claiming you can send a 422.** Some generated code will construct an error
  response and return it as though the status were still open. It is not. The
  status was decided the moment the first byte went out.
- **`chunk.choices[0]` without a guard.** Azure sends chunks with an empty
  `choices` list — a prompt-filter chunk on filtered deployments. This throws
  an `IndexError` intermittently, which means it will work in your demo.
- **Counting the role-only opening chunk as content.** It carries no text.
- **Not closing the stream.** An abandoned stream holds a connection and you
  keep being billed for tokens you have decided to discard.
- **Offering "just retry the whole request" as the fix.** The reader has half an
  assessment on screen. Retrying produces a second one underneath it.

### Then answer — this is the exercise

**What should you have validated before the first byte?**

Everything. Auth, schema, case state, whether retrieval actually returned
anything, content policy on the prompt.

> Streaming does not remove error handling.
> It moves all of it earlier, and makes the part that happens later much harder.

And state the rule that follows:

> **Never stream the structured decision profile.** A partially streamed JSON
> object is not valid JSON — you cannot schema-validate it, cannot check for
> forbidden fields, and cannot refuse to release it, because you already sent
> 200 OK.
>
> Stream prose to a human. Return structure as a whole, validated.

---

# Exercise 6 · Is `retryable` documentation, or behaviour?

**25 minutes · the one that outranks the five scenarios**

### The brief

You have designed a `retryable` field. **Does the consuming team's client
actually branch on it?**

If nobody consumes it, you have designed documentation. This exercise turns
that from a talking point into a number.

### The prompt

```
Write a Colab cell that simulates 1,000 API requests against three different
client retry implementations and counts what each one wastes.

Error mix (name, probability, status, retryable, retry_after):
  success               0.70  202  None   None
  schema-violation      0.05  400  False  None
  document-unreadable   0.04  422  False  None
  content-filtered      0.03  422  False  None
  rate-limited          0.05  429  True   30
  unsupported-format    0.03  500  False  None
  extraction-timeout    0.04  504  True   60
  model-capacity        0.03  503  True   15
  invalid-model-output  0.03  500  True   5

Three clients, each deciding whether to retry:
  A. naive          — retries anything that is not 2xx
  B. status only    — retries 5xx
  C. reads retryable — branches on the retryable field

Rules: at most 4 attempts. A retryable failure clears on about 60% of retries.

For each client, count and print:
  - total calls made
  - WASTED calls: retries of something that will never succeed
  - LOST CASES: a retryable failure the client refused to retry
  - dollar cost of the wasted calls, at $0.014 per processed request

Then explain, referring to specific rows in the error mix, why client B fails
in BOTH directions.
```

### Accept it when

- [ ] All three clients are counted separately
- [ ] Wasted calls **and** lost cases are both reported
- [ ] Client B shows failures in both columns — not zero
- [ ] The explanation names the two rows that cause it

### It will probably get wrong

- **Producing an error mix where B scores perfectly.** If every 5xx is retryable
  and every 4xx is not, "retry 5xx" is exactly equivalent to reading the field
  and the whole argument collapses. That is why the mix above contains a
  **retryable 429** and a **non-retryable 500** — if your generated version has
  quietly dropped or altered those rows, put them back.
- **Only counting wasted calls.** Lost cases are the opposite failure and the
  more expensive one: a transient blip nobody retried, so a case sits
  unprocessed until somebody chases it. A client can score zero waste by never
  retrying anything.
- **Concluding "always retry 5xx".** The mix contains a deterministic 500 that
  will fail identically every time.

### The finding

Your table should land roughly here:

| Client | Calls | Wasted | Lost cases |
|---|---|---|---|
| Naive | ~1,700 | ~470 | 0 |
| **Status code only** | ~1,250 | **~78** | **~35** |
| Reads `retryable` | ~1,220 | **0** | **0** |

The middle row is the point. It is the one that **looks professional** and it
fails in both directions at once, because two error types have a status code
that disagrees with the instruction:

```
rate-limited        429  retryable=True    a 4xx it should retry and does not
unsupported-format  500  retryable=False   a 5xx it retries and never should
```

Neither is contrived. A 429 is one of the commonest errors any API returns.

### Then do the actual FDE work

Draft the message you would send the consuming team **before go-live**:

> *"Our error bodies carry a `retryable` boolean and `retry_after_seconds`.
> Does your client branch on those, or on the status code? If the status code —
> here are the two error types where that breaks, and what it costs."*

If the answer is *"we retry 5xx"*, you have three options and all are fine
**provided you choose one**:

1. Change your taxonomy so the heuristic works — no retryable 5xx
2. Get them to read the field
3. Accept the waste, and know the number

What is not fine is emitting the field, documenting it, and assuming.

> **If nobody consumes it, you have designed documentation, not behaviour.**

---

# The defence — 15 minutes

Present to another pod. No notebook open.

1. For every 4xx in your taxonomy, what does the caller do differently to
   succeed? If you cannot answer for one of them, it is probably a 5xx.
2. Which of your errors is a **retryable 5xx**, and which is a **non-retryable
   5xx**? What does that do to a client using "retry 5xx"?
3. A replayed idempotency key returns the original 202. **What returns a 409**,
   and is that difference written down anywhere a consuming developer will see?
4. Your streaming endpoint fails at chunk 12. What does the reader see, and
   what should they see?
5. Which endpoints stream, and what stops somebody adding streaming to one that
   returns structured output?
6. **What did Gemini get wrong that you caught** — and what did you only catch
   because the acceptance criteria told you to look?

The last question is what is being assessed. Anybody can accept working code.
The job is knowing what "working" was supposed to mean.

---

## Trainer notes

**Do not release the failure lists in advance.** Print with those sections
folded, or hand out briefs and prompts first and release the rest at the
halfway point.

**Exercise 2 is where most pods will fail, and it is the useful failure.**
Returning 500 for the unreadable document is the single most likely generated
error today, and the reasoning offered — *"an upstream service failed, so it is
a server error"* — is exactly the kind of plausible wrongness this session
exists to inoculate against. Let a pod present it before you correct it.

**Exercise 6 has a trap for the exercise itself.** If a pod's generated error
mix quietly drops the retryable 429 or the non-retryable 500, client B scores
zero on both columns and they will conclude the field is unnecessary. Watch for
it — and if it happens, that is a better teaching moment than getting it right,
because it shows how easily a benchmark can be built that proves the thing you
already believed.

**Three lanes.** Someone should be maintaining the acceptance criteria as a
checklist the pod signs off against, and someone should be drafting the
Exercise 6 message to the consuming team — which is the artefact that actually
reaches the ADR.

**Budget.** Roughly **8 live calls per pod** — Exercise 3 uses two, Exercise 5
uses one stream, the rest are local. Trivial. But confirm before twenty-five
people start streaming concurrently against one deployment.
