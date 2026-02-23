---
name: data-profiler
description: Automated data quality and distribution analysis with Plotly visualizations. Generates column statistics (count, nulls, unique values), distribution charts, percentile analysis, JSON key profiling, and PII detection. Use to understand data characteristics before analysis.
---

# Data Profiler

Comprehensive data profiling with data type detection, distribution visualizations, and quality insights.

## When to Use

**AUTO-INVOKES for:**
- "Profile [table]" / "Analyze [table]"
- "Show stats for [table]"
- "Data quality check"
- "What's in [table]?"

**Examples:** "Profile orders table", "Show data quality for user_events last 7 days", "Distribution of revenue"

---

## Workflow

Execute in order:
1. Get schema: `tdx describe database.table`
2. Statistics: Row count, date range, null analysis
3. Data type detection: JSON, Array, Nested, Numeric, Categorical
4. Sample values: 3-5 examples per column
5. **JSON columns:** Sample 20+ records, parse to discover ALL key paths, profile each key
6. **PII detection:** Identify PII by name patterns and content
7. **PII profiling:** Type, completeness, format validation, masking status
8. Distribution queries for key columns
9. Visualizations: Completeness chart, distributions, type breakdown, PII summary
10. **NULL/NOT NULL breakdown:** Tables showing NULL vs NOT NULL counts/percentages for all columns
11. Quality score
12. Recommendations: Including PII handling and NULL resolution

---

## Required Output

1. **Overview:** Total records, columns, date range, quality score, PII count
2. **Data Type Detection:** JSON, Array, String, etc.
3. **Column Statistics:** With sample values
4. **JSON Key Profiling:** Profile ALL keys in JSON columns
5. **PII Analysis:** Identify, type, quality, compliance
6. **NULL/NOT NULL Breakdown:** Separate tables for PII and all columns
7. **Quality Analysis:** Completeness, uniqueness, validity, PII quality
8. **Visualizations:** 2-3 charts (completeness, distributions, PII)
9. **Key Findings:** Strengths, issues, PII handling, NULL concerns
10. **Recommendations:** Prioritized, including PII compliance and NULL resolution

---

## Data Type Detection

**JSON (varchar containing JSON):**
- Detect: Values starting with `{` or `[`
- Report: "JSON Object" or "JSON Array"
- Extract: Top-level keys, nested depth
- Sample 20+ and parse ALL key paths
- Profile each key: completeness, unique values, samples, type

**Array:**
- Detect: Values like `[1,2,3]` or array types
- Report: "ARRAY<type>"
- Show: Length distribution, element types

**Nested:**
- Detect: Columns with `.` in name or nested structures
- Report: "Nested (parent.child)"
- Show: Parent-child relationships

**Numeric:**
- Detect: bigint, double, decimal
- Report: "Numeric (bigint)" or "Numeric (double)"
- Stats: Min, max, avg, percentiles

**Categorical:**
- Detect: varchar with low cardinality (<100 unique values)
- Report: "Categorical (varchar)"
- Show: Top 10 values with counts

**Timestamp:**
- Detect: bigint named "time", "*_at", "*_time"
- Report: "Timestamp (Unix epoch)"
- Show: Date range, time distribution

---

## PII Detection

**PII Types:**
- **EMAIL:** email, email_address, mail, contact_email
- **PHONE:** phone, phone_number, mobile, cell_phone
- **ADDRESS:** address, street, zip, postal_code, city, state
- **NAME:** first_name, last_name, full_name
- **IDENTIFIER:** ssn, passport, driver_license, national_id
- **FINANCIAL:** credit_card, bank_account, routing_number
- **HEALTH:** medical_record, health_id, insurance_id
- **CONSENT:** consent, opt_in, gdpr_consent

**Detection:**
1. Column name pattern matching
2. Content analysis (format validation)
3. Categorize by sensitivity: CRITICAL, HIGH, MEDIUM

**PII Profiling:**
```sql
select
  count(*) as total,
  count([pii_column]) as non_null,
  count(distinct [pii_column]) as unique_values,
  count(*) - count([pii_column]) as null_count,
  cast(count([pii_column]) as double) / count(*) as completeness_pct
from table where td_interval(time, '-30d', 'JST')
```

**Format Validation:**
- EMAIL: Contains @ and domain
- PHONE: Matches phone patterns
- SSN: XXX-XX-XXXX format

---

## Core Statistics

**Per column:**
```sql
select
  count(*) as total_rows,
  count([column]) as non_null_count,
  count(distinct [column]) as unique_values,
  count(*) - count([column]) as null_count,
  cast(count([column]) as double) / count(*) * 100 as completeness_pct
from database.table
where td_interval(time, '-30d', 'JST')
```

**Numeric stats:**
```sql
select
  min([column]) as min_value,
  max([column]) as max_value,
  avg([column]) as avg_value,
  approx_percentile([column], 0.25) as p25,
  approx_percentile([column], 0.50) as median,
  approx_percentile([column], 0.75) as p75,
  approx_percentile([column], 0.95) as p95
from table where td_interval(time, '-30d', 'JST')
```

**Categorical distribution:**
```sql
select [column], count(*) as count
from table where td_interval(time, '-30d', 'JST')
group by [column]
order by count desc limit 10
```

---

## JSON Profiling

**Sample and parse:**
```sql
select [json_column]
from table where td_interval(time, '-1d', 'JST')
limit 20
```

**Extract all key paths:**
- Parse JSON to discover: `$.key`, `$.parent.child`, `$.array[*].field`
- List ALL discovered paths

**Profile each key:**
```sql
select
  count(*) as total,
  count(json_extract_scalar([json_column], '$.key')) as key_present,
  count(distinct json_extract_scalar([json_column], '$.key')) as unique_values
from table where td_interval(time, '-30d', 'JST')
```

