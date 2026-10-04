Implement a reusable, automated Claude Code Issue Resolution workflow for this
project, integrated with searlsco/prove_it.

You have access to the current searlsco/prove_it repository. Treat its current
source and documentation as the source of truth for all prove_it commands,
configuration, hooks, Skills, signals, and behavior.

Before implementation, inspect the current prove_it repository and the target
project. If this prompt conflicts with the current prove_it implementation,
preserve the workflow intent but use the repository's current supported
mechanism and explain the adjustment.

Do NOT build the project's overall Ticket → Implementation → Review workflow.
This request is ONLY for a reusable Issue Resolution workflow that can be
invoked from any other workflow or Claude Code Skill.

======================================================================
GOAL
======================================================================

Automate a repetitive workflow that I currently perform manually whenever
Claude:

- identifies one or more issues/findings,
- proposes recommended solutions,
- gives a development response that contains questionable findings or
  recommendations,
- or produces changes that need validation.

Today I repeatedly ask Claude to:

1. explain the issues in plain, concise, easy-to-understand language;
2. verify that the issues are legitimate;
3. verify that the proposed/recommended solutions are valid;
4. verify compliance with applicable CLAUDE.md/project rules;
5. critically and skeptically challenge the issues and recommendations;
6. select the strongest solution unless user-only information is required;
7. implement the selected resolution;
8. verify the actual changes for correctness and rule compliance;
9. critically and skeptically review the actual implementation;
10. repeat the entire process if new or remaining issues are found.

Automate this complete cycle.

The workflow must continue until one of these terminal states:

COMPLETE
- no material unresolved issues remain;
- the selected resolutions were implemented;
- relevant verification passes;
- the final skeptical review finds no material unresolved problem.

NEEDS_USER
- resolution genuinely requires information, business/product intent,
  conflicting-requirement resolution, authorization, or another decision only
  the user can reasonably provide.

UNRESOLVED
- a genuine technical blocker remains after reasonable investigation and
  alternative approaches have been attempted.

Do NOT stop merely because a reviewer disagrees or because an arbitrary retry
count was reached.

======================================================================
ARCHITECTURE
======================================================================

Create/refactor a reusable Claude Code Issue Resolution Skill.

The existing MAIN Claude Code session remains the Staff Engineer and owns the
complete workflow, reasoning, implementation, routing, and accumulated context.

prove_it is the verification/enforcement layer, NOT the workflow orchestrator.

Use independent reviewers for critical review. A reviewer should challenge the
Staff Engineer, not simply repeat its reasoning.

Process all current related issues as ONE batch so the workflow can detect:

- duplicate findings;
- shared root causes;
- dependencies;
- conflicting recommendations;
- cross-issue side effects.

All explanations intended for the user should be concise, plain, and easy to
understand while remaining technically accurate.

Always inspect and follow the applicable CLAUDE.md hierarchy and other existing
project rules.

======================================================================
ISSUE RESOLUTION LOOP
======================================================================

The workflow is:

issues/findings/questionable recommendation
        ↓
1. VALIDATE ISSUES
        ↓
2. GENERATE + RECOMMEND SOLUTIONS
        ↓
3. RE-VERIFY RECOMMENDATIONS
        ↓
4. CRITICAL + SKEPTICAL PRE-IMPLEMENTATION REVIEW
        ↓
5. IMPLEMENT THE APPROVED RESOLUTION
        ↓
6. VERIFY THE ACTUAL CHANGES
        ↓
7. CRITICAL + SKEPTICAL POST-IMPLEMENTATION REVIEW
        ↓
new or remaining issues?
        ├─ YES → return to Step 1 with affected issue set ↺
        └─ NO  → final prove_it completion enforcement → COMPLETE

A later step may invalidate an earlier conclusion. When that happens, route
back to the earliest affected step and rerun everything downstream that depends
on it.

======================================================================
STEP 1 — VALIDATE ISSUES
======================================================================

Enter planning/analysis mode using the current supported prove_it phase:

    prove_it phase plan

For the complete issue set:

- normalize and deduplicate findings while preserving traceability;
- explain each issue plainly;
- inspect relevant code, tests, requirements, plans, documentation,
  configuration, runtime behavior, and other available evidence;
- determine whether each alleged issue actually exists;
- identify the root cause where evidence supports one;
- distinguish facts, evidence, assumptions, and speculation;
- identify relationships between issues;
- reject false positives.

Use sensible evidence classifications such as:

VERIFIED
= directly demonstrated by execution, tests, observable repository behavior,
  or similarly direct evidence.

STRONGLY_SUPPORTED
= strong repository evidence exists but the claim has not been directly
  demonstrated.

PLAUSIBLE
= reasonable hypothesis requiring further verification.

Do not treat agent agreement as evidence.

Result for each finding should be one of:

CONFIRMED
NOT_AN_ISSUE
UNCERTAIN

UNCERTAIN findings require further investigation unless only the user can
supply the missing information.

======================================================================
STEP 2 — GENERATE + RECOMMEND SOLUTIONS
======================================================================

For every CONFIRMED issue:

- produce up to 2–3 materially distinct viable solutions;
- do not invent weak alternatives just to reach three;
- if only one credible solution exists, say so;
- explain each solution plainly;
- show how it addresses the validated root cause;
- evaluate correctness, scope, complexity, maintainability, risk,
  testability, repository fit, and applicable CLAUDE.md rules;
- consider interactions across the entire issue batch;
- recommend the strongest solution.

Use prove_it's current `prove-approach` review where it adds value, especially
to challenge:

- root-cause assumptions;
- cognitive fixation;
- approach viability;
- overlooked structurally different alternatives.

Do not automatically accept `prove-approach`; treat it as independent evidence
and critically assess its findings.

======================================================================
STEP 3 — RE-VERIFY THE RECOMMENDATIONS
======================================================================

Deliberately challenge each selected recommendation again.

Verify:

- it actually addresses the confirmed root cause;
- it fits the real codebase;
- it satisfies relevant requirements;
- it follows applicable CLAUDE.md/project rules;
- it does not introduce unacceptable side effects;
- it is not unnecessarily complex;
- no materially simpler, safer, or stronger solution was overlooked;
- the recommendations work coherently together.

Use `/prove <claim>` selectively for important concrete claims that can be
demonstrated with evidence.

Examples:

- "This proposed change removes the root cause."
- "This change preserves backward compatibility."
- "This solution correctly handles condition X."

Do not mechanically invoke `/prove` for trivial statements.

======================================================================
STEP 4 — CRITICAL + SKEPTICAL PRE-IMPLEMENTATION REVIEW
======================================================================

This is mandatory.

Run a FRESH independent review of the complete proposed resolution package.

The reviewer must actively try to disprove or weaken the Staff Engineer's
conclusions.

Review:

- whether the alleged issues are actually real;
- strength of evidence;
- root-cause accuracy;
- assumptions;
- missed evidence;
- alternative explanations;
- proposed solutions;
- recommendation quality;
- unnecessary complexity;
- overlooked simpler approaches;
- cross-issue interactions;
- risks and side effects;
- applicable CLAUDE.md/project-rule compliance.

The reviewer must return actionable PASS/FAIL findings.

If FAIL:

- return the specific findings to the SAME main Staff Engineer;
- investigate/revise;
- route to the earliest affected step;
- repeat the affected steps;
- run another fresh skeptical review.

Do not implement until this review passes.

If no existing prove_it review Skill completely covers this purpose, create a
minimal project-specific independent reviewer rather than distorting an
unrelated built-in reviewer.

Reuse prove_it mechanisms for independent agent review where appropriate.

======================================================================
STEP 5 — IMPLEMENT THE APPROVED RESOLUTION
======================================================================

This means implementation in the GENERAL/local sense.

It is NOT the project's full Ticket Implementation phase.

It simply means applying the approved fixes necessary to resolve the current
issue set, which may involve code, tests, configuration, documentation, or
other project artifacts.

For behavior-changing code or bug fixes, use:

    prove_it phase implement

Use prove_it's current TDD/red-green enforcement where applicable.

For genuinely behavior-preserving restructuring, use:

    prove_it phase refactor

Do not label a functional behavior change as a refactor merely to bypass TDD
requirements.

Implement the smallest coherent solution that satisfies the validated
requirements and project rules.

======================================================================
STEP 6 — VERIFY THE ACTUAL CHANGES
======================================================================

Do not assume implementation correctness merely because the planned solution
was approved.

Verify the actual diff and behavior.

Use the project's real tests/checks and prove_it's supported verification
capabilities where appropriate.

Verify:

- the original confirmed issues are actually resolved;
- correctness;
- relevant requirements;
- relevant acceptance behavior;
- integration behavior;
- regressions;
- error/edge cases;
- applicable CLAUDE.md rules;
- unnecessary complexity;
- tests actually exercise the changed behavior.

