---
title: v2 API Change Log
---

## 2015-05-19 ##
Add support for arbitrary params; users can send service-specific configuration parameters along with provision, update and bind requests.

## 2015-05-15 ##
Made it explicit that specified fields are supported for responses to create and bind requests, and that others will be ignored.

## 2015-04-22 ##
Move Asynchronous Operations to separate document and remove v2.5 experimental documentation. Async operations will be added to a v2.X doc when they are considered non-experimental.

## 2015-03-02 ##
Introduce experimental version of Asynchronous Operations. The API allows for provisioning, updating, and deprovisioning of service instances in an asynchronous manner. The change includes a new endpoint for checking the status of service instance operations. It also allows brokers to return `202 Accepted` in response to provision, update, or deprovision requests with an updated response body to communicate the status of the requested operation.

## 2015-01-14 ##
Updated Orphans section of the API to document the new behavior of reducing likelihood of orphaned instances in cf-release v196.

Bug fix in documentation. Clarified what valid JSON is for API responses expected: valid JSON are JSON Objects ({}) and not arrays.

## 2015-01-05 ##
Bug found in [the v2 doc](api.html). It refers to the service broker API Catalog attribute `plan_updateable` as `plan_updatable` (correct spelling). This is an unfortunate mispelling. Instead of correcting the spelling and breaking backward compatibility, we have opted to keep that misspelling and change the docs to use the misspelled attribute. We have also added a note to explain the issue.

## 2014-09-26 ##
Bug found in [the v2 doc](api.html). It indicated that the broker would try to
unbind or unprovision when a bind or provision request times out. The
documentation also indicated that the Cloud Controller will retry requests.
Both are incorrect.

## 2014-10-31 ##
Added support for service instance plan update, bumped version to 2.4. Included in cf-release final build 192.

## 2014-04-23 ##
Added dashboard_client parameters to /v2/catalog in support of [service dashboard sso](dashboard-sso.html), bumped version to 2.3. Included in cf-release final build 169.

## 2014-03-31 ##
Added free field to service plans, bumped version to 2.2, included in cf-release final build 164.

## 2014-01-27 ##
Removed example request bodies for Unbind and Delete, as DELETE requests do not
include a body. Clarified that documented request fields are query parameters,
not body fields. Updated example cUrl to remove body and add query
parameters.

## 2013-12-27 ##
Added field, bumped version to 2.1, included in cf-release final build 152.

## 2013-12-12 ##
Renamed 'writing-service' document to 'api'. Updated [the api document](api.html) to reflect the v2.1 api.  Moved [the v2.0 doc](api-v2.0.html) to a separate page for archival purposes.

## 2013-12-11 ##
Bug found in [the v2.0 doc](api-v2.0.html). It was indicated that the
credentials field returned by the broker after binding a service to an app is
required, but it is actually optional.

## 2013-12-09 ##
Bug found in [the v2.0 doc](api-v2.0.html). We discovered another place in the
docs that indicated that a 404 returned for a unbind or delete would be
interpreted by cloud controller as a success. This was incorrect. Cloud
controller accepts 200 and 410 as successes and 404 as a failure. We have
updated the documentation again and the API version remains at 2.0.

## 2013-11-26 ##
Bug found in [the v2.0 doc](api-v2.0.html). It was indicated that a 404
returned for a unbind or delete would be interpreted by cloud controller as a
success. This was incorrect. Cloud controller accepts 200 and 410 as successes
and 404 as a failure. Doc has been updated and API version remains at 2.0.
