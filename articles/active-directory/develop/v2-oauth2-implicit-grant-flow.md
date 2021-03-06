---
title: Secure single-page applications using the Azure AD v2.0 implicit flow | Microsoft Docs
description: Building web applications using Azure AD's v2.0 implementation of the implicit flow for single-page apps.
services: active-directory
documentationcenter: ''
author: CelesteDG
manager: mtillman
editor: ''

ms.assetid: 3605931f-dc24-4910-bb50-5375defec6a8
ms.service: active-directory
ms.subservice: develop
ms.workload: identity
ms.tgt_pltfrm: na
ms.devlang: na
ms.topic: article
ms.date: 10/02/2018
ms.author: celested
ms.reviewer: hirsin
ms.custom: aaddev
---

# v2.0 Protocols - SPAs using the implicit flow

[!INCLUDE [active-directory-develop-applies-v2](../../../includes/active-directory-develop-applies-v2.md)]

With the v2.0 endpoint, you can sign users into your single-page apps with both personal and work or school accounts from Microsoft. Single-page and other JavaScript apps that run primarily in a browser face a few interesting challenges when it comes to authentication:

* The security characteristics of these apps are significantly different from traditional server-based web applications.
* Many authorization servers and identity providers do not support CORS requests.
* Full page browser redirects away from the app become particularly invasive to the user experience.

