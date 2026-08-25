# OA-001 — Variant Compatibility Mechanism

**Status:** Draft · **Decision target:** Providers PEP · **Date:** 2026-05-23
**Traces to:** SYS-006, SYS-007, SYS-008, PROV-001, PROV-002, PROV-003, PROV-009, SEC-002, SEC-006, SEC-007, RES-001, RES-003, RES-004, RES-007, UX-001, OV-001, OV-003, OV-004, OV-005, OV-007

---

## Summary

This analysis evaluates four structurally distinct ways to determine, at install time, which wheel variants are compatible with the current environment, plus one policy variant of the recommended option (Option 2a: provider plugins with no security guardrails). The choice constrains the rest of the providers PEP and has multi-year consequences for who maintains accelerator-specific code, how quickly new hardware reaches users, and where trust boundaries sit.

**Recommendation:** Option 2 — external provider plugins — for the general case. Option 4 (user-declared environment, no detection code) is affirmatively provided for as a configuration of Option 2: via the override controls in OV-001/OV-003/OV-004, and via a first-class static environment file (OV-007; format expected to be standardized in a future PEP) that is independently needed for deterministic cross-target builds. This makes the no-detection-code mode available to trust-conscious and HPC environments without requiring it as the default. Option 2a — the same plugin architecture with every named provider run automatically — is rejected outright: it fails hard `must` requirements (SEC-007, PROV-003) regardless of how well it scores elsewhere in the matrix. The other two alternatives either freeze the design at today's hardware landscape (Option 1) or scatter compatibility logic across every package that ships variants (Option 3).

---

## Context

Variant compatibility cannot be determined from static metadata alone. GPU vendor and architecture, accelerator runtime versions (CUDA, ROCm, oneAPI), CPU instruction-set features beyond standard platform tags, MPI ABI flavors, and BLAS choices all require runtime detection against the host system — *or* require the user to declare these facts about their environment in some form.

The four options below differ on a single structural question: **where does the code that knows how to detect a CUDA driver version live, if it lives anywhere at all?** That question has knock-on effects through extensibility, trust, maintenance allocation, and ecosystem coordination that this document tries to make legible. One policy variant (Option 2a) shares Option 2's answer to the structural question and differs only on whether the named code runs by consent or automatically.

---

## The options

### Option 1 — Installer-hardcoded compatibility logic

Detection code for each variant axis (CUDA, ROCm, AVX-512, etc.) lives inside installer codebases — pip, uv, Poetry, and so on. The wheel and its metadata declare *what* variant they are; the *whether-compatible* judgment is the installer's, made from logic the installer ships.

- **Who maintains CUDA-specific code:** the pip team, the uv team, the Poetry team — each separately.
- **How a new accelerator gets supported:** the vendor opens issues against each installer; installer maintainers review, implement, and ship in a release.
- **Trust model:** no new trust surface. Compatibility logic runs inside code the user already trusts to install packages.

A presentational variant of this option is "extended PEP 508-style environment markers with installer-side detection" — wheels declare requirements like `Requires-Variant: cuda_runtime_version >= "12.0"` and the installer hardcodes how to measure each marker. This is the same option in terms of where detection code lives and who maintains it; only the declaration syntax differs. It is *not* Option 4, which requires the values to be user-supplied rather than installer-detected.

### Option 2 — External provider plugins

Detection code lives in installable Python packages — "providers" — distributed through the same channels as ordinary packages. The wheel's metadata names which providers it needs. During resolution, the installer loads the named providers and asks each whether it considers a candidate variant compatible.

- **Who maintains CUDA-specific code:** a provider package (for example `nvidia-variant-provider`), maintained by the vendor, a community, or a designated working group.
- **How a new accelerator gets supported:** a vendor or community member publishes a provider package. No installer change required.
- **Trust model:** provider packages are code that the installer executes during resolution. This reuses the existing package trust surface but moves code execution earlier in the install lifecycle (resolution rather than install). Providers execute only after a wheel candidate has been selected, never at index-query time (PROV-002). Controls in PROV-003 (enumerable providers) and OV-003 (disable specific providers) narrow the exposure, and installers may vendor, reimplement, or allow-list commonly used providers so that the default path involves no third-party code at all.

### Option 2a — Provider plugins, automatic execution (no guardrails)

