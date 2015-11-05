---
title: Route Services
---

## <a id='introduction'></a>Introduction ##

Cloud Foundry application developers may wish to inject various middleware tools onto requests before they reach the application. Common examples of these are authentication, rate limiting, and caching services. Route Services allow Cloud Foundry users to apply various content transformations to their applications by binding the application's route to a service instance.

## <a id='enabling-route-services-in-cloudfoundry'></a>Enabling Route Services in Cloud Foundry ##

To enable the Route Service feature, the GoRouter requires a secret to encrypt a signature header that is sent along with the request to the route service.

This property can be configured in the cf-release manifest:

```
properties:
  router:
    route_services_secret: QgIp1D0LXRZhZOb+dSFoWw==
```

This secret should be randomly generated for your deployment. See the [gorouter spec](https://github.com/cloudfoundry/cf-release/blob/master/jobs/gorouter/spec) in cf-release for more details.

## <a id='broker-responsibilities'></a>Broker Responsibilities ##

#### Catalog Endpoint
Brokers must add a `requires: [“route_forwarding”]` field to the catalog endpoint for services to be considered as Route Services.

#### Binding Endpoint
In response to a [binding request](http://docs.cloudfoundry.org/services/api.html#binding) with a `route` key inside of the `bind_resource`, the broker should return the URL where the routing service will receive requests. This URL must have a `https` scheme, otherwise the Cloud Controller will reject the binding.

#### Timeouts
Timeouts are determined by the GoRouter's configuration manifest. For more information, see the [spec](https://github.com/cloudfoundry/cf-release/blob/master/jobs/gorouter/spec)

The routing service must forward the request back to the GoRouter in the number of seconds configured by the `router.route_service_timeout` property (default 60 seconds).

In addition, all requests must respond in the number of seconds configured by the `request_timeout_in_seconds` property (default 900 seconds).

## <a id='service-instance-responsibilities'></a>Service Instance Responsibilities ##

#### Headers
The route service instance should not strip off the `X-CF-Proxy-Signature` and `X-CF-Proxy-Metadata`, the GoRouter relies on these headers.

The `X-CF-Forwarded-Url` header contains the URL of the application being intercepted. The Route Service should forward the request to this URL.

## How It Works (Under Construction)

Binding a service instance to a route will associate that service's `route_service_url` with the route. This will trigger an update to any running apps mapped to that route, allowing the GoRouter to proxy any request to that route through the route service.

## Example Route Service
- Logging Route Service : https://github.com/cloudfoundry-samples/logging-route-service
