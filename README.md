# 📊 Project Title – One-Line Description

<img src="./images/dashboard.gif" aria-label="A gif showing the finacial dashboard" />

## 📑 Table of Contents

- [🔎 Executive Summary](#-executive-summary)
- [❓ Problem Statement](#-problem-statement)
- [🎯 Objectives](#-objectives)
- [📥 Data Sourcing](#-data-sourcing)
- [🏗️ Data Model Formation](#-data-model-formation)
- [🛠️ Skills & Tools Demonstrated](#️-skills--tools-demonstrated)
- [📂 Project Workflow](#-project-workflow)
- [📈 Key Results & Insights](#-key-results--insights)
- [💡 What I Learned](#-what-i-learned)
- [✅ Best Approach (My Perspective)](#-best-approach-my-perspective)
- [🚀 How to Reproduce This Project](#-how-to-reproduce-this-project)
- [Challenges I faced](#challenges-i-faced)
- [📊 Project Structure](#-project-structure)
- [📌 Next Steps](#-next-steps)
- [🔜 My Next Project](#-my-next-project)
- [🙋 About Me](#-about-me)

https://app.powerbi.com/groups/me/reports/19c45587-ee7c-4119-ad00-436c31009fa8/01bd393a8b71a60ca994?experience=power-bi&bookmarkGuid=36654765797e8b09abe8

## 🔎 Executive Summary

Provide a concise summary of your project. Highlight the problem you solved, your methodology, and the key insights or results. Keep this section short but impactful to grab attention.

---

## ❓ Problem Statement

Describe the business problem or analytical challenge that motivated the project. Make it relatable to real-world scenarios or industry needs.

_Example:_  
Customer churn poses a significant risk to subscription-based businesses. Understanding why customers leave and predicting who is at risk enables proactive retention strategies.

---

## 🎯 Objectives

- Clearly state what you set out to achieve.
- Mention the analytical or business goals.

---
## 📈 Key Results & Insights

- List 3–5 top findings from your analysis.
- Present insights in clear, recruiter-friendly language.
- Use visuals (charts, screenshots) if possible.

## 🛠️ Skills & Tools Demonstrated
- Data extraction through REST APIs.
- Ability interpret **SEC EDGAR API** documentation to identify the appropriate URL field and parameters such as the taxonomy and tags necessary for retrieving and preparing accurate data for analysis.
- Data cleaning and transformation using Power Query (**M language**, **user-defined functions**, **column pivoting**, **API request and error handling** )
- Performed **data modeling** by defining relationships between tables, defining measures with **DAX**, creating hierarchies, and ensuring proper use of star schema design to enhance calculation accuracy and report performance.
- Data visualization with built-in Power BI visuals and **custom KPI visuals** using **Deneb (vega-lite)**.
- **Workflow Tools:** Power BI, Git/GitHub.


## 📂 Project Workflow
### 📥 Data Sourcing
The data was sourced from the **SEC EDGAR (Electronic Data Gathering, Analysis, and Retrieval) API**. It is a web-based interface provided by the U.S Securities and Exchange Commission (SEC) that allows to programmatically access public company filings (10-K, 10-Q, 8-K, etc) and financial data submitted to the SEC. Read more about the [**EDGAR API.**](https://www.sec.gov/search-filings/edgar-application-programming-interfaces)

The company-concept API was used to retrieve the XBRL disclosures from a single company (CIK) and concept (a taxonomy and tag) into a single JSON file.   

API: `https://data.sec.gov/api/xbrl/companyconcept/CIK##########/{taxonomy}/{tag}.json`
- `CIK##########` - represent the CIK number for a company. You can find the CIK number of a company [here.](https://www.sec.gov/search-filings/cik-lookup)
- `taxonomy` - A taxonomy is a structured dictionary of financial reporting concepts defined by accounting standards like `us-gaap` or `ifrs`. Each taxonomy has it own standardized concepts (tags). For example: `us-gaap` has concepts such as  `Assets`, `Revenues`, `Liabilities`, etc.
- `tag` - is the individual data elements within a taxonomy.
    For example, under the us-gaap taxonomy, you might have tags like:
    - `us-gaap:Assets` — Total assets reported by the company
    - `us-gaap:Liabilities` — Total liabilities
    - `us-gaap:Revenues` — Reported revenues  
To know the available taxonomy and tags from the financial statement, open any financial form in XBRL viewer and click on any highlight value to get the taxonomy and tag.
An example of Zoom, Inc 10-K is provided [here](https://www.sec.gov/ix?doc=/Archives/edgar/data/0001585521/000158552125000042/zm-20250131.htm#fact-identifier-82).

    ```fsharp
        let
            GetAPIData = (CIK as text, tag as text) =>
            let
                Source = Json.Document(
                    Web.Contents(
                        "https://data.sec.gov/api/xbrl/companyconcept/CIK" & CIK& "/us-gaap/" & tag & ".json",
                        [
                            Headers = [
                                #"User-Agent" = "Emmanuel Fosu emmanuel.fosuduffour@gmail.com"
                            ]
                        ])),
                Data   = Table.FromRecords({Source}) 
            in
                Data
        in
            GetAPIData
    ```
    _This is the query function used to get various tag values with us-gaap taxonomy from the financial forms._

### Data Cleaning and Transformation
1. A company–tag mapping table was created where each company (identified by cik) is paired with all relevant tag attributes.
    ```fsharp
    Source = Table.SelectColumns(company_dim, "cik"),
        #"Added tags" = Table.AddColumn(Source, "tags", each tags_dim),
        #"Expanded tags" = Table.ExpandTableColumn(#"Added tags", "tags", {"tag", "name", "tag_1", "tag_2"}, {"tag", "name", "tag_1", "tag_2"})
    ```

2. After building the company–tag mapping table, the next stage involved fetching financial data from an external API using each company’s cik and associated tag values.
    ```fsharp
     #"Added fectched data" = Table.AddColumn(
                                    #"Expanded tags", 
                                    "custom", 
                                    each
                                        try GetAPIData([cik], [tag])
                                        otherwise try GetAPIData([cik], [tag_1])
                                        otherwise null
                                    ),
    #"Expanded custom" = Table.ExpandTableColumn(#"Added fectched data", "custom", {"units"}, {"units"}),
    #"Expanded units" = Table.ExpandRecordColumn(#"Expanded custom", "units", {"USD", "USD/shares", "shares"}, {"USD", "USD/shares", "shares"}),
    #"Added data" = Table.AddColumn(#"Expanded units", "USD_new", each if [USD] <> null then [USD] else if [shares] <> null then [shares] else [#"USD/shares"]),
    ```

3. After consolidating all financial values into the unified USD_new column, redundant and intermediate columns are removed to simplify the dataset.
    ```fsharp
    #"Removed extra data" = Table.RemoveColumns(#"Added data",{"USD", "USD/shares", "shares", "tag", "tag_1", "tag_2"}),

    ```

4. This step expanded the contents of the unified `USD_new` column to expose detailed financial information returned by the API. This produces a flat, structured table where each company and tag combination is linked with its corresponding financial `value`, `form type`, and `reporting frame` making the dataset ready for filtering and time-based analysis.
    ```fsharp
        #"Expanded USD_new" = Table.ExpandListColumn(#"Removed extra data", "USD_new"),
        #"Expanded USD_new1" = Table.ExpandRecordColumn(#"Expanded USD_new", "USD_new", {"val", "form", "frame"}, {"val", "form", "frame"}),
    ```

5. Next, the dataset was filtered to include only annual financial filings (Form 10-K) and removes records without a valid reporting frame.
    ```fsharp
        #"Only 10-K Rows" = Table.SelectRows(
        #"Expanded USD_new1",
        each ([form] = "10-K") and ([frame] <> null)
    )
    ```
6. After filtering out 10-K values, the dataset was validated and filtered keep only records with properly formatted reporting frames.
It adds a `valid_frame` column using a regular expression to identify entries like `CY2022` or `CY2022Q4I`, then filters out any invalid or missing frames — ensuring that only reliable annual and quarterly data remain for analysis.
    ```fsharp
    "Added 'valid_frame'" = Table.AddColumn(#"Only 10-K Rows", "valid_frame", each regex_test("^(CY\\d{4}|CY\\d{4}Q(3|4)I)$", [frame])),
        #"Filtered Rows with valid frame" = Table.SelectRows(#"Added 'valid_frame'", each ([valid_frame] = true)),
    ```

7. Next step extracted the fiscal year from each record’s frame (e.g., "CY2022" → 2022) and converts it to a numeric type.
It standardizes the year format, making it easier to perform time-based analysis, establish relationships with a fiscal calendar table, and support year-over-year comparisons in Power BI.
    ```fsharp
    #"Extracted Year from 'frame'" = Table.AddColumn(#"Filtered Rows with valid frame", "fiscal_year", each Text.Middle([frame], 2, 4), type text),
    #"Changed Year Type" = Table.TransformColumnTypes(#"Extracted Year from 'frame'",{{"fiscal_year", Int64.Type}}),
    ```

8. This step kept only the last seven fiscal years of data, ensuring that the analysis remains current and focused on recent performance trends.
Additionally, redundant columns were removed from the dataset to make it readable, lighter, and ready for modeling in Power BI or other analytical tools.
    ```fsharp
    CurrentYear = Date.Year(DateTime.LocalNow()),
    LastYear = CurrentYear - 1,
    MinYear = LastYear - 6,
    #"Filtered Row Not in Years Range" = Table.SelectRows(#"Changed Year Type", each [fiscal_year] <= LastYear and [fiscal_year] >= MinYear),
    #"Removed Columns" = Table.RemoveColumns(#"Filtered Row Not in Years Range",{"form", "frame", "valid_frame"}),
    ```

9. The final step converted `val` to numeric form for accurate calculations pivoted the table so each financial metric (e.g., Assets, Liabilities, Revenue) becomes its own column and formatted fiscal_year as a date to support time-based analysis.

```fsharp
#"Changed val to number" =
    Table.TransformColumnTypes(#"Removed Columns", {{"val", type number}}),

#"Pivoted Column" =
    Table.Pivot(
        #"Changed val to number",
        List.Distinct(#"Changed val to number"[name]),
        "name",
        "val",
        List.Sum
    ),

#"Changed fiscal_year to text" =
    Table.TransformColumnTypes(#"Pivoted Column", {{"fiscal_year", type text}}),

#"Changed fiscal_year to date" =
    Table.TransformColumnTypes(#"Changed fiscal_year to text", {{"fiscal_year", type date}})

```

### Data Modeling
#### Overview
This data model integrates **financial performance data** and **market share prices** to enable analysis of company fundamentals, valuation trends, and year-over-year performance.  
It follows a **star schema** structure, combining fact tables (`revenues_fact`, `sharePrice_fact`) with related dimension tables (`company_dim`, `Fiscal Year`, `Share Price Date`).

---
<img src="./images/semantic_model_disconnected.png" />

*The star schema showing various tables and relationship formed.*

#### Tables in the Model

**1. `company_dim`**
- **Primary Key:** `cik`
- Contains company-level information such as:
  - `cik` – Central Index Key (unique company identifier)
  - `company` – Company name
  - `logo_URL` – URL of the company’s logo

**2. `revenues_fact`**
- **Foreign Keys:**
  - `cik` → `company_dim.cik`
  - `fiscal_year` → `Fiscal Year.Year`
- Stores annual financial and accounting data for each company.
- Key fields include:
  - `Assets`
  - `Capital Expenditures`
  - `Cost of Sales`
  - `EarningsPerShare`
  - `Free Cash Flow`
  - `Current Assets`
  - `Current Liabilities`
  - `Common Stock Outstanding`
  - `fiscal_year`
  - `Gross Profit`
  - `Income Tax Expense`
  - `Interest Expense`
  - `Inventory`
  - `Liabilities`
  - `Stockholders Equity`
  - `Revenue`
  - `Preferred Equity`

 **3. `Fiscal Year`**
- **Primary Key:** `Year`
- Provides fiscal time dimension for financial data.
- Fields:
  - `Date`
  - `Year`

**4. `sharePrice_fact`**
- **Foreign Keys:**
  - `cik` → `company_dim.cik`
  - `date` → `Share Price Date.Date`
- Contains daily or periodic stock price data.
- Key fields:
  - `cik`
  - `date`
  - `price`
  - `ticker`

**5. `Share Price Date`**
- **Primary Key:** `Date`
- Provides a calendar time dimension for share price analysis.
- Fields:
  - `Date`
  - `Year`


#### Relationships

| Relationship | Type | Description |
|---------------|-----|-------------|
| `company_dim.cik` → `revenues_fact.cik` | One-to-Many | Each company has multiple yearly financial records |
| `company_dim.cik` → `sharePrice_fact.cik` | One-to-Many | Each company has multiple share price entries over time |
| `Fiscal Year.Year` → `revenues_fact.fiscal_year` | One-to-Many | Each fiscal year can have multiple company financial reports |
| `Share Price Date.Date` → `sharePrice_fact.date` | One-to-Many | Each date can have multiple company price records |
| `Share Price Date.Date` → `Fiscal Year.Date` | Many-to-One | Each trading date maps to exactly one fiscal year period — enables aligning share prices to the corresponding fiscal reporting period |

#### Data Modeling Structure

The model consists of **two fact tables** connected through a **shared company dimension**:

- **Fact Tables:**
  - `revenues_fact` (financial data)
  - `sharePrice_fact` (market price data)

- **Dimension Tables:**
  - `company_dim` (company details)
  - `Fiscal Year` (fiscal calendar)
  - `Share Price Date` (trading date dimension)

This structure allows cross-analysis between financial performance and stock price behavior.

---

#### Model Design Notes

- **Schema Type:** Star Schema  
- **Granularity:**
  - `revenues_fact`: One record per company per fiscal year
  - `sharePrice_fact`: One record per company per trading date
- **Key Relationships:**
  - `cik` provides a consistent linkage between company and both fact tables
  - Date tables (`Fiscal Year` and `Share Price Date`) ensure proper time-based slicing
- **Use Case Examples:**
  - Compare **EPS vs Share Price** over time
  - Analyze **Free Cash Flow** trends across fiscal years
  - Track **Company growth metrics** alongside market performance


### Data Visualization and Analysis



## 💡 What I Learned

1. Multiple tabs using Power BI bookmarks. The bookmark features allowed me to create a tab-like visual to change the view based on what tab (button) is selected.
2. Creating new visuals in Power BI with **Deneb (Vega-Lite)** visual open the opportunity to create new stunning, customizable visual that are not available in Power BI. For example, the ROE, ROA, and Price-to-Boo Ratio KPIs used a new visual indicator created with Deneb visuals.
3. Using M Language `Web.Page` to execute JavaScript. This extends the capabilities of M Language opening doors for complex computations which are not available yet.
4. Calculating YoY Growth % with `SAMPEPERIODLASTYEAR()` time intelligence function.  
   _A code snippet showing how the assets growth YoY was calculated._

   ```sql
   Assets Growth % =
   VAR APreviousYear = CALCULATE(
               AVERAGE(revenues_fact[Assets]),
               SAMEPERIODLASTYEAR( 'Fiscal Year'[Date] )
       )
   RETURN
       DIVIDE(
           AVERAGE(revenues_fact[Assets]) - APreviousYear,
           ABS(APreviousYear)
       )
   ```

5. Creating a field measure to define each company;s color. This makes it easy to customize the color showed when you change the company. This can also be used to create dark-light theme.  
   *A code snippet showing how the company color field measure was defined.*

   ```sql
    Company Color =
        SWITCH(
            [Selected Company],
            "Zoom, Inc", "#0D6ABF",
            "Synopsys, Inc", "#893395"
        )
    ```

## ✅ Lessons Learned and Future Improvements

1. **Code repetition and non-modularity**: When defining formatting measures for the tooltip and card visuals in DAX, I found myself using the same expressions and statements but with different arguments. This makes the code more complex, error-prone and will be difficult to change in the future.  
   **Improvements:** Power BI **DAX user-defined functions (UDFs)** let you create reusable parameterized DAX logic into your models. It is in preview now and will learn it to improve this and any future Power BI project.
2. **Multiple YoY measures:** Year-over-year calculation was important for this project but I had to calulcate it for all the financial fields which ended up defining multiple YoY measures. More code and susceptible to error.  
   **Improvements:** With Power BI **Calculation groups**, I can define a single time intelligence calculation group and use for the various financial fields without defining additional measures.

---

## 🚀 How to SetUp This Project

There are two ways to setup this project.

1. Download the zip file from GitHub and extract.

   - Navigate to this [GitHub page](https://github.com/emma-fosu/Financial-Performance-Dashboard-Zoom-Inc.-Synopsys-Inc.-)
   - Click on the green `code` button at the top right.
   - Click on `Download ZIP`.
   - Extract the downloaded zip and open `financial performance.pbix`

2. Clone from GitHub. - Open your Command Prompt program and paste in `git clone https://github.com/emma-fosu/Financial-Performance-Dashboard-Zoom-Inc.-Synopsys-Inc.-.git`. - Press `Enter` to clone the repository. - Open the folder to access `financial performance.pbix`.  
   **NOTE**: You need to have Power BI installed locally to view this file.  
   To view the dashboard online, navigate to this [page](https://app.powerbi.com/groups/me/reports/19c45587-ee7c-4119-ad00-436c31009fa8/01bd393a8b71a60ca994?experience=power-bi&bookmarkGuid=36654765797e8b09abe8).

## 📊 Project Structure

    ```bash
    ├── images/                          # Images used for README.md
    ├── financial performance.pbix       # Financial Dashboard Power BI file
    └── README.md                        # Project documentation
    ```

## Challenges I faced

1. **Data source:** Initially, the proposal for the data source was to get all the available 10-K forms of both Zoom, Inc. and Synopsys, Inc. into a folder then use power to select the necessary sheets and transform the data. But, both companies' financial statements have different sheets layout and names, different financial name for the same thing, different value formats and different table structure.  
   **Solution**: SEC EDGAR (Electronic Data Gathering, Analysis, and Retrieval) APIs.  
   It is a web-based interface provided by the U.S. Securities and Exchange Commission (SEC) that allows developers and analysts to programmatically access public company filings and financial data submitted to the SEC.  
   The company-concept API was used and it returns all the XBRL disclosures from a single company (CIK) and concept (a taxonomy and tag) into a single JSON file, with a separate array of facts for each units on measure that the company has chosen to disclose (e.g. net profits reported in U.S. dollars and in Canadian dollars).
   `https://data.sec.gov/api/xbrl/companyconcept/CIK[Company's CIK Number]/[Taxonomy]/[Tag].json`.  
   The JSON structures are updated throughout the day.
2. **Tag Names Mismatch:** Because EDGAR allows companies to extend the service with their own tag names, there were some mismatches and errors when the same tag is used for both companies.  
   **Solution**: A tag dimension table was created to map different tage names to a single well-defined name. The tag table provided three alternatives for the same tag and in case the first tag fails, subsequent alternatives are used until there is a resolution.
   <img src="./images/tags.png" />
   _For example, the `CostOfRevenue` has another tag `CostofGoodAndServicesSold`. If `CostOfRevenue` is not a recognized tag name for the company, `CostofGoodAndServicesSold` is used instead. If it also fails, then no data will be collected for such tag._

3. **Testing for valid frames:** Every EDGAR XBRL disclosures has a `frame` field that referst to the snapshot of a company's financial data for a specific reporting period or time frame. Eg. a frame with the value `CY2023Q4I` means Calender Year 2023, Quarter 4, Instant value. For only 10-K Forms, the frame values are different for each tag. To retrieve the right frame, each has to be tested to the right format or pattern. To do this, a regular expression pattern matching is needed but unfortunately, M language does not support it.  
    **Solution**: A workaround was done by executing a JavaScript code with the `RegExp` Object. M language support executing JS code with the `Web.Page` function.

    ```fsharp
        let
            regex_test = (regex as text, str as text, optional options as text) as logical =>
            let
                optionsActual = if options = null then "" else options,
                script = "<script>
                            regex = '"&regex&"';
                            str = '"&str&"';
                            options = '"&optionsActual&"';
                            document.write(new RegExp(regex, options).test(str));
                        </script>",
                res = Web.Page(script){0}[Data]{0}[Children]{1}[Children]{0}[Text]
            in
                Logical.FromText(res)
        in regex_test
    ```

    _The code snippet of the `regex_test` function used to test for valid frames._  
4. **Extracting the year from the frames:** Similarly to testing for valid frames, the fiscal year for each tag was needed. Power Query M Language does not support extracting string with a regular expression pattern matching technique, so the same workaround for testing valid frames was used.  
**Solution**: A `regex_search` function that used JavaScript code to search for the substring matching the provided pattern.
    ```fsharp
    (regex as text, str as text, optional options as text) =>
    let
        optionsActual = if options = null then "" else options,
        script = "<script>
                    regex = '"&regex&"';
                    str = '"&str&"';
                    options = '"&optionsActual&"';
                    document.write(str.search(new RegExp(regex, options)));
                </script>",
        res = Web.Page(script){0}[Data]{0}[Children]{1}[Children]{0}[Text]
    in
        res
    ```

## 🔜 My Next Project

### Divvy Bike-Sharing Riding Pattern

My next project will focus on determing consumer pattern for a bike-sharing company. The marketing team wants to understand how the two subscribers (casual and member) use the service. The team has proposed that converting more casuals into members will increase revenue significantly. This next project will aim at stating the riding patterns of casual and members and how to convert casual to member.

## What to Learn Next

1. DAX user-defined functions (UDFs).
2. Visual Calulations.
3. Calculation groups.
4. DAX Time Intelligence functions.
5. Deneb Visual (Vega-Lite).

## 🙋 Contact Me

🔗 LinkedIn: [www.linkedin.com/in/emma-fosu](www.linkedin.com/in/emma-fosu)

🔗 Portfolio: []()

🔗 Email: [emmanuelfosu.duffour@gmail.com](mailto:emmanuelfosu.duffour@gmail.com)`
