---
description: 'Power BI: DAX best practices and row-level security (RLS)'
applyTo: '**/*.{pbix,dax,md,txt,json}'
---


# Power BI DAX Best Practices

## Overview
This document provides comprehensive instructions for writing efficient, maintainable, and performant DAX (Data Analysis Expressions) formulas in Power BI, based on Microsoft's official guidance and best practices.

## Core DAX Principles

### 1. Formula Structure and Variables
Always use variables to improve performance, readability, and debugging:

```dax
// ✅ PREFERRED: Using variables for clarity and performance
Sales YoY Growth % =
VAR CurrentSales = [Total Sales]
VAR PreviousYearSales = 
    CALCULATE(
        [Total Sales],
        SAMEPERIODLASTYEAR('Date'[Date])
    )
RETURN
    DIVIDE(CurrentSales - PreviousYearSales, PreviousYearSales)

// ❌ AVOID: Repeated calculations without variables  
Sales YoY Growth % =
DIVIDE(
    [Total Sales] - CALCULATE([Total Sales], SAMEPERIODLASTYEAR('Date'[Date])),
    CALCULATE([Total Sales], SAMEPERIODLASTYEAR('Date'[Date]))
)
```

**Key Benefits of Variables:**
- **Performance**: Calculations are evaluated once and cached
- **Readability**: Complex formulas become self-documenting
- **Debugging**: Can temporarily return variable values for testing
- **Maintainability**: Changes need to be made in only one place

### 2. Proper Reference Syntax
Follow Microsoft's recommended patterns for column and measure references:

```dax
// ✅ ALWAYS fully qualify column references
Customer Count = 
DISTINCTCOUNT(Sales[CustomerID])

Profit Margin = 
DIVIDE(
    SUM(Sales[Profit]),
    SUM(Sales[Revenue])
)

// ✅ NEVER fully qualify measure references
YTD Sales Growth = 
DIVIDE([YTD Sales] - [YTD Sales PY], [YTD Sales PY])

// ❌ AVOID: Unqualified column references
Customer Count = DISTINCTCOUNT([CustomerID])  // Ambiguous

// ❌ AVOID: Fully qualified measure references
Growth Rate = DIVIDE(Sales[Total Sales] - Sales[Total Sales PY], Sales[Total Sales PY])  // Breaks if measure moves
```

### 3. Error Handling Strategies
Implement robust error handling using appropriate patterns:

```dax
// ✅ PREFERRED: Use DIVIDE function for safe division
Profit Margin = 
DIVIDE([Total Profit], [Total Revenue])

// ✅ PREFERRED: Use defensive strategies in model design
Average Order Value = 
VAR TotalOrders = COUNTROWS(Orders)
VAR TotalRevenue = SUM(Orders[Amount])
RETURN
    IF(TotalOrders > 0, DIVIDE(TotalRevenue, TotalOrders))

// ❌ AVOID: ISERROR and IFERROR functions (performance impact)
Profit Margin = 
IFERROR([Total Profit] / [Total Revenue], BLANK())

// ❌ AVOID: Complex error handling that could be prevented
Unsafe Calculation = 
IF(
    OR(
        ISBLANK([Revenue]),
        [Revenue] = 0
    ),
    BLANK(),
    [Profit] / [Revenue]
)
```

## DAX Function Categories and Best Practices

### Aggregation Functions
```dax
// Use appropriate aggregation functions for performance
Customer Count = DISTINCTCOUNT(Sales[CustomerID])  // ✅ For unique counts
Order Count = COUNTROWS(Orders)                    // ✅ For row counts  
Average Deal Size = AVERAGE(Sales[DealValue])      // ✅ For averages

// Avoid COUNT when COUNTROWS is more appropriate
// ❌ COUNT(Sales[OrderID]) - slower for counting rows
// ✅ COUNTROWS(Sales) - faster and more explicit
```

### Filter and Context Functions
```dax
// Efficient use of CALCULATE with multiple filters
High Value Customers = 
CALCULATE(
    DISTINCTCOUNT(Sales[CustomerID]),
    Sales[OrderValue] > 1000,
    Sales[OrderDate] >= DATE(2024,1,1)
)

// Proper context modification patterns
Same Period Last Year = 
CALCULATE(
    [Total Sales],
    SAMEPERIODLASTYEAR('Date'[Date])
)

// Using FILTER appropriately (avoid as filter argument)
// ✅ PREFERRED: Direct filter expression
High Value Orders = 
CALCULATE(
    [Total Sales],
    Sales[OrderValue] > 1000
)

// ❌ AVOID: FILTER as filter argument (unless table manipulation needed)
High Value Orders = 
CALCULATE(
    [Total Sales],
    FILTER(Sales, Sales[OrderValue] > 1000)
)
```

