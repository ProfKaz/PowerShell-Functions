# Function to validate Group ID format in a column in a CSV file

This script reads the `ExistingGroup` column from a CSV file, validating each record to ensure that the Group ID matches with the right ID format. If any values do not conform to the Group ID format, a counter increments for each error. If the counter is greater than 0 at the end, the script exits and displays the total number of errors.

```powershell
function ValidateExistingGroupField
{
    # Regular expression to match GUID format
    $guidPattern = '^[a-fA-F0-9]{8}-[a-fA-F0-9]{4}-[a-fA-F0-9]{4}-[a-fA-F0-9]{4}-[a-fA-F0-9]{12}$'
	$CountGUIDError = 0
	
	Write-Host "`n########## Validating Group ID format ##########`n" -ForeGroundColor DarkYellow
    # Iterate over each record in the CSV
    foreach ($record in $CSVFile) 
	{
        # Validate the ExistingGroup field
        if ($record.ExistingGroup -match $guidPattern) 
		{
            Write-Host "ExistingGroup valid: $($record.ExistingGroup)"
        }elseif($record.ExistingGroup -eq "")
		{
            Write-Host "Missing ExistingGroup: Not Set"
        }else 
		{
            Write-Host "Invalid value set in ExistingGroup: $($record.ExistingGroup)"
			$CountGUIDError++
        }
	}
	if($CountGUIDError -gt 0)
	{
		Write-Host "`nTotal of Group ID errors found : " -NoNewline
		Write-Host $CountGUIDError -ForegroundColor Green
		Write-Host "Please review the file located at $ConfigurationFile and validate the Group IDs added to the file."
		Write-Host "`n#####################################################`n" -ForeGroundColor DarkYellow
		exit
	}
	Write-Host "`n#####################################################`n" -ForeGroundColor DarkYellow
}
```
<br><br>
