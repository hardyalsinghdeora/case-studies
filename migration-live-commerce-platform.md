# Moving a live commerce platform between two databases without a maintenance window

*A decision record. Roughly four years of a legacy estate, six product lines, and two databases that both had to keep serving while the data moved between them.*

---

## The system, briefly

A B2B commerce platform: institutional customers buy annual, seat-based contracts. An order has to become a quotation, an approved price, an invoice, an activation, and finally a working account for each seat. Six product lines sold through the same motion, each with its own pricing shape.

The original system was built over roughly fifteen years on MySQL. The replacement was a set of domain services on PostgreSQL. Both had live customers throughout.

I designed the migration and wrote the specifications the team executed against — around 25,000 words across seven documents. This is what I would tell another engineer facing the same problem.

---

## 1. The constraint that shaped everything

**There was no maintenance window, and there was never going to be one.**

Not "we'd prefer to avoid downtime." The customers were institutions mid-contract; their users were active on any given weekday, and the annual sales cycle meant there was no quiet fortnight anywhere in the calendar. A weekend of downtime was not a cost to be negotiated — it was a thing the business could not sell around.

That single constraint eliminated most of the standard playbook before design started. It is worth being precise about *why*, because the constraint is what makes the rest of the decisions legible.

The second constraint was subtler and mattered more. The legacy products were not being retired. They would keep running, keep serving, and keep *writing* for a long time — years, not months. So this was never "migrate and switch off." It was "two systems of record, both live, indefinitely, converging slowly." Any design that assumed an end date was solving a different problem.

---

## 2. What we rejected, and why

**Freeze, dump, restore.** The obvious approach, and the one everyone suggests first. Rejected on the constraint above: the dump alone would have exceeded any window we could obtain, and a failed restore leaves you with no way back that doesn't extend the outage further. The failure mode is unbounded, which is the disqualifying property.

**Big-bang cutover behind a flag.** More sophisticated, still wrong. It concentrates all risk into a single moment, and the rollback path is the part nobody rehearses. The deeper problem: with six product lines and a dozen services, "cut over" isn't one event. Something would have been half-migrated at the moment of truth, and the recovery procedure for a partial cutover is not a procedure — it's an incident.

**Strangler at the edge only.** Route new traffic to the new system, leave old data behind, let the legacy shrink by attrition. Genuinely attractive, and we borrowed from it. But it fails when the legacy products keep reading the *shared* records — a user's identity and subscription had to be visible to both worlds on the same day. Attrition doesn't help when both systems need the same row now.

**Dual-write from the application layer.** Have the service write to both databases in the same transaction. Rejected because it isn't one transaction — it's two, across two engines, and the moment they diverge you've built a distributed-consistency problem into your application code and given every developer the opportunity to forget one of the writes. Correctness that depends on discipline is not correctness.

What survived from each: the strangler's domain-by-domain incrementalism, and dual-write's recognition that both sides need current data. The design below is those two ideas with the failure modes engineered out.

---

## 3. The shape we chose: two flows, kept rigorously separate

The single most useful decision in the entire project was recognising that "migration" was **two different problems** that teams routinely conflate — and building them as separate mechanisms with separate code, separate schedules, and separate failure modes.

### Flow one — the historical backfill (legacy → new, one-time)

A bulk port of historical records into the new schema. Read from a read-only snapshot, never from the live primary. Bounded by a fixed historical cut-off, because records beyond a certain age had no operational purpose and porting them would have multiplied the volume, the runtime, and the reconciliation surface for no benefit. That cut-off was an argument worth having; "port everything" is a default, not a decision.

Critically: **designed to be re-runnable.** Not "we hope to run it once." A backfill you can safely run again is a backfill you can afford to get wrong, and we did get it wrong, more than once. Re-runnability is what converts a terrifying one-shot event into an ordinary operation.

### Flow two — the continuous reverse sync (new → legacy, ongoing)

Every write that landed in the new system propagated back to the legacy tables, so the legacy products kept reading the shared records and kept serving. This is the flow people skip, and skipping it is why migrations stall: teams build the forward path, discover the legacy still needs to be current, and bolt on something reactive under time pressure.

It runs in the opposite direction from the migration itself, which is exactly why it needs to be designed rather than discovered. The new system is the system of record; the legacy is a projection of it. Naming that relationship explicitly settled a dozen smaller arguments before they happened.

**These two must not share code.** They have different triggers, volumes, latency requirements, idempotency semantics and failure responses. Unifying them produces a component that is wrong for both.

---

## 4. The hard part: two schemas with incompatible worldviews

This is where the interesting work was, and it wasn't the data movement.

The legacy schema was **product-partitioned**. Each product line had its own family of tables — its own order master, its own seat breakdown, its own rate card. Six products, five table families each. Adding a product meant adding tables, and every query that spanned products was a union written by hand.

