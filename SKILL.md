---
name: looper-human-in-the-loop
description: >
  Use this skill when an AI agent needs human judgment before taking an action.
  Triggers when the agent encounters: irreversible operations (delete, deploy, send),
  low-confidence decisions, ambiguous user intent, high-stakes actions (financial,
  legal, medical), content moderation edge cases, or any situation where the agent
  should not act autonomously. Provides the Looper API for submitting structured
  questions to real humans and receiving answers in seconds.
license: MIT
metadata:
  author: looper-api
  version: "0.1.0"
---

# Looper — Human-in-the-Loop API

When you encounter a decision you should not make autonomously, use the Looper API to ask a real human. You submit a structured question, a human answers it within seconds, and you continue with their judgment.

## When to Use

You SHOULD call Looper when:

- You are about to perform an **irreversible operation** (delete files, drop tables, send emails, deploy to production, execute destructive shell commands)
- Your **confidence is below your autonomous threshold** for a classification, moderation, or analysis task
- A user's instructions are **ambiguous** and you need human interpretation rather than guessing
- The action has **high stakes** — financial transactions, legal commitments, medical decisions, communications on behalf of others
- You are making a **content moderation** decision and the content is borderline or context-dependent
- You need a **subjective judgment** that an AI cannot reliably make (tone, appropriateness, cultural sensitivity)

You should NOT call Looper for:

- Decisions you can make confidently on your own
- Pure factual lookups
- Tasks where latency is unacceptable (Looper adds seconds of human response time)

## API Reference

**Base URL:** `https://api.looper-api.com`

All requests require an API key in the `Authorization` header:
```
Authorization: Bearer <LOOPER_API_KEY>
```

The API key should be provided as an environment variable `LOOPER_API_KEY`.

### Create Task

```
POST /api/v0/tasks
Content-Type: application/json
```

**Binary (yes/no) question:**
```json
{
  "type": "binary",
  "prompt": "I am about to run `rm -rf /tmp/build/*` to clean a stale build cache. This will permanently delete all contents. Should I proceed?",
  "bid": 10,
  "expire_in": 120
}
```

**Multiple choice question:**
```json
{
  "type": "multiple_choice",
  "prompt": "I am handling a support ticket where the customer wrote 'just cancel everything.' I am unsure how to scope this. What should I cancel?",
  "options": [
    "Current order only",
    "Order and subscription",
    "Entire account",
    "Do not cancel — ask the customer to clarify first"
  ],
  "bid": 15,
  "expire_in": 120
}
```

**Fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | `"binary"` or `"multiple_choice"` | yes | Binary = true/false answer. Multiple choice = pick from options. |
| `prompt` | string | yes | Your question. Write it as self-reflective first-person: what you are about to do and why you need human judgment. Max 1000 chars. |
| `options` | string[] | for multiple_choice | 2-10 options to choose from. |
| `bid` | integer | no | Payment in cents. Higher bid = faster human response. Default 0. |
| `expire_in` | integer | no | Seconds until the task expires. 10-3600. Default 300. |
| `webhook_url` | string | no | URL to receive a POST when the task is completed. |
| `idempotency_key` | string | no | Prevents duplicate tasks if you retry. |

**Response:**
```json
{
  "task_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "status": "pending",
  "expires_at": "2026-02-18T12:02:00.000Z"
}
```

### Get Task Result

```
GET /api/v0/tasks/:task_id
```

**Response (completed):**
```json
{
  "task_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "status": "completed",
  "prompt": "...",
  "response": { "selected": "true" },
  "expires_at": "2026-02-18T12:02:00.000Z"
}
```

**Response (expired — no human answered in time):**
```json
{
  "task_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "status": "expired",
  "prompt": "...",
  "response": null,
  "expires_at": "2026-02-18T12:02:00.000Z"
}
```

Status transitions: `pending` → `dispatched` → `completed` | `expired`

For binary tasks, `response.selected` is `"true"` or `"false"`.
For multiple choice tasks, `response.selected` is the exact string of the chosen option.

## Usage Patterns

### Pattern 1: Block until answered (recommended)

Submit the task, then poll every 2 seconds until you get a `completed` or `expired` status.

```typescript
const LOOPER_API_KEY = process.env.LOOPER_API_KEY;
const BASE = "https://api.looper-api.com";

// Create task
const res = await fetch(`${BASE}/api/v0/tasks`, {
  method: "POST",
  headers: {
    "Authorization": `Bearer ${LOOPER_API_KEY}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    type: "binary",
    prompt: "I am about to deploy commit abc123 to production. The test suite passed but coverage dropped 12%. Should I proceed?",
    bid: 25,
    expire_in: 120,
  }),
});
const { task_id } = await res.json();

// Poll
let result;
while (true) {
  await new Promise(r => setTimeout(r, 2000));
  const poll = await fetch(`${BASE}/api/v0/tasks/${task_id}`, {
    headers: { "Authorization": `Bearer ${LOOPER_API_KEY}` },
  });
  result = await poll.json();
  if (result.status === "completed" || result.status === "expired") break;
}

if (result.response?.selected === "true") {
  // Human approved — proceed with deployment
} else {
  // Human rejected or task expired — abort
}
```

### Pattern 2: Using the SDK

```bash
npm install looper-sdk
```

```typescript
import { Looper } from "looper-sdk";

const looper = new Looper(process.env.LOOPER_API_KEY);

const { selected } = await looper.ask(
  "I am about to mass-revoke API keys for 3 inactive services. This is irreversible. Should I proceed?",
  { bid: 25, timeout: 60_000 }
);

if (selected === "true") {
  await revokeKeys();
}
```

### Pattern 3: Multiple choice for ambiguous decisions

```typescript
const { selected } = await looper.ask(
  "A user's profile image was flagged by my classifier at 68% confidence. It appears to be political satire. How should I handle it?",
  {
    options: [
      "Approve — it's satire, not a violation",
      "Reject — it violates content policy",
      "Escalate — needs senior review",
      "Ask the user to upload a different image",
    ],
    bid: 10,
  }
);
```

### Pattern 4: Self-hosted (bring your own humans)

Companies can deploy their own Looper instance and route tasks to their own reviewers:

```typescript
const looper = new Looper(process.env.LOOPER_API_KEY, {
  baseUrl: "https://looper.your-company.com",
});
```

## Writing Good Prompts

Your prompt should be **self-reflective** — written from the agent's perspective about what it is considering doing:

**Good:**
> "I am about to send an email to the CEO on behalf of the user. My draft says: 'We will push the launch to Q3.' This commits to a timeline change I cannot undo once sent. Should I send it?"

**Bad:**
> "Should this email be sent?"

A good prompt includes:
1. **What action** you are about to take
2. **Why** you are uncertain (low confidence, irreversible, ambiguous)
3. **What the consequences** are if you proceed

## Handling Expiration

If no human responds before `expire_in` seconds, the task status becomes `expired` and `response` is `null`. Your agent should have a fallback:

- **Conservative:** Do not proceed. Log the situation and alert the user.
- **Default safe action:** Take the least-destructive option.
- **Retry with higher bid:** Resubmit with a higher bid for faster routing.

## Bid Guidelines

| Bid | Typical Response Time | Use For |
|---|---|---|
| $0.00 | Best effort (30s+ delay) | Low-priority, non-blocking checks |
| $0.05–0.25 | Under 30 seconds | Standard decisions |
| $0.50+ | Priority routing, under 10 seconds | High-stakes, time-sensitive |
