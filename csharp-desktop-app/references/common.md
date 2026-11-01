# Shared files (identical for WPF and WinForms)

Replace `AppName` everywhere (namespace, folder, strings, paths). Pinned package versions below are known-good; if `dotnet` is available prefer `dotnet add package <name>` which resolves the latest stable.

Always add `using System.IO;` explicitly (WPF does not import it implicitly; it is harmless in WinForms).

## Directory.Build.props (solution root)
```xml
<Project>
  <PropertyGroup>
    <LangVersion>latest</LangVersion>
    <AnalysisLevel>latest</AnalysisLevel>
    <WarningsAsErrors>$(WarningsAsErrors);nullable</WarningsAsErrors>
    <Deterministic>true</Deterministic>
  </PropertyGroup>
</Project>
```

## .editorconfig (solution root)
```ini
root = true

[*]
charset = utf-8
indent_style = space
indent_size = 4
insert_final_newline = true
trim_trailing_whitespace = true

[*.{xaml,csproj,props,json,yml}]
indent_size = 2

[*.cs]
csharp_style_namespace_declarations = file_scoped:suggestion
csharp_using_directive_placement = outside_namespace:suggestion
dotnet_sort_system_directives_first = true
```

## .gitignore (solution root)
```
bin/
obj/
publish/
installer/
.vs/
*.user
```

## Models/Item.cs
```csharp
namespace AppName;

public class Item
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public string Name { get; set; } = "";
}
```
Adapt fields to the requested app. Keep models plain (no UI types) so they can be linked into the test project.

## Services/AppPaths.cs
```csharp
using System.IO;
namespace AppName;

public static class AppPaths
{
    public const string AppName = "AppName";

    public static string Root { get; } = Path.Combine(
        Environment.GetFolderPath(Environment.SpecialFolder.ApplicationData), AppName);

    public static string DataFile => Path.Combine(Root, "data.json");
    public static string SettingsFile => Path.Combine(Root, "settings.json");
    public static string LogsDir => Path.Combine(Root, "logs");
}
```

## Services/DataStore.cs
Atomic save (temp file then move) and corrupt-file recovery: a damaged file is copied aside, never silently overwritten.
```csharp
using System.IO;
using System.Text.Json;
using Serilog;
namespace AppName;

public class DataStore
{
    private static readonly JsonSerializerOptions JsonOptions = new() { WriteIndented = true };
    private readonly string _path;

    /// <summary>Set when a corrupt data file was backed up during the last load.</summary>
    public string? RecoveredFrom { get; private set; }

    public DataStore(string? path = null) => _path = path ?? AppPaths.DataFile;

    public async Task<List<Item>> LoadAsync()
    {
        RecoveredFrom = null;
        if (!File.Exists(_path)) return new();
        try
        {
            await using var stream = File.OpenRead(_path);
            return await JsonSerializer.DeserializeAsync<List<Item>>(stream) ?? new();
        }
        catch (JsonException ex)
        {
            var backup = $"{_path}.corrupt-{DateTime.Now:yyyyMMdd-HHmmss}";
            File.Copy(_path, backup, overwrite: true);
            RecoveredFrom = backup;
            Log.Error(ex, "Data file is corrupt; backed up to {Backup}", backup);
            return new();
        }
    }

    public async Task SaveAsync(List<Item> items)
    {
        Directory.CreateDirectory(Path.GetDirectoryName(Path.GetFullPath(_path))!);
        var tmp = _path + ".tmp";
        await using (var stream = File.Create(tmp))
        {
            await JsonSerializer.SerializeAsync(stream, items, JsonOptions);
        }
        File.Move(tmp, _path, overwrite: true);
    }
}
```
`IOException`/`UnauthorizedAccessException` are deliberately not caught here: the caller (UI layer) logs and shows a friendly message.

## Services/AppSettings.cs
```csharp
using System.IO;
using System.Text.Json;
using Serilog;
namespace AppName;

public enum AppTheme { Light, Dark }

public class AppSettings
{
    private static readonly JsonSerializerOptions JsonOptions = new() { WriteIndented = true };

    /// <summary>First run follows the Windows setting; afterwards the saved choice wins.</summary>
    public AppTheme Theme { get; set; } = NativeTheme.DetectSystemTheme();

    public static AppSettings Load()
    {
        try
        {
            if (File.Exists(AppPaths.SettingsFile))
                return JsonSerializer.Deserialize<AppSettings>(File.ReadAllText(AppPaths.SettingsFile)) ?? new();
        }
        catch (Exception ex)
        {
            Log.Warning(ex, "Could not read settings; using defaults");
        }
        return new();
    }

    public void Save()
    {
        try
        {
            Directory.CreateDirectory(AppPaths.Root);
            File.WriteAllText(AppPaths.SettingsFile, JsonSerializer.Serialize(this, JsonOptions));
        }
        catch (Exception ex)
        {
            Log.Warning(ex, "Could not save settings");
        }
    }
}
```
Settings failures are non-critical, so they log a warning and do not interrupt the user.

