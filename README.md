# Mission Control (MCC)

One base URL for every model, with per-request cost, budgets and alerts, run-level traces and exports.

- Site: https://missioncontrol.center
- API base URL: `https://api.missioncontrol.center`
- Docs: https://missioncontrol.center/docs/
- For agents: https://missioncontrol.center/docs/for-agents/
- llms.txt: https://missioncontrol.center/llms.txt (full text: https://missioncontrol.center/llms-full.txt)
- OpenAPI: https://missioncontrol.center/openapi.yaml

## What it is

Change the base URL in the SDK you already use. Every request then carries a request id, and its cost can be read back by that id. You can cap spend per key, client or org, group the requests of one agent task into a traced run, and export everything. Agents can sign up and mint their own keys through the API.

Keys start with `mcc_live_`. Create one at https://missioncontrol.center.

## Quickstart

```python
# pip install openai anthropic
import os, anthropic
from openai import OpenAI

key = os.environ["MCC_API_KEY"]
oa = OpenAI(base_url="https://api.missioncontrol.center/v1", api_key=key)
r = oa.chat.completions.with_raw_response.create(model="gpt-4.1-mini", messages=[{"role": "user", "content": "Hello"}])
print(r.headers["x-mcc-request-id"], r.parse().choices[0].message.content)

an = anthropic.Anthropic(base_url="https://api.missioncontrol.center", api_key=key)  # no /v1
m = an.messages.create(model="claude-sonnet-4-5", max_tokens=64, messages=[{"role": "user", "content": "Hello"}])
print(m.content[0].text)
```

Read what a request cost:

```bash
curl https://api.missioncontrol.center/v1/requests/<x-mcc-request-id> -H "Authorization: Bearer $MCC_API_KEY"
```

The list of served models is public: `GET https://api.missioncontrol.center/v1/models`.

## Claude Code plugin

```
claude plugin marketplace add kartikey54/mcc
claude plugin install missioncontrol@missioncontrol
```

On Claude Code 2.1.275 or later: `/plugin install missioncontrol --marketplace kartikey54/mcc`. The plugin adds a skill that sets `ANTHROPIC_BASE_URL` and explains the request id, cost lookup and errors. Details: https://missioncontrol.center/docs/tools/claude-code/

## Packages

- `@missioncontrol/mcp`: MCP server for keys, budgets, usage and the model catalog. The package name is reserved; it is not published to npm yet. Setup notes are in the docs: https://missioncontrol.center/docs/mcp/
- `@missioncontrol/sdk`: typed client for the Mission Control API. Coming to this repository.

Source for both lands here when they are published.

## Links

- Quickstart: https://missioncontrol.center/docs/quickstart/
- Budgets: https://missioncontrol.center/docs/budgets/
- Errors: https://missioncontrol.center/docs/errors/
- Claude Code: https://missioncontrol.center/docs/tools/claude-code/
- Support: support@missioncontrol.center

## Contributing and license

See [CONTRIBUTING.md](CONTRIBUTING.md). Released under the [MIT License](LICENSE).
