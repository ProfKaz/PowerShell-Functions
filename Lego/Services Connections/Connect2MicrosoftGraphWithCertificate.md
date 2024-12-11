# Function to connect to Microsoft Graph using a certificate using invoke method

To connect to Microsoft Graph API exist 3 common methods:
- Through cmdlet `Connect-IPPSSession`
- Through Invoke using a Secret Key
- Theough Invoke using a certificate

This function show how to reach the 3rd option using a certificate, this certificate needs to be installed locally and added in a [Service Principal](../CreateNewEntraApp.md) previously created.
The script uses the variable `$CertThumbprint` that needs to be previosuly set, through a [general variable](../ScriptVariables.md) or read it from a [Configuration File](../CreateConfigFile.md).
<br>

```powershell
function Connect2MicrosoftGraph
{
	Write-Output "Testing connection using certificate..."
	try {
		# Retrieve the certificate from CurrentUser\My
		$Certificate = Get-Item -Path Cert:\CurrentUser\My\$CertThumbprint
		if (-not $Certificate) {
			throw "Certificate with thumbprint $CertThumbprint not found in CurrentUser store."
		}

		# Ensure the certificate has a private key
		if (-not $Certificate.HasPrivateKey) {
			throw "The certificate does not have an associated private key."
		}
		Write-Output "Certificate found and has a private key."

		# Step 1: Create JWT client assertion
		Write-Output "Creating JWT client assertion..."
		$JwtHeader = @{
			#alg: Algorithm
			#typ: Type
			#x5t: X.509 Thumbprint
			alg = "RS256"
			typ = "JWT"
			x5t = [Convert]::ToBase64String([System.Convert]::FromHexString($CertThumbprint))
		} | ConvertTo-Json -Depth 10 -Compress

		$JwtPayload = @{
			#aud: Audience
			#iss: Issuer
			#sub: Subject
			#jti: JWT ID
			#nbf: Not Before (time in Unix seconds)
			#exp: Expiration (time in Unix seconds)
			aud = $TokenEndpoint
			iss = $ClientId
			sub = $ClientId
			jti = [guid]::NewGuid().ToString()
			nbf = [int][System.DateTimeOffset]::UtcNow.AddMinutes(-5).ToUnixTimeSeconds()
			exp = [int][System.DateTimeOffset]::UtcNow.AddMinutes(55).ToUnixTimeSeconds()
		} | ConvertTo-Json -Depth 10 -Compress

		$EncodedHeader = [Convert]::ToBase64String([System.Text.Encoding]::UTF8.GetBytes($JwtHeader))
		$EncodedPayload = [Convert]::ToBase64String([System.Text.Encoding]::UTF8.GetBytes($JwtPayload))

		# Sign the JWT with the certificate's private key
		$CryptoProvider = [System.Security.Cryptography.X509Certificates.RSACertificateExtensions]::GetRSAPrivateKey($Certificate)
		$DataToSign = [System.Text.Encoding]::UTF8.GetBytes("$EncodedHeader.$EncodedPayload")
		$SignatureBytes = $CryptoProvider.SignData($DataToSign, [System.Security.Cryptography.HashAlgorithmName]::SHA256, [System.Security.Cryptography.RSASignaturePadding]::Pkcs1)
		$Signature = [Convert]::ToBase64String($SignatureBytes)

		$ClientAssertion = "$EncodedHeader.$EncodedPayload.$Signature"

		# Step 2: Acquire Access Token using the client assertion
		Write-Output "Acquiring Access Token using client assertion..."
		$TokenResponse = Invoke-RestMethod -Method Post -Uri $TokenEndpoint -ContentType "application/x-www-form-urlencoded" -Body @{
			client_id = $ClientId
			client_assertion = $ClientAssertion
			client_assertion_type = "urn:ietf:params:oauth:client-assertion-type:jwt-bearer"
			grant_type = "client_credentials"
			scope = $Scope
		}
		Write-Host "Access Token Acquired... " -NoNewLine
		Write-Host "`tSuccessfully:" -ForeGroundColor Green
		#Write-Output $TokenResponse.access_token
		$script:AccessToken = $TokenResponse.access_token
	} catch {
		Write-Output "Error during connection test: $($_.Exception.Message)"
		if ($_.Exception.Response -ne $null) {
			try {
				$ErrorContent = $_.Exception.Response.Content.ReadAsStringAsync().Result
				Write-Output "HTTP Response Content: $ErrorContent"
			} catch {
				Write-Output "Unable to capture response content: $($_.Exception.Message)"
			}
		}
		throw $_
	}
}
```


<br><br>
