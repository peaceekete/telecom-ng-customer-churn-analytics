DAX Measures – Telecom NG Churn Analytics

All the measures behind this dashboard, grouped by what they're actually for rather than dumped alphabetically. Duplicates from my build process have been removed.

Customer Retention \& Churn

```dax
// Counts the total number of customers who have churned.
Churned Customers = CALCULATE(COUNTROWS('TelecomChurn\_Nigeria\_Clean'), 'TelecomChurn\_Nigeria\_Clean'\[Churn] = "Yes")

// Calculates the percentage of customers who have churned.
Churn Rate = DIVIDE(\[Churned Customers], \[Total Customers])

// Counts customers who remain active.
Retained Customers = CALCULATE(COUNTROWS('TelecomChurn\_Nigeria\_Clean'), 'TelecomChurn\_Nigeria\_Clean'\[Churn] = "No")

Retention Rate = 
DIVIDE(
    \[Retained Customers],
    \[Total Customers]
)

Monthly Revenue = 
SUM('TelecomChurn\_Nigeria\_Clean'\[Monthly\_Bill])

Revenue Per Customer = 
DIVIDE(
    \[Monthly Revenue],
    \[Total Customers]
)

// Calculates the monthly revenue lost from customers who have churned.
Revenue Lost to Churn = 
CALCULATE(
    \[Monthly Revenue],
    'TelecomChurn\_Nigeria\_Clean'\[Churn] = "Yes"
)

// per-customer version of the above, useful for justifying retention spend
Avg Revenue Lost per Churned Customer = 
DIVIDE(
    \[Revenue Lost to Churn],
    CALCULATE(
        COUNTROWS('TelecomChurn\_Nigeria\_Clean'),
        'TelecomChurn\_Nigeria\_Clean'\[Churn] = "Yes"
    ),
    0
)

Average Revenue Per User = \[Monthly Revenue] / \[Total Customers]

Total Outstanding Balance = 
SUM(TelecomChurn\_Nigeria\_Clean\[Outstanding\_Balance])

// customers who are current on their bill vs total billed
Collection Rate = 
VAR TotalBilled = SUM('TelecomChurn\_Nigeria\_Clean'\[Monthly\_Bill])
VAR TotalUnpaid = SUM('TelecomChurn\_Nigeria\_Clean'\[Outstanding\_Balance])
RETURN
    DIVIDE(TotalBilled - TotalUnpaid, TotalBilled, 0)

Collection Efficiency Index = 
DIVIDE(
    SUM('TelecomChurn\_Nigeria\_Clean'\[Monthly\_Bill]) - SUM('TelecomChurn\_Nigeria\_Clean'\[Outstanding\_Balance]),
    SUM('TelecomChurn\_Nigeria\_Clean'\[Monthly\_Bill]),
    0
)

// revenue sitting with people who are active but haven't opened the app in 30 days, basically ghost customers
Revenue from Inactive Customers = 
CALCULATE(
    SUM('TelecomChurn\_Nigeria\_Clean'\[Monthly\_Bill]),
    FILTER(
        'TelecomChurn\_Nigeria\_Clean',
        'TelecomChurn\_Nigeria\_Clean'\[Churn] = "No"
            \&\& 'TelecomChurn\_Nigeria\_Clean'\[Mobile\_App\_Login\_Last30Days] = 0
    )
)
```

Averages / general stats


Avg Tenure = AVERAGE('TelecomChurn\_Nigeria\_Clean'\[Tenure\_Months])
Avg Satisfaction = AVERAGE('TelecomChurn\_Nigeria\_Clean'\[Customer\_Satisfaction])
Avg Customer CSAT = AVERAGE('TelecomChurn\_Nigeria\_Clean'\[Customer\_Satisfaction])
Avg Monthly Data Usage = AVERAGE('TelecomChurn\_Nigeria\_Clean'\[Monthly\_Data\_Usage\_GB])
Avg Monthly Bill = AVERAGE('TelecomChurn\_Nigeria\_Clean'\[Monthly\_Bill])
Avg Monthly App Logins = AVERAGE('TelecomChurn\_Nigeria\_Clean'\[Mobile\_App\_Login\_Last30Days])
Avg CLV = AVERAGE('TelecomChurn\_Nigeria\_Clean'\[Estimated\_CLV])
Average Customer Lifetime Value = AVERAGE(TelecomChurn\_Nigeria\_Clean\[Estimated\_CLV])
Avg Call Minutes = AVERAGE('TelecomChurn\_Nigeria\_Clean'\[Monthly\_Call\_Minutes])
Average Risk Score = AVERAGE(TelecomChurn\_Nigeria\_Clean\[Risk\_Score])

Average Support Tickets per Customer = 
DIVIDE(
    SUM('TelecomChurn\_Nigeria\_Clean'\[Support\_Tickets]),
    COUNTROWS('TelecomChurn\_Nigeria\_Clean'),
    0
)

// data usage divided by call minutes, a rough sense of whether people lean data-heavy or call-heavy
GB consumed per Voice Mins = 
DIVIDE(
    AVERAGE('TelecomChurn\_Nigeria\_Clean'\[Monthly\_Data\_Usage\_GB]),
    AVERAGE('TelecomChurn\_Nigeria\_Clean'\[Monthly\_Call\_Minutes]),
    0
)

