# Jira Cost Estimation Model

Reference for generating development cost reports from Jira ticket exports. Used when the user provides a Jira ticket table (markdown or CSV) and asks for a cost/effort report as an enterprise HTML document (document type ES — Effort/Billing Statement).

## Man-Day Estimation by Ticket Type

Effort is estimated using a ticket-type-based weight model. Each Jira ticket type carries a standard effort weight reflecting typical scope for offshore development:

| Ticket Type | Man-Days / Ticket | Rationale |
|---|---|---|
| Epic | 0 | Container/parent — effort is captured in child Stories/Tasks, not double-counted |
| Story | 8 | Full-stack feature: backend API + frontend UI + integration testing |
| Task | 4 | Single-domain development: either backend or frontend, including unit testing |
| Sub-Task | 2 | Focused sub-item: small enhancement, validation change, or minor fix |

## Daily Rate

- **Blended Vietnam offshore rate: USD $120/man-day** — mid-level engineer, development effort only.
- Excludes QA, DevOps, PM, and BA overhead.
- Individual developer rates may vary by seniority; the blended rate is for estimation, not invoicing line-items.

## Parsing Jira Markdown Tables

Jira exports arrive as pipe-delimited markdown tables. Parse with a regex matching the row structure:

```python
m = re.match(
    r'^\|\s+(\w[\w-]*)\s+\|\s+\[([A-Z]+-\d+)\]\(([^)]+)\)\s+\|\s+(.*?)\s+\|\s+(.*?)\s+\|\s+(.*?)\s+\|\s+(.*?)\s+\|\s+(.*?)\s+\|\s+(\d{4}-\d{2}-\d{2}T\d{2}:\d{2})\s+\|\s+(\d{4}-\d{2}-\d{2}T\d{2}:\d{2})\s+\|$',
    line
)
```

Fields captured: Type, Key, URL, Summary, Assignee, Reporter, Priority, Status, Created, Updated.

**Empty assignee field:** Jira exports use whitespace-only cells for unassigned tickets. Normalize to `"Unassigned"` after parsing.

**"Former user":** Represents developers who have left the team. Keep in the breakdown but note in the report that these are departed contributors.

## Cost Breakdown Dimensions

Generate these breakdown tables in the report:

1. **By Developer** — tickets, man-days, cost, % of total. Sort by man-days descending.
2. **By Status** — group Jira statuses into delivery phases:
   - Delivered: ON UAT + Dev Done
   - In Progress: On Develop + Code Review
   - Backlog: TO DO
3. **By Ticket Type** — count, man-days, cost, % of total.
4. **By Month (Created date)** — monthly trend of tickets created and effort estimated.
5. **Ticket Inventory** — top N tickets (50) ranked by man-days, with Jira link, assignee, priority, status.

## Report Structure (ES Document Type)

Standard sections for a Jira cost report:

1. Executive Summary — stat cards (total tickets, total MD, daily rate, total cost, delivered/in-progress/backlog split, active developer count)
2. Estimation Methodology — type-weight table + basis-of-estimate callout + exclusions disclaimer
3. Cost Breakdown by Developer — table + top contributors callout
4. Cost Breakdown by Status — table + delivery status callout
5. Monthly Effort Trend — table + peak period callout
6. Ticket Inventory (Top 50) — detail table with Jira links
7. Financial Summary — stat cards (total/delivered/in-progress/backlog cost) + priority distribution table + important notes callout
8. Approval & Sign-Off — PM, Finance Approver, Client Sponsor (ES default roles)
9. Glossary — define MD, blended rate, ticket types, Jira statuses, project acronym
10. Version History

## Adjusting the Model

The type-based weights and daily rate are defaults. If the user specifies different rates or a more granular estimation method (e.g., story points, actual timesheets), use those instead. The model is a starting point — always state the basis clearly in the Estimation Methodology section so the client understands what the figures represent.
