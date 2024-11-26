# Function to validate if the cmdlet can be executed

The following function is designed to determine whether the `Export-ContentExplorerData` cmdlet is available for execution. By performing this check, potential errors are avoided when attempting to run the cmdlet.

```powershell
function CheckContentExplorerPermissions
{
	 if (-not (Get-Command -Name Export-ContentExplorerData -ErrorAction SilentlyContinue)) 
	 {
		Write-Host "You don´t have the permissions required to execute the cmdlet Export-ContentExplorerData"
		Write-Host "Please sign-in again with an account with these permissions assigned :"
		Write-Host "`t* Content Explorer Content Viewer"
		Write-Host "`t* Content Explorer List Viewer"
		Write-Host "`nYou can connect manually running " -NoNewline
		Write-Host "PS C:\>Connect-IPPSSession -UseRPSSession:$false -ShowBanner:$false"
		exit
	 }
}
```

Here another example to validate if you have the cmdlets required to Connecto to Microsoft Graph and Microsoft Exhange Online.

```powershell
function ValidateConnectsCmdlets
{
	$NotPassed = 0
	Write-Host "`n"
	if (-not (Get-Command -Name Connect-MgGraph -ErrorAction SilentlyContinue)) 	
	{
		Write-Host "Check connection to Microsoft Graph API..." -NoNewline
		Write-Host "`tFailed" -ForeGroundColor DarkRed
		NotPassed++
	}else
	{
		Write-Host "Check connection to Microsoft Graph API..." -NoNewline
		Write-Host "`tPassed" -ForeGroundColor Green
	}
	
	if (-not (Get-Command -Name Connect-ExchangeOnline -ErrorAction SilentlyContinue)) 	
	{
		Write-Host "Check connection to Microsoft Exchange..." -NoNewline
		Write-Host "`tFailed" -ForeGroundColor DarkRed
		NotPassed++
	}else
	{
		Write-Host "Check connection to Microsoft Exchange..." -NoNewline
		Write-Host "`tPassed" -ForeGroundColor Green
	}
	
	Start-Sleep -s 10
	
	if($NotPassed -gt 1)
	{
		Write-Host "`nYou don´t have the PowerShell module required to Connect to the services required."
		Write-Host "Please execute this script using :"
		Write-Host "`t* .\YourScript.ps2.ps1 -CheckDependencies"
		exit
	}
}
```