// resolution time had some negative values in the raw data (bad timestamps somewhere upstream), filtering those out so the average isn't skewed
Avg Resolution Time = 
CALCULATE(
    AVERAGE('TelecomChurn\_Nigeria\_Clean'\[Average\_Resolution\_Time]),
    'TelecomChurn\_Nigeria\_Clean'\[Average\_Resolution\_Time] >= 0
)

// CLV split by whether they stayed or left, good for seeing what churn is actually costing in lifetime value terms
CLV Retained = CALCULATE(AVERAGE('TelecomChurn\_Nigeria\_Clean'\[Estimated\_CLV]), 'TelecomChurn\_Nigeria\_Clean'\[Churn] = "No")
CLV Churned = CALCULATE(AVERAGE('TelecomChurn\_Nigeria\_Clean'\[Estimated\_CLV]), 'TelecomChurn\_Nigeria\_Clean'\[Churn] = "Yes")

First Contact Resolution Rate = 
DIVIDE(
    CALCULATE(
        COUNT('TelecomChurn\_Nigeria\_Clean'\[Support\_Tickets]),
        'TelecomChurn\_Nigeria\_Clean'\[Support\_Tickets] <= 1
    ),
    COUNT('TelecomChurn\_Nigeria\_Clean'\[Support\_Tickets]),
    0
)
```

Segmenting customers

```dax
// buckets people by how urgent their risk of churning actually is, calibrated to this dataset (median risk score sits around 85, so the usual 70/40 thresholds you'd expect don't work here)
Churn Risk Status = 
VAR \_Risk = SELECTEDVALUE('TelecomChurn\_Nigeria\_Clean'\[Risk\_Score])
RETURN
SWITCH(
    TRUE(),
    \_Risk >= 90, "High Risk of Churn",
    \_Risk >= 60, "Moderate Risk of Churn",
    "Low Risk of Churn"
)

// this is the one driving the whole Priority Contacts page, sorts everyone into who actually needs a call today
Customer Priority Segment = 
SWITCH(
    TRUE(),
    'TelecomChurn\_Nigeria\_Clean'\[Risk\_Score] >= 90 \&\& 'TelecomChurn\_Nigeria\_Clean'\[Monthly\_Bill] >= 15000, "High Value Customer at Risk to Churn",
    'TelecomChurn\_Nigeria\_Clean'\[Outstanding\_Balance] > 0 \&\& 'TelecomChurn\_Nigeria\_Clean'\[Outstanding\_Balance] >= ('TelecomChurn\_Nigeria\_Clean'\[Monthly\_Bill] \* 2.0), "High Outstanding Balance",
    'TelecomChurn\_Nigeria\_Clean'\[Risk\_Score] < 50 \&\& 'TelecomChurn\_Nigeria\_Clean'\[Late\_Payment\_Count] >= 1 \&\& 'TelecomChurn\_Nigeria\_Clean'\[Outstanding\_Balance] > 0, "Payment Reminder Required",
    "No Immediate Action Required"
)

// combines risk with how much money is actually on the line, someone risky but paying next to nothing shouldn't outrank a big spender
Action Priority Score = 
AVERAGE('TelecomChurn\_Nigeria\_Clean'\[Risk\_Score]) \* (AVERAGE('TelecomChurn\_Nigeria\_Clean'\[Monthly\_Bill]) + AVERAGE('TelecomChurn\_Nigeria\_Clean'\[Outstanding\_Balance]))

// plain-language suggestion that shows up on the customer detail card
Recommended Action = 
VAR \_Group =
    SELECTEDVALUE('TelecomChurn\_Nigeria\_Clean'\[Customer Priority Segment])
RETURN "Recommend: " \&
SWITCH(
    TRUE(),
    \_Group = "High Value Customer at Risk to Churn",
        "Retention call recommended.",
    \_Group = "High Outstanding Balance",
        "Follow up on outstanding payment.",
    \_Group = "Payment Reminder Required",
        "Send a payment reminder.",
    " "
)

High Risk Customers = 
CALCULATE(
    COUNTROWS(TelecomChurn\_Nigeria\_Clean),
    TelecomChurn\_Nigeria\_Clean\[Risk\_Score] >= 80
)

High Risk Account Count = 
CALCULATE(
    DISTINCTCOUNT('TelecomChurn\_Nigeria\_Clean'\[Customer\_ID]),
    'TelecomChurn\_Nigeria\_Clean'\[Risk\_Score] >= 70
)

Premium Data Users = 
CALCULATE(
    COUNT('TelecomChurn\_Nigeria\_Clean'\[Customer\_ID]),
    'TelecomChurn\_Nigeria\_Clean'\[Monthly\_Data\_Usage\_GB] > 50
)

Frequent Support Tickets = 
CALCULATE(
    COUNT('TelecomChurn\_Nigeria\_Clean'\[Customer\_ID]), 
    'TelecomChurn\_Nigeria\_Clean'\[Support\_Tickets] > 3
)

Frequent Late Payments = 
CALCULATE(
    COUNT('TelecomChurn\_Nigeria\_Clean'\[Customer\_ID]), 
    'TelecomChurn\_Nigeria\_Clean'\[Late\_Payment\_Count] > 2
)