### Time Intelligence Patterns
```dax
// Standard time intelligence measures
YTD Sales = 
CALCULATE(
    [Total Sales],
    DATESYTD('Date'[Date])
)

MTD Sales = 
CALCULATE(
    [Total Sales],
    DATESMTD('Date'[Date])
)

// Moving averages with proper date handling
3-Month Moving Average = 
VAR CurrentDate = MAX('Date'[Date])
VAR StartDate = EDATE(CurrentDate, -2)
RETURN
    CALCULATE(
        DIVIDE([Total Sales], 3),
        DATESBETWEEN(
            'Date'[Date],
            StartDate,
            CurrentDate
        )
    )

// Quarter over quarter growth
QoQ Growth = 
VAR CurrentQuarter = [Total Sales]
VAR PreviousQuarter = 
    CALCULATE(
        [Total Sales],
        DATEADD('Date'[Date], -1, QUARTER)
    )
RETURN
    DIVIDE(CurrentQuarter - PreviousQuarter, PreviousQuarter)
```

### Advanced DAX Patterns
```dax
// Ranking with proper context
Product Rank = 
RANKX(
    ALL(Product[ProductName]),
    [Total Sales],
    ,
    DESC,
    DENSE
)

// Running totals
Running Total = 
CALCULATE(
    [Total Sales],
    FILTER(
        ALL('Date'[Date]),
        'Date'[Date] <= MAX('Date'[Date])
    )
)

// ABC Analysis (Pareto)
ABC Classification = 
VAR CurrentProductSales = [Total Sales]
VAR TotalSales = CALCULATE([Total Sales], ALL(Product))
VAR RunningTotal = 
    CALCULATE(
        [Total Sales],
        FILTER(
            ALL(Product),
            [Total Sales] >= CurrentProductSales
        )
    )
VAR PercentageOfTotal = DIVIDE(RunningTotal, TotalSales)
RETURN
    SWITCH(
        TRUE(),
        PercentageOfTotal <= 0.8, "A",
        PercentageOfTotal <= 0.95, "B",
        "C"
    )
```

## Performance Optimization Techniques

### 1. Efficient Variable Usage
```dax
// ✅ Store expensive calculations in variables
Complex Measure = 
VAR BaseCalculation = 
    CALCULATE(
        SUM(Sales[Amount]),
        FILTER(
            Product,
            Product[Category] = "Electronics"
        )
    )
VAR PreviousYear = 
    CALCULATE(
        BaseCalculation,
        SAMEPERIODLASTYEAR('Date'[Date])
    )
RETURN
    DIVIDE(BaseCalculation - PreviousYear, PreviousYear)
```

### 2. Context Transition Optimization
```dax
// ✅ Minimize context transitions in iterator functions
Total Product Profit = 
SUMX(
    Product,
    Product[UnitPrice] - Product[UnitCost]
)

// ❌ Avoid unnecessary calculated columns in large tables
// Create in Power Query instead when possible
```

### 3. Efficient Filtering Patterns
```dax
// ✅ Use table expressions efficiently
Top 10 Customers = 
CALCULATE(
    [Total Sales],
    TOPN(
        10,
        ALL(Customer[CustomerName]),
        [Total Sales]
    )
)

// ✅ Leverage relationship filtering
Sales with Valid Customers = 
CALCULATE(
    [Total Sales],
    FILTER(
        Customer,
        NOT(ISBLANK(Customer[CustomerName]))
    )
)
```

## Common DAX Anti-Patterns to Avoid

### 1. Performance Anti-Patterns
```dax
// ❌ AVOID: Nested CALCULATE functions
Inefficient Nested = 
CALCULATE(
    CALCULATE(
        [Total Sales],
        Product[Category] = "Electronics"
    ),
    'Date'[Year] = 2024
)

// ✅ PREFERRED: Single CALCULATE with multiple filters
Efficient Single = 
CALCULATE(
    [Total Sales],
    Product[Category] = "Electronics",
    'Date'[Year] = 2024
)

// ❌ AVOID: Converting BLANK to zero unnecessarily
Sales with Zero = 
IF(ISBLANK([Total Sales]), 0, [Total Sales])

// ✅ PREFERRED: Keep BLANK as BLANK for better visual behavior
Sales = SUM(Sales[Amount])
```

### 2. Readability Anti-Patterns
```dax
// ❌ AVOID: Complex nested expressions without variables
Complex Without Variables = 
DIVIDE(
    CALCULATE(SUM(Sales[Revenue]), Sales[Date] >= DATE(2024,1,1)) - 
    CALCULATE(SUM(Sales[Revenue]), Sales[Date] >= DATE(2023,1,1), Sales[Date] < DATE(2024,1,1)),
    CALCULATE(SUM(Sales[Revenue]), Sales[Date] >= DATE(2023,1,1), Sales[Date] < DATE(2024,1,1))
)

// ✅ PREFERRED: Clear variable-based structure
Year Over Year Growth = 
VAR CurrentYear = 
    CALCULATE(
        SUM(Sales[Revenue]),
        Sales[Date] >= DATE(2024,1,1)
    )
VAR PreviousYear = 
    CALCULATE(
        SUM(Sales[Revenue]),
        Sales[Date] >= DATE(2023,1,1),
        Sales[Date] < DATE(2024,1,1)
    )
RETURN
    DIVIDE(CurrentYear - PreviousYear, PreviousYear)
```

