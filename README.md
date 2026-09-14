# HSE MONTHLY INPATIENT & OUTPATIENT PERFORMANCE DASHBOARD (2024 - 2026) 

## 1. PROJECT OVERVIEW

This Power BI KPI Dashboard tracks, visualizes, and benchmarks national hospital 
activity across England/HSE healthcare systems using monthly provisional open data 
from January 2024 through July 2026. 

The Data Source: https://digital.nhs.uk/data-and-information/publications/statistical/provisional-monthly-hospital-episode-statistics-for-admitted-patient-care-outpatient-and-accident-and-emergency-data/april-2025---january-2026

The dashboard decouples acute inpatient care (admissions, consultant episodes, 
day-case conversions, emergency ratios) and outpatient clinical utilization 
(appointments, attendances, DNAs, first-to-follow-up patterns). It provides 
surveillance across three analytical layers: macro national trends, demographic 
age cohorts, and clinical treatment specialties.

Reporting Timeframe : January 2024 - December 2026 (Provisional monthly data)
Platform            : Microsoft Power BI Desktop / Service
Architecture        : Multi-Fact Star Schema with dedicated Dimension Tables
Pages               : 6 Dedicated Analytical Dashboards


## 2. DATA SOURCES & INGESTION

The model integrates three primary CSV datasets representing different grains:

1. HES_M04_2627_OPEN_DATA.csv
   - Granularity: National Monthly Aggregate (1 row per month).
   - Metrics    : Finished Consultant Episodes (FCE), Finished Admission 
                  Episodes (FAE), Day Cases, Ordinary Admissions, Emergency, 
                  Outpatient Appointments, Attended, DNA, First & Follow-Up.

2. HES_M04_2627_OPEN_DATA_AGE_GROUPS.csv
   - Granularity: Monthly x 24 Age Bands (0-4 through 90+ and unknown).
   - Metrics    : FCE, FAE, Day Cases, Emergency Admissions, Outpatient 
                  Appointments, Attended, DNAs, First and Follow-up Attendances.

3. HES_M04_2627_OPEN_DATA_TREATMENT_SPECIALTY.csv
   - Granularity: Monthly x ~318 Treatment Specialties (TRETSPEF).
   - Metrics    : Inpatient and Outpatient volume metrics broken down by 
                  medical and surgical service lines.


## 3. ETL, DATA CLEANING & MODEL WIRING

A. Date Normalization (Power Query M):
   - National Open Data: Parsed text format 'YY-Mon' (including single-digit 
     pre-2010 years like '8-Jan') using Text.PadStart to prevent misparsing as '0208':
       Date.FromText("01-" & [Month] & "-20" & Text.PadStart([YY], 2, "0"), "en-US")
   - Specialty & Age Data: Converted 'DDMMMYY' and 'DDMMMYYYY' strings to standardized 
     first-of-month or end-of-month date objects.
   - Filter Applied: Date >= #date(2024, 1, 1) and Date <= #date(2026, 12, 31).

B. Handling Missing Values:
   - In Specialty data, null entries in inpatient-only or outpatient-only specialties 
     were replaced with 0 across all volume and rate columns to maintain additive integrity.

C. Star Schema Relationships:
   - Date [Date]                (1) ---> (*) National_Monthly [Date]
   - Date [Date]                (1) ---> (*) Age_Group[Date]
   - Date [Date]                (1) ---> (*) Treatment_Specialty [Date]
   - AgeGroup [Age_Band]        (1) ---> (*) Age_Group [Age_Band]
   - Specialty [TRETSPEF]       (1) ---> (*) Treatment_Specialty [TRETSPEF]
   * Cross-filter direction: Single (1 -> *) across all relationships.


## 4. COMPLETE DAX MEASURES CATALOG (BY DISPLAY FOLDER)

DISPLAY FOLDER: 01 - Total Inpatient
-------------------------------------
[Total Inpatient FAE]
    = SUM(National_Monthly[APC_Finished_Admission_Episodes])

[Total Inpatient FCE]
    = SUM(National_Monthly[APC_Finished_Consultant])

[Inpatient Day Case Rate]
    = DIVIDE(
        SUM(National_Monthly[APC_Day_Case_Episodes]),
        [Total Inpatient FAE],
        0
    )

[Inpatient Emergency Rate]
    = DIVIDE(
        SUM(National_Monthly[APC_Emergency]),
        [Total Inpatient FAE],
        0
    )

