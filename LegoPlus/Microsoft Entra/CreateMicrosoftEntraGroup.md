# Function to create groups in Microsoft Entra

Using Microsoft Graph API, this function permit to update an create a `Security` or `Microsoft365` group under Microsoft Entra using a new name set in a CSV file, adding users from an Array that contains Users Ids.

> [!WARNING]
> This function uses `Microsoft Graph API`

```powershell
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
}

```
<br><br>
