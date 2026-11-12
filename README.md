# Agent Control Room

A phone-first control room for a team of AI agents. I built it to see, from my phone, every agent session, every approval waiting on me, and what is stuck. It is a personal tool, currently in trial run. It is not a product and has no customers.

> Built by Zahi Michaeli, with a team of Claude agents.

## Why

I work with a team of agents that have defined roles (architect, DevOps and QA, privacy, UX, code reviewer, and a coordinator that routes between them). Once there are many sessions, the problem stops being "can the agent do it" and becomes "what is running, what is waiting for me, and what went quiet". A chat window does not answer that. A board does.

## How it works

```
agent session --(hook: small event)--> Worker + D1 --(read)--> board on my phone
                                          |
                          behind Cloudflare Access (one user, Google login)
```

- Hooks in the agent sessions send small events: a thread started, a thread needs review, an approval is requested.
- A Cloudflare Worker with a D1 database stores the events and computes state. Status comes from deterministic events, not from running a model.
- The board shows what is working now, what waits for approval, what is stuck or silent, and a short history.
- The hooks fail safe: if the board is down, the agents keep working.

### An event

Only a title and a state travel. No message bodies, no secrets.

```json
{
  "v": 1,
  "id": "evt_2026_11_09_0042",
  "ts": "2026-11-09T10:15:00Z",
  "source": "hook",
  "kind": "approval.requested",
  "project": "example-project",
  "thread": { "id": "thread-123", "title": "Prepare release notes" },
  "summary": "Publish the release notes?",
  "data": { "approvalId": "a-17", "reversible": false }
}
```

## Human in the loop, by design

- An approval button never executes anything. It only prepares the approval text for me to send. Steps that cannot be undone (production, store, deletion, money, outbound messages) stay with me.
- Writes to production are refused in code. Writes to the test environment need an explicit confirmation:

```ts
export function guardWrite(env: "test" | "production", action: string, confirm: string | null) {
  if (env === "production") return { ok: false, reason: "writes are disabled in production" };
  if (confirm !== action) return { ok: false, reason: "explicit confirmation required" };
  return { ok: true };
}
```

## Security

- The dashboard sits behind Cloudflare Access with a single allowed user.
- The event intake is the only path outside Access. It accepts POST only, requires a bearer token or an HMAC-signed body, and fails closed when no secret is configured.
- Events are size-limited, de-duplicated by id, and pruned after 30 days.

## Quality

Unit tests (Vitest), type checking, a deploy guard that fails before deploying if the routing is wrong, and visual QA of the screens on phone and desktop.

## Lessons

1. **Root cause, not patch.** The board once stopped updating for a few hours. Restarting would have hidden it. The cause: leading spaces made a hook skip a list in silence. The fix went into the code and the lesson into the team's knowledge base as a rule.
2. **Structure before sketch.** Sketches drawn before the navigation and information hierarchy were defined caused rework. I added a UX and systems-analyst role to the team.
3. **State that looks lost is often a data-source bug.** A project picker seemed to forget my choice. The choice was saved; the list was built from filtered data. Fixed at the source.
4. **Human approval as design, not slogan.**

## Status

In trial run. Private deployment, no external users. Some items (a template for others, a clean-account install test) are not done yet.

## Contact

Zahi Michaeli: [LinkedIn](https://www.linkedin.com/in/zahi-michaeli)
