
Create or refactor this Claude Code Skill using Anthropic's current
Agent Skills architecture and progressive-disclosure principles.

Use current Anthropic conventions as the source of truth. Do not invent
metadata fields, directory conventions, resource types, or Claude Code
capabilities. If something is not defined by Anthropic, do not present it
as an Anthropic standard.

Goal:
Keep SKILL.md focused on the core instructions Claude needs when the skill
activates. Organize supporting resources so additional content is read or
executed only when required.

Requirements:

1. SKILL.md
   - Include valid YAML frontmatter, including name and description.
   - Keep the core workflow, essential instructions, decision/selection logic,
     and navigation to supporting resources.
   - Move detailed material out when it can be loaded on demand.
   - Keep SKILL.md concise; Anthropic recommends under approximately 500 lines
     when practical.

2. Bundled resources
   Use Anthropic's standard resource directories when appropriate:
   - references/ → documentation, detailed procedures, policies, domain
     knowledge, schemas, edge cases, and other information Claude should
     load into context when needed.
   - scripts/ → executable code for deterministic or repetitive operations.
   - assets/ → templates, boilerplate, images, fonts, or other files primarily
     used in produced output.

   Do not create a directory merely because it is listed above.
   Create only resources the skill actually needs.

   Create additional/custom directories only when there is a concrete need.
   Do not describe a custom directory as an Anthropic standard.

3. Progressive disclosure
   - Organize the entire skill around what information is needed at each step.
   - Do not replace one large SKILL.md with one large supporting file.
   - Split supporting content by meaningful responsibility, workflow stage,
     variant, or usage condition when doing so enables selective loading.
   - Keep information together when it is normally needed together.
   - Avoid both oversized catch-all files and unnecessary fragmentation.
   - Avoid duplicating instructions across files.

4. Navigation
   - Reference supporting resources from SKILL.md.
   - State clearly when Claude should read, use, or execute each resource.
   - Allow supporting resources to point to more specialized resources when
     useful for progressive disclosure.

5. Scripts
   - Use scripts when deterministic or repetitive execution is preferable to
     having Claude reproduce the operation through reasoning.
   - Do not create scripts for ordinary instructions that Claude can execute
     reliably with its existing tools.
   - Validate newly created scripts when practical.

6. Structure
   - Do not create empty, placeholder, or ceremonial files/directories.
   - Choose file boundaries based on responsibility and loading behavior,
     not arbitrary file-size thresholds.
   - Preserve existing useful behavior and content.

7. Before modifying files
   - Inspect the existing skill and its supporting resources.
   - Identify its responsibilities and natural progressive-disclosure boundaries.
   - Determine which supporting resources are actually justified.
   - Propose the resulting directory tree with a brief reason for each file.
   - Verify that the proposed structure uses Anthropic-standard conventions
     correctly and clearly identifies any custom conventions.
   - Then perform the refactor.

Preserve existing behavior unless a change is necessary for correctness,
progressive disclosure, or maintainability. Do not silently remove requirements
or capabilities during the refactor.



Draw.io skill: https://github.com/Agents365-ai/drawio-skill⁠
Excalidraw skill: https://github.com/coleam00/excalidraw-diagram-skill⁠


I verified this against the current skill repos, their open issues, Excalidraw documentation, and Playwright documentation. The key difference is:

Draw.io is already largely offline-capable. Excalidraw requires an internal fork/patch of its renderer to be truly offline.

Draw.io skill — internal/offline setup

Copy the skill into your internal Git repo and install it under:
.claude/skills/drawio-skill/
Ensure the machine has Python 3. The skill’s core XML/IR, validation, query, sync, review, and related functions are explicitly documented as stdlib-only and offline. 
For PNG/SVG/PDF rendering, distribute the draw.io Desktop .deb/.rpm through your internal software repository. On headless Debian/Ubuntu also install xvfb:
xvfb-run -a drawio --version
Graphviz is optional and only needed for some automatic layouts. 

Avoid the network-dependent features. Do not use viewer.diagrams.net as a fallback. The skill’s normal draw.io shape index is local, but its AI/LLM logo helper uses public CDNs by default. For those logos, either vendor them internally or generate them with --embed before transferring them into the isolated environment. 
Test with outbound internet blocked:
python3 scripts/validate.py diagram.drawio --score

xvfb-run -a drawio \
  -x -f png --width 2000 \
  -o diagram.png diagram.drawio
Verified: no modification of the core Draw.io skill is necessary for offline generation/validation. You only need local dependencies and must avoid its optional web/CDN paths.


