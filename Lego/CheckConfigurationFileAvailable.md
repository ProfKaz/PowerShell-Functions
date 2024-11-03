# Check if the CSV file used as an input exist

I use this simple script to check if the required file is available. If the file does not exist, it is created using the [CreateCSVFile](/Lego/CreateCSVFile.md) function. This check is implemented in the [NestedGroupsBasedOnManager script](/Samples/NestedGroupsBasedOnManager.md) in this way:

```powershell
$ConfigurationFile = "$PSScriptRoot\ConfigFiles\ManagerGroupsMatrix.csv"
CheckConfigurationFileAvailable
$CSVFile = Import-Csv -Path $ConfigurationFile

MainScript
```

```powershell
function CheckConfigurationFileAvailable
{
	# Check if the file exists
    if (-Not (Test-Path -Path $ConfigurationFile)) 
	{
		CreateCSVFile
		Write-Host "`nAn Empty CSV configuration file was created.`n"
		Start-Sleep -s 1
		Return
    }
}
```
<br><br>
