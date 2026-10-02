Create a reusable Claude Code Issue Resolution Skill for this project.

PURPOSE

Automate the issue-resolution process currently performed manually during
Ticket Planning, Implementation, and Review.

Whenever one or more substantive issues are identified, the existing MAIN
Claude Code session must act as the persistent Staff Engineer Orchestrator and
drive the complete process:

validate issues
→ generate viable solutions
→ recommend solutions
→ independently re-verify recommendations
→ perform a final skeptical review
→ revise and repeat failed gates until they pass

The user should participate only when resolution genuinely requires information
or authority unavailable from the repository, requirements, project artifacts,
or existing conversation context.

Use clear, concise, plain language for explanations while retaining precise
technical terminology where required for evidence and correctness.

======================================================================
ARCHITECTURE
======================================================================

Use:

Main Claude Code session
= persistent Staff Engineer Orchestrator

Reusable Issue Resolution Skill
= defines the gates, retry behavior, and resolution procedure

Independent Gate Reviewer custom subagent
= independently and skeptically reviews each gate

The Staff Engineer Orchestrator is a ROLE performed by the existing main
Claude session. Do not create a separate Staff Engineer subagent.

Keep the Issue Resolution Skill in the main conversation:
- do not use `context: fork`
- do not set `disable-model-invocation: true`

The existing Ticket Planning, Implementation, and Review phases must be able
to invoke this Skill when issues are encountered.

Reuse one Independent Gate Reviewer definition, but invoke a NEW/FRESH reviewer
instance for every review attempt. Never resume a previous reviewer.

Process the COMPLETE current issue set together as one batch so duplicate
issues, shared root causes, dependencies, conflicting solutions, and
cross-issue effects can be evaluated together.

Follow all applicable CLAUDE.md/project instructions, including the project's
existing Andrej Karpathy-derived engineering rules. Do not invent or duplicate
those rules.

======================================================================
COMMON GATE LOOP
======================================================================

Every gate follows this pattern:

1. The SAME Staff Engineer produces or revises the complete gate result.
2. Invoke a FRESH Independent Gate Reviewer.
3. Reviewer returns:
   - PASS
   - FAIL
   - NEEDS_USER

PASS:
Advance to the next gate.

FAIL:
- Return specific findings and required changes to the same Staff Engineer.
- Revise the failed or affected portions.
- Preserve already-valid work unless a dependency invalidates it.
- Re-evaluate the complete batch for cross-issue consistency.
- Invoke another fresh reviewer.
- Repeat.

NEEDS_USER:
- The main Staff Engineer asks the user the smallest necessary clarification.
- After the answer, the SAME main session continues the affected gate with its
  existing context.

Use a configurable bounded retry limit; default to 3 review cycles per gate.

A gate exits only with:
- PASS
- NEEDS_USER
- UNRESOLVED after the retry limit

Do not escalate merely because the Staff Engineer and reviewer disagree.

======================================================================
GATE 1 — VALIDATE THE ISSUE SET
======================================================================

For all reported issues:

- normalize and deduplicate findings while preserving traceability
- explain each issue plainly
- inspect relevant code, requirements, tests, design/plan artifacts,
  configuration, documentation, repository behavior, and available evidence
- determine whether the issue genuinely exists
- identify root cause when supported
- distinguish facts from assumptions
- identify common causes and dependencies
- dismiss false positives

Classify each issue:

CONFIRMED
NOT_AN_ISSUE
UNCERTAIN

Classify evidence:

VERIFIED
= directly demonstrated by execution, tests, existing behavior, or equivalent
  direct evidence

STRONGLY_SUPPORTED
= strongly supported by repository evidence but not directly demonstrated

PLAUSIBLE
= technically reasonable but requiring further verification

Never call something VERIFIED merely because agents agree.

The reviewer must challenge:
- issue legitimacy
- evidence
- root cause
- assumptions
- missed evidence
- duplicate/related issue handling
- cross-issue relationships

Gate 1 passes when every issue has a defensible disposition and no material
validation finding remains unresolved.

An issue classified NOT_AN_ISSUE does not cause the gate to fail if that
classification is adequately supported.

======================================================================
GATE 2 — GENERATE AND RECOMMEND SOLUTIONS
======================================================================

For every CONFIRMED issue:

- generate up to 2–3 materially distinct, viable solutions
- never invent weak alternatives simply to reach three
- state explicitly if only one credible solution exists
- explain each option plainly
- show how it addresses the validated root cause
- evaluate correctness, risk, scope, maintainability, implementation impact,
  repository compatibility, and testability
