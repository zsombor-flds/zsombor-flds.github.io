---
date: 2024-10-13

showTableOfContents: false
tags: ["dbt",]
title: "Microbatch Incremental Models in dbt"
type: "post"
---
Working with large-scale time-series datasets often presents challenges in maintaining a balance between efficiency, resiliency, and simplicity within data pipelines. To address these challenges, dbt is introducing a new feature called: **Microbatch Incremental Models**.
This strategy is designed to enhance the way you handle incremental data loads, making your data pipelines more efficient and robust. In the upcoming paragraphs, I collected my thoughts on the benefits of using this strategy.

![captionless image](https://miro.medium.com/v2/resize:fit:878/format:webp/1*sIUxRvMq1efgKTvDvvWcDg.png)

## What is a Microbatch Incremental Model?


In dbt, incremental models are a powerful materialization strategy that allows you to update your data warehouse tables by processing only the new or changed/modified data since the last run. Traditionally, this manifests into adding new rows or updating existing ones without reprocessing the entire dataset (defined by the selected [incremental strategy](https://docs.getdbt.com/docs/build/incremental-strategy))

Microbatch incremental models take this concept a step further.
Instead of processing all new data in one go, microbatching splits the data into smaller, more atomic batches based on a time-based partitioning scheme (e.g : daily). Each batch corresponds to a specific time window and is processed independently.
This approach offers several benefits:

*   **Efficiency**: By processing smaller chunks of data, you can optimize resource usage and reduce processing time.
*   **Resiliency**: If a batch fails, you can retry it without reprocessing other batches.
*   **Concurrency**: In future updates, batches can be processed concurrently, further speeding up data transformations.
*   **Idempotency**: Since each batch is independent, reprocessing a batch should yield the same result.

## Enabling Microbatch in dbt


As of writing this post, the microbatch strategy is available in [**dbt Core v1.9**](https://github.com/dbt-labs/dbt-core/discussions/10672). To enable it, set the environment variable:

```
DBT_EXPERIMENTAL_MICROBATCH=True
```

## How Microbatch Works


![captionless image](https://miro.medium.com/v2/resize:fit:968/format:webp/1*qbEiEhyQK41WvvYML8MxEQ.png)

When you run a microbatch model, dbt:

By default, the time window is set to **daily** batches, but you can configure this granularity based on your needs. With large events like clickstream, I saw performance benefits with hourly batches as well.

When writing your model query:

*   **Process One Batch at a Time**: Your query should handle data for a single batch only.
*   **No Need for** `**is_incremental()**` **Conditions**: Microbatching abstracts away the need for these conditions.
*   **Auto-Filtering of Upstream Models**: dbt automatically filters upstream models based on the `event_time` config.
*   **Handle Late arriving record:** you can configure a loopback window for events outside the daily interval.

A sample Incremental with the Microbatch strategy:

```
{{ config(
    materialized='incremental',
    incremental_strategy='microbatch',
    event_time='event_occurred_at',
    begin='2024-01-01',
    batch_size='day',
    lookback=3,
    full_refresh=False
) }}
/* This is automatically filtered by the
event_time(event_occurred_at), with the window defined by batch_size
*/
select * from {{ ref('stg_events') }} 
```

## Benefits compared to the legacy Incremental strategies


Traditional incremental models in dbt require you to define what constitutes “new” data using a `is_incremental()` conditional block. You're responsible for crafting SQL that accurately filters new or updated records.

Microbatch incremental models simplify this by:

*   **Eliminating the Need for Conditional Logic**: No `is_incremental()` blocks are necessary.
*   **Automating Data Partitioning**: dbt handles the batching based on your `event_time` and `batch_size`.
*   **Enhancing Efficiency and Resiliency**: Independent batch processing allows for better error handling and potential concurrent execution.

Side note for Snowflake Users
-----------------------------

Although you can gain these benefits by leveraging microbatches in most adapters. If you are on Snowflake I would highly suggest testing the [dbt-incremental-stream](https://github.com/arnoN7/dbt-incremental-stream) package first. The incr_stream package uses [Snowflake Stream And Task](https://docs.snowflake.com/en/user-guide/streams-intro) which can significantly boost your dbt model loading window.

## Conclusion


The microbatch incremental strategy in dbt is a significant advancement in handling time-series data transformations. By breaking down data processing into independent, idempotent batches, you can gain benefits in the names of efficiency, resiliency, and scalability in your dbt models.
Now we just need to finally resolve the issues with Snapshots — but that’s a whole different story for another day.
You can read more details about the dbt’s microbatch strategy in the official documentation [here](https://docs.getdbt.com/docs/build/incremental-strategy).