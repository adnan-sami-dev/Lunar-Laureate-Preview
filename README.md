# 🌙 Lunar Laureate

<img width="1387" height="957" alt="image" src="https://github.com/user-attachments/assets/5574cfae-c09d-49e1-afca-ba3e822cd3dc" />


> A real-time video classroom for tutors and students: schedule a class, invite students, and let the room open itself right on time.

**[🚀 Live App](https://lunar-laureate.vercel.app)** · **[📦 Repository](https://github.com/adnan-sami-dev/Lunar-Laureate-A-Video-Conferencing-App-for-Tutors)**

**Stack:** Next.js · TypeScript · Stream Video SDK · Clerk · Tailwind CSS

---

## About This Repository

This is a **source-available portfolio excerpt**. The full implementation lives in a private repository to keep the IP clean.

If you'd like to see the complete codebase, I'm happy to:

- Walk through any part of it live during an interview, or
- Grant read access to the private repo after an initial conversation.

Just reach out: [sami.adnandev@gmail.com](mailto:sami.adnandev@gmail.com) · [LinkedIn](https://www.linkedin.com/in/adnan-sami-3ab549220/)

---

## Table of Contents

- [What It Does](#what-it-does)
- [How It Works](#how-it-works)
- [Tech Stack](#tech-stack)
- [Things I Learned Building This](#things-i-learned-building-this)
- [Bugs That Taught Me Something](#bugs-that-taught-me-something)
- [What I'd Do Differently](#what-id-do-differently)

---

## What It Does

Lunar Laureate is a video classroom for one-on-one tutoring. A tutor schedules a class, adds students as members, and the room opens automatically during a **10-minute window** before the start time. Students see their upcoming classes on a dashboard and join with one click. Past classes can be cleared from the list without actually deleting them.

The video calls themselves weren't the hard part, since the SDK handles that. The interesting work was in the edges: making sure the "Join" button unlocks at the right moment, keeping the UI in sync when several events fire at once, and working around a few gaps in the SDK's query API.

---

## How It Works

```text
Client (Next.js App Router)
  │
  ├──► Clerk              (who is logged in, route protection)
  ├──► Stream Video SDK   (the actual video calls)
  │       │
  │       ├── Create calls with a start time
  │       ├── Query calls the user can see
  │       └── Read live call state (participants, endedAt)
```

A few simple rules hold it together:

- **Auth happens on the server.** The root layout checks `auth()` and redirects logged-out users before any protected page renders. No flash of private content.
- **The start time is stored on the call itself.** Stream keeps it as metadata; the client decides when the room is actually joinable.
- **"Delete" means hide.** Stream has no delete API, so hidden calls are tagged in `custom` data and filtered out on the client.
- **State beats events.** When two events race, the toast system reads `call.state.endedAt` instead of guessing which event fired first.

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

## Things I Learned Building This

### 1. The join window didn't need backstage mode, and that's a real trade-off

Stream offers **backstage mode** for scheduled calls: the host can prep before viewers join, and `join_ahead_time_seconds` lets people in early. For a classroom it felt like overkill. It needs a custom call type, a `JOIN_BACKSTAGE` permission, and a "go live" step the tutor has to remember every single class.

So I did it client-side: a `now` state that ticks every second, and `isDue = startsAt - now <= 10 minutes`. The button unlocks on its own, with no server round-trip, no polling, and no webhook.

**The catch:** someone who knows the SDK could bypass the check by calling `call.join()` directly. For a portfolio app with trusted users, that's fine. For a real classroom where entry needs to be enforced, backstage is the right tool, and I'd reach for it then.

---

### 2. The "Join" button needed a clock, not a poll

The upcoming list only shows future meetings, thanks to Stream's `starts_at` filter. But the Join button has to flip from disabled to enabled at the right moment, without asking the user to refresh.

A one-second tick in each meeting card did the job. It's crude, but there's no network traffic and no extra state to keep in sync.

**The catch:** one timer per card. Twenty meetings means twenty timers. If the list ever grows, I'd lift the interval to the parent and pass `now` down as a prop. That's a small change now and a much bigger one later.

---

### 3. "Delete" is really "hide," because Stream won't let me delete

Stream's `queryCalls` supports filtering by custom fields, but not by negation. This quietly fails:

```json
{ "custom.hidden_userId": { "$ne": true } }
```

Equality checks on custom fields work. `$ne` doesn't.

**The workaround:** tag hidden calls with `custom.hidden_${userId}: true`, fetch everything, and filter on the client. It's not pretty, but it fits inside Stream's API.

**The catch:** every fetch pulls down calls the user has hidden and throws them away. Fine for a handful of meetings. At scale, this needs a server-side solution or a different data model entirely.

---

## Bugs That Taught Me Something

### 1. "Call left" when the tutor had actually ended the call

- **What I saw:** When a tutor ended the call for everyone, students saw "Call left" instead of "Call ended."
- **Why it happened:** The `onLeave` callback fired first and set the toast to "Call left." The `call.ended` WebSocket event arrived a moment later, but the toast state was already set and the component was already unmounting.
- **The fix:** Stop trusting which event fired and start reading `call.state.endedAt` instead. Both triggers call the same handler, which checks the state. If `endedAt` is set, the call was ended. If not, the user left. The event tells you *when*; the state tells you *what*.

> **Lesson:** In real-time apps, events and state race. When they disagree, the state is the truth.

---

### 2. The active-call banner that never appeared

- **What I saw:** The banner on the home page stayed empty, even when a call was clearly in progress.
- **Why it happened:** Two things compounded:
  1. `client.streamClient.user` was `undefined` on the first render, because Clerk and Stream finish attaching at slightly different times. The effect bailed out early and never re-ran.
  2. The query filter used `members: { $in: [userId] }`, but calls created by the user weren't automatically adding them as members.
- **The fix:** Use `useConnectedUser()` so the user ID becomes reactive, add it to the effect's dependency array, and filter on both cases:

  ```ts
  $or: [{ created_by_user_id: userId }, { members: { $in: [userId] } }]
  ```

> **Lesson:** An early return that never clears a loading flag is just an infinite spinner in disguise. And in Stream, "creator" and "member" are two different things.

---

### 3. The scary "Failed to get audio stream" error that wasn't a bug

- **What I saw:** Scheduling a call with `call.getOrCreate()` logged a "Failed to get audio stream" error.
- **Why it happened:** The SDK sets up audio devices internally during call operations. Since the user hadn't opened a meeting room yet, the browser blocked `getUserMedia`.
- **The fix:** Nothing. The error is logged, not thrown. `getOrCreate` resolves cleanly, and the call is created on the server.

> **Lesson:** Not every red line in the console is your bug. If the promise resolves, the operation worked, and the log is just noise from the SDK.

---

## What I'd Do Differently

1. **Use backstage mode for real classrooms.** Client-side time gating is fine for a portfolio, but production classrooms need entry enforced on the server.
2. **Move the time tick into shared context.** One timer for the whole list, not one per card.
3. **Write integration tests for the toast race.** The `endedAt` fix works, but it's subtle. A test that simulates both event orders would keep it working as the code evolves.
4. **Model "hidden" calls server-side from day one.** The client-side filter works today, but the first time the list gets big, it won't.
