# Function to create a Microsoft Entra App

To automate specific tasks, using a Service Principal is preferred over a user account. With API permissions, this approach allows various activities to be performed securely. The following function creates a Service Principal, commonly referred to as a Microsoft Entra Application, with specified Microsoft Graph API permissions:
- `User.Read` as Delegated (default)
- `User.Read.All` as Application
- `Group.ReadWrite.All` as Application
- `Directory.Read.All` as Application


<p align="center">
<img src="https://github.com/user-attachments/assets/b5ba2c8b-7d49-40bd-ab82-add5b1b5b840" width="650"></p>
<p align="center">How to identify API Id and Permission Id</p>
<br>

For unattended access, Microsoft Entra applications typically use one of two authentication methods: a `Secret Key` or a `Certificate Thumbprint`. This function includes steps to generate a certificate, install it locally, associate it with the Microsoft Entra Application, and update the existing configuration file. [Configuration file](/CreateConfigFile.md)

> [!IMPORTANT]
> Permissions granted in Microsoft Entra require final approval from a Global Admin, who must grant access permissions to the APIs specified in the Microsoft Entra Application.

```powershell
function CreateNewEntraApp
{
    Connect-MgGraph -Scopes "Application.ReadWrite.All", "AppRoleAssignment.ReadWrite.All", "Directory.ReadWrite.All", "User.ReadWrite.All" -NoWelcome

	$CONFIGFILE = $PSScriptRoot+"\ConfigFiles\EntraConfig.json"
	if(-not (Test-Path -path $CONFIGFILE))
	{
		CreateConfigFile
	}
	
	$json = Get-Content -Raw -Path $CONFIGFILE
	[PSCustomObject]$config = ConvertFrom-Json -InputObject $json
	
    $appName = "Microsoft Entra Groups"
    Get-MgApplication -ConsistencyLevel eventual -Count appCount -Filter "startsWith(DisplayName, 'Microsoft Entra Groups')" | Out-Null
    if ($appCount -gt 0)
    {   
        Write-Host "'Microsoft Entra Groups' app already exists.`n"
		Exit
    }

    # app parameters and API permissions definition
    $params = @{
        DisplayName = $appName
        SignInAudience = "AzureADMyOrg"
        RequiredResourceAccess = @(
            @{
            # Microsoft Graph API ID
            ResourceAppId = "00000003-0000-0000-c000-000000000000"
            ResourceAccess = @(
                @{
                    # This is the default permission added every time that a MIcrosoft Entra App is created
                    # User.Read - Delegated
                    Id = "e1fe6dd8-ba31-4d61-89e7-88639da4683d"
                    Type = "Scope"
                },
                @{
                    # Group.ReadWrite.All - Application
                    Id = "62a82d76-70ea-41e2-9197-370581804d09"
                    Type = "Role"
                },
                @{
                    # User.Read.All
                    Id = "df021288-bdef-4463-88db-98f22de89214"
                    Type = "Role"
                },
                @{
                    # Directory.Read.All
                    Id = "7ab1d382-f21e-4acd-a863-ba3e13f7da61"
                    Type = "Role"
                }
            )
        }
        )
    }
	
    # create application
    $app = New-MgApplication @params
    $appId = $app.Id

    # assign owner
    $userId = (Get-MgUser -UserId (Get-MgContext).Account).Id
    $params = @{
        "@odata.id" = "https://graph.microsoft.com/v1.0/directoryObjects/$userId"
    }
    New-MgApplicationOwnerByRef -ApplicationId $appId -BodyParameter $params

    # ask for certificate name
    $certName = "Microsoft Entra Groups"

    # certificate life
    $validMonths = 24

    # create key
    $cert = New-SelfSignedCertificate -DnsName $certName -CertStoreLocation "cert:\CurrentUser\My" -NotAfter (Get-Date).AddMonths($validMonths)
    $certBase64 = [System.Convert]::ToBase64String($cert.RawData)
    $keyCredential = @{
        type = "AsymmetricX509Cert"
        usage = "Verify"
        key = [System.Text.Encoding]::ASCII.GetBytes($certBase64)
    }
    while (-not (Get-MgApplication -ApplicationId $appId -ErrorAction SilentlyContinue)) 
    {
        Write-Host "Waiting while app is being created..."
        Start-Sleep -Seconds 5
    }
    Update-MgApplication -ApplicationId $appId -KeyCredentials $keyCredential -ErrorAction Stop
	$TenantID = (Get-MgContext).TenantId
	

    Write-Host "`nAzure application was created."
    Write-Host "App Name: $appName"
    Write-Host "App ID: $($app.AppId)"
	Write-Host "Tenant ID: $TenantID"
    Write-Host "Certificate thumbprint: $($cert.Thumbprint)"

    Write-Host "`nPlease go to the Azure portal to manually grant admin consent:"
    Write-Host "https://portal.azure.com/#view/Microsoft_AAD_RegisteredApps/ApplicationMenuBlade/~/CallAnAPI/appId/$($app.AppId)`n" -ForegroundColor Cyan

    $config.TenantGUID = $TenantID
	$config.AppClientID = $app.AppId
    $config.CertificateThumb = $cert.Thumbprint
	
	$config | ConvertTo-Json | Out-File $CONFIGFILE

    Remove-Variable cert
    Remove-Variable certBase64
}
```
