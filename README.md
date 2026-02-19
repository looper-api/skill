# Looper Agent Skill

An agent skill that teaches AI agents how to use the [Looper API](https://api.looper-api.com) — a human-in-the-loop service that lets agents request real human judgment on-demand.

## Install

```bash
npx skills add looper-api/skill
```

## What it does

When an AI agent encounters a decision it shouldn't make alone — irreversible operations, low-confidence calls, ambiguous intent, high-stakes actions — this skill gives it the Looper API to ask a real human and get an answer in seconds.

## Examples

An agent about to delete production files:
```
→ Looper: "I am about to run `rm -rf /tmp/build/*` to clean a stale build cache. Should I proceed?"
← Human: "false"
→ Agent aborts.
```

An agent unsure how to interpret a vague instruction:
```
→ Looper: "A customer wrote 'just cancel everything.' What should I cancel?"
  Options: ["Current order only", "Order + subscription", "Entire account", "Ask customer to clarify"]
← Human: "Ask customer to clarify"
→ Agent follows up with the customer.
```

## SDK

```bash
npm install looper-sdk
```

```typescript
import { Looper } from "looper-sdk";

const looper = new Looper(process.env.LOOPER_API_KEY);

const { selected } = await looper.ask(
  "I want to deploy commit abc123 to production. Tests passed but coverage dropped 12%. Should I proceed?",
  { bid: 25 }
);
```

## Links

- [API Documentation](https://api.looper-api.com)
- [SDK on npm](https://www.npmjs.com/package/looper-sdk)
- [Looper on GitHub](https://github.com/looper-api)