---

## NULL/NOT NULL Breakdown

**For each column:**
```sql
select
  '[column_name]' as column_name,
  count(*) as total_rows,
  count([column_name]) as not_null_count,
  count(*) - count([column_name]) as null_count,
  cast(count([column_name]) as double) / count(*) * 100 as not_null_pct,
  cast((count(*) - count([column_name])) as double) / count(*) * 100 as null_pct
from table where td_interval(time, '-30d', 'JST')
```

**Present as:**

### NULL/NOT NULL Breakdown - PII Columns

| Column | Total | NOT NULL | NULL | NOT NULL % | NULL % |
|--------|-------|----------|------|------------|--------|
| email | 10,000 | 9,500 | 500 | 95.0% | 5.0% |
| phone | 10,000 | 8,000 | 2,000 | 80.0% | 20.0% |

### NULL/NOT NULL Breakdown - All Data Types

| Column | Type | Total | NOT NULL | NULL | NOT NULL % | NULL % |
|--------|------|-------|----------|------|------------|--------|
| order_id | Numeric | 10,000 | 10,000 | 0 | 100.0% | 0.0% |
| customer_id | Numeric | 10,000 | 10,000 | 0 | 100.0% | 0.0% |
| email | Categorical | 10,000 | 9,500 | 500 | 95.0% | 5.0% |

---

## Visualizations

Generate Plotly charts as needed:
- Completeness chart (bar chart showing % complete per column)
- Distribution charts (histogram for numeric, bar for categorical)
- PII summary chart

Use TD color palette: `#44BAB8`, `#8FD6D4`, `#DAF1F1`, `#2E41A6`, `#828DCA`, `#8CC97E`, `#EEB53A`

---

## Quality Score

**Calculate:**
```
Quality Score = (Completeness Score × 0.4) + (Uniqueness Score × 0.3) + (Validity Score × 0.3)

Completeness = avg(column_completeness_pct)
Uniqueness = avg(column_uniqueness_ratio)
Validity = % columns passing validation rules
```

**Rating:**
- 90-100: Excellent
- 75-89: Good
- 60-74: Fair
- <60: Poor

---

## Output Format

```markdown
# Data Profiling Report: [database.table]

## Overview
- **Total Records:** 10,000 (Last 30 days)
- **Columns:** 15
- **Date Range:** 2024-11-20 to 2024-12-20
- **Quality Score:** 85/100 (Good)
- **PII Columns:** 3 (email, phone, address)

## Data Type Detection

| Column | Schema Type | Detected Type | Notes |
|--------|-------------|---------------|-------|
| order_id | bigint | Numeric | Primary key |
| metadata | varchar | JSON Object | 5 keys detected |
| tags | varchar | ARRAY<string> | Avg 3 elements |

## Column Statistics

| Column | Type | Total | Non-Null | Unique | Completeness | Samples |
|--------|------|-------|----------|--------|--------------|---------|
| order_id | Numeric | 10,000 | 10,000 | 10,000 | 100% | 12345, 12346 |
| email | Categorical | 10,000 | 9,500 | 9,500 | 95% | user@example.com |

## JSON Key Profiling: metadata

**Discovered Keys:** 5
- `$.customer_id` (100% present, 8,500 unique)
- `$.source` (98% present, 3 unique: "web", "mobile", "api")
- `$.campaign_id` (45% present, 120 unique)
- `$.tags` (30% present, ARRAY<string>)
- `$.preferences.theme` (85% present, 2 unique: "light", "dark")

## PII Analysis

**Detected PII Columns:** 3

| Column | PII Type | Sensitivity | Completeness | Unique Values | Format Valid |
|--------|----------|-------------|--------------|---------------|--------------|
| email | EMAIL | HIGH | 95% | 9,500 | 98% |
| phone | PHONE | HIGH | 80% | 8,000 | 85% |
| address | ADDRESS | HIGH | 70% | 7,000 | N/A |

**Compliance Status:**
- Masking: Not detected
- Encryption: Not detected
- Access Control: Review needed

## NULL/NOT NULL Breakdown

### PII Columns

| Column | Total | NOT NULL | NULL | NOT NULL % | NULL % |
|--------|-------|----------|------|------------|--------|
| email | 10,000 | 9,500 | 500 | 95.0% | 5.0% |
| phone | 10,000 | 8,000 | 2,000 | 80.0% | 20.0% |
| address | 10,000 | 7,000 | 3,000 | 70.0% | 30.0% |

### All Data Types

| Column | Type | NOT NULL | NULL | NOT NULL % | NULL % |
|--------|------|----------|------|------------|--------|
| order_id | Numeric | 10,000 | 0 | 100% | 0% |
| email | Categorical | 9,500 | 500 | 95% | 5% |
| metadata | JSON | 9,800 | 200 | 98% | 2% |

## Data Quality Analysis

**Completeness:** 92% (Excellent)
**Uniqueness:** 85% (Good)
**Validity:** 78% (Good)

**Issues:**
- Email: 5% NULL values
- Phone: 20% NULL, 15% invalid format
- Address: 30% NULL values

## Visualizations

[Completeness Chart]
[Distribution Chart - Amount]
[PII Completeness Chart]

## Key Findings

**Strengths:**
- High completeness (92% overall)
- Good primary key uniqueness
- Clean numeric data

**Issues:**
- PII columns have NULL values (5-30%)
- Phone format validation issues (15%)
- No PII masking detected

## Recommendations

1. Implement PII masking for dev/test environments
2. Add phone format validation
3. Investigate NULL values in PII columns
4. Implement encryption for PII columns
5. Add access controls for PII data
```

