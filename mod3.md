# Module 3: Cleaning the Real World

> **Big idea:** Real data is messy. Before you can analyze anything, you must clean it. Analysts often spend 60-80% of their time on this step, and for good reason: **if the input is wrong, the answer is wrong, no matter how fancy your analysis is.**

**Golden rule: Garbage In → Garbage Out (GIGO).**

---

## Table of Contents

1. Why data is messy
2. The sample dataset we'll use throughout
3. Lesson 29: Missing Data
4. Lesson 30: Duplicates
5. Lesson 31: Invalid Values
6. Lesson 32: Inconsistent Categories
7. Lesson 33: Incorrect Data Types
8. Lesson 34: Outliers
9. Lesson 35: Data Validation
10. Lesson 36: Data Transformation
11. The complete cleaning workflow
12. Common beginner mistakes
13. Quick reference cheat sheet
14. Practice exercises

---

## 1. Why is real-world data messy?

Data is created by **humans** (typing, filling forms), **machines** (sensors, apps, logs) and **systems** (merging databases). All three make mistakes:

| Source of mess | Example |
|---|---|
| Human typing | Someone types "Mumbay" instead of "Mumbai" |
| Optional form fields | Customer skips the "Age" box |
| Sensor failure | Thermometer reads -999 when it disconnects |
| Merging systems | System A stores dates as `12/03/2025`, System B as `2025-03-12` |
| Double submission | Customer clicks "Order" twice |
| Different standards | One team writes "Male/Female", another writes "M/F" |

**Why cleaning matters, a tiny story:**
A shop wants the average age of customers. Ages recorded: 25, 30, 28, 27, **250** (typo for 25), 26.
- Average with the typo = **64.3**
- Average after fixing = **27.2**

One bad value made the shop believe its customers are senior citizens. That is the cost of not cleaning.

---

## 2. The sample dataset used throughout

Imagine an online store's customer orders. Here is the **raw, messy** table:

| order_id | customer | age | city | gender | order_date | amount | payment |
|---|---|---|---|---|---|---|---|
| 101 | Ravi | 25 | Mumbai | Male | 2025-01-05 | 1200 | Card |
| 102 | Priya | *(blank)* | Delhi | female | 05/01/2025 | 800 | UPI |
| 103 | Arjun | 31 | Hyderabad | M | 2025-01-07 | 1500 | card |
| 102 | Priya | *(blank)* | Delhi | female | 05/01/2025 | 800 | UPI |
| 104 | Sneha | -4 | Bengaluru | F | 2025-01-09 | 950 | Cash |
| 105 | Kiran | 29 | Bangalore | Male | Jan 10, 2025 | 99999 | UPI |
| 106 | Meena | 250 | mumbai | Female | 2025-01-11 | "1,100" | CARD |

Look closely. This one tiny table has **every problem in this module**:
- Blank age → *missing data*
- Order 102 appears twice → *duplicate*
- Age -4 and 250 → *invalid values*
- Bengaluru / Bangalore, Mumbai / mumbai, card / Card / CARD, Male / M → *inconsistent categories*
- `"1,100"` is text, not a number → *incorrect data type*
- 99999 amount → *outlier*
- Three date formats → *transformation / standardization*

We'll fix everything step by step.

**Loading it in Python (pandas):**

```python
import pandas as pd
import numpy as np

df = pd.read_csv("orders.csv")
print(df.head())      # first 5 rows
print(df.info())      # column types + non-null counts
print(df.describe())  # min, max, mean for numbers
```

> **Habit #1:** Before cleaning anything, always *look* at your data: `head()`, `info()`, `describe()`, `shape`. You cannot fix what you haven't seen.

---

## 3. Lesson 29: Missing Data

### What is it?
Cells that should contain a value but are **empty**. Shown as blank, `NULL`, `NaN`, `N/A`, `-`, or `None`.