Frequent Late Payers with High Support Tickets = 
CALCULATE(
    COUNT('TelecomChurn\_Nigeria\_Clean'\[Customer\_ID]),
    FILTER(
        'TelecomChurn\_Nigeria\_Clean',
        'TelecomChurn\_Nigeria\_Clean'\[Churn] = "No" \&\&
        'TelecomChurn\_Nigeria\_Clean'\[Late\_Payment\_Count] > 2 \&\&
        'TelecomChurn\_Nigeria\_Clean'\[Support\_Tickets] > 2
    )
)
```

Revenue at risk / recovery

```dax
Revenue at Risk of Churn = 
CALCULATE(
    SUM('TelecomChurn\_Nigeria\_Clean'\[Monthly\_Bill]),
    'TelecomChurn\_Nigeria\_Clean'\[Risk\_Score] >= 0.70 \&\& 'TelecomChurn\_Nigeria\_Clean'\[Churn] = "No"
)

// exposure specifically from the customers who are both risky and worth a lot to us
Revenue at Risk from High-Value Customers = 
CALCULATE(
    SUM('TelecomChurn\_Nigeria\_Clean'\[Outstanding\_Balance]) + SUM('TelecomChurn\_Nigeria\_Clean'\[Monthly\_Bill]),
    'TelecomChurn\_Nigeria\_Clean'\[Risk\_Score] >= 70,
    'TelecomChurn\_Nigeria\_Clean'\[Monthly\_Income] >= 250000
)

// not everyone at risk can actually be saved, this scales the target down to something realistic per day
Daily Revenue Recovery Target = 
DIVIDE(\[Revenue at Risk from High-Value Customers] \* 0.65, 30, 0)

// same idea, a slightly different cut using a 40% recovery assumption on outstanding balances specifically
Realizable Recovery Value = 
SUMX(
    FILTER(
        'TelecomChurn\_Nigeria\_Clean',
        'TelecomChurn\_Nigeria\_Clean'\[Risk\_Score] >= 70
    ),
    'TelecomChurn\_Nigeria\_Clean'\[Outstanding\_Balance] \* 0.40
)

Revenue At Risk % = 
DIVIDE(
    \[Annual Revenue At Risk],
    \[Monthly Revenue] \* 12
)
```

Month-over-month / year-over-year trend labels

These all follow the same shape, compare this period to the last one, show an arrow and a percentage, stay blank if there's nothing to compare against.

```dax
CSAT MoM Label = 
VAR \_CM = \[Avg Customer CSAT]
VAR \_CurrentMonth = SELECTEDVALUE('Date Table'\[Month Number])
VAR \_CurrentYear = SELECTEDVALUE('Date Table'\[Year])
VAR \_PM = 
    CALCULATE(
        \[Avg Customer CSAT],
        ALL('Date Table'),
        'Date Table'\[Month Number] = IF(\_CurrentMonth = 1, 12, \_CurrentMonth - 1),
        'Date Table'\[Year] = IF(\_CurrentMonth = 1, \_CurrentYear - 1, \_CurrentYear)
    )
VAR \_perc = DIVIDE(\_CM - \_PM, \_PM)
VAR \_percformat = FORMAT(\_perc, "+0.0%;-0.0%;0.0%")
RETURN
    IF(
        NOT HASONEVALUE('Date Table'\[Month Number]) || NOT HASONEVALUE('Date Table'\[Year]),
        "",
        IF(
            ISBLANK(\_PM),
            "", 
            IF(
                \_perc > 0,
                UNICHAR(129157) \& " " \& \_percformat \& " VS LM",
                UNICHAR(129158) \& " " \& \_percformat \& " VS LM"
            )
        )
    )

CLV MoM Label = 
VAR \_CM = \[Avg CLV]
VAR \_CurrentMonth = SELECTEDVALUE('Date Table'\[Month Number])
VAR \_CurrentYear = SELECTEDVALUE('Date Table'\[Year])
VAR \_PM = 
    CALCULATE(
        \[Avg CLV],
        ALL('Date Table'),
        'Date Table'\[Month Number] = IF(\_CurrentMonth = 1, 12, \_CurrentMonth - 1),
        'Date Table'\[Year] = IF(\_CurrentMonth = 1, \_CurrentYear - 1, \_CurrentYear)
    )
VAR \_perc = DIVIDE(\_CM - \_PM, \_PM)
VAR \_percformat = FORMAT(\_perc, "+0.0%;-0.0%;0.0%")
RETURN
    IF(
        NOT HASONEVALUE('Date Table'\[Month Number]) || NOT HASONEVALUE('Date Table'\[Year]),
        "",
        IF(
            OR(ISBLANK(\_CM), ISBLANK(\_PM)),
            "", 
            IF(
                \_perc > 0,
                UNICHAR(129157) \& " " \& \_percformat \& " VS LM",
                UNICHAR(129158) \& " " \& \_percformat \& " VS LM"
            )
        )
    )

