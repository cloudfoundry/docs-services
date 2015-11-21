---
title: Route Services (Experimental)
---

<p class="note"><strong>Note</strong>: Service Instance responsibilities below are experimental and subject to change.</p>

## <a id='introduction'></a>Introduction ##

Cloud Foundry application developers may wish to inject various middleware tools onto requests before they reach the application. Common examples of these are authentication, rate limiting, and caching services. Route Services allow Cloud Foundry users to apply various content transformations to their applications by binding the application's route to a service instance.

## <a id='enabling-route-services-in-cloudfoundry'></a>Enabling Route Services in Cloud Foundry ##

To enable support for Route Services in a Cloud Foundry deployment, the operator must provide a passphrase used by Gorouter to encrypt a header that is sent with the request to the route service. This header is used by Gorouter to validate the request sent by the route service to the application route.

This property can be configured in the cf-release manifest:

```
properties:
  router:
    route_services_secret: QgIp1D0LXRZhZOb+dSFoWw==
```

This secret should be randomly generated for your deployment. See the [gorouter spec](https://github.com/cloudfoundry/cf-release/blob/master/jobs/gorouter/spec) in cf-release for more details.

## <a id='broker-responsibilities'></a>Broker Responsibilities ##

#### Catalog Endpoint
Brokers must include `requires: [“route_forwarding”]` for a service in the catalog endpoint. If this is not present, Cloud Foundry will not permit users to bind an instance of the service to a route.

#### Binding Endpoint
When users bind a route to a service instance, Cloud Foundry will send a [bind request](http://docs.cloudfoundry.org/services/api.html#binding) to the broker, including the route address with `bind_resource.route`. A route is an address used by clients to reach apps mapped to the route. The broker may return `route_service_url`, containing a URL where Cloud Foundry should proxy requests for the route. This URL must have a `https` scheme, otherwise the Cloud Controller will reject the binding. `route_service_url` is optional; not returning this field enables a broker to dynamically configure a network component already in the request path for the route, requiring no change in the Cloud Foundry router.

## <a id='service-instance-responsibilities'></a>Service Instance Responsibilities ##

The following applies only when a broker returns `route_service_url` in the bind response.

#### How It Works

Binding a service instance to a route will associate the `route_service_url` with the route in the Cloud Foundry router. All requests for the route will be proxied to the URL specified by `route_service_url`.

Once a route service completes its function, it is expected to forward the request to the route the original request was sent to. The Cloud Foundry router will include a header that provides the address of the route, as well as two headers that are used by the route itself to validate the request sent by the route service.

#### Headers
The `X-CF-Forwarded-Url` header contains the URL of the application route. The route service should forward the request to this URL.

The route service should not strip off the `X-CF-Proxy-Signature` and `X-CF-Proxy-Metadata`, as the GoRouter relies on these headers to validate that the request.

#### Timeouts

Route services must forward the request to the application route within the number of seconds configured by the `router.route_service_timeout` property (default 60 seconds).

In addition, all requests must respond in the number of seconds configured by the `request_timeout_in_seconds` property (default 900 seconds).

Timeouts are configurable for the router using the cf-release BOSH deployment manifest. For more information, see the [spec](https://github.com/cloudfoundry/cf-release/blob/master/jobs/gorouter/spec)

## Example Route Service
- [Logging Route Service](https://github.com/cloudfoundry-samples/logging-route-service): This route service can be pushed as an app to Cloud Foundry. It fulfills the service instance responsibilities above.