A policy variant of Option 2, not a structurally distinct option: detection code lives in the same provider packages, named by the same wheel metadata. The difference is that the installer automatically installs and runs whatever provider a wheel's metadata names — no vendoring, no allow-list, no opt-in prompt, no user action of any kind. This is the design point the plugin architecture naturally suggests before security requirements are applied, and it is included here so that the cost of each guardrail is priced against it explicitly rather than assumed.

- **Who maintains CUDA-specific code:** identical to Option 2 — a provider package maintained by the vendor, a community, or a designated working group.
- **How a new accelerator gets supported:** a provider is published and immediately works for every user, with zero installer changes and zero user action. The fastest time-to-support of any option in this analysis.
- **Trust model:** wheel metadata becomes an arbitrary-code-execution vector at resolution time. Any package — or a compromised package, or a compromised index — can name a provider that the installer will fetch and run automatically, before the user has agreed to install anything. This is structurally the historical `setup.py` model, and it is the same class of risk as the FlashAttention source-distribution workaround that PEP 817 documents under "Security Risk"; the PyTorch nightly-index compromise shows the threat class is not hypothetical. As written, it violates SEC-007 (no surprise code execution in a default-configured installer) and PROV-003 (the user can enumerate and disallow the providers that would run).

### Option 3 — Package-internal compatibility logic

Each package that ships variants includes its own detection code. The package's metadata or build artefacts declare a small executable shim that the installer runs to choose among that package's variants. There is no shared mechanism across packages.

- **Who maintains CUDA-specific code:** every package author, separately. PyTorch maintains its own copy, JAX maintains its own, CuPy maintains its own.
- **How a new accelerator gets supported:** each package author who wants to ship for it writes their own detection code.
- **Trust model:** detection code ships with the package; trust extends to whatever the package itself already has, but the surface multiplies with the number of variant-shipping packages.

### Option 4 — User-declared environment with declarative markers

The wheel format adds an extended set of environment markers covering accelerator runtimes, GPU vendors, CPU instruction sets, MPI ABIs, and similar dimensions. Wheels declare their compatibility using these markers. The installer matches the wheel's markers against environment values that the user has declared — by hand in `pyproject.toml` or a config file, or generated by a separate tool the user runs once and feeds in. No detection code lives in the installer or in any plugin loaded during resolution.

- **Who maintains CUDA-specific code:** no one, in the spec. The spec defines the marker namespace; users (or vendors shipping helper tools outside the spec) determine values themselves.
- **How a new accelerator gets supported:** the marker namespace gets extended (a spec change), but no new code is required in installers or providers. Users start declaring the new marker.
- **Trust model:** the smallest of any option. No code runs to make compatibility decisions. All inputs are user-supplied. Closest to today's purely-declarative metadata model.

Note that the user-types-values and user-runs-a-detection-tool variants are equivalent at the spec level: the spec defines the marker schema and the matching behavior, and is silent on how the user obtains the values. Tools that generate the values are useful add-ons but require no standardization.

---

## Desirable properties

Requirements with `must` priority act as pass/fail gates, not weights: an option that violates one is rejected regardless of how well it scores elsewhere in the matrix (this is what disqualifies Option 2a against SEC-007 and PROV-003). The properties below are the comparative dimensions used to weigh the options that pass the gates. Each traces to the requirements register where applicable. Priorities are a draft proposal for discussion, not settled; see Dissent.

| #  | Property                                                                      | Priority | Traces to            |
|----|-------------------------------------------------------------------------------|----------|----------------------|
| C1 | Extensibility to new accelerator dimensions without spec or installer changes | Critical | SYS-007              |
| C2 | Time-to-support for new hardware (vendor commitment → user install)           | High     | implied by SYS-007   |
| C3 | Trust surface: where untrusted code runs and what it can do                   | Critical | SYS-008, SEC-002, SEC-006, SEC-007 |
| C4 | Maintenance allocation: which team carries which burden                       | Medium   | — (unsourced; priority especially draft) |
| C5 | Installer and index complexity for non-variant users                          | Critical | SYS-006              |
| C6 | Cross-package coordination (consistent CUDA major across PyTorch and CuPy)    | High     | RES-003              |
| C7 | User-facing simplicity and explainability                                     | Critical | UX-001               |
| C8 | Degradation when relevant detection code is missing                           | Medium   | RES-004, RES-007     |

