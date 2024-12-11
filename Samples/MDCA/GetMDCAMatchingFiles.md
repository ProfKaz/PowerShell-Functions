# Script: Solution to identify matching files with a MDCA Policy.

I developed this script based on a customer requirement, the main objective get a list of all files matching in a MDCA file policy.

The following functions from the 'Lego' folder were utilized:
- [ScriptVariables](../../Lego/ScriptVariables.md)
- Set-ApiConfiguration
- Build-FilterBodyByPolicyId
- Fetch-FilesFromApi
- Retrieve-AllFiles
- Export-FilesToCsv

To configure this script, you need to obtain your `<MDCA-URL>` and `<API-Token>`. These can be found in [Microsoft Defender](https://security.microsoft.com) under **Settings** > **MDCA Settings** > **API Tokens**.

<p align="center">
<img src="https://github.com/user-attachments/assets/34ba8b96-a898-49c7-be9a-cc1791139d18" width="650"></p>
<p align="center">Create a MDCA API token</p>
<br>

Obtaining the Policy ID is slightly more complex. You'll need to locate the File Policy, then extract the Policy ID directly from the URL, as demonstrated in the image below.

<p align="center">
<img src="https://github.com/user-attachments/assets/599d25e6-9f44-42e7-817b-25142f59deeb" width="650"></p>
<p align="center">Identify Policy ID</p>
<br>

<details>
<summary>You can find the complete script here</summary>

```powershell
# Script to get all files matching in a File Policy under MDCA

#Function to set global variables
function ScriptVariables
{
	$script:apiToken = "<API-Token>"
	$script:tenantUrl = "<MDCA-URL>"
	$script:policyID = "651daa212e00000347cccc0b1" # Replace with the desired policy Id
}

# Function to Set API Token and URL
function Set-ApiConfiguration {
    param (
        [string]$ApiToken,
        [string]$TenantUrl
    )
    @{
        "Headers" = @{
            "Authorization" = "Token $ApiToken"
            "Content-Type"  = "application/json"
        }
        "ApiUrl" = "$TenantUrl/api/v1/files/"
    }
}

# Function to Build the Filter Body by Policy
function Build-FilterBodyByPolicy {
    param (
        [string]$PolicyID,
        [int]$Limit = 1000
    )
    @{
        "filters" = @{
            "policy" = @{
                "cabinetmatchedrulesequals" = @($PolicyID)
            }
        }
        "limit" = $Limit
    } | ConvertTo-Json -Depth 10
}

# Function to Fetch Files from API
function Fetch-FilesFromApi {
    param (
        [string]$ApiUrl,
        [hashtable]$Headers,
        [string]$Body
    )
    $response = Invoke-RestMethod -Uri $ApiUrl -Method Post -Headers $Headers -Body $Body
    $response
}

# Function to Handle Pagination
function Retrieve-AllFiles {
    param (
        [string]$ApiUrl,
        [hashtable]$Headers,
        [string]$Body
    )
    $allFiles = @()
    do {
        $response = Invoke-RestMethod -Uri $ApiUrl -Method Post -Headers $Headers -Body $Body
        $allFiles += $response.data
        $nextLink = $response."@odata.nextLink"
        if ($nextLink) {
            $ApiUrl = $nextLink
        }
    } while ($nextLink)
    $allFiles
}

# Function to Export Data to CSV
function Export-FilesToCsv {
    param (
        [array]$Files,
        [string]$FilePath
    )
    $Files | Export-Csv -Path $FilePath -NoTypeInformation
}

# Main Script Execution
function Main {
    $config = Set-ApiConfiguration -ApiToken $apiToken -TenantUrl $tenantUrl

    # Build Filter for Files by Policy Name
    $filterBody = Build-FilterBodyByPolicy -PolicyID $policyID

    # Retrieve All Files
    $allFiles = Retrieve-AllFiles -ApiUrl $config.ApiUrl -Headers $config.Headers -Body $filterBody

    # Export to CSV
    $outputPath = "FilteredFilesByPolicy.csv"
    if ($allFiles) {
        Export-FilesToCsv -Files $allFiles -FilePath $outputPath
        Write-Host "Export completed. File saved to $outputPath"
    } else {
        Write-Host "No matching files found for the specified policy."
    }
}

# Set global variables
ScriptVariables
# Run the Main Function
Main
```

</details>

<br><br>
