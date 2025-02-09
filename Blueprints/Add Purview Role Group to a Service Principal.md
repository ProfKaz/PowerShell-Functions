# How to add a Service Principal (Microsoft Entra ID App) to a Purview Role Group

By default, the Web Interface does not allow a Service Principal to be directly assigned to a Purview Role Group. It is not possible to select the Service Principal as a user, add it to a group, and then assign that group to the Role Group—this approach does not work.

The only way to achieve this is through PowerShell. You need to connect to Purview and create a new Service Principal under that workload, using the same IDs from Microsoft Entra ID. This process effectively generates a new Service Principal that remains linked to the Microsoft Entra instance.

The steps to achieve this are:

```powershell
$AppClientID = "<Set your App ID>"

# Here you need to connect with a user with the right permissions, or a Global Admin account
Connect-MgGraph -Scopes AppRoleAssignment.ReadWrite.All,Application.Read.All -NoWelcome

#We are adding all the values from the Microsoft Entra Service Principal into a Variable
$MicrosoftEntraApp = Get-MgServicePrincipal -Filter "AppId eq '$AppClientID'"

#On the same session, under the same console we need to connect using a Compliance Administrator account
Connect-IPPSSession -UseRPSSession:$false -ShowBanner:$false

#Now we will create the "New" Service Principal under Purview using the same App ID and Object ID from MIcrosoft Entra ID App
New-ServicePrincipal -AppId $MicrosoftEntraApp.AppId -ObjectId $MicrosoftEntraApp.Id -DisplayName "SP for Data Explorer PowerShell"

#A new variable is created getting the values from the "New" Service Principal
$SP = Get-ServicePrincipal -Identity $MicrosoftEntraApp.AppId

#Finally, we can assign the Purview Role to the Service Principal, in this case the role assigned is "Content Explorer Content Viewer"
Add-RoleGroupMember -Identity "ContentExplorerContentViewer" -Member $SP.Identity
```
<br><br>
