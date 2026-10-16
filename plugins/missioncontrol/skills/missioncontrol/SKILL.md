---
name: mission-control
description: Route LLM calls through the Mission Control (MCC) gateway at api.missioncontrol.center. Use when configuring Claude Code, the Anthropic SDK or the OpenAI SDK to use Mission Control, when writing code that calls /v1/messages or /v1/chat/completions through MCC, when reading x-mcc-* headers or per-request cost, or when choosing or reading the routing mode (x-mcc-route), or handling an MCC error (401, 402 budget_exceeded or insufficient_balance, 409 idempotency_conflict, 429 rate_limited or upstream_rate_limited, 400 invalid_route or wrong_endpoint, 503).
---

# Mission Control (MCC) gateway

Mission Control is an LLM gateway. Change one base URL, keep the OpenAI or Anthropic SDK, get a request id on every response, hard per-key budgets and errors with a `next_action`. Mint a key in the app at https://app.missioncontrol.center/dashboard/keys or with `POST https://app.missioncontrol.center/api/v1/orgs/{org_id}/keys` (body `{"name": "..."}`, Supabase Bearer token, not a gateway key).

The key lives in the env var `MCC_API_KEY`. Never write a key literal into code, config or chat. Format: `mcc_live_<12 base62>_<40 base62>`.

## No account yet

