# Azure-metrics-exporter

Azure metrics exporter for [Prometheus.](https://prometheus.io)

Allows for the exporting of metrics from Azure applications using the [Azure monitor API.](https://docs.microsoft.com/en-us/azure/monitoring-and-diagnostics/monitoring-rest-api-walkthrough)

## Install

```bash
go get -u github.com/RobustPerception/azure_metrics_exporter
```

## Usage
```bash
./azure_metrics_exporter --help
```

## Rate limits

Note that Azure imposes an [API read limit of 15,000 requests per hour](https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-manager-request-limits) so the number of metrics you're querying for should be proportional to your scrape interval.

## Retrieving Metric definitions

In order to get all the metric definitions for the resources specified in your configuration file, run the following:

```bash
./azure-metrics-exporter --list.definitions
```

This will print your resource id's application/service name along with a list of each of the available metric definitions that you can query for for that resource.

## Exporter configuration

This exporter requires a configuration file. By default, it will look for the azure.yml file in the CWD.

### Azure account requirements

This exporter reads metrics from an existing Azure subscription with these requirements:

  * An application must be registered (e.g., Azure Active Directory -> App registrations -> New application registration)
  * The registered application must have reading permission to Azure Monitor (e.g., Subscriptions -> your_subscription -> Access control (IAM) -> Role assignments -> Add -> Add role assignment -> Role : "Monitoring Reader", Select:  my_app)

### Example azure-metrics-exporter config

`azure_resource_id` and `subscription_id` can be found under properties in the Azure portal for your application/service.

`azure_resource_id`  should start with `/resourceGroups...` (`/subscriptions/xxxxxxxx-xxxx-xxxx-xxx-xxxxxxxxx` must be removed from the begining of `azure_resource_id` property value)

`tenant_id` is found under `Azure Active Directory > Properties` and is listed as `Directory ID`.

The `client_id` and `client_secret` are obtained by registering an application under 'Azure Active Directory'.

`client_id` is the `application_id` of your application and the `client_secret` is generated by selecting your application/service under Azure Active Directory, selecting 'keys', and generating a new key.

If you want to scrape metrics from Azure national clouds (e.g. AzureChinaCloud, AzureGermanCloud), you should provide `active_directory_authority_url` and `resource_manager_url` parameters. `active_directory_authority_url` is AzureAD url for getting access token. `resource_manager_url` is Azure API management url.
If you won't provide `active_directory_authority_url` and `resource_manager_url` parameters, azure-metrics-exporter scrapes metrics from global cloud.
You can find endpoints for national clouds [here](http://www.azurespeed.com/Information/AzureEnvironments)

```
active_directory_authority_url: "https://login.microsoftonline.com/"
resource_manager_url: "https://management.azure.com/"
credentials:
  subscription_id: <secret>
  client_id: <secret>
  client_secret: <secret>
  tenant_id: <secret>

targets:
  - resource: "azure_resource_id"
    metrics:
    - name: "BytesReceived"
    - name: "BytesSent"
  - resource: "azure_resource_id"
    aggregations:
    - Minimum
    - Maximum
    - Average
    metrics:
    - name: "Http2xx"
    - name: "Http5xx"

resource_groups:
  - resource_group: "webapps"
    resource_types:
    - "Microsoft.Compute/virtualMachines"
    resource_name_include_re:
    - "testvm.*"
    resource_name_exclude_re:
    - "testvm12"
    metrics:
    - name: "CPU Credits Consumed"

```

By default, all aggregations are returned (`Total`, `Maximum`, `Average`, `Minimum`). It can be overridden per resource.


### Resource group filtering

Resources in a resource group can be filtered using the the following keys:

`resource_types`:
List of resource types to include (corresponds to the `Resource type` column in the Azure portal).

`resource_name_include_re`:
List of regexps that is matched against the resource name.
Metrics of all matched resources are exported (defaults to include all)

`resource_name_exclude_re`:
List of regexps that is matched against the resource name.
Metrics of all matched resources are ignored (defaults to exclude none)
Excludes take precedence over the include filter.

## Prometheus configuration

### Example config
```
global:
  scrape_interval:     60s # Set a high scrape_interval either globally or per-job to avoid hitting Azure Monitor API limits.

scrape_configs:
  - job_name: azure
    static_configs:
      - targets: ['localhost:9276']
```