### Why does it happen?
- Customer skipped an optional field
- Data was never collected (old system didn't have that column)
- Sensor or upload failed
- Error while merging tables

### Three types of missingness (simple version)

| Type | Meaning | Example |
|---|---|---|
| **Random** | Missing for no pattern | A clerk forgot to type the age sometimes |
| **Systematic** | Missing because of a reason | Older customers refuse to share their age |
| **Structural** | Missing because it doesn't apply | "Spouse name" is blank for unmarried people |

Why it matters: if the reason is systematic, filling in the average can *hide* a real pattern.

### Step 1: Detect it

```python
df.isnull().sum()               # count missing per column
df.isnull().mean() * 100        # percentage missing per column
df[df["age"].isnull()]          # show the rows with missing age
```

Example output:
```
order_id     0
customer     0
age          2     ← problem
city         0
```
"2 out of 7 ages missing" = **28.6% missing.** That's a lot.

### Step 2: Choose a strategy

**Option A: Delete the rows (dropna)**
```python
df_clean = df.dropna(subset=["age"])
```
- ✅ Simple, no guessing
- ❌ You lose data. If 40% of rows have a gap, deleting throws away almost half your dataset
- **Use when:** very few rows (under ~5%) are missing and it looks random

**Option B: Delete the column**
- If a column is 80-90% empty, it has almost no useful info. Drop it.
```python
df = df.drop(columns=["fax_number"])
```

**Option C: Fill with a typical value (imputation)**

| Column type | Fill with | Why |
|---|---|---|
| Numbers, no outliers | **Mean** (average) | Balanced |
| Numbers, with outliers | **Median** (middle) | Not pulled by extremes |
| Categories | **Mode** (most common) | Most likely value |
| Time-ordered data | Previous value (forward fill) | Yesterday ≈ today |

```python
df["age"] = df["age"].fillna(df["age"].median())
df["city"] = df["city"].fillna(df["city"].mode()[0])
df["temperature"] = df["temperature"].ffill()
```

**Option D: Fill with a label like "Unknown"**
```python
df["payment"] = df["payment"].fillna("Unknown")
```
Best for categories when *not knowing* is itself informative.

**Option E: Leave it missing on purpose.** Sometimes blank is the truth (e.g., "date of return" for an order that hasn't been returned). Don't fill it.

### Worked example

Ages: 25, 30, *blank*, 28, *blank*, 26

- Mean of known = (25+30+28+26)/4 = **27.25**
- Median of known = **27**
- After median fill: 25, 30, **27**, 28, **27**, 26

### Decision guide

```
How much is missing?
 ├─ Under 5%  → delete rows (usually fine)
 ├─ 5–40%     → fill (median / mode / Unknown)
 └─ Over 60%  → consider dropping the column
Is blank meaningful? → keep it blank or label "Unknown"
```

### Common mistakes
- Filling with **0**: this says "age is zero," which is a fake value, not "unknown"
- Using the mean when outliers exist (one billionaire ruins the "average income")
- Not documenting what you did. Always note "Filled 2 missing ages with median 27."

---

## 4. Lesson 30: Duplicates

### What is it?
The **same record appearing more than once**.

### Why does it happen?
- User double-clicks "Submit"
- Data imported twice
- Two systems merged with overlapping records
- Copy-paste mistakes

### Why is it dangerous?
Duplicates **inflate everything**:

| | With duplicates | Correct |
|---|---|---|
| Orders counted | 7 | 6 |
| Total revenue | 6,349 (order 102 counted twice) | 5,549 |
| Customers | Priya seems to buy twice as much | Priya bought once |

### Two kinds of duplicates

**1. Exact duplicates:** every column identical.

| order_id | customer | amount |
|---|---|---|
| 102 | Priya | 800 |
| 102 | Priya | 800 |

**2. Partial duplicates:** same entity, slightly different data.

| customer_id | name | email |
|---|---|---|
| 55 | Rahul Sharma | rahul@mail.com |
| 55 | Rahul  Sharma | rahul@mail.com |
| 78 | Rahul Sharma | rahul@mail.com |

Same person entered three times: extra space in one, different ID in another. Partial duplicates are harder to catch.

### Detect

```python
df.duplicated().sum()                          # how many exact duplicate rows
df[df.duplicated(keep=False)]                  # show ALL copies (including the first)
df.duplicated(subset=["order_id"]).sum()       # duplicates based on order_id only
```

`keep=False` shows every copy so you can compare them. Without it, the first copy is hidden.

### Remove

```python
df = df.drop_duplicates()                              # exact duplicates
df = df.drop_duplicates(subset=["order_id"])           # keep first per order_id
df = df.drop_duplicates(subset=["order_id"], keep="last")  # keep the latest instead
```

### Deciding which copy to keep
- **Keep first:** when all copies are identical
- **Keep last:** when the later one is an *update* (e.g., a newer address)
- **Merge them:** when each copy has different useful info

### ⚠️ Not every repeat is a duplicate!
A customer who buys the same shirt twice on different days has two **legitimate** rows. Duplicates depend on **what makes a row unique**. Ask: *what column(s) identify one real-world event?* Usually an ID, or a combination such as `customer + date + product`.

### Fixing partial duplicates (clean first, then dedupe)

```python
df["name"] = df["name"].str.strip().str.lower()   # remove spaces, unify case
df["email"] = df["email"].str.strip().str.lower()
df = df.drop_duplicates(subset=["email"])
```

---

## 5. Lesson 31: Invalid Values

### What is it?
Values that are **present but impossible or nonsensical**. Different from missing (no value) and outliers (unusual but possible).

### Examples

| Column | Invalid value | Why invalid |
|---|---|---|
| age | -4, 250 | People can't be negative or 250 years old |
| percentage score | 145% | Max is 100 |
| email | "ravi@@mail" | Broken format |
| phone number | "12345" | Too short |
| order_date | 2031-05-01 | In the future |
| gender | "Banana" | Not a real option |
| PIN code | "ABCD12" | Should be 6 digits |
| quantity | 0 or -3 | Can't order zero/negative items |
| end_date | earlier than start_date | Time travel |

### Invalid vs Outlier: the key difference

| | Invalid | Outlier |
|---|---|---|
| Possible in real life? | **No** | **Yes**, but unusual |
| Example | Age = -4 | Salary = ₹5 crore |
| Action | Fix or remove | Investigate, then decide |

### Detect

```python
# 1. Look at min and max
df["age"].describe()

# 2. Rule-based filtering
df[(df["age"] < 0) | (df["age"] > 120)]

# 3. List distinct values for small-category columns
df["gender"].unique()

# 4. Pattern check for emails
df[~df["email"].str.contains(r"^[\w\.-]+@[\w\.-]+\.\w+$", na=False)]

# 5. Logical check
df[df["end_date"] < df["start_date"]]
```

### Fix: your options

| Option | When | Code |
|---|---|---|
| **Correct it** | You know the true value (250 → 25 obvious typo, confirmed with source) | `df.loc[df["age"]==250, "age"] = 25` |
| **Set to missing** | Value is wrong, truth unknown | `df.loc[df["age"]<0, "age"] = np.nan` then treat as missing |
| **Delete the row** | Row is unusable | `df = df[df["age"].between(0,120)]` |
| **Go back to source** | Data is important | Ask the data owner |

> **Never silently "guess" corrections.** If 250 could be 25 *or* 52, you don't know. Set it to missing instead.

### Worked example

Ages: 25, -4, 31, 250, 29

1. Rule: valid age = 0 to 120
2. Invalid found: -4, 250
3. Set them to `NaN` → 25, NaN, 31, NaN, 29
4. Fill using median (29) → 25, 29, 31, 29, 29

Notice how **invalid values become missing values, then use Lesson 29's tools.** The lessons connect.

---

## 6. Lesson 32: Inconsistent Categories

### What is it?
The **same thing written in different ways**, so the computer treats them as different things.

### Why it hurts
Computers are literal. `"Mumbai"`, `"mumbai"`, `"MUMBAI"`, `"Mumbai "` (trailing space) and `"Mumbay"` are five different categories to them.

Count of orders by city (raw):

| city | count |
|---|---|
| Mumbai | 40 |
| mumbai | 25 |
| Mumbai␣ | 10 |
| Bombay | 5 |

A manager reads this as four cities. Actually it's **one city with 80 orders**, and the biggest bar in your chart is understated by 50%.

### Common causes and their fixes

| Problem | Example | Fix |
|---|---|---|
| Upper/lower case | `Card`, `card`, `CARD` | `.str.lower()` or `.str.title()` |
| Extra spaces | `"Delhi "`, `" Delhi"` | `.str.strip()` |
| Spelling variants | `Bengaluru`, `Bangalore` | `.replace()` / mapping |
| Typos | `Mumbay` | mapping or fuzzy matching |
| Abbreviations | `M`, `F`, `Male`, `Female` | mapping dictionary |
| Punctuation | `U.S.A`, `USA`, `U S A` | remove punctuation, then map |
| Synonyms | `Yes`, `Y`, `True`, `1` | mapping |

### Step 1: See the mess

```python
df["city"].value_counts()
df["gender"].unique()
```
Always start with `value_counts()`. It's the single best tool for finding inconsistent categories.

### Step 2: Fix in layers

```python
# Layer 1: basic hygiene (fixes spaces and case)
df["city"] = df["city"].str.strip().str.title()

# Layer 2: map variants to one standard
city_map = {
    "Bangalore": "Bengaluru",
    "Bombay": "Mumbai",
    "Mumbay": "Mumbai"
}
df["city"] = df["city"].replace(city_map)

# Layer 3: standardize codes
gender_map = {"M": "Male", "F": "Female", "male": "Male", "female": "Female"}
df["gender"] = df["gender"].replace(gender_map)
```

### Result on our sample

| Before (city) | After |
|---|---|
| Mumbai, mumbai, Mumbay | Mumbai |
| Bengaluru, Bangalore | Bengaluru |
| Delhi | Delhi |
| Hyderabad | Hyderabad |

| Before (payment) | After |
|---|---|
| Card, card, CARD | Card |
| UPI | UPI |
| Cash | Cash |

### Step 3: Verify

```python
df["city"].value_counts()   # should now show clean, fewer categories
```

### Tips
- **Decide a standard first** (e.g., "Bengaluru" not "Bangalore"), then map everything to it.
- Keep a **mapping table** and reuse it for future data.
- For many typos, look up "fuzzy matching" (libraries like `rapidfuzz`). It groups look-alike text automatically, but always review results by eye.
- In Excel: `TRIM()`, `PROPER()`, `LOWER()`, and Find & Replace do the same job.

---

## 7. Lesson 33: Incorrect Data Types

### What is it?
A column stored as the **wrong kind of data**, e.g., numbers stored as text, dates stored as text.

### Why it matters
The computer decides what operations are allowed by the type.

| You want to do | Needs type | If it's actually text… |
|---|---|---|
| Add up amounts | number | `"100" + "200"` = `"100200"` (glued together!) |
| Find latest date | datetime | `"09/01/2025"` sorts before `"10/01/2024"` alphabetically |
| Calculate average | number | Error or nonsense |
| Sort numbers | number | `"100" < "20"` because "1" comes before "2" |

### Common data types

| Type | Meaning | Example |
|---|---|---|
| `int` | whole numbers | 25, 100 |
| `float` | decimals | 99.5, 3.14 |
| `str` / object | text | "Mumbai" |
| `bool` | True/False | True |
| `datetime` | dates and times | 2025-01-05 |
| `category` | fixed set of labels | "Card", "UPI", "Cash" |

### Classic problems and fixes

**Problem 1: Numbers stored as text because of symbols**

Values: `"₹1,100"`, `"$45.50"`, `"20%"`

```python
df["amount"] = (df["amount"]
                .astype(str)
                .str.replace("₹", "", regex=False)
                .str.replace(",", "", regex=False)
                .str.strip())
df["amount"] = pd.to_numeric(df["amount"], errors="coerce")
```
`errors="coerce"` turns anything unconvertible (like "N/A") into `NaN` instead of crashing. Then use Lesson 29 to handle those.

**Problem 2: Dates stored as text**

```python
df["order_date"] = pd.to_datetime(df["order_date"], errors="coerce")
```
Once it's a real date you can do `df["order_date"].dt.month`, `.dt.day_name()`, and find differences between dates.

⚠️ **Day/month confusion:** Is `05/01/2025` 5 January or 1 May? Specify it:
```python
pd.to_datetime(df["order_date"], format="%d/%m/%Y")
```

**Problem 3: Codes stored as numbers**

PIN code `011001` (Delhi) read as number becomes `11001`, so the leading zero is lost. Phone numbers, PIN codes, and IDs are **labels, not quantities**. Store them as text.
```python
df = pd.read_csv("file.csv", dtype={"pin_code": str, "phone": str})
```
Rule of thumb: *"Would it make sense to add or average this?"* If not, it's text.

**Problem 4: True/False as text**
```python
df["is_member"] = df["is_member"].map({"Yes": True, "No": False})
```

**Problem 5: Repeating labels → category type** (saves memory, helps clarity)
```python
df["payment"] = df["payment"].astype("category")
```

### Detect problems
```python
df.dtypes          # what pandas thinks each column is
df.info()
```
🚩 **Red flag:** a column that should be a number shows as `object`. That almost always means hidden text like symbols, spaces or "N/A" inside.

### Find the culprit rows
```python
df[pd.to_numeric(df["amount"], errors="coerce").isnull() & df["amount"].notnull()]
```
This shows rows that have a value but can't be converted, i.e. the ones causing trouble.

---

## 8. Lesson 34: Outliers

### What is it?
A value **far away from the rest** of the data. Unusual, but not necessarily wrong.

### Intuition
Salaries of 5 employees: ₹30k, ₹32k, ₹35k, ₹31k, **₹5,00,000**.
- Mean = ₹1,25,600 (misleading: nobody earns anywhere near this)
- Median = ₹32,000 (representative)

The CEO's salary is a real number, but it distorts the average. That's an outlier.

### Why do outliers exist? Three reasons

| Reason | Example | Action |
|---|---|---|
| **Error** | Typed 99999 instead of 999 | Fix or remove |
| **Rare but real** | A customer really bought ₹2 lakh of goods | **Keep**, maybe analyze separately |
| **Different population** | Bulk/business buyers mixed with normal shoppers | Split into a separate group |

> **The most important rule: never delete an outlier just because it's inconvenient.** Ask first: *is it an error, or is it real?* Real outliers often contain the most interesting insights (fraud, VIP customers, rare events).

### How to detect outliers

**Method 1: Just look (visual)**
```python
df["amount"].plot(kind="box")      # dots outside the box = outliers
df["amount"].plot(kind="hist")
```
A **box plot** shows the middle 50% of data as a box; dots beyond the whiskers are potential outliers.

**Method 2: The IQR rule (most popular)**

Steps:
1. Q1 = 25th percentile, Q3 = 75th percentile
2. IQR = Q3 − Q1 (the spread of the middle half)
3. Lower fence = Q1 − 1.5 × IQR
4. Upper fence = Q3 + 1.5 × IQR
5. Anything outside the fences is an outlier

**Worked example** with amounts: 800, 900, 950, 1000, 1100, 1200, 1500, 99999

- Q1 ≈ 937.5, Q3 ≈ 1275
- IQR = 1275 − 937.5 = 337.5
- Lower fence = 937.5 − 506.25 = **431.25**
- Upper fence = 1275 + 506.25 = **1781.25**
- 99999 > 1781.25 → **outlier** ✅ (1500 is fine)

```python
Q1 = df["amount"].quantile(0.25)
Q3 = df["amount"].quantile(0.75)
IQR = Q3 - Q1
lower = Q1 - 1.5 * IQR
upper = Q3 + 1.5 * IQR
outliers = df[(df["amount"] < lower) | (df["amount"] > upper)]
```

**Method 3: Z-score (for roughly bell-shaped data)**

Z-score = how many standard deviations a value is from the mean. Typically |z| > 3 is an outlier.
```python
z = (df["amount"] - df["amount"].mean()) / df["amount"].std()
outliers = df[abs(z) > 3]
```
⚠️ Extreme outliers inflate the mean and standard deviation, so they can hide themselves. IQR is more robust.

**Method 4: Business logic.** "No order in our shop exceeds ₹50,000." Domain knowledge beats formulas.

### How to handle outliers

| Approach | When | Code |
|---|---|---|
| **Keep** | Real and relevant | (do nothing, note it) |
| **Remove** | Clearly an error | `df = df[df["amount"] <= upper]` |
| **Cap / Winsorize** (limit to fence) | Want to keep row, reduce influence | `df["amount"] = df["amount"].clip(lower, upper)` |
| **Transform** (e.g., log) | Heavy right skew | `np.log1p(df["amount"])` |
| **Analyze separately** | Different group | Split into two datasets |
| **Use robust stats** | Can't decide | Report median instead of mean |

### Mini case study
An app shows average session time = 47 min. After checking, 3 sessions lasted 20+ hours because users left the app open overnight. Excluding those (documented as "idle sessions"), the average is 6 min. The business decisions change completely.

---

## 9. Lesson 35: Data Validation

### What is it?
**Setting rules that data must obey, and automatically checking them.** Lessons 29-34 are about fixing problems. Validation is about **catching them (and preventing them) systematically.**

Think of validation as the **security check at an airport**: rules decide what's allowed through.

### Two moments to validate

| When | Name | Example |
|---|---|---|
| **At entry** | Preventive | A web form refuses an age of 250 |
| **After collection** | Detective | You run checks on a downloaded file |

### Types of validation rules

| Rule type | Question it asks | Example |
|---|---|---|
| **Data type check** | Is it the right type? | `age` must be a number |
| **Range check** | Within min-max? | `age` between 0 and 120 |
| **Format check** | Matches a pattern? | Email has `@` and `.`; PIN is 6 digits |
| **List (allowed values)** | In an approved list? | `payment` ∈ {Card, UPI, Cash} |
| **Required (not null)** | Is it filled? | `order_id` can't be blank |
| **Uniqueness** | No repeats? | `order_id` must be unique |
| **Consistency / cross-field** | Do fields agree? | `delivery_date ≥ order_date` |
| **Referential** | Does the linked record exist? | Every `customer_id` in orders exists in customers table |
| **Length check** | Right length? | Phone = 10 digits |

### Practical validation in Python

```python
# Rule-by-rule checks: each returns rows that BREAK the rule
errors = {}

errors["age_range"]     = df[~df["age"].between(0, 120)]
errors["payment_list"]  = df[~df["payment"].isin(["Card", "UPI", "Cash"])]
errors["id_missing"]    = df[df["order_id"].isnull()]
errors["id_duplicate"]  = df[df["order_id"].duplicated(keep=False)]
errors["date_logic"]    = df[df["delivery_date"] < df["order_date"]]
errors["pin_format"]    = df[~df["pin"].astype(str).str.match(r"^\d{6}$")]

for rule, bad_rows in errors.items():
    print(f"{rule}: {len(bad_rows)} problem rows")
```

Sample output:
```
age_range: 2 problem rows
payment_list: 0 problem rows
id_missing: 0 problem rows
id_duplicate: 2 problem rows
```

### A simple validation report

| Rule | Rows failing | Status |
|---|---|---|
| Age between 0-120 | 2 | ❌ Fail |
| Payment in allowed list | 0 | ✅ Pass |
| Order ID unique | 2 | ❌ Fail |
| Amount > 0 | 0 | ✅ Pass |

Run this **before** cleaning (to know what's wrong) and **after** (to prove it's fixed). It becomes your "proof of quality."

### Using `assert` as a safety net
```python
assert df["order_id"].is_unique, "Duplicate order IDs found!"
assert df["amount"].min() > 0, "Non-positive amounts found!"
```
If a rule fails, the script stops loudly instead of silently producing wrong results.

### Validation in Excel / Google Sheets
- **Data → Data validation:** dropdown lists, number ranges, date ranges, custom formulas
- **Conditional formatting:** highlight cells breaking the rules in red

### Validation vs Cleaning

| Cleaning | Validation |
|---|---|
| **Fixes** bad data | **Checks** data against rules |
| Reactive | Reactive *and* preventive |
| "Repair the road" | "Put up signs so no one damages it" |

---

## 10. Lesson 36: Data Transformation

### What is it?
**Changing the shape, scale, or format of data so it is ready for analysis.** Cleaning removes errors; transformation *reshapes* the data to be useful.

### Common transformations

#### 10.1 Standardizing formats
Convert everything to one format.

| Before (dates) | After |
|---|---|
| 2025-01-05, 05/01/2025, Jan 10, 2025 | 2025-01-05, 2025-01-05, 2025-01-10 |

```python
df["order_date"] = pd.to_datetime(df["order_date"], format="mixed", dayfirst=True)
```

#### 10.2 Creating new columns (feature engineering)

Derive new, useful info from existing columns.

```python
df["order_month"]   = df["order_date"].dt.month
df["day_of_week"]   = df["order_date"].dt.day_name()
df["is_weekend"]    = df["order_date"].dt.dayofweek >= 5
df["total_price"]   = df["quantity"] * df["unit_price"]
df["full_name"]     = df["first_name"] + " " + df["last_name"]
```

#### 10.3 Binning (grouping numbers into buckets)

Turn ages into groups so patterns become clear.

```python
df["age_group"] = pd.cut(df["age"],
                         bins=[0, 18, 30, 45, 60, 120],
                         labels=["Teen", "Young Adult", "Adult", "Middle Age", "Senior"])
```

| age | age_group |
|---|---|
| 16 | Teen |
| 25 | Young Adult |
| 41 | Adult |
| 67 | Senior |

#### 10.4 Scaling / normalization

When columns have very different ranges (age: 0-100, salary: 0-1,00,000), big numbers dominate. Scaling brings them to the same range.

**Min-Max scaling** (squeezes into 0 to 1):
```
scaled = (x − min) / (max − min)
```
Example: salaries 20k, 50k, 80k. min=20k, max=80k.
- 20k → 0
- 50k → (50−20)/(80−20) = **0.5**
- 80k → 1

**Standardization (z-score):** `(x − mean) / std` → centered at 0.

```python
df["amount_scaled"] = (df["amount"] - df["amount"].min()) / (df["amount"].max() - df["amount"].min())
```
Scaling matters most for machine learning and for comparing different-unit columns.

#### 10.5 Log transformation
For heavily skewed data (like incomes, house prices) where a few huge values stretch everything.
```python
df["log_income"] = np.log1p(df["income"])
```
Incomes 1,000 / 10,000 / 1,00,000 / 10,00,000 become roughly 6.9 / 9.2 / 11.5 / 13.8, which are much closer together.

#### 10.6 Encoding categories as numbers

Many algorithms need numbers.

**Label encoding** (ordered categories):
```python
df["size_num"] = df["size"].map({"Small": 1, "Medium": 2, "Large": 3})
```

**One-hot encoding** (unordered categories, one yes/no column per category):
```python
pd.get_dummies(df["payment"], prefix="pay")
```

| payment | → | pay_Card | pay_UPI | pay_Cash |
|---|---|---|---|---|
| Card | | 1 | 0 | 0 |
| UPI | | 0 | 1 | 0 |
| Cash | | 0 | 0 | 1 |

Don't label-encode unordered categories as 1, 2, 3, because the computer will think UPI (2) is "twice" Card (1).

#### 10.7 Aggregation (summarizing)
Rolling detail up.
```python
df.groupby("city")["amount"].sum()                       # revenue per city
df.groupby(["city", "payment"])["amount"].agg(["count", "mean"])
```

#### 10.8 Reshaping: wide ↔ long

**Wide:**

| product | Jan | Feb | Mar |
|---|---|---|---|
| A | 100 | 120 | 90 |

**Long:**

| product | month | sales |
|---|---|---|
| A | Jan | 100 |
| A | Feb | 120 |
| A | Mar | 90 |

```python
long_df = df.melt(id_vars="product", var_name="month", value_name="sales")   # wide → long
wide_df = long_df.pivot(index="product", columns="month", values="sales")    # long → wide
```
Long format is easier for charting and grouping; wide is easier for humans to read.

#### 10.9 Splitting and combining columns
```python
df[["first", "last"]] = df["full_name"].str.split(" ", n=1, expand=True)
df["address"] = df["street"] + ", " + df["city"]
```

#### 10.10 Merging / joining tables
```python
merged = orders.merge(customers, on="customer_id", how="left")
```
Combines info from separate tables (like VLOOKUP in Excel).

#### 10.11 Filtering and sorting
```python
recent = df[df["order_date"] >= "2025-01-01"]
df = df.sort_values("amount", ascending=False)
```

### Cheat table: which transformation when?

| Situation | Transformation |
|---|---|
| Dates in mixed formats | Standardize with `to_datetime` |
| Need month/weekday trends | Extract from date |
| Too many distinct ages | Binning |
| Columns on very different scales | Min-max / z-score scaling |
| Heavily skewed numbers | Log transform |
| Text categories for a model | One-hot / label encode |
| Data too detailed | Aggregate |
| Chart tool wants one row per observation | Wide → long |

---

## 11. The complete cleaning workflow

Follow this order every time:

```
1. INSPECT      → head(), info(), describe(), shape
2. COPY         → work on df.copy(); never touch the raw file
3. FIX TYPES    → numbers as numbers, dates as dates
4. STANDARDIZE  → strip spaces, fix case, unify categories
5. DUPLICATES   → detect and remove
6. INVALID      → apply range/format rules → set to NaN
7. MISSING      → drop / fill / label
8. OUTLIERS     → detect, investigate, decide
9. TRANSFORM    → new columns, scaling, encoding
10. VALIDATE    → rerun rules, confirm all pass
11. DOCUMENT    → write down every change
12. SAVE        → export clean file: df.to_csv("orders_clean.csv", index=False)
```

**Why this order?** Fix types first (you can't check ranges on text). Standardize categories *before* removing duplicates (so "Priya " and "priya" are seen as the same). Handle invalid values *before* missing values (invalid ones often become missing). Validate last to prove it worked.

### The full pipeline on our sample data

```python
import pandas as pd, numpy as np

df = pd.read_csv("orders.csv")
df = df.copy()

# Types
df["amount"] = pd.to_numeric(df["amount"].astype(str).str.replace(",", ""), errors="coerce")
df["order_date"] = pd.to_datetime(df["order_date"], format="mixed", dayfirst=True, errors="coerce")

# Standardize categories
df["city"] = df["city"].str.strip().str.title().replace({"Bangalore": "Bengaluru"})
df["payment"] = df["payment"].str.strip().str.title()
df["gender"] = df["gender"].str.strip().str.title().replace({"M": "Male", "F": "Female"})

# Duplicates
df = df.drop_duplicates(subset=["order_id"])

# Invalid → missing
df.loc[~df["age"].between(0, 120), "age"] = np.nan

# Missing
df["age"] = df["age"].fillna(df["age"].median())

# Outliers (flag, don't blindly delete)
Q1, Q3 = df["amount"].quantile([0.25, 0.75])
upper = Q3 + 1.5 * (Q3 - Q1)
df["amount_outlier"] = df["amount"] > upper

# Validate
assert df["order_id"].is_unique
assert df["age"].between(0, 120).all()

df.to_csv("orders_clean.csv", index=False)
```

### Before → After

| order_id | customer | age | city | gender | order_date | amount | payment |
|---|---|---|---|---|---|---|---|
| 101 | Ravi | 25 | Mumbai | Male | 2025-01-05 | 1200 | Card |
| 102 | Priya | 29 | Delhi | Female | 2025-01-05 | 800 | Upi |
| 103 | Arjun | 31 | Hyderabad | Male | 2025-01-07 | 1500 | Card |
| 104 | Sneha | 29 | Bengaluru | Female | 2025-01-09 | 950 | Cash |
| 105 | Kiran | 29 | Bengaluru | Male | 2025-01-10 | 99999 | Upi |
| 106 | Meena | 29 | Mumbai | Female | 2025-01-11 | 1100 | Card |

(Note: `.str.title()` turned "UPI" into "Upi". A good reminder to **always check results**, and fix with a final `.replace({"Upi": "UPI"})`.)

---

## 12. Common beginner mistakes

1. **Cleaning without looking first.** Always inspect.
2. **Overwriting the raw data.** Keep the original file untouched.
3. **Filling missing values with 0.** Creates fake data.
4. **Deleting outliers automatically.** They may be your most important rows.
5. **Deleting too many rows.** Check `df.shape` before and after every step.
6. **Assuming repeated rows are duplicates.** Define what makes a row unique.
7. **Ignoring day/month order in dates.**
8. **Treating IDs and PIN codes as numbers.**
9. **Not documenting changes.** Future-you (and your team) won't remember.
10. **Skipping validation at the end.**

> **Habit #2:** After each cleaning step, print the row count and re-check. Numbers you can't explain = bugs you haven't found.

---

## 13. Quick reference cheat sheet

| Task | pandas code |
|---|---|
| Missing count | `df.isnull().sum()` |
| Drop missing rows | `df.dropna(subset=["col"])` |
| Fill with median | `df["col"].fillna(df["col"].median())` |
| Fill with mode | `df["col"].fillna(df["col"].mode()[0])` |
| Count duplicates | `df.duplicated().sum()` |
| Remove duplicates | `df.drop_duplicates(subset=["id"])` |
| Range check | `df[~df["age"].between(0,120)]` |
| Distinct categories | `df["col"].value_counts()` |
| Trim + case | `df["col"].str.strip().str.title()` |
| Replace values | `df["col"].replace({"old":"new"})` |
| To number | `pd.to_numeric(df["col"], errors="coerce")` |
| To date | `pd.to_datetime(df["col"], errors="coerce")` |
| Change type | `df["col"].astype("category")` |
| IQR outliers | `Q1 - 1.5*IQR` / `Q3 + 1.5*IQR` |
| Cap values | `df["col"].clip(lower, upper)` |
| Binning | `pd.cut(df["age"], bins, labels)` |
| One-hot | `pd.get_dummies(df["col"])` |
| Group summary | `df.groupby("col")["x"].sum()` |
| Wide → long | `df.melt(...)` |
| Join tables | `df1.merge(df2, on="key")` |

**Excel / Sheets equivalents:** `TRIM`, `PROPER`, `LOWER`, `VALUE`, `DATEVALUE`, Remove Duplicates, Find & Replace, `IF`, `VLOOKUP/XLOOKUP`, Data Validation, Conditional Formatting, Pivot Tables.

---

## 14. Practice exercises

**Exercise 1: Missing data.** A column has the values `[10, 12, NaN, 11, NaN, 200]`. Compute mean and median (ignoring NaN). Which would you use to fill the blanks and why?
*Hint:* the 200 is an outlier → median.

**Exercise 2: Duplicates.** Which of these are true duplicates?
| id | name | date |
|---|---|---|
| 1 | Asha | Jan 3 |
| 1 | Asha | Jan 3 |
| 2 | Asha | Jan 9 |
*Answer:* Rows 1 and 2. Row 3 is a different order by the same person.

**Exercise 3: Invalid vs outlier.** Classify each: (a) age = -3 (b) salary = ₹8 crore (c) score = 130/100 (d) marathon time = 2 minutes.
*Answer:* (a) invalid (b) outlier (c) invalid (d) invalid, as it's physically impossible.

**Exercise 4: Categories.** Standardize: `["Delhi", "delhi ", "DELHI", "New Delhi", "Dilli"]` into one label. Write the code.

**Exercise 5: Data types.** A column of prices reads `["₹1,200", "₹950", "N/A"]`. Write code to turn it into numbers. What happens to "N/A"?

**Exercise 6: IQR.** Data: `[5, 7, 8, 9, 10, 11, 12, 60]`. Find Q1, Q3, IQR, the fences, and any outliers.
*Answer:* Q1 = 7.75, Q3 = 11.25, IQR = 3.5 → fences 2.5 and 16.5 → **60** is an outlier.

**Exercise 7: Validation.** Write 5 validation rules for a *student registration* table (`student_id`, `email`, `age`, `course`, `join_date`).

**Exercise 8: Transformation.** Given `birth_date`, create an `age` column and an `age_group` column.

**Exercise 9: Full project.** Take any messy CSV (Kaggle "Titanic" is a great start: it has missing ages, cabin data, and mixed categories). Apply the 12-step workflow and write a short cleaning report: *what was wrong, what you did, and how many rows changed.*

---

## Key takeaways

- **Clean data = trustworthy insights.**
- **Missing:** delete, fill, label, or leave. Choose based on how much is missing and why.
- **Duplicates:** decide what makes a row unique, then remove extras.
- **Invalid:** impossible values → correct, set to missing, or remove.
- **Inconsistent categories:** strip, case, map to one standard.
- **Data types:** numbers as numbers, dates as dates, IDs as text.
- **Outliers:** investigate before acting. They can be errors *or* discoveries.
- **Validation:** write rules, run checks before and after cleaning.
- **Transformation:** reshape, scale, encode, and create new columns so the data is ready for analysis.
- **Always:** keep raw data safe, work step by step, document everything.