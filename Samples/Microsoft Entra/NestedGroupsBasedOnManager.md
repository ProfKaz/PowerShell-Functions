# Script solution to identify nested direct reports across multiple levels.

I developed this script based on a customer requirement. During discussions about using dynamic groups in Microsoft Entra, we realized that the default option only returns one level of direct reports, while the customer needed to identify multiple levels. Additional requirements included setting the manager as the group owner, checking if the manager is part of the group, adding an external owner, specifying the group type as either `Security` or `Microsoft365`, and defining the recursion depth for each group.

The following functions from the 'Lego' folder were utilized:
- [CheckIfElevated](/Lego/CheckIfElevated.md)
- [CheckPowerShellVersion](/Lego/CheckPowerShellVersion.md)
- [CheckRequiredModules](/Lego/CheckRequiredModules.md)
- [CreateConfigFile](/Lego/CreateConfigFile.md)
- [CheckCertificateInstalled](/Lego/CheckCertificateInstalled.md)
- [CreateNewEntraApp](/Lego/CreateNewEntraApp.md)
- [CreateCodeSigningCertificate](/Lego/CreateCodeSigningCertificate.md)
- [SelfSign](/Lego/SelfSign)
- [CreateTaskSchedulerTask](/Lego/CreateTaskSchedulerTask.md)
- [Connect2MicrosoftGraphService](/Lego/Connect2MicrosoftGraphService.md)
- [CreateCSVFile](/Lego/CreateCSVFile.md)
- [CheckConfigurationFileAvailable](/Lego/CheckConfigurationFileAvailable.md)



<details>
<summary>You can find the complete script here</summary>

Additional helper functions are available, such as `ShowHelpMenu`, which provides guidance on using the complete script, and `CSVHelp`, which assists in populating the CSV file.



```powershell
<#PSScriptInfo

.VERSION 2.0.1

.GUID 883af802-165c-4713-ffc1-352686c02f01

.AUTHOR 
https://www.linkedin.com/in/profesorkaz/; Sebastian Zamorano

.COMPANYNAME 
Sebastian Zamorano

.TAGS 
#Microsoft365 #M365 #MicrosoftEntra

.PROJECTURI 
https://github.com/ProfKaz?tab=repositories

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
The script helps to get nested users from a main manager. 

#>

<#
HISTORY
Script      : NestedGroupsBasedOnManager.ps1
Author      : Sebastian Zamorano
Co-Author   : 
Version     : 2.0.1
Date		: 07-10-2024(dd-MM-yyyy)
Description : The script permits to get all users, including nested users, under a certain manager.
			
.NOTES 
	07-10-2024	S. Zamorano		- First release.
	25-10-2024	S. Zamorano		- Preview Release\
	26-10-2024	S. Zamorano		- Public Release
#>

param(
	[Parameter()] 
        [switch]$ManualConnection,
	[Parameter()] 
        [switch]$HowToCSV,
	[Parameter()] 
        [switch]$ShowHelpMenu,
	[Parameter()] 
        [switch]$Gridview,
	[Parameter()] 
        [switch]$SignScript,
	[Parameter()] 
        [switch]$CreateTaskSchedulerTask,
	[Parameter()] 
        [switch]$CreateCSVFile,
	[Parameter()] 
        [switch]$CreateConfigFile,
	[Parameter()] 
        [switch]$CreateEntraApp,
	[Parameter()] 
        [switch]$ExportToCSV,
	[Parameter()] 
        [int]$SetRecursionDepth,
    [int]$RecursionDepth = 3
)

# To install PowerShell modules or create a Task on Task Scheduler is required to execute this script with admin rights
function CheckIfElevated
{
    $IsElevated = ([Security.Principal.WindowsPrincipal] [Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
    if (!$IsElevated)
    {
        Write-Host "`nPlease start PowerShell as Administrator.`n" -ForegroundColor Yellow
        exit(1)
    }
}

# Validate that PowerShell 7 is in use
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

#Validate the current PowerShell modules required to execute this script
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

#This file is called when the Microsoft Entra App is created, nevertheless, if the file is deleted can be created thhrough the attibute -CreateConfigFile
function CreateConfigFile
{
	# read config file
    $configfile = $PSScriptRoot+"\ConfigFiles\EntraConfig.json"
	
	if(-Not (Test-Path $configfile ))
	{
		Write-Host "Export data directory is missing, creating a new folder called ConfigFiles"
		New-Item -ItemType Directory -Force -Path "$PSScriptRoot\ConfigFiles" | Out-Null
	}
	
	if (-not (Test-Path -Path $configfile))
    {
		$config = [ordered]@{
		AppClientID = ""
		TenantGUID = ""
		CertificateThumb = ""
		}
    }else
	{
		Write-Host "Configuration file is available under ConfigFiles folder"
	}
	
	$config | ConvertTo-Json | Out-File "$configfile"
    Write-Host "New config file was created under ConfigFile folder." -ForegroundColor Yellow
}

# The function Connect2MicrosoftGraphService validate if the certificate set in the EntraConfig.json file matches withe the certifictaes installed in this machine
function CheckCertificateInstalled($thumbprint)
{
	$var = "False"
	$certificates = @(Get-ChildItem Cert:\CurrentUser\My | Where-Object {$_.EnhancedKeyUsageList -like "*Client Authentication*"}| Select-Object Thumbprint) 
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
		Write-Host "`t`t`t`tPassed!" -ForegroundColor Green
		return $var
	}else
	{
		Write-Host "`nCertificate installed on this machine is missing!!!" -ForeGroundColor Yellow
		Write-Host "To execute this script unattended a certificate needs to be installed, the same used under Microsoft Entra App"
		Start-Sleep -s 1
		return $var
	}
}

