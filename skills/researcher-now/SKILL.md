---
name: researcher-now
description: |
  Article, video, and deep research analysis via researcher.now. Use when the user says: summarize this YouTube video, extract this article, YouTube transcript, research X, find sources on Y, what does this video say. Single URLs return formatted content PLUS finished structured analysis (claims, facts, quotes) in seconds for cents; deep research returns a cited, synthesized Research Report with reusable source records, a live watch URL, and run-scoped Q&A.
license: MIT
metadata:
  author: Researcher
  homepage: https://researcher.now
  source: https://github.com/researcher-now/skill
---

# Researcher

Use Researcher when the work should become a durable, inspectable research run instead of a one-off answer. A good run gives the user a watch URL immediately, then stores sources, artifacts, costs, report output, and run-scoped chat context.

## Product Model

Think in five verbs:

1. Plan: preview a vague or high-stakes request before spending.
2. Run: start durable server-side work and share the watch URL immediately.
3. Inspect: read the truth bundle, markdown, sources, source extractions, usage, warnings, and events.
4. Iterate: fork, continue, deepen, focus, steer, re-extract, or regenerate the report.
5. Export: save markdown plus structured sources/source-extractions downstream.

## Setup

- Required input: `RESEARCHER_API_KEY` or `RESEARCHER_TOKEN`, a customer Researcher API token from `https://researcher.now/account/?setup=agent`.
- Optional input: `RESEARCHER_BASE_URL`; default it to `https://researcher.now`.
- Use `RESEARCHER_API_KEY` as the bearer token. Accept `RESEARCHER_TOKEN` as an alias if the environment already uses it.
- If no customer token is available, send the user to `https://researcher.now/account/?setup=agent`.
- Never search local files, shell history, or logs for customer tokens.
- Never use admin-token, payment-bypass, or internal-only routes in customer workflows.

## Choose The Operation

- Use `POST /v1/analyze` with `{ "url": ... }` for a SINGLE article or YouTube video when the user wants it read/summarized/analyzed now: synchronous (seconds), costs cents, returns formatted markdown or a speaker-aware transcript PLUS the structured analysis (claims, facts, quantitative data, quotes). Prefer this over a run for one-URL asks — only start a run for multi-source synthesis or a durable Research Report.
- Use `POST /v1/runs` with `preflightPlan:true` before spending when the customer prompt is vague, high-stakes, or needs an explicit success checklist.
- Use `POST /v1/runs` for all new durable work. Pick `source.type` as `topic`, `url`, `feed`, or `video`; include `requestedBy` and an `Idempotency-Key`.
- Use `source.type:"video"` for a YouTube URL, podcast/video link, or transcript-first task. Do not scrape video as a generic webpage.
- Use `source.type:"url"` for a known webpage/domain extraction and `source.type:"feed"` for one-shot RSS, Atom, or JSON Feed ingestion.
- Use `GET /v1/me` to confirm whether the bearer token is a customer or admin token before spending.
- Use `GET /v1/runs`, `/v1/library`, or `/v1/library/recall?q=*` to enumerate existing runs and recall.
- Use `GET /v1/runs/:id/job` for live run state. Read top-level `status`, `currentStage`, `currentActivity`, and `state`; legacy `job.status` remains nested under `job`.
- Use `POST /v1/webhooks` with `event:"run.complete"` before starting unattended or multi-run work. If no webhook receiver is available, stream or poll until `succeeded`, `failed`, or `cancelled`.
- Use `GET /v1/runs/:id/results` or `/markdown` to read a completed run.
- Use `GET /v1/runs/:id/transcript` for typed video transcript segments instead of raw transcript metadata blobs.
- Use `POST /v1/runs/:id/sources/:sourceId/redo` when a source extracted weakly and the run can be salvaged without forking.
- Use `GET /v1/library/search?claim=...` for account-level claim recall across completed runs.
- Use `POST /v1/runs/:id/chat` for questions scoped to one completed run, or the `/chat/sessions` endpoints for multi-turn UI flows.
- Use `POST /v1/runs/:id/iterate` as the canonical iteration wrapper for start, pause, continue, deepen, focus, steer, report, fork, evaluate, cancel, and stop. Include `Idempotency-Key` on paid iterate calls.
- Use `POST /v1/runs/:id/iterate` with `action:"cancel"` or `action:"stop"` to abort active work.
- Use `/tags`, `/collections`, `/diff`, `/export`, and `/webhooks` when organizing runs, comparing continuations, or sending terminal runs to downstream systems.

