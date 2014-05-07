---
title: Integrating Dashboard Single Sign-On
---

## Introduction

Single sign-on (SSO) allows Cloud Foundry users to authenticate with third-party services using their Cloud Foundry credentials. SSO provides a streamlined experience to users, limiting repeated logins and multiple accounts across their managed services.

SSO was introduced in [cf-release v169](https://github.com/cloudfoundry/cf-release/tree/v169),
so v169 is the minimum release version needed to support the SSO feature.

In order to make this possible, CF provides API endpoints for configuration and
verification of the user credentials. This allows third-party services to verify user credentials without needing the user to login again. The user's credentials are never directly transmitted to the service since the OAuth2 protocol handles authentication. Similarly, Cloud Controller provides an endpoint to determine a user's authorization.

## The SSO interaction and points of integration

### Registering the Dashboard

1. To enable the SSO feature, the Cloud Controller requires a UAA client with sufficient permissions to create and delete clients for the service brokers that request them. This client can be configured by including the following snippet in the runtime manifest:

  ```
  properties:
    uaa:
      clients:
        cc_service_broker_client:
          secret: cc-broker-secret
          scope: cloud_controller.write,openid,cloud_controller.read,cloud_controller_service_permissions.read
          authorities: clients.read,clients.write,clients.admin
          authorized-grant-types: client_credentials
  ```

1. In order to integrate a dashboard with CF, service brokers need to include the necessary properties in their catalog. Specifically, each service implementing this feature must advertise a `dashboard_client` property in their JSON response from `/v2/catalog`. A valid response would appear as follows:

  ```
  {
    "services": [
      {
        "id": "44b26033-1f54-4087-b7bc-da9652c2a539",
        ...
        "dashboard_client": {
          "id": "p-mysql-client",
          "secret": "p-mysql-secret",
          "redirect_uri": "http://p-mysql.example.com"
        }
      }
    ]
  }
  ```

  `id` is the unique identifier for the OAuth2 client that will be created for your service UI on the token server (UAA), and will be used by your service UI to authenticate with the token server (UAA).

  `secret` is the shared secret your service UI will use to authenticate with the token server (UAA).

  `redirect_uri` is used by the token server as an additional security precaution. UAA will not provide a token if the callback URL declared by the service dashboard doesn't match the domain name in `redirect_uri`. The token server matches on the domain name, so any paths will also match; e.g. a service dashboard requesting a token and declaring a callback URL of `http://p-mysql.example.com/manage/auth` would be approved if `redirect_uri` for its client is `http://p-mysql.example.com/`.

1. Whenever the catalog for this service broker is created or updated, Cloud Controller will create or update UAA clients for any services that advertise SSO capability. This client will be used by the service (dashboard UI) to authenticate users.

### OAuth2 Login Flow for Dashboards

A `dashboard_url` is provided by the broker when a service instance is [provisioned](./api.html#provisioning). At this point, users can navigate to the `dashboard_url`, seamlessly allowing them to access the dashboard for the service. The Cloud Controller provides `dashboard_url` as part of its JSON response when a user makes an API request for a service instance.

The service dashboard UI at this URL should initiate the OAuth2 login flow. The overall flow is described in [section 1.2](http://tools.ietf.org/html/rfc6749#section-1.2) of the OAuth2 RFC. OAuth2 expects the presence of two endpoints on the authorization server.  The URLs for the [Authorization Endpoint](http://tools.ietf.org/html/rfc6749#section-3.1) and [Token Endpoint](http://tools.ietf.org/html/rfc6749#section-3.2) can be retrieved from Cloud Controller's info endpoint (GET `/info`).

#### Dashboard UI Implementation

A Dashboard UI should implement the [OAuth2 RFC 4.1 Authorization Code Grant](http://tools.ietf.org/html/rfc6749#section-4.1) type:

1. The dashboard UI should redirect the user's browser to the Authorization Endpoint providing the `client_id`, the `redirect_uri` with a domain matching the `redirect_uri` domain above, and the requested scope. Ideally, the scope should be as limited as possible, with the most limited scope for this workflow being `cloud_controller_service_permissions.read` and `openid`. For an explanation of the various scopes and the permissions associate with them, [see below](#on-scopes).

1. UAA authenticates the user, who then approves or denies the permission requested by the dashboard UI.

1. Assuming the user grants access, UAA redirects the user's browser back to the dashboard UI using the `redirect_uri` provided earlier.  The `redirect_uri` includes an authorization code.

1. The dashboard UI should request an access token from the Token Endpoint by including the authorization code received in the previous step.  When making the request, the dashboard UI must authenticate with UAA. The dashboard UI should include the `redirect_uri` used to obtain the authorization code for verification.

1. UAA authenticates the dashboard UI, validates the authorization code, and ensures that the redirection URI received matches the URI used to redirect the client in step (3).  If valid, UAA responds back with an access token and a refresh token.

1. UAA is responsible for authenticating a user and providing the service with an access token with the requested permissions. However, after the user has been logged in, it is the responsibility of the service to verify that the user making the request to manage an instance currently has access to that service instance.

  The service can accomplish this with a GET to the `/v2/service_instances/:guid/permissions` endpoint on the Cloud Controller. The request must include a token for an authenticated user and the service instance guid.

  Example:

  ```
  curl -H 'Content-Type: application/json' \
       -H 'Authorization: bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoidWFhLWlkLTciLCJlbWFpbCI6ImVtYWlsLTdAc29tZWRvbWFpbi5jb20iLCJzY29wZSI6WyJjbG91ZF9jb250cm9sbGVyLmFkbWluIl0sImF1ZCI6WyJjbG91ZF9jb250cm9sbGVyIl0sImV4cCI6MTM5Mjc0NzIzNH0.IUsMEB95qiBazm-iyVlekBomBEuMYHTufeB3SLiGpWM' \
       http://api.cloudfoundry.com/v2/service_instances/44b26033-1f54-4087-b7bc-da9652c2a539/permissions

  =>
    {
      "manage": true
    }
  ```

  This request should use the token obtained in step (5) for authorization.

  The response will indicate to the service whether this user is allowed to manage the given instance. A `true` value for the `manage` key indicates sufficient permissions; `false` would indicate insufficient permissions.  Since administrators may change the permissions of users, the service should check this endpoint whenever a user uses the SSO flow to access the service's UI.

#### <a name="#on-scopes"></a> On scopes

Scopes let you specify exactly what type of access you need. Scopes limit access for OAuth tokens. They do not grant any additional permission beyond that which the user already has.

##### Minimum Scopes
The following two scopes are necessary to implement the integration. Most dashboard shouldn't need more permissions than these scopes enabled.

| Scope                                            | Permissions   |
| ------------------------------------------------ | ------------- |
| `openid`                                         | Allows access to basic data about the user, such as email addresses |
| `cloud_controller_service_permissions.read`      | Allows access to the CC endpoint that specifies whether the user can manage a given service instance |

##### Additional scopes
Dashboards with extended capabilities may need to request these additional scopes:

| Scope                                            | Permissions   |
| ------------------------------------------------ | ------------- |
| `cloud_controller.read`                          | Allows read access to all resources the user is authorized to read |
| `cloud_controller.write`                         | Allows write access to all resources the user is authorized to update / create / delete |

## Reference Implementation

The [MySQL Service Broker][example-broker] is an example of a broker that also implements a SSO dashboard. The login flow is implemented using the [OmniAuth library](https://github.com/intridea/omniauth) and a custom [UAA OmniAuth Strategy](https://github.com/cloudfoundry/omniauth-uaa-oauth2). See this [OmniAuth wiki page](https://github.com/intridea/omniauth/wiki/Strategy-Contribution-Guide) for instructions on how to create your own strategy.

The UAA OmniAuth strategy is used to first get an authorization code, as documented in [this section](https://github.com/cloudfoundry/uaa/blob/master/docs/UAA-APIs.rst#authorization-code-grant) of the UAA documentation. The user is redirected back to the service (as specified by the `callback_path` option or the default `auth/cloudfoundry/callback` path) with the authorization code. Before the application / action is dispatched, the OmniAuth strategy uses the authorization code to [get a token](https://github.com/cloudfoundry/uaa/blob/master/docs/UAA-APIs.rst#client-obtains-token-post-oauth-token) and uses the token to request information from UAA to fill the `omniauth.auth` environment variable. When OmniAuth returns control to the application, the `omniauth.auth` environment variable hash will be filled with the token and user information obtained from UAA as seen in the [Auth Controller](https://github.com/cloudfoundry/cf-mysql-broker/blob/master/app/controllers/manage/auth_controller.rb).

## Restrictions

 * UAA clients are scoped to services.  There must be a `dashboard_client` entry for each service that uses SSO integration.
 * Each `dashboard_client id` must be unique across the CloudFoundry deployment.

<a id="resources"></a>
## Resources
  * [OAuth2](http://oauth.net/2/)
  * [Example broker with SSO implementation][example-broker]
  * [Cloud Controller API Docs](http://apidocs.cfapps.io/)
  * [User Account and Authentication (UAA) Service APIs](https://github.com/cloudfoundry/uaa/blob/master/docs/UAA-APIs.rst)

[example-broker]: https://github.com/cloudfoundry/cf-mysql-broker
