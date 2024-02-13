---
title: "Scraping metrics with Prometheus"
linkTitle: "Prometheus metrics"
weight: 30
---

To scrape metrics from Vertica, you currently need to setup a SQL exporter.  This exporter issues queries against Vertica, taking data from various DC tables and virtual tables.  This can be costly as it needs to go through the query plan optimizer. For complicated queries, this might result in joins from different tables. In version 23.3.0, Vertica introduced in-database metrics. Vertica is already very rich in metrics with the various system and DC tables that it offers, but now you can get them cheaply with Prometheus in-database metrics.

Vertica scrapes these metrics and outputs them in [Prometheus text-based exposition format](https://github.com/prometheus/docs/blob/main/content/docs/instrumenting/exposition_formats.md). This format applies context-specific labels to each metric so you can group metrics when you visualize your data. To export the metrics, we rely on the [HTTPS service](https://docs.vertica.com/latest/en/admin/managing-db/https-service/) which exposes general-purpose endpoints for interacting with your database. Vertica exposes metrics through the /v1/metrics endpoint so you can scrape them with a GET request. For example:

```shell
curl https://host:8443/v1/metrics \
    --key path/to/client_key.key \
    --cert path/to/client_cert.pem \
```

The following example outlines the output:
```shell
# HELP metric-name metric-definition
# TYPE metric-name metric-type
metric-name{label-key="label-value"[, ...]} metric-value
```

For example, the following is a snippet of the request response that provides details about the vertica_resource_pool_memory_size_actual_kb metric:

```shell
curl https://10.20.30.40:8443/v1/metrics \
    --key path/to/client_key.key \
    --cert path/to/client_cert.pem \
...
# HELP vertica_resource_pool_memory_size_actual_kb Current amount of memory (in kilobytes) allocated to the resource pool by the resource manager.
# TYPE vertica_resource_pool_memory_size_actual_kb gauge
vertica_resource_pool_memory_size_actual_kb{node_name="v_vmart_node0001",pool_name="metadata",revive_instance_id="114b25c4aab6fec8c26b121cff2b52"} 84381
vertica_resource_pool_memory_size_actual_kb{node_name="v_vmart_node0001",pool_name="blobdata",revive_instance_id="114b25c4aab6fec8c26b121cff2b52"} 0
vertica_resource_pool_memory_size_actual_kb{node_name="v_vmart_node0001",pool_name="jvm",revive_instance_id="114b25c4aab6fec8c26b121cff2b52"} 0
...
```

For a comprehensive list of metrics, see [Prometheus metrics](https://docs.vertica.com/latest/en/admin/managing-db/https-service/prometheus-metrics/).

By exposing the metrics through an HTTP endpoint, we can not only make it easy for Vertica to be a Prometheus target but also visualize the Vertica metrics through a Grafana dashboard.

[Prometheus](https://prometheus.io/) is a monitoring and alerting system that is very popular with cloud native deployments.  It collects time series events for different machines, called targets. In this case, the targets are Vertica nodes. To get metrics, Prometheus requires an exposed HTTP endpoint. If an endpoint is available, Prometheus can start scraping numerical data, capture it as a time series, and store it in a local database suited to time-series data. Prometheus is often combined with Grafana in order to visualize collected time-series data.

Grafana is an open-source solution that uses metrics to run analytics and provide insights into the complex infrastructure and massive amounts of data that services process. Grafana also provides customizable dashboards to visualize these analytics. It connects with every possible data source such as Graphite, Prometheus, Influx DB, ElasticSearch, MySQL, PostgreSQL, etc. You can now build dashboards in order to get the most from Vertica Prometheus metrics.

To help you get started, Vertica provides a set of dashboards in the new [grafana-dashboards](https://github.com/vertica/grafana-dashboards) project. The project is open-source, and Vertica [welcomes contributions](https://github.com/vertica/grafana-dashboards/blob/main/CONTRIBUTING.md).

So far it consists of four dashboards(EON database only) that are public and available on [Grafana](https://grafana.com/grafana/dashboards/):

- [Vertica Overview](https://grafana.com/grafana/dashboards/19917-vertica-overview-prometheus/): the overall state of your cluster.
- [Vertica Queries](https://grafana.com/grafana/dashboards/19915-vertica-queries-prometheus/): details about queries that are currently running in the cluster.
- [Vertica Depot](https://grafana.com/grafana/dashboards/19914-vertica-depot-prometheus/): details about depot usage.
- [Vertica Resource](https://grafana.com/grafana/dashboards/19916-vertica-resource-management-prometheus/) Management: details about user-defined and built-in resource pool usage.

You can find all Vertica Prometheus dashboards on [Grafana](https://grafana.com/grafana/dashboards/) by typing vertica on the search bar.