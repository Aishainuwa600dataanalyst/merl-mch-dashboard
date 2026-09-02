# Build This Dashboard in Tableau — Step by Step

This guide takes you from the CSV files to the four finished sheets. It assumes you have
never built a dashboard in Tableau before, so nothing is skipped.

Work through it in order. Do not jump to the pretty charts — the connection and the
first calculated field are where the actual learning is, and everything after that is
repetition.

**Time needed:** about 3 hours the first time. Maybe 40 minutes once you know it.

**What you need:** Tableau Public (free — tableau.com/products/public/download) or
Tableau Desktop. Tableau Public saves your work online, which is what you want anyway
because it gives you a live link to put on LinkedIn.

> One warning about Tableau Public: **everything you save is public.** That is fine here
> — our data is simulated. Never put real patient or facility data on Tableau Public.

---

## Part 1 — Connect the data (20 minutes)

### Step 1.1 Open Tableau and connect

1. Open Tableau. On the left under **Connect → To a File**, click **Text file**.
2. Browse to the `data` folder and pick **`mch_indicators.csv`**.
3. Click **Open**.

You will land on the **Data Source** page and see your 1,440 rows.

**Why this file and not the raw one?** `mch_indicators.csv` is already in *tidy / long*
shape — one row per facility per month per indicator, with the value, the target and the
direction all on the row. Tableau is built for exactly this shape. If you connect the raw
counts instead, you will spend the rest of the evening writing division formulas inside
Tableau. The division is already done, once, in the SQL — which is the whole point of
having an indicator layer.

### Step 1.2 Add the facilities file (a join)

We want catchment population and facility type, which live in the other file.

1. On the Data Source page, look at the canvas at the top. You will see a box called
   `mch_indicators.csv`.
2. In the left panel under **Files**, drag **`facilities.csv`** onto the canvas next to it.
3. Tableau will open a join dialog. Set it up like this:
   - Join type: **Left**
   - Left side field: **Facility Code**
   - Right side field: **Facility Code**
4. Close the dialog.

**Left join, not inner — and this matters.** A left join keeps every indicator row even
if the facility somehow has no matching row in the facilities file. An inner join would
silently delete those rows, and your totals would quietly be wrong with nothing on screen
to tell you. In health data, always notice what a join throws away.

After joining you will see duplicated columns (`Facility Name`, `Facility Name (facilities.csv)`).
Right-click the duplicates from the second file and choose **Hide**. Keep the ones you need:
`Catchment Population`, `Facility Type`, `Performance Profile`.

### Step 1.3 Fix the field types

Still on the Data Source page, check the little icon above each column name:

| Field | Should be | Fix if wrong |
|---|---|---|
| Report Month | **Date** | Click the icon → Date. If Tableau struggles, see below. |
| Indicator Value | **Number (decimal)**, `#` | Click the icon → Number (decimal) |
| Target Value | Number (decimal) | same |
| Numerator, Denominator | Number (decimal) | same |
| Facility Code, Facility Name, LGA, Indicator Code | **Text**, `Abc` | usually correct |
| Met Target | Text is fine | we will handle this with our own calculation |

**If `Report Month` will not convert to a date:** our months are stored as `2024-07`,
with no day. Some Tableau versions choke on that. Fix it with a calculated field:

1. Right-click in the data pane → **Create Calculated Field**
2. Name it: `Month Date`
3. Formula:
   ```
   DATE(DATEPARSE("yyyy-MM", [Report Month]))
   ```
4. If `DATEPARSE` is unavailable on your connection, use this instead:
   ```
   DATE(MAKEDATE(INT(LEFT([Report Month], 4)),
                 INT(RIGHT([Report Month], 2)),
                 1))
   ```

Use `Month Date` everywhere this guide says "Report Month" on an axis.

### Step 1.4 Go to your first sheet

Bottom-left, click **Sheet 1**. You are now in the worksheet. Take one minute to learn
the four things you will use constantly:

- **Data pane** (left) — your fields. Blue = dimension (a category, something you slice
  *by*). Green = measure (a number, something Tableau *adds up*).
