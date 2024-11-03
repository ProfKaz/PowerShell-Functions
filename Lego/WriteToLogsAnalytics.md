# Function used to export data to Logs Analytics

This function exports data stored in a variable in JSON format and sends it to a Log Analytics workspace. It leverages [Build-Signature](/Lego/Build-Signature.md) to establish the connection to Log Analytics.
To use this function, simply pass a `TableName` and an array containing the data in JSON format.
Sample way to use:
```powershell
WriteToLogsAnalytics -LogAnalyticsTableName $TableName -body $TotalResults
```

```powershell
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
```
<br><br>
