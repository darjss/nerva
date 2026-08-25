# PROJECT SOUL — ADVERSARIAL RESEARCH DOSSIER

## Dear Agent: Try to Kill This Idea Before You Help Us Build It

> This is a red-team brief, not a validation exercise.
>
> Your job is to find every system, paper, runtime, database, build engine, programming model, and distributed protocol that can prove we are reinventing something, expose an invariant we missed, show a simpler abstraction, reveal a performance cliff, or hand us machinery we would be stupid to rebuild.
>
> **Assume the project is wrong until it survives prior art.**
>
> Do not protect the architecture. Beat it into the ground.

---

# 0. THE THESIS UNDER ATTACK

Current vision:

```text
sources
  ↓
arbitrary reactive server computations
  ↓
computation-to-computation dependency graph
  ↓
remote reactive edge
  ↓
framework-native reactive graph
  ↓
UI
```

Sources may include:

```text
Postgres
Stripe
GitHub
Redis
R2 / S3
webhooks
queues
timers
filesystem
LLM streams
other APIs
```

Desired user-facing code is boring TypeScript:

```ts
export const tasks = computed(async () => {
  return db.select().from(tasks).where(eq(tasks.projectId, projectId()));
});
```

Frontend:

```tsx
const tasks = api.tasks();
```

Mutation:

```ts
export const renameTask = mutation(async (id, title) => {
  await db.update(tasks).set({ title }).where(eq(tasks.id, id));
});
```

Desired consequence:

```text
DB changes
  ↓
runtime already knows dependencies
  ↓
affected computation updates
  ↓
subscribed client updates
```

without ordinary application code containing:

```text
invalidateQueries()
refetch()
socket.emit()
socket.on()
manual cache patch
"task.updated"
"refresh-dashboard"
```

The core should remain framework-neutral.

Effect v4 may be used **underneath** as an execution kernel for fibers, cancellation, scopes, streams, resource lifetime, retries, timeouts, typed internal failures, and observability. Normal users should not need to know Effect.

Solid 2 is the intended first/best frontend integration, but React/Svelte/Vue bindings should be terminal adapters.

We are testing a deeper primitive than an Iris-style live route:

```text
Iris-ish:
source → live route → client

Our hypothesis:
source → computed → computed → remote edge → client graph
```

Now attack it.

---

# 1. FIVE QUESTIONS THAT CAN KILL THE PROJECT

## Kill Question A — Does fine-grained server computation actually buy enough?

Compare:

```text
source changed
    ↓
rerun /board
    ↓
send snapshot
```

with:

```text
source changed
    ↓
propagate through internal graph
    ↓
recompute affected nodes
    ↓
publish affected values
```

If the first version is cheap, simple, and already sufficient for realistic apps, our graph may be architecture astronautics.

**Required evidence: benchmark it.**

## Kill Question B — Can arbitrary external sources have coherent semantics?

Convex can provide strong query consistency partly because it controls the database/runtime and restricts reactive queries.

We want:

```ts
computed(async () => {
  const rows = await postgres(...)
  const customer = await stripe(...)
  const repo = await github(...)
  return ...
})
```

There is no shared transaction timestamp across Postgres, Stripe, and GitHub.

What does “consistent” mean here?

If the answer is fuzzy, the abstraction is lying.

## Kill Question C — Are arbitrary async computations too effectful to rerun safely?

If `computed()` can contain arbitrary JavaScript:

```ts
computed(async () => {
  await stripe.charges.create(...)
})
```

then automatic reruns can double-charge somebody. Retries can repeat effects. Cancellation cannot undo an email.

Maybe `computed()` must be semantically read-only and repeatable, while writes live in `mutation()` / `action()` / `effect()`.

Investigate hard.

## Kill Question D — Is one graph across machines economically stupid?

A logical graph does not imply a physical actor/RPC per node.

If every fine-grained node becomes:

```text
actor
RPC
network message
durable record
```

we lose.

Distributed reactive programming research shows stronger propagation guarantees cost communication and coordination. Orleans explicitly warns against chatty virtual actors.

Maybe the logical graph must be coarsened into physical execution partitions.

## Kill Question E — Are we rebuilding an incremental computation/dataflow engine badly?

Before writing the scheduler, deeply inspect:

```text
Buck2 DICE
Bazel Skyframe
Salsa
Adapton
Jane Street Incremental
Timely Dataflow
Differential Dataflow
Noria
DBSP
```

If our algorithm is a toy version of one of these, stop and steal the semantics.

---

# 2. S-TIER — BUCK2 DICE

Buck2 is powered by an incremental computation graph. Modern DICE is dangerously close to the local heart of our proposed runtime: keyed computations, dependency tracking, memoization, invalidation, shared work, parallel async execution, and equality-based early cutoff.

Sources:

- https://buck2.build/docs/insights_and_knowledge/modern_dice/
- https://buck2.build/docs/about/why/
- https://buck2.build/docs/bxl/faq/

## Questions

### Identity

What exactly is a DICE key? How are function identity and arguments represented? What assumptions exist around hash/equality?

Map that to our likely key:

```text
computation definition
+ arguments
+ tenant
+ auth scope
+ execution zone
+ deployment version?
```

### Early cutoff

This may be enormous.

```text
source invalidates A
A reruns
A returns logically same value
```

Does B downstream need to rerun?

Potential rule:

```text
invalidation ≠ changed result
```

Study DICE equality and early cutoff carefully.

### Shared work

If 100 consumers request the same dirty node concurrently, how does DICE ensure one computation is shared?

What happens when some consumers cancel?

### Dynamic dependencies

How are old edges reconciled after:

```ts
if (flag) A();
else B();
```

switches branches?

Do failed runs ever commit dependency changes?

### Parallel dependency collection

Modern DICE discusses `joinAll`, separate dependency collection for parallel futures, and merging dependency information afterward.

This may directly inform our async graph design.

### Central graph state vs parallel work

