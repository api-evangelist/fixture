# Fixture

Fixture is an AI-native CRM built for startups, backed by Y Combinator (Winter 2026 batch). It aggregates every customer interaction — email, calendar, Slack and Slack Connect, meeting notes from Granola, Notion and Circleback, and Stripe billing events — into a single structured activity graph across Accounts, Contacts, Deals, Leads, Tasks and Notes, then uses agents to surface next actions and keep records current without manual data entry.

- Website: https://fixture.app
- Docs: https://fixture.app/docs
- API reference: https://fixture.app/docs/api-reference/overview
- Status: https://status.fixture.app

## API surface

| Surface | Where |
| --- | --- |
| REST v1 API | `https://beta-api.fixture.app/api/v1` |
| OpenAPI 3.1 | [`openapi/fixture-v1-openapi.json`](openapi/fixture-v1-openapi.json) — generated and published by the provider |
| Remote MCP server | `https://beta-api.fixture.app/api/mcp` (OAuth 2.1, PKCE S256, dynamic client registration) |
| CLI | `fixture` — hosted installer, agent-oriented (`fixture agent-context`) |
| llms.txt | https://fixture.app/docs/llms.txt |

Auth is bearer-only: a workspace API key (`fx_` prefix) for service-to-service integrations, or a Fixture OAuth access token for CLI, Agent, and MCP clients. Session cookies are rejected on `/api/v1/*`.

Backed by: y-combinator
