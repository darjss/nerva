# PROJECT SOUL v2

## One Program. Many Worlds. One Causal Model.

> **To Codex, future agents, future contributors, and future me:**
>
> This is not a ticket.
>
> This is not a PRD.
>
> This is not a cautious npm-library proposal.
>
> This is the thing we actually want.
>
> This document exists so that when ten agents investigate ten different codebases, nobody accidentally turns the project into:
>
> - Convex with Postgres,
> - Iris with extra steps,
> - Solid on the server,
> - Effect with a nicer README,
> - a distributed EventEmitter,
> - a local-first database we never meant to build,
> - or a 48-package monorepo that has forgotten why it exists.
>
> Read this before designing APIs.
>
> Read it before proposing persistence.
>
> Read it before touching the scheduler.
>
> Read it before deciding the project needs Kafka, CRDTs, R2 WALs, logical replication, a compiler, a custom database, actor placement, or multi-region anything.
>
> Understand the soul first.
>
> Then help us build the **smallest machine capable of proving the biggest idea**.

---

# 0. THE FEELING

Modern web development has normalized something deeply stupid.

A value changes.

Then we manually explain that fact to the application five more times.

```text
database changes
    ↓
backend mutation knows
    ↓
event publisher knows
    ↓
WebSocket knows
    ↓
query cache knows
    ↓
frontend store knows
    ↓
component knows
    ↓
DOM finally knows
```

Why?

Why do we write:

```ts
queryClient.invalidateQueries(...)
```

after the code literally just performed the mutation that made the query stale?

Why do we invent:

```text
task.updated
project.changed
invoice.paid
refresh-dashboard
refetch-users
```

as handwritten causal breadcrumbs for information the runtime could potentially infer?

Why is realtime treated like a feature bolted onto an application instead of a natural consequence of state being alive?

Why does a backend feel like a cemetery of request handlers rather than a living computation graph?

And why, in the age of coding agents, are we asking agents to reproduce **more synchronization choreography**, not less?

That is the itch.

That is the project.

---

# 1. THE BIG IDEA

We want software where:

> **Change propagates because dependency exists.**

Not because a developer remembered to wire an event.

Not because an agent guessed a cache key correctly.

Not because some component called `refetch()`.

Not because a WebSocket handler happened to stay aligned with the backend schema.

Because dependency exists.

Period.

```text
change happens somewhere

        ↓

the runtime already knows
what depends on it

        ↓

the affected computation
becomes dirty

        ↓

necessary work recomputes

        ↓

changed results propagate

        ↓

the application becomes correct
```

The application’s world should be able to participate:

```text
Postgres
Stripe
GitHub
Redis
R2 / S3
webhooks
filesystem
queues
timers
LLM streams
external APIs
other computations
anything trackable

        ↓

reactive server graph

        ↓

reactive remote edge

        ↓

client reactive graph

        ↓

UI
```

The application should feel less like:

```text
backend
+ API
+ cache
+ invalidation
+ event bus
+ realtime layer
+ frontend store
```

and more like:

> **one program with multiple execution worlds connected by explicit causal edges.**

That sentence is important.

---

# 2. THE DAN ABRAMOV CORRECTION

Our earlier instinct was:

> “Make the network disappear.”

That is emotionally right and technically dangerous.

The better lesson from Dan Abramov’s RSC writing is:

> **Do not hide the network. Hide the choreography.**

The browser and server are not actually one machine.

Latency exists.

Failure exists.

Serialization exists.

Authorization exists.

Versions exist.

Disconnects exist.

The boundary matters.

So the goal is not:

```text
pretend remote code is local code
```

The goal is:

```text
make the remote boundary
a first-class part of composition
instead of a second application architecture
```

Better formulation:

> **The network should feel like a first-class reactive edge, not like an entirely separate state-management system.**

The edge must remain semantically visible.

It should be typed.

Inspectable.

Cancelable.

Versioned.

Serializable.

Permission-aware.

Failure-aware.

But developers should not manually orchestrate its synchronization.

---

# 3. ONE PROGRAM, MULTIPLE EXECUTION WORLDS

Dan’s “two computers” model sharpens our own thesis.

Do not organize the conceptual system as:

```text
BACKEND PROGRAM
     +
FRONTEND PROGRAM
```

connected by random HTTP strings.

Think:

```text
ONE LOGICAL APPLICATION

      ┌──────────────┐
      │ server world │
      └──────┬───────┘
             │
       remote edge
             │
      ┌──────▼───────┐
      │ client world │
      └──────────────┘
```

Long term there may be more than two worlds:

```text
database-adjacent actor
        ↓
regional server partition
        ↓
edge
        ↓
browser
```

But do **not** prematurely design “N-computer distributed graph runtime”.

The conceptual insight matters now:

> Execution location is a property of a computation boundary, not necessarily the top-level shape of the application.

---

# 4. THE NORTH STAR

Ordinary application code should describe **business relationships**.

Not synchronization machinery.

The application should not normally contain:

```text
invalidate query
refetch
publish event
socket.on(...)
socket.emit(...)
broadcast(...)
cache.set(...)
cache.patch(...)
queryClient.setQueryData(...)
manual subscription cleanup
duplicated realtime DTOs
"task.updated"
"project.changed"
"refresh-dashboard"
```

Those mechanisms can exist internally.

Adapters can use them.

Transport can use them.

Infrastructure can use them.

But application code should mostly describe:

```ts
const dashboard = computed(async () => {
  const tasks = await tasksFor(projectId());
  const billing = await billingFor(accountId());
  const repo = await githubRepo(repoId());

  return buildDashboard(tasks, billing, repo);
});
```

