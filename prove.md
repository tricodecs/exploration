

Critical assessment
The previous prompt is directionally correct, but I would revise it before using it. It addresses Claude’s decision authority, yet it risks introducing so much evaluation and review that Claude becomes slower, more hesitant, and overly focused on process.
Anthropic’s current best practices favor concise persistent instructions, repository-grounded decisions, executable verification, and independent reviews focused on material defects. They specifically warn that excessive review can produce unnecessary findings and overengineering.
Problems I found
Area	Critical finding	Recommended correction
Decision authority	Says Claude can make routine decisions, but doesn’t clearly establish that it should choose and proceed without approval.	Make autonomous selection and execution the default.
Decision procedure	Requiring 2–3 alternatives for nontrivial decisions could waste time and create artificial tradeoffs.	Examine alternatives proportionally; use one obvious solution when appropriate.
Escalation threshold	“Significant architectural change” is too subjective. Claude might still ask about minor internal design decisions.	Distinguish local implementation design from cross-system architectural changes and breaking contracts.
Review overhead	Fresh skeptical reviewers before and after every significant decision may duplicate work and increase false positives.	Use lightweight internal scrutiny for routine choices; independent review for substantial risks, complex findings, and completed meaningful changes.
Review authority	The prompt doesn’t clearly state that reviewer findings can be wrong.	Require evidence; Staff Engineer independently verifies findings and rejects false positives.
CLAUDE.md conflicts	The proposed priority list could conflict with existing project instructions.	Detect and reconcile contradictory rules. Don’t simply claim one file overrides another.
Workflow integration	Doesn’t explicitly preserve your Standard phase approvals and required Ticket clarification.	Make those exceptions explicit.
Effectiveness testing	Asking Claude to verify hypothetical scenarios may result in Claude merely asserting that the instructions will work.	Test the behavior in actual or controlled Claude Code sessions.
Two additional findings matter.
First, Claude Code’s .claude/rules/ files are an appropriate way to make a decision policy persistent. Unscoped rules load automatically, but they are instructions, not deterministic enforcement. Anthropic recommends Skills for procedures and hooks for checks that must happen every time.
Second, the earlier /doctor prompt-audit recommendation is valid in current Claude Code versions 2.1.283 and later. It is worth using to detect instruction conflicts and bloat.
Improved prompt
This version concentrates on making Claude behave like a Staff Engineer with delegated engineering authority, rather than turning every implementation decision into a formal review process.
Implement a persistent Staff Engineer Decision-Making and Autonomy Policy for this Claude Code project.
Objective
Claude currently asks me to make routine technical decisions, choose between proposed solutions, and repeatedly request critical verification—even when it has enough repository context and engineering knowledge to decide independently.
I want Claude to act like a competent Staff Engineer who investigates, decides, implements, verifies, and proceeds autonomously.
I should participate only when the decision genuinely requires my knowledge or authority, involves unresolved business/product intent, or requires a material architectural or contractual change.
This policy must work automatically in my existing Claude Code VS Code main chat session, including within existing Skills.
1. Inspect existing instructions
Before modifying anything, inspect:
* Root and applicable nested CLAUDE.md files.
* .claude/rules/, Skills, and relevant agents.
* Existing issue-resolution/reviewer mechanisms.
* prove_it, if installed.
* Existing development workflow conventions.
Read the actual Andrej Karpathy-derived engineering rules already present in CLAUDE.md. Preserve their meaning; do not invent or duplicate them.
Identify instructions that unnecessarily require user approval or contradict autonomous engineering decision-making.
Follow current official Anthropic Claude Code conventions.
2. Create a concise persistent autonomy policy
Create .claude/rules/engineering-decisions.md, or update an existing equivalent if one is already present.
The policy must apply automatically to the main Claude Code conversation.
It must explicitly establish:
The default action is to investigate, make a justified engineering decision, and proceed—not ask the user to select a solution.
Autonomous decisions
Claude is authorized to independently:
* Choose implementation approaches, internal patterns, algorithms, and data structures.
* Choose among technically viable solutions.
* Make routine design decisions within the existing architecture.
* Select tests and verification strategies.
* Fix bugs, test failures, and verified review findings within task scope.
* Perform minimal necessary internal refactoring.
* Update implementation plans/checklists when validated discoveries require it.
* Reject invalid reviewer findings using evidence.
Follow these principles:
1. Satisfy the task requirements and acceptance criteria.
2. Respect applicable project instructions, security constraints, and established contracts.
3. Reuse existing repository conventions and architecture when appropriate.
4. Prefer the simplest correct solution that fully addresses the root cause.
5. Minimize code changes, dependencies, abstractions, and unrelated modifications.
6. Follow the project’s existing Karpathy-derived engineering rules.
7. Use tests and observable evidence to establish correctness.
Do not blindly follow a repository convention if it prevents a correct, secure solution. Investigate the conflict and prefer the smallest justified deviation within authorized scope.
Do not introduce speculative improvements or architectural redesigns.
Decision process
For routine, low-risk decisions:
* Follow established patterns.
* Choose the obvious sound solution.
* Proceed without formal option comparison or additional approval.
For complex or uncertain technical decisions:
1. Investigate the actual code and requirements.
2. Validate the reported issue/root cause.
3. Compare materially different viable solutions when useful—not mechanically.
4. Recommend and select the strongest solution.
5. Critically challenge that selection, including simpler alternatives and unintended consequences.
6. Proceed with implementation once adequately supported.
7. Verify the result against actual requirements and behavior.
Do not present me with an options menu when Claude has sufficient evidence and authority to decide.
3. Explicit escalation boundaries
Claude must independently handle ordinary technical decisions, including local internal design changes.
Escalate ONLY when investigation establishes that:
* Requirements or intended product behavior remain materially ambiguous.
* Requirements conflict and repository evidence cannot resolve them.
* The necessary solution involves a major architectural departure, cross-system redesign, breaking API/contract change, or consequential data migration beyond authorized scope.
* A material scope expansion or irreversible/destructive external action requires authorization.
* A security, compliance, or business-policy decision requires user authority.
* A genuine blocker cannot be resolved using available evidence, tools, or reasonable alternatives.
A difficult engineering problem alone is NOT grounds for escalation.
Before escalating:
* Investigate relevant code, tests, documentation, existing decisions, and project conventions.
* Consider a simpler in-scope alternative.
* Determine whether the uncertainty is actually user-owned.
If escalation is unavoidable, explain the problem, impact, and recommended action concisely, and ask the minimum necessary question.
Never bypass tool permissions or established authorization requirements.
4. Automated skeptical evaluation
Integrate with existing Issue Resolution and reviewer mechanisms where present.
For substantive findings or meaningful changes, perform:
Before implementation:
* Verify issues and root causes.
* Evaluate and select solutions.
* Critically challenge recommendations, assumptions, complexity, and risks.
After implementation:
* Verify actual behavior, tests, and project-rule compliance.
* Critically challenge the resulting changes for regressions, incomplete fixes, and unnecessary complexity.
* If material issues remain, validate them, fix them, and repeat verification.
Use fresh independent review when the complexity or risk warrants it, especially for significant changes and final correctness verification.
Reviewers must identify evidence-backed problems affecting correctness, requirements, security, or established project rules—not speculative style preferences.
Claude must independently validate reviewer findings. Reject false positives with evidence rather than implementing unnecessary changes.
Do not create endless review loops. If progress stalls, reconsider the underlying assumptions or approach; escalate only for a genuine blocker.
Reuse existing prove_it or review infrastructure when appropriate. Do not create overlapping hook systems.
5. Preserve existing workflow behavior
Apply the decision policy to the existing Ticket Planning, Implementation, and Finalize/Review Skills.
* Ticket Planning: retain interactive requirements clarification with me.
* Implementation: choose technical solutions, implement, test, fix findings, and reconcile plan.md/design.md when necessary.
* Finalize: independently evaluate review findings, fix legitimate issues, and verify changes.
For the E2E orchestrator:
* STANDARD mode still requires user approval at completed phase boundaries.
* AUTO mode advances between phases without approval.
* Both modes retain requirements clarification and genuine NEEDS_USER escalation.
Routine engineering decisions must not create additional approval checkpoints in either mode.
Keep the main Claude session responsible for decisions and context.
6. Communication behavior
During engineering work:
* Make justified decisions and continue without asking me to confirm them.
* Explain issues in clear, concise, plain language.
* Avoid repetitive option lists and unnecessary deliberation reports.
* Report important decisions and their rationale at meaningful checkpoints.
* Show verification evidence when claiming work is complete.
* Distinguish verified results from assumptions.
7. Implement and evaluate
Use the smallest maintainable change to existing project instructions and Skills.
Do not install another orchestration framework merely for this decision policy.
Before finishing:
1. Check the resulting instructions for conflicts, redundancy, and ambiguity.
2. Run /doctor prompt-audit if supported by the installed Claude Code version.
3. Exercise representative decision scenarios in controlled or actual sessions:
    * two valid approaches → choose autonomously;
    * familiar repository pattern → reuse it;
    * valid reviewer finding → fix automatically;
    * false-positive finding → reject with evidence;
    * local design choice → decide without asking;
    * genuinely ambiguous product requirement → ask me;
    * unavoidable major architecture change → seek approval;
    * implementation completed → verify before claiming success.