- follow existing repository patterns and CLAUDE.md rules
- recommend the strongest solution

Evaluate the solutions collectively for:
- conflicts
- shared changes
- duplicated work
- cross-issue side effects
- opportunities for a simpler combined solution

The reviewer must challenge:
- solution viability
- root-cause coverage
- overlooked alternatives
- unnecessary complexity
- risk
- recommendation quality
- cross-solution conflicts
- requirements/design/CLAUDE.md compliance

Gate 2 passes only when the complete solution set and recommendations are
adequately supported.

======================================================================
GATE 3 — RE-VERIFY THE RECOMMENDATIONS
======================================================================

Do not trust Gate 2 recommendations merely because they passed once.

The Staff Engineer must deliberately re-evaluate every selected solution.

Verify:
- it addresses the validated root cause
- it fits the actual codebase
- it satisfies requirements and applicable design/plan constraints
- it follows CLAUDE.md/project rules
- it does not create unacceptable side effects
- it minimizes unnecessary complexity and scope
- no materially better, simpler, or safer alternative was overlooked
- the selected solutions work coherently together

For each recommendation record:
- selected solution
- why it was selected
- alternatives considered
- why alternatives were rejected
- evidence level
- implementation-time verification still required

Do not claim an unimplemented solution is proven to work.

Invoke a fresh reviewer to independently challenge these conclusions.

Gate 3 passes only when every recommendation and the combined solution set
survive independent re-verification.

======================================================================
GATE 4 — FINAL SKEPTICAL END-TO-END REVIEW
======================================================================

Perform an adversarial review of the entire resolution package:

- validated and dismissed issues
- evidence
- root causes
- assumptions
- alternatives
- recommendations
- rejected alternatives
- risks
- cross-issue interactions

The fresh reviewer must actively search for:
- false issues
- unsupported assumptions
- incorrect root causes
- weak evidence
- missing risks
- flawed solutions
- unnecessary complexity
- overlooked simpler solutions
- contradictions
- cross-solution conflicts
- violations of requirements or project rules

If Gate 4 fails, the reviewer must recommend the earliest affected gate.

The Staff Engineer must validate that routing recommendation rather than
blindly accepting it.

Routing:

validation/evidence problem
→ Gate 1

solution-set problem
→ Gate 2

recommendation problem
→ Gate 3

final-review-only problem
→ Gate 4

When an earlier gate is reopened, invalidate and rerun every downstream gate
whose conclusions depend on the changed result:

Gate 1 reopened → rerun 1 → 2 → 3 → 4
Gate 2 reopened → rerun 2 → 3 → 4
Gate 3 reopened → rerun 3 → 4
Gate 4 reopened → rerun 4

Finish only when Gate 4 passes.

======================================================================
REVIEWER RESPONSE CONTRACT
======================================================================

Require every reviewer invocation to return a consistent structured response.

This is a response contract, not assumed runtime schema enforcement.

Expected structure:

{
  "gate": "ISSUE_VALIDATION | SOLUTION_GENERATION |
           RECOMMENDATION_VERIFICATION | FINAL_SKEPTICAL_REVIEW",

  "overall_verdict": "PASS | FAIL | NEEDS_USER",

  "issues": [
    {
      "id": "...",
      "verdict": "PASS | FAIL",
      "feedback": ["..."],
      "required_changes": ["..."]
    }
  ],

  "cross_issue_findings": ["..."],

  "recommended_retry_target":
    "ISSUE_VALIDATION |
     SOLUTION_GENERATION |
     RECOMMENDATION_VERIFICATION |
     FINAL_SKEPTICAL_REVIEW |
     null",

  "plain_summary": "...",

  "user_question": null
}

If the response is malformed or materially incomplete, ask that fresh reviewer
to correct its response before using it for gate routing.

The reviewer identifies problems and recommends routing.
The Staff Engineer owns the engineering response and final routing decision.

======================================================================
USER ESCALATION
======================================================================

Make routine engineering decisions autonomously.

Ask the user only when resolution depends on unavailable information or
authority, for example:

- ambiguous or contradictory requirements
- missing business/product intent
- missing external constraints
- material scope expansion requiring authorization
- consequential behavior changes requiring explicit direction

