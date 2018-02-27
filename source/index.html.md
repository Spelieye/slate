---
title: GENI API Reference

language_tabs: # must be one of https://git.io/vQNgJ
  - shell
  - ruby
  - python
  - javascript

toc_footers:
  - <a href='https://github.com/lord/slate'>Documentation Powered by Slate</a>

includes:
  - errors
  - metrics

search: true
---

# Introduction

Welcome to Geli's GENI web-based API! 

This document describes Geli’s web-based APIs in GENI. It is primarily written for Geli developers, QA, technical managers, and portions for future API consumers (partners, utilities, etc).

<aside class="notice">
GENI Zone also communicates with EOS nodes by sending compressed messages via STOMP over Websockets. That protocol and supported messages is not covered in this document.
</aside>

## Purpose of GENI 

“Let me explain … No, there is too much. Let me sum up.” —Inigo Montoya

GENI has these goals: 

* Register sites and nodes, and track the status of physical node hardware out in the field
* Configure each node with a device/port model, and some module (driver/app) configurations
* Collect lots of telemetry from nodes and store it
* Process telemetry in real time to detect change events
* Let users subscribe to notifications from these change events (alerts)
* Allow registering user accounts, giving them roles and permissions, and assigning to groups
* Let groups of users see status and historical telemetry for sites, or restrict to specific resources
* Interact with running applications and drivers on EOS nodes (review data, change mode)
* Aggregate and control energy assets together

## General

### Stateless 
All of the information needed to process a request is part of the request. The server generally does not remember any state for the client. When possible, requests and actions are one-step

### Synchronous 
The client waits with an open TCP connection for their response, which is expected to arrive within several seconds. For example, updating a site record with an address correction can take a little while (dozens of ms) to complete because of the geocoding and database activity. The response will contain not only a successful status code, but also a modified, geocoded site with new latitude and longitude, so the client knows the record has been updated.

### No Guarantee Without Explicit Confirmation 
Although the APIs are synchronous, this does not mean that a request has completely succeeded if the API request does not completely fail. 

Some requests return “success” after an action has been queued in another system, or even when a request was silently refused. In cases where the API does not provide an explicit confirming response — such as a modified object to “show” that the object was modified — the client should not assume that an action has occurred or completed.

### Shared Authentication 
After logging in, the user’s token is portable across servers / services that may be on different machines or even in different locations. As described in the Authorization API section, there is a session, but it is stored by the client.

## JSON Object Transfer 
All requests and responses are transferred as JSON objects. UTF-8 encoding is preferred. All objects used as API inputs or outputs are serialized according to the following rules:

1. A field “@dto” must be present, specifying the type of the object. When handling any response, the client must check that this is the expected type (see [Error Handling](#error-handling)). 
2. Null fields should not be serialized.
3. Unrecognized fields and any fields that do not apply to the request are silently ignored. 
4. The required fields are minimal. On an “update” (PUT) request, for example, the caller should only specify the fields that are being updated plus the required “@dto” field.

## Specified DTO Types 
Data Transfer Objects (DTOs) with well-known types are used for all API inputs and outputs. All DTO types are defined in the package net.geli.dto. You can review that package to see all possible fields for a certain DTO or you may be able to rely on the examples of typical usage in this doc.

All API inputs and results are wrapped in a well-known typed DTO. For example, if an API method returns a list of objects, it will be a ListDto object of other DTO objects, not a plain JSON list. 

DTOs may contain basic types including lists, maps, numbers, booleans, etc., and they may contain other typed DTOs. Object classes can also be used within a DTO that are not a DTO (do not contain a @dto field), but those objects cannot be an API input or output by themselves.

Some DTO types are reused for more than one type of request / response, with optional fields that are only examined under certain conditions. 


> Example Resource DTOs  

```shell
curl -X GET 
  "https://geni1.geli.net/api/v1/sites/example"
  -H 'Authorization: Token token="GENITOKEN"
```

> The above command returns JSON structure like this: 

```json 
{
    "@dto": "Site",
    "siteCode": "my-site-47155",
    "name": "The Site Name #47155",
    "streetAddress": "123 Fake St.",
    "unitDesignation": "Suite #9",
    "city": "San Francisco",
    "stateOrProvinceCode": "CA",
    "zipCode": "94105",
    "countryCodeA2": "US",
    "manualGeolocation": false,
    "timeZone": "America/Los_Angeles",
    "manualTimeZone": false,
    "siteHost": "site-owner-example@geli.net"
}
```

Collection resource endpoints (e.g. /sites ) provide an example DTO demonstrating what fields should be present when using POST to create a new resource or other request. These are available with the virtual resource code “example”. One of them is
`GET  /sites/example`

## URL Parameters 

The resource path in the URL specifies the item or collection to work with, the verb specifies the action, and the body if present specifies the content. So not all endpoints have URL parameters.

Many endpoints have a “detail” parameter which can be “minimal”, “all” or other values specified in the endpoint documentation. Another use of URL parameters is to filter the output.

## Dates and Times 
Timestamp data gathered by GENI and EOS are always in UTC from the beginning. When an object has a timestamp field, no timezone is typically specified, because it is assumed to be UTC. 

Timestamps are usually stored with millisecond precision. When a timestamp is transmitted as a long integer (typical), it is always encoded as the number of milliseconds since the epoch, 1970-01-01T00:00:00Z.

However, when interacting with external systems - such as handling tariff data or user-provided usage data - local time may be used. If possible, timestamps will be converted to UTC for storage and future use. For cases where this is not practical, fields and methods will be obviously named for local time, and time zone specifiers will be prominently included with the stored data.

# Authentication API

Authentication and basic self-service account functions are under `/auth` . This includes self-registration with email verification, changing one’s own password, and account recovery via email.

This particular API only has methods related to identity. It does not use permissions, although some endpoints may require role = SUPERUSER. It interacts with sessions that are in RECOVERING and VERIFYING and EXPIRED status, while other APIs only work with ACTIVE sessions.	 .

Several auth endpoints are available to unauthenticated users. Because of this, error messages may be intentionally not useful, or methods may appear to work (e.g. return 200 / 204) even if the request was actually refused.

## Email Identity 
The user’s email address is their username. Users of the system must have a valid email address and must either verify the address, or have their account created by an administrator. If you don't have a GENI account you can get one here: [Geni account registration](https://geni1.geli.net/#/user/account/register)

## Session = Token 
Upon authenticating via password, the client is given a timestamped, encrypted and cryptographically signed *token* that identifies them to the system for a period of time. The server can examine this token later and determine whether or not it was signed by GENI.

The token contains the user ID, the issue time, and other flags which control the token’s expiration and possibly provide other session information to GENI.

GENI does not keep track of each token after generating them; it trusts the timestamp and signature in the token itself. Altering or forging a token without knowing the secrets used by the server to encrypt and sign it should be impossible. If you figure out how to do it, please let us know.

To determine whether or not to accept a token, the server examines its timestamp. If the token is too old, the session is EXPIRED and the client will need to re-authenticate. Expired tokens are treated exactly like not having a token at all, except at **GET /auth/login**.

## XSS / CSRF Protection 

To prevent Cross Site Scripting (XSS) and Cross Site Request Forgery (CSRF) attacks, there are actually two tokens provided to the client.

The sensitive session token itself is returned in an “HttpOnly” cookie. This mitigates some types of XSS vulnerabilities, because it prevents JavaScript in the browser from being able to see the value of the session token.

A second token, called the “CSRF Token” (actually an anti-CSRF token), is simultaneously provided in a header, which must be handled by the client as described in the following section. The CSRF Token is a secondary SHA1 HMAC signature on the session token, using a different secret. For a primary session token to be accepted, the corresponding CSRF Token must also be provided in the request headers. 

Combined with HSTS, this strategy helps prevent CSRF attacks, since any script that is not running from a web app loaded from Geli servers should not be able to access the CSRF Token, and no script may directly access the HttpOnly cookie token.

## CSRF Token Issue and Reissue 

The **POST /auth/login** endpoint, as well as any other GENI API endpoint, will offer a CSRF token via the header “Set-Authorization” at the same time that a new token cookie is issued. The client must remember the CSRF token, and include it in a header "Authorization" on each subsequent request.

To keep an active session alive, as the client makes calls to GENI API endpoints, its cookie and CSRF token will be renewed. This does not happen with every request, only when the token is a few minutes old. The client should discard its existing CSRF Token and use the new one.


## Registration Procedure 

1. **POST /auth/register** (“Request GENI Account”) with a username / email address. If accepted, an email is sent with an account verification link, including a token in the URL. 
2. When user clicks link, **GET /auth/verify** (“Verify Email”) to confirm token and get username.  
3. Visit **GET /auth/myaccount/questions** to retrieve a list of available security questions. (currently not required) 
4. After user fills out form, **POST /auth/verify** (“Activate Account”) with the user’s name, security questions and other account details. 
5. In case of failure (password too short, etc), display message to user and try again. 
6. On success, account is active. User is not logged in, but may now log in.

## Recovery Procedures 

1. **POST /auth/recover** (“Request Account Recovery”) with email address. If accepted1, an email is sent with an account recovery link, including a token in the URL. 
2. When user clicks link, **GET /auth/recover/step1** (“Start Account Recovery”) to confirm token and get security questions. 
3. Next, **POST /auth/recover/step2** (“Complete Account Recovery”) with answers to the security questions and new password. 
4. If successful, password is changed and user can attempt to log in again.

## Security Considerations 

The GENI application servers transmit sensitive data, such as tokens and passwords, in plaintext over HTTP. Yet secure usage of the APIs requires that it be deployed in a secure environment, and accessed by secure clients.

This means that it must be deployed on a secure network behind a proxy, since HTTPS support is not part of the GENI app server. Such features are best left to a high-performance native web server such as *nginx*. 

## Get Login 

```shell 
curl
  https://geni1.geli.net/auth/login
```
 
> The above command returns JSON structured like this:

```json
{
    "@dto": "Account",
    "firstName": "Default",
    "groups": [],
    "lastName": "Admin",
    "permissions": [
        "SITE:view,detail,use,manage",
        "*"
    ],
    "roles": [
        "ADMIN",
        "USER",
        "SUPERUSER"
    ],
    "username": "admin@geli.net"
}
```

Examine tokens in request headers. If valid and current, and CSRF token matches cookie, return detailed information about the session user. If it is valid but expired, or CSRF token does not match, return only the username.  

No token is the same as token equals empty string, or token equals “deleted”.

### HTTP request 

`GET https://geni1.geli.net/auth/login`

### URL Parameters

No URL Parameters 

## Post Login 

> To login and get GENI Token, use this:

```shell 
curl -X POST \
  https://geni1.geli.net/auth/login \
  -H 'Content-Type: application/json' \
  -d '{
  "@dto":"Account",
  "password" : "password",
  "username" : "user@geli.net"
}'
```

> The above command returns JSON structured like this:

```json 
{
    "@dto": "Account",
    "firstName": "Default",
    "groups": ["d72f8bbd-5c72-4b26-99b5-524f68c99995"],
    "lastName": "Admin",
    "permissions": [
        "SITE:view,detail,use,manage",
        "*"
    ],
    "roles": [
        "ADMIN",
        "USER",
        "SUPERUSER"
    ],
    "username": "admin@geli.net"
}
```

Attempt to authenticate with a username and password. If successful, the response headers will contain a cookie and CSRF token as described in the Issue and Reissue section. 

### HTTP request 

`POST https://geni1.geli.net/auth/login`

### URL Parameters

No URL Parameters

### Request Body 
`{
  "@dto":"Account", 
  "password" : "password",
  "username" : "user@geli.net"
}`

<aside name="notice">
Replace "password" and "username" with your GENI username and password. 
</aside> 

## Get All Roles 

```shell 
curl -X GET \
  https://geni1.geli.net/auth/roles \
  -H 'Authorization: Token token=\"GENITOKEN"' \
```

> The above command returns JSON structured like this:

```json 
{
    "@dto": "List",
    "items": [
        {
            "@dto": "Role",
            "name": "DEVICE"
        },
        {
            "@dto": "Role",
            "name": "USER"
        },
        {
            "@dto": "Role",
            "name": "ADMIN"
        },
        {
            "@dto": "Role",
            "name": "SUPERUSER"
        }
    ]
}
```

Returns a list of all global roles e.g. SUPERUSER, USER, DEVICE, ADMIN.

### HTTP Request 

`GET https://geni1.geli.net/auth/roles`

### URL Parameters

No URL Parameters

### Authorization 

role: SUPERUSER 

## Get a Specific Role

```shell 
curl -X GET \
  https://geni1.geli.net/auth/roles/user \
  -H 'Authorization: Token token=\"GENITOKEN"' \
```

> The above command returns JSON structured like this:

```json 
{
    "@dto": "Role",
    "name": "USER"
}
```

Returns a single role, not including the members list.

### HTTP Request 

`GET https://geni1.geli.net/auth/roles/{role}`

### Query Parameters

Parameter | Description
--------- | -----------
role | The name of the role to retrieve

### URL Parameters

No URL Parameters

### Authorization 

role: SUPERUSER 

## Get Members in a Role

```shell 
curl -X GET \
  https://geni1.geli.net/auth/roles/user/members \
  -H 'Authorization: Token token=\"GENITOKEN"' \
```

> The above command returns JSON structured like this:

```json 
{
    "@dto": "List",
    "items": [
        {
            "@dto": "UserProfile",
            "email": "admin@geli.net"
        },
        {
            "@dto": "UserProfile",
            "email": "test@geli.net"
        },
        {
            "@dto": "UserProfile",
            "email": "ann@geli.net"
        }
    ]
}
```

Returns a list of all members of the specified role.

### HTTP Request 

`GET https://geni1.geli.net/auth/roles/{role}/members`

### Query Parameters

Parameter | Description
--------- | -----------
role | The name of the role to retrieve

### URL Parameters

No URL Parameters

### Authorization 

role: SUPERUSER 

## Add a Member to a Role

```shell 
curl -X POST \
  https://geni1.geli.net/auth/roles/user/members \
  -H 'Authorization: Token token=\"GENITOKEN"' \
  -d '{
    "@dto": "UserProfile",
    "email": "test@geli.net"
}'
```

> The above command returns JSON structured like this:

```json 
{
    "@dto": "UserProfile",
    "email": "test@geli.net"
}
```

Add a member to a role.

### HTTP Request 

`POST https://geni1.geli.net/auth/roles/{role}/members`

### Query Parameters

Parameter | Description
--------- | -----------
role | The name of the role to retrieve

### URL Parameters

No URL Parameters

### Authorization 

role: SUPERUSER 

## Delete a Member from a Role

```shell 
curl -X DELETE \
  https://geni1.geli.net/auth/roles/user/members/test@geli.net \
  -H 'Authorization: Token token=\"GENITOKEN"' \
```

> The above command returns an empty response (HTTP 204):

```json 

```

Remove a user from a role.

### HTTP Request 

`DELETE https://geni1.geli.net/auth/roles/{role}/members/{email}`

### Query Parameters

Parameter | Description
--------- | -----------
role | The name of the role to retrieve
email | User's URL-encoded email address 

### URL Parameters

No URL Parameters

### Authorization 

role: SUPERUSER 

# Zone HTTP API 

The Zone HTTP API provides services for GENI user interface applications (site configuration, end user utility view, etc). 

It’s a mostly-RESTful API concerned with performing CRUD operations on GENI Zone domain objects. 

It also has RPC-style methods to generate straightforward telemetry and event history reports for resources, and to send commands directly to EOS modules and devices for modal control of applications and remote device fault clearing.

The Zone HTTP API is distinct from the queue-over-websockets communication protocol between EOS and GENI, although they share some DTO types and conventions.

## API Prefix 

> Get name and version information for the API 
```shell 
curl \
  https://geni1.geli.net/api/v1/ \
  -H 'Authorization: Token token=\"GENITOKEN"' \
```

> The above command returns JSON structured like this:

```json
{
    "@dto": "AppEnvironment",
    "software": "release.38"
    [...]
 
}
```

Current prefix: 

`/api/v1`

<aside name="notice">
 This prefix does NOT apply to the authentication **/auth/** endpoints.
 </aside> 

# Site Management 

## Get All Sites  


```shell 
curl \
  https://geni1.geli.net/api/v1/sites \
  -H 'Authorization: Token token=\"GENITOKEN"' \
```

> The above command returns JSON structured like this:

```json 
 {
            "@dto": "Site",
            "siteCode": "amped1",
            "name": "Amped - Panasonic",
            "streetAddress": "2001 Sanyo Ave",
            "city": "San Diego",
            "stateOrProvinceCode": "CA",
            "zipCode": "92154",
            "countryCodeA2": "US",
            "centerLatitude": 32.562986,
            "centerLongitude": -116.938555,
            "manualGeolocation": false,
            "timeZone": "America/Los_Angeles",
            "manualTimeZone": true,
            "id": "fe29b575-9da5-473c-a9e8-ad742ac060a9",
            "commissioned": 1519340459183,
            "status": {
                "timestamp": 1519696104562,
                "metrics": {
                    "NET_METER": 118434
                }
            }
        },
        [...]
 }
```

Retrieve a list of all sites the current user is allowed to access.

### HTTP Request 

`GET https://geni1.geli.net/api/v1/sites`

### URL Parameters

Parameter (?) | Description | Function (=)
--------- | ----------- | ---------
max | maximum number of results | 
onlyDeleted | returns a list of “deleted” sites
detail | detail requested | status, name, all 

### Function Descriptions 
Function | Description 
--------- | -----------
status | status only 
name | status, name, location 
all | status, name, location and available nodes/devices

### Authorization 

role: any authenticated account . role SUPERUSER is required to use option onlyDeleted=true

## Get a Specific Site


```shell 
curl \
  https://geni1.geli.net/api/v1/sites/amped1 \
  -H 'Authorization: Token token=\"GENITOKEN"' \
```

> The above command returns JSON structured like this:

```json 
 {
            "@dto": "Site",
            "siteCode": "amped1",
            "name": "Amped - Panasonic",
            "streetAddress": "2001 Sanyo Ave",
            "city": "San Diego",
            "stateOrProvinceCode": "CA",
            "zipCode": "92154",
            "countryCodeA2": "US",
            "centerLatitude": 32.562986,
            "centerLongitude": -116.938555,
            "manualGeolocation": false,
            "timeZone": "America/Los_Angeles",
            "manualTimeZone": true,
            "id": "fe29b575-9da5-473c-a9e8-ad742ac060a9",
            "commissioned": 1519340459183,
            "status": {
                "timestamp": 1519696104562,
                "metrics": {
                    "NET_METER": 118434
                }
            }
        }
 }
```

Retrieve a specific site by sitecode.

### HTTP Request 

`GET https://geni1.geli.net/api/v1/sites/{site}`

### URL Parameters

Parameter (?) | Description | Function (=)
--------- | ----------- | ---------
detail | detail requested | status, name, all 

### Function Descriptions 
Function | Description 
--------- | -----------
status | status only 
name | status, name, location 
all | status, name, location and available nodes/devices

### Authorization 

role: any authenticated account to view site details .


