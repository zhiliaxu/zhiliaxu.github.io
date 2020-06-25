---
title: How Do ASP.NET Core Services Validate JWT Signature Signed by AAD?
nav_order: 1
---

# How Do ASP.NET Core Services Validate JWT Signature?
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Background

 If we need to use JWT Bearer tokens issued by AAD (to either a user or service principal) for authentication, usually we can add below code to `ConfigureServices` in `Startup.cs`.

```csharp
services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = $"https://login.microsoftonline.com/{tenantId}";
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidIssuer = "...",
            ValidAudience = "..."
        };
    });
```

Then, by adding `[Authorize]` attribute for the `Controller` class, we get our APIs protected by AAD authentication. Authorization is also possible by some additional configurations.

It works without the need to provide any one of `IssuerSigningKey*` properties in `TokenValidationParameters`. It is common for us to ask below questions:

* **Is JWT signature validated after all?**
* **How is JWT signature validated by ASP.NET Core?**
* **Do I need to provide any signing key of certificate with public key to validate JWT signatures?**
* **Does my service need to call some REST API provided by AAD to validate the signature on each request?**

I try to answer these questions below, illustrated by [ASP.NET Core](https://github.com/dotnet/aspnetcore) and [AAD Identity Model Extensions for .NET](https://github.com/AzureAD/azure-activedirectory-identitymodel-extensions-for-dotnet) source code. If you want short answers to the questions, just skip to Summary section.

> [!TIP]
> The source code links below will be out of date soon when new versions come. Please help me update them by sending pull requests. Thank you!

## Configuration

As the `ConfigureServices` sample code in Background section shows, `AddJwtBearer` during configuration. In this [extension method](https://github.com/dotnet/aspnetcore/blob/master/src/Security/Authentication/JwtBearer/src/JwtBearerExtensions.cs#L44),

```csharp
public static AuthenticationBuilder AddJwtBearer<TService>(this AuthenticationBuilder builder, string authenticationScheme, string displayName, Action<JwtBearerOptions, TService> configureOptions) where TService : class
{
    builder.Services.TryAddEnumerable(ServiceDescriptor.Singleton<IPostConfigureOptions<JwtBearerOptions>, JwtBearerPostConfigureOptions>());
    return builder.AddScheme<JwtBearerOptions, JwtBearerHandler, TService>(authenticationScheme, displayName, configureOptions);
}
```

`builder.Services.TryAddEnumerable(ServiceDescriptor.Singleton<IPostConfigureOptions<JwtBearerOptions>, JwtBearerPostConfigureOptions>());` is called to register a `JwtBearerPostConfigureOptions`. `JwtBearerPostConfigureOptions` has a `PostConfigure()` method (see [source code](https://github.com/dotnet/aspnetcore/blob/master/src/Security/Authentication/JwtBearer/src/JwtBearerPostConfigureOptions.cs#L22)). `JwtBearerPostConfigureOptions.PostConfigure()` is called when the first request comes, within `AuthenticationHandler<JwtBearerOptions>.InitializeAsync()` method (`AuthenticationHandler<JwtBearerOptions>` is the base class of `JwtBearerHandler`) -> ... -> `OptionsFactory.Create()` (see [source code](https://github.com/dotnet/runtime/blob/master/src/libraries/Microsoft.Extensions.Options/src/OptionsFactory.cs#L58)).

In `JwtBearerPostConfigureOptions.PostConfigure()`, it sets the `JwtBearerOptions.ConfigurationManager` property. `options.MetadataAddress` is set based on `options.Authority`. For example, if `options.Authority` is `https://login.microsoftonline.com/<tenandId>`, then `options.MetadataAddress` is set to `https://login.microsoftonline.com/<tenandId>/.well-known/openid-configuration`. **In this URL, we can find signing keys at `https://login.microsoftonline.com/common/discovery/keys`.** `options.MetadataAddress` is passed to the `ConfigurationManager`'s constructor.

## Handle Authentication

When each request comes, `JwtBearerHandler.HandleAuthenticateAsync()` is called to do authentication (see [source code](https://github.com/dotnet/aspnetcore/blob/master/src/Security/Authentication/JwtBearer/src/JwtBearerHandler.cs#L46)). If it is called for the first time, it will retrieve and cache a configuration by calling `Options.ConfigurationManager.GetConfigurationAsync()`. Then the `TokenValidationParameters.IssuerSigningKeys` is set to `_configuration.SigningKeys` (see [source code](https://github.com/dotnet/aspnetcore/blob/master/src/Security/Authentication/JwtBearer/src/JwtBearerHandler.cs#L97)). **That's why we don't need to provide signing key ourselves in `Startup.ConfigureServices()`.**

Then it calls `validator.ValidateToken()` for each `validator` in `Options.SecurityTokenValidators`. There is only one `validator` in `Options.SecurityTokenValidators` - `JwtSecurityTokenHandler`.

## Validate Token

In `JwtSecurityTokenHandler.ValidateToken()` -> `JwtSecurityTokenHandler.ValidateSignature()` (see [source code](https://github.com/AzureAD/azure-activedirectory-identitymodel-extensions-for-dotnet/blob/dev/src/System.IdentityModel.Tokens.Jwt/JwtSecurityTokenHandler.cs#L902)), if `TokenValidationParameters.IssuerSigningKeyResolver` is null, `ResolveIssuerSigningKey()` and then `JwtTokenUtilities.FindKeyMatch()` will be called to find the matched key based on `kid` (key ID) and `x5t` (X.509 certificate SHA-1 thumbprint) in JWT header among the signing keys retrieved via `MetadataAddress`. Then the matched key will be used to validate JWT signature, with the help of the `token` itself, `signature`, `header.Alg` and crypto provider.

## Summary

Now it is clear that

* JWT signature is validated without providing any key or certification in our service's source code.
* JWT signing key is retrieved from the well-known URL `https://login.microsoftonline.com/common/discovery/keys`, based on `JwtBearerOptions.Authority` property.
* The signing key is cached in the `JwtBearerHandler` singleton instance, and so our ASP.NET Core service only needs to retrieve it once throughout its life cycle.