The runtime discovers:

```text
postgres:tasks(project=42) ───────┐
stripe:subscription:sub_123 ──────┼── dashboard()
github:repo:foo/bar ──────────────┘
```

Then a source changes.

The graph handles causality.

---

# 5. THE DREAM API

These snippets are directional.

They communicate the **feeling**.

Do not cargo-cult the syntax.

---

## 5.1 Database → frontend

Server:

```ts
export const tasks = computed(async () => {
  return db.select().from(task).where(eq(task.projectId, currentProjectId()));
});
```

Frontend:

```tsx
const tasks = api.tasks()

<For each={tasks()}>
  {task => <Task task={task} />}
</For>
```

That should almost be the story.

Not:

```ts
useQuery(...)
invalidateQueries(...)
socket.on(...)
setTasks(...)
refetch(...)
```

Conceptually:

```text
database
   ↓
tracked source read
   ↓
server computation
   ↓
remote reactive edge
   ↓
client reactive node
   ↓
DOM
```

---

## 5.2 Mutation should be boring

Server:

```ts
export const renameTask = mutation(async (id: string, title: string) => {
  await db.update(task).set({ title }).where(eq(task.id, id));
});
```

Client:

```ts
await api.renameTask(task.id, "Ship it");
```

That is it.

The mutation changes authoritative state.

The source layer learns about the committed change.

The graph invalidates the right dependency.

Changed results propagate.

No application-level:

```ts
invalidateTasks();
```

unless explicitly needed.

---

# 6. BUT `computed()` CANNOT MEAN “DO ARBITRARY SHIT”

This is new wisdom we must not forget.

Consider:

```ts
computed(async () => {
  await stripe.charges.create(...)
  return "done"
})
```

If the runtime reruns this because a dependency changed:

**congratulations, we invented Reactive Double Charging.**

So our beautiful API requires serious semantic categories.

Likely something roughly like:

```text
computed
→ repeatable tracked read / derivation

mutation
→ authoritative state transition

action
→ unrestricted external side effect

channel
→ ephemeral live state
```

Names can change.

The distinction probably cannot.

Convex’s separation between query / mutation / action is not arbitrary bureaucracy.

It exists because retryable reactive reads and unrestricted side effects are fundamentally different things.

Do not erase this distinction because the unified API looks prettier without it.

---

# 7. COMPLEX STATE SHOULD BE STRUCTURALLY RICH, NOT SYNCHRONIZATIONALLY MESSY

```ts
const tasks = computed(async () => db.select().from(task));

const overdue = computed(() => tasks().filter(isOverdue));

const completed = computed(() => tasks().filter((task) => task.done));

const stats = computed(() => ({
  total: tasks().length,
  overdue: overdue().length,
  completed: completed().length,
}));

const dashboard = computed(() => ({
  tasks: tasks(),
  overdue: overdue(),
  stats: stats(),
}));
```

Graph:

```text
tasks source
    │
    ▼
  tasks()
  / |   \
 ▼  ▼    ▼
due done stats
 \   |   /
  \  |  /
 dashboard
```

The point is not that this syntax is revolutionary.

The point is that **the graph becomes the synchronization mechanism**.

Complexity appears as relationships.

Not query keys, socket topics, cache patches, and stale-state choreography.

---

# 8. MULTIPLE SYSTEMS SHOULD COMPOSE LIKE VALUES

```ts
const account = computed(async () => {
  const user = await db.user(currentUserId());
  const subscription = await stripe.subscription(user.stripeId);
  const repos = await github.repos(user.githubLogin);
  const avatar = await files.object(user.avatarKey);

  return {
    user,
    subscription,
    repos,
    avatar,
  };
});
```

Graph:

```text
Postgres user ─────┐
Stripe sub ────────┤
GitHub repos ──────┼── account()
R2 avatar ─────────┘
```

Frontend:

```tsx
const account = api.account()

<h1>{account().user.name}</h1>
<PlanBadge plan={account().subscription.plan} />
<RepoList repos={account().repos} />
```

But here is the important truth:

```text
Postgres
Stripe
GitHub
```

do NOT share one transactional timestamp.

So `account()` is not magically a globally consistent snapshot.

The runtime must not lie.

---

# 9. CONSISTENCY IS A PROPERTY OF A ZONE, NOT A MAGIC GLOBAL PROMISE

This is one of the biggest upgrades to the soul.

Different parts of the graph may have different guarantees.

Example:

```text
single Postgres transaction
→ atomic source batch

single local graph partition
→ glitch-free propagation may be possible

Stripe + GitHub + Postgres
→ latest-known / eventual composition

presence
→ best-effort latest value
```

We need a truthful model.

Do not slap the word “reactive” on all of these and pretend they are equivalent.

Research distributed reactive programming:

```text
causal
glitch-free
atomic
```

and understand the coordination cost of each.

The project may need multiple consistency classes.

But the public API must make them understandable.

---

# 10. THE WORD “GLITCH”

Suppose:

```text
      A
     / \
    B   C
     \ /
      D
```

A changes.

B updates.

C has not updated yet.

D temporarily computes:

```text
B(new)
C(old)
```

That intermediate state may be impossible according to the logical program.

That is a reactive **glitch**.

Locally, topological scheduling can help.

Across machines:

```text
latency
reordering
disconnects
partial failure
```

turn this into a distributed-systems problem.

Therefore:

> “One causal graph across machines” is not a slogan we are allowed to use casually.