For these applications (AngularJS, Ember.js, React.js, etc), Azure Active Directory (Azure AD) supports the OAuth 2.0 Implicit Grant flow. The implicit flow is described in the [OAuth 2.0 Specification](https://tools.ietf.org/html/rfc6749#section-4.2). Its primary benefit is that it allows the app to get tokens from Azure AD without performing a backend server credential exchange. This allows the app to sign in the user, maintain session, and get tokens to other web APIs all within the client JavaScript code. There are a few important security considerations to take into account when using the implicit flow specifically around [client](https://tools.ietf.org/html/rfc6749#section-10.3) and [user impersonation](https://tools.ietf.org/html/rfc6749#section-10.3).

If you want to use the implicit flow and Azure AD to add authentication to your JavaScript app, we recommend you use the open source JavaScript library, [msal.js](https://github.com/AzureAD/microsoft-authentication-library-for-js).

However, if you prefer not to use a library in your single-page app and send protocol messages yourself, follow the general steps below.

> [!NOTE]
> Not all Azure AD scenarios and features are supported by the v2.0 endpoint. To determine if you should use the v2.0 endpoint, read about [v2.0 limitations](active-directory-v2-limitations.md).

## Protocol diagram

The following diagram shows what the entire implicit sign-in flow looks like and the sections that follow describe each step in more detail.

![OpenId Connect Swimlanes](./media/v2-oauth2-implicit-grant-flow/convergence_scenarios_implicit.png)

## Send the sign-in request

To initially sign the user into your app, you can send an [OpenID Connect](v2-protocols-oidc.md) authorization request and get an `id_token` from the v2.0 endpoint.

> [!IMPORTANT]
> To successfully request an ID token, the app registration in the [registration portal](https://apps.dev.microsoft.com) must have the **Allow Implicit Flow** enabled for the Web client. If it is not enabled, an `unsupported_response` error will be returned: **The provided value for the input parameter 'response_type' is not allowed for this client. Expected value is 'code'**

```
// Line breaks for legibility only

https://login.microsoftonline.com/{tenant}/oauth2/v2.0/authorize?
client_id=6731de76-14a6-49ae-97bc-6eba6914391e
&response_type=id_token
&redirect_uri=http%3A%2F%2Flocalhost%2Fmyapp%2F
&scope=openid
&response_mode=fragment
&state=12345
&nonce=678910
```

> [!TIP]
> To test signing in using the implicit flow, click <a href="https://login.microsoftonline.com/common/oauth2/v2.0/authorize?client_id=6731de76-14a6-49ae-97bc-6eba6914391e&response_type=id_token&redirect_uri=http%3A%2F%2Flocalhost%2Fmyapp%2F&scope=openid&response_mode=fragment&state=12345&nonce=678910" target="_blank">https://login.microsoftonline.com/common/oauth2/v2.0/authorize...</a> After signing in, your browser should be redirected to `https://localhost/myapp/` with an `id_token` in the address bar.
>

| Parameter |  | Description |
| --- | --- | --- |
| `tenant` | required |The `{tenant}` value in the path of the request can be used to control who can sign into the application. The allowed values are `common`, `organizations`, `consumers`, and tenant identifiers. For more detail, see [protocol basics](active-directory-v2-protocols.md#endpoints). |
| `client_id` | required |The Application Id that the registration portal ([apps.dev.microsoft.com](https://apps.dev.microsoft.com/?referrer=https://azure.microsoft.com/documentation/articles&deeplink=/appList)) assigned your app. |
| `response_type` | required |Must include `id_token` for OpenID Connect sign-in. It may also include the response_type `token`. Using `token` here will allow your app to receive an access token immediately from the authorize endpoint without having to make a second request to the authorize endpoint. If you use the `token` response_type, the `scope` parameter must contain a scope indicating which resource to issue the token for. |
| `redirect_uri` | recommended |The redirect_uri of your app, where authentication responses can be sent and received by your app. It must exactly match one of the redirect_uris you registered in the portal, except it must be url encoded. |
| `scope` | required |A space-separated list of scopes. For OpenID Connect, it must include the scope `openid`, which translates to the "Sign you in" permission in the consent UI. Optionally you may also want to include the `email` or `profile` [scopes](v2-permissions-and-consent.md) for gaining access to additional user data. You may also include other scopes in this request for requesting consent to various resources. |
| `response_mode` | optional |Specifies the method that should be used to send the resulting token back to your app. Defaults to query for an access token, but fragment if the request includes an id_token. |
| `state` | recommended |A value included in the request that will also be returned in the token response. It can be a string of any content that you wish. A randomly generated unique value is typically used for [preventing cross-site request forgery attacks](https://tools.ietf.org/html/rfc6749#section-10.12). The state is also used to encode information about the user's state in the app before the authentication request occurred, such as the page or view they were on. |
| `nonce` | required |A value included in the request, generated by the app, that will be included in the resulting id_token as a claim. The app can then verify this value to mitigate token replay attacks. The value is typically a randomized, unique string that can be used to identify the origin of the request. Only required when an id_token is requested. |
| `prompt` | optional |Indicates the type of user interaction that is required. The only valid values at this time are 'login', 'none', 'select_account', and 'consent'. `prompt=login` will force the user to enter their credentials on that request, negating single-sign on. `prompt=none` is the opposite - it will ensure that the user is not presented with any interactive prompt whatsoever. If the request cannot be completed silently via single-sign on, the v2.0 endpoint will return an error. `prompt=select_account` sends the user to an account picker where all of the accounts remembered in the session will appear. `prompt=consent` will trigger the OAuth consent dialog after the user signs in, asking the user to grant permissions to the app. |
| `login_hint`  |optional |Can be used to pre-fill the username/email address field of the sign in page for the user, if you know their username ahead of time. Often apps will use this parameter during re-authentication, having already extracted the username from a previous sign-in using the `preferred_username` claim.|
| `domain_hint` | optional |Can be one of `consumers` or `organizations`. If included, it will skip the email-based discovery process that user goes through on the v2.0 sign in page, leading to a slightly more streamlined user experience. Often apps will use this parameter during re-authentication, by extracting the `tid` claim from the id_token. If the `tid` claim value is `9188040d-6c67-4c5b-b112-36a304b66dad` (the Microsoft Account consumer tenant), you should use `domain_hint=consumers`. Otherwise, you can use `domain_hint=organizations` during re-authentication. |

At this point, the user will be asked to enter their credentials and complete the authentication. The v2.0 endpoint will also ensure that the user has consented to the permissions indicated in the `scope` query parameter. If the user has consented to **none** of those permissions, it will ask the user to consent to the required permissions. For more info, see [permissions, consent, and multi-tenant apps](v2-permissions-and-consent.md).

Once the user authenticates and grants consent, the v2.0 endpoint will return a response to your app at the indicated `redirect_uri`, using the method specified in the `response_mode` parameter.

#### Successful response

A successful response using `response_mode=fragment` and `response_type=id_token+token` looks like the following (with line breaks for legibility):

```
GET https://localhost/myapp/#
access_token=eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsIng1dCI6Ik5HVEZ2ZEstZnl0aEV1Q...
&token_type=Bearer
&expires_in=3599
&scope=https%3a%2f%2fgraph.microsoft.com%2fmail.read 
&id_token=eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsIng1dCI6Ik5HVEZ2ZEstZnl0aEV1Q...
&state=12345
```

| Parameter | Description |
| --- | --- |
| `access_token` |Included if `response_type` includes `token`. The access token that the app requested, in this case for the Microsoft Graph. The access token should not be decoded or otherwise inspected, it should be treated as an opaque string. |
| `token_type` |Included if `response_type` includes `token`. Will always be `Bearer`. |
| `expires_in`|Included if `response_type` includes `token`. Indicates the number of seconds the token is valid, for caching purposes. |
| `scope` |Included if `response_type` includes `token`. Indicates the scope(s) for which the access_token will be valid. May not include all of the scopes requested, if they were not applicable to the user (in the case of AAD-only scopes being requested when an personal account is used to log in). |
| `id_token` | A signed JSON Web Token (JWT). The  app can decode the segments of this token to request information about the user who signed in. The  app can cache the values and display them, but it should not rely on them for any authorization or security boundaries. For more information about id_tokens, see the [`id_token reference`](id-tokens.md). <br> **Note:** Only provided if `openid` scope was requested. |
| `state` |If a state parameter is included in the request, the same value should appear in the response. The app should verify that the state values in the request and response are identical. |

#### Error response

Error responses may also be sent to the `redirect_uri` so the app can handle them appropriately:

```
GET https://localhost/myapp/#
error=access_denied
&error_description=the+user+canceled+the+authentication
```

| Parameter | Description |
| --- | --- |
| `error` |An error code string that can be used to classify types of errors that occur, and can be used to react to errors. |
| `error_description` |A specific error message that can help a developer identify the root cause of an authentication error. |

## Validate the id_token

Just receiving an id_token is not sufficient to authenticate the user; you must also validate the id_token's signature and verify the claims in the token based on your app's requirements. The v2.0 endpoint uses [JSON Web Tokens (JWTs)](https://self-issued.info/docs/draft-ietf-oauth-json-web-token.html) and public key cryptography to sign tokens and verify that they are valid.

You can choose to validate the `id_token` in client code, but a common practice is to send the `id_token` to a backend server and perform the validation there. Once you've validated the signature of the id_token, there are a few claims you will be required to verify. See the [`id_token` reference](id-tokens.md) for more information, including [validating tokens](id-tokens.md#validating-an-idtoken) and [important information about signing key rollover](active-directory-signing-key-rollover.md). We recommend making use of a library for parsing and validating tokens - there is at least one available for most languages and platforms.

You may also wish to validate additional claims depending on your scenario. Some common validations include:

* Ensuring the user/organization has signed up for the app.
* Ensuring the user has proper authorization/privileges.
* Ensuring a certain strength of authentication has occurred, such as multi-factor authentication.

Once you have completely validated the id_token, you can begin a session with the user and use the claims in the id_token to obtain information about the user in your app. This information can be used for display, records, personalization, etc.

## Get access tokens

Now that you've signed the user into your single-page app, you can get access tokens for calling web APIs secured by Azure AD, such as the [Microsoft Graph](https://developer.microsoft.com/graph). Even if you already received a token using the `token` response_type, you can use this method to acquire tokens to additional resources without having to redirect the user to sign in again.

In the normal OpenID Connect/OAuth flow, you would do this by making a request to the v2.0 `/token` endpoint. However, the v2.0 endpoint does not support CORS requests, so making AJAX calls to get and refresh tokens is out of the question. Instead, you can use the implicit flow in a hidden iframe to get new tokens for other web APIs: 

```
// Line breaks for legibility only

https://login.microsoftonline.com/{tenant}/oauth2/v2.0/authorize?
client_id=6731de76-14a6-49ae-97bc-6eba6914391e
&response_type=token
&redirect_uri=http%3A%2F%2Flocalhost%2Fmyapp%2F
&scope=https%3A%2F%2Fgraph.microsoft.com%2Fmail.read&response_mode=fragment
&state=12345&nonce=678910
&prompt=none
&domain_hint=organizations
&login_hint=myuser@mycompany.com
```

For details on the query parameters in the URL, see [send the sign in request](#send-the-sign-in-request).

> [!TIP]
> Try copy & pasting the below request into a browser tab! (Don't forget to replace the `domain_hint` and the `login_hint` values with the correct values for your user)
>
>`https://login.microsoftonline.com/common/oauth2/v2.0/authorize?client_id=6731de76-14a6-49ae-97bc-6eba6914391e&response_type=token&redirect_uri=http%3A%2F%2Flocalhost%2Fmyapp%2F&scope=https%3A%2F%2Fgraph.microsoft.com%2Fmail.read&response_mode=fragment&state=12345&nonce=678910&prompt=none&domain_hint=consumers-or-organizations&login_hint=your-username`
>

Thanks to the `prompt=none` parameter, this request will either succeed or fail immediately and return to your application. A successful response will be sent to your app at the indicated `redirect_uri`, using the method specified in the `response_mode` parameter.

#### Successful response

A successful response using `response_mode=fragment` looks like:

```
GET https://localhost/myapp/#
access_token=eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsIng1dCI6Ik5HVEZ2ZEstZnl0aEV1Q...
&state=12345
&token_type=Bearer
&expires_in=3599
&scope=https%3A%2F%2Fgraph.windows.net%2Fdirectory.read
```

| Parameter | Description |
| --- | --- |
| `access_token` |Included if `response_type` includes `token`. The access token that the app requested, in this case for the Microsoft Graph. The access token should not be decoded or otherwise inspected, it should be treated as an opaque string. |
| `token_type` | Will always be `Bearer`. |
| `expires_in` | Indicates the number of seconds the token is valid, for caching purposes. |
| `scope` | Indicates the scope(s) for which the access_token will be valid. May not include all of the scopes requested, if they were not applicable to the user (in the case of AAD-only scopes being requested when an personal account is used to log in). |
| `id_token` | A signed JSON Web Token (JWT). Included if `response_type` includes `id_token`. The  app can decode the segments of this token to request information about the user who signed in. The  app can cache the values and display them, but it should not rely on them for any authorization or security boundaries. For more information about id_tokens, see the [`id_token` reference](id-tokens.md). <br> **Note:** Only provided if `openid` scope was requested. |
| `state` |If a state parameter is included in the request, the same value should appear in the response. The app should verify that the state values in the request and response are identical. |


#### Error response

Error responses may also be sent to the `redirect_uri` so the app can handle them appropriately. In the case of `prompt=none`, an expected error will be:

```
GET https://localhost/myapp/#
error=user_authentication_required
&error_description=the+request+could+not+be+completed+silently
```

| Parameter | Description |
| --- | --- |
| `error` |An error code string that can be used to classify types of errors that occur, and can be used to react to errors. |
| `error_description` |A specific error message that can help a developer identify the root cause of an authentication error. |

If you receive this error in the iframe request, the user must interactively sign in again to retrieve a new token. You can choose to handle this case in whatever way makes sense for your application.

## Validating access tokens

Once you receive an access_token, make sure to validate the signature of the token as well as the following claims. You may also choose to validate additional claims based on your scenario. 

* **audience** claim, to ensure that the token was intended to be given to your app
* **issuer** claim, to verify that the token was issued to your app by the v2.0 endpoint
* **not before** and **expiration time** claims, to verify that the token has not expired

For more information about the claims present in the access token, see the [access token reference](access-tokens.md)

## Refreshing tokens

The implicit grant does not provide refresh tokens. Both `id_token`s and `access_token`s will expire after a short period of time, so your app must be prepared to refresh these tokens periodically. To refresh either type of token, you can perform the same hidden iframe request from above using the `prompt=none` parameter to control Azure AD's behavior. If you want to receive a new `id_token`, be sure to use `response_type=id_token` and `scope=openid`, as well as a `nonce` parameter.

## Send a sign out request

The OpenIdConnect `end_session_endpoint` allows your app to send a request to the v2.0 endpoint to end a user's session and clear cookies set by the v2.0 endpoint. To fully sign a user out of a web application, your app should end its own session with the user (usually by clearing a token cache or dropping cookies), and then redirect the browser to:

```
https://login.microsoftonline.com/{tenant}/oauth2/v2.0/logout?post_logout_redirect_uri=https://localhost/myapp/
```

| Parameter |  | Description |
| --- | --- | --- |
| `tenant` |required |The `{tenant}` value in the path of the request can be used to control who can sign into the application. The allowed values are `common`, `organizations`, `consumers`, and tenant identifiers. For more detail, see [protocol basics](active-directory-v2-protocols.md#endpoints). |
| `post_logout_redirect_uri` | recommended | The URL that the user should be returned to after logout completes. This value must match one of the redirect URIs registered for the application. If not included, the user will be shown a generic message by the v2.0 endpoint. |

## Next steps

* Go over the [MSAL JS samples](https://github.com/AzureAD/microsoft-authentication-library-for-js/wiki/Samples) to get started coding.
