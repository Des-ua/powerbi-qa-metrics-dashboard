# QA Metrics Dashboard — Power BI

A Power BI report built on bug-tracker and test-execution data for a fictional
e-commerce project (the same domain as my OpenCart testing project).

**The data is synthetic.** It was generated with a script to imitate a raw Jira
export, including the mess a real export usually contains. The analysis, the
cleaning decisions and the report are my own work.

![QA metrics dashboard](dashboard.jpg)

---

## Files

| File | What it is |
|---|---|
| `QA_Dashboard.pbix` | The Power BI report |
| `bugs.csv` | 263 raw bug reports |
| `test_runs.csv` | 794 raw test-case executions |
| `dashboard.jpg` | Screenshot of the report page |

---

## Data model

**bugs.csv** — one row per bug report: `bug_id`, `summary`, `module`, `severity`,
`priority`, `status`, `reported_date`, `closed_date`, `reporter`, `assignee`,
`environment`, `found_in_build`.

**test_runs.csv** — one row per test-case execution: `run_id`, `test_case_id`,
`module`, `build`, `run_date`, `result`, `executed_by`, `duration_min`.

Eight modules: User Account, Payments, Cart, Shipping, Admin Panel, Checkout,
Search, Product Page. Eighteen builds, 1.4.0 through 1.9.2. Reporting period:
March–September 2026.

---

## Data cleaning in Power Query

This is the part that mattered most. The dashboard took an evening; working out
what to do with the dirty values took longer and is the reason the numbers can be
trusted.

| Problem found | What I did | Why |
|---|---|---|
| Three date formats in one column: `2026-04-09`, `15/05/2026`, `05.05.2026` | Custom column with a conditional M expression that detects the separator and rebuilds the value with `#date(...)` | Power BI's automatic type detection silently failed on the mixed column and left it as text. Parsing by separator is explicit and reproducible. |
| Build numbers converted to dates: `1.4.0` became `01.04.2000` | Set `found_in_build` and `build` to **Text** in the applied type step, replacing the existing step rather than adding a new one | This is a silent corruption — no error is raised and 18 builds collapse into nonsense. It has to be caught by looking at the data, not at the error count. |
| Inconsistent capitalisation: `Checkout`, `CHECKOUT`, `checkout` | Transform → Format → Capitalize Each Word | Without it, one module is counted as three and every per-module figure is wrong. |
| Leading and trailing spaces | Trim (not Clean — Clean removes non-printable characters, which was not the problem here) | Same reason: ` Payments` and `Payments` group separately. |
| Two fully identical rows | Remove Duplicates **across all columns** | Important detail: running Remove Duplicates with a single column selected collapsed the table from 260 rows to 19, because many bugs legitimately share a `summary`. The step has to be applied with no column selected. |
| Placeholder row `BUG-1261` with empty fields | Removed | It carries no information and distorts counts. |
| 102 empty `closed_date` values | **Kept** | An empty `closed_date` does not mean missing data — it means the bug is still open. Deleting these rows would delete exactly the bugs the dashboard exists to surface, and would make the average time-to-close look better than it is. |
| 32 empty `assignee` values | **Kept** | Unassigned bugs are a real finding, not a defect in the export. |
| No time-to-close measure | Added `days_to_close` = `Duration.Days([closed_date] - [reported_date])`, null when the bug is open | Computed at the source so every visual uses the same definition. |
| Months needed for the trend chart | Added a calculated column `FORMAT([reported_date], "YYYY-MM")` | Using the built-in date hierarchy gave drill-down behaviour rather than a flat month axis; `MMMM` would have sorted alphabetically (April, August, December…). `YYYY-MM` sorts chronologically on its own. |

**Verification after cleaning:** 260 rows, 8 distinct modules, 5 distinct
severities, 6 distinct statuses, 0 errors in the date columns, 158 bugs with a
closed date and 102 without.

---

## The report page

**KPI cards:** total bugs (260), open bugs (102 — Open, In Progress, Reopened),
Blocker + Critical (29), average time to close (23.5 days).

**Bugs by module and severity** — stacked column chart. The palette is ordered
dark red → red → orange → yellow → grey so that severity reads as a gradient
rather than as eight unrelated colours.

**Status distribution** — donut chart. Green shades for resolved states, blue for
in-flight, red for Reopened, grey for Rejected.

**Bugs reported per month** — column chart on a single neutral colour. Red is
reserved for severity elsewhere on the page, so reusing it here would make the
page say two different things with the same colour.

**Oldest open bugs** — table filtered to Open / In Progress / Reopened, sorted by
report date ascending.

**Slicers:** module, severity, build, reporting period.

---

## What the dashboard shows

- **User Account carries the most bugs (41) and the most critical ones (7).**
  Payments is second overall (39) but has fewer Blocker/Critical issues (4). If
  test effort were allocated by headline bug count alone, User Account would still
  be the right place to spend it — which is not always the case and is worth
  checking rather than assuming.
- **Reporting peaked in May and July (50 bugs each) and has fallen since**
  (30 in August, 26 in September). On its own this is ambiguous: it can mean the
  product is stabilising, or that testing slowed down. Pass Rate per build —
  second page, in progress — is what separates the two.
- **102 of 260 bugs are still open, and the oldest has been open since 2 March
  2026.** Several Critical and Blocker issues sit in that list. A backlog of
  reopened and critical bugs older than six months is a process finding, not a
  testing finding.
- **Reopened is the third-largest status (35).** A bug that is closed and comes
  back points at either incomplete fixes or verification that is too shallow.

---

## Still to do

Second page, after the modelling and DAX work: a date table, a module lookup
table with a one-to-many relationship to both fact tables, and measures for Pass
Rate, Failed Runs, Bugs per Build and Avg Days to Close, shown as quality per
build.

---

## Tools

Power BI Desktop, Power Query (M), DAX.

