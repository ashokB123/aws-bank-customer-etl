# AWS Bank Customer ETL: S3, Glue Data Catalog, Athena, Parquet

A small data pipeline on AWS that takes a raw bank customer CSV, cleans it with SQL, stores it as compressed Parquet in S3, and answers a few business questions on the result.

## Architecture

CSV in S3 (raw bucket) -> Glue Data Catalog table -> Athena CTAS query (clean + convert) -> Parquet (Snappy) in S3 (processed bucket) -> Athena analysis queries

## Dataset

UCI Bank Marketing dataset (bank.csv): 4,521 customers, 17 columns. The file is semicolon-separated, has a header row, and wraps text values in double quotes.

## What I built

1. Created two S3 buckets, one for raw data and one for processed data, and an IAM role for Glue.
2. Created a Glue database and a source table over the raw CSV. I used the OpenCSVSerde with separatorChar `;` and quoteChar `"`, and set `skip.header.line.count = 1`, so the quotes and header row don't end up in the data.
3. Wrote an Athena CTAS query (`sql/01_create_processed_table.sql`) that:
   - casts age and balance to integers
   - drops columns not needed for the analysis (contact, day, month, duration, campaign, pdays, previous, poutcome, y)
   - removes rows with an unknown job or education
   - writes the result as Snappy-compressed Parquet to the processed bucket and registers it as a new table
4. Ran analysis queries on the processed table (`sql/02_analysis_queries.sql`).

## Results

- 4,521 raw rows -> 4,311 cleaned rows (210 rows, about 4.6%, removed for unknown job or education)
- Retired customers have the highest average balance (2,388), followed by housemaid (1,981) and management (1,749). Services (1,041) and unemployed (1,045) are lowest.
- Customers with secondary education carry the most debt: 18.1% have a personal loan and 62.2% have a housing loan, versus 13.0% and 49.4% for tertiary education.
- Average balance rises with age: customers 60+ average 2,857, about 2.8 times the under-30 average of 1,035. The 60+ group has only 156 customers, so this is a pattern in a small group.

## Screenshots

See the `screenshots/` folder: raw file in S3, Glue table, raw data in Athena, successful CTAS query, Parquet output in S3, row count, and analysis results.

## Notes and next steps

- IAM: I used AWS managed policies (AmazonS3FullAccess, AWSGlueServiceRole, AmazonAthenaFullAccess) to keep the setup simple. A production setup should use least-privilege custom policies.
- I planned a Glue PySpark ETL job for the transformation, but AWS blocked Glue job creation on my account, so the cleaning was done with an Athena CTAS query instead. A PySpark version is a next step once that is resolved.
- Balances are shown without a currency because the dataset doesn't state one.

## Tools

Amazon S3, AWS Glue Data Catalog, Amazon Athena, AWS IAM, SQL, Parquet