The weights are consistent with the recommendation's revealed preferences: C1 and C7 are Critical because they are what the recommendation turns on (C1 disqualifies Option 1; C7 is the argument against Option 4 as default). C3 is Critical as well: reviewer response made clear that resolution-time code execution is a hard no for a substantial constituency — the breadth of the reaction to Option 2a is the evidence — not one concern among several. That makes the central tension of this analysis explicit: the recommended option scores only mixed on a Critical property, and survives because the gates (SEC-007, PROV-003) plus the mitigations in Option 2's trust model bound the exposure. If those mitigations are weakened, the Critical weight is precisely why the recommendation must be revisited in Option 4's favor (see the contingency note in §Recommendation) — and, because the design affirmatively provides the Option 4 mode for those who reject resolution-time execution entirely, a hard-no reviewer is not asked to accept Option 2's trust profile, only to tolerate its availability as the default for others. C8 is Medium because the `must` requirements behind it guarantee every surviving option an acceptable fallback, so it barely differentiates.

---

## Assessment

> Legend (each rating pairs a distinct shape with a distinct color, so the matrix stays readable under color-vision deficiency or monochrome printing): **✅✅** strong · **✅** adequate · **⚠️** mixed / partial · **⛔** weak / fails · **➖** not applicable. The prose after each indicator explains the rating.

| Property            | Option 1 — hardcoded                              | Option 2 — providers                                                      | Option 2a — providers, no guardrails                                | Option 3 — per-package                                              | Option 4 — user-declared                                              |
|---------------------|---------------------------------------------------|---------------------------------------------------------------------------|---------------------------------------------------------------------|---------------------------------------------------------------------|-----------------------------------------------------------------------|
| **C1 Extensibility** | ⛔ Poor — every new dimension needs installer changes | ✅✅ Strong — providers ship independently                                     | ✅✅ As Option 2                                                        | ⚠️ Strong per package, weak across the ecosystem                       | ⚠️ Moderate — new dimensions require spec-level marker namespace extension, but no installer code |
| **C2 Time-to-support** | ⛔ Months — vendor waits on installer release cycles | ✅ Weeks — provider publishes when ready                                     | ✅✅ Best of any option — provider publishes and is picked up with no installer or user step | ⚠️ Per package; no shared timeline                                     | ⚠️ Spec change first (months); after that, immediate per user            |
| **C3 Trust surface** | ✅ No new trust surface — logic runs in already-trusted installer code (but see Discussion) | ⚠️ New: provider code runs at resolution. Mitigated by PROV-002, PROV-003 and OV-003   | ⛔ Worst of any option — automatic resolution-time execution of code named by untrusted metadata | ⚠️ Per package; detection runs at install for every variant-shipping package | ✅✅ Smallest of any option — no detection code runs anywhere              |
| **C4 Maintenance allocation** | ⛔ Concentrated on installer teams; vendors have no direct contribution path | ✅✅ Distributed to vendors and community; installer stays generic             | ✅✅ As Option 2                                                        | ⛔ Duplicated across packages; each author maintains the same logic    | ⚠️ Distributed to users (and optionally to vendor-shipped helper tools outside the spec) |
| **C5 Non-variant complexity** | ✅✅ None                                              | ✅ Minimal — providers only loaded when a variant package is in resolution   | ✅ As Option 2                                                        | ✅✅ None for non-variant packages                                       | ✅✅ None                                                                  |
| **C6 Cross-package coordination** | ✅ Possible — installer has the full picture        | ✅ Possible — shared providers can coordinate                                | ✅ As Option 2                                                        | ⛔ Difficult — no shared mechanism                                     | ✅ Possible — installer has the user-declared values to consistency-check against |
| **C7 User simplicity** | ⚠️ Best for the default case; worst when detection is wrong (no extension point) | ✅ Moderate — users may need to install a provider; errors explainable per provider | ✅✅ Zero-friction default — the UX ceiling Option 2 is measured against | ⛔ Worst — every package presents detection differently                | ⚠️ Poor for casual users (must know CUDA version, instruction sets, etc.); strong for users who already manage their environment carefully |
| **C8 Missing detection** | ➖ N/A — detection is always present by construction | ✅ Falls back to null variant per RES-004; explainable                       | ✅ Provider auto-installed, so rarely missing; falls back to null variant per RES-004 when it cannot be fetched | ⚠️ Falls back per package; behavior varies                             | ✅ Falls back to null variant if user has not declared values; explainable |

