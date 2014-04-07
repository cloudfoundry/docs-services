---
title: Integrating Dashboard Single Sign-On
---

## Introduction

Single sign-on (SSO) allows Cloud Foundry users to authenticate with third-party services
using their Cloud Foundry credentials. SSO provides a streamlined experience to users,
limiting repeated logins and multiple accounts across their managed services.

In order to make this possible, CF provides API endpoints for configuration and
verification of the user credentials. This allows third-party services to verify user credentials
without needing the user to login again. The user's credentials are never directly transmitted to
the service since the OAuth2 protocol handles authentication. Similarly, Cloud Controller provides
an endpoint to determine a user's authorization.

## The SSO interaction and points of integration
  1. In order to integrate a dashboard with CF, service brokers need to include the necessary
    properties in their catalog. Specifically, each service implementing this feature must
    advertise a `dashboard_client` property in their JSON response from `/v2/catalog`. A valid
    response would appear as follows:

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

    `id` is the unique identifier for the OAuth client that will be created for your service UI on the token server (UAA), and will be used by your service UI to authenticate with the token server (UAA).

    `secret` is the shared secrect your service UI will use to authenticate with the token server (UAA).

    `redirect_uri` is used by the token server as an additional security precaution. UAA will not provide a token if the callback URL declared by the service dashboard doesn't match the domain name in `redirect_uri`. The token server matches on the domain name, so any paths will also match; e.g. a service dashboard requesting a token and declaring a callback URL of `http://p-mysql.example.com/manage/auth` would be approved if `redirect_uri` for its client is `http://p-mysql.example.com/`. 

  2. Whenever the catalog for this service broker is created or updated, Cloud Controller will
    create or update UAA clients for any services that advertise SSO capability. This client
    will be used by the service to authenticate users. Operators verify the existance of
    service UAA clients using the [uaac](https://github.com/cloudfoundry/cf-uaac) tool.

    ```
    $ uaac clients
      admin
        scope: uaa.none
        resource_ids: none
        authorized_grant_types: client_credentials
        authorities: password.write clients.write clients.read scim.read uaa.admin clients.secret
      p-mysql-client
        scope: cloud_controller.read cloud_controller.write openid
        resource_ids: none
        authorized_grant_types: authorization_code refresh_token
        redirect_uri: http://p-mysql.example.com
        authorities: uaa.none
    ```

  3. At this point, users will be able to click the `Manage` link on their CF Web console,
    seamlessly allowing them to access the dashboard for the service.
    <br />
+   ![Managing a Service](../images/web-ui-manage-service.png)

  4. At this point, the service will need to obtain an access token from UAA.  The service should
    use the standardized OAuth2 protocol to retrieve this token.  See [Resources](#resources)
    for more information regarding OAuth2 and for an example broker/service dashboard implementation.

    Users will be redirected from the service dashboard to the login server where they will be prompted to authorize the permissions requested by the service dashboard.  Upon authorizing the requested permissions, the user will be redirected back to the service (assuming the redirect URL matches `redirect_uri` as described above).  The service will then need to check with Cloud Controller whether the authenticated user currently has permission to manage the service instance.

  5. UAA is responsible for authenticating a user and providing the service with an access token
    with the requested permissions.  However, it is the responsibility of the service to verify
    that the user making the request to manage an instance currently has access to that service instance.  The service
    can accomplish this with a GET to the `/v2/service_instances/:guid/permissions` endpoint on the
    Cloud Controller. The request must include a token for an authenticated user and the service instance guid. 

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

    This request should use the token of the user for authorization.

    The response will indicate to the service whether this user is allowed to manage the given instance.
    A `true` value for the `manage` key indicates sufficient permissions; `false` would indicate insufficient
    permissions.  Since administrators may change the permissions of users, the service should check this endpoint whenever a user uses the SSO flow to access the service's UI.

## Restrictions

 * UAA clients are scoped to services.  There must be a `dashboard_client` entry for each service that uses SSO integration.
 * Each `dashboard_client id` must be unique across the CloudFoundry deployment.

<a id="resources"></a>
## Resources
  * [OAuth2](http://oauth.net/2/)
  * [Example broker SSO implementation](https://github.com/cloudfoundry/cf-mysql-broker)
  * [Cloud Controller API Docs](http://apidocs.cfapps.io/)
