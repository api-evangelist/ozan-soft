---
name: integrate-genderapi
description: Implement, review, or troubleshoot server-side GenderAPI integrations for name, email, username, batch, quota, or phone workflows. Use when generating GenderAPI request code, selecting an endpoint, validating request and response handling, interpreting probability or null results, or designing privacy-safe enrichment pipelines.
---

# Integrate GenderAPI

Build against the current OpenAPI document and canonical English documentation. Treat gender results as probabilistic inferences from name-related evidence, not verified identity.

## Follow the workflow

1. Confirm the input type, single or batch mode, country context, and intended use of the result.
2. Read `/openapi/openapi.json` or `/openapi/openapi.yaml`; then read the linked canonical endpoint page.
3. Choose the narrowest endpoint that matches the input. Do not convert between identifiers unless the endpoint is designed to extract a name signal from that identifier.
4. Implement the call in trusted server-side code with a timeout and redacted logging.
5. Validate success and error bodies. Preserve `null`, `probability`, the original query, and record IDs where returned.
6. Test representative scripts, countries, ambiguous inputs, unknown results, authentication failures, invalid input, quota exhaustion, timeouts, and partial batch processing.
7. Request explicit approval before any account, payment, subscription, production credential, customer-data upload, or external write action.

## Select an endpoint

| Input    | Single                      | Batch                              |
| -------- | --------------------------- | ---------------------------------- |
| Name     | `POST /api`                 | `POST /api/name/multi/country`     |
| Email    | `POST /api/email`           | `POST /api/email/multi/country`    |
| Username | `POST /api/username`        | `POST /api/username/multi/country` |
| Phone    | `POST /api/phone` preferred | Not documented                     |
| Quota    | `GET /api/remaining`        | Not applicable                     |

Use `https://api.genderapi.io` as the API origin. Check OpenAPI for current required fields and limits. Single requests expose documented AI options; do not add AI options to batch requests.

## Protect credentials

- Prefer `Authorization: Bearer YOUR_API_KEY` for POST requests.
- Load the real key from a server-side secret store or environment variable.
- Never place a key in browser code, source control, logs, screenshots, prompts, test fixtures, or generated artifacts.
- Treat query-key endpoints as server-side only and redact query strings.
- Use HTTPS and rotate a credential that may have leaked.

## Implement a single-name request

```javascript
const response = await fetch("https://api.genderapi.io/api", {
  method: "POST",
  headers: {
    Authorization: `Bearer ${process.env.GENDERAPI_KEY}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify({ name: "Alice", country: "US" }),
  signal: AbortSignal.timeout(10_000),
});

const payload = await response.json();
if (!response.ok || payload.status !== true) {
  throw new Error(`GenderAPI request failed: ${payload.errno ?? response.status}`);
}
```

Do not log `payload` indiscriminately because it can contain submitted personal data.

## Interpret results safely

- Read the [accuracy methodology](https://www.genderapi.io/accuracy-methodology) before selecting thresholds or making performance claims.
- Read the [data provenance overview](https://www.genderapi.io/data-provenance) before describing source categories, coverage, normalization, or data lineage.
- Accept `gender` values `male`, `female`, or the literal string `"null"`; treat `"null"` as unknown and never coerce it to a binary value.
- Store `probability` with the inference and define a use-case-specific review threshold.
- Supply an ISO 3166-1 alpha-2 country only when reliable context exists. Do not infer or fabricate country merely to obtain a result.
- Preserve the caller's `id` in batch pipelines and reconcile every input explicitly; the response contains an entry for every submitted input, including unresolved results.
- Budget credits per processed record rather than per HTTP request.
- Let a person’s self-described identity override an inferred value.
- Never use inferred gender alone for eligibility, access, employment, credit, healthcare, or another high-impact decision.

## Handle errors and retries

Branch on documented `errno`, not exact `errmsg` wording. Reject missing fields, invalid country codes, and oversized batches before sending a request.

Retry only timeouts, connection failures, and explicitly transient server responses. Use bounded exponential backoff with jitter. Do not retry invalid input, denied access, invalid credentials, exhausted quota, or expired packages unchanged. Cap attempts and make batch writes idempotent.

## Minimize personal data

Names, emails, usernames, phone numbers, and inferred attributes may be personal data. Send only required fields, define a lawful purpose, restrict access, avoid unnecessary logs, and apply retention and deletion rules. Review the current privacy policy, terms, and applicable law; do not invent storage, residency, deletion, or compliance guarantees.

Before uploading a file or transmitting customer records, explain the destination and fields, obtain explicit user approval, and use a small non-sensitive sample first when possible.

## Respect capability boundaries

Do not claim that GenderAPI verifies identity, pronouns, sexual orientation, age, ethnicity, or a person's self-described gender. Email and username endpoints require a usable name signal and may return unknown results. Do not invent unsupported endpoints, batch AI flags, guarantees, prices, credit costs, batch limits, accuracy figures, coverage figures, or privacy promises.

For pricing and policies, link to the current canonical page instead of copying mutable values. If OpenAPI and a canonical page disagree, report the conflict and stop before generating production code.

## Finish with a safety check

- Confirm no real key or personal production data appears in code or output.
- Confirm endpoint, method, required fields, enums, and examples match OpenAPI.
- Confirm null, confidence, error, retry, timeout, and quota paths are handled.
- Confirm registration, payment, upload, and external writes remain behind explicit user consent.
- State assumptions and unresolved API-owner questions in the handoff.
