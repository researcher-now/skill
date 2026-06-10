# Researcher API Recipes

Use `RESEARCHER_API_KEY` or `RESEARCHER_TOKEN` for bearer auth.

```bash
export RESEARCHER_BASE_URL="${RESEARCHER_BASE_URL:-https://researcher.now}"
export RESEARCHER_AUTH_HEADER="Authorization: Bearer ${RESEARCHER_API_KEY:-$RESEARCHER_TOKEN}"
```

## Health And Auth

```bash
curl -fsS "$RESEARCHER_BASE_URL/health"

curl -fsS "$RESEARCHER_BASE_URL/v1/me" \
  -H "$RESEARCHER_AUTH_HEADER"
```

## Plan Before Spending

Use `preflightPlan:true` for vague or high-stakes prompts.

```bash
curl -fsS -X POST "$RESEARCHER_BASE_URL/v1/runs" \
  -H "$RESEARCHER_AUTH_HEADER" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: plan-$(date +%s)" \
  -d '{
    "preflightPlan": true,
    "requestedBy": "customer-agent",
    "source": {"type": "topic", "topic": "Personal AI adoption and food-market impact"},
    "depth": "standard",
    "instructions": "Return success criteria, evidence lanes, likely sources, and uncertainty."
  }'
```

## Create Runs

All new work starts through `POST /v1/runs`. Change `source.type` for topics, URLs, feeds, or videos.

```bash
curl -fsS -X POST "$RESEARCHER_BASE_URL/v1/runs" \
  -H "$RESEARCHER_AUTH_HEADER" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: run-$(date +%s)" \
  -d '{
    "requestedBy": "customer-agent",
    "source": {"type": "topic", "topic": "Personal AI adoption and food-market impact"},
    "mode": "collection",
    "depth": "standard",
    "instructions": "Research the thesis, cite sources, map demand shifts, list implications, and flag uncertainty.",
    "limits": {"maxResearchLoops": 3, "maxCostUsd": 25}
  }'
```

```bash
curl -fsS -X POST "$RESEARCHER_BASE_URL/v1/runs" \
  -H "$RESEARCHER_AUTH_HEADER" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: source-$(date +%s)" \
  -d '{
    "requestedBy": "customer-agent",
    "source": {"type": "url", "url": "https://example.com/report", "scope": "domain"},
    "depth": "standard",
    "limits": {"maxSources": 300, "maxCostUsd": 25}
  }'
```

```bash
curl -fsS -X POST "$RESEARCHER_BASE_URL/v1/runs" \
  -H "$RESEARCHER_AUTH_HEADER" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: feed-$(date +%s)" \
  -d '{
    "requestedBy": "customer-agent",
    "source": {"type": "feed", "url": "https://example.com/feed.xml", "limit": 50},
    "limits": {"maxCostUsd": 10}
  }'
```

```bash
curl -fsS -X POST "$RESEARCHER_BASE_URL/v1/runs" \
  -H "$RESEARCHER_AUTH_HEADER" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: video-$(date +%s)" \
  -d '{
    "requestedBy": "customer-agent",
    "source": {"type": "video", "url": "https://www.youtube.com/watch?v=...", "transcriptMode": "native"},
    "instructions": "Extract transcript-grounded concepts, quotes, references, and gaps."
  }'
```

## Read A Run

```bash
RUN_ID="..."

curl -fsS "$RESEARCHER_BASE_URL/v1/runs/$RUN_ID/job" -H "$RESEARCHER_AUTH_HEADER"
curl -N "$RESEARCHER_BASE_URL/v1/runs/$RUN_ID/stream" -H "$RESEARCHER_AUTH_HEADER"
curl -fsS "$RESEARCHER_BASE_URL/v1/runs/$RUN_ID/results" -H "$RESEARCHER_AUTH_HEADER"
curl -fsS "$RESEARCHER_BASE_URL/v1/runs/$RUN_ID/markdown" -H "$RESEARCHER_AUTH_HEADER"
curl -fsS "$RESEARCHER_BASE_URL/v1/runs/$RUN_ID/sources" -H "$RESEARCHER_AUTH_HEADER"
curl -fsS "$RESEARCHER_BASE_URL/v1/runs/$RUN_ID/extractions" -H "$RESEARCHER_AUTH_HEADER"
curl -fsS "$RESEARCHER_BASE_URL/v1/runs/$RUN_ID/transcript" -H "$RESEARCHER_AUTH_HEADER"
curl -fsS "$RESEARCHER_BASE_URL/v1/runs/$RUN_ID/usage" -H "$RESEARCHER_AUTH_HEADER"
```

