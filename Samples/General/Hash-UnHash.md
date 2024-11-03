# Solution to learn about hash passwords

Often, there’s a need to store `credentials` locally, which can pose a **security risk** if stored in **plain text**. In this example, a configuration file in JSON format is created, where you can manually add a username and password. This setup demonstrates how only the password value can be hashed, while the rest of the file remains in plain text. An attribute called `HashedPasswords` acts as a flag to indicate if the password is hashed.

The same script can also unhash the stored password, providing a simple solution for secure password storage. This process uses PowerShell cmdlets that rely on both the machine ID and the logged-in user. Consequently, only the original user on the same machine can unhash the password, adding an extra layer of security if the file is moved to another machine or accessed by a different user."


```powershell
# Script to explain how to hash passwords using PowerShell

# Attributes to be used with the script
param(
	[Parameter()] 
        [switch]$Hash,
	[Parameter()] 
        [switch]$UnHash,
	[Parameter()] 
        [switch]$CredentialsFile
)

# Define paths
$configFolder = $PSScriptRoot+"\ConfigFiles"
$jsonFile = "$configFolder\MyCredentials.json"

function ConfigFile
{
	# Validate if the directory exist
	if(-Not (Test-Path $configFolder ))
	{
		Write-Host "`nExport data directory is missing, creating a new folder called ConfigFiles"
		New-Item -ItemType Directory -Force -Path $configFolder  | Out-Null
	}
	
	# Validate if the configuration file exists
	if (-Not (Test-Path -Path $jsonFile))
    {
		$config = [ordered]@{
			HashedPassword = "False"
			MyUser = ""
			MyPassword = ""
		}
    }else
	{
		Write-Host "`nConfiguration file is available under ConfigFiles folder.`n"
		exit
	}
	
	$config | ConvertTo-Json | Out-File $jsonFile
    Write-Host "`nNew config file was created under ConfigFile folder.`n" -ForegroundColor Yellow
}

function ValidateConfigurationFile
{
	if (-not (Test-Path -Path $jsonFile))
		{
			Write-Host "`nMissing config file '$jsonFile'." -ForegroundColor Yellow
			Write-Host "`nTo create the missing file please execute " -NoNewLine
			Write-Host ".\Hash-UnHash.ps1 -CredentialsFile`n" -ForegroundColor Green
			exit
		}
	}

function HashCredentials
{
    # Validate if the password file exists  
    ValidateConfigurationFile
	
    $json = Get-Content -Raw -Path $jsonFile
    [PSCustomObject]$config = ConvertFrom-Json -InputObject $json
    $HashedPassword = $config.HashedPassword

    # Check if already encrypted
    if ($HashedPassword -eq "True")
    {
        Write-Host "`nAccording to the configuration settings (HashedPassword: True), password is already hashed." -ForegroundColor Yellow
        Write-Host "`nNo actions taken.`n"
        return
    }

    # Encrypt password
    $UserPassword = $config.MyPassword
    $UserPassword = $UserPassword | ConvertTo-SecureString -AsPlainText -Force | ConvertFrom-SecureString

    # Write results to the password file
    $config.HashedPassword = "True"
    $config.MyPassword = $UserPassword

    $date = Get-Date -Format "yyyyMMddHHmmss"
    Move-Item "$jsonFile" "$PSScriptRoot\ConfigFiles\MyCredentials_$date.json"
    Write-Host "`nPassword hashed."
    Write-Host "`nA backup was created with name " -NoNewLine
	Write-Host "'MyCredentials_$date.json'`n" -ForegroundColor Green
    $config | ConvertTo-Json | Out-File $jsonFile

    Write-Host "Warning!" -ForegroundColor DarkRed
    Write-Host "Please note that encrypted keys can be decrypted only on this machine, using the same account.`n"
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

function MainScript
{
	# Script is executed without any attribute
	if(-Not $Hash -AND -Not $UnHash -AND -Not $CredentialsFile)
	{
		cls
		Write-Host "`n`nThis script demonstrates how to hash and unhash passwords. Follow the steps below:"
		Write-Host "`n`t 1. Run" -NoNewLine
		Write-Host ".\Hash-UnHash.ps1 -CredentialsFile " -NoNewLine -ForegroundColor Green
		Write-Host "to generate a JSON file where you can input your username and password. The file, named MyCredentials.json, will be created in a folder called ConfigFiles."
		Write-Host "`n`t 2. Open the MyCredentials.json file located in the ConfigFiles folder, and enter a username and password inside the quotes for each attribute."
		Write-Host "`n`t 3. Run" -NoNewLine
		Write-Host ".\Hash-UnHash.ps1 -Hash " -NoNewLine -ForegroundColor Green
		Write-Host "to hash the password stored in the JSON file."
		Write-Host "`n`t 4. Run" -NoNewLine
		Write-Host ".\Hash-UnHash.ps1 -Unhash " -NoNewLine -ForegroundColor Green
		Write-Host "to unhash the password stored in the JSON file.`n`n"
		exit
	}
	
	if($CredentialsFile)
	{
		ConfigFile
		exit
	}
	
	if($Hash)
	{
		HashCredentials
		exit
	}
	
	if($UnHash)
	{
		# Validate if the configuration file exists  
		ValidateConfigurationFile
		$json = Get-Content -Raw -Path $jsonFile
		[PSCustomObject]$config = ConvertFrom-Json -InputObject $json
		$HashedPassword = $config.HashedPassword
		$UserPassword = $config.MyPassword
		if ($HashedPassword -eq "True")
		{
			$UserPassword = UnHashCredentials $UserPassword
		}
		
		$config.HashedPassword = "False"
		$config.MyPassword = $UserPassword
		$config | ConvertTo-Json | Out-File $jsonFile
		Write-Host "`nYour password inside MyCredentials.json is unhash.`n"
		exit
	}
}

MainScript
```
<br><br>
