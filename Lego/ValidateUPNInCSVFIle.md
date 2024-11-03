# Function to validate email format in a column in a CSV file

This script reads the `ManagerUPN` and `GroupOwner` columns from a CSV file, validating each record to ensure that the UPN matches an email format. If any values do not conform to the email format, a counter increments for each error. If the counter is greater than 0 at the end, the script exits and displays the total number of errors.

```powershell
function ValidateUPNInCSVFIle
{
    # Regular expression to match email format
    $emailPattern = '^[\w-\.]+@([\w-]+\.)+[\w-]{2,4}$'
	$CountUPNError = 0
	
	Write-Host "`n########## Validating UPN format ##########`n" -ForeGroundColor DarkYellow
    # Iterate over each record in the CSV
    foreach ($record in $CSVFile) 
	{
        # Validate the ManagerUPN field
        if ($record.ManagerUPN -match $emailPattern)
		{
            Write-Host "ManagerUPN valid: $($record.ManagerUPN)"
        }else
		{
            Write-Host "Invalid or missing ManagerUPN: $($record.ManagerUPN)"
			$CountUPNError++
        }

        # Validate the GroupOwner field
        if ($record.GroupOwner -match $emailPattern)
		{
            Write-Host "GroupOwner valid: $($record.GroupOwner)"
        }elseif($record.GroupOwner -eq "")
		{
            Write-Host "Missing GroupOwner: Not Set"
        }else 
		{
            Write-Host "Invalid format GroupOwner: $($record.GroupOwner)"
			$CountUPNError++
        }
    }
	if($CountUPNError -gt 0)
	{
		Write-Host "`nTotal of UPN errors found : " -NoNewline
		Write-Host $CountUPNError -ForegroundColor Green
		Write-Host "Please review the file located at $ConfigurationFile and validate the UPNs added to the file."
		Write-Host "`n###########################################`n" -ForeGroundColor DarkYellow
		exit
	}
	Write-Host "`n###########################################`n" -ForeGroundColor DarkYellow
}
```