## DAX Debugging and Testing Strategies

### 1. Variable-Based Debugging
```dax
// Use this pattern for step-by-step debugging
Debug Measure = 
VAR Step1 = CALCULATE([Sales], 'Date'[Year] = 2024)
VAR Step2 = CALCULATE([Sales], 'Date'[Year] = 2023)  
VAR Step3 = Step1 - Step2
VAR Step4 = DIVIDE(Step3, Step2)
RETURN
    -- Return different variables for testing:
    -- Step1  -- Test current year sales
    -- Step2  -- Test previous year sales  
    -- Step3  -- Test difference calculation
    Step4     -- Final result
```

### 2. Testing Patterns
```dax
// Include data validation in measures
Validated Measure = 
VAR Result = [Complex Calculation]
VAR IsValid = 
    Result >= 0 && 
    Result <= 1 && 
    NOT(ISBLANK(Result))
RETURN
    IF(IsValid, Result, BLANK())
```

## Measure Organization and Naming

### 1. Naming Conventions
```dax
// Use descriptive, consistent naming
Total Sales = SUM(Sales[Amount])
Total Sales YTD = CALCULATE([Total Sales], DATESYTD('Date'[Date]))
Total Sales PY = CALCULATE([Total Sales], SAMEPERIODLASTYEAR('Date'[Date]))
Sales Growth % = DIVIDE([Total Sales] - [Total Sales PY], [Total Sales PY])

// Prefix for measure categories
KPI - Revenue Growth = [Sales Growth %]
Calc - Days Since Last Order = DATEDIFF(MAX(Orders[OrderDate]), TODAY(), DAY)
Base - Order Count = COUNTROWS(Orders)
```

### 2. Measure Dependencies
```dax
// Build measures hierarchically for reusability
// Base measures
Revenue = SUM(Sales[Revenue])
Cost = SUM(Sales[Cost])

// Derived measures  
Profit = [Revenue] - [Cost]
Margin % = DIVIDE([Profit], [Revenue])

// Advanced measures
Profit YTD = CALCULATE([Profit], DATESYTD('Date'[Date]))
Margin Trend = [Margin %] - CALCULATE([Margin %], PREVIOUSMONTH('Date'[Date]))
```

## Model Integration Best Practices

### 1. Working with Star Schema
```dax
// Leverage proper relationships
Sales by Category = 
CALCULATE(
    [Total Sales],
    Product[Category] = "Electronics"
)

// Use dimension tables for filtering
Regional Sales = 
CALCULATE(
    [Total Sales],
    Geography[Region] = "North America"
)
```

### 2. Handle Missing Relationships
```dax
// When direct relationships don't exist
Cross Table Analysis = 
VAR CustomerList = VALUES(Customer[CustomerID])
RETURN
    CALCULATE(
        [Total Sales],
        FILTER(
            Sales,
            Sales[CustomerID] IN CustomerList
        )
    )
```

## Advanced DAX Concepts

### 1. Row Context vs Filter Context
```dax
// Understanding context differences
Row Context Example = 
SUMX(
    Sales,
    Sales[Quantity] * Sales[UnitPrice]  // Row context
)

Filter Context Example = 
CALCULATE(
    [Total Sales],  // Filter context
    Product[Category] = "Electronics"
)
```

### 2. Context Transition
```dax
// When row context becomes filter context
Sales Per Product = 
SUMX(
    Product,
    CALCULATE([Total Sales])  // Context transition happens here
)
```

### 3. Extended Columns and Computed Tables
```dax
// Use for complex analytical scenarios
Product Analysis = 
ADDCOLUMNS(
    Product,
    "Total Sales", CALCULATE([Total Sales]),
    "Rank", RANKX(ALL(Product), CALCULATE([Total Sales])),
    "Category Share", DIVIDE(
        CALCULATE([Total Sales]),
        CALCULATE([Total Sales], ALL(Product[ProductName]))
    )
)
```

### 4. Advanced Time Intelligence Patterns
```dax
// Multi-period comparisons with calculation groups
// Example showing how to create dynamic time calculations
Dynamic Period Comparison = 
VAR CurrentPeriodValue = 
    CALCULATE(
        [Sales],
        'Time Intelligence'[Time Calculation] = "Current"
    )
VAR PreviousPeriodValue = 
    CALCULATE(
        [Sales],
        'Time Intelligence'[Time Calculation] = "PY"
    )
VAR MTDCurrent = 
    CALCULATE(
        [Sales],
        'Time Intelligence'[Time Calculation] = "MTD"
    )
VAR MTDPrevious = 
    CALCULATE(
        [Sales],
        'Time Intelligence'[Time Calculation] = "PY MTD"
    )
RETURN
    DIVIDE(MTDCurrent - MTDPrevious, MTDPrevious)

// Working with fiscal years and custom calendars
Fiscal YTD Sales = 
VAR FiscalYearStart = 
    DATE(
        IF(MONTH(MAX('Date'[Date])) >= 7, YEAR(MAX('Date'[Date])), YEAR(MAX('Date'[Date])) - 1),
        7,
        1
    )
VAR FiscalYearEnd = MAX('Date'[Date])
RETURN
    CALCULATE(
        [Total Sales],
        DATESBETWEEN(
            'Date'[Date],
            FiscalYearStart,
            FiscalYearEnd
        )
    )
```

