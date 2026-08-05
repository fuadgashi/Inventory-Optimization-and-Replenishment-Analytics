# DAX Measure Catalog

Every measure below composes the [functions](dax-functions.md) shared across the
solution. Formula logic is reproduced exactly as authored; only identifiers are
translated to English.

## 1. Stock position

Raw on-hand quantity and value, plus the same position expressed in the units a
warehouse actually plans around — cases and pallets, not just base units:

```dax
Stock Quantity =
    VAR TotalStock = SUM('Price & Stock List'[OnHandQuantity])
    RETURN
        IF(ISBLANK(TotalStock) || TotalStock <= 0, BLANK(), TotalStock)

Stock Value (Net) =
    VAR TotalStock =
        SUMX('Price & Stock List', 'Price & Stock List'[OnHandQuantity] * [Net Selling Price, Trailing 12 Months])
    RETURN
        IF(ISBLANK(TotalStock) || TotalStock <= 0, BLANK(), TotalStock)

Stock Quantity (Cases) =
    VAR TotalStock =
        SUMX('Price & Stock List', DIVIDE('Price & Stock List'[OnHandQuantity], RELATED(Products[UnitsPerCase])))
    RETURN
        IF(TotalStock <= 0 || ISBLANK(TotalStock), BLANK(), TotalStock)

Stock Quantity (Pallets) =
    VAR TotalStock =
        SUMX(
            'Price & Stock List',
            DIVIDE('Price & Stock List'[OnHandQuantity], RELATED(Products[UnitsPerCase]) * RELATED(Products[CasesPerPallet]))
        )
    RETURN
        IF(TotalStock <= 0 || ISBLANK(TotalStock), BLANK(), TotalStock)

Net Selling Price, Trailing 12 Months =
    Trailing12Months([Net Selling Price])

Stock Share of Total =
    DIVIDE(
        [Stock Value (Net)],
        CALCULATE([Stock Value (Net)], REMOVEFILTERS(Products)),
        0
    )
```

## 2. Sales velocity (the input every coverage measure depends on)

Stock coverage is meaningless without a run-rate to compare it to. These measures
turn actual sales into daily, weekly, and monthly velocity figures:

```dax
Sales Quantity, Current Year YTD =
    YearToDate([Sales Quantity])

Weekly Average Sales Quantity, Current Year =
    WeeklyAverage([Sales Quantity, Current Year YTD])

Monthly Average Sales Quantity, Current Year =
    MonthlyAverage([Sales Quantity, Current Year YTD])

Daily Average Sales Quantity, Current Year =
    DailyAverage([Sales Quantity, Current Year YTD])

Daily Average Sales Value, Current Year =
    DailyAverageConstant([Net Sales Value, Current Year YTD])

Sales Quantity, Trailing 12 Months (Dynamic) =
    Trailing12Months([Sales Quantity])
```

## 3. Stock coverage vs. brand targets

Coverage is expressed in weeks of forward sales the current stock would last at
current velocity — then benchmarked against each brand's own min/max target rather
than one blanket rule:

```dax
Weeks of Stock Coverage (by Value) =
    DIVIDE([Stock Value (Net)], [Weekly Average Sales Value, Current Year], BLANK())

Weeks of Stock Coverage (by Quantity) =
    DIVIDE([Stock Quantity], [Weekly Average Sales Quantity, Current Year], BLANK())

Days of Stock Coverage, Rest of Year (by Quantity) =
    DIVIDE(
        [Stock Quantity],
        ([Sales Quantity, Projected Rest of Year] / [Working Days Remaining This Year]),
        BLANK()
    )

-- Sales-weighted minimum coverage target, blended across each brand's own target
Minimum Coverage Target (Weighted) =
    DIVIDE(
        SUMX(Brands, Brands[MinCoverageWeeks] * [Net Sales Value, Current Year YTD]),
        SUMX(Brands, [Net Sales Value, Current Year YTD])
    )

Coverage vs. Minimum Target, Difference =
    [Weeks of Stock Coverage (by Value)] - [Minimum Coverage Target (Weighted)]

Coverage vs. Minimum Target, Index =
    DIVIDE(
        [Weeks of Stock Coverage (by Value)] - [Minimum Coverage Target (Weighted)],
        [Minimum Coverage Target (Weighted)],
        0
    )
```

`Coverage vs. Minimum Target, Index` carries a conditional format string that bands
its result into **"Over ↑" / "✔" / "Out ↓"** directly on the visual — a product
sitting comfortably above its brand's minimum shows green, one running low shows red,
with no separate status column or DAX branch needed.

## 4. Replenishment signal — projected rest-of-year demand

The centerpiece of this solution: rather than a flat run-rate projection, remaining-year
sales quantity is projected from **prior-year rest-of-year sales, scaled by this
year's YoY growth index** — capped at 5× to stop one outlier week from blowing up the
forecast — and falls back to a simple daily-run-rate projection when there isn't
enough growth history to trust:

```dax
Quantity Growth Index, YTD =
    CALCULATE(DIVIDE([Sales Quantity, Current Year YTD], [Sales Quantity, Prior Year YTD], 0))

Sales Quantity, Prior Year, Rest of Year =
    FromToday([Sales Quantity, Prior Year])

Sales Quantity, Projected Rest of Year =
    IF(
        NOT ISBLANK([Quantity Growth Index, YTD]) && [Quantity Growth Index, YTD] > 0,
        IF(
            [Quantity Growth Index, YTD] > 5,
            [Sales Quantity, Prior Year, Rest of Year] * 5,          -- capped: don't extrapolate an outlier
            IF(
                [Quantity Growth Index, YTD] > 0,
                [Sales Quantity, Prior Year, Rest of Year] * [Quantity Growth Index, YTD],
                [Daily Average Sales Quantity, Current Year] * Calendar[WorkingDaysRemaining]
            )
        ),
        IF(
            NOT ISBLANK([Sales Quantity, Prior Year, Rest of Year]) && [Sales Quantity, Prior Year, Rest of Year] > 0,
            [Sales Quantity, Prior Year, Rest of Year],               -- no growth signal yet: fall back to last year's actuals
            [Daily Average Sales Quantity, Current Year] * Calendar[WorkingDaysRemaining]  -- new product: fall back to run-rate
        )
    )

-- The reorder signal itself: how much projected remaining demand exceeds what's on hand
Suggested Order Quantity =
    [Sales Quantity, Projected Rest of Year] - [Stock Quantity]

Quantity YoY Growth Index, YTD =
    GrowthIndex([Sales Quantity, Current Year YTD], [Sales Quantity, Prior Year YTD])
```

A negative `Suggested Order Quantity` means current stock already covers the
projected remaining-year demand; a positive value is the shortfall to plan for.
