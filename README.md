# 🧾 PAN Number Validation using SQL (PostgreSQL)

### Author: Sandeep Batra  
### Domain: Data Quality & Validation (Credit Risk / Regulatory Data Management)  
---

## 📘 Project Overview
This project focuses on validating and cleaning Indian **Permanent Account Numbers (PAN)** using SQL in **PostgreSQL**.  
It ensures each PAN follows the correct **format, sequence, and uniqueness** based on defined business rules.  
The logic mimics real-world data quality validation pipelines often seen in **regulatory and risk data** environments.

---

## 🎯 Objectives
- Validate **PAN structure** (`AAAAA9999A`) as per standard rules.  
- Detect and handle:
  - Incorrect length or format  
  - Repeated adjacent characters  
  - Sequential letters or digits  
  - Null and duplicate PANs  
- Produce a **summary report** showing valid and invalid PAN counts.

---

## 🧠 Business Rules Implemented

### 🔹 PAN Format
A valid PAN has:
- Exactly **10 characters**.  
- **First 5:** Alphabets (A–Z).  
- **Next 4:** Digits (0–9).  
- **Last 1:** Alphabet (A–Z).  
**Regex Pattern:** `^[A-Z]{5}[0-9]{4}[A-Z]$`

### 🔹 Alphabet Rules
1. Adjacent alphabets **cannot be identical** (❌ `AABCD`, ✅ `ABXCD`)  
2. Alphabets **cannot be sequential** (❌ `ABCDE`, ✅ `ABCDX`)

### 🔹 Digit Rules
1. Adjacent digits **cannot be identical** (❌ `1123`, ✅ `1923`)  
2. Digits **cannot form a sequence** (❌ `1234`, ✅ `1934`)

---

## 🧩 Data Cleaning Steps
| Step | Action | SQL Functions Used | Purpose |
|------|---------|--------------------|----------|
| 1 | Load raw data into staging table | `CREATE TABLE`, `COPY` | Prepare workspace |
| 2 | Handle NULLs and trim spaces | `COALESCE()`, `TRIM()` | Normalize inputs |
| 3 | Convert to uppercase | `UPPER()` | Enforce consistent case |
| 4 | Remove duplicates | `ROW_NUMBER() OVER (PARTITION BY ...)` | Keep one record per PAN |
| 5 | Validate PAN pattern | Regex `~ '^[A-Z]{5}[0-9]{4}[A-Z]$'` | Check format |
| 6 | Check repeated chars | Regex `(.)\1` | Find adjacent duplicates |
| 7 | Check sequential chars | `ASCII()`, `SUBSTR()` | Detect increasing sequences |
| 8 | Summarize results | `CASE`, `COUNT()` | Produce report |

---

## 🧮 SQL Logic Summary

Below is the condensed version of the validation query used:

WITH cleaned_data AS (
  
  SELECT 
    TRIM(UPPER(COALESCE(pan_number, ''))) AS pan_number
  FROM stg_pan_number
  WHERE pan_number IS NOT NULL
),

unique_data AS (
  SELECT pan_number, 
         ROW_NUMBER() OVER (PARTITION BY pan_number ORDER BY pan_number) AS rn
  FROM cleaned_data
),

validated AS (
  
  SELECT 
    
  pan_number,
    
   CASE
  
  WHEN pan_number !~ '^[A-Z]{5}[0-9]{4}[A-Z]$' THEN 'Invalid Format'
  
  WHEN substring(pan_number,1,5) ~ '(.)\1' THEN 'Invalid – Adjacent alphabets repeat'
      
  WHEN substring(pan_number,6,4) ~ '(.)\1' THEN 'Invalid – Adjacent digits repeat'
      
  WHEN (
        
  ascii(substr(pan_number,2,1)) = ascii(substr(pan_number,1,1)) + 1 AND
  
  ascii(substr(pan_number,3,1)) = ascii(substr(pan_number,2,1)) + 1 AND
  
  ascii(substr(pan_number,4,1)) = ascii(substr(pan_number,3,1)) + 1 AND
  
  ascii(substr(pan_number,5,1)) = ascii(substr(pan_number,4,1)) + 1
  
  ) THEN 'Invalid – Sequential alphabets'
      
  WHEN (
    
  CAST(substr(pan_number,7,1) AS int) = CAST(substr(pan_number,6,1) AS int) + 1 AND
  
  CAST(substr(pan_number,8,1) AS int) = CAST(substr(pan_number,7,1) AS int) + 1 AND
  
  CAST(substr(pan_number,9,1) AS int) = CAST(substr(pan_number,8,1) AS int) + 1
  
  ) THEN 'Invalid – Sequential digits'
  
  ELSE 'Valid PAN'
  
  END AS validation_result
  
  FROM unique_data
  
  WHERE rn = 1
)

SELECT 
  
  COUNT(*) AS total_records,
  
  SUM(CASE WHEN validation_result = 'Valid PAN' THEN 1 ELSE 0 END) AS total_valid,
  
  SUM(CASE WHEN validation_result <> 'Valid PAN' THEN 1 ELSE 0 END) AS total_invalid

FROM validated;

## Key SQL functions used — purpose & why

Below are the main PostgreSQL functions and constructs used in the project, with short explanations:
	
  •	TRIM(text)

Purpose: remove leading/trailing spaces.

Why: input may have spaces; normalization prevents false invalidation.

  •	COALESCE(expr, alt)

Purpose: replace NULL with something (e.g., 'Unknown').

Why: safe handling of missing PAN values before deduplication.
	
  •	UPPER(text)

Purpose: convert to uppercase.

Why: PANs must be uppercase — ensures consistent comparisons/regex.
	
  •	Window Functions: ROW_NUMBER() OVER (PARTITION BY ... ORDER BY ...)

Purpose: deduplicate while keeping one representative row.

Why: remove duplicates from raw staging set safely.
	
  •	Regular expression match ~ and patterns like ^[A-Z]{5}[0-9]{4}[A-Z]$

Purpose: enforce PAN basic structure (5 letters + 4 digits + 1 letter).

Why: fastest, concise format validation.
	
  •	Regex (.)\1 on substring

Purpose: detect adjacent identical characters (captures any char followed by same).

Why: business rule forbids adjacent same characters (letters & digits).
	
  •	substr(text, start, length)

Purpose: slice out specific character positions (e.g., first 5 letters).

Why: to run adjacency / ASCII checks on specific parts.
	
  •	ascii(char)

Purpose: numeric code for a character (A → 65, B → 66, …).

Why: checking sequences by verifying successive ascii values increment by 1.
	
  •	CAST(... AS int) or ::int

Purpose: convert digit characters to numeric for arithmetic checks.

Why: verify numeric sequence (e.g., 1,2,3,4 are consecutive).
	
  •	CASE ... WHEN ... THEN

Purpose: produce readable validation result strings for each PAN.

Why: provide business-friendly labels for valid/invalid reasons.
	
  •	Aggregation COUNT() / SUM(CASE WHEN ... THEN 1 END)

Purpose: produce the final summary report (total, valid, invalid).

Why: simple KPI for project result.