Churn Rate MoM = 
VAR \_CM = \[Churn Rate]
VAR \_CurrentMonth = SELECTEDVALUE('Date Table'\[Month Number])
VAR \_CurrentYear = SELECTEDVALUE('Date Table'\[Year])
VAR \_PM = 
    CALCULATE(
        \[Churn Rate],
        ALL('Date Table'),
        'Date Table'\[Month Number] = IF(\_CurrentMonth = 1, 12, \_CurrentMonth - 1),
        'Date Table'\[Year] = IF(\_CurrentMonth = 1, \_CurrentYear - 1, \_CurrentYear)
    )
VAR \_perc = DIVIDE(\_CM - \_PM, \_PM)
VAR \_percformat = FORMAT(\_perc, "+0.0%;-0.0%;0.0%")
RETURN
    IF(
        NOT HASONEVALUE('Date Table'\[Month Number]) || NOT HASONEVALUE('Date Table'\[Year]),
        "",
        IF(
            ISBLANK(\_PM),
            "", 
            IF(
                \_perc > 0,
                UNICHAR(129157) \& " " \& \_percformat \& " VS LM",
                UNICHAR(129158) \& " " \& \_percformat \& " VS LM"
            )
        )
    )

FCR MoM Label = 
VAR \_CM = \[First Contact Resolution Rate]
VAR \_CurrentMonth = SELECTEDVALUE('Date Table'\[Month Number])
VAR \_CurrentYear = SELECTEDVALUE('Date Table'\[Year])
VAR \_PM = 
    CALCULATE(
        \[First Contact Resolution Rate],
        ALL('Date Table'),
        'Date Table'\[Month Number] = IF(\_CurrentMonth = 1, 12, \_CurrentMonth - 1),
        'Date Table'\[Year] = IF(\_CurrentMonth = 1, \_CurrentYear - 1, \_CurrentYear)
    )
VAR \_perc = DIVIDE(\_CM - \_PM, \_PM)
VAR \_percformat = FORMAT(\_perc, "+0.0%;-0.0%;0.0%")
RETURN
    IF(
        NOT HASONEVALUE('Date Table'\[Month Number]) || NOT HASONEVALUE('Date Table'\[Year]),
        "",
        IF(
            OR(ISBLANK(\_CM), ISBLANK(\_PM)),
            "", 
            IF(
                \_perc > 0,
                UNICHAR(129157) \& " " \& \_percformat \& " VS LM",
                UNICHAR(129158) \& " " \& \_percformat \& " VS LM"
            )
        )
    )

Resolution Time MoM = 
VAR \_CM = \[Avg Resolution Time]
VAR \_CurrentMonth = SELECTEDVALUE('Date Table'\[Month Number])
VAR \_CurrentYear = SELECTEDVALUE('Date Table'\[Year])
VAR \_PM = 
    CALCULATE(
        \[Avg Resolution Time],
        ALL('Date Table'),
        'Date Table'\[Month Number] = IF(\_CurrentMonth = 1, 12, \_CurrentMonth - 1),
        'Date Table'\[Year] = IF(\_CurrentMonth = 1, \_CurrentYear - 1, \_CurrentYear)
    )
VAR \_perc = DIVIDE(\_CM - \_PM, \_PM)
VAR \_percformat = FORMAT(\_perc, "+0.0%;-0.0%;0.0%")
RETURN
    IF(
        NOT HASONEVALUE('Date Table'\[Month Number]) || NOT HASONEVALUE('Date Table'\[Year]),
        "",
        IF(
            OR(ISBLANK(\_CM), ISBLANK(\_PM)),
            "", 
            IF(
                \_perc > 0,
                UNICHAR(129157) \& " " \& \_percformat \& " VS LM",
                UNICHAR(129158) \& " " \& \_percformat \& " VS LM"
            )
        )
    )

Call Minutes MoM = 
VAR \_CM = \[Avg Call Minutes]
VAR \_CurrentMonth = SELECTEDVALUE('Date Table'\[Month Number])
VAR \_CurrentYear = SELECTEDVALUE('Date Table'\[Year])
VAR \_PM = 
    CALCULATE(
        \[Avg Call Minutes],
        ALL('Date Table'),
        'Date Table'\[Month Number] = IF(\_CurrentMonth = 1, 12, \_CurrentMonth - 1),
        'Date Table'\[Year] = IF(\_CurrentMonth = 1, \_CurrentYear - 1, \_CurrentYear)
    )
VAR \_perc = DIVIDE(\_CM - \_PM, \_PM)
VAR \_percformat = FORMAT(\_perc, "+0.0%;-0.0%;0.0%")
RETURN
    IF(
        NOT HASONEVALUE('Date Table'\[Month Number]) || NOT HASONEVALUE('Date Table'\[Year]),
        "",
        IF(
            OR(ISBLANK(\_CM), ISBLANK(\_PM)),
            "", 
            IF(
                \_perc > 0,
                UNICHAR(129157) \& " " \& \_percformat \& " VS LM",
                UNICHAR(129158) \& " " \& \_percformat \& " VS LM"
            )
        )
    )