Investigate whether a mostly single-owner graph-state manager plus parallel asynchronous evaluators is simpler/faster than fine-grained locking.

Potential architecture:

```text
single graph state owner per partition
+
Effect fibers doing execution
```

### Deliverable

```text
WHAT DICE SOLVES
WHAT DOES NOT MAP
INVARIANTS TO COPY
DATA STRUCTURES TO COPY
BENCHMARKS TO COPY
WHY WE SHOULD OR SHOULD NOT PORT ITS MODEL
```

---

# 3. S-TIER — BAZEL SKYFRAME

Skyframe is another mature incremental computation engine.

Core shape:

```text
SkyKey
→ SkyFunction
→ requests dependencies through environment
→ SkyValue
```

Values are immutable. Reads become graph edges. Reverse dependencies are invalidated. Skyframe uses **change pruning**: if an invalidated node recomputes to the same value, downstream work may be resurrected/skipped.

Sources:

- https://bazel.build/reference/skyframe
- https://bazel.build/versions/7.7.1/reference/skyframe
- https://bazel.build/contribute/codebase

## Attack questions

Why are SkyValues immutable?

Why must every changing input be read through the tracked environment?

What breaks with:

```ts
computed(() => globalMutableThing.value);
```

?

Potential rule for us:

> If changing state affects a computation, it must enter through a tracked source.

Skyframe gets correctness from hermetic-ish assumptions. Our external API ambition weakens those assumptions. Identify exactly which guarantees we lose.

## Node decomposition lesson

Skyframe often gets more incremental behavior by decomposing expensive work into smaller nodes rather than mutating pieces of one node incrementally.

Maybe our sane fine-grained model is:

```text
tasks()
stats()
billing()
deployment()
dashboard()
```

plus equality cutoff, rather than magical field-level diff propagation.

---

# 4. S-TIER — SALSA

Salsa is a Rust framework for incremental, on-demand programs. It tracks dependencies of memoized functions and uses the **red-green algorithm**.

Sources:

- https://salsa-rs.github.io/salsa/
- https://salsa-rs.github.io/salsa/overview.html
- https://salsa-rs.github.io/salsa/reference/algorithm.html
- https://salsa-rs.github.io/salsa/cycles.html

## Concepts to understand

### Revisions

Separate concepts such as:

```text
source revision
execution generation
committed graph revision
published remote version
```

may all matter. Do not collapse them casually.

### Red-green validation

Understand exactly when Salsa validates dependency freshness versus re-executes a query.

### Backdating

If a tracked function reruns but produces a logically unchanged result, Salsa can preserve the older “changed at” revision so downstream work does not rerun.

Our analog:

```text
table invalidates tasksForProject(42)
query reruns
rows are same
dashboard does not rerun
```

This may make coarse source invalidation surprisingly viable.

### Tracked identity / interning

Study Salsa tracked/interned structures before inventing stable remote node identity.

### Cycles

Salsa detects cycles and can optionally recover through fixed-point-style mechanisms. V0 should probably reject cycles loudly, but study the prior art first.

---

# 5. S-TIER — SELF-ADJUSTING COMPUTATION / ADAPTON

Our idea is not new at the computer-science level. Self-adjusting computation studies programs that record dependencies and efficiently update outputs when inputs change.

Read:

- “Adapton: Composable, Demand-Driven Incremental Computation”
- “A Consistent Semantics of Self-Adjusting Computation”
- work by Umut Acar, Matthew Hammer, Khoo Yit Phang, Michael Hicks, Jeffrey Foster

Starting point:

- https://www.cs.umd.edu/class/fall2015/cmsc631/Potential_projects.html

## Steal this correctness oracle

A central idea in this research family is essentially **from-scratch consistency**:

> after propagation, the observable incremental result should agree with evaluating from scratch on the new inputs.

This should become a property test.

```text
optimized runtime result
==
dumb throw-everything-away and recompute result
```

under randomized:

```text
source changes
dynamic dependency switches
async delays
errors
subscription churn
```

## Demand-driven lesson

Adapton combines incrementality and demand. That maps directly to:

```text
client observes remote value
    ↓
server node activates
    ↓
upstream dependencies activate
```

and zero observers eventually permitting passivation/collection.

---

# 6. S-TIER — DISTRIBUTED REACTIVE PROGRAMMING

This research branch is probably the most dangerous to our slogan “one causal graph across boundaries.” Researchers have literally studied distributed reactive propagation and the cost of consistency.

Read:

```text
Distributed REScala
DREAM
On the Semantics of Distributed Reactive Programming: The Cost of Consistency
Fault-tolerant Distributed Reactive Programming
Thread-Safe Reactive Programming
newer consistent distributed RP work
```

Sources:

- https://www.rescala-lang.com/publications
- https://dl.acm.org/doi/10.1145/2611286.2611290
- https://programming-group.com/assets/pdf/papers/2014_We-Have-a-DREAM-Distributed-Reactive-Programming-with-Consistency-Guarantees.pdf
- https://arxiv.org/abs/1902.00524
- https://arxiv.org/abs/2502.20534
- https://drops.dagstuhl.de/storage/00lipics/lipics-vol109-ecoop2018/LIPIcs.ECOOP.2018.1/LIPIcs.ECOOP.2018.1.pdf

## Learn the word: GLITCH

```text
      A
     / \
    B   C
     \ /
      D
```

A changes. B updates first. D sees:

```text
B(new)
C(old)
```

That intermediate state may be impossible according to the source program. That is a glitch.

Across a network we add:

```text
latency
reordering
partial failure
independent machines
retries
disconnects
```

Now semantics get expensive.

## DREAM's propagation guarantees

DREAM explicitly studies different guarantees such as:

```text
causal
glitch-free
atomic
```

with different implementation costs.

The agent must explain each in our terminology and produce a table:

```text
GUARANTEE | MEANING | COST | WHERE NEEDED | WHERE IMPOSSIBLE/POINTLESS
```

Map them across:

```text
single process
single graph partition
single DB transaction
multiple partitions
Postgres + Stripe
browser reconnect
presence
```

## Project-threatening conclusion to consider

There may be **no single universal consistency model**.

Potentially:

```text
DB transaction graph → atomic/glitch-free
Stripe + GitHub composition → eventual/latest-known
presence → best effort/latest wins
```

Find the simplest truthful semantics.

---

# 7. S-TIER — CONVEX

Convex is our production teacher for reactive server reads.

Sources:

- https://docs.convex.dev/functions/query-functions
- https://docs.convex.dev/functions/mutation-functions
- https://docs.convex.dev/functions/actions
- https://docs.convex.dev/understanding/zen
- https://docs.convex.dev/database/advanced/occ

Convex query semantics include:

```text
automatic caching
reactivity
single logical database snapshot per query
```

Queries/mutations are deterministic and third-party APIs are kept out of them; unrestricted third-party effects live in actions.

That restriction buys semantics.

## Adversarial question

We want to permit arbitrary external reads. What guarantees do we lose?

Do not answer “Effect handles retries.” Effect handles execution, not cross-system consistency.

## Semantic separation

Investigate why Convex has:

```text
query
mutation
action
```

Maybe our core uses different names, but our semantics may still need distinctions like:

```text
reactive read-only computation
transactional mutation
unrestricted side-effecting action
ephemeral channel
```

For each Convex guarantee, report:

```text
what assumption enables it?
do we have that assumption?
if not, what weaker guarantee can we honestly provide?
```

---

# 8. S-TIER — NORIA

Noria is frighteningly relevant: dynamic, partially-stateful dataflow for high-performance web applications.

Sources:

- https://www.usenix.org/conference/osdi18/presentation/gjengset
- https://www.usenix.org/system/files/osdi18-gjengset.pdf

Ideas include:

```text
parameterized queries
dataflow
incremental updates
shared computation
partial materialization
eviction
state reconstruction on demand
live graph changes
```

## Questions

What did Noria learn about:

```text
state explosion
partial materialization
demand-driven state
query parameterization
reuse across related queries
graph changes
```

?

Why did this architecture not simply become the default web backend model?

Do not guess. Research the tradeoffs and later evolution.

Its partial materialization/upquery ideas may inform sleeping server graph state.

---

# 9. S-TIER — DBSP / DIFFERENTIAL DATAFLOW / TANSTACK DB

Stop pretending fine-grained SQL is easy.

DBSP provides automatic incremental view maintenance for rich query languages. Differential Dataflow maintains computations over changing collections. TanStack DB uses `d2ts`, a TypeScript differential-dataflow implementation, for live queries.

Sources:

- https://arxiv.org/abs/2203.16684
- https://materialize.com/blog/differential-from-scratch/
- https://tanstack.com/db/latest/docs/overview
- https://timelydataflow.github.io/timely-dataflow/

If we eventually want incremental:

```text
JOIN
GROUP BY
ORDER BY
LIMIT
aggregates
nested queries
```

we should probably integrate or learn from a real IVM/dataflow engine rather than inventing SQL invalidation rules in our graph.

Potential layering:

```text
Postgres
   ↓
CDC
   ↓
d2ts / IVM adapter
   ↓
reactive source
   ↓
our computation graph
```

not:

```text
our runtime becomes a SQL optimizer
```

---

# 10. S-TIER — EFFECT V4

Current hypothesis: Effect is an execution substrate, not the graph ontology.

User:

```ts
computed(async () => ...)
```

Internally:

```text
our computation node
      ↓
Effect Fiber
Scope
Stream
Cause
retry/timeout
resource finalizers
tracing
```

Sources:

- https://effect.website/
- https://effect.website/blog/releases/effect/40-beta
- https://github.com/Effect-TS/effect

As of August 2026 the Effect site presents 4.0 as a release candidate.

## Investigate exactly

```text
Fiber / interruption
Scope
Stream
Cause
unstable/reactivity/Atom
unstable/reactivity/Reactivity
AtomRegistry
RPC
Schema
SQL integrations
Cluster / Entity if relevant
```

Effect reactivity already contains process-local invalidation and atom dependency/lifecycle ideas. Study them; do not automatically adopt their ontology.

## Architectural question

A:

```text
build everything ourselves
```

B:

```text
make Effect Atom our graph
```

C:

```text
own graph semantics
use Effect for execution mechanics
```

Current preference: C. Challenge it.

## Never conflate machinery and semantics

Effect can answer:

```text
how do I interrupt this fiber safely?
```

It cannot decide:

```text
what does interruption mean for our reactive generation semantics?
```

Effect can close a Scope. We decide when a remote node is logically dead.

---

# 11. S-TIER — DAN ABRAMOV / RSC PROGRAMMING MODEL

Read:

- React for Two Computers
- Progressive JSON
- JSX Over The Wire
- One Roundtrip Per Navigation
- Impossible Components
- What Does "use client" Do?

Sources:

- https://overreacted.io/react-for-two-computers/
- https://overreacted.io/progressive-json/
- https://overreacted.io/one-roundtrip-per-navigation/
- https://overreacted.io/jsx-over-the-wire/
- https://overreacted.io/impossible-components/

## Lessons to test

### One program, multiple worlds

Do not force frontend/backend into unrelated programming models merely because they execute separately.

### Explicit but composable boundary

Hide synchronization choreography, not physics.

Potential slogan:

> The network becomes a first-class reactive edge, not a second application architecture.

### Avoid parallel hierarchies

Bad:

```text
server: "project-dashboard"
client: ["project-dashboard", id]
socket: "project.updated"
```

Better:

```ts
import { project } from "./project";
const p = remote(() => project(id));
```

One symbol, typed and navigable.

### Fine-grained programming model, coalesced transport

If a page observes:

```text
project()
tasks()
members()
billing()
deployment()
```

that must not mean five network waterfalls.

Potential invariant:

> **Fine-grained composition. Coalesced transport.**

### Progressive protocol hypothesis

