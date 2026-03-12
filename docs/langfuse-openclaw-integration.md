# Langfuse + OpenClaw: practical setup pitfalls and how to avoid them

This document is a practical reference for setting up Langfuse with OpenClaw without wasting time on avoidable mistakes.

It is not a host-specific postmortem and not a dump of one machine's local setup. The goal is simpler: list the common failure modes, show how to validate each layer, and make it harder to misdiagnose what is actually broken.

## Scope

Goal:
- get **technical observability** from OpenClaw into Langfuse
- get **content-level traces** for chat runs, not only technical spans
- avoid leaking secrets into docs or config examples
- avoid leaking internal prompt junk, full history, or hidden reasoning into Langfuse

Non-goals:
- this is not a generic Langfuse tutorial
- this is not a polished upstream plugin package
- this is not a guarantee that stock OpenClaw versions will ship the same dependency state forever

## Final architecture

Two layers were used:

1. **Stock OpenClaw `diagnostics-otel` plugin**
   - exports technical OTel diagnostics
   - useful for `message.processed`, `model.usage`, queue/session metrics
   - does **not** give a nice chat transcript view by itself

2. **Custom workspace plugin `langfuse-content`**
   - hooks into OpenClaw runtime lifecycle
   - sends `trace-create`, `generation-create/update`, and `span-create/update` to Langfuse ingestion API
   - gives content-level chat traces and tool spans

In practice:
- keep `diagnostics-otel` if you want technical telemetry
- add `langfuse-content` if you want actual conversational traces

## Where to inspect when something is unclear

The exact file paths depend on how OpenClaw is installed, so do not hard-code one machine's layout into your mental model.

What usually matters:

### OpenClaw config
- the main OpenClaw config file, typically something like `~/.openclaw/openclaw.json`

### Stock OTel exporter/plugin
- the `diagnostics-otel` extension source or package metadata
- its runtime dependencies
- plugin load status in OpenClaw

### Custom content plugin
- your plugin manifest (`openclaw.plugin.json`)
- the plugin entry file
- the plugin config block under `plugins.entries.<id>.config`

### Useful reference material
- OpenClaw logging docs
- OpenClaw plugin docs
- plugin manifest/schema docs
- plugin hook type definitions

If local docs are available, use them. If not, inspect the installed package source and hook types directly.

## Prerequisites

You need:
- Langfuse project created already
- Langfuse public key
- Langfuse secret key
- access to patch OpenClaw config
- ability to restart the OpenClaw gateway

Do **not** paste live keys into notes, commits, or docs.
Use placeholders in documentation and redacted config output when quoting anything.

## Step 1. Inspect schema before changing config

Do not guess config paths.
OpenClaw already exposes schema lookup. Use it first.

Relevant paths:
- `diagnostics`
- `diagnostics.otel`
- `plugins`
- `plugins.entries.<id>`
- `plugins.entries.<id>.config`

Why this matters:
- avoids inventing nonexistent fields
- keeps patches aligned with current OpenClaw version
- lets you verify restart semantics and plugin config placement

## Step 2. Enable stock `diagnostics-otel`

### Intended effect

This sends technical telemetry to Langfuse via OTLP.
It is useful, but by itself it is **not** the full chat/content integration.

### Minimal config shape

Use your real values, not the placeholders below.

```json5
{
  diagnostics: {
    enabled: true,
    otel: {
      enabled: true,
      endpoint: "https://cloud.langfuse.com/api/public/otel",
      protocol: "http/protobuf",
      headers: {
        Authorization: "Basic <base64(publicKey:secretKey)>"
      },
      serviceName: "openclaw-gateway",
      traces: true,
      metrics: false,
      logs: false,
      sampleRate: 1,
      flushIntervalMs: 60000
    }
  },
  plugins: {
    entries: {
      "diagnostics-otel": {
        enabled: true
      }
    }
  }
}
```

### Important notes

- Langfuse OTLP endpoint used here was:
  - `https://cloud.langfuse.com/api/public/otel`
- OpenClaw stock exporter appends `/v1/traces` itself when needed.
- Protocol supported in this path was `http/protobuf`.

## Step 3. Expect one ugly surprise: stock plugin may not load cleanly

Observed failure:

```text
Cannot find module '@opentelemetry/api'
```

Possible cause:
- the bundled `diagnostics-otel` extension exists
- but its runtime OTel deps are not actually present on disk in the installed extension directory

Practical fix to try:

```bash
cd <openclaw-install>/extensions/diagnostics-otel
npm install --omit=dev
```

Then restart OpenClaw and verify the plugin really loads.

### Why this is annoying

Because config can be correct while the plugin still fails at runtime.
So "config looks right" is not enough. You must verify the plugin actually loads.

## Step 4. Verify stock diagnostics separately from content tracing

Check plugin state:

```bash
openclaw plugins list
```

Check gateway state:

```bash
openclaw status
```

Check logs:

```bash
openclaw logs --plain --limit 200
```

