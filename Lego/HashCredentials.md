# Function to hash the password

You can find a simple exercise in this [Hash-Unhash script](/Samples/General/Hash-UnHash.md) where we are able to create a [configuration file](/Lego/CreateConfigFile.md) that is used to store a password, the function read the value in the Json file and update the value `MyPassword` with the hashed one.
You can check the function [UnHashCredentials](/Lego/UnHashCredentials) to work with hashed passwords.

```powershell
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
```
<br><br>
