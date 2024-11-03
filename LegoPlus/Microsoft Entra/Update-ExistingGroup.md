# Function to update groups in Microsoft Entra

Using Microsoft Graph API, this function permit to update an existing group under Microsoft Entra using the Group ID as identity, adding users from an Array that contains Users Ids.

```powershell
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
```
<br><br>