Data Usage MoM Label = 
VAR \_CM = \[Avg Monthly Data Usage]
VAR \_CurrentMonth = SELECTEDVALUE('Date Table'\[Month Number])
VAR \_CurrentYear = SELECTEDVALUE('Date Table'\[Year])
VAR \_PM = 
    CALCULATE(
        \[Avg Monthly Data Usage],
        ALL('Date Table'),
        'Date Table'\[Month Number] = IF(\_CurrentMonth = 1, 12, \_CurrentMonth - 1),
        'Date Table'\[Year] = IF(\_CurrentMonth = 1, \_CurrentYear - 1, \_CurrentYear)
    )
VAR \_perc = DIVIDE(\_CM - \_PM, \_PM)
VAR \_percformat = FORMAT(\_perc, "+0.0%;-0.0%;0.0%")
RETURN
    IF(
        NOT HASONEVALUE('Date Table'\[Month Number]) || NOT HASONEVALUE('Date Table'\[Year]),
        "",
        IF(
            OR(ISBLANK(\_CM), ISBLANK(\_PM)),
            "", 
            IF(
                \_perc > 0,
                UNICHAR(129157) \& " " \& \_percformat \& " VS LM",
                UNICHAR(129158) \& " " \& \_percformat \& " VS LM"
            )
        )
    )

Data to Voice Ratio MoM = 
VAR \_CM = \[GB consumed per Voice Mins]
VAR \_CurrentMonth = SELECTEDVALUE('Date Table'\[Month Number])
VAR \_CurrentYear = SELECTEDVALUE('Date Table'\[Year])
VAR \_PM = 
    CALCULATE(
        \[GB consumed per Voice Mins],
        ALL('Date Table'),
        'Date Table'\[Month Number] = IF(\_CurrentMonth = 1, 12, \_CurrentMonth - 1),
        'Date Table'\[Year] = IF(\_CurrentMonth = 1, \_CurrentYear - 1, \_CurrentYear)
    )
VAR \_perc = DIVIDE(\_CM - \_PM, \_PM)
VAR \_percformat = FORMAT(\_perc, "+0.0%;-0.0%;0.0%")
RETURN
    IF(
        NOT HASONEVALUE('Date Table'\[Month Number]) || NOT HASONEVALUE('Date Table'\[Year]),
        "",
        IF(
            OR(ISBLANK(\_CM), ISBLANK(\_PM)),
            "", 
            IF(
                \_perc > 0,
                UNICHAR(129157) \& " " \& \_percformat \& " VS LM",
                UNICHAR(129158) \& " " \& \_percformat \& " VS LM"
            )
        )
    )

Tickets MoM Label = 
VAR \_CM = COUNT('TelecomChurn\_Nigeria\_Clean'\[Support\_Tickets])
VAR \_CurrentMonth = SELECTEDVALUE('Date Table'\[Month Number])
VAR \_CurrentYear = SELECTEDVALUE('Date Table'\[Year])
VAR \_PM = 
    CALCULATE(
        COUNT('TelecomChurn\_Nigeria\_Clean'\[Support\_Tickets]),
        ALL('Date Table'),
        'Date Table'\[Month Number] = IF(\_CurrentMonth = 1, 12, \_CurrentMonth - 1),
        'Date Table'\[Year] = IF(\_CurrentMonth = 1, \_CurrentYear - 1, \_CurrentYear)
    )
VAR \_perc = DIVIDE(\_CM - \_PM, \_PM)
VAR \_percformat = FORMAT(\_perc, "+0.0%;-0.0%;0.0%")
RETURN
    IF(
        NOT HASONEVALUE('Date Table'\[Month Number]) || NOT HASONEVALUE('Date Table'\[Year]),
        "",
        IF(
            ISBLANK(\_PM),
            "", 
            IF(
                \_perc > 0,
                UNICHAR(129157) \& " " \& \_percformat \& " VS LM",
                UNICHAR(129158) \& " " \& \_percformat \& " VS LM"
            )
        )
    )

Revenue Risk MoM Label = 
VAR \_CM = SUM('TelecomChurn\_Nigeria\_Clean'\[Monthly\_Bill])
VAR \_CurrentMonth = SELECTEDVALUE('Date Table'\[Month Number])
VAR \_CurrentYear = SELECTEDVALUE('Date Table'\[Year])
VAR \_PM = 
    CALCULATE(
        SUM('TelecomChurn\_Nigeria\_Clean'\[Monthly\_Bill]),
        ALL('Date Table'),
        'Date Table'\[Month Number] = IF(\_CurrentMonth = 1, 12, \_CurrentMonth - 1),
        'Date Table'\[Year] = IF(\_CurrentMonth = 1, \_CurrentYear - 1, \_CurrentYear)
    )
VAR \_perc = DIVIDE(\_CM - \_PM, \_PM)
VAR \_percformat = FORMAT(\_perc, "+0.0%;-0.0%;0.0%")
RETURN
    IF(
        NOT HASONEVALUE('Date Table'\[Month Number]) || NOT HASONEVALUE('Date Table'\[Year]),
        "",
        IF(
            ISBLANK(\_PM),
            "", 
            IF(
                \_perc > 0,
                UNICHAR(129157) \& " " \& \_percformat \& " VS LM",
                UNICHAR(129158) \& " " \& \_percformat \& " VS LM"
            )
        )
    )

