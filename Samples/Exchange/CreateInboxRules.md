# Script to create an inbox folder and set an inboox rule by subject or sender

This script allows you to create a new folder under a user's inbox if it doesn't already exist. Users can be loaded either from a CSV file or directly through the Microsoft Graph API. The script can also be automated by creating a Microsoft Entra Application with the necessary permissions pre-configured.

In its current version, the script utilizes the Exchange Online PowerShell module. It requires the Exchange Administrator role to grant the necessary permissions for creating folders and inbox rules.

```powershell
# Function to create a mail folder under inbox and then an Inbox rule to send certain emails to that folder

[CmdletBinding(DefaultParameterSetName = "None")]
param(
	[Parameter()] 
        [switch]$CreateEntraApp,
	[Parameter()] 
        [switch]$CreateConfigurationFile,
	[Parameter()] 
        [switch]$ChangeInput,
	[Parameter()] 
        [switch]$CheckDependencies
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
		@{Name="ExchangeOnlineManagement"; MinVersion="0.0"},
        @{Name="Microsoft.Graph.Mail"; MinVersion="0.0"},
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

function UnHashCredentials
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

function CreateConfigFile
{
	if(-Not (Test-Path $Configfile ))
	{
		Write-Host "Export data directory is missing, creating a new folder called ConfigFiles"
		New-Item -ItemType Directory -Force -Path "$PSScriptRoot\ConfigFiles" | Out-Null
	}
	
	if (-not (Test-Path -Path $configfile))
    {
		$config = [ordered]@{
		EncryptedKeys = "False"
		AppClientID = ""
		TenantGUID = ""
		CertificateThumb = ""
		TenantDomain = ""
		OnmicrosoftTenant = ""
		InputMethod = "CSV"
		RuleType = "Subject"
		RuleValue = "[E-Migrator]"
		}
    }else
	{
		Write-Host "Configuration file is available under ConfigFiles folder"
		return
	}
	
	$config | ConvertTo-Json | Out-File "$configfile"
    Write-Host "New config file was created under ConfigFile folder." -ForegroundColor Yellow
}

#Creates the CSV file used as main an unique input to create groups
function CreateCSVFile
{	
	if(-not (Test-Path -Path $PathFolder))
	{
		New-Item -ItemType Directory -Force -Path "$PSScriptRoot\ConfigFiles" | Out-Null
	}
	
	# Check if the CSV file already exists
    if (-Not (Test-Path $csvFilePath))
	{
		# Create a CSV structure
		$UPN = "UserPrincipalName@yourdomain.com"
		$data = @{
			AccountUPN		= $UPN
		}
		# If file does not exist, create it with headers
		$data | Export-Csv -Path $csvFilePath -NoTypeInformation
		Write-Host "Created new CSV file: $csvFilePath"
		Write-Host "`nPlease complete the file with the right UPNs.`n" -ForegroundColor Blue
		Write-host "You can change the input method to use Microsoft Graph API instead of CSV file executing:"
		Write-host ".\ResolveMailAccount.ps1 -ChangeInput `n" -ForeGroundColor Green
		exit
    } else
	{
		# If file exists, append new data
		Write-Host "File is existing on path."
		return
    }
}

function CheckCertificateInstalled($thumbprint)
{
	$var = "False"
	$certificates = @(Get-ChildItem Cert:\CurrentUser\My -SSLServerAuthentication | Select-Object Thumbprint) 
	#$thumbprint -in $certificates
	foreach($certificate in $certificates)
	{
		if($thumbprint -in $certificate.Thumbprint)
		{
			$var = "True"
		}
	}
	if($var -eq "True")
	{
		Write-Host "Certificate validation..." -NoNewLine
		Write-Host "`t`tPassed!" -ForegroundColor Green
		return $var
	}else
	{
		Write-Host "`nCertificate installed on this machine is missing!!!" -ForeGroundColor Yellow
		Write-Host "To execute this script unattended a certificate needs to be installed, the same used under Microsoft Entra App"
		Start-Sleep -s 1
		return $var
	}
}

