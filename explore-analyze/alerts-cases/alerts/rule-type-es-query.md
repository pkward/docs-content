---
navigation_title: "{{es}} query"
mapped_pages:
  - https://www.elastic.co/guide/en/kibana/current/rule-type-es-query.html
applies_to:
  stack: ga
  serverless: ga
products:
  - id: kibana
---

# Elasticsearch query [rule-type-es-query]

The {{es}} query rule type runs a user-configured query, compares the number of matches to a configured threshold, and schedules actions to run when the threshold condition is met.

In **{{stack-manage-app}}** > **{{rules-ui}}**, click **Create rule**. Select the **{{es}} query** rule type then fill in the name and optional tags. An {{es}} query rule can be defined using {{es}} Query Domain Specific Language (DSL), {{es}} Query Language (ES|QL), {{kib}} Query Language (KQL), or Lucene.

## Define the conditions [_define_the_conditions_2]

When you create an {{es}} query rule, your choice of query type affects the information you must provide. For example:

:::{image} /explore-analyze/images/kibana-rule-types-es-query-conditions.png
:alt: Define the condition to detect
:screenshot:
:::

1. Define your query

    * If you use [query DSL](../../query-filter/languages/querydsl.md), you must select an index and time field then provide your query. Only the `query`, `fields`, `_source` and `runtime_mappings` fields are used, other DSL fields are not considered. For example:

    ```sh
    {
        "query":{
          "match_all" : {}
        }
     }
    ```

    * If you use [KQL](../../query-filter/languages/kql.md) or [Lucene](../../query-filter/languages/lucene-query-syntax.md), you must specify a data view then define a text-based query. For example, `http.request.referrer: "https://example.com"`.

   * If you use [ES|QL](../../query-filter/languages/esql.md), you must provide a source command followed by an optional series of processing commands, separated by pipe characters (|).

        :::{admonition} Added in 8.16.0
        This functionality was added in 8.16.0.
        :::

        For example:

        ```sh
        FROM kibana_sample_data_logs
        | STATS total_bytes = SUM(bytes) BY host
        | WHERE total_bytes > 200000
        | SORT total_bytes DESC
        | LIMIT 10
        ```

