# Skills Plugin Marketplace

Personal collection of GitHub Copilot CLI skills, instructions, and agents — designed to be shared across machines.

## Structure

```
├── skills/           # Copilot CLI skill definitions (SKILL.md per folder)
├── instructions/     # Coding standards & conventions (.instructions.md)
├── agents/           # Custom agent definitions (.agent.md)
```

## Setup on a New Machine

Copy the contents into your local Copilot CLI config directory:

```powershell
# Windows
Copy-Item -Recurse .\skills\* "$env:USERPROFILE\.copilot\skills\"
Copy-Item -Recurse .\instructions\* "$env:USERPROFILE\.copilot\instructions\"
Copy-Item -Recurse .\agents\* "$env:USERPROFILE\.copilot\agents\"
```

## Contents

### Skills (47)

Workflow skills for daily planning, engineering tasks, learning, meetings, finance, health tracking, and more.

### Instructions (17)

Coding standards covering TypeScript, C#, React, API design, testing, security, performance, observability, Git workflow, CI/CD, Docker/Kubernetes, database, and documentation.

### Agents (1)

- **kss-session-manager** — Manages ADC Windows Bi-Weekly Knowledge Sharing Sessions.