Use prove_it built-ins where relevant:

`/prove <claim>`
- behavioral/acceptance claims that need direct evidence.

`/prove-test-validity`
- determine whether tests provide genuine evidence rather than false
  confidence.

`/prove-coverage`
- identify meaningful coverage gaps for changed behavior.

`/prove-done`
- broad correctness, integration, security, testing, and omission review.

`/prove-dry`
- optional for substantial changes where duplicated behavior is a meaningful
  concern; do not require it mechanically for every tiny change.

Also use the project's actual `script/test_fast`, `script/test`, or equivalent
prove_it-configured deterministic checks where applicable.

Passing tests alone is NOT sufficient proof that the original issue was
correctly resolved.

======================================================================
STEP 7 — CRITICAL + SKEPTICAL POST-IMPLEMENTATION REVIEW
======================================================================

This second skeptical evaluation is mandatory and is distinct from Step 4.

Step 4 challenged the PROPOSED solution.
Step 7 challenges what was ACTUALLY implemented.

Use a FRESH independent reviewer.

Critically examine:

- the actual diff;
- whether the implementation truly resolves each original issue;
- whether requirements were preserved;
- whether tests provide convincing evidence;
- regressions;
- integration problems;
- security or reliability concerns where relevant;
- unnecessary complexity;
- incorrect assumptions;
- CLAUDE.md/project-rule violations;
- missing edge cases;
- new issues introduced by the fix;
- whether a simpler implementation now appears preferable.

If ANY material new or remaining issue is found:

1. collect the complete affected issue set;
2. feed it back into Step 1;
3. repeat:
   validate
   → solutions
   → re-verification
   → skeptical review
   → implementation
   → implementation verification
   → skeptical implementation review.

Continue until clean.

======================================================================
NO-PROGRESS / STUCK HANDLING
======================================================================

Do not use an arbitrary retry limit as the normal stopping mechanism.

If the Staff Engineer is genuinely cycling or making no progress:

- use the current documented `prove_it signal stuck` behavior where useful;
- use `prove-approach` to challenge the current direction;
- gather new evidence;
- try a materially different viable approach.

Return UNRESOLVED only when there is a real technical blocker and no reasonable
next action remains.

If the missing information is user-owned rather than technically unavailable,
return NEEDS_USER instead.

======================================================================
USER ESCALATION
======================================================================

Make ordinary engineering decisions autonomously.

Do NOT ask the user merely because:

- several viable technical solutions exist;
- a decision is difficult;
- a reviewer disagrees;
- implementation requires normal engineering judgment.

Choose the strongest supported solution.

Ask the user only when resolution depends on something Claude cannot reasonably
derive, such as:

- conflicting or ambiguous requirements;
- missing business/product intent;
- external constraints unavailable to the repository;
- authorization for a material scope/behavior change;
- another decision that genuinely belongs to the user.

After the user answers, continue the same Issue Resolution cycle from the
affected step.

======================================================================
prove_it COMPLETION ENFORCEMENT
======================================================================

The Issue Resolution Skill owns all INTERNAL routing and loops.

Do not misuse prove_it Stop hooks as internal step transitions.

Use prove_it's manual/built-in reviewers and deterministic checks during the
internal workflow.

Only when Steps 1–7 indicate that the entire current resolution cycle is
complete should Claude declare the coherent unit of work done using the
current documented mechanism:

    prove_it signal done

Configure/use appropriate signal-gated completion checks so that a final Stop
cannot succeed prematurely.

At minimum, final completion enforcement should include the project's relevant
full deterministic tests and a strong independent completion review.

Where appropriate, include a project-specific Issue Resolution completion
reviewer in addition to prove_it's built-in `prove-done`, because generic
pre-ship verification may not know the original issue set and chosen
resolutions.

Respect prove_it's documented semantics:

- `signal done` activates done-gated tasks on the next Stop;
- if a Stop task FAILs, completion is blocked;
- its failure evidence is returned to Claude;
- the done signal is preserved after failure;
- Claude fixes/revises;
- the gated checks run again on the next Stop;
- the signal clears after a successful Stop.

Therefore the final behavior should be:

Claude believes resolution is COMPLETE
        ↓
prove_it signal done
        ↓
full checks + independent completion review
        ↓
FAIL
        ↓
failure evidence returned to same Claude
        ↓
feed material issue(s) back into appropriate Issue Resolution step
        ↓
fix / verify / skeptical review
        ↓
try completion again
        ↺
        ↓
