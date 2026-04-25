---
name: session-knowledge
description: >
  Extract reusable knowledge from Copilot sessions and persist it as Copilot
  instructions, skill updates, new skills, or Notion documentation. Use this
  skill when the user says "save what we learned", "update instructions",
  "capture knowledge", "save this for next time", "create instruction from
  session", "what should we remember", or "knowledge harvest". Also invoked
  by session-outcomes and close-day to surface knowledge automatically.
---

# Session Knowledge — Capture & Persist Reusable Knowledge from Sessions

Analyzes Copilot sessions to identify reusable knowledge — deployment procedures, environment configurations, debugging patterns, architectural decisions, new workflows — and persists them as Copilot instructions, skill updates/creations, or Notion documentation so future sessions benefit automatically.

## Why This Skill Exists

During sessions, you discover valuable operational knowledge:
- How to deploy a service to a specific environment
- Which NuGet feeds or config exclusions are needed
- Debugging steps for pipeline failures
- Authentication patterns (SPN + KeyVault, Managed Identity)
- Team-specific workflows and conventions

This knowledge currently lives only in session history and the user's memory. It's lost to future sessions. This skill extracts it and routes it to the right place in the Copilot ecosystem.

## Knowledge Categories

| Category | What It Captures | Where It Goes |
|----------|-----------------|---------------|
| **Deployment** | Deploy steps, environments, pipelines, rollback procedures | `.instructions.md` or repo `.github/instructions/` |
| **Environment** | Config files, NuGet feeds, secrets setup, infrastructure | `.instructions.md` or repo `.github/instructions/` |
| **Debugging** | Troubleshooting steps, error patterns, root cause fixes | `.instructions.md` or Notion team page |
| **Architecture** | Design decisions, auth patterns, service integration | `architecture-decision` skill or `.instructions.md` |
| **Workflow** | New task patterns, multi-step procedures, team processes | New or updated Copilot skill |
| **Tool Usage** | CLI commands, API patterns, SDK usage, configuration | `.instructions.md` or learning notes |
| **Team Convention** | Naming, branching, review process, repo structure | Repo `.github/instructions/` or team Notion page |

## Trigger Modes

| Mode | Trigger | Behavior |
|------|---------|----------|
| **Ad-hoc** | "save what we learned", "capture knowledge", "update instructions" | Analyze current or specified session |
| **Integrated (session-outcomes)** | Called during session-outcomes processing | Surface knowledge alongside outcome extraction |
| **Integrated (close-day)** | Called during close-day routine | Quick scan of today's sessions for knowledge |
| **In-session** | "save this for next time", "remember this" | Capture specific knowledge from the current conversation |
| **Targeted** | "create instruction for {topic}" | Create a specific instruction from session context |

---

## Instructions

### Step 1: Determine Knowledge Scope

#### Ad-hoc / integrated invocation:
Gather session data (same as session-outcomes Step 1):
```sql
-- Checkpoints (richest source)
SELECT checkpoint_number, title, overview, work_done, technical_details, important_files
FROM checkpoints
WHERE session_id = '{session_id}'
ORDER BY checkpoint_number
```

```sql
-- Key user messages with knowledge signals
SELECT turn_index, user_message
FROM turns
WHERE session_id = '{session_id}'
ORDER BY turn_index
```

```sql
-- Files created/modified (especially config, instructions, docs)
SELECT file_path, tool_name
FROM session_files
WHERE session_id = '{session_id}'
ORDER BY turn_index
```

#### In-session (current conversation):
Use the current conversation context directly — no need to query session store.

### Step 2: Identify Knowledge Nuggets

Scan the session data for **knowledge signals** — information that would be valuable in future sessions.

#### High-Signal Patterns (always extract):

**Deployment knowledge:**
- Steps to deploy a service (commands, pipelines, environments)
- Environment-specific configurations (PME, PPE, Prod)
- Pipeline YAML structure, stage ordering, approval gates
- Rollback procedures
- Signal words: "deploy", "pipeline", "release", "rollout", "publish", "Ev2", "OneBranch"

**Environment & configuration:**
- NuGet feed setup, package source mappings
- ConfigGuard rules, exclusions, reporting config
- Authentication setup (SPN, KeyVault, Managed Identity, certificates)
- Connection strings patterns, endpoint URLs
- Docker/container configurations
- Signal words: "config", "nuget", "feed", "authentication", "key vault", "managed identity", "appsettings"

**Debugging & troubleshooting:**
- Root cause analysis patterns
- Error resolution steps (what failed → why → fix)
- Build failure patterns (ConfigGuard, security policy, compilation)
- Workarounds and their limitations
- Signal words: "error", "fix", "debug", "failed", "resolved", "workaround", "root cause"