The new schema was **generic**: one order table, one line-item table, one item-detail table, one configuration table. Products became data rather than schema.

That is the right target design, and the transformation is genuinely hard: roughly thirty product-specific tables collapsing into four generic ones, with the product-specific meaning that used to live in *table names* now having to live in *columns and a JSON configuration*.

Three things I'd underline for anyone doing this:

**The derivation of the configuration column deserved its own specification.** One JSON column, holding what had previously been expressed by which table a row lived in. Getting it right required going product by product through what each legacy shape actually meant — and discovering, repeatedly, that two products used a similarly-named column for genuinely different things. It took over a thousand words to specify one column, and that was the correct amount of effort. When the schema stops encoding meaning, something else has to, and that something must be written down.

**Ordering constraints are prerequisites, not preferences.** Organisations had to migrate before orders; orders before users and subscriptions. Not a nice-to-have sequence — a hard dependency, because the later entities carry foreign keys to the earlier ones. We stated these as blocking preconditions in the specification, and the runner enforced them. Every migration has this structure and it is usually discovered at 2am rather than declared in advance.

**Idempotency guards belong on the record, not in the runbook.** Records could arrive by more than one path — the migration, or an existing integration with the CRM. Porting a record that had already arrived by the other route would silently duplicate it. The guard was simple: port an order only when its external-origin identifier was null. One condition, in the query, permanently. The alternative — "remember not to run it twice" — is not an engineering control.

---

## 5. Reconciliation, and the thing I got wrong

Here is the honest part.

**We built reconciliation in response to the first drift we found, not before it.**

The design was sound. Both flows worked. And then a comparison turned up rows that disagreed between the two systems — not many, but not zero, and the cause was not obvious. What followed was the expensive part of the project: manual comparison, by hand, across two schemas whose shapes did not line up, to work out which side was right and why they had diverged.

The technical fix was straightforward once we understood it. The process lesson is the one worth carrying:

> **In any dual-write system, drift is not a risk. It is a certainty with an unknown arrival time.** The reconciliation harness is not a safety net you add if there's budget. It is part of the mechanism, and it must exist before the first byte moves — because its real job is not fixing drift, it is *telling you drift happened* while the cause is still recent enough to diagnose.

We were measuring correctness at the end. We should have been measuring it continuously, from the first day the second write existed. The difference between those two postures is the difference between a bug and an investigation.

---

## 6. What it cost

- **Timeline:** under six months from the first specification to steady state
- **Records moved:** over half a million in the historical backfill — organisations, orders, users and their product subscriptions, consolidated from two upstream systems of record
- **Scope:** six product lines, roughly thirty legacy tables → four generic ones
- **Specification:** ~25,000 words across seven documents
- **Team:** a platform team; I designed and specified, the team executed
- **Incidents:** none customer-visible. The drift in §5 was found by our own comparison and resolved before any customer saw it
- **Rollbacks exercised:** none. Every phase was built to be reversed and none had to be; what actually absorbed the mistakes was the re-runnable backfill in §3, which turned out to be the cheaper property to have

---

## 7. What I'd do differently

**Instrument reconciliation from day one.** The lesson above. Before the first write, not after the first surprise.

**Write the configuration derivation before the transfer mechanism.** We built the movement, then discovered the semantic collisions. The semantics are the hard part and they should set the pace — the plumbing is comparatively easy and gets easier once you know precisely what you're moving.

**State the cut-off decision earlier and louder.** "How far back do we port?" was treated as a detail and turned out to shape volume, runtime, and reconciliation surface. It's a scoping decision with architectural consequences, and it belongs on page one.

**Push harder on the two-flows separation, sooner.** It took a while for it to become the organising idea rather than an implementation detail. Once it was explicit, a whole category of confused conversation stopped — people knew which mechanism they were talking about. Naming things is not documentation overhead; it's what stops two engineers solving different problems in the same meeting.

---

## What generalises

Strip away the specifics and four things transfer to any live migration:

1. **Separate the one-time historical port from the continuous sync.** Different problems, different code, different failure modes. Conflating them is the single most common way these projects stall.
2. **Make every phase independently reversible.** The one thing you cannot promise is that nothing will go wrong. You can promise that anything that goes wrong can be undone.
3. **Put idempotency in the data, not in the process.** A guard condition in a query outlives every runbook and every person who remembers the rule.
4. **Reconciliation is part of the mechanism, not a phase after it.** If you cannot continuously answer "do these two systems agree," you do not have a migration — you have a hope.

The specification was the deliverable, not the code. Around 25,000 words, precise enough that someone other than the author could execute the cutover — which is the actual test of whether a migration is designed or merely intended.

---

*Names, schemas, figures and product details are omitted deliberately: the architecture and the trade-offs are mine to discuss, the company's data is not. Everything above describes a class of problem I have solved and would solve again — happily in more detail, in conversation.*