We must earn whatever consistency semantics we claim.

---

# 11. THE BACKEND SHOULD BE A REAL COMPUTATION GRAPH

Iris appears to show the power of:

```text
source
  ↓
live route
  ↓
client
```

That may be exactly the right abstraction for many apps.

We want to test the deeper primitive:

```text
source
  ↓
computed
  ↓
computed
  ├────▶ computed
  │
  └────▶ computed
           ↓
       remote edge
```

Arbitrary server computations can reuse other server computations.

This may provide:

```text
shared computation
smaller recomputation units
better composition
source neutrality
cleaner client bindings
```

But it also introduces:

```text
scheduler complexity
identity complexity
lifecycle complexity
distributed semantics
debugging complexity
```

We must prove the deeper graph earns its existence.

---

# 12. FINE-GRAINED GRAPH DOES NOT REQUIRE PERFECTLY FINE-GRAINED SOURCES

The graph can be fine-grained.

Adapters can be coarse.

A first Postgres source may know:

```text
postgres:table:tasks
```

A better adapter:

```text
postgres:tasks:project=42
```

A better one:

```text
postgres:tasks:row=891
```

A serious incremental-query engine might know:

```text
tasks
WHERE project_id = 42
AND status = 'open'
```

These are separate levels.

> **Graph granularity and source granularity are separate concerns.**

Do not build an incremental SQL engine inside core.

---

# 13. NEW HYPOTHESIS: COARSE INVALIDATION MAY BE FINE

DICE, Salsa, Skyframe, and other incremental systems teach an important idea:

```text
invalidated
≠
changed
```

Suppose:

```text
tasks table changes
```

so:

```text
tasksForProject(42)
```

becomes dirty.

It reruns.

Its result is logically equal to the previous result.

Then downstream work may stop.

```text
source invalidates
      ↓
node dirty
      ↓
recompute
      ↓
new result == old result?
      ↓ yes
STOP PROPAGATION
```

This may be enormously important.

Maybe the practical 80/20 is:

```text
coarse source invalidation
+
lazy observed graph
+
result equality / early cutoff
```

rather than heroic predicate-level tracking.

Research this before obsessing over perfect DB dependencies.

---

# 14. FROM-SCRATCH CONSISTENCY

Self-adjusting computation gives us a powerful correctness ideal:

> After incremental propagation settles, the visible result should equal what a clean from-scratch execution would produce over the same logical inputs.

That should inspire our testing strategy.

Build a stupid reference runtime:

```text
anything changes
→ throw away derived graph state
→ recompute everything observed
```

Then fuzz the optimized runtime.

Compare results.

If the incremental runtime diverges from the dumb correct model, we have a bug.

This is a better north star than “the unit tests pass”.

---

# 15. THE CORE SHOULD BE ALMOST OFFENSIVELY SMALL

The core should not know:

```text
Postgres
Stripe
Solid
React
Elysia
Cloudflare
R2
```

At the conceptual center:

```ts
source();
computed();
subscribe();
invalidate();
transaction();
```

Maybe additional primitive categories emerge.

Maybe names change.

But the center should remain:

```text
read dependency
    ↓
record edge
    ↓
dependency changes
    ↓
dirty dependents
    ↓
recompute
    ↓
compare
    ↓
propagate changed results
```

Everything else should earn its place.

---

# 16. EFFECT v4: THE ENGINE ROOM, NOT THE RELIGION

Current hypothesis:

> **Effect handles nasty asynchronous execution mechanics underneath our runtime while the user-facing API stays Effect-free.**

User code:

```ts
const tasks = computed(async () => {
  return db.select().from(tasks);
});
```

Internally:

```text
ComputedNode
    ↓
Effect fiber
    ├─ Scope
    ├─ cancellation
    ├─ finalizers
    ├─ timeout/retry machinery
    ├─ tracing
    ├─ failure Cause
    └─ Stream machinery
```

This separation is extremely attractive.

---

# 17. WHAT EFFECT SHOULD OWN

Potentially:

```text
structured concurrency
fiber execution
cancellation/interruption
resource scopes
finalizers
streams
retry schedules
timeouts
typed internal failures
dependency injection internally
tracing/observability
RPC/schema machinery where useful
```

We should **not** rebuild these merely to prove we can.

Use other people’s battle scars.

---

# 18. WHAT EFFECT CANNOT DECIDE FOR US

Effect can cancel a fiber.

It cannot decide:

```text
what does cancellation mean
for a reactive generation?
```

Effect can provide Scope.

It cannot decide:

```text
when should an unobserved server node die?
```

Effect can provide Stream.

It cannot decide:

```text
do we preserve every version
or coalesce to latest?
```

Effect can model errors.

It cannot decide:

```text
should a failed future generation
replace the last committed value?
```

Effect provides execution machinery.

**We own reactive semantics.**

That boundary matters.

---

# 19. DO NOT MAKE EFFECT ATOM OUR ONTOLOGY WITHOUT EVIDENCE

Effect v4’s reactivity/Atom work is extremely relevant prior art.

Study it deeply.

But do not automatically define:

```text
our ComputedNode = Effect Atom
```

because then our project’s semantics become constrained by an unstable abstraction designed for potentially different use cases.

Preferred investigation:

```text
OUR logical graph
       ↓
Effect execution substrate
```

not:

```text
Effect Atom
       ↓
somehow turn it into our distributed model
```

Challenge this preference during research.

---

# 20. EFFECT ESCAPE HATCH FOR POWER USERS

Normal API:

```ts
const user = computed(async () => db.user(id()));
```

Advanced Effect-native API could exist:

```ts
const billing = computed.effect(
  Effect.gen(function* () {
    const stripe = yield* Stripe;

    return yield* stripe.subscription(id());
  }),
);
```

Both produce the same logical graph-node semantics.

Effect should be opt-in at the authoring layer.

Not mandatory literacy for a todo app.

---

# 21. REMOTE COMPUTATION SHOULD BE DECLARATIVE

Dan’s function-call vs declarative-description distinction matters.

Calling:

```ts
api.tasks();
```

should not necessarily mean:

```text
immediately open a socket
execute SQL
allocate a server graph
```

It may represent a **remote computation description**.

Observation activates it.

Conceptually:

```text
remote reference exists
        ↓
client graph observes it
        ↓
remote edge activates
        ↓
server node becomes observed
        ↓
upstream graph activates
```

Then:

```text
client stops observing
        ↓
remote subscription disappears
        ↓
server node loses observers
        ↓
eventually collect/passivate
```

This aligns beautifully with fine-grained reactive ownership.

Do not rush the API.

But preserve the declarative lifecycle model.

---

# 22. NEVER CREATE PARALLEL HIERARCHIES ACROSS THE NETWORK

We must avoid reinventing this:

```text
server:
"/project-dashboard"

client cache:
["project-dashboard", id]

WebSocket:
"project.updated"

types:
ProjectDashboardDTO
```

Four identities for one concept.

That is the old sickness.

The desired model is closer to:

```ts
export const project =
  serverComputed(async (id: string) => ...)
```

Then elsewhere:

```ts
const p = remote(() => project(props.id));
```

One symbol.

Typed.

Navigable.

Find-references works.

Agents can follow it.

Compiler/build tooling can know the edge crosses execution worlds.

This is one of the strongest lessons from RSC.

---

# 23. ORGANIZE CODE AROUND FEATURES, NOT MACHINES

Do not force:

```text
frontend/
backend/
shared/
```

to be the fundamental conceptual architecture.

Prefer:

```text
task/
project/
billing/
deployment/
```

with explicit cross-world boundaries inside those feature concepts.

Physical deployment remains separate.

But the codebase should not duplicate the domain hierarchy by machine.

This is another RSC/tierless-programming lesson worth stealing.

---

# 24. FINE-GRAINED COMPOSITION, COALESCED TRANSPORT

This should become a commandment.

Suppose the page observes:

```ts
const project = remote(projectFor(id));
const tasks = remote(tasksFor(id));
const members = remote(membersFor(id));
const billing = remote(billingFor(id));
const deploy = remote(deploymentFor(id));
```

A naïve runtime might create:

```text
5 subscription negotiations
5 RPC waterfalls
5 connections
```

That would be elegant code sitting on garbage networking.

No.

The rule:

> **Fine-grained programming model. Coalesced transport.**

Client graph may observe twenty remote nodes.

Transport should batch observation and share a connection/session where possible.

Server can keep fine-grained graph identities internally.

Network should not mirror graph chatter one packet per edge.

---

# 25. INITIAL LOADING AND LIVE UPDATES MAY BE THE SAME PROBLEM

This is a research hypothesis inspired by Progressive JSON.

Imagine a remote result:

```text
dashboard = {
  project: $17,
  tasks: $18,
  billing: $19,
  deployment: $20
}
```

Values arrive independently:

```text
$17 @ v4
$18 @ v8
$20 @ v2
$19 @ v7
```

Later:

```text
$18 @ v9
```

Maybe:

```text
initial load
streaming
live updates
```

are all just:

> **versions of remote graph values arriving over time.**

That would be beautiful.

It may also be protocol hell.

Do not build it yet.

Benchmark snapshots first.

But investigate.

---

# 26. SNAPSHOTS ARE NOT EMBARRASSING

Do not fetishize graph-node wire protocols.

The simplest first transport may be:

```text
computation reruns
    ↓
send whole snapshot
```

That may be good enough.

Snapshots compress well.

They are easy to reconnect.

They are easy to debug.

Before building:

```text
stable graph IDs
structural diffing
partial node patches
resume cursors
```

measure real payloads.

Complex protocols must earn themselves.

---

# 27. SOLID 2 IS THE FIRST CLIENT BECAUSE IT SPEAKS OUR LANGUAGE

Solid 2 is not the product.

It is the first frontend where this can feel absurdly natural.

Solid believes:

> async belongs in the reactive graph.

We believe:

> remote computation belongs in the reactive graph.

Desired:

```ts
const project = remote(() => api.project(props.id));
```

Then:

```tsx
<h1>{project().name}</h1>
```

Not:

```ts
const {
  data,
  loading,
  error,
  refetch
} = useRemoteQuery(...)
```

The server graph should terminate into the client graph.

```text
SERVER

source
  ↓
computed
  ↓
computed
  ↓
remote edge

══════════════════

CLIENT

Solid node
  ↓
memo
  ↓
DOM
```

Do not invent a second cache state machine if Solid can already represent the reactive state.

---

# 28. REACT / SVELTE / VUE SHOULD BE BORING

Framework binding:

```text
RemoteValue
   ├── Solid → accessor / async node
   ├── React → useSyncExternalStore
   ├── Svelte → store / rune
   ├── Vue → ref
   └── Vanilla → subscribe()
```

Frameworks differ at the last mile.

Not in server semantics.

If the React adapter needs to reinvent:

```text
cache
invalidation
transport
subscription identity
```

then our remote primitive is wrong.

