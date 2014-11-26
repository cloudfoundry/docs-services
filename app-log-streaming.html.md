---
title: Application Log Streaming
---

By binding an application to an instance of an applicable service, Cloud Foundry will stream logs for the bound application to the service instance.

- Logs for all apps bound to a log-consuming service instance will be streamed to that instance
- Logs for an app bound to multiple log-consuming service instances will be streamed to all instances

To enable this functionality, a service broker must implement the following:

1. In the [catalog](api.html#catalog-mgmt) endpoint, the broker must include `requires: syslog_drain`. This minor security measure validates that a service returning a `syslog_drain_url` in response to the [bind](api.html#binding) operation has also declared that it expects log streaming. If the broker does not include `requires: syslog_drain`, and the bind request returns a value for `syslog_drain_url`, Cloud Foundry will return an error for the bind operation.

2. In response to a [bind](api.html#binding) request, the broker should return a value for `syslog_drain_url`. The syslog URL has a scheme of syslog, syslog-tls, or https and can include a port number. For example:

    `"syslog_drain_url": "syslog://logs.example.com:1234"`

## How does it work?

- Broker returns a value for `syslog_drain_url` in response to bind
- When application is restarted, [VCAP_SERVICES](../devguide/deploy-apps/environment-variable.html#VCAP-SERVICES) is updated with the key and value for `syslog_drain_url`
- DEAs continuously stream application logs to the Loggregator
- If `syslog_drain_url` is present in [VCAP_SERVICES](../devguide/deploy-apps/environment-variable.html#VCAP-SERVICES), the DEA tags the logs with this field
- Loggregator streams logs tagged with this key to the location specified as its value

Users can manually configure app logs to be streamed to a location of their choice using User-provided Service Instances. For details, see [Using Third-Party Log Management Services](../devguide/services/log-management.html).
