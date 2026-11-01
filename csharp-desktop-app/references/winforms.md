# WinForms reference (code-behind)

Shared files (`AppPaths`, `DataStore`, `AppSettings`, `AppLogger`, `NativeTheme`, `Strings`, `Item`) are in `common.md`. This file has only the WinForms-specific parts. Replace `AppName` everywhere.

Write the form entirely in code (no Designer file) so the project is self-contained and compiles without Visual Studio. Build controls in a `BuildUi()` method.

## AppName/AppName.csproj
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <TargetFramework>net8.0-windows</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <UseWindowsForms>true</UseWindowsForms>
    <ApplicationHighDpiMode>SystemAware</ApplicationHighDpiMode>
    <ForceDesignerDpiUnaware>true</ForceDesignerDpiUnaware>
    <Version>1.0.0</Version>
    <Product>AppName</Product>
    <Description>One-line description of the app</Description>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Serilog" Version="4.0.2" />
    <PackageReference Include="Serilog.Sinks.File" Version="6.0.0" />
  </ItemGroup>
</Project>
```

## Program.cs
Logger first, then global handlers, then the UI. `SetUnhandledExceptionMode` must run before any control is created.
```csharp
using Serilog;
namespace AppName;

static class Program
{
    [STAThread]
    static void Main()
    {
        AppLogger.Init();

        Application.SetUnhandledExceptionMode(UnhandledExceptionMode.CatchException);
        Application.ThreadException += (_, e) =>
        {
            Log.Error(e.Exception, "Unhandled UI exception");
            Msg.Error(Strings.UnexpectedError);
        };
        AppDomain.CurrentDomain.UnhandledException += (_, e) =>
            Log.Fatal(e.ExceptionObject as Exception, "Unhandled exception (process may terminate)");
        TaskScheduler.UnobservedTaskException += (_, e) =>
        {
            Log.Error(e.Exception, "Unobserved task exception");
            e.SetObserved();
        };

        try
        {
            ApplicationConfiguration.Initialize();
            var settings = AppSettings.Load();
            Application.Run(new MainForm(settings));
        }
        finally
        {
            Log.CloseAndFlush();
        }
    }
}
```

## Services/Msg.cs
All dialogs go through here (RTL-aware, one title, consistent icons).
```csharp
namespace AppName;

public static class Msg
{
    private static MessageBoxOptions Options =>
        Strings.IsRtl ? MessageBoxOptions.RtlReading | MessageBoxOptions.RightAlign : default;

    public static void Info(string text) =>
        MessageBox.Show(text, Strings.AppTitle, MessageBoxButtons.OK, MessageBoxIcon.Information,
            MessageBoxDefaultButton.Button1, Options);

    public static void Error(string text) =>
        MessageBox.Show(text, Strings.AppTitle, MessageBoxButtons.OK, MessageBoxIcon.Error,
            MessageBoxDefaultButton.Button1, Options);

    public static bool Confirm(string text) =>
        MessageBox.Show(text, Strings.AppTitle, MessageBoxButtons.YesNo, MessageBoxIcon.Question,
            MessageBoxDefaultButton.Button2, Options) == DialogResult.Yes;
}
```

## Services/ThemeService.cs
WinForms (on net8.0) has no built-in dark mode, so colors are applied recursively. Call `Apply` again after adding controls dynamically.
```csharp
namespace AppName;

public static class ThemeService
{
    private sealed record Palette(Color Window, Color Surface, Color Text, Color Muted, Color Border, Color Accent);

    private static readonly Palette LightPalette = new(
        Color.FromArgb(245, 246, 248), Color.White, Color.FromArgb(31, 35, 40),
        Color.FromArgb(107, 114, 128), Color.FromArgb(208, 215, 222), Color.FromArgb(37, 99, 235));

    private static readonly Palette DarkPalette = new(
        Color.FromArgb(24, 26, 31), Color.FromArgb(33, 36, 43), Color.FromArgb(230, 233, 238),
        Color.FromArgb(150, 157, 168), Color.FromArgb(58, 63, 74), Color.FromArgb(96, 165, 250));

    public static AppTheme Current { get; private set; } = AppTheme.Light;

    private static Palette Active => Current == AppTheme.Dark ? DarkPalette : LightPalette;

    /// <summary>Secondary text color (status labels, hints) for the active theme.</summary>
    public static Color Muted => Active.Muted;

