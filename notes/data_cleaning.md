# Data Cleaning

## 1. Null Values

### 1.1 Types of Missing Data

- **MCAR (Missing Completely at Random)** — missingness has no relationship with any variable. Example: a sensor randomly fails to record a reading.
- **MAR (Missing at Random)** — missingness is related to other observed variables but not the missing value itself. Example: younger respondents skip income questions more often — missingness depends on age (observed), not income (missing).
- **MNAR (Missing Not at Random)** — missingness is related to the missing value itself. Example: high-income earners are less likely to report their income — the value itself drives the missingness.

Why it matters: the type of missingness determines which handling strategy is appropriate. Dropping MNAR data introduces bias. Filling MCAR data with mean is generally safe.

### 1.2 Investigate Nulls

Before handling nulls, understand them:

- How many nulls per column? What percentage?
- Is there a pattern? (e.g., entire rows missing, specific columns correlated)
- Where are they coming from? (source system issues, ETL bugs, user behavior)
- Is the missingness MCAR, MAR, or MNAR?

### 1.3 Handle Nulls

#### 1.3.1 Drop

- **Drop rows** — if too many column values are missing, drop the entire row. Set a threshold (e.g., drop rows with >50% nulls).
- **Drop columns** — if too many row values are missing, drop the entire column.

Best for: MCAR data with a small proportion of missing values.

#### 1.3.2 Fill (Imputation)

**Numerical:**

- **Mean** — use when the distribution is symmetrical (normal). Sensitive to outliers.
- **Median** — use when the distribution is skewed. Robust to outliers.
- **Interpolation** — use for time-series or ordered data.

**Categorical:**

- **Mode** — fill with the most frequent value.
- **"Unknown" / "Other"** — create an explicit category for missing values.

Best for: MCAR or MAR data. Avoid for MNAR — imputation will introduce bias.

#### 1.3.3 Flag (Keep as-is)

- Create a binary indicator column to preserve the information that data was missing.
- Useful when the missingness itself is a signal (MNAR).
- Avoid filling if it would distort the distribution.

---

## 2. Duplicates

### 2.1 Identify Duplicates

- What is the unique combination of columns (natural key)?
- Check the primary key (PK) — should be not null and unique.
- If no PK exists, define one from a group of columns.

### 2.2 Types of Duplicates

- **Exact duplicates** — every column is identical across rows.
- **Partial duplicates** — key columns match but other columns differ (e.g., same order captured twice with different timestamps).

### 2.3 Handle Duplicates

- Decide which record to keep — most recent? Most complete? First arrived?
- Use window functions (ROW_NUMBER, RANK, DENSE_RANK) partitioned by the unique key and ordered by a tiebreaker (e.g., timestamp) to pick the winner.

`ROW_NUMBER()` vs `RANK()` vs `DENSE_RANK()`:

- **ROW_NUMBER** — always assigns unique numbers, best for deduplication.
- **RANK** — ties get the same rank, next rank is skipped.
- **DENSE_RANK** — ties get the same rank, next rank is not skipped.

---

## 3. Outliers

### 3.1 Detect Outliers

- **IQR method** — values below Q1 - 1.5 \* IQR or above Q3 + 1.5 \* IQR.
- **Z-score** — values more than 3 standard deviations from the mean.
- **Visual inspection** — box plots, histograms, scatter plots.
- **Domain knowledge** — does this value make sense in context? (e.g., a person aged 200)

### 3.2 Handle Outliers

- **Remove** — if they are data entry errors or clearly invalid.
- **Cap/floor (winsorize)** — clip to upper/lower bounds to limit their impact.
- **Keep** — if they represent real, meaningful data points. Not all outliers are bad.

Ask: is this an error or a real extreme value? The answer determines the approach.

---

## 4. Formatting

### 4.1 Column Names

- Lowercase, snake_case, no special characters.
- Consistent naming convention across the entire dataset.

### 4.2 Data Types

- Dates stored as strings → convert to datetime.
- Numbers stored as strings → convert to numeric.
- Low-cardinality text → convert to category.

### 4.3 Data Values

- Trim whitespace in strings.
- Standardize casing (e.g., "Male", "male", "MALE" → pick one).
- Standardize categories (e.g., "NY", "New York", "new york" → pick one).

---

## 5. Validation

After cleaning, verify the results:

- No remaining unexpected nulls.
- No duplicate primary keys.
- Data types are correct.
- Value ranges make sense (e.g., no negative ages, no dates in the future).
- Row count is reasonable — didn't accidentally drop too much data.

---

**Next step:** EDA (Exploratory Data Analysis)