Progressive JSON sends referenced pieces out-of-order as they become ready.

Speculative extension:

```text
initial load
+
later live update
```

might be representable through a versioned graph-value protocol:

```text
dashboard = { project: $17, tasks: $18, billing: $19 }
$17@v4 = ...
$18@v8 = ...
$19@v2 = ...

later:
$18@v9 = ...
```

This is only a research hypothesis. Benchmark it against boring snapshots before building protocol machinery.

---

# 12. A-TIER — METEOR TRACKER

Study the tiny ancestor:

```text
Tracker.currentComputation
Dependency.depend()
Dependency.changed()
invalidate
flush
cleanup
```

A getter registers the current computation. A dependency changes. The computation reruns.

Why this matters: we risk overengineering a primitive that can begin with brutally clear semantics.

Investigate async tracking and nested lifecycle behavior. Use modern async context / Effect rather than mutable globals if appropriate.

---

# 13. A-TIER — JANE STREET INCREMENTAL / GLIMMER AUTOTRACKING

Study mature graph implementations for:

```text
dependency tags / versions
equality cutoff
graph ordering
dynamic edge reconciliation
unused-node collection
debugging
```

The repeated appearance of cutoff semantics across DICE, Skyframe, Salsa, and Incremental is a major signal.

---

# 14. A-TIER — TIERLESS WEB LANGUAGES

Our phrase “one program with different execution zones” has ancestors.

Study:

```text
Links
Eliom
Ur/Web
Hop
Gavial
multi-tier FRP
```

Sources:

- https://links-lang.org/
- https://ocsigen.org/tuto/latest/application.html
- https://www.impredicative.com/ur/
- https://arxiv.org/abs/2002.06188

Gavial is especially relevant because it combines multi-tier programming with FRP and explicitly considers glitches and network traffic.

## Uncomfortable question

If one program across browser/server/database is beautiful, why did tierless programming not conquer mainstream web development?

Investigate actual causes:

```text
new-language adoption
compiler lock-in
ecosystem isolation
debugging
hidden network behavior
deployment assumptions
interop
framework gravity
```

Potential lesson: ordinary TypeScript + explicit module boundaries may win where a new language loses.

---

# 15. A-TIER — TIMELY DATAFLOW / NAIAD

If we ever distribute the graph seriously, Timely becomes relevant.

Sources:

- https://timelydataflow.github.io/timely-dataflow/
- https://timelydataflow.github.io/timely-dataflow/chapter_5/chapter_5_2.html
- https://michaelisard.com/pubs/naiad_sosp2013.pdf

Timely uses logical timestamps and distributed progress tracking. A frontier expresses which logical times may still appear.

This addresses:

> How does a downstream operator know it has seen everything relevant for logical time T?

Our future distributed graph may encounter:

```text
A@v5 arrived
B@v5 not yet arrived
is version 5 complete?
```

Do not invent a toy global version counter without understanding this literature.

Also note: Timely's progress tracking has real overhead, and its own docs warn that too many timestamps are expensive. V0 probably does not need it.

---

# 16. A-TIER — ZERO / ELECTRIC / LOCAL-FIRST

Be explicit about what this project is not.

Zero provides query-driven partial sync and a local replica.

Sources:

- https://zero.rocicorp.dev/
- https://zero.rocicorp.dev/docs/when-to-use

If a user needs:

```text
offline reads
instant local writes
local query execution
sync after reconnect
```

our server reactive graph alone does not solve it.

Investigate composition instead of conquest. Maybe our remote graph can feed or consume local-first systems later.

---

# 17. A-TIER — POSTGRES CHANGE DETECTION

The graph answers:

> what depends on X?

Something still must answer:

> how did we learn X changed?

Investigate:

```text
instrumented ORM writes
LISTEN / NOTIFY
triggers
logical decoding / CDC
transactional outbox
polling
```

Sources:

- https://www.postgresql.org/docs/current/logicaldecoding.html
- https://www.postgresql.org/docs/current/logicaldecoding-output-plugin.html
- https://www.postgresql.org/docs/current/sql-notify.html
- https://debezium.io/documentation/reference/stable/transformations/outbox-event-router.html

Postgres logical decoding exposes committed row changes and preserves transaction boundaries/commit ordering. `NOTIFY` inside a transaction is delivered only upon successful commit.

## Required matrix

```text
MECHANISM       EXTERNAL WRITES?  DURABLE?  TX BOUNDARY?  GRANULARITY  OPS COST
ORM wrapping
LISTEN/NOTIFY
trigger+NOTIFY
logical decoding
Debezium
polling
```

Decide what v0 supports and what it deliberately does not.

---

# 18. THE DUAL-WRITE TRAP

Suppose mutation does:

```text
UPDATE tasks
publish invalidation
```

Failure windows:

```text
DB commits
process crashes before publish
→ clients remain stale
```

or:

```text
publish happens
DB rolls back
→ clients recompute unnecessarily / see old state
```

This is the classic dual-write problem.

Investigate:

```text
transactional outbox
CDC
WAL/logical decoding
```

Do not call:

```ts
await db.update(...)
broadcast(...)
```

“correct” without admitting the failure window.

---

# 19. A-TIER — ACTORS / DURABLE OBJECTS / ORLEANS / RESTATE

Logical graph:

```text
A → B → C → D
```

does not imply:

```text
Actor A → RPC B → RPC C → RPC D
```

Sources:

- https://learn.microsoft.com/en-us/dotnet/orleans/overview
- https://learn.microsoft.com/en-us/dotnet/orleans/resources/best-practices
- https://learn.microsoft.com/en-us/dotnet/orleans/host/configuration-guide/activation-collection
- https://docs.restate.dev/foundations/services
- https://docs.restate.dev/tour/microservice-orchestration
- https://developers.cloudflare.com/durable-objects/concepts/durable-object-lifecycle/

Orleans explicitly warns against chatty grain communication and recommends combining highly chatty grains.

## Core question