    public static AppTheme Toggle() => Current == AppTheme.Dark ? AppTheme.Light : AppTheme.Dark;

    public static void Apply(Control root, AppTheme theme)
    {
        Current = theme;
        var p = Active;
        Style(root, p);

        if (root is Form form)
        {
            if (form.IsHandleCreated)
                NativeTheme.SetDarkTitleBar(form.Handle, theme == AppTheme.Dark);
            else
                form.HandleCreated += (_, _) =>
                    NativeTheme.SetDarkTitleBar(form.Handle, Current == AppTheme.Dark);
        }
    }

    private static void Style(Control c, Palette p)
    {
        c.ForeColor = p.Text;
        switch (c)
        {
            case Button b:
                b.FlatStyle = FlatStyle.Flat;
                b.BackColor = p.Surface;
                b.FlatAppearance.BorderColor = p.Border;
                b.FlatAppearance.MouseOverBackColor = p.Window;
                b.UseVisualStyleBackColor = false;
                break;
            case TextBox t:
                t.BackColor = p.Surface;
                t.BorderStyle = BorderStyle.FixedSingle;
                break;
            case ListBox l:
                l.BackColor = p.Surface;
                l.BorderStyle = BorderStyle.FixedSingle;
                break;
            case DataGridView g:
                StyleGrid(g, p);
                break;
            case Label:
                break; // labels stay transparent and inherit the parent's background
            default:
                c.BackColor = p.Window;
                break;
        }
        foreach (Control child in c.Controls) Style(child, p);
    }

    private static void StyleGrid(DataGridView g, Palette p)
    {
        g.EnableHeadersVisualStyles = false;
        g.BorderStyle = BorderStyle.None;
        g.BackgroundColor = p.Window;
        g.GridColor = p.Border;
        g.DefaultCellStyle.BackColor = p.Surface;
        g.DefaultCellStyle.ForeColor = p.Text;
        g.DefaultCellStyle.SelectionBackColor = p.Accent;
        g.DefaultCellStyle.SelectionForeColor = Color.White;
        g.ColumnHeadersDefaultCellStyle.BackColor = p.Window;
        g.ColumnHeadersDefaultCellStyle.ForeColor = p.Text;
        g.ColumnHeadersDefaultCellStyle.SelectionBackColor = p.Window;
        g.RowHeadersDefaultCellStyle.BackColor = p.Window;
    }
}
```
Controls not covered above (ComboBox, TabControl, MenuStrip, etc.) keep system colors in dark mode: add a `case` for each one the app actually uses (for `MenuStrip`/`ToolStrip` set a custom `Renderer` or avoid them in dark-capable apps). `Application.SetColorMode` exists only on newer .NET versions and is experimental; do not rely on it with `net8.0`.

## MainForm.cs (list + form pattern)
```csharp
using Serilog;
namespace AppName;

public class MainForm : Form
{
    private readonly DataStore _store = new();
    private readonly AppSettings _settings;
    private List<Item> _items = new();
    private readonly ListBox _list = new();
    private readonly TextBox _nameBox = new();
    private readonly Label _status = new();

    public MainForm(AppSettings settings)
    {
        _settings = settings;
        Text = Strings.AppTitle;
        Font = new Font("Segoe UI", 9F);
        Size = new Size(760, 480);
        MinimumSize = new Size(600, 400);
        StartPosition = FormStartPosition.CenterScreen;
        if (Strings.IsRtl) { RightToLeft = RightToLeft.Yes; RightToLeftLayout = true; }

        BuildUi();
        ApplyTheme(_settings.Theme);
        Load += async (_, _) => await LoadItemsAsync();
    }

