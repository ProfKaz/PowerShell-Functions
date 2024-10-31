# Function to check if you PowerShell have all the PowerShell modules required

Every time we work with different scripts, we need to load specific PowerShell modules to access the necessary cmdlets. With this in mind, it's essential to ensure that anyone running the script has the correct components installed.

```powershell
function CheckRequiredModules 
{
    # Check PowerShell modules
    Write-Host "Checking PowerShell modules..."
	
	$requiredModules = @(
			@{Name="MicrosoftGraph"; MinVersion="0.0"},
			@{Name="Microsoft.Graph.Authentication"; MinVersion="0.0"},
			@{Name="Microsoft.Graph.Users"; MinVersion="0.0"},
			@{Name="Microsoft.Graph.Groups"; MinVersion="0.0"}
			)
	
	if($CreateEntraApp)
	{
		$requiredModules += @(@{Name="Microsoft.Graph.Applications"; MinVersion="0.0"}) 
	}

    $modulesToInstall = @()
    foreach ($module in $requiredModules)
    {
        Write-Host "`t$($module.Name) - " -NoNewline
        $installedVersions = Get-Module -ListAvailable $module.Name
        if ($installedVersions)
        {
            if ($installedVersions[0].Version -lt [version]$module.MinVersion)
            {
                Write-Host "`t`t`tNew version required" -ForegroundColor Red
                $modulesToInstall += $module.Name
            }
            else 
            {
                Write-Host "`t`t`tInstalled" -ForegroundColor Green
            }
        }
        else
        {
            Write-Host "`t`t`tNot installed" -ForegroundColor Red
            $modulesToInstall += $module.Name
        }
    }

    if ($modulesToInstall.Count -gt 0)
    {
        CheckIfElevated
		$choices  = '&Yes', '&No'

        $decision = $Host.UI.PromptForChoice("", "Misisng required modules. Proceed with installation?", $choices, 0)
        if ($decision -eq 0) 
        {
            Write-Host "Installing modules..."
            foreach ($module in $modulesToInstall)
            {
                Write-Host "`t$module"
				Install-Module $module -ErrorAction Stop
                
            }
            Write-Host "`nModules installed. Please start the script again."
            exit(0)
        } 
        else 
        {
            Write-Host "`nExiting setup. Please install required modules and re-run the setup."
            exit(1)
        }
    }
}
```
