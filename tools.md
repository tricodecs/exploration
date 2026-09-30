
A Dynamic Workflow is Claude Code’s native way to move orchestration into executable JavaScript instead of relying on one Claude conversation to remember the sequence. The workflow script controls which subagents run, in what order, what loops repeat, what branches are taken, and what intermediate results are stored.

Core features

* agent() — launch one subagent.
* parallel() — run multiple agents concurrently.
* pipeline() — run an agent across a list of items.
* phase() — group stages in the workflow UI.
* Structured JSON outputs via schema.
* Normal JavaScript if, while, loops, variables, retries, etc.
* Background execution with /workflows monitoring.
* Runs can be paused/resumed within the same session.
* Saved workflows become reusable /commands.

Agents spawned by the workflow can read/write files, execute Bash, run tests, use MCP, edit code, etc. The workflow JavaScript itself only coordinates them.

Good use cases

Use it when you want deterministic orchestration such as:

implement
  ↓
run tests
  ↓
fail?
├─ yes → fixer → retest ─┐
└─ no                    │
  ↓                      │
review ←──────────────────┘
  ↓
findings?
├─ yes → fix → test → re-review
└─ no → finish

Anthropic specifically recommends workflows for repeated fix/check loops, large migrations, parallel audits, independent cross-review, and other tasks where the orchestration itself should be repeatable.

How to create one properly

The easiest/safest method is have Claude generate it first, rather than hand-writing the API from memory:

Use a dynamic workflow to implement the approved plan,
run tests until they pass or no progress is made,
perform an independent review,
fix validated findings,
and re-review until clean or 3 attempts are reached.

Claude creates and runs the script. Once it works:

1. Open /workflows.
2. Select the successful run.
3. Press s.
4. Save it to:
   * .claude/workflows/ for the project, or
   * ~/.claude/workflows/ for personal/global use.
5. It then becomes a command such as /dev-auto.

A saved workflow has this basic form:

export const meta = {
 name: 'dev-auto',
 description: 'Implement, test, review, and remediate a ticket'
}
phase('Implementation')
const implementation = await agent(
 'Implement the approved plan and run the required tests.',
 { schema: implementationSchema }
)
phase('Review')
let review = await agent(
 'Independently review the implementation.',
 { schema: reviewSchema }
)
while (!review.passed) {
 await agent(`Fix these findings: ${JSON.stringify(review.findings)}`)
 review = await agent(
   'Re-run tests and independently review the changes.',
   { schema: reviewSchema }
 )
}
return review

Anthropic recommends running /workflow-authoring before editing saved workflow JavaScript so Claude loads the current workflow API guidance; after editing, use /reload-skills before rerunning it.

One important limitation for your design: a Dynamic Workflow cannot normally stop in the middle and have a conversational interaction with you. For human approval between phases, split the process or let the main Claude session handle that boundary.

For your project, I would use Dynamic Workflow primarily for Auto mode and the test/fix/review loops, not for the interactive Jira clarification step.






Create a reusable Claude Code Dynamic Workflow for resolving issues encountered
during Ticket Planning, Implementation, or Review.

Purpose:
Allow the main Claude Code session to invoke one reusable issue-resolution
workflow whenever a phase identifies one or more issues.

Keep the main Claude session as the overall orchestrator and owner of project
context. Use the Dynamic Workflow only to investigate, validate, critically
review, and resolve the identified issues. Return a compact structured result
to the main session so it can continue the original phase.

Do not require the user to manually drive routine technical decisions.

Before implementation:
1. Inspect the existing:
  - CLAUDE.md
  - .claude/skills/
  - .claude/agents/
  - .claude/workflows/
  - existing ticket/design/plan/report artifacts
  - repository conventions
2. Use Anthropic's current Dynamic Workflow API and conventions.
3. Load/use the current workflow-authoring guidance before generating workflow
  code rather than inventing workflow APIs or parameters.
4. Reuse existing project conventions instead of creating unnecessary new ones.
5. Briefly propose the workflow structure and agent responsibilities before
  implementing it.

======================================================================
CALLING MODEL
======================================================================

The main Claude session invokes this workflow when ANY step of the Ticket,
Implementation, or Review phase produces substantive issues requiring analysis.

Pass only the context needed to resolve those issues, including when available:

- current phase
- current phase step
- Jira/ticket requirements
- relevant design.md / plan.md sections
- issue findings
- relevant files or code locations
- test/review findings
- previous relevant decisions
- repository constraints

Do not assume the workflow automatically inherits the complete parent
conversation.

Return all information that the main session needs to continue explicitly in
the workflow result.

======================================================================
STEP 1 — NORMALIZE ISSUES
======================================================================

Before deep analysis:

- Normalize the reported findings.
- Deduplicate findings that describe the same underlying problem.
- Preserve traceability from each normalized issue to the original finding(s).
- Do not discard conflicting findings; retain them for investigation.

Process every unique substantive issue.

======================================================================
STEP 2 — STAFF ENGINEER ISSUE INVESTIGATION
======================================================================

Use a fresh "Staff Engineer – Issue Investigator" agent.