function CreateNewEntraApp
{
	$appName = "E-Migrator Mail resolver"
	
	if (Get-MgContext) 
	{
        Write-Host "Disconnecting from previous session opened..."
		disconnect-MgGraph
    }
	
	Write-Host "Connecting to Microsoft Graph API"
	Connect-MgGraph -Scopes "Application.ReadWrite.All", "AppRoleAssignment.ReadWrite.All", "Directory.ReadWrite.All", "User.ReadWrite.All", "Domain.Read.All" -NoWelcome

	if(-not (Test-Path -path $Configfile))
	{
		CreateConfigFile
	}
	
	$json = Get-Content -Raw -Path $Configfile
	[PSCustomObject]$config = ConvertFrom-Json -InputObject $json
	
    Get-MgApplication -ConsistencyLevel eventual -Count appCount -Filter "startsWith(DisplayName, '$appName')" | Out-Null
    if ($appCount -gt 0)
    {   
        cls
		Write-Host "`n`n'$appName' app already exists.`n"
		Write-Host "You can run now the script as:" -NoNewline
		Write-Host "`t.\ResolveMailAccount.ps1`n`n" -ForeGroundColor Green
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
						# This permission is required to create the new folder on Inbox and create then the inbox rule
						# Mail.ReadWrite - Application
						Id = "e2a3a72e-5f79-4c64-b1b1-878b674786c9"
						Type = "Role"
					},
					@{
						# This permission permit get a list of EXO licensed users
						# User.Read.All - Application
						Id = "df021288-bdef-4463-88db-98f22de89214"
						Type = "Role"
					}
				)
			},
			@{
            # Office 365 Exchange Online API
            ResourceAppId = "00000002-0000-0ff1-ce00-000000000000"
            ResourceAccess = @(
					@{
						# This permission is required to create the new folder on Inbox and create then the inbox rule
						# Exchange.ManageAsApp - Application
						Id = "dc50a0fb-09a3-484d-be87-e023b12c6440"
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

    # ask for certificate name
    $certName = "$appName"+" Certificate"

    # certificate life
    $validMonths = 24

    # create key
    $cert = New-SelfSignedCertificate -DnsName $certName -CertStoreLocation "cert:\CurrentUser\My" -NotAfter (Get-Date).AddMonths($validMonths)
    $certBase64 = [System.Convert]::ToBase64String($cert.RawData)
    $keyCredential = @{
        type = "AsymmetricX509Cert"
        usage = "Verify"
        key = [System.Text.Encoding]::ASCII.GetBytes($certBase64)
    }
    while (-not (Get-MgApplication -ApplicationId $appId -ErrorAction SilentlyContinue)) 
    {
        Write-Host "Waiting while app is being created..."
        Start-Sleep -Seconds 5
    }
    Update-MgApplication -ApplicationId $appId -KeyCredentials $keyCredential -ErrorAction Stop
	$TenantID = (Get-MgContext).TenantId
	
	#Get main domains
	$Domains = Get-MgDomain -All
	$OnMicrosoftDomain = $Domains | Where-Object { $_.isInitial -eq $true }
	$PrimaryDomain = $Domains | Where-Object { $_.IsDefault -eq $true }

    Write-Host "`nAzure application was created."
    Write-Host "App Name: $appName"
    Write-Host "App ID: $($app.AppId)"
	Write-Host "Tenant ID: $TenantID"
    Write-Host "Certificate thumbprint: $($cert.Thumbprint)"
	Write-Host "Tenant default domain: $($PrimaryDomain.Id)"
	Write-Host "Tenant onmicrosoft domain: $($OnMicrosoftDomain.Id)"

    Write-Host "`nPlease go to the Azure portal to manually grant admin consent:"
    Write-Host "https://portal.azure.com/#view/Microsoft_AAD_RegisteredApps/ApplicationMenuBlade/~/CallAnAPI/appId/$($app.AppId)`n" -ForegroundColor Cyan

    $config.TenantGUID = $TenantID
	$config.AppClientID = $app.AppId
    $config.CertificateThumb = $cert.Thumbprint
	$config.TenantDomain = $PrimaryDomain.Id
	$config.OnmicrosoftTenant = $OnMicrosoftDomain.Id
	
	$config | ConvertTo-Json | Out-File $Configfile

    Remove-Variable cert
    Remove-Variable certBase64
}

function Connect2MicrosoftGraphService
{			
	$json = Get-Content -Raw -Path $ConfigFile
	[PSCustomObject]$config = ConvertFrom-Json -InputObject $json
	
	$EncryptedKeys = $config.EncryptedKeys
	$AppClientID = $config.AppClientID
	$CertificateThumb = $config.CertificateThumb
	$TenantGUID = $config.TenantGUID

	$status = CheckCertificateInstalled -thumbprint $CertificateThumb
	
	if($status -eq "True")
	{
		Connect-MgGraph -CertificateThumbPrint $CertificateThumb -AppID $AppClientID -TenantId $TenantGUID -NoWelcome
	}
	
	if (!(Get-MgContext)) {
        throw "Failed to connect to Microsoft Graph."
		exit
    }
}

function connect2ExchangeOnline
{	
	$json = Get-Content -Raw -Path $Configfile
	[PSCustomObject]$config = ConvertFrom-Json -InputObject $json
	
	$EncryptedKeys = $config.EncryptedKeys
	$AppClientID = $config.AppClientID
	$CertificateThumb = $config.CertificateThumb
	$OnmicrosoftTenant = $config.OnmicrosoftTenant
	if ($EncryptedKeys -eq "True")
	{
		$CertificateThumb = UnHashCredentials $CertificateThumb
	}
	$status = "True"
	
	if($status -eq "True")
	{
		Connect-ExchangeOnline -CertificateThumbPrint $CertificateThumb -AppID $AppClientID -Organization $OnmicrosoftTenant -ShowBanner:$false
	}else
	{
		Write-Host "`nThe Certificate set in config.json don't match with the certificates installed on this machine, you can try to execute using manual connection, to do that extecute: "
		Write-Host ".\GetDataExplorer2.ps1 -ManualConnection" -ForeGroundColor Green
		exit
	}
}

function GetM365Accounts
{
	$mailEnabledUser = Get-MgUser -Filter "assignedLicenses/`$count ne 0 and userType eq 'Member'" -ConsistencyLevel eventual -CountVariable licensedUserCount -All
	Write-Host "`nTotal active users : "$mailEnabledUser.count
	Write-Host "Identifying users with email enabled.`n"
	return $mailEnabledUser
}

function ValidateUPNInCSVFIle
{
    $CSVFile = Import-Csv -Path $csvFilePath
	# Regular expression to match email format
    $emailPattern = '^[\w-\.]+@([\w-]+\.)+[\w-]{2,4}$'
	$CountUPNError = 0
	
	Write-Host "`n########## Validating UPN format ##########`n" -ForeGroundColor DarkYellow
    # Iterate over each record in the CSV
    foreach ($record in $CSVFile) 
	{
        # Validate the ManagerUPN field
        if ($record.AccountUPN -match $emailPattern)
		{
            Write-Host "Account UPN valid: $($record.AccountUPN)"
        }else
		{
            Write-Host "Invalid or missing Account UPN: $($record.AccountUPN)"
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

function Create-MailFolder($UPN)
{
	#Validate if the folder previously exist
	try
	{
        # Check if the folder already exists
        $existingFolder = Get-MgUserMailFolderChildFolder -UserId $UPN -MailFolderId "inbox" | Where-Object { $_.DisplayName -eq $FolderName }

        if ($null -ne $existingFolder) {
            Write-Host "Folder '$FolderName' already exists under Inbox for $UPN. Skipping creation."
            #return $existingFolder.Id
        }

        # Create the folder if it doesn't exist
        $newFolder = New-MgUserMailFolderChildFolder -UserId $UPN -MailFolderId "inbox" -BodyParameter @{
            DisplayName = $FolderName
        }

        Write-Host "Folder '$FolderName' created successfully under Inbox for $UPN."
        return $newFolder.Id
    }catch
	{
        # Log the error to the error collector CSV
        $errorMessage = $_.Exception.Message
        $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
        $errorEntry = "$UPN,$errorMessage,$timestamp"
        Add-Content -Path $ErrorFolderCreation -Value $errorEntry
        Write-Error "Failed to create folder for $UPN. Error logged to $ErrorFolderCreation."
    }
}

function Create-InboxRuleUsingExchangeOnline($UPN)
{
	$json = Get-Content -Raw -Path $Configfile
	[PSCustomObject]$config = ConvertFrom-Json -InputObject $json
	$RuleType = $config.RuleType
	$RuleValue = $config.RuleValue
	$TargetFolder = "$($UPN):\Inbox\"+$FolderName
	
	$mailbox = Get-Mailbox -Identity $UPN -ErrorAction SilentlyContinue
	
	if ($mailbox -eq $null)
	{
		Write-Output "Mailbox is inactive or not provisioned for user $UPN."
		return
	}else 
	{
		Write-Output "Mailbox is active for user: $($mailbox.DisplayName)"
	}
	
	$folder = Get-MgUserMailFolderChildFolder -UserId $UPN -MailFolderId "inbox" | Where-Object { $_.DisplayName -eq $FolderName }
	
	if ($null -eq $folder) 
	{
		Write-Host "Folder '$FolderName' not found under Inbox for $UPN."
		Create-MailFolder -UPN $UPN
    }
	
	#Get all the inbox rules per user
	$inboxRules = Get-InboxRule -Mailbox $UPN
	
	# Check if the rule already exists
	if($RuleType -eq "Sender")
	{
		$matchingRules = $inboxRules | Where-Object {$_.Name -eq "Move Emails From $RuleValue"}
	}elseif($RuleType -eq "Subject")
	{
		 $matchingRules = $inboxRules | Where-Object {$_.Name -eq "Move Emails With Subject $RuleValue"}
	}
   
	if ($matchingRules.count -gt 0)
	{
		Write-Host "Inbox rule for '$RuleValue' already exists for $UPN. Skipping creation."
		return
	}
	
	try
	{
        # Create the rule
		Write-Host "Target folder " $TargetFolder
		if ($RuleType -eq "Sender") 
		{
            New-InboxRule -Mailbox $UPN -Name "Move Emails From $RuleValue" `
                -From $RuleValue -MoveToFolder $TargetFolder
        } elseif ($RuleType -eq "Subject") 
		{
            New-InboxRule -Mailbox $UPN -Name "Move Emails With Subject $RuleValue" `
                -SubjectContainsWords @($RuleValue) -MoveToFolder $TargetFolder
        }

        Write-Host "Inbox rule created to move emails by $RuleType is '$RuleValue' to folder '$FolderName' for $UPN."
		$FolderID = $TargetFolder
        $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
        $TrackEntry = "$UPN,$FolderName,$RuleType,$RuleValue,$timestamp"
        Add-Content -Path $TrackFile -Value $TrackEntry
    }catch
	{
        # Log the error to the error collector CSV
        $errorMessage = $_.Exception.Message
        $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
        $errorEntry = "$UPN,$FolderName,$RuleType,$RuleValue,$errorMessage,$timestamp"

        Add-Content -Path $ErrorRuleCreation -Value $errorEntry
        Write-Error "Failed to create Inbox rule for $UPN. Error logged to $ErrorRuleCreation."
    }
	
}

#All the changes related to group are set in the CSV file, if it's the file is open can drop the script
function ValidateIfCSVisOpenByAnotherApp
{
    # Keep checking until the file is available
    while ($true) {
        try {
            # Try to open the file with exclusive access
            $fileStream = [System.IO.File]::Open($csvFilePath, [System.IO.FileMode]::Open, [System.IO.FileAccess]::Read, [System.IO.FileShare]::None)
            $fileStream.Close()
            Write-Host "File is now available." -ForegroundColor Green
            break
        }
        catch {
            # If the file is locked, show a blinking message
            Write-Host "`r[WARNING] The file is currently open by another application. Please close it to proceed..." -ForegroundColor Red -NoNewline
            Start-Sleep -Milliseconds 1000
            Write-Host "`r                                                    " -NoNewline
            Start-Sleep -Milliseconds 500
        }
    }
}

function MainFunction
{
	cls
	
	if(-Not (Test-Path -Path $Configfile ))
	{
		Write-Host "`nIf you need to validate that you have the right PowerShell modules you can execute:`n"
		Write-host ".\ResolveMailAccount.ps1 -CheckDependencies" -ForeGroundColor Green
		Write-Host "`nConfiguration file not available, you need to execute:`n"
		Write-host ".\ResolveMailAccount.ps1 -CreateEntraApp `n`n" -ForeGroundColor Green
		exit
	}
	
	Connect2MicrosoftGraphService
	
	if (-Not (Test-Path -Path $ErrorFolderCreation)) 
	{
        # Create the file with headers if it doesn't exist
        "UPN,ErrorMessage,TimeStamp" | Out-File -FilePath $ErrorFolderCreation -Encoding UTF8
    }
	
	# Ensure the error collector file exists
    if (!(Test-Path -Path $ErrorRuleCreation)) {
        # Create the file with headers if it doesn't exist
        "UPN,FolderName,RuleType,RuleValue,ErrorMessage,TimeStamp" | Out-File -FilePath $ErrorRuleCreation -Encoding UTF8
    }
	
	if (-Not (Test-Path -Path $TrackFile)) 
	{
        # Create the file with headers if it doesn't exist
        "UPN,FolderName,RuleType,$uleValue,timestamp" | Out-File -FilePath $TrackFile -Encoding UTF8
    }
	
	if($ChangeInput)
	{
		$json = Get-Content -Raw -Path $Configfile
		[PSCustomObject]$config = ConvertFrom-Json -InputObject $json
		$method = $config.InputMethod
		
		if($method -eq "CSV")
		{
			$config.InputMethod = "GRAPH"
		}elseif($method -eq "GRAPH")
		{
			$config.InputMethod = "CSV"
		}
		
		$config | ConvertTo-Json | Out-File "$configfile"
	}
	
	$json = Get-Content -Raw -Path $Configfile
	[PSCustomObject]$config = ConvertFrom-Json -InputObject $json
	$InputMethodToUse = $config.InputMethod
	if($InputMethodToUse -eq "CSV")
	{
		CreateCSVFile
		ValidateIfCSVisOpenByAnotherApp
		$CSVFile = Import-Csv -Path $csvFilePath
		ValidateUPNInCSVFIle
		$TotalRows = $CSVFile.count
		if($TotalRows -eq "1")
		{
			if($CSVFile.AccountUPN -eq "UserPrincipalName@yourdomain.com")
			{
				Write-Host "You are using the sample data in the file, please replace with the right one."
				Write-Host "Exiting...`n`n"
				exit
			}
		}
		
		Write-Host "`nConnecting to Exchange Online...`n" -ForeGroundColor Green
		connect2ExchangeOnline
		
		foreach($account in $CSVFile)
		{
			Create-InboxRuleUsingExchangeOnline -UPN $account.AccountUPN
		}
		Write-Host "`nProcess finished...`n"
		Write-Host "################################################################################`n`n" -ForeGroundColor DarkYellow
	}
	if($InputMethodToUse -eq "GRAPH")
	{
		Write-Host "`nConnecting to Exchange Online...`n" -ForeGroundColor Green
		connect2ExchangeOnline
		
		$Accounts = GetM365Accounts
		foreach($account in $Accounts)
		{
			Create-InboxRuleUsingExchangeOnline -UPN $account.UserPrincipalName
		}
		Write-Host "`nProcess finished...`n"
		Write-Host "################################################################################`n`n" -ForeGroundColor DarkYellow
	}
}

# Here global variables are set
$ConfigFile = $PSScriptRoot+"\ConfigFiles\configurationFile.json"
$ErrorFolderCreation = $PSScriptRoot+"\ConfigFiles\ErrorFolderCreation.Csv"
$ErrorRuleCreation = $PSScriptRoot+"\ConfigFiles\ErrorRuleCreation.Csv"
$TrackFile = $PSScriptRoot+"\ConfigFiles\TrackFile.Csv"
$csvFilePath = "$PSScriptRoot\ConfigFiles\InputMailAccounts.csv"
$PathFolder = $PSScriptRoot+"\ConfigFiles"
$FolderName = "E-Migrator"

# Only to create the Microsoft Entra App to automate the proecess
if($CreateEntraApp)
{
	CreateNewEntraApp
	exit
}

# Validate if all the minimal requirements are available
if($CheckDependencies)
{
	cls
	Write-Host "`nValidating dependencies...`n" -ForeGroundColor Green
	CheckPowerShellVersion
	CheckIfElevated
	CheckRequiredModules
	Write-Host "`n`n"
	return
}

if($CreateConfigurationFile)
{
	CreateConfigFile
	exit
}

# Main script
MainFunction
```
