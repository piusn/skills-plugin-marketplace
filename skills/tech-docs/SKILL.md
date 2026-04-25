---
description: "Generate technical or user documentation for products and features. Use this skill when the user says 'document feature', 'write documentation', 'update docs', 'tech docs', 'user guide', 'API documentation', or 'documentation review'. Analyzes codebases and existing docs to produce comprehensive documentation."
---

# Technical/User Documentation Skill

## Context
Good documentation is the foundation of maintainable software. This skill helps generate and update documentation by analyzing codebases, existing documentation, and team standards. It supports both technical docs (for developers) and user docs (for end users).

## When to Use
- When a new feature needs documentation
- When existing documentation is outdated
- When onboarding materials need updating
- When reviewing documentation completeness

## Workflow

### Step 1: Identify Documentation Scope
Ask the user:
```
ask_user: "What would you like to document?"
  choices: ["A specific feature/product", "API documentation", "Architecture overview", "User guide", "Full documentation review"]
```

### Step 2: Gather Source Material

**For code-based documentation:**
- Analyze the relevant codebase (explore agent)
- Check existing documentation in the repo
- Look for README files, inline docs, comments

**For product documentation:**
- Check existing documentation repositories
- Review Notion pages for the product
- Check eng.ms for existing docs:
  ```
  enghub-search(query: "[product name] documentation")
  ```

**For team documentation:**
- Reference team documentation standard from Notion:
  ```
  notion-API-get-block-children(block_id: "31c891a6-db0d-8125-98b4-fab2e24f72ff")
  ```

### Step 3: Determine Documentation Format

**DocFX format** (for repos using DocFX):
- Follow `.github/copilot-instructions.md` in the target repo
- Use proper metadata headers (ms.author, ms.service, ms.date)
- Follow toc.yml structure
- Use DocFX-compatible markdown (alerts, tabs, mermaid)

**Standard Markdown:**
- Clean headers and structure
- Code examples with language tags
- Mermaid diagrams for architecture
- Tables for API endpoints and parameters

### Step 4: Generate Documentation

Structure based on type:

**Technical Documentation:**
```markdown
# [Feature/Product] Technical Documentation

## Overview
[What it does, why it exists]

## Architecture
```mermaid
graph TD
  [architecture diagram]
```

## Components
### [Component 1]
- Purpose: [what it does]
- Location: [file/module path]
- Dependencies: [what it depends on]

## API Reference
| Endpoint | Method | Description | Request | Response |
|----------|--------|-------------|---------|----------|
| /api/v1/resource | GET | List resources | — | Resource[] |

## Configuration
| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| [setting] | string | [default] | [description] |

## Deployment
[How to deploy and configure]

## Troubleshooting
| Issue | Cause | Resolution |
|-------|-------|------------|
| [issue] | [cause] | [fix] |
```

**User Documentation:**
```markdown
# [Product] User Guide

## Getting Started
[First-time setup instructions]

## Features
### [Feature 1]
[How to use with screenshots/steps]

## FAQ
[Common questions and answers]
```

### Step 5: Review and Refine
- Check for accuracy against the codebase
- Ensure examples are runnable
- Verify links and references
- Check consistency with existing docs

### Step 6: Save Documentation
- Save to the appropriate repository/location
- If DocFX: validate with `docfx build`
- If Notion: create/update page
- Update table of contents if applicable

## Tools & APIs Used
- Explore agent — Codebase analysis
- `enghub-search` / `enghub-fetch` — Existing documentation
- `notion-API-*` — Notion documentation pages
- DocFX tools — If applicable
- `markdown-to-word` skill — Export to Word
- `md2pptx` skill — Export to slides

## Output Format
Documentation in the appropriate format (DocFX markdown, standard markdown, or Notion page) with architecture diagrams, API references, and troubleshooting guides.

## Notes
- Always check the target repo's copilot-instructions.md for documentation standards
- Use Mermaid for diagrams — they render in GitHub, Notion, and DocFX
- Include practical examples, not just API specs
- Keep user docs separate from technical docs
- Documentation should be maintainable — avoid hardcoding versions or dates