What is the relationship between:

```text
logical graph node
```

and:

```text
physical coordination partition
```

Likely:

```text
one actor / DO / cell contains many graph nodes
```

Partition around workspace/project/document/tenant/room, not every memo.

Study activation/passivation: logical identity persists while in-memory materialization comes and goes.

---

# 20. A-TIER — CURSOR CONTINUITY / CELLD / DISPOSABLE COMPUTE

Study the pattern:

```text
durable truth
+
disposable warm compute
+
notification as optimization
```

Relevant systems/ideas:

```text
Cursor Continuity
Deno celld
Cloudflare Durable Objects
```

Questions:

```text
what is authoritative?
what is cache?
how is ownership acquired?
what if notification is lost?
how does a cold actor prove freshness?
```

Do not implement R2/S3 graph durability before the local graph proves itself.

---

# 21. A-TIER — TEMPORAL / DURABLE EXECUTION

Temporal makes workflows durable via history/replay and therefore imposes determinism constraints.

Sources:

- https://docs.temporal.io/workflows
- https://docs.temporal.io/encyclopedia/event-history/event-history-typescript

Do not accidentally assume “graph survives crash” means “replay arbitrary `computed()` JavaScript.”

Separate:

```text
reactive recomputation
```

from:

```text
durable workflow execution
```

Maybe a graph restarts by re-reading current sources, not replaying old computation.

---

# 22. SIDE EFFECTS ARE A LAND MINE

Attack this:

```ts
computed(async () => {
  const user = await db.user(...)
  await sendEmail(user.email)
  return user
})
```

Ask:

```text
what if invalidated twice?
what if runtime retries?
what if fiber interrupts after email sends?
what if process crashes after effect but before commit?
what if computation is shared?
what if subscriber reconnects?
```

There is no magic answer.

Likely rule to challenge:

> `computed()` is repeatable read/derivation logic, not arbitrary side-effecting work.

External reads may still be nondeterministic, but external writes need explicit action semantics.

---

# 23. NODE IDENTITY MAY BE HARDER THAN THE GRAPH

What is the identity of:

```ts
project(42);
```

?

Potential key:

```text
code/computation identity
+ args
+ tenant
+ auth scope
+ execution zone
+ deployment/schema version?
```

Now redeploy. Is the old node “the same” node?

Research how DICE, Salsa, RSC module references, durable workflows, and caches version code identity.

Questions:

```text
stable build-time exported computation IDs?
HMR?
rolling deploy with two code versions?
old client protocol vs new server protocol?
cache reuse across deploy?
```

V0 can use a blunt rule like deployment mismatch → reconnect/rebuild, but the assumption must be explicit.

---

# 24. VALUE EQUALITY MAY BE THE SECRET WE ARE UNDERRATING

DICE, Skyframe, Salsa, and Jane Street Incremental all point toward:

```text
dirty input
≠
changed output
```

Suppose table-level invalidation reruns:

```text
tasksForProject(42)
```

because project 91 changed.

If result is equal, propagation can stop.

This may drastically reduce the need for heroic row/predicate tracking.

## But equality costs money

Compare:

```text
reference equality
shallow equality
deep equality
stable hash
structural sharing
adapter-defined equality
source version
```

Benchmark large results.

---

# 25. COARSE INVALIDATION + EARLY CUTOFF MAY BE THE 80/20

Explicit experiment:

### A

```text
table invalidation
+
rerun relevant query
+
result equality cutoff
+
fine-grained computed graph
```

### B

```text
row/predicate-level dependency tracking
```

### C

```text
rerun whole live route + snapshot
```

Maybe A wins on complexity/performance tradeoff. Do not assume more granularity is automatically better.

---

# 26. CYCLES

What happens if:

```text
A depends on B
B depends on A
```

Possible policies:

```text
reject
fixed point
last-value semantics
event-loop semantics
```

V0 probably rejects with an excellent dependency path. Study Salsa/dataflow cycle handling before long-term commitments.

---

# 27. TIME IS A SOURCE

This is subtly wrong:

```ts
const overdue = computed(() => tasks().filter((t) => t.dueAt < Date.now()));
```

Nothing invalidates when time advances.

Time must be reactive:

```ts
const now = clock.every("minute");

const overdue = computed(() => tasks().filter((t) => t.dueAt < now()));
```

Same issue for:

```text
randomness
environment variables
filesystem
feature flags
process globals
```

This is a user-model problem, not just implementation detail. Build systems learned this through hermeticity.

---

# 28. AUTHORIZATION IS A REACTIVE DEPENDENCY

Suppose:

```ts
const project = computed(async () => {
  assertCanRead(currentUser(), projectId());
  return db.project(projectId());
});
```

Then membership changes.

What invalidates the existing subscription?

Auth state itself may need to be part of the reactive graph:

```text
membership
role
permission set
API token validity
```

Attack cases:

```text
user loses workspace membership while socket stays open
role changes
external IdP revokes access
shared computation changes from safe to unsafe
reconnect with different auth
```

Security correctness beats cache hit rate.

---

# 29. BACKPRESSURE

What if:

```text
100k source changes/sec
      ↓
slow computation
      ↓
mobile client
```

Policies differ:

```text
queue every version
coalesce to latest
drop intermediate states
snapshot periodically
apply deltas
disconnect slow consumer
```

Reactive **state** and event **logs** are different.

Presence probably wants latest-value semantics. Audit events may require every event.

Effect Stream can provide mechanics. We must define semantics.

---

# 30. STATE VS EVENTS

State asks:

```text
what is true now?
```

Events ask:

```text
what happened?
```

Do not force reliable event processing into the reactive-value graph accidentally.

If `payment.failed` must never disappear, that belongs to durable event/workflow semantics, not merely “a signal changed.”

---

# 31. TRANSPORT COALESCING

Fine-grained graph semantics must not imply fine-grained network chatter.

Required benchmark:

```text
page observes 20 remote computations
```

Compare:

```text
20 subscriptions / roundtrips
```

versus:

```text
one connection
batched observation frame
server resolves graph internally
```

Potential invariant:

> **Fine-grained composition. Coalesced transport.**

---

# 32. SNAPSHOTS VS PATCHES VS STABLE GRAPH NODE UPDATES

Three possible wire models:

### Snapshot

```text
rerun dashboard
send dashboard
```

### Structural patch

```text
rerun dashboard
diff old/new
send patch
```

### Stable node values

```text
dashboard references $tasks $billing $deploy
send node version updates
```

Do not pick the fanciest. Benchmark payload size, compression, CPU, reconnect complexity, and implementation cost.

---

# 33. RECONNECT IS PART OF SEMANTICS

Client disconnects at v41. Server reaches v57.

Options:

```text
fresh snapshot
replay 42..57
resume cursor
rebuild graph references
```

V0 can likely snapshot. Do not accidentally promise replay or exactly-once delivery.

---

# 34. DEPLOYMENT / HMR

A live graph exists. New code deploys.

Questions:

```text
old client + new server?
old and new server versions simultaneously?
node IDs stable?
serialized shape changed?
dependency function changed?
```

A perfectly acceptable v0 policy may be:

```text
deployment/protocol version mismatch
→ reconnect
→ rebuild subscriptions from scratch
```

But write it down.

---

# 35. OBSERVABILITY MUST RECORD CAUSALITY

Graph tooling should eventually answer:

```text
why did X rerun?
which source invalidated it?
which edge propagated?
did output actually change?
how long did it take?
was work shared?
how many subscribers?
how many bytes published?
was run cancelled?
was invalidation coalesced?
which auth scope owns it?
```

An implicit system without causal inspection will be unbearable.

---

# 36. AGENT-NATIVE GRAPH INSPECTION

Machine-readable commands could eventually include:

```text
explain node dashboard(42)
explain invalidation #9182
show hottest nodes
show fanout > 1000
show nodes with frequent unchanged recomputation
show coarse source edges
show leaked/unobserved nodes
```

This is a serious part of the agent-native thesis.

---

# 37. REQUIRED ADVERSARIAL BENCHMARKS

## Benchmark 1 — coarse route vs graph

Build board three ways:

```text
A. touch route + full rerun + snapshot
B. table invalidation + computed graph + equality cutoff
C. finest practical row/query invalidation
```

Measure:

```text
LOC
DB queries
CPU
memory
network bytes
latency
implementation complexity
```

## Benchmark 2 — broad invalidation

10,000 projects. Change one task. See how much lazy observation + equality cutoff saves under table-level invalidation.

## Benchmark 3 — 10,000 subscribers

Shared safe computation should execute once and fan out. Then repeat with unique auth scopes to expose worst case.

## Benchmark 4 — dynamic dependencies

Randomly switch:

```ts
computed(() => (flag() ? A() : B()));
```

Old dependency must stop invalidating.

## Benchmark 5 — async races

```text
run #1 starts
invalidate
run #2 starts
#2 finishes
#1 finishes
```

Older run must not overwrite newer committed state.

## Benchmark 6 — invalidation while running

Hammer a node while waiting on slow I/O. Final visible state must correspond to latest relevant source state.

## Benchmark 7 — failure during dependency discovery

Read A and B, then throw. Last committed graph must remain valid.

## Benchmark 8 — dropped broadcast

Drop distributed notification. Does correctness recover eventually? If not, notification is authoritative and must be durable.

## Benchmark 9 — crash after DB commit

Crash between DB commit and app-level broadcast. Validate CDC/outbox recovery story.

## Benchmark 10 — reconnect

Disconnect during rapid writes, reconnect, compare with a clean fresh read.

## Benchmark 11 — auth revocation

Keep subscription open, revoke permission, ensure no further protected data flows and no shared cache leak occurs.

## Benchmark 12 — 20 remote nodes

Ensure elegant API does not create 20 network waterfalls.

## Benchmark 13 — large result equality

Compare deep equality, hashing, structural sharing, and just resending snapshots.

## Benchmark 14 — hot node

One source changes 10,000 times/sec. Verify scheduler coalescing/backpressure semantics.

## Benchmark 15 — agent coding task

Give agents identical realtime app tasks under:

```text
traditional query-cache + socket architecture
vs
our runtime
```

Measure:

```text
tool calls
tokens
files touched
LOC
bugs
test failures
time
manual synchronization concepts
```

---

# 38. FUZZ / PROPERTY TEST ORACLE

Build a deliberately stupid reference runtime:

```text
on any relevant source change
throw away all derived observed state
recompute from scratch
```

Fuzz:

```text
source writes
dynamic branches
random awaits
errors
subscription churn
transactions
```

Compare optimized visible state to the reference. This should be inspired by from-scratch consistency in self-adjusting computation research.

---

# 39. THINGS WE PROBABLY SHOULD NOT BUILD IN V0

Until evidence requires them:

```text
custom database
CRDT engine
offline replica
incremental SQL engine
distributed global graph
R2 WAL
multi-region consensus
custom actor runtime
custom RPC type system
custom stream library
custom structured-concurrency runtime
custom workflow engine
React binding
Svelte binding
hosted cloud platform
billing
```

Use other people's decades.

---

# 40. PARALLEL RESEARCH AGENT ASSIGNMENTS

## Agent A — Incremental Computation

Study:

```text
DICE
Skyframe
Salsa
Adapton
Jane Street Incremental
Meteor Tracker
```

Return:

```text
common invariant
key/identity model
revision model
equality/cutoff model
dynamic dependency algorithm
lifecycle model
cycle model
what we are naively reinventing
```

## Agent B — Distributed Reactive Semantics

Study:

```text
Distributed REScala
DREAM
cost of consistency
fault-tolerant DRP
thread-safe RP
Gavial
```

Return:

```text
definition of glitch
causal vs glitch-free vs atomic
minimum coordination for each
failure behavior
network reorder behavior
which semantics are feasible for v0
which of our slogans are dishonest
```

