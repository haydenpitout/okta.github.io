---
layout: docs_page
title: How to Create a Token with a Groups Claim
excerpt: How to create an ID token or access token which contains a groups claim
---

## How to Create a Token with an Apps Groups Claim

Because Okta allows you to specify a profile with whatever JSON-compliant contents you wish, you can create a list of groups that are included in the token
in the groups claim. You can add a group claim for any app (Okta, Active Directory, or Workday, for example) into ID tokens or access tokens for API Access Management.

To do this, create an OpenID Connect client, assign a group whitelist to it, and configure a groups claim: 

1. Create an OpenID client at `/{yourOktaDomain}.com/api/v1/apps`
   
   Request Example:
   
    ~~~curl
    curl -X POST \
      http://{yourOktaDomain}.com/api/v1/apps \
      -H 'accept: application/json' \
      -H 'authorization: SSWS 00nWY8cjOh8nhyf-Un1SNZh-_ujW0bRee3jOYtT1-M' \
      -H 'cache-control: no-cache' \
      -H 'content-type: application/json' \
      -H 'postman-token: 87361ffc-a17f-1e4c-f812-6124ad763542' \
      -d '{
        "name": "oidc_client",
        "label": "Sample Client",
        "signOnMode": "OPENID_CONNECT",
        "credentials": {
          "oauthClient": {
            "token_endpoint_auth_method": "client_secret_post"
          }
        },
        "settings": {
          "oauthClient": {
            "client_uri": "http://www.example-application.com",
            "logo_uri": "http://application.example.com/assets/images/logo-new.png",
            "redirect_uris": [
              "https://example.com/oauth2/callback",
              "myapp://callback"
            ],
            "response_types": [
              "token",
              "id_token",
              "code"
            ],
            "grant_types": [
              "implicit",
              "authorization_code"
            ],
            "application_type": "native"
          }
        }
    }'
    ~~~


2. GET the group IDs you want at `{yourOktaDomain}.com/api/v1/groups`
   Note that this example uses one `groupId` for simplicity's sake.

   Request Example:
    ~~~
    curl -X GET \
      http://mysticorp.oktapreview.com/api/v1/groups \
      -H 'accept: application/json' \
      -H 'authorization: SSWS 00nWY8cjOh8nhyf-Un1SNZh-_ujW0bRee3jOYtT1-M' \
      -H 'cache-control: no-cache' \
      -H 'content-type: application/json' \
      -H 'postman-token: 4717a190-ebf9-d68c-f334-12458ccd1a4c' \
      -d '{
      "profile": {
        "name": "iPhone User",
        "description": "Example group of iPhone Users"
      }
    }'
    ~~~
    
    Response Example Fragment:
    ~~~json
        {
            "id": "00g5t60il64f9pkDW0h7",
            "created": "2016-02-09T23:10:21.000Z",
            "lastUpdated": "2016-02-09T23:10:21.000Z",
            "lastMembershipUpdated": "2017-01-13T18:43:45.000Z",
            "objectClass": [
                "okta:user_group"
            ],
            "type": "BUILT_IN",
            "profile": {
                "name": "Everyone",
                "description": "All users in your organization"
            } 
    ~~~
    
3. PUT the group IDs in the profile property bag: /{yourOktaDomain}.com/api/v1/apps/:clientId
    
    Request Example Fragment:
    ~~~curl
    "profile": {
            "groupwhitelist": [
                "00g5t60il64f9pkDW0h7"
                ]
       },
    ~~~
    
    You can select any group or set of groups, either app groups or user groups. 

    If you want to use this group whitelist for every client that gets this claim in a token, put the `groupId` in the first parameter of the `getFilteredGroups` function described below. 
 
4. Add an ID token claim on Custom Authorization Server in the Okta UI with the following function: getFilteredGroups({group ID list}, {value to represent the group in the token}, {max # of groups to include in token. Must be <=100})

        
    For more information, see [the `getFilteredGroups` documentation](xxx).

5. POST to the token endpoint `/{yourOktaDomain}.com/oauth2/:authorizationServerId/v1/token` with the user or client you want. Make sure the user is assigned to the app and to one of the groups from your whitelist.

6. Decode the JWT to see that the groups are in the token. 