### 5. Advanced Performance Optimization Techniques
```dax
// Optimized running totals
Running Total Optimized = 
VAR CurrentDate = MAX('Date'[Date])
RETURN
    CALCULATE(
        [Total Sales],
        FILTER(
            ALL('Date'[Date]),
            'Date'[Date] <= CurrentDate
        )
    )

// Efficient ABC Analysis using RANKX
ABC Classification Advanced = 
VAR ProductRank = 
    RANKX(
        ALL(Product[ProductName]),
        [Total Sales],
        ,
        DESC,
        DENSE
    )
VAR TotalProducts = COUNTROWS(ALL(Product[ProductName]))
VAR ClassAThreshold = TotalProducts * 0.2
VAR ClassBThreshold = TotalProducts * 0.5
RETURN
    SWITCH(
        TRUE(),
        ProductRank <= ClassAThreshold, "A",
        ProductRank <= ClassBThreshold, "B",
        "C"
    )

// Efficient Top N with ties handling
Top N Products with Ties = 
VAR TopNValue = 10
VAR MinTopNSales = 
    CALCULATE(
        MIN([Total Sales]),
        TOPN(
            TopNValue,
            ALL(Product[ProductName]),
            [Total Sales]
        )
    )
RETURN
    IF(
        [Total Sales] >= MinTopNSales,
        [Total Sales],
        BLANK()
    )
```

### 6. Complex Analytical Scenarios
```dax
// Customer cohort analysis
Cohort Retention Rate = 
VAR CohortMonth = 
    CALCULATE(
        MIN('Date'[Date]),
        ALLEXCEPT(Sales, Sales[CustomerID])
    )
VAR CurrentMonth = MAX('Date'[Date])
VAR MonthsFromCohort = 
    DATEDIFF(CohortMonth, CurrentMonth, MONTH)
VAR CohortCustomers = 
    CALCULATE(
        DISTINCTCOUNT(Sales[CustomerID]),
        'Date'[Date] = CohortMonth
    )
VAR ActiveCustomersInMonth = 
    CALCULATE(
        DISTINCTCOUNT(Sales[CustomerID]),
        'Date'[Date] = CurrentMonth,
        FILTER(
            Sales,
            CALCULATE(
                MIN('Date'[Date]),
                ALLEXCEPT(Sales, Sales[CustomerID])
            ) = CohortMonth
        )
    )
RETURN
    DIVIDE(ActiveCustomersInMonth, CohortCustomers)

// Market basket analysis
Product Affinity Score = 
VAR CurrentProduct = SELECTEDVALUE(Product[ProductName])
VAR RelatedProduct = SELECTEDVALUE('Related Product'[ProductName])
VAR TransactionsWithBoth = 
    CALCULATE(
        DISTINCTCOUNT(Sales[TransactionID]),
        Sales[ProductName] = CurrentProduct
    ) +
    CALCULATE(
        DISTINCTCOUNT(Sales[TransactionID]),
        Sales[ProductName] = RelatedProduct
    ) -
    CALCULATE(
        DISTINCTCOUNT(Sales[TransactionID]),
        Sales[ProductName] = CurrentProduct,
        CALCULATE(
            COUNTROWS(Sales),
            Sales[ProductName] = RelatedProduct,
            Sales[TransactionID] = EARLIER(Sales[TransactionID])
        ) > 0
    )
VAR TotalTransactions = DISTINCTCOUNT(Sales[TransactionID])
RETURN
    DIVIDE(TransactionsWithBoth, TotalTransactions)
```

