---
name: phoenix-tool-trace-search
description: Use when searching Phoenix traces or spans for whether a specific tool was called, especially with phoenix-cli, PHOENIX_* environment variables, project names, trace IDs, span names, or OpenTelemetry tool.name attributes.
---

# Phoenix Tool Trace Search

Use `phoenix-cli` to answer "was tool X called?" from Phoenix traces. Prefer exact GraphQL span filtering over broad `span list` output, because `span list --name` can return noisy surrounding spans and may not filter as strictly as expected on older Phoenix servers.

## Inputs

Read `.env` or supplied config for:

- `PHOENIX_COLLECTOR_ENDPOINT`, usually ending in `/v1/traces`
- `PHOENIX_API_KEY`
- `PHOENIX_PROJECT_NAME`

Convert collector endpoint to Phoenix API endpoint by removing `/v1/traces`.

```bash
API_ENDPOINT="${PHOENIX_COLLECTOR_ENDPOINT%/v1/traces}"
```

## Workflow

1. Confirm `phoenix-cli` exists:

```bash
command -v phoenix-cli
phoenix-cli --help
```

2. List projects and resolve the actual project ID/name. Phoenix project filtering may be substring-style, and service names may include prefixes not present in `.env`.

```bash
phoenix-cli project list \
  --endpoint "$API_ENDPOINT" \
  --api-key "$PHOENIX_API_KEY" \
  --format json \
  --no-progress \
  --limit 100
```

3. Use GraphQL exact span-name filtering on the resolved project ID. This is the most reliable yes/no check.

```bash
phoenix-cli api graphql \
  "{ node(id:\"$PROJECT_ID\") { ... on Project { recordCount(filterCondition:\"name == \\\"$TOOL_NAME\\\"\") spans(first:10, filterCondition:\"name == \\\"$TOOL_NAME\\\"\") { edges { node { id name spanKind startTime endTime statusCode statusMessage context { traceId spanId } attributes input { value mimeType } output { value mimeType } } } } } } }" \
  --endpoint "$API_ENDPOINT" \
  --api-key "$PHOENIX_API_KEY"
```

If `recordCount` is `0`, report no exact span-name calls in that project.

4. If exact span-name search returns nothing but the user expects calls, try the OpenInference tool attribute as a fallback:

```bash
phoenix-cli api graphql \
  "{ node(id:\"$PROJECT_ID\") { ... on Project { recordCount(filterCondition:\"tool.name == \\\"$TOOL_NAME\\\"\") spans(first:10, filterCondition:\"tool.name == \\\"$TOOL_NAME\\\"\") { edges { node { id name spanKind startTime endTime statusCode context { traceId spanId } input { value mimeType } output { value mimeType } } } } } } }" \
  --endpoint "$API_ENDPOINT" \
  --api-key "$PHOENIX_API_KEY"
```

If the condition fails validation, use `validateSpanFilterCondition` to confirm syntax.

```bash
phoenix-cli api graphql \
  "{ node(id:\"$PROJECT_ID\") { ... on Project { validateSpanFilterCondition(condition:\"name == \\\"$TOOL_NAME\\\"\") { isValid errorMessage } } } }" \
  --endpoint "$API_ENDPOINT" \
  --api-key "$PHOENIX_API_KEY"
```

## CLI Fallbacks

For quick local inspection or when GraphQL fields differ by Phoenix version, write spans to `/tmp` and parse with `jq`:

```bash
phoenix-cli span list /tmp/phoenix_tool_spans.json \
  --endpoint "$API_ENDPOINT" \
  --api-key "$PHOENIX_API_KEY" \
  --project "$PROJECT_NAME" \
  --format json \
  --no-progress \
  --span-kind TOOL \
  --name "$TOOL_NAME" \
  -n 1000

jq '[.[] | select((.name == "'"$TOOL_NAME"'") or ((.attributes["tool.name"]? // "") == "'"$TOOL_NAME"'"))] | length' /tmp/phoenix_tool_spans.json
jq '[.[] | select((.name == "'"$TOOL_NAME"'") or ((.attributes["tool.name"]? // "") == "'"$TOOL_NAME"'")) | {id, name, trace_id:.context.trace_id, start_time, end_time, status_code, input:.attributes["input.value"], output:.attributes["output.value"]}]' /tmp/phoenix_tool_spans.json
```

Do not trust the raw length of a `span list --name` result as the exact match count; inspect `name` and `attributes["tool.name"]`.

## Reporting

Report:

- Phoenix API endpoint used
- resolved project name and ID
- exact match count
- for matches: timestamp, trace ID, span ID, status, input, output
- if no matches: mention any nearby or related tool calls only if discovered during fallback inspection

Do not expose full API keys in the final answer.
