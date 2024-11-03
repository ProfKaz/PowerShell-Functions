# Function to Connect to EDM service

You can find a complete detailed way to use Microsoft Purview Exact Data Match in this [link](https://github.com/ProfKaz/EDM-Post-Tasks), from another of my projects.
To accomplish this connection an application is required to pre-install first and set other variables, based on that installation.

```powershell
function Connect2EDM
{
	$CONFIGFILE = "$PSScriptRoot\EDMConfig.json"
	if (-not (Test-Path -Path $CONFIGFILE))
	{
		$CONFIGFILE = "$PSScriptRoot\EDM_RemoteConfig.json"
	}
	
	$json = Get-Content -Raw -Path $CONFIGFILE
	[PSCustomObject]$config = ConvertFrom-Json -InputObject $json
	$EncryptedKeys = $config.EncryptedKeys
	$EDMFolder = $config.EDMAppFolder
	$user = $config.User
	$SharedKey = $config.Password
	
	if ($EncryptedKeys -eq "True")
	{
		$SharedKey = DecryptSharedKey $SharedKey
		Set-Location $EDMFolder | cmd
		Clear-Host
		cls
		Write-Host "Validating connection to EDM..." -ForegroundColor Green
		.\EdmUploadAgent.exe /Authorize /Username $user /Password $SharedKey 
	}else{
		Set-Location $EDMFolder | cmd
		Clear-Host
		cls
		Write-Host "Validating connection to EDM..." -ForegroundColor Green
		.\EdmUploadAgent.exe /Authorize /Username $user /Password $SharedKey
	}
}
```
<br><br>