[Inpatient FAE MoM]
    VAR prev_month = 
	CALCULATE(
    	[Total Inpatient FAE],
    	DATEADD('Date'[Date], -1, MONTH)
	)
    RETURN
	DIVIDE([Total Inpatient FAE] - prev_month, prev_month, 0)

DISPLAY FOLDER: 02 - Total Outpatient
--------------------------------------
[Total Appointments]
    = SUM(National_Monthly[Outpatient_Total_Appointments])

[Total Attended]
    = SUM(National_Monthly[Outpatient_Attended_Appointments])

[Total DNA]
    = SUM(National_Monthly[Outpatient_DNA_Appointment])

[Outpatient Attendance Rate]
    = DIVIDE([Total Attended], [Total Appointments], 0)

[Outpatient DNA Rate]
    = DIVIDE([Total DNA], [Total Attended], 0)

[Outpatient Appts MoM]
    VAR prev_month = 
	CALCULATE(
    	[Total Appointments],
    	DATEADD('Date'[Date], -1, MONTH)
	)
    RETURN
	DIVIDE([Total Appointments] - prev_month, prev_month, 0)

DISPLAY FOLDER: 03 - Inpatient by Age Group
--------------------------------------------
[Age Inpatient Admission]
    = SUM(Age_Group[FAE])

[Specialty Emergency Rate]
    = DIVIDE(
        SUM(Age_Group[EMERGENCY]),
        [Age Inpatient Admission],
        0
    )

[Top Age Band Admissions]
    = CALCULATE(
        [Age Inpatient Admission],
        TOPN(
            1,
            ALLSELECTED(AgeGroup[Age_Band]),
            [Age Inpatient Admission],
            DESC
        )
    )

[Top Age Band Label]
    = VAR TopBand = 
          TOPN(
              1,
              ALLSELECTED(AgeGroup[Age_Band]),
              [Age Inpatient Admission],
              DESC
          )
      RETURN
          "Top Age Band (" & CONCATENATEX(TopBand, AgeGroup[Age_Band], ", ") & ")"


DISPLAY FOLDER: 04 - Outpatient by Age Group
---------------------------------------------
[Age Outpatient Appts]
    = SUM(Age_Group[Total_Appointments])

[Age Outpatient DNA]
    = SUM(Age_Group[DNA_Appointments])

[Total Age Attended]
    = SUM(Age_Group[Attended_Appointments])

[Age Outpatient DNA Rate]
    = DIVIDE([Age Outpatient DNA], [Total Age Attended], 0)

[Top Age Band Appts]
    = CALCULATE(
        [Age Outpatient Appts],
        TOPN(
            1,
            ALLSELECTED(AgeGroup[Age_Band]),
            [Age Outpatient Appts],
            DESC
        )
    )

[Top Age Band Label 2]
    = VAR TopBand = 
          TOPN(
              1,
              ALLSELECTED(AgeGroup[Age_Band]),
              [Age Outpatient Appts],
              DESC
          )
      RETURN
          "Top Outpatient Band (" & CONCATENATEX(TopBand, AgeGroup[Age_Band], ", ") & ")"


DISPLAY FOLDER: 05 - Inpatient by Specialty
--------------------------------------------
[Specialty Inpatient Admission]
    = SUM(Treatment_Specialty[FAE])

[Specialty Emergency Rate]
    = DIVIDE(
        SUM(Treatment_Specialty[EMERGENCY]),
        [Specialty Inpatient Admission],
        0
    )


DISPLAY FOLDER: 06 - Outpatient by Specialty
---------------------------------------------
[Specialty Outpatient Appts]
    = SUM(Treatment_Specialty[Total_Appointments])

[Specialty Outpatient Attended]
    = SUM(Treatment_Specialty[Attended_Appointments])

[Specialty Outpatient DNA]
    = SUM(Treatment_Specialty[DNA_Appointments])

[Specialty DNA Rate]
    = DIVIDE(
        [Specialty Outpatient DNA],
        [Specialty Outpatient Attended],
        0
    )


## 5. DETAILED PAGE-BY-PAGE DASHBOARD STRUCTURE

PAGE 1: TOTAL MONTHLY INPATIENT
--------------------------------------------------------------------------------
* Purpose: Macro surveillance of national hospital admissions, clinical episode 
  handover volume, month-over-month expansion, and emergency vs. elective mix.
* Top Slicers: Date[Financial Year], Date[Month Name].

