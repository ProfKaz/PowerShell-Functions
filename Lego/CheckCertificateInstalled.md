# Function to Check Installed Certificates

When working with APIs and Service Principals, the connection string often requires a Certificate Thumbprint. For this, the certificate must be installed locally. This function retrieves locally installed certificates and checks if any match the thumbprint stored in a configuration file, which is then used in the connection string.

By default, the function checks certificates in `Cert:\CurrentUser\My`, which lists certificates installed for the current user. However, certificates can also be installed at the machine level, in which case the location should be `Cert:\LocalMachine\My`.

Depending on your needs, you may encounter different types of certificates, such as those for SSL authentication or code signing. This function specifically searches for certificates associated with **Client Authentication**. However, this approach has a limitation if the operating system is in a language other than English.

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

Using these options avoids potential language-related issues, ensuring the function works consistently across various OS language settings.
<br><br>

```powershell
function CheckCertificateInstalled($thumbprint)
{
	$var = "False"
	$certificates = @(Get-ChildItem Cert:\CurrentUser\My | Where-Object {$_.EnhancedKeyUsageList -like "*Client Authentication*"}| Select-Object Thumbprint) 
	#$thumbprint -in $certificates
	foreach($certificate in $certificates)
	{
		if($thumbprint -in $certificate.Thumbprint)
		{
			$var = "True"
		}
	}
	if($var -eq "True")
	{
		Write-Host "Certificate validation..." -NoNewLine
		Write-Host "`t`t`t`tPassed!" -ForegroundColor Green
		return $var
	}else
	{
		Write-Host "`nCertificate installed on this machine is missing!!!" -ForeGroundColor Yellow
		Write-Host "To execute this script unattended a certificate needs to be installed, the same used under Microsoft Entra App"
		Start-Sleep -s 1
		return $var
	}
}
```