### Very important interpretation detail

At first it looked like Langfuse was empty because `traces` looked empty.
That was misleading.

What actually happened:
- stock OTel data showed up in **Langfuse observations / telemetry-like objects**
- not as the nice content traces we wanted

So the wrong early conclusion was:
- "delivery is broken"

The correct conclusion was:
- "delivery works, but this exporter is only giving technical observability"

## Step 5. Build a separate content-tracing plugin

### Why a custom plugin was needed

Stock `diagnostics-otel` is for observability.
It does not naturally produce a clean Langfuse chat-style record with:
- user input
- assistant output
- tool spans grouped under the same run

So a workspace plugin was added at:

- `.openclaw/extensions/langfuse-content/`

### Plugin manifest

The plugin needs a valid `openclaw.plugin.json`.
OpenClaw validates discoverability/config from the manifest without executing the module.

Minimal shape used:

```json
{
  "id": "langfuse-content",
  "name": "Langfuse Content Tracing",
  "description": "Send content-level OpenClaw runs to Langfuse using the ingestion API.",
  "version": "0.1.0",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "enabled": { "type": "boolean" },
      "baseUrl": { "type": "string" },
      "publicKey": { "type": "string" },
      "secretKey": { "type": "string" },
      "environment": { "type": "string" },
      "traceName": { "type": "string" },
      "includeHistory": { "type": "boolean" },
      "includeSystemPrompt": { "type": "boolean" },
      "captureTools": { "type": "boolean" },
      "maxFieldChars": { "type": "integer" }
    }
  }
}
```

### Runtime hooks used

Relevant typed hooks in OpenClaw:
- `llm_input`
- `llm_output`
- `before_tool_call`
- `after_tool_call`
- `agent_end`

These hook names and event shapes are documented in:
- `/usr/lib/node_modules/openclaw/dist/plugin-sdk/plugins/types.d.ts`

### Data model used

For each LLM run:
- `trace-create` at input time
- `generation-create` at input time
- `generation-update` at output time
- `trace-create` again to update trace output/metadata
- `span-create` / `span-update` for tool calls

Transport used:
- Langfuse ingestion API: `POST /api/public/ingestion`

## Step 6. Enable the custom plugin correctly

### Config pattern

```json5
{
  plugins: {
    allow: ["langfuse-content", "telegram"],
    entries: {
      "diagnostics-otel": {
        enabled: true
      },
      "langfuse-content": {
        enabled: true,
        config: {
          enabled: true,
          baseUrl: "https://cloud.langfuse.com",
          publicKey: "<public-key>",
          secretKey: "<secret-key>",
          environment: "default",
          traceName: "openclaw.chat",
          includeHistory: false,
          includeSystemPrompt: false,
          captureTools: true,
          maxFieldChars: 12000
        }
      }
    }
  }
}
```

### Why `plugins.allow` matters

Without pinning trust, OpenClaw will warn that workspace plugin discovery is happening without explicit trust.

Observed warning shape:
- plugin auto-loaded from workspace
- OpenClaw recommended explicit trust via `plugins.allow`

Use `plugins.allow` for predictable loading and less noise.

## Step 7. Keep content payloads clean

This was the most important hygiene step.

### Bad early behavior

Early traces accidentally captured too much:
- full system prompt
- full history
- `lastAssistant`, which could include hidden/internal fields such as `thinking`

That is bad because:
- it bloats Langfuse
- it leaks internal scaffolding
- it increases token/data noise
- it violates the user's explicit preference not to leak hidden reasoning

### Fixes that were applied

Set plugin config to:
- `includeHistory: false`
- `includeSystemPrompt: false`
- `maxFieldChars: 12000`

And in code:
- do **not** store `lastAssistant` in Langfuse output
- store only `assistantTexts`

### Recommended output capture

Safe enough default:
- `trace.input`: current user prompt only
- `trace.output`: final assistant text only
- `generation.input`: current prompt plus small metadata like `imagesCount`
- `generation.output`: assistant text array only
- tool spans: keep, but sanitize aggressively

## Step 8. Sanitize aggressively

The custom plugin used a sanitizer with these principles:
- truncate long strings
- cut recursion depth
- omit obviously sensitive/binary-ish fields
- mask keys matching patterns such as:
  - `signature`
  - `audio`
  - `image`
  - `binary`
  - `base64`

This is not perfect, but it reduces accidental junk and binary payloads.

## Step 9. Validate through API, not vibes

Do not declare success because:
- config exists
- plugin loads
- logs are quiet

Actual validation must query Langfuse.

### Useful checks

#### Check traces

```bash
curl -u "$LANGFUSE_PUBLIC_KEY:$LANGFUSE_SECRET_KEY" \
  'https://cloud.langfuse.com/api/public/traces?limit=10&name=openclaw.chat'
```

#### Check generations

```bash
curl -u "$LANGFUSE_PUBLIC_KEY:$LANGFUSE_SECRET_KEY" \
  'https://cloud.langfuse.com/api/public/observations?limit=10&name=llm.run'
```

