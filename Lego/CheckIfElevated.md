# Function to check if you PowerShell script is running with Elevate Privileges

Some times when we execute some scripts we need to run PowerShell with administrator rights to accomplish activities like install a PowerShell module or create a task under task scheduler.

```powershell
function CheckIfElevated
{
    $IsElevated = ([Security.Principal.WindowsPrincipal] [Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
    if (!$IsElevated)
    {
        Write-Host "`nPlease start PowerShell as Administrator.`n" -ForegroundColor Yellow
        exit(1)
    }
}
```
