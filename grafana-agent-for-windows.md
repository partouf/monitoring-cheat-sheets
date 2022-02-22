# Grafana Agent for Windows

How to install the Agent on Windows and enable metrics to monitor MSSQL.

## Prometheus

Prometheus is a monitoring toolkit that can collect and serve metrics (https://prometheus.io/docs/introduction/overview/).

It communicates through the use of a simple text format that any other program can provide to use for monitoring. https://prometheus.io/docs/instrumenting/exposition_formats/

There are a bunch of client libraries that can help developers in offering a source of metrics to build into their applications. https://prometheus.io/docs/instrumenting/clientlibs/

### Prometheus for Delphi

I have implemented a small portion of the client library, but have not been able to get much production use out of it yet. For those who are interested, you can find it here https://github.com/partouf/prometheus-client-delphi

## Installing the Grafana Agent

The agent that collects metrics about the system and that can transmit these to Grafana (Prometheus endpoint) can be downloaded here: https://grafana.com/docs/agent/latest/getting-started/install-agent-on-windows/

### Installing the Windows Exporter

Extra windows metrics are supported by the Grafana Agent, but depending on the version, it may sometimes be broken (for example up until a month ago https://github.com/grafana/agent/issues/1164)

In this case, you can download the Windows Exporter seperately at https://github.com/prometheus-community/windows_exporter/releases

The slight annoyance about the Windows Exporter is that for non-default metrics to be enabled, you will need to install the application differently. You can find full instructions here https://github.com/prometheus-community/windows_exporter#installation

In my case I have installed it with `ENABLED_COLLECTORS=logical_disk,mssql`. If installed correctly, you can find the kind of parameters setup in the windows_exporter service.

## Configuring the agent

Once the Agent and the Windows Exporter have been installed, you will need to configure the agent to send the metrics them to your Grafana endpoint.

You can configure this by editing the file `\program files\grafana agent\agent-config.yaml`

To send the metrics that the Agent collects, you will need to add a remote_write integration to the configuration. 

Add to the end of the yaml file the following:

```
integrations:
  prometheus_remote_write:
  - basic_auth:
      password: apikey
      username: username
    url: https://prometheus-prod-01-eu-west-0.grafana.net/api/prom/push
```

You can find the exact settings if you use to the Prometheus Send Metrics or Details buttons on your Grafana Cloud Stack page.

## Collecting other metrics

To configure other metrics like that of the Windows Exporter, or your own applications, you will need to configure the Agent to "scrape" and remote write the metrics seperately.

You can do this by adding under the `metrics` section of the yaml file the following:

```
  configs:
    - name: agent
      scrape_configs:
        - job_name: windows_exporter
          static_configs:
            - targets: [ 'localhost:9182' ]
              labels:
                agent_hostname: 'myhostname'
      remote_write:
        - basic_auth:
            password: apikey
            username: username
          url: https://prometheus-prod-01-eu-west-0.grafana.net/api/prom/push
```

The port 9182 in this case is the default port that the Windows Exporter uses.

After changing your agent's configuration, make sure you restart the Grafana Agent service.

## MSSQL Metrics in Grafana

Once you have set up your Grafana Agent and Windows Exporter correctly, you will be able to query for those metrics in your Grafana Dashboard.

To show the amount of Active Transactions you can for example use the query:

`windows_mssql_transactions_active{agent_hostname="myhostname"}`

Or for the amount concurrent connections:

`windows_mssql_genstats_user_connections{agent_hostname="myhostname"}`

## Disk metrics

To query the free diskspace, you will need to use something like this:

`windows_logical_disk_free_bytes{agent_hostname="myhostname",volume="D:"} / 1024 / 1024 / 1024`