## Services/AppLogger.cs
```csharp
using System.IO;
using System.Reflection;
using Serilog;
namespace AppName;

public static class AppLogger
{
    public static void Init()
    {
        const string template = "{Timestamp:yyyy-MM-dd HH:mm:ss.fff} [{Level:u3}] {Message:lj}{NewLine}{Exception}";
        var config = new LoggerConfiguration()
            .WriteTo.File(
                Path.Combine(AppPaths.LogsDir, "log-.txt"),
                rollingInterval: RollingInterval.Day,
                retainedFileCountLimit: 14,
                outputTemplate: template);
#if DEBUG
        config = config.MinimumLevel.Debug();
#else
        config = config.MinimumLevel.Information();
#endif
        Log.Logger = config.CreateLogger();

        var version = Assembly.GetExecutingAssembly().GetName().Version;
        Log.Information("{App} {Version} started", AppPaths.AppName, version);
    }
}
```
Call `AppLogger.Init()` first thing at startup and `Log.CloseAndFlush()` on exit. Logs: `%AppData%\AppName\logs\`. Never log personal data or file contents.

## Services/NativeTheme.cs
System theme detection and dark title bar (the title bar is drawn by Windows, so XAML/WinForms colors alone leave it white).
```csharp
using System.Runtime.InteropServices;
using Microsoft.Win32;
namespace AppName;

public static class NativeTheme
{
    [DllImport("dwmapi.dll")]
    private static extern int DwmSetWindowAttribute(IntPtr hwnd, int attribute, ref int value, int size);

    public static AppTheme DetectSystemTheme()
    {
        try
        {
            using var key = Registry.CurrentUser.OpenSubKey(
                @"Software\Microsoft\Windows\CurrentVersion\Themes\Personalize");
            return key?.GetValue("AppsUseLightTheme") is int v && v == 0 ? AppTheme.Dark : AppTheme.Light;
        }
        catch
        {
            return AppTheme.Light; // detection is best-effort; Light is a safe default
        }
    }

    public static void SetDarkTitleBar(IntPtr hwnd, bool dark)
    {
        if (hwnd == IntPtr.Zero) return;
        var value = dark ? 1 : 0;
        // Attribute 20 = Windows 10 20H1+/11; 19 = older Windows 10 builds.
        if (DwmSetWindowAttribute(hwnd, 20, ref value, sizeof(int)) != 0)
            DwmSetWindowAttribute(hwnd, 19, ref value, sizeof(int));
    }
}
```

## Strings.cs
All user-visible text lives here. Change the language by editing this one file. Identifiers stay English; values use the UI language. For an Arabic UI set `IsRtl = true` and translate every value.
```csharp
namespace AppName;

public static class Strings
{
    public const bool IsRtl = false;   // true for Arabic / Hebrew UI

    public const string AppTitle = "AppName";
    public const string Name = "Name";
    public const string Add = "Add";
    public const string Update = "Update";
    public const string Delete = "Delete";
    public const string ToggleTheme = "Light / Dark";

    public const string NameRequired = "Name is required.";
    public const string ConfirmDelete = "Delete this item?";
    public const string SaveFailed = "Could not save your data. Check that the disk is available and try again.";
    public const string LoadFailed = "Could not read your saved data.";
    public const string UnexpectedError = "Something went wrong. The problem was recorded in the log file.";

    public static string ItemsCount(int n) => $"{n} items";
    public static string DataRecovered(string path) =>
        $"The data file was damaged. A copy was kept at:\n{path}";
}
```
Arabic example values: `Add = "إضافة"`, `Delete = "حذف"`, `NameRequired = "الاسم مطلوب."`, `ConfirmDelete = "هل تريد حذف هذا العنصر؟"`, `ItemsCount(n) => $"{n} عنصر"`.

## Error-handling rules
- Never write an empty `catch` or `catch { }` (the only tolerated exception is best-effort detection such as `DetectSystemTheme`, with a comment saying why).
- In a catch: `Log.Error(ex, "<what failed>")`, then `Msg.Error(Strings.X)` with a friendly, actionable sentence. Never show `ex.Message` or stack traces to the user.
- Validation problems are not exceptions: show `Msg.Info`, focus the field, return.
- Global handlers (registered in `App.OnStartup` / `Program.Main`) are the last line of defense: log, show `Strings.UnexpectedError`, keep the app alive where safe. Always flush the log on exit.
- Persist the user's data first, then update the UI; if saving fails, tell the user and keep the UI consistent with what is on disk.

## RTL rules (Arabic / Hebrew UI)
1. Set `Strings.IsRtl = true`. WPF: `FlowDirection = RightToLeft` on the Window (inherited by children). WinForms: `RightToLeft = Yes` and `RightToLeftLayout = true` on the Form, which mirrors the whole window, so write the layout as normal LTR code and it flips correctly (a `Dock.Right` panel appears on the left).
2. Dialogs: always use `Msg` (it passes `RtlReading | RightAlign` so message boxes are mirrored).
3. LTR islands: inputs holding emails, URLs, file paths, IDs, codes, or phone numbers stay left-to-right (WPF `FlowDirection="LeftToRight"` on that TextBox; WinForms `RightToLeft = RightToLeft.No`).
4. Do not flip icons, logos, charts, or media controls. Arrows indicating direction (back/next) should mirror; give them `FlowDirection` accordingly.
5. Fonts: `Segoe UI` (covers Arabic) with `Tahoma` fallback. Do not set `xml:lang`/culture to Arabic just to get Arabic text; it can switch digits unexpectedly. Western digits (0-9) are the default unless the user asks for Arabic-Indic digits.
6. Mixed text: never build sentences by concatenating Arabic with English/numbers inline; use format strings in `Strings.cs`. If an English fragment ends a sentence and punctuation jumps to the wrong side, append `\u200F` (RLM) to the sentence.
7. Date/number formatting: use `CultureInfo.CurrentCulture` or an explicit culture; never hand-format with `/` or `,`.
8. Check: tab order follows reading order (right to left), column order in grids reads from the right, and nothing is clipped when the window is at `MinWidth`.