---

# 29. REMOTE VALUE IS A FIRST-CLASS CONCEPT

We need a neutral primitive meaning:

> This value is produced in another execution world, may change, and has lifecycle/version/failure semantics.

Maybe:

```ts
interface RemoteValue<T> {
  get(): T;
  subscribe(fn: () => void): () => void;
}
```

Maybe:

```ts
AsyncIterable<T>;
```

Maybe richer.

Do not freeze this prematurely.

Preserve the architectural shape:

```text
server computation
        ↓
remote edge protocol
        ↓
neutral remote value
        ↓
framework-native reactive value
```

---

# 30. SOURCES ARE THE EYES AND EARS OF THE GRAPH

The graph cannot psychically know the world changed.

The source adapter answers:

> **How did we learn dependency X changed?**

The graph answers:

> **What depends on X?**

```text
Postgres
Stripe
GitHub
Redis
R2
filesystem
webhook
queue
     ↓
source adapter
     ↓
invalidate(dep)
     ↓
reactive graph
     ↓
affected computations
```

Never mix these responsibilities.

---

# 31. THE APPLICATION’S WORLD SHOULD BE REACTIVE

Eventually adapters may include:

```text
Postgres
MySQL
SQLite
Drizzle
Prisma
Kysely
Redis
Stripe
GitHub
R2
S3
webhooks
polling APIs
SSE
WebSocket
queues
timers
filesystem
LLM streams
```

All reduce conceptually to:

```text
read source
    ↓
establish dependency

source changes
    ↓
invalidate dependency
```

This is what makes the project more general than a reactive database.

---

# 32. SOURCES HAVE DIFFERENT PHYSICS

Push:

```text
Stripe webhook
GitHub webhook
CDC
R2 notification
Redis pubsub
SSE
WebSocket
```

Pull:

```text
legacy REST
RSS
vendor API with no webhook
```

Local:

```text
filesystem watcher
timer
process state
memory
```

Some are:

```text
transactional
eventual
durable
best-effort
polling
```

The abstraction should unify programming.

It must not erase reality.

---

# 33. TIME IS A SOURCE

This is wrong:

```ts
const overdue = computed(() => tasks().filter((t) => t.dueAt < Date.now()));
```

Nothing tells the graph when time changes.

Instead:

```ts
const now = clock.every("minute");

const overdue = computed(() => tasks().filter((t) => t.dueAt < now()));
```

Same for:

```text
randomness
feature flags
environment state
filesystem state
permissions
```

If a changing fact matters, it must enter through a tracked dependency.

This is the reactive equivalent of hermeticity.

---

# 34. AUTHORIZATION IS A REACTIVE DEPENDENCY

Suppose:

```ts
const project = computed(async () => {
  assertCanRead(currentUser(), projectId());
  return db.project(projectId());
});
```

Then membership changes.

If permissions are not part of the graph, an open subscription may continue receiving data.

Therefore:

```text
membership
roles
permissions
auth scope
token validity
```

may themselves be reactive inputs.

Auth is not merely part of a cache key.

It may be part of causality.

Security beats deduplication.

Always.

---

# 35. SHARED COMPUTATION

If 5,000 clients safely observe:

```text
dashboard(tenant=42)
```

we want:

```text
              dashboard(42)
          /    /    |    \    \
         ▼    ▼     ▼     ▼    ▼
      client client ... client client
```

not:

```text
dashboard(42) × 5,000
```

But computation identity may need:

```text
definition
arguments
tenant
auth scope
permissions version
locale
deployment version
```

This is one of the hardest areas.

Do not optimize sharing before security semantics are explicit.

---

# 36. NODE IDENTITY MAY BE HARDER THAN DEPENDENCY TRACKING

What is:

```text
project(42)
```

?

Potential identity:

```text
computation definition
+
arguments
+
tenant
+
auth scope
+
execution world
+
code/deployment version
```

Now deploy new code.

Is old:

```text
project(42)
```

the same logical node?

Maybe yes for subscription continuity.

Maybe no for cached results.

This needs a deliberate v0 story.

Simple acceptable v0:

```text
deployment version changes
→ reconnect
→ rebuild remote graph
```

Fine.

But say it.

---

# 37. TRANSACTIONS ARE CORRECTNESS

Suppose one DB transaction:

```text
moves task
decrements project A
increments project B
writes audit row
```

The client should not see:

```text
task new
counts old
```

if the source guarantees atomicity.

Desired:

```text
source transaction commits
      ↓
group invalidations
      ↓
recompute affected local graph
      ↓
commit one visible graph version
      ↓
publish
```

This may only hold within one transactional source/partition.

Do not extrapolate it globally.

---

# 38. POSTGRES CHANGE DETECTION IS NOT A DETAIL

The graph only propagates after somebody learns a source changed.

Possible Postgres strategies:

```text
instrument ORM writes
LISTEN / NOTIFY
triggers
logical decoding / CDC
transactional outbox
polling
```

Each has different:

```text
external-write coverage
durability
ordering
transaction semantics
operational cost
granularity
```

V0 may begin with instrumentation.

But if we want:

```text
DataGrip edits DB
→ browser updates
```

we need an external-write detection path.

Research before choosing.

---

# 39. THE DUAL-WRITE TRAP

This is not correct enough:

```ts
await db.update(...)
broadcast("tasks changed")
```

Failure window:

```text
DB commits
process crashes
broadcast never happens
```

Stale clients.

Or:

```text
broadcast happens
DB rolls back
```

False invalidation.