**Architectural patterns:**
- Service integration patterns (API → Service → Data)
- Auth flows (SPN + KeyVault → token → API call)
- Data pipeline patterns (source → transform → sink)
- Infrastructure patterns (Container Apps, AKS, App Service)
- Signal words: "architecture", "design", "pattern", "integration", "service", "API"

**New workflows & procedures:**
- Multi-step procedures that could become skills
- Repeatable processes for team-specific tasks
- Migration steps, conversion processes
- Signal words: "steps to", "procedure", "process", "how to", "workflow"

**Tool & SDK usage:**
- CLI commands with specific flags/options
- SDK initialization patterns
- API call patterns with auth headers
- Configuration file formats and required fields
- Signal words: "command", "CLI", "SDK", "API call", "endpoint"

#### For Each Knowledge Nugget, Extract:

```
{
  "title": "Descriptive title",
  "category": "deployment | environment | debugging | architecture | workflow | tool-usage | team-convention",
  "description": "What was learned (2-3 sentences)",
  "content": "The actual knowledge — commands, steps, configs, patterns",
  "repository": "Which repo this applies to (or null for global)",
  "team": "Which team this is relevant to (or null for personal)",
  "scope": "global | repo-specific | team-specific",
  "confidence": "high | medium",
  "destination": "instruction | skill-update | new-skill | notion | architecture-decision"
}
```

### Step 3: Determine Destination for Each Nugget

Route knowledge to the right place based on category and scope:

#### Decision Tree:

```
Is this a repeatable multi-step procedure that could automate future work?
  YES → Is it complex enough for a full skill (5+ steps, multiple tools)?
    YES → Create new Copilot skill (Step 5a)
    NO  → Could it extend an existing skill?
      YES → Update existing skill (Step 5b)
      NO  → Create instruction file (Step 5c)
  NO → Is this repo-specific knowledge (applies only when working in that repo)?
    YES → Create/update repo .github/instructions/ file (Step 5d)
    NO  → Is this team-specific operational knowledge?
      YES → Add to team Notion page (Step 5e)
      NO  → Is this a significant architectural decision?
        YES → Suggest architecture-decision skill (Step 5f)
        NO  → Create/update global instruction file (Step 5c)
```

### Step 4: Present Findings to User

Before persisting anything, present the extracted knowledge:

```markdown
## 🧠 Knowledge Extracted from Session

### Found {N} knowledge nuggets:

| # | Knowledge | Category | Destination | Scope |
|---|-----------|----------|-------------|-------|
| 1 | OneBranch + Ev2 deployment to PME | Deployment | Repo instructions | reliability.tools.ooa |
| 2 | ConfigGuard exclusion patterns | Environment | Repo instructions | reliability.tools.ooa |
| 3 | SPN + KeyVault auth for ADLS Gen1 | Architecture | Global instruction | All repos |
| 4 | Lens→ADF migration conversion steps | Workflow | New skill candidate | RDE team |

### Actions I'll take:
- 📝 Create `bangtest-deployment.instructions.md` in repo instructions
- 📝 Update `azure-infrastructure.instructions.md` with ADLS auth pattern
- 🔧 Create Notion page for RDE migration workflow
- 💡 Flag Lens→ADF conversion as potential new skill

Proceed with all, or adjust?
```

```
ask_user:
  question: "Should I persist all extracted knowledge, or would you like to adjust?"
  choices:
    - "Proceed with all (Recommended)"
    - "Let me review each one"
    - "Skip for now — just note them"
```

### Step 5: Persist Knowledge

#### Step 5a: Create New Copilot Skill

When the knowledge represents a repeatable multi-step procedure:

1. **Create the skill directory:**
   ```
   mkdir ~/.copilot/skills/{skill-name}/
   ```

2. **Generate SKILL.md** following the established pattern:
   ```markdown
   ---
   name: {skill-name}
   description: >
     {description with trigger phrases}
   ---

   # {Skill Title}

   ## Context
   {Why this skill exists — what problem it solves}

   ## When to Use
   {Trigger conditions}

   ## Workflow
   ### Step 1: ...
   ### Step 2: ...

   ## Tools & APIs Used
   {List of tools}
   ```

3. **Inform the user:**
   ```
   ✅ New skill created: {skill-name}
   Location: ~/.copilot/skills/{skill-name}/SKILL.md
   Trigger phrases: "{phrase1}", "{phrase2}"
   ```

#### Step 5b: Update Existing Skill

When knowledge extends an existing skill:

1. **Read the current skill:**
   ```
   view ~/.copilot/skills/{skill-name}/SKILL.md
   ```

2. **Identify where to add the knowledge:**
   - New step in the workflow?
   - New edge case handling?
   - Updated tool usage?
   - New integration point?

3. **Edit the skill** with the new knowledge, preserving existing structure

4. **Inform the user:**
   ```
   ✅ Updated skill: {skill-name}
   Added: {brief description of what was added}
   ```

#### Step 5c: Create/Update Instruction File

For knowledge that applies when working with specific file types or globally:

**Global instructions** (apply across all repos):
Location: `~/.copilot/instructions/{topic}.instructions.md`

```markdown
---
description: "{description}"
applyTo: "{glob pattern}"
---

# {Title}

## {Section}
{Knowledge content}
```

**Check if an existing instruction file covers this topic:**
1. List current instructions: `ls ~/.copilot/instructions/`
2. If a relevant file exists, append the new knowledge to it
3. If no relevant file exists, create a new one

**Naming convention for new instruction files:**
- Use kebab-case: `{topic}.instructions.md`
- Be specific: `onebranch-deployment.instructions.md` not `deployment.instructions.md`
- Match the `applyTo` glob to the files this knowledge applies to

#### Step 5d: Create/Update Repo-Specific Instructions

For knowledge that only applies when working in a specific repository:

Location: `{repo_root}/.github/instructions/{topic}.instructions.md`

1. **Check if the repo has a .github/instructions/ directory:**
   - If not, create it
   
2. **Create the instruction file** with appropriate `applyTo` patterns