Excalidraw skill — internal/offline setup

This one does require an internal fork. The current skill’s render_template.html still contains:

import { exportToSvg }
  from "https://esm.sh/@excalidraw/excalidraw?bundle";
so the renderer as currently shipped is not offline.  There are also current bug reports caused by this CDN dependency. 

Use this architecture:

excalidraw-diagram/
├── SKILL.md
└── references/
    ├── render_excalidraw.py
    ├── render_template.html
    ├── ...
    └── vendor/
        ├── excalidraw.bundle.js
        └── fonts/
On an internet-connected build machine or your internal npm mirror, pin an approved @excalidraw/excalidraw version and build a browser bundle containing exportToSvg. Commit the resulting bundle into references/vendor/. Local bundling is also the recommended offline solution discussed in the skill’s own issue tracker. 
Change render_template.html from the esm.sh import to:
import { exportToSvg }
  from "./vendor/excalidraw.bundle.js";
Vendor the fonts too. Excalidraw officially documents that its fonts normally come from a CDN. For offline use, copy:
node_modules/@excalidraw/excalidraw/dist/prod/fonts
into your local vendor/fonts/ directory and configure EXCALIDRAW_ASSET_PATH to point there. 

Serve the renderer from localhost, rather than depending on file://. For example, have render_excalidraw.py start a small local HTTP server for its references/ directory and navigate Playwright to:
http://127.0.0.1:<port>/render_template.html
This is entirely local and avoids browser module/CORS problems that have been reported with the current renderer. 

Mirror Playwright + Chromium internally. The current skill requires Python ≥3.11 and Playwright, and its renderer launches Chromium.  Playwright officially supports an internal browser artifact repository:
PLAYWRIGHT_DOWNLOAD_HOST=http://your-internal-artifact-server \
  playwright install chromium
You can also place the browser in a shared internal location using PLAYWRIGHT_BROWSERS_PATH. 

Finally, block external internet but leave loopback available and run:
uv run python render_excalidraw.py \
  test.excalidraw \
  --output test.png
If the PNG renders correctly and your firewall/proxy logs show no requests to esm.sh, excalidraw.com, npm, or other external CDNs, the skill is truly operating internally.

Verified result


Draw.io skill

Excalidraw skill

Offline core

Yes already

Generation yes; renderer no

Needs internal fork

Usually no

Yes

Public CDN dependency

Optional AI icons/web fallback

Yes, currently mandatory for renderer

Local renderer

draw.io Desktop

Playwright + Chromium

Extra assets to vendor

Optional icons

Excalidraw JS + fonts

Fully internal achievable

Yes

Yes, after patching

For your environment, Draw.io is substantially easier and lower-maintenance to deploy internally. Excalidraw is feasible, but I would treat the offline renderer as an internal maintained fork rather than installing the GitHub skill unchanged.





Official ADR repo: https://github.com/architecture-decision-record/architecture-decision-record
ECC ADR skill: https://github.com/affaan-m/ECC
JamesC ADR workflow skills: https://github.com/jamesc/skills





Mostly native to Claude Code. You generally do not install separate packages.

Feature	Separate install?	What you need
/goal	No	Built into current Claude Code. 
/loop	No	Bundled Anthropic Skill shipped with Claude Code. 
Dynamic Workflows	No	Built-in workflow runtime; create/save .claude/workflows/*.js. 
Custom subagents	No	Create .claude/agents/*.md.
/subtask	No	Built-in; requires a sufficiently recent Claude Code version. 
Hooks	No	Configure in Claude Code/project settings or skill/agent definitions.
/background, /fork	No	Built-in. 
/batch	No	Bundled Anthropic Skill; requires a Git repo. 
/run, /verify	No	Bundled Skills; project-specific setup may be needed for Claude to know how to run your app. 
/schedule / Routines	No	Built-in cloud feature, but availability depends on account/environment. 
Agent Teams	No package install	Disabled by default; enable CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1. 
MCP	Sometimes	Claude supports MCP natively, but the specific MCP server (Jira, GitLab, etc.) may require installation/config/authentication. 

So for your workflow, you can stay almost entirely inside your existing Claude Code installation:

Claude Code
├── CLAUDE.md
├── .claude/rules/
├── .claude/skills/
├── .claude/agents/
├── .claude/workflows/
└── hooks/settings

The main thing I’d verify is that your Claude Code extension/runtime is current enough, because features such as /subtask and Dynamic Workflows were added in newer versions.




