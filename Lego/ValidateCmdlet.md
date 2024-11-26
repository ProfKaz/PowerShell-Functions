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
