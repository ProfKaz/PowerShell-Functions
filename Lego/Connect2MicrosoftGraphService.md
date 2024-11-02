# Function to connect to Microsoft Graph API manually or automatically

Use this function to connect to the Microsoft Graph API, either via a `ManualConnection` or through a [Microsoft Entra App](CreateNewEntraApp.md), where connection details are stored in a [Config File](CreateConfigFile.md). For `ManualConnection`, ensure the necessary scopes are specified for your tasks; this function currently uses the following scopes:
- `Group.ReadWrite.All`
- `Directory.ReadWrite.All`
- `User.Read.All`

If connecting via a  [Microsoft Entra Application](CreateNewEntraApp.md), set the required API permissions within the app configuration.

> [!NOTE]
> When running the script with `.\MyScript.ps1` to establish an automatic connection, if the configuration file is missing, you’ll receive a message indicating that you can run the script with an attribute to create a [Microsoft Entra Application](CreateNewEntraApp.md). Remember set this function in your script.

```powershell
function Connect2MicrosoftGraphService
{		

	<#
	.NOTES
	Special permissions to Microsoft Graph can be required, check the initial notes in each script
	#>
	if($ManualConnection)
	{
		Write-Host "`nAuthentication is required, please check your browser" -ForegroundColor Green
		Write-Host "Please note that manual connection might not work because some additional permissions may be required." -ForegroundColor DarkYellow
		Connect-MgGraph -Scopes "Group.ReadWrite.All", "Directory.ReadWrite.All", "User.Read.All" -NoWelcome
	}else
	{
		$ConfigFile = $PSScriptRoot+"\ConfigFiles\Config.json"
		
		#Check if the configuration file exist or not
		if(-Not (Test-Path -Path $ConfigFile))
		{
			Write-Host "`nConfiguration file not available, you have these options:"
			Write-Host "You can use for a manual connection : " -NoNewLine
			Write-Host "`t.\MyScript.ps1 -ManualConnection" -ForeGroundColor Green
			Write-Host "You can configure a Microsoft Entra App to automate the connection using : " -NoNewLine
			Write-host "`t.\MyScript.ps1 -CreateEntraApp`n`n" -ForeGroundColor Green
			exit
		}
		
		$json = Get-Content -Raw -Path $ConfigFile
		[PSCustomObject]$config = ConvertFrom-Json -InputObject $json
		
		$EncryptedKeys = $config.EncryptedKeys
		$AppClientID = $config.AppClientID
		$CertificateThumb = $config.CertificateThumb
		$TenantGUID = $config.TenantGUID

		$status = CheckCertificateInstalled -thumbprint $CertificateThumb
		
		if($status -eq "True")
		{
			Connect-MgGraph -CertificateThumbPrint $CertificateThumb -AppID $AppClientID -TenantId $TenantGUID -NoWelcome
		}else
		{
			Write-Host "`nThe Certificate set in EntraConfig.json don't match with the certificates installed on this machine, you can try to execute using manual connection, to do that extecute: "
			Write-Host ".\NestedGroupsBasedOnManager.ps1 -ManualConnection" -ForeGroundColor Green
			exit
		}
		
	}
}
```

To use this function you need to set at the begin of the script a `param` variables like this:
```powershell
param(
	[Parameter()] 
        [switch]$ManualConnection
)
```

Having that parameter set you can call your script in this way to use `ManualConnection`:
```powershell
.\MyScript.ps1 -ManualConnection
```
<br><br>
