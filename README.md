# Maternal & Child Health Programme Dashboard (MERL)

**A facility-level monitoring dashboard that turns routine monthly health reports into a ranked supervision visit list.**

Built with PostgreSQL, Tableau and Excel by **Aisha O. Inuwa**, Healthcare Data Analyst, Abuja, Nigeria.

> **Please read this first.** Every number in this project is **simulated**. The dataset was
> constructed to mirror the structure of routine monthly reporting for 12 health facilities in
> the FCT, Nigeria, over 24 months. No real patient record, no real facility record and no real
> DHIS2 extract was used. Real routine health data cannot be published openly, but the *logic*
> of how it is analysed can be shown in full. Everything else here is exactly how it is done on
> a live programme: the indicator definitions, the SQL, the exception rule and the data quality
> checks.

---

## 1. The problem

A maternal and child health programme runs in three Area Councils of the FCT: Gwagwalada, Kuje
and Bwari. Twelve facilities submit a monthly report — how many pregnant women came, how many
delivered, how many babies were immunised.

Every month somebody types those numbers into a spreadsheet. The programme manager looks at the
total, sees "ANC4 coverage 59%", and asks the only question that matters:

> **So which facility do I visit next month, and what do I tell them when I get there?**

A spreadsheet total cannot answer that. This project answers it.

---

## 2. What the analysis found

The programme average is 58.8% ANC4 coverage against a 70% target. Stop there and the conclusion
is "the whole programme is underperforming, we need more funding." That conclusion is wrong.

Broken down by facility, the 12 sites are not one weak programme. They are **two different
programmes wearing the same name:**

| | Strong group (7 sites) | Struggling group (5 sites) |
|---|---|---|
| ANC4 coverage | 66–82% | 31–45% |
| Immunisation completion | 80–87% | 57–61% |
| **Immunisation dropout** | **4–9%** | **28–35%** |
| Targets met (out of 5) | 3–5 | 0 |

Five facilities — Kuje PHC, Ushafa PHC, Dobi PHC, Gaube PHC and Igu PHC — miss **all five
targets, every month, for 24 consecutive months.** This is not a bad month. Nothing in the trend
is correcting itself.

The sharpest finding is in the dropout rate. In those five sites, roughly **one in three babies
who START immunisation never returns to finish it.** In the strong sites it is about one in
twenty-five.

That matters because a baby who started but did not complete is not a family that refused the
service. That family already trusted the facility enough to walk in once. Something then happened
— a vaccine stock-out, the health worker absent on immunisation day, no return date written on
the card, no follow-up when they missed. **That is a fixable problem, and far cheaper to fix than
recruiting a new family.**

### The one line for the review meeting

> Coverage tells you how many people you reached. Dropout tells you whether the service actually
> worked when they got there. We do not have a funding problem across 12 facilities — we have a
> service-delivery problem in 5 named facilities, and dropout is where to start.

Spread next quarter's supervision budget evenly across 12 sites and nothing changes anywhere. Put
it into those 5, on immunisation defaulter tracing and ANC return-visit reminders, and the
programme average moves.

---

## 3. The five indicators

Every indicator here has three things written down: a **numerator**, a **denominator** and a
**target**. Without all three you do not have an indicator — you have a number.

| Code | Indicator | Numerator | Denominator | Target | Good direction |
|---|---|---|---|---|---|
| `ANC4` | ANC4 coverage | Pregnant women who completed 4+ antenatal visits | Expected pregnancies in the catchment this month | 70% | Higher |
| `SBA` | Skilled birth attendance | Deliveries attended by a doctor, nurse or midwife | All deliveries the facility reported | 85% | Higher |
| `FDR` | Facility delivery rate | Deliveries that happened inside a facility | Expected deliveries in the catchment this month | 65% | Higher |
| `IMM` | Immunisation completion | Infants who completed the routine schedule | Expected surviving infants in the catchment this month | 80% | Higher |
| `DROP` | Immunisation dropout | Infants who started **minus** those who completed | Infants who started | 10% | **Lower** |

**Why some denominators are "expected" and others are "reported".** SBA uses deliveries the
facility reported — of the women who delivered here, how many had a skilled attendant? That is a
*quality of care* question. FDR uses expected deliveries from the catchment population — of all
the women in this area due to deliver, how many reached a facility at all? That is an *access*
question. Same deliveries, different denominator, completely different meaning. Mixing them up is
the most common indicator error in routine health reporting.

