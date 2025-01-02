# Script solution to get Microsoft Purview Data Loss Prevention configuration

<p align="center">
<img src="[https://github.com/user-attachments/assets/3002d585-f35e-4c08-bb06-a4998e080ff4](https://github.com/user-attachments/assets/0596d3c0-5d83-4e28-bdbd-ad9efa3dddd8)" width="600"></p>
<p align="center">Power BI report based on data collected in Logs Analytics</p>
<br>

I developed this script to simplify the way to delivery a Microsoft Purview Information Protection Assessment. This script permit to collect Sensitivity Labels and Labes Policies, the information is exported automatically in Json format, nevertheless can be exported to CSV or Logs Analytics.

The following functions from the 'Lego' folder were utilized:
- [CheckIfElevated](/Lego/CheckIfElevated.md)
- [CheckPowerShellVersion](/Lego/CheckPowerShellVersion.md)
- [CheckRequiredModules](/Lego/CheckRequiredModules.md)

Other modules used on this script will be shared soon, nevertheless, next you can find all the code.

<details>
<summary>You can find the complete script here</summary>

Additional helper functions are available, such as `Help`, which provides guidance on using the complete script.

```powershell
<#PSScriptInfo

.VERSION 2.0.5

.GUID 883af802-166c-4708-f4d1-352686c02f01

.AUTHOR 
https://www.linkedin.com/in/profesorkaz/; Sebastian Zamorano

.COMPANYNAME 
Microsoft Purview Advanced Rich Reports

.TAGS 
#Microsoft365 #M365 #MPARR #MicrosoftPurview #ActivityExplorer

.PROJECTURI 
https://aka.ms/MPARR-YouTube 

.RELEASENOTES
The MIT License (MIT)
Copyright (c) 2015 Microsoft Corporation
Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:
The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.
THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

#>

<# 

.DESCRIPTION 
This script permit to export Information Protection configuration

#>

<#
HISTORY
	Script      : MSPurviewDLPCollector.ps1
	Author      : S. Zamorano
	Version     : 2.0.5
	Description : Export DLP policies and rules to CSV or Json format.
	17-04-2024		S. Zamorano		- Public release
	12-08-2024		S. Zamorano		- Version 2 Public release
	16-08-2024		S. Zamorano		- Conditions field added to the query
	19-08-2024		S. Zamorano		- Added field to identify users scope for policies
	20-08-2024		S. Zamorano		- Fix export name
#>

[CmdletBinding(DefaultParameterSetName = "None")]
param(
	[string]$DLPRuleTableName = "MSPurviewDLPRulesDetailed",
	[string]$DLPPoliciesTableName = "MSPurviewDLPPoliciesDetailed",
	[Parameter()] 
        [switch]$Help,
	[Parameter()] 
        [switch]$ExportToCsv,
	[Parameter()] 
        [switch]$ExportToLogsAnalytics,
	[Parameter()] 
        [switch]$OnlyRules,
	[Parameter()] 
        [switch]$OnlyPolicies
)

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

function CheckIfElevated
{
    $IsElevated = ([Security.Principal.WindowsPrincipal] [Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
    if (!$IsElevated)
    {
        Write-Host "`nPlease start PowerShell as Administrator.`n" -ForegroundColor Yellow
        exit(1)
    }
}

function CheckRequiredModules 
{
    # Check PowerShell modules
    Write-Host "Checking PowerShell modules..."
    $requiredModules = @(
        @{Name="ExchangeOnlineManagement"; MinVersion="0.0"}
        )

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

function CheckPrerequisites
{
    CheckPowerShellVersion
	CheckRequiredModules
}

function connect2service
{
	Write-Host "`nAuthentication is required, please check your browser" -ForegroundColor DarkYellow
	Connect-IPPSSession -UseRPSSession:$false -ShowBanner:$false
}

function DecryptSharedKey 
{
    param(
        [string] $encryptedKey
    )

    try {
        $secureKey = $encryptedKey | ConvertTo-SecureString -ErrorAction Stop  
    }
    catch {
        Write-Error "Workspace key: $($_.Exception.Message)"
        exit(1)
    }
    $BSTR =  [System.Runtime.InteropServices.Marshal]::SecureStringToBSTR($secureKey)
    $plainKey = [System.Runtime.InteropServices.Marshal]::PtrToStringAuto($BSTR)
    $plainKey
}