4. Distinguish behavior actually tested from behavior only expected based on the instructions.
5. Report the files changed, any remaining conflicts, and evidence that autonomous decision-making improved.
The success criterion is not merely that the new rule exists.
Success means Claude makes sound routine engineering decisions, performs the necessary work, independently challenges important decisions and changes, and involves me only when my participation is genuinely necessary.
Why this is more effective
The crucial improvement is proportional engineering judgment.
Routine implementation choice
Choose, implement, verify, continue
Complex technical decision
Investigate, compare alternatives, skeptically evaluate, decide, proceed
Genuine user-owned or major architectural decision
Explain the tradeoff, recommend an option, and ask you
This follows Anthropic’s recommendation to avoid over-specified instructions and excessive adversarial reviews while giving Claude concrete ways to verify its work.
My conclusion: Start with this decision policy, not another autonomous-agent repository. Your existing engineering Skills already define how the work should proceed. What is missing is the explicit authorization for Claude to make the technical decision and execute it without treating you as the default decision-maker.






Recommendation: use 
prove_it
to enforce the Staff Engineer policy, not to make decisions
I reviewed the current searlsco/prove_it⁠ repository and Anthropic’s Claude Code documentation.
prove_it is a strong complement to your Staff Engineer autonomy policy, but it is not a decision-making orchestrator. It runs independent reviewers, tests, and lifecycle checks. When a completion check fails, it returns actionable feedback so Claude can correct its work rather than declaring success.
Your desired behavior should be:
MAIN CLAUDE SESSION — STAFF ENGINEER
        ↓
Investigate the issue
        ↓
Evaluate and select the strongest solution
        ↓
Independent skeptical review
        ↓
Implement without asking for routine approval
        ↓
Tests + independent implementation review
        ↓
Failures?
  YES → Claude verifies findings, fixes, retries ↺
  NO
        ↓
prove_it completion enforcement
        ↓
