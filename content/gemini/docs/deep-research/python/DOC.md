---
name: deep-research
description: "Gemini Deep Research API via the Interactions endpoint for autonomous multi-step research with structured reports"
metadata:
  languages: "python"
  versions: "1.0.0"
  revision: 1
  updated-on: "2026-03-09"
  source: community
  tags: "gemini,google,deep-research,research,interactions,agent"
---

# Gemini Deep Research API (Python)

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
- **Store required**: `store: True` is required when using `background: True`
- **No SDK wrapper**: As of March 2026, the `google-genai` SDK does not wrap the Interactions API. Use `requests` directly.

## Authentication

Use the same Gemini API key. Set `GEMINI_API_KEY` environment variable or pass directly.

```python
import os

API_KEY = os.environ["GEMINI_API_KEY"]
BASE_URL = "https://generativelanguage.googleapis.com/v1beta/interactions"
AGENT = "deep-research-pro-preview-12-2025"
HEADERS = {
    "Content-Type": "application/json",
    "x-goog-api-key": API_KEY,
}
```

## Step 1: Create an Interaction

```python
import requests

def create_interaction(query: str) -> dict:
    """Start a Deep Research interaction. Returns the interaction resource."""
    response = requests.post(
        BASE_URL,
        headers=HEADERS,
        json={
            "input": query,
            "agent": AGENT,
                        "background": True,
            "store": True,
        },
    )
    response.raise_for_status()
    return response.json()

interaction = create_interaction("Research the current state of AI in disaster relief")
print(f"Started: {interaction['name']}")
# Output: Started: interactions/abc123...
```

### Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `input` | string | Yes | The research query or topic |
| `agent` | string | Yes | Must be `deep-research-pro-preview-12-2025` |
| `background` | boolean | Yes | Set to `True` for async execution |
| `store` | boolean | Yes | Must be `True` when `background` is `True` |

## Step 2: Poll for Completion

Research takes 5–15 minutes. Poll every 10 seconds.

```python
import time

def poll_interaction(interaction_name: str, timeout_minutes: int = 60) -> str:
    """Poll until research completes. Returns the markdown report."""
    # Extract ID from resource name
    interaction_id = interaction_name.split("/")[-1] if "/" in interaction_name else interaction_name
    url = f"{BASE_URL}/{interaction_id}"

    deadline = time.time() + timeout_minutes * 60

    while time.time() < deadline:
        time.sleep(10)  # Poll every 10 seconds

        try:
            response = requests.get(url, headers=HEADERS)

            if response.status_code >= 400 and response.status_code < 500 and response.status_code != 429:
                raise RuntimeError(f"Fatal API error: {response.status_code} {response.text}")

            if not response.ok:
                continue  # Retry on transient errors

            data = response.json()
            status = data.get("status", "").lower()

            if status == "completed":
                outputs = data.get("outputs", [])
                if outputs:
                    return outputs[-1]["text"]
                return "No content in response"

            if status == "failed":
                raise RuntimeError(f"Research failed: {data.get('error')}")

        except requests.RequestException:
            continue  # Retry on network errors

    raise TimeoutError(f"Research timed out after {timeout_minutes} minutes")

report = poll_interaction(interaction["name"])
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

```python
from pathlib import Path

slug = "ai-disaster-relief"
filename = f"report-{slug}-{int(time.time())}.md"
Path(filename).write_text(report)
print(f"Saved: {filename}")
```

## Complete Example

```python
import os
import time
import requests
from pathlib import Path

API_KEY = os.environ["GEMINI_API_KEY"]
BASE_URL = "https://generativelanguage.googleapis.com/v1beta/interactions"
AGENT = "deep-research-pro-preview-12-2025"
HEADERS = {"Content-Type": "application/json", "x-goog-api-key": API_KEY}


def deep_research(query: str, timeout_minutes: int = 60) -> str:
    """Run Gemini Deep Research and return the markdown report."""
    # Create interaction
    resp = requests.post(
        BASE_URL,
        headers=HEADERS,
        json={"input": query, "agent": AGENT, "background": True, "store": True},
    )
    resp.raise_for_status()
    interaction_id = resp.json()["name"].split("/")[-1]
    print(f"Research started: {interaction_id}")

    # Poll for completion
    url = f"{BASE_URL}/{interaction_id}"
    deadline = time.time() + timeout_minutes * 60

    while time.time() < deadline:
        time.sleep(10)

        try:
            r = requests.get(url, headers=HEADERS)
            if not r.ok:
                continue
            data = r.json()
            status = data.get("status", "").lower()

            if status == "completed":
                return data["outputs"][-1]["text"]
            if status == "failed":
                raise RuntimeError(f"Failed: {data.get('error')}")
        except requests.RequestException:
            continue

    raise TimeoutError("Research timed out")


# Usage
report = deep_research("Current state of AI in disaster relief operations")
Path("report.md").write_text(report)
print("Report saved.")
```

## Optional: Generate a Research Plan First

For higher quality results, generate a structured research plan before
sending to Deep Research:

```python
from google import genai

client = genai.Client()  # uses GEMINI_API_KEY env var

plan = client.models.generate_content(
    model="gemini-2.5-pro",
    contents=f'Create a detailed research plan for: "{topic}". '
             "Break it into key areas and questions. Be concise but comprehensive.",
)

# Pass the refined plan to Deep Research
report = deep_research(f"Execute the following research plan:\n\n{plan.text}")
```

## Tips and Gotchas

- **Long-running**: Expect 5–15 minutes per research task. Budget for this in your application.
- **No streaming**: Unlike `generateContent`, there is no streaming variant. You must poll.
- **No SDK wrapper**: The `google-genai` Python SDK does not have a method for the Interactions API. Use `requests` directly against the REST endpoint.
- **Rate limits**: The Interactions API has separate rate limits from `generateContent`. Expect lower throughput.
- **Report quality**: The agent performs real web searches. More specific queries produce better reports.
- **Plan first**: For best results, generate a research plan with `generateContent` (using `gemini-2.5-pro`), then pass the refined plan to Deep Research.
- **Agent name**: As of March 2026, the only available agent is `deep-research-pro-preview-12-2025`. No 3.x-based agent exists yet.
- **Background must be True**: The `background: True` parameter is required. Synchronous mode is not supported for this agent.
- **Output is markdown**: Reports come back as structured markdown with headers, bullet points, and citations.
- **Dependencies**: Only `requests` is needed — no special Google SDK required for this endpoint.
