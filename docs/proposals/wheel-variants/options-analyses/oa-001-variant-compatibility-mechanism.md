# OA-001 — Variant Compatibility Mechanism

**Status:** Draft · **Decision target:** Providers PEP · **Date:** 2026-05-23
**Related requirements:** SYS-006, SYS-007, SYS-008, PROV-001, PROV-002, PROV-003, SEC-002, SEC-006, RES-003, RES-004, UX-001, OV-003

---

## Summary

This analysis evaluates four structurally distinct ways to determine, at install time, which wheel variants are compatible with the current environment. The choice constrains the rest of the providers PEP and has multi-year consequences for who maintains accelerator-specific code, how quickly new hardware reaches users, and where trust boundaries sit.

**Recommendation:** Option 2 — external provider plugins — for the general case. Option 4 (user-declared environment, no detection code) is reachable as a configuration of Option 2 via the override controls in OV-001/OV-003, which makes it available to trust-conscious and HPC environments without requiring it as the default. The other two alternatives either freeze the design at today's hardware landscape (Option 1) or scatter compatibility logic across every package that ships variants (Option 3).

---

## Context

Variant compatibility cannot be determined from static metadata alone. GPU vendor and architecture, accelerator runtime versions (CUDA, ROCm, oneAPI), CPU instruction-set features beyond standard platform tags, MPI ABI flavors, and BLAS choices all require runtime detection against the host system — *or* require the user to declare these facts about their environment in some form.

The four options below differ on a single structural question: **where does the code that knows how to detect a CUDA driver version live, if it lives anywhere at all?** That question has knock-on effects through extensibility, trust, maintenance allocation, and ecosystem coordination that this document tries to make legible.

---

## The four options

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
- **Trust model:** provider packages are code that the installer executes during resolution. This reuses the existing package trust surface but moves code execution earlier in the install lifecycle (resolution rather than install). Controls in PROV-003 (enumerable providers) and OV-003 (disable specific providers) narrow the exposure.

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

## Evaluation criteria

Drawn from requirements where applicable.

| #  | Criterion                                                                    | Source(s)            |
|----|------------------------------------------------------------------------------|----------------------|
| C1 | Extensibility to new accelerator dimensions without spec or installer changes | SYS-007              |
| C2 | Time-to-support for new hardware (vendor commitment → user install)           | implied by SYS-007   |
| C3 | Trust surface: where untrusted code runs and what it can do                   | SYS-008, SEC-002, SEC-006 |
| C4 | Maintenance allocation: which team carries which burden                       | —                    |
| C5 | Installer and index complexity for non-variant users                          | SYS-006              |
| C6 | Cross-package coordination (consistent CUDA major across PyTorch and CuPy)    | RES-003              |
| C7 | User-facing simplicity and explainability                                     | UX-001               |
| C8 | Degradation when relevant detection code is missing                           | RES-004, RES-007     |

---

## Assessment

| Criterion           | Option 1 — hardcoded                              | Option 2 — providers                                                      | Option 3 — per-package                                              | Option 4 — user-declared                                              |
|---------------------|---------------------------------------------------|---------------------------------------------------------------------------|---------------------------------------------------------------------|-----------------------------------------------------------------------|
| **C1 Extensibility** | Poor — every new dimension needs installer changes | Strong — providers ship independently                                     | Strong per package, weak across the ecosystem                       | Moderate — new dimensions require spec-level marker namespace extension, but no installer code |
| **C2 Time-to-support** | Months — vendor waits on installer release cycles | Weeks — provider publishes when ready                                     | Per package; no shared timeline                                     | Spec change first (months); after that, immediate per user            |
| **C3 Trust surface** | Smallest — no new code paths                      | New: provider code runs at resolution. Mitigated by PROV-003 and OV-003   | Per package; detection runs at install for every variant-shipping package | Smallest of any option — no detection code runs anywhere              |
| **C4 Maintenance allocation** | Concentrated on installer teams; vendors have no direct contribution path | Distributed to vendors and community; installer stays generic             | Duplicated across packages; each author maintains the same logic    | Distributed to users (and optionally to vendor-shipped helper tools outside the spec) |
| **C5 Non-variant complexity** | None                                              | Minimal — providers only loaded when a variant package is in resolution   | None for non-variant packages                                       | None                                                                  |
| **C6 Cross-package coordination** | Possible — installer has the full picture        | Possible — shared providers can coordinate                                | Difficult — no shared mechanism                                     | Possible — installer has the user-declared values to consistency-check against |
| **C7 User simplicity** | Best for the default case; worst when detection is wrong (no extension point) | Moderate — users may need to install a provider; errors explainable per provider | Worst — every package presents detection differently                | Poor for casual users (must know CUDA version, instruction sets, etc.); strong for users who already manage their environment carefully |
| **C8 Missing detection** | N/A — always present                              | Falls back to null variant per RES-004; explainable                       | Falls back per package; behavior varies                             | Falls back to null variant if user has not declared values; explainable |