- **Rows and Columns shelves** (top) — what goes on each axis.
- **Marks card** (left, under Filters) — colour, size, label, tooltip, and the chart type.
- **Filters shelf** — what to leave out.

That blue/green distinction explains 90% of "why does my chart look wrong". If a number
you expected as an average is showing as a giant total, Tableau is treating it as a
measure and summing it. Which brings us to the single most important step in this guide.

---

## Part 2 — The calculated fields (35 minutes)

Build these six before you build any chart. Every sheet uses them.

Create each one with: right-click in the data pane → **Create Calculated Field**.

### 2.1 `Indicator %` — read this one carefully

```
AVG([Indicator Value])
```

That is the whole formula. It looks trivial. It is the most important line in the project.

**Why:** if you drag `Indicator Value` onto a shelf, Tableau's default is `SUM`. Sum the
ANC4 percentages of 12 facilities across 24 months and you get something like 16,934% —
nonsense, but Tableau will draw it in a confident-looking bar and you will not
necessarily notice. **You can never add percentages together.** Using an explicit average
field stops that mistake from ever happening.

> **The deeper version of this, which is worth knowing for interviews.** Averaging
> percentages is itself imperfect. `AVG` of facility percentages treats a 24,000-person
> general hospital and a 2,400-person PHC as equally important. The statistically correct
> programme figure is `SUM(numerator) / SUM(denominator) * 100` — add the people first,
> divide once at the end. That is what `v_programme_monthly` does in the SQL, and it is
> why that view exists.
>
> For a *facility-level* chart, `AVG` is right — each facility is its own unit. For a
> *programme total*, the weighted version is right. If someone asks you about this in an
> interview, that answer is the difference between "knows Tableau" and "knows measurement".
>
> To have it available, create `Weighted Indicator %` now too:
> ```
> SUM([Numerator]) / SUM([Denominator]) * 100
> ```

### 2.2 `Target` — the target as a number

```
AVG([Target Value])
```

Same reason. The target is 70, always — not 70 × 288 rows.

### 2.3 `Met Target?` — direction-aware

```
IF [Direction] = "higher_is_better" THEN
    IIF(AVG([Indicator Value]) >= AVG([Target Value]), "Met", "Missed")
ELSE
    IIF(AVG([Indicator Value]) <= AVG([Target Value]), "Met", "Missed")
END
```

**Why the IF on direction:** four of our indicators are good when high. Dropout is good
when **low**. A single `>= target` rule would paint a facility with 35% dropout green,
which is the exact opposite of the truth. Handling direction in a data column instead of
hard-coding it per chart means adding a sixth indicator later needs no chart edits.

### 2.4 `Performance Band` — the green / amber / red colour

```
IF [Direction] = "higher_is_better" THEN
    IF   AVG([Indicator Value]) >= AVG([Target Value])      THEN "1 - On target"
    ELSEIF AVG([Indicator Value]) >= AVG([Target Value]) - 10 THEN "2 - Close (within 10 points)"
    ELSE "3 - Far below target"
    END
ELSE
    IF   AVG([Indicator Value]) <= AVG([Target Value])      THEN "1 - On target"
    ELSEIF AVG([Indicator Value]) <= AVG([Target Value]) + 10 THEN "2 - Close (within 10 points)"
    ELSE "3 - Far below target"
    END
END
```

The `1 -`, `2 -`, `3 -` prefixes force Tableau to sort the legend in a sensible order
instead of alphabetically. Small trick, saves an argument every time.

**Amber exists for a reason.** "Missed the target" lumps a facility at 68% together with
one at 31%. The first needs a phone call; the second needs a visit and a plan. If your
dashboard cannot tell those apart, it is not helping anybody prioritise.

### 2.5 `Gap vs Target` — how far off, in points

```
IF [Direction] = "higher_is_better" THEN
    AVG([Indicator Value]) - AVG([Target Value])
ELSE
    AVG([Target Value]) - AVG([Indicator Value])
END
```

Written so that **negative always means bad**, for every indicator including dropout.
Otherwise you have to remember which way round each chart reads, and in a meeting you
will get it wrong.

### 2.6 `Targets Met (of 5)` — the ranking number