## Agent C — Reactive Databases / Dataflow

Study:

```text
Noria
DBSP
Differential Dataflow
Timely
Materialize
TanStack DB / d2ts
Zero
Electric
```

Return:

```text
what each solves
where source invalidation ends
where query IVM begins
whether d2ts could be an adapter
memory/cost tradeoffs
partial materialization
shared computation
what not to build ourselves
```

## Agent D — Runtime Kernel

Study Effect v4:

```text
Fiber
Scope
Stream
Cause
Reactivity
Atom
AtomRegistry
RPC
Schema
SQL
Cluster/Entity
```

Return:

```text
machinery to delegate
semantics we must own
unstable APIs not to couple to
Promise-wrapping cost
cancellation behavior
Scope ↔ node lifetime mapping
Stream ↔ remote value mapping
```

## Agent E — Cross-Machine Programming Model

Study:

```text
Dan's RSC essays
Links
Eliom
Ur/Web
Hop
Gavial
```

Return:

```text
boundary representation
code colocation
serialization typing
roundtrip coalescing
why tierless programming didn't dominate
how to feel local without lying about network physics
```

## Agent F — Physical Distribution / Durability

Study:

```text
Cloudflare Durable Objects
Orleans
Restate
celld
Cursor Continuity
Temporal
```

Return:

```text
logical identity vs active process
activation/passivation
partition granularity
single-writer semantics
recovery
durable truth
notification vs correctness
why one actor per node is bad
```

## Agent G — Postgres Source Adapter

Study:

```text
ORM instrumentation
LISTEN/NOTIFY
triggers
logical decoding
replication slots
Debezium
transactional outbox
```

Return:

```text
smallest v0 for in-app writes
smallest design for arbitrary external writes
transaction boundaries
ordering
failure modes
operational burden
exactly-once myths
recommended adapter levels
```

## Agent H — Security / Multi-Tenancy

Attack:

```text
shared computation keys
auth revocation
permission changes
tenant isolation
cache poisoning
cross-user result reuse
reconnect with changed auth
```

Return exploit scenarios, not happy paths.

## Agent I — Protocol

Compare:

```text
full snapshots
JSON patches
stable graph-node updates
RSC-style outlined/progressive serialization
SSE
WebSocket
HTTP streaming
```

Return:

```text
initial load
live updates
reconnect
backpressure
batching
versioning
bandwidth
CPU
complexity
```

No protocol design without evidence.

---

# 41. REQUIRED REPORT FORMAT FOR EVERY AGENT

Every agent report must contain:

```text
1. SYSTEM / PAPER
2. WHAT PROBLEM IT ACTUALLY SOLVES
3. CORE SEMANTICS
4. KEY INVARIANTS
5. ASSUMPTIONS THAT ENABLE THOSE INVARIANTS
6. WHAT WE ARE CURRENTLY REINVENTING
7. WHAT WE SHOULD STEAL
8. WHAT WE MUST NOT COPY
9. WHERE OUR VISION IS STRICTLY HARDER
10. WHERE OUR VISION IS UNNECESSARILY HARDER
11. FAILURE MODES
12. PERFORMANCE COSTS
13. APIS WORTH STUDYING
14. TESTS / BENCHMARKS WORTH COPYING
15. DOES THIS CHANGE PROJECT SOUL?
16. DOES THIS KILL ANY PART OF THE PROJECT?
17. OPEN QUESTIONS
```

Do not submit generic summaries. We want architectural consequences.

---

# 42. EVIDENCE DISCIPLINE

Treat every statement as one of:

```text
FACT
INFERENCE
HYPOTHESIS
UNKNOWN
```

Especially for unreleased Iris.

Do not silently convert our guesses into facts.

---

# 43. DECISION GATES BEFORE A REAL IMPLEMENTATION PLAN

## Gate 1 — Local graph algorithm

What minimum algorithm survives comparison with DICE/Salsa/Skyframe/Adapton?

## Gate 2 — `computed()` semantics

Define:

```text
repeatability
external reads
side effects
errors
retry
cancellation
time
randomness
```

## Gate 3 — Commit semantics

What does one committed graph update mean in:

```text
one process
one source transaction
multiple external sources
```

## Gate 4 — Node identity

Include auth and deployment/version assumptions.

## Gate 5 — Postgres invalidation

ORM-only? NOTIFY? CDC? Outbox?

## Gate 6 — Effect boundary

What does Effect own? What semantics remain ours?

## Gate 7 — Simplest wire protocol

Start boring unless evidence demands graph protocol machinery.

## Gate 8 — Benchmark against coarse live route

If graph cannot beat the simple version in meaningful application complexity or efficiency, reconsider.

---

# 44. CURRENT LIKELY SHAPE — AUTHORIZED TO BE DESTROYED

```text
                  USER API
             boring TypeScript

       computed / mutation / action
          channel / remote

                   │
                   ▼

             OUR SEMANTICS

       logical dependency graph
       node identity + revisions
       equality / early cutoff
       observation / lifecycle
       transaction commit model
       source dependency model
       remote edge semantics

                   │
                   ▼

              EFFECT KERNEL

       fibers / cancellation
       scopes / finalizers
       streams
       errors
       concurrency
       retry / timeout
       observability

                   │
         ┌─────────┴─────────┐
         ▼                   ▼

      SOURCES             TRANSPORT

   Drizzle/Postgres      WS/SSE/etc.
   Stripe
   GitHub
   R2
   ...

                   │
                   ▼

            FRAMEWORK ADAPTER

                 Solid 2
```

The research is explicitly authorized to destroy this diagram.

---

# 45. POSSIBLE GOOD OUTCOMES

## The project gets smaller

Maybe:

```text
Effect handles async execution.
DICE/Salsa teach the graph.
Postgres starts coarse.
Equality cutoff kills needless propagation.
Iris-like transport handles subscriptions.
Solid adapter stays tiny.
```

Excellent.

