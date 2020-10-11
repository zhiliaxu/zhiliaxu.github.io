---
title: Get Access Tokens for Managed Service Identity on Azure App Service
nav_order: 1
---

# Get Access Tokens for Managed Service Identity on Azure App Service
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

In this post, I give some complete examples online about how to get access token for managed service identity on Azure App Service, both system assigned and user assigned.

## Environment variables

Let's first check what environment variables you can leverage to call the identity endpoint on App Service. You can run below PowerShell script in Kudu site to list all environment variables.

```PowerShell
dir env:
```

The below environment variables exist for managed service identity.

```
IDENTITY_ENDPOINT              http://127.0.0.1:41566/MSI/token/
IDENTITY_HEADER                AD36************************31E3
```

`MSI_ENDPOINT` is an alias for `IDENTITY_ENDPOINT`, and `MSI_ENDPOINT` is an alias for `IDENTITY_HEADER`.

## Get JWT

Now let's get an access token by leveraging the identity endpoint on localhost, assuming we want to access Azure Storage from the App Service.

### System Assigned Managed Identity

Enable system assigned managed service identity in Itentity tab. And then run below PowerShell script to get a JWT for the system managed identity.

Using the `2019-08-01` API version.

```PowerShell
$headers=@{"X-IDENTITY-HEADER"=$env:IDENTITY_HEADER}
$ProgressPreference = "SilentlyContinue"
$response = Invoke-WebRequest -UseBasicParsing -Uri "$($env:IDENTITY_ENDPOINT)?resource=https://storage.azure.com/&api-version=2019-08-01" -Headers $headers
$response.RawContent
```

Using the `2017-09-01` API version.

```PowerShell
$headers=@{"secret"=$env:IDENTITY_HEADER}
$ProgressPreference = "SilentlyContinue"
$response = Invoke-WebRequest -UseBasicParsing -Uri "$($env:IDENTITY_ENDPOINT)?resource=https://storage.azure.com/&api-version=2017-09-01" -Headers $headers
$response.RawContent
```

Note that the header value is `secret` for `2017-09-01` version, and `X-IDENTITY-HEADER` for `2019-08-01` version.

### User Assigned Managed Identity

Add a user assigned managed identity according to [https://docs.microsoft.com/en-us/azure/app-service/overview-managed-identity?tabs=dotnet#add-a-user-assigned-identity](https://docs.microsoft.com/en-us/azure/app-service/overview-managed-identity?tabs=dotnet#add-a-user-assigned-identity). And then run below PowerShell script to get a JWT for the user assigned identity.

Using the `2019-08-01` API version.

```PowerShell
$headers=@{"X-IDENTITY-HEADER"=$env:IDENTITY_HEADER}
$ClientId = "********-****-****-****-************" # The Client ID of the user assigned identity
$ProgressPreference = "SilentlyContinue"
$response = Invoke-WebRequest -UseBasicParsing -Uri "$($env:IDENTITY_ENDPOINT)?resource=https://storage.azure.com/&client_id=$ClientId&api-version=2019-08-01" -Headers $headers
$response.RawContent
```

Using the `2017-09-01` API version.

```
$headers=@{"secret"=$env:IDENTITY_HEADER}
$ClientId = "********-****-****-****-************" # The Client ID of the user assigned identity
$ProgressPreference = "SilentlyContinue"
$response = Invoke-WebRequest -UseBasicParsing -Uri "$($env:IDENTITY_ENDPOINT)?resource=https://storage.azure.com/&clientid=$ClientId&api-version=2017-09-01" -Headers $headers
$response.RawContent
```

Note that the header value is `secret` for `2017-09-01` version, and `X-IDENTITY-HEADER` for `2019-08-01` version. The query string key for client ID is `clientid` for `2017-09-01` version, and `client_id` for `2019-08-01` version. Very tricky.

### Leveraging `Azure.Identity` NuGet Package

It is much easier to get access token using the `Azure.Identity` NuGet Package (https://www.nuget.org/packages/Azure.Identity). Under the hood, the library gets access token from the identity endpoint on App Service via a REST call.

Here is the sample code using system assigned identity.

```csharp
AccessToken token =
    await new DefaultAzureCredential()
    .GetTokenAsync(
        new TokenRequestContext(
            new[] { "https://storage.azure.com/" }
        ));
Console.WriteLine(token.Token);
```

Here is the sample code using user assigned identity.

```csharp
string clientId = "********-****-****-****-************" // The Client ID of the user assigned identity
AccessToken token =
    await new DefaultAzureCredential(
        new DefaultAzureCredentialOptions
        {
            ManagedIdentityClientId = clientId
        })
    .GetTokenAsync(
        new TokenRequestContext(
            new[] { "https://storage.azure.com/" }
        ));
Console.WriteLine(token.Token);
```

## Other values for `resource`

If you want to access other resources from App Service, you can change the `resource` query string to `https://storage.azure.com/`, `https://vault.azure.net/`, `https://graph.microsoft.com/`, and so on.
