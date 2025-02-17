# Create a Microsoft Purview Role Group

Microsoft Purview enables you to work using the principle of least privilege. This means you can create "Role Groups," which are subgroups of roles designed for specific activities.

The following PowerShell script creates a Microsoft Purview Role Group. The group’s name is defined at the beginning of the script in the **ScriptVariables** section.

This approach is particularly useful when you have multiple automated scripts that require a limited set of permissions, as it automatically assigns only the specific permissions needed.

To verify that you have access to the correct cmdlets, run the following command:

```powershell
Get-Command -Module tmp* | ft Name, CommandType -AutoSize
```

This command will provide insights into the available commands and the information accessible with the roles assigned to the new role group.

```powershell
# Script to Create Role Groups.
# You need to execute this script manually with enough permissions to create role groups, Compliance Administrator role is recommended. 

function ScriptVariables
{
	[string]$script:RoleGroupName = "My custom role group"
}

function Get-RoleSelected
{
	 return @{
        "Admin Unit Extension Manager" = $false
        "Attack Simulator Admin" = $false
        "Attack Simulator Payload Author" = $false
        "Audit Logs" = $false
        "Billing Admin" = $false
        "Case Management" = $false
        "Communication" = $false
        "Communication Compliance Admin" = $false
        "Communication Compliance Analysis" = $false
        "Communication Compliance Case Management" = $false
        "Communication Compliance Investigation" = $false
        "Communication Compliance Viewer" = $true
        "Compliance Administrator" = $false
        "Compliance Manager Administration" = $false
        "Compliance Manager Assessment" = $false
        "Compliance Manager Contribution" = $false
        "Compliance Manager Reader" = $false
        "Compliance Search" = $false
        "Credential Reader" = $false
        "Credential Writer" = $false
        "Custodian" = $false
        "Data Classification Content Download" = $false
        "Data Classification Content Viewer" = $true
        "Data Classification Feedback Provider" = $false
        "Data Classification Feedback Reviewer" = $false
        "Data Classification List Viewer" = $true
        "Data Connector Admin" = $false
        "Data Governance Administrator" = $false
        "Data Investigation Management" = $false
        "Data Map Reader" = $false
        "Data Map Writer" = $false
        "Data Security Investigation Admin" = $false
        "Data Security Investigation Investigator" = $false
        "Data Security Investigation Reviewer" = $false
        "Data Security Viewer" = $false
        "DLP Compliance Management" = $false
        "Disposition Management" = $false
        "Exchange Administrator" = $false
        "Exact Data Match Upload Admin" = $false
        "Export" = $false
        "Hold" = $false
        "IB Compliance Management" = $false
        "Insights Reader" = $false
        "Insights Writer" = $false
        "Information Protection Admin" = $false
        "Information Protection Analyst" = $false
        "Information Protection Investigator" = $false
        "Information Protection Reader" = $true
        "Insider Risk Management Admin" = $false
        "Insider Risk Management Analysis" = $false
        "Insider Risk Management Approval" = $false
        "Insider Risk Management Audit" = $false
        "Insider Risk Management Investigation" = $false
        "Insider Risk Management Permanent contribution" = $false
        "Insider Risk Management Reports Administrator" = $false
        "Insider Risk Management Sessions" = $false
        "Insider Risk Management Temporary contribution" = $false
        "Knowledge Admin" = $false
        "License Usage Reader" = $false
        "Manage Alerts" = $false
        "Manage Review Set Tags" = $false
        "MyBaseOptions" = $false
        "Organization Configuration" = $false
        "Preview" = $false
        "Priority Cleanup Admin" = $false
        "Priority Cleanup Viewer" = $false
        "Privacy Management Admin" = $false
        "Privacy Management Analysis" = $false
        "Privacy Management Investigation" = $false
        "Privacy Management Permanent contribution" = $false
        "Privacy Management Temporary contribution" = $false
        "Privacy Management Viewer" = $false
        "Purview Copilot Workspace Contributor" = $false
        "Purview Domain Manager" = $false
        "Purview Evaluation Administrator" = $false
        "Quarantine" = $false
        "RecordManagement" = $false
        "Retention Management" = $false
        "Review" = $false
        "RMS Decrypt" = $false
        "Role Management" = $false
        "Scan Reader" = $false
        "Scan Writer" = $false
        "Scope Manager" = $false
        "Search And Purge" = $false
        "Security Administrator" = $false
        "Security Reader" = $false
        "Sensitivity Label Administrator" = $false
        "Sensitivity Label Reader" = $true
        "Service Assurance View" = $false
        "Source Reader" = $false
        "Source Writer" = $false
        "Subject Rights Request Admin" = $false
        "Subject Rights Request Approver" = $false
        "Supervisory Review Administrator" = $false
        "Tag Contributor" = $false
        "Tag Manager" = $false
        "Tag Reader" = $false
        "Tenant AllowBlockList Manager" = $false
        "View-Only Audit Logs" = $false
        "View-Only Case" = $false
        "View-Only DLP Compliance Management" = $false
        "View-Only Device Management" = $false
        "View-Only IB Compliance Management" = $false
        "View-Only Manage Alerts" = $false
        "View-Only Record Management" = $false
        "View-Only Recipients" = $false
        "View-Only Retention Management" = $true
    }
}

function Create-PurviewRoleGroup
{
	$rolesSelected = Get-RoleSelected
	Write-Host "`nThis script will create a role group with the following information:"
	Write-Host "* Role Group Name`t:" -NoNewLine
	Write-Host "`t'$RoleGroupName'" -ForeGroundColor DarkBlue
	foreach ($role in $rolesSelected.Keys)
	{
        if ($rolesSelected[$role]) 
		{
            Write-Host "* Role selected`t`t:" -NoNewLine
			Write-Host "`t'$role'" -ForeGroundColor Green
        }
    }
	
	Write-Host "`nPress any key to continue..." -ForegroundColor DarkYellow
	Write-Host "(Alternatively, press Ctrl+C to exit and make any necessary changes.)"
	$key = ([System.Console]::ReadKey($true)) | Out-Null
	
	$existingGroup = Get-RoleGroup | Where-Object {$_.Name -eq $RoleGroupName}
	$rolesToAssign = $rolesSelected.GetEnumerator() | Where-Object { $_.Value } | ForEach-Object { $_.Key }
	if ($existingGroup)
	{
        Write-Host "`nRole group '$RoleGroupName' already exists. Exiting.`n"
        exit
    }else
	{
		Write-Host "`nCreating role group '$RoleGroupName'..."
		New-RoleGroup -Name "$RoleGroupName" -DisplayName "$RoleGroupName" -Roles $rolesToAssign -Description "Role group for '$RoleGroupName' Purview tasks" -ErrorAction Stop |Out-Null
	}
	
	Write-Host "Role group '$RoleGroupName' created and roles assigned successfully.`n`n"
}

function MainScript
{
	cls
	ScriptVariables
	Write-Host "`nTo run this script, you must have sufficient permissions to create role groups." 
	Write-Host "It is recommended to have the Compliance Administrator role for optimal execution."
	Write-Host "`nConnecting to Purview..."
	Write-Host "Please check your browser.`n"
	Connect-IPPSSession -UseRPSSession:$false -ShowBanner:$false
	Create-PurviewRoleGroup	
}

MainScript
```

<br><br>
