---
title: Handle errors and exceptions in MSAL for Python
description: Learn how to handle errors and exceptions, Conditional Access claims challenges, and retries in MSAL for Python applications.
---
# Handle errors and exceptions in MSAL for Python

The Microsoft Authentication Library (MSAL) apps generate log messages that can help diagnose issues. An app can configure logging with a few lines of code, and have custom control over the level of detail and whether or not personal and organizational data is logged. We recommend you create an MSAL logging implementation and provide a way for users to submit logs when they have authentication issues.

## Logging levels

MSAL provides several levels of logging detail:

- LogAlways: No level filtering is done on this log level. Log messages of all levels will be logged.
- Critical: Logs that describe an unrecoverable application or system crash, or a catastrophic failure that requires immediate attention.
- Error: Indicates something has gone wrong and an error was generated. Used for debugging and identifying problems.
- Warning: There hasn't necessarily been an error or failure, but are intended for diagnostics and pinpointing problems.
- Informational: MSAL will log events intended for informational purposes not necessarily intended for debugging.
- Verbose (Default): MSAL logs the full details of library behavior.

> [!NOTE]
> Not all log levels are available for all MSAL SDK's

## Personal and organizational data

By default, the MSAL logger doesn't capture any highly sensitive personal or organizational data. The library provides the option to enable logging personal and organizational data if you decide to do so.

The following sections provide more details about MSAL error logging for your application.

## Error handling in MSAL for Python

In MSAL for Python, most errors are conveyed as a return value from the API call. The error is represented as a dictionary containing the JSON response from the Microsoft identity platform.

* A successful response contains the `"access_token"` key. The format of the response is defined by the OAuth2 protocol. For more information, see [5.1 Successful Response](https://tools.ietf.org/html/rfc6749#section-5.1)
* An error response contains `"error"` and usually `"error_description"`. The format of the response is defined by the OAuth2 protocol. For more information, see [5.2 Error Response](https://tools.ietf.org/html/rfc6749#section-5.2)

When an error is returned, the `"error"` key contains a machine-readable code. If the `"error"` is, for example, an `"interaction_required"`, you may prompt the user to provide additional information to complete the authentication process. If the `"error"` is `"invalid_grant"`, you may prompt the user to reenter their credentials. The following snippet is an example of error handling in MSAL for Python.

```python

from msal import ConfidentialClientApplication

authority_url = "https://login.microsoftonline.com/your_tenant_id"
client_id = "your_client_id"
client_secret = "your_client_secret"
scopes = ["https://graph.microsoft.com/.default"]

app = ConfidentialClientApplication(client_id, authority=authority_url, client_credential=client_secret)

result = app.acquire_token_silent(scopes=scopes, account=None)

if not result:
    result = app.acquire_token_silent(scopes=scopes)

if "access_token" in result:
    print("Access token: %s" % result["access_token"])
else:
    print("Error: %s" % result.get("error"))

```

When an error is returned, the `"error_description"` key also contains a human-readable message, and there is typically also an `"error_code"` key which contains a machine-readable Microsoft identity platform error code. For more information about the various Microsoft identity platform error codes, see [Authentication and authorization error codes](/azure/active-directory/develop/reference-error-codes).

In MSAL for Python, exceptions are rare because most errors are handled by returning an error value. The `ValueError` exception is only thrown when there's an issue with how you're attempting to use the library, such as when API parameter(s) are malformed.

## Conditional Access and claims challenges

When getting tokens silently, your application may receive errors when a [Conditional Access claims challenge](/azure/active-directory/develop/v2-conditional-access-dev-guide) such as MFA policy is required by an API you're trying to access.

The pattern for handling this error is to interactively acquire a token using MSAL. This prompts the user and gives them the opportunity to satisfy the required Conditional Access policy.

In certain cases when calling an API requiring Conditional Access, you can receive a claims challenge in the error from the API. For instance if the Conditional Access policy is to have a managed device (Intune) the error will be something like [AADSTS53000: Your device is required to be managed to access this resource](/azure/active-directory/develop/reference-error-codes) or something similar. In this case, you can pass the claims in the acquire token call so that the user is prompted to satisfy the appropriate policy.


## Retrying after errors and exceptions

MSAL makes HTTP calls to the Azure AD service, and occasionally failures can occur.
For example the network can go down or the server is overloaded.

MSAL Python 1.11+ automatically performs one retry attempt for you.
You may customize this behavior by following
[this instruction](https://msal-python.readthedocs.io/en/latest/#msal.ConfidentialClientApplication.params.http_client).

### HTTP 429

When the Service Token Server (STS) is overloaded with too many requests,
it returns HTTP error 429 with a hint about how long until you can try again in the `Retry-After` response field.

Your app was expected to throttle the subsequent requests, and only retry after the specified period.
That was not an easy task.

MSAL Python 1.16+ made it easy for you, in that your app could blindly retry in any given time
(say, whenever the end user clicks the sign-in button again),
MSAL Python 1.16+ would automatically throttle those retry attempts by returning same error response from an HTTP cache,
and only sending out a real HTTP call when that call is attempted after the specified period.

By default, this throttle mechanism works by saving throttle information into a built-in in-memory HTTP cache.
You may provide your own `dict`-like object as the HTTP cache, which you can control how to persist its content.
See [MSAL Python's API document](https://msal-python.readthedocs.io/en/latest/#msal.PublicClientApplication.params.http_cache)
for more details.

## Next steps

Consider enabling [Logging in MSAL for Python](msal-logging-python.md) to help you diagnose and debug issues.