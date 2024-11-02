# Function to create a Task under a new folder on Task Scheduler

This function creates a scheduled task named **RunMyScript** in Task Scheduler, organizing tasks by creating a folder called **MyScripts** if it doesn’t already exist. The task is configured to run every 30 days, and it will be skipped if a task with the same name already exists.

> [!IMPORTANT]
> To execute this function, PowerShell must be run with Administrator rights. Use the [CheckIfElevated function](CheckIfElevated.md) to ensure the correct permissions are in place.

```powershell
function CreateTaskSchedulerTask
{
	# Default folder for Microsoft Entra tasks
    $MyScriptFolder = "MyScripts"
	  $taskFolder = "\"+$MyScriptFolder+"\"
	
	# Nested Groups Based On Manager script
    $taskName = "RunMyScript"
	
	# Task execution
    $validDays = 30

    # calculate date
    $dt = Get-Date 
    $reminder = $dt.Day % $validDays
    $dt = $dt.AddDays(-$reminder)
    $startTime = [datetime]::new($dt.Year, $dt.Month, $dt.Day, $dt.Hour, $dt.Minute, 0)

    #create task
    $trigger = New-ScheduledTaskTrigger -Once -At $startTime -RepetitionInterval (New-TimeSpan -Days $validDays)
    $action = New-ScheduledTaskAction -Execute "`"$PSHOME\pwsh.exe`"" -Argument ".\MPARR-MicrosoftEntraRoles.ps1" -WorkingDirectory $PSScriptRoot
    $settings = New-ScheduledTaskSettingsSet -StartWhenAvailable -DontStopOnIdleEnd -AllowStartIfOnBatteries `
         -MultipleInstances IgnoreNew -ExecutionTimeLimit (New-TimeSpan -Hours 1)

    if (Get-ScheduledTask -TaskName $taskName -TaskPath $taskFolder -ErrorAction SilentlyContinue) 
    {
        Write-Host "`nScheduled task named '$taskName' already exists.`n" -ForegroundColor Yellow
		exit
    }
    else 
    {
        Register-ScheduledTask -TaskName $taskName -Action $action -Trigger $trigger -Settings $settings `
        -RunLevel Highest -TaskPath $taskFolder -ErrorAction Stop | Out-Null
        Write-Host "`nScheduled task named '$taskName' was created.`nFor security reasons you have to specify run as account manually.`n`n" -ForegroundColor Yellow
    }
}
```
<br><br>
