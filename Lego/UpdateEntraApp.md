# Function to update a Microsoft Entra App

Occasionally, we release new versions of our scripts that include enhancements or new features. These updates may require additional API permissions. Providing clear instructions to end users on how to configure these permissions is crucial to prevent potential issues or unexpected behavior, especially for users with limited experience. By simplifying and clearly documenting the required steps, we can help ensure a smoother implementation process.

```powershell
function UpdateMicrosoftEntraApp
{
	Connect-MgGraph -Scopes "Application.ReadWrite.All", "AppRoleAssignment.ReadWrite.All" -NoWelcome
	Clear-Host
	
	Write-Host "`n`n----------------------------------------------------------------------------------------"
	Write-Host "`nMicrosoft Entra App update!" -ForegroundColor DarkGreen
	Write-Host "This menu helps to validate that the Microsoft Entra App previously created have all the API permissions required." -ForegroundColor DarkGreen
	Write-Host "You will need to consent permissions Under Microsoft Entra portal to the app and the new permissions." -ForegroundColor DarkGreen
	Write-Host "`n----------------------------------------------------------------------------------------"
	
	$json = Get-Content -Raw -Path $ConfigFile
	[PSCustomObject]$config = ConvertFrom-Json -InputObject $json
	$AppID = $config.AppClientID
	
    $filter = "AppId eq '$AppId'"
    $servicePrincipal = Get-MgServicePrincipal -All -Filter $filter
    $roles = Get-MgServicePrincipalAppRoleAssignment -ServicePrincipalId ($servicePrincipal.Id)
    if ($roles.AppRoleId -notcontains "dc50a0fb-09a3-484d-be87-e023b12c6440")
    {
        Write-Host "Office 365 Exchange Online API permission 'Exchange.ManageAsApp'" -NoNewLine
        Write-Host "`tNot Found!" -ForegroundColor Red
		Write-Host "App ID used:" $AppId
        Write-Host "Press any key to continue..."
        $key = ([System.Console]::ReadKey($true))
        Write-Host "`nAdding permission...`n"
        # app parameters and API permissions definition
        $params = @{
            AppId = $AppID
            RequiredResourceAccess = @(
                @{
                    # Office 365 Exchange Online API ID
					ResourceAppId = "00000002-0000-0ff1-ce00-000000000000"
                    ResourceAccess = @(
                        @{
                            # Permission used to execute ExchangeOnline cmdlets
							# Exchange.ManageAsApp - Application
							Id = "dc50a0fb-09a3-484d-be87-e023b12c6440"
                            Type = "Role"
                        }
                    )
                }
        
            )
        }
        Update-MgApplicationByAppId @params
        Write-Host "Permission added." -ForegroundColor Green
        Write-Host "`nPlease go to the Azure portal to manually grant admin consent:"
        Write-Host "https://portal.azure.com/#view/Microsoft_AAD_RegisteredApps/ApplicationMenuBlade/~/CallAnAPI/appId/$($AppId)`n" -ForegroundColor Cyan    
    }
    else 
    {
        Write-Host "Office 365 Exchange Online API permission..." -NoNewLine
		Write-Host "`t'Exchange.ManageAsApp'" -NoNewLine -ForegroundColor Green
        Write-Host "`tpermission already in place." 
		Start-Sleep -s 3
    }
}
```
