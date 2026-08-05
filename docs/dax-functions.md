# DAX Functions Used in This Solution

Inventory Analytics draws on a subset of a shared DAX time-intelligence function
library — the same reusable engine used across the companion Power BI projects, so a
"weekly average" or "year-to-date" here behaves identically to one anywhere else in
that family. Only the functions this solution actually calls are documented below.

```dax
/// Year-to-date: evaluates the measure for the current calendar year, dates up to
/// and including today. KEEPFILTERS preserves any existing date filters.
function YearToDate =
    (Measure: anyref expr) =>
        CALCULATE(
            Measure,
            KEEPFILTERS(Calendar[Year] = YEAR(TODAY())),
            KEEPFILTERS(Calendar[Date] <= TODAY())
        )

/// Rest-of-year from today: evaluates the measure for the current calendar year,
/// dates from today onward. The forward-looking counterpart of YearToDate — the
/// basis for every "projected remaining sales" calculation in this solution.
function FromToday =
    (Measure: anyref expr) =>
        CALCULATE(
            Measure,
            KEEPFILTERS(Calendar[Year] = YEAR(TODAY())),
            KEEPFILTERS(Calendar[Date] >= TODAY())
        )

/// Rolling 12 months: evaluates the measure for dates from 12 months before today
/// onward, respecting the current filter context.
function Trailing12Months =
    (Measure: anyref expr) =>
        VAR Last12Months = EDATE(TODAY(), -12)
        RETURN
            CALCULATE(Measure, KEEPFILTERS(Calendar[Date] >= Last12Months))

/// Weekly average: averages the measure across all weeks up to the current week,
/// removing other Calendar filters. Blank-safe.
function WeeklyAverage =
    (Measure: anyref expr) =>
        IF(
            ISBLANK(Measure),
            BLANK(),
            CALCULATE(
                AVERAGEX(
                    FILTER(ALL(Calendar[Week]), Calendar[Week] <= MAX(Calendar[Week])),
                    Measure
                ),
                REMOVEFILTERS(Calendar)
            )
        )

/// Monthly average: averages the measure across all months up to the current month,
/// removing other Calendar filters. Blank-safe.
function MonthlyAverage =
    (Measure: anyref expr) =>
        IF(
            ISBLANK(Measure),
            BLANK(),
            CALCULATE(
                AVERAGEX(
                    FILTER(ALL(Calendar[Month]), Calendar[Month] <= MAX(Calendar[Month])),
                    Measure
                ),
                REMOVEFILTERS(Calendar)
            )
        )

/// Running daily average: averages the measure across all days up to the latest date
/// in context. Returns blank if the measure is blank — the basis for this
/// solution's daily sales run-rate.
function DailyAverage =
    (Measure: anyref expr) =>
        VAR LatestDate = MAX(Calendar[Date])
        RETURN
            IF(
                ISBLANK(Measure),
                BLANK(),
                CALCULATE(
                    AVERAGEX(FILTER(ALL(Calendar[Date]), Calendar[Date] <= LatestDate), Measure)
                )
            )

/// Constant daily average to today: averages the measure across all days up to
/// TODAY(), independent of the date selected in context (the constant counterpart
/// of DailyAverage).
function DailyAverageConstant =
    (Measure: anyref expr) =>
        VAR CurrentDate = TODAY()
        RETURN
            IF(
                ISBLANK(Measure),
                BLANK(),
                CALCULATE(
                    AVERAGEX(FILTER(ALL(Calendar[Date]), Calendar[Date] <= CurrentDate), Measure)
                )
            )

/// Percentage difference (CurrentValue - PriorValue) / PriorValue for normal
/// positive-base measures. Guards: blank when both blank or current-blank on a
/// future date; returns 1 when current exists but prior was blank on a past date, or
/// when prior is negative.
function GrowthIndex =
    (CurrentValue: decimal val, PriorValue: decimal val) =>
        VAR _IsCurrentBlank = ISBLANK(CurrentValue)
        VAR _IsPriorBlank   = ISBLANK(PriorValue)
        VAR _IsFutureDate   = MAX(Calendar[Date]) > TODAY()
        VAR _IsPastDate     = MAX(Calendar[Date]) < TODAY()
        RETURN
            SWITCH(
                TRUE(),
                _IsCurrentBlank && _IsPriorBlank, BLANK(),
                _IsCurrentBlank && _IsFutureDate && NOT (_IsPriorBlank), BLANK(),
                NOT (_IsCurrentBlank) && _IsPriorBlank && _IsPastDate, 1,
                NOT (_IsCurrentBlank) && PriorValue < 0, 1,
                DIVIDE(CurrentValue - PriorValue, PriorValue)
            )
```

These 8 are drawn from a larger 23-function library covering the full solution's
time-intelligence needs (see the [Sales Analytics](https://github.com/fuadgashi/Sales-Analytics)
repository for the complete set, including the period-extreme and negative-base
comparison functions not needed here).
