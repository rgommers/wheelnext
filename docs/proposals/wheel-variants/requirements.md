# Wheel Variants — Requirements Register

**Status:** Draft sketch · **Companion to:** the Wheel Variants PEP series — PEP 817 (informational umbrella) plus PEP 825 and the Standards Track PEPs that follow · **Maintainers:** WheelNext working group

---

!!! note "Which kind of “requirements”?"

    In packaging discussions, "requirements" usually means normative obligations on
    implementers — the RFC 2119 MUST/SHOULD/MAY language of a spec. **That is not what
    this document is.** Nothing here binds any tool.

    The entries in this register are *requirements on the design*, not requirements *in*
    the design: the outcomes — capability, usability, security, maintainability,
    compatibility — that the finished design must achieve, and how we would verify that
    it does. The PEPs make the design choices that meet them, and the PEPs alone carry
    normative language for implementers. (Systems engineering calls these two kinds
    *performance requirements* and *design requirements* — where "performance" means
    what the system must accomplish, not how fast it runs.)

    Two consequences for reading this document:

    - The "shall" in each Statement addresses the design effort — the working group
      holding itself accountable — not tool authors.
    - The Priority field (`must`/`should`/`may`) ranks how critical a requirement is to
      the effort's success. It is not RFC 2119 vocabulary.

## How to read this document

This is the **requirements layer** for the Wheel Variants work. It is *not* a normative specification — the PEPs are. This document exists to:

1. Make the *intent* of the design explicit and addressable, separate from the *mechanism* (which lives in the PEPs).
2. Give reviewers, stakeholders, and the Packaging Council a single map of the whole problem, so per-PEP discussions can be anchored to shared requirements rather than re-litigated each time.
3. Keep *alternatives considered and why rejected* findable, tied to the requirements that motivated the decision: a requirement's *Analysis* field links to where the alternatives are worked through — a PEP's *Rejected Alternatives* section, or, for load-bearing design choices, a companion [options analysis](./options-analyses/index.md).
4. Make it visible which requirements are allocated to which PEP, which are deferred, and which are still open.

If a requirement here conflicts with a PEP, the PEP wins; please file an issue so we can reconcile.

### Per-stakeholder entry points

