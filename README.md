# QueryStats

Figure out how your Athena queries are performing over time

## Overview

It can sometimes be difficult to figure out if you're running into concurrency limits with Athena or if you have queries scanning more data than expected.

This tool extracts your Athena execution history and uploads it back into S3 so you can analyze it with Athena. 😲

## Analyzing in Athena

What we really want is an image of queries per second or minute.

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


WITH runtimes AS (
    SELECT QueryExecutionId,status.state,
    from_iso8601_timestamp(status.submissiondatetime) AS submission_ts,
    from_iso8601_timestamp(status.completiondatetime) AS completion_ts,
    statistics.engineexecutiontimeinmillis, statistics.datascannedinbytes,
    date_diff('millisecond', from_iso8601_timestamp(status.submissiondatetime), from_iso8601_timestamp(status.completiondatetime)) AS runtime_diff
    FROM athena_queries
    WHERE StatementType = 'DML'
)

SELECT *, runtime_diff - engineexecutiontimeinmillis AS queue_time
FROM runtimes
```

This kind of gets us what we want, but it's not ideal.

```sql
WITH runtimes AS (
    SELECT QueryExecutionId,status.state,
    from_iso8601_timestamp(status.submissiondatetime) AS submission_ts,
    from_iso8601_timestamp(status.completiondatetime) AS completion_ts,
    statistics.engineexecutiontimeinmillis, statistics.datascannedinbytes,
    date_diff('millisecond', from_iso8601_timestamp(status.submissiondatetime), from_iso8601_timestamp(status.completiondatetime)) AS runtime_diff
    FROM athena_queries
    WHERE StatementType = 'DML'
)

SELECT r1.QueryExecutionId, r1.submission_ts, r1.completion_ts, 
  r2.QueryExecutionId, r2.submission_ts, r2.completion_ts
FROM runtimes r1
LEFT JOIN runtimes r2 ON (
  r2.submission_ts >= r1.submission_ts AND
  r2.submission_ts <= r1.completion_ts AND
  r1.QueryExecutionId != r2.QueryExecutionId
)
```

Cool, this is really close. :)
Need to convert to minutes.

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

seconds AS (
  SELECT
    CAST(date_column AS TIMESTAMP) date_column
  FROM
    (VALUES
        (SEQUENCE(FROM_ISO8601_DATE('2019-02-11'), 
                  FROM_ISO8601_DATE('2019-02-12'), 
                  INTERVAL '1' HOUR)
        )
    ) AS t1(date_array)
  CROSS JOIN
    UNNEST(date_array) AS t2(date_column)
)

SELECT date_column, COUNT(*) FROM seconds s
LEFT JOIN runtimes r ON (date_column>=date_trunc('hour',r.submission_ts) AND date_column<=date_trunc('hour',r.completion_ts))

GROUP BY 1
```

- Show me all seconds in a day where more than 1 query was running

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

seconds AS (
  SELECT
    CAST(date_column AS TIMESTAMP) date_column
  FROM
    (VALUES
        (SEQUENCE(FROM_ISO8601_DATE('2019-02-11'), 
                  FROM_ISO8601_DATE('2019-02-12'), 
                  INTERVAL '1' SECOND)
        )
    ) AS t1(date_array)
  CROSS JOIN
    UNNEST(date_array) AS t2(date_column)
)

SELECT date_column, COUNT(r.QueryExecutionId) AS queries_running FROM seconds s
LEFT JOIN runtimes r ON (date_column>=date_trunc('second',r.submission_ts) AND date_column<=date_trunc('second',r.completion_ts))
GROUP BY 1
HAVING COUNT(*) > 1
ORDER BY 1 ASC
```

- Here's a similar query, but (possibly) less complex.

I haven't compared them yet to validate which approach is correct.
The one above is asking "were there queries running during this specific interval.
The one below is saying "this query was running during this interval"

Trying to compare them, they have similar times but totally different outputs :\

Running this on the one below shows that our logic below is flawed. We should have 3 queries running at that time.
Because I had a `LIMIT 10` in the CTE. Derp.

One below is missing data because INTERVAL copmmand is exclusive(?)
Yep - https://prestodb.github.io/docs/current/functions/array.html

```
2019-02-11 00:02:24.388 -08:00	2019-02-11 00:02:28.194 -08:00	[2019-02-11 08:02:24.388, 2019-02-11 08:02:25.388, 2019-02-11 08:02:26.388, 2019-02-11 08:02:27.388]
```

```sql
select * from runtimes
where submission_ts >= timestamp '2019-02-11 04:02:36.000' AND submission_ts <= timestamp '2019-02-11 06:02:36.000'
order by submission_ts asc
```

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
  SELECT SEQUENCE(cast(submission_ts as timestamp), cast(completion_ts as timestamp), INTERVAL '1' second) AS intervals
  FROM runtimes
  LIMIT 10
)

SELECT date_trunc('second',interval_marker), count(*) FROM query_intervals
CROSS JOIN UNNEST(intervals) AS t (interval_marker)
GROUP BY 1
```

OK, had to do some date truncation things to ensure we were inclusive of queries crossing time boundaries.

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

At the end of the day, the second one will always be more performant because the first one is generating a ton of data
(one row for every second 0_o) that it then ends up throwing away. 

`(Run time: 2.95 seconds, Data scanned: 267.91 KB)`

vs

`(Run time: 6 minutes 38 seconds, Data scanned: 267.94 KB)`

## TODO

- [ ] Extract Athena API functionality into it's own class
  - [ ] Progress reporter based on current date / 45-day history
  - [ ] Track the most recent Query ID so we can resume
- [ ] Convert into Parquet on the fly
- [ ] Option to leave out `Query` field
- [ ] Deliver in partitioned keys
- [ ] Option to automatically create Athena table