# Function to create a CSV file used later as an input for a script

The following function creates a CSV file named `ManagerGroupsMatrix.csv` in a folder called `ConfigFiles`. This CSV serves as input for a script that identifies nested direct reports across multiple levels, which can be configured within the same CSV file. If the file does not already exist, the function initializes it with the required structure. The function contains two key sections: one defines the list of fields to be included in the CSV, and the other arranges these fields in a specific order.

```powershell
function CreateCSVFile
{
	if(-not (Test-Path -Path $PathFolder))
	{
		New-Item -ItemType Directory -Force -Path "$PSScriptRoot\ConfigFiles" | Out-Null
	}
	
	# Check if the CSV file already exists
        if (-Not (Test-Path $ConfigurationFile))
	{
		# Create a CSV structure
		$ManagerUPN = "YourManagerUserPrincipalName@yourdomain.com"
		$GroupOwner = "OtherGroupOwnerUserPrincipalName@yourdomain.com"
		$IncludeManager = "TRUE"
		$ManagerAsOwner = "FALSE"
		$NewGroup = "Set the name of your new group"
		$GroupDescription = "Set your group description"
		$GroupType = "Use 'security' or 'microsoft365'"
		[pscustomobject]$data = [ordered]@{
			ManagerUPN		= $ManagerUPN
			IncludeManager	= $IncludeManager #Include the manager in the same group or not
			ManagerAsOwner	= $ManagerAsOwner #Set manager as a group Owner
			GroupOwner		= $GroupOwner #Set a group Owner
			NewGroup		= $NewGroup
			GroupDescription= $GroupDescription
			GroupType		= $GroupType
			ExistingGroup	= $ExistingGroup
			RecursionDepth	= $RecursionDepth
		}
		# If file does not exist, create it with headers
		$data | Export-Csv -Path $ConfigurationFile -NoTypeInformation
		Write-Host "Created new CSV file: $ConfigurationFile"
    } else
	{
		# If file exists, append new data
		Write-Host "File is existing on path."
		exit
    }
}
```

<br><br>
