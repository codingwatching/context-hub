---
name: deep-research
description: "Gemini Deep Research API via the Interactions endpoint for autonomous multi-step research with structured reports"
metadata:
  languages: "javascript"
  versions: "1.0.0"
  revision: 1
  updated-on: "2026-03-09"
  source: community
  tags: "gemini,google,deep-research,research,interactions,agent"
---

# Gemini Deep Research API (JavaScript/TypeScript)

The Deep Research API lets you run Google's autonomous research agent
programmatically. It performs multi-step web research and returns structured
markdown reports. This uses the **Interactions API** — a separate endpoint from
the standard `generateContent` API.

## Official References

- [Interactions API Guide](https://ai.google.dev/gemini-api/docs/interactions)
- [Deep Research Agent Guide](https://ai.google.dev/gemini-api/docs/deep-research)

## Key Concepts

- **Interactions API**: REST endpoint at `https://generativelanguage.googleapis.com/v1beta/interactions`
- **Asynchronous**: You create an interaction, then poll for completion (typically 5–15 minutes)
- **Agent model**: `deep-research-pro-preview-12-2025` (the only available agent as of March 2026)
- **Authentication**: Uses `x-goog-api-key` header (same `GEMINI_API_KEY`)
- **Output**: Markdown report with citations and structured sections
- **Store required**: `store: true` is required when using `background: true`

## Authentication

Use the same Gemini API key. Set `GEMINI_API_KEY` environment variable or pass directly.

## Step 1: Create an Interaction

```typescript
const API_KEY = process.env.GEMINI_API_KEY;
const BASE_URL = "https://generativelanguage.googleapis.com/v1beta/interactions";
const AGENT = "deep-research-pro-preview-12-2025";

async function createInteraction(query: string): Promise<{ name: string; id: string; status: string }> {
  const response = await fetch(BASE_URL, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "x-goog-api-key": API_KEY!,
    },
    body: JSON.stringify({
      input: query,
      agent: AGENT,
      background: true,
      store: true,
    }),
  });

  if (!response.ok) {
    throw new Error(`Create failed: ${response.status} ${await response.text()}`);
  }

  return response.json();
}

const interaction = await createInteraction("Research the current state of AI in disaster relief");
console.log(`Started: ${interaction.name}`);
// Output: Started: interactions/abc123...
```

### Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `input` | string | Yes | The research query or topic |
| `agent` | string | Yes | Must be `deep-research-pro-preview-12-2025` |
| `background` | boolean | Yes | Set to `true` for async execution |
| `store` | boolean | Yes | Must be `true` when `background` is `true` |

## Step 2: Poll for Completion

Research takes 5–15 minutes. Poll every 10 seconds.

```typescript
interface InteractionResponse {
  name?: string;
  id?: string;
  status?: string; // "in_progress" | "completed" | "failed"
  error?: any;
  outputs?: Array<{ text: string }>;
}

async function pollInteraction(interactionName: string): Promise<string> {
  // Extract ID from resource name (e.g., "interactions/abc123" -> "abc123")
  const id = interactionName.includes("/")
    ? interactionName.split("/").pop()
    : interactionName;

  const url = `${BASE_URL}/${id}`;
  const TIMEOUT_MS = 60 * 60 * 1000; // 1 hour max
  const startTime = Date.now();

  while (true) {
    if (Date.now() - startTime > TIMEOUT_MS) {
      throw new Error("Research timed out after 60 minutes");
    }

    await new Promise((r) => setTimeout(r, 10_000)); // 10s interval

    const response = await fetch(url, {
      headers: { "x-goog-api-key": API_KEY! },
    });

    if (!response.ok) {
      // Retry on 5xx, abort on 4xx (except 429)
      if (response.status >= 400 && response.status < 500 && response.status !== 429) {
        throw new Error(`Fatal error: ${response.status}`);
      }
      continue; // Retry on transient errors
    }

    const data: InteractionResponse = await response.json();

    if (data.status === "completed" || data.status === "COMPLETED") {
      const lastOutput = data.outputs?.[data.outputs.length - 1];
      return lastOutput?.text ?? "No content in response";
    }

    if (data.status === "failed" || data.status === "FAILED") {
      throw new Error(`Research failed: ${JSON.stringify(data.error)}`);
    }

    // Still in_progress — continue polling
  }
}

const report = await pollInteraction(interaction.name);
```

### Response Shape

```json
{
  "name": "interactions/abc123",
  "id": "abc123",
  "status": "completed",
  "outputs": [
    {
      "text": "# Research Report\n\n## Executive Summary\n..."
    }
  ]
}
```

### Status Values

| Status | Meaning |
|--------|---------|
| `in_progress` | Agent is still researching |
| `completed` | Report is ready in `outputs[].text` |
| `failed` | Research failed — check `error` field |

## Step 3: Save the Report

```typescript
import fs from "fs/promises";

const slug = "ai-disaster-relief";
const filename = `report-${slug}-${Date.now()}.md`;
await fs.writeFile(filename, report);
console.log(`Saved: ${filename}`);
```

## Complete Example

```typescript
import fs from "fs/promises";

const API_KEY = process.env.GEMINI_API_KEY;
const BASE_URL = "https://generativelanguage.googleapis.com/v1beta/interactions";
const AGENT = "deep-research-pro-preview-12-2025";

async function deepResearch(query: string): Promise<string> {
  // Create
  const createRes = await fetch(BASE_URL, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "x-goog-api-key": API_KEY!,
    },
    body: JSON.stringify({ input: query, agent: AGENT, background: true, store: true }),
  });

  if (!createRes.ok) throw new Error(`Create failed: ${createRes.status}`);
  const { name } = await createRes.json();
  const id = name.split("/").pop();
  console.log(`Research started: ${id}`);

  // Poll
  while (true) {
    await new Promise((r) => setTimeout(r, 10_000));

    const pollRes = await fetch(`${BASE_URL}/${id}`, {
      headers: { "x-goog-api-key": API_KEY! },
    });

    if (!pollRes.ok) continue;
    const data = await pollRes.json();

    if (data.status === "completed" || data.status === "COMPLETED") {
      return data.outputs[data.outputs.length - 1].text;
    }
    if (data.status === "failed" || data.status === "FAILED") {
      throw new Error(`Failed: ${JSON.stringify(data.error)}`);
    }
  }
}

// Usage
const report = await deepResearch("Current state of AI in disaster relief operations");
await fs.writeFile("report.md", report);
```

## Tips and Gotchas

- **Long-running**: Expect 5–15 minutes per research task. Budget for this in your application.
- **No streaming**: Unlike `generateContent`, there is no streaming variant. You must poll.
- **Rate limits**: The Interactions API has separate rate limits from `generateContent`. Expect lower throughput.
- **Report quality**: The agent performs real web searches. More specific queries produce better reports.
- **Plan first**: For best results, generate a research plan with `generateContent` (using `gemini-2.5-pro`), then pass the refined plan to Deep Research.
- **Agent name**: As of March 2026, the only available agent is `deep-research-pro-preview-12-2025`. No 3.x-based agent exists yet.
- **Background must be true**: The `background: true` parameter is required. Synchronous mode is not supported for this agent.
- **Output is markdown**: Reports come back as structured markdown with headers, bullet points, and citations.
