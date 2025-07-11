---
mapped_pages:
  - https://www.elastic.co/guide/en/elasticsearch/reference/current/watch-cluster-status.html
applies_to:
  stack: ga
products:
  - id: elasticsearch
---

# Watching the status of an Elasticsearch cluster [watch-cluster-status]

You can easily configure a basic watch to monitor the health of your Elasticsearch cluster:

* [Schedule the watch and define an input](#health-add-input) that gets the cluster health status.
* [Add a condition](#health-add-condition) that evaluates the health status to determine if action is required.
* [Take action](#health-take-action) if the cluster is RED.

## Schedule the watch and add an input [health-add-input]

A watch [schedule](trigger-schedule.md) controls how often a watch is triggered. The watch [input](input.md) gets the data that you want to evaluate.

The simplest way to define a schedule is to specify an interval. For example, the following schedule runs every 10 seconds:

```console
PUT _watcher/watch/cluster_health_watch
{
  "trigger" : {
    "schedule" : { "interval" : "10s" } <1>
  }
}
```

1. Schedules are typically configured to run less frequently. This example sets the interval to 10 seconds to you can easily see the watches being triggered. Since this watch runs so frequently, don’t forget to [delete the watch](#health-delete) when you’re done experimenting.

To get the status of your cluster, you can call the [cluster health API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-cluster-health):

```console
GET _cluster/health?pretty
```

To load the health status into your watch, you simply add an [HTTP input](input-http.md) that calls the cluster health API:

```console
PUT _watcher/watch/cluster_health_watch
{
  "trigger" : {
    "schedule" : { "interval" : "10s" }
  },
  "input" : {
    "http" : {
      "request" : {
        "host" : "localhost",
        "port" : 9200,
        "path" : "/_cluster/health"
      }
    }
  }
}
```

If you’re using Security, then you’ll also need to supply some authentication credentials as part of the watch configuration:

```console
PUT _watcher/watch/cluster_health_watch
{
  "trigger" : {
    "schedule" : { "interval" : "10s" }
  },
  "input" : {
    "http" : {
      "request" : {
        "host" : "localhost",
        "port" : 9200,
        "path" : "/_cluster/health",
        "auth": {
          "basic": {
            "username": "elastic",
            "password": "x-pack-test-password"
          }
        }
      }
    }
  }
}
```

It would be a good idea to create a user with the minimum privileges required for use with such a watch configuration.

Depending on how your cluster is configured, there may be additional settings required before the watch can access your cluster such as keystores, truststores, or certificates. For more information, see [{{watcher}} settings](elasticsearch://reference/elasticsearch/configuration-reference/watcher-settings.md).

If you check the watch history, you’ll see that the cluster status is recorded as part of the `watch_record` each time the watch executes.

For example, the following request retrieves the last ten watch records from the watch history:

```console
GET .watcher-history*/_search
{
  "sort" : [
    { "result.execution_time" : "desc" }
  ]
}
```

## Add a condition [health-add-condition]

A [condition](condition.md) evaluates the data you’ve loaded into the watch and determines if any action is required. Since you’ve defined an input that loads the cluster status into the watch, you can define a condition that checks that status.

For example, you could add a condition to check to see if the status is RED.

```console
PUT _watcher/watch/cluster_health_watch
{
  "trigger" : {
    "schedule" : { "interval" : "10s" } <1>
  },
  "input" : {
    "http" : {
      "request" : {
       "host" : "localhost",
       "port" : 9200,
       "path" : "/_cluster/health"
      }
    }
  },
  "condition" : {
    "compare" : {
      "ctx.payload.status" : { "eq" : "red" }
    }
  }
}
```

1. Schedules are typically configured to run less frequently. This example sets the interval to 10 seconds to you can easily see the watches being triggered.

If you check the watch history, you’ll see that the condition result is recorded as part of the `watch_record` each time the watch executes.

To check to see if the condition was met, you can run the following query.

```console
GET .watcher-history*/_search?pretty
{
  "query" : {
    "match" : { "result.condition.met" : true }
  }
}
```

## Take action [health-take-action]

Recording `watch_records` in the watch history is nice, but the real power of {{watcher}} is being able to do something in response to an alert. A watch’s [actions](actions.md) define what to do when the watch condition is true—you can send emails, call third-party webhooks, or write documents to an Elasticsearch index or log when the watch condition is met.

For example, you could add an action to index the cluster status information when the status is RED.

```console
PUT _watcher/watch/cluster_health_watch
{
  "trigger" : {
    "schedule" : { "interval" : "10s" }
  },
  "input" : {
    "http" : {
      "request" : {
       "host" : "localhost",
       "port" : 9200,
       "path" : "/_cluster/health"
      }
    }
  },
  "condition" : {
    "compare" : {
      "ctx.payload.status" : { "eq" : "red" }
    }
  },
  "actions" : {
    "send_email" : {
      "email" : {
        "to" : "username@example.org",
        "subject" : "Cluster Status Warning",
        "body" : "Cluster status is RED"
      }
    }
  }
}
```

For {{watcher}} to send email, you must configure an email account in your [`elasticsearch.yml`](/deploy-manage/stack-settings.md) configuration file and restart {{es}}. To add an email account, set the `xpack.notification.email.account` property.

For example, the following snippet configures a single Gmail account named `work`:

```yaml
xpack.notification.email.account:
  work:
    profile: gmail
    email_defaults:
      from: <email> <1>
    smtp:
      auth: true
      starttls.enable: true
      host: smtp.gmail.com
      port: 587
      user: <username> <2>
      password: <password> <3>
```

1. Replace `<email>` with the email address from which you want to send notifications.
2. Replace `<username>` with your Gmail user name (typically your Gmail address).
3. Replace `<password>` with your Gmail password.

::::{note}
If you have advanced security options enabled for your email account, you need to take additional steps to send email from {{watcher}}. For more information, see [Configuring email accounts](actions-email.md#configuring-email).
::::

You can check the watch history or the `status_index` to see that the action was performed.

```console
GET .watcher-history*/_search?pretty
{
  "query" : {
    "match" : { "result.condition.met" : true }
  }
}
```

## Delete the watch [health-delete]

Since the `cluster_health_watch` is configured to run every 10 seconds, make sure you delete it when you’re done experimenting. Otherwise, you’ll spam yourself indefinitely.

To remove the watch, use the [delete watch API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-watcher-delete-watch):

```console
DELETE _watcher/watch/cluster_health_watch
```
