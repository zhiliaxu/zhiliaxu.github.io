## Environment variables

You can run below PowerShell script in Kudu site to list all environment variables.

```PowerShell
dir env:
```

The below environment variables are used for managed service identity.

```
MSI_ENDPOINT                   http://127.0.0.1:41566/MSI/token/               
MSI_SECRET                     AD36************************31E3
```

## Get JWT

You can run below PowerShell script to get a JWT.

```
$headers=@{"secret"=$env:MSI_SECRET}
$ProgressPreference = "SilentlyContinue"
$response = Invoke-WebRequest -UseBasicParsing -Uri "$($env:MSI_ENDPOINT)?resource=https://management.azure.com/&api-version=2017-09-01" -Headers $headers
$response.RawContent
```

Other valid values for `resource` are `https://storage.azure.com/`, `https://keyvault.azure.com/`, `https://graph.microsoft.com/`, and so on.