Expected figures come from standard population estimates: pregnancies approximately 4.0% of
catchment population per year, deliveries 3.9%, surviving infants 3.6%, divided by 12 for the
month.

**Why DROP earns its place.** ANC4, SBA, FDR and IMM all ask "how many did we reach?" DROP is the
only one that asks "did it work?" A facility can look respectable on coverage while quietly
losing a third of its babies halfway through the schedule. DROP makes that visible.

---

## 4. What is in this repository

```
data/                      6 CSV files - the full simulated dataset
  facilities.csv           12 facilities: LGA, type, catchment population, expected denominators
  indicator_targets.csv    the rule book - 5 indicators, definitions, targets, direction
  mch_monthly_raw.csv      288 rows - the raw monthly counts, exactly what a facility submits
  mch_indicators.csv       1,440 rows - tidy/long: one row per facility x month x indicator
  exception_list.csv       49 rows - who is failing what, and for how many months in a row
  data_quality_log.csv     32 rows - every problem the checks caught, with a plain reason

sql/                       PostgreSQL - run in order 01 -> 04
  01_schema_and_load.sql   tables, keys, indexes, CSV load
  02_indicator_views.sql   raw counts -> indicators; programme, LGA and quarter roll-ups
  03_league_table_and_exceptions.sql   the league table + gaps-and-islands streak logic
  04_data_quality_checks.sql           5 checks that run BEFORE anyone opens the dashboard

dashboard/                 the 4 Tableau dashboard sheets exported as PNG
docs/
```

### How to run it yourself

You need PostgreSQL (or DBeaver connected to any PostgreSQL database).

1. Open `sql/01_schema_and_load.sql` and run it. It creates the tables and loads the six CSV
   files from `data/`. Adjust the file paths at the top of the script to match where you saved
   the repository.
2. Run `sql/02_indicator_views.sql`. This converts the raw monthly counts into the five
   indicators and builds the programme, LGA and quarterly roll-ups.
3. Run `sql/03_league_table_and_exceptions.sql` for the facility league table and the exception
   list with streak lengths.
4. Run `sql/04_data_quality_checks.sql` for the five data quality checks.

### How the simulated dataset was built

The 12 facilities, their Area Councils, facility types and catchment populations were written out
by hand in Excel, using realistic FCT population sizes. The expected denominators were then
calculated in Excel with the standard population percentages above.

The 24 months of facility reports were generated in Excel as well: each facility was given a
baseline performance level and a small month-to-month variation, so that seven sites sit around
or above target and five sit well below it. A handful of deliberate data problems were typed in
on purpose - one impossible value, some missing monthly reports, a few late submissions and one
transposed-digit spike - so that the data quality checks in `04_data_quality_checks.sql` have
something real to catch.

Everything after that point is SQL. The indicators, the roll-ups, the league table, the streak
logic and the quality log are all produced by the four scripts in `sql/`, and the CSV files in
`data/` are the exported results of those queries.

---

## 5. The exception rule

A dashboard that only shows what happened is a report. A dashboard that tells you who to visit is
a decision tool. This is the rule that makes it one:

> **A facility becomes an exception on an indicator when it misses that target for 2 or more
> reported months in a row.**

**Why "in a row" and not "on average"?** One bad month is usually a stock-out, a flood, or the
midwife on leave — it corrects itself, and a supervision visit is wasted. Two bad months in a row
is a system that is not correcting itself, and that is exactly what a supervision visit can help.
Averaging over the year hides both: a facility that collapsed in the last four months and one that
had a single terrible January can show the same annual average.

In SQL this is the **gaps-and-islands** pattern: number all the rows, number only the failing
rows, subtract the two. Consecutive failures share the same difference, so that difference becomes
the streak's group id — then count each group.

One rule had to be made explicit: **a month with no report does not reset the streak, and does not
count as a pass.** Only reported months are counted. A facility should never get a cleaner
exception record simply by not submitting.

Priority comes from streak length: 6+ consecutive months is High, 3–5 is Medium, 2 is Watch. That
gives the programme manager a ranked visit list instead of 49 undifferentiated problems. At the top
of that list are PHC-002 Dobi PHC and PHC-006 Gaube PHC, both High priority.

