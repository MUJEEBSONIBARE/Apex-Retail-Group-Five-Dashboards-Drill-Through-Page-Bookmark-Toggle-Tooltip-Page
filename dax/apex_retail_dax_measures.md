# Apex Retail Group — DAX Catalogue

## Foundation

```DAX
Total Revenue = SUM(Sales[revenue_gbp])

Total Cost = SUM(Sales[cost_gbp])

Gross Profit =
VAR vRevenue = [Total Revenue]
VAR vCost = [Total Cost]
RETURN vRevenue - vCost

Gross Margin % =
VAR vRevenue = [Total Revenue]
VAR vProfit = [Gross Profit]
RETURN IF(vRevenue = 0, BLANK(), DIVIDE(vProfit, vRevenue))

Total Discount = SUM(Sales[discount_gbp])

Discount Rate % =
VAR vRevenue = [Total Revenue]
VAR vDiscount = [Total Discount]
RETURN IF(vRevenue = 0, BLANK(), DIVIDE(vDiscount, vRevenue))

Total Units = SUM(Sales[units_sold])

Total Transactions = SUM(Sales[transactions])

Avg Basket Size =
VAR vRevenue = [Total Revenue]
VAR vTxn = [Total Transactions]
RETURN IF(vTxn = 0, BLANK(), DIVIDE(vRevenue, vTxn))

Revenue per Sq Ft =
VAR vRevenue = [Total Revenue]
VAR vSqFt = SUM(Stores[sq_footage])
RETURN IF(vSqFt = 0, BLANK(), DIVIDE(vRevenue, vSqFt))

Revenue Target =
CALCULATE(
    SUM(Targets[revenue_target_gbp]),
    USERELATIONSHIP(DateTable[Date], Targets[target_month])
)

Variance to Target =
VAR vRevenue = [Total Revenue]
VAR vTarget = [Revenue Target]
RETURN IF(ISBLANK(vTarget), BLANK(), vRevenue - vTarget)

Variance to Target % =
VAR vVariance = [Variance to Target]
VAR vTarget = [Revenue Target]
RETURN IF(ISBLANK(vTarget), BLANK(), DIVIDE(vVariance, vTarget))
```

## Time Intelligence

```DAX
Revenue YTD =
TOTALYTD([Total Revenue], DateTable[Date])

Revenue SPLY =
CALCULATE(
    [Total Revenue],
    SAMEPERIODLASTYEAR(DateTable[Date])
)

Revenue vs SPLY % =
VAR vRevenue = [Total Revenue]
VAR vSPLY = [Revenue SPLY]
RETURN IF(ISBLANK(vSPLY), BLANK(), DIVIDE(vRevenue - vSPLY, vSPLY))

Gross Profit YTD =
TOTALYTD([Gross Profit], DateTable[Date])

Gross Margin % SPLY =
CALCULATE(
    [Gross Margin %],
    SAMEPERIODLASTYEAR(DateTable[Date])
)

Revenue Rolling 3M =
VAR vLastDate = MAX(DateTable[Date])
RETURN
    CALCULATE(
        AVERAGE(Sales[revenue_gbp]),
        DATESINPERIOD(DateTable[Date], vLastDate, -3, MONTH)
    )

Revenue Prior Month =
CALCULATE(
    [Total Revenue],
    DATEADD(DateTable[Date], -1, MONTH)
)

Revenue MoM % Change =
VAR vCurrent = [Total Revenue]
VAR vPrior = [Revenue Prior Month]
RETURN IF(ISBLANK(vPrior), BLANK(), DIVIDE(vCurrent - vPrior, vPrior))

Revenue Cumulative =
VAR vMaxDate = MAX(DateTable[Date])
RETURN
    CALCULATE(
        [Total Revenue],
        DateTable[Date] <= vMaxDate,
        ALL(DateTable)
    )
```

## Ranking

```DAX
Store Revenue Rank (Company) =
VAR vRevenue = [Total Revenue]
RETURN
    IF(
        ISBLANK(vRevenue),
        BLANK(),
        RANKX(ALL(Stores), [Total Revenue], vRevenue, DESC, DENSE)
    )

Store Margin Rank (Company) =
VAR vMargin = [Gross Margin %]
RETURN
    IF(
        ISBLANK(vMargin),
        BLANK(),
        RANKX(ALL(Stores), [Gross Margin %], vMargin, DESC, DENSE)
    )

Store Revenue Rank (Region) =
VAR vRevenue = [Total Revenue]
RETURN
    IF(
        ISBLANK(vRevenue),
        BLANK(),
        RANKX(
            ALLEXCEPT(Stores, Stores[region]),
            [Total Revenue],
            vRevenue,
            DESC,
            DENSE
        )
    )

Store Margin Rank (Region) =
VAR vMargin = [Gross Margin %]
RETURN
    IF(
        ISBLANK(vMargin),
        BLANK(),
        RANKX(
            ALLEXCEPT(Stores, Stores[region]),
            [Gross Margin %],
            vMargin,
            DESC,
            DENSE
        )
    )

Region Avg Revenue =
VAR vRegionTotal =
    CALCULATE([Total Revenue], ALLEXCEPT(Stores, Stores[region]))
VAR vRegionStoreCount =
    CALCULATE(
        DISTINCTCOUNT(Stores[store_id]),
        ALLEXCEPT(Stores, Stores[region])
    )
RETURN IF(vRegionStoreCount = 0, BLANK(), DIVIDE(vRegionTotal, vRegionStoreCount))

vs Region Avg % =
VAR vRevenue = [Total Revenue]
VAR vRegionAvg = [Region Avg Revenue]
RETURN IF(ISBLANK(vRegionAvg), BLANK(), DIVIDE(vRevenue - vRegionAvg, vRegionAvg))

Is Bottom Quartile (Region) =
VAR vRank = [Store Revenue Rank (Region)]
VAR vStoreCount =
    CALCULATE(
        DISTINCTCOUNT(Stores[store_id]),
        ALLEXCEPT(Stores, Stores[region])
    )
VAR vQuartileThreshold = CEILING(vStoreCount * 0.25, 1)
RETURN IF(
    vRank >= vStoreCount - vQuartileThreshold + 1,
    TRUE(),
    FALSE()
)
```

