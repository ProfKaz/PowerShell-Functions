# Function to hash the password

You can find a simple exercise in this [Hash-Unhash script](/Samples/General/Hash-UnHash.md) where we are able to create a [configuration file](/Lego/CreateConfigFile.md) that is used to store a password, the function read the value in the Json file and update the value `MyPassword` with the Unhashed password and change the attribute `HashedPassword`to `False` this attrbute works as a flag.

You can check the function [UnHashCredentials](/Lego/HashCredentials.md) to work with hashed passwords.

To use this function you need to pass the hashed password and the function return the value unhashed, something like this:

```powershell
if ($HashedPassword -eq "True")
		{
			$UserPassword = UnHashCredentials $UserPassword
		}
```

Now you can start using this function to UnHash your password.

```powershell
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
```
<br><br>