Treat this as a senior engineering investigation, not a superficial review.

For EACH issue:

1. Explain the issue in clear, plain, easy-to-understand language.

  Explain:
  - what appears to be wrong
  - where it occurs
  - why it matters
  - what could happen if left unresolved

2. Verify whether the issue is actually real.

  Inspect relevant evidence such as:
  - source code
  - repository behavior
  - requirements
  - design/plan artifacts
  - tests
  - configuration
  - documentation
  - existing implementation patterns
  - version history when useful

  Never assume that a finding from another agent, reviewer, test, or tool is
  automatically correct.

3. Determine the root cause when evidence supports one.

4. Classify the issue as:
  - CONFIRMED
  - NOT_AN_ISSUE
  - UNCERTAIN

5. Distinguish evidence strength.

  Use:
  - VERIFIED:
    demonstrated by execution, test, prototype, existing implementation, or
    similarly direct evidence
  - STRONGLY_SUPPORTED:
    well supported by repository evidence but not directly executed
  - PLAUSIBLE:
    technically reasonable but still requiring implementation-time validation

  Do not call something "verified" merely because agents agree about it.

6. For each CONFIRMED issue, produce UP TO 2–3 materially distinct viable
  solutions.

  Do not invent weaker alternatives merely to reach a fixed count.
  If only one credible solution exists, state that explicitly.

For each proposed solution:

- explain it plainly
- explain why it addresses the root cause
- identify implementation impact
- identify meaningful tradeoffs
- identify risks
- identify compatibility/scope implications
- determine how it can be verified
- check that it fits the current codebase and requirements

Follow the engineering principles and repository-wide rules in CLAUDE.md,
including the existing Andrej Karpathy-derived rules.

Prefer solutions that:
- solve the actual root cause
- are simple and targeted
- minimize unnecessary abstraction
- minimize scope expansion
- follow existing repository patterns
- minimize unintended behavior changes
- are straightforward to test and verify

The investigator should be read-only by default.

Allow repository inspection and non-destructive verification commands when
needed, but do NOT:
- modify source code
- commit
- push
- create/update an MR
unless this workflow is explicitly redesigned later to perform remediation.

======================================================================
STEP 3 — INDEPENDENT STAFF ENGINEER CRITIQUE
======================================================================

Use a fresh "Independent Staff Engineer – Critic" agent.

Do not ask the original investigator to approve its own conclusions.

Give the critic:
- the normalized issue
- relevant requirements/context
- relevant repository evidence
- the investigator's findings
- proposed solutions

Do not give it unnecessary conversational reasoning.

Require the critic to independently challenge:

- whether the issue is actually real
- whether the claimed root cause is supported
- whether evidence is sufficient
- whether impact/severity was overstated or understated
- whether each proposed solution will actually solve the issue
- whether proposed solutions introduce new defects or unnecessary complexity
- whether important alternatives were missed
- whether a simpler or safer solution exists
- whether the solution violates ticket requirements
- whether it conflicts with design/plan artifacts
- whether it conflicts with repository conventions
- whether it violates applicable CLAUDE.md rules

The critic must distinguish facts from assumptions.

The critic should also be read-only by default.

======================================================================
STEP 4 — RECONCILE
======================================================================

Use deterministic workflow logic to reconcile investigator and critic results.

For each issue:

- If evidence shows it is not an issue:
 classify it as dismissed and explain why.

- If evidence remains insufficient:
 perform a bounded additional investigation.

- If independently validated:
 retain it as confirmed.

- If criticism invalidates a proposed solution:
 remove or revise that solution.

- If the critic identifies a materially better solution:
 evaluate it under the same standards before considering it.

Do not allow endless investigator/critic cycling.

Use a bounded retry/investigation limit and return an unresolved result if
confidence cannot be established.

======================================================================
STEP 5 — SELECT THE BEST ROUTINE TECHNICAL SOLUTION
======================================================================

For each confirmed issue, select the best supported solution automatically when
the decision is a routine engineering decision.

Do NOT ask the user to choose merely because multiple technically valid options
exist.

Evaluate candidate solutions against:

- correctness
- ticket requirements
- existing design/plan
- repository conventions
- CLAUDE.md rules
- simplicity
- maintainability
- risk
- reversibility
- scope
- testability
- evidence strength

Record:

- selected solution
- evidence level
- why it was selected
- alternatives considered
- why alternatives were rejected
- required verification during implementation

Do not claim that an unimplemented solution is proven to work.

Use STRONGLY_SUPPORTED or PLAUSIBLE where appropriate and require implementation
verification later.

======================================================================
STEP 6 — USER ESCALATION
======================================================================

Escalate ONLY when resolution genuinely depends on information or authority
Claude does not have.

Examples:

- ambiguous or contradictory product requirements
- missing user/business intent
- material scope expansion
- externally imposed constraints not represented in available project context
- breaking behavior requiring explicit authorization
- consequential decision that cannot be derived from requirements, code,
 project policy, or established conventions

Do NOT escalate because:
- several technically valid solutions exist
- the decision is an ordinary implementation detail
- an existing repository precedent provides a reasonable answer