    private void BuildUi()
    {
        _list.Dock = DockStyle.Fill;
        _list.DisplayMember = nameof(Item.Name);
        _list.AccessibleName = Strings.Name;
        _list.SelectedIndexChanged += (_, _) =>
        { if (_list.SelectedItem is Item i) _nameBox.Text = i.Name; };

        var panel = new FlowLayoutPanel
        {
            Dock = DockStyle.Right, Width = 220, FlowDirection = FlowDirection.TopDown,
            WrapContents = false, Padding = new Padding(10)
        };
        _nameBox.Width = 190;
        _nameBox.AccessibleName = Strings.Name;
        panel.Controls.Add(new Label { Text = Strings.Name, AutoSize = true });
        panel.Controls.Add(_nameBox);
        panel.Controls.Add(MakeButton(Strings.Add, AddClick));
        panel.Controls.Add(MakeButton(Strings.Update, UpdateClick));
        panel.Controls.Add(MakeButton(Strings.Delete, DeleteClick));
        var themeButton = MakeButton(Strings.ToggleTheme, ThemeClick);
        themeButton.Margin = new Padding(3, 18, 3, 3);
        panel.Controls.Add(themeButton);
        _status.AutoSize = true;
        panel.Controls.Add(_status);

        _list.TabIndex = 0;
        panel.TabIndex = 1;
        for (var i = 0; i < panel.Controls.Count; i++) panel.Controls[i].TabIndex = i;

        // The Fill control is added first, edge-docked panels after.
        Controls.Add(_list);
        Controls.Add(panel);
    }

    private static Button MakeButton(string text, EventHandler onClick)
    {
        var b = new Button { Text = text, Width = 190, Height = 30 };
        b.Click += onClick;
        return b;
    }

    private void ApplyTheme(AppTheme theme)
    {
        ThemeService.Apply(this, theme);
        _status.ForeColor = ThemeService.Muted;
    }

    private async Task LoadItemsAsync()
    {
        try
        {
            _items = await _store.LoadAsync();
            if (_store.RecoveredFrom is not null) Msg.Info(Strings.DataRecovered(_store.RecoveredFrom));
        }
        catch (Exception ex)
        {
            Log.Error(ex, "Failed to load data");
            Msg.Error(Strings.LoadFailed);
        }
        RefreshList();
    }

    private void RefreshList()
    {
        _list.DataSource = null;
        _list.DataSource = _items;
        _list.DisplayMember = nameof(Item.Name);
        _status.Text = Strings.ItemsCount(_items.Count);
    }

    private async void AddClick(object? s, EventArgs e)
    {
        var name = _nameBox.Text.Trim();
        if (name.Length == 0) { Msg.Info(Strings.NameRequired); _nameBox.Focus(); return; }
        _items.Add(new Item { Name = name });
        if (await SaveAsync()) _nameBox.Clear();
    }

    private async void UpdateClick(object? s, EventArgs e)
    {
        if (_list.SelectedItem is not Item item) return;
        var name = _nameBox.Text.Trim();
        if (name.Length == 0) { Msg.Info(Strings.NameRequired); _nameBox.Focus(); return; }
        item.Name = name;
        await SaveAsync();
    }

    private async void DeleteClick(object? s, EventArgs e)
    {
        if (_list.SelectedItem is not Item item) return;
        if (!Msg.Confirm(Strings.ConfirmDelete)) return;
        _items.Remove(item);
        await SaveAsync();
    }

    private void ThemeClick(object? s, EventArgs e)
    {
        var next = ThemeService.Toggle();
        ApplyTheme(next);
        _settings.Theme = next;
        _settings.Save();
    }

    private async Task<bool> SaveAsync()
    {
        try
        {
            await _store.SaveAsync(_items);
            return true;
        }
        catch (Exception ex)
        {
            Log.Error(ex, "Failed to save data");
            Msg.Error(Strings.SaveFailed);
            return false;
        }
        finally
        {
            RefreshList();
        }
    }
}
```
Do not name a method `Refresh()` on a Form: it hides `Control.Refresh`. Use `RefreshList()`.

## Item.cs, DataStore.cs and the other services
In `common.md`. In WinForms `System.IO` is implicitly imported, but keep the explicit `using System.IO;` so the files are identical in both frameworks.

## WinForms gotchas
- Add `Dock` controls in the right order: the Fill control is added first, edge-docked panels after.
- Use `DataGridView` for tabular data (`AutoGenerateColumns = true`, `ReadOnly = true`, `SelectionMode = FullRowSelect`); `ThemeService` already styles it.
- RTL: `RightToLeft = Yes` + `RightToLeftLayout = true` mirrors the whole form, so write the layout as normal LTR code (a right-docked panel appears on the left). Inputs for emails/URLs/paths/codes get `RightToLeft = RightToLeft.No`.
- High-DPI: keep `ApplicationConfiguration.Initialize()` and the `ApplicationHighDpiMode` property in the csproj.
- Set `TabIndex` explicitly; the default creation order is rarely the reading order.
- Never enable trimming (`PublishTrimmed`) for WinForms.
