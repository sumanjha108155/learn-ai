# Spec-Driven Development with OpenSpec — Implementing Jira Tickets

**Status:** Draft for team review
**Owner:** _<add your name>_
**Applies to:** All repositories adopting OpenSpec
**Last reviewed:** 2026-08-17

---

## 1. Purpose

This page is the single reference for how we implement a Jira ticket using **OpenSpec**, a spec-driven development (SDD) layer for AI coding assistants.

The core idea: **the spec leads, the code follows.** Instead of pasting a Jira ticket into an AI assistant and hoping for the best, we translate the ticket into reviewable planning artifacts (proposal, specs, design, tasks), get human agreement on those artifacts, and only then let the assistant write code against them. Completed work is folded back into a living set of specifications that becomes our source of truth.

### What this solves

| Problem today | How OpenSpec addresses it |
|---|---|
| Requirements live only in chat history and are lost | Every change gets a version-controlled folder in the repo |
| AI output is non-deterministic and drifts from intent | The agent implements against a written spec, not a prompt |
| Reviewers see a 900-line PR with no stated intent | The PR carries the proposal, design decisions, and task list |
| Nobody knows what the system is *supposed* to do | `openspec/specs/` accumulates as the source of truth |
| Ticket scope creeps silently mid-implementation | Scope changes are an explicit artifact revision, not a quiet commit |

### Scope and non-scope

**In scope:** installation, repo initialisation, configuration, the per-ticket workflow, review gates, CI enforcement, troubleshooting.

**Out of scope:** choosing an AI assistant, our Git branching policy in general, Jira workflow administration, prompt engineering.

> **Important distinction.** Some of what follows is OpenSpec functionality; some is **our team convention** layered on top (change naming, branch naming, review gates, CI, Jira linkage). Convention sections are marked **[Team convention]** so you can tell what the tool enforces from what we enforce.

---

## 2. Audience and prerequisites

### Audience

- Engineers implementing tickets
- Tech leads and reviewers approving plans before code is written
- Anyone onboarding onto a repo that already uses OpenSpec

### Prerequisites

| Requirement | Notes |
|---|---|
| Node.js **20.19.0 or higher** | Hard requirement of the CLI |
| A supported AI coding assistant | Claude Code, Cursor, Windsurf, Copilot (IDE), Codex, Kiro, Gemini, and ~30 others |
| Repo write access | OpenSpec commits real files into the repo |
| Jira access | Ideally via an Atlassian MCP connector in your assistant, so it can read tickets directly |
| Git CLI | Only needed for our branch/PR conventions |

---

## 3. Mental model and glossary

Read this section once. Most confusion about OpenSpec is vocabulary confusion.

| Term | Meaning |
|---|---|
| **Spec** | A durable description of how a capability behaves. Lives in `openspec/specs/`. This is the source of truth, not a historical record. |
| **Change** | One unit of proposed work — in our process, one Jira ticket. Lives in its own folder under `openspec/changes/<change-name>/`. |
| **Artifact** | A file inside a change folder: `proposal.md`, `specs/**/*.md`, `design.md`, `tasks.md`. |
| **Delta spec** | The spec file *inside a change folder*. It describes only what is being added, modified, removed, or renamed — not the whole capability. |
| **Schema** | The workflow shape: which artifacts exist and in what dependency order. The built-in default is `spec-driven` (proposal → specs → design → tasks). |
| **Profile** | Which slash commands get generated for your assistant. `core` is the default (a lean set); `custom` lets you enable the expanded set. |
| **Sync** | Merging a change's delta specs into the main specs. |
| **Archive** | Finalising a completed change: validate, sync, then move the folder into `openspec/changes/archive/`. |
| **Store** | *(Beta)* A standalone OpenSpec repo registered on your machine, so planning can live outside the code repo. |
| **Workset** | *(Beta)* A personal, local, named view of several folders you work on together. |

### The two command surfaces

This is the single most common source of confusion:

| Surface | Where you type it | Examples |
|---|---|---|
| **CLI** | Your terminal | `openspec init`, `openspec list`, `openspec validate` |
| **Slash commands / skills** | Your AI assistant's chat | `/opsx:propose`, `/opsx:apply`, `/opsx:archive` |

The CLI handles setup, inspection, validation, and lifecycle mechanics. The slash commands are where the AI actually authors artifacts and writes code. They operate on the same files.

### Flow at a glance

```
Jira ticket ABC-1234
        │
        ▼
  /opsx:explore ..................... optional: think it through, no files created
        │
        ▼
  /opsx:propose ..................... creates proposal → specs → design → tasks
        │
        ▼
  HUMAN REVIEW GATE  ◄────────── openspec validate --strict, openspec show
        │                              (revise with /opsx:update if needed)
        ▼
  /opsx:apply ....................... writes code, ticks off tasks.md
        │
        ▼
  /opsx:verify ...................... optional: does the code match the artifacts?
        │
        ▼
  PR REVIEW ......................... reviewer reads artifacts + diff together
        │
        ▼
  /opsx:archive ..................... syncs delta specs into openspec/specs/, archives folder
        │
        ▼
  openspec/specs/ is now current
```

---

## 4. One-time setup

### 4.1 Install the CLI (once per machine)

```bash
npm install -g @fission-ai/openspec@latest
openspec --version
```

To upgrade later:

```bash
npm update -g @fission-ai/openspec
openspec update          # run inside each repo to refresh generated files
```

> Do not skip `openspec update` after a CLI upgrade. The generated assistant instructions are written into the repo, so they go stale if you only bump the global package.

### 4.2 Initialise a repository (once per repo)

```bash
cd /path/to/repo
openspec init
```

The interactive flow asks which AI tools to configure. For a reproducible, scripted setup:

```bash
# non-interactive, specific tools
openspec init --tools claude,cursor

# all supported tools
openspec init --tools all

# no assistant integration (CLI only)
openspec init --tools none

# skip prompts and auto-clean legacy files
openspec init --force
```

Useful flags:

| Flag | Purpose |
|---|---|
| `--tools <list>` | `all`, `none`, or a comma-separated list of tool IDs |
| `--profile <core\|custom>` | Override the global profile for this run |
| `--force` | Auto-clean legacy files without prompting |

Recognised tool IDs include: `claude`, `cursor`, `windsurf`, `github-copilot`, `codex`, `gemini`, `kiro`, `kilocode`, `roocode`, `continue`, `cline`, `crush`, `qwen`, `iflow`, `junie`, `trae`, `opencode`, `amazon-q`, `factory`, `qoder`, `lingma`, `costrict`, `codebuddy`, `auggie`, `antigravity`, `forgecode`, `hermes`, `zcode`, `vibe`, `pi`, `oh-my-pi`, `bob`, `kimi`, `codeartsagent`.

### 4.3 What initialisation creates

```
openspec/
├── specs/              # source of truth — grows as changes are archived
├── changes/            # active changes, one folder per ticket
│   └── archive/        # completed changes, dated
└── config.yaml         # project configuration

.claude/skills/         # if claude was selected
.cursor/skills/         # if cursor was selected
.cursor/commands/       # if the delivery mode includes commands
...                     # one directory per configured tool
```

**Commit all of it.** The `openspec/` directory and the generated tool directories are project assets, not local scratch. If your `.gitignore` excludes assistant config directories, add explicit negations for the OpenSpec-generated paths.

### 4.4 Restart your assistant

After `init` or `update`, restart your AI tool so it picks up the new skills. If slash commands are not recognised, this is almost always the reason.

### 4.5 Choose a profile **[Team recommendation]**

The default global profile is `core`, which generates a lean command set: propose, explore, apply, sync, archive. The default delivery mode is `both` (skills and commands).

The **expanded** set adds `/opsx:new`, `/opsx:continue`, `/opsx:ff`, `/opsx:verify`, `/opsx:bulk-archive`, and `/opsx:onboard`.

```bash
openspec config profile      # interactive wizard
openspec update              # apply the selections to this project
```

The wizard opens with a summary of your current state, then lets you change delivery and/or workflows, or exit without writing anything. In the workflow checklist, `[x]` means selected in global config; selections only reach project files when you run `openspec update` (or accept the prompt to apply immediately). `Ctrl+C` cancels cleanly.

Fast preset:

```bash
openspec config profile core   # switch workflows to core, keep delivery mode
```

**Our recommendation:** enable the expanded set, primarily for `/opsx:verify` (an implementation-vs-artifact audit before PR) and `/opsx:continue` (one artifact at a time, which matters on high-risk tickets).

> Profile settings are **global to your machine**, not per repo. Each engineer sets their own profile; `openspec update` reconciles the repo's generated files with it. If a teammate's generated files differ from yours, that is expected — do not fight it in review. If the repo files are out of sync with your global profile, OpenSpec warns and suggests `openspec update`.

### 4.6 Configure project context — do this properly, once

`openspec/config.yaml` is the highest-leverage file in this whole process. Everything you put here stops being something you re-explain in every ticket.

```yaml
# openspec/config.yaml

schema: spec-driven

context: |
  Service: payments-api
  Stack: TypeScript 5.x, NestJS 10, PostgreSQL 15 via Prisma, Redis for caching.
  Deployed to EKS via ArgoCD. Node 20.

  Architecture: controller → service → repository. Controllers never touch
  Prisma directly. All cross-service calls go through the clients/ directory.

  Conventions:
  - class-validator decorators on every DTO
  - ParseUUIDPipe for all UUID path params
  - DELETE returns 204 with no body
  - services return null/false for not-found; controllers throw NotFoundException
  - all money values are integer minor units, never floats

  Testing: Jest. Unit tests colocated as *.spec.ts. Integration tests in test/.
  Any new endpoint requires at least one integration test.

  Non-negotiables:
  - no breaking changes to public API without a versioned route
  - every DB migration must be reversible
  - no PII in logs
```

Per-artifact rules let you steer individual artifacts. Consult the customisation docs for the exact key layout in your installed version, then keep rules short and imperative, for example:

- **proposal** — always state the Jira ticket key and the user-visible outcome; list explicit non-goals.
- **specs** — write requirements as testable scenarios; cover error and empty states.
- **design** — record rejected alternatives and why; call out migration and rollback.
- **tasks** — every task must be independently verifiable; include a test task per requirement.

> **Team convention:** changes to `openspec/config.yaml` require tech lead review. It silently shapes every future ticket in the repo.

### 4.7 Optional: shell completions and health checks

```bash
openspec completion install          # auto-detects shell
openspec completion install zsh      # or specify: bash, zsh, fish, powershell
openspec doctor                      # read-only health report for the resolved root
openspec context                     # what the current working set actually is
```