### 7. Advanced Debugging and Profiling
```dax
// Debug measure with detailed variable inspection
Complex Measure Debug = 
VAR Step1_FilteredSales = 
    CALCULATE(
        [Sales],
        Product[Category] = "Electronics",
        'Date'[Year] = 2024
    )
VAR Step2_PreviousYear = 
    CALCULATE(
        [Sales],
        Product[Category] = "Electronics",
        'Date'[Year] = 2023
    )
VAR Step3_GrowthAbsolute = Step1_FilteredSales - Step2_PreviousYear
VAR Step4_GrowthPercentage = DIVIDE(Step3_GrowthAbsolute, Step2_PreviousYear)
VAR DebugInfo = 
    "Current: " & FORMAT(Step1_FilteredSales, "#,0") & 
    " | Previous: " & FORMAT(Step2_PreviousYear, "#,0") &
    " | Growth: " & FORMAT(Step4_GrowthPercentage, "0.00%")
RETURN
    -- Switch between these for debugging:
    -- Step1_FilteredSales    -- Test current year
    -- Step2_PreviousYear     -- Test previous year
    -- Step3_GrowthAbsolute   -- Test absolute growth
    -- DebugInfo              -- Show debug information
    Step4_GrowthPercentage    -- Final result

// Performance monitoring measure
Query Performance Monitor = 
VAR StartTime = NOW()
VAR Result = [Complex Calculation]
VAR EndTime = NOW()
VAR ExecutionTime = DATEDIFF(StartTime, EndTime, SECOND)
VAR WarningThreshold = 5 // seconds
RETURN
    IF(
        ExecutionTime > WarningThreshold,
        "⚠️ Slow: " & ExecutionTime & "s - " & Result,
        Result
    )
```

### 8. Working with Complex Data Types
```dax
// JSON parsing and manipulation
Extract JSON Value = 
VAR JSONString = SELECTEDVALUE(Data[JSONColumn])
VAR ParsedValue = 
    IF(
        NOT(ISBLANK(JSONString)),
        PATHCONTAINS(JSONString, "$.analytics.revenue"),
        BLANK()
    )
RETURN
    ParsedValue

// Dynamic measure selection
Dynamic Measure Selector = 
VAR SelectedMeasure = SELECTEDVALUE('Measure Selector'[MeasureName])
RETURN
    SWITCH(
        SelectedMeasure,
        "Revenue", [Total Revenue],
        "Profit", [Total Profit],
        "Units", [Total Units],
        "Margin", [Profit Margin %],
        BLANK()
    )
```

## DAX Formula Documentation

### 1. Commenting Best Practices
```dax
/* 
Business Rule: Calculate customer lifetime value based on:
- Average order value over customer lifetime
- Purchase frequency (orders per year)  
- Customer lifespan (years since first order)
- Retention probability based on last order date
*/
Customer Lifetime Value = 
VAR AvgOrderValue = 
    DIVIDE(
        CALCULATE(SUM(Sales[Amount])),
        CALCULATE(DISTINCTCOUNT(Sales[OrderID]))
    )
VAR OrdersPerYear = 
    DIVIDE(
        CALCULATE(DISTINCTCOUNT(Sales[OrderID])),
        DATEDIFF(
            CALCULATE(MIN(Sales[OrderDate])),
            CALCULATE(MAX(Sales[OrderDate])),
            YEAR
        ) + 1  -- Add 1 to avoid division by zero for customers with orders in single year
    )
VAR CustomerLifespanYears = 3  -- Business assumption: average 3-year relationship
RETURN
    AvgOrderValue * OrdersPerYear * CustomerLifespanYears
```

### 2. Version Control and Change Management
```dax
// Include version history in measure descriptions
/*
Version History:
v1.0 - Initial implementation (2024-01-15)
v1.1 - Added null checking for edge cases (2024-02-01)  
v1.2 - Optimized performance using variables (2024-02-15)
v2.0 - Changed business logic per stakeholder feedback (2024-03-01)

Business Logic:
- Excludes returns and cancelled orders
- Uses ship date for revenue recognition
- Applies regional tax calculations
*/
```

## Testing and Validation Framework

### 1. Unit Testing Patterns
```dax
// Create test measures for validation
Test - Sales Sum = 
VAR DirectSum = SUM(Sales[Amount])
VAR MeasureResult = [Total Sales]
VAR Difference = ABS(DirectSum - MeasureResult)
RETURN
    IF(Difference < 0.01, "PASS", "FAIL: " & Difference)
```

### 2. Performance Testing
```dax
// Monitor execution time for complex measures
Performance Monitor = 
VAR StartTime = NOW()
VAR Result = [Complex Calculation]
VAR EndTime = NOW()
VAR Duration = DATEDIFF(StartTime, EndTime, SECOND)
RETURN
    "Result: " & Result & " | Duration: " & Duration & "s"
```

Remember: Always validate DAX formulas with business users to ensure calculations match business requirements and expectations. Use Power BI's Performance Analyzer and DAX Studio for performance optimization and debugging.

---


# Power BI Security and Row-Level Security Best Practices

## Overview
This document provides comprehensive instructions for implementing robust security patterns in Power BI, focusing on Row-Level Security (RLS), dynamic security, and governance best practices based on Microsoft's official guidance.

## Row-Level Security Fundamentals

### 1. Basic RLS Implementation
```dax
// Simple user-based filtering
[EmailAddress] = USERNAME()

// Role-based filtering with improved security
IF(
    USERNAME() = "Worker",
    [Type] = "Internal",
    IF(
        USERNAME() = "Manager",
        TRUE(),
        FALSE()  // Deny access to unexpected users
    )
)
```

