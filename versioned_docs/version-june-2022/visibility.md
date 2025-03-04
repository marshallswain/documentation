---
id: visibility
title: Visibility
sidebar_label: Visibility
description: This guide is meant to be a comprehensive overview of Temporal Visibility.
toc_max_heading_level: 4
---

<!-- THIS FILE IS GENERATED. DO NOT EDIT THIS FILE DIRECTLY -->

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

This guide is meant to be a comprehensive overview of Temporal Visibility.

The term Visibility, within the Temporal Platform, refers to the subsystems and APIs that enable an operator to view Workflow Executions that currently exist within a Cluster.

## Standard Visibility

Standard Visibility, within the Temporal Platform, is the subsystem and APIs that list Workflow Executions by a predefined set of filters.

Open Workflow Executions can be filtered by a time constraint and either a Workflow Type, Workflow Id, or Run Id.

Closed Workflow Executions can be filtered by a time constraint and either a Workflow Type, Workflow Id, Run Id, or Execution Status (Completed, Failed, Timed Out, Terminated, Canceled, or Continued-As-New).

## Advanced Visibility

Advanced Visibility, within the Temporal Platform, is the subsystem and APIs that enable the listing, filtering, and sorting of Workflow Executions through a custom SQL-like [List Filter](#list-filters).

To use Advanced Visibility, your Temporal Cluster must be [integrated with Elasticsearch](/cluster-deployment-guide/#advanced-visibility).
We highly recommend operating a Temporal Cluster with Elasticsearch for any use case that spawns more than just a few Workflow Executions.
Elasticsearch takes on the Visibility request load, relieving potential performance issues.

### List Filters

A List Filter is the SQL-like string that is provided as the parameter to an [Advanced Visibility](#advanced-visibility) List API.

- [How to use a List Filter using tctl](/tctl/workflow/list/#--query)

The following is an example List Filter:

```
WorkflowType = "main.YourWorkflowDefinition" and ExecutionStatus != "Running" and (StartTime > "2021-06-07T16:46:34.236-08:00" or CloseTime > "2021-06-07T16:46:34-08:00") order by StartTime desc
```

[More example List Filters](#example-list-filters)

A List Filter contains [Search Attribute](#search-attributes) names, Search Attribute values, and Operators.

- The following operators are supported in List Filters:

  - **AND, OR, ()**
  - **=, !=, >, >=, <, <=**
  - **IN**
  - **BETWEEN ... AND**
  - **ORDER BY**

- A List Filter applies to a single Namespace.

- The range of a List Filter timestamp (StartTime, CloseTime, ExecutionTime) cannot exceed 9223372036854775807 (that is, maxInt64 - 1001).

- A List Filter that uses a time range has a resolution of 1 ms on Elasticsearch 6 and 1 ns on Elasticsearch 7.

- List Filter Search Attribute names are case sensitive.

- An Advanced List Filter API may take longer than expected if it is retrieving a large number of Workflow Executions (more than 10 million).

- A `ListWorkflow` API supports pagination.
  Use the page token in the following call to retrieve the next page; continue until the page token is `null`/`nil`.

- To efficiently count the number of Workflow Executions, use the `CountWorkflow` API.

#### Example List Filters

```sql
WorkflowId = '<workflow-id>'
```

```sql
WorkflowId = '<workflow-id>' or WorkflowId = '<another-workflow-id>'
```

```sql
WorkflowId = '<workflow-id>' order by StartTime desc
```

```sql
WorkflowId = '<workflow-id>' and ExecutionStatus = 'Running'
```

```sql
WorkflowId = '<workflow-id>' or ExecutionStatus = 'Running'
```

```sql
WorkflowId = '<workflow-id>' and StartTime > '2021-08-22T15:04:05+00:00'
```

```sql
ExecutionTime between '2021-08-22T15:04:05+00:00' and '2021-08-28T15:04:05+00:00'
```

```sql
ExecutionTime < '2021-08-28T15:04:05+00:00' or ExecutionTime > '2021-08-22T15:04:05+00:00'
```

```sql
order by ExecutionTime
```

```sql
order by StartTime desc, CloseTime asc
```

```sql
order by CustomIntField asc
```

### Search Attributes

A Search Attribute is an indexed field used in a [List Filter](#list-filters) to filter a list of Workflow Executions that have the Search Attribute in their metadata.

If a Temporal Cluster does not have Elasticsearch integrated, but a Workflow Execution is spawned and tagged with Search Attributes, no errors occur.
However, you won't be able to use Advanced Visibility List APIs and List Filters to find and list the Workflow Execution.

When using [Continue-As-New](/workflows/#continue-as-new) or a [Temporal Cron Job](/workflows/#cron-jobs), Search Attributes are carried over to the new Run by default.

#### Default Search Attributes

A Temporal Cluster that is integrated with Elasticsearch has a set of default Search Attributes already available.
These Search Attributes are created when the initial index is created.

| NAME                  | TYPE     | DEFINITION                                                                                                                                                                   |
| --------------------- | -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| WorkflowType          | Keyword  | The type of Workflow.                                                                                                                                                        |
| WorkflowId            | Keyword  | Identifies the Workflow Execution.                                                                                                                                           |
| ExecutionStatus       | Keyword  | The current state of the Workflow Execution.                                                                                                                                 |
| StartTime             | Datetime | The time at which the Workflow Execution started.                                                                                                                            |
| CloseTime             | Datetime | The time at which the Workflow Execution completed.                                                                                                                          |
| ExecutionTime         | Datetime | Same as StartTime for the most cases but different for cron Workflows and retried Workflows. For them it is the time at which the Workflow Execution actually begin running. |
| RunId                 | Keyword  | Identifies the current Workflow Execution Run.                                                                                                                               |
| ExecutionDuration     | Int      | The time needed to run the Workflow Execution. Available only for closed Workflows.                                                                                          |
| HistoryLength         | Int      | The number of events in the history of Workflow Execution. Available only for closed Workflows.                                                                              |
| StateTransitionCount  | Int      | The number of times that Workflow Execution has persisted its state. Available only for closed Workflows.                                                                    |
| TaskQueue             | Keyword  | Task Queue used by Workflow Execution.                                                                                                                                       |
| TemporalChangeVersion | Keyword  | If workflow versioning is enabled, list of change/version pairs will be stored here.                                                                                         |
| BinaryChecksums       | Keyword  | List of binary Ids of Workers that run the Workflow Execution.                                                                                                               |
| BatcherNamespace      | Keyword  | Used by internal batcher to indicate the Namespace where batch operation was applied to.                                                                                     |
| BatcherUser           | Keyword  | Used by internal batcher to indicate the user who started the batch operation.                                                                                               |

- All default Search Attributes are reserved and read-only.
  (You cannot create a custom one with the same name or alter the existing one.)

- ExecutionStatus values correspond to Workflow Execution Statuses: Running, Completed, Failed, Canceled, Terminated, ContinuedAsNew, TimedOut.

- StartTime, CloseTime, and ExecutionTime are stored as dates but are supported by queries that use either EpochTime in nanoseconds or a string in [RFC3339Nano format](https://pkg.go.dev/time#pkg-constants) (such as "2006-01-02T15:04:05.999999999Z07:00").

- ExecutionDuration is stored in nanoseconds but is supported by queries that use integers in nanoseconds, [Golang duration format](https://pkg.go.dev/time#ParseDuration), or "hh:mm:ss" format.

- CloseTime, HistoryLength, StateTransitionCount, and ExecutionDuration are present only in a Closed Workflow Execution.

- ExecutionTime can differ from StartTime in retry and cron use cases.

#### Custom Search Attributes

Custom Search Attributes can be [added to a Temporal Cluster only by using `tctl`](/tctl/how-to-add-a-custom-search-attribute-to-a-cluster-using-tctl).
Adding a Search Attribute makes it available to use with Workflow Executions within that Cluster.

There is no hard limit on the number of attributes you can add.
However, we recommend enforcing the following limits:

- Number of Search Attributes: 100 per Workflow
- Size of each value: 2 KB per value
- Total size of names and values: 40 KB per Workflow

:::note

Due to Elasticsearch limitations, you can only add Search Attributes.
It is not possible to rename Search Attributes or remove them from the index schema.

:::

The [temporalio/auto-setup](https://hub.docker.com/r/temporalio/auto-setup) Docker image uses a pre-defined set of custom Search Attributes that are handy for testing.
Their names indicate their types:

- CustomTextField
- CustomKeywordField
- CustomIntField
- CustomDoubleField
- CustomBoolField
- CustomDatetimeField

#### Types

Search Attributes must be one of the following types:

- Text
- Keyword
- Int
- Double
- Bool
- Datetime

Note:

- **Double** is backed up by `scaled_float` Elasticsearch type with scale factor 10000 (4 decimal digits).
- **Datetime** is backed up by `date` type with milliseconds precision in Elasticsearch 6 and `date_nanos` type with nanoseconds precision in Elasticsearch 7.
- **Int** is 64-bit integer (`long` Elasticsearch type).
- **Keyword** and **Text** types are concepts taken from Elasticsearch. Each word in a **Text** is considered a searchable keyword.
  For a UUID, that can be problematic because Elasticsearch indexes each portion of the UUID separately.
  To have the whole string considered as a searchable keyword, use the **Keyword** type.
  For example, if the key `ProductId` has the value of `2dd29ab7-2dd8-4668-83e0-89cae261cfb1`:
  - As a **Keyword** it would be matched only by `ProductId = "2dd29ab7-2dd8-4668-83e0-89cae261cfb1`.
  - As a **Text** it would be matched by `ProductId = 2dd8`, which could cause unwanted matches.
- The **Text** type cannot be used in the "Order By" clause.

- [How to view Search Attributes using tctl](/tctl/cluster/list-search-attributes)

#### Search Attributes as Workflow Execution metadata

To actually have results from the use of a [List Filter](#list-filters), Search Attributes must be added to a Workflow Execution as metadata.
How to do this entirely depends on the method by which you spawn the Workflow Execution:

- [How to set Search Attributes as Workflow Execution metadata in Go](/go/startworkflowoptions-reference/#searchattributes)
