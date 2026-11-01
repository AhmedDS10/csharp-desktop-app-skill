# Tests, publish, installer, README

Replace `AppName` everywhere. Skip this file's parts only when the user asked for no tests / no installer.

## Strategy
- The test project is plain `net8.0` (no Windows dependency), so `dotnet test` runs on any OS and in CI.
- It does not reference the WPF/WinForms project. It **links** the UI-free source files (`Item`, `DataStore`, `AppPaths`) so the real production code is tested.
- Test the data layer and any pure logic classes the app has (parsers, calculators, validators). Do not try to unit-test XAML or form code.
- If the app adds more UI-free classes in `Services/` or `Models/`, add them to the `<Compile Include=...>` list.

## AppName.Tests/AppName.Tests.csproj
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <IsPackable>false</IsPackable>
    <IsTestProject>true</IsTestProject>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.11.1" />
    <PackageReference Include="xunit" Version="2.9.2" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.8.2" />
    <PackageReference Include="Serilog" Version="4.0.2" />
  </ItemGroup>
  <ItemGroup>
    <Compile Include="..\AppName\Models\Item.cs" Link="Linked\Item.cs" />
    <Compile Include="..\AppName\Services\AppPaths.cs" Link="Linked\AppPaths.cs" />
    <Compile Include="..\AppName\Services\DataStore.cs" Link="Linked\DataStore.cs" />
  </ItemGroup>
</Project>
```

## AppName.Tests/DataStoreTests.cs
```csharp
using System.IO;
namespace AppName.Tests;

public class DataStoreTests : IDisposable
{
    private readonly string _dir = Path.Combine(Path.GetTempPath(), "AppNameTests-" + Guid.NewGuid());
    private string FilePath => Path.Combine(_dir, "data.json");

    public void Dispose()
    {
        if (Directory.Exists(_dir)) Directory.Delete(_dir, recursive: true);
    }

    [Fact]
    public async Task Load_ReturnsEmptyList_WhenFileMissing()
    {
        var store = new DataStore(FilePath);

        var items = await store.LoadAsync();

        Assert.Empty(items);
        Assert.Null(store.RecoveredFrom);
    }

    [Fact]
    public async Task SaveThenLoad_RoundTripsItems()
    {
        var store = new DataStore(FilePath);
        var saved = new List<Item> { new() { Name = "أول" }, new() { Name = "Second" } };

        await store.SaveAsync(saved);
        var loaded = await new DataStore(FilePath).LoadAsync();

        Assert.Equal(new[] { "أول", "Second" }, loaded.Select(i => i.Name));
        Assert.Equal(saved.Select(i => i.Id), loaded.Select(i => i.Id));
    }

    [Fact]
    public async Task Load_BacksUpCorruptFile_AndReturnsEmptyList()
    {
        Directory.CreateDirectory(_dir);
        await File.WriteAllTextAsync(FilePath, "{ this is not json");
        var store = new DataStore(FilePath);

        var items = await store.LoadAsync();

        Assert.Empty(items);
        Assert.NotNull(store.RecoveredFrom);
        Assert.True(File.Exists(store.RecoveredFrom));
    }

    [Fact]
    public async Task Save_LeavesNoTempFileBehind()
    {
        var store = new DataStore(FilePath);

        await store.SaveAsync(new List<Item> { new() { Name = "x" } });

        Assert.False(File.Exists(FilePath + ".tmp"));
        Assert.True(File.Exists(FilePath));
    }
}
```
For apps with their own logic classes, add one `*Tests.cs` per class: at least the happy path, an empty/invalid input, and one boundary case.

## publish.ps1 (solution root)
Runs the tests first, then publishes a single-file executable to `publish/`. Self-contained by default (no .NET install needed on the target PC; larger file). `-FrameworkDependent` produces a small file that needs the .NET 8 Desktop Runtime.
```powershell
param(
    [string]$Rid = "win-x64",
    [switch]$FrameworkDependent
)
$ErrorActionPreference = "Stop"
$root = $PSScriptRoot
$selfContained = if ($FrameworkDependent) { "false" } else { "true" }

dotnet test "$root\AppName.sln" -c Release
if ($LASTEXITCODE -ne 0) { throw "Tests failed - publish aborted." }

dotnet publish "$root\AppName\AppName.csproj" -c Release -r $Rid `
    --self-contained $selfContained `
    -p:PublishSingleFile=true `
    -p:IncludeNativeLibrariesForSelfExtract=true `
    -p:DebugType=None -p:DebugSymbols=false `
    -o "$root\publish"
if ($LASTEXITCODE -ne 0) { throw "Publish failed." }

Write-Host "Done: $root\publish\AppName.exe"
```
Never add `PublishTrimmed` (unsupported for WPF/WinForms). `PublishReadyToRun` is optional and speeds up startup at the cost of size.

