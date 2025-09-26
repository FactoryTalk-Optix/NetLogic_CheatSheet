# FactoryTalk Optix Studio CLI

## Export an application to be loaded in an Optix Panel via USB

This script leverages the FactoryTalk Optix Studio CLI to export an application and prepare it to be imported in an Optix Panel via USB.

> [!TIP]
> The `LoadApplicationWidget` should be used to load the application in the Optix Panel, this script only prepares the ZIP file with the correct structure and encryption.

```powershell
$projectPath = "D:\temp\NewHMIProject12\NewHMIProject12.optix"
$optixIdePath = "C:\Program Files\Rockwell Automation\FactoryTalk Optix\Studio 1.6.2.36\FTOptixStudio.exe"
$7zipPath = "C:\Program Files\7-Zip\7z.exe"
$outputPath = "C:\Users\LabUser\Downloads\"
$platform = "Yocto_arm64"
$optixPanelPassword = "YourOptixPanelAdminPassword"

# Create a temporary directory for the export process
Write-Host "Creating temporary directory for export..."
$tempDir = New-Item -ItemType Directory -Path "$env:TEMP\OptixExportTemp" -Force
Write-Host "Temporary directory created at $tempDir"

# Export the project using Optix IDE
Write-Host "Exporting project using Optix IDE..."
Start-Process -FilePath "$optixIdePath" -ArgumentList "export `"$projectPath`" --platform=`"$platform`" --location=`"$tempDir`"" -Wait

# Update the .optix file with the EncryptionMode
Write-Host "Updating .optix file with EncryptionMode..."
$optixFilePath = Get-ChildItem -Path $tempDir -Recurse -Filter "*.optix" | Select-Object -First 1
if ($null -eq $optixFilePath) {
	Write-Error "No .optix file found in the exported directory."
	Write-Host "Searching for all files in temp directory for debugging:"
	Get-ChildItem -Path $tempDir -Recurse | ForEach-Object { Write-Host $_.FullName }
	exit 1
}
Write-Host "Found .optix file at: $($optixFilePath.FullName)"

# Extract the CoreVersion from the .optix YAML file and add encryption settings
$optixFileContent = Get-Content -Path $optixFilePath.FullName
$coreVersionLine = $optixFileContent | Where-Object { $_ -match "CoreVersion:" }
if ($null -eq $coreVersionLine) {
	Write-Error "CoreVersion not found in the .optix file."
	exit 1
}

$coreVersion = $coreVersionLine -replace "CoreVersion:\s*", ""
Write-Host "CoreVersion found: $coreVersion"

# Add encryption settings after CoreVersion line
$updatedContent = @()
foreach ($line in $optixFileContent) {
    $updatedContent += $line
    if ($line -match "CoreVersion:") {
        $updatedContent += " EncryptionVersion: 1"
        $updatedContent += " EncryptionMode: YAMLFiles"
        Write-Host "Added encryption settings after CoreVersion: $coreVersion"
    }
}

# Write the updated content back to the .optix file
$updatedContent | Set-Content -Path $optixFilePath.FullName

# Write the content of the XML file
Write-Host "Creating DeviceConfiguration.xml..."
$xmlFilePath = Join-Path -Path $tempDir -ChildPath "DeviceConfiguration.xml"
$xmlContent = "<?xml version=""1.0"" encoding=""utf-8""?>
<DeviceConfiguration>
	<Actions>
		<ImportFTOptixApplication EncryptProject=""true""/>
	</Actions>
</DeviceConfiguration>"

try {
    $xmlContent | Out-File -FilePath $xmlFilePath -Encoding Ascii
    Write-Host "DeviceConfiguration.xml created successfully at: $xmlFilePath"
} catch {
    Write-Error "Failed to create XML file: $($_.Exception.Message)"
    exit 1
}

# Create a zip archive of the exported project
Write-Host "Creating ZIP archive..."
$outputZipPath = Join-Path -Path $outputPath -ChildPath "FTOptixApplication.z"

# Compress it to ZIP file with Fast method and AES-256 encryption
Write-Host "Compressing to ZIP with AES-256 encryption..."
& "$7zipPath" a -tzip -mx=3 "-p$optixPanelPassword" "$outputZipPath" "$tempDir\*"

# Check if the compression was successful
if ($LASTEXITCODE -ne 0) {
    Write-Error "Failed to create ZIP archive."
    exit 1
}

# Clean up the temporary directory
Remove-Item -Path $tempDir -Recurse -Force

Write-Host "Export completed. The ZIP file is located at $outputZipPath"
```
