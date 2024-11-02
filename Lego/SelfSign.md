# Function to sign the scripts with a self-sign certificate

This is a simple version of a function that search for certificates preciously installed used to sign code and uses the latest one available.

To ensure compatibility across different OS languages, you can use the following options, replacing the language-specific filter with direct attributes:

- For **Code Signing** certificates:
  
  ```powershell
  $certificates = @(Get-ChildItem Cert:\CurrentUser\My -CodeSigningCert | Select-Object Thumbprint)
  ```
  Instead of:
   ```powershell
  $certificates = @(Get-ChildItem Cert:\CurrentUser\My | Where-Object {$_.EnhancedKeyUsageList -like "*Code Signing*"} | Select-Object Thumbprint)
  ```

- For **SSL Server Authentication** certificates:
  
  ```powershell
  $certificates = @(Get-ChildItem Cert:\CurrentUser\My -SSLServerAuthentication | Select-Object Thumbprint)
  ```
  Instead of:
   ```powershell
  $certificates = @(Get-ChildItem Cert:\CurrentUser\My | Where-Object {$_.EnhancedKeyUsageList -like "*Client Authentication"}| Select-Object Thumbprint)
  ```
>[!IMPORTANT]
>By default, the function checks certificates in `Cert:\CurrentUser\My`, which lists certificates installed for the current user. However, certificates can also be installed at the machine level, in which case the location should be `Cert:\LocalMachine\My`.

>[!NOTES]
>In the code we can see that the function will sign the file called `MyMainScript.ps1`, nevertheless, we can use wildcards like as `*.ps1` to get all the script files to be signed.

```powershell
function SelfSign
{
	$certificates = @(Get-ChildItem Cert:\CurrentUser\My | Where-Object {$_.EnhancedKeyUsageList -like "*Code Signing*"}| Sort-Object NotBefore -Descending | Select-Object Subject, Thumbprint, NotBefore, NotAfter)
	$cert = Get-ChildItem Cert:\CurrentUser\My -CodeSigningCert | Where-Object {$_.Thumbprint -eq $certificates[0].Thumbprint}
	$MainScript = Get-ChildItem -Path .\MyMainScript.ps1
	Set-AuthenticodeSignature -FilePath ".\$($MainScript.Name)" -Certificate $cert
}
```
