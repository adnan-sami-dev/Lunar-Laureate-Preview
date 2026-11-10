# 🌙 Lunar Laureate

<img width="1397" height="958" alt="image" src="https://github.com/user-attachments/assets/6e4e2c34-905d-4a07-997f-707089280491" />


> A real-time video classroom built for tutors and students: scheduled meetings, participant management, role-aware controls, and a join window that respects the calendar.

**[🚀 Live App](https://lunar-laureatte.vercel.app)** · **[📦 Repository](https://github.com/adnan-sami-dev/Lunar-Laureate-A-Video-Conferencing-App-for-Tutors)**

## About this repository

This is a **source-available portfolio excerpt**. The full implementation
lives in a private repository to keep the IP clean.

If you'd like to see the complete codebase, I'm happy to:
- Walk through any part of it live during an interview, or
- Grant read access to the private repo after an initial conversation.

Just reach out: [sami.adnandev@gmail.com] · [https://www.linkedin.com/in/adnan-sami-3ab549220/]

**Stack:** Next.js · TypeScript · Stream Video SDK · Clerk · Tailwind CSS · PostgreSQL

---

## Table of Contents

- [What It Does](#what-it-does)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Insights & Design Decisions](#insights--design-decisions)
- [Bug Fixes & Debugging Notes](#bug-fixes--debugging-notes)
- [What I Would Do Differently](#what-i-would-do-differently)

---

## What It Does

Lunar Laureate lets tutors schedule video classes, invite students as members, and control when the room opens. Students see upcoming meetings on their dashboard and can join within a **10-minute window** before the scheduled start. The app handles call creation, live participant state, and soft-delete for clearing past meetings from a user's list.

The interesting problems were not the video calls themselves. They were the edges: **time gating, event/state races, and working within the SDK's query limitations.**

---

## Architecture

```text
Client (Next.js App Router)
  │
  ├──► Clerk                  (auth, route protection, user session)
  ├──► Stream Video SDK       (call lifecycle, real-time media)
  │       │
  │       ├── Call creation   (getOrCreate with starts_at)
  │       ├── Query calls     (member/creator visibility)
  │       └── Call state      (participants, endedAt, custom data)
```

**Key properties:**

- **Server-side auth:** the root layout calls `auth()` and redirects unauthenticated users before any protected page renders.
- **Scheduled calls use `starts_at`:** Stream stores this as metadata, and the client enforces the join window.
- **Soft-delete for lists:** Stream has no delete API, so hidden calls are flagged in `custom` and filtered client-side.
- **State over events:** the toast system reads `call.state.endedAt` instead of trusting which event fired first.

---

## Tech Stack

| Layer | Technology |
|-------|------------|
| Framework | Next.js (App Router) |
| Language | TypeScript |
| Auth | Clerk (middleware + server-side checks) |
| Video | Stream Video React SDK |
| Styling | Tailwind CSS |

---

## Insights & Design Decisions

### 1. Why the join window is client-side, not backstage mode

Stream offers **backstage mode** for scheduled calls. The host can prepare before viewers join, and `join_ahead_time_seconds` allows early entry. For a classroom, backstage is overkill. It requires a custom call type, permission grants (`JOIN_BACKSTAGE`), and adds a "go live" step the tutor must remember.

**The trade-off:** Client-side gating means a technically savvy user could bypass the time check by calling `call.join()` directly. For a self-dev app with trusted users, that is acceptable. For a production classroom, backstage would be the correct enforcement layer.

---

### 2. Why scheduled calls use `starts_at` and a 1-second interval

Stream's `queryCalls` supports filtering by `starts_at`, so the upcoming list only shows future meetings. But the "Join" button needs to enable when the time arrives, without a page refresh.

**The solution:** a `now` state that ticks every second. `isDue` recalculates on each render: `startsAt - now <= 10 minutes`. This is crude but effective. No server round-trip, no polling, no webhook.

**The trade-off:** One interval per meeting card. For a list of 20 meetings, that is 20 timers. If the list grows, lift the interval to the parent and pass `now` down as a prop.

---

### 3. Why soft-delete instead of actual deletion

Stream's `queryCalls` does not support the `$ne` operator on custom fields. Attempting the following returns an error:

```json
{ "custom.hidden_userId": { "$ne": true } }
```

The API accepts equality checks on custom fields, but not negation.

**The workaround:** mark hidden calls with `custom.hidden_${userId}: true`, fetch all calls, then filter client-side. This is not elegant, but it works within Stream's constraints.

**The trade-off:** Every fetch retrieves calls the user has hidden, then discards them. For a portfolio app with a handful of meetings, that is negligible. At scale, this would need a server-side solution or a different data model.

---

## Bug Fixes & Debugging Notes

### 1. The toast always said "Call left" even when the owner ended the call

- **Symptom:** When a tutor ended the call for everyone, participants saw "Call left" instead of "Call ended."
- **Cause:** The `onLeave` callback fired first, setting the message to "Call left." The `call.ended` WebSocket event arrived a moment later, but the toast state was already set and the component was unmounting.
- **Fix:** Stop trusting the event source. Both triggers call the same handler, which reads `call.state.endedAt`. If `endedAt` is set, the call was ended. If not, the user left. The state is the truth; the event is just the trigger.

> **Lesson:** In real-time apps, events tell you *when* something happened. State tells you *what* happened. When they race, trust the state.

---

### 2. `useActiveCall` returned `null` forever on the home page

- **Symptom:** The active call banner never appeared, even when a call was ongoing.
- **Cause:** Two issues compounded:
  1. `client.streamClient.user` was `undefined` on first render because Clerk and Stream attach at different times. The effect early-returned and never re-ran.
  2. The query filter used `members: { $in: [userId] }`, but calls created by the user were not automatically adding them as members.
- **Fix:** Use `useConnectedUser()` for a reactive user ID, add `userId` to the effect's dependency array, and filter with:

  ```ts
  $or: [{ created_by_user_id: userId }, { members: { $in: [userId] } }]
  ```

> **Lesson:** A guard that returns early without setting loading state creates an infinite spinner. And Stream's `members` is not the same as `created_by_user_id`.

---

### 3. Stream logged "Failed to get audio stream" during scheduling

- **Symptom:** Creating a scheduled call via `call.getOrCreate()` logged an audio device error.
- **Cause:** The SDK initializes device management internally during call operations. Since the user had not interacted with the meeting room yet, the browser blocked `getUserMedia`.
- **Fix:** None needed. The error is logged, not thrown. `getOrCreate` resolves successfully, and the call is created on the server.

> **Lesson:** Not every console error is your bug. Distinguish between SDK noise and actual failures by checking whether the promise rejects.

---

## What I Would Do Differently

1. **Use backstage mode for real classrooms.** Client-side time gating is fine for a portfolio, but a production classroom needs server-enforced entry windows.
2. **Move the time tick to a shared context.** One interval for all meeting cards, not one per card.
3. **Add integration tests for the toast race.** The `endedAt` fix is subtle. A test that simulates both event orders would prevent regression.