// yearly versions, same logic but comparing to last year instead of last month
Average Revenue Lost per Churned Customer YoY = 
VAR \_CM = \[Avg Revenue Lost per Churned Customer]
VAR \_CurrentYear = SELECTEDVALUE('Date Table'\[Year])
VAR \_PM =
    CALCULATE(
        \[Avg Revenue Lost per Churned Customer],
        ALL('Date Table'),
        'Date Table'\[Year] = \_CurrentYear - 1
    )
VAR \_Perc = DIVIDE(\_CM - \_PM, \_PM)
VAR \_PercFormat = FORMAT(\_Perc, "+0.0%;-0.0%;0.0%")
RETURN
IF(
    NOT HASONEVALUE('Date Table'\[Year]),
    "",
    IF(
        ISBLANK(\_PM),
        "",
        IF(
            \_Perc < 0,
            UNICHAR(129157) \& " " \& \_PercFormat \& " VS LY",
            UNICHAR(129158) \& " " \& \_PercFormat \& " VS LY"
        )
    )
)

ARPU YoY = 
VAR \_CM = \[Average Revenue Per User]
VAR \_CurrentYear = SELECTEDVALUE('Date Table'\[Year])
VAR \_PM =
    CALCULATE(
        \[Average Revenue Per User],
        ALL('Date Table'),
        'Date Table'\[Year] = \_CurrentYear - 1
    )
VAR \_Perc = DIVIDE(\_CM - \_PM, \_PM)
VAR \_PercFormat = FORMAT(\_Perc, "+0.0%;-0.0%;0.0%")
RETURN
IF(
    NOT HASONEVALUE('Date Table'\[Year]),
    "",
    IF(
        ISBLANK(\_PM),
        "",
        IF(
            \_Perc > 0,
            UNICHAR(129157) \& " " \& \_PercFormat \& " VS LY",
            UNICHAR(129158) \& " " \& \_PercFormat \& " VS LY"
        )
    )
)

Revenue YoY = 
VAR \_CM = SUM('TelecomChurn\_Nigeria\_Clean'\[Monthly\_Bill])
VAR \_CurrentYear = SELECTEDVALUE('Date Table'\[Year])
VAR \_PM = 
    CALCULATE(
        SUM('TelecomChurn\_Nigeria\_Clean'\[Monthly\_Bill]),
        ALL('Date Table'),
        'Date Table'\[Year] = \_CurrentYear - 1
    )
VAR \_perc = DIVIDE(\_CM - \_PM, \_PM)
VAR \_percformat = FORMAT(\_perc, "+0.0%;-0.0%;0.0%")
RETURN
    IF(
        NOT HASONEVALUE('Date Table'\[Year]),
        "",
        IF(
            ISBLANK(\_PM),
            "", 
            IF(
                \_perc > 0,
                UNICHAR(129157) \& " " \& \_percformat \& " VS LY",
                UNICHAR(129158) \& " " \& \_percformat \& " VS LY"
            )
        )
    )

Outstanding YoY = 
VAR \_CM = SUM('TelecomChurn\_Nigeria\_Clean'\[Outstanding\_Balance])
VAR \_CurrentYear = SELECTEDVALUE('Date Table'\[Year])
VAR \_PM = 
    CALCULATE(
        SUM('TelecomChurn\_Nigeria\_Clean'\[Outstanding\_Balance]),
        ALL('Date Table'),
        'Date Table'\[Year] = \_CurrentYear - 1
    )
VAR \_perc = DIVIDE(\_CM - \_PM, \_PM)
VAR \_percformat = FORMAT(\_perc, "+0.0%;-0.0%;0.0%")
RETURN
    IF(
        NOT HASONEVALUE('Date Table'\[Year]),
        "",
        IF(
            ISBLANK(\_PM),
            "", 
            IF(
                \_perc > 0,
                UNICHAR(129157) \& " " \& \_percformat \& " VS LY",
                UNICHAR(129158) \& " " \& \_percformat \& " VS LY"
            )
        )
    )
```

Conditional formatting colours

These just return a hex code depending on whether things moved in a good or bad direction, used to colour the little up/down indicators on the KPI cards. Worth remembering that "up" isn't always good, e.g. ticket volume going up is bad, but FCR going up is good, so the logic flips depending on what's being measured.

```dax
CF Tickets = 
VAR \_CM = COUNT('TelecomChurn\_Nigeria\_Clean'\[Support\_Tickets])
VAR \_CurrentMonth = SELECTEDVALUE('Date Table'\[Month Number])
VAR \_CurrentYear = SELECTEDVALUE('Date Table'\[Year])
VAR \_PM = CALCULATE(COUNT('TelecomChurn\_Nigeria\_Clean'\[Support\_Tickets]), ALL('Date Table'), 'Date Table'\[Month Number] = IF(\_CurrentMonth = 1, 12, \_CurrentMonth - 1), 'Date Table'\[Year] = IF(\_CurrentMonth = 1, \_CurrentYear - 1, \_CurrentYear))
VAR \_perc = DIVIDE(\_CM - \_PM, \_PM)
RETURN
    IF(
        NOT HASONEVALUE('Date Table'\[Month Number]) || ISBLANK(\_PM),
        "#6B7280",
        SWITCH(TRUE(), \_perc > 0, "#B91C1C", \_perc < 0, "#15803D", "#6B7280") // more tickets is bad, so up = red
    )

