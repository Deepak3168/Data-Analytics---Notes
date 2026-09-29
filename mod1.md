# Module 1: Getting Data

> **Goal of this module:** By the end, you should be able to look at any data source (a file, a website API, a database, a public dataset), understand what shape the data is in, and write Python code to bring it onto your computer, ready to analyze.

Every data analysis project starts with the same question: **"Where is the data, and how do I get it?"** You can't clean, explore, or visualize what you don't have. This module is about that very first step.

---

## Table of Contents

1. [What is Data?](#1-what-is-data)
2. [Structured vs Semi-Structured vs Unstructured Data](#2-structured-vs-semi-structured-vs-unstructured-data)
3. [CSV, JSON, and Excel](#3-csv-json-and-excel)
4. [APIs as Data Sources](#4-apis-as-data-sources)
5. [Databases as Data Sources](#5-databases-as-data-sources)
6. [Downloading Real Datasets](#6-downloading-real-datasets)
7. [Python for Data Acquisition](#7-python-for-data-acquisition)
   - 7.1 HTTP Requests
   - 7.2 Working with APIs in Python
   - 7.3 Working with JSON in Python
   - 7.4 Working with Files in Python
   - 7.5 Basic Automation
8. [Mini Project: Build a Weather Data Collector](#8-mini-project-build-a-weather-data-collector)
9. [Common Mistakes and How to Avoid Them](#9-common-mistakes-and-how-to-avoid-them)
10. [Practice Exercises](#10-practice-exercises)
11. [Glossary](#11-glossary)
12. [Cheat Sheet](#12-cheat-sheet)

---

## Setup (do this once)

You'll need Python 3.9+ and a few libraries. Open a terminal and run:

```bash
pip install requests pandas openpyxl sqlalchemy
```

- `requests` → talk to websites and APIs
- `pandas` → work with tables of data (the most important library in this course)
- `openpyxl` → lets pandas read/write Excel `.xlsx` files
- `sqlalchemy` → connects Python to databases

Everything else used here (`json`, `csv`, `sqlite3`, `pathlib`, `time`, `logging`) ships with Python itself.

---

# 1. What is Data?

## 1.1 The plain-English definition

**Data is recorded facts about the world.**

That's it. A fact becomes data when someone *writes it down* (on paper, in a spreadsheet, in a computer).

- "It is 32°C in Hyderabad right now" → a fact.
- Writing `32` in a column named `temperature_c` next to `Hyderabad` → **data**.

Data by itself isn't very useful. It's raw material, like flour, eggs, and sugar before you bake a cake.

## 1.2 Data → Information → Insight

People mix these words up. Here's the difference:

| Level | What it is | Example |
|---|---|---|
| **Data** | Raw recorded facts | `72, 80, 65, 91, 55` |
| **Information** | Data with context and some processing | "The 5 students' test scores average 72.6" |
| **Insight** | A conclusion that helps you decide something | "Most of the class is passing, but one student (55) needs help" |

Data analytics is the craft of walking up this staircase: **data → information → insight → decision**.

## 1.3 Rows, columns, records, fields

Most data you'll work with looks like a table. Let's learn the vocabulary using a tiny example:

| customer_id | name | age | city | purchased |
|---|---|---|---|---|
| 1 | Asha | 28 | Hyderabad | True |
| 2 | Ravi | 35 | Bengaluru | False |
| 3 | Meena | 42 | Chennai | True |

- **Row** (also called a **record** or **observation**): one "thing". Row 1 is *one customer*, Asha.
- **Column** (also called a **field**, **variable**, or **feature**): one *property* of those things. The `age` column holds the age of every customer.
- **Cell**: one single value, e.g., `28`.
- **Header**: the top row of names (`customer_id`, `name`, ...).

> **Mental model:** Rows = *who/what*. Columns = *what we know about them*.

## 1.4 Types of values (data types)

Every column holds values of a certain **type**. The type decides what you're allowed to do with the values.

| Type | Meaning | Examples | What you can do |
|---|---|---|---|
| **Integer** | Whole numbers | `28`, `-5`, `1000` | add, subtract, average |
| **Float** | Decimal numbers | `3.14`, `72.6` | add, subtract, average |
| **String / Text** | Characters | `"Asha"`, `"Hyderabad"` | search, join, split (can't average) |
| **Boolean** | True or False | `True`, `False` | filter, count |
| **Date / Datetime** | A point in time | `2026-09-29`, `2026-09-29 14:30:00` | sort by time, find gaps, group by month |
| **Null / Missing** | No value recorded | `NaN`, `None`, empty cell | must be handled explicitly |

### A common beginner trap: numbers stored as text

A phone number like `9876543210` or a PIN code like `500001` *looks* like a number, but you never do math on it. It should be text. Meanwhile, a price stored as `"₹1,200"` (with symbol and comma) is text that *should* be a number after cleaning. Always check whether a column's type matches its meaning.

## 1.5 Categories of data (a second way to classify columns)

Besides types, analysts think about columns like this:

- **Numerical**
  - *Continuous*: can take any value in a range (height, temperature, price).
  - *Discrete*: countable whole numbers (number of children, number of orders).
- **Categorical**
  - *Nominal*: categories with no order (city, color, payment method).
  - *Ordinal*: categories with an order (small/medium/large, rating 1-5 stars).
- **Time-based**: dates and timestamps.

You'll use this later to decide which chart or statistic is appropriate. (Example: you can average a continuous number, but averaging "city" makes no sense.)

## 1.6 Where does data come from?

Data is created whenever something is recorded. Common origins:

- **Transactions**: every purchase, payment, booking.
- **Sensors / IoT**: temperature sensors, GPS, smartwatches.
- **Logs**: what users clicked on a website or app.
- **Surveys and forms**: people fill them in.
- **Public records**: census, government open data.
- **Social media / web**: posts, reviews, comments.
- **Other people's systems**: partners and vendors share data via files or APIs.

## 1.7 The data journey (where "Getting Data" fits)

```
Get Data  →  Clean  →  Explore  →  Visualize  →  Model / Decide  →  Communicate
   ▲
 We are here (Module 1)
```

Garbage in, garbage out: if you get the wrong data or get it incorrectly, everything after is affected. That's why we give acquisition its own module.

---

# 2. Structured vs Semi-Structured vs Unstructured Data

The single most useful thing to know about a piece of data before working with it is: **how organized is it?**

## 2.1 Structured data

**Definition:** Data organized into a fixed grid of rows and columns, where every row has the same columns and each column has a known type.

**Analogy: a school attendance register.** Every student is a line. Every line has the same boxes: roll number, name, present/absent. There's no surprise.

**Examples:**
- Excel spreadsheet
- CSV file
- Tables in a database (MySQL, PostgreSQL)
- Bank statement

**Strengths:**
- Easy to search, filter, sort, and calculate.
- Works directly with SQL and pandas.

**Weakness:**
- Rigid. If you want to store something that doesn't fit the columns, you must redesign the table.

## 2.2 Semi-structured data

**Definition:** Data that has *labels* (so it isn't random) but *doesn't follow a rigid table shape*. Records can have different fields, and fields can contain other records inside them (nesting).

**Analogy: a job application form with optional sections.** Everyone has a name and email. Some people fill in "Certifications", some don't. Some list three previous jobs, some list none.

**Example (JSON):**

```json
[
  {
    "name": "Asha",
    "email": "asha@example.com",
    "skills": ["Python", "SQL"]
  },
  {
    "name": "Ravi",
    "email": "ravi@example.com",
    "certifications": ["AWS Cloud Practitioner"],
    "address": { "city": "Bengaluru", "pin": "560001" }
  }
]
```

Notice: Asha has `skills` but no `address`. Ravi has `address` (with nested `city` and `pin`) but no `skills`. The shape varies, yet every value has a **label** (a *key*).

**Other examples:** JSON, XML, HTML pages, YAML, most API responses, log files with key=value pairs, MongoDB documents.

**Strengths:**
- Flexible. Perfect when data varies from record to record.
- The standard language of the web.

**Weakness:**
- You usually must **flatten** it into a table before analysis.

## 2.3 Unstructured data

**Definition:** Data with *no predefined model or fixed fields*. It's just raw content.

**Analogy: a pile of loose papers on a desk** photos, letters, receipts, sticky notes. There's information in there, but no filing system.

**Examples:**
- Free text: emails, reviews, chat messages, PDFs
- Images and scanned documents
- Audio and video
- Social media posts

**Strengths:**
- Rich. Contains the most real-world information (estimates commonly say the majority of all data is unstructured).

**Weakness:**
- You need extra techniques to extract structure: OCR for images/scans, speech-to-text for audio, NLP (natural language processing) or LLMs for text.

## 2.4 Side-by-side comparison

| | Structured | Semi-structured | Unstructured |
|---|---|---|---|
| **Shape** | Fixed rows & columns | Labeled but flexible, often nested | No fixed shape |
| **Analogy** | Attendance register | Job application form | Pile of loose papers |
| **Examples** | CSV, Excel, SQL tables | JSON, XML, API responses | Emails, images, audio, PDFs |
| **Easy to analyze?** | Very | Moderate (needs flattening) | Hard (needs extraction) |
| **Typical tools** | SQL, pandas | `json`, pandas `json_normalize` | OCR, NLP, LLMs, computer vision |

## 2.5 The real world is a mix

A single business uses all three:

- Sales table in a database (**structured**)
- Product catalog fetched from a partner's API as JSON (**semi-structured**)
- Customer support emails and call recordings (**unstructured**)

A big part of an analyst's job is **converting semi- and unstructured data into structured tables** so that the tools of analysis can be applied.

## 2.6 Quick self-test

Decide for each: structured, semi, or unstructured?

1. An Excel sheet of monthly sales → **Structured**
2. A tweet → **Unstructured** (text; though the tweet *object* returned by Twitter/X's API is semi-structured JSON containing the text)
3. The response from a weather API → **Semi-structured**
4. A photo of a receipt → **Unstructured**
5. A `customers` table in PostgreSQL → **Structured**

---

# 3. CSV, JSON, and Excel

These three file formats will show up in almost every project. Learn what each looks like, what it's good for, and where it can bite you.

## 3.1 CSV (Comma-Separated Values)

### What it is
A **plain text file** where each line is a row and values within the line are separated by commas. That's the whole format.

### What it looks like
Open a `.csv` in Notepad and you'll see:

```csv
customer_id,name,age,city,purchased
1,Asha,28,Hyderabad,True
2,Ravi,35,Bengaluru,False
3,Meena,42,Chennai,True
```

- **Line 1 = header** (column names)
- **Each next line = one row**
- **Commas** separate the columns

### Why it's popular
- Tiny and simple.
- Opens in anything: Excel, Google Sheets, Python, R, databases.
- Human-readable.
- Great for sharing big datasets.

### Things that go wrong (the gotchas)

1. **Values that contain commas.** What if a city is `"Visakhapatnam, AP"`? The comma would be mistaken for a separator. The fix: wrap in double quotes.
   ```csv
   1,Asha,"Visakhapatnam, AP"
   ```
2. **Different separators.** Some CSVs use semicolons (`;`), tabs (`\t`, called TSV), or pipes (`|`). In Europe, semicolons are common because commas are used as decimal marks.
3. **Encoding.** Text must be turned into bytes using some encoding. Most files are UTF-8, but some (especially from older Excel) are `latin-1` or `cp1252`. Wrong encoding → weird characters like `Ã©` instead of `é`.
4. **No types.** CSV has no idea what's a number or date; everything is text until your tool guesses.
5. **Missing values** can appear as empty, `NA`, `NULL`, `-`, or `N/A`.

### Reading a CSV in Python

**With pandas (recommended):**

```python
import pandas as pd

df = pd.read_csv("customers.csv")
print(df.head())      # first 5 rows
print(df.shape)       # (rows, columns)
print(df.dtypes)      # data type of each column
```

**Common options:**

```python
df = pd.read_csv(
    "customers.csv",
    sep=";",                  # if separator isn't a comma
    encoding="latin-1",       # if you see strange characters
    na_values=["NA", "-", ""],# treat these as missing
    parse_dates=["signup_date"],  # convert to datetime
)
```

**With Python's built-in `csv` module** (useful to understand what's happening):

```python
import csv

with open("customers.csv", newline="", encoding="utf-8") as f:
    reader = csv.DictReader(f)      # each row becomes a dictionary
    for row in reader:
        print(row["name"], row["city"])
```

### Writing a CSV

```python
df.to_csv("output.csv", index=False)   # index=False avoids an extra unnamed column
```

---

## 3.2 JSON (JavaScript Object Notation)

### What it is
A text format for storing **labeled, possibly nested** data. It's the main language APIs use to talk to each other.

### The building blocks

JSON has only a few pieces:

| JSON piece | Looks like | Python equivalent |
|---|---|---|
| **Object** (key-value pairs) | `{"name": "Asha", "age": 28}` | `dict` |
| **Array** (ordered list) | `[1, 2, 3]` | `list` |
| **String** | `"Hyderabad"` | `str` |
| **Number** | `28`, `3.14` | `int`, `float` |
| **Boolean** | `true`, `false` | `True`, `False` |
| **Null** | `null` | `None` |

> Note the lowercase `true`/`false`/`null` in JSON versus Python's `True`/`False`/`None`.

### Reading a real example

```json
{
  "city": "Hyderabad",
  "coordinates": { "lat": 17.38, "lon": 78.48 },
  "readings": [
    { "time": "2026-09-29T09:00", "temp_c": 28.4 },
    { "time": "2026-09-29T10:00", "temp_c": 29.9 }
  ],
  "is_estimated": false
}
```

Read it like a tree:

- The top level is an **object** with keys `city`, `coordinates`, `readings`, `is_estimated`.
- `coordinates` is an object *inside* the object (nested).
- `readings` is an **array** of objects: a list of records. **This part is essentially a table!** Each object is a row; `time` and `temp_c` are columns.

### Rules that trip beginners
- Keys **must** be in double quotes: `"name"`, not `name` or `'name'`.
- No trailing commas: `[1, 2, 3,]` is invalid.
- No comments allowed.
- Single quotes are not allowed for strings.

### Reading JSON in Python

```python
import json

# From a file
with open("weather.json", encoding="utf-8") as f:
    data = json.load(f)          # returns a dict (or list)

print(data["city"])                       # Hyderabad
print(data["coordinates"]["lat"])         # 17.38
print(data["readings"][0]["temp_c"])      # 28.4  (first reading)

# From a string
text = '{"a": 1, "b": [2, 3]}'
obj = json.loads(text)           # loads = "load string"
```

### Writing JSON

```python
with open("out.json", "w", encoding="utf-8") as f:
    json.dump(data, f, indent=2, ensure_ascii=False)
```

`indent=2` makes it pretty; `ensure_ascii=False` keeps non-English characters readable.

### From JSON to table

Since analysis happens on tables, you often convert:

```python
import pandas as pd

df = pd.DataFrame(data["readings"])
print(df)
#                time  temp_c
# 0  2026-09-29T09:00    28.4
# 1  2026-09-29T10:00    29.9
```

For deeply nested JSON, use `pd.json_normalize` (covered in section 7.3).

### JSON Lines (`.jsonl`)
Sometimes big JSON data is stored **one JSON object per line**. Read with:

```python
df = pd.read_json("big.jsonl", lines=True)
```

---

## 3.3 Excel (`.xlsx`)

### What it is
A **binary (zipped) file** that stores a workbook: multiple sheets, formulas, formatting, charts, colors, merged cells, and more.

### Why it's everywhere
Businesses run on Excel. You will get Excel files from finance, HR, sales, operations, and government offices.

### Structure of an Excel file
- **Workbook** = the whole file.
- **Sheet** (tab) = one page inside. `Sheet1`, `Sheet2`, etc.
- Each sheet is a grid of cells.

### The gotchas
1. **Not plain text.** You can't open it in Notepad.
2. **Multiple sheets.** You must choose which one to read.
3. **Human-friendly formatting breaks machine reading**: title rows above the header, merged cells, blank rows, subtotals at the bottom, notes in the margins.
4. **Formulas.** Pandas reads the *last computed value*, not the formula.
5. **Hidden data.** Hidden sheets, rows, or columns still exist.
6. **Dates** are stored as numbers under the hood, which can occasionally show up wrong.

### Reading Excel in Python

```python
import pandas as pd

# First sheet by default
df = pd.read_excel("sales.xlsx")

# A specific sheet, skipping junk rows at the top
df = pd.read_excel("sales.xlsx", sheet_name="Q3", skiprows=2)

# List all sheet names
xls = pd.ExcelFile("sales.xlsx")
print(xls.sheet_names)

# Read all sheets at once -> dict of DataFrames
all_sheets = pd.read_excel("sales.xlsx", sheet_name=None)
for name, sheet_df in all_sheets.items():
    print(name, sheet_df.shape)
```

### Writing Excel

```python
with pd.ExcelWriter("report.xlsx") as writer:
    df1.to_excel(writer, sheet_name="Summary", index=False)
    df2.to_excel(writer, sheet_name="Details", index=False)
```

---

## 3.4 Which format should I use?

| | CSV | JSON | Excel |
|---|---|---|---|
| **Type** | Plain text | Plain text | Binary (zipped XML) |
| **Structure** | Flat table only | Nested / flexible | Multiple flat sheets |
| **Human-friendly to edit** | Fair | Fair | Excellent |
| **File size** | Small | Medium (labels repeat) | Medium-large |
| **Data types** | None (all text) | Basic (string/number/bool/null) | Yes (numbers, dates, ...) |
| **Best for** | Big flat datasets, data exchange | APIs, nested data, configs | Business reports, small data, people who love Excel |
| **Watch out for** | Encoding, delimiters, quotes | Nesting, inconsistent keys | Merged cells, extra header rows, multiple sheets |

**Rule of thumb:**
- Sharing a large table with another program → **CSV**.
- Talking to a web service or storing nested info → **JSON**.
- Receiving from or delivering to non-technical business users → **Excel**.

---

# 4. APIs as Data Sources

## 4.1 What is an API?

**API = Application Programming Interface.** It sounds scary, but the idea is simple:

> **An API is a menu of requests that one program allows other programs to make.**

### The restaurant analogy

You (customer) don't walk into the kitchen and cook your own food. You give your order to a **waiter**, who takes it to the **kitchen**, and brings back your food.

| Restaurant | API world |
|---|---|
| You | Your Python program (the **client**) |
| The menu | The API **documentation** (what you're allowed to ask for) |
| The waiter | The **API** itself |
| The kitchen | The company's **server / database** |
| Your order | The **request** |
| The meal | The **response** (usually JSON) |

Why does this matter for data? Companies don't want to hand you their whole database. They expose a controlled window: "You can ask for today's weather, or a stock price, or a list of movies, in these specific ways."

## 4.2 Why APIs are great data sources

- **Fresh**: data is current, not a stale download.
- **Precise**: you request only what you need (a city, a date range).
- **Automatable**: your code can fetch it every hour.
- **Official**: the company supports it and documents it.

## 4.3 Web APIs and REST

Most APIs you'll use are **web APIs** built on **HTTP**, the same protocol your browser uses. The dominant style is called **REST**. In a REST API:

- Things are called **resources** (users, products, weather readings).
- Each resource lives at a **URL** called an **endpoint**.
- You use an **HTTP method** to say what you want to do.

## 4.4 Anatomy of an API request

Consider this URL:

```
https://api.open-meteo.com/v1/forecast?latitude=17.38&longitude=78.48&hourly=temperature_2m
└──────────┬──────────┘└──────┬──────┘└───────────────────────┬───────────────────────────┘
       base URL            endpoint                        query parameters
```

| Part | Meaning |
|---|---|
| **Base URL** | The server's address: `https://api.open-meteo.com` |
| **Endpoint / path** | The specific resource: `/v1/forecast` |
| **`?`** | Marks the start of parameters |
| **Query parameters** | `key=value` pairs joined by `&` → filters and options |

### HTTP methods (the verbs)

| Method | Meaning | Restaurant equivalent |
|---|---|---|
| **GET** | Fetch/read data | "May I see the menu?" |
| **POST** | Send data to create something or run a query | "Please place this new order." |
| **PUT / PATCH** | Update existing data | "Change my order." |
| **DELETE** | Remove data | "Cancel my order." |

**For data acquisition, 95% of the time you'll use GET** (sometimes POST).

### Headers
Extra information sent with a request, like the label on an envelope. Examples: your API key, the format you accept (`Accept: application/json`), your app's name (`User-Agent`).

### Request body
Used with POST: the data you're sending (usually JSON).

## 4.5 The response

The server replies with:

1. **Status code**: a 3-digit number telling you what happened.
2. **Headers**: metadata (content type, rate-limit info, etc.).
3. **Body**: the actual data, typically JSON.

### Status codes cheat sheet

| Code | Meaning | What to do |
|---|---|---|
| **200** OK | Success | Use the data |
| **201** Created | Something was created (POST) | Success |
| **400** Bad Request | Your request was malformed | Check your parameters |
| **401** Unauthorized | Missing/invalid API key | Fix authentication |
| **403** Forbidden | You're not allowed | Check permissions/plan |
| **404** Not Found | Endpoint or resource doesn't exist | Check the URL |
| **429** Too Many Requests | You hit the rate limit | Slow down and retry later |
| **500** Internal Server Error | Their server broke | Wait and retry |
| **503** Service Unavailable | Server overloaded / down | Wait and retry |

> **Memory trick:** 2xx = success, 4xx = *your* mistake, 5xx = *their* mistake.

## 4.6 Authentication: proving who you are

Many APIs require identification.

- **No auth**: open APIs (e.g., Open-Meteo). Easiest for learning.
- **API key**: a long secret string, passed as a parameter or header.
  ```
  https://api.example.com/data?api_key=YOUR_KEY
  ```
  or
  ```
  Authorization: Bearer YOUR_KEY
  ```
- **OAuth**: a more complex flow used when accessing a user's account (Google, GitHub).

### 🔐 Golden rule of keys
**Never hard-code API keys in your script or upload them to GitHub.** Store them in environment variables:

```python
import os
API_KEY = os.environ["MY_API_KEY"]
```

## 4.7 Rate limits

APIs limit how many requests you can make (e.g., 60 per minute, 1000 per day). If you exceed it, you get a `429` error or get blocked. Respect the limit by adding pauses between requests.

## 4.8 Pagination: getting data in pages

If an API has 100,000 records, it won't send them all at once. It sends "pages":

```
GET /users?page=1&per_page=100
GET /users?page=2&per_page=100
...
```

You loop through pages until there's no more data. (Code in section 7.2.)

## 4.9 Reading API documentation

Before using any API, find its docs and answer:
1. What is the **base URL**?
2. What **endpoints** exist and what does each return?
3. What **parameters** are required/optional?
4. How do I **authenticate**?
5. What's the **rate limit**?
6. What does a **sample response** look like?

Good docs make you productive in minutes; skipping them wastes hours.

---

# 5. Databases as Data Sources

## 5.1 What is a database?

A **database** is an organized, permanent storage system designed so that many people/programs can store, find, and update data quickly and safely.

**Analogy:** A giant, well-organized filing cabinet with labeled drawers (tables), where each drawer holds forms that all look alike (rows), and a very efficient clerk (the database engine) who fetches exactly what you ask for.

Why not just use files? Files are fine for small data. But databases handle:
- Millions/billions of rows fast
- Many users at once
- Safety (no half-saved data, backups)
- Relationships between datasets
- Access control

## 5.2 Relational databases (SQL databases)

The most common type. Data is stored in **tables**, and tables can be **linked** through shared IDs.

### Example: two linked tables

**customers**

| id | name | city |
|---|---|---|
| 1 | Asha | Hyderabad |
| 2 | Ravi | Bengaluru |

**orders**

| order_id | customer_id | amount | order_date |
|---|---|---|---|
| 101 | 1 | 1200 | 2026-09-01 |
| 102 | 1 | 450 | 2026-09-10 |
| 103 | 2 | 800 | 2026-09-11 |

`orders.customer_id` points to `customers.id`. This is called a **foreign key**. Instead of copying Asha's name into every order, we store her ID once and *link*. This avoids repetition and mistakes.

### Key vocabulary
- **Table**: one collection of rows with the same columns.
- **Primary key**: a column that uniquely identifies each row (`customers.id`).
- **Foreign key**: a column that refers to another table's primary key.
- **Schema**: the blueprint: which tables, columns, and types exist.
- **Query**: a question you ask the database.

**Popular relational databases:** PostgreSQL, MySQL, SQLite, SQL Server, Oracle.

## 5.3 SQL: the language of databases

**SQL (Structured Query Language)** is how you ask questions. Learn these six words first:

```sql
-- Get columns from a table
SELECT name, city
FROM customers;

-- Filter rows
SELECT name, city
FROM customers
WHERE city = 'Hyderabad';

-- Sort and limit
SELECT *
FROM orders
ORDER BY amount DESC
LIMIT 5;

-- Summarize with grouping
SELECT customer_id, SUM(amount) AS total_spent
FROM orders
GROUP BY customer_id;

-- Combine two tables (JOIN)
SELECT c.name, o.amount, o.order_date
FROM orders AS o
JOIN customers AS c
  ON o.customer_id = c.id;
```

Reading order for a beginner: `SELECT` *what columns* → `FROM` *which table* → `WHERE` *which rows* → `GROUP BY` *how to summarize* → `ORDER BY` *how to sort* → `LIMIT` *how many*.

## 5.4 NoSQL databases (brief overview)

Not all data fits neatly into tables. NoSQL databases use other shapes:

| Type | Stores | Example | Use |
|---|---|---|---|
| **Document** | JSON-like documents | MongoDB | Flexible records, semi-structured data |
| **Key-value** | Simple key → value | Redis | Caching, sessions |
| **Column-family** | Wide tables | Cassandra | Huge scale writes |
| **Graph** | Nodes & relationships | Neo4j | Social networks, recommendations |

For this course, focus on relational/SQL. It's the most common in analytics.

## 5.5 Getting data out of a database with Python

### Option A: SQLite (zero setup, great for practice)

SQLite stores an entire database in a single file. Python includes it by default.

```python
import sqlite3
import pandas as pd

# Connect (creates the file if it doesn't exist)
conn = sqlite3.connect("shop.db")

# Create a table and insert data (one time)
conn.execute("""
    CREATE TABLE IF NOT EXISTS customers (
        id INTEGER PRIMARY KEY,
        name TEXT,
        city TEXT
    )
""")
conn.execute("INSERT INTO customers (name, city) VALUES ('Asha', 'Hyderabad')")
conn.execute("INSERT INTO customers (name, city) VALUES ('Ravi', 'Bengaluru')")
conn.commit()

# Read query results directly into a DataFrame
df = pd.read_sql_query("SELECT * FROM customers WHERE city = 'Hyderabad'", conn)
print(df)

conn.close()
```

### Option B: PostgreSQL / MySQL with SQLAlchemy

For real servers you need connection details (host, port, database name, username, password).

```python
import os
import pandas as pd
from sqlalchemy import create_engine, text

db_url = os.environ["DATABASE_URL"]
# e.g. postgresql+psycopg2://user:password@host:5432/dbname
engine = create_engine(db_url)

query = text("""
    SELECT c.name, SUM(o.amount) AS total_spent
    FROM orders o
    JOIN customers c ON c.id = o.customer_id
    WHERE o.order_date >= :start
    GROUP BY c.name
    ORDER BY total_spent DESC
""")

df = pd.read_sql_query(query, engine, params={"start": "2026-09-01"})
print(df.head())
```

(You'll also need the driver: `pip install psycopg2-binary` for PostgreSQL, `pip install pymysql` for MySQL.)

### Safety note: never build SQL by gluing strings
Bad (vulnerable to **SQL injection**):
```python
query = f"SELECT * FROM users WHERE name = '{user_input}'"   # ❌
```
Good (use parameters, as shown above with `:start`, or `?` placeholders in sqlite3):
```python
pd.read_sql_query("SELECT * FROM users WHERE name = ?", conn, params=(user_input,))  # ✅
```

### Read-only access
When analyzing a company's production database, ask for a **read-only** account. You want to read data, never accidentally change it.

## 5.6 Files vs APIs vs Databases: when to use which

| Source | Best when |
|---|---|
| **File** (CSV/Excel/JSON) | Someone hands you a snapshot; one-time analysis |
| **API** | Data belongs to an external service; you need fresh data or a filtered slice |
| **Database** | Data is large, constantly updated, inside your organization |

---

# 6. Downloading Real Datasets

Practice on real data. It's messy, surprising, and much more educational than made-up examples.

## 6.1 Where to find datasets

| Source | What's there | Notes |
|---|---|---|
| **Kaggle Datasets** (kaggle.com/datasets) | Thousands of community datasets on every topic | Free account needed; great for practice |
| **data.gov.in** | Indian government open data (agriculture, health, census, etc.) | Good for Indian context |
| **data.gov** | US government data | Huge catalog |
| **UCI Machine Learning Repository** | Classic clean datasets (Iris, Wine, Adult) | Small, good for learning |
| **World Bank Open Data** | Country-level economic and social indicators | Has an API too |
| **Google Dataset Search** | Search engine for datasets across the web | Great for discovery |
| **Hugging Face Datasets** | Text, image, audio datasets for AI | More AI-focused |
| **OpenML** | ML datasets with metadata | Good for modeling |
| **GitHub** | Many repos host CSVs (search "awesome public datasets") | Check licenses |

## 6.2 How to evaluate a dataset before you use it

Don't download blindly. Ask:

1. **Who created it?** Government, university, company, random person?
2. **When was it last updated?** A 2015 dataset may not represent today.
3. **What's the license?** Can you use it commercially? Do you have to give credit?
4. **Is there documentation / a data dictionary?** A data dictionary lists every column and its meaning. Without it, `col_7` is a mystery.
5. **How big is it?** Will it fit in your computer's memory?
6. **What's the coverage?** Which places, times, populations? Who's *missing*?
7. **Is it biased or sampled?** A survey of 200 people is not "all of India".

## 6.3 Downloading in the browser vs Python

**Browser:** click "Download", save the file, put it in your project's `data/` folder.

**Python (better: reproducible!):** anyone can rerun your script and get the same data.

```python
import requests
from pathlib import Path

url = "https://raw.githubusercontent.com/mwaskom/seaborn-data/master/tips.csv"
Path("data").mkdir(exist_ok=True)

response = requests.get(url, timeout=30)
response.raise_for_status()             # raises an error if the download failed

with open("data/tips.csv", "wb") as f:  # "wb" = write binary
    f.write(response.content)
print("Saved", len(response.content), "bytes")
```

Or even shorter, since pandas can read directly from a URL:

```python
import pandas as pd
df = pd.read_csv("https://raw.githubusercontent.com/mwaskom/seaborn-data/master/tips.csv")
```

**Large files** (stream in chunks so you don't load it all in memory):

```python
with requests.get(url, stream=True, timeout=60) as r:
    r.raise_for_status()
    with open("data/big_file.csv", "wb") as f:
        for chunk in r.iter_content(chunk_size=1024 * 1024):  # 1 MB
            f.write(chunk)
```

**Kaggle via Python** (optional): install `kaggle`, put your API token in `~/.kaggle/kaggle.json`, then:
```bash
kaggle datasets download -d <owner>/<dataset-name> --unzip -p data/
```

## 6.4 Your "first look" routine

The moment you load any new dataset, run these:

```python
import pandas as pd

df = pd.read_csv("data/tips.csv")

print(df.shape)          # how many rows, columns?
print(df.head())         # first rows: do they look right?
print(df.tail())         # last rows: any junk footer?
print(df.dtypes)         # are types sensible?
print(df.info())         # non-null counts per column
print(df.describe())     # summary stats for numeric columns
print(df.isna().sum())   # missing values per column
print(df.duplicated().sum())  # duplicate rows
```

This 10-second habit prevents hours of confusion later.

## 6.5 Organize your project folders

```
my_project/
├── data/
│   ├── raw/          # untouched originals, NEVER edit these
│   └── processed/    # cleaned versions you create
├── notebooks/
├── scripts/
└── README.md         # where the data came from, when, and license
```

**Golden rule:** keep raw data raw. Do cleaning in code, so you can always redo it.

---

# 7. Python for Data Acquisition

Now we combine everything. Python is the glue that fetches data from anywhere and turns it into a table.

## 7.1 HTTP Requests

### What is HTTP?
**HTTP (HyperText Transfer Protocol)** is the language browsers and servers use. When you type a URL, your browser sends an HTTP **GET request**, and the server replies with a **response**. The `requests` library lets Python do the same thing.

### Your first request

```python
import requests

response = requests.get("https://api.open-meteo.com/v1/forecast",
                        params={"latitude": 17.38, "longitude": 78.48,
                                "current_weather": "true"},
                        timeout=10)

print(response.status_code)   # 200 means success
print(response.url)           # the full URL requests built for you
print(response.headers["Content-Type"])   # application/json
print(response.text[:200])    # raw text (first 200 characters)
print(response.json())        # parsed into Python dict
```

### Important parts of a `Response` object

| Attribute / method | What it gives you |
|---|---|
| `response.status_code` | Number like 200, 404 |
| `response.ok` | `True` if status < 400 |
| `response.text` | Body as text |
| `response.content` | Body as raw bytes (for files, images) |
| `response.json()` | Body parsed as JSON → dict/list |
| `response.headers` | Dictionary of response headers |
| `response.raise_for_status()` | Raises an exception if status is 4xx/5xx |

### Always use `timeout`
Without `timeout`, your script may wait forever if the server never answers. Use `timeout=10` (seconds) as a habit.

### Passing parameters the right way
Use `params=` instead of building the URL string yourself. `requests` handles spaces and special characters (URL-encoding).

```python
requests.get(url, params={"q": "New Delhi", "units": "metric"})
# becomes ...?q=New+Delhi&units=metric
```

### Sending headers

```python
headers = {
    "Accept": "application/json",
    "User-Agent": "learning-data-analytics/1.0",
}
response = requests.get(url, headers=headers, timeout=10)
```

### POST requests (sending data)

```python
payload = {"query": "python", "limit": 10}
response = requests.post("https://api.example.com/search", json=payload, timeout=10)
```
`json=payload` automatically converts your dict to JSON and sets the right header.

### Handling errors properly

Things fail: no internet, server down, bad input. Write code that expects failure.

```python
import requests

def safe_get(url, params=None):
    try:
        r = requests.get(url, params=params, timeout=10)
        r.raise_for_status()          # turn 4xx/5xx into an exception
        return r
    except requests.exceptions.Timeout:
        print("Request timed out.")
    except requests.exceptions.HTTPError as e:
        print("Bad status:", e)
    except requests.exceptions.ConnectionError:
        print("Could not connect. Check your internet.")
    except requests.exceptions.RequestException as e:
        print("Something else went wrong:", e)
    return None
```

### Sessions (reuse connections)
When making many requests to the same server, a `Session` is faster and lets you set shared headers once:

```python
session = requests.Session()
session.headers.update({"User-Agent": "learning-data-analytics/1.0"})
r = session.get(url, timeout=10)
```

---

## 7.2 Working with APIs in Python

### Step-by-step recipe

1. Read the API docs.
2. Test in a browser or with one `requests.get`.
3. Inspect the JSON structure.
4. Extract the part you need.
5. Convert to a DataFrame.
6. Save.

### Example: no-key API → DataFrame

```python
import requests
import pandas as pd

url = "https://api.open-meteo.com/v1/forecast"
params = {
    "latitude": 17.38,
    "longitude": 78.48,
    "hourly": "temperature_2m,relative_humidity_2m",
    "timezone": "Asia/Kolkata",
    "forecast_days": 2,
}

r = requests.get(url, params=params, timeout=15)
r.raise_for_status()
data = r.json()

# Explore the structure first!
print(data.keys())
print(data["hourly"].keys())

# hourly is a dict of equal-length lists -> perfect for a DataFrame
df = pd.DataFrame(data["hourly"])
df["time"] = pd.to_datetime(df["time"])
print(df.head())
```

**Habit:** whenever you get a new API response, `print(data.keys())` or `print(json.dumps(data, indent=2)[:1500])` to see its shape before writing extraction code.

### Using an API key safely

```python
import os
import requests

API_KEY = os.environ.get("MY_API_KEY")
if not API_KEY:
    raise RuntimeError("Set the MY_API_KEY environment variable first.")

r = requests.get(
    "https://api.example.com/v1/data",
    headers={"Authorization": f"Bearer {API_KEY}"},
    params={"city": "Hyderabad"},
    timeout=10,
)
```

Setting an environment variable:
- macOS/Linux: `export MY_API_KEY="abc123"`
- Windows PowerShell: `$env:MY_API_KEY="abc123"`

### Pagination loop (page-number style)

```python
import requests
import time

def fetch_all_pages(url, per_page=100, max_pages=50):
    all_items = []
    page = 1
    while page <= max_pages:                     # safety cap to avoid infinite loops
        r = requests.get(url, params={"page": page, "per_page": per_page}, timeout=15)
        r.raise_for_status()
        items = r.json()

        if not items:                            # empty page -> we've reached the end
            break

        all_items.extend(items)
        page += 1
        time.sleep(0.5)                          # be polite: respect rate limits
    return all_items
```

**Other pagination styles you'll meet:**
- **Offset/limit:** `?offset=200&limit=100`
- **Cursor/token:** response includes `next_cursor` or `next_page_token`; you pass it to get the next page.
- **Link header / `next` URL:** response gives the full URL of the next page.

Always check the docs to see which one applies.

### Retrying on temporary failures

```python
import time
import requests

def get_with_retry(url, params=None, retries=3, backoff=2):
    for attempt in range(1, retries + 1):
        try:
            r = requests.get(url, params=params, timeout=15)
            if r.status_code == 429:               # rate limited
                wait = int(r.headers.get("Retry-After", backoff * attempt))
                print(f"Rate limited. Waiting {wait}s...")
                time.sleep(wait)
                continue
            r.raise_for_status()
            return r
        except requests.exceptions.RequestException as e:
            print(f"Attempt {attempt} failed: {e}")
            time.sleep(backoff * attempt)          # wait longer each time
    raise RuntimeError("All retries failed")
```

This pattern ("wait a bit longer each retry") is called **exponential/linear backoff**.

### Be a good API citizen
- Respect rate limits.
- Cache results you've already fetched instead of asking again.
- Read the terms of use.
- Identify your app with a `User-Agent`.

---

## 7.3 Working with JSON in Python

### Navigating nested JSON

Given:

```python
data = {
    "user": {"id": 7, "name": "Asha", "address": {"city": "Hyderabad"}},
    "orders": [
        {"id": 101, "amount": 1200, "items": [{"sku": "A1", "qty": 2}]},
        {"id": 102, "amount": 450,  "items": [{"sku": "B7", "qty": 1}]},
    ],
}
```

- `data["user"]["name"]` → `"Asha"`
- `data["user"]["address"]["city"]` → `"Hyderabad"`
- `data["orders"][0]["amount"]` → `1200`
- `data["orders"][1]["items"][0]["sku"]` → `"B7"`

Read it like directions: **key → key → index → key**.

### Missing keys: use `.get()`

```python
data["user"]["phone"]            # KeyError! crashes
data["user"].get("phone")        # returns None
data["user"].get("phone", "N/A") # returns "N/A" default
```

APIs often omit fields, so `.get()` saves you from crashes.

### Flattening nested JSON with pandas

```python
import pandas as pd

# 1) A list of flat-ish records → DataFrame directly
df_orders = pd.DataFrame(data["orders"])

# 2) Flatten nested dictionaries with json_normalize
records = [
    {"id": 1, "name": "Asha", "address": {"city": "Hyderabad", "pin": "500001"}},
    {"id": 2, "name": "Ravi", "address": {"city": "Bengaluru", "pin": "560001"}},
]
df = pd.json_normalize(records)
print(df)
#    id   name address.city address.pin
# 0   1  Asha  Hyderabad      500001
# 1   2  Ravi  Bengaluru      560001

# 3) Explode a list inside each record into its own rows
df = pd.json_normalize(
    data["orders"],
    record_path="items",          # the nested list to expand into rows
    meta=["id", "amount"],        # parent fields to carry along
    meta_prefix="order_",
)
print(df)
#   sku  qty order_id order_amount
# 0  A1    2      101         1200
# 1  B7    1      102          450
```

This last one is critical: it converts "one order with many items" into "one row per item", a proper table.

### Saving and loading JSON
See section 3.2. Remember `json.load` (file), `json.loads` (string), `json.dump` (to file), `json.dumps` (to string).

### Pretty-printing to understand an unknown response

```python
import json
print(json.dumps(data, indent=2)[:2000])
```

---

## 7.4 Working with Files in Python

### Paths: use `pathlib`

File paths differ between Windows (`C:\Users\...`) and Mac/Linux (`/home/...`). `pathlib.Path` handles both.

```python
from pathlib import Path

data_dir = Path("data")
data_dir.mkdir(parents=True, exist_ok=True)      # create folder if missing

file_path = data_dir / "raw" / "sales.csv"      # `/` joins path parts
print(file_path.exists())      # does it exist?
print(file_path.name)          # sales.csv
print(file_path.suffix)        # .csv
print(file_path.stem)          # sales
```

### Reading and writing plain text

```python
# Write
with open("notes.txt", "w", encoding="utf-8") as f:
    f.write("Downloaded on 2026-09-29\n")

# Append
with open("notes.txt", "a", encoding="utf-8") as f:
    f.write("Source: data.gov.in\n")

# Read
with open("notes.txt", "r", encoding="utf-8") as f:
    content = f.read()
```

**Modes:** `"r"` read, `"w"` write (overwrites!), `"a"` append, add `"b"` for binary (`"rb"`, `"wb"`).

**Why `with`?** It automatically closes the file even if an error occurs.

### Listing files and reading many at once

```python
from pathlib import Path
import pandas as pd

files = sorted(Path("data/raw").glob("sales_*.csv"))
print(files)

frames = [pd.read_csv(f) for f in files]
combined = pd.concat(frames, ignore_index=True)
print(combined.shape)
```

This "many monthly files → one DataFrame" pattern is extremely common.

### Reading big CSVs in chunks

```python
total = 0
for chunk in pd.read_csv("huge.csv", chunksize=100_000):
    total += chunk["amount"].sum()
print(total)
```

### Other formats worth knowing

```python
df.to_parquet("data.parquet")       # fast, compressed, keeps types (pip install pyarrow)
pd.read_parquet("data.parquet")

import zipfile
with zipfile.ZipFile("data.zip") as z:
    z.extractall("data/raw")        # unzip everything
```

### Encoding troubleshooting

If `read_csv` raises `UnicodeDecodeError`, try:

```python
pd.read_csv("file.csv", encoding="utf-8-sig")   # files saved by Excel with BOM
pd.read_csv("file.csv", encoding="latin-1")     # older/Western European files
pd.read_csv("file.csv", encoding="cp1252")      # Windows default
```

---

## 7.5 Basic Automation

**Automation** = making your code do the repetitive work so you don't have to click, download, and copy-paste by hand.

### Level 1: Loops

Fetch data for many cities:

```python
import requests
import pandas as pd

cities = {
    "Hyderabad": (17.38, 78.48),
    "Bengaluru": (12.97, 77.59),
    "Chennai": (13.08, 80.27),
}

rows = []
for name, (lat, lon) in cities.items():
    r = requests.get(
        "https://api.open-meteo.com/v1/forecast",
        params={"latitude": lat, "longitude": lon, "current_weather": "true"},
        timeout=10,
    )
    r.raise_for_status()
    w = r.json()["current_weather"]
    rows.append({"city": name, "temp_c": w["temperature"], "wind_kmh": w["windspeed"], "time": w["time"]})

df = pd.DataFrame(rows)
print(df)
```

### Level 2: Functions and structure

Break code into reusable steps:

```python
def fetch(city, lat, lon): ...
def parse(response_json): ...
def save(df, path): ...

def main():
    ...

if __name__ == "__main__":
    main()
```

The `if __name__ == "__main__":` line means "run `main()` only when this file is executed directly", which makes it scriptable.

### Level 3: Timestamped file names

Never overwrite yesterday's data by mistake:

```python
from datetime import datetime
from pathlib import Path

stamp = datetime.now().strftime("%Y-%m-%d_%H%M")
out = Path("data/raw") / f"weather_{stamp}.csv"
df.to_csv(out, index=False)
```

### Level 4: Logging (instead of `print`)

A script running unattended needs a diary so you can see what happened.

```python
import logging

logging.basicConfig(
    filename="collector.log",
    level=logging.INFO,
    format="%(asctime)s | %(levelname)s | %(message)s",
)

logging.info("Starting fetch")
logging.warning("Rate limit hit, sleeping 30s")
logging.error("Failed to fetch Chennai")
```

### Level 5: Scheduling

Running the script automatically at set times:

| Method | Works on | Example |
|---|---|---|
| **cron** | macOS / Linux | `0 * * * * /usr/bin/python3 /path/collector.py` (every hour) |
| **Task Scheduler** | Windows | GUI: create task → trigger hourly → action: run `python collector.py` |
| **`schedule` library** | Any (script must stay running) | See below |
| **Cloud schedulers** | Cloud | GitHub Actions cron, AWS EventBridge, GCP Cloud Scheduler |

```python
# pip install schedule
import schedule, time

def job():
    print("Fetching...")   # call your main() here

schedule.every(1).hours.do(job)
schedule.every().day.at("09:00").do(job)

while True:
    schedule.run_pending()
    time.sleep(30)
```

**Cron syntax quick guide:** `minute hour day-of-month month day-of-week command`
- `*/15 * * * *` → every 15 minutes
- `0 9 * * 1-5` → 9:00 AM Monday to Friday

### Automation best practices
- **Idempotent:** running the script twice shouldn't corrupt your data.
- **Fail loudly, log clearly.**
- **Don't store secrets in code.**
- **Keep raw data.** Append new files rather than overwriting.
- **Test manually** before scheduling.

---

# 8. Mini Project: Build a Weather Data Collector

Let's put everything together. This script:
1. Calls a free API (no key needed) for several cities.
2. Handles errors and retries.
3. Converts JSON → DataFrame.
4. Saves a timestamped CSV.
5. Logs what happened.

Change the city list to whatever you like.

```python
"""weather_collector.py
Fetch current weather for several cities and save to a timestamped CSV.
"""
import logging
import time
from datetime import datetime
from pathlib import Path

import pandas as pd
import requests

API_URL = "https://api.open-meteo.com/v1/forecast"
CITIES = {
    "Hyderabad": (17.38, 78.48),
    "Bengaluru": (12.97, 77.59),
    "Chennai": (13.08, 80.27),
    "Mumbai": (19.08, 72.88),
}
OUTPUT_DIR = Path("data/raw")

logging.basicConfig(
    filename="collector.log",
    level=logging.INFO,
    format="%(asctime)s | %(levelname)s | %(message)s",
)


def fetch_city(name, lat, lon, retries=3):
    """Return one dict of current weather for a city, or None if it fails."""
    params = {"latitude": lat, "longitude": lon, "current_weather": "true"}
    for attempt in range(1, retries + 1):
        try:
            r = requests.get(API_URL, params=params, timeout=10)
            r.raise_for_status()
            w = r.json()["current_weather"]
            return {
                "city": name,
                "observed_at": w["time"],
                "temperature_c": w["temperature"],
                "windspeed_kmh": w["windspeed"],
                "fetched_at": datetime.now().isoformat(timespec="seconds"),
            }
        except (requests.exceptions.RequestException, KeyError) as e:
            logging.warning("%s attempt %d failed: %s", name, attempt, e)
            time.sleep(2 * attempt)
    logging.error("Giving up on %s", name)
    return None


def main():
    OUTPUT_DIR.mkdir(parents=True, exist_ok=True)
    rows = []
    for name, (lat, lon) in CITIES.items():
        row = fetch_city(name, lat, lon)
        if row:
            rows.append(row)
        time.sleep(0.5)  # be polite to the API

    if not rows:
        logging.error("No data collected. Nothing saved.")
        return

    df = pd.DataFrame(rows)
    stamp = datetime.now().strftime("%Y-%m-%d_%H%M")
    out_path = OUTPUT_DIR / f"weather_{stamp}.csv"
    df.to_csv(out_path, index=False)
    logging.info("Saved %d rows to %s", len(df), out_path)
    print(df)


if __name__ == "__main__":
    main()
```

**Run it:** `python weather_collector.py`
**Then automate it:** schedule it hourly with cron or Task Scheduler. After a few days you'll have your own weather dataset, built by you.

**Stretch ideas:**
- Add humidity and rain by extending the `hourly`/`current` parameters (check the Open-Meteo docs).
- Append everything into a SQLite table instead of separate CSVs.
- Combine all CSVs with `Path.glob` and `pd.concat`.

---

# 9. Common Mistakes and How to Avoid Them

| Mistake | Why it hurts | Fix |
|---|---|---|
| Not checking `status_code` | You parse an error page as if it were data | `r.raise_for_status()` |
| No `timeout` | Script hangs forever | `timeout=10` |
| Hard-coding API keys | Keys leak on GitHub | Environment variables |
| Editing raw files by hand | You can't reproduce your steps | Keep `raw/` untouched; clean in code |
| Ignoring encoding | Garbled characters | Try `utf-8`, `utf-8-sig`, `latin-1` |
| Forgetting `index=False` | Extra "Unnamed: 0" column in CSV | `df.to_csv(..., index=False)` |
| Assuming all JSON records have the same keys | `KeyError` crashes | Use `.get()` |
| Fetching more often than the rate limit | Blocked / banned | `time.sleep`, caching |
| Overwriting old downloads | Lose history | Timestamped filenames |
| Building SQL with f-strings | SQL injection | Parameterized queries |
| Trusting a dataset blindly | Wrong conclusions | Check source, date, license, dictionary |
| Skipping the "first look" | Bugs discovered late | `head`, `info`, `describe`, `isna().sum()` |
| Using Excel's "pretty" layout as data | Merged cells and title rows break parsing | `skiprows`, `header=`, or ask for a clean export |

---

# 10. Practice Exercises

## Beginner
1. **Classify:** For each item, say structured / semi-structured / unstructured: a) a `.csv` of exam marks, b) a WhatsApp voice note, c) a JSON response from a movie API, d) a scanned invoice PDF, e) a table in MySQL.
2. **Read a CSV:** Download `tips.csv` from the seaborn-data repo (URL in section 6.3). Print its shape, dtypes, and first 5 rows.
3. **JSON by hand:** Write a JSON object describing yourself with a name, age, a list of hobbies, and a nested `address` object. Load it with `json.load` and print your second hobby.
4. **Write files:** Save a small DataFrame to `.csv` and `.xlsx`, then read both back and confirm they match.

## Intermediate
5. **Status codes:** Call `https://httpbin.org/status/404` and `https://httpbin.org/status/200`. Print the status codes and handle the 404 gracefully with `try/except`.
6. **Flatten:** Fetch `https://jsonplaceholder.typicode.com/users` (a free fake API). It returns nested JSON (address, company). Use `pd.json_normalize` to make a flat table, then save it.
7. **Pagination:** Use `https://jsonplaceholder.typicode.com/posts?_page=1&_limit=10` and loop through pages 1-5, collecting all posts into one DataFrame.
8. **SQLite:** Create a database with `customers` and `orders` tables (use the examples in section 5.2). Write a SQL query that joins them and returns total spend per customer, loaded into pandas.
9. **Multi-file:** Create three small CSVs (`sales_jan.csv`, `sales_feb.csv`, `sales_mar.csv`) and combine them into one DataFrame with `glob` + `concat`.

## Advanced
10. **Weather collector:** Extend the mini project to save results into SQLite (`INSERT` with `df.to_sql`) and schedule it to run every hour for a day.
11. **Robust downloader:** Write a function `download(url, path)` that streams the file, retries up to 3 times, skips if the file already exists, and logs the result.
12. **API exploration:** Pick any public API from a list like `public-apis` on GitHub, read its docs, and build a script that fetches 200+ records across multiple pages and saves them as clean CSV.

---

# 11. Glossary

| Term | Meaning |
|---|---|
| **API** | A defined way for programs to request data/actions from another program |
| **API key** | A secret string that identifies you to an API |
| **Authentication** | Proving who you are |
| **Base URL** | The root address of an API |
| **Boolean** | A True/False value |
| **Cell** | A single value in a table |
| **Client** | The program making a request (your script) |
| **Column / Field / Feature** | One property recorded for every row |
| **CSV** | Comma-Separated Values; plain text table |
| **DataFrame** | pandas' table object in Python |
| **Data dictionary** | Document describing each column of a dataset |
| **Database** | Organized, persistent storage system for data |
| **Encoding** | How text characters are converted into bytes (UTF-8 is standard) |
| **Endpoint** | A specific URL of an API that returns a specific resource |
| **Foreign key** | Column referencing another table's primary key |
| **Header (HTTP)** | Metadata sent with a request/response |
| **Header (table)** | The first row holding column names |
| **HTTP** | The protocol of the web |
| **JSON** | JavaScript Object Notation; labeled, nested text format |
| **Key (JSON)** | The label in a key-value pair |
| **Null / NaN** | A missing value |
| **Pagination** | Splitting large results into pages |
| **Parameter (query)** | `key=value` option added to a URL |
| **Primary key** | Column uniquely identifying each row |
| **Query** | A question asked to a database (or a URL's parameters) |
| **Rate limit** | Max number of requests allowed in a time span |
| **Record / Row** | One item in a table |
| **REST** | Common web API style built around resources and HTTP methods |
| **Response** | What the server sends back |
| **Schema** | Blueprint of a database: tables, columns, types |
| **Server** | The computer that answers requests |
| **SQL** | Language for querying relational databases |
| **Status code** | Number describing the result of a request |
| **Structured / Semi / Unstructured** | Degrees of data organization |
| **Timeout** | Max seconds to wait for a response |
| **XLSX** | Excel workbook format |

---

# 12. Cheat Sheet

## Choosing the source
```
Snapshot handed to you   → File (CSV / Excel / JSON)
Live / external service  → API
Company's internal data  → Database (SQL)
Learning / practice      → Public datasets (Kaggle, data.gov.in, UCI)
```

## Pandas I/O one-liners
```python
pd.read_csv("f.csv")            df.to_csv("f.csv", index=False)
pd.read_excel("f.xlsx", sheet_name="S")   df.to_excel("f.xlsx", index=False)
pd.read_json("f.json")          df.to_json("f.json", orient="records", indent=2)
pd.read_sql_query(sql, conn)    df.to_sql("table", conn, if_exists="append", index=False)
pd.json_normalize(list_of_dicts)
```

## Requests one-liners
```python
r = requests.get(url, params={...}, headers={...}, timeout=10)
r.raise_for_status()
data = r.json()
r = requests.post(url, json={...}, timeout=10)
```

## JSON one-liners
```python
json.load(f)   json.loads(s)   json.dump(obj, f, indent=2)   json.dumps(obj)
d.get("key", default)
```

## Files one-liners
```python
Path("data").mkdir(parents=True, exist_ok=True)
Path("data").glob("*.csv")
with open(p, "w", encoding="utf-8") as f: ...
```

## First-look routine
```python
df.shape; df.head(); df.info(); df.describe(); df.isna().sum(); df.duplicated().sum()
```

## Status code memory trick
```
2xx = success     4xx = you messed up     5xx = they messed up     429 = slow down
```

---

## ✅ Module 1 Self-Check

Before moving on to Module 2, you should be able to answer "yes" to each:

- [ ] I can explain data, rows, columns, and data types in my own words.
- [ ] I can tell structured, semi-structured, and unstructured data apart and give examples.
- [ ] I can read and write CSV, JSON, and Excel files in Python.
- [ ] I can explain what an API is, including endpoint, parameters, status codes, and rate limits.
- [ ] I can make a `requests.get` call, check the status, and turn the JSON into a DataFrame.
- [ ] I can query a SQLite/SQL database from Python and load results into pandas.
- [ ] I know where to find real datasets and how to judge whether to trust them.
- [ ] I can write a small script that fetches data for multiple inputs, saves a timestamped file, and logs what happened.

If any box is unchecked, revisit that section and try the matching exercise.

**Next up: Module 2.** Now that you can get data, you'll learn to make sense of what you got.