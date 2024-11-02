# Function to Create a Configuration File for Scripts

This function creates a configuration file named `Config.json` inside a folder called `ConfigFiles`. If `ConfigFiles` does not exist, the function creates it. The configuration file includes the following attributes:
- `AppClientID`: Initially set as empty, though a default value can be assigned here if needed.
- `TenantGUID`: Set as empty by default; this value can also be populated by another function if necessary.
- `CertificateThumb`: Initially set as empty.

This function allows you to set multiple attributes, enabling customization based on the requirements of different scripts.

```powershell
function CreateConfigFile
{
	  # Set the path to the config file
    $configfile = $PSScriptRoot+"\ConfigFiles\Config.json"
	
	if(-Not (Test-Path $configfile ))
	{
		Write-Host "Export data directory is missing, creating a new folder called ConfigFiles"
		New-Item -ItemType Directory -Force -Path "$PSScriptRoot\ConfigFiles" | Out-Null
	}
	
	if (-not (Test-Path -Path $configfile))
    {
		$config = [ordered]@{
		AppClientID = ""
		TenantGUID = ""
		CertificateThumb = ""
		}
    }else
	{
		Write-Host "Configuration file is available under ConfigFiles folder"
	}
	
	$config | ConvertTo-Json | Out-File "$configfile"
    Write-Host "New config file was created under ConfigFile folder." -ForegroundColor Yellow
}
```
<br><br>