CF Revenue = 
VAR \_CM = SUM('TelecomChurn\_Nigeria\_Clean'\[Monthly\_Bill])
VAR \_CurrentYear = SELECTEDVALUE('Date Table'\[Year])
VAR \_PM = CALCULATE(SUM('TelecomChurn\_Nigeria\_Clean'\[Monthly\_Bill]), ALL('Date Table'), 'Date Table'\[Year] = \_CurrentYear - 1)
VAR \_perc = DIVIDE(\_CM - \_PM, \_PM)
RETURN
    IF(
        NOT HASONEVALUE('Date Table'\[Year]) || ISBLANK(\_PM),
        "#6B7280", 
        SWITCH(TRUE(), \_perc > 0, "#15803D", \_perc < 0, "#B91C1C", "#6B7280")
    )

CF Resolution Time = 
VAR \_CM = \[Avg Resolution Time]
VAR \_CurrentMonth = SELECTEDVALUE('Date Table'\[Month Number])
VAR \_CurrentYear = SELECTEDVALUE('Date Table'\[Year])
VAR \_PM = 
    CALCULATE(
        \[Avg Resolution Time],
        ALL('Date Table'),
        'Date Table'\[Month Number] = IF(\_CurrentMonth = 1, 12, \_CurrentMonth - 1),
        'Date Table'\[Year] = IF(\_CurrentMonth = 1, \_CurrentYear - 1, \_CurrentYear)
    )
VAR \_perc = DIVIDE(\_CM - \_PM, \_PM)
RETURN
    IF(
        NOT HASONEVALUE('Date Table'\[Month Number]) || OR(ISBLANK(\_CM), ISBLANK(\_PM)),
        "#6B7280",
        SWITCH(TRUE(), \_perc > 0, "#B91C1C", \_perc < 0, "#15803D", "#6B7280") // waiting longer for resolution is bad
    )

CF Outstanding Color = 
VAR \_CM = SUM('TelecomChurn\_Nigeria\_Clean'\[Outstanding\_Balance])
VAR \_CurrentYear = SELECTEDVALUE('Date Table'\[Year])
VAR \_PM = CALCULATE(SUM('TelecomChurn\_Nigeria\_Clean'\[Outstanding\_Balance]), ALL('Date Table'), 'Date Table'\[Year] = \_CurrentYear - 1)
VAR \_perc = DIVIDE(\_CM - \_PM, \_PM)
RETURN
    IF(
        NOT HASONEVALUE('Date Table'\[Year]) || ISBLANK(\_PM),
        "#6B7280",
        SWITCH(TRUE(), \_perc > 0, "#B91C1C", \_perc < 0, "#15803D", "#6B7280")
    )

CF FCR Color = 
VAR \_CM = \[First Contact Resolution Rate]
VAR \_CurrentMonth = SELECTEDVALUE('Date Table'\[Month Number])
VAR \_CurrentYear = SELECTEDVALUE('Date Table'\[Year])
VAR \_PM = 
    CALCULATE(
        \[First Contact Resolution Rate],
        ALL('Date Table'),
        'Date Table'\[Month Number] = IF(\_CurrentMonth = 1, 12, \_CurrentMonth - 1),
        'Date Table'\[Year] = IF(\_CurrentMonth = 1, \_CurrentYear - 1, \_CurrentYear)
    )
VAR \_perc = DIVIDE(\_CM - \_PM, \_PM)
RETURN
    IF(
        NOT HASONEVALUE('Date Table'\[Month Number]) || OR(ISBLANK(\_CM), ISBLANK(\_PM)),
        "#6B7280",
        SWITCH(TRUE(), \_perc > 0, "#15803D", \_perc < 0, "#B91C1C", "#6B7280") // this one's the opposite, resolving faster is good so up = green
    )

CF Data Usage = 
VAR \_CM = \[Avg Monthly Data Usage]
VAR \_CurrentMonth = SELECTEDVALUE('Date Table'\[Month Number])
VAR \_CurrentYear = SELECTEDVALUE('Date Table'\[Year])
VAR \_PM = CALCULATE(\[Avg Monthly Data Usage], ALL('Date Table'), 'Date Table'\[Month Number] = IF(\_CurrentMonth = 1, 12, \_CurrentMonth - 1), 'Date Table'\[Year] = IF(\_CurrentMonth = 1, \_CurrentYear - 1, \_CurrentYear))
VAR \_perc = DIVIDE(\_CM - \_PM, \_PM)
RETURN
    IF(
        NOT HASONEVALUE('Date Table'\[Month Number]) || OR(ISBLANK(\_CM), ISBLANK(\_PM)),
        "#6B7280",
        SWITCH(TRUE(), \_perc > 0, "#15803D", \_perc < 0, "#B91C1C", "#6B7280")
    )