#### Check tool spans

```bash
curl -u "$LANGFUSE_PUBLIC_KEY:$LANGFUSE_SECRET_KEY" \
  'https://cloud.langfuse.com/api/public/observations?limit=10&name=tool.exec'
```

### What success looks like

For content tracing:
- trace name like `openclaw.chat`
- `trace.input` contains the actual user request
- `trace.output` contains the assistant reply
- `llm.run` generation exists with input/output
- tool spans appear under the same trace

### What partial success looks like

If only technical integration works:
- you may see `message.processed`
- you may see `model.usage`
- you may see OTel-style traces/observations
- but no clean chat input/output

That means content plugin is missing or broken.

## Operational pitfalls discovered

### 1. `traces` being empty does not automatically mean delivery is broken

Sometimes you are looking at the wrong object class.
Stock OTel data may appear in a shape that is not the chat-style trace you expected.

### 2. `config.patch` may require raw patching

Depending on OpenClaw version and tool surface, structured patching may be insufficient in some cases.
If that happens, use raw config patching carefully and validate the resulting config before restart.

### 3. Config hashes can race

If `config.patch` says config changed since last load:
- re-run `config.get`
- use the new `baseHash`
- retry

Do not guess.

### 4. Gateway restart may be deferred

OpenClaw can defer a restart if active work is still running.
So "restart requested" is not always "new code is already active".
Check actual post-reload behavior.

### 5. Auto-discovered local plugins are noisy until explicitly trusted

If you see plugin provenance warnings, add `plugins.allow`.

### 6. Tool spans can get large fast

If you capture `exec` output or large tool results, Langfuse noise explodes quickly.
Use truncation and field omission.

### 7. Hidden reasoning can leak through structured assistant objects

Do **not** dump raw assistant objects like `lastAssistant` unless you are certain the object is already sanitized.
In this integration, that field was removed from Langfuse payloads.

## Recommended clean configuration

If you want a sane default, use:

### Stock diagnostics
- `diagnostics.enabled = true`
- `diagnostics.otel.enabled = true`
- `traces = true`
- `metrics = false`
- `logs = false`

### Content plugin
- `includeHistory = false`
- `includeSystemPrompt = false`
- `captureTools = true`
- `maxFieldChars = 12000`

This gives:
- technical observability
- conversational traces
- less garbage
- less risk of leaking internal context

## Fast recovery checklist for another bot

If you need to reproduce this quickly, do this in order:

1. Inspect schema for `diagnostics.otel` and `plugins`.
2. Enable `diagnostics-otel` with Langfuse OTLP endpoint.
3. If the plugin fails with missing OTel modules, install the missing deps in the local `diagnostics-otel` extension directory:
   ```bash
   cd <openclaw-install>/extensions/diagnostics-otel
   npm install --omit=dev
   ```
4. Restart OpenClaw.
5. Verify `openclaw plugins list` and `openclaw logs`.
6. Do **not** stop here if you need chat content.
7. Add a workspace plugin with hooks: `llm_input`, `llm_output`, `before_tool_call`, `after_tool_call`, `agent_end`.
8. Enable it under `plugins.entries.langfuse-content.config`.
9. Set:
   - `includeHistory = false`
   - `includeSystemPrompt = false`
10. Ensure output payload does **not** include raw assistant objects that may contain hidden reasoning.
11. Verify via Langfuse API:
   - traces named `openclaw.chat`
   - `llm.run` generations with input/output
   - tool spans like `tool.exec`
12. Only then declare success.

## Safe snippets to reuse

### Basic Auth header

```text
Authorization: Basic base64(<publicKey>:<secretKey>)
```

### Langfuse endpoints used here

- OTLP technical telemetry:
  - `https://cloud.langfuse.com/api/public/otel`
- Ingestion API for content events:
  - `https://cloud.langfuse.com/api/public/ingestion`
- Read API for verification:
  - `https://cloud.langfuse.com/api/public/traces`
  - `https://cloud.langfuse.com/api/public/observations`

## What I would improve next if this became a permanent integration

- move `langfuse-content` from workspace hack into a proper maintained plugin
- add explicit field-level redaction policy for tool outputs
- add tests for:
  - no system prompt leakage
  - no history leakage
  - no hidden reasoning leakage
  - bounded tool payload sizes
- optionally add per-tool allow/deny capture lists
- optionally attach stable external ids for better grouping

## Bottom line

If you only want Langfuse lights blinking, stock `diagnostics-otel` is enough.

If you want **useful chat traces**, you need an extra content-level integration.

The main traps were:
- assuming empty `traces` meant nothing was delivered
- trusting config without verifying plugin load
- leaking system prompts/history/assistant internals into Langfuse
- declaring success before checking Langfuse API directly

Do the validation from the API side and keep payloads narrow.
That is the difference between "kind of wired" and "actually usable".
