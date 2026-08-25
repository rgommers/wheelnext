# Wheel Variants — Requirements

This section of wheelnext.dev documents the **requirements on the design** underlying the wheel variants PEP series — what the design must achieve, not implementation requirements for tools (those live in the PEPs). It is a companion to the PEPs, not a replacement for them.

If you are landing here for the first time, the next three sections should orient you in about five minutes.

---

## What this is, in one minute

Wheel Variants is the largest change to Python's binary distribution format in many years. It spans data model, plugin trust, dependency resolution, user experience, security, build tooling, indexing, and migration. Because of that scope, the design has been split across a sequence of PEPs, under an informational umbrella:

- **PEP 817** — Wheel Variants: Beyond Platform Tags (informational umbrella for the series)
- **PEP 825** — Wheel Variants: Package Format (the data model)
- a providers PEP (the plugin model that decides variant compatibility)
- a UX / security / maintainability PEP
- a building PEP (build backend interface)

The Standards Track PEPs are the **normative** specification. They are where decisions are recorded, and where the Packaging Council and the Steering Council will issue rulings.

This section is the **requirements layer**: an explicit, addressable record of what the design has to do and why, separate from how each PEP does it. For the handful of load-bearing design choices where the requirements alone cannot make the rationale legible, this section also hosts **options analyses** that work out the alternatives in detail. Both are companion artefacts to the PEPs, not replacements for them, and both exist to support the PEP process, not to route around it.

---

## Why a requirements layer exists

The PEP format was designed for relatively contained changes — one document, one mechanism, one decision. The wheel variants work has a different shape: a multi-component system where the same requirement (say, "the user must be able to override automatic selection") touches three or four PEPs, and any single PEP needs to satisfy a dozen requirements coming from different stakeholders.

In practice that has produced four recurring problems on the discussion threads:

1. **No shared reference frame.** When a reviewer raises a concern, there is no anchor to point to. The same concern surfaces in several threads, in slightly different forms, and the responses are re-derived each time.
2. **Alternatives considered are scattered.** The reasoning behind rejected designs lives in old Discourse posts, GitHub PR comments, and PEP revision history. New reviewers cannot reasonably be expected to reconstruct it.
3. **Stakeholders cannot find their slice.** A security reviewer, a build-backend maintainer, and a data scientist all need different entry points into the design. The PEPs are linear documents; they cannot serve all three.
4. **Cross-cutting concerns lose their home.** Override and pinning, for example, could legitimately live in three different PEPs. Without a requirements layer, the decision becomes a process argument rather than a design one.

