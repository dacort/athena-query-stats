# QueryStats

Figure out how your Athena queries are performing over time

## Overview

It can sometimes be difficult to figure out if you're running into concurrency limits with Athena or if you have queries scanning more data than expected.

This tool extracts your Athena execution history and uploads it back into S3 so you can analyze it with Athena. 😲

## Analyzing in Athena

- Create a table with a limited amount of columns that we need

```sql
CREATE EXTERNAL TABLE athena_queries (
    QueryExecutionId string,
    Query string,
    StatementType string,
    Status struct<State:string,SubmissionDateTime:string,CompletionDateTime:string>,
    Statistics struct<EngineExecutionTimeInMillis:int,DataScannedInBytes:int>
)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
LOCATION 's3://<BUCKET>/<PREFIX>' ;
```

- Determine how many queries are running at any given second

```sql
WITH runtimes AS (
    SELECT QueryExecutionId,status.state,
    from_iso8601_timestamp(status.submissiondatetime) AS submission_ts,
    from_iso8601_timestamp(status.completiondatetime) AS completion_ts,
    statistics.engineexecutiontimeinmillis, statistics.datascannedinbytes,
    date_diff('millisecond', from_iso8601_timestamp(status.submissiondatetime), from_iso8601_timestamp(status.completiondatetime)) AS runtime_diff
    FROM athena_queries
    WHERE StatementType = 'DML'
),

query_intervals AS (
  SELECT QueryExecutionId, submission_ts, completion_ts, SEQUENCE(
      date_trunc('second', cast(submission_ts as timestamp)),
      date_trunc('second', cast(completion_ts as timestamp)),
      INTERVAL '1' second
  ) AS intervals
  FROM runtimes
)

SELECT date_trunc('second',interval_marker), count(*) FROM query_intervals
CROSS JOIN UNNEST(intervals) AS t (interval_marker)
GROUP BY 1
ORDER BY 1 ASC
```

## TODO

- [ ] Extract Athena API functionality into it's own class
  - [ ] Progress reporter based on current date / 45-day history
  - [ ] Track the most recent Query ID so we can resume
- [ ] Convert into Parquet on the fly
- [ ] Option to leave out `Query` field
- [ ] Deliver in partitioned keys
- [ ] Option to automatically create Athena table