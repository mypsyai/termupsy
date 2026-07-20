# Wayland/Windows Compatibility Roadmap (Phase 1–2)

This document breaks down implementation into repo-specific epics and milestones for:

- **Phase 1:** Architecture RFC + compatibility matrix + non-goals
- **Phase 2:** x86_64 prototype for Wayland + Sway microcontainer profile

It treats Termux as a **universal terminal space** across heterogeneous systems ("**TerminOpsa native termin**"), while keeping implementation boundaries explicit per repository layer.

## Scope

### In Scope (this repository)
- App UX and profile surfaces
- Runtime/session integration points
- Packaging metadata and feature flags
- Safety gates for privileged capabilities

### Out of Scope (other layers/repos)
- Kernel modifications and host admin model internals
- Full Wayland stack internals
- Wine/Box64 package maintenance (belongs to package repos)

## Expansion Targets (Heteromorphic Platforms)

Primary expansion targets for this plan:

1. **Windows 11 IoT devices**
2. **Debian Linux systems**
3. **RK3399 Chromebooks**
4. **Additional x86_64/ARM64 endpoints** that can participate through profile contracts

Target outcome: one consistent terminal experience contract (profile selection, session lifecycle, diagnostics, fallback behavior) regardless of host class.

## Milestone M1 — Phase 1 Architecture Baseline

### Epic P1-E1: RFC and capability model
**Goal:** Establish architecture decisions and trust boundaries for Wayland-native and compatibility profiles.

**Issues:**
1. Draft RFC: runtime profile architecture for Android-hosted sessions  
2. Define capability gating model for expanded authority features  
3. Define audit/logging contract for privileged actions  
4. Document non-goals and layer boundaries (app vs package vs kernel)

**Done Criteria:**
- RFC merged in docs with approved architecture diagrams/flows
- Capability model includes opt-in controls and least-privilege defaults
- Non-goals explicitly prevent scope creep into kernel/package internals

### Epic P1-E2: Compatibility and packaging matrix
**Goal:** Create a source-of-truth matrix for platform/runtime support.

**Issues:**
1. Publish ABI/runtime matrix (x86, x86_64, armv7, arm64) and profile eligibility  
2. Define distribution channel matrix (Play Store constraints vs GitHub/F-Droid capability set)  
3. Add host-class matrix (Win11 IoT, Debian Linux, RK3399 Chromebook, other endpoints) and expected compatibility mode  
4. Add packaging metadata checklist for profile flag rollout  
5. Define fallback behavior matrix for unsupported runtime combinations

**Done Criteria:**
- Matrix exists in docs and is referenced by implementation tasks
- Release channel constraints are explicit and testable
- Host-class compatibility targets are explicit, with ownership boundaries per layer
- Fallback behavior is deterministic and user-visible

### Epic P1-E3: App integration inventory
**Goal:** Identify current app integration points to implement runtime profiles.

**Issues:**
1. Map current session startup flow and extension points  
2. Identify config sources for profile selection and defaults  
3. Define UX entry points for selecting advanced profiles  
4. Document telemetry events needed for profile launch success/failure

**Done Criteria:**
- Integration map links concrete classes/paths in this repo
- Profile injection points are agreed and versioned in RFC
- Telemetry event schema is defined for prototype validation

## Milestone M2 — Phase 2 x86_64 Wayland + Sway Prototype

### Epic P2-E1: Prototype profile wiring
**Goal:** Add one experimental runtime profile for Wayland + Sway microcontainer on x86_64.

**Issues:**
1. Add feature-flagged runtime profile descriptor (`wayland_sway_x64_experimental`)  
2. Add profile validation logic for x86_64-only eligibility  
3. Add safe fallback path to default terminal session on invalid/ineligible launch  
4. Add launch-time diagnostics surfaced to user-visible logs  
5. Define prototype target validation checklist for Win11 IoT-hosted workflows and Debian-linked environments (where applicable via compatibility layers)

**Done Criteria:**
- Feature flag can enable/disable profile without rebuild of architecture logic
- Non-x86_64 devices never attempt profile launch
- Fallback path is automatic and preserves usable terminal access
- Prototype checklist records target status for Win11 IoT, Debian Linux, and RK3399 Chromebook follow-on work

### Epic P2-E2: Session orchestration hooks
**Goal:** Connect profile selection to session/runtime orchestration.

**Issues:**
1. Add session bootstrap hook for profile-specific preflight checks  
2. Add profile-specific environment contract (variables, paths, runtime markers)  
3. Add bounded timeout and retry policy for profile startup  
4. Add structured error codes for startup failure classes

**Done Criteria:**
- Startup path distinguishes profile launch from standard launch
- Preflight failures produce actionable error categories
- Timeout/retry policy prevents stuck startup loops

### Epic P2-E3: Prototype UX and controls
**Goal:** Provide minimal UX for experimental profile activation and observability.

**Issues:**
1. Add experimental settings toggle for Wayland+Sway profile  
2. Add profile selection option in session creation flow (when eligible)  
3. Add warning/consent text for elevated or experimental behavior  
4. Add in-app troubleshooting panel entry for profile diagnostics

**Done Criteria:**
- Users can discover, enable, and disable the profile in-app
- Consent and warning text is shown before first launch
- Diagnostics are accessible without external tooling

### Epic P2-E4: Expansion target onboarding contracts
**Goal:** Prepare standardized onboarding contracts for additional host targets after x86_64 prototype validation.

**Issues:**
1. Define minimum readiness checklist per host target (Win11 IoT, Debian Linux, RK3399 Chromebook)  
2. Define host-specific preflight assertions and failure mapping to shared error codes  
3. Define profile portability checklist for ARM64-targeted follow-up prototypes  
4. Define acceptance criteria for promoting a target from experimental to supported

**Done Criteria:**
- Each target class has a documented readiness contract
- Failure mapping is consistent with existing fallback/error model
- Follow-on ARM64 work can start from a reusable contract instead of ad-hoc onboarding

## Cross-Milestone Security Requirements

Apply these to all M1/M2 issues:

1. Privileged capabilities must be **explicitly opt-in**  
2. Defaults must remain **least privilege**  
3. Privileged actions must be **auditable** via structured logs  
4. Feature flags must support **rapid rollback**  
5. Unsupported combinations must **fail closed** to standard terminal behavior

## Suggested Issue Labels

- `roadmap:wayland`
- `roadmap:compatibility`
- `phase:1`
- `phase:2`
- `epic`
- `security`
- `ux`
- `packaging`
- `observability`
- `roadmap:expansion-targets`