## installer.iss (solution root; Inno Setup 6.3+)
Create it when the user asks for an installer or the app is meant for end users. Generate a fresh GUID for `AppId` (for example with `uuidgen` or PowerShell `[guid]::NewGuid()`) and keep the doubled opening brace.
```
#define AppName "AppName"
#define AppVersion "1.0.0"
#define AppExe "AppName.exe"

[Setup]
AppId={{<NEW-GUID>}
AppName={#AppName}
AppVersion={#AppVersion}
DefaultDirName={autopf}\{#AppName}
DefaultGroupName={#AppName}
OutputDir=installer
OutputBaseFilename={#AppName}-Setup-{#AppVersion}
Compression=lzma2
SolidCompression=yes
ArchitecturesInstallIn64BitMode=x64compatible
PrivilegesRequired=lowest
WizardStyle=modern

[Languages]
Name: "english"; MessagesFile: "compiler:Default.isl"
; Arabic wizard: download Arabic.isl (Inno Setup unofficial translations), place it
; next to this file, then uncomment the next line.
; Name: "arabic"; MessagesFile: "Arabic.isl"

[Tasks]
Name: "desktopicon"; Description: "{cm:CreateDesktopIcon}"; GroupDescription: "{cm:AdditionalIcons}"

[Files]
Source: "publish\{#AppExe}"; DestDir: "{app}"; Flags: ignoreversion

[Icons]
Name: "{group}\{#AppName}"; Filename: "{app}\{#AppExe}"
Name: "{autodesktop}\{#AppName}"; Filename: "{app}\{#AppExe}"; Tasks: desktopicon

[Run]
Filename: "{app}\{#AppExe}"; Description: "{cm:LaunchProgram,{#AppName}}"; Flags: nowait postinstall skipifsilent
```
Build order: `./publish.ps1` then `iscc installer.iss` (output in `installer/`). The installer is per-user (no admin rights). User data in `%AppData%\AppName` is left untouched on uninstall.

## README.md template (10 lines max)
```markdown
# AppName
One-line description.

- Run: `dotnet run --project AppName`
- Test: `dotnet test`
- Publish: `./publish.ps1` (single-file exe in `publish/`; add `-FrameworkDependent` for a small build)
- Installer (optional): `iscc installer.iss` after publishing (needs Inno Setup)
- Data: `%AppData%\AppName\data.json` | Settings: `settings.json` | Logs: `%AppData%\AppName\logs\`
- Requires Windows 10/11 and the .NET 8 SDK (to build) or nothing extra for the self-contained exe.
```

## AppName.sln fallback (only when `dotnet new sln` is unavailable)
Preferred: `dotnet new sln -n AppName` then `dotnet sln add AppName/AppName.csproj AppName.Tests/AppName.Tests.csproj`. Otherwise write this file (keep the GUIDs; they only need to be unique and consistent):
```
Microsoft Visual Studio Solution File, Format Version 12.00
# Visual Studio Version 17
VisualStudioVersion = 17.0.31903.59
MinimumVisualStudioVersion = 10.0.40219.1
Project("{9A19103F-16F7-4668-BE54-9A1E7A4F7556}") = "AppName", "AppName\AppName.csproj", "{6C1F2A10-4B7D-4E5A-9A3B-0D1E2F3A4B5C}"
EndProject
Project("{9A19103F-16F7-4668-BE54-9A1E7A4F7556}") = "AppName.Tests", "AppName.Tests\AppName.Tests.csproj", "{7D2A3B21-5C8E-4F6B-8B4C-1E2F3A4B5C6D}"
EndProject
Global
	GlobalSection(SolutionConfigurationPlatforms) = preSolution
		Debug|Any CPU = Debug|Any CPU
		Release|Any CPU = Release|Any CPU
	EndGlobalSection
	GlobalSection(ProjectConfigurationPlatforms) = postSolution
		{6C1F2A10-4B7D-4E5A-9A3B-0D1E2F3A4B5C}.Debug|Any CPU.ActiveCfg = Debug|Any CPU
		{6C1F2A10-4B7D-4E5A-9A3B-0D1E2F3A4B5C}.Debug|Any CPU.Build.0 = Debug|Any CPU
		{6C1F2A10-4B7D-4E5A-9A3B-0D1E2F3A4B5C}.Release|Any CPU.ActiveCfg = Release|Any CPU
		{6C1F2A10-4B7D-4E5A-9A3B-0D1E2F3A4B5C}.Release|Any CPU.Build.0 = Release|Any CPU
		{7D2A3B21-5C8E-4F6B-8B4C-1E2F3A4B5C6D}.Debug|Any CPU.ActiveCfg = Debug|Any CPU
		{7D2A3B21-5C8E-4F6B-8B4C-1E2F3A4B5C6D}.Debug|Any CPU.Build.0 = Debug|Any CPU
		{7D2A3B21-5C8E-4F6B-8B4C-1E2F3A4B5C6D}.Release|Any CPU.ActiveCfg = Release|Any CPU
		{7D2A3B21-5C8E-4F6B-8B4C-1E2F3A4B5C6D}.Release|Any CPU.Build.0 = Release|Any CPU
	EndGlobalSection
EndGlobal
```
