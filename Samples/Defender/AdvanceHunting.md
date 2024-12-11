# Script that permit to run Advance Hunting queries

This script permit to run any Advance Hunting KQL query, by default creates a file called `Query.txt` that request activities from Copilot, nevertheless, the KQL can be changed on the mentioned fila and any kind of query can be added.

The following functions from the 'Lego' folder were utilized:
- [ScriptVariables](/Lego/ScriptVariables.md)
- [CheckIfElevated](/Lego/CheckIfElevated.md)
- [CheckPowerShellVersion](/Lego/CheckPowerShellVersion.md)
- [CheckRequiredModules](/Lego/CheckRequiredModules.md)
- [CreateConfigFile](/Lego/CreateConfigFile.md)
- [CreateNewEntraApp](/Lego/CreateNewEntraApp.md)
- [Build-Signature](/Lego/Build-Signature.md)
- [WriteToLogsAnalytics](/Lego/WriteToLogsAnalytics.md)

<details>
<summary>You can find the complete script here</summary>

```powershell
#this script is thought to get Copilot Activities through an Advanced Hunting query

param (
	[Parameter()] 
        [switch]$ExportToJSONFile,
	[Parameter()] 
        [switch]$CreateEntraApp
)

function ScriptVariables
{
	# Log Analytics table where the data is written to. Log Analytics will add an _CL to this name.
	$script:TableName = "CopilotActivities"
	$script:ConfigPath = $PSScriptRoot + "\ConfigFiles\Config.json"
	$script:QueryPath = $PSScriptRoot + "\ConfigFiles\Query.txt"
	$script:ExportFolderName = "ExportedData"
	$script:ExportPath = $PSScriptRoot + "\" + $ExportFolderName
	$script:GraphEndpoint = "https://graph.microsoft.com/v1.0/security/microsoft.graph.security.runHuntingQuery"
}

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
        @{Name="Microsoft.Graph.Authentication"; MinVersion="0.0"},
		@{Name="Microsoft.Graph.Applications"; MinVersion="0.0"},
		@{Name="Microsoft.Graph.Users"; MinVersion="0.0"},
		@{Name="Microsoft.Graph.Identity.DirectoryManagement"; MinVersion="0.0"}
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

function CreateCopilotQueryToFile 
{
    $Query = @"
CloudAppEvents 
| where Timestamp >= now(-180d)
| where Application contains 'Microsoft Copilot for Microsoft 365'
"@

    Write-Output "Writing query to $QueryPath"
    $Query | Out-File -FilePath $QueryPath -Encoding UTF8
    Write-Output "Query written successfully to $QueryPath"
}

function CreateConfigFile
{
	if(-Not (Test-Path $ConfigPath ))
	{
		Write-Host "Export data directory is missing, creating a new folder called ConfigFiles"
		New-Item -ItemType Directory -Force -Path "$PSScriptRoot\ConfigFiles" | Out-Null
	}
	
	if (-not (Test-Path -Path $ConfigPath))
    {
		$config = [ordered]@{
		ClientId = ""
		TenantId = ""
		ClientSecret = ""
		Buffer = "1000"
		WorkspaceID = ""
		WorkspacePrimaryKey = ""
		}
    }else
	{
		Write-Host "Configuration file is available under ConfigFiles folder"
		return
	}
	
	$config | ConvertTo-Json | Out-File "$ConfigPath"
    Write-Host "New config file was created under ConfigFile folder." -ForegroundColor Yellow
}

function CreateNewEntraApp
{
    cls
	Write-Host "'nYou will be prompted to add your Global Administrator credentials to login to Microsoft Entra and create a Microsoft Entra App..."
	Connect-MgGraph -Scopes "Application.ReadWrite.All", "AppRoleAssignment.ReadWrite.All", "Directory.ReadWrite.All", "User.ReadWrite.All" -NoWelcome

	if(-not (Test-Path -path $ConfigPath))
	{
		CreateConfigFile
	}
	
	$json = Get-Content -Raw -Path $ConfigPath
	[PSCustomObject]$config = ConvertFrom-Json -InputObject $json
	
    $appName = "Get Advanced Hunting"
    Get-MgApplication -ConsistencyLevel eventual -Count appCount -Filter "startsWith(DisplayName, '$appName')" | Out-Null
    if ($appCount -gt 0)
    {   
        Write-Host "'$appName' app already exists.`n"
		Exit
    }

    # app parameters and API permissions definition
    $params = @{
        DisplayName = $appName
        SignInAudience = "AzureADMyOrg"
        RequiredResourceAccess = @(
            @{
            # Microsoft Graph API ID
            ResourceAppId = "00000003-0000-0000-c000-000000000000"
            ResourceAccess = @(
                @{
                    # This is the default permission added every time that a MIcrosoft Entra App is created
                    # User.Read - Delegated
                    Id = "e1fe6dd8-ba31-4d61-89e7-88639da4683d"
                    Type = "Scope"
                },
                @{
                    # ThreatHunting.Read.All - Application
                    Id = "dd98c7f5-2d42-42d3-a0e4-633161547251"
                    Type = "Role"
                }
            )
        }
        )
    }
	
    # create application
    $app = New-MgApplication @params
    $appId = $app.Id

    # assign owner
    $userId = (Get-MgUser -UserId (Get-MgContext).Account).Id
    $params = @{
        "@odata.id" = "https://graph.microsoft.com/v1.0/directoryObjects/$userId"
    }
    New-MgApplicationOwnerByRef -ApplicationId $appId -BodyParameter $params
	
	# ask for client secret name
    $keyName = "Advanced Hunting App Secret key"

    # create client secret
    $passwordCred = @{
        displayName = $keyName
        endDateTime = (Get-Date).AddMonths(24)
     }
     
    $secret = Add-MgApplicationPassword -applicationId $appId -PasswordCredential $passwordCred

	$TenantID = (Get-MgContext).TenantId
	
    Write-Host "`nAzure application was created."
    Write-Host "App Name: $appName"
    Write-Host "App ID: $($app.AppId)"
	Write-Host "Tenant ID: $TenantID"
    Write-Host "Secret password: $($secret.SecretText)"
    Write-Host "`nPlease go to the Azure portal to manually grant admin consent:"
    Write-Host "https://portal.azure.com/#view/Microsoft_AAD_RegisteredApps/ApplicationMenuBlade/~/CallAnAPI/appId/$($app.AppId)`n" -ForegroundColor Cyan

    $config.TenantId = $TenantID
	$config.ClientId = $app.AppId
    $config.ClientSecret = $secret.SecretText
	
	$config | ConvertTo-Json | Out-File $ConfigPath
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
	$json = Get-Content -Raw -Path $ConfigPath
	[PSCustomObject]$config = ConvertFrom-Json -InputObject $json
	
	#$EncryptedKeys = $config.EncryptedKeys
	$WLA_CustomerID = $config.WorkspaceID
	$WLA_SharedKey = $config.WorkspacePrimaryKey
	
	<#if ($EncryptedKeys -eq "True")
	{
		$WLA_SharedKey = DecryptSharedKey $WLA_SharedKey
	}#>

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
	$TotalRows = $bodyJson.Count

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
        Write-Information -MessageData "$TotalRows rows written to Log Analytics workspace $uri" -InformationAction Continue
    }
}

