---
name: defender
description: Blue-team release-gate analysis for smart contract deployment and upgrade readiness. Classifies repositories, checks deploy/upgrade execution paths, CI/CD trust boundaries, config drift, secrets/signer operational security, and outputs evidence-backed release verdicts. Use when preparing deployments, validating upgrade runbooks, reviewing deploy scripts or CI pipelines, and enforcing pre-release operational safety.
---

# Defender

A blue-team release-gate skill for smart contract systems.

Defender determines whether a repository is ready to deploy or upgrade safely. It focuses on release execution risk rather than exploit novelty.

Use Defender to identify:
- wrong-chain and wrong-address deployment risk
- broken initialization and unsafe upgrade execution
- leaked or mishandled secrets
- unsafe CI/CD deployment trust boundaries
- signer and admin concentration risk
- unrehearsed role transfers and post-deploy gaps
- false confidence from test-passing without deployment readiness

## Use when

Use when:
- deploying smart contracts
- preparing upgrades
- running deployment scripts
- hardening CI/CD
- rotating admin roles
- validating ownership handoff
- periodically enforcing blue-team release hygiene

## Non-goals

Defender is not a replacement for:
- full exploit-focused security audits
- protocol economic review
- invariant discovery across all business logic
- deep proxy vulnerability analysis already covered by `proxy-upgrade-safety`

Where proxy or upgradeability exists, Defender focuses on **release execution safety**, not full upgrade vulnerability discovery.

## Core operating rule

**Evidence first.**

Only report findings from repository evidence such as:
- contracts
- deploy scripts
- test and fork scripts
- CI workflows
- manifest files
- dependency manifests
- lockfiles
- docs and runbooks
- address books and config files

Do not give generic recommendations as findings. Separate:
- **Detection**: what the repository proves
- **Policy**: what release posture should be enforced

## Required execution order

Defender must run in this order:

1. Project classification
2. Defence pass
3. Severity scoring
4. Release verdict

Do not skip directly to advice.

## Reference packs

Load reference material selectively to keep reasoning deterministic and evidence-first.

- Always load:
  - `references/finding-catalog.md`
  - `references/severity-model.md`
  - `references/evidence-query-playbook.md`
- Load by category:
  - classification: `references/project-classification.md`
  - CI and supply chain: `references/ci-supply-chain.md`
  - deploy drift: `references/config-drift-checks.md`
  - upgrade flow: `references/upgrade-readiness.md`
  - signer/admin posture: `references/signer-opsec.md`
  - false confidence guardrails: `references/false-confidence.md`
  - immediate validation: `references/post-deploy-validation.md`
  - incident context: `references/case-study-mapping.md`
  - edge-case policy: `references/compensating-controls.md`
  - concrete detection examples: `references/good-vs-bad-snippets.md`
- Load templates as needed:
  - baseline format: `templates/defender-report-template.md`
  - worked verdict examples:
    - `templates/defender-report-block-deploy-example.md`
    - `templates/defender-report-proceed-with-risk-example.md`
    - `templates/defender-report-ready-for-staged-release-example.md`
  - operational checklists:
    - `templates/pre-mainnet-checklist.md`
    - `templates/upgrade-checklist.md`
    - `templates/post-deploy-smoke-tests.md`
    - `templates/signer-role-mapping.md`
    - `templates/incident-response-checklist.md`

---

## Phase 1 — Project classification

Classify the repository before assessing risk.

### 1. Framework
Detect one of:
- Foundry
- Hardhat
- hybrid
- other

Evidence may include:
- `foundry.toml`
- `forge-std`
- `hardhat.config.*`
- `package.json` scripts
- deployment tooling structure

### 2. Language
Detect one of:
- Solidity
- Vyper
- Cairo
- mixed

Evidence may include:
- `src/**/*.sol`
- `contracts/**/*.sol`
- `*.vy`
- `Scarb.toml`
- Starknet project layout

### 3. Upgradeability
Classify as:
- upgradeable
- immutable
- mixed

Evidence may include:
- OZ upgradeable imports
- proxy admin scripts
- UUPS / Transparent / Beacon / Diamond patterns
- initializer usage
- upgrade tasks or scripts

### 4. Protocol type
Infer best-fit category:
- token
- vault
- AMM
- lending
- bridge
- governance
- NFT
- staking
- other

State uncertainty if classification is mixed or incomplete.

### 5. Deployment surface
Document whether deployment appears to be:
- local/manual
- script-driven
- CI-triggered
- multisig-driven
- upgrade-task driven

### 6. CI surface
Classify:
- GitHub Actions
- GitLab CI
- other
- none

Output a short classification block before moving on.

---

## Phase 2 — Defence pass

Evaluate all categories below.

### A. Build integrity

Goal: determine whether release artifacts are reproducible and configuration is internally consistent.