function Build-Signature ($customerId, $sharedKey, $date, $contentLength, $method, $contentType, $resource) 
{
    # ---------------------------------------------------------------   
    #    Name           : Build-Signature
    #    Value          : Creates the authorization signature used in the REST API call to Log Analytics
    # ---------------------------------------------------------------

	#Original function to Logs Analytics
    $xHeaders = "x-ms-date:" + $date
    $stringToHash = $method + "`n" + $contentLength + "`n" + $contentType + "`n" + $xHeaders + "`n" + $resource

    $bytesToHash = [Text.Encoding]::UTF8.GetBytes($stringToHash)
    $keyBytes = [Convert]::FromBase64String($sharedKey)

    $sha256 = New-Object System.Security.Cryptography.HMACSHA256
    $sha256.Key = $keyBytes
    $calculatedHash = $sha256.ComputeHash($bytesToHash)
    $encodedHash = [Convert]::ToBase64String($calculatedHash)
    $authorization = 'SharedKey {0}:{1}' -f $customerId,$encodedHash
    return $authorization
}

function WriteToLogsAnalytics($body, $LogAnalyticsTableName) 
{
    # ---------------------------------------------------------------   
    #    Name           : Post-LogAnalyticsData
    #    Value          : Writes the data to Log Analytics using a REST API
    #    Input          : 1) PSObject with the data
    #                     2) Table name in Log Analytics
    #    Return         : None
    # ---------------------------------------------------------------
    
	#Read configuration file
	$CONFIGFILE = "$PSScriptRoot\ConfigFiles\MSPurviewDLPConfiguration.json"
	$json = Get-Content -Raw -Path $CONFIGFILE
	[PSCustomObject]$config = ConvertFrom-Json -InputObject $json
	
	$EncryptedKeys = $config.EncryptedKeys
	$WLA_CustomerID = $config.Workspace_ID
	$WLA_SharedKey = $config.WorkspacePrimaryKey
	if ($EncryptedKeys -eq "True")
	{
		$WLA_SharedKey = DecryptSharedKey $WLA_SharedKey
	}

	# Your Log Analytics workspace ID
	$LogAnalyticsWorkspaceId = $WLA_CustomerID

	# Use either the primary or the secondary Connected Sources client authentication key   
	$LogAnalyticsPrimaryKey = $WLA_SharedKey
	
	#Step 0: sanity checks
    if($body -isnot [array]) {return}
    if($body.Count -eq 0) {return}
	
	#Step 1: convert the body.ResultData to JSON
	$json_array = @()
	$parse_array = @()
	$parse_array = $body #| ConvertFrom-Json
	foreach($item in $parse_array) 
	{
		$json_array += $item
	}
	$json = $json_array | ConvertTo-Json -Depth 12
	
	#Step 2: convert the PSObject to JSON
	$bodyJson = $json
	#Step 2.5: sanity checks
	if($bodyJson.Count -eq 0) {return}

    #Step 3: get the UTF8 bytestream for the JSON
    $bodyJsonUTF8 = ([System.Text.Encoding]::UTF8.GetBytes($bodyJson))
	
	#Step 4: build the signature        
    $method = "POST"
    $contentType = "application/json"
    $resource = "/api/logs"
    $rfc1123date = [DateTime]::UtcNow.ToString("r")
    $contentLength = $bodyJsonUTF8.Length    
    $signature = Build-Signature -customerId $LogAnalyticsWorkspaceId -sharedKey $LogAnalyticsPrimaryKey -date $rfc1123date -contentLength $contentLength -method $method -contentType $contentType -resource $resource
    
    #Step 5: create the header
    $headers = @{
        "Authorization" = $signature;
        "Log-Type" = $LogAnalyticsTableName;
        "x-ms-date" = $rfc1123date;
    };

    #Step 6: REST API call
    $uri = 'https://' + $LogAnalyticsWorkspaceId + ".ods.opinsights.azure.com" + $resource + "?api-version=2016-04-01"
    $response = Invoke-WebRequest -Uri $uri -Method Post -Headers $headers -ContentType $contentType -Body $bodyJsonUTF8 -UseBasicParsing

    if ($Response.StatusCode -eq 200) {   
        $rows = $bodyJsonUTF8.Count
        Write-Information -MessageData "$rows rows written to Log Analytics workspace $uri" -InformationAction Continue
    }
}

