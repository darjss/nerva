# Iris — Reference Notes for Codex

> **Purpose:** Iris is not yet a stable, fully documented OSS dependency we can rely on.
> This file records what we currently know from SaltyAom's public posts, the deployed Iris Board demo, and the code snippet shared alongside it.
>
> Treat this as **context for architectural comparison**, not authoritative implementation documentation.
>
> **Critical rule: separate observed facts from our inferences. Do not invent Iris internals where source is unavailable.**

---

# 1. What Iris is, according to its author

Iris is being developed by SaltyAom, creator of Elysia.

Public statements we have seen describe Iris as roughly:

> “kind of a sync engine library”

and more specifically as a system that:

- makes computation reactive
- tracks dependency relationships
- is data-source agnostic
- does not require a database
- does not require a sidecar
- works well with Elysia, but is not conceptually limited to Elysia
- can integrate with external data sources
- has adapters/plans around Drizzle, Prisma, and Kysely
- can automatically react to database updates through adapters
- is currently coarser-grained than systems like Convex/Zero
- can use explicit/manual invalidation when it cannot infer exactly what changed
- can coexist with TanStack DB for local/client-side concerns
- is still undergoing internal redesign
- was not yet stable, general OSS when these notes were written

The important conceptual point:

> **Iris does not require application data to live inside an Iris-owned database.**

That is a major distinction from Convex.

---

# 2. Iris does not appear to promise perfect fine-grained invalidation

SaltyAom explicitly indicated that Iris is not currently fine-grained in the Convex/Zero sense.

The reason matters:

> Iris cannot automatically know the semantic meaning of arbitrary application data.

An adapter may know:

```text
tasks table changed
```

without knowing:

```text
only row 918 changed
and only query project=42,status=open is affected
```

So the current Iris model appears intentionally pragmatic.

Conceptually:

```text
source change
    ↓
invalidate known dependency / live route
    ↓
rerun affected computation
    ↓
publish fresh result
```

rather than:

```text
source change
    ↓
perfect predicate-level incremental maintenance
    ↓
recompute exact affected field only
```

Do not assume Iris is trying to be an incremental database engine.

---

# 3. The Iris Board demo

A deployed demo was shared at:

```text
https://iris-board.millennium.sh
```

The public snippet accompanying the demo looked approximately like this:

```ts
const sync = iris({
  getIdentity: ({ headers }) => headers["x-viewer"],

  bus: {
    durable: bunPostgres(db.$client),
    broadcast: redisBroadcast(redisUrl),
  },
});

export const app = new Elysia()
  .use(sync)

  // durable state
  .get(
    "/board",
    {
      live: {
        share: true,
      },
    },
    async () => ({
      issues: await listTasks(),
      team,
    }),
  )

  // ephemeral state
  .get(
    "/board/presence",
    {
      channel: {
        of: viewerSchema,
        share: true,
      },
    },
    ({ channel }) => ({
      viewers: [...channel.entries].sort((a, b) => a.id - b.id),
    }),
  )

  // normal write
  .patch(
    "/api/issues/:id",
    {
      body: taskBodySchema,

      touch: {
        route: "/api/board",
      },
    },
    async ({ params, body }) => {
      if (!(await replaceTask(params.id, body)))
        return problem(404, {
          detail: "Issue not found",
        });
    },
  )

  // high-write ephemeral presence
  .ws("/board/presence", {
    body: viewerSchema,

    publish: {
      route: "/api/board/presence",
    },

    message({ channel, body, headers }) {
      channel.publish({
        ...body,
        id: headers["x-viewer"],
      });
    },

    close({ channel }) {
      channel.remove();
    },
  });

// client

const api = treaty<App>(url).use(live());

const { data } = useLive(api.board.live);

const presence = useLive(api.board.presence.live);
```

The exact route prefixes may have been abbreviated or mounted under a prefix.

Do **not** treat `/board` vs `/api/board` as a confirmed bug. The code was presented as “something like this,” not necessarily byte-for-byte production source.

---

# 4. What the board snippet tells us with relatively high confidence

## 4.1 A normal Elysia GET can become a live computation

This:

```ts
.get('/board', {
  live: {
    share: true
  }
}, async () => ({
  issues: await listTasks(),
  team
}))
```

suggests Iris can turn an ordinary route handler into a live value.

The handler still looks like ordinary application code.

The developer does not manually:

```text
open WebSocket
serialize event
broadcast snapshot
manage subscriber list
reconnect
```

That machinery is abstracted by Iris.

This is excellent DX.

---

# 5. `share: true` is important

Observed:

```ts
live: {
  share: true;
}
```

and:

```ts
channel: {
  of: viewerSchema,
  share: true
}
```

Likely meaning:

> multiple subscribers can share one live computation/channel instance when safe.

Conceptually:

```text
board computation
      ↓
   shared
  /  |  \
 A   B   C
```

rather than:

```text
board(A)
board(B)
board(C)
```

However, we do **not** yet know the exact computation-key semantics.

Unknowns:

- are arguments included automatically?
- is auth/identity included?
- how are tenant boundaries handled?
- is sharing process-local or distributed?
- does a shared computation stay hot with zero subscribers?
- is there a TTL/cache policy?

Do not assume.

---

# 6. `touch` appears to be explicit invalidation

Observed:

```ts
touch: {
  route: "/api/board";
}
```

on a mutation.

This appears to mean:

```text
mutation succeeds
    ↓
mark board live route dirty
    ↓
rerun it
    ↓
publish new result
```

Conceptually, it is close to:

```ts
invalidate("/board");
```

This strongly matches SaltyAom's statement that Iris is not perfectly fine-grained and can use manual invalidation.

This is not necessarily a weakness.

It is a pragmatic abstraction.

A huge amount of realtime plumbing disappears even if invalidation itself is coarse.

---

# 7. Likely ORM adapter behavior

SaltyAom has discussed adapters for:

```text
Drizzle
Prisma
Kysely
```

The stated idea is that database access can become trackable.

A plausible high-level model is:

```text
live computation executes
    ↓
ORM adapter observes reads
    ↓
dependencies are associated with computation
```

Then:

```text
ORM write
    ↓
adapter knows some source/table changed
    ↓
invalidate affected live computations
```

Current public statements suggest this may initially be table-level or otherwise coarse.

Example:

```text
computation reads tasks
    ↓
dependency: tasks table

tasks table changes
    ↓
rerun computation
```

rather than precise predicate-level invalidation.

> The exact implementation is unknown until Iris source or formal docs are available.

Do not assume SQL AST analysis, triggers, CDC, or row-level tracking unless confirmed.

---

# 8. External database changes

SaltyAom indicated that Iris can synchronize when the database is updated outside the normal application client.

Examples:

```text
DataGrip
psql
another service
admin script
```

changes the DB.

Iris can apparently observe enough external change to update subscribers.

That requires some detection mechanism.

Possible mechanisms in systems like this include:

```text
Postgres triggers
NOTIFY/LISTEN
CDC
logical replication
change table
polling
```

But:

> **We do not currently know which mechanism Iris uses.**

Do not claim one.

This is an important area to inspect when source becomes available.

---

# 9. Durable bus + broadcast bus

Observed configuration:

```ts
bus: {
  durable: bunPostgres(db.$client),
  broadcast: redisBroadcast(redisUrl)
}
```

Architecturally, this is very interesting.

It strongly suggests Iris separates:

```text
durable coordination / history / recovery
```

from:

```text
fast cross-instance fanout
```

Our likely interpretation:

```text
Postgres durable bus
     ↓
correctness / persistence / recovery information

Redis broadcast
     ↓
low-latency multi-instance notification
```

But the exact semantics are unknown.

Questions to answer from source/docs:

- What exactly is persisted?
- Is the durable bus an event log?
- Is it only a cursor/store?
- Does it guarantee ordering?
- Can it recover after downtime?
- Is Redis purely an optimization?
- What if Redis drops a message?
- Can Iris recover from only the durable bus?
- What are the delivery guarantees?

Do not overclaim.

The **separation itself** is worth studying.

---

# 10. Durable state and ephemeral state are separate

The Board demo makes this distinction explicit.

Durable board state:

```ts
.get('/board', {
  live: {
    share: true
  }
}, ...)
```

Presence:

```ts
.get('/board/presence', {
  channel: {
    of: viewerSchema,
    share: true
  }
}, ...)
```

Presence writes use:

```ts
.ws(...)
```

and:

```ts
channel.publish(...)
channel.remove()
```

This suggests Iris has different primitives for:

```text
business/durable state
```

and:

```text
high-frequency ephemeral state
```

Examples of ephemeral state:

```text
cursor
presence
typing
selection
mouse position
voice activity
```

These should not necessarily go through the same durability machinery as:

```text
issues
orders
documents
accounts
```

We should learn from this.

---

# 11. Identity is a sync-layer primitive

Observed:

```ts
getIdentity: ({ headers }) => headers["x-viewer"];
```

The Board uses identity for presence:

```ts
channel.publish({
  ...body,
  id: headers["x-viewer"],
});
```