CF CLV = 
VAR \_CM = \[Avg CLV]
VAR \_CurrentMonth = SELECTEDVALUE('Date Table'\[Month Number])
VAR \_CurrentYear = SELECTEDVALUE('Date Table'\[Year])
VAR \_PM = CALCULATE(\[Avg CLV], ALL('Date Table'), 'Date Table'\[Month Number] = IF(\_CurrentMonth = 1, 12, \_CurrentMonth - 1), 'Date Table'\[Year] = IF(\_CurrentMonth = 1, \_CurrentYear - 1, \_CurrentYear))
VAR \_perc = DIVIDE(\_CM - \_PM, \_PM)
RETURN
    IF(
        NOT HASONEVALUE('Date Table'\[Month Number]) || OR(ISBLANK(\_CM), ISBLANK(\_PM)),
        "#6B7280",
        SWITCH(TRUE(), \_perc > 0, "#15803D", \_perc < 0, "#B91C1C", "#6B7280")
    )

CF Churn Color = 
VAR \_CM = \[Churn Rate]
VAR \_PM = \[Churn Rate LY]
VAR \_perc = DIVIDE(\_CM - \_PM, \_PM)
RETURN
    SWITCH(
        TRUE(),
        \_perc > 0, "#B91C1C",
        \_perc < 0, "#15803D", 
        "#6B7280"             
    )

CF Churn = 
VAR \_CM = \[Churn Rate]
VAR \_CurrentMonth = SELECTEDVALUE('Date Table'\[Month Number])
VAR \_CurrentYear = SELECTEDVALUE('Date Table'\[Year])
VAR \_PM = CALCULATE(\[Churn Rate], ALL('Date Table'), 'Date Table'\[Month Number] = IF(\_CurrentMonth = 1, 12, \_CurrentMonth - 1), 'Date Table'\[Year] = IF(\_CurrentMonth = 1, \_CurrentYear - 1, \_CurrentYear))
VAR \_perc = DIVIDE(\_CM - \_PM, \_PM)
RETURN
    IF(
        NOT HASONEVALUE('Date Table'\[Month Number]) || ISBLANK(\_PM),
        "#6B7280",
        SWITCH(TRUE(), \_perc > 0, "#B91C1C", \_perc < 0, "#15803D", "#6B7280")
    )
```

Customer detail panel (Priority Contacts page)

These build the little text card that shows up when you click on a customer in the priority table. Wrote these so they read like plain sentences instead of raw field values.

```dax
Selected Customer = 
SELECTEDVALUE(
    'TelecomChurn\_Nigeria\_Clean'\[Customer\_ID],
    " "
)

Monthly Bill = 
VAR \_Bill =
    SELECTEDVALUE('TelecomChurn\_Nigeria\_Clean'\[Monthly\_Bill])
RETURN
"Monthly Bill: " \&
IF(
    ISBLANK(\_Bill),
    " ",
    FORMAT(\_Bill, "₦#,##0")
)

Outstanding Balance = 
VAR \_Balance =
    SELECTEDVALUE('TelecomChurn\_Nigeria\_Clean'\[Outstanding\_Balance])
RETURN
"Outstanding Balance: " \&
IF(
    ISBLANK(\_Balance),
    "",
    FORMAT(\_Balance,"₦#,##0")
)

Late Payments = 
VAR \_Late =
    SELECTEDVALUE('TelecomChurn\_Nigeria\_Clean'\[Late\_Payment\_Count])
RETURN
"Late Payments: " \&
FORMAT(COALESCE(\_Late,0),"0")

Support Tickets = 
VAR \_Tickets =
    SELECTEDVALUE('TelecomChurn\_Nigeria\_Clean'\[Support\_Tickets])
RETURN
"Support Tickets: " \&
FORMAT(COALESCE(\_Tickets,0),"0")

Risk Score = 
VAR \_Risk =
    SELECTEDVALUE('TelecomChurn\_Nigeria\_Clean'\[Risk\_Score])
RETURN
"Risk Score: " \&
IF(
    ISBLANK(\_Risk),
    "",
    FORMAT(\_Risk,"0") \& "%"
)

Complaint = 
VAR \_Complaint =
    SELECTEDVALUE('TelecomChurn\_Nigeria\_Clean'\[Complaint\_Category])
RETURN
"Complaint: " \&
COALESCE(\_Complaint,"")

Plan Type = 
VAR \_Plan =
    SELECTEDVALUE('TelecomChurn\_Nigeria\_Clean'\[Plan\_Type])
RETURN
"Plan: " \& COALESCE(\_Plan, " ")
```

Odds and ends

A few measures I built while testing things out, some are used in older versions of segments and kept for reference.

```dax
Total Tickets by Category = 
CALCULATE(SUM('TelecomChurn\_Nigeria\_Clean'\[Support\_Tickets]))

High Value At Risk Retained = CALCULATE(COUNTROWS('TelecomChurn\_Nigeria\_Clean'), 'TelecomChurn\_Nigeria\_Clean'\[Estimated\_CLV] > 94764, 'TelecomChurn\_Nigeria\_Clean'\[Risk\_Score] >= 40, 'TelecomChurn\_Nigeria\_Clean'\[Churn] = "No")

High Risk Retained = CALCULATE(COUNTROWS('TelecomChurn\_Nigeria\_Clean'), 'TelecomChurn\_Nigeria\_Clean'\[Risk\_Score] >= 40, 'TelecomChurn\_Nigeria\_Clean'\[Churn] = "No")
```