### 2. Dynamic RLS with Custom Data
```dax
// Using CUSTOMDATA() for dynamic filtering
VAR UserRole = CUSTOMDATA()
RETURN
    SWITCH(
        UserRole,
        "SalesPersonA", [SalesTerritory] = "West",
        "SalesPersonB", [SalesTerritory] = "East",
        "Manager", TRUE(),
        FALSE()  // Default deny
    )
```

### 3. Advanced Security Patterns
```dax
// Hierarchical security with territory lookups
=DimSalesTerritory[SalesTerritoryKey]=LOOKUPVALUE(
    DimUserSecurity[SalesTerritoryID], 
    DimUserSecurity[UserName], USERNAME(), 
    DimUserSecurity[SalesTerritoryID], DimSalesTerritory[SalesTerritoryKey]
)

// Multiple condition security
VAR UserTerritories = 
    FILTER(
        UserSecurity,
        UserSecurity[UserName] = USERNAME()
    )
VAR AllowedTerritories = SELECTCOLUMNS(UserTerritories, "Territory", UserSecurity[Territory])
RETURN
    [Territory] IN AllowedTerritories
```

## Embedded Analytics Security

### 1. Static RLS Implementation
```csharp
// Static RLS with fixed roles
var rlsidentity = new EffectiveIdentity(
    username: "username@contoso.com", 
    roles: new List<string>{ "MyRole" },
    datasets: new List<string>{ datasetId.ToString()}
);
```

### 2. Dynamic RLS with Custom Data
```csharp
// Dynamic RLS with custom data
var rlsidentity = new EffectiveIdentity(
    username: "username@contoso.com",
    roles: new List<string>{ "MyRoleWithCustomData" },
    customData: "SalesPersonA",
    datasets: new List<string>{ datasetId.ToString()}
);
```

### 3. Multi-Dataset Security
```json
{
    "accessLevel": "View",
    "identities": [
        {
            "username": "France",
            "roles": [ "CountryDynamic"],
            "datasets": [ "fe0a1aeb-f6a4-4b27-a2d3-b5df3bb28bdc" ]
        }
    ]
}
```

## Database-Level Security Integration

### 1. SQL Server RLS Integration
```sql
-- Creating security schema and predicate function
CREATE SCHEMA Security;
GO

CREATE FUNCTION Security.tvf_securitypredicate(@SalesRep AS nvarchar(50))
    RETURNS TABLE
WITH SCHEMABINDING
AS
    RETURN SELECT 1 AS tvf_securitypredicate_result
WHERE @SalesRep = USER_NAME() OR USER_NAME() = 'Manager';
GO

-- Applying security policy
CREATE SECURITY POLICY SalesFilter
ADD FILTER PREDICATE Security.tvf_securitypredicate(SalesRep)
ON sales.Orders
WITH (STATE = ON);
GO
```

### 2. Fabric Warehouse Security
```sql
-- Creating schema for Security
CREATE SCHEMA Security;
GO

-- Creating a function for the SalesRep evaluation
CREATE FUNCTION Security.tvf_securitypredicate(@UserName AS varchar(50))
    RETURNS TABLE
WITH SCHEMABINDING
AS
    RETURN SELECT 1 AS tvf_securitypredicate_result
WHERE @UserName = USER_NAME()
OR USER_NAME() = 'BatchProcess@contoso.com';
GO

-- Using the function to create a Security Policy
CREATE SECURITY POLICY YourSecurityPolicy
ADD FILTER PREDICATE Security.tvf_securitypredicate(UserName_column)
ON sampleschema.sampletable
WITH (STATE = ON);
GO
```

## Advanced Security Patterns

### 1. Paginated Reports Security
```json
{
    "format": "PDF",
    "paginatedReportConfiguration":{
        "identities": [
            {"username": "john@contoso.com"}
        ]
    }
}
```

### 2. Power Pages Integration
```html
{% powerbi authentication_type:"powerbiembedded" path:"https://app.powerbi.com/groups/00000000-0000-0000-0000-000000000000/reports/00000000-0000-0000-0000-000000000001/ReportSection" roles:"pagesuser" %}
```

### 3. Multi-Tenant Security
```json
{
  "datasets": [
    {
      "id": "fff1a505-xxxx-xxxx-xxxx-e69f81e5b974",
    }
  ],
  "reports": [
    {
      "allowEdit": false,
      "id": "10ce71df-xxxx-xxxx-xxxx-814a916b700d"
    }
  ],
  "identities": [
    {
      "username": "YourUsername",
      "datasets": [
        "fff1a505-xxxx-xxxx-xxxx-e69f81e5b974"
      ],
      "roles": [
        "YourRole"
      ]
    }
  ],
  "datasourceIdentities": [
    {
      "identityBlob": "eyJ…",
      "datasources": [
        {
          "datasourceType": "Sql",
          "connectionDetails": {
            "server": "YourServerName.database.windows.net",
            "database": "YourDataBaseName"
          }
        }
      ]
    }
  ]
}
```

## Security Design Patterns