If user input is required:

1. End the current Dynamic Workflow run cleanly.
2. Return:
  - status = NEEDS_USER
  - plain-language issue description
  - verified evidence
  - viable choices if relevant
  - the smallest necessary clarification question
  - all context required to continue
3. Let the MAIN Claude session ask the user.
4. After the user answers, invoke the issue-resolution workflow again with the
  new information in its arguments.

Do NOT assume a Dynamic Workflow can pause for normal conversational input and
then continue from the same execution state.

======================================================================
STEP 7 — RETURN STRUCTURED RESULTS
======================================================================

Return a concise structured result to the main Claude session.

Use structured output/schema support where available.

For each issue include:

- issue_id
- original_finding_ids
- plain_language_summary
- validation_status
- evidence_level
- evidence_summary
- root_cause
- solutions_considered
- critic_findings
- selected_solution
- selection_rationale
- rejected_alternatives
- implementation_verification_required
- needs_user
- user_question, if applicable
- information_needed_by_calling_phase

Also return an overall workflow status such as:

- RESOLVED
- PARTIALLY_RESOLVED
- NEEDS_USER
- UNRESOLVED

Do not return verbose hidden reasoning or unnecessary intermediate discussion.

======================================================================
CONTEXT AND PERSISTENCE
======================================================================

Keep the MAIN Claude session as the owner of overall Ticket → Implementation →
Review context.

Do not depend on subagent context for later phases.

The Dynamic Workflow should return the important resolution information to the
main session explicitly.

Do not make the issue-resolution workflow invent its own decision-log format.

After the workflow returns, let the main orchestrator persist important
decisions using the project's existing artifact/decision mechanism, if one
exists.

If no persistence mechanism exists, report that fact rather than silently
inventing one.

======================================================================
AGENT / CLAUDE.MD REQUIREMENTS
======================================================================

Ensure the Staff Engineer Investigator and Staff Engineer Critic operate under
the applicable repository engineering rules.

Prefer custom agent configurations that load/apply the project's CLAUDE.md and
required supporting skills.

Do not assume every built-in subagent type inherits all project instructions.

If a selected agent type does not receive the required CLAUDE.md instructions,
explicitly provide the necessary applicable rules rather than reconstructing or
inventing them.

======================================================================
IMPLEMENTATION REQUIREMENTS
======================================================================

Implement this as a reusable Claude Code Dynamic Workflow.

Use the workflow for:
- deterministic sequencing
- branching
- looping
- bounded retries
- intermediate structured state
- structured handoffs between agents

Use agents for:
- repository investigation
- technical reasoning
- evidence collection
- independent critique
- non-destructive verification

Keep workflow control logic separate from engineering reasoning.

Do not duplicate large policies already contained in CLAUDE.md or existing
project resources.

Do not create an additional Skill merely to wrap or duplicate this workflow
unless there is a separate reusable methodology that genuinely needs to be
consumed outside the Dynamic Workflow.

======================================================================
EXPECTED FLOW
======================================================================

Main Claude session
       ↓
Ticket / Implementation / Review phase
       ↓
one or more issues encountered
       ↓
invoke issue-resolution Dynamic Workflow
       ↓
normalize + deduplicate issues
       ↓
for each issue:

Staff Engineer – Issue Investigator
       ↓
plain-language explanation
       ↓
verify issue + evidence
       ↓
root cause
       ↓
up to 2–3 materially distinct viable solutions
       ↓
Independent Staff Engineer – Critic
       ↓
challenge issue + evidence + root cause + solutions
       ↓
reconcile
       ↓
confirmed?
┌──────────────┴──────────────┐
no / dismissed               yes
│                              ↓
record result          select best routine solution
                               ↓
                        needs user knowledge?
                        ┌──────┴──────┐
                       no            yes
                        │              │
                        ↓              ↓
                    resolved       NEEDS_USER
                        │              │
                        │       workflow run ends
                        │              ↓
                        │        main session asks user
                        │              ↓
                        │       invoke workflow again
                        │        with new information
                        │
                        └──────────────┐
                                       ↓
                            structured workflow result
                                       ↓
                              main Claude session
                                       ↓
                        persist important decision using
                          existing project convention
                                       ↓
                        continue original phase/step

======================================================================
FINAL VERIFICATION
======================================================================

Before declaring implementation complete:

- Verify the workflow can be invoked from Ticket Planning, Implementation, and
 Review.
- Verify the main session receives sufficient structured context to continue.
- Verify investigator and critic contexts remain appropriately independent.
- Verify issues are not automatically trusted.
- Verify duplicate findings are normalized.
- Verify the workflow does not fabricate 2–3 alternatives when fewer are viable.
- Verify "verified" is used only when supported by direct evidence.
- Verify routine technical decisions do not unnecessarily stop for the user.
- Verify genuinely user-dependent decisions return NEEDS_USER cleanly.
- Verify retry loops are bounded.
- Verify issue-resolution agents cannot accidentally modify production code,
 commit, push, or alter an MR.
- Verify the implementation uses the actual current Claude Code Dynamic Workflow
 API rather than assumed or invented APIs.
