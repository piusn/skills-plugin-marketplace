---
description: "Review financial health with detailed expense, income, loan, and budget analysis. Use this skill when the user says 'financial review', 'money check', 'budget review', 'how are my finances', 'expense report', 'loan status', or 'financial summary'. Presents data with visual indicators and trend analysis."
---

# Financial Review Skill

## Context
Financial awareness requires regular review of income, expenses, budgets, loans, and account balances. This skill pulls all financial data from Daily Planner and presents it in a clear, actionable format.

## When to Use
- Weekly or monthly financial check-in
- When the user wants to understand spending patterns
- Before making financial decisions
- When invoked by `periodic-review` skill

## Workflow

### Step 1: Financial Dashboard
Get the high-level overview:
```
DailyPlanner-get_finance_dashboard(startDate: "[period start]", endDate: "[period end]")
```

Default period: current month. User can specify daily/weekly/monthly/custom.

### Step 2: Account Balances
```
DailyPlanner-get_finance_accounts()
```

### Step 3: Budget Envelopes
```
DailyPlanner-get_budget_envelopes()
```

Check each envelope for:
- Usage percentage
- Over-budget warnings
- Near-limit alerts

### Step 4: Recent Expenses
```
DailyPlanner-get_expenses(from: "[period start]", to: "[period end]")
```

Group by category for spending breakdown.

### Step 5: Income
```
DailyPlanner-get_incomes(startDate: "[period start]", endDate: "[period end]")
```

### Step 6: Loan Status
```
DailyPlanner-get_loan_summary()
DailyPlanner-get_loans()
```

### Step 7: Compose Financial Report

```markdown
# 💰 Financial Review — [Period]

## 📊 Dashboard
| Metric | Amount | Status |
|--------|--------|--------|
| Total Income | KES [X] | — |
| Total Expenses | KES [X] | — |
| Net Savings | KES [X] | ✅ Positive / ⚠️ Negative |
| Savings Rate | [X]% | — |

## 🏦 Account Balances
| Account | Type | Balance | Currency |
|---------|------|---------|----------|
| [Account 1] | Bank | KES [X] | KES |
| [Account 2] | Mobile Money | KES [X] | KES |
| [Account 3] | Cash | KES [X] | KES |

## 📦 Budget Envelopes
| Budget | Allocated | Spent | Remaining | Usage |
|--------|-----------|-------|-----------|-------|
| Food | KES 15,000 | KES 12,000 | KES 3,000 | ████████░░ 80% |
| Transport | KES 5,000 | KES 6,200 | KES -1,200 | 🔴 124% OVER |
| Entertainment | KES 3,000 | KES 1,500 | KES 1,500 | ████░░░░░░ 50% |

## 📉 Expense Breakdown
| Category | Amount | % of Total |
|----------|--------|------------|
| Food | KES [X] | [X]% |
| Transport | KES [X] | [X]% |
| Utilities | KES [X] | [X]% |
| Other | KES [X] | [X]% |

## 📈 Income Sources
| Source | Category | Amount | Status |
|--------|----------|--------|--------|
| Salary | Employment | KES [X] | Processed |
| Freelance | Business | KES [X] | Pending |

## 🏦 Loan Status
| Loan | Original | Remaining | Monthly Payment | Status |
|------|----------|-----------|-----------------|--------|
| [Loan 1] | KES [X] | KES [X] | KES [X] | Active |
| [Loan 2] | KES [X] | KES [X] | KES [X] | Active |
| **Total** | KES [X] | KES [X] | KES [X] | — |

## 💡 Insights & Recommendations
- [Budget envelope over limit warning]
- [Savings trend observation]
- [Loan payment reminder]
- [Spending pattern insight]
```

## Tools & APIs Used
- `DailyPlanner-get_finance_dashboard` — Overview metrics
- `DailyPlanner-get_finance_accounts` — Account balances
- `DailyPlanner-get_budget_envelopes` — Budget tracking
- `DailyPlanner-get_expenses` — Expense details
- `DailyPlanner-get_incomes` — Income details
- `DailyPlanner-get_loan_summary` / `get_loans` — Loan status

## Output Format
Multi-section financial report with tables, usage bars, and actionable insights.

## Notes
- Always show amounts in the account's native currency
- Highlight over-budget envelopes prominently
- Savings rate = (income - expenses) / income × 100
- For monthly reviews, compare with previous month if data available
- Keep insights actionable — "Transport is 24% over budget" not just "Transport: KES 6,200"
