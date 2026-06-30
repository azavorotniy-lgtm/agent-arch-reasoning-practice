# Requirements — Real-Time Collaborative Article Editing

> **Source:** Requirements interview with the Conduit engineering team lead.
> **Feature owner / interviewee:** **Alex Chen** — Engineering Team Lead, Conduit.
> **Interviewer:** Architecture Reasoning Agent (architect).
> **Status:** Draft for architecture review — no implementation decisions made yet.

---

## 1. Who I am talking to

The team lead being interviewed is **Alex Chen**, engineering team lead at **Conduit**, a lean
indie publishing platform (a Medium-style clone). Alex owns the delivery of the requested feature
and speaks from a product/delivery perspective — pragmatic, deadline-aware, and writer-empathetic.
Alex defers distributed-systems implementation choices (CRDTs, OT, Redis pub/sub, orchestration)
to the architect.

**Test — "Who are you?"** → *"I'm Alex Chen, the engineering team lead here at Conduit. I'm the one
pushing for real-time collaborative editing for our writers. Happy to walk you through what we need —
but fair warning, I'll keep asking 'what's the simplest version of this?'"*

---

## 2. Feature summary

Allow two or more writers to co-edit a single article **simultaneously**, with **live cursors**,
**live keystroke-level changes**, and a **presence indicator** showing who is currently editing —
comparable to Google Docs. This is the **#1 requested feature** from Conduit power users, who today
have to pass a Google Doc back and forth and paste the final version back into Conduit.

---

## 3. Goals

- A writer can **invite a co-author** to an article and edit it together in real time.
- Both authors see **each other's cursor position** and **changes as they type**.
- A **coloured presence indicator** (avatar) shows who is currently in the document.
- **No lost work.** Concurrent edits must never silently drop a writer's content.
- Ship a usable **MVP by Q4** with a 4-person team and existing budget headroom.

## 4. Non-goals (MVP)

- No more than **5 simultaneous editors** per article.
- No real-time co-editing of comments, profiles, or any entity other than article body.
- No offline editing / sync-on-reconnect guarantees beyond "don't lose committed text".
- No voice/video, no inline chat, no commenting threads inside the editor.
- No migration off the single VPS *unless* the architect proves it is unavoidable for Q4.

---

## 5. Target scale & capacity

| Dimension | Target / assumption |
|---|---|
| Registered users | ~14,000 (growing ~18% MoM) |
| Articles published / month | ~2,600 (growing ~22% MoM) |
| Max simultaneous editors per article (MVP) | **5** |
| Expected concurrent *collaborative sessions* at launch | low tens (power-user feature, opt-in) |
| Expected concurrent *open WebSocket connections* | plan for low hundreds at MVP, headroom to ~1–2k |
| Document size | typical long-form article (a few thousand words) |
| Latency target for remote edits to appear | sub-second ("feels live"), p95 < ~500 ms |
| Growth horizon to design for | 12 months at current MoM growth before re-architecting |

**Alex's framing:** *"This is an opt-in power-user feature, not something all 14k users hit at once.
Don't size it for Google's traffic. But it should not fall over if a dozen articles are being
co-edited during a weekday morning."*

---

## 6. Availability & reliability requirements

- **Data loss is a hard blocker.** If two people type at the same time, the system must converge
  to a consistent document with **no silently dropped edits**. "Whose version wins" must be a
  defined, predictable rule — not luck.
- **Graceful degradation:** if the real-time layer is down, the editor must **fall back to
  standard single-user editing with save** rather than showing an error or a blank page. Writers
  must never be blocked from writing.
- **Reconnect handling:** a brief network drop or server restart must not lose committed text.
  On reconnect, the writer's in-progress content should be recoverable.
- **Auto-save / persistence:** edits must be persisted durably (this also addresses the existing
  "no draft auto-save" pain point — ~30 lost-work support tickets/month).
- **Single-VPS constraint (known risk):** Conduit runs on **one VPS** with **no Redis, no queue,
  no CDN, no cache layer**. Any design that requires multiple app processes for WebSockets must
  explain how presence/state is shared across processes, or justify staying single-process for MVP.
- **Availability target:** best-effort during business hours; this is not yet a 99.9% SLA feature,
  but it must not jeopardize the availability of the rest of the monolith.

**Alex's framing:** *"Our writers care most about not losing work. If the live part breaks, I'd
rather it quietly drop back to normal editing than ever lose a paragraph someone just wrote."*

---

## 7. Pain points this addresses (and constraints from them)

From `data/context/company.md` and the interview:

1. **No real-time collaboration** — co-authors pass a Google Doc back and forth (the #1 missing
   feature). This feature directly resolves it.
2. **No draft auto-save** — closing a tab loses the draft (~30 tickets/month). Persistence work
   for this feature should also deliver durable auto-save.
3. **Single VPS, no Redis/queue/CDN** — hard infra constraint; the feature must fit the existing
   topology or incrementally extend it within budget.
4. **Auth sessions never expire (JWT, no refresh tokens)** — relevant because the WebSocket
   connection must be authenticated; see security below.