### 1. Partial RLS Implementation
```dax
// Create summary table for partial RLS
SalesRevenueSummary =
SUMMARIZECOLUMNS(
    Sales[OrderDate],
    "RevenueAllRegion", SUM(Sales[Revenue])
)

// Apply RLS only to detail level
Salesperson Filter = [EmailAddress] = USERNAME()
```

### 2. Hierarchical Security
```dax
// Manager can see all, others see their own
VAR CurrentUser = USERNAME()
VAR UserRole = LOOKUPVALUE(
    UserRoles[Role], 
    UserRoles[Email], CurrentUser
)
RETURN
    SWITCH(
        UserRole,
        "Manager", TRUE(),
        "Salesperson", [SalespersonEmail] = CurrentUser,
        "Regional Manager", [Region] IN (
            SELECTCOLUMNS(
                FILTER(UserRegions, UserRegions[Email] = CurrentUser),
                "Region", UserRegions[Region]
            )
        ),
        FALSE()
    )
```

### 3. Time-Based Security
```dax
// Restrict access to recent data based on role
VAR UserRole = LOOKUPVALUE(UserRoles[Role], UserRoles[Email], USERNAME())
VAR CutoffDate = 
    SWITCH(
        UserRole,
        "Executive", DATE(1900,1,1),  // All historical data
        "Manager", TODAY() - 365,     // Last year
        "Analyst", TODAY() - 90,      // Last 90 days
        TODAY()                       // Current day only
    )
RETURN
    [Date] >= CutoffDate
```

## Security Validation and Testing

### 1. Role Validation Patterns
```dax
// Security testing measure
Security Test = 
VAR CurrentUsername = USERNAME()
VAR ExpectedRole = "TestRole"
VAR TestResult = 
    IF(
        HASONEVALUE(SecurityRoles[Role]) && 
        VALUES(SecurityRoles[Role]) = ExpectedRole,
        "PASS: Role applied correctly",
        "FAIL: Incorrect role or multiple roles"
    )
RETURN
    "User: " & CurrentUsername & " | " & TestResult
```

### 2. Data Exposure Audit
```dax
// Audit measure to track data access
Data Access Audit = 
VAR AccessibleRows = COUNTROWS(FactTable)
VAR TotalRows = CALCULATE(COUNTROWS(FactTable), ALL(FactTable))
VAR AccessPercentage = DIVIDE(AccessibleRows, TotalRows) * 100
RETURN
    "User: " & USERNAME() & 
    " | Accessible: " & FORMAT(AccessibleRows, "#,0") & 
    " | Total: " & FORMAT(TotalRows, "#,0") & 
    " | Access: " & FORMAT(AccessPercentage, "0.00") & "%"
```

## Governance and Administration

### 1. Automated Security Group Management
```powershell
# Add security group to Power BI workspace
# Sign in to Power BI
Login-PowerBI

# Set up the security group object ID
$SGObjectID = "<security-group-object-ID>"

# Get the workspace
$pbiWorkspace = Get-PowerBIWorkspace -Filter "name eq '<workspace-name>'"

# Add the security group to the workspace
Add-PowerBIWorkspaceUser -Id $($pbiWorkspace.Id) -AccessRight Member -PrincipalType Group -Identifier $($SGObjectID)
```

### 2. Security Monitoring
```powershell
# Monitor Power BI access patterns
$workspaces = Get-PowerBIWorkspace
foreach ($workspace in $workspaces) {
    $users = Get-PowerBIWorkspaceUser -Id $workspace.Id
    Write-Host "Workspace: $($workspace.Name)"
    foreach ($user in $users) {
        Write-Host "  User: $($user.UserPrincipalName) - Access: $($user.AccessRight)"
    }
}
```

### 3. Compliance Reporting
```dax
// Compliance dashboard measures
Users with Data Access = 
CALCULATE(
    DISTINCTCOUNT(AuditLog[Username]),
    AuditLog[AccessType] = "DataAccess",
    AuditLog[Date] >= TODAY() - 30
)

High Privilege Users = 
CALCULATE(
    DISTINCTCOUNT(UserRoles[Email]),
    UserRoles[Role] IN {"Admin", "Manager", "Executive"}
)

Security Violations = 
CALCULATE(
    COUNTROWS(AuditLog),
    AuditLog[EventType] = "SecurityViolation",
    AuditLog[Date] >= TODAY() - 7
)
```

## Best Practices and Anti-Patterns

### ✅ Security Best Practices

#### 1. Principle of Least Privilege
```dax
// Always default to restrictive access
Default Security = 
VAR UserPermissions = 
    FILTER(
        UserAccess,
        UserAccess[Email] = USERNAME()
    )
RETURN
    IF(
        COUNTROWS(UserPermissions) > 0,
        [Territory] IN SELECTCOLUMNS(UserPermissions, "Territory", UserAccess[Territory]),
        FALSE()  // No access if not explicitly granted
    )
```

