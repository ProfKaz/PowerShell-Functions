# Function to validate if a critical file is open by another application

While working with scripts that read or write configuration files during execution, errors often occur if a file is open in another application. This function detects if a file is open and pauses script execution until the file is closed, ensuring smooth processing.

```powershell
function ValidateIfCSVisOpenByAnotherApp
{
    # Keep checking until the file is available
    while ($true) {
        try {
            # Try to open the file with exclusive access
            $fileStream = [System.IO.File]::Open($ConfigurationFile, [System.IO.FileMode]::Open, [System.IO.FileAccess]::Read, [System.IO.FileShare]::None)
            $fileStream.Close()
            Write-Host "File is now available." -ForegroundColor Green
            break
        }
        catch {
            # If the file is locked, show a blinking message
            Write-Host "`r[WARNING] The file is currently open by another application. Please close it to proceed..." -ForegroundColor Red -NoNewline
            Start-Sleep -Milliseconds 1000
            Write-Host "`r                                                    " -NoNewline
            Start-Sleep -Milliseconds 500
        }
    }
}
```
<br><br>
