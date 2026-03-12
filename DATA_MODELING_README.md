# Power BI Data Modeling & DAX — README

> **Adventure Works Dataset** | Power BI Desktop Exercise Guide

---

## Table of Contents

1. [Data Modeling Schemas](#1-data-modeling-schemas)
   - [Flat Schema](#flat-schema)
   - [Star Schema](#star-schema)
   - [Snowflake Schema](#snowflake-schema)
   - [Comparison Table](#schema-comparison)
2. [Step 1: Load & Connect Data](#step-1-load--connect-data)
3. [Step 2: Create a Quick Measure — Running Total in Year](#step-2-quick-measure--running-total-in-year)
4. [Step 3: DAX Measure — Total Revenue with SUMX](#step-3-dax-measure--total-revenue-with-sumx)
5. [Step 4: Advanced DAX Formulas](#step-4-advanced-dax-formulas)
   - [Boolean Filter with CALCULATE](#boolean-filter-with-calculate)
   - [Multi-Filter — Black Road Bikes](#multi-filter--black-road-bikes)
   - [USERELATIONSHIP — Sales by Shipping Date](#userelationship--august-sales-by-shipping-date)
   - [Time Intelligence — Year-over-Year Revenue](#time-intelligence--year-over-year-revenue)
6. [Step 5: Build a Date Table with CALENDAR](#step-5-build-a-date-table-with-calendar)
7. [Calculated Table & Columns with ADDCOLUMNS](#calculated-table--columns-with-addcolumns)
8. [DAX Function Reference](#dax-function-reference)

---

## 1. Data Modeling Schemas

A **data model** defines how tables are structured and related to each other. Choosing the right schema directly impacts query performance, DAX complexity, and maintainability.

---

### Flat Schema

A **Flat Schema** stores all data in a single, denormalized table with no relationships.

```
┌─────────────────────────────────────────────────────┐
│                     Sales_Data                      │
│  OrderID | Date | Product | Category | Color |      │
│  Region  | Salesperson | Quantity | UnitPrice | ... │
└─────────────────────────────────────────────────────┘
```

**How it works:**
- All columns live in one table — no joins needed.
- Every row repeats dimension values (e.g., product name, region name).

**When to use it:**
- Quick prototypes or very small datasets.
- When the data source is a single flat export (e.g., a CSV).

**Drawbacks:**
- Massive data duplication (every row repeats "Mountain Bike" as a product name).
- Slow performance at scale — Power BI has to scan one huge table.
- Difficult to maintain; updating a product name means updating thousands of rows.
- DAX measures become complex and error-prone.

---

### Star Schema

A **Star Schema** has one central **Fact table** surrounded by multiple **Dimension tables**. This is the **recommended schema for Power BI**.

```
                   ┌──────────┐
                   │   Date   │
                   └────┬─────┘
                        │
┌──────────┐    ┌───────┴──────┐    ┌──────────┐
│  Region  ├────┤  Sales       ├────┤ Product  │
└──────────┘    │  (Fact)      │    └──────────┘
                └───────┬──────┘
                        │
                   ┌────┴─────┐
                   │  Person  │
                   └──────────┘
```

**Tables in Adventure Works Star Schema:**

| Table | Type | Key Columns |
|-------|------|-------------|
| Sales | Fact | OrderID, DateKey, ProductKey, RegionKey, PersonKey |
| Date | Dimension | DateKey, Date, Year, Month, Quarter |
| Product | Dimension | ProductKey, ProductName, Category, Color, UnitPrice |
| Region | Dimension | RegionKey, RegionName, Country |
| Salesperson | Dimension | PersonKey, Name, Email |

**Relationships:**
- All dimension tables connect to the Fact table via a **one-to-many** relationship.
- Filter direction flows **from dimension → fact** (single direction).

**Why Power BI loves Star Schema:**
- Power BI's in-memory engine (VertiPaq) is optimized for this structure.
- DAX formulas are simpler — you can write `RELATED(Products[Color])` easily.
- Slicers and filters on dimension tables work instantly.

---

### Snowflake Schema

A **Snowflake Schema** extends the Star Schema by further normalizing dimension tables into sub-dimension tables.

```
                        ┌──────────┐
          ┌─────────────┤   Date   │
          │             └──────────┘
          │                  │
     ┌────┴─────┐       ┌────┴──────┐
     │   Year   │       │   Month   │
     └──────────┘       └───────────┘

          ┌──────────┐
          │ Product  │
          └────┬─────┘
          ┌────┴─────┐
          │ Category │
          └────┬─────┘
          ┌────┴────────┐
          │ Subcategory │
          └─────────────┘
```

**Key difference from Star Schema:**
- Dimensions are split into multiple tables (e.g., Product → Category → Subcategory).
- Reduces data redundancy by storing each unique value once.

**When to use it:**
- Very large dimensions where storage is a concern.
- When the source system is a normalized OLTP database.

**Drawbacks in Power BI:**
- More table joins = slower query performance.
- DAX measures require chaining RELATED across multiple hops.
- Power BI's engine is less optimized for snowflake structures.

---

### Schema Comparison

| Feature | Flat | Star | Snowflake |
|---------|------|------|-----------|
| Number of tables | 1 | Multiple | Many |
| Data redundancy | Very High | Low | Minimal |
| Query performance | Poor | ✅ Excellent | Moderate |
| DAX complexity | High | ✅ Simple | Complex |
| Storage efficiency | Poor | Moderate | ✅ Best |
| Power BI recommended | ❌ No | ✅ Yes | ⚠️ Sometimes |
| Ease of maintenance | Poor | Good | Moderate |

> **💡 Best Practice:** Always use a **Star Schema** in Power BI unless you have a specific reason to normalize further. The performance and DAX simplicity benefits far outweigh the minor storage savings of Snowflake.

---

## Step 1: Load & Connect Data

### Import Adventure Works Dataset

1. Open **Power BI Desktop** → `File` → `New`
2. Go to **Home** tab → `Get Data` → `Excel Workbook`
3. Navigate to the Adventure Works folder and select the file
4. In the **Navigator**, select these tables and click **Load**:
   - `Sales`
   - `Product`
   - `Region`
   - `Date`
   - `Salesperson`

### Remove Duplicates

1. Open **Power Query Editor** (Home → Transform Data)
2. Select the `Sales` query
3. Right-click the `SalesOrderNumber` column → **Remove Duplicates**
4. Click **Close & Apply**

### Configure Relationships

1. Switch to **Model view**
2. Click **Manage Relationships**
3. Set relationships between tables:

| From (Many) | To (One) | Cardinality | Filter Direction |
|-------------|----------|-------------|-----------------|
| Sales[DateKey] | Date[DateKey] | Many-to-One | Single |
| Sales[ProductKey] | Product[ProductKey] | Many-to-One | Single |
| Sales[RegionKey] | Region[RegionKey] | Many-to-One | Single |
| Sales[PersonKey] | Salesperson[PersonKey] | Many-to-One | Single |

---

## Step 2: Quick Measure — Running Total in Year

A **Running Total** accumulates values from the start of the year up to the current date context.

### How to create it

1. Right-click the `Sales` table in the Data pane → **New Quick Measure**
2. Select **Running Total** from the calculation dropdown
3. Configure:
   - **Base value:** `Sales[Total Sales]`
   - **Field:** `Date[Year]`
4. Click **OK**
5. Select the new measure → set **Format** to `Currency`, decimal places to `2`

> **Tip:** Power BI's Quick Measure feature generates the DAX automatically. The underlying formula uses `CALCULATE` with `FILTER` to accumulate totals within each year boundary.

---

## Step 3: DAX Measure — Total Revenue with SUMX

**SUMX** iterates over a table row-by-row, evaluates an expression for each row, and sums the results.

### Formula

```dax
Total Revenue = SUMX ( Sales, Sales[Quantity] * Sales[Unit Price] )
```

### How it works

- `SUMX` loops through every row in the `Sales` table.
- For each row it multiplies `Quantity` × `Unit Price`.
- It then sums all those row-level results into one total.

### How to create it

1. Select the `Sales` table in the Data pane
2. Go to **Home** → **New Measure** (or click the formula bar)
3. Paste the formula above
4. Press **Enter**
5. In the Measure Tools ribbon, set **Format** to `Currency`, decimal places `2`

> **Note:** Use `SUMX` instead of `SUM` whenever you need to multiply or calculate across columns before aggregating.

---

## Step 4: Advanced DAX Formulas

### Boolean Filter with CALCULATE

A **Boolean filter** is a TRUE/FALSE expression passed as a filter argument to `CALCULATE`.

```dax
Sales of high-end products = 
CALCULATE (
    SUM ( 'Sales'[Total Sales] ),
    FILTER ( Sales, Sales[Unit Price] >= 500 )
)
```

**Rules for Boolean filter expressions:**
- ✅ Can reference columns from a **single table**
- ❌ Cannot reference **measures**
- ❌ Cannot use a **nested CALCULATE**
- ❌ Cannot use **functions that return a table** (e.g., `SUMX`, `COUNTROWS`)

---

### Multi-Filter — Black Road Bikes

Multiple filter arguments in `CALCULATE` act as **AND** conditions — all must be true simultaneously.

```dax
Black Road Bikes Sales = CALCULATE(
    [Total Revenue],
    Products[Subcategory] = "Road Bikes",
    Products[Color] = "Black"
)
```

This returns revenue only where the product is both a Road Bike **and** Black.

---

### USERELATIONSHIP — August Sales by Shipping Date

By default, Power BI uses the **active** relationship between tables. `USERELATIONSHIP` temporarily activates an **inactive** relationship for the duration of the calculation.

```dax
August Sales by Shipping date = 
CALCULATE (
    SUM(Sales[Sales Amount]),
    FILTER('Date', 'Date'[Month] = "August"),
    USERELATIONSHIP(Sales[Shipping Date], 'Date'[Date])
)
```

**Why use it?**
- The `Sales` table may have both an `Order Date` and a `Shipping Date`.
- Only one relationship to the `Date` table can be active at a time.
- `USERELATIONSHIP` lets you switch to the Shipping Date relationship for this specific measure.

---

### Time Intelligence — Year-over-Year Revenue

#### Previous Year Revenue

```dax
Revenue PY =
VAR RevenuePreviousYear =
    CALCULATE ([Revenue], SAMEPERIODLASTYEAR (Sales[OrderDate].[Date]))
RETURN
RevenuePreviousYear
```

- `VAR` stores the prior year result in a variable for clarity and performance.
- `SAMEPERIODLASTYEAR` shifts the date context back exactly one year.

#### Year-over-Year Growth %

```dax
Revenue YoY =
DIVIDE(
    [Total Revenue] - [Revenue PY],
    [Revenue PY]
)
```

- `DIVIDE` is always preferred over `/` because it safely returns `BLANK()` instead of an error when the denominator is zero.

---

## Step 5: Build a Date Table with CALENDAR

A **dedicated Date table** is essential for time intelligence functions. Power BI requires a continuous, unbroken date range with no gaps.

### Create the base table

1. Go to **Home** tab → Calculations group → **New Table**
2. Enter this formula:

```dax
Date = CALENDAR ( DATE(2015, 1, 1), DATE(2021, 12, 31) )
```

This creates a table with one row per day across the specified range.

### Add calculated columns

After creating the Date table, add each column via **New Column**:

```dax
Year = YEAR ( 'Date'[Date] )
```
Returns the 4-digit year number (e.g., 2020).

```dax
Month = FORMAT ( 'Date'[Date], "MMMM" )
```
Returns the full month name (e.g., "January"). Use `"MMM"` for short names (e.g., "Jan").

```dax
Month Number = MONTH ( 'Date'[Date] )
```
Returns the month as a number 1–12. Useful for sorting the Month column correctly.

```dax
Day of the Week = FORMAT ( WEEKDAY( 'Date'[Date] ), "dddd" )
```
Returns the full weekday name (e.g., "Monday").

```dax
Week Number = WEEKNUM ( 'Date'[Date] )
```
Returns the ISO week number (1–52) for each date.

### Mark as a Date Table

1. Select the Date table
2. Go to **Table Tools** → **Mark as Date Table**
3. Select the `Date` column as the unique date identifier

> **⚠️ Important:** Always sort `Month` by `Month Number` so months appear in calendar order (Jan, Feb, Mar…) instead of alphabetical order (Apr, Aug, Dec…).

---

## Calculated Table & Columns with ADDCOLUMNS

### Create a Calculated Table

`ADDCOLUMNS` creates a new table by appending calculated columns to an existing table.

```dax
Yearly Sales by color =
ADDCOLUMNS (
    Sales,
    "Year",  RELATED ( 'Date'[Year]),
    "Color", RELATED ( Products[Color])
)
```

**How to create it:**
1. Switch to **Model view**
2. Calculations group → **New Table**
3. Paste the formula above

**Result:** A new table with all Sales columns plus `Year` and `Color` — 11 columns total.

---

### Add Calculated Columns to Existing Tables

#### Date Table — Quarter column

```dax
Qtr = QUARTER('Date'[Date])
```
Returns each date's fiscal quarter as a number (1–4).

#### Date Table — Short Month Name

```dax
Month = LEFT ( 'Date'[Month], 3 )
```
Trims the full month name to 3 characters (e.g., "January" → "Jan").

#### Product Table — Bring in Color

```dax
Product Color = RELATED ( Products[Color] )
```
Retrieves the `Color` value from the related `Products` table.

**How to add a calculated column:**
1. Select the target table in the Data pane
2. Model view → **New Column**
3. Paste the formula into the formula bar

---

## DAX Function Reference

| Function | Category | Description |
|----------|----------|-------------|
| `ADDCOLUMNS` | Table manipulation | Adds calculated columns to a table or expression |
| `RELATED` | Relationship | Returns a related value from a table on the "one" side |
| `CALCULATE` | Filter context | Evaluates an expression with modified filter context |
| `FILTER` | Table manipulation | Returns a filtered table based on a condition |
| `SUMX` | Iterator | Iterates a table row-by-row and sums an expression |
| `QUARTER` | Date/Time | Returns the quarter number (1–4) from a date |
| `YEAR` | Date/Time | Extracts the 4-digit year from a date |
| `MONTH` | Date/Time | Extracts the month number (1–12) from a date |
| `FORMAT` | Text | Converts a value to text using a format string |
| `LEFT` | Text | Returns N characters from the left of a string |
| `WEEKDAY` | Date/Time | Returns the day of the week as a number |
| `WEEKNUM` | Date/Time | Returns the week number of the year |
| `CALENDAR` | Date/Time | Creates a single-column date table |
| `USERELATIONSHIP` | Relationship | Activates an inactive relationship for a calculation |
| `SAMEPERIODLASTYEAR` | Time Intelligence | Shifts date context back one year |
| `DIVIDE` | Math | Safe division — returns BLANK instead of error on zero |
| `VAR` / `RETURN` | Variables | Stores an intermediate result for reuse in a formula |

---

## Tips & Best Practices

- **Always verify column names** in your data match exactly what is referenced in DAX formulas — names are case-insensitive but spelling must be exact.
- **Use Star Schema** — it produces the simplest DAX and the fastest Power BI queries.
- **Mark your Date table** — required for time intelligence functions like `SAMEPERIODLASTYEAR` to work correctly.
- **Use `DIVIDE` over `/`** — prevents errors when the denominator is zero.
- **Use `VAR`** — storing intermediate results in variables improves both readability and query performance.
- **Format measures immediately** — set Currency/Percentage format right after creating a measure so visuals display correctly from the start.
- **Sort Month by Month Number** — prevents months appearing in alphabetical order in charts.

---

*Adventure Works Dataset · Power BI Desktop Exercise · DAX Data Modeling Guide*