PASS → Complete
Which 
prove_it
components to use
Component	Where to use it	Purpose
ruleFile + custom agent reviewer	Important engineering decisions and completion reviews	Independently check the chosen solution against your engineering decision policy and applicable project rules.
/prove-approach	Before implementing a complex or disputed solution	Challenge root cause, fixation on one approach, and overlooked simpler alternatives.
/prove	Important correctness or compatibility claims	Require evidence that the claim is true, including attempts to disprove it.
prove_it phase implement	Code changes	Apply its TDD guidance and red/green tracking.
script/test_fast and script/test	During and after coding	Run actual deterministic checks instead of trusting Claude’s judgment.
/prove-test-validity, /prove-coverage	After meaningful changes	Challenge whether the tests are convincing and adequately cover changed behavior.
/prove-done	Final completion review	Check correctness, integration, security, and omissions.
signal done + Stop checks	Genuine completion boundary	Block premature completion and return failure evidence to Claude.
Review backchannel	Reviewer returns an incorrect finding	Let Claude contest the finding with evidence instead of making an unnecessary fix.
These are documented components; the built-in reviews can be invoked as Skills or configured as independent reviewer tasks.
The distinction matters: your decision policy authorizes Claude to choose; prove_it checks whether that choice and the resulting implementation are adequately justified.
How I would configure it
1. Reinforce autonomous decisions before implementation
Use your persistent .claude/rules/engineering-decisions.md policy to establish that Claude must select routine technical solutions itself.
For complex decisions, have the main-session Staff Engineer invoke /prove-approach and evaluate its findings. The reviewer can question whether the root cause is correct, whether Claude overlooked a simpler solution, and whether its preferred approach is sound.
The main Claude session still makes the final engineering decision. A review failure does not automatically mean the user must decide.
2. Add one custom completion reviewer
I would add a staff-engineer-completion reviewer to the existing .claude/prove_it/config.json.
Example task to merge into the existing hooks.claude.Stop array:
{
  "name": "staff-engineer-completion",
  "type": "agent",
  "model": "sonnet",
  "when": {
    "signal": "done"
  },
  "ruleFile": ".claude/rules/engineering-decisions.md",
  "prompt": "Independently verify completion. Inspect the task requirements, acceptance criteria, applicable CLAUDE.md files, repository conventions, actual changes, and test evidence. Confirm that the implementation is correct, minimal, in scope, and satisfies the requirements. Check that material technical choices have a sound engineering rationale and that no routine decision was unnecessarily left for the user. Challenge unsupported claims, incomplete fixes, regressions, unnecessary abstractions, and rule violations. Return PASS if adequately verified. Return FAIL only for concrete material problems, including specific evidence and required corrections. Do not demand user approval for ordinary implementation choices.\n\nWork completed: {{signal_message}}\n\nChanges: {{session_diff}}\n\nTest evidence: {{test_output}}"
}
This uses documented prove_it capabilities: independent agent tasks, ruleFile, template variables, and signal conditions. It is an illustrative task entry, not a complete replacement configuration.
For reliable results, the Staff Engineer must make the actual task requirements accessible to the reviewer, using existing plan.md, acceptance criteria, or a concise work-summary artifact. It cannot assume that an independent reviewer knows everything discussed in the main chat.
3. Enforce the fix-and-verify loop
AT A GENUINE COMPLETION BOUNDARY
Staff Engineer signals completion
prove_it signal done
prove_it runs checks
Tests + prove-done + custom Staff Engineer reviewer
PASS
Allow completion
FAIL
Return evidence to Claude
Claude corrects legitimate findings and retries
On failure, prove_it preserves the completion signal so the configured checks run again on the next Stop. This is the enforcement behavior you want.
For your full E2E Auto mode, don’t use a Stop-hook pass to advance from Ticket to Implementation or Implementation to Finalize. Run internal checks directly; reserve Stop enforcement for true stopping points. In Standard mode, a completed phase can be a genuine stopping point.
Important limitation for your main concern
prove_it does not automatically prevent Claude from asking you unnecessary engineering questions. Its default checks examine code, tests, and completion evidence; they aren’t a continuous judge of whether a conversational question should have been asked.
You need one additional rule in the main-session decision policy:
Before asking the user to choose an implementation approach, independently determine whether the choice is an ordinary engineering decision. If it is, investigate the available evidence, select the strongest solution, and proceed. Only ask when the unresolved question genuinely requires user-owned knowledge, authority, or approval for a material architectural change. Explain why escalation is necessary when it occurs.
An optional dedicated escalation reviewer could strengthen this, but I would first test whether the persistent rule eliminates the behavior without adding another hook.
What I would deploy initially
Only three pieces:
* Staff Engineer decision policy — Claude is explicitly authorized to make routine technical choices.
* Selective /prove-approach review — challenges complex recommendations before Claude implements them.
* Existing prove_it tests + prove-done + one custom completion reviewer — validates actual work and drives corrections on failed completion checks.
That is enough for an initial implementation. You do not need a new orchestrator, four more reviewers, or a separate autonomous agent.
One caution: the prove_it repository still warns about frequent unstable releases. Pin a tested version and validate hook behavior before using it as a mandatory completion gate.
My recommendation: Make the autonomy policy responsible for decisions and prove_it responsible for evidence and completion enforcement. This is simpler, more reliable, and less likely to introduce another layer of review that slows Claude down or makes it ask you even more questions.








I would integrate them as two shared capabilities used by all three of your existing Claude Code Skills, without replacing those Skills.
* Staff Engineer autonomy policy: Tells the main Claude Code session what decisions it should make independently, how to choose solutions, and when it must consult you.
* prove_it: Supplies independent reviews, tests, verification evidence, and mechanisms that prevent Claude from claiming completion prematurely.
Your existing Skills remain responsible for their own internal steps, including calling other Skills.
Recommended architecture
MAIN CLAUDE CODE CHAT — STAFF ENGINEER
  Persistent engineering decision policy
        ↓
TICKET PLANNING SKILL
  → Internal steps / supporting Skills
  → Requirements clarification with YOU
  → Automated decisions and issue resolution
  → Verify design.md + plan.md
        ↓
  STANDARD: stop for phase approval
  AUTO: continue
        ↓
IMPLEMENTATION SKILL
  → Follow plan.md checklist
  → Decide implementation approaches
  → Implement, test, resolve issues
  → Revise plan/design if discoveries require it
  → Verify implementation
        ↓
  STANDARD: stop for phase approval
  AUTO: continue
        ↓
PRE-MR REVIEW SKILL
  → Independent code review
  → Validate and resolve findings
  → Re-test and skeptically review changes
  → Verify acceptance criteria and MR readiness
        ↓
  COMPLETE / existing MR handoff