Outbox / CDC / WAL-derived signals exist for a reason.

We do not need the hardest version in v0.

But we cannot pretend the failure window does not exist.

---

# 40. DURABLE AND EPHEMERAL STATE ARE DIFFERENT

Steal this instinct from Iris.

Durable:

```text
orders
issues
documents
accounts
subscriptions
```

Ephemeral:

```text
cursor
typing
presence
selection
hover
voice activity
```

Do not WAL mouse movement.

Different state deserves different guarantees.

A possible future API:

```ts
computed(...)
mutation(...)
action(...)
channel(...)
```

may reflect that distinction.

---

# 41. STATE AND EVENTS ARE DIFFERENT

State asks:

> What is true now?

Event asks:

> What happened?

Reactive graph values are primarily about state.

A payment event may need durable processing even if nobody is subscribed.

Do not force reliable event processing into the same abstraction as live state.

This project is not automatically a workflow engine.

---

# 42. BACKPRESSURE EXISTS

What if:

```text
100,000 source updates/sec
          ↓
slow computation
          ↓
mobile client
```

Options:

```text
process every version
coalesce to latest
drop intermediate state
disconnect slow subscriber
snapshot periodically
```

Different data deserves different semantics.

Presence may want latest-only.

Audit events may need every event.

Effect Stream can provide machinery.

We must decide semantics.

---

# 43. THE LOGICAL GRAPH IS NOT THE PHYSICAL DISTRIBUTION

This is critical.

Logical:

```text
A → B → C → D
```

must NOT imply:

```text
Actor A
→ network
Actor B
→ network
Actor C
→ network
Actor D
```

That would be catastrophically chatty.

Orleans already teaches this lesson.

Likely:

```text
ProjectGraphActor
{
  many local reactive nodes
}
```

The actor / Durable Object / cell is a **physical coordination partition**.

Computed nodes are logical graph structure inside it.

Fine-grained semantics.

Coarse physical placement.

---

# 44. SERVERLESS SHOULD NOT BREAK THE MODEL

Logical computation identity must not require an immortal process.

Long-running:

```text
Bun / Node
    ↓
hot graph
```

Cloudflare:

```text
Worker
   ↓
Durable Object
   ↓
graph partition
   ↓
hibernate
   ↓
wake
```

Future actor/cell runtime:

```text
cell
 ↓
local materialization
 ↓
durable backing
```

Logical identity can persist while hot execution comes and goes.

---

# 45. THE GRAPH SHOULD BE DISPOSABLE

Crash?

Fine.

Eviction?

Fine.

Machine disappears?

Fine.

Reconstruct from:

```text
computation identity
active subscriptions
source cursors
durable source truth
re-execution
```

Prefer:

```text
recompute
+
rediscover dependencies
```

over:

```text
persist every in-memory pointer forever
```

when practical.

The graph is derived state.

---

# 46. OBJECT STORAGE MAY BECOME BEDROCK — LATER

celld / Cursor Continuity suggest:

```text
durable truth
+
disposable warm compute
+
notifications as optimization
```

Future possibility:

```text
R2 / S3
  ├── logs
  ├── checkpoints
  ├── source cursors
  ├── manifests
  └── versions
        ↓
hot graph partition
        ↓
clients
```

Interesting.

Not v0.

Do not turn our first prototype into distributed storage research.

---

# 47. OBSERVABILITY IS PART OF THE PRODUCT

If the runtime is automatic, it must be **violently inspectable**.

We should answer:

> Why did this rerun?

Example:

```text
dashboard(tenant:42)

STATUS
shared
3 subscribers
generation 918

DEPENDS ON
├─ postgres:tasks(project=42)
├─ postgres:users(tenant=42)
├─ stripe:subscription:sub_123
└─ computed:permissions(user=18)

LAST INVALIDATION
postgres:tasks(project=42)
transaction 58192

PROPAGATION
tasks()
  ↓
projectStats()
  ↓
dashboard()

RESULT
3.1ms compute
output unchanged
propagation stopped
```

That last line matters.

The runtime should expose not just invalidation, but **why propagation stopped**.

---

# 48. AGENTS SHOULD QUERY THE GRAPH

Machine-readable introspection is core to the agent-native thesis.

Potential questions:

```text
why did dashboard rerun?
which source invalidated it?
show hottest nodes
show nodes that rerun but rarely change output
show fanout > 1000
show coarse source invalidations
show leaked unobserved nodes
show auth scopes
show longest critical propagation path
```

An agent should debug causality directly.

Not grep eight synchronization layers.

---

# 49. THIS COULD BE A BETTER WORLD FOR CODING AGENTS

Today an agent must align:

```text
DB mutation
API route
query key
cache
invalidation
WebSocket topic
frontend store
component lifecycle
```

All must agree.

That is unnecessary state space.

We want one model:

```text
read dependency
    ↓
runtime records edge
    ↓
dependency changes
    ↓
affected computation updates
```

A better agent-native framework does not merely give AI better documentation.

It **removes categories of code AI should never have needed to generate.**

This must eventually be benchmarked.

Not just believed.

---

# 50. V0 SHOULD FIT IN ONE BRAIN

Maybe:

```text
packages/
  core/
  drizzle/
  solid/

examples/
  board/
```

Maybe less.

Core proves:

```text
logical node identity
fine-grained dependency graph
async tracking
dynamic dependency cleanup
coalescing
stale-run protection
result equality / early cutoff
computation-to-computation dependencies
subscriptions
lifecycle
transaction grouping
inspection
```

Effect may handle:

```text
fiber execution
Scope
cancellation
resource cleanup
streams
internal errors
```

Drizzle proves:

```text
normal reads can establish source dependencies
normal writes can invalidate
```

Solid proves:

```text
remote computation feels like normal reactive state
```

Board proves:

```text
DataGrip edits DB
    ↓
browser updates

second browser mutates
    ↓
first browser updates

no manual realtime plumbing
```

Enough.

---

# 51. THE KILLER DEMO

We want code simple enough that the reaction is:

> **“Wait… that's all the code?”**

Example server:

```ts
export const board = computed(async (projectId: string) => {
  const tasks = await db.select().from(task).where(eq(task.projectId, projectId));

  return {
    tasks,
    stats: {
      total: tasks.length,
      done: tasks.filter((t) => t.done).length,
    },
  };
});
```

Frontend:

```tsx
const board = remote(() =>
  api.board(props.projectId)
)

<TaskBoard tasks={board().tasks} />
<Stats value={board().stats} />
```

Now:

```text
psql / DataGrip / another app instance
changes a task
```

UI updates.

No:

```text
WebSocket topic
manual cache key
invalidation call
refetch
```

If that works reliably and the runtime remains understandable, we have evidence.

---

# 52. FALSIFY THE FINE-GRAINED GRAPH

Compare:

### A — Iris-style coarse live route

```text
touch("/board")
→ rerun
→ snapshot
```

### B — coarse source + fine graph + early cutoff

```text
table invalidates
→ observed nodes rerun
→ equality stops unchanged propagation
```

### C — fine source dependencies

```text
row / predicate invalidation
→ smallest possible graph work
```

Measure:

```text
LOC
CPU
memory
DB queries
network bytes
latency
implementation complexity
debuggability
```

If A wins comfortably:

build A.

The soul is not “fine-grained topology”.

The soul is **remove synchronization choreography**.

---

# 53. TEST LIKE A RUNTIME

Do not only test:

```text
set value
expect value
```

Test:

```text
1,000 concurrent computations
random await timing
random invalidation timing
dynamic branch switching
subscribe/unsubscribe storms
errors
duplicate invalidations
nested computations
transactions
reconnects
auth changes
```

Use fuzz/property tests.

Compare optimized execution against from-scratch execution.

Trust the graph because we tortured it.

---

# 54. NON-NEGOTIABLE INVARIANTS

### Async isolation

Concurrent computations must never steal each other’s dependencies.

### Dynamic dependencies

```ts
computed(() => (flag() ? A() : B()));
```

must stop depending on A after switching to B.

### Stale execution

Older async results must never overwrite newer committed state.

### Dirty while running

Invalidation during execution must eventually be reflected.

### Failed rerun

A failed speculative execution must not corrupt the last committed dependency graph.

### Equality cutoff

Dirty does not automatically mean changed.

### Transactions

Atomic source changes should remain atomic inside the guarantee zone.

### Lifecycle

Unobserved nodes must eventually disappear/passivate.

### Auth

Shared computations must never cross security boundaries.

### Inspectability

Every automatic decision must be explainable.

---

# 55. WHAT WE SHOULD STUDY BEFORE THE REAL IMPLEMENTATION PLAN

## Incremental computation

Beat these into the ground:

```text
Buck2 DICE
Salsa
Bazel Skyframe
Adapton
Jane Street Incremental
Meteor Tracker
```

Questions:

```text
node identity
revision model
early cutoff
dynamic edges
shared work
async dependency collection
cycles
lifecycle
```

---

## Distributed reactive semantics

Study:

```text
Distributed REScala
DREAM
cost of consistency
fault-tolerant distributed reactive programming
Gavial
```

Questions:

```text
what is a glitch?
causal vs glitch-free vs atomic?
what coordination does each require?
what can we honestly promise?
```

---

## Reactive databases / dataflow

Study:

```text
Noria
DBSP
Differential Dataflow
Timely Dataflow
Materialize
TanStack DB / d2ts
Zero
Electric
```

Question:

> Where does our source adapter stop and a real incremental query engine begin?

We do not want to invent database theory by accident.

---

## Runtime kernel

Study Effect v4:

```text
Fiber
Scope
Stream
Cause
Atom
Reactivity
RPC
Schema
SQL integrations
Cluster/Entity
```

Question:

> Which nasty execution semantics should we refuse to reinvent?

---

## Cross-machine programming model

Study Dan:

```text
React for Two Computers
Progressive JSON
JSX Over The Wire
One Roundtrip Per Navigation
Impossible Components
```

Also:

```text
Links
Eliom
Ur/Web
Hop
Gavial
```

Question:

> How do we make remote composition feel local without hiding network reality?

And:

> Why did tierless programming not take over the web already?

That answer matters.

---

## Physical distribution

Study:

```text
Cloudflare Durable Objects
Orleans
Restate
celld
Cursor Continuity
Temporal
```

Question:

> How does logical graph identity map onto physical placement, passivation, recovery, and durability?

---

## Postgres source semantics

Study:

```text
ORM instrumentation
LISTEN/NOTIFY
triggers
logical decoding
CDC
Debezium
transactional outbox
```

Question:

> What is the smallest reliable external-write story?

---

# 56. RESEARCH RULE: STEAL SEMANTICS, NOT BRAND IDENTITY

From Solid:

```text
dependency graph
async reactivity
ownership
```

From Convex:

```text
reactive query correctness
transaction boundaries
semantic separation of reads/writes/actions
```

From Iris:

```text
excellent live DX
share
touch escape hatch
durable vs broadcast
ephemeral channels
identity
```