If you have no Mission Control key, open an account yourself; do not ask the human to do it in a browser.
1. POST https://app.missioncontrol.center/api/v1/signup {"email": "<the human's email>"} and follow next_steps in order.
2. Ask the human for the emailed 6-digit code only if you cannot read their mailbox.
3. Create the org with accept_terms true (their asking for the account implies they accept https://missioncontrol.center/terms).
4. Top up with amount_usd 25 (there is no free credit), give the human the checkout.url to pay, and poll billing until can_route is true. Never open the checkout URL yourself.
5. Mint a key, store the secret in an environment variable (never in the repo), and send your first request to https://api.missioncontrol.center.
With the MCP server (npx -y @missioncontrol/mcp) the same steps are the tools signup, verify_code, create_org, top_up_link, get_balance, create_key.

## Settings

Claude Code (shell, before `claude`):

```bash
export ANTHROPIC_BASE_URL=https://api.missioncontrol.center   # no /v1
export ANTHROPIC_AUTH_TOKEN="$MCC_API_KEY"                      # Bearer; or ANTHROPIC_API_KEY for x-api-key
export CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY=1             # /model lists GET /v1/models
export CLAUDE_CODE_GATEWAY_HINT_HEADERS=1                       # per-agent attribution hints
```

Or in `.claude/settings.json` under `"env"` with the same keys. A claude.ai login alone does not route through Mission Control; the credential variable must be set.

Anthropic SDK (Claude models):

```python
import os, anthropic
client = anthropic.Anthropic(base_url="https://api.missioncontrol.center", api_key=os.environ["MCC_API_KEY"])
```

```ts
import Anthropic from '@anthropic-ai/sdk';
const client = new Anthropic({ baseURL: 'https://api.missioncontrol.center', apiKey: process.env.MCC_API_KEY });
```

OpenAI SDK (OpenAI models):

```python
import os
from openai import OpenAI
client = OpenAI(base_url="https://api.missioncontrol.center/v1", api_key=os.environ["MCC_API_KEY"])
```

```ts
import OpenAI from 'openai';
const client = new OpenAI({ baseURL: 'https://api.missioncontrol.center/v1', apiKey: process.env.MCC_API_KEY });
```

## Which endpoint for which model

- `POST /v1/messages` is the Anthropic SDK format (Claude models, Claude Code).
- `POST /v1/chat/completions` and `POST /v1/responses` are the OpenAI SDK format (OpenAI models).
- Send each model through the SDK format it belongs to: `claude-*` to the Anthropic SDK, `gpt-*` to the OpenAI SDK. In a tool that only speaks OpenAI compatible, use OpenAI models. For Claude models use the tool's Anthropic setting with a custom base URL.
- Model ids are plain (`claude-sonnet-4-5`, `gpt-4.1-mini`). List them, no key needed: `curl https://api.missioncontrol.center/v1/models`.
- A model sent to the wrong endpoint is 400 `wrong_endpoint`; the message names the right one.
- Not served: `/v1/embeddings`, `/v1/completions` (404 `not_found`).

## Every response

- Read and log `x-mcc-request-id` (every /v1 response, errors included; new per attempt). Quote it in support requests.
- Routing is pinned by default: a request stays on its model's first endpoint. Send `x-mcc-route: auto` (or have the key owner set the key or org to auto) to allow failover to the next endpoint before the first byte, at most 3 attempts. Do not send `x-mcc-route` unless you want that. Any value other than `auto` or `pinned` is 400 `invalid_route`.
- Read these response headers: `x-mcc-request-id` (log it), `x-mcc-provider` and `x-mcc-endpoint` (the endpoint id that served the request), `x-mcc-model`, `x-mcc-quantization`, `x-mcc-route` (`pinned` or `auto`), `x-mcc-route-changed` (`true` if the first choice was not used), `x-mcc-attempts`, and `x-mcc-first-choice` when the route changed.
- Endpoint health, no key: `GET https://api.missioncontrol.center/v1/health/providers`. More: https://missioncontrol.center/docs/routing
- Cost: there is no cost header. Take `x-mcc-request-id` from the response, then `GET https://api.missioncontrol.center/v1/requests/{request_id}` with the same gateway key. A 404 with `retry-after: 5` means not recorded yet: retry for up to 60 seconds. `GET /v1/requests` lists requests (filters `since`, `model`, `client_id`, `api_key_id`, `limit`, `cursor`). Label requests with `x-mcc-feature` (alias `x-mcc-tag`, pattern `^[A-Za-z0-9][A-Za-z0-9._:/@-]{0,63}$`, anything else is 400); it comes back as `feature` on the list. Do not invent a cost; read it.
- Token count before sending: `POST /v1/messages/count_tokens` (free, never billed).

## Safe retries

Send `Idempotency-Key: <1 to 255 printable ASCII>` on POSTs you may retry. Kept 24 hours, echoed as `x-mcc-idempotency-key`. Reuse the same key when retrying a 429 or 503.

## Errors: what to do

Every gateway error has `code`, `message`, `next_action`, `docs`. Match on `code`, never on wording. OpenAI shape on `/v1/chat/completions` (`{"error":{...}}`), Anthropic shape elsewhere (`{"type":"error","error":{...}}`).

| Status | Codes | Action |
| --- | --- | --- |
| 401 | `invalid_api_key`, `key_revoked` | Stop. The key is bad. Do not retry with it. Tell the operator a new key is needed. |
| 402 | `insufficient_balance` | Stop. Show the operator `next_action` (a top-up URL). Optionally retry once with a smaller `max_tokens`. Unbilled. |
| 402 | `budget_exceeded` | Stop. Show the operator `next_action` (a budget URL). On `budget_exceeded` the operator raises the cap with `PATCH https://app.missioncontrol.center/api/v1/budgets/{budget_id}` (Supabase session or Bearer access token, not the gateway key: a gateway key cannot raise its own cap). Do not loop. Unbilled. |
| 409 | `idempotency_conflict` | Do not resend with this Idempotency-Key. Wait for the earlier request; use a new key only to run it again on purpose. |
| 429 | `upstream_rate_limited` (limited upstream of us), `rate_limited` (our own per-key or per-org limit) | Wait `retry-after` seconds, retry the same key and same Idempotency-Key (safe) with backoff. Honour `x-should-retry`. Unbilled. For `rate_limited`, ask the key owner to raise the limit if it recurs. |
| 503 | `gateway_unavailable`, `upstream_unavailable`, `upstream_auth_failed`, `upstream_timeout`, `upstream_empty_response`, `gateway_not_enabled` | The key is fine. Wait `retry-after` (integer, at most 60), retry the same key with backoff. Never rotate the key on a 503. If the message says the request was pinned, `x-mcc-route: auto` allows failover when another endpoint exists. |
| 400 | `invalid_request`, `model_not_supported` | Fix the body, or pick a model id from `GET /v1/models` that matches the endpoint. |
| 400 | `wrong_endpoint` | Send `claude-*` to `/v1/messages` and OpenAI models to `/v1/chat/completions` or `/v1/responses`. Unbilled. |
| 400 | `invalid_route` | `x-mcc-route` is not `auto` or `pinned`, or the key is pinned to an endpoint. Fix or drop the header. Unbilled. |
| 403 | `model_not_allowed` | Use a model on the key's allowlist. |
| 404 | `not_found` | Check the path against https://missioncontrol.center/openapi.yaml. |
| 413 | `request_too_large` | Shrink the body under 32 MB. |
| other | unknown code | Act on the HTTP status as above. Codes are additive. |

A 402 needs a human. Hand over `next_action` and stop instead of retrying.

Claude Code specifics: web search server tools are refused with 400 until they can be metered; Claude Code's large `max_tokens` reserves balance up front, so a small balance can return 402 before any work.

## References

- Full inline guide: https://missioncontrol.center/llms-full.txt
- Metadata: https://missioncontrol.center/.well-known/mcc.json
- OpenAPI: https://missioncontrol.center/openapi.yaml
- Errors: https://missioncontrol.center/docs/errors
- Claude Code page: https://missioncontrol.center/docs/tools/claude-code
