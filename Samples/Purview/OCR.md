# Script to use Azure AI services for OCR

Today, Azure AI services provide powerful OCR capabilities for processing images and PDFs. In this exercise, I am using version 3.2, which supports only image-based OCR. However, version 4, currently in preview, introduces the ability to work directly with PDFs, offering even greater flexibility.  

```powershell
# Configuration

function ScriptVariables
{
	$script:AzureEndpoint = "https://<YourAzureAIService>.cognitiveservices.azure.com/vision/v3.2/read/analyze"
	$script:ApiKey = "<Your-AzureAI-Key>"
	$script:FolderPath = "C:\Purview\PurviewOCR\Images"
	$script:date = (Get-Date).ToString("yyyy-MM-dd HHmm")
}

# Function: Get all image files from the folder
function Get-ImageFiles([string]$FolderPath)
{
    Write-Host "Reading image files from folder: $FolderPath"
    Get-ChildItem -Path $FolderPath -File | Where-Object { $_.Extension -in ".jpg", ".png", ".jpeg", ".bmp" }
}

# Function: Send image to Azure OCR and return Operation-Location
function Invoke-OCR([string]$ImagePath)
{
    Write-Host "Sending image for OCR: $ImagePath"

    $imageBytes = [System.IO.File]::ReadAllBytes($ImagePath)

    $headers = @{
        "Ocp-Apim-Subscription-Key" = $ApiKey
        "Content-Type"              = "application/octet-stream"
    }

    try {
        $response = Invoke-WebRequest -Uri $AzureEndpoint -Method Post -Headers $headers -Body $imageBytes -ErrorAction Stop
        if ($response.Headers["Operation-Location"]) {
            Write-Host "Operation-Location received for $ImagePath"
            return $response.Headers["Operation-Location"]
        } else {
            Write-Error "No Operation-Location found for $ImagePath"
            return $null
        }
    } catch {
        Write-Error "Error sending image: $_"
        return $null
    }
}

# Function: Poll for OCR result
function Get-OCRResult([string]$OperationLocation)
{
    Write-Host "Polling for OCR result from: $OperationLocation"

    $headers = @{
        "Ocp-Apim-Subscription-Key" = $ApiKey
    }

    $status = "running"
    while ($status -eq "running") {
        try {
            $response = Invoke-RestMethod -Uri $OperationLocation -Headers $headers -Method Get -ErrorAction Stop
            $status = $response.status
            if ($status -eq "succeeded") {
                Write-Host "OCR processing succeeded.`n" -ForeGroundColor Green
                return $response.analyzeResult.readResults
            } elseif ($status -eq "failed") {
                Write-Error "OCR processing failed.`n" -ForeGroundColor Red
                return $null
            }
        } catch {
            Write-Error "Error fetching OCR results: $_"
            return $null
        }
        Start-Sleep -Seconds 2
    }
}

# Function: Process all images in the folder
function Process-ImagesForOCR([string]$FolderPath, [string]$OutputFile)
{
    $files = Get-ImageFiles -FolderPath $FolderPath
    if ($files.Count -eq 0) {
        Write-Error "No image files found in folder: $FolderPath"
        return
    }
	Write-Host "`nTotal images found :`t" -NoNewline
	Write-Host $files.Count -ForeGroundColor Green

    $results = @()
    foreach ($file in $files) {
        Write-Host "`nProcessing file: $($file.FullName)"

        $operationLocation = Invoke-OCR -ImagePath $file.FullName
        if ($operationLocation) {
            $ocrResult = Get-OCRResult -OperationLocation $operationLocation
            if ($ocrResult) {
                $results += [PSCustomObject]@{
                    FileName = $file.Name
                    OCRResult = $ocrResult
                }
            }
        }
    }

    # Save results to JSON
    if ($results.Count -gt 0) {
        $results | ConvertTo-Json -Depth 10 | Set-Content -Path $OutputFile
        Write-Host "`nOCR processing complete. Results saved to " -NoNewline
		Write-Host "$OutputFile." -ForeGroundColor Cyan
    } else {
        Write-Error "No OCR results to save."
    }
}

# Function: Filter and Format OCR Results
function Filter-OCRResults([string]$InputFile, [string]$OutputFile)
{
    Write-Host "Filtering OCR results from: $InputFile"

    # Load OCR JSON Data
    $ocrData = Get-Content -Path $InputFile | ConvertFrom-Json

    # Initialize Results Array
    $filteredResults = @()

    foreach ($entry in $ocrData) {
        $fileName = $entry.FileName
        $ocrLines = $entry.OCRResult.lines

        foreach ($line in $ocrLines) {
            $text = $line.text
            $name = if ($line.PSObject.Properties["appearance"] -and $line.appearance.PSObject.Properties["style"] -and $line.appearance.style.PSObject.Properties["name"]) {
                $line.appearance.style.name
            } else {
                "N/A"
            }
            $confidence = if ($line.PSObject.Properties["appearance"] -and $line.appearance.PSObject.Properties["style"] -and $line.appearance.style.PSObject.Properties["confidence"]) {
                $line.appearance.style.confidence
            } else {
                "N/A"
            }

            $filteredResults += [PSCustomObject]@{
                FileName   = $fileName
                Text       = $text
                Name       = $name
                Confidence = $confidence
            }
        }
    }

    # Save Filtered Results to JSON
    if ($filteredResults.Count -gt 0) {
        $filteredResults | ConvertTo-Json -Depth 10 | Set-Content -Path $OutputFile
        Write-Host "Filtered OCR results saved to $OutputFile."
    } else {
        Write-Error "No filtered results to save."
    }
}

function MainFunction
{
	#Welcome header
	cls
	Clear-Host
	
	Write-Host "`n`n----------------------------------------------------------------------------------------"
	Write-Host "`nWelcome to OCR script!" -ForegroundColor Green
	Write-Host "This script will permit to request OCR Services."
	Write-Host "`n----------------------------------------------------------------------------------------" 
	
	##Show variables set
	Write-Host "OCR services set to`t:" -NoNewline
	Write-Host "`t$AzureEndpoint" -ForegroundColor Green
	Write-Host "Image folder set to`t:" -NoNewline
	Write-Host "`t$FolderPath" -ForegroundColor Green
	Write-Host "Raw export set to`t:" -NoNewline
	Write-Host "`t$OutputFile" -ForegroundColor Green
	Write-Host "Filtered export set to`t:" -NoNewline
	Write-Host "`t$FilteredFile" -ForegroundColor Green
	Write-Host "`nCalling script..."
	
	$OutputFile = "C:\Purview\PurviewOCR\OCRResults "+$date+".json"
	$FilteredFile = "C:\Purview\PurviewOCR\FilteredOCRResults "+$date+".json" # New filtered JSON file
	
	# Execute the OCR Process
	Process-ImagesForOCR -FolderPath $FolderPath -OutputFile $OutputFile
	
	# Filter and Export the Results
	Filter-OCRResults -InputFile $OutputFile -OutputFile $FilteredFile
}

ScriptVariables
MainFunction
```