```
SUM(
  IF [Direction] = "higher_is_better" THEN
      IIF([Indicator Value] >= [Target Value], 1, 0)
  ELSE
      IIF([Indicator Value] <= [Target Value], 1, 0)
  END
) / COUNTD([Report Month])
```

This counts, per facility, how many of the five targets it is meeting on average. Divide
by the number of months so the answer stays on a 0–5 scale instead of growing with the
time period. This is the field you rank the league table by.

---

## Part 3 — Sheet 1: Programme Overview (35 minutes)

### 3a. The KPI cards

1. Rename the sheet (double-click the tab at the bottom): **KPI Cards**
2. Drag **Indicator Name** to **Columns**.
3. Drag your **`Indicator %`** field to **Text** on the Marks card.
4. Change the Marks type dropdown from Automatic to **Text**.
5. Filter to the recent period, so the card is current rather than a two-year average:
   - Drag **Report Month** (or `Month Date`) to **Filters**
   - Choose **Relative dates → Months → Last 6**
   - (If your version fights you: use **Range of dates** and pick the last 6 months by hand.)
6. Add the colour: drag **`Performance Band`** to **Colour**. Set the palette by hand —
   click the colour legend → **Edit Colours**:
   - `1 - On target` → green (#2E8B6F)
   - `2 - Close` → amber (#E0A13A)
   - `3 - Far below target` → red (#C8503C)
7. Add context inside the card: drag **`Target`** to **Text** as well. Then click
   **Text → the … button** to edit the layout:
   ```
   <AGG(Indicator %)>
   Target <AGG(Target)>
   ```
8. Format the number: right-click your `Indicator %` field → **Default Properties →
   Number Format → Number (Custom)**, 1 decimal place, suffix `%`.

**A KPI with no target is decoration.** 58.8% means nothing on its own — it could be
excellent or a disaster. 58.8% against a target of 70% is information. Never ship a KPI
card without the target next to it.

### 3b. The trend lines

1. New sheet (the tab with a `+` at the bottom). Name it **Trend vs Target**.
2. Drag **Report Month** to **Columns**. Right-click it → choose **Month** under the
   *second* (green, continuous) group in the menu — you want a continuous axis, not
   discrete year-month headers.
3. Drag **`Indicator %`** to **Rows**.
4. Drag **Indicator Name** to **Rows**, placing it *before* `Indicator %`. You now have
   five small charts stacked — a trellis. Cleaner than five lines fighting in one axis.
5. Add the target line — this is the step people miss:
   - Drag **`Target`** to **Rows**, to the right of `Indicator %`
   - Right-click the new axis → **Dual Axis**
   - Right-click the right-hand axis → **Synchronise Axis** — **do not skip this.**
     Un-synchronised dual axes are the classic way to draw a chart that is confidently
     wrong: the line can appear above the target when the number is below it.
   - Right-click the right axis → untick **Show Header** (you do not need the number twice)
   - On the Marks card, switch to the `Target` pane and change its type to **Line**, make
     it dashed and red.
6. Fix the axis: right-click the left axis → **Edit Axis** → Fixed, 0 to 100.

**Always start a percentage axis at zero.** Tableau's automatic axis will zoom into 55–62%
and turn a flat, unchanging line into a dramatic mountain range. That is how honest
analysts accidentally mislead people.

### 3c. Coverage by LGA

1. New sheet: **Coverage by LGA**
2. **LGA** to Columns, **`Indicator %`** to Rows.
3. **Indicator Code** to Columns, after LGA — grouped bars.
4. Exclude dropout so the chart reads one direction only: drag **Indicator Code** to
   **Filters**, untick `DROP`.
5. Sort descending (the sort button on the toolbar). Add labels: `Indicator %` → **Label**.

**Why remove DROP here:** in a chart where taller bars read as "better", a bar that means
the opposite is a trap. Dropout gets its own chart on Sheet 3, where it can be labelled
properly.

---

## Part 4 — Sheet 2: Facility League Table (30 minutes)

This is a **highlight table** — the workhorse of programme reporting. Learn it once and
you will use it forever.

1. New sheet: **Facility League Table**
2. **Facility Name** → **Rows**
3. **Indicator Code** → **Columns**
4. **`Indicator %`** → **Text**
5. Marks type → **Square**
6. **`Performance Band`** → **Colour** (same green / amber / red as before)
7. Widen the squares: the size slider on the Marks card, drag right until the cells look
   like table cells.
8. Rank the rows properly — do not leave them alphabetical:
   - Drag **`Targets Met (of 5)`** to **Rows**, after Facility Name
   - Right-click it → **Discrete**
   - Then right-click **Facility Name** → **Sort** → By Field → Descending →
     `Targets Met (of 5)`
9. Add **LGA** to Rows, before Facility Name, so facilities group by council.
10. Format the numbers to 1 decimal place.

**Sorting is not cosmetic.** Alphabetical order puts Bwari PHC (your best site) next to
Dobi PHC (one of your worst) and hides the pattern completely. Sorted by performance, the
two-camps story appears on its own, without you having to say a word.

### Add the tooltip that answers the follow-up question

Drag **Numerator**, **Denominator** and **`Gap vs Target`** to **Tooltip**, then click
**Tooltip** to edit the text:

```
<Facility Name> — <Indicator Name>

Value:   <AGG(Indicator %)>   (target <AGG(Target)>)
Gap:     <AGG(Gap vs Target)> points
Based on <SUM(Numerator)> of <SUM(Denominator)> clients
```

When someone in the meeting says "is 40% really bad, or is that two women out of five?",
the tooltip answers it instantly. In small facilities that question is not pedantic — it
is the right question, because a percentage from a denominator of 5 is noise.

---

## Part 5 — Sheet 3: The Dropout Story (30 minutes)

This is the sheet that changes the decision, so build it carefully.

### 5a. The scatter plot — coverage against dropout

1. New sheet: **Coverage vs Dropout**
2. This needs the two indicators side by side on one row, so build two fields:

   `IMM %`:
   ```
   AVG(IF [Indicator Code] = "IMM" THEN [Indicator Value] END)
   ```
   `DROP %`:
   ```
   AVG(IF [Indicator Code] = "DROP" THEN [Indicator Value] END)
   ```

   Both use the same pattern: the `IF` with no `ELSE` returns NULL for other rows, and
   `AVG` ignores NULLs. This is how you pivot inside Tableau without touching the data.

3. **`IMM %`** → **Columns**, **`DROP %`** → **Rows**
4. **Facility Name** → **Detail** (one dot per facility)
5. Marks type → **Circle**. Increase the size.
6. **Facility Name** → **Label**
7. Add the two reference lines — they turn a cloud of dots into four quadrants:
   - Right-click the X axis → **Add Reference Line** → Constant → **80** (immunisation
     target), green
   - Right-click the Y axis → **Add Reference Line** → Constant → **10** (dropout target), red
8. Colour the dots by risk. New field `Dropout Risk`:
   ```
   IF   [DROP %] > 20 THEN "High dropout"
   ELSEIF [DROP %] > 10 THEN "Above target"
   ELSE "Acceptable"
   END
   ```
   Drag it to **Colour**.

**Read the result out loud, because this is your interview answer.** Bottom-right is
where you want to be: high completion, low dropout. Top-left is the emergency: low
completion *and* high dropout. Our 12 facilities sit in two tight clusters with nothing
in between — no gentle gradient, two distinct groups. That is the finding. A programme
average of 74% completion describes neither group.

### 5b. Dropout by facility

1. New sheet: **Dropout by Facility**
2. **Facility Name** → Columns, **`Indicator %`** → Rows
3. **Indicator Code** → Filters → keep only `DROP`
4. Sort descending — worst first. On this chart, worst means highest.
5. Reference line on the Y axis → Constant → **10**, dashed.
6. Colour by `Performance Band` (it already knows dropout is reversed — that is why you
   built direction into the field rather than the chart).
7. **Title it in words, not code:** "Immunisation dropout by facility — lower is better".
   Never make the reader work out which direction is good.

### 5c. The funnel

The funnel uses the raw counts, so connect the second file:

1. **Data → New Data Source** → Text file → `mch_monthly_raw.csv`
2. New sheet: **Care Pathway Funnel**
3. Drag **Measure Names** to **Rows** and **Measure Values** to **Columns**.
4. On the Measure Values card, keep only these five, in this order: `Anc1 Visits`,
   `Anc4 Visits`, `Deliveries Facility`, `Immunisation Started`, `Immunisation Completed`.
   Remove everything else.
5. Marks type → **Bar**, horizontal. Add **Measure Values** to **Label**.
6. Rename the labels to plain language via **aliases** (right-click a row header → Edit
   Alias): "Attended first antenatal visit", "Completed 4 antenatal visits", "Delivered in
   a facility", "Started immunisation", "Completed immunisation".

**Say the drop-off in words on the chart.** "25,000 → 18,000" makes the reader do
arithmetic. "Roughly 1 in 3 women who start antenatal care never finish the four visits"
lands immediately. Put that sentence on the sheet as a text annotation.

---

## Part 6 — Sheet 4: Exceptions & Data Quality (25 minutes)

1. **Data → New Data Source** → `exception_list.csv`
2. New sheet: **Exception Report**
3. **Facility Name** → Rows, **Indicator Code** → Rows, **Months Missed In A Row** → Columns
4. Marks type → **Bar**
5. Sort by `Months Missed In A Row`, descending. Filter to the **Top 10** (drag the field
   to Filters → **Top** tab → By Field → Top 10 by `Months Missed In A Row`).
6. **Priority** → **Colour**: High red, Medium amber, Watch grey.
7. **Worst Value In Streak** and **Streak Start Month** → **Tooltip**.
8. Title it as an instruction, not a description: **"Visit these facilities first"**.

Then a second sheet from `data_quality_log.csv`:

1. New sheet: **Data Quality**
2. **Issue Type** → Rows, **Number of Records** (or `COUNT(*)`) → Columns, bar chart,
   sorted descending.
3. **Facility Name** and **Detail** → Tooltip — so the reader can see the actual offending
   row, not just a count.

**Put the data quality page in the dashboard, not in an appendix.** It is tempting to
hide it. Doing so is the mistake. Showing that you check your own data and publish what
you find is what makes the rest of the dashboard believable. Every experienced M&E
person looks for this page — and its absence tells them something.

---

## Part 7 — Assemble the dashboards (25 minutes)

At the bottom of the window, click the **New Dashboard** icon (next to the new-sheet icon).

### Layout

1. Set the size: right panel → **Size** → Fixed size → **1600 × 900**. Fixed beats
   Automatic — automatic layouts rearrange themselves on someone else's screen and your
   careful design falls apart.
2. Drag sheets from the left panel onto the canvas. Build four dashboards, one per story
   beat:

| Dashboard | Sheets on it |
|---|---|
| **1. Programme Overview** | KPI Cards (across the top), Trend vs Target, Coverage by LGA |
| **2. Facility League Table** | Facility League Table + a gap bar chart beside it |
| **3. Where Are We Losing People?** | Care Pathway Funnel, Coverage vs Dropout, Dropout by Facility |
| **4. Exceptions & Data Quality** | Exception Report, Data Quality |

3. Drag a **Text** object from the bottom-left onto each dashboard for the title. Write
   the title as a **finding**, not a label:
   - Weak: "MCH Indicators Dashboard"
   - Strong: "5 of 12 facilities are missing every target, every month"
4. Add another **Text** object at the bottom of every dashboard:
   > *Simulated data, built to demonstrate MERL indicator logic. 12 facilities, FCT
   > Nigeria, Jul 2024 – Jun 2026. No real patient or facility records used.*

   Put this on **every** dashboard, not just the first. People screenshot single pages and
   send them on, and a screenshot without that line is a claim about real facilities.

### Make the filters work across sheets

1. Right-click a sheet on the dashboard → **Filters** → **LGA** (adds a filter control).
2. Click the filter's dropdown arrow → **Apply to Worksheets** → **All Using This Data
   Source**. Now one click filters the whole dashboard instead of one chart.
3. Add **Report Month** as a filter too → show it as a **Range of Dates** slider.
4. Turn on cross-highlighting: click the funnel-shaped **Use as Filter** icon in the
   corner of the league table. Clicking a facility now filters everything else — which
   turns the dashboard from a report into something a manager can interrogate live in the
   meeting.

### Publish

1. **File → Save to Tableau Public As…** (sign in / create the free account).
2. Name it: `MCH Programme MERL Dashboard — Aisha Inuwa`
3. Once saved, it opens in your browser. Copy that link — that is what goes in your CV,
   your LinkedIn featured section and your GitHub README.
4. Click the **Download** icon → **Image** to get a PNG of each dashboard for LinkedIn
   posts (LinkedIn does not preview Tableau links well, so post the image and put the link
   in the first comment).

---

## Part 8 — Checks before you show anybody

Walk through this list. It catches the errors that get noticed in interviews.

- [ ] No percentage is being **summed** anywhere. Every one uses AVG or a proper weighted
      calculation. (Check every axis for a suspiciously large number.)
- [ ] Every percentage axis is **fixed 0–100**. No zoomed axes making flat lines look
      dramatic.
- [ ] Every dual axis is **synchronised**.
- [ ] Dropout is coloured so that **low is green**, and its chart title says
      "lower is better".
- [ ] Every KPI shows its **target** beside the value.
- [ ] Every dashboard carries the **simulated data** note.
- [ ] Chart titles state **findings**, not field names.
- [ ] Tooltips show the **numerator and denominator**, so "40%" can be checked against
      "2 out of 5".
- [ ] Facilities are sorted by **performance**, never alphabetically.
- [ ] You can explain the **exception rule** — 2+ consecutive months, and why consecutive
      rather than average — without reading it off the screen.

---

## Part 9 — Be ready for these questions

Anybody who knows this field will ask some version of these. Have your answers ready.

**"Why did you average the percentages?"**
For facility comparisons I did, because each facility is its own unit of analysis. For the
programme-wide figure the correct method is sum of numerators over sum of denominators —
add the people first, divide once — because averaging percentages lets a 2,400-person PHC
carry the same weight as a 24,000-person hospital. Both versions are in the SQL;
`v_programme_monthly` is the weighted one.

**"Where do the denominators come from?"**
Standard population estimates applied to catchment: pregnancies about 4.0% of population
per year, deliveries 3.9%, surviving infants 3.6%, divided by 12. And I would flag the
risk honestly — on a real programme, catchment figures are often years out of date, and a
stale denominator produces a wrong rate no matter how clean the numerator is. Checking
them would come before any dashboard work.

**"Why 2 consecutive months for an exception?"**
One bad month is usually a stock-out, a flood, or staff on leave — it corrects itself and
a supervision visit is wasted. Two in a row is a system not correcting itself, which is
exactly what supervision can help. An annual average would hide both cases: a facility
that collapsed in the last four months and one with a single bad January can show the
same average.

**"What is the difference between SBA and facility delivery rate?"**
Denominator. SBA divides by deliveries the facility reported — of the women who delivered
here, how many had a skilled attendant? A quality question. FDR divides by expected
deliveries in the catchment — of all women due to deliver in this area, how many reached a
facility? An access question. A facility can score 95% SBA and 40% FDR, meaning it treats
well but most women in the ward never arrive.

**"What would you do differently with real data?"**
Start with the denominators and the data quality log, not the dashboard. Then check
whether the pattern I am seeing is a service pattern or a reporting pattern — in routine
health data a dip is more often a missing form than a real fall. And I would not present a
league table to facility staff without context on their ward: a hard-to-reach site in rainy
season is not doing the same job as an urban PHC.

**"So what would you actually recommend?"**
Stop spreading supervision evenly across 12 facilities. Concentrate the next quarter on
the five sites failing everything, and start with immunisation dropout specifically —
because those families already came in once, so the trust problem is solved and what is
left is operational: defaulter tracing, return dates written on cards, vaccine stock on
immunisation day. It is the cheapest available win, and coverage will follow.

---

*If you can build these four sheets and answer the questions in Part 9 in your own words,
you can defend this project in any interview — because at that point you genuinely
understand it.*
