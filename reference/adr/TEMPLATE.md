# ADR-NNN: <short decision title, written as the decision, not the topic>

- **Status:** proposed | accepted | superseded by ADR-NNN | deprecated
- **Date:** YYYY-MM-DD
- **Deciders:** names
- **Consulted / informed:** names or teams
- **Affects:** services, teams, or systems

## Context and problem

What forced a decision *now*. The constraints that were real at the time: scale, deadline,
team size, existing systems, compliance, budget. Two or three paragraphs, no more.

Write this so that a stranger in 18 months understands why the obvious-in-hindsight option was
not obvious. That is the entire value of the document.

## Decision drivers

- Driver 1 (e.g. must survive a single-AZ failure without data loss)
- Driver 2 (e.g. team of four, no dedicated platform engineer)
- Driver 3 (e.g. p99 under 200 ms for the checkout path)

## Considered options

1. **Option A** — one-line description
2. **Option B** — one-line description
3. **Option C (do nothing)** — always list it

### Option A
- Pros: …
- Cons: …
- Cost / effort: …

### Option B
- Pros: …
- Cons: …
- Cost / effort: …

> Each rejected option must list at least one genuine advantage. If every alternative is
> obviously bad, you have written a justification, not a decision record.

## Decision

We chose **Option X** because …

State it in one or two sentences. Name the driver that decided it.

## Consequences

**Positive**
- …

**Negative — what we are accepting**
- …
- …

**Neutral / follow-on work**
- …

> The negative section is mandatory and must be non-empty. A decision with no downside was not
> a decision.

## Revisit trigger

The specific, observable condition that should make someone reopen this:
e.g. "if write throughput exceeds 5k/s", "if a second team needs to own this data",
"if the managed service adds feature Y", "review by 2027-06".

## References

- Links to the design doc, benchmark, incident, or prior ADR that informed this.
