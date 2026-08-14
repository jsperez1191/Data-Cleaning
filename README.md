# Layoffs Data Cleaning (SQL)

A SQL project that cleans a raw tech-industry layoffs dataset — removing duplicates, standardizing inconsistent values, handling nulls, and dropping unusable rows/columns — to produce an analysis-ready table.

## Overview

The raw `layoffs` table contains company layoff records (company, location, industry, total laid off, percentage laid off, date, funding stage, country, funds raised). Before this data can be used for analysis, it needs cleaning: duplicate rows, inconsistent text formatting, mixed date formats, and missing values all need to be resolved.

This project walks through that cleaning process end-to-end in MySQL, moving from the raw `layoffs` table to a fully cleaned `layoffs_staging2` table.

## Tech Stack

- **Database:** MySQL (uses `ROW_NUMBER() OVER()`, window functions, `STR_TO_DATE`)
- **Tool:** Any MySQL client (MySQL Workbench, CLI, etc.)

## Cleaning Steps

1. **Remove Duplicates**
   - Staged the raw data into `layoffs_staging` to avoid modifying the source table.
   - Used `ROW_NUMBER()` partitioned across all columns to flag exact duplicate rows.
   - Copied the result (with the row-number flag) into `layoffs_staging2` and deleted rows where `row_num > 1`.

2. **Standardize the Data**
   - Trimmed whitespace from `company` names.
   - Consolidated inconsistent `industry` values (e.g. `Crypto`, `Crypto Currency`, `CryptoCurrency` → `Crypto`).
   - Removed trailing periods from `country` values (e.g. `United States.` → `United States`).
   - Converted the `date` column from text (`%m/%d/%Y`) to a proper SQL `DATE` type using `STR_TO_DATE`, then altered the column type.

3. **Handle Null / Blank Values**
   - Identified rows with missing `industry` values and backfilled them by matching against other rows for the same `company` and `location` that had an industry populated (e.g. Airbnb → `Travel`).
   - Left `total_laid_off` / `percentage_laid_off` nulls in place where no reasonable substitute existed, since inventing numbers would misrepresent the data.

4. **Remove Unusable Rows/Columns**
   - Deleted rows where both `total_laid_off` and `percentage_laid_off` were `NULL`, since they carry no usable layoff metric.
   - Dropped the helper `row_num` column once deduplication was complete, since it was only needed for the cleaning process itself.

## Result

`layoffs_staging2` is the final cleaned table: deduplicated, consistently formatted, with backfilled industries where possible and unusable rows removed — ready for downstream analysis or visualization.

## How to Run

1. Load the raw dataset into a `layoffs` table in your MySQL instance.
2. Run the script top to bottom — it stages the data, deduplicates, standardizes, and cleans nulls in order.
3. Query `layoffs_staging2` for the cleaned result.

## Notes

- The original `layoffs` and `layoffs_staging` tables are left untouched throughout, so the raw data is always recoverable if a step needs to be redone.
- `ROW_NUMBER()` is used instead of `DISTINCT` for deduplication since it allows inspecting duplicates before deleting them.

## Exploratory Data Analysis

With `layoffs_staging2` cleaned, the second half of the project explores the data to surface patterns in layoffs across companies, countries, industries, and time.

- **Scale of layoffs:** Checked the largest single layoffs by count and by percentage (`MAX(total_laid_off)`, `MAX(percentage_laid_off)`), and pulled companies where `percentage_laid_off = 1` (i.e. the whole company was laid off), ranked by funds raised, to see which well-funded companies still shut down entirely.
- **By company:** Summed `total_laid_off` grouped by `company` to find which companies cut the most jobs overall, and broke that down further by `company` + `YEAR(date)` to see year-over-year patterns per company.
- **By geography:** Summed layoffs by `country`, and calculated average `percentage_laid_off` by `country` to compare the scale vs. severity of layoffs across regions.
- **By time:** Found the date range of the dataset (`MIN`/`MAX` of `date`), summed layoffs by `YEAR(date)`, and grouped by year-month (`SUBSTRING(date, 1, 7)`) to see monthly trends.
- **By funding stage:** Summed layoffs grouped by `stage` (e.g. Post-IPO, Series C) to see whether layoffs were concentrated in earlier- or later-stage companies.
- **Rolling total:** Built a CTE that sums monthly layoffs, then used `SUM() OVER(ORDER BY MONTH)` as a window function to calculate a running total of layoffs over time.
- **Top companies per year:** Used a two-layer CTE — first aggregating `total_laid_off` by `company` and `YEAR(date)`, then applying `DENSE_RANK() OVER(PARTITION BY years ORDER BY total_laid_off DESC)` — to identify the top 5 companies with the most layoffs in each year.

### Skills Demonstrated

- Data cleaning: deduplication with window functions, text/date standardization, null handling
- Aggregation: `GROUP BY`, `SUM`, `AVG`, `MAX`/`MIN` across multiple dimensions
- Window functions: `ROW_NUMBER()`, `DENSE_RANK()`, rolling totals with `SUM() OVER()`
- CTEs, including layered/nested CTEs for multi-step analysis
- Time-series analysis (yearly and monthly trends)
