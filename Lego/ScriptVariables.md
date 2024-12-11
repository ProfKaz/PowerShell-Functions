# Function to set global variables

When scripting, I often use the param() option at the beginning of my scripts to define environment variables. However, this approach has a drawback: these variables appear as attributes when the script is executed, which might not be ideal.

Placing variables outside of functions is another option, but in lengthy scripts, some variables can become difficult to track or might inadvertently be overwritten.

To address this issue, a better approach is to encapsulate all environment variables within a dedicated function. This method keeps the script organized and minimizes the risk of losing or misusing variables. Here's how it can be implemented:

```powershell
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
```

> [!NOTE]
> Here you need to replace the common variable set as $Variable by $script:Variable that permit to reach this goal.

<br><br> 
