---
title: Access Control
---

By default, all new service plans are private. This means that when adding a new broker, or when adding a new plan to an existing broker's catalog, new service plans won't immediately be available to end users. This enables an admin to control which service plans are available to end users, and to manage limited availability.

## <a id='prerequisites'></a>Prerequisites
- CLI v6.4.0
- Cloud Controller API v2.9.0 (cf-release v179)
- Admin user access; the following commands can be run only by an admin user

To determine your API version, curl `/v2/info` and look for `api_version`.

<pre class="terminal">
$ cf curl /v2/info
{
   "name": "vcap",
   "build": "2222",
   "support": "http://support.cloudfoundry.com",
   "version": 2,
   "description": "Cloud Foundry sponsored by Pivotal",
   "authorization_endpoint": "https://login.system-domain.com",
   "token_endpoint": "https://uaa.system-doman.com",
   "api_version": "2.13.0",
   "logging_endpoint": "wss://loggregator.system-domain.com:443"
}
</pre>

## <a id='display-access'></a>Display Access to Service Plans

The `service-access` CLI command enables an admin to see the current access control setting for every service plan in the marketplace, across all service brokers.

<pre class="terminal">
$ cf service-access
getting service access as admin...
broker: p-riakcs
   service    plan        access    orgs
   p-riakcs   developer   limited

broker: p-mysql
   service   plan        access   orgs
   p-mysql   100mb-dev   all
</pre>

The `access` column has values `all`, `limited`, or `none`. `all` means a service plan is available to all users of the Cloud Foundry instance; this is what we mean when we say the plan is "public". `none` means the plan is not available to  anyone; this is what we mean when we say the plan is "private". `limited` means that the service plan is available to users of one or more select organizations. When a plan is `limited`, organizations that have been granted access are listed.

Flags provide filtering by broker, service, and organization.

<pre class="terminal">
$ cf help service-access
NAME:
   service-access - List service access settings

USAGE:
   cf service-access [-b BROKER] [-e SERVICE] [-o ORG]

OPTIONS:
   -b 	access for plans of a particular broker
   -e 	access for plans of a particular service offering
   -o 	plans accessible by a particular organization
</pre>

## <a id='enable-access'></a>Enable Access to Service Plans

Service access is managed at the granularity of service plans, though CLI commands allow an admin to modify all plans of a service at once.

Enabling access to a service plan for organizations allows users of those organizations to see the plan listed in the marketplace (`cf marketplace`), and if users have the Space Developer role in a targeted space, to provision instances of the plan.

<pre class="terminal">
$ cf enable-service-access p-riakcs
Enabling access to all plans of service p-riakcs for all orgs as admin...
OK

$ cf service-access
getting service access as admin...
broker: p-riakcs
   service    plan        access   orgs
   p-riakcs   developer   all
</pre>

An admin can use `enable-service-access` to:

- Enable access to all plans of a service for users of all orgs (access:`all`)
- Enable access to one plan of a service for users of all orgs (access:`all`)
- Enable access to all plans of a service for users of a specified organization (access: `limited`)
- Enable access to one plan of a service for users of a specified organization (access: `limited`)

<pre class="terminal">
$ cf help enable-service-access
NAME:
   enable-service-access - Enable access to a service or service plan for one or all orgs

USAGE:
   cf enable-service-access SERVICE [-p PLAN] [-o ORG]

OPTIONS:
   -p 	Enable access to a particular service plan
   -o 	Enable access to a particular organization
</pre>

## <a id='disable-access'></a>Disable Access to Service Plans

<pre class="terminal">
$ cf disable-service-access p-riakcs
Disabling access to all plans of service p-riakcs for all orgs as admin...
OK

$ cf service-access
getting service access as admin...
broker: p-riakcs
   service    plan        access   orgs
   p-riakcs   developer   none
</pre>

An admin can use the `disable-service-access` command to:

- Disable access to all plans of a service for users of all orgs (access:`all`)
- Disable access to one plan of a service for users of all orgs (access:`all`)
- Disable access to all plans of a service for users of select orgs (access: `limited`)
- Disable access to one plan of a service for users of select orgs (access: `limited`)

<pre class="terminal">
$ cf help disable-service-access
NAME:
   disable-service-access - Disable access to a service or service plan for one or all orgs

USAGE:
   cf disable-service-access SERVICE [-p PLAN] [-o ORG]

OPTIONS:
   -p 	Disable access to a particular service plan
   -o 	Disable access to a particular organization
</pre>

### Limitations

- You cannot disable access to a service plan for an organization if the plan is currently available to all organizations. You must first disable access for all organizations; then you can enable access for a  particular organization.