Identity may also matter for:

```text
subscription scoping
shared computation safety
presence ownership
authorization
reconnection
per-user state
```

Exact semantics are unknown.

For our project:

> auth/identity cannot be bolted on after computation sharing exists.

This is a correctness concern.

---

# 12. Client DX

Observed:

```ts
const api = treaty<App>(url).use(live());
```

Then:

```ts
const { data } = useLive(api.board.live);
```

and:

```ts
const presence = useLive(api.board.presence.live);
```

This means Iris integrates with Elysia Treaty's typed API model.

The user does not appear to define separate:

```text
socket message types
manual channel names
subscription DTOs
event mappings
```

Realtime stays inside the typed API surface.

This is worth stealing philosophically:

> **Do not make realtime a second API surface.**

---

# 13. Our current mental model of Iris

This is an inference, not confirmed source architecture.

The simplest model consistent with the public demo is:

```text
                   SOURCE
                     │
                     ▼
             dependency / touch
                     │
                     ▼
              LIVE COMPUTATION
               (often route)
                     │
              share / dedupe
                     │
                     ▼
              transport layer
                     │
                     ▼
                  useLive()
```

For a write:

```text
PATCH /issue/:id
      ↓
database changes
      ↓
touch('/board')
      ↓
rerun board handler
      ↓
broadcast fresh board
```

For ORM auto-tracking:

```text
board handler
      ↓
reads tasks table
      ↓
Iris records dependency
      ↓
tasks source changes
      ↓
board reruns
```

For presence:

```text
WebSocket message
      ↓
channel.publish()
      ↓
presence live route changes
      ↓
subscribers update
```

This is a very sensible architecture.

It is much simpler than a fully arbitrary fine-grained distributed computation graph.

That simplicity may be a feature, not a limitation.

---

# 14. How Iris differs from the runtime we are exploring

Do not turn this into competitive chest-thumping.

We are testing a different abstraction depth.

Iris currently appears centered around:

```text
live route / live computation
```

Our vision asks whether the deeper primitive should be:

```text
arbitrary computed node
```

Iris-like model:

```text
tasks table
    ↓
/board live route
    ↓
clients
```

Our target experiment:

```text
tasks source
    ↓
tasks()
   /   \
  ▼     ▼
stats overdue
   \     /
    dashboard()
        ↓
      clients
```

Potential benefit:

- graph reuse
- smaller recomputation scope
- natural derived state
- sources beyond DB
- same computation used by many outputs
- potentially direct Solid 2 graph integration

Potential cost:

- harder scheduling semantics
- graph bookkeeping
- lifecycle complexity
- auth-sensitive sharing
- distributed graph placement
- stale async generations
- harder DevTools
- runtime overhead with little benefit on simple apps

We must prove the deeper graph earns its complexity.

---

# 15. What we SHOULD steal from Iris

## 15.1 Ordinary code should become live

Good:

```ts
.get('/board', {
  live: {
    share: true
  }
}, async () => ...)
```

The live system decorates normal application code.

It does not force developers into a separate universe.

## 15.2 Explicit sharing

Good idea:

```ts
share: true;
```

Make shared computation intentional or at least inspectable.

Do not silently share auth-sensitive work.

## 15.3 Explicit invalidation escape hatch

Good:

```ts
touch: {
  route: ...
}
```

Even with automatic dependency tracking, keep an escape hatch.

## 15.4 Separate durable from broadcast

Potentially excellent architecture:

```text
durable bus
+
fast broadcast bus
```

Do not force low-latency delivery and durable recovery to be the same mechanism.

## 15.5 Separate ephemeral channels

Presence/cursor/typing deserves a different primitive from durable application state.

## 15.6 Identity is not an afterthought

Sharing + realtime + distributed state requires clear identity/security semantics.

## 15.7 End-to-end typing

Realtime should not create a second handwritten API contract.

---

# 16. What we should NOT blindly copy

Do not blindly make:

```text
route = fundamental reactive node
```

until we decide that is actually the best core abstraction.

Do not assume:

```text
coarse table invalidation
```

is our final target.

Do not make Elysia required by the core.

Do not make Redis or Postgres required by the core.

Do not copy undocumented implementation details from guesses.

Do not reproduce Iris APIs merely because they look good.

Understand the semantics first.

---

# 17. Unknowns Codex should investigate if source becomes available

## Core graph model

- What exactly is a dependency node?
- Is there computation-to-computation dependency tracking?
- Or only source-to-live-route dependencies?
- Are dependencies dynamically replaced after reruns?
- How are async dependencies tracked?
- Does tracking survive `await`?
- How are concurrent computations isolated?

