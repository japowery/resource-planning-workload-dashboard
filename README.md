# V&R Resource Planner - SQLite Multi-User App

This package is a SQLite-backed web application for V&R portfolio/resource planning.

## Run

1. Unzip the package.
2. Double-click `start_windows.bat`, or run:

```bat
python vr_app.py --import-on-start
```

3. Open:

```text
http://127.0.0.1:8765
```

For team use, run the app on one central internal machine and have users browse to:

```text
http://<host-machine>:8765
```

Do **not** let multiple users directly open or sync `resource_planner.sqlite` through OneDrive/SharePoint. One server process should own the SQLite database. The VantagePoint Excel exports can live in the OneDrive/SharePoint-synced folder and be imported into SQLite.

## What changed in this version

- Project Detail phase table narrowed so it no longer stretches across the full page.
- Depreciation Study Phases table now uses consistent centered row text.
- Project Detail staff snapshot now includes Active, Upcoming, and On Hold projects.
- Phase status dropdowns now save correctly and retain their color coding after refresh.
- Base Hrs was removed from the phase table; Scaled Hrs is now editable and reallocates the project’s pre-filing budgeted hours across included phases.
- Portfolio project registry defaults to ascending alphabetical sort by project description.
- Project status is color-coded.
- `Unassigned` was removed from the status list. Blank/unassigned rows migrate to `Active`.
- Nearest deadline panel is now **Nearest Upcoming Filing Deadlines** and excludes filing dates before today.
- Hearing can be marked **N/A**.
- Hearing N/A suppresses post-filing forecast effort.
- Project descriptions can be manually overridden in the registry without changing the VantagePoint source description.
- Dashboard note columns were removed; task comments remain in Project Detail.
- Portfolio financial progress is now based on `effort_to_date / budget`, not task completion.
- Portfolio rows show **Total Hours Remaining** = `max(0, budget - effort_to_date) / blended_rate`.
- Financial progress bars now use one color and show only Effort to Date / VantagePoint Budget.
- Individual daily budgeted-hours chart was rebuilt with axes, labels, non-stretched SVG text, and hover tooltips.
- Average task schedule variance by team was removed and replaced by a budget-risk panel.

## Post-filing mechanics

Phase 10 is post-filing and includes:

- Discovery Requests
- Rebuttal Testimony
- Hearing

The VantagePoint budget often starts as pre-filing only. After filing, it may or may not switch to a combined pre-filing + post-filing budget. Because post-filing work is often time-and-materials and can vary widely, the app now separates **budget accounting** from **workload forecasting**.

Default post-filing forecast rule:

```text
Post-filing forecast hours = 50% of pre-filing budgeted hours
```

This applies only when a hearing exists. If Hearing N/A is checked or the hearing date is blank, post-filing forecast hours are zero.

Post-filing is defined as **post Day 210** in the effort distribution curve. The app uses rows after Day 210 to allocate post-filing forecast hours by role and across the period from Filing Date to Hearing Date. This means witness time can increase relative to PM/Analyst if the distribution curve says it should.

## Import flow

`config.json` points to the source files:

- `financials_path`: VantagePoint Financials Export
- `labor_detail_path`: Labor Detail
- `staff_csv_path`: V&R staff file
- `effort_distribution_csv_path`: V&R effort distribution curve
- `task_library_csv_path`: depr-study task list

Run:

```bat
python vr_app.py --import-only
```

or click **Import Latest XLSX** in the UI.

Imports update financial/labor data and seed defaults for new projects. Manual overrides in the registry and task status tables are preserved.

## Migrating data from an older app database

Use `migrate_from_existing_db.py` to copy manual edits from an older SQLite database into this version.

Best practice:

1. Stop the old app.
2. Copy all three old files into one folder if they exist:
   - `resource_planner.sqlite`
   - `resource_planner.sqlite-wal`
   - `resource_planner.sqlite-shm`
3. Back up the new database.
4. From this package folder, run:

```bat
copy resource_planner.sqlite resource_planner_BACKUP_before_migration.sqlite
python migrate_from_existing_db.py --from "C:\path\to\old\resource_planner.sqlite" --to resource_planner.sqlite
```

Use `--copy-blanks` only if you intentionally want blank fields in the old database to clear values in the new database.

The migration script copies:

- project status
- witness / PM / analyst
- start / filing / hearing dates
- Hearing N/A flag when available or inferred from hearing date text
- budget mode and post-filing fields when available
- task statuses, include flags, and task comments
- audit log rows

It does not copy old source financial/labor lines. Those should come from the latest VantagePoint and Labor Detail imports.

## Concurrency

Editable project registry rows and task rows have revision numbers. If two users edit the same row at the same time, the stale write receives a conflict response instead of silently overwriting someone else.


## Portfolio Capacity Metric Definitions

The Portfolio page separates three concepts that should not be divided into one another:

- **Budget Remaining**: visible project budget less effort-to-date. This is total backlog dollars and is not horizon-limited.
- **Budget Remaining as Hours**: Budget Remaining divided by the blended planning rate.
- **Horizon Planned Budgeted Hours**: the portion of budgeted/forecast hours scheduled between the selected as-of date and horizon end, based on project start, filing, and hearing dates. Projects with filing dates before the as-of date or missing filing dates are flagged because their remaining budget is not automatically scheduled as pre-filing work.
- **Target Utilization Hours**: the target billable hours needed for the selected staff/team/group to hit utilization during the same horizon.
- **Planned Hours - Target Utilization**: Horizon Planned Budgeted Hours minus Target Utilization Hours. Positive means the visible workload is above target; negative means the visible workload is below target.

A large Budget Remaining value divided by a much smaller Horizon Planned Budgeted Hours value is not an implied billing rate; it usually means remaining budget exists outside the selected horizon, on projects with past filing dates, or on projects with missing/incorrect dates.

## vNext date-highlighting patch

Filing and hearing dates are now highlighted red in Portfolio > Project Registry + Financials and Individual > Active Project Snapshot when the date is before the current date. Hearing N/A remains gray and does not trigger past-date formatting.

## vNext UI Refinements

- Staff Horizon Planned Hours bars are now measured against each staff member's own target utilization hours, not relative to the highest person on the chart.
- Staff bars use green for within +/-10% of target, orange for below target, and red for more than 10% above target.
- Financial progress bars now use one color and show Effort to Date / VantagePoint Budget only.
- Project Registry + Financials hides budget mode and post-filing override controls for now. Post-filing forecast is automatic: Hearing N/A/blank = zero; valid hearing date = 50% of pre-filing hours.
- Start Date is no longer conditionally flagged as past due; only Filing Date and Hearing Date are flagged red when before today.
- Project Registry and Individual tables have draggable column headers for width adjustments.

## vNext Role + Effort Curve Inputs

- Added editable **Role Assumptions** on the Data + Audit tab. Role share, standard hours, billing rates, and target utilization can be changed globally. If role share is edited, the app preserves total standard project hours and recalculates standard hours from the edited shares.
- Added editable **Effort Distribution Curve** on the Data + Audit tab. Rows through Day 210 are treated as pre-filing. Rows after Day 210 are treated as post-filing.
- Added reset buttons for both tables to restore V&R defaults.
- Imports no longer overwrite customized role assumptions or effort distribution values; those tables are seeded only when empty or when a user intentionally clicks reset.
- The blended planning rate, budgeted hours, role allocations, staff capacity, pre-filing timing, and post-filing timing all use the saved assumption tables from SQLite.

## Final portfolio / individual chart polish

This version tightens the Portfolio layout so the Status, Deadline, and Budget Risk panels use less space while the Staff Horizon Planned Hours panel has enough vertical room to show every staff member. The staff utilization bars are measured against each person's own target utilization hours, not against the largest person in the chart.

The Individual daily-hours chart is now stacked by project. Each business-day bar shows the project-by-project makeup of the planned hours for that day, and hovering over a segment shows the project and hours.

## vNext update: manual projects, overrides, planning units, and Individual / Team polish

This version adds a guarded workflow for non-standard cases:

- **Manual / upcoming projects:** add a project before it exists in VantagePoint. If the same code later appears in the VantagePoint export, the financial import updates the project financials while retaining manually-entered registry fields, task statuses, comments, and planning units.
- **Financial overrides:** imported budget and effort remain visible and preserved. Optional budget/effort overrides are stored separately, require a reason, and appear with an Override indicator in the UI. Clear the override to return to the imported VantagePoint values.
- **Planning units / subprojects:** split one VantagePoint project code into multiple app-native planning units for capacity planning. The parent project remains the financial reconciliation container; planning units are used for staff assignments, dates, and workload forecasts.
- **Individual / Team tab:** renamed from Individual because the same analysis supports individuals, teams, and the full group. The daily budgeted-hours chart is stacked by project/planning unit, and hovering a bar segment shows only that segment's project/time. The Active Project Snapshot can be sorted by any column; default sort remains descending by Next 30d Hrs, which is also displayed as a bar.

Best practice: use planning units when one VantagePoint code contains multiple regulatory assignments or multiple analysts/PMs. Keep allocations near 100% across planning units unless you intentionally want to plan only part of the parent project.


## Planning-hours calculation update

Pre-filing resource planning now uses remaining pre-filing hours, not total original budgeted hours. The app calculates remaining pre-filing hours as max(0, effective pre-filing budget - pre-filing effort to date) / blended rate, then schedules those remaining hours from the selected as-of date through the filing date. Changing a filing date therefore changes the daily and horizon planned-hour load. Post-filing forecast hours remain independent of VantagePoint budget and are scheduled only between filing and hearing when a hearing is required.

## vNext Phase-Level Financials Update

This version uses `data/VantagePoint Financials Export.xlsx` as the preferred financial source. The importer reads phase-level total rows from the VantagePoint export:

- Project number is parsed from the VantagePoint project number field before the period.
- Phase is parsed from the phase field / phase total row.
- The application creates one planning row per project-phase combination.
- Revenue true-up / ZZREV phases default to `Exclude`, which behaves like inactive for planning but keeps the source line available for reconciliation.

Portfolio users can directly edit Budget and Effort cells when a temporary planning override is needed. The app stores those values as overrides separate from imported VantagePoint values. Use the reset icon next to Budget or Effort to clear the override and return to the imported VantagePoint value.

Manual projects remain supported for upcoming work before a project appears in VantagePoint. If the same project/phase code later appears in the VantagePoint export, financial values refresh from VantagePoint while manually entered dates, staff, and task statuses are retained.

Planning units/subproject splits have been removed from the UI because phase-level VantagePoint financials are now the preferred native mechanism for subproject planning and reconciliation.

Added dashboards:

- Go / No-Go: prototype scenario simulator for testing a hypothetical project against team and analyst capacity.
- Historic Data: placeholder for future trend/insight analytics.
- Projections: placeholder for future 3-5 year revenue and staffing forecasts.

## Final Prototype Update - Phase Financials Workflow

This version uses the simplified `data/VantagePoint Financials Export.xlsx` format. Each project-phase row is treated as its own planning row. The Portfolio registry now uses Start Date + End Date + Post-Filing? instead of separate filing/hearing fields. For pre-filing rows, End Date generally means filing date. For post-filing rows, End Date generally means hearing or post-filing completion date.

Financial overrides are now edited directly in the Portfolio registry table by typing into Budget or Effort. The small reset icon next to either value clears that override and relinks the value to the VantagePoint source import.

The older Data + Audit financial override and planning-unit split UI has been removed. Manual/upcoming projects can be added directly above the Portfolio registry table.

## Latest QA update: numeric phase import and financial totals

The VantagePoint financial import now uses only numeric phase total rows from `data/VantagePoint Financials Export.xlsx`. Non-numeric phases such as `BDM`, `ZZREV`, `ZZZZZ`, and similar rows are excluded during import rather than carried into the planning database. Project-level total rows are also excluded so project budgets/effort are not double-counted against phase-level rows.

For example, TVA / Tennessee Valley Authority now imports the numeric pre-filing phase once, with budget $70,000 and effort $69,725.05, instead of also importing the project-level total row.

The Portfolio registry displays Budget and Effort in whole-dollar accounting-style formatting with commas. Total Hours Remaining is rounded to whole hours.

## vNext UI + Settings polish