---

## 6. Data quality — the page people skip

In routine health data, **a fall in a chart is more often a reporting problem than a service
problem.** If you cannot tell the two apart, you will send a supervision team to a facility whose
only crime was a phone with no airtime. Five checks run before anyone opens the dashboard:

1. **Missing monthly report** — build every facility × month pair that *should* exist, then find
   the ones that do not. A missing row is invisible unless you generate the full grid first; it
   just quietly drags the average down.
2. **Impossible values** — ANC4 greater than ANC1, skilled births greater than total deliveries,
   completed greater than started. These are not judgement calls, they are arithmetically
   impossible: a subset cannot be bigger than the set it came from. The log found one — Zuba PHC,
   March 2026, ANC4 = 28 against ANC1 = 22.
3. **Suspicious spike** — a month more than 3× that facility's **own** median. Compared against
   itself, not the programme average: 200 visits is normal for a general hospital and impossible
   for a 12-bed PHC. Usual causes are a transposed digit (28 keyed as 82) or an outreach campaign
   counted in two registers.
4. **Denominator sanity** — more clients than the catchment population can produce means either
   the catchment figure is stale or the facility is serving a neighbouring ward. Both matter, and
   the second means the neighbouring facility's low numbers are not its own fault.
5. **Late submission** — reports are due by the 10th. Late data is not wrong data, but a dashboard
   reviewed on the 12th with half the reports missing produces a confident, wrong decision. 30
   late submissions are logged.

**NULL and 0 are not the same thing.** `0` means "we opened, we worked, we saw nobody." `NULL`
means "we do not know what happened here." Write 0 where you meant NULL and you have invented a
failure that never occurred. The schema stores NULL for a non-submitted month, and every indicator
view excludes those rows from the denominator rather than treating them as zeros.

---

## 7. The four dashboard sheets

### 1. Programme Overview
![Programme Overview](dashboard/01_programme_overview.png)

Five KPI cards with a target and a direction-aware trend arrow (for dropout, falling is improving,
and the card says so in words), the 24-month trend of each indicator against its target line, and
coverage by Area Council. This is the "where are we" page.

### 2. Facility League Table
![Facility League Table](dashboard/02_facility_league_table.png)

Every facility × every indicator, colour-coded green / amber / red, ranked by how many of the five
targets it meets. This is the page that ends the "is it really that bad in Dobi?" argument, because
you can point at the row.

### 3. Where Are We Losing Mothers and Babies?
![Dropout Story](dashboard/03_dropout_story.png)

The funnel from ANC1 through ANC4, facility delivery, immunisation started and immunisation
completed; dropout by facility; and coverage plotted against dropout, where the two-group pattern
becomes impossible to miss. This is the page that changes the decision.

### 4. Exception Report & Data Quality
![Exceptions and Quality](dashboard/04_exceptions_and_quality.png)

The ranked visit list with streak lengths, how many indicators each site is failing, the quality
issues found, and reporting timeliness. This is the page you take into the meeting.

The order is deliberate: **where are we → who is behind → why → what to do about it.** A dashboard
should walk the reader to a decision, not dump 20 charts on one page and leave them to work it out.

---

## 8. Honest limitations

- **The data is simulated.** The patterns were designed by me, so the analysis "finds" what was
  built in. It demonstrates the method; it is not evidence about any real FCT facility.
- **The denominators are population estimates.** Real catchment figures are often years out of
  date, and a wrong denominator produces a wrong rate no matter how clean the numerator is. On a
  real programme, checking the denominators would come before any of this.
- **Facility reports are not the whole picture.** Deliveries at home with a traditional birth
  attendant never enter this dataset, so facility-based indicators systematically miss the women
  who are hardest to reach.
- **Below-target does not automatically mean poor performance.** A facility in a hard-to-reach ward
  during rainy season faces a different task from an urban PHC. The exception list starts a
  conversation; it does not conclude one.

---

## 9. Tools used

PostgreSQL (window functions, CTEs, gaps-and-islands) · DBeaver · Tableau · Excel · MERL indicator
design · DHIS2-style routine data · data quality assurance

---

*Aisha O. Inuwa — Healthcare & Program Data Analyst, Abuja, Nigeria. I build the measurement systems
that tell health programmes where to act next.*