Check for:
- compiler version pinned
- optimizer runs pinned
- `evmVersion` pinned or intentionally omitted
- lockfiles committed and consistent
- duplicate or conflicting toolchain configuration across Foundry/Hardhat/subprojects
- artifact directories or generated outputs that appear stale or inconsistent with source
- source verification metadata readiness
- local-only assumptions required to rebuild artifacts

Examples of evidence:
- `foundry.toml`
- `hardhat.config.ts`
- `package.json`
- `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`
- build scripts
- verifier scripts

Escalate when:
- compiler settings used for deployment differ from verification settings
- multiple configs imply ambiguous build output
- release artifacts cannot be reproduced from repo state

### B. Dependency, secrets, and supply chain

Goal: identify release-path compromise risk from dependency and secret handling.

Check for:
- plaintext secrets committed to repository
- documentation or scripts encouraging plaintext private keys
- `.env` patterns for private keys or sensitive deploy secrets
- missing redaction or secret scanning guidance
- floating versions in `package.json`
- unpinned git dependencies
- arbitrary install or postinstall scripts
- dependency confusion risk in package naming or private registry assumptions
- abandoned, suspicious, or conflicting packages
- mismatched OpenZeppelin versions across packages/modules
- unsafe transitive tooling in TS, Python, Rust, or Cairo sidecars
- downloaded binaries without provenance checks

### Secret-handling policy

Defender should advocate against storing private keys in plaintext `.env` files.

Preferred recommendation:
- use **Foundry keystore-backed accounts** for deployment and signing
- keep private keys out of plaintext repository-adjacent files
- treat private-key-in-env patterns as risky

Classify:
- plaintext private key in repo or deploy docs: **BLOCKER** or **HIGH** depending on exposure
- production deploy scripts wired to plaintext secret env variables: **HIGH**
- generic `.env.example` for non-secret config only: acceptable if clearly non-sensitive

### C. CI/CD trust and security

Goal: determine whether automation can safely build, test, and deploy production releases.

Check for:
- unpinned GitHub Actions refs such as `@main`, `@master`, or floating tags where commit pinning is expected
- workflows exposing secrets to untrusted triggers
- production deploys allowed from non-protected branches, tags, or weak triggers
- no manual approval gate for production deployment
- too many jobs with access to deploy secrets
- downloading and executing remote scripts without checksum or signature validation
- suspicious binaries or curl-bash style installation in release path
- self-hosted runner trust assumptions with no hardening notes
- workflows that combine test and production deploy trust zones carelessly

Escalate when CI can deploy to production without strong gatekeeping.

### D. Deployment config drift

Goal: catch catastrophic mismatches between intended and actual deployment target.

Verify:
- chain ID matches intended network
- RPC endpoint corresponds to the intended chain
- deployer address is the expected account
- constructor and initializer args appear final, reviewed, and environment-specific
- token, oracle, router, treasury, pauser, fee recipient, and admin addresses match intended chain
- no testnet or stale addresses appear in production config
- decimal assumptions match live dependencies where such assumptions are encoded
- explorer verifier settings and metadata are pinned
- no dangerous default-network fallback behavior exists
- no silent env fallback can switch target network or deployer unexpectedly

Flag Foundry FFI usage:
- Foundry FFI in deploy path is **HIGH** by default
- escalate to **BLOCKER** if FFI influences deployment parameters, secrets, addresses, or execution flow in a way that cannot be safely audited

### E. Deploy-script safety

Goal: ensure scripts are explicit, reviewable, and not fragile.

Check for:
- chain assertions in scripts
- deployer assertions in scripts
- explicit environment validation before broadcast
- initialization steps present and ordered correctly
- idempotency assumptions documented or enforced
- no accidental reuse of test config or mock addresses
- address book writes or generated manifests are reviewable
- no opaque runtime branching that changes deployment outcome unexpectedly
- verification commands are rehearsed and recorded

Escalate when a script can silently deploy to the wrong environment or leave contracts half-configured.

### F. Upgrade readiness

Only if project classification indicates upgradeable or mixed.

Goal: assess upgrade execution safety, not all theoretical upgrade vulnerabilities.

Check for:
- implementation contract initialized or intentionally uninitializable
- initializer calldata reviewed
- storage layout diff reviewed and documented
- proxy admin ownership confirmed
- timelock or multisig upgrade path confirmed
- pause and rollback authority known
- upgrade rehearsed on a fork or staging equivalent
- upgrade scripts validate implementation, proxy, and admin addresses explicitly

Escalate when upgrade execution depends on undocumented assumptions.

### G. Signer and admin opsec

Goal: determine whether critical privileges are handled with appropriate separation and signer safety.