## Store Detail / Margin

```DAX
Store EZ Status =
VAR vEZ = SELECTEDVALUE(Stores[enterprise_zone])
RETURN IF(ISBLANK(vEZ), BLANK(), IF(vEZ, "Yes", "No"))

EZ Tax Relief % =
VAR vRelief = SELECTEDVALUE(Stores[ez_tax_relief_pct])
RETURN IF(ISBLANK(vRelief), BLANK(), vRelief / 100)

Gross Margin Rolling 3M % =
VAR vLastDate = MAX(DateTable[Date])
VAR vRollingRevenue =
    CALCULATE(
        [Total Revenue],
        DATESINPERIOD(DateTable[Date], vLastDate, -3, MONTH)
    )
VAR vRollingProfit =
    CALCULATE(
        [Gross Profit],
        DATESINPERIOD(DateTable[Date], vLastDate, -3, MONTH)
    )
RETURN IF(
    vRollingRevenue = 0,
    BLANK(),
    DIVIDE(vRollingProfit, vRollingRevenue)
)

Category Benchmark Margin % =
DIVIDE(AVERAGE(Categories[gross_margin_pct]), 100)

Adjusted Gross Margin % =
VAR vBaseMargin = [Gross Margin %]
VAR vImprovementPct =
    'Margin Improvement %'[Margin Improvement % Value] / 100
RETURN IF(
    ISBLANK(vBaseMargin),
    BLANK(),
    vBaseMargin + vImprovementPct
)
```

## Enterprise Zone

```DAX
EZ Adjusted Cost =
SUMX(
    Sales,
    VAR vCost = Sales[cost_gbp]
    VAR vStoreID = Sales[store_id]
    VAR vIsEZ = RELATED(Stores[enterprise_zone])
    VAR vReliefPct = RELATED(Stores[ez_tax_relief_pct]) / 100
    RETURN IF(vIsEZ, vCost * (1 - vReliefPct), vCost)
)

EZ Adjusted Gross Profit =
VAR vRevenue = [Total Revenue]
VAR vAdjCost = [EZ Adjusted Cost]
RETURN vRevenue - vAdjCost

EZ Adjusted Gross Margin % =
VAR vRevenue = [Total Revenue]
VAR vAdjProfit = [EZ Adjusted Gross Profit]
RETURN IF(vRevenue = 0, BLANK(), DIVIDE(vAdjProfit, vRevenue))

EZ Relief Impact (ppts) =
VAR vAdjMargin = [EZ Adjusted Gross Margin %]
VAR vBaseMargin = [Gross Margin %]
RETURN IF(ISBLANK(vAdjMargin), BLANK(), vAdjMargin - vBaseMargin)

Store Margin Rank - EZ Adjusted (Region) =
VAR vAdjMargin = [EZ Adjusted Gross Margin %]
RETURN
    IF(
        ISBLANK(vAdjMargin),
        BLANK(),
        RANKX(
            ALLEXCEPT(Stores, Stores[region]),
            [EZ Adjusted Gross Margin %],
            vAdjMargin,
            DESC,
            DENSE
        )
    )

Is Bottom Quartile - EZ Adjusted =
VAR vRank = [Store Margin Rank - EZ Adjusted (Region)]
VAR vStoreCount =
    CALCULATE(
        DISTINCTCOUNT(Stores[store_id]),
        ALLEXCEPT(Stores, Stores[region])
    )
VAR vThreshold = CEILING(vStoreCount * 0.25, 1)
RETURN IF(
    vRank >= vStoreCount - vThreshold + 1,
    TRUE(),
    FALSE()
)

Rank Change - EZ Adjustment =
VAR vOriginalRank = [Store Margin Rank (Region)]
VAR vAdjustedRank = [Store Margin Rank - EZ Adjusted (Region)]
RETURN IF(
    OR(ISBLANK(vOriginalRank), ISBLANK(vAdjustedRank)),
    BLANK(),
    vOriginalRank - vAdjustedRank
)
```

## Dynamic Title

```DAX
Dynamic Chart Title =
VAR vMetricName =
    SELECTEDVALUE(
        'Performance Metric Parameter'[Performance Metric Parameter],
        "Performance"
    )
VAR vRegionName =
    SELECTEDVALUE(Stores[region], "All Regions")
RETURN vRegionName & " - Store Rankings by " & vMetricName
```