---

## Discussion

The matrix simplifies. Four nuances deserve narrative.

**The trust calculation in Option 2 is not new code execution; it is new *timing* of code execution.** Installing any package already runs arbitrary setup code under the existing model. What changes with providers is that the trusted code now runs during *resolution*, not only during install. That is a real shift, but it is a smaller one than "introducing arbitrary code execution into pip," which is sometimes how the concern is framed in passing. The mitigations in PROV-003 (the user can enumerate which providers will run) and OV-003 (the user can disable specific providers) narrow it further.

**Option 1's "no new trust surface" advantage is partly an accounting trick.** The compatibility logic still has to exist somewhere and still has to be maintained by humans. Locating it inside the installer means installer maintainers become responsible for understanding every accelerator and ABI on every platform. The trust surface is the same size; it is just allocated entirely to one team that has not signed up for the work. The same calculus applies to risk: bugs in CUDA detection have to be fixed by the installer team, on the installer team's release cadence, regardless of NVIDIA's urgency.

**Option 3 is the status quo, extrapolated.** Today's `numpy` / `numpy-cuda12` workarounds are a degenerate Option 3 with separate package names standing in for the missing variant mechanism. Codifying Option 3 as the design would entrench a pattern the community already finds unsatisfactory: every package solving the same problem differently, no consistent user experience, no shared audit story.

**Option 4 has a real existing constituency.** The HPC world already operates roughly in Option 4 mode via modulefiles, Spack environments, and EasyBuild — users (or their site administrators) declare the available toolchain, accelerator, MPI flavor, and so on, and software is built and installed against that declaration. For these users, automatic detection is unwanted: they want explicit, auditable, reproducible builds where the environment is part of the lockfile rather than something discovered at install time. The same calculus applies in locked-down corporate environments where the trust profile of resolution-time code execution is a blocker. Option 4 is not academic; it is the working model for a non-trivial fraction of scientific Python's existing user base. The question is whether to require this model for everyone (which the matrix shows costs heavily on C7 for casual users) or to make it reachable as an opt-in configuration.

---

## Recommendation

**Adopt Option 2 (external provider plugins) as the default mechanism, with Option 4 behavior reachable as a configuration.**

Option 2 is the only option that satisfies SYS-007 (extensibility to future dimensions) without paying excessively on SYS-006 (bounded complexity for non-users) or on cross-package coordination (C6). Its costs concentrate on the trust model and are real but addressable — they are the subject of the SEC bucket, particularly SEC-002 (no new trust root), SEC-006 (no privilege escalation), and the explicit provider trust controls in PROV-003 and OV-003.

Option 4 has the strongest trust profile and addresses a real existing constituency (HPC, locked-down corporate environments), but the C7 cost of requiring all users to declare their environment by hand is too high for the general case. The crucial observation is that Option 4-style usage is reachable as a *configuration* of Option 2: a user can disable providers via OV-003 and supply variant identity directly via OV-001, which gives them the no-detection-code experience Option 4 promises. Treating Option 4 as the default would force this UX on users who neither want nor need it; treating it as an opt-in path on top of Option 2 serves both constituencies.

The recommendation is contingent on the providers PEP carrying through on the override and disable controls (OV-001, OV-003, PROV-003). If they are weakened during drafting, the Option 4-as-configuration argument collapses and the recommendation should be revisited.

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

---

## References

- **Requirements:** SYS-006, SYS-007, SYS-008, PROV-001 through PROV-010, SEC-002, SEC-006, RES-003, RES-004, RES-007, UX-001, OV-001, OV-003
- **PEPs:** providers PEP (in drafting), [PEP 825](https://peps.python.org/pep-0825/) (data model context)
- **Discourse:** [PEP 817 thread](https://discuss.python.org/t/pep-817-wheel-variants-beyond-platform-tags/105860)
- **Related options analyses:** none yet. Further OAs will be added as load-bearing decisions are identified.
