# 🏅 Olympic Insights Using Data Analytics

> **An end-to-end Power BI analytics project analysing 126 years of Olympic Games history —
> covering medal trends, country performance, athlete demographics, sport-level dominance,
> host country analysis, and gender participation patterns.**

---

## 📋 Table of Contents

- [Project Overview](#project-overview)
- [Business Problem](#business-problem)
- [Analytical Objectives](#analytical-objectives)
- [Tools & Technologies](#tools--technologies)
- [Dataset Overview](#dataset-overview)
- [Data Cleaning & Transformation](#data-cleaning--transformation)
- [Data Model — Star Schema](#data-model--star-schema)
- [Key DAX Measures](#key-dax-measures)
- [Dashboard Pages](#dashboard-pages)
- [Drill-Through Analysis](#drill-through-analysis)
- [Business Insights & Recommendations](#business-insights--recommendations)
- [Key Challenges & Learnings](#key-challenges--learnings)
- [Data Limitations](#data-limitations)
- [Project Structure](#project-structure)
- [Author](#author)

---

## Project Overview

This project builds a complete analytical solution on historical Olympic Games data
spanning **1896 to 2022**, using Microsoft Power BI as the primary platform.
The goal is not simply to build a dashboard — it is to answer real business questions
about what drives Olympic success, how efficiently countries convert athlete
participation into medals, whether hosting the Games provides a measurable
advantage, and how the demographic profile of Olympic athletes has changed
over 126 years.

The project covers the full data analytics lifecycle:
raw CSV data → data profiling → Power Query cleaning → star schema modelling →
DAX measure development → interactive dashboards → business insights.

---

## Business Problem

The IOC and national sports federations need data-driven answers to questions
that raw medal tables cannot answer:

- Which countries are genuinely efficient, and which only appear strong because of scale?
- Which sports offer the best medal return relative to athlete investment?
- Does hosting the Olympic Games produce a measurable medal advantage?
- How has the gender balance and age profile of Olympic athletes evolved?
- Which individual country–sport combinations are historically dominant?

---

## Analytical Objectives

| # | Business Question | Dashboard Page |
|---|---|---|
| 1 | Which countries dominate overall and how has this changed over time? | Page 1 |
| 2 | Which countries win the most medals relative to the athletes they send? | Page 2 |
| 3 | Who are the most decorated athletes and what does career longevity look like? | Page 3 |
| 4 | Which sports produce the most medals and which countries dominate each sport? | Page 4 |
| 5 | Does hosting the Olympics correlate with higher medal counts? | Page 5 |
| 6 | How has gender representation and athlete age changed over Olympic history? | Page 6 |

---

## Tools & Technologies

| Tool | Purpose |
|---|---|
| **Power BI Desktop** | Dashboard development, data modelling, DAX measures, slicers, drill-through |
| **Power Query (M)** | Data loading, cleaning, type conversion, custom columns, duplicate removal |
| **DAX** | KPI measures, calculated columns, context transition, RANKX, TOPN, MAXX |
| **Star Schema** | 3 fact tables, 3 dimensions, 7 validated relationships |

---

## Dataset Overview

**Source:** 6 historical Olympic CSV files covering Summer and Winter Games, 1896–2022

| Table (Power BI) | Source File | Rows | Grain |
|---|---|---|---|
| `Dim_Athlete` | Olympic_Athlete_Bio.csv | 155,861 | 1 row per athlete |
| `Fact_Event_Results` | Olympic_Athlete_Event_Results.csv | 315,626 | 1 row per athlete per event per edition |
| `Fact_Medal_Tally` | Olympic_Games_Medal_Tally.csv | 1,807 | 1 row per country per edition |
| `Fact_Results` | Olympic_Results.csv | 7,394 | 1 row per event per edition |
| `Dim_Country` | Olympics_Country.csv | 235 | 1 row per NOC code |
| `Dim_Games` | Olympics_Games.csv | 64 | 1 row per Olympic edition |

> **Critical design rule applied throughout:**
> Country-level medal totals always source from `Fact_Medal_Tally` (pre-aggregated, team-safe).
> Athlete-level and sport-level analysis always sources from `Fact_Event_Results` (individual grain).
> Mixing these two tables for the wrong question was the most common bug identified and corrected during build.

---

## Data Cleaning & Transformation

All cleaning was performed in Power Query. Key steps applied per table:

### Dim_Athlete
- **Trimmed `country` column** — 100% of 155,861 rows had a systemic leading space
- **Split `weight` into `weight_min` / `weight_max`** — 961 rows stored weight as a range (e.g. `"58-60"`); split before type conversion, then handled 22 comma-format values via Replace Errors → null
- **Rebuilt `Born_Year`** using digit extraction with 1850–present range validation — raw `born` field contained mixed formats including full dates, year-only entries, and location-prefixed text
- **Added `Has_Physical_Data` flag** — binary indicator for athletes missing both height and weight

### Fact_Event_Results
- **Removed 1,208 fully duplicate rows** (316,834 → 315,626)
- **Added `Medal_Won`** — explicit "No Medal" label replacing nulls
- **Added `Is_Medalist`** — 0/1 numeric flag

### Dim_Country
- **Resolved duplicate NOC code** — `ROC` appeared twice with two different country name labels

### Dim_Games
- **Added `Games_Status`** — Held / Cancelled / Scheduled-Future
- **Added `Season`** — Summer / Winter / Other

### Age Calculations (Power Query — not DAX)
`Age_At_Edition`, `Age_Band`, and `Age_Band_Sort` were built in Power Query rather
than as DAX calculated columns. A DAX calculated column cannot sort another DAX
calculated column in the same table without raising a circular dependency error.

```m
// Born_Year extraction
let
    Digits    = Text.Select(Text.Trim([born]), {"0".."9"}),
    Candidate = if Text.Length(Digits) >= 4
                then Number.FromText(Text.End(Digits, 4))
                else null,
    Validated = if Candidate <> null
                    and Candidate >= 1850
                    and Candidate <= Date.Year(DateTime.LocalNow())
                then Candidate else null
in Validated
```

---

## Data Model — Star Schema

**Schema type:** Star Schema
**Fact tables:** 3 · **Dimension tables:** 3 · **Relationships:** 7 (6 active, 1 inactive)

| Relationship | Key | Status |
|---|---|---|
| Fact_Event_Results → Dim_Athlete | athlete_id | Active |
| Fact_Event_Results → Dim_Country | country_noc → noc | Active |
| Fact_Event_Results → Dim_Games | edition_id | Active |
| Fact_Event_Results → Fact_Results | result_id | **Inactive** |
| Fact_Medal_Tally → Dim_Country | country → country | Active |
| Fact_Medal_Tally → Dim_Games | edition_id | Active |
| Fact_Results → Dim_Games | edition_id | Active |

### Key Modelling Decisions

**edition_id used as relationship key (not edition text)**
Two editions had different text spellings across tables, silently orphaning 342 rows
and injecting a phantom blank row into `Dim_Games`. Switching to `edition_id`
(numeric, 100% clean across all tables) resolved both issues.

**Dim_Athlete → Dim_Country relationship not created**
Autodetected by Power BI but incorrect. 2,107 athlete bio rows contain multiple
countries concatenated into one text field. Also creates an ambiguous filter path
alongside `Fact_Event_Results → Dim_Country`. Removed.

**Fact → Fact relationship kept Inactive**
`Fact_Event_Results` and `Fact_Results` share `result_id`. An active relationship
closes a triangle through `Dim_Games` which Power BI cannot resolve.

**No daily Calendar table**
Olympic Games occur irregularly every ~4 years with 5 cancelled editions.
`Dim_Games` serves as the time dimension, extended with `Edition_Sequence`
(a `RANKX`-based ordinal column) for edition-over-edition comparisons.

**Auto date/time disabled**
Power BI's hidden `LocalDateTable` objects closed a cyclic reference that blocked
all refreshes. Disabled under File → Options → Current File → Data Load.

---

## Key DAX Measures

> **Note:** DAX measures reflect the final validated version used in the dashboard.
> Some measures were revised during build after identifying incorrect results through
> visual cross-checks — see [Key Challenges & Learnings](#key-challenges--learnings).

### Global KPIs

```dax
Total Medals Won = SUM ( Fact_Medal_Tally[total] )

Total Countries Participated =
CALCULATE (
    DISTINCTCOUNT ( Fact_Event_Results[country_noc] ),
    Fact_Event_Results[country_noc] <> "IFR"
)

Total Editions Held =
CALCULATE (
    DISTINCTCOUNT ( Dim_Games[edition_id] ),
    Dim_Games[Games_Status] = "Held"
)
```

### Country Performance

```dax
Medal Efficiency =
DIVIDE ( [Total Medals Won] * 100, [Total Athletes Represented], 0 )

Best Performing Sport =
VAR SportMedals =
    ADDCOLUMNS (
        VALUES ( Fact_Event_Results[sport] ),
        "MedalCount", CALCULATE (
            COUNTROWS ( Fact_Event_Results ),
            Fact_Event_Results[medal] <> BLANK() )
    )
VAR TopSport =
    TOPN ( 1, SportMedals, [MedalCount], DESC,
        Fact_Event_Results[sport], ASC )
RETURN
    CONCATENATEX ( TopSport, Fact_Event_Results[sport], ", " )
```

### Athlete Analysis

```dax
Most Olympic Appearances =
MAXX (
    VALUES ( Fact_Event_Results[athlete_id] ),
    CALCULATE ( DISTINCTCOUNT ( Fact_Event_Results[edition_id] ) )
)
```

> Inner `CALCULATE` is essential for context transition. Without it,
> `DISTINCTCOUNT` returns the global edition count for every athlete.

```dax
Avg. Age at Medal Win =
CALCULATE (
    AVERAGE ( Fact_Event_Results[Age_At_Edition] ),
    Fact_Event_Results[medal] <> BLANK()
)
```

### Sport & Event Analysis

```dax
Edition_Sequence =
RANKX (
    FILTER ( ALL ( Dim_Games ), NOT ISBLANK ( Dim_Games[year] ) ),
    Dim_Games[year], , ASC, DENSE
)

Events Added vs Previous Edition =
VAR CurrentSeq =
    CALCULATE ( MAX ( Dim_Games[Edition_Sequence] ),
        Dim_Games[Games_Status] = "Held" )
VAR CurrentSeason =
    CALCULATE ( SELECTEDVALUE ( Dim_Games[Season] ),
        FILTER ( ALL(Dim_Games),
            Dim_Games[Edition_Sequence] = CurrentSeq ) )
VAR PriorSeq =
    CALCULATE ( MAX ( Dim_Games[Edition_Sequence] ),
        FILTER ( ALL(Dim_Games),
            Dim_Games[Edition_Sequence] < CurrentSeq
            && Dim_Games[Season] = CurrentSeason
            && Dim_Games[Games_Status] = "Held" ) )
RETURN [Events in Latest Edition] - PriorEvents
```

### Host Country Analysis

```dax
Host Advantage % =
IF (
    ISBLANK ( SELECTEDVALUE ( Dim_Country[noc] ) ),
    BLANK (),
    DIVIDE (
        [Medals Won As Host] - [Avg. Medals Won As Non-Host],
        [Avg. Medals Won As Non-Host], BLANK() )
)

Is_Host_Country =
IF (
    CALCULATE ( COUNTROWS ( Dim_Games ),
        FILTER ( ALL ( Dim_Games ),
            Dim_Games[country_noc] = Dim_Country[noc]
            && Dim_Games[Games_Status] = "Held" ) ) > 0,
    "Yes", "No"
)
```

### Gender & Demographics

```dax
Male Athletes =
CALCULATE (
    DISTINCTCOUNT ( Fact_Event_Results[athlete_id] ),
    Dim_Athlete[sex] = "Male"
)

Female Athletes =
CALCULATE (
    DISTINCTCOUNT ( Fact_Event_Results[athlete_id] ),
    Dim_Athlete[sex] = "Female"
)
```

---

## Dashboard Pages

---

### Page 1 — Global Medal Overview

![Page 1 — Global Medal Overview](images/Page1_Global_Medal_Overview.png)

**Purpose:** Executive-level snapshot of the entire Olympic movement — scale, growth,
and historical nation-level dominance.

| KPI | Value |
|---|---|
| Total Medals Won | 20,412 |
| Countries Participated | 230 |
| Total Editions Held | 53 |
| Total Events | 963 |

**Visuals:** Top 10 Countries by Total Medals · Medal Trends Over Time (Summer vs Winter)
· Events Held by Year

**Slicers:** Season · Games_Status (default: Held) · Year Range 1896–2022

**Key Findings:**
- The top 10 nations account for a disproportionate share of all 20,412 medals
  awarded — medal concentration is structural and has persisted across 126 years.
- Summer Games have grown from 43 events in 1896 to 298 in 1992, a 6.9× expansion.
  Summer and Winter Games follow two distinctly different trend lines.

---

### Page 2 — Country Performance Insights

> This page is interactive. Selecting any country from the slicer updates all KPIs
> and charts in real time. Two example views are shown below.

**Page 2 — United States**

![Page 2 — Country Insights USA](images/Page2_Country_Insights_USA.png)

**Page 2 — Soviet Union**

![Page 2 — Country Insights Soviet Union](images/Page2_Country_Insights_Soviet_Union.png)

**Purpose:** Single-country deep dive — medal efficiency, sport strengths,
and year-over-year performance. Reached via drill-through from Page 1 or direct
slicer selection.

**KPIs:** Athletes Represented · Total Medals Won · Medal Efficiency (per 100 Athletes)
· Best Performing Sport

**Slicers:** `Dim_Country[country]` (single-select) · Season

**Country Comparison:**

| Country | Athletes | Medals | Efficiency | Best Sport | Gold % |
|---|---|---|---|---|---|
| United States | 11,812 | 2,985 | 25.27 | Athletics | 39.63% |
| Norway | 2,486 | 567 | 22.81 | Cross Country Skiing | 30.51% |
| China | 3,245 | 713 | 21.97 | Artistic Gymnastics | 39.97% |
| Sweden | 4,236 | 680 | 16.05 | Ice Hockey | 35.44% |
| Japan | 4,985 | 575 | 11.53 | Artistic Gymnastics | 36.70% |

**Key Findings:**
- Medal efficiency reveals which countries punch above their weight.
  Norway achieves nearly double Japan's efficiency (22.81 vs 11.53) despite
  fielding roughly half the athletes.
- China's gold proportion (39.97%) is the highest of all countries shown —
  a programme structured around winning, concentrated in Artistic Gymnastics,
  Diving, and Table Tennis.

---

### Page 3 — Athlete Spotlight

![Page 3 — Athlete Spotlight](images/Page3_Athlete_Spotlight.png)

**Purpose:** Individual performance — top medal winners, career longevity,
age demographics, and gender distribution of medal winners.

| KPI | Value |
|---|---|
| Total Athletes | 1,55,861 |
| % Athletes Who Medalled | 20.26% |
| Most Olympic Appearances | 10 |
| Avg. Age at Medal Win | 26.36 yrs |
| Youngest Medal Winner | 11 yrs |
| Oldest Medal Winner | 73 yrs |

**Slicers:** Season · Sex (Male / Female) · Year Range 1896–2022

**Top 10 Athletes by Medal Count:**

| Athlete | Medals |
|---|---|
| Michael Phelps | 28 |
| Larisa Latynina | 18 |
| Marit Bjørgen | 15 |
| Nikolay Andrianov | 15 |
| Boris Shakhlin, Edoardo Mangiarotti, Ireen Wüst, Jenny Thompson, Ole Einar Bjørndalen, Takashi Ono | 13 each |

**Key Findings:**
- Only 20.26% of 1,55,861 competing athletes ever won a medal — over 79%
  finish without a podium result.
- Sports offering multiple individual medal events per Games (Swimming, Gymnastics)
  structurally provide more per-athlete medal opportunities.
- The 25–30 age band is the largest at medal wins. The oldest winner at 73 came
  from a discipline with no effective upper age restriction.

---

### Page 4 — Sport & Event Analysis

![Page 4 — Sport & Event Analysis](images/Page4_Sport_Event_Analysis.png)

**Purpose:** Programme growth, sport-level medal concentration, country dominance
per sport, and athlete participation by sport.

| KPI | Value |
|---|---|
| Total Sports | 112 |
| Total Events | 963 |
| Events Added vs Previous Edition | +5 |
| Most Dominant Country–Sport Pairing | USA — Athletics |

**Slicers:** Season · Year Range · Sport (multi-select, 112 sports)

**Key Findings:**
- USA–Athletics leads all country–sport pairings with 4,500+ medal appearances.
- High participation in a sport does not guarantee medal returns — Boxing ranks
  in the top 5 for athlete participation but not for medals awarded.
- Sport-wise Medal Trends confirms Athletics and Swimming have grown consistently
  since the early 1900s, while Artistic Gymnastics shows a sharp rise from the 1950s.

---

### Page 5 — Host Country Analysis

> This page is filtered to host nations only (25 of 234 countries have hosted).
> Two views are shown: the default overview and the United States selection,
> which provides the richest host analysis with 8 editions hosted.

**Page 5 — Default Overview (no country selected)**

![Page 5 — Host Country Analysis](images/Page5_Host_Country_Analysis.png)

**Page 5 — United States Selected**

![Page 5 — Host Country Analysis USA](images/Page5_Host_Country_Analysis_USA.png)

**Purpose:** Quantify whether hosting the Olympic Games correlates with a
medal advantage for the host nation.

**KPIs:** Total Olympics Hosted · Medals Won As Host · Avg. Medals as Non-Host
· Host Advantage %

**Slicers:** `Dim_Country[country]` filtered to `Is_Host_Country = "Yes"` (25 nations) · Season

**Key Design Note:** No modelled relationship exists between `Dim_Country` and
`Dim_Games` — adding one would create an ambiguous filter path. All host-country
filtering is handled explicitly in DAX using `FILTER(ALL(Dim_Games))`.

**Key Findings:**
- USA has hosted 8 editions — the highest of any country.
- Host advantage is measurable for multi-time hosts but statistically limited
  for single-host nations (one data point vs. a historical average).
- When no country is selected, KPIs return blank — an `ISBLANK` guard
  prevents a misleading −100% from displaying as the default state.

---

### Page 6 — Gender & Demographics

![Page 6 — Gender & Demographics](images/Page6_Gender_Demographics.png)

**Purpose:** Track gender participation balance and athlete age profile
across 126 years of Olympic history.

| KPI | Value |
|---|---|
| Female Athletes % | 25.89% |
| Male Athletes % | 74.11% |
| Youngest Medal Winner | 11 yrs |
| Oldest Medal Winner | 73 yrs |
| Avg. Age of Participants | 25.96 yrs |

**Slicers:** Season · Year Range 1896–2022

**Key Findings:**
- Female athletes represented 0% in 1896 and have grown to 25.89% across
  all-time records — a sustained increase visible across the full timeline chart.
- Age demographics are sport-dependent, not universal — the oldest winner at 73
  came from a discipline with no effective upper age restriction.

---

## Drill-Through Analysis

### Country Dominance Detail

![Drill-Through — Country Dominance Details](images/DrillThrough_Country_Dominance_Details.png)

**Triggered from:** Page 4 — Sport & Event Analysis (right-click any sport)

**Drill field:** `Fact_Event_Results[sport]`

**Dynamic page title:** `"Country Dominance — " & SELECTEDVALUE(Fact_Event_Results[sport])`

**Visuals:** Bar chart (Top 10 countries in selected sport) · Detail table
(Country · Sport Gold · Sport Silver · Sport Bronze · Medal Appearances)

**Why sport-specific DAX measures are used here:**

```dax
Sport Gold   = CALCULATE ( COUNTROWS ( Fact_Event_Results ),
                 Fact_Event_Results[medal] = "Gold" )
Sport Silver = CALCULATE ( COUNTROWS ( Fact_Event_Results ),
                 Fact_Event_Results[medal] = "Silver" )
Sport Bronze = CALCULATE ( COUNTROWS ( Fact_Event_Results ),
                 Fact_Event_Results[medal] = "Bronze" )
```

`Fact_Medal_Tally` has no sport column — using it here would return all-sport
totals for each country, making the breakdown misleading. These measures source
from `Fact_Event_Results`, which carries the sport filter applied by the
drill-through context.

---

### Country Insights Drill-Through

**Triggered from:** Page 1 — Top 10 Countries by Total Medals (right-click any country)

**Drill field:** `Dim_Country[country]` · **Keep all filters:** ON

Delivers the full Page 2 Country Insights view pre-filtered to the right-clicked country —
athletes, medals, efficiency, sport breakdown, and year trend — in one click.
No manual navigation or slicer re-selection required.

---

## Business Insights & Recommendations

### Key Insights

1. **Medal concentration is structural.** The top 10 nations hold a disproportionate
   share of all-time medals — this pattern has persisted across 126 years.
2. **Raw counts mislead.** Norway (22.81 efficiency) outperforms Japan (11.53) on medals
   per athlete despite fielding roughly half the athletes.
3. **Sport specialisation drives efficiency.** All top-performing countries show
   concentration in 2–5 disciplines, not broad multi-sport coverage.
4. **USA–Athletics is the most dominant country–sport pairing in Olympic history**
   with 4,500+ medal appearances.
5. **Host advantage exists for multi-time hosts — but is not universal.**
   One hosting occasion is one data point, not a proven effect.
6. **Female participation has grown from 0% to 25.89% over 126 years.**
   Era-based trend analysis is more actionable than the all-time aggregate.

### Recommendations

| # | Recommendation | Basis |
|---|---|---|
| 1 | Use Medal Efficiency as the standard cross-country comparison metric | Raw totals structurally favour large programmes |
| 2 | Treat host advantage as per-country evidence, not a universal rule | Reliable only for multi-time hosts |
| 3 | Concentrate investment in 2–5 high-opportunity sports | All top performers show sport-level concentration |
| 4 | Use separate metrics for participation vs medal production | High participation ≠ medal returns |
| 5 | Apply era-based gender analysis for IOC policy decisions | All-time ratio is skewed by early male-only decades |
| 6 | State the 2022 data boundary explicitly in stakeholder presentations | No data exists for Paris 2024 or later |

---

## Key Challenges & Learnings

| Challenge | Root Cause | Solution |
|---|---|---|
| Team-sport medal double-counting | `Fact_Event_Results` has 1 row per athlete per team medal | Country totals exclusively from `Fact_Medal_Tally` |
| Cyclic reference blocked all refreshes | Auto date/time built hidden `LocalDateTable` objects | Disabled Auto date/time in Options |
| `Most Olympic Appearances` returned 24 for every athlete | Missing inner `CALCULATE` — no context transition in `MAXX` | Added `CALCULATE(DISTINCTCOUNT(edition_id))` inside `MAXX` |
| `Edition_Sequence` started from 2, not 1 | Edition text mismatch orphaned 342 rows, injecting a blank row into `Dim_Games` | Switched all `Dim_Games` relationships to `edition_id` |
| Age values above 1,000 on the histogram | Non-standard birth date text formats produced malformed 3-digit years | Rebuilt `Born_Year` in Power Query; added 10–75 bounds guard |
| Events Added returned −196 | Cross-season comparison (2022 Winter vs 2020 Summer) | Added season-matching before selecting the prior edition |
| Country Dominance drill-through showed all-sport totals | `Fact_Medal_Tally` has no sport column | Rebuilt using `Fact_Event_Results`-sourced measures |

> **Core learning:** A measure can be syntactically correct and still return wrong results.
> Knowing which table answers which question — and why — is the real skill in a
> multi-fact-table Power BI model. Every major bug traced back to a grain mismatch
> or a missing context transition, not a syntax error.

---

## Data Limitations

- **Data ends at 2022** — no data for Paris 2024 or later
- **Age calculations are approximate** — `Born_Year` extracted from text; can be ± 1 year
- **Host advantage is correlation, not causation** — investment cycles and programme changes are not controlled
- **`pos` (finishing position)** is too inconsistently formatted for numeric analysis — excluded
- **IFR-coded athletes** (non-national representatives) excluded from country-level KPIs
- **Historical NOCs** (e.g. `URS`) treated as distinct from successor states — consistent with IOC records

---

## Project Structure

```
Olympic-Insights-Power-BI/
│
├── README.md
├── Olympic_Insights_Dashboard.pbix
├── Olympic_Insights_Project_Documentation.pdf
│
├── data/
│   ├── Olympic_Athlete_Bio.csv
│   ├── Olympic_Athlete_Event_Results.csv
│   ├── Olympic_Games_Medal_Tally.csv
│   ├── Olympic_Results.csv
│   ├── Olympics_Country.csv
│   └── Olympics_Games.csv
│
└── images/
    ├── Page1_Global_Medal_Overview.png
    ├── Page2_Country_Insights_USA.png
    ├── Page2_Country_Insights_Soviet_Union.png
    ├── Page3_Athlete_Spotlight.png
    ├── Page4_Sport_Event_Analysis.png
    ├── Page5_Host_Country_Analysis.png
    ├── Page5_Host_Country_Analysis_USA.png
    ├── Page6_Gender_Demographics.png
    └── DrillThrough_Country_Dominance_Details.png
```

---

## Author

**Kavita Kanwar Naruka**
Business Analytics | Power BI | DAX | Data Modelling | Power Query
[LinkedIn] (https://www.linkedin.com/in/kavita-kanwar1190/)

---

*Dataset: Historical Olympic Games records, 1896–2022*
*Platform: Microsoft Power BI Desktop*
*Project type: End-to-end Business Analytics portfolio project*