`doctor` answers "is this healthy?"; `context` answers "what is included?". Neither modifies anything. Note that `doctor` exits 0 even when it reports findings — only outright command failures exit 1, so don't gate CI on its exit code alone; read the status fields.

---

## 5. Naming conventions **[Team convention]**

OpenSpec enforces the *shape* of a change name. We enforce the *content*.

### What OpenSpec requires

Change names must be lowercase kebab-case: lowercase letters, numbers, and single hyphens. Not allowed: spaces, underscores, uppercase letters, consecutive hyphens, leading or trailing hyphens. A **leading number is permitted**, so numeric prefixes can be used for ordering or tiering.

### What we require

**Change name = lowercased Jira key + short verb-led slug.**

```
<jira-key-lowercased>-<verb>-<subject>
```

| Jira ticket | Change name |
|---|---|
| ABC-1234 "Add SSO login" | `abc-1234-add-sso-login` |
| PAY-88 "Refund webhook returns 500" | `pay-88-fix-refund-webhook-500` |
| PLAT-501 "Extract auth into shared lib" | `plat-501-extract-auth-lib` |

Why the key goes first: `openspec list` sorts and greps cleanly, archived folders stay traceable years later, and nobody has to ask which ticket a spec came from.

**Avoid:** `update`, `changes`, `wip`, `fix`, `misc`, `refactor`, `abc-1234` with no slug.

### Branch and PR conventions **[Team convention]**

```bash
git checkout -b feat/abc-1234-add-sso-login     # feat | fix | chore | refactor
```

PR title: `ABC-1234: Add SSO login`

PR description must include:

```markdown
## Jira
ABC-1234

## OpenSpec change
`openspec/changes/abc-1234-add-sso-login/`

## Plan approved by
@tech-lead-handle on <date>  (link to the plan-approval comment)

## Spec impact
Adds: auth/sso — Requirement "SAML assertion validation" (3 scenarios)
Modifies: auth/session — session TTL now configurable

## Archive status
- [ ] `/opsx:archive` run and specs synced (do this after approval, before merge)
```

---

## 6. The per-ticket workflow

This is the operational core of the page. One Jira ticket → one OpenSpec change → one PR.

### Step 0 — Preconditions

Confirm before you start:

- The ticket has acceptance criteria. If it doesn't, that's a Jira problem — fix it there first, not in OpenSpec.
- You are on an up-to-date main branch with a clean working tree.
- You have created your feature branch.

```bash
git checkout main && git pull
git checkout -b feat/abc-1234-add-sso-login
openspec list                 # see what else is active; watch for overlap
openspec list --specs         # what already exists in this area?
```

### Step 1 — Explore (optional, and skipped too often)

Run this when the ticket is ambiguous, touches unfamiliar code, or has more than one plausible approach.

```
/opsx:explore ABC-1234 — read this Jira ticket and work out how it fits our codebase
```

If your assistant has an Atlassian/Jira MCP connector, it can pull the ticket itself. If not, paste the ticket body — description, acceptance criteria, and any linked design doc.

What explore does: opens an unstructured investigation, reads the codebase to answer questions, compares approaches, and can draw diagrams. **It creates no artifacts.** It is a thinking surface with no cost to abandoning it.

Exit criterion: you can state in one paragraph what you're building and why this approach over the alternatives.

> Skipping explore is fine for small, well-specified tickets. Skipping it on a vague ticket means your proposal is a guess wearing a suit.

### Step 2 — Propose

```
/opsx:propose abc-1234-add-sso-login
```

This creates `openspec/changes/abc-1234-add-sso-login/` and generates every artifact required before implementation — for the `spec-driven` schema that's `proposal.md`, `specs/**/*.md`, `design.md`, and `tasks.md` — then stops.

Add context in the same message if the assistant doesn't already have it:

```
/opsx:propose abc-1234-add-sso-login

Jira ABC-1234. Add SAML-based SSO for enterprise tenants.
Acceptance criteria:
- Tenant admin can configure an IdP metadata URL
- Users on an SSO-enabled tenant are redirected to the IdP instead of the password form
- Existing password login keeps working for non-SSO tenants
Out of scope: SCIM provisioning (that's ABC-1240), IdP-initiated flows.
```

**Alternative: step-by-step control (expanded profile).** For high-risk or large tickets, create artifacts one at a time so you can review each before the next is built on it:

```
/opsx:new abc-1234-add-sso-login       # scaffold folder + .openspec.yaml only
/opsx:continue                          # create the next ready artifact
/opsx:continue                          # ...and the next
```

`/opsx:continue` reads the dependency graph, shows what's ready versus blocked, and creates the first ready artifact using its dependencies as context. Several artifacts can become ready at once.

Or fast-forward everything at once (equivalent in spirit to `propose`):

```
/opsx:ff abc-1234-add-sso-login
```

| Situation | Use |
|---|---|
| Clear, small-to-medium ticket | `/opsx:propose` or `/opsx:ff` |
| Large, risky, or unfamiliar area | `/opsx:new` + `/opsx:continue` |
| Requirements still fuzzy | `/opsx:explore` first |

### Step 3 — Inspect what was produced

Move to the terminal. Read the artifacts yourself; do not approve them from the chat summary.

```bash
openspec status --change abc-1234-add-sso-login
openspec show abc-1234-add-sso-login
openspec validate abc-1234-add-sso-login --strict
```

`openspec status` prints artifact-by-artifact progress:

```
Change: abc-1234-add-sso-login
Schema: spec-driven
Progress: 4/4 artifacts complete

[x] proposal
[x] specs
[x] design
[x] tasks
```

Status markers:

| Marker | Meaning |
|---|---|
| `[x]` | Done |
| `[ ]` | Ready to be created |
| `[-]` | Blocked, with the blocking dependency named |
| `[~]` | Skipped, because the change declares `skip_specs: true` |

Other inspection commands:

```bash
openspec show abc-1234-add-sso-login --json --deltas-only   # just the spec deltas
openspec show auth --type spec                              # an existing main spec
openspec show auth --type spec --json --requirements        # requirements only
openspec view                                               # interactive dashboard
openspec instructions --change abc-1234-add-sso-login       # what OpenSpec thinks is next
```

### Step 4 — Human review gate **[Team convention — mandatory]**

**No code is written until a second human approves the plan.** This gate is where OpenSpec earns its cost. Approving a plan takes ten minutes; unwinding two days of confidently wrong AI implementation does not.

Reviewer checklist:

**Proposal**
- Does it reference the Jira key?
- Does it describe the same problem the ticket describes?
- Are non-goals stated explicitly?

**Specs (deltas)**
- Is every acceptance criterion in the ticket represented as a requirement?
- Are requirements testable, or are they vague aspirations?
- Are error paths, empty states, and permission boundaries covered?
- Do the deltas correctly say ADDED vs MODIFIED, and do MODIFIED entries match what's actually in `openspec/specs/` today?

**Design**
- Are alternatives and rejection reasons recorded?
- Migration and rollback addressed?
- Does it respect the conventions in `config.yaml`, or silently invent new patterns?

**Tasks**
- Is each task independently verifiable?
- Is there test coverage for each requirement?
- Anything obviously missing: migrations, feature flags, config, observability, docs?

**Scope**
- Is this one ticket's worth of work, or three tickets pretending to be one?

Record approval as a comment on the Jira ticket, linking the change folder. Include the reviewer handle and date in the PR description later.

> **Anti-pattern:** approving the plan and the code in the same review pass. The whole point is that the plan gate happens *first*, cheaply.

### Step 5 — Revise the plan when it's wrong

Do **not** hand-edit artifacts and hope they stay consistent with each other. Use:

```
/opsx:update abc-1234-add-sso-login — we're storing the session in a cookie, not localStorage
```

What it does: reads the change's artifacts, applies your revision, then reconciles the *other* artifacts in any direction — a design edit can ripple back into the proposal. It confirms each edit with you before writing, one artifact at a time. It touches planning artifacts only and never edits code. It finishes by recommending the next step.

Constraints worth knowing:

- It revises existing artifacts; it will not create missing ones (that's `/opsx:continue`).
- If the change is already implemented, follow up with `/opsx:apply` so the code catches up to the revised plan.
- If the revision changes the *intent* of the ticket, don't stretch the change — archive or abandon it and start a new one.

**Rule of thumb [Team convention]:** if the Jira ticket itself needs re-scoping, stop, fix Jira, then start a fresh change. A change folder that no longer matches its ticket is worse than no change folder.

### Step 6 — Implement

```
/opsx:apply abc-1234-add-sso-login
```

The assistant reads `tasks.md`, identifies incomplete tasks, works through them one at a time writing code and running tests, and marks each complete with `[x]`.

Because completion state lives in `tasks.md` checkboxes, it is **resumable** — if the session dies, run `/opsx:apply` again and it picks up where it stopped. Naming the change explicitly lets you run parallel changes without them confusing each other.

**Practices during apply [Team convention]:**

- Commit per task or per small group of tasks, referencing the ticket: `git commit -m "ABC-1234: validate SAML assertion signature"`
- Run the test suite yourself. Do not take "tests pass" on trust.
- If you discover the plan is wrong mid-implementation, **stop** and go back to `/opsx:update`. Don't let the code and the spec diverge — that's exactly the failure mode this process exists to prevent.
- If a task turns out to be unnecessary, delete it from `tasks.md` with a one-line note rather than ticking it off untouched.

### Step 7 — Verify (expanded profile)

```
/opsx:verify abc-1234-add-sso-login
```

It searches the codebase for implementation evidence and reports across three dimensions:

| Dimension | Question it asks |
|---|---|
| **Completeness** | Are all tasks done, all requirements implemented, all scenarios covered? |
| **Correctness** | Does the implementation match spec intent? Are the spec's edge cases handled? |
| **Coherence** | Are design decisions actually reflected in the code? Are patterns consistent? |

Findings come back as CRITICAL, WARNING, or SUGGESTION. **Verify does not block archive** — it surfaces issues and it is your job to act on them.

**Our policy [Team convention]:** zero CRITICAL findings before you open the PR. Each WARNING must be either fixed, or answered in the PR description with a reason. Common legitimate warnings: a design doc that describes an approach you deliberately changed during implementation — fix the design doc, don't fix the code to match a stale document.

Without the expanded profile, do this manually: re-read `specs/` and `design.md` next to your diff.

### Step 8 — Pre-PR checks

```bash
openspec validate abc-1234-add-sso-login --strict
openspec status --change abc-1234-add-sso-login     # expect all [x], all tasks ticked
# then your normal repo checks
npm run lint && npm test && npm run build
git push -u origin feat/abc-1234-add-sso-login
```

Open the PR using the description template in §5.

### Step 9 — PR review

The reviewer's advantage here is that intent is written down. Read the artifacts first, then the diff, and ask one question: **does this diff implement these artifacts?**

Reviewer checklist:

- Diff matches `tasks.md`; nothing significant is present that no task called for
- Delta specs accurately describe the behaviour change
- Tests exist for each requirement's scenarios
- No unrelated drive-by refactoring (that's a separate ticket)
- `openspec validate --strict` is green in CI

### Step 10 — Archive

Archive **after approval, before merge**, so the synced specs land in the same PR as the code that implements them.

```
/opsx:archive abc-1234-add-sso-login
```

or from the terminal:

```bash
openspec archive abc-1234-add-sso-login
openspec archive abc-1234-add-sso-login --yes        # non-interactive
```

What archive does:

1. Validates the change (skippable with `--no-validate`, which requires confirmation)
2. Prompts for confirmation, unless `--yes`
3. Merges the delta specs into `openspec/specs/`
4. Moves the folder to `openspec/changes/archive/YYYY-MM-DD-<name>/`

The slash-command version also checks artifact and task completion and warns — but does not hard-block — on incomplete tasks, and offers to sync the delta specs if you haven't already.

> **Non-interactive gotcha:** without a terminal — an AI agent, a CI job, or any run with stdin closed — archive cannot answer the confirmation prompt. It stops before touching anything, exits 1, and tells you to rerun with `--yes` and the change name. Pass `--yes` up front in automation.

Then commit and merge:

```bash
git add openspec/
git commit -m "ABC-1234: archive change, sync specs"
git push
```

Finally, move the Jira ticket to Done and paste the archive path into the ticket:

```
OpenSpec: openspec/changes/archive/2026-08-17-abc-1234-add-sso-login/
```

### Step 11 — Optional: sync without archiving

`/opsx:sync` merges delta specs into the main specs while leaving the change active. Archive prompts to sync when needed, so most people never call this directly.

```
/opsx:sync abc-1234-add-sso-login
```

The merge is intelligent rather than copy-paste: it parses ADDED / MODIFIED / REMOVED / RENAMED sections, can add scenarios to an existing requirement without duplicating it, and preserves existing content the delta doesn't mention.

Call it manually when:

| Scenario | Sync early? |
|---|---|
| Long-running change; you want the specs visible in main sooner | Yes |
| Parallel changes need the updated base specs | Yes |
| You want to review the merge separately from the archive step | Yes |
| Small change going straight to archive | No — archive handles it |

---

## 7. Special cases and recipes

### 7.1 Tickets with no behaviour change (refactor, tooling, docs, CI)

A change with **zero spec deltas fails validation** unless the change declares it. Set the flag in the change's `.openspec.yaml`:

```yaml
# openspec/changes/plat-501-extract-auth-lib/.openspec.yaml
skip_specs: true
```

Effects:

- `openspec status` shows `[~] specs (skipped: change declares skip_specs)` and excludes it from the progress count
- `openspec instructions specs` returns a warning only — the artifact must not be created
- The change archives with no extra flag

There is also a one-run escape hatch:

```bash
openspec archive plat-501-extract-auth-lib --skip-specs
```

**Prefer the declaration over the flag.** A change that permanently has no spec deltas should say so in its own metadata; the flag is for one-off runs and is easy to forget.

> If you find yourself reaching for `skip_specs` on something users can observe, that's a signal the ticket *does* have spec impact and you haven't articulated it yet.

### 7.2 Hotfixes and production incidents

Do not let process block a page. Sequence:

1. Fix production. Ship it. Normal incident process applies.
2. **Same or next business day**, create the retroactive change:

```bash
openspec new change ops-9911-fix-null-tenant-crash \
  --description "Retroactive spec for hotfix shipped 2026-08-17"
```

3. Have the assistant write the artifacts describing what was actually done, then archive it so the specs reflect reality.

A spec set that lies is worse than one with a gap. Backfilling within 24 hours is the discipline that keeps it honest.

### 7.3 Large tickets — split them

If a proposal produces more than roughly 15–20 tasks, or touches more than a couple of capabilities, split it in Jira and create one change per sub-ticket. Numeric prefixes are permitted, which makes ordering explicit:

```
100-abc-1235-sso-idp-config
200-abc-1236-sso-redirect-flow
300-abc-1237-sso-session-mapping
```

Sync each change as it completes so later changes build on the updated base specs.

### 7.4 Parallel changes touching the same specs

Two changes editing the same capability will conflict at sync time. Options, best first:

1. **Sequence them.** Archive the first, then let the second build on updated specs.
2. **Sync early.** Run `/opsx:sync` on the first as soon as its specs are settled.
3. **Bulk archive.** `/opsx:bulk-archive` lists completed changes, validates each, detects cross-change spec conflicts, inspects the codebase to resolve them, and archives in creation order — prompting before overwriting spec content.

Conflict resolution in bulk archive is agentic: it looks at what's actually implemented. Read its resolution plan before accepting it.

### 7.5 Adopting OpenSpec in an existing (brownfield) repo

Do **not** try to document the whole system up front. It won't get finished and it won't get read.

1. `openspec init`, then invest properly in `config.yaml` context (§4.6).
2. Leave `openspec/specs/` empty.
3. Run the process on new tickets only. Specs accumulate naturally as changes archive.
4. Optionally, backfill specs for the two or three areas that cause the most incidents — as their own tickets, sized and reviewed like anything else.

Point new joiners at `/opsx:onboard`, which walks the full workflow interactively using the real codebase: it scans for a genuine small improvement, creates a real change, implements it, and archives it, narrating each step. Budget 15–30 minutes. The change it creates is real, so keep or discard it deliberately.

### 7.6 Ticket types and how they map

| Jira issue type | OpenSpec treatment |
|---|---|
| Story / Feature | Standard flow. Full artifact set. |
| Bug | Standard flow. The spec delta usually MODIFIES an existing requirement, or ADDS the scenario that was missing. |
| Task (tooling/CI/build) | `skip_specs: true` |
| Spike | `/opsx:explore` only. No change folder. Output goes into the Jira ticket or a Confluence page. |
| Epic | Never a single change. One change per child ticket. |
| Hotfix | §7.2 — retroactive change. |

---

## 8. Definition of Ready / Definition of Done **[Team convention]**

### Definition of Ready — before `/opsx:apply`

- [ ] Jira ticket has acceptance criteria
- [ ] Change folder exists, named per §5
- [ ] All required artifacts created — `openspec status` shows no gaps
- [ ] `openspec validate <change> --strict` passes
- [ ] A second engineer or the tech lead has approved the plan, recorded in Jira
- [ ] Scope confirmed as one ticket's worth of work

### Definition of Done — before merge

- [ ] All tasks in `tasks.md` ticked
- [ ] Tests written and passing; coverage for each requirement's scenarios
- [ ] `/opsx:verify` shows zero CRITICAL; warnings fixed or answered in the PR
- [ ] `openspec validate <change> --strict` passes locally and in CI
- [ ] PR approved
- [ ] `/opsx:archive` run; delta specs synced into `openspec/specs/`
- [ ] Archive committed in the same PR as the implementation
- [ ] Jira updated with the archive path and moved to Done

---

## 9. CI enforcement **[Team convention]**

Validation is worthless if it's optional. Gate the PR.

```yaml
# .github/workflows/openspec.yml
name: OpenSpec

on:
  pull_request:
    branches: [main]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20.19.0'

      - name: Install OpenSpec
        run: npm install -g @fission-ai/openspec@latest

      - name: Validate all changes and specs
        run: openspec validate --all --strict --json
        env:
          OPENSPEC_TELEMETRY: '0'
          OPENSPEC_CONCURRENCY: '12'
```

The CLI exits `0` on success and `1` on error (validation failure, missing files), so a failing validation fails the job with no extra scripting.

**Suggested additional gates** — implement as small scripts against `openspec list --json` and `openspec status --change <name> --json`:

| Gate | Rule |
|---|---|
| Change exists | A PR whose branch matches `*/[a-z]+-[0-9]+-*` must have a matching active or newly archived change |
| Naming | Every active change name starts with a lowercased Jira key |
| Task completion | No PR merges with unticked tasks in its change |
| Archive hygiene | Nothing sits in `openspec/changes/` for more than N days without activity |

Environment variables available for CI:

| Variable | Effect |
|---|---|
| `OPENSPEC_TELEMETRY=0` | Disable telemetry |
| `DO_NOT_TRACK=1` | Disable telemetry (standard signal) |
| `OPENSPEC_CONCURRENCY` | Parallel validation limit (default 6) |
| `NO_COLOR` | Plain output for logs |

---

## 10. Command reference

### 10.1 Slash commands — core profile

| Command | Purpose |
|---|---|
| `/opsx:explore [topic]` | Investigate and think before committing to a change. Creates no artifacts. |
| `/opsx:propose [name-or-description]` | Create the change and generate all pre-implementation artifacts. |
| `/opsx:apply [change]` | Work through `tasks.md`, writing code and ticking tasks. |
| `/opsx:update [change]` | Revise planning artifacts and reconcile them with each other. Never edits code. |
| `/opsx:sync [change]` | Merge delta specs into main specs; change stays active. |
| `/opsx:archive [change]` | Finalise: check completion, offer sync, move to archive. |

### 10.2 Slash commands — expanded profile

| Command | Purpose |
|---|---|
| `/opsx:new [name] [--schema <name>]` | Scaffold the change folder and `.openspec.yaml` only. |
| `/opsx:continue [change]` | Create the next ready artifact in the dependency chain. |
| `/opsx:ff [change]` | Fast-forward: create all planning artifacts at once. |
| `/opsx:verify [change]` | Audit implementation against artifacts (completeness / correctness / coherence). |
| `/opsx:bulk-archive [changes...]` | Archive several completed changes, resolving spec conflicts. |
| `/opsx:onboard` | Interactive tutorial through a full cycle on the real codebase. |

The change name is optional on most of these — it's inferred from context. **Pass it explicitly** when you have multiple changes in flight.

### 10.3 Syntax by assistant

| Tool | Form |
|---|---|
| Claude Code | `/opsx:propose`, `/opsx:apply` |
| Cursor | `/opsx-propose`, `/opsx-apply` |
| Windsurf | `/opsx-propose`, `/opsx-apply` |
| GitHub Copilot (IDE only) | `/opsx-propose`, `/opsx-apply` |
| Trae, Oh My Pi | `/opsx-propose`, `/opsx-apply` |
| Codex | Skills under `.codex/skills/openspec-*` |
| Kimi Code | `/skill:openspec-propose`, `/skill:openspec-apply-change` |
| CodeArts | `/openspec-propose`, `/openspec-apply-change` |

Intent is identical across tools; only the surface differs. Note that Copilot's prompt files work in the VS Code / JetBrains / Visual Studio extensions but **not** in the Copilot CLI.

**Legacy commands** — `/openspec:proposal`, `/openspec:apply`, `/openspec:archive` — use the older all-at-once workflow. They still work and the artifact structure is compatible, so a legacy change can be continued with OPSX commands. Don't start new work on them.

### 10.4 CLI reference

**Setup**

| Command | Purpose |
|---|---|
| `openspec init [path]` | Initialise. `--tools <list>` `--profile <core\|custom>` `--force` |
| `openspec update [path]` | Regenerate assistant files after upgrade or profile change. `--force` |

**Browsing**

| Command | Purpose |
|---|---|
| `openspec list` | List active changes. `--specs` `--changes` `--sort recent\|name` `--json` |
| `openspec show [item]` | Show a change or spec. `--type change\|spec` `--json` `--deltas-only` `--requirements` `--no-scenarios` `-r <n>` `--no-interactive` |
| `openspec view` | Interactive terminal dashboard |

**Validation and lifecycle**

| Command | Purpose |
|---|---|
| `openspec validate [item]` | `--all` `--changes` `--specs` `--type` `--strict` `--json` `--concurrency <n>` `--no-interactive` |
| `openspec archive [change]` | `-y, --yes` `--skip-specs` `--no-validate` |

**Workflow**

| Command | Purpose |
|---|---|
| `openspec new change <name>` | Create change scaffolding. `--description` `--goal` `--schema` `--store` `--json` |
| `openspec status` | Artifact completion. `--change <id>` `--schema` `--json` |
| `openspec instructions [artifact]` | Enriched instructions for `proposal`/`specs`/`design`/`tasks`/`apply`. `--change` `--schema` `--json` |
| `openspec templates` | Resolved template paths. `--schema` `--json` |
| `openspec schemas` | Available schemas with their artifact flows. `--json` |

**Schemas**

| Command | Purpose |
|---|---|
| `openspec schema init <name>` | New project-local schema. `--description` `--artifacts <list>` `--default` `--force` `--json` |
| `openspec schema fork <source> [name]` | Copy a schema into the project to customise. `--force` `--json` |
| `openspec schema validate [name]` | Validate schema structure and templates. `--verbose` `--json` |
| `openspec schema which [name]` | Show where a schema resolves from. `--all` `--json` |

Schema precedence: project `openspec/schemas/<name>/` → user `~/.local/share/openspec/schemas/<name>/` → built-in package schemas.

**Config, health, utility**

| Command | Purpose |
|---|---|
| `openspec config path\|list\|get\|set\|unset\|reset\|edit` | Global configuration |
| `openspec config profile [preset]` | Configure workflow profile and delivery |
| `openspec doctor` | Read-only health report. `--store <id>` `--json` |
| `openspec context` | Assembled working set. `--json` `--code-workspace <path> [--force]` |
| `openspec feedback <message>` | File a GitHub issue. Requires an authenticated `gh` CLI. `--body <text>` |
| `openspec completion generate\|install\|uninstall [shell]` | Shell completions: bash, zsh, fish, powershell |

**Beta surfaces** — `openspec store setup|register|unregister|remove|list|doctor` and `openspec workset create|list|open|remove`. These let planning live in a standalone repo shared across a team, and let you reopen a set of folders by name. **Command names, flags, file formats, and JSON output may change between releases.** Do not build team process on them yet; revisit once they stabilise.

### 10.5 Which commands agents can drive

Interactive, human-only: `init`, `view`, `workset open`, `config edit`, `feedback`, `completion install`.

Agent-safe via `--json`: `list`, `show`, `validate`, `status`, `instructions`, `templates`, `schemas`, `new change`, and the store/workset read commands. Anything destructive in automation needs `--yes`.

---

## 11. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Slash commands not recognised | Assistant hasn't loaded generated skills | `openspec init` if never run; `openspec update` to regenerate; confirm `.claude/skills/` (or your tool's path) exists; **restart the assistant** |
| "Change not found" | Ambiguous or missing change | Pass the name explicitly (`/opsx:apply abc-1234-add-sso-login`); `openspec list`; check you're in the right directory |
| "No artifacts ready" | Everything is done or blocked | `openspec status --change <name>` to see blockers; create the missing dependency first |
| "Schema not found" | Typo or missing custom schema | `openspec schemas`; `openspec schema which <name>`; `openspec schema init <name>` if it should exist |
| Validation fails with no spec deltas | Change has no spec impact and hasn't declared it | Add `skip_specs: true` to the change's `.openspec.yaml` (§7.1) |
| Artifacts are thin, generic, or wrong | Assistant lacks project context | Fill in `config.yaml` `context:` and per-artifact rules; give more detail in the propose message; use `/opsx:continue` instead of `/opsx:ff` |
| Archive exits 1 in CI or an agent session | No terminal to answer the confirmation prompt | Rerun with `--yes` and the change name, keeping any other flags |
| Generated files differ between teammates | Profile is machine-global | Expected. Agree a team profile; each engineer runs `openspec config profile` then `openspec update` |
| Repo files out of sync with your profile | Profile changed without applying | `openspec update` |
| Specs don't reflect shipped behaviour | Changes implemented but never archived | Archive them; use `/opsx:bulk-archive` for a backlog; add the CI archive-hygiene gate (§9) |
| Sync produced a mangled spec | Delta sections mislabelled (ADDED vs MODIFIED) | Fix the delta, re-run sync; review the merge plan before accepting next time |

---

## 12. Anti-patterns

| Anti-pattern | Why it hurts | Do instead |
|---|---|---|
| Skipping the plan review gate | You've reintroduced "prompt and pray" with extra file overhead | Enforce §4 review; it's ten minutes |
| Hand-editing artifacts to keep the AI happy | Artifacts silently contradict each other | `/opsx:update` |
| One change spanning several tickets | Unreviewable; specs merge badly; can't revert cleanly | One ticket, one change |
| Implementing first, writing the spec after | The spec becomes fiction justifying the code | Spec first; hotfixes get a same-day retroactive change (§7.2) |
| `skip_specs: true` on user-visible behaviour | Your source of truth develops holes | Articulate the spec impact |
| Never archiving | `openspec/specs/` stays empty and the whole exercise was theatre | Archive as part of the PR |
| Empty `config.yaml` | Every ticket re-litigates your conventions | Invest once (§4.6) |
| Trusting `/opsx:verify` as a merge gate | It reports; it does not block | Read it; act on CRITICALs yourself |
| Building process on stores/worksets | Beta surface, subject to change | Wait for stability |
| Drive-by refactors during `/opsx:apply` | Diff no longer matches the tasks; review breaks down | Separate ticket |

---

## 13. FAQ

**Does this replace Jira?**
No. Jira remains the system of record for *what we're doing and when*. OpenSpec is the system of record for *what the software does*. The change name links the two.

**Do I have to use an AI assistant?**
The artifacts are plain Markdown, so you can write them by hand and use the CLI for validation, status, and archive. You lose most of the leverage, but the process is coherent without an AI.

**Doesn't this slow us down?**
It front-loads time into planning and takes it out of rework and review. Small, well-understood tickets are barely affected — `/opsx:propose`, skim, `/opsx:apply`. The payoff scales with ambiguity and blast radius.

**What if my ticket is a one-line config change?**
Judgement applies. Trivial changes with no behavioural impact don't need a change folder; use your normal PR flow. Write down where your team draws that line so it isn't relitigated per ticket.

**Where does design documentation live now — Confluence or `design.md`?**
`design.md` for decisions scoped to one change. Confluence for cross-cutting architecture. Link from `design.md` to Confluence, not the other way round; the change folder should be self-contained enough to read on its own.

**Can two people work on the same change?**
Yes, but coordinate: `tasks.md` checkboxes are the shared state and will conflict in Git if you both run `/opsx:apply` simultaneously. Split by task range, or by change.

**What happens to archived changes?**
They stay in `openspec/changes/archive/YYYY-MM-DD-<name>/` as an audit trail: original proposal, design, tasks, deltas. Excellent context when someone asks in eighteen months why a thing works the way it does.

**How do I see the full history of a capability?**
`openspec show <spec> --type spec` for current truth; `git log openspec/specs/<path>` plus the dated archive folders for how it got there.

---

## 14. Rollout checklist for a new repo **[Team convention]**

- [ ] Confirm Node ≥ 20.19.0 on all dev machines and in CI
- [ ] `openspec init --tools <team's tools>`
- [ ] Write `openspec/config.yaml` context; tech lead reviews it
- [ ] Agree the team profile; everyone runs `openspec config profile` + `openspec update`
- [ ] Confirm `openspec/` and generated tool directories are **not** gitignored
- [ ] Add the CI validation workflow (§9)
- [ ] Add the PR description template (§5) to `.github/PULL_REQUEST_TEMPLATE.md`
- [ ] Link this page from the repo README and the team onboarding page
- [ ] Each engineer runs `/opsx:onboard` once
- [ ] Pick two or three low-risk tickets as pilots before mandating the process
- [ ] Retro after two weeks; update this page

---

## 15. Sources and further reading

Official documentation (verify against your installed CLI version — this tool moves quickly):

- OpenSpec repository — https://github.com/Fission-AI/OpenSpec
- Slash command reference — https://github.com/Fission-AI/OpenSpec/blob/main/docs/commands.md
- CLI reference — https://github.com/Fission-AI/OpenSpec/blob/main/docs/cli.md
- Workflows — https://github.com/Fission-AI/OpenSpec/blob/main/docs/workflows.md
- Customisation (schemas, templates, per-artifact rules) — https://github.com/Fission-AI/OpenSpec/blob/main/docs/customization.md
- Getting started — https://github.com/Fission-AI/OpenSpec/blob/main/docs/getting-started.md
- Examples and recipes — https://github.com/Fission-AI/OpenSpec/blob/main/docs/examples.md
- Explore-first guide — https://github.com/Fission-AI/OpenSpec/blob/main/docs/explore.md
- Existing/brownfield projects — https://github.com/Fission-AI/OpenSpec/blob/main/docs/existing-projects.md
- Supported tools — https://github.com/Fission-AI/OpenSpec/blob/main/docs/supported-tools.md
- Stores (beta) guide — https://github.com/Fission-AI/OpenSpec/blob/main/docs/stores-beta/user-guide.md

Run `openspec <command> --help` for the authoritative flag list in your installed version.

---

## 16. Change log for this page

| Date | Author | Change |
|---|---|---|
| 2026-08-17 | _<your name>_ | Initial draft |# Spec-Driven Development with OpenSpec — Implementing Jira Tickets

**Status:** Draft for team review
**Owner:** _<add your name>_
**Applies to:** All repositories adopting OpenSpec
**Last reviewed:** 2026-08-17

---

## 1. Purpose

This page is the single reference for how we implement a Jira ticket using **OpenSpec**, a spec-driven development (SDD) layer for AI coding assistants.

The core idea: **the spec leads, the code follows.** Instead of pasting a Jira ticket into an AI assistant and hoping for the best, we translate the ticket into reviewable planning artifacts (proposal, specs, design, tasks), get human agreement on those artifacts, and only then let the assistant write code against them. Completed work is folded back into a living set of specifications that becomes our source of truth.

### What this solves

| Problem today | How OpenSpec addresses it |
|---|---|
| Requirements live only in chat history and are lost | Every change gets a version-controlled folder in the repo |
| AI output is non-deterministic and drifts from intent | The agent implements against a written spec, not a prompt |
| Reviewers see a 900-line PR with no stated intent | The PR carries the proposal, design decisions, and task list |
| Nobody knows what the system is *supposed* to do | `openspec/specs/` accumulates as the source of truth |
| Ticket scope creeps silently mid-implementation | Scope changes are an explicit artifact revision, not a quiet commit |

### Scope and non-scope

**In scope:** installation, repo initialisation, configuration, the per-ticket workflow, review gates, CI enforcement, troubleshooting.

**Out of scope:** choosing an AI assistant, our Git branching policy in general, Jira workflow administration, prompt engineering.

> **Important distinction.** Some of what follows is OpenSpec functionality; some is **our team convention** layered on top (change naming, branch naming, review gates, CI, Jira linkage). Convention sections are marked **[Team convention]** so you can tell what the tool enforces from what we enforce.

---

## 2. Audience and prerequisites

### Audience

- Engineers implementing tickets
- Tech leads and reviewers approving plans before code is written
- Anyone onboarding onto a repo that already uses OpenSpec

### Prerequisites

| Requirement | Notes |
|---|---|
| Node.js **20.19.0 or higher** | Hard requirement of the CLI |
| A supported AI coding assistant | Claude Code, Cursor, Windsurf, Copilot (IDE), Codex, Kiro, Gemini, and ~30 others |
| Repo write access | OpenSpec commits real files into the repo |
| Jira access | Ideally via an Atlassian MCP connector in your assistant, so it can read tickets directly |
| Git CLI | Only needed for our branch/PR conventions |

---

## 3. Mental model and glossary

Read this section once. Most confusion about OpenSpec is vocabulary confusion.

| Term | Meaning |
|---|---|
| **Spec** | A durable description of how a capability behaves. Lives in `openspec/specs/`. This is the source of truth, not a historical record. |
| **Change** | One unit of proposed work — in our process, one Jira ticket. Lives in its own folder under `openspec/changes/<change-name>/`. |
| **Artifact** | A file inside a change folder: `proposal.md`, `specs/**/*.md`, `design.md`, `tasks.md`. |
| **Delta spec** | The spec file *inside a change folder*. It describes only what is being added, modified, removed, or renamed — not the whole capability. |
| **Schema** | The workflow shape: which artifacts exist and in what dependency order. The built-in default is `spec-driven` (proposal → specs → design → tasks). |
| **Profile** | Which slash commands get generated for your assistant. `core` is the default (a lean set); `custom` lets you enable the expanded set. |
| **Sync** | Merging a change's delta specs into the main specs. |
| **Archive** | Finalising a completed change: validate, sync, then move the folder into `openspec/changes/archive/`. |
| **Store** | *(Beta)* A standalone OpenSpec repo registered on your machine, so planning can live outside the code repo. |
| **Workset** | *(Beta)* A personal, local, named view of several folders you work on together. |

### The two command surfaces

This is the single most common source of confusion:

| Surface | Where you type it | Examples |
|---|---|---|
| **CLI** | Your terminal | `openspec init`, `openspec list`, `openspec validate` |
| **Slash commands / skills** | Your AI assistant's chat | `/opsx:propose`, `/opsx:apply`, `/opsx:archive` |

The CLI handles setup, inspection, validation, and lifecycle mechanics. The slash commands are where the AI actually authors artifacts and writes code. They operate on the same files.

### Flow at a glance

```
Jira ticket ABC-1234
        │
        ▼
  /opsx:explore ..................... optional: think it through, no files created
        │
        ▼
  /opsx:propose ..................... creates proposal → specs → design → tasks
        │
        ▼
  HUMAN REVIEW GATE  ◄────────── openspec validate --strict, openspec show
        │                              (revise with /opsx:update if needed)
        ▼
  /opsx:apply ....................... writes code, ticks off tasks.md
        │
        ▼
  /opsx:verify ...................... optional: does the code match the artifacts?
        │
        ▼
  PR REVIEW ......................... reviewer reads artifacts + diff together
        │
        ▼
  /opsx:archive ..................... syncs delta specs into openspec/specs/, archives folder
        │
        ▼
  openspec/specs/ is now current
```

---

## 4. One-time setup

### 4.1 Install the CLI (once per machine)

```bash
npm install -g @fission-ai/openspec@latest
openspec --version
```

To upgrade later:

```bash
npm update -g @fission-ai/openspec
openspec update          # run inside each repo to refresh generated files
```

> Do not skip `openspec update` after a CLI upgrade. The generated assistant instructions are written into the repo, so they go stale if you only bump the global package.

### 4.2 Initialise a repository (once per repo)

```bash
cd /path/to/repo
openspec init
```

The interactive flow asks which AI tools to configure. For a reproducible, scripted setup:

```bash
# non-interactive, specific tools
openspec init --tools claude,cursor

# all supported tools
openspec init --tools all

# no assistant integration (CLI only)
openspec init --tools none

# skip prompts and auto-clean legacy files
openspec init --force
```

Useful flags:

| Flag | Purpose |
|---|---|
| `--tools <list>` | `all`, `none`, or a comma-separated list of tool IDs |
| `--profile <core\|custom>` | Override the global profile for this run |
| `--force` | Auto-clean legacy files without prompting |

Recognised tool IDs include: `claude`, `cursor`, `windsurf`, `github-copilot`, `codex`, `gemini`, `kiro`, `kilocode`, `roocode`, `continue`, `cline`, `crush`, `qwen`, `iflow`, `junie`, `trae`, `opencode`, `amazon-q`, `factory`, `qoder`, `lingma`, `costrict`, `codebuddy`, `auggie`, `antigravity`, `forgecode`, `hermes`, `zcode`, `vibe`, `pi`, `oh-my-pi`, `bob`, `kimi`, `codeartsagent`.

### 4.3 What initialisation creates

```
openspec/
├── specs/              # source of truth — grows as changes are archived
├── changes/            # active changes, one folder per ticket
│   └── archive/        # completed changes, dated
└── config.yaml         # project configuration

.claude/skills/         # if claude was selected
.cursor/skills/         # if cursor was selected
.cursor/commands/       # if the delivery mode includes commands
...                     # one directory per configured tool
```

**Commit all of it.** The `openspec/` directory and the generated tool directories are project assets, not local scratch. If your `.gitignore` excludes assistant config directories, add explicit negations for the OpenSpec-generated paths.

### 4.4 Restart your assistant

After `init` or `update`, restart your AI tool so it picks up the new skills. If slash commands are not recognised, this is almost always the reason.

### 4.5 Choose a profile **[Team recommendation]**

The default global profile is `core`, which generates a lean command set: propose, explore, apply, sync, archive. The default delivery mode is `both` (skills and commands).

The **expanded** set adds `/opsx:new`, `/opsx:continue`, `/opsx:ff`, `/opsx:verify`, `/opsx:bulk-archive`, and `/opsx:onboard`.

```bash
openspec config profile      # interactive wizard
openspec update              # apply the selections to this project
```

The wizard opens with a summary of your current state, then lets you change delivery and/or workflows, or exit without writing anything. In the workflow checklist, `[x]` means selected in global config; selections only reach project files when you run `openspec update` (or accept the prompt to apply immediately). `Ctrl+C` cancels cleanly.

Fast preset:

```bash
openspec config profile core   # switch workflows to core, keep delivery mode
```

**Our recommendation:** enable the expanded set, primarily for `/opsx:verify` (an implementation-vs-artifact audit before PR) and `/opsx:continue` (one artifact at a time, which matters on high-risk tickets).

> Profile settings are **global to your machine**, not per repo. Each engineer sets their own profile; `openspec update` reconciles the repo's generated files with it. If a teammate's generated files differ from yours, that is expected — do not fight it in review. If the repo files are out of sync with your global profile, OpenSpec warns and suggests `openspec update`.

### 4.6 Configure project context — do this properly, once

`openspec/config.yaml` is the highest-leverage file in this whole process. Everything you put here stops being something you re-explain in every ticket.

```yaml
# openspec/config.yaml

schema: spec-driven

context: |
  Service: payments-api
  Stack: TypeScript 5.x, NestJS 10, PostgreSQL 15 via Prisma, Redis for caching.
  Deployed to EKS via ArgoCD. Node 20.

  Architecture: controller → service → repository. Controllers never touch
  Prisma directly. All cross-service calls go through the clients/ directory.

  Conventions:
  - class-validator decorators on every DTO
  - ParseUUIDPipe for all UUID path params
  - DELETE returns 204 with no body
  - services return null/false for not-found; controllers throw NotFoundException
  - all money values are integer minor units, never floats

  Testing: Jest. Unit tests colocated as *.spec.ts. Integration tests in test/.
  Any new endpoint requires at least one integration test.

  Non-negotiables:
  - no breaking changes to public API without a versioned route
  - every DB migration must be reversible
  - no PII in logs
```

Per-artifact rules let you steer individual artifacts. Consult the customisation docs for the exact key layout in your installed version, then keep rules short and imperative, for example:

- **proposal** — always state the Jira ticket key and the user-visible outcome; list explicit non-goals.
- **specs** — write requirements as testable scenarios; cover error and empty states.
- **design** — record rejected alternatives and why; call out migration and rollback.
- **tasks** — every task must be independently verifiable; include a test task per requirement.

> **Team convention:** changes to `openspec/config.yaml` require tech lead review. It silently shapes every future ticket in the repo.

### 4.7 Optional: shell completions and health checks

```bash
openspec completion install          # auto-detects shell
openspec completion install zsh      # or specify: bash, zsh, fish, powershell
openspec doctor                      # read-only health report for the resolved root
openspec context                     # what the current working set actually is
```

`doctor` answers "is this healthy?"; `context` answers "what is included?". Neither modifies anything. Note that `doctor` exits 0 even when it reports findings — only outright command failures exit 1, so don't gate CI on its exit code alone; read the status fields.

---

## 5. Naming conventions **[Team convention]**

OpenSpec enforces the *shape* of a change name. We enforce the *content*.

### What OpenSpec requires

Change names must be lowercase kebab-case: lowercase letters, numbers, and single hyphens. Not allowed: spaces, underscores, uppercase letters, consecutive hyphens, leading or trailing hyphens. A **leading number is permitted**, so numeric prefixes can be used for ordering or tiering.

### What we require

**Change name = lowercased Jira key + short verb-led slug.**

```
<jira-key-lowercased>-<verb>-<subject>
```

| Jira ticket | Change name |
|---|---|
| ABC-1234 "Add SSO login" | `abc-1234-add-sso-login` |
| PAY-88 "Refund webhook returns 500" | `pay-88-fix-refund-webhook-500` |
| PLAT-501 "Extract auth into shared lib" | `plat-501-extract-auth-lib` |

Why the key goes first: `openspec list` sorts and greps cleanly, archived folders stay traceable years later, and nobody has to ask which ticket a spec came from.

**Avoid:** `update`, `changes`, `wip`, `fix`, `misc`, `refactor`, `abc-1234` with no slug.

### Branch and PR conventions **[Team convention]**

```bash
git checkout -b feat/abc-1234-add-sso-login     # feat | fix | chore | refactor
```

PR title: `ABC-1234: Add SSO login`

PR description must include:

```markdown
## Jira
ABC-1234

## OpenSpec change
`openspec/changes/abc-1234-add-sso-login/`

## Plan approved by
@tech-lead-handle on <date>  (link to the plan-approval comment)

## Spec impact
Adds: auth/sso — Requirement "SAML assertion validation" (3 scenarios)
Modifies: auth/session — session TTL now configurable

## Archive status
- [ ] `/opsx:archive` run and specs synced (do this after approval, before merge)
```

---

## 6. The per-ticket workflow

This is the operational core of the page. One Jira ticket → one OpenSpec change → one PR.

### Step 0 — Preconditions

Confirm before you start:

- The ticket has acceptance criteria. If it doesn't, that's a Jira problem — fix it there first, not in OpenSpec.
- You are on an up-to-date main branch with a clean working tree.
- You have created your feature branch.

```bash
git checkout main && git pull
git checkout -b feat/abc-1234-add-sso-login
openspec list                 # see what else is active; watch for overlap
openspec list --specs         # what already exists in this area?
```

### Step 1 — Explore (optional, and skipped too often)

Run this when the ticket is ambiguous, touches unfamiliar code, or has more than one plausible approach.

```
/opsx:explore ABC-1234 — read this Jira ticket and work out how it fits our codebase
```

If your assistant has an Atlassian/Jira MCP connector, it can pull the ticket itself. If not, paste the ticket body — description, acceptance criteria, and any linked design doc.

What explore does: opens an unstructured investigation, reads the codebase to answer questions, compares approaches, and can draw diagrams. **It creates no artifacts.** It is a thinking surface with no cost to abandoning it.

Exit criterion: you can state in one paragraph what you're building and why this approach over the alternatives.

> Skipping explore is fine for small, well-specified tickets. Skipping it on a vague ticket means your proposal is a guess wearing a suit.

### Step 2 — Propose

```
/opsx:propose abc-1234-add-sso-login
```

This creates `openspec/changes/abc-1234-add-sso-login/` and generates every artifact required before implementation — for the `spec-driven` schema that's `proposal.md`, `specs/**/*.md`, `design.md`, and `tasks.md` — then stops.

Add context in the same message if the assistant doesn't already have it:

```
/opsx:propose abc-1234-add-sso-login

Jira ABC-1234. Add SAML-based SSO for enterprise tenants.
Acceptance criteria:
- Tenant admin can configure an IdP metadata URL
- Users on an SSO-enabled tenant are redirected to the IdP instead of the password form
- Existing password login keeps working for non-SSO tenants
Out of scope: SCIM provisioning (that's ABC-1240), IdP-initiated flows.
```

**Alternative: step-by-step control (expanded profile).** For high-risk or large tickets, create artifacts one at a time so you can review each before the next is built on it:

```
/opsx:new abc-1234-add-sso-login       # scaffold folder + .openspec.yaml only
/opsx:continue                          # create the next ready artifact
/opsx:continue                          # ...and the next
```

`/opsx:continue` reads the dependency graph, shows what's ready versus blocked, and creates the first ready artifact using its dependencies as context. Several artifacts can become ready at once.

Or fast-forward everything at once (equivalent in spirit to `propose`):

```
/opsx:ff abc-1234-add-sso-login
```

| Situation | Use |
|---|---|
| Clear, small-to-medium ticket | `/opsx:propose` or `/opsx:ff` |
| Large, risky, or unfamiliar area | `/opsx:new` + `/opsx:continue` |
| Requirements still fuzzy | `/opsx:explore` first |

### Step 3 — Inspect what was produced

Move to the terminal. Read the artifacts yourself; do not approve them from the chat summary.

```bash
openspec status --change abc-1234-add-sso-login
openspec show abc-1234-add-sso-login
openspec validate abc-1234-add-sso-login --strict
```

`openspec status` prints artifact-by-artifact progress:

```
Change: abc-1234-add-sso-login
Schema: spec-driven
Progress: 4/4 artifacts complete

[x] proposal
[x] specs
[x] design
[x] tasks
```

Status markers:

| Marker | Meaning |
|---|---|
| `[x]` | Done |
| `[ ]` | Ready to be created |
| `[-]` | Blocked, with the blocking dependency named |
| `[~]` | Skipped, because the change declares `skip_specs: true` |

Other inspection commands:

```bash
openspec show abc-1234-add-sso-login --json --deltas-only   # just the spec deltas
openspec show auth --type spec                              # an existing main spec
openspec show auth --type spec --json --requirements        # requirements only
openspec view                                               # interactive dashboard
openspec instructions --change abc-1234-add-sso-login       # what OpenSpec thinks is next
```

### Step 4 — Human review gate **[Team convention — mandatory]**

**No code is written until a second human approves the plan.** This gate is where OpenSpec earns its cost. Approving a plan takes ten minutes; unwinding two days of confidently wrong AI implementation does not.

Reviewer checklist:

**Proposal**
- Does it reference the Jira key?
- Does it describe the same problem the ticket describes?
- Are non-goals stated explicitly?

**Specs (deltas)**
- Is every acceptance criterion in the ticket represented as a requirement?
- Are requirements testable, or are they vague aspirations?
- Are error paths, empty states, and permission boundaries covered?
- Do the deltas correctly say ADDED vs MODIFIED, and do MODIFIED entries match what's actually in `openspec/specs/` today?

**Design**
- Are alternatives and rejection reasons recorded?
- Migration and rollback addressed?
- Does it respect the conventions in `config.yaml`, or silently invent new patterns?

**Tasks**
- Is each task independently verifiable?
- Is there test coverage for each requirement?
- Anything obviously missing: migrations, feature flags, config, observability, docs?

**Scope**
- Is this one ticket's worth of work, or three tickets pretending to be one?

Record approval as a comment on the Jira ticket, linking the change folder. Include the reviewer handle and date in the PR description later.

> **Anti-pattern:** approving the plan and the code in the same review pass. The whole point is that the plan gate happens *first*, cheaply.

### Step 5 — Revise the plan when it's wrong

Do **not** hand-edit artifacts and hope they stay consistent with each other. Use:

```
/opsx:update abc-1234-add-sso-login — we're storing the session in a cookie, not localStorage
```

What it does: reads the change's artifacts, applies your revision, then reconciles the *other* artifacts in any direction — a design edit can ripple back into the proposal. It confirms each edit with you before writing, one artifact at a time. It touches planning artifacts only and never edits code. It finishes by recommending the next step.

Constraints worth knowing:

- It revises existing artifacts; it will not create missing ones (that's `/opsx:continue`).
- If the change is already implemented, follow up with `/opsx:apply` so the code catches up to the revised plan.
- If the revision changes the *intent* of the ticket, don't stretch the change — archive or abandon it and start a new one.

**Rule of thumb [Team convention]:** if the Jira ticket itself needs re-scoping, stop, fix Jira, then start a fresh change. A change folder that no longer matches its ticket is worse than no change folder.

### Step 6 — Implement

```
/opsx:apply abc-1234-add-sso-login
```

The assistant reads `tasks.md`, identifies incomplete tasks, works through them one at a time writing code and running tests, and marks each complete with `[x]`.

Because completion state lives in `tasks.md` checkboxes, it is **resumable** — if the session dies, run `/opsx:apply` again and it picks up where it stopped. Naming the change explicitly lets you run parallel changes without them confusing each other.

**Practices during apply [Team convention]:**

- Commit per task or per small group of tasks, referencing the ticket: `git commit -m "ABC-1234: validate SAML assertion signature"`
- Run the test suite yourself. Do not take "tests pass" on trust.
- If you discover the plan is wrong mid-implementation, **stop** and go back to `/opsx:update`. Don't let the code and the spec diverge — that's exactly the failure mode this process exists to prevent.
- If a task turns out to be unnecessary, delete it from `tasks.md` with a one-line note rather than ticking it off untouched.

### Step 7 — Verify (expanded profile)

```
/opsx:verify abc-1234-add-sso-login
```

It searches the codebase for implementation evidence and reports across three dimensions:

| Dimension | Question it asks |
|---|---|
| **Completeness** | Are all tasks done, all requirements implemented, all scenarios covered? |
| **Correctness** | Does the implementation match spec intent? Are the spec's edge cases handled? |
| **Coherence** | Are design decisions actually reflected in the code? Are patterns consistent? |

Findings come back as CRITICAL, WARNING, or SUGGESTION. **Verify does not block archive** — it surfaces issues and it is your job to act on them.

**Our policy [Team convention]:** zero CRITICAL findings before you open the PR. Each WARNING must be either fixed, or answered in the PR description with a reason. Common legitimate warnings: a design doc that describes an approach you deliberately changed during implementation — fix the design doc, don't fix the code to match a stale document.

Without the expanded profile, do this manually: re-read `specs/` and `design.md` next to your diff.

### Step 8 — Pre-PR checks

```bash
openspec validate abc-1234-add-sso-login --strict
openspec status --change abc-1234-add-sso-login     # expect all [x], all tasks ticked
# then your normal repo checks
npm run lint && npm test && npm run build
git push -u origin feat/abc-1234-add-sso-login
```

Open the PR using the description template in §5.

### Step 9 — PR review

The reviewer's advantage here is that intent is written down. Read the artifacts first, then the diff, and ask one question: **does this diff implement these artifacts?**

Reviewer checklist:

- Diff matches `tasks.md`; nothing significant is present that no task called for
- Delta specs accurately describe the behaviour change
- Tests exist for each requirement's scenarios
- No unrelated drive-by refactoring (that's a separate ticket)
- `openspec validate --strict` is green in CI

### Step 10 — Archive

Archive **after approval, before merge**, so the synced specs land in the same PR as the code that implements them.

```
/opsx:archive abc-1234-add-sso-login
```

or from the terminal:

```bash
openspec archive abc-1234-add-sso-login
openspec archive abc-1234-add-sso-login --yes        # non-interactive
```

What archive does:

1. Validates the change (skippable with `--no-validate`, which requires confirmation)
2. Prompts for confirmation, unless `--yes`
3. Merges the delta specs into `openspec/specs/`
4. Moves the folder to `openspec/changes/archive/YYYY-MM-DD-<name>/`

The slash-command version also checks artifact and task completion and warns — but does not hard-block — on incomplete tasks, and offers to sync the delta specs if you haven't already.

> **Non-interactive gotcha:** without a terminal — an AI agent, a CI job, or any run with stdin closed — archive cannot answer the confirmation prompt. It stops before touching anything, exits 1, and tells you to rerun with `--yes` and the change name. Pass `--yes` up front in automation.

Then commit and merge:

```bash
git add openspec/
git commit -m "ABC-1234: archive change, sync specs"
git push
```

Finally, move the Jira ticket to Done and paste the archive path into the ticket:

```
OpenSpec: openspec/changes/archive/2026-08-17-abc-1234-add-sso-login/
```

### Step 11 — Optional: sync without archiving

`/opsx:sync` merges delta specs into the main specs while leaving the change active. Archive prompts to sync when needed, so most people never call this directly.

```
/opsx:sync abc-1234-add-sso-login
```

The merge is intelligent rather than copy-paste: it parses ADDED / MODIFIED / REMOVED / RENAMED sections, can add scenarios to an existing requirement without duplicating it, and preserves existing content the delta doesn't mention.

Call it manually when:

| Scenario | Sync early? |
|---|---|
| Long-running change; you want the specs visible in main sooner | Yes |
| Parallel changes need the updated base specs | Yes |
| You want to review the merge separately from the archive step | Yes |
| Small change going straight to archive | No — archive handles it |

---

## 7. Special cases and recipes

### 7.1 Tickets with no behaviour change (refactor, tooling, docs, CI)

A change with **zero spec deltas fails validation** unless the change declares it. Set the flag in the change's `.openspec.yaml`:

```yaml
# openspec/changes/plat-501-extract-auth-lib/.openspec.yaml
skip_specs: true
```

Effects:

- `openspec status` shows `[~] specs (skipped: change declares skip_specs)` and excludes it from the progress count
- `openspec instructions specs` returns a warning only — the artifact must not be created
- The change archives with no extra flag

There is also a one-run escape hatch:

```bash
openspec archive plat-501-extract-auth-lib --skip-specs
```

**Prefer the declaration over the flag.** A change that permanently has no spec deltas should say so in its own metadata; the flag is for one-off runs and is easy to forget.

> If you find yourself reaching for `skip_specs` on something users can observe, that's a signal the ticket *does* have spec impact and you haven't articulated it yet.

### 7.2 Hotfixes and production incidents

Do not let process block a page. Sequence:

1. Fix production. Ship it. Normal incident process applies.
2. **Same or next business day**, create the retroactive change:

```bash
openspec new change ops-9911-fix-null-tenant-crash \
  --description "Retroactive spec for hotfix shipped 2026-08-17"
```

3. Have the assistant write the artifacts describing what was actually done, then archive it so the specs reflect reality.

A spec set that lies is worse than one with a gap. Backfilling within 24 hours is the discipline that keeps it honest.

### 7.3 Large tickets — split them

If a proposal produces more than roughly 15–20 tasks, or touches more than a couple of capabilities, split it in Jira and create one change per sub-ticket. Numeric prefixes are permitted, which makes ordering explicit:

```
100-abc-1235-sso-idp-config
200-abc-1236-sso-redirect-flow
300-abc-1237-sso-session-mapping
```

Sync each change as it completes so later changes build on the updated base specs.

### 7.4 Parallel changes touching the same specs

Two changes editing the same capability will conflict at sync time. Options, best first:

1. **Sequence them.** Archive the first, then let the second build on updated specs.
2. **Sync early.** Run `/opsx:sync` on the first as soon as its specs are settled.
3. **Bulk archive.** `/opsx:bulk-archive` lists completed changes, validates each, detects cross-change spec conflicts, inspects the codebase to resolve them, and archives in creation order — prompting before overwriting spec content.

Conflict resolution in bulk archive is agentic: it looks at what's actually implemented. Read its resolution plan before accepting it.

### 7.5 Adopting OpenSpec in an existing (brownfield) repo

Do **not** try to document the whole system up front. It won't get finished and it won't get read.

1. `openspec init`, then invest properly in `config.yaml` context (§4.6).
2. Leave `openspec/specs/` empty.
3. Run the process on new tickets only. Specs accumulate naturally as changes archive.
4. Optionally, backfill specs for the two or three areas that cause the most incidents — as their own tickets, sized and reviewed like anything else.

Point new joiners at `/opsx:onboard`, which walks the full workflow interactively using the real codebase: it scans for a genuine small improvement, creates a real change, implements it, and archives it, narrating each step. Budget 15–30 minutes. The change it creates is real, so keep or discard it deliberately.

### 7.6 Ticket types and how they map

| Jira issue type | OpenSpec treatment |
|---|---|
| Story / Feature | Standard flow. Full artifact set. |
| Bug | Standard flow. The spec delta usually MODIFIES an existing requirement, or ADDS the scenario that was missing. |
| Task (tooling/CI/build) | `skip_specs: true` |
| Spike | `/opsx:explore` only. No change folder. Output goes into the Jira ticket or a Confluence page. |
| Epic | Never a single change. One change per child ticket. |
| Hotfix | §7.2 — retroactive change. |

---

## 8. Definition of Ready / Definition of Done **[Team convention]**

### Definition of Ready — before `/opsx:apply`

- [ ] Jira ticket has acceptance criteria
- [ ] Change folder exists, named per §5
- [ ] All required artifacts created — `openspec status` shows no gaps
- [ ] `openspec validate <change> --strict` passes
- [ ] A second engineer or the tech lead has approved the plan, recorded in Jira
- [ ] Scope confirmed as one ticket's worth of work

### Definition of Done — before merge

- [ ] All tasks in `tasks.md` ticked
- [ ] Tests written and passing; coverage for each requirement's scenarios
- [ ] `/opsx:verify` shows zero CRITICAL; warnings fixed or answered in the PR
- [ ] `openspec validate <change> --strict` passes locally and in CI
- [ ] PR approved
- [ ] `/opsx:archive` run; delta specs synced into `openspec/specs/`
- [ ] Archive committed in the same PR as the implementation
- [ ] Jira updated with the archive path and moved to Done

---

## 9. CI enforcement **[Team convention]**

Validation is worthless if it's optional. Gate the PR.

```yaml
# .github/workflows/openspec.yml
name: OpenSpec

on:
  pull_request:
    branches: [main]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20.19.0'

      - name: Install OpenSpec
        run: npm install -g @fission-ai/openspec@latest

      - name: Validate all changes and specs
        run: openspec validate --all --strict --json
        env:
          OPENSPEC_TELEMETRY: '0'
          OPENSPEC_CONCURRENCY: '12'
```

The CLI exits `0` on success and `1` on error (validation failure, missing files), so a failing validation fails the job with no extra scripting.

**Suggested additional gates** — implement as small scripts against `openspec list --json` and `openspec status --change <name> --json`:

| Gate | Rule |
|---|---|
| Change exists | A PR whose branch matches `*/[a-z]+-[0-9]+-*` must have a matching active or newly archived change |
| Naming | Every active change name starts with a lowercased Jira key |
| Task completion | No PR merges with unticked tasks in its change |
| Archive hygiene | Nothing sits in `openspec/changes/` for more than N days without activity |

Environment variables available for CI:

| Variable | Effect |
|---|---|
| `OPENSPEC_TELEMETRY=0` | Disable telemetry |
| `DO_NOT_TRACK=1` | Disable telemetry (standard signal) |
| `OPENSPEC_CONCURRENCY` | Parallel validation limit (default 6) |
| `NO_COLOR` | Plain output for logs |

---

## 10. Command reference

### 10.1 Slash commands — core profile

| Command | Purpose |
|---|---|
| `/opsx:explore [topic]` | Investigate and think before committing to a change. Creates no artifacts. |
| `/opsx:propose [name-or-description]` | Create the change and generate all pre-implementation artifacts. |
| `/opsx:apply [change]` | Work through `tasks.md`, writing code and ticking tasks. |
| `/opsx:update [change]` | Revise planning artifacts and reconcile them with each other. Never edits code. |
| `/opsx:sync [change]` | Merge delta specs into main specs; change stays active. |
| `/opsx:archive [change]` | Finalise: check completion, offer sync, move to archive. |

### 10.2 Slash commands — expanded profile

| Command | Purpose |
|---|---|
| `/opsx:new [name] [--schema <name>]` | Scaffold the change folder and `.openspec.yaml` only. |
| `/opsx:continue [change]` | Create the next ready artifact in the dependency chain. |
| `/opsx:ff [change]` | Fast-forward: create all planning artifacts at once. |
| `/opsx:verify [change]` | Audit implementation against artifacts (completeness / correctness / coherence). |
| `/opsx:bulk-archive [changes...]` | Archive several completed changes, resolving spec conflicts. |
| `/opsx:onboard` | Interactive tutorial through a full cycle on the real codebase. |

The change name is optional on most of these — it's inferred from context. **Pass it explicitly** when you have multiple changes in flight.

### 10.3 Syntax by assistant

| Tool | Form |
|---|---|
| Claude Code | `/opsx:propose`, `/opsx:apply` |
| Cursor | `/opsx-propose`, `/opsx-apply` |
| Windsurf | `/opsx-propose`, `/opsx-apply` |
| GitHub Copilot (IDE only) | `/opsx-propose`, `/opsx-apply` |
| Trae, Oh My Pi | `/opsx-propose`, `/opsx-apply` |
| Codex | Skills under `.codex/skills/openspec-*` |
| Kimi Code | `/skill:openspec-propose`, `/skill:openspec-apply-change` |
| CodeArts | `/openspec-propose`, `/openspec-apply-change` |

Intent is identical across tools; only the surface differs. Note that Copilot's prompt files work in the VS Code / JetBrains / Visual Studio extensions but **not** in the Copilot CLI.

**Legacy commands** — `/openspec:proposal`, `/openspec:apply`, `/openspec:archive` — use the older all-at-once workflow. They still work and the artifact structure is compatible, so a legacy change can be continued with OPSX commands. Don't start new work on them.

### 10.4 CLI reference

**Setup**

| Command | Purpose |
|---|---|
| `openspec init [path]` | Initialise. `--tools <list>` `--profile <core\|custom>` `--force` |
| `openspec update [path]` | Regenerate assistant files after upgrade or profile change. `--force` |

**Browsing**

| Command | Purpose |
|---|---|
| `openspec list` | List active changes. `--specs` `--changes` `--sort recent\|name` `--json` |
| `openspec show [item]` | Show a change or spec. `--type change\|spec` `--json` `--deltas-only` `--requirements` `--no-scenarios` `-r <n>` `--no-interactive` |
| `openspec view` | Interactive terminal dashboard |

**Validation and lifecycle**

| Command | Purpose |
|---|---|
| `openspec validate [item]` | `--all` `--changes` `--specs` `--type` `--strict` `--json` `--concurrency <n>` `--no-interactive` |
| `openspec archive [change]` | `-y, --yes` `--skip-specs` `--no-validate` |

**Workflow**

| Command | Purpose |
|---|---|
| `openspec new change <name>` | Create change scaffolding. `--description` `--goal` `--schema` `--store` `--json` |
| `openspec status` | Artifact completion. `--change <id>` `--schema` `--json` |
| `openspec instructions [artifact]` | Enriched instructions for `proposal`/`specs`/`design`/`tasks`/`apply`. `--change` `--schema` `--json` |
| `openspec templates` | Resolved template paths. `--schema` `--json` |
| `openspec schemas` | Available schemas with their artifact flows. `--json` |

**Schemas**

| Command | Purpose |
|---|---|
| `openspec schema init <name>` | New project-local schema. `--description` `--artifacts <list>` `--default` `--force` `--json` |
| `openspec schema fork <source> [name]` | Copy a schema into the project to customise. `--force` `--json` |
| `openspec schema validate [name]` | Validate schema structure and templates. `--verbose` `--json` |
| `openspec schema which [name]` | Show where a schema resolves from. `--all` `--json` |

Schema precedence: project `openspec/schemas/<name>/` → user `~/.local/share/openspec/schemas/<name>/` → built-in package schemas.

**Config, health, utility**

| Command | Purpose |
|---|---|
| `openspec config path\|list\|get\|set\|unset\|reset\|edit` | Global configuration |
| `openspec config profile [preset]` | Configure workflow profile and delivery |
| `openspec doctor` | Read-only health report. `--store <id>` `--json` |
| `openspec context` | Assembled working set. `--json` `--code-workspace <path> [--force]` |
| `openspec feedback <message>` | File a GitHub issue. Requires an authenticated `gh` CLI. `--body <text>` |
| `openspec completion generate\|install\|uninstall [shell]` | Shell completions: bash, zsh, fish, powershell |

**Beta surfaces** — `openspec store setup|register|unregister|remove|list|doctor` and `openspec workset create|list|open|remove`. These let planning live in a standalone repo shared across a team, and let you reopen a set of folders by name. **Command names, flags, file formats, and JSON output may change between releases.** Do not build team process on them yet; revisit once they stabilise.

### 10.5 Which commands agents can drive

Interactive, human-only: `init`, `view`, `workset open`, `config edit`, `feedback`, `completion install`.

Agent-safe via `--json`: `list`, `show`, `validate`, `status`, `instructions`, `templates`, `schemas`, `new change`, and the store/workset read commands. Anything destructive in automation needs `--yes`.

---

## 11. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Slash commands not recognised | Assistant hasn't loaded generated skills | `openspec init` if never run; `openspec update` to regenerate; confirm `.claude/skills/` (or your tool's path) exists; **restart the assistant** |
| "Change not found" | Ambiguous or missing change | Pass the name explicitly (`/opsx:apply abc-1234-add-sso-login`); `openspec list`; check you're in the right directory |
| "No artifacts ready" | Everything is done or blocked | `openspec status --change <name>` to see blockers; create the missing dependency first |
| "Schema not found" | Typo or missing custom schema | `openspec schemas`; `openspec schema which <name>`; `openspec schema init <name>` if it should exist |
| Validation fails with no spec deltas | Change has no spec impact and hasn't declared it | Add `skip_specs: true` to the change's `.openspec.yaml` (§7.1) |
| Artifacts are thin, generic, or wrong | Assistant lacks project context | Fill in `config.yaml` `context:` and per-artifact rules; give more detail in the propose message; use `/opsx:continue` instead of `/opsx:ff` |
| Archive exits 1 in CI or an agent session | No terminal to answer the confirmation prompt | Rerun with `--yes` and the change name, keeping any other flags |
| Generated files differ between teammates | Profile is machine-global | Expected. Agree a team profile; each engineer runs `openspec config profile` then `openspec update` |
| Repo files out of sync with your profile | Profile changed without applying | `openspec update` |
| Specs don't reflect shipped behaviour | Changes implemented but never archived | Archive them; use `/opsx:bulk-archive` for a backlog; add the CI archive-hygiene gate (§9) |
| Sync produced a mangled spec | Delta sections mislabelled (ADDED vs MODIFIED) | Fix the delta, re-run sync; review the merge plan before accepting next time |

---

## 12. Anti-patterns

| Anti-pattern | Why it hurts | Do instead |
|---|---|---|
| Skipping the plan review gate | You've reintroduced "prompt and pray" with extra file overhead | Enforce §4 review; it's ten minutes |
| Hand-editing artifacts to keep the AI happy | Artifacts silently contradict each other | `/opsx:update` |
| One change spanning several tickets | Unreviewable; specs merge badly; can't revert cleanly | One ticket, one change |
| Implementing first, writing the spec after | The spec becomes fiction justifying the code | Spec first; hotfixes get a same-day retroactive change (§7.2) |
| `skip_specs: true` on user-visible behaviour | Your source of truth develops holes | Articulate the spec impact |
| Never archiving | `openspec/specs/` stays empty and the whole exercise was theatre | Archive as part of the PR |
| Empty `config.yaml` | Every ticket re-litigates your conventions | Invest once (§4.6) |
| Trusting `/opsx:verify` as a merge gate | It reports; it does not block | Read it; act on CRITICALs yourself |
| Building process on stores/worksets | Beta surface, subject to change | Wait for stability |
| Drive-by refactors during `/opsx:apply` | Diff no longer matches the tasks; review breaks down | Separate ticket |

---

## 13. FAQ

**Does this replace Jira?**
No. Jira remains the system of record for *what we're doing and when*. OpenSpec is the system of record for *what the software does*. The change name links the two.

**Do I have to use an AI assistant?**
The artifacts are plain Markdown, so you can write them by hand and use the CLI for validation, status, and archive. You lose most of the leverage, but the process is coherent without an AI.

**Doesn't this slow us down?**
It front-loads time into planning and takes it out of rework and review. Small, well-understood tickets are barely affected — `/opsx:propose`, skim, `/opsx:apply`. The payoff scales with ambiguity and blast radius.

**What if my ticket is a one-line config change?**
Judgement applies. Trivial changes with no behavioural impact don't need a change folder; use your normal PR flow. Write down where your team draws that line so it isn't relitigated per ticket.

**Where does design documentation live now — Confluence or `design.md`?**
`design.md` for decisions scoped to one change. Confluence for cross-cutting architecture. Link from `design.md` to Confluence, not the other way round; the change folder should be self-contained enough to read on its own.

**Can two people work on the same change?**
Yes, but coordinate: `tasks.md` checkboxes are the shared state and will conflict in Git if you both run `/opsx:apply` simultaneously. Split by task range, or by change.

**What happens to archived changes?**
They stay in `openspec/changes/archive/YYYY-MM-DD-<name>/` as an audit trail: original proposal, design, tasks, deltas. Excellent context when someone asks in eighteen months why a thing works the way it does.

**How do I see the full history of a capability?**
`openspec show <spec> --type spec` for current truth; `git log openspec/specs/<path>` plus the dated archive folders for how it got there.

---

## 14. Rollout checklist for a new repo **[Team convention]**

- [ ] Confirm Node ≥ 20.19.0 on all dev machines and in CI
- [ ] `openspec init --tools <team's tools>`
- [ ] Write `openspec/config.yaml` context; tech lead reviews it
- [ ] Agree the team profile; everyone runs `openspec config profile` + `openspec update`
- [ ] Confirm `openspec/` and generated tool directories are **not** gitignored
- [ ] Add the CI validation workflow (§9)
- [ ] Add the PR description template (§5) to `.github/PULL_REQUEST_TEMPLATE.md`
- [ ] Link this page from the repo README and the team onboarding page
- [ ] Each engineer runs `/opsx:onboard` once
- [ ] Pick two or three low-risk tickets as pilots before mandating the process
- [ ] Retro after two weeks; update this page

---

## 15. Sources and further reading

Official documentation (verify against your installed CLI version — this tool moves quickly):

- OpenSpec repository — https://github.com/Fission-AI/OpenSpec
- Slash command reference — https://github.com/Fission-AI/OpenSpec/blob/main/docs/commands.md
- CLI reference — https://github.com/Fission-AI/OpenSpec/blob/main/docs/cli.md
- Workflows — https://github.com/Fission-AI/OpenSpec/blob/main/docs/workflows.md
- Customisation (schemas, templates, per-artifact rules) — https://github.com/Fission-AI/OpenSpec/blob/main/docs/customization.md
- Getting started — https://github.com/Fission-AI/OpenSpec/blob/main/docs/getting-started.md
- Examples and recipes — https://github.com/Fission-AI/OpenSpec/blob/main/docs/examples.md
- Explore-first guide — https://github.com/Fission-AI/OpenSpec/blob/main/docs/explore.md
- Existing/brownfield projects — https://github.com/Fission-AI/OpenSpec/blob/main/docs/existing-projects.md
- Supported tools — https://github.com/Fission-AI/OpenSpec/blob/main/docs/supported-tools.md
- Stores (beta) guide — https://github.com/Fission-AI/OpenSpec/blob/main/docs/stores-beta/user-guide.md

Run `openspec <command> --help` for the authoritative flag list in your installed version.

---

## 16. Change log for this page

| Date | Author | Change |
|---|---|---|
| 2026-08-17 | Suman Jha | Initial draft |