- **End users / data scientists** — start with [System Requirements](#top-level-system-requirements), then [UX](#ux-user-experience) and [Override & Pinning](#ov-override-pinning).
- **Package maintainers** — start with [Building](#bld-building), [Data Model](#dm-data-model), [Variant Providers](#prov-variant-providers).
- **Installer and index maintainers** — start with [Resolution & Selection](#res-resolution-selection) and [Index Serving](#idx-index-serving).
- **Security reviewers** — start with [Security & Trust](#sec-security-trust), then the [Provider](#prov-variant-providers) trust model.
- **Steering / Packaging Council** — start with [System Requirements](#top-level-system-requirements) and the [PEP Allocation Matrix](#pep-allocation-matrix).

---

## Document conventions

### Requirement IDs

`<BUCKET>-NNN`, e.g. `DM-004`, `SEC-003`. Buckets:

| Bucket  | Name                          |
|---------|-------------------------------|
| SYS     | Top-level system              |
| DM      | Data Model                    |
| PROV    | Variant Providers             |
| RES     | Resolution & Selection        |
| UX      | User Experience               |
| OV      | Override & Pinning            |
| SEC     | Security & Trust              |
| BLD     | Building                      |
| IDX     | Index Serving                 |
| MIG     | Migration & Compatibility     |

IDs are immutable once assigned, even if the requirement is later deferred or rejected. New requirements get the next free number in their bucket. When an existing requirement is split into more granular sub-requirements during drafting, the original ID is preserved with letter suffixes (e.g., DM-001 becomes DM-001a and DM-001b); this keeps earlier citations of the original ID resolvable to both successors.

### Requirement template

Only **Statement**, **Rationale**, and **Priority** are expected on every fully written-out requirement; the remaining fields are optional and appear where they earn their place.

- **Statement** — a single sentence in "shall" form; the "shall" addresses the design effort, not tool implementers.
- **Rationale** — why this matters.
- **Priority** — `must`, `should`, or `may`: how critical the requirement is to the effort's success. These are not RFC 2119 keywords.
- **Traces to** — the system goal (SYS-*) or more general requirement this entry refines.
- **Allocation** — which PEP currently carries this, or `unallocated`. The [PEP allocation matrix](#pep-allocation-matrix) is authoritative at bucket level; this field adds PEP-section detail.
- **Status** — `proposed`, `accepted`, `deferred`, `rejected`, `superseded`.
- **Source** — where the requirement originated (Discourse thread, issue, stakeholder).
- **Verification** — how we will know the requirement is met (benchmark, conformance test, prototype, deployment evidence). Used where verification is non-obvious; absence means the default of verification by review of the realized design.
- **Dependencies** — peer requirement IDs this genuinely depends on; conflicts, on the rare occasions they exist, are called out explicitly as such. Refinement of a more general requirement belongs in *Traces to*, not here.
- **Analysis** — a link to the options analysis or PEP section that works through the related design choice. Design content — including rejected alternatives — lives there, not here.

Filing rule: every requirement has **one home bucket**. Related concerns in other buckets cross-reference it by ID rather than restating it.

Full entries are written out because they are load-bearing — hence their priorities are overwhelmingly `must`. Compact entries carry an inline *Priority* annotation only where one has been assigned; an unannotated compact entry has not yet been prioritized.

Requirements are written out in full when they become decision-relevant; the rest are listed compactly. This page is the single home of every requirement — there are no separate detail pages.

---

## Top-level system requirements

These are the high-level commitments the whole design is accountable to. Every lower-level requirement should trace to at least one of these.

### SYS-001 — Hardware and ABI variant support

- **Statement:** The system shall allow a single project to ship multiple wheels for the same `(name, version)` that differ along hardware, accelerator, ABI, or other build-time dimensions.
- **Rationale:** Existing platform tags cannot express GPU vendor/architecture, CPU instruction-set features, accelerator runtimes (CUDA, ROCm, oneAPI), MPI ABIs, or BLAS choices. The status quo forces ad-hoc workarounds (separate package names, environment markers, install scripts).
- **Priority:** must
- **Allocation:** umbrella (all PEPs)
- **Verification:** end-to-end prototype installs a GPU-variant wheel correctly on matching hardware and rejects it on non-matching hardware.

### SYS-002 — Single project name

- **Statement:** Variants shall live under one project name; the design shall not require splitting variants across separate PyPI projects.
- **Rationale:** The `pkg`/`pkg-cpu`/`pkg-cuda12` pattern is the workaround the proposal is *replacing*. Splitting names breaks dependency resolution across the ecosystem, complicates upgrades, and enables name squatting.
- **Priority:** must
- **Allocation:** PEP 825 (data model)
- **Dependencies:** SEC-003

### SYS-003 — Backward compatibility

- **Statement:** Existing non-variant wheels, indices, installers, and build backends shall continue to function unchanged.
- **Priority:** must
- **Allocation:** PEP 825, MIG-* requirements
- **Note:** PEP 825 §Variant label introduces a constraint that Python tags in wheel filenames MUST NOT start with a digit, applying to all wheels (not only variant wheels) once the PEP is accepted. SYS-003 is satisfied conditionally on this: existing wheels (which do not start their Python tags with digits) remain installable, but the constraint becomes ecosystem-wide for new wheels going forward.

### SYS-004 — Auditability

- **Statement:** A user or auditor shall be able to determine, after the fact, which variant was selected and why.
- **Rationale:** Variant selection introduces a new dynamic step in installation. Without an audit trail, reproducibility and incident response suffer.
- **Priority:** must
- **Allocation:** UX, SEC, lockfile spec
- **Dependencies:** UX-001, SEC-004, RES-005

### SYS-005 — User override

- **Statement:** The user shall be able to deterministically override automatic variant selection.
- **Rationale:** Automated selection will be wrong sometimes — wrong hardware detection, deliberate cross-installs, debugging, reproducibility. The escape hatch must exist and must not require expert knowledge to use safely.
- **Priority:** must
- **Allocation:** *under discussion* — see [Open Allocation Questions](#open-allocation-questions)
- **Dependencies:** OV-*

### SYS-006 — Bounded complexity for non-users

- **Statement:** Users and packages that do not use variants shall not pay measurable cost in installer latency, index complexity, or learning curve.
- **Rationale:** This is a feature for a subset of the ecosystem (scientific / ML / HPC). It must not tax everyone else.
- **Priority:** must
- **Verification:** benchmark `pip install` for a non-variant workload before and after; no regression beyond noise.

### SYS-007 — Extensibility to future dimensions

- **Statement:** The design shall accommodate new variant axes (e.g. future accelerators, new ABIs, sandbox flavors) without further spec changes.
- **Rationale:** Hard-coding today's hardware landscape into the spec guarantees we will be back here in five years.
- **Priority:** must
- **Allocation:** PROV-* (the provider model is the extension point)

### SYS-008 — No new trust root

- **Statement:** The design shall not introduce a new trust authority, code-signing root, or curation gatekeeper beyond what the existing PyPI/index ecosystem already provides.
- **Rationale:** Provider plugins are code. Distributing them through any channel other than the existing package-distribution channel multiplies the trust surface and the supply-chain blast radius.
- **Priority:** must
- **Allocation:** PEP for providers
- **Note:** refined for provider distribution by SEC-002.

---

## DM — Data Model

The shape of the wheel file, the variant metadata, and how variants are identified at the file and index level.

### DM-001a — Variant identity in filename

- **Statement:** A variant wheel's *identity* within a `(name, version)` shall be expressible in its filename.
- **Rationale:** Index servers, mirrors, caches, and existing wheel-handling tooling operate on filenames. A wheel with no filename-level distinguishing component cannot be referenced or selected without content inspection.
- **Priority:** must
- **Allocation:** PEP 825 §Variant label
- **Note:** "Identity" here means *which variant of this package this wheel is*, not *what compatibility properties this variant has*. The properties are covered by DM-001b.

### DM-001b — Variant properties may require metadata fetch

- **Statement:** Determining the *properties* (compatibility data) of a variant from its label MAY require fetching the index-level variant metadata file or the wheel's own variant metadata, in addition to inspecting the filename.
- **Rationale:** Encoding properties in the filename would either require very long filenames (problematic on Windows and some filesystems) or impose arbitrary limits on property counts, making the format unsuitable for multidimensional compatibility matrices. PEP 825 §Rejected Ideas → "Predictable variant labels" engages this trade-off and rejects the encode-in-filename design for these reasons.
- **Priority:** must
- **Allocation:** PEP 825 §Variant label, §Variant metadata, §Index-level metadata
- **Analysis:** PEP 825 §Rejected Ideas ("Predictable variant labels") works through the rejected alternatives — properties encoded directly in the label, and properties as a hash of the property set.

### DM-002 — Variant metadata schema

- **Statement:** Variant metadata shall be expressed in a machine-readable schema with a documented version field.
- **Priority:** must
- **Allocation:** PEP 825
- **Verification:** JSON Schema (or equivalent) published with the PEP; round-trip tests in the reference implementation.

### DM-003 — Index-level discoverability

- **Statement:** Clients shall be able to enumerate the variants offered for a `(name, version)` without downloading the wheels themselves.
- **Rationale:** Downloading every wheel just to decide which one to keep would defeat the purpose for large GPU wheels.
- **Priority:** must
- **Allocation:** PEP 825 + companion index PEP

### Additional DM requirements (compact)

- **DM-004** — Wheel files remain valid wheels per the existing wheel spec; variant support is an extension, not a fork.
- **DM-005** — Variant metadata round-trips losslessly through unpack/repack cycles.
- **DM-006** — Multiple orthogonal variant axes (e.g. `cuda=12` AND `cpu_features=avx512`) shall be expressible in a single variant identifier.
- **DM-007** — Feature values within a `(namespace, feature)` tuple shall be treated as a set and serialized in canonical lexically-sorted form, so tools can compare metadata using equality. Note: this is narrower than the original draft, which incorrectly implied namespace order was non-semantic; per PEP 825 §Default priorities, namespace order is semantically meaningful (it drives variant selection priority) and is not canonical-izable.
- **DM-008** — The null variant (no variant specified) shall always be a valid value.
- **DM-009** — Variant wheels shall be able to express dependencies whose presence or version is conditional on variant identity (e.g. a CUDA-variant wheel having different transitive deps from a CPU-variant wheel of the same package and version). *Allocation:* PEP 825 §Environment markers (which provides this via four new markers — `variant_namespaces`, `variant_features`, `variant_properties`, `variant_label` — as the design realisation).

---

## PROV — Variant Providers

The plugin model that decides whether a given variant is compatible with the current environment.

### PROV-001 — Typed provider model

- **Statement:** Variant compatibility shall be determined by named, versioned "provider" plugins, each owning one or more axes of the variant space.
- **Rationale:** The set of relevant hardware/feature dimensions is open-ended (SYS-007). Hard-coding detection logic into installers would freeze the design.
- **Priority:** must
- **Analysis:** See [OA-001 — Variant Compatibility Mechanism](./options-analyses/oa-001-variant-compatibility-mechanism.md) for the full options analysis behind this choice, including the alternatives considered (PEP-specified markers with installer-implemented detection, per-package detection, and user-declared environments with declarative markers) and the trade-offs that led to the typed-provider recommendation.

### PROV-002 — Provider discoverability without code execution at index time

- **Statement:** Determining which providers a wheel needs shall not require executing arbitrary code at index-query time.
- **Rationale:** Index queries happen frequently, sometimes against untrusted indices, and often in environments (CI runners, locked-down hosts) where executing third-party code is unacceptable.
- **Priority:** must
- **Traces to:** SEC-001
- **Analysis:** the accepted design — provider declared in wheel metadata, executed only after wheel-candidate selection, never at index-query time — and the rejected execute-during-index-parsing alternative are worked through in [OA-001](./options-analyses/oa-001-variant-compatibility-mechanism.md) and the providers PEP draft.

### PROV-003 — Provider trust model is explicit

- **Statement:** The user shall be able to enumerate which provider plugins would run during a resolution, and shall be able to disallow specific providers.
- **Priority:** must
- **Dependencies:** SEC-002, OV-003

### Additional PROV requirements (compact)

- **PROV-004** — Providers declare the axes they handle and the value space they emit.
- **PROV-005** — Provider versioning follows PEP 440.
- **PROV-006** — Multiple providers may contribute to a single resolution decision.
- **PROV-007** — Provider failure modes are well-defined: a provider that errors does not silently change the answer.
- **PROV-008** — Providers are distributed through the same channel as ordinary Python packages (no new package source).
- **PROV-009** — Providers can be pinned in lockfiles by name and version.
- **PROV-010** — Provider plugins shall not be required to be installed for users who only consume null-variant wheels.

---

## RES — Resolution & Selection

### RES-001 — Deterministic selection

- **Statement:** Given the same inputs (installed providers, environment, available variants), the installer shall always select the same variant.
- **Rationale:** Reproducibility. Non-determinism here would propagate into every CI run, every Docker image build, every lockfile.
- **Priority:** must

### RES-002 — Documented selection priority

- **Statement:** The rules for choosing among multiple compatible variants shall be documented, predictable, and stable across installer implementations.
- **Priority:** must
- **Verification:** cross-installer conformance tests (pip, uv, Poetry).

### Additional RES requirements (compact)

- **RES-003** — Variant selection coordinates across packages in a resolution (e.g. consistent CUDA major version across dependent libraries).
- **RES-004** — A null-variant fallback is always considered when no specific variant matches.
- **RES-005** — The selected variant identity is recordable in lockfiles in a portable form.
- **RES-006** — Resolution caches variant decisions; cache invalidation rules are documented. *Priority:* should.
- **RES-007** — When no compatible variant exists, the installer fails with a diagnosable error rather than silently installing nothing or installing something arbitrary.
- **RES-008** — Variant selection occurs during dependency resolution, not as a post-resolution install-time step. *Rationale:* selection affects what gets installed; deferring it would split the resolution graph in two and break lockfiles. *Priority:* must.
- **RES-009** — Mixed-awareness resolutions (some packages variant-aware, some not) are handled gracefully.
- **RES-010** — Resolution-time cost for *variant users* is bounded: per package, variant selection adds only the index-level metadata fetch (small and cacheable, IDX-002; never wheel payloads, DM-003) and a bounded number of provider invocations, with results cached per environment (RES-006). Complements SYS-006, which bounds the cost for non-users. *Rationale:* the packages this feature serves are the largest on PyPI; trading download pain for resolution pain would forfeit the win. *Priority:* should.

---

## UX — User Experience

Boundary with [OV](#ov-override-pinning): UX covers *observing and understanding* variant behavior; OV covers *controlling* it.

### UX-001 — Explainable selection

- **Statement:** A user shall be able to ask the installer "why did you pick this variant?" and receive a human-readable answer that names the providers involved and the relevant environment facts.
- **Rationale:** Without this, every variant-related bug report becomes archaeology.
- **Priority:** must
- **Verification:** `pip install --explain-variant` (or equivalent) exists and is documented; the variant decision is also visible without installing (e.g. in `--dry-run` output).

### Additional UX requirements (compact)

- **UX-002** — Variant choice is surfaced in normal install output (succinctly), not hidden behind a flag. *Priority:* should.
- **UX-003** — Lockfile entries for variant packages are unambiguous to a human reader.
- **UX-004** — Error messages distinguish "no variant of X matches your environment" from "X is not available."
- **UX-005** — Documentation provides per-stakeholder onboarding (this site is part of the answer). *Priority:* should.
- **UX-006** — Variant identifiers in user-facing output are stable strings that can be copy-pasted into override flags.

---

## OV — Override & Pinning

This bucket is currently **unallocated** to a specific PEP. See [Open Allocation Questions](#open-allocation-questions). Whichever PEP carries it, the requirements stand.

### OV-001 — Pin by variant identifier

- **Statement:** The user shall be able to specify, by identifier, exactly which variant they want installed, bypassing automatic selection.
- **Priority:** must

### OV-002 — Force null variant

- **Statement:** The user shall be able to instruct the installer to ignore variants entirely and install the null-variant wheel.
- **Rationale:** Recovery path. Debugging path. Forced-portability path (e.g. building a container image for unknown target hardware).
- **Priority:** must

### Additional OV requirements (compact)

- **OV-003** — User can disable specific providers without uninstalling them.
- **OV-004** — Overrides do not require any provider plugin to be installed.
- **OV-005** — Overrides are encodable in lockfiles.
- **OV-006** — When multiple override sources apply (CLI flag, static environment file, lockfile), their precedence is documented. *Priority:* should.
- **OV-007** — Static override metadata (e.g. a JSON file declaring "this environment is X") is a first-class mechanism, not a hack.

---

## SEC — Security & Trust

### SEC-001 — No code execution at index time

- **Statement:** Index-time variant filtering shall not require executing untrusted code.
- **Priority:** must
- **Note:** refined for provider discovery by PROV-002.

### SEC-002 — No new trust root

- **Statement:** Provider plugin distribution shall use the same trust model as ordinary package distribution.
- **Priority:** must
- **Traces to:** SYS-008

### SEC-003 — Squatting resistance

- **Statement:** Variant identifiers shall not create new name-squatting opportunities beyond those that exist in the current namespace.
- **Rationale:** This is the concern raised in PEP 817's own Security Risk note: if users learn that `numpy-cuda12` is a real-looking name, attackers register `numpy-cuda13`. Variants under one project name (SYS-002) are the answer, but the spec must hold the line.
- **Priority:** must
- **Source:** PEP 817 Abstract, [Discourse PEP 817 thread](https://discuss.python.org/t/pep-817-wheel-variants-beyond-platform-tags/105860) (multiple reviewers).

### Additional SEC requirements (compact)

- **SEC-004** — The audit log records which provider produced which selection, with provider version.
- **SEC-005** — Variant existence shall not be hidable from a maintainer or auditor inspecting the index.
- **SEC-006** — Provider plugins run with no more privilege than the installer itself.
- **SEC-007** — Default behavior is safe: a user who installs an installer with no extra configuration gets no surprise code execution beyond what is already true today.
- **SEC-008** — Variant namespaces shall have a governance model that prevents arbitrary parties from claiming widely-recognised namespace names (e.g. `nvidia`, `amd`, `intel`). *Rationale:* without a governance model, the namespace dimension becomes a supply-chain attack vector — a malicious actor could publish a wheel claiming the `nvidia` namespace and inject compatibility assertions the real vendor would never make. *Allocation:* future namespace-governance PEP, explicitly deferred from PEP 825 §Open Issues ("governance of variant namespaces").

---

## BLD — Building

### BLD-001 — Variants declared in `pyproject.toml`

- **Statement:** Build backends shall be able to declare a wheel's variant from project configuration, without backend-specific hacks.
- **Priority:** must

### Additional BLD requirements (compact)

- **BLD-002** — Variant metadata is generated from declared configuration; hand-rolling is discouraged but not blocked.
- **BLD-003** — Building a variant does not require the target hardware to be present at build time.
- **BLD-004** — Existing CI ergonomics (one job per variant) work without bespoke tooling. *Priority:* should.
- **BLD-005** — Standard build commands (`pip wheel`, `python -m build`, or equivalent) can produce variant wheels for common cases without extra plugins when a variant is explicitly requested (e.g. via a `config_settings` argument); a build with no variant request produces a non-variant wheel. *Priority:* should.

---

## IDX — Index Serving

### IDX-001 — PEP 503 / 691 compatibility

- **Statement:** Variant metadata shall be served by existing simple-repository APIs without API-breaking changes.
- **Priority:** must

### Additional IDX requirements (compact)

- **IDX-002** — Variant metadata is fetchable independently of the wheel payload (small JSON, cacheable separately).
- **IDX-003** — Existing mirror tooling replicates variants correctly without understanding them.
- **IDX-004** — A variant-unaware client served by a variant-aware index gets the non-variant wheel if one exists; otherwise no compatible wheel is found. Variant wheels are filtered out by ordinary filename verification (per PEP 825 §Backwards Compatibility). The null variant is not what unaware clients install — see MIG-002a.
- **IDX-005** — A variant-aware client served by a variant-unaware index degrades to current behavior.
- **IDX-006** — Index tooling shall guarantee consistency between the index-level variant metadata file (`{name}-{version}-variants.json`) and the variant metadata embedded in each variant wheel for the same `(name, version)`. *Rationale:* auditors and downstream tooling rely on the index-level file as a faithful summary of what the wheels actually declare; divergence between the two would silently break variant selection or hide variants from audit. *Allocation:* PEP 825 §Index-level metadata.

---

## MIG — Migration & Compatibility

### MIG-001 — Existing wheels unchanged

- **Statement:** Wheels that exist today shall remain installable, with no metadata or repackaging required.
- **Priority:** must

### Additional MIG requirements (compact)

- **MIG-002** — Tools without variant awareness shall install the non-variant wheel by default. Variant wheels are filename-distinguishable from non-variant wheels (per PEP 825 §Variant label), so unaware tools that perform full filename verification reject them as malformed rather than installing them by accident.
- **MIG-002a** — The null variant (label `null`, zero properties) is a *variant-aware fallback*, sorted between any other compatible variant and the non-variant wheel (per PEP 825 §Variant ordering). It is not what variant-unaware tools install — those tools cannot parse the variant label component and install the non-variant wheel per MIG-002.
- **MIG-003** — Installer support precedes producer obligation: no one is required to ship variants before installers can consume them.
- **MIG-004** — Existing variant workarounds (e.g. `xgboost`/`xgboost-cpu`) have a documented migration path.
- **MIG-005** — Lockfile formats evolve compatibly; existing lockfiles remain valid.

---

## PEP allocation matrix

The four-PEP sequence (per current plan):

| PEP                   | Scope                                          | Requirement buckets                           | Informed by                                  |
|-----------------------|------------------------------------------------|-----------------------------------------------|----------------------------------------------|
| PEP 825 (data model)  | Wheel format, variant identity, metadata schema | DM-*, parts of MIG-*, parts of IDX-*          | —                                            |
| PEP NNN (providers)   | Provider plugin model, trust, discovery        | PROV-*, parts of SEC-*                        | [OA-001](./options-analyses/oa-001-variant-compatibility-mechanism.md) |
| PEP NNN (UX/sec/maint)| User-facing behavior, audit, configuration     | UX-*, SEC-*, parts of RES-*                   | —                                            |
| PEP NNN (building)    | Build backend interface, variant production    | BLD-*                                         | —                                            |

The *Informed by* column lists the [options analyses](./options-analyses/index.md) whose recommendations a PEP carries. An empty cell does not mean no analysis was done — it means no analysis was formal enough to warrant a standalone document. Most decisions live in the PEPs' own *Rejected Alternatives* sections, linked from the relevant requirement's *Analysis* field.

Cross-cutting buckets:

- **SYS-*** — umbrella, not allocated to any single PEP.
- **RES-*** — split between the providers PEP (mechanism) and the UX/sec/maint PEP (policy and surfacing).
- **OV-*** — currently unallocated; see below.

Cross-cutting threads — single topics that span several buckets, listed here so they stay visible as threads:

- **Lockfiles** — RES-005, OV-005, PROV-009, UX-003, MIG-005.

### Open allocation questions

Documented here so reviewers can see what we have *not* yet decided, rather than discovering it mid-thread.

- **OV-* (Override & Pinning)** — could land in providers, in UX/sec/maint, or in a fifth follow-up PEP. The author group reserves the decision pending the providers PEP draft. The requirements themselves are agreed.
- **RES-003 (cross-package coordination)** — mechanism may belong in the providers PEP; surfacing belongs in UX/sec/maint. Allocation pending.
- **SEC-005 (no hiding)** — current draft places this in the data-model PEP via mandatory metadata declaration; alternative placement in the index PEP is under discussion.

---

## Status legend

- **proposed** — author group has written it down; not yet socialized.
- **accepted** — reflected in current PEP draft or implementation.
- **deferred** — accepted in principle, not in scope for the current PEP series.
- **rejected** — considered and decided against, with the rationale recorded in the entry.
- **superseded** — replaced by another requirement (cross-link).

---

## Glossary

- **Variant** — one of several wheels for the same `(name, version)` differing along build-time dimensions (hardware, ABI, accelerator runtime, etc.).
- **Null variant** — the wheel that is selected when no variant-specific logic applies; equivalent in role to today's wheels.
- **Provider** — a plugin owning one or more variant axes, responsible for determining compatibility between a variant value and the current environment.
- **Axis** — a single dimension along which variants differ (e.g. `cuda_major`, `cpu_features`, `blas`).
- **Override** — a user instruction that bypasses or constrains automatic variant selection.
- **Requirement (as used here)** — a requirement *on* the design (an outcome it must achieve), not a requirement *in* the design (a normative obligation on tools). The latter live in the PEPs.

---

## Document status and process

- This document is **non-normative**. PEPs are normative.
- Changes require a PR against this site and are reviewed by the WheelNext working group.
- A requirement's *status* may change without re-numbering; the ID is permanent.
- Adding a new requirement is cheap; deleting one is not (it is set to `rejected` with rationale instead).
- Cross-links to Discourse threads, GitHub issues, and PEP sections are added as discussions resolve, so the requirement carries its provenance.