- Removed the top-bar `Your name` field and `Import Latest XLSX` button from the browser UI. Refresh remains available.
- Renamed **Data + Audit** to **Settings** and moved it to the far right of the tab bar.
- Settings now focuses on user-editable staff, billing-rate, planning-role, and effort-curve assumptions. Source import and audit panels are no longer shown there.
- Added Staff Capacity + Revenue Settings, including role, team, billing role, hours/week, PTO days, and utilization goal. Group projected annual revenue is shown only in aggregate in the Settings summary.
- Added editable 2026 GFVRC billing rates. These are seeded from the uploaded 2026 rate sheet and can be reset to defaults.
- Portfolio **Planned Hours - Target Utilization** now shows a capacity factor and uses green when planned hours are within +/-10% of target, orange when below target, and red when more than 10% above target.
- Staff Horizon Planned Hours now uses a compact table with Employee, Active Projects, Capacity Factor, and Hours columns.
- Go / No-Go now emphasizes the visual fit bars and revenue-impact card; the detailed table is hidden.

## Latest Staff Horizon Panel Adjustment

The Staff Horizon Planned Hours panel now limits the Active Projects count and planned-hour calculation to rows with Status = Active. Upcoming and On Hold rows are excluded from that specific panel. The Active Projects count is de-duplicated by parent project number, so multiple active phases under the same project count as one active project.

The Employee column is no longer bolded, and Active Projects, Capacity Factor, and Hrs columns are centered with tighter spacing to reduce vertical and horizontal footprint.

## v12 Operating-System Enhancements

This version adds several workflow features on top of the existing v12 UI:

- Portfolio work queue chips: All Visible, Active Only, Needs Setup, Past End Date, Ending Soon, Over 90% Spent, and Post-Filing.
- Today's Decisions cards to surface active rows needing setup, past end dates, near-term deadlines, staff over target, and budget-risk rows.
- Recent-change panel using the audit log plus latest source-file import summaries.
- Data-confidence scoring in the Portfolio project table.
- Data hygiene summary for status/date/staff mismatches.
- Role-capacity panel for Witness, PM, Analyst, and Admin capacity markets.
- Individual / Team weekly work plan above the daily stacked chart.
- Project Detail row-level change history panel.
- Settings staffing-scenario sandbox and future external-data input roadmap.

These changes do not require new external data. They use the existing SQLite tables, audit log, source file import log, staff settings, role assumptions, and project registry.


## v12.2 UI + Detail Cleanup

- Portfolio search was replaced with a Work Queue dropdown.
- What Changed Recently and Data Confidence panels were removed from Portfolio.
- Role Capacity is fixed to all active rows and no longer responds to portfolio staff/team/status filters.
- Project registry confidence column was removed; staff columns were widened; date/financial columns were tightened.
- Budget/Effort financial override reset is now one compact VP reset button for both fields.
- Project Detail now preserves the selected project, supports quick search, and shows pre-filing projects/tasks only.
- Labor Detail Defaults now pull labor by base project number so phase-level rows populate correctly.
- Settings no longer exposes PTO days; capacity assumes 20 PTO days plus 10 holidays per year for everyone.


## v23 debug rebuild

Rebuilt from the working v21 package and re-applied only the Individual / Team snapshot updates: removed Schedule Variance and set Task Variance to n/a for post-filing rows.

## v24 UI/Go-No-Go Fixes

This package includes the v23 app plus the following targeted updates:
- Staff removal from Settings now requires three explicit confirmations before marking a staff member inactive.
- Portfolio panels are kept together left-to-right, with Nearest Upcoming End Dates moved to the right of Staff Horizon Planned Hours.
- Nearest Upcoming End Dates was tightened so the end date remains visible.
- Individual / Team Active Project Snapshot has a narrower Code column to preserve the Next 30d Hrs column.
- Go / No-Go workload simulator now prorates hypothetical project hours to the selected horizon based on scenario dates, uses role-appropriate team/analyst load, and should no longer overstate capacity impact by applying the entire project load to every team/analyst.

## v26 project-detail updates

- Project Detail now tracks depreciation-study progress at the phase level rather than each individual task.
- The nine pre-filing phases are shown in the requested order, with Selection of Estimates and Interviews / Field Visit before Depreciation Calculation.
- Each phase has one status and one editable base-hours total. Underlying task rows remain in SQLite for historical continuity and task-progress calculations.
- Phase rows include expandable typical-task breakdowns.
- Labor Detail Defaults has been renamed Effort-to-Date.
- Project Detail includes a Staff View selector and a compact active-project snapshot for that staff member.
- Quick Search now auto-loads the only matching project, fixing the one-result selection issue.
