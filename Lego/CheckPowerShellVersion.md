# Function to check if you are running PowerShell v7 or higher

Some functions are using some capabilities that are available only if you are using PowerShell 7, through this function you can validate if you are using that version.

```powershell
function CheckPowerShellVersion
{
    # Check PowerShell version
    Write-Host "`nChecking PowerShell version... " -NoNewline
    if ($Host.Version.Major -gt 5)
    {
        Write-Host "`t`t`t`tPassed!" -ForegroundColor Green
    }
    else
    {
        Write-Host "Failed" -ForegroundColor Red
        Write-Host "`tCurrent version is $($Host.Version). PowerShell version 7 or newer is required."
        exit(1)
    }
}
```
<br><br>
