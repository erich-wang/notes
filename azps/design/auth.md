# Auth for Auto-Gen Cmdlets

## Why

For management plane cmdlets, the resource id(AD service endpoint) is unique and the authentication is unified. However, this is not the case for data plane cmdlets:

1. App configuration supports both AAD authentication and HMAC(Hash-based Message Authentication Code)

2. Storage supports both AAD authentication and connection string

3. Acr supports customzied authentication logic to get refresh key and access key

As auth for data plane cmdlets are diverse, we have to rely on service team to implement the customzied auth logic. It becomes curcial that Auto.PowerShell provides one way for module authors to easily configure and implement authentication.

## Curent Implementation

Auth for auto-generated management cmdlets is supported only.

The related auth logic is in Az.Accounts, need tweak the cmdlets and auth logic during module initialization.

### OnNewRequest

Add steps/handlers to HTTP pipeline

- Step to add user agent
- Step to Patch request URI
- Step to authorize request

### EventListener

Support debug trace and telemetry

## Proposed Design

Split `OnNewRequest` into three different handlers:

-AddRequestUserAgentHandler
-AddPatchRequestUriHandler
-AddAuthorizeRequestHandler

NOTE: The original `OnNewRequest` will be keeped for back-compatibility with released modules.

### Authentication Customization Level

- Cmdlet
- Module
- **Module Plane: Data Plane or Mgmt Plane**
<!--
```yml
set:
  auth-policy:
    type: Customized #supported types: AAD/Customized
    properties:
      name: HMACAuthPolicy #inherited from IAuthPolicy
```

```csharp
    interface IAuthPolicy
    {
        Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken token, Action cancel, SignalDelegate signal, NextDelegate next);
    }
```
-->
AAD auth is popular one for data plane, so we should provide one built-in way to support it. However, there's slight difference between different services, for example:

1. App configuration supports token audience as `https://<configName>.azconfig.io` or `https://*.azconfig.io`.
2. Storage supports token audience as `https://<account>.blob.core.windows.net`, `https://<account>.queue.core.windows.net` or `https://storage.azure.com/`.

```yml
set:
  auth-token-aud-converter: "AppConfigTokenAudienceConverter"
```

```csharp
    interface ITokenAudienceConverter
    {
        string Convert(string, string, string, string, System.Uri requestUri);
    }
```

*By default, AAD auth policy will be applied to auto-generated cmdlets if no auth-policy is set.*

```yml
set:
  endpoint-key: "AppConfigurationEndpointResourceId" # must be same as one in Environment
  endpoint-suffix-key: "AppConfigurationEndpointSuffix"
```

## Open Questions

1. Independent of Az.Accounts.

2. Think ARM authentication is one built-in authentication method.

3. Change from delegate to interface for better maintainence?