A requirements register addresses all four. It is also a familiar artifact: the engineers and program managers funding this work at NVIDIA, Astral, Red Hat, Meta, Quansight, Intel, AMD, Huawei, and elsewhere recognize it from their own day jobs — where the distinction it embodies is called *performance requirements* (what the system must accomplish; nothing to do with speed) versus *design requirements* (how it is built, which here is the PEPs' territory).

---

## How it fits with the PEPs

```
   ┌──────────────────────────────────────────────────────┐
   │   System goals (SYS-*)                               │   ← why the project exists
   ├──────────────────────────────────┬───────────────────┤
   │   Subsystem requirements         │   Options         │   ← what the design must do,
   │   DM · PROV · RES · UX · OV ·    │   analyses        │     and the load-bearing
   │   SEC · BLD · IDX · MIG          │   (OA-*)          │     "why this option"
   ├──────────────────────────────────┴───────────────────┤
   │   PEPs                                                │   ← how it is specified
   │   825  ·  Providers  ·  UX/Sec  ·  Build             │     (normative)
   ├──────────────────────────────────────────────────────┤
   │   Reference implementation, prototypes                │   ← evidence it works
   └──────────────────────────────────────────────────────┘
```

The arrows are intentional. Requirements **trace upward** to system goals and **allocate downward** to PEPs. Options analyses sit at the same level as requirements, attached to specific decisions: each one cites the requirements it weighs and the PEP it informs. A change to a system goal forces a review of requirements; a change to a requirement forces a review of PEP allocation and of any options analyses that depend on it; a change to a PEP forces a check against the requirements and analyses it carries.

When a discussion in a PEP thread raises a new concern, the right response is often: file it as a requirement first, then decide which PEP it lands in. That ordering keeps the PEP review focused on the PEP, not on re-opening the scope.

---

## Options analyses

A small number of design choices in this project are too consequential to be settled by a one-line "Rejected Alternatives" note in a PEP. The provider mechanism, the metadata schema, the override design, and the trust model each shape the next decade of Python's binary distribution and each have several plausible alternatives that deserve serious treatment.

For these, this section hosts **options analyses**: bounded, structured documents that lay out the alternatives, weigh them against criteria drawn from the requirements, recommend one, and record dissent. Each analysis has a stable ID (`OA-001`, `OA-002`, …) so it can be cited from PEP threads and review comments.

We expect somewhere between four and six options analyses for the full PEP series. Medium-importance design choices live inside the requirements doc's *Alternatives Considered* fields; small ones stay in the PEPs' own *Rejected Alternatives* sections. The discipline matters: an options analysis is a deliberate, considered artefact, not the default container for every design question.

Like the requirements doc, options analyses are non-normative. They record the analysis that supports a decision; the decision itself is made through the PEP process.

---

## Where to start

Different readers want different doors into the design. Pick the one that fits.

- **End users and data scientists** — start with the [System Requirements](./requirements.md#top-level-system-requirements), then [UX](./requirements.md#ux-user-experience) and [Override & Pinning](./requirements.md#ov-override-pinning). You are looking for: what will change for you, and what escape hatches exist.
- **Package maintainers** — start with [Building](./requirements.md#bld-building), [Data Model](./requirements.md#dm-data-model), and [Variant Providers](./requirements.md#prov-variant-providers). You are looking for: what you will declare, in what file, and how it ends up in a wheel.
- **Installer and index maintainers** — start with [Resolution & Selection](./requirements.md#res-resolution-selection) and [Index Serving](./requirements.md#idx-index-serving). For the provider mechanism choice specifically, see [OA-001](./options-analyses/oa-001-variant-compatibility-mechanism.md).
- **Security reviewers** — start with [Security & Trust](./requirements.md#sec-security-trust), then the [Provider](./requirements.md#prov-variant-providers) trust model, and read [OA-001](./options-analyses/oa-001-variant-compatibility-mechanism.md) for the trust-surface analysis behind the provider choice.
- **Steering and Packaging Council members** — start with [System Requirements](./requirements.md#top-level-system-requirements) and the [PEP Allocation Matrix](./requirements.md#pep-allocation-matrix). For the headline design choice (provider mechanism), see [OA-001](./options-analyses/oa-001-variant-compatibility-mechanism.md).

The full requirements document: **[Requirements Register](./requirements.md)**.
The options analyses index: **[Options Analyses](./options-analyses/index.md)**.

---

## What this section is not

A few things worth being explicit about, because both the requirements doc and the options analyses could be misread otherwise.

- **They are not the spec.** The PEPs are. Where these documents and a PEP conflict, the PEP wins; please file an issue so we can reconcile.
- **They are not a decision venue.** Decisions happen in PEP threads on [discuss.python.org](https://discuss.python.org/c/packaging/14), in pull requests against [python/peps](https://github.com/python/peps), and by the relevant councils. This site records the requirements and analyses that those decisions are made against.
- **They are not an end-run around the PEP process.** Every requirement and every options analysis here is meant to support — and where possible, be cited from — the normal PEP review.
- **They are not a marketing site.** The audience is reviewers, implementers, and the councils. If this section ever starts reading like it is selling something, that is a bug — please file it.

---

## How it evolves

- **Source of truth:** this site is built from a markdown source tree in the [wheelnext.dev repository](https://github.com/wheelnext/wheelnext.dev). All changes go through pull request review by the WheelNext working group.
- **Requirement and OA IDs are permanent.** Once a requirement or options analysis has an ID (`PROV-007`, `OA-001`), it keeps that ID — even if the content is later deferred, rejected, or superseded. This is so external references (in PEPs, threads, lockfiles, postmortems) stay valid.
- **Status, not deletion.** A rejected requirement or a superseded analysis is marked accordingly with a rationale, rather than removed. The history of what was considered and why is part of what makes this useful to later reviewers.
- **Provenance is recorded.** When a requirement or analysis traces to a specific Discourse thread, issue, or stakeholder, the source is linked. New items without a clear source are still accepted, but the field is filled in as discussions surface.
- **Allocation can change.** Where a requirement lands in the PEP sequence is itself a decision that can be revisited; see the [Open Allocation Questions](./requirements.md#open-allocation-questions) section. Options analyses can likewise be revisited if the underlying requirements change.

---

## Status of the PEP sequence

| PEP                              | Scope                                          | Status                             |
|----------------------------------|------------------------------------------------|------------------------------------|
| [PEP 817](https://peps.python.org/pep-0817/) | Informational umbrella for the series          | Under discussion                   |
| [PEP 825](https://peps.python.org/pep-0825/) | Wheel Variants: Package Format (data model)    | Under discussion                   |
| Providers (draft)                | Provider plugin model, trust, discovery        | In drafting                        |
| UX / Security / Maintainability (draft) | User-facing behavior, audit, configuration  | In drafting                        |
| Building (draft)                 | Build backend interface                        | In drafting                        |

Reference implementations, prototype installers, and provider examples live in the [wheelnext GitHub organization](https://github.com/wheelnext).

---

## Status of options analyses

| ID                                                          | Topic                                       | Decision target  | Status |
|-------------------------------------------------------------|---------------------------------------------|------------------|--------|
| [OA-001](./options-analyses/oa-001-variant-compatibility-mechanism.md) | Variant compatibility mechanism (providers vs alternatives) | Providers PEP    | Draft  |

Further analyses will be added as load-bearing decisions are identified. The expected scale of this list across the full PEP series is four to six entries.

---

## Frequently asked

**Is this section the specification?**
No. The PEPs are normative. This section is a companion artifact that records the requirements the PEPs are designed against — requirements *on* the design, not requirements *in* the design.

**The requirements say "shall" and "must" — are tools required to follow them?**
No. Nothing on this site binds any tool. The "shall" in a requirement's Statement addresses the design effort — it is the working group holding itself accountable for an outcome — and the `must`/`should`/`may` in the Priority field ranks how critical a requirement is, deliberately *not* in the RFC 2119 sense. When a requirement is realised, the corresponding normative MUST for implementers appears in a PEP, and only there.

**Where do I discuss a requirement I disagree with?**
On the relevant Discourse thread, linked from the requirement's *Source* field. If the requirement does not yet have a thread, open an issue in [python/peps](https://github.com/python/peps) on the relevant PEP, or in the WheelNext repo for cross-cutting concerns. Discussion on this site happens via pull request review; substantive design debate happens on Discourse.

**Why not put this material inside the PEPs themselves?**
A single requirement often touches more than one PEP. Putting it in one PEP either duplicates it across the others (creating a synchronization problem) or makes the other PEPs ambiguous about whether they own it. A separate requirements layer lets each PEP focus on its mechanism and reference the shared requirements by ID.

**Does this slow down the PEP process?**
The intent is the opposite. By giving discussions a shared reference frame and pinning *alternatives considered* to specific requirements, threads can resolve faster and re-litigation is reduced. The work of writing the requirements down is real, but it is work the PEP authors would otherwise do informally and repeatedly in thread responses.

**Who decides what goes in here?**
Additions and changes are made by PR; the WheelNext working group reviews. The bar is *traceability to a real concern*, not consensus on the answer. A requirement can be in the document with status `proposed` and an open question attached.

**Can I cite a requirement ID in a PEP discussion?**
Yes — that is what the IDs are for. `OV-001`, `SEC-003`, `PROV-002`, `OA-001` are stable references you can use in threads, PRs, and reviews.

**What is an options analysis, and how is it different from a requirement?**
A requirement says *what must be true* (e.g. "the system shall be extensible to new accelerator dimensions"). An options analysis says *given the requirements, here are several ways to satisfy them, here is how each one trades off against the others, and here is the recommendation*. Requirements live in the [requirements doc](./requirements.md); options analyses live in [their own section](./options-analyses/index.md) and each gets its own page. An options analysis cites the requirements it weighs.

**When does a decision get an options analysis rather than just a line in the PEP?**
Only the small number of load-bearing decisions — the ones whose consequences are large, whose alternatives have non-obvious trade-offs, and which the community would reasonably want to see analysed before the PEP is finalised. Medium-importance design choices live inside the requirements doc's *Alternatives Considered* field. Small ones stay in the PEP's *Rejected Alternatives* section. Four to six options analyses across the full PEP series is the expected scale.

---

## Get involved

- **Repo:** [github.com/wheelnext/wheelnext.dev](https://github.com/wheelnext/wheelnext.dev)
- **PEP discussions:** [discuss.python.org/c/packaging](https://discuss.python.org/c/packaging/14)
- **Reference implementations:** [github.com/wheelnext](https://github.com/wheelnext)
- **Working group:** see the [WheelNext home page](../../index.md) for current members and meeting cadence.