1. Executive KPI Cards Ribbon (Top):
   - Card 1: [Total Inpatient FAE] (Formatted as '0.00M')
   - Card 2: [Total Inpatient FCE] (Episode volume)
   - Card 3: [Inpatient Day Case Rate] (Percentage '0.0%')
   - Card 4: [Inpatient Emergency Rate] (Percentage '0.0%')

2. Main Trend Visual (Line & Clustered Column Chart):
   - X-Axis            : Date[Month-Year] (Sorted by YearMonthOrder)
   - Column Y-Axis     : [Total Inpatient FAE]
   - Line Y-Axis       : [Inpatient FAE MoM] (Formatted as percentage '+0.0%;-0.0%')
   - Insight           : Evaluates volume surges against month-over-month growth speed.

3. Emergency Acuity Trend (Line Chart):
   - X-Axis            : Date[Month-Year]
   - Y-Axis            : [Inpatient Emergency Rate]
   - Insight           : Tracks seasonal shifts toward unscheduled emergency admissions.

4. Bed Setting & Clinical Modality Mix (100% Stacked Area Chart):
   - X-Axis            : Date[Month-Year]
   - Y-Axis Values     : SUM(APC_Ordinary_Episodes), SUM(APC_Day_Case_Episodes), 
                         SUM(APC_Emergency)
   - Insight           : Longitudinal proportion of day cases vs. ordinary overnight 
                         stays vs. emergency intake.


PAGE 2: TOTAL MONTHLY OUTPATIENT
--------------------------------------------------------------------------------
* Purpose: Operational monitoring of outpatient clinic volume, attendance success, 
  capacity waste via DNAs (Did Not Attend), and referral intake dynamics.
* Top Slicers: Date[Financial Year], Date[Month Name].

1. Executive KPI Cards Ribbon (Top):
   - Card 1: [Total Appointments]
   - Card 2: [Total Attended]
   - Card 3: [Outpatient Attendance Rate]
   - Card 4: [Outpatient DNA Rate]

2. Activity Velocity & Throughput (Line & Clustered Column Chart):
   - X-Axis            : Date[Month-Year]
   - Column Y-Axis     : [Total Appointments]
   - Line Y-Axis       : [Outpatient Appts MoM]
   - Insight           : Tracks clinic demand growth against month-to-month delivery limits.

3. Capacity Loss Trend (Line Chart):
   - X-Axis            : Date[Month-Year]
   - Y-Axis            : [Outpatient DNA Rate]
   - Insight           : Visualizes missed appointment rate patterns across the year.

4. Referral & Attendance Profile (100% Stacked Area Chart):
   - X-Axis            : Date[Month-Year]
   - Y-Axis Values     : SUM(First_Attendance), SUM(Follow_Up_Attendance)
   - Insight           : Detects whether clinic capacity is being absorbed by follow-up 
                         backlogs rather than new patient assessments.


PAGE 3: MONTHLY INPATIENT BY AGE GROUP
--------------------------------------------------------------------------------
* Purpose: Demographic distribution of acute admissions and emergency vulnerability.
* Top Slicers: Date[Financial Year], Date[Month Name], AgeGroup[Age_Band].

1. Dynamic Benchmark Card:
   - Single Card Value : [Top Age Band Admissions]
   - Dynamic Title     : [Top Age Band Label] (Displays which cohort holds highest volume)

2. Demographic Volume (Clustered Bar Chart):
   - Y-Axis            : AgeGroup[Age_Band]
   - X-Axis            : [Age Inpatient Admission]
   - Insight           : Rank-orders age cohorts driving acute hospital bed usage.

3. Modality Mix Across Cohorts (100% Stacked Area Chart):
   - X-Axis            : AgeGroup[Age_Band]
   - Y-Axis Values     : SUM(Ordinary_Admission_Episodes), SUM(FCE_DAY_CASES), 
                         SUM(EMERGENCY)
   - Insight           : Shows pediatric/geriatric divergence in emergency dependency.

4. Cohort Performance Matrix (Matrix Visual):
   - Rows              : AgeGroup[Age_Band]
   - Values            : [Age Inpatient Admission], [Specialty Emergency Rate]
   - Formatting        : Data bars on emergency rate.


PAGE 4: MONTHLY OUTPATIENT BY AGE GROUP
--------------------------------------------------------------------------------
* Purpose: Evaluates clinic scheduling demand, missed care, and attendance behavior 
  across generational cohorts.
