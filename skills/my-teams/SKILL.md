---
description: "Get the list of teams I work with as Teams Architect under ADC Platform Health / Adaptive Society. Use this skill when the user asks about 'my teams', 'which teams', 'team list', 'team tags', 'team members', or needs to reference team-specific information. Each team has a dedicated Notion page with members, products, resources, and documentation."
---

# My Teams Skill

## Role Context
Pius Ngugi is a **Teams Architect** within the **Adaptive Society team under Tarik**, part of **ADC Platform Health (Nairobi)**. He oversees the following teams' engineering architecture, quality, and delivery.

## Teams

### 1. Reliability Data Engineering
- **Notion Page ID:** `1e9891a6-db0d-809b-8632-f864d2db3ae7`
- **Focus:** Reliability data pipelines, crash analysis, and automated bug filing for Windows
- **Products:** Bang Analyze (Web App), Fast Filer, Lens-to-ADF migration
- **ADO Board:** Reliability Reporting sprint board
- **Tag:** `Reliability Data Engineering`

### 2. Benchmarking
- **Notion Page ID:** `1e9891a6-db0d-80c3-941f-e77ff2d0127a`
- **Focus:** Windows benchmarks with unified results and smarter defense posture
- **Products:** FUN Gates (Web App - fungates.azurewebsites.net)
- **ADO Board:** PH-Benchmarking sprint board
- **Tag:** `Benchmarking`

### 3. Data Analytics & Anomaly Detection
- **Notion Page ID:** `1e9891a6-db0d-80f2-bf34-c3c003ec6bd5`
- **Focus:** Anomaly detection, data analytics, and alerting systems for Platform Health
- **Products:** Anomaly Detection Portal, Alerting System
- **ADF Pipelines:** Reliability WatsonSnapshot DailyHits (UserMode)
- **Tag:** `Anomaly Detection`

### 4. Sustainability
- **Notion Page ID:** `1e9891a6-db0d-80bd-8f62-eaa264109fb2`
- **Focus:** Sustainability engineering — energy efficiency, carbon footprint, and environmental impact analysis
- **Products:** Sustainability Service (Data Pipeline)
- **Tag:** `Sustainability`

### 5. Power, Performance & Sustainability Data Engineering
- **Notion Page ID:** `1e9891a6-db0d-80bb-8b1f-e8e9f72212da`
- **Focus:** Combined power/performance/sustainability data engineering
- **Standup:** [Standup] Performance, Power & Sustainability Data Engineering
- **Tag:** `Performance`

### 6. Gates & Defense (Emmanuel's Team)
- **Notion Page ID:** `b359111c-26e7-4417-b741-fddcf2abb50d`
- **Focus:** FunGates defense and quality gates
- **Products:** FunGates (fungates.azurewebsites.net)
- **Tag:** `Gates & Defense`

### 7. COSINE Reliability Data Team
- **Focus:** PR reviews, reliability workflows
- **Tag:** `COSINE`

### 8. Team Duma
- **Focus:** Daily standups, sustainability & agent work
- **Tag:** `Team Duma`

## Parent Pages in Notion
- **Our Teams/Products:** `16e978b5-b3d1-4251-818d-60528797580d`
- **Platform Health:** `4e52f836-0ec4-43e7-8c09-129e26a6113c`
- **Team Documentation Standard:** `31c891a6-db0d-8125-98b4-fab2e24f72ff`
- **My Role as Architect:** `31c891a6-db0d-8155-a498-d6b892f9c713`

## How to Get Team Details from Notion
To retrieve full team details (members, products, resources), use the Notion API:

```
GET https://api.notion.com/v1/blocks/{notion_page_id}/children
```

This returns headings (👥 The Team, 📦 Products, 📂 Shared Resources) with nested content.

## All Team Tags (for tagging tasks)
Use these tags when creating or updating Daily Planner tasks:
`Reliability Data Engineering`, `Benchmarking`, `Anomaly Detection`, `Sustainability`, `Performance`, `Gates & Defense`, `COSINE`, `Team Duma`

## WorkIQ Integration — Team Communications

### Get Team-Related Communications
To pull emails, chats, and meetings related to a specific team, use WorkIQ:

**Recent emails with a team:**
```
workiq-ask_work_iq: "What recent emails have I exchanged with [team name] team members in the past week? Summarize key topics."
```

**Teams chats about a team's work:**
```
workiq-ask_work_iq: "What Teams messages or discussions have I had about [product name or team focus area] recently?"
```

**Meeting history with a team:**
```
workiq-ask_work_iq: "What meetings have I had with [team name] team in the past month? List dates, topics, and any action items."
```

### Cross-Reference with Daily Planner
To see tasks related to a specific team:
```
DailyPlanner-get_tasks(tag: "[team tag]")
```

### Workflow: Full Team Intel
1. Get team details from Notion (page content)
2. Get team tasks from Daily Planner (by tag)
3. Get team communications from WorkIQ (emails, chats, meetings)
4. Combine into a comprehensive team status view