---

## Discussion

The matrix simplifies. Five nuances deserve narrative.

**The trust calculation in Option 2 is not new code execution; it is new *timing* of code execution.** Installing any package already runs arbitrary setup code under the existing model. What changes with providers is that the trusted code now runs during *resolution*, not only during install. That is a real shift, but it is a smaller one than "introducing arbitrary code execution into pip," which is sometimes how the concern is framed in passing. The mitigations in PROV-003 (the user can enumerate which providers will run) and OV-003 (the user can disable specific providers) narrow it further.

**Option 2a prices the guardrails.** The delta between Option 2a and Option 2 is exactly what the guardrails cost: a slightly slower path to a working install for a brand-new provider, and one consent step (or none, where the provider is vendored or allow-listed) for the user. What that buys is the difference between "code runs because someone trusted it" and "code runs because metadata named it" — under Option 2a, publishing a wheel that names a malicious provider is sufficient to execute code on every machine that merely *resolves* against it, before any install decision. At the same time, PEP 817's security discussion warns about security fatigue: an opt-in that prompts too often erodes toward Option 2a in practice, one blanket "yes" at a time. That is the argument for vendoring and allow-listing commonly used providers — keeping the default path silent *and* safe — rather than either removing the guardrails or prompting for every provider.

**Option 1's "no new trust surface" advantage is partly an accounting trick.** The compatibility logic still has to exist somewhere and still has to be maintained by humans. Locating it inside the installer means installer maintainers become responsible for understanding every accelerator and ABI on every platform. The trust surface is the same size; it is just allocated entirely to one team that has not signed up for the work. The same calculus applies to risk: bugs in CUDA detection have to be fixed by the installer team, on the installer team's release cadence, regardless of NVIDIA's urgency.

**Option 3 is the status quo, extrapolated.** Today's `numpy` / `numpy-cuda12` workarounds are a degenerate Option 3 with separate package names standing in for the missing variant mechanism. Codifying Option 3 as the design would entrench a pattern the community already finds unsatisfactory: every package solving the same problem differently, no consistent user experience, no shared audit story.

**Option 4 has a real existing constituency.** The HPC world already operates roughly in Option 4 mode via modulefiles, Spack environments, and EasyBuild — users (or their site administrators) declare the available toolchain, accelerator, MPI flavor, and so on, and software is built and installed against that declaration. For these users, automatic detection is unwanted: they want explicit, auditable, reproducible builds where the environment is part of the lockfile rather than something discovered at install time. The same calculus applies in locked-down corporate environments where the trust profile of resolution-time code execution is a blocker. Option 4 is not academic; it is the working model for a non-trivial fraction of scientific Python's existing user base. The question is whether to require this model for everyone (which the matrix shows costs heavily on C7 for casual users) or to make it reachable as an opt-in configuration.

---

## Recommendation

**Adopt Option 2 (external provider plugins) as the default mechanism, with Option 4 behavior reachable as a configuration.**

Option 2 is the only option that satisfies SYS-007 (extensibility to future dimensions) without paying excessively on SYS-006 (bounded complexity for non-users) or on cross-package coordination (C6). Its costs concentrate on the trust model and are real but addressable — they are the subject of the SEC bucket, particularly SEC-002 (no new trust root), SEC-006 (no privilege escalation), and the explicit provider trust controls in PROV-003 and OV-003.

Option 2a is rejected outright despite its unmatched C2 and C7 scores. It fails two of the pass/fail gates defined in §Desirable properties — SEC-007 (no surprise code execution in a default-configured installer) and PROV-003 (the user can enumerate and disallow the providers that would run) — and gates are not tradeable against scores. Its role in this analysis is calibration: it defines the UX ceiling that guardrail design should approach, and it makes the cost of each guardrail explicit rather than assumed.

Option 4 has the strongest trust profile and addresses a real existing constituency (HPC, locked-down corporate environments), but the C7 cost of requiring all users to declare their environment by hand is too high for the general case. The crucial observation is that Option 4-style usage is reachable as a *configuration* of Option 2: a user can disable providers via OV-003 and supply variant identity directly via OV-001 — with OV-004 guaranteeing that overrides work with no provider plugin installed at all — which gives them the no-detection-code experience Option 4 promises.

