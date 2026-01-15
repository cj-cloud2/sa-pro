**Puzzle 1: Cost Management Mastery**

**Scenario:** You're the cloud architect at E-Shop Corp, an e-commerce platform. Last month, total AWS costs jumped 40% to \$50K, mostly from untagged EC2 instances spanning 50+ services across dev/prod environments. No one knows which team or project caused it. Leadership tasks you with:

1. **Pinpoint root causes** using historical trends and forecasts (Hint: *Look for a tool that visualizes spend patterns over time, RI recommendations, and "what-if" predictions without needing raw data exports.*)
2. **Alert on future overruns** if monthly spend hits 80% of \$40K baseline (Hint: *Choose something for proactive notifications tied to custom thresholds, not just reports.*)
3. **Attribute costs accurately** to teams/projects despite missing tags (Hint: *Focus on a tagging system that links resources post-facto, even if user tags are incomplete.*)
4. **Enable custom SQL analysis** for a monthly dashboard slicing costs by service, region, and custom fields (Hint: *Pick the granular dataset deliverable to S3 for Athena/BI tools.*)

**Your Task (10-15 mins):** For each challenge, select **one primary tool** (or combo if needed), rank its top 2 features for this case, and sketch a 1-step action plan. Discuss in groups: Why not the others? Time yourself!

**Submit:** Tool choices + quick rationale per challenge.

***

### Solution Key with Detailed Justifications

#### Challenge 1: Pinpoint Root Causes → **AWS Cost Explorer**

**Teaching from Scratch:** AWS billing starts with understanding *consolidated billing*—all accounts/services roll up into payer views. Cost Explorer is a free, interactive dashboard (no setup needed beyond activation) that processes 13-14 months of historical data into visuals. It groups by API/service (e.g., EC2), linked accounts, or tags, showing daily/weekly/monthly trends. Key: It auto-generates *forecasts* using ML (e.g., "spend will hit \$60K next month") and RI/SP savings plans recommendations. Unlike reports, it's real-time queryable—no S3 downloads.

**Why Best Here:** For E-Shop's spike, filter by "Service: EC2" + "Usage Type" to spot untagged instance families (e.g., m5.large). Forecast predicts overruns.
**Action Plan:** Activate Cost Explorer → Group by Service/Tag → View Forecast tab.
**Why Not Others?** Budgets are alerts-only (no history); CUR is raw data (overkill for visuals); Tags enable grouping but don't analyze.

#### Challenge 2: Alert on Future Overruns → **AWS Budgets**

**Teaching from Scratch:** Budgets are rule-based alerts (free, account/account-group scoped) for cost control, separate from billing views. Create "cost budgets" with thresholds (e.g., 80% of \$40K), forecast adjustments (e.g., +20% growth), and notifications via SNS/email/Slack. Unlike Cost Explorer's passive forecasts, Budgets *act*—triggering at actual/forecasted breaches. Supports shared budgets across OUs.

**Why Best Here:** Set \$40K monthly cost budget → 80% alert → SNS to finance Slack. Handles E-Shop's baseline perfectly.
**Action Plan:** Console > Budgets > Create → Cost budget → \$40K recurring → Alerts at 80%/100%.
**Why Not Others?** Cost Explorer forecasts but doesn't alert natively; CUR needs custom processing; Tags don't notify.

#### Challenge 3: Attribute Costs Accurately → **AWS Cost Allocation Tags**

**Teaching from Scratch:** Tags are metadata key-value pairs (e.g., "Project:Checkout", "Team:DevOps") on resources. *User tags* (manual) need activation for billing; *AWS-generated tags* (e.g., createdBy) are auto-available. Cost Allocation Tags activate user tags for cost attribution (24-48hr delay), letting Cost Explorer/Budgets/CUR slice by them (e.g., \$10K to "Project:Checkout"). Critical: Retroactively applies to historical data once activated—no re-tagging needed.

**Why Best Here:** Activate "Project" and "Team" tags → Costs now split (e.g., Dev team owns 60% EC2 spike).
**Action Plan:** Tag Policies > Activate "Project/Team" → Wait 24hrs → View in Cost Explorer.
**Why Not Others?** Cost Explorer visualizes tags but doesn't create them; Budgets/CUR rely on tags for granularity.

#### Challenge 4: Custom SQL Analysis → **AWS Cost and Usage Reports (CUR)**

**Teaching from Scratch:** CUR is the *granular billing dataset* (CSV/Parquet to S3, hourly/daily granularity), including line-item details like resource ID, tags, pricing, reservations. Unlike Cost Explorer's summaries, CUR has 1000+ fields for custom queries (e.g., Athena: SELECT sum(cost) WHERE tag:Project='Checkout' AND region='us-east-1'). Setup: IAM role + S3 bucket; Athena integration for SQL. Essential for BI tools like QuickSight.

**Why Best Here:** Export CUR → Athena query for dashboard (e.g., costs by service + custom "ProfitCenter" field).
**Action Plan:** CUR Console > Create report (daily, all payer data) → S3 bucket → Athena table.
**Why Not Others?** Cost Explorer lacks exportable raw data; Budgets/Tags are metadata, not queryable datasets.