## Run Workflow

1. Convert the user's ask into a precise research prompt. Preserve requested output shape, required sources, time horizon, evaluation criteria, and exclusions.
2. Create the run and return the watch URL as soon as the API returns it. Do not wait for completion before sharing the link.
3. For unattended or multi-run work, register an account-scoped `run.complete` webhook or keep polling/streaming until the run reaches `succeeded`, `failed`, or `cancelled`.
4. Remove terminal runs from queued or in-flight lists immediately. For failed/cancelled runs, surface the status, error, watch URL, and usage instead of waiting for a report.
5. For succeeded runs, fetch results, markdown, sources, extractions, and usage/cost if the user needs the final artifact or an audit.
6. For quality-sensitive work, review the report against the original prompt. Continue, deepen, focus, or fork instead of declaring success when prompt requirements are missing.

## Prompt Contract

For research runs, include these fields in the prompt whenever they are known:

- `Objective`: the decision, hypothesis, or question being answered.
- `Output shape`: memo, ranked table, bullets, source list, thesis map, etc.
- `Evidence bar`: primary sources, expert commentary, recent sources, contrarian views, quantitative support, or transcript timestamps.
- `Scope`: geography, dates, companies, market segment, source types, and exclusions.
- `Quality checks`: what must be true before the run is considered good.

## Billing And Failure Handling

- Use `maxCostUsd` as a cap when the user gives a budget. A budget is a ceiling, not a target spend.
- If the API returns `402` or wallet/payment authorization errors, use `account_funding_url` from the response when present. Otherwise send the user to `https://researcher.now/account/`.
- If a stage fails, treat the run as terminal, surface the run URL and the useful error message, and remove it from queued/in-flight state. Do not replace a failed Researcher run with a generic web-search summary.
- If the run has weak evidence or misses prompt requirements, use continuation or forked follow-up work rather than pretending the output is complete.

## Output Style

When creating a run, return:

- the run title or topic
- the Researcher run URL
- the run id
- budget/cap if one was set
- what the run is currently doing, if available

When reading or reviewing a run, cite the Researcher URL and state whether the report satisfies the user's prompt contract. Include concrete missing requirements when it does not.

## Iteration Decision Table

- `iterate`: canonical wrapper; pass `action` as `start`, `pause`, `continue`, `deepen`, `focus`, `steer`, `report`, `fork`, `evaluate`, `cancel`, or `stop`.
- `fork`: create a separate reviewable child version without starting new paid work.
- `continue`: add more work to the same run or an approved fork.
- `deepen`: convenience continuation for broader/deeper acquisition.
- `focus`: convenience continuation narrowed to named topics.
- `steer`: mid-run guidance consumed at worker checkpoints.
- `report`: regenerate/update the report from the current corpus without more acquisition.
- `cancel`/`stop`: abort active work and settle measured usage.

## Downstream Export

When saving a completed run to another knowledge store, fetch:

- `/markdown` for the human-readable memo.
- `/sources` for stored source rows.
- `/extractions` for structured claims, facts, caveats, quantitative facts, stakeholder positions, and citation-ready source records.
- `/transcript` for typed transcript segments on YouTube/video runs.
- `/usage` for cost audit.

For YouTube/video runs, avoid sending `metadata.raw.content` transcript fragment blobs into downstream LLM calls. Prefer the report, source records, `/transcript`, or transcript tab data.

Use `POST /v1/runs/:id/export` for webhook-shaped one-off export, or register account completion webhooks with `POST /v1/webhooks`. `run.complete` subscriptions receive succeeded, failed, and cancelled terminal deliveries; verify `x-researcher-signature` when a secret is configured.

## Recipes

For copy-pasteable HTTP examples, endpoint payloads, and run-control recipes, load `references/api-recipes.md`.