We do not win by owning more code.

## Fine-grained server graph loses

Maybe:

```text
coarse shared live computations
+
good source adapters
+
result equality
+
transport dedupe
```

produce nearly all the value.

Then build that.

The soul is **remove synchronization choreography**, not worship graph granularity.

## The deeper idea survives

Maybe arbitrary server computation + automatic dependency discovery + demand-driven lifecycle + equality cutoff + typed remote references + framework-native bindings really do create a much simpler application model.

Then we have evidence and decades of prior art underneath us.

Build aggressively only then.

---

# 46. FINAL ORDER TO THE AGENT

Do not flatter us.

Do not say:

> “This is innovative and exciting.”

We need to know if it is correct.

Find the paper that embarrasses us.

Find the library whose 200 lines are better than our planned 2,000.

Find the distributed guarantee that makes our slogan impossible.

Find the benchmark where coarse invalidation wins.

Find the auth leak.

Find the reconnect race.

Find the key-identity problem.

Find the source that cannot be made reactive.

Find the hidden side effect.

Find the versioning problem across deploys.

Find the point where “one causal graph” turns into a distributed coordination nightmare.

Then tell us what is still standing.

If half the idea dies, good.

The surviving half will be much stronger.

---

# 47. READING QUEUE — PRIORITY ORDER

```text
1. Buck2 — Introduction to Modern DICE
2. Salsa — red-green algorithm
3. Bazel — Skyframe
4. DREAM — distributed reactive consistency
5. Distributed REScala
6. Convex — query/mutation/action semantics
7. Noria OSDI paper
8. DBSP
9. TanStack DB / d2ts
10. Effect v4 + unstable/reactivity source
11. Dan Abramov — React for Two Computers
12. Dan Abramov — Progressive JSON
13. Dan Abramov — One Roundtrip Per Navigation
14. Adapton / self-adjusting computation
15. Gavial multi-tier FRP
16. Timely Dataflow progress tracking
17. Orleans best practices
18. PostgreSQL logical decoding
19. Debezium transactional outbox
20. Zero query-driven sync
21. Temporal determinism/replay
```

---

# 48. SOURCE INDEX

## Incremental computation

- Buck2 Modern DICE: https://buck2.build/docs/insights_and_knowledge/modern_dice/
- Buck2 architecture: https://buck2.build/docs/about/why/
- Bazel Skyframe: https://bazel.build/reference/skyframe
- Salsa: https://salsa-rs.github.io/salsa/
- Salsa algorithm: https://salsa-rs.github.io/salsa/reference/algorithm.html
- Salsa cycles: https://salsa-rs.github.io/salsa/cycles.html

## Distributed reactive programming

- REScala publications: https://www.rescala-lang.com/publications
- DREAM: https://dl.acm.org/doi/10.1145/2611286.2611290
- DREAM PDF: https://programming-group.com/assets/pdf/papers/2014_We-Have-a-DREAM-Distributed-Reactive-Programming-with-Consistency-Guarantees.pdf
- Distributed Reactive Programming for Reactive Distributed Systems: https://arxiv.org/abs/1902.00524
- Fault-tolerant DRP: https://drops.dagstuhl.de/storage/00lipics/lipics-vol109-ecoop2018/LIPIcs.ECOOP.2018.1/LIPIcs.ECOOP.2018.1.pdf
- Consistent Distributed Reactive Programming with Retroactive Computation: https://arxiv.org/abs/2502.20534

## Dataflow / incremental databases

- Noria: https://www.usenix.org/conference/osdi18/presentation/gjengset
- DBSP: https://arxiv.org/abs/2203.16684
- TanStack DB: https://tanstack.com/db/latest/docs/overview
- Timely Dataflow: https://timelydataflow.github.io/timely-dataflow/
- Timely progress tracking: https://timelydataflow.github.io/timely-dataflow/chapter_5/chapter_5_2.html
- Differential Dataflow introduction: https://materialize.com/blog/differential-from-scratch/

## Reactive app / sync systems

- Convex queries: https://docs.convex.dev/functions/query-functions
- Convex mutations: https://docs.convex.dev/functions/mutation-functions
- Convex actions: https://docs.convex.dev/functions/actions
- Zero: https://zero.rocicorp.dev/

## Runtime

- Effect: https://effect.website/
- Effect repository: https://github.com/Effect-TS/effect

## Cross-machine / tierless programming

- React for Two Computers: https://overreacted.io/react-for-two-computers/
- Progressive JSON: https://overreacted.io/progressive-json/
- One Roundtrip Per Navigation: https://overreacted.io/one-roundtrip-per-navigation/
- JSX Over The Wire: https://overreacted.io/jsx-over-the-wire/
- Impossible Components: https://overreacted.io/impossible-components/
- Links: https://links-lang.org/
- Eliom: https://ocsigen.org/tuto/latest/application.html
- Ur/Web: https://www.impredicative.com/ur/
- Gavial: https://arxiv.org/abs/2002.06188

## Physical distribution / durability

- Orleans: https://learn.microsoft.com/en-us/dotnet/orleans/overview
- Orleans best practices: https://learn.microsoft.com/en-us/dotnet/orleans/resources/best-practices
- Orleans activation collection: https://learn.microsoft.com/en-us/dotnet/orleans/host/configuration-guide/activation-collection
- Restate services: https://docs.restate.dev/foundations/services
- Cloudflare Durable Object lifecycle: https://developers.cloudflare.com/durable-objects/concepts/durable-object-lifecycle/
- Temporal workflows: https://docs.temporal.io/workflows

## PostgreSQL change propagation

- Logical decoding: https://www.postgresql.org/docs/current/logicaldecoding.html
- Logical decoding output plugins: https://www.postgresql.org/docs/current/logicaldecoding-output-plugin.html
- NOTIFY: https://www.postgresql.org/docs/current/sql-notify.html
- Debezium outbox: https://debezium.io/documentation/reference/stable/transformations/outbox-event-router.html
