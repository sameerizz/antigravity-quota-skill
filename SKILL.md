---
name: "Antigravity IDE Quota Architecture"
description: "Mental models, API secrets, and troubleshooting steps for building quota monitoring extensions for the Antigravity IDE / Codeium ecosystem."
---

# Antigravity IDE Quota Architecture & Mental Models

This skill document was generated to prevent future AI agents and human developers from repeating a lengthy debugging cycle when working with the Antigravity IDE / Codeium quota systems.

## 1. The Core Misunderstanding: The "Frankenstein" Architecture
Antigravity IDE's "Model Quota" UI is a composite of two completely disjointed backends:
1. **Google Cloud Code API (`cloudcode-pa.googleapis.com`)**: Tracks immediate compute bandwidth. It enforces and returns the **5-Hour Limit**.
2. **Codeium Language Server / Backend**: Tracks the user's paid subscription tier. It enforces and returns the **Weekly Limit**.

**Crucial Mental Model:** The Google API physically *does not know* your Weekly Limit. It only tracks the 5-Hour limit. The Antigravity UI silently merges these two data sources. Future agents must never assume that a single API endpoint holds both limits just because they appear side-by-side in the UI.

## 2. API Secrets: `fetchAvailableModels`
When querying `cloudcode-pa.googleapis.com/v1internal:fetchAvailableModels`:
- **The `remainingFraction` is the Literal UI Text:** If the API returns `0.26`, the IDE UI will literally display `26%`. Do NOT subtract this from 100 under the assumption that it is a "consumed" fraction. It represents the *remaining* quota. Multiply by 100 and round it.
- **The Duplicate Limit Illusion:** The API will return the exact same `remainingFraction` (e.g., `0.26`) for every single model tier (both `pro` and `flash`). It is impossible to extract the Weekly Limit from this endpoint. If building a UI, you must display `N/A` for the Weekly limit to avoid mirroring the 5-Hour limit.
- **The "Glitching 100%" Load-Balancer Bug:** Google's backend has a caching bug. Roughly 50% of requests will hit a stale server and erroneously return `"remainingFraction": 1` (100% remaining). 
  - **The Fix:** You MUST implement an invisible retry loop. If the parsed percentage evaluates to exactly 100%, sleep for 1 second and retry the request up to 3 times before updating the user interface.

## 3. Retrospective: What went wrong?
### What the AI Lacked:
- **Blind Trust in UI Assumptions:** The AI looked at a circular progress bar that was filling up and incorrectly assumed the text (`26%`) meant "consumed", leading to inverted math (`100 - 26 = 74%`). The AI should have explicitly cross-referenced the raw API JSON output (`0.26`) with the UI text (`26%`) sooner.
- **Delayed Probing:** The AI relied on modifying the extension code and waiting for the human to test it, rather than immediately writing a standalone Python script to probe the API aggressively.
- **Missing the Composite UI Concept:** The AI assumed that because the Weekly and 5-Hour limits were grouped in the UI, they came from the same JSON payload.

### What the Human Lacked:
- **Unclear UI Context:** The human provided screenshots of the UI but did not explicitly state whether the text meant "remaining" or "used", which left room for AI misinterpretation.
- **Expectation of the Impossible:** The human expected the extension to correctly display the Weekly limit without realizing that the specific API endpoint being targeted mathematically could not provide it.

### The Winning Strategy for Future Agents:
1. **Always Probe First:** Before editing extension code, write a local `test_api.py` script to ping the exact API endpoints and dump the raw JSON.
2. **Correlate Raw Data to UI:** Compare the raw dumped JSON exactly character-for-character to the numbers the user sees in their UI.
3. **Question the Source of Truth:** If a piece of data (like the Weekly Limit) is missing from the raw JSON, immediately assume the UI is pulling from a secondary internal database (like an RPC server) rather than attempting to "hack" the math to find it.