PASS
        ↓
COMPLETE

Use prove_it's reviewer backchannel if a reviewer finding is demonstrably
inapplicable rather than modifying correct code merely to satisfy an incorrect
review.

======================================================================
REVIEW CONTEXT
======================================================================

Independent reviewers must have enough context to make meaningful judgments,
but should not depend on hidden reasoning from the main conversation.

Maintain a compact durable/transient Issue Resolution state artifact containing
at least:

- original findings;
- current issue dispositions;
- evidence/root causes;
- solutions considered;
- selected recommendations and rationale;
- applicable requirements;
- current implementation changes;
- verification/test evidence;
- unresolved findings;
- current workflow step.

First reuse an existing project workflow/state convention if one exists.

If none exists, create the smallest project-local transient artifact needed for
reviewers to independently inspect the current resolution state.

Do not assume prove_it reviewer subprocesses automatically receive the main
Claude conversation.

Ensure independent reviewers can read:

- this resolution state;
- relevant repository files;
- applicable CLAUDE.md/project rules;
- relevant diffs and test evidence.

Use prove_it `ruleFile` where appropriate, but do not assume a single ruleFile
automatically represents all nested/applicable CLAUDE.md instructions.

======================================================================
AUTOMATIC INVOCATION
======================================================================

Design the Issue Resolution Skill so Claude can invoke it automatically when a
substantive development response contains:

- one or more issues/findings;
- competing recommended fixes;
- a meaningful questionable technical recommendation;
- failed verification;
- review findings requiring remediation.

Avoid triggering the expensive workflow for trivial informational responses
that contain no actionable development finding or recommendation.

The Skill must also be manually invocable.

======================================================================
IMPLEMENTATION REQUIREMENTS
======================================================================

Before changing anything:

1. Inspect the target project's:
   - applicable CLAUDE.md files;
   - `.claude/skills/`;
   - `.claude/rules/`;
   - `.claude/agents/` if present;
   - `.claude/prove_it/`;
   - current prove_it configuration;
   - test scripts;
   - existing hooks;
   - existing workflow/state conventions.

2. Inspect the CURRENT searlsco/prove_it repo/docs and verify every prove_it
   command/configuration feature used here.

3. Run `prove_it doctor` if prove_it is installed and available.

4. Preserve existing working configuration and behavior.

5. Do not blindly run `prove_it init`, `reinit`, or overwrite customized
   configuration.

6. Do not invent prove_it commands, conditions, signals, or APIs.

7. Reuse built-in prove_it Skills where they actually fit instead of copying
   their behavior.

8. Add custom review logic only where the Issue Resolution requirements are
   not covered by prove_it's existing reviewers.

9. Keep reviewer permissions least-privilege/read-only where practical.

10. Use current Claude Code Skill conventions and progressive disclosure.
    Keep the primary Issue Resolution SKILL.md focused on orchestration and move
    detailed review criteria/supporting material into supporting resources only
    when justified.

======================================================================
BEFORE IMPLEMENTING
======================================================================

First provide a concise design showing:

- files to create or modify;
- Issue Resolution Skill structure;
- any custom independent reviewer required;
- prove_it built-ins reused;
- prove_it config changes;
- resolution-state mechanism;
- how Steps 1–7 loop;
- how final `signal done` enforcement works;
- how NEEDS_USER and UNRESOLVED work.

Then implement it.

======================================================================
FINAL VERIFICATION
======================================================================

Do not declare the implementation complete from static inspection alone.

Demonstrate that:

- the Issue Resolution Skill can be invoked;
- it processes multiple related issues as one batch;
- Steps 1–4 occur before implementation;
- the pre-implementation skeptical review is mandatory;
- implementation uses the correct prove_it phase;
- the actual implementation is independently verified;
- the post-implementation skeptical review is mandatory;
- new findings return to Step 1;
- the cycle continues until clean;
- ordinary reviewer disagreement does not unnecessarily involve the user;
- user-only decisions produce NEEDS_USER;
- genuine no-progress technical blockers can produce UNRESOLVED;
- applicable CLAUDE.md rules are considered;
- independent reviewers receive sufficient explicit context;
- `prove_it signal done` is used only at the genuine completion boundary;
- a failing final prove_it check blocks completion and feeds useful evidence
  back into the workflow;
- final PASS means the full resolution cycle, not merely the tests, is clean.

If the current prove_it implementation differs from any assumption in this
prompt, use the current repository behavior and report the difference rather
than fabricating unsupported functionality.