## Scheduling

- How are repeated invalidations coalesced?
- What happens if invalidation arrives while a computation is running?
- Are stale async generations cancelled or ignored?
- Are reruns serialized?

## Sharing

- What exactly does `share: true` key on?
- Arguments?
- Identity?
- Headers?
- Tenant?
- Auth?
- Is sharing distributed?

## ORM tracking

- Table-level?
- Row-level?
- Query-level?
- Predicate-level?
- How are joins handled?
- How are external DB writes detected?

## Durability

- What does `bunPostgres(...)` persist?
- Events?
- Versions?
- Cursors?
- Subscribers?
- Results?
- Is it authoritative?
- Can the runtime recover after Redis loss?

## Broadcast

- What are Redis delivery guarantees?
- What happens if a broadcast is missed?
- Is recovery cursor-based?

## Client protocol

- Snapshots or patches?
- Sequence numbers?
- Reconnect protocol?
- Replay?
- Resume cursor?
- Backpressure?
- Equality/diff semantics?

## Channels

- Process-local or distributed?
- Ownership model?
- Disconnect cleanup?
- Stale presence cleanup?

## Security

- Where is authorization evaluated?
- Is auth context part of shared computation identity?
- What happens when permissions change mid-subscription?

## Lifecycle

- When are shared live computations destroyed?
- Reference counting?
- TTL?
- GC?
- Reconnect grace period?

---

# 18. The most important architectural lesson from Iris

Iris appears to be doing something very smart:

> **It removes most realtime plumbing without requiring the developer to adopt an entirely new database or programming environment.**

That alone is valuable.

Our project must not lose that ergonomic lesson while chasing a deeper graph.

We do not want the user experience to become:

```text
learn distributed graph theory
configure 12 runtime layers
annotate every dependency
understand actor placement
```

just to get a live task board.

If our internals are more sophisticated, the public surface should become **simpler**, not more sophisticated.

The target feeling remains:

```ts
const tasks = api.tasks();
```

and then the data stays alive.

---

# 19. Current high-level comparison

```text
Convex
------
reactive server queries
strong correctness semantics
owns DB/runtime/platform

Iris
----
existing-app-friendly live computations/routes
data-source agnostic
explicit + adapter-driven invalidation
Elysia/Treaty-first ergonomics
durable + broadcast + channels

Our experiment
--------------
framework-neutral reactive computation runtime
arbitrary server computed nodes
source adapters with variable granularity
remote reactive primitive
Solid 2 first-class, React/Svelte later
aim: one causal graph across server/client boundaries
```

This is a hypothesis.

Not a superiority claim.

---

# 20. Instructions to Codex

When Iris source becomes available:

Do **not** begin by porting it.

First produce a semantic report:

```text
1. What is the smallest core abstraction?
2. What are the exact invariants?
3. How are async dependencies tracked?
4. What does share mean?
5. What is durable?
6. What is merely broadcast?
7. How are external DB changes detected?
8. What does touch actually invalidate?
9. How does reconnect work?
10. Which pieces are Elysia integration versus Iris core?
```

Then produce:

```text
WHAT TO STEAL
WHAT TO AVOID
WHAT WE ALREADY DO DIFFERENTLY
WHAT INVALIDATES OUR ASSUMPTIONS
WHAT TESTS WE SHOULD COPY
```

If Iris reveals that our proposed deeper graph is unnecessary or too expensive, say so.

Do not defend our architecture out of loyalty.

The point of studying Iris is to become less wrong.

---

# TL;DR FOR AN AGENT

```text
Iris is an unreleased/experimental sync-reactivity library from SaltyAom.

It appears to make normal Elysia routes live.

A live route can use:
  live: { share: true }

Mutations can explicitly invalidate live routes using:
  touch: { route: ... }

There are ORM integrations planned/mentioned for Drizzle, Prisma, Kysely.

Tracking is currently not necessarily fine-grained.
Coarse/table-level invalidation is acceptable.

Iris is data-source agnostic and does not require its own database.

The Board demo separates:
  durable live application state
from
  ephemeral channel/presence state.

It also configures:
  durable: bunPostgres(...)
  broadcast: redisBroadcast(...)

which suggests durable recovery/coordination and fast fanout are separate concerns.

Identity is configured centrally:
  getIdentity(...)

Client DX uses typed Elysia Treaty + live integration:
  useLive(api.board.live)

Our project is inspired by this DX, but is testing a deeper primitive:
  arbitrary fine-grained server computations as graph nodes,
  not only live routes.

Do not assume Iris internals beyond these observations.
```
