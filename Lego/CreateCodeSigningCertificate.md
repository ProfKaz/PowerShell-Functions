# Function to create a self-sign certificate for Code Signin

This function normally is called from another function to create a local certificate that is installed under Current User (`Cert:\CurrentUser\My"`) profile, nevertheless, this function can be changed to create the certificates as Local Machine (`Cert:\LocalMachine\My`).

```powershell
function CreateCodeSigningCertificate
{
	#CMDLET to create certificate
	$ScriptingCert = New-SelfSignedCertificate -Subject "CN=Self-Sign Code Signing Cert" -Type "CodeSigning" -CertStoreLocation "Cert:\CurrentUser\My" -HashAlgorithm "sha256"
		
	### Add Self Signed certificate as a trusted publisher
		
		# Add the self-signed Authenticode certificate to the computer's root certificate store.
		## Create an object to represent the CurrentUser\Root certificate store.
		$rootStore = [System.Security.Cryptography.X509Certificates.X509Store]::new("Root","CurrentUser")
		## Open the root certificate store for reading and writing.
		$rootStore.Open("ReadWrite")
		## Add the certificate stored in the $authenticode variable.
		$rootStore.Add($ScriptingCert)
		## Close the root certificate store.
		$rootStore.Close()
			 
		# Add the self-signed Authenticode certificate to the computer's trusted publishers certificate store.
		## Create an object to represent the CurrentUser\TrustedPublisher certificate store.
		$publisherStore = [System.Security.Cryptography.X509Certificates.X509Store]::new("TrustedPublisher","CurrentUser")
		## Open the TrustedPublisher certificate store for reading and writing.
		$publisherStore.Open("ReadWrite")
		## Add the certificate stored in the $authenticode variable.
		$publisherStore.Add($ScriptingCert)
		## Close the TrustedPublisher certificate store.
		$publisherStore.Close()	
}
```
