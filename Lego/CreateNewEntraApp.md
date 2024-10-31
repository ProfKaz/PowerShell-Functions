# Function to create a Microsoft Entra App

To automate certain activities, the normal way is use a Service Principal instead of an account, in that case through API permissions we are able to permit different kind of activities, the next function permit to create this kind of Service Principal, commonly known as Microsoft Entra Application.
In the sample function below I'm adding Microsoft Graph API application permissions correspondign to:
- User.Read as Delegated : This is by default
- User.Read.All as Application
- Group.ReadWrite.All as Application
- Directory.Read.All as Application


<p align="center">
<img src="https://github.com/user-attachments/assets/b5ba2c8b-7d49-40bd-ab82-add5b1b5b840" width="650"></p>
<p align="center">How to identify API Id and Permission Id</p>
<br>

To work in an unattended way with this kind of Microsoft Entra Application normally exist 2 common ways to connect, one is setting a `Secret Key`and the other one is through a `Certificate thumbprint` in this function we can find some lines that create that certificate, install the certificate locally, import the certificate under the Microsoft Entra Application and update an existing [Configuration file](/CreateConfigFile.md)

> [!IMPORTANT]
> The permissions granted at Microsoft Entra requires a last step by a Global Admin user, who need to grant access permissions to the APIs set in our Microsot Entra App.

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