This is not left as a theoretical configuration; the design affirmatively provides for it. The recommendation is that installers add UX to accept a static file declaring an environment's variant-relevant properties (OV-007 makes this a first-class mechanism, not a hack; the file format is expected to be standardized in a future PEP), and/or UX for selecting individual variant features directly (extending OV-001-style pinning from whole variants to per-feature choices). Crucially, the static file is not merely a trust accommodation: it is independently required for deterministic selection whenever the install target is not the machine running the installer — building container images for other hardware, populating wheelhouses, lockfile-driven rebuilds in CI (RES-001, OV-005). Because mainstream workflows depend on it, the Option 4 path will exist and be maintained regardless of how many users choose it for trust reasons — it cannot become a neglected side door.

Treating Option 4 as the default would force its UX on users who neither want nor need it; treating it as a supported path on top of Option 2 serves both constituencies.

The recommendation is contingent on the providers PEP (and the future static-file PEP) carrying through on the override and disable controls and the static declaration path (OV-001, OV-003, OV-004, OV-007, PROV-003). If they are weakened during drafting, the Option 4-as-configuration argument collapses and the recommendation should be revisited.

---

## Consequences if adopted

**Positive.** Hardware vendors get a direct contribution path to ecosystem support. Installer teams keep their codebases generic. New accelerators reach users in weeks rather than installer release cycles. Cross-package coordination on accelerator versions becomes structurally possible.

**Negative.** A new artefact type (provider plugins) joins the ecosystem with its own maintenance, versioning, and trust questions. Resolution-time code execution is a real change to the installer security model and will be a discussion topic on its own. Users may need to install a provider as part of onboarding to a variant-shipping package, which is a new step.

**Neutral.** Detection logic moves from installer codebases to provider packages. The total amount of detection code in the ecosystem does not change materially; its ownership does.

---

## Dissent and open concerns

- **Resolution-time code execution is contested.** Some reviewers argue the existing install-time execution boundary should be preserved. The counter-argument is in §Discussion; this remains a live discussion and should be linked to the relevant Discourse threads as they accumulate.
- **Provider versioning interacts with lockfiles in ways not yet specified.** PROV-009 records the requirement; the mechanism is open.
- **The boundary between "external provider plugin" and "package shipping its own provider" is fuzzy** for packages that ship a single dominant variant axis. Worth a follow-up OA if the pattern becomes common.
- **Option 4 as default vs as configuration is itself contestable.** The recommendation treats Option 4 as a reachable configuration of Option 2, but some reviewers may argue Option 4 should be the default for trust reasons, with provider-based detection as the opt-in. The matrix's C7 assessment is the main counter-argument; if the working group's read of the user base differs, the recommendation should be revisited.
- **The Option 2 / Option 2a distinction may erode in practice.** If opt-in prompts are frequent enough, users will blanket-enable every provider, converging on Option 2a de facto while retaining its ceremony — the security-fatigue risk PEP 817 raises. This is the strongest argument for vendoring and allow-listing commonly used providers rather than prompting per provider: the guardrails only hold if the safe path is also the low-friction path.
- **Property priorities are a first draft.** C4 in particular traces to no requirement, so its Medium weight is pure judgment. C3 was raised from an initial High to Critical after review — the breadth of hard-no reactions to Option 2a showed trust is not one concern among several. With C3 Critical, the Option-4-as-default position (preceding bullet) sharpens from a disagreement about weights to a direct question about scores: whether Option 2's mitigations genuinely bound the exposure enough to justify a mixed C3 rating rather than a weak one.

---

## References

- **Traces to:** SYS-006, SYS-007, SYS-008, PROV-001, PROV-002, PROV-003, PROV-009, SEC-002, SEC-006, SEC-007, RES-001, RES-003, RES-004, RES-007, UX-001, OV-001, OV-003, OV-004, OV-005, OV-007 (see the [requirements register](../requirements.md))
- **PEPs:** providers PEP (in drafting), [PEP 825](https://peps.python.org/pep-0825/) (data model context)
- **Discourse:** [PEP 817 thread](https://discuss.python.org/t/pep-817-wheel-variants-beyond-platform-tags/105860)
- **Related options analyses:** none yet. Further OAs will be added as load-bearing decisions are identified.