At any point, an issue-resolution cycle can run:
Validate issue → choose solution → skeptical review → apply fix → verify changes → skeptical review → repeat if necessary.
The same Staff Engineer retains responsibility for decisions. Independent reviewers return findings, not choices for you to make.
Exact integration points
Phase	Staff Engineer’s responsibility	prove_it tools and checks
1. Ticket Planning	Independently analyze technical approaches, ask you to clarify requirements, create design.md and plan.md, resolve planning issues.	prove_it phase plan, /prove-approach for complex design choices, selective /prove for important technical claims, custom planning reviewer for requirements/design/plan consistency.
2. Implementation	Choose implementation approaches, execute checklist, make code changes, fix issues, update design/plan when necessary.	prove_it phase implement or refactor, script/test_fast, full tests, /prove-test-validity, /prove-coverage, /prove-done.
3. Pre-MR Review	Review code and requirements, verify findings, independently select and implement fixes, repeat until the changes are ready for MR.	prove_it phase plan for read-only review; switch to implement for fixes; /prove-done, /prove, tests, and a custom acceptance-criteria reviewer.
Final completion	Determine whether all requirements and review findings are resolved.	prove_it signal done with configured completion checks at a genuine stopping point.
These commands and reviewer capabilities are documented in the prove_it repository⁠.
What happens when an issue appears in any Skill?
Issue detected during any phase
Staff Engineer makes the decision
Validate issue → assess solutions → choose best → skeptically evaluate recommendation
Apply the fix
Change planning documents during Ticket Planning; code/tests during Implementation or Pre-MR Review as appropriate
prove_it + independent review
Verify actual changes, test evidence, and CLAUDE.md compliance; skeptically challenge implementation
FAIL
Return findings to Staff Engineer; resolve and repeat
PASS
Continue the original phase automatically
The reviewer should not ask you to select a solution. It should return evidence-backed concerns to the Staff Engineer, who determines whether they are legitimate and makes the correction.
What to change in your existing Claude Code project
You need four targeted changes—not a new E2E framework.
1. Add a persistent Staff Engineer policy
Create an unscoped .claude/rules/engineering-decisions.md file so Claude loads it automatically into the main chat session. It should explicitly authorize routine coding decisions, require the simplest correct implementation compatible with repository rules, and reserve escalation for requirements, user-owned decisions, major architecture changes, or genuine blockers. Anthropic documents this as the appropriate use of project rules.
2. Update your three existing phase Skills
Add short instructions at their decision and verification points:
* Apply the Staff Engineer policy automatically.
* Invoke the reusable Issue Resolution Skill for substantive findings.
* Never defer an ordinary technical choice to the user.
* Continue the phase after successful verification.
Keep the phase Skills running in the main conversation for context continuity. Do not add context: fork to them.
3. Configure prove_it once at project level
In .claude/prove_it/config.json, configure real test commands, relevant source/test globs, and only the independent completion reviewers you need. Use a custom reviewer for checking task acceptance criteria and engineering rules, because prove-done alone cannot know requirements that were never supplied to it.
Make the original Jira criteria, design, plan, and test evidence readable to reviewers. Use ruleFile for the autonomy policy where appropriate, but ensure nested CLAUDE.md rules are also considered.
4. Distinguish internal verification from actual completion
This is critical for Auto mode.
A Stop hook runs when Claude finishes its response—not when a Skill finishes an internal step. For intermediate steps, invoke checks and reviewers directly, then continue. At an actual session stopping boundary, prove_it signal done can activate heavyweight checks. If a check fails, prove_it returns feedback and preserves the signal for retry.
Also, prove_it phase plan disables its TDD guidance; it does not automatically disable all configured Stop-time tests and reviewers. Scope those checks carefully so a requirement-clarification question does not trigger an inappropriate completion gate.
How I would initially deploy it
Keep the setup minimal:
* One always-loaded Staff Engineer decision rule.
* Your existing three Skills, with explicit issue-resolution and verification checkpoints.
* One reusable Issue Resolution Skill with both skeptical reviews—before and after changes.
* prove_it tests, selective built-in reviewers, and one task-aware completion reviewer.
Then test it with a real Jira task where Claude previously asked you to choose between implementation solutions. The success test is whether it now chooses, implements, verifies, and continues without asking you, while still stopping for required clarification or significant architectural decisions.
That is the behavior you want, rather than merely having more review prompts.











