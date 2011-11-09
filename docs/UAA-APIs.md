# User Account and Authentication Service APIs

## Overview

The User Account and Authentication Service (UAA) is:

* A new, separate application from the Cloud Controller
* Owns the user accounts and authentication sources
* Called via JSON APIs
* support for standard protocols to provide single sign-on and delegated authorization to web applications
in addition to JSON APIs to support the Cloud Controller and Collaboration Spaces
* Will also eventually support APIs for a Web UI for user account management

(Rather than trigger arguments about how RESTful these APIs are we'll just refer to them as JSON APIs. Most of them are defined by the specs for the OAuth2, OpenID Connect, and SCIM standards.)

APIs in this document:

Authentication and Delegated Authorization APIs

* [Get authorization approval or access token: /authorize](#authorize)
* [Validate access token: /check_token](#check_token)
* [Get access token: /token](#token)
* [Validate authentication token: /check_id](#check_id)
* [Get user information: /userinfo](#userinfo)
* [Get login information and prompts: /login_info](#login_info)

User Account APIs

* [Create a user: /User](#createuser)
* [Update a user: /User/{id}](#updateuser)
* [Query for information about users: /Users](#queryuser)
* [Delete a user: /User/{id}](#deleteuser)


## Configuration Options

These options are expected to be set in configuration files.  Settings
for a handful of standard scenarios can be externalized and switched
using environment variables or system properties.

*  **Internal username/password authentication source**

    The UAA will contain a user account database. This account can be used for password based authentication similar to existing Cloud Foundry user accounts. The UAA accounts could be configured with password policy such as length, accepted/required character types, expiration times, reset policy, etc.

* **Other Authentication sources**

    External authentication sources can also be built and tested as
    standard alternatives.  The most common and therefore the expected
    starting point are LDAP server, or an external OpenID provider
    (e.g. Google). Another expected authentication source would be Horizon Application Manager either through OAuth2 (preferred), or SAML protocols. General SAML2 support is not currently planned but could be added and would provide capabilities similar to OpenID and OAuth. 

## Authentication and Delegated Authorization API Endpoints

This section deals with machine interactions, not with browsers, although some of them may have browsable content for authenticated users.  All machine requests have accept headers indicating JSON (or a derivation perhaps).  

The `/userinfo`, `/check_id`, and `/token` endpoints are specified in the [OpenID Connect][] and [OAuth2][] standards and should be used by web applications on a cloud foundry instance such as studio, www, support, but will not be used by flows from vmc.

[OAuth2]: http://tools.ietf.org/html/draft-ietf-oauth-v2-22 "OAuth2, draft 22"
[OpenID Connect]: http://openid.net/openid-connect "OpenID Connect Spec Suite"


### <a id="authorize"/>OAuth2 Authorization Approval

An [OAuth2][] defined endpoint to provide various tokens and authorization codes.

For the flows in this document, we will probably be using the OAuth2
Implicit grant type (to avoid a second round trip to `/token` and so vmc does not need to securely store a refresh token). The
authentication method for the user is undefined by OAuth2 but a POST
to this endpoint is acceptable, although a GET must also be supported
([see the spec][OAuth2-3.1]).

[OAuth2-3.1]: http://tools.ietf.org/html/draft-ietf-oauth-v2-22#section-3.1

Effectively this means that the endpoint is used to authenticate _and_
obtain an access token in the same request.  Note the correspondence
with the UI endpoints (this is similar to the `/login` endpoint with a
different representation).

N.B. a GET is used in the [relevant
section](http://tools.ietf.org/html/draft-ietf-oauth-v2-22#section-4.2.1)
of the spec that talks about the implicit grant, but a POST is explicitly allowed in the 
[section on the `/authorize` endpoint, paragraph 5][OAuth2-3.1].

All requests to this endpoint MUST be over SSL. 

For our initial use we will use the OAuth2 `scope` parameter to bind any resulting access\_tokens to a cloud foundry instance. In the future scopes could be further refined such that the user could restrict the authorizations represented by the access\_token to a specific subset of resources (such as within an Org) or to a subset of permissions (such as read-only).

* Request: `POST /authorize`
* Request Body: some parameters specified by the spec, appended to the query component using the "application/x-www-form-urlencoded" format,

  * `response_type=token`
  * `client_id=vmc`
  * `scope=http://api.cloudfoundry.example.com`
  * `redirect_uri` is optional because it can be pre-registered, but a dummy is still needed where vmc is concerned (it doesn't redirect)

  and some required by us for authentication, submitted as a parameter named "credentials" which contains the 
  required information in JSON as returned from the [Login Information Endpoint](#login_info) endpoint,

  * e.g. username/password for internal authentication, or for LDAP
  * ... others as needed for other authentication types

* Response Header: location as defined in the spec includes `access_token` if successful

        HTTP/1.1 302 Found
        Location: https://client.example.com/cb?code=SplxlOBeZQQYbYS6WxSbIA
        
        
        HTTP/1.1 200 OK
        Content-Type: application/json;charset=UTF-8
        Cache-Control: no-store
        Pragma: no-cache
        {
        "access_token":"2YotnFZFEjr1zCsicMWpAA",
        "token_type":"example",
        "expires_in":3600,
        "example\_parameter":"example\_value"
        }        
        

* Response Codes: 

        302 - Found

### <a id="check_token"/>OAuth2 Token Validation Service

An endpoint that allows a resource server such as the cloud controller to validate an access token. Interactions between the resource server and the authorization provider are not specified in OAuth2, so we are adding this endpoint. The request should be over SSL and use basic auth with the shared secret between the UAA and the cloud controller. The POST body should be the access\_token and the response should include the userID, user_name, scope in json format.  The client (not the user) is authenticated via basic auth for this call.

OAuth2 access\_tokens could contain all needed information such as userID, scope(s), lifetime, user attributes and be encrypted with the resource server's (e.g. cloud countroller's) secret key. That way the resource server could validate and use the token with no contact with the UAA. However, it may be useful -- at least during development -- for the UAA to specify a short, opaque token and then provide a way for the resource server to return it to the UAA to validate and open. That is what this endpoint does. It does not return general user account information like the /userinfo endpoint, it is specifically to validate and return the information represented by access\_token that the user presented to the resource server. 

This endpoint mirrors the OpenID Connect `/check_id` endpoint, so not very RESTful, but we want to make it look and feel like the others. 

* Request: the basic authorization is as per OAuth2 spec, assuming the resource server is a registered client: `base64(resource_server:shared_secret)`

        POST /check_token HTTP/1.1
        Host: server.example.com
        Authorization: Basic QWxhZGRpbjpvcGVuIHNlc2FtZQ==
        Content-Type: application/x-www-form-encoded

        token=eyJ0eXAiOiJKV1QiL

* Successful Response: 

        HTTP/1.1 200 OK
        Content-Type: application/json

        {
            "user_id":"9017092310y301qhe",
            "user_name":"bjensen",
            "scope": "https://api.cloudfoundry.example.com",
            "expiration": "231231231231"
        }

* Error Responses: see [OAuth2 Error reponses](http://tools.ietf.org/html/draft-ietf-oauth-v2-22#section-5.2) and this addition:

            HTTP/1.1 400 Bad Request
            Content-Type: application/json;charset=UTF-8
            Cache-Control: no-store
            Pragma: no-cache

            { "error":"invalid_token" }

### <a id="token"/>OAuth2 Token Endpoint

_Not needed for first phase. This endpoint would be used by web flows, not vmc._

An OAuth2 defined endpoint which accepts authorization code or refresh tokens and provides access\_tokens. The access\_tokens can then be used to gain access to resources within a resource server. 

* Request: `POST /token`

### <a id="check_id"/>OpenID Check ID Endpoint

_Not needed for first phase. This endpoint would be used by web flows, not vmc._

An OpenID Connect defined endpoint. It accepts an id_token, which contains claims about the authentication event. It validates the token and returns information contained in the token in JSON format. Basically makes it so that clients do not need to have full token handling implementations.

* Request: `POST /check_id`

### <a id="userinfo"/>OpenID User Info Endpoint

_Not needed for first phase. This endpoint would be used by web flows, not vmc._

An OAuth2 protected resource and an OpenID Connect endpoint. Given an appropriate access_token, returns information about a user. Defined fields include various standard user profile fields. The response may include other user information such as group membership.

* Request: `GET /userinfo`

### <a id="login_info"/>Login Information Endpoint

An endpoint which returns login information, e.g prompts for authorization codes or one-time passwords. This allows vmc to determine what login information it should collect from the user.
	
This call will be unauthenticated.

* Request: `GET /login_info`
* Request body: _empty_
* Response body: _example_

        HTTP/1.1 200 OK
        Content-Type: application/json

        "prompt": {
            "email":["text", "validated email address"],
            "password": ["hidden", "your UAA password" ]
            "otp":["hidden", "security code"],
        }


## User Account APIs and Endpoints

The plan is to support
[Simple Cloud Identity Management (SCIM)](http://simplecloud.info) for
these APIs and endpoints.  Authentication is by OAuth2 token, and
access decision is undefined - which users are allowed to do these
operations?  Since this is independent of Collab Spaces a simple
(role-based) decision based on local data should be fine.  TODO: how
should it be bootstrapped - how does the first user get created?

SCIM has endpoints in /group/* as well which are probably useful (for
the local access decisions in the UAA), but we don't need to support
groups in UAA yet. We need to pass through based on attributes from
external stores like LDAP (which could be groups).

### <a id="createuser"/>Create a User

See [SCIM - Creating Resources](http://www.simplecloud.info/specs/draft-scim-rest-api-01.html#create-resource).

* Request: `POST /User`
* Request Body:

        {
          "schemas":["urn:scim:schemas:core:1.0"],
          "userName":"bjensen",
          "externalId":"bjensen",
          "name":{
            "formatted":"Ms. Barbara J Jensen III",
            "familyName":"Jensen",
            "givenName":"Barbara"
          }
        }

* Response Body:

        HTTP/1.1 201 Created
        Content-Type: application/json
        Location: https://example.com/v1/User/uid=bjensen,dc=example,dc=com

        {
          "schemas":["urn:scim:schemas:core:1.0"],
          "id":"uid=bjensen,dc=example,dc=com",
          "externalId":"bjensen",
          "meta":{
            "created":"2011-08-01T21:32:44.882Z",
            "lastModified":"2011-08-01T21:32:44.882Z"
          },
          "name":{
            "formatted":"Ms. Barbara J Jensen III",
            "familyName":"Jensen",
            "givenName":"Barbara"
          },
          "userName":"bjensen"
        }

* Response Codes:

        201 - Created successfully
        400 - Bad Request (unparseable, syntactically incorrect etc)
        401 - Unauthorized


### <a id="updateuser"/>Update a User

See [SCIM - Modifying with PUT](http://www.simplecloud.info/specs/draft-scim-rest-api-01.html#edit-resource-with-put)

* Request: `PUT /User/:id`
* Request Body:

        Host: example.com
        Accept: application/json
        Authorization: Bearer h480djs93hd8
        ETag: "a330bc54f0671c9"

        {
          "schemas":["urn:scim:schemas:core:1.0"],
          "id":"uid=bjensen,dc=example,dc=com",
          "userName":"bjensen",
          "externalId":"bjensen",
          "name":{
            "formatted":"Ms. Barbara J Jensen III",
            "familyName":"Jensen",
            "givenName":"Barbara",
            "middleName":"Jane"

          },
          "emails":[
            {
                "value":"bjensen@example.com"
            },
            {
                "value":"babs@jensen.org"
            }
          ]
        }

* Response Body:
        As for create operation, returns the entire, updated record, with the Location header pointing to the resource.
* Response Codes:

        200 - Updated successfully
        400 - Bad Request
        401 - Unauthorized
        404 - Not found

  Note: SCIM also optionally supports partial update using PATCH.

### <a id="queryuser"/>Query for Information about a User

See [SCIM - List/Query Resources](http://www.simplecloud.info/specs/draft-scim-rest-api-01.html#query-resources)

Get information about a user. This is needed by Collab Spaces to convert names and email addresses to immutable ids, and immutable ids to display names. SCIM supports more complex querying than we require. A simple "equals" filter for attributes should be adequate.

* Request: `GET /Users?attributes=:requestedAttributes&filter=:filter`
* Response Body (for `GET /Users?attributes=id&filter=emails.value eq bjensen@example.com`):

        HTTP/1.1 200 OK
        Content-Type: application/json

        {
          "totalResults":1,
          "schemas":["urn:scim:schemas:core:1.0"],
          "Resources":[
            {
              "id":"uid=bjensen,dc=example,dc=com"
            }
          ]
        }


* Response Codes:

        200 - Success
        400 - Bad Request
        401 - Unauthorized

### <a id="deleteuser"/>Delete a User

See [SCIM - Deleting Resources](http://www.simplecloud.info/specs/draft-scim-rest-api-01.html#delete-resource)

* Request: `DELETE /User/:id
* Request Body: Empty
* Response Body: Empty
* Response Codes: 

        200 - Success
        401 - Unauthorized
        404 - Not found


## UI Endpoints

_N.B. This section contains an initial proposal. These endpoints are not planned for the initial implementation phase._

Web app clients need UI endpoints for the OAuth2 and OpenID redirects. Clients that do not ask for a JSON content type will get HTML.

### Internal Login Form

* Request: `GET /login?error=:error`
* Response Body: form with all the relevant prompts
* Response Codes: 

    200 - Success

### Internal Login

* Request: `POST /login` 
* Request Body: depending on configuration (e.g. do we need OTP / PIN / password etc.)

    username=:username&password=:password...

* Response Header: includes location if redirect, and cookie for subsequent interaction (e.g. authorization)

    Location: http://myapp.cloudfoundry.com/mycoolpage
    Set-Cookie: JSESSIONID=ldfjhsdhafgkasd

* Response Codes: 

    302 - Found
    200 - Success

### OAuth2 Authorization Confirmation

* Request: `GET /authorize/confirm`
* Request Body: HTML form posts back to `/authorize`

    Do you approve the application "foo" to access your CloudFoundry 
    resources with scope "read_cloudfoundry"? Approve/Deny.

* Response Codes: 

    200 - Success

#### OAuth2 Authorization

The precise form of this request is not given by the spec (which just says "obtain authorization"), but the response is.

* Request: `POST /authorize?user_oauth_approval=true`
* Request Header: needed to ensure the currently authenticated client is the one that is authorizing

    Cookie: JSESSIONID=ldfjhsdhafgkasd

* Response Header: location as defined in the spec (e.g. includes auth code for that grant type, and error information)
* Response Codes: 

    302 - Found

### External Hosted Login Form (OpenID)

* Request: `GET /login`
* Response Header: 

    Location: http://www.google.com/etc/blah
    Set-Cookie: JSESSIONID=ldfjhsdhafgkasd
    
* Response Codes: 

    302 - Found