Check for:
- deployer is dedicated, not a general personal wallet
- hardware-backed or secure signer path is preferred or documented
- admin is multisig, not EOA, for production-sensitive systems
- emergency roles documented
- signer separation exists between deployer, upgrader, pauser, treasury, and other sensitive roles
- no hot-wallet production admin unless explicitly accepted and justified
- dry-run transaction hashes or transaction review process exists before signing

Do not assume a multisig exists if not shown. Report what evidence exists.

### H. Address and role mapping

Goal: extract the control plane.

Map evidence for:
- owner/admin
- proxy admin
- upgrader
- pauser/guardian
- treasury
- fee recipient
- oracle or updater role
- keeper if present
- timelock if present

Flag:
- role concentration in one signer
- EOA-only critical control
- zero or placeholder addresses
- undocumented role transfer sequence

### I. Fork-rehearsed deployment

Goal: reject false confidence from untested release paths.

Require evidence of fork, staging, or equivalent rehearsal for:
- deployment scripts
- initializer calls
- role transfers
- ownership transfers
- upgrade execution
- verification commands
- immediate smoke tests

If no evidence exists, this is usually at least **HIGH** for mainnet release readiness.

### J. Post-deploy readiness enforced pre-deploy

Goal: ensure the team knows what must happen immediately after release.

Check for a defined plan covering:
- bytecode or source verification
- ownership and admin assertions
- pause test path or emergency validation plan
- oracle heartbeat sanity or dependency liveness checks
- expected event emissions
- smoke test integrations
- monitoring and alerting handoff
- deployment manifest and address archive

Absence of a post-deploy plan should lower confidence materially.

### K. Unsafe deploy ergonomics

Goal: catch convenience shortcuts that create operational blast radius.

Check for:
- one-command broadcast with weak guardrails
- missing confirmations or environment assertions
- hidden defaults and implicit env resolution
- CREATE / CREATE2 assumptions undocumented where address determinism matters
- nonce-sensitive deployment steps undocumented
- scripts that are too convenient to review safely under pressure

---

## Phase 3 — False confidence section

This section is mandatory.

Defender must state explicitly that passing the following does **not** prove deploy safety:
- unit tests
- `forge test`
- static analysis
- lint
- typecheck

Defender should recommend evidence-driven release controls such as:
- dry run on fork
- permission diff before and after deployment
- storage layout diff
- expected address diff
- emitted event assertions
- role inventory snapshot
- post-deploy smoke calls

The purpose of this section is to prevent AI-assisted or checklist-only workflows from overstating release readiness.

---

## Phase 4 — Severity scoring

Use these severities.

### BLOCKER
Can directly cause:
- wrong deployment target
- lost admin or control
- leaked secrets
- broken initialization
- broken or unsafe upgrade execution
- deploy to wrong chain
- unverifiable release artifacts
- unrecoverable configuration failure

### HIGH
Materially increases compromise risk or release failure likelihood.
Examples:
- production admin remains EOA without risk acceptance
- fork rehearsal absent for mainnet path
- unsafe CI deploy permissions
- FFI used in deploy-critical logic
- stale or cross-network addresses in config

### MEDIUM
Should be fixed before mainnet but may not block testnet or staging.
Examples:
- docs incomplete but release path otherwise evidenced
- role mapping partly implicit
- verification process weakly documented

### LOW
Hygiene, observability, documentation, or process gaps that weaken assurance without invalidating release.

Be explicit when a finding blocks:
- all releases
- mainnet only
- upgrade releases only

---

## Phase 5 — Release verdict

Always finish with a verdict.

Allowed verdicts:
- `VERDICT: BLOCK DEPLOY`
- `VERDICT: PROCEED WITH RISK`
- `VERDICT: READY FOR STAGED RELEASE`

Support the verdict with:
- top blockers
- required actions before release
- evidence reviewed

If evidence is incomplete, say so plainly.

---

## Required output format

Use this structure:

```text
DEFENDER REPORT

1. Project Classification
- Framework:
- Language:
- Upgradeability:
- Protocol Type:
- Deployment Surface:
- CI Surface:

2. Release Findings
BLOCKER:
- ...
HIGH:
- ...
MEDIUM:
- ...
LOW:
- ...

3. False Confidence Warnings
- ...

4. Release Verdict
VERDICT: BLOCK DEPLOY / PROCEED WITH RISK / READY FOR STAGED RELEASE

Top blockers:
- ...

Required actions before release:
- ...

Evidence reviewed:
- ...
```

---

## Analysis style

- be precise
- be deterministic
- prefer short evidence-backed bullets
- distinguish repository facts from recommended policy
- avoid filler explanations
- do not overstate certainty

## Relationship to other QuillShield skills

Defender complements, but does not replace:
- exploit-focused auditing
- proxy vulnerability analysis
- oracle manipulation analysis
- reentrancy analysis
- invariant reasoning

Use Defender as the **release gate** after or alongside offensive analysis.