function WriteToJson($results, $ExportFolder, $QueryType, $date)
{
	$json_array = @() 
	$parse_array = @()
	$parse_array = $results 
	foreach($item in $parse_array) 
	{
		$json_array += $item
	}
	$json = $json_array | ConvertTo-Json -Depth 6
	$FileName = "Microsoft Purview DLP export - "+"$QueryType"+" - "+"$date"+".Json"
	$pathJson = $PSScriptRoot+"\"+$ExportFolder+"\"+$FileName
	$path = $pathJson
	$json | Add-Content -Path $path
	Write-Host "`nData exported to... :" -NoNewLine
	Write-Host $pathJson -ForeGroundColor Cyan
	Write-Host "`n----------------------------------------------------------------------------------------`n`n" -ForeGroundColor DarkBlue
}

function WriteToCsv($results, $ExportFolder, $QueryType, $date)
{
	$parse_array = @()
	$nextpages_array = @()
	$TotalResults = @()
	$TotalResults = $results
	foreach($item in $TotalResults)
	{
		$FileName = "Microsoft Purview DLP export - "+"$QueryType"+" - "+"$date"+".Csv"
		$pathCsv = $PSScriptRoot+"\"+$ExportFolder+"\"+$FileName
		$path = $pathCsv
		$parse_array = $item
		$values = $parse_array[0].psobject.properties.name
		$parse_array | Export-Csv -Path $path -NTI -Force -Append | Out-Null
	}
	Write-Host "Total results $($results.count)"
	Write-Host "`nData exported to..." -NoNewline
	Write-Host "`n$pathCsv" -ForeGroundColor Cyan
	Write-Host "`n----------------------------------------------------------------------------------------`n`n" -ForeGroundColor DarkBlue
}

function MSPuviewIPCollectorHelp
{
	cls
	Write-Host "`n"
	Write-Host "################################################################################" -ForegroundColor Green
	Write-Host "`n How to use this script `n" -ForegroundColor Green
	Write-Host "################################################################################" -ForegroundColor Green
	Write-Host "`nDescription: " -ForegroundColor Blue -NoNewLine
	Write-Host "This menu"
	Write-Host ".\MSPurviewDLPCollector.ps1 -Help" -ForeGroundColor DarkYellow
	Write-Host "`n`nDescription: " -ForegroundColor Blue -NoNewLine
	Write-Host "Using only the script by default, you'll be able to get your DLP Rules and Policies in Json format."
	Write-Host ".\MSPurviewDLPCollector.ps1" -ForeGroundColor DarkYellow
	Write-Host "`n`nDescription: " -ForegroundColor Blue -NoNewLine
	Write-Host "Using the attribute '-OnlyRules' you will be able only to export DLP information"
	Write-Host ".\MSPurviewDLPCollector.ps1 -OnlyRules" -ForeGroundColor DarkYellow
	Write-Host "`n`nDescription: " -ForegroundColor Blue -NoNewLine
	Write-Host "Using the attribute '-OnlyPolicies' you will be able only to export DLP Policies information"
	Write-Host ".\MSPurviewDLPCollector.ps1 -OnlyPolicies" -ForeGroundColor DarkYellow
	Write-Host "`n`nDescription: " -ForegroundColor Blue -NoNewLine
	Write-Host "Using the attribute '-ExportToLogsAnalytics' you will be able only to export all the data to a Logs Analytics workspace"
	Write-Host ".\MSPurviewDLPCollector.ps1 -ExportToLogsAnalytics" -ForeGroundColor DarkYellow
	Write-Host "`n`nDescription: " -ForegroundColor Blue -NoNewLine
	Write-Host "If you are not comfortable working with JSON format, you can use the attribute '-ExportToCsv' to export the data in CSV format."
	Write-Host ".\MSPurviewDLPCollector.ps1 -ExportToCsv" -ForeGroundColor DarkYellow
	Write-Host "`n`nDescription: " -ForegroundColor Blue -NoNewLine
	Write-Host "You can combine different attributes available in the script to customize its functionality. For example:"
	Write-Host ".\MSPurviewDLPCollector.ps1 -OnlyRules -ExportToLogsAnalytics" -ForeGroundColor DarkYellow
	Write-Host "`n"
	Write-Host "### You can now proceed using any of the options listed in the Help menu. ###" -ForegroundColor Green
	Write-Host "`n"
	return
}

function GetDataLossPreventionData($ExportFormat, $ExportFolder, $ExportOption)
{
	Write-Host "`nExecuting Get cmdlet for your selection..." -ForeGroundColor Blue
	
	$date = (Get-Date).ToString("yyyy-MM-dd HHmm")
	$ExportExtension = $ExportFormat
	if($ExportFormat -eq "LA")
	{
		$ExportExtension="Json"
	}
	if($ExportOption -eq "All")
	{
		#Request DLP Rules
		$results = New-Object PSObject
		$TotalResults = @()
		$Query = "DLPRules"
		$results = Get-DlpComplianceRule
		$TotalResults += $results
		if($results.TotalResultCount -eq "0")
			{
				Write-Host "The previous combination does not return any values."
				Write-Host "Exiting...`n"
			}else
			{
				Write-Host "`nCollecting data..." -ForegroundColor DarkBlue -NoNewLine
				Write-Host $TotalResults.Count -ForegroundColor Blue -NoNewLine
				Write-Host " records returned"
				#Run the below steps in loop until all results are fetched

				if($ExportFormat -eq "Csv")
				{
					$CSVresults = $TotalResults
					WriteToCsv -results $CSVresults -ExportFolder $ExportFolder -QueryType $Query -date $date
				}elseif($ExportFormat -eq "LA")
				{
					WriteToLogsAnalytics -LogAnalyticsTableName $DLPRuleTableName -body $TotalResults
				}else
				{
					WriteToJson -results $TotalResults -ExportFolder $ExportFolder -QueryType $Query -date $date
				}
			}
		#Request DLP policies
		$results = New-Object PSObject
		$TotalResults = @()
		$Query = "DLPPolicies"
		$results = Get-DlpCompliancePolicy
		$TotalResults += $results
		if($results.TotalResultCount -eq "0")
			{
				Write-Host "The previous combination does not return any values."
				Write-Host "Exiting...`n"
			}else
			{
				Write-Host "`nCollecting data..." -ForegroundColor DarkBlue -NoNewLine
				Write-Host $TotalResults.Count -ForegroundColor Blue -NoNewLine
				Write-Host " records returned"
				#Run the below steps in loop until all results are fetched

				if($ExportFormat -eq "Csv")
				{
					$CSVresults = $TotalResults
					WriteToCsv -results $CSVresults -ExportFolder $ExportFolder -QueryType $Query -date $date
				}elseif($ExportFormat -eq "LA")
				{
					WriteToLogsAnalytics -LogAnalyticsTableName $DLPPoliciesTableName -body $TotalResults
				}else
				{
					WriteToJson -results $TotalResults -ExportFolder $ExportFolder -QueryType $Query -date $date
				}
			}
	}elseif($ExportOption -eq "OnlyRules")
	{
		$results = New-Object PSObject
		$TotalResults = @()
		$Query = "DLPRules"
		$results = Get-DlpComplianceRule
		$TotalResults += $results
		if($results.TotalResultCount -eq "0")
			{
				Write-Host "The previous combination does not return any values."
				Write-Host "Exiting...`n"
			}else
			{
				Write-Host "`nCollecting data..." -ForegroundColor DarkBlue -NoNewLine
				Write-Host $TotalResults.count -ForegroundColor Blue -NoNewLine
				Write-Host " records returned"
				#Run the below steps in loop until all results are fetched

				if($ExportFormat -eq "Csv")
				{
					$CSVresults = $TotalResults
					WriteToCsv -results $CSVresults -ExportFolder $ExportFolder -QueryType $Query -date $date
				}elseif($ExportFormat -eq "LA")
				{
					WriteToLogsAnalytics -LogAnalyticsTableName $DLPRuleTableName -body $TotalResults
				}else
				{
					WriteToJson -results $TotalResults -ExportFolder $ExportFolder -QueryType $Query -date $date
				}
			}
	}elseif($ExportOption -eq "OnlyPolicies")
	{
		$results = New-Object PSObject
		$TotalResults = @()
		$Query = "DLPPolicies"
		$results = Get-DlpCompliancePolicy
		$TotalResults += $results
		if($results.TotalResultCount -eq "0")
			{
				Write-Host "The previous combination does not return any values."
				Write-Host "Exiting...`n"
			}else
			{
				Write-Host "`nCollecting data..." -ForegroundColor DarkBlue -NoNewLine
				Write-Host $results.TotalResultCount -ForegroundColor Blue -NoNewLine
				Write-Host " records returned"
				#Run the below steps in loop until all results are fetched

				if($ExportFormat -eq "Csv")
				{
					$CSVresults = $TotalResults
					WriteToCsv -results $CSVresults -ExportFolder $ExportFolder -QueryType $Query -date $date
				}elseif($ExportFormat -eq "LA")
				{
					WriteToLogsAnalytics -LogAnalyticsTableName $DLPPoliciesTableName -body $TotalResults
				}else
				{
					WriteToJson -results $TotalRFesults -ExportFolder $ExportFolder -QueryType $Query -date $date
				}
			}
	}
}