Treat `succeeded`, `failed`, and `cancelled` as terminal statuses. Remove terminal runs from queued or in-flight lists immediately. For failed or cancelled runs, surface status, error, watch URL, and usage instead of waiting for a report.

## Completion Webhooks

For unattended or multi-run agents, register an account-scoped terminal webhook before starting work:

```bash
curl -fsS -X POST "$RESEARCHER_BASE_URL/v1/webhooks" \
  -H "$RESEARCHER_AUTH_HEADER" \
  -H "Content-Type: application/json" \
  -d '{
    "event": "run.complete",
    "url": "https://example.com/researcher-webhook",
    "secret": "at-least-8-chars"
  }'
```

```bash
curl -fsS "$RESEARCHER_BASE_URL/v1/webhooks" -H "$RESEARCHER_AUTH_HEADER"
curl -fsS -X DELETE "$RESEARCHER_BASE_URL/v1/webhooks/$WEBHOOK_ID" -H "$RESEARCHER_AUTH_HEADER"
```

`run.complete` subscriptions receive terminal deliveries for succeeded, failed, and cancelled runs. The `x-researcher-event` header and payload event identify the actual terminal event, such as `run.succeeded` or `run.failed`. When a secret is configured, verify `x-researcher-signature`: `sha256=` plus HMAC-SHA256 of the raw JSON body.

## Chat With A Run

```bash
curl -fsS -X POST "$RESEARCHER_BASE_URL/v1/runs/$RUN_ID/chat" \
  -H "$RESEARCHER_AUTH_HEADER" \
  -H "Content-Type: application/json" \
  -d '{"message":"What are the strongest sources for the main conclusion?"}'
```

## Iterate

```bash
curl -fsS -X POST "$RESEARCHER_BASE_URL/v1/runs/$RUN_ID/iterate" \
  -H "$RESEARCHER_AUTH_HEADER" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: iterate-$(date +%s)" \
  -d '{
    "action": "focus",
    "focus": ["contrarian evidence", "quantitative support"],
    "instructions": "Tighten the report against the original prompt contract.",
    "limits": {"maxCostUsd": 5}
  }'
```

Supported actions are `start`, `pause`, `continue`, `deepen`, `focus`, `steer`, `report`, `fork`, `evaluate`, `cancel`, and `stop`.

```bash
curl -fsS -X POST "$RESEARCHER_BASE_URL/v1/runs/$RUN_ID/sources/$SOURCE_ID/redo" \
  -H "$RESEARCHER_AUTH_HEADER" \
  -H "Content-Type: application/json" \
  -d '{"instructions":"Pay special attention to concrete caveats."}'
```

## Organize, Search, Export

```bash
curl -fsS "$RESEARCHER_BASE_URL/v1/runs/$RUN_ID/diff?against=$PARENT_RUN_ID" -H "$RESEARCHER_AUTH_HEADER"
curl -fsS -X POST "$RESEARCHER_BASE_URL/v1/runs/$RUN_ID/tags" -H "$RESEARCHER_AUTH_HEADER" -H "Content-Type: application/json" -d '{"tags":["customer-ready"]}'
curl -fsS "$RESEARCHER_BASE_URL/v1/library/search?claim=agent%20payment%20rails" -H "$RESEARCHER_AUTH_HEADER"
curl -fsS "$RESEARCHER_BASE_URL/v1/library/evidence?q=agent%20payment%20rails" -H "$RESEARCHER_AUTH_HEADER"
curl -fsS "$RESEARCHER_BASE_URL/v1/library/contradictions" -H "$RESEARCHER_AUTH_HEADER"
curl -fsS -X POST "$RESEARCHER_BASE_URL/v1/runs/$RUN_ID/export" -H "$RESEARCHER_AUTH_HEADER" -H "Content-Type: application/json" -d '{"target":{"type":"webhook","url":"https://example.com/researcher-webhook"}}'
```

## Enumerate Account Work

```bash
curl -fsS "$RESEARCHER_BASE_URL/v1/runs" -H "$RESEARCHER_AUTH_HEADER"
curl -fsS "$RESEARCHER_BASE_URL/v1/account/chats" -H "$RESEARCHER_AUTH_HEADER"
curl -fsS "$RESEARCHER_BASE_URL/v1/account/balance" -H "$RESEARCHER_AUTH_HEADER"
curl -fsS "$RESEARCHER_BASE_URL/v1/account/keys" -H "$RESEARCHER_AUTH_HEADER"
```