5. **Team capacity** — 2 backend devs, 1 frontend dev, 1 part-time DevOps; **no prior real-time
   collab experience**, only basic WebSocket/chat-tutorial familiarity. Operational simplicity is
   a first-class requirement, not a nice-to-have.

---

## 8. Security & compliance requirements

- **Authorization:** only the article's **author and explicitly invited co-authors** may join a
  collaborative session. Joining must be checked server-side on every connection, not just in the UI.
- **WebSocket authentication:** the live connection must be authenticated with the user's identity.
  **Known issue:** JWTs currently **never expire** and there are no refresh tokens — a leaked token
  is valid forever. The architect must address how the real-time channel validates/expires sessions
  so a stale or leaked token cannot silently join a private draft. (JWT expiry hardening is already
  flagged by the security team.)
- **Authorization revocation:** if a co-author's access is removed, their live session must be
  terminated promptly.
- **Transport security:** all real-time traffic over TLS (wss://), same-origin / CORS-controlled.
- **Tenant/data isolation:** edits for one article must never leak into another session.
- **Audit / attribution:** it should be possible to attribute changes to a specific author
  (supports trust and dispute resolution between co-authors).
- **GDPR / data handling:** no new categories of personal data introduced beyond existing user
  identity; presence data (who is editing) is ephemeral and should not be retained longer than the
  session needs.
- **Abuse / limits:** enforce the 5-editor cap and reasonable per-connection rate limits to protect
  the single VPS.

**Alex's framing:** *"Security flagged that our tokens never expire. I don't want to bolt a live
editing channel onto an auth system that can't kick out a leaked login. Tell me what we need to fix
there before this ships."*

---

## 9. Team, budget & timeline constraints

| Constraint | Value |
|---|---|
| Team | 2 backend (Node/Express/Sequelize/PostgreSQL), 1 frontend (React/Vite), 1 part-time DevOps |
| Relevant experience | Basic WebSockets only; **no** CRDT/OT/distributed-systems experience |
| Additional infra budget | up to **€1,200/month** (current spend ~€280/month) |
| Target | **Q4** MVP (premium tier ships Q3 first — this is the next priority) |
| Maintainability bar | The team must be able to operate and debug it; no service "nobody can maintain at 2am" |

---

## 10. Open questions for the architect

These are the questions Alex expects the architecture review to answer (in plain language, with
tradeoffs for *writers* and for *operational complexity*):

1. How do we resolve concurrent edits without losing work — and what does that approach
   (e.g. CRDT vs. operational transformation) mean for our deploy complexity and for our team?
2. Can we ship the MVP on the **single VPS / single process**, or do we genuinely need Redis or a
   second service for WebSocket fan-out? If we need it, what does it cost to run and maintain?
3. What is the simplest persistence model that guarantees "no lost paragraphs" and also gives us
   the **auto-save** win for free?
4. What must we fix in **JWT/session handling** before a live editing channel is safe to expose?
5. What is the **fallback** path when the real-time layer is unavailable, and how invisible is it
   to writers?
6. What is the realistic **scope and timeline** for a 4-person team to hit Q4 — and what can be
   cut from the MVP without breaking the core promise?

---

## Appendix A — Interview transcript (condensed)

**Architect:** Who are you, and what do you want to build?
**Alex:** I'm Alex Chen, engineering team lead at Conduit. I want real-time collaborative article
editing — two or more writers co-editing the same article, live cursors, presence indicators, like
Google Docs. It's our #1 power-user request; today they pass a Google Doc around and paste it back.

**Architect:** What scale should this support?
**Alex:** It's opt-in for power users, not all 14k users at once. MVP caps at 5 simultaneous editors
per article. Plan for low hundreds of live connections at launch with room to grow as we keep
adding ~18% users a month. Edits should feel instant — under half a second.

**Architect:** What are your availability and reliability expectations?
**Alex:** Data loss is a hard no. If two people type at once, the result has to be consistent and
predictable — no dropped text. If the live layer breaks, drop back to normal editing with save;
never show writers a blank page. And if someone's connection blips, they can't lose committed work.

**Architect:** What constraints are we working within?
**Alex:** One VPS. No Redis, no queue, no CDN. Four people — two backend, one frontend, one
part-time DevOps — and none of us have built collaborative editing before. Budget headroom is about
€1,200/month, and the target is Q4 after the Q3 premium launch. If it needs a whole new service
nobody can maintain at 2am, that's a problem.

**Architect:** Any security or compliance concerns?
**Alex:** Only the author and invited co-authors should be able to join — checked on the server.
The live connection has to be authenticated, but our JWTs never expire and we have no refresh
tokens; security already flagged it. Tell me what we have to fix there before this is safe to ship.
Presence data is ephemeral; we don't want to retain anything we don't need.

**Architect:** What would make this a success for you?
**Alex:** The simplest version that lets two writers co-author without losing work, ships by Q4,
and doesn't bury my team in infrastructure they can't run. Show me 2–3 options with the tradeoffs
in plain terms and I'll pick one.