3. **If the repo is a git repo, note the change (don't auto-commit):**
   ```
   ℹ️ Created repo instruction: .github/instructions/{topic}.instructions.md
   Remember to commit and push this file so it applies for all team members.
   ```

#### Step 5e: Add to Team Notion Page

For team-specific operational knowledge:

1. **Identify the team Notion page** (from my-teams skill mapping):
   | Team | Notion Page ID |
   |------|---------------|
   | Reliability Data Engineering | `1e9891a6-db0d-809b-8632-f864d2db3ae7` |
   | Benchmarking | `1e9891a6-db0d-80c3-941f-e77ff2d0127a` |
   | Data Analytics & Anomaly Detection | `1e9891a6-db0d-80f2-bf34-c3c003ec6bd5` |
   | Sustainability | `1e9891a6-db0d-80bd-8f62-eaa264109fb2` |
   | Power, Performance & Sustainability DE | `1e9891a6-db0d-80bb-8b1f-e8e9f72212da` |
   | Gates & Defense | `b359111c-26e7-4417-b741-fddcf2abb50d` |

2. **Create a knowledge page** under the team page:
   ```
   notion-mcp-create_page(
     parentId: "{team_notion_page_id}",
     parentType: "page",
     title: "🧠 {knowledge_title}"
   )
   ```

3. **Add structured content:**
   - Context: Why this knowledge matters
   - Steps/Details: The actual knowledge
   - Related: Links to repos, tasks, other pages
   - Source: Session ID and date

#### Step 5f: Suggest Architecture Decision Record

For significant architectural decisions:

```
💡 This looks like a significant architectural decision:
   "{decision description}"

Would you like to document it as an ADR using the architecture-decision skill?
This creates a formal record with context, options considered, and rationale.
```

If yes, invoke the `architecture-decision` skill with the extracted context.

### Step 6: Summary Report

```markdown
## 🧠 Knowledge Persisted

| # | Knowledge | Destination | Status |
|---|-----------|-------------|--------|
| 1 | {title} | `{file_path}` | ✅ Created |
| 2 | {title} | Skill: `{skill_name}` | ✅ Updated |
| 3 | {title} | Notion: {team} page | ✅ Created |
| 4 | {title} | ADR suggested | 💡 Pending |

### What This Means
- Future sessions working in {repo} will automatically get: {instruction names}
- The {skill_name} skill now covers: {new capability}
- {team} team page now has: {knowledge title}
```

---

## Integration with Other Skills

### session-outcomes integration
Add to session-outcomes Step 2 (after extracting distinct work items):
```
After extracting work items, also scan for reusable knowledge:
- Invoke session-knowledge for each work item that involves
  deployment, configuration, debugging, or new patterns
- Knowledge extraction runs alongside outcome tracking
```

### close-day integration
Add to close-day after session-outcomes:
```
After session outcomes are extracted, quick-scan for knowledge:
- Focus on high-signal patterns only (deployment, environment, debugging)
- Present knowledge nuggets alongside the day summary
- User can choose to persist or skip
```

### start-task / workflow skills integration
At the **end** of any workflow (engineering-task, workflow-quickfix, etc.):
```
### Final Step: Knowledge Capture
Before closing this task, scan the session for reusable knowledge:
- What deployment steps were used?
- What configuration was needed?
- What debugging was required?
- What patterns were established?

Invoke session-knowledge to extract and persist.
```

### learning-notes integration
When knowledge is about a topic in the user's learning path:
- Route to learning-notes in Notion as well
- Cross-reference with learning subjects/topics

---

## Instruction File Best Practices

When creating or updating instruction files, follow these rules:

### Structure
```markdown
---
description: "Clear, concise description of what this instruction covers"
applyTo: "glob pattern matching relevant files"
---

# Title

## Section
- Actionable, specific guidance
- Commands with examples
- Configuration snippets with explanation
```

### applyTo Patterns
| Knowledge Type | Pattern |
|---------------|---------|
| Deployment | `**/.pipelines/**,**/deploy/**,**/*.pipeline.yml` |
| Docker/Containers | `**/Dockerfile*,**/docker-compose*` |
| C# code | `**/*.cs` |
| TypeScript | `**/*.ts,**/*.tsx` |
| Infrastructure | `**/*.bicep,**/*.tf,**/infra/**` |
| Config files | `**/*.json,**/*.yaml,**/*.yml` |
| All files in repo | `**/*` |

### Content Quality
- ✅ Be specific — include actual commands, configs, URLs
- ✅ Include context — why this pattern exists, not just what to do
- ✅ Include examples — show the pattern in use
- ✅ Note exceptions — when this pattern does NOT apply
- ❌ Don't be vague — "use best practices" is not an instruction
- ❌ Don't duplicate — check existing instructions first
- ❌ Don't include secrets — reference Key Vault or env vars instead

---

## Skill Creation Guidelines

When creating a new skill from session knowledge:

### When to Create a Skill vs Instruction
| Create a Skill when... | Create an Instruction when... |
|------------------------|------------------------------|
| Multi-step procedure (5+ steps) | Single pattern or convention |
| Requires tool orchestration | Applies passively to file editing |
| Has conditional logic or branching | Straightforward guidance |
| Would be invoked explicitly | Should apply automatically |
| Needs user interaction | No interaction needed |

### Skill Quality Checklist
- [ ] Clear trigger phrases in the description
- [ ] Step-by-step workflow with numbered steps
- [ ] Tool calls specified with example parameters
- [ ] Edge cases documented
- [ ] Integration points with other skills noted

---

## Edge Cases

| Scenario | Handling |
|----------|----------|
| Knowledge already exists in an instruction | Update/enrich the existing instruction, don't create duplicates |
| Conflicting knowledge (new info contradicts old instruction) | Present both to user, ask which is correct, update accordingly |
| Knowledge is too specific (one-time procedure) | Skip — not worth persisting. Note in session activity log instead |
| Knowledge spans multiple categories | Create one primary entry, cross-reference from others |
| Repo doesn't have .github/instructions/ | Create the directory structure |
| User rejects a knowledge nugget | Respect the decision, don't persist |
| Session has no extractable knowledge | Report "No reusable knowledge found" and move on |

---

## Tools & APIs Used

### File System
- `view` — Read existing instructions and skills
- `create` — Create new instruction files and skills
- `edit` — Update existing instruction files and skills
- `glob` — Find existing instruction files
- `powershell` — Create directories

### Session Store (read-only)
- `sql` (session_store) — Query session history, checkpoints, files, refs

### Notion
- `notion-mcp-create_page` — Create knowledge pages on team pages
- `notion-mcp-append_block_children` — Add content to pages
- `notion-mcp-search` — Find existing knowledge pages

### Daily Planner
- `DailyPlanner-add_activity_log` — Log knowledge capture activity

### Other Skills
- `architecture-decision` — For significant architectural decisions
- `learning-notes` — For learning-related knowledge
- `my-teams` — For team Notion page routing

---

## Examples of User Requests That Trigger This Skill

| User Says | Action |
|-----------|--------|
| "save what we learned" | Extract knowledge from current session |
| "update instructions" | Scan session for instruction-worthy knowledge |
| "capture knowledge" | Full knowledge extraction from session |
| "save this for next time" | Capture specific knowledge from current conversation |
| "create instruction for deployment" | Create deployment instruction from session context |
| "remember this pattern" | Persist a specific pattern as instruction |
| "what should we remember from this session" | Full session knowledge scan |
| "knowledge harvest" | Comprehensive knowledge extraction |
| "this should be a skill" | Create a new skill from current workflow |
| "update the quickfix skill with this" | Update specific skill with session knowledge |