2. Specify details for grouping alerts based on your query language.

    * If you use query DSL, KQL, or Lucene, set the group and theshold.

        When
        :   Specify how to calculate the value that is compared to the threshold. The value is calculated by aggregating a numeric field within the time window. The aggregation options are: `count`, `average`, `sum`, `min`, and `max`. When using `count` the document count is used and an aggregation field is not necessary.

        Over or Grouped Over
        :   Specify whether the aggregation is applied over all documents or split into groups using up to four grouping fields. If you choose to use grouping, it’s a [terms](elasticsearch://reference/aggregations/search-aggregations-bucket-terms-aggregation.md) or [multi terms aggregation](elasticsearch://reference/aggregations/search-aggregations-bucket-multi-terms-aggregation.md); an alert will be created for each unique set of values when it meets the condition. To limit the number of alerts on high cardinality fields, you must specify the number of groups to check against the threshold. Only the top groups are checked.

        Threshold
        :   Defines a threshold value and a comparison operator  (`is above`, `is above or equals`, `is below`, `is below or equals`, or `is between`). The value calculated by the aggregation is compared to this threshold.

    * {applies_to}`stack: ga 9.2` If you use {{esql}}, specify a time field and how to group alerts. 

        Time field
        :   Choose the time field to use when filtering query results by the time window that you later specify for the rule. You can choose any time field that's availble on the index you're querying, for example, the `@timestamp` field.

        Alert group
        :   Select **Create an alert if matches are found** to create a single alert for multiple events matching the {{esql}} query. Select **Create an alert for each row** to create a separate alert for each event that matches the {{esql}} query. Whenever possible, each alert is given a unique ID. 
 

3. Set the time window, which defines how far back to search for documents.
4. If you use query DSL, KQL, or Lucene, set the number of documents to send to the configured actions when the threshold condition is met.
5. If you use query DSL, KQL, or Lucene, choose whether to avoid alert duplication by excluding matches from the previous run. This option is not available when you use a grouping field.
6. Set the check interval, which defines how often to evaluate the rule conditions. Generally this value should be set to a value that is smaller than the time window, to avoid gaps in detection.
7. In the advanced options, you can change the number of consecutive runs that must meet the rule conditions before an alert occurs. The default value is `1`.
8. Select a scope value, which affects the [{{kib}} feature privileges](../../../deploy-manage/users-roles/cluster-or-deployment-auth/kibana-privileges.md#kibana-feature-privileges) that are required to access the rule. For example when it’s set to `Stack Rules`, you must have the appropriate **Management > {{stack-rules-feature}}** feature privileges to view or edit the rule.

## Test your query [_test_your_query]

Use the **Test query** feature to verify that your query is valid.

If you use query DSL, KQL, or Lucene, the query runs against the selected indices using the configured time window. The number of documents that match the query is displayed. For example:

:::{image} /explore-analyze/images/kibana-rule-types-es-query-valid.png
:alt: Test {{es}} query returns number of matches when valid
:screenshot:
:::

If you use an ES|QL query, a table is displayed. For example:

:::{image} /explore-analyze/images/kibana-rule-types-esql-query-valid.png
:alt: Test ES|QL query returns a table when valid
:screenshot:
:::

If the query is not valid, an error occurs.

## Add actions [_add_actions]

You can optionally send notifications when the rule conditions are met and when they are no longer met. In particular, this rule type supports:

* alert summaries
* actions that run when the query is matched
* recovery actions that run when the rule conditions are no longer met

For each action, you must choose a connector, which provides connection information for a {{kib}} service or third party integration. For more information about all the supported connectors, go to [*Connectors*](../../../deploy-manage/manage-connectors.md).

After you select a connector, you must set the action frequency. You can choose to create a summary of alerts on each check interval or on a custom interval. For example, send email notifications that summarize the new, ongoing, and recovered alerts at a custom interval:

:::{image} /explore-analyze/images/kibana-es-query-rule-action-summary.png
:alt: UI for defining alert summary action in an {{es}} query rule
:screenshot:
:::

Alternatively, you can set the action frequency such that actions run for each alert. Choose how often the action runs (at each check interval, only when the alert status changes, or at a custom action interval). You must also choose an action group, which indicates whether the action runs when the query is matched or when the alert is recovered. Each connector supports a specific set of actions for each action group. For example:

:::{image} /explore-analyze/images/kibana-es-query-rule-action-query-matched.png
:alt: UI for defining a recovery action
:screenshot:
:::

You can further refine the conditions under which actions run by specifying that actions only run when they match a KQL query or when an alert occurs within a specific time frame.

## Add action variables [_add_action_variables]

When you create a rule in {{kib}}, it provides an example message that is appropriate for each action. For example, the following message is provided for server log connector actions that run for each alert:

```handlebars
Elasticsearch query rule '{{rule.name}}' is active:

- Value: {{context.value}}
- Conditions Met: {{context.conditions}} over {{rule.params.timeWindowSize}}{{rule.params.timeWindowUnit}}
- Timestamp: {{context.date}}
- Link: {{context.link}}
```

Rules use rule action variables and Mustache templates to pass contextual details into the alert notifications. There is a set of [variables common to all rules](create-manage-rules.md#defining-rules-actions-variables) and a set that is specific to this rule. To view the list of variables in {{kib}}, click the "add rule variable" button. For example:

:::{image} /explore-analyze/images/kibana-es-query-rule-action-variables.png
:alt: Passing rule values to an action
:screenshot:
:::

The following variables are specific to the {{es}} query rule:

`context.conditions`
:   (string) A description of the condition. For example: `Query matched documents`.

`context.date`
:   (string) The date, in ISO format, that the rule met the condition. For example: `2024-04-30T00:55:42.765Z`.

`context.hits`
:   (array of objects) The most recent documents that matched the query. Using the [Mustache](https://mustache.github.io/) template array syntax, you can iterate over these hits to get values from the {{es}} documents into your actions. For example, the message in an email connector action might contain:

    ```handlebars
    Elasticsearch query rule '{{rule.name}}' is active:

    {{#context.hits}}
    Document with {{_id}} and hostname {{_source.host.name}} has
    {{_source.system.memory.actual.free}} bytes of memory free
    {{/context.hits}}
    ```

    The documents returned by `context.hits` include the [`_source`](elasticsearch://reference/elasticsearch/mapping-reference/mapping-source-field.md) field. If the {{es}} query search API’s [`fields`](elasticsearch://reference/elasticsearch/rest-apis/retrieve-selected-fields.md#search-fields-param) parameter is used, documents will also return the `fields` field, which can be used to access any runtime fields defined by the [`runtime_mappings`](../../../manage-data/data-store/mapping/define-runtime-fields-in-search-request.md) parameter. For example:

    ```handlebars
    {{#context.hits}}
    timestamp: {{_source.@timestamp}}
    day of the week: {{fields.day_of_week}} <1>
    {{/context.hits}}
    ```

    1. The `fields` parameter here is used to access the `day_of_week` runtime field.


    As the [`fields`](elasticsearch://reference/elasticsearch/rest-apis/retrieve-selected-fields.md#search-fields-response) response always returns an array of values for each field, the [Mustache](https://mustache.github.io/) template array syntax is used to iterate over these values in your actions. For example:

    ```handlebars
    {{#context.hits}}
    Labels:
    {{#fields.labels}}
    - {{.}}
    {{/fields.labels}}
    {{/context.hits}}
    ```

`context.link`
:   (string) The URL for the rule that generated the alert. For example: `/app/management/insightsAndAlerting/triggersActions/rule/47754354-d894-49d3-87ec-05745a74e2b7`.

`context.message`
:   (string) A preconstructed message for the rule. For example:<br> `Document count is 100 in the last 1h. Alert when greater than 50.`

`context.sourceFields`
:   (object) If the rule was configured to copy source fields into alerts, for each source field there is an array of strings that contains its values. For example: `{'host.id': ['1'], 'host.name': ['host-1']}`.

`context.title`
:   (string) A preconstructed title for the rule. Example: `rule 'my-query-rule' matched query`.

`context.value`
:   (number) The value that met the rule threshold condition.

`rule.params`
:   (object) The rule parameters, such as `searchType`, `timeWindowSize`, and `timeWindowUnit`. For the definitive list of parameters for this rule, refer to the API documentation.

## Handling multiple matches of the same document [_handling_multiple_matches_of_the_same_document]

By default, **Exclude matches from previous run** is turned on and the rule checks for duplication of document matches across multiple runs. If you configure the rule with a schedule interval smaller than the time window and a document matches a query in multiple runs, it is alerted on only once.

The rule uses the timestamp of the matches to avoid alerting on the same match multiple times. The timestamp of the latest match is used for evaluating the rule conditions when the rule runs. Only matches between the latest timestamp from the previous run and the current run are considered.

Suppose you have a rule configured to run every minute. The rule uses a time window of 1 hour and checks if there are more than 99 matches for the query. The {{es}} query rule type does the following:

|     |     |     |
| --- | --- | --- |
| `Run 1 (0:00)` | Rule finds 113 matches in the last hour: `113 > 99` | Rule is active and user is alerted. |
| `Run 2 (0:01)` | Rule finds 127 matches in the last hour. 105 of the matches are duplicates that were already alerted on previously, so you actually have 22 matches: `22 !> 99` | No alert. |
| `Run 3 (0:02)` | Rule finds 159 matches in the last hour. 88 of the matches are duplicates that were already alerted on previously, so you actually have 71 matches: `71 !> 99` | No alert. |
| `Run 4 (0:03)` | Rule finds 190 matches in the last hour. 71 of them are duplicates that were already alerted on previously, so you actually have 119 matches: `119 > 99` | Rule is active and user is alerted. |