Critical assessment
The previous integration design is sound, but it is not yet strong enough to reliably produce the Staff Engineer behavior you want.
The biggest weakness is that it explains which tools Claude should use, but not precisely enough how Claude must independently make decisions, reject unnecessary review findings, and continue work without involving you.
I checked the current Anthropic Claude Code best practices and searlsco/prove_it documentation. Anthropic recommends concise persistent instructions, focused Skills, independent verification, and executable checks. It also warns that excessive instructions can reduce reliability.
1. Findings from the skeptical review
Area	Assessment	Necessary correction
Staff Engineer decision authority	Incomplete	Explicitly require Claude to select and execute the strongest in-scope solution, not just recommend one.
When to involve you	Too ambiguous	Clearly distinguish routine internal design choices from material architecture, API, data, security, or product changes.
Integration with your three Skills	Incomplete	Audit every internal step and supporting Skill for instructions that force unnecessary user choices.
Automatic issue resolution	Correct concept	Preserve both skeptical reviews and the fix/reverify loop, but trigger it only for substantive findings.
Use of prove_it	Needs refinement	Distinguish direct mid-work verification from Stop-hook completion enforcement.
Independent reviewer	Good	Give reviewers actual requirements, relevant code, evidence, and applicable rules. Do not assume they receive the main conversation.
Plan/design reconciliation	Correct	Retain automatic plan.md checklist updates and design reconciliation during implementation.
Effectiveness verification	Insufficient	Test actual Claude behavior, not merely that rules or Skills exist.
2. The most important changes
A. Make independent decision-making an explicit requirement
I would strengthen the persistent rule to:
Staff Engineer decision authority: For any in-scope technical decision, investigate available evidence, select the strongest solution consistent with requirements, project rules, and existing architecture, then implement and verify it without asking the user for approval. Do not present alternative solutions for the user to select unless the decision requires their intent or authority.
This is stronger than simply saying “be autonomous.”
The decision process should be proportional:
Routine technical decision
Choose the simplest correct solution using repository conventions. Implement and test immediately.
Complex technical decision
Investigate, compare viable options, critically challenge the recommendation, make the decision, and proceed.
Genuine user-owned decision
Escalate only when the answer requires your product intent, business knowledge, authority, or approval of a material architectural/contractual change.
This makes choosing and proceeding the normal behavior, rather than asking by default.
B. Apply the policy inside every existing Skill
This is the most important missing integration requirement.
Your Ticket Planning, Implementation, and Pre-MR Review Skills each contain multiple steps and call supporting Skills.
One of those supporting Skills may still say:
* Present alternatives and wait for the user to choose.
* Ask for confirmation before implementing a recommendation.
* Stop when review discovers new issues.
An autonomy rule elsewhere does not guarantee that those contradictory instructions will disappear.
The coding agent must inspect and reconcile instructions throughout the Skill hierarchy, while preserving the Ticket requirements-clarification step and Standard-mode phase approvals. Anthropic confirms that persistent rules are advisory instructions rather than enforced configuration.
C. Use 
prove_it
selectively, not at every decision
I would use these exact capabilities:
Workflow point	prove_it component	Role
Ticket planning	phase plan	Suspend TDD enforcement during planning.
Complex proposed solution	/prove-approach	Challenge fixation, root cause, and better alternatives.
Important technical claim	/prove	Gather evidence for and against correctness.
Implementation	phase implement or refactor	Appropriate test-first or refactoring guidance.
During coding	script/test_fast	Fast failure evidence.
After meaningful changes	/prove-test-validity, /prove-coverage	Verify test quality and adequacy.
Before completion	/prove-done and task-specific reviewer	Skeptical correctness/requirements review.
Genuine completion boundary	signal done + Stop checks	Block unsupported completion and send failure feedback.
These are supported in the current repository. Importantly, phase plan controls TDD behavior; it does not automatically disable other configured Stop checks.
Do not put every skeptical review behind signal done. In your Auto E2E workflow, Claude needs to review and continue working inside phases without stopping. Invoke those reviews directly; reserve Stop hooks for genuine completion points.
D. Review both the solution and the actual changes
Your full Issue Resolution loop should remain:
Validate issues
Select and re-verify solutions
Skeptical review of proposed solution
Apply validated resolution
Verify actual changes
Skeptical review of implemented changes
Issues remain → repeat from validation
No material issues → return to the calling phase
Use a fresh reviewer for the two skeptical assessments. Review findings must be evidence-based, and the Staff Engineer should independently verify them before modifying code.
E. Enforce acceptance criteria, not just code quality
prove-done and passing tests do not guarantee that the Jira ticket’s requirements were fulfilled.
Add explicit phase-specific completion criteria:
* Ticket: requirements → design.md → plan.md consistency.
* Implementation: completed checklist, actual behavior, tests, acceptance criteria, and reconciliation of modified design/plan artifacts.
* Pre-MR Review: final diff, valid review findings resolved, regression checks, project-rule compliance, and MR readiness.
For independent reviewers, pass an explicit requirements/acceptance summary or let them read the actual artifacts. prove_it agent tasks run as independent processes with explicitly supplied prompts and context variables.
3. Revised Staff Engineer policy to add to the integration
The following is the essential addition I would make to the earlier implementation prompt:
Staff Engineer Autonomy and Decision Enforcement
The primary objective is to eliminate unnecessary user involvement in engineering decisions throughout the existing Ticket Planning, Implementation, and Pre-MR Review Skills.
Default behavior: investigate → decide → implement → verify → continue.
1. The main Claude Code session acts as the Staff Engineer and is responsible for selecting solutions, not merely presenting alternatives.
2. For routine technical choices, follow existing repository conventions and the simplest correct approach. Proceed without asking.
3. For complex choices, independently evaluate viable alternatives, select the strongest supported solution, skeptically challenge it, and proceed.
4. Follow applicable CLAUDE.md rules, including the existing Andrej Karpathy-derived principles. Avoid unnecessary abstractions, unrelated refactoring, speculative changes, and architecture expansion.
5. Escalate only when the decision requires unavailable product/business intent, conflicts with requirements, exceeds authorized scope, involves a material architectural or contractual change, or reaches a genuine blocker.
6. Before asking the user, explicitly check whether the question can be answered from the repository, requirements, tests, conventions, or reasonable engineering judgment. If yes, decide autonomously.
7. Inspect all three phase Skills and their supporting Skills. Remove or reconcile instructions that unnecessarily request user selection or routine approval. Preserve mandatory Ticket requirements clarification and Standard-mode phase approvals.
8. Automatically invoke the existing Issue Resolution workflow for substantive actionable findings. Perform skeptical evaluation before applying resolutions and again after verifying the actual changes. Repeat for legitimate remaining issues.
9. Use prove_it for evidence, independent review, testing, and completion enforcement. It must not replace the Staff Engineer’s decision-making authority.
10. Verify work against task requirements, acceptance criteria, actual behavior, and project rules—not just reviewer agreement or passing tests.
11. Do not create redundant reviewers or multiple competing Stop-hook retry systems.
12. Keep decisions and explanations concise. Report important choices and evidence at phase checkpoints rather than asking the user for routine approval.
Success criterion: Claude independently resolves ordinary engineering choices, applies validated fixes, rejects false-positive review findings, repeats verification until material issues are resolved, and only contacts the user when their knowledge or authority is genuinely required.
Validate this behavior in actual or controlled Claude Code sessions. Do not claim success based only on inspecting the policy files.
4. Final recommendation
I would keep the architecture minimal:
Persistent decision policy → grants Staff Engineer authority.
Your existing three phase Skills → execute their internal workflows, with conflicting manual-decision instructions removed.
Reusable Issue Resolution Skill → performs iterative validation, selection, implementation, and two skeptical reviews.
prove_it → supplies independent verification, deterministic checks, and completion enforcement.
The main limitation remains that Claude Code rules and Skills guide behavior but do not guarantee it. prove_it cannot automatically prove that Claude made the optimal decision, nor can it reliably prevent every unnecessary question. Test representative sessions and refine the policy based on actual outcomes.
My assessment: The previous integration design was largely correct. These changes make it more effective by assigning decision authority explicitly, reconciling the existing Skills, and using prove_it only where it provides measurable verification rather than adding redundant review overhead.





================================