Do not ask the user merely because:
- several technically valid choices exist
- an implementation decision is difficult
- Staff Engineer and reviewer initially disagree
- repository precedent provides sufficient guidance

======================================================================
CONTEXT AND PERSISTENCE
======================================================================

Keep the same main Claude session as Staff Engineer throughout the workflow.

Give each fresh reviewer only the context needed to independently assess the
current gate:
- relevant requirements
- relevant artifacts/code/evidence
- current gate result
- applicable project instructions

Do not provide unnecessary prior reviewer discussion or hidden reasoning that
could anchor the reviewer.

The reviewer returns its findings to the main Staff Engineer.

Do not rely exclusively on conversational context for durable decisions.
When appropriate, record validated decisions in the project's EXISTING
design/plan/decision artifacts.

Do not invent a new persistence convention when one already exists.

======================================================================
FINAL RESULT
======================================================================

After Gate 4 passes, provide a concise resolution package.

For every issue include:
- plain-language description
- final disposition
- evidence summary and level
- root cause when applicable
- viable solutions considered
- recommended solution
- selection rationale
- rejected alternatives and reasons
- remaining implementation verification

Also include:
- important cross-issue interactions
- shared risks
- unresolved assumptions, if any

Then continue the Ticket Planning, Implementation, or Review phase from the
point that invoked Issue Resolution.

======================================================================
IMPLEMENTATION
======================================================================

Before changing files:

1. Inspect CLAUDE.md, .claude/skills/, .claude/agents/, the existing Ticket
   Planning/Implementation/Review Skills, and existing project artifacts.

2. Follow current Anthropic Claude Code conventions. Do not invent unsupported
   metadata, APIs, tools, or lifecycle behavior.

3. Propose the minimal directory/file structure first.

4. Implement:
   - one directory-based Issue Resolution Skill running in the main conversation
   - one Independent Gate Reviewer custom subagent

5. Apply progressive disclosure:
   - keep SKILL.md focused on core orchestration
   - move detailed material into supporting resources only when justified
   - avoid both monolithic supporting files and unnecessary fragmentation

6. Configure the reviewer with least-privilege/non-mutating capabilities where
   practical. It may inspect and perform safe verification but must not edit
   implementation code, commit, push, or modify an MR.

7. Ensure the reviewer receives applicable CLAUDE.md/project instructions.
   Do not set `omitClaudeMd: true` unless there is a specific justified reason.

8. Do not introduce a Dynamic Workflow merely to duplicate this orchestration.
   If current Claude Code capabilities indicate a concrete advantage that still
   preserves the persistent main-session Staff Engineer requirement, explain
   the proposed change before implementing it.

======================================================================
VERIFY THE IMPLEMENTATION
======================================================================

Verify that:

- all three existing phases can invoke Issue Resolution
- the same main Staff Engineer retains the accumulated phase context
- all current issues are evaluated as one batch
- each gate has its own bounded retry loop
- no gate advances without PASS
- failed reviews return actionable feedback to the same Staff Engineer
- every review attempt uses a fresh reviewer instance
- already-valid results are preserved unless dependencies invalidate them
- Gate 4 routes backward correctly
- reopening a gate reruns dependent downstream gates
- false positives can cleanly become NOT_AN_ISSUE
- weak alternatives are not fabricated to reach 2–3 solutions
- VERIFIED requires direct evidence
- reviewer responses follow the response contract
- NEEDS_USER is reserved for genuine user-dependent decisions
- retry exhaustion results in UNRESOLVED rather than an endless loop
- important decisions use existing project persistence conventions
- the original phase resumes correctly after resolution






Main Claude session
= Staff Engineer Orchestrator
= retains accumulated context

        ↓
ALL issues as one batch

Gate 1: Validate issues
        ↓
Fresh Reviewer
   FAIL ↺ Staff Engineer revises
   PASS ↓

Gate 2: Generate 2–3 viable solutions
        + recommend one per issue
        ↓
Fresh Reviewer
   FAIL ↺ Staff Engineer revises
   PASS ↓

Gate 3: Re-verify recommendations
        ↓
Fresh Reviewer
   FAIL ↺ Staff Engineer revises
   PASS ↓

Gate 4: Skeptical end-to-end challenge
        ↓
Fresh Reviewer
   PASS → RESOLVED

   FAIL → route to affected gate
          ↓
          re-run that gate AND
          all dependent downstream gates

At any gate:
NEEDS_USER → ask you → same Staff Engineer continues
retry limit → UNRESOLVED