# Used to connect automatically through Microsoft Graph API to collect the data and crete or upgrade groups
function CreateNewEntraApp
{
    Connect-MgGraph -Scopes "Application.ReadWrite.All", "AppRoleAssignment.ReadWrite.All", "Directory.ReadWrite.All", "User.ReadWrite.All" -NoWelcome

	$CONFIGFILE = $PSScriptRoot+"\ConfigFiles\EntraConfig.json"
	if(-not (Test-Path -path $CONFIGFILE))
	{
		CreateConfigFile
	}
	
	$json = Get-Content -Raw -Path $CONFIGFILE
	[PSCustomObject]$config = ConvertFrom-Json -InputObject $json
	
    $appName = "Microsoft Entra Groups"
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
            ResourceAppId = "00000003-0000-0000-c000-000000000000"
            ResourceAccess = @(
                @{
                    Id = "e1fe6dd8-ba31-4d61-89e7-88639da4683d"
                    Type = "Scope"
                },
                @{
                    Id = "62a82d76-70ea-41e2-9197-370581804d09"
                    Type = "Role"
                },
                @{
                    Id = "df021288-bdef-4463-88db-98f22de89214"
                    Type = "Role"
                },
                @{
                    Id = "7ab1d382-f21e-4acd-a863-ba3e13f7da61"
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
    $certName = "Microsoft Entra Groups"

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
	

    Write-Host "`nAzure application was created."
    Write-Host "App Name: $appName"
    Write-Host "App ID: $($app.AppId)"
	Write-Host "Tenant ID: $TenantID"
    Write-Host "Certificate thumbprint: $($cert.Thumbprint)"

    Write-Host "`nPlease go to the Azure portal to manually grant admin consent:"
    Write-Host "https://portal.azure.com/#view/Microsoft_AAD_RegisteredApps/ApplicationMenuBlade/~/CallAnAPI/appId/$($app.AppId)`n" -ForegroundColor Cyan

    $config.TenantGUID = $TenantID
	$config.AppClientID = $app.AppId
    $config.CertificateThumb = $cert.Thumbprint
	
	$config | ConvertTo-Json | Out-File $CONFIGFILE

    Remove-Variable cert
    Remove-Variable certBase64
}

# Create a certificate self-sign to sign script code
function CreateCodeSigningCertificate
{
	#CMDLET to create certificate
	$MicrosoftEntraCert = New-SelfSignedCertificate -Subject "CN=Microsoft Entra PowerShell Code Signing Cert" -Type "CodeSigning" -CertStoreLocation "Cert:\CurrentUser\My" -HashAlgorithm "sha256"
		
	### Add Self Signed certificate as a trusted publisher (details here https://adamtheautomator.com/how-to-sign-powershell-script/)
		
		# Add the self-signed Authenticode certificate to the computer's root certificate store.
		## Create an object to represent the CurrentUser\Root certificate store.
		$rootStore = [System.Security.Cryptography.X509Certificates.X509Store]::new("Root","CurrentUser")
		## Open the root certificate store for reading and writing.
		$rootStore.Open("ReadWrite")
		## Add the certificate stored in the $authenticode variable.
		$rootStore.Add($MicrosoftEntraCert)
		## Close the root certificate store.
		$rootStore.Close()
			 
		# Add the self-signed Authenticode certificate to the computer's trusted publishers certificate store.
		## Create an object to represent the CurrentUser\TrustedPublisher certificate store.
		$publisherStore = [System.Security.Cryptography.X509Certificates.X509Store]::new("TrustedPublisher","CurrentUser")
		## Open the TrustedPublisher certificate store for reading and writing.
		$publisherStore.Open("ReadWrite")
		## Add the certificate stored in the $authenticode variable.
		$publisherStore.Add($MicrosoftEntraCert)
		## Close the TrustedPublisher certificate store.
		$publisherStore.Close()	
}

# Used to sign the script using a self-signed certificate 
function SelfSign
{
	$certificates = @(Get-ChildItem Cert:\CurrentUser\My | Where-Object {$_.EnhancedKeyUsageList -like "*Code Signing*"}| Sort-Object NotBefore -Descending | Select-Object Subject, Thumbprint, NotBefore, NotAfter)
	$cert = Get-ChildItem Cert:\CurrentUser\My -CodeSigningCert | Where-Object {$_.Thumbprint -eq $certificates[0].Thumbprint}
	$EntraScript = Get-ChildItem -Path .\NestedGroupsBasedOnManager.ps1
	Set-AuthenticodeSignature -FilePath ".\$($EntraScript.Name)" -Certificate $cert
}

# Menu explainig how to complete the CSV file used in this script
function CSVHelp
{
    # Explanation Header
    Write-Host "'n'nWelcome to the Group Creation Helper!" -ForegroundColor Green
    Write-Host "This script uses a CSV file to automate group creation or update. Below are the fields you need to complete in the CSV file:"
    Write-Host "-------------------------------------------------------------"

	# How to generate the CSV file
	Write-Host "`nTo create the CSV file to populate with the next information, you need to execute:"
	Write-Host "`t.\NestedGroupsBasedOnManager.ps1 -CreateCSVFile" -ForegroundColor Blue
    
	# CSV Parameters
    Write-Host "`n1. ManagerUPN:" -ForegroundColor Magenta
    Write-Host "   - The User Principal Name (UPN) of the manager responsible for the group."
    Write-Host "   - This can is used to identify all the direct reports or nested direct reports from this maanger."
    Write-Host "   Example: manager@domain.com" -ForegroundColor Cyan

    Write-Host "`n2. IncludeManager:" -ForegroundColor Magenta
    Write-Host "   - Set this value to 'True' or 'False'."
    Write-Host "   - If 'True', the manager will be included in the group as a member."
    Write-Host "   Example: True" -ForegroundColor Cyan

    Write-Host "`n3. ManagerAsOwner:" -ForegroundColor Magenta
    Write-Host "   - Set this value to 'True' or 'False'."
    Write-Host "   - If 'True', the manager will also be added as an owner of the group."
    Write-Host "   Example: True" -ForegroundColor Cyan

    Write-Host "`n4. GroupOwner:" -ForegroundColor Magenta
    Write-Host "   - The UPN of the additional owner for the group (other than the manager, if applicable)."
    Write-Host "   - This will be the main owner."
    Write-Host "   Example: GroupOwner@domain.com" -ForegroundColor Cyan

    Write-Host "`n5. NewGroup:" -ForegroundColor Magenta
    Write-Host "   - The display name for the new group being created."
    Write-Host "   - This name will appear in the directory and be used to identify the group."
    Write-Host "   Example: 'Finance Team Group'" -ForegroundColor Cyan

    Write-Host "`n6. GroupDescription:" -ForegroundColor Magenta
    Write-Host "   - A short description of the group, explaining its purpose."
    Write-Host "   - This helps in providing context about the group."
    Write-Host "   Example: 'This group is used for all finance department communications.'" -ForegroundColor Cyan

    Write-Host "`n7. GroupType:" -ForegroundColor Magenta
    Write-Host "   - Specify the type of group: 'Security' or 'Microsoft365'."
    Write-Host "   - 'Security' for security groups, 'Microsoft365' for Microsoft 365 groups."
    Write-Host "   Example: Security" -ForegroundColor Cyan

    Write-Host "`n8. ExistingGroup:" -ForegroundColor Magenta
    Write-Host "   - If the group already exists and needs to be modified, enter the group’s Object ID here."
    Write-Host "   - Leave this blank if creating a new group."
    Write-Host "   Example: 'abc12345-xxxx-xxxx-xxxx-xxxxxxxxxx'" -ForegroundColor Cyan

    Write-Host "`n9. RecursionDepth:" -ForegroundColor Magenta
    Write-Host "   - Used for dynamic groups to set how deep the membership hierarchy should be checked."
    Write-Host "   - Set to '1' for no recursion, higher values for nested group checks."
    Write-Host "   Example: 3" -ForegroundColor Cyan

    Write-Host "`n`n-------------------------------------------------------------"
    Write-Host "Complete your CSV file with these values, and the script will handle the group creation or modification process automatically."
    Write-Host "For further details on specific parameters, consult the documentation or run Get-Help on related cmdlets."
    Write-Host "-------------------------------------------------------------"
    
    # Group Creation Cases
    Write-Host "`nCase Scenarios for Group Creation:" -ForegroundColor Green
    Write-Host "-------------------------------------------------------------"

    # Case 1: Microsoft 365 Group
    Write-Host "`n1. Microsoft 365 Group:" -ForegroundColor Magenta
    Write-Host "   -GroupType: Microsoft365"
    Write-Host "This is a standard Microsoft 365 group that is mail-enabled but not security-enabled."
    Write-Host "Example: New-AzureADGroup -DisplayName 'M365 Group' -MailEnabled \$true -MailNickName 'GroupAlias' -SecurityEnabled \$false"

    # Case 2: Security Group
    Write-Host "`n2. Security Group:" -ForegroundColor Magenta
    Write-Host "   -GroupType: Security"
    Write-Host "This is a standard Security group without email functionality."
    Write-Host "Example: New-AzureADGroup -DisplayName 'Security Group' -MailEnabled \$false -SecurityEnabled \$true"

    Write-Host "`n`n-------------------------------------------------------------"
    Write-Host "These cases will guide you in creating different types of groups based on the values in your CSV file.`n`n"
}

# How to use this script
function ShowHelpMenu 
{  
    $ScriptName = ".\NestedGroupsBasedOnManager.ps1"
	Write-Host "`n`nUsage: $ScriptName [<attributes>]" -ForegroundColor Cyan
    Write-Host "`nAttributes:" -ForegroundColor Yellow
    Write-Host "`n  -ManualConnection" -ForegroundColor Green
    Write-Host "    Establishes a manual Microsoft Graph connection. Can be combined with ExportToCSV or GridView."

    Write-Host "`n  -CreateCSVFile" -ForegroundColor Green
    Write-Host "    Creates a CSV file for input to this script. Should be run alone."

    Write-Host "`n  -CreateEntraApp" -ForegroundColor Green
    Write-Host "    Creates a Microsoft Entra App to automate the connection. Should be run alone."
	
	Write-Host "`n  -ConfigFile" -ForegroundColor Green
    Write-Host "    Creates a configuration file to store connection details. Runs automatically with CreateEntraApp or can be run standalone."

    Write-Host "`n  -SignScript" -ForegroundColor Green
    Write-Host "    Generates a self-signed code signing certificate installed locally and used to sign this script."

    Write-Host "`n  -ExportToCSV" -ForegroundColor Green
    Write-Host "    Exports the results of the script into CSV format. Can be used with ManualConnection or not."

    Write-Host "`n  -GridView" -ForegroundColor Green
    Write-Host "    Displays the results in a PowerShell pop-up using GridView. Can be used with ManualConnection or not."

    Write-Host "`n  -CreateTaskSchedulerTask" -ForegroundColor Green
    Write-Host "    Creates a task under Task Scheduler to run this script every 30 days by default. The task is created in a new folder called MicrosoftEntra."

    Write-Host "`n  -HowToCSV" -ForegroundColor Green
    Write-Host "    Displays help on how to fulfill the CSV file for this script."

    Write-Host "`n  -SetRecursionDepth" -ForegroundColor Green
    Write-Host "    Sets the recursion depth. Defaults to 3 in the script or can be set through this attribute. The value in the CSV file is mandatory by default."

    Write-Host "`nInstructions:" -ForegroundColor Yellow
    Write-Host "  If the script is executed without any attributes, it will:"
    Write-Host "    - Use the ConfigFile to connect"
    Write-Host "    - Use the CSV file for input data"
    Write-Host "    - Connect to the Microsoft Graph API to create new Microsoft Entra groups or update existing ones.`n`n"
}

#This function creates a Folder called "MicrosoftEntra" inside of Task Scheduler app and add a task called "NestedGroups"
function CreateTaskSchedulerTask
{
	# Default folder for Microsoft Entra tasks
    $MicrosoftEntraFolder = "MicrosoftEntra"
	$taskFolder = "\"+$MicrosoftEntraFolder+"\"
	
	# Nested Groups Based On Manager script
    $taskName = "NestedGroups"
	
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

#Used to connect Manually or Automatically connection
function Connect2MicrosoftGraphService
{		

	<#
	.NOTES
	Special permissions to Microsoft Graph can be required, check the initial notes in each script
	#>
	if($ManualConnection)
	{
		Write-Host "`nAuthentication is required, please check your browser" -ForegroundColor Green
		Write-Host "Please note that manual connection might not work because some additional permissions may be required." -ForegroundColor DarkYellow
		Connect-MgGraph -Scopes "Group.ReadWrite.All", "Directory.ReadWrite.All", "User.Read.All" -NoWelcome
	}else
	{
		$ConfigFile = $PSScriptRoot+"\ConfigFiles\EntraConfig.json"
		
		#Check if the configuration file exist or not
		if(-Not (Test-Path -Path $ConfigFile))
		{
			Write-Host "`nConfiguration file not available, you have these options:"
			Write-Host "You can use for a manual connection : " -NoNewLine
			Write-Host "`t.\NestedGroupsBasedOnManager.ps1 -ManualConnection" -ForeGroundColor Green
			Write-Host "You can configure a Microsoft Entra App to automate the connection using : " -NoNewLine
			Write-host "`t.\NestedGroupsBasedOnManager.ps1 -CreateEntraApp`n`n" -ForeGroundColor Green
			exit
		}
		
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
		}else
		{
			Write-Host "`nThe Certificate set in EntraConfig.json don't match with the certificates installed on this machine, you can try to execute using manual connection, to do that extecute: "
			Write-Host ".\NestedGroupsBasedOnManager.ps1 -ManualConnection" -ForeGroundColor Green
			exit
		}
		
	}
}

#Creates the CSV file used as main an unique input to create groups
function CreateCSVFile
{
	$PathFolder = $PSScriptRoot+"\ConfigFiles"
	
	if(-not (Test-Path -Path $PathFolder))
	{
		New-Item -ItemType Directory -Force -Path "$PSScriptRoot\ConfigFiles" | Out-Null
	}
	# Define the CSV file path
    $csvFilePath = $PSScriptRoot+"\ConfigFiles\ManagerGroupsMatrix.csv"
	
	# Check if the CSV file already exists
    if (-Not (Test-Path $csvFilePath))
	{
		# Create a CSV structure
		$ManagerUPN = "YourManagerUserPrincipalName@yourdomain.com"
		$GroupOwner = "OtherGroupOwnerUserPrincipalName@yourdomain.com"
		$IncludeManager = "TRUE"
		$ManagerAsOwner = "FALSE"
		$NewGroup = "Set the name of your new group"
		$GroupDescription = "Set your group description"
		$GroupType = "Use 'security' or 'microsoft365'"
		$data = @{
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
		
		# Create a custom object with a defined order of properties
		$SortedCSV = [pscustomobject]@{
			ManagerUPN		= $data.ManagerUPN
			IncludeManager	= $data.IncludeManager #Include the manager in the same group or not
			ManagerAsOwner	= $data.ManagerAsOwner #Set manager as a group Owner
			GroupOwner		= $data.GroupOwner #Set a group Owner
			NewGroup		= $data.NewGroup
			GroupDescription= $data.GroupDescription
			GroupType		= $data.GroupType
			ExistingGroup	= $data.ExistingGroup
			RecursionDepth	= $data.RecursionDepth
		}
		# If file does not exist, create it with headers
		$SortedCSV | Export-Csv -Path $csvFilePath -NoTypeInformation
		Write-Host "Created new CSV file: $csvFilePath"
    } else
	{
		# If file exists, append new data
		Write-Host "File is existing on path."
		exit
    }
}

# At the begins of the script a global variable is set related to ManagerGroupsMatrix.csv file, this script validate and create the file if it's required
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

#function to validate the format of the UPN added to the file
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

#function to validate the format of the Group ID added to the file
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

#function to validate that at least NewGroup or Existing group contains a record
function ValidateGroupsFields
{
	$CountEmpty = 0
	
	Write-Host "`n########## Validating Group Info added ##########`n" -ForeGroundColor DarkYellow
    # Iterate over each record in the CSV
    foreach ($record in $CSVFile) 
	{
        # Validate the ExistingGroup field
        if ($record.NewGroup -eq "" -and $record.ExistingGroup -eq "") 
		{
			$CountEmpty++
        }
	}
	if($CountEmpty -gt 0)
	{
		Write-Host "Total of records without any group set : " -NoNewline
		Write-Host $CountEmpty -ForegroundColor Green
		Write-Host "Please review the file located at $ConfigurationFile and set a group under NewGroup or Group ID under ExistingGroup."
		Write-Host "`n#####################################################`n" -ForeGroundColor DarkYellow
		exit
	}
	Write-Host "`n#####################################################`n" -ForeGroundColor DarkYellow
}

#All the changes related to group are set in the CSV file, if it's the file is open can drop the script
function ValidateIfCSVisOpenByAnotherApp
{
    # Keep checking until the file is available
    while ($true) {
        try {
            # Try to open the file with exclusive access
            $fileStream = [System.IO.File]::Open($ConfigurationFile, [System.IO.FileMode]::Open, [System.IO.FileAccess]::Read, [System.IO.FileShare]::None)
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

#This script returns all the direct reports from a manager and returns a list of people on this group that is manager
function DirectReports($managers,$IncludeManager)
{
	# Initialize an array to hold all users (managers + end users)
	$allUsers = @()
	$NestedManagers = @()
	
	foreach($UserLead in $managers)
	{
		$ManagedUsers = Get-MgUserDirectReport -UserId $UserLead
		if($IncludeManager -eq "True")
		{
			$allUsers += $UserLead
		}
		# Loop through first-level managers
		foreach ($manager in $ManagedUsers) {
			# Write-host "`t" $manager.Id -ForeGroundColor Green
			$IsItManager = Get-MgUserDirectReport -UserId $manager.Id
			if ($IsItManager.count -ne 0)
			{
				$NestedManagers += $manager.Id
			}
			$allUsers += $manager.Id
		}
	}
	#Write-Host "Nested managers :"$NestedManagers.count
	return @{
			AllUsers = $allUsers
			NestedManagers = $NestedManagers
			}
}

#When you use the attribute ExportToCSV this function write the results in CSV format.
function WriteToCsv($results, $Manager, $RecursionDepth)
{
	$date = (Get-Date).ToString("yyyy-MM-dd")
	
	##Export folder Name
	$ExportFolderName = "ExportedData"
	$ExportPath = "$PSScriptRoot\$ExportFolderName"
	if(-Not (Test-Path $ExportPath))
	{
		New-Item -ItemType Directory -Force -Path "$PSScriptRoot\$ExportFolderName" | Out-Null
		Write-Host "Folder $ExportFolderName was created"
	}	
	
	$FileName = "$date"+" - Nested direct reports - From "+$Manager+" - Depth "+$RecursionDepth+".Csv"
	$pathCsv = $PSScriptRoot+"\"+$ExportFolderName+"\"+$FileName
	$results | Export-Csv -Path $pathCsv -NTI -Force -Append | Out-Null

	Write-Host "`nData exported to..." -NoNewline
	Write-Host "`n$pathCsv" -ForeGroundColor Cyan
}

#If the ExistingGroup field under the CSV file is populated with the ID of the group this information is used to update the groups
function Update-ExistingGroup($GroupId, $GroupDescription, $GroupType, [array]$MembershipGroup)
{
    ValidateExistingGroupField
	
	# Fetch the group to validate its existence
    $existingGroup = Get-MgGroup -GroupId $
	$currentMembers = Get-MgGroupMember -GroupId $GroupId | Select-Object -ExpandProperty Id
	
	Write-Host "Updating an existing group: $($GroupId)`n" -ForegroundColor Yellow
	
    if ($existingGroup) {
        Write-Host "Updating group: $($existingGroup.DisplayName), ID: $GroupId" -ForegroundColor Yellow
        
        # Determine the group type based on MailEnabled and SecurityEnabled
        if ($GroupType -eq "Microsoft365") {
            # Microsoft 365 Group
            Update-MgGroup -GroupId $GroupId -Description $GroupDescription
            Write-Host "Updated Microsoft 365 group: $($existingGroup.DisplayName)" -ForegroundColor Green

        }elseif ($GroupType -eq "Security" )
		{
            # Security Group
            Update-MgGroup -GroupId $GroupId -Description $GroupDescription
            Write-Host "Updated Security group: $($existingGroup.DisplayName)" -ForegroundColor Green

        }else 
		{
            Write-Host "Unknown group type. Unable to update." -ForegroundColor Red
        }

        # Add members to the existing group from the MembershipGroup array
        foreach ($userId in $MembershipGroup) {
            if ($currentMembers -contains $userId)
			{
				Write-Host "User $userId is already a member of the group $($newGroup.DisplayName). Skipping..."
			}else
			{
				New-MgGroupMember -GroupId $GroupId -DirectoryObjectId $userId
				Write-Host "Added user $userId to new group $($newGroup.DisplayName)"
			}
        }
		
		# Remove members from currentMembers that are not in MembershipGroup
        $membersToRemove = $currentMembers | Where-Object { $MembershipGroup -notcontains $_ }
        foreach ($userId in $membersToRemove) {
            #Remove-MgGroupMember -GroupId $GroupId -DirectoryObjectId $userId
			Remove-MgGroupMemberByRef -GroupId $GroupId -DirectoryObjectId $userId
            Write-Host "Removed user $userId from group $($existingGroup.DisplayName)"
        }
		
    } else {
        Write-Host "Group with ID $GroupId does not exist." -ForegroundColor Red
    }
}

#This function is used to create the new Microsoft Entra Groups set under the column "NewGroup"
function CreateMicrosoftEntraGroup([array]$MembershipGroup, $GroupName, $GroupType, $GroupDescription, $GroupOwner, $ManagerAsOwner)
{
	$groups = get-mggroup -All
	$GroupExists = $groups | Where-Object { $_.DisplayName -eq $GroupName }
	if($GroupExists)
	{
		Write-Host "Group '$GroupName' exists." -ForeGroundColor DarkYellow
		Return
	}else
	{
		# Case: Create a new group
		if ($GroupType -eq "Microsoft365") {
			# Create Microsoft 365 Group
			$newGroup = New-MgGroup -DisplayName $GroupName `
									-Description $GroupDescription `
									-MailEnabled:$true `
									-SecurityEnabled:$false `
									-MailNickname ($GroupName -replace " ", "") `
									-GroupTypes @("Unified")
		}
		elseif ($GroupType -eq "Security") {
			# Create Security Group
			$newGroup = New-MgGroup -DisplayName $GroupName `
									-Description $GroupDescription `
									-MailNickname ($GroupName -replace " ", "") `
									-MailEnabled:$false `
									-SecurityEnabled:$true
		}

		# If the group was successfully created
		if ($newGroup) 
		{
			# Update the CSV: Clear NewGroup, populate ExistingGroup with Group ID
			# Return an array to reeplace values
			#$group.ExistingGroup = $newGroup.Id
			#$group.NewGroup = ""  # Clear the NewGroup field
			
			Write-Host "Group created: $($newGroup.DisplayName), ID: $($newGroup.Id)" -ForegroundColor Green
			$currentMembers = Get-MgGroupMember -GroupId $newGroup.Id | Select-Object -ExpandProperty Id
			$currentOwners = Get-MgGroupOwner -GroupId $newGroup.Id | Select-Object -ExpandProperty Id

			# Optionally add Manager as Owner
			if ($ManagerAsOwner) 
			{
				$manager = Get-MgUser -Filter "userPrincipalName eq '$($ManagerAsOwner)'"
				$ManagerId = $manager.Id
				if ($currentOwners -contains $ManagerId)
				{
					Write-Host "User $userId is already a owner of the group $($newGroup.DisplayName). Skipping..."
				}else
				{
					New-MgGroupOwner -GroupId $newGroup.Id -DirectoryObjectId $manager.Id
					Write-Host "Added manager $($ManagerAsOwner) as owner."
				}
			}

			if ($GroupOwner) 
			{
				$owner = Get-MgUser -Filter "userPrincipalName eq '$($GroupOwner)'"
				$OwnerId = $owner.Id
				if ($currentOwners -contains $OwnerId)
				{
					Write-Host "User $userId is already a owner of the group $($newGroup.DisplayName). Skipping..."
				}else
				{
					New-MgGroupOwner -GroupId $newGroup.Id -DirectoryObjectId $owner.Id
					Write-Host "Added additional user $($GroupOwner) as owner."
				}

			}

			# Add members from the MembershipGroup array to the newly created group
			foreach ($userId in $MembershipGroup)
			{
				if ($currentMembers -contains $userId)
				{
					Write-Host "User $userId is already a member of the group $($newGroup.DisplayName). Skipping..."
				}else
				{
					New-MgGroupMember -GroupId $newGroup.Id -DirectoryObjectId $userId
					Write-Host "Added user $userId to new group $($newGroup.DisplayName)"
				}
			}
		}else 
		{
			Write-Host "Failed to create group: $($group.NewGroup)" -ForegroundColor Red
		}
		
		$GroupID = $newGroup.Id
		Return $GroupID
	}
    # Save updated CSV file
    #$groups | Export-Csv -Path $ConfigurationFile -NoTypeInformation
    #Write-Host "CSV file updated successfully." -ForegroundColor Green
}

# Identify the main configuration based on the CSV file
function GetUsersFromManager
{
	ValidateUPNInCSVFIle
	ValidateGroupsFields
	
	# Define the top-level manager UPN (or Object ID)
	foreach($lead in $CSVFile)
	{
		# Get the top-level manager
		$topManager = Get-MgUser -UserId $lead.ManagerUPN
		$LevelManagerId = $topManager.Id
		$ManagerContact = $lead.ManagerUPN
		$NewGroup = $lead.NewGroup
		$ExistingGroup = $lead.ExistingGroup
		$GroupType = $lead.GroupType
		$CSVRecursionDepth = [int]$lead.RecursionDepth
		$IncludeManager = $lead.IncludeManager
		$GroupDescription = $lead.GroupDescription
		$ManagerAsOwner = $lead.ManagerAsOwner
		$GroupOwner = $lead.GroupOwner
		
		if($SetRecursionDepth)
		{
			$Count = $SetRecursionDepth
		}else
		{
			if($CSVRecursionDepth -gt 0)
			{
				$Count = $CSVRecursionDepth
			}Else
			{
				$Count = $RecursionDepth
			}
		}
		Write-Host "Recursion depth set to:" $Count
		
		# If the description field is empty the value will be replaced with a default value
		$DefaultGroupDescription = "This group was created automatically using NestedGroupsBasedOnManager.ps1 script."
		if($GroupDescription -eq "")
		{
			$GroupDescription = $DefaultGroupDescription
		}
		
		$ReportingUsers = @()
		$DepthLevel = $Count
		
		while ($Count -ne 0)
		{
			Write-Host "Level Managers :" $LevelManagerId
			$results = DirectReports -managers $LevelManagerId -IncludeManager $IncludeManager
			$ReportingUsers += $results.AllUsers
			$Count--
			$ReportingManagers = $results.NestedManagers
			Write-Host "Recursion depth reduced to:" $Count
			Write-Host "Managers found in the recursion:" $ReportingManagers.count
			if($ReportingManagers.count -eq 0)
			{
				$Count = 0
			}
			$LevelManagerId = $ReportingManagers
		}
	
		$MembershipGroup = $ReportingUsers | Sort-Object | Get-Unique
		Write-Host "All users: " -NoNewLine
		Write-Host $MembershipGroup.count -ForeGroundColor Green

		if($Gridview)
		{
			$ShowUsers = @()
			foreach ($MemeberId in $MembershipGroup)
			{
				$UserDetails = Get-MgUser -UserId "$MemeberId" | select Id, DisplayName, Mail, UserPrincipalName
				$ShowUsers += $UserDetails
			}
			$ShowUsers | out-gridview
		}elseif($ExportToCSV)
		{
			$ShowUsers = @()
			foreach ($MemeberId in $MembershipGroup)
			{
				$UserDetails = Get-MgUser -UserId "$MemeberId" | select Id, DisplayName, Mail, UserPrincipalName
				$ShowUsers += $UserDetails
			}
			WriteToCsv -results $ShowUsers -manager $lead.ManagerUPN -RecursionDepth $DepthLevel
		}else
		{
			if ($NewGroup -ne "") 
			{
				if($ManagerAsOwner -eq "False" -and $GroupOwner -eq "")
				{
					Write-Host "No owner was set."
					$NewGroupID = CreateMicrosoftEntraGroup -MembershipGroup $MembershipGroup -GroupType $GroupType -GroupName $NewGroup -GroupDescription $GroupDescription
				}elseif($ManagerAsOwner -eq "False" -and $GroupOwner -ne "")
				{
					Write-Host "Group owner was set."
					#I need to pass UPN
					$NewGroupID = CreateMicrosoftEntraGroup -MembershipGroup $MembershipGroup -GroupType $GroupType -GroupName $NewGroup -GroupDescription $GroupDescription -GroupOwner $GroupOwner
				}elseif($ManagerAsOwner -eq "TRUE" -and $GroupOwner -eq "")
				{
					Write-Host "Manager set as group owner." 
					#I need to pass UPN
					$NewGroupID = CreateMicrosoftEntraGroup -MembershipGroup $MembershipGroup -GroupType $GroupType -GroupName $NewGroup -GroupDescription $GroupDescription -ManagerAsOwner $ManagerContact
				}elseif($ManagerAsOwner -eq "TRUE" -and $GroupOwner -ne "")
				{
					Write-Host "Manager and Group owner was set."
					#I need to pass UPN
					$NewGroupID = CreateMicrosoftEntraGroup -MembershipGroup $MembershipGroup -GroupType $GroupType -GroupName $NewGroup -GroupDescription $GroupDescription -ManagerAsOwner $ManagerContact -GroupOwner $GroupOwner
				}
				$lead.ExistingGroup = $NewGroupID
				$lead.NewGroup = ""
			}elseif ($NewGroup -eq "" -and $ExistingGroup -ne "")
			{
				# Case: Update an existing group
				$NewGroupID = $ExistingGroup
				Update-ExistingGroup -GroupId $ExistingGroup `
									 -GroupDescription $GroupDescription `
									 -GroupType $GroupType `
									 -MembershipGroup $MembershipGroup
			}			
		}
		$lead.NewGroup = ""
		$lead.ExistingGroup = $NewGroupID
		Write-Host "`n"
	}
	# Export the modified data back to the CSV file
	if($ExportToCSV -or $Gridview)
	{
		return
	}else
	{
		$CSVFile | Export-Csv -Path $ConfigurationFile -NoTypeInformation -Force
		Write-Host "CSV file updated successfully." -ForeGroundColor Green
		Write-Host "`n`n"
	}
}

# Main script
function MainScript
{
	if($SignScript)
	{
		cls
		Write-Host "`nWe will generate a digital certificate for code sign, please press yes in the pop-up that will be appear.`n" -ForeGroundColor Green
		CreateCodeSigningCertificate
		SelfSign
		Write-Host "`n`nScript was digital signed, please execute again.`n`n"
		exit
	}

	if($CreateTaskSchedulerTask)
	{
		cls
		CheckIfElevated
		Write-Host "`nPlease remember that to use a PowerShell script under task scheduler is recommended to sign your script, you can accomplish executing:"
		Write-Host "`n`t.\NestedGroupsBasedOnManager.ps1 -SignScript`n`n" -ForeGroundColor Green
		CreateTaskSchedulerTask
		exit
	}

	if($CreateCSVFile)
	{
		cls
		CreateCSVFile
		exit
	}

	if($CreateConfigFile)
	{
		cls
		CreateConfigFile
		exit
	}
	
	if($CreateEntraApp)
	{
		cls
		CheckRequiredModules
		CreateNewEntraApp
		exit
	}
	
	if($HowToCSV)
	{
		cls
		CSVHelp
		exit
	}
	
	if($ShowHelpMenu)
	{
		cls
		ShowHelpMenu
		exit
	}
	
	cls
	ValidateIfCSVisOpenByAnotherApp
	CheckPowerShellVersion
	CheckRequiredModules
	Connect2MicrosoftGraphService
	GetUsersFromManager
}

$ConfigurationFile = "$PSScriptRoot\ConfigFiles\ManagerGroupsMatrix.csv"
CheckConfigurationFileAvailable
$CSVFile = Import-Csv -Path $ConfigurationFile

MainScript

```
</details>
<br><br>