Use the following in your Claude Code VS Code chat extension, opened at the root of your project.
The prompt is designed to integrate four things without replacing your existing work: a persistent Staff Engineer decision policy, your three sequential phase Skills, a reusable issue-resolution loop, and prove_it verification.
I checked the current Anthropic guidance: unscoped .claude/rules/*.md files load automatically, Skills can run in the main conversation, and prove_it supports the phases, independent reviewers, tests, signals, and completion checks described below.
1. Copy and paste this prompt
Implement Staff Engineer Autonomy and prove_it Integration
Objective
Upgrade my existing Claude Code project so the main Claude Code chat session acts as an autonomous Staff Engineer, making routine software-engineering decisions without requiring me to repeatedly choose solutions, request critical reviews, or manually drive issue resolution.
Integrate this behavior into my three existing custom Skills:
1. Ticket Planning
2. Implementation
3. Pre-MR Review / Finalize
Each Skill already contains multiple internal steps and may invoke supporting Skills. Preserve their existing behavior unless changes are necessary to accomplish this objective.
Use searlsco/prove_it for evidence-based verification, independent review, and completion enforcement.
Default behavior: investigate → decide → implement → verify → continue.
I should participate only for requirements clarification, genuinely user-owned decisions, authorization for consequential changes, or unresolved blockers.
Before modifying anything, inspect the current project and read the official Claude Code documentation and the current searlsco/prove_it repository. Do not invent unsupported APIs or assume the existing Skills have a particular structure.
 
⸻
 
1. Persistent Staff Engineer decision policy
Create or update an unscoped project-level rule:
.claude/rules/engineering-decisions.md
This rule must load automatically into the main Claude Code session and apply across all three Skills and their supporting procedures.
Decision authority
Claude must independently:
* Verify whether reported technical issues are legitimate.
* Investigate root causes using repository evidence.
* Evaluate and select implementation approaches.
* Choose between technically viable solutions.
* Make routine internal design and implementation decisions.
* Follow suitable repository conventions.
* Select the simplest correct and maintainable solution.
* Implement validated resolutions.
* Run relevant tests and verification.
* Evaluate review findings and reject false positives.
* Correct legitimate failures and continue working.
Do not present implementation options and wait for my selection when Claude has sufficient information and authority to make the decision.
Engineering principles
Follow the existing CLAUDE.md files, including their Andrej Karpathy-derived engineering rules.
Read the actual rules; do not invent, paraphrase incorrectly, or duplicate them unnecessarily.
In general:
1. Satisfy requirements and acceptance criteria.
2. Respect applicable project rules, security constraints, and established contracts.
3. Follow existing architecture and repository patterns when appropriate.
4. Prefer the simplest correct solution.
5. Make minimal, focused changes.
6. Avoid unnecessary abstractions, dependencies, unrelated refactoring, and speculative improvements.
7. Verify behavior with evidence instead of relying on confidence or reviewer agreement.
If an existing convention conflicts with a correct implementation, investigate and choose the smallest justified deviation within authorized scope.
Proportional decision-making
For straightforward decisions:
* Follow established patterns.
* Select the obvious sound solution.
* Implement and verify without formal option comparisons.
For complex decisions:
* Investigate the actual code and requirements.
* Validate the suspected issue and root cause.
* Consider materially distinct viable solutions when useful.
* Select and re-verify the strongest recommendation.
* Critically and skeptically challenge its correctness, complexity, alternatives, and risks.
* Make the decision and proceed without unnecessary user approval.
Do not invent multiple alternatives merely to reach a prescribed number.
User escalation boundaries
Ask me ONLY when:
* Product or business requirements remain materially ambiguous.
* Requirements conflict and available evidence cannot resolve them.
* Information genuinely exists only with the user.
* The necessary change requires a major architectural departure, breaking contract/API change, consequential data migration, or material scope expansion beyond existing authorization.
* Security/compliance policy or irreversible/destructive actions require explicit authority.
* A genuine blocker remains after investigating reasonable alternatives.
A difficult engineering choice or reviewer disagreement is NOT a reason to escalate.
Before asking me, determine whether repository evidence, requirements, existing patterns, tests, or engineering judgment can resolve the question.
If escalation is necessary, clearly explain the issue, recommend a solution where possible, and ask the minimum essential question.
Do not bypass Claude Code permissions, security restrictions, or existing required approvals.
 
⸻
 
2. Integrate the policy into existing Skills
Inspect ALL three phase Skills, their supporting Skills, and relevant project instructions.
Find instructions that unnecessarily require me to:
* choose technical solutions;
* approve ordinary implementation approaches;
* manually request verification;
* ask for skeptical reviews;
* decide how to fix reviewer findings;
* repeatedly instruct Claude to continue after recoverable failures.
Remove or reconcile these instructions with the Staff Engineer policy.
Preserve legitimate requirements clarification, explicit authorization requirements, and existing workflow capabilities.
Do not duplicate the complete decision policy inside every Skill. Use a short reference or integration instruction where needed.
Keep the main Claude Code conversation as the coordinating Staff Engineer. Do not unnecessarily fork the phase Skills into isolated agents.
Ticket Planning Skill
Preserve its existing multi-step workflow and supporting Skills.
It must:
* retrieve and understand the task/Jira ticket;
* perform the existing interactive requirements-clarification step with me;
* independently evaluate technical approaches;
* resolve substantive planning issues;
* create and verify design.md and plan.md;
* ensure the plan’s implementation checklist covers the task requirements and acceptance criteria.
Requirements clarification remains interactive in BOTH Standard and Auto modes.
Routine technical decisions must not create additional user-approval stops.
Issue-resolution changes during this phase normally affect planning/design artifacts, not production feature code.
Implementation Skill
Preserve execution of the plan.md task checklist.
Claude must:
* independently choose implementation details;
* make code/test/configuration changes;
* run verification;
* resolve findings automatically;
* critically review its changes;
* continue until the implementation satisfies its acceptance criteria.
If implementation discoveries invalidate the plan:
1. Validate the issue.
2. Select and implement the appropriate resolution.
3. Update plan.md checklist items when tasks change.
4. Update design.md if the technical design changes.
5. Preserve traceability for materially superseded tasks.
6. Verify ticket requirements ↔ design ↔ plan ↔ actual code.
7. Continue from the revised checklist.
Do not restart Ticket Planning for ordinary technical changes.
Escalate only if discoveries materially alter requirements, product behavior, authorization, or architecture outside the approved scope.
Pre-MR Review / Finalize Skill
Preserve its existing internal review and MR-preparation steps.
Claude must:
* review actual code changes;
* validate findings rather than blindly trusting reviewers;
* select fixes independently;
* implement legitimate corrections;
* rerun tests and verification;
* perform skeptical evaluation of the resulting changes;
* repeat until no material unresolved findings remain;
* verify MR readiness against task requirements.
Preserve existing commit, push, MR description, and MR creation behavior. Do not add duplicate GitLab operations.
 
⸻
 
3. Reusable automated Issue Resolution Skill
Inspect whether a reusable Issue Resolution Skill already exists.
Refactor it if present; otherwise create the smallest appropriate model-invocable Skill.
It must be callable automatically from ANY internal step of Ticket Planning, Implementation, or Pre-MR Review when substantive findings arise.
Do not trigger it on trivial informational responses or recursively invoke another instance while one resolution cycle is already active.
Process the current issues as one batch, considering dependencies, shared causes, duplicate findings, and conflicting solutions.
The same main-session Staff Engineer owns the process and decisions.
Complete resolution cycle
Step 1 — Validate issues
* Explain issues in concise, plain language.
* Verify each reported issue actually exists.
* Inspect relevant evidence and establish root causes.
* Separate confirmed issues, false positives, and uncertain findings.
* Follow applicable project rules.
Step 2 — Evaluate and choose solutions
* Validate any existing recommended solutions.
* If necessary, generate up to 2–3 materially distinct viable options.
* Compare correctness, simplicity, risk, scope, maintainability, and repository compatibility.
* Independently select the best justified solution.
* Do not ask me to make routine selections.
Step 3 — Re-verify the recommendation
Challenge whether the selected solution:
* actually resolves the root cause;
* is compatible with the repository;
* follows CLAUDE.md;
* avoids unacceptable side effects;
* is simpler and safer than meaningful alternatives.
Step 4 — Independent skeptical review before changes
Invoke a fresh independent reviewer to critically challenge:
* issue legitimacy;
* evidence/root causes;
* assumptions;
* proposed solutions;
* selected recommendations;
* overlooked alternatives;
* unnecessary complexity;
* cross-issue conflicts;
* project-rule compliance.
If the review finds material problems, return actionable findings to the same Staff Engineer.
Validate the findings, revise the affected work, and re-review.
Do not proceed until the proposed resolution is adequately supported.
Step 5 — Apply the resolution
Implement the validated solution in the artifact appropriate to the calling phase.
Ticket: normally design/plan/analysis documents.
Implementation: code, tests, configuration, documentation.
Pre-MR Review: actual fixes to code/tests/configuration/review artifacts.
Step 6 — Verify actual changes
Confirm:
* the original issue was resolved;
* changes are correct;
* requirements are preserved;
* relevant tests/checks pass;
* no material regressions were introduced;
* applicable project rules are satisfied.
Step 7 — Independent skeptical review after changes
Use a fresh independent reviewer to challenge the ACTUAL changes, not merely the original proposed solution.
Look for:
* incomplete fixes;
* regressions;
* invalid tests;
* new issues;
* incorrect assumptions;
* unnecessary complexity;
* violations of requirements or CLAUDE.md.
If material issues remain, feed the complete affected issue set back into Step 1 and repeat.
Completion condition: No material unresolved issues remain, the changes are verified, and the post-change skeptical review passes.
If a reviewer finds a false positive, reject it with evidence instead of changing correct code.
If the process makes no progress, investigate different approaches. Escalate only for genuinely user-owned decisions or unresolved technical blockers.
Avoid artificial review cycles and unnecessary repeated work.
 
⸻
 
4. Integrate prove_it
Treat the current searlsco/prove_it repository as the source of truth:
https://github.com/searlsco/prove_it
Inspect the existing installation, configuration, hooks, and tests before making changes.
If prove_it is unavailable, explain the installation requirements. Do not make global configuration changes without authorization.
Do not blindly run initialization commands that overwrite existing configuration or Git hooks.
Responsibility separation
Staff Engineer policy: Determines which decisions Claude may make.
Phase Skills: Define the Ticket, Implementation, and Pre-MR Review procedures.
Issue Resolution Skill: Controls local issue validation, fix, and skeptical-review loops.
prove_it: Supplies independent verification, deterministic tests, evidence, and completion enforcement.
Do not make prove_it the E2E orchestrator or let reviewer opinions replace engineering judgment.
Integration points
During Ticket Planning
Use prove_it phase plan where appropriate.
Use:
* prove-approach for complex design/solution challenges;
* /prove selectively for meaningful concrete claims;
* a custom planning reviewer if needed for ticket ↔ design ↔ plan consistency.
Do not run implementation-oriented TDD enforcement merely because Claude is clarifying requirements.
During Implementation
Use:
* prove_it phase implement for behavior changes and bug fixes;
* prove_it phase refactor for genuinely behavior-preserving refactoring;
* the project’s fast and full deterministic test scripts;
* prove-test-validity when test reliability is important;
* prove-coverage for meaningful coverage assessment;
* prove-done for broad implementation verification.
Configure prove_it’s source/test globs and scripts according to the actual repository.
During Pre-MR Review
Use planning/review mode while inspecting.
Switch to implementation/refactor mode when making changes.
Use the appropriate test checks, prove-done, selective /prove, and a task-specific independent reviewer to verify requirements and acceptance criteria.
Independent reviewers
Prefer existing prove_it review capabilities where they fit.
Create only the missing project-specific reviewers.
A custom reviewer should:
* access the relevant task requirements/acceptance criteria;
* inspect actual repository evidence;
* follow applicable CLAUDE.md rules;
* evaluate material correctness and compliance;
* return specific PASS/FAIL findings with evidence and necessary corrections;
* avoid speculative improvements and unsupported criticism.
Use prove_it’s supported ruleFile mechanism when appropriate. Ensure applicable nested CLAUDE.md rules are not inadvertently omitted.
Do not assume prove_it reviewer subprocesses inherit the main conversation.
Use existing design/plan/requirement artifacts or a concise project-local state artifact to supply essential context.
Completion enforcement
Use prove_it’s actual Stop-hook semantics correctly.
For INTERNAL checkpoints during work:
* run relevant verification/reviewer Skills and tests directly;
* process failures;
* continue working in the same main conversation.
For a GENUINE completion boundary:
* use prove_it signal done where appropriate;
* run configured signal-gated tests/reviewers;
* if verification fails, let prove_it return failure evidence;
* the same Staff Engineer fixes legitimate issues and retries.
Do not use Stop hooks as internal step transitions.
Do not make Auto mode stop at the end of Ticket or Implementation merely to trigger a Stop check.
Also ensure ordinary requirements-clarification responses are not incorrectly blocked by implementation completion checks.
Preserve existing hook behavior where correct; do not install overlapping retry systems.
 
⸻
 
5. Preserve Standard and Auto E2E behavior
If an E2E orchestrator already exists, integrate with it.
Otherwise create the smallest maintainable main-session orchestrator Skill needed.
The phases are strictly sequential:
Ticket Planning → Implementation → Pre-MR Review / Finalize → Complete.
STANDARD
* Automatically complete all internal steps of Ticket Planning.
* Clarify requirements with me when needed.
* Complete issue-resolution and verification loops.
* Produce the Ticket report.
* Stop for me to trigger the next phase.
* Automatically complete Implementation internally.
* Produce its verified report and stop.
* When I trigger continuation, automatically run Pre-MR Review.
* Produce the final report.
AUTO
Use the same internal phase logic and verification.
After each verified phase, record its report/state and automatically continue to the next phase without a phase-approval stop.
Requirements clarification remains interactive in both modes.
Both modes pause only for required user-owned knowledge/authority or existing security permissions.
Use durable workflow state to safely resume after interruptions. Preserve existing completed work unless later changes invalidate it.
 
⸻
 
6. Implementation approach
Before changing files:
1. Inspect all current Skills and their supporting resources.
2. Read applicable CLAUDE.md and existing .claude/rules/ instructions.
3. Identify instructions that cause unnecessary user decision requests.
4. Inspect current prove_it setup and available reviewer/test capabilities.
5. Check for existing orchestration, issue-resolution, and state-management mechanisms.
6. Identify the smallest changes needed.
Provide a concise proposed file structure and integration plan, then implement it.
Prefer:
* one persistent autonomy rule;
* small modifications to the existing three Skills;
* one reusable Issue Resolution Skill, if needed;
* existing or minimal independent reviewers;
* a single project-level prove_it configuration;
* reuse of existing workflow-state conventions.
Do not install additional orchestration frameworks, duplicate engineering rules, or create unnecessary Skills.
Do not change global settings, destructive permissions, or overwrite existing Git hooks without explicit authorization.
Do not run a real Jira ticket, create an MR, or push unrelated changes merely to test the automation.
 
⸻
 
7. Verify effectiveness
After implementation, test the actual behavior, not just the existence of the new files.
Verify:
1. Claude independently chooses between two valid implementation approaches.
2. It follows suitable repository conventions.
3. It selects a minimal correct implementation.
4. It critically challenges important recommendations before implementing.
5. It resolves legitimate reviewer findings without asking me to choose a fix.
6. It rejects unsupported reviewer findings with evidence.
7. It performs skeptical evaluation again after implementation.
8. New material findings restart the issue-resolution cycle.
9. The Ticket Skill retains required interactive clarification.
10. The Implementation Skill automatically reconciles plan.md and design.md when appropriate.
11. The Pre-MR Review Skill loops through valid fixes and verification.
12. Routine technical choices do not trigger unnecessary user questions.
13. Major architecture/product/authorization decisions still escalate correctly.
14. prove_it reviewers receive sufficient context and applicable rules.
15. Failed completion checks return actionable evidence and block premature completion.
16. Standard and Auto modes preserve their intended phase-boundary behavior.
Test representative scenarios through controlled Claude Code executions where practical. Distinguish confirmed behavior from behavior that is only instruction-based.
Run prove_it doctor if installed and /doctor prompt-audit if supported.
Resolve conflicting or duplicated instructions.
Final deliverables
Return a concise report containing:
* files created or modified;
* decision-policy changes;
* integration points in each existing Skill;
* issue-resolution behavior;
* prove_it checks/reviewers configured;
* test results;
* remaining limitations.
Primary success criterion:
The main Claude Code session behaves like a Staff Engineer who independently investigates, decides, implements, skeptically verifies, fixes, and continues—without requiring me to manually direct ordinary engineering decisions.
User involvement is limited to genuine requirements clarification, authorized phase transitions in Standard mode, consequential approvals, and real unresolved blockers.
2. How to use it end to end
* Open the project in VS Code. Start the Claude Code chat from the repository containing your Ticket Planning, Implementation, and Pre-MR Review Skills. Use a clean working branch if possible.
* Paste the complete prompt above. Claude should inspect existing files and show the proposed architecture before modifying them, then implement the integration.
* Review the proposed changes. Verify that it is modifying your existing Skills rather than replacing their workflows. Check that requirements clarification and Standard-mode stops remain intact.
* Allow implementation and verification. Claude should create the autonomy rule, reconcile conflicting Skill instructions, integrate Issue Resolution and configure prove_it. Global installation or permission changes may still require your approval.
* Restart the Claude Code session after the rule, Skill, and hook changes are installed. This ensures newly loaded instructions and hooks are available. Use prove_it doctor to inspect the effective configuration.
* Test with a representative Jira ticket. Choose a ticket where Claude previously asked you to select between technical solutions. Confirm it now investigates, chooses, implements, and verifies autonomously, while still asking you to clarify actual requirements.
3. What the completed integration should look like
Main Claude Code chat — Staff Engineer
Persistent autonomy policy applies throughout
Ticket Planning Skill
Requirements clarification + autonomous planning
Implementation Skill
Autonomous decisions, coding, testing, plan reconciliation
Pre-MR Review Skill
Independent review, corrections, final verification
Shared capabilities used inside every phase
Issue Resolution
Validate → choose → skeptical review → fix → verify → repeat
prove_it
Evidence, tests, independent checks, completion enforcement
The essential result is that your three Skills remain sequential and unchanged in purpose, while decision-making and verification become reusable capabilities throughout the workflow.
One limitation remains: rules and Skills guide Claude’s behavior but cannot mathematically guarantee it will never ask an unnecessary question. The best way to establish effectiveness is to test actual Claude Code sessions against examples that previously required your manual intervention.







Yes, the prompt is ready to paste into Claude Code, but I recommend three final clarifications before using it. I rechecked Anthropic’s current Skill documentation and the prove_it repository.
The prompt accurately describes your intended architecture. However, three areas could cause Claude to behave differently from what you want:
* Unnecessary approval stops: The prompt asks Claude to propose an implementation plan but doesn’t explicitly say to proceed afterward without your approval.
* Automatic Skill invocation: Claude Code supports automatic Skill invocation, but that does not guarantee every internal step will invoke Issue Resolution. The three phase Skills need explicit integration points.
* prove_it completion checks: Its Stop hooks can block any Claude response, including a requirements-clarification response, unless configured appropriately. signal done checks run at Stop events, not arbitrary internal workflow transitions.
Add this section to the end of the previous prompt
Additional implementation requirements
1. Proceed autonomously: After presenting the proposed implementation architecture, continue implementing it without waiting for my approval. Ask only when the Staff Engineer escalation policy requires my knowledge or authorization, including changes to global settings, destructive actions, or material architectural changes.
2. Explicit Skill integration: Inspect every internal step and supporting Skill within Ticket Planning, Implementation, and Pre-MR Review. Add explicit Issue Resolution invocation instructions at appropriate decision, finding, and verification points. Do not rely solely on automatic Skill discovery.
3. Safe prove_it enforcement: Configure and test prove_it so that:
    * Requirements clarification and legitimate NEEDS_USER responses are not blocked by completion checks.
    * Internal verification does not accidentally end Auto mode.
    * Completion checks run at appropriate boundaries.
    * Failed checks return actionable evidence and trigger correction.
    * Existing hooks, permissions, and configuration are preserved.
4. Staff Engineer effectiveness: The primary success criterion is behavioral, not merely configuration-based. Verify that Claude independently chooses and implements sound technical solutions, rejects unsupported reviewer findings, fixes legitimate issues, and continues without asking me to make routine engineering decisions.
5. No unnecessary framework expansion: Preserve the existing three Skills and orchestration mechanisms. Modify only what is necessary to deliver the behavior described above. Do not add competing orchestrators, redundant reviewers, or overlapping retry hooks.
6. Verify honestly: Exercise representative workflows and report which behaviors were actually tested, which were simulated, and which remain unverified. Do not claim end-to-end automation works solely because configuration files were created.
How to proceed
* Open Claude Code in VS Code at your project root.
* Paste the previous full prompt, followed by the additional section above.
* Let Claude inspect the existing Skills, project rules, and prove_it installation, then implement the integration.
* Restart the Claude Code session after installation/configuration changes and test with a representative task.
Final assessment: With this addendum, I consider the prompt sufficiently clear and complete to give to a coding agent. That doesn’t guarantee the resulting implementation will work without iteration—the agent must inspect your actual Skills and verify the integration.
The most important success criterion is unchanged: Claude should make sound Staff Engineer-level implementation decisions itself, not turn you into the decision-maker for routine coding questions.