#### 2. Explicit Role Validation
```dax
// Validate expected roles explicitly
Role-Based Filter = 
VAR UserRole = LOOKUPVALUE(UserRoles[Role], UserRoles[Email], USERNAME())
VAR AllowedRoles = {"Analyst", "Manager", "Executive"}
RETURN
    IF(
        UserRole IN AllowedRoles,
        SWITCH(
            UserRole,
            "Analyst", [Department] = LOOKUPVALUE(UserDepartments[Department], UserDepartments[Email], USERNAME()),
            "Manager", [Region] = LOOKUPVALUE(UserRegions[Region], UserRegions[Email], USERNAME()),
            "Executive", TRUE()
        ),
        FALSE()  // Deny access for unexpected roles
    )
```

### ❌ Security Anti-Patterns to Avoid

#### 1. Overly Permissive Defaults
```dax
// ❌ AVOID: This grants full access to unexpected users
Bad Security Filter = 
IF(
    USERNAME() = "SpecificUser",
    [Type] = "Internal",
    TRUE()  // Dangerous default
)
```

#### 2. Complex Security Logic
```dax
// ❌ AVOID: Overly complex security that's hard to audit
Overly Complex Security = 
IF(
    OR(
        AND(USERNAME() = "User1", WEEKDAY(TODAY()) <= 5),
        AND(USERNAME() = "User2", HOUR(NOW()) >= 9, HOUR(NOW()) <= 17),
        AND(CONTAINS(VALUES(SpecialUsers[Email]), SpecialUsers[Email], USERNAME()), [Priority] = "High")
    ),
    [Type] IN {"Internal", "Confidential"},
    [Type] = "Public"
)
```

## Security Integration Patterns

### 1. Azure AD Integration
```csharp
// Generate token with Azure AD user context
var tokenRequest = new GenerateTokenRequestV2(
    reports: new List<GenerateTokenRequestV2Report>() { new GenerateTokenRequestV2Report(reportId) },
    datasets: datasetIds.Select(datasetId => new GenerateTokenRequestV2Dataset(datasetId.ToString())).ToList(),
    targetWorkspaces: targetWorkspaceId != Guid.Empty ? new List<GenerateTokenRequestV2TargetWorkspace>() { new GenerateTokenRequestV2TargetWorkspace(targetWorkspaceId) } : null,
    identities: new List<EffectiveIdentity> { rlsIdentity }
);

var embedToken = pbiClient.EmbedToken.GenerateToken(tokenRequest);
```

### 2. Service Principal Authentication
```csharp
// Service principal with RLS for embedded scenarios
public EmbedToken GetEmbedToken(Guid reportId, IList<Guid> datasetIds, [Optional] Guid targetWorkspaceId)
{
    PowerBIClient pbiClient = this.GetPowerBIClient();

    var rlsidentity = new EffectiveIdentity(
       username: "username@contoso.com",
       roles: new List<string>{ "MyRole" },
       datasets: new List<string>{ datasetId.ToString()}
    );
    
    var tokenRequest = new GenerateTokenRequestV2(
        reports: new List<GenerateTokenRequestV2Report>() { new GenerateTokenRequestV2Report(reportId) },
        datasets: datasetIds.Select(datasetId => new GenerateTokenRequestV2Dataset(datasetId.ToString())).ToList(),
        targetWorkspaces: targetWorkspaceId != Guid.Empty ? new List<GenerateTokenRequestV2TargetWorkspace>() { new GenerateTokenRequestV2TargetWorkspace(targetWorkspaceId) } : null,
        identities: new List<EffectiveIdentity> { rlsIdentity }
    );

    var embedToken = pbiClient.EmbedToken.GenerateToken(tokenRequest);

    return embedToken;
}
```

## Security Monitoring and Auditing

### 1. Access Pattern Analysis
```dax
// Identify unusual access patterns
Unusual Access Pattern = 
VAR UserAccessCount = 
    CALCULATE(
        COUNTROWS(AccessLog),
        AccessLog[Date] >= TODAY() - 7
    )
VAR AvgUserAccess = 
    CALCULATE(
        AVERAGE(AccessLog[AccessCount]),
        ALL(AccessLog[Username]),
        AccessLog[Date] >= TODAY() - 30
    )
RETURN
    IF(
        UserAccessCount > AvgUserAccess * 3,
        "⚠️ High Activity",
        "Normal"
    )
```

### 2. Data Breach Detection
```dax
// Detect potential data exposure
Potential Data Exposure = 
VAR UnexpectedAccess = 
    CALCULATE(
        COUNTROWS(AccessLog),
        AccessLog[AccessResult] = "Denied",
        AccessLog[Date] >= TODAY() - 1
    )
RETURN
    IF(
        UnexpectedAccess > 10,
        "🚨 Multiple Access Denials - Review Required",
        "Normal"
    )
```

Remember: Security is layered - implement defense in depth with proper authentication, authorization, data encryption, network security, and comprehensive auditing. Regularly review and test security implementations to ensure they meet current requirements and compliance standards.