function MainFunction
{
	$Config = Get-Content -Path $ConfigPath | ConvertFrom-Json
	
	if(-Not(Test-Path -Path $QueryPath))
	{
		CreateCopilotQueryToFile
	}
	$Query = Get-Content -Path $QueryPath -Raw
	# Extract configuration values
	$TenantId = $Config.TenantId
	$ClientId = $Config.ClientId
	$ClientSecret = $Config.ClientSecret
	$Buffer = $Config.Buffer
	$TokenEndpoint = "https://login.microsoftonline.com/$TenantId/oauth2/v2.0/token"
	
	# Step 1: Acquire Access Token using Secret Key for Microsoft Graph
	Write-Output "Acquiring Access Token for Microsoft Graph..."
	try {
		$TokenResponse = Invoke-RestMethod -Method Post -Uri $TokenEndpoint -ContentType "application/x-www-form-urlencoded" -Body @{
			client_id = $ClientId
			client_secret = $ClientSecret
			grant_type = "client_credentials"
			scope = "https://graph.microsoft.com/.default"
		}
		$AccessToken = $TokenResponse.access_token
		Write-Output "Access Token Acquired Successfully."
	} catch {
		Write-Output "Error Acquiring Access Token: $($_.Exception.Message)"
		throw $_
	}
	
	# Step 2: Query Execution using Microsoft Graph API with Pagination
	$Body = @{
		"Query" = $Query
	} | ConvertTo-Json -Depth 10

	$OutputFilePath = $PSScriptRoot + "\ConfigFiles\QueryResults.json"
	$Headers = @{
		"Authorization" = "Bearer $AccessToken"
		"Content-Type"  = "application/json"
	}
	
	$AllResults = @() # Array to store all results
	Write-Output "Executing query via Microsoft Graph API..."
	try {
		$NextLink = $GraphEndpoint
		do {
			# Make the request
			$Response = Invoke-RestMethod -Method Post -Uri $NextLink -Headers $Headers -Body $Body
			$Results = $Response.results

			# Append results to the main array
			$AllResults += $Results

			# Check for next page
			$NextLink = $Response.'@odata.nextLink'
			Write-Output "Fetched $($Results.Count) records. Total so far: $($AllResults.Count)."
		} while ($NextLink -ne $null)

		# Save all results to JSON file
		if($ExportToJSONFile)
		{
			
	if(-Not (Test-Path $ExportPath))
	{
		New-Item -ItemType Directory -Force -Path "$PSScriptRoot\$ExportFolderName" | Out-Null
	}
			$date = (Get-Date).ToString("yyyy-MM-dd HHmm")
			$OutputFilePath = $ExportPath + "\QueryResults - " + $date + ".json"
			$AllResults | ConvertTo-Json -Depth 10 | Out-File -FilePath $OutputFilePath -Encoding UTF8
		}else
		{
			WriteToLogsAnalytics -LogAnalyticsTableName $TableName -body $AllResults
		}		
		
		Write-Output "Query results saved to $OutputFilePath."
	} catch {
		Write-Output "Error Executing Query. HTTP Status Code: $($_.Exception.Response.StatusCode)"
		Write-Output "Reason Phrase: $($_.Exception.Response.ReasonPhrase)"
		Write-Output "Error Content: $($_.Exception.Response.Content.ReadAsStringAsync().Result)"
		throw $_
	}
	
}

ScriptVariables

if($CreateEntraApp)
{
	CheckPowerShellVersion
	CheckRequiredModules
	CreateNewEntraApp
	CreateCopilotQueryToFile
	exit
}

CheckPowerShellVersion
MainFunction
```

</details>

<br><br>