function MainFunction
{
	#Welcome header
	cls
	Clear-Host
	
	Write-Host "`n`n----------------------------------------------------------------------------------------"
	Write-Host "`nWelcome to Data Loss Prevention Export script!" -ForegroundColor Green
	Write-Host "This script will permit to collect data from DLP Rules and Policies related"
	Write-Host "`n----------------------------------------------------------------------------------------" 
	
	
	#Initiate variables
	
	$ExportOption = "All"
		
	##List only DLP
	if($OnlyRules)
	{
		$ExportOption = "OnlyRules"
	}
	if($OnlyPolicies)
	{
		$ExportOption = "OnlyPolicies"
	}
	
	##Export format
	$ExportFormat = "Json"
	if($ExportToCsv)
	{
		$ExportFormat = "Csv"
	}
	if($ExportToLogsAnalytics)
	{
		$ExportFormat = "LA"
		$LogsAnalyticsConfigurationFile = "$PSScriptRoot\ConfigFiles\MSPurviewDLPConfiguration.json"
		if(-not (Test-Path -Path $LogsAnalyticsConfigurationFile))
		{
			Write-Host "`nConfiguration file is not present" -ForegroundColor DarkYellow
			Write-Host "Please download the configuration file from http://activityexplorer.kaznets.com and save inside of the ConfigFiles folder.`n"
			Write-Host "Press any key to continue..."
			$key = ([System.Console]::ReadKey($true))
			exit
		}	
	}
	
	##Export folder Name
	$ExportFolderName = "ExportedData"
	$ExportPath = "$PSScriptRoot\$ExportFolderName"
	if(-Not (Test-Path $ExportPath))
	{
		New-Item -ItemType Directory -Force -Path "$PSScriptRoot\$ExportFolderName" | Out-Null
		$StatusFolder = "Created"
	}else
	{
		$StatusFolder = "Available"
	}
	
	##Show variables set
	Write-Host "Export format set to`t`t`t:" -NoNewline
	Write-Host "`t$ExportFormat" -ForegroundColor Green
	Write-Host "Export folder set to`t`t`t:" -NoNewline
	Write-Host "`t$ExportFolderName ($StatusFolder)" -ForegroundColor Green
	Write-Host "Export Option selected`t`t`t:" -NoNewline
	Write-Host "`t$ExportOption" -ForegroundColor Green
	if($ExportToLogsAnalytics)
	{
		if($OnlyRules)
		{
			Write-Host "Table name for DLP Rules`t:" -NoNewline
			Write-Host "`t$DLPRuleTableName" -ForegroundColor Green
		}elseif($OnlyPolicies)
		{
			Write-Host "Table name for Policies DLP`t`t:" -NoNewline
			Write-Host "`t$DLPPoliciesTableName" -ForegroundColor Green
		}else
		{
			Write-Host "Table name for DLP Rules`t:" -NoNewline
			Write-Host "`t$DLPRuleTableName" -ForegroundColor Green
			Write-Host "Table name for Policies DLP`t`t:" -NoNewline
			Write-Host "`t$DLPPoliciesTableName" -ForegroundColor Green
		}
	}
	Write-Host "`n`nYou will be prompted for your credentials, remember that you need Compliance Administrator role"
	Write-Host "Press any key to continue..."
    $key = ([System.Console]::ReadKey($true))
	connect2service
	
	Write-Host "Calling script..."
	
	#Call function to export data from Activity Explorer
	GetDataLossPreventionData -ExportFormat $ExportFormat -ExportFolder $ExportFolderName -ExportOption $ExportOption
}

if($Help)
{
	MSPuviewIPCollectorHelp
	exit
}

CheckPrerequisites
MainFunction
```

</details>
<br><br>