* Top Slicers: Date[Financial Year], Date[Month Name], AgeGroup[Age_Band].

1. Dynamic Benchmark Card:
   - Single Card Value : [Top Age Band Appts]
   - Dynamic Title     : [Top Age Band Label 2]

2. Total Appointments by Cohort (Clustered Bar Chart):
   - Y-Axis            : AgeGroup[Age_Band]
   - X-Axis            : [Age Outpatient Appts]
   - Insight           : Pinpoints demographic cohorts consuming the most clinic slots.

3. Attendance Type Breakdown (100% Stacked Area Chart):
   - X-Axis            : Dim_AgeGroup[Age_Band]
   - Y-Axis Values     : SUM(First_Attendance), SUM(Follow_Up_Attendance)
   - Insight           : Tracks First vs. Follow-up distribution over time.

4. Demographic Attendance Matrix (Matrix Visual):
   - Rows              : AgeGroup[Age_Band]
   - Values            : [Total Age Attended], [Age Outpatient DNA], [Age Outpatient DNA Rate]
   - Formatting        : Conditional formatting highlighting cohorts with excessive DNA rates.


PAGE 5: MONTHLY INPATIENT BY TREATMENT SPECIALTY
--------------------------------------------------------------------------------
* Purpose: Surgical and medical service-line admission demand, emergency loading, 
  and procedural day-case efficiency.
* Top Slicers: Date[Financial Year], Date[Month Name], Specialty[Specialty].
* Note: Designed without KPI cards to maximize screen canvas for departmental comparison.

1. High-Volume Specialties (Clustered Bar Chart):
   - Y-Axis            : Specialty[Specialty] (Filter: Top 20 by Admission)
   - X-Axis            : [Specialty Inpatient Admission]
   - Insight           : Isolates top 20 services.

2. Clinical Setting Breakdown (100% Stacked Area Chart):
   - X-Axis            : Specialty[Specialty] (Filter: Top 20 by Specialty)
   - Y-Axis Values     : SUM(Ordinary_Admission_Episodes), SUM(FCE_DAY_CASES), 
                         SUM(EMERGENCY)
   - Insight           : Displays structural shifts between day-case and inpatient beds.

3. Specialty Risk & Volume Matrix (Matrix Visual):
   - Rows              : Specialty[Specialty]
   - Values            : [Specialty Inpatient Admission], [Specialty Emergency Rate]
   - Insight           : Enables tabular lookup and ranking of emergency acuity by service line.


PAGE 6: MONTHLY OUTPATIENT BY TREATMENT SPECIALTY
--------------------------------------------------------------------------------
* Purpose: Departmental clinic appointment volume, attendance attrition, and 
  new-to-follow-up scheduling balance.
* Top Slicers: Date[Financial Year], Date[Month Name], Specialty[Specialty].
* Note: Designed without KPI cards to maximize canvas space for detailed service comparison.

1. Top Outpatient Services (Clustered Bar Chart):
   - Y-Axis            : Specialty[Specialty] (Filter: Top 20 by Appointments)
   - X-Axis            : [Specialty Outpatient Appts]
   - Insight           : Identifies departments with the heaviest booking loads (e.g., Ophthalmology).

2. Departmental Referral Dynamics (100% Stacked Area Chart):
   - X-Axis            : Specialty[Specialty] (Filter: Top 20 by Admission)
   - Y-Axis Values     : SUM(First_Attendance), SUM(Follow_Up_Attendance)
   - Insight           : Tracks chronic follow-up loading over time by specialty.

3. Departmental Attendance & DNA Matrix (Matrix Visual):
   - Rows              : Specialty[Specialty]
   - Values            : [Specialty Outpatient Attended], [Specialty Outpatient DNA], 
                         [Specialty DNA Rate]
   - Formatting        : Conditional color gradient on [Specialty DNA Rate] to highlight 
                         clinics with the highest lost appointment capacity.


## 6. UI & REPORT DESIGN BEST PRACTICES

- Canvas Dimensions: 16:9 widescreen standard (1920 x 1080).
- Color Palette: Frontier
- Axis Sorting: Always sort 'Month-Year' chronologically using the hidden column 
  'YearMonthOrder' in Dim_Date. Sort 'Age_Band' using 'Age_Sort_Order'.
- Cross-Filtering: Maintain bidirectional or single-click cross-filtering across visuals 
  so selecting a specialty or age band dynamically filters related charts and matrices.
