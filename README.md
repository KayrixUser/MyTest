# Enterprise Executive Overview — Info & Definitions

## Scope, sources and counting

This report summarizes records available in the selected reporting population.
It is not an inventory of every enterprise system or risk.

Archer A&A supplies authorization-package and POAM information. Archer Issues
Management and policy-reference data, prepared in Snowflake, supply Findings,
Remediation Plans, Exceptions and policy links. The existing IRAMP/ATO
reporting model supplies C2S risk scores.

Counts use distinct identifiers for the item being reported: SAP, POAM,
Finding, Remediation Plan or Exception. These are different record types;
do not add their totals together. Business selections and applicable visual
filters change the displayed population.

## 1. SAP ATO Status — Archer A&A

Shows distinct SAP identifiers in the CCL/ISSO-reconciled reporting population.
A SAP represents an authorization-package/system reporting record, not an
individual device. ATO means Authorization to Operate.

- Legacy C2S: Packages identified by the existing legacy-C2S scope rule;
  displayed separately from the expiration categories.
- ATO Current: Non-C2S packages with a recorded expiration later than the
  ATO calculation’s comparison date.
- ATO Expired: Non-C2S packages with a recorded expiration on or before
  that comparison date.
- No Expiration Date: Non-C2S packages without a recorded expiration.

The donut center is the selected SAP population. Legacy C2S identifies
reporting membership, not an authorization decision. A missing expiration
date does not prove that authorization was never granted. These categories
describe the report’s recorded-date classification, not a new authorization
determination.

## 2. POAMs by Status — Archer A&A

POAMs, or Plans of Action and Milestones, track corrective work and milestones
in the authorization process.

Counts distinct POAM identifiers in the chart’s selected scope. The current
displayed statuses are Ongoing, Not Started, Awaiting Review and Rejected.

The center total represents those included statuses, not all historical
POAMs. Rejected is displayed for visibility; it does not mean approved work
is progressing. POAMs are separate from the Issues Management Remediation
Plans below.

## 3. Average C2S Risk Score by Business — IRAMP/ATO

Uses the existing combined package-level risk calculation and averages the
resulting scores within each Business. It is not a count of assets, a
percentage compliant, or a simple average of Business averages.

Risk bands:
- Low: 100–399
- Medium: 400–699
- High: 700–1000

Lower scores indicate lower risk on this reporting scale. A Low Business
average does not mean every system in that Business is Low risk. Missing
risk information is not evidence of Low risk; zero is outside the defined
risk bands.

## 4. Findings Summary and Remediation Due Windows — Issues Management

Counts distinct FINDING_KEY values.

- Open Findings: OVERALL_STATUS = Open.
- Draft Findings: OVERALL_STATUS = Draft. This is a status count, not time
  spent in Draft.
- Closed Findings — 12 Months: Currently Closed Findings with a recorded
  DATE_CLOSED inside the report’s last-12-month filter. This is not YTD or
  a complete history of closure/reopening events. Source-classified Invalid
  records may also have an Overall Status of Closed; Closed does not
  automatically mean successfully remediated.

The due-window chart includes Open Findings only and uses
EXPECTED_REMEDIATION_DATE, evaluated by calendar date at model refresh:

| Window | Definition |
|---|---|
| Overdue | Recorded date is before the calculation date. |
| 0–30 Days | Due today through 30 days ahead, inclusive. |
| 31–60 Days | Due 31 through 60 days ahead, inclusive. |
| 61+ Days | Due more than 60 days ahead. |
| No Due Date | No expected remediation date is recorded. |

Each selected open Finding belongs to one window. Missing dates remain
included; they are not automatically classified as overdue.

## 5. Open Findings by Calculated Policy Level 3 — Issues Management

Counts distinct open Finding keys associated with each recorded calculated
Policy Level 3 reference, using the Finding–Policy mapping. The policy master
supplies the reference and title; the visual is ranked by linked Finding count.

Only Findings with represented policy links contribute. A Finding can link
to several policies, so adding bars does not produce a unique-Finding total.

This is not coverage of every open Finding, a policy pass/fail result,
or a risk-weighted ranking.

## 6. Remediation Plans by Estimated Completion Window — Issues Management

A Remediation Plan tracks corrective actions, responsibility and estimated
completion. A plan can address several Findings; a Finding can have several
plans.

Counts distinct REMEDIATION_PLAN_KEY values.

Total Remediation Plans includes all source statuses in the selected scope.
The timing chart and its associated selected-status count include
In Progress and Not Started only. Do not interpret that subset as every
unresolved plan.

Uses the plan’s own ESTIMATED_COMPLETION_DATE:

- Past Estimated Date: Before the calculation date.
- 0–30 Days: Today through 30 days ahead, inclusive.
- 31–60 Days: 31 through 60 days ahead, inclusive.
- 61+ Days: More than 60 days ahead.
- No Estimated Date: No estimated completion date is recorded.

Original, approved-plan, review and Finding dates are not substituted.
Passing an estimate is not, by itself, proof that an approved deadline
was breached.

Organizational selections use the plan’s recorded assignments, not its
related Finding’s organization. Expanded membership rows do not create
additional plans: each plan is counted once within the selection.

## 7. Exception Requests by Risk and Review Window — Issues Management

Counts distinct EXCEPTION_KEY values, grouped by the recorded RISK_RATING
and review-date window. The current visual does not restrict the population
to an Open or Approved status.

- Past Review Date: The review date used by the chart is before its
  comparison date.
- No Review Date: That review-date field is blank.

Review timing is separate from expiration. An Approved request can have
a past review date and a future expiration.

Risk Rating is the Exception’s source rating, not the C2S system score.
A blank rating means missing information, not Low risk.

## Detail Pages, Selections and Interpretation

Open the Remediation or Findings detail page for additional analysis.
Right-click an available chart segment and select Drillthrough to inspect
records in that selected context. Clear selections to restore the broader
view; check the active Business and filters before comparing totals.

Monthly detail charts group recorded dates for currently Closed records:
Findings use DATE_CLOSED; Remediation Plans use ACTUAL_COMPLETION_DATE.
Missing dates are excluded. Use the displayed period and treat partial
months accordingly.

Count cards display 0 when no loaded records meet the current selection.
This is not a statement about the unfiltered enterprise population.
Not recorded, missing dates and missing ratings remain data gaps, not
zero values.

A slicer choice containing several names selects that stored combination;
it does not automatically select every record associated with each
individual name.

The refresh timestamp identifies the loaded reporting snapshot, not
guaranteed real-time source activity. Stored due-window calculations
update at refresh.

Timing colors describe distance to the recorded date, not risk or compliance.
A green future-date bar does not mean the work is complete or low risk.
Use the category labels to distinguish future dates from missing dates;
risk-chart colors follow the numerical risk bands.