From Effect:

```text
structured async execution
resources
streams
errors
```

From Dan/RSC:

```text
explicit composable machine boundary
direct symbol relationships
no parallel API hierarchies
coalesced network work
```

From DICE/Salsa/Skyframe:

```text
incremental graph algorithms
early cutoff
revisions
shared work
```

From distributed reactive research:

```text
consistency vocabulary
glitch semantics
cost of guarantees
```

From Noria/dataflow:

```text
partial materialization
incremental query maintenance
demand
```

Do not cosplay any one system.

Build the deeper simple thing that survives all of them.

---

# 57. DEAR CODEX: DO NOT TURN THIS INTO ENTERPRISE SOUP

You are powerful.

You are also capable of generating 60 files because somebody wrote “production-ready”.

Please resist.

Do not automatically invent:

```text
repository pattern
service layer
manager layer
factory layer
plugin registry
DI container
telemetry framework
custom logger
configuration framework
20 generic interfaces
```

before semantics deserve them.

We care about **semantic clarity**.

If `track()` is 40 lines, let it be 40 lines.

If `computed()` needs 200 careful lines and 100 brutal tests, do that.

Do not hide a weak primitive behind architecture cosplay.

---

# 58. HUGE VISION. RUTHLESSLY SMALL PROOF.

Do not let “MVP thinking” flatten this into:

> “just build CRUD and see if anyone wants it.”

No.

The ambition tells us what primitive matters.

But do not let ambition become:

```text
multi-region R2-backed
event-sourced actor mesh
with custom dataflow protocol
```

before we prove:

```text
source changes
    ↓
graph knows why
    ↓
affected computation updates
    ↓
client becomes correct
```

Huge vision.

Ruthlessly small proof.

---

# 59. YES, THE AMBITION IS STILL TO BOIL THE OCEAN

We have learned more.

The ambition did not shrink.

It became less stupid.

If the primitive survives:

```text
database changes
external SaaS changes
files change
jobs finish
AI streams progress
permissions change
deployment state changes

          ↓

one causal application model

          ↓

server computations

          ↓

explicit remote reactive edges

          ↓

client computations / UI
```

Maybe frontend and backend stop feeling like unrelated state machines.

Maybe “realtime” stops being a feature category.

Maybe cache invalidation becomes plumbing.

Maybe API state management starts looking like a historical workaround.

Maybe coding agents become dramatically better at full-stack software because the runtime removes synchronization concepts from their search space.

Maybe an ecosystem appears:

```text
@thing/postgres
@thing/stripe
@thing/github
@thing/redis
@thing/r2
@thing/cloudflare
@thing/solid
@thing/react
@thing/svelte
```

Maybe Iris-style live routes become a higher-level layer.

Maybe Lakebed-style agent-native environments emerge.

Maybe other frameworks use the core.

We are allowed to think this big.

> **Boil the ocean in vision. Sip it in implementation.**

---

# 60. THE PRODUCT FEEL

If this works, the developer reaction should be:

> **“Wait… that’s all the code?”**

Not:

> “Wow, that is a sophisticated distributed-systems implementation.”

Nobody cares.

The sophistication exists to make this boring:

```ts
const tasks = api.tasks();
```

and this normal:

```tsx
<For each={tasks()}>{(task) => <Task task={task} />}</For>
```

while:

```text
the database changes somewhere else
```

and the UI simply becomes correct.

Complex state should be rich in relationships.

Not synchronization boilerplate.

---

# 61. THE SHARPER ONE-SENTENCE VISION

> **Build full-stack applications as reactive computations connected across explicit execution boundaries. Read from databases, external services, streams, and other computations; when tracked dependencies change, the runtime propagates only the necessary changed state through the graph and across reactive remote edges.**

That is more precise than:

> “everything is magically reactive.”

And more ambitious than:

> “live queries.”

---

# 62. MESSAGE TO FUTURE AGENTS

Do not protect this architecture.

Attack it.

Find:

```text
the paper that already solved our scheduler
the runtime that makes our async machinery unnecessary
the consistency theorem that breaks our slogan
the benchmark where coarse invalidation wins
the auth leak
the reconnect race
the source that cannot be tracked
the side effect that cannot be safely rerun
the deployment identity problem
the protocol complexity that buys nothing
```

Then tell us what survives.

If half the architecture dies:

good.

The surviving half is stronger.

We are looking for the **deepest simple thing**.

Not the most elaborate thing.

---

# SOUL

This project is not fundamentally a sync engine.

It is not fundamentally a database.

It is not fundamentally a frontend framework.

It is not fundamentally a realtime library.

It is not fundamentally Effect.

It is not fundamentally Solid.

It is a runtime for expressing software as **reactive computation across explicit execution boundaries**.

The north star:

```text
a fact changes somewhere

        ↓

a tracked dependency captures
what that fact affects

        ↓

necessary computation recomputes

        ↓

unchanged results stop propagation

        ↓

changed state crosses
explicit reactive boundaries

        ↓

the application becomes correct
without synchronization choreography
```

Do not hide physics.

Do not duplicate causality.

Do not reinvent solved machinery.

Do not worship graph granularity.

Do not promise consistency we cannot provide.

Do not let the user see the horror underneath unless they ask.

Make the API boring.

Make the runtime terrifyingly correct.

And if we can make database state, external systems, server computation, remote values, client reactivity, and UI feel like one coherent causal program—

then yeah:

# GO CRAZY.

Build the thing that makes the old synchronization tower look ridiculous.
