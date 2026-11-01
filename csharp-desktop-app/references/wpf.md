# WPF reference (code-behind)

Shared files (`AppPaths`, `DataStore`, `AppSettings`, `AppLogger`, `NativeTheme`, `Strings`, `Item`) are in `common.md`. This file has only the WPF-specific parts. Replace `AppName` everywhere.

## AppName/AppName.csproj
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <TargetFramework>net8.0-windows</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <UseWPF>true</UseWPF>
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

## App.xaml
No `StartupUri`: the window is created in code after the logger and theme are ready.
```xml
<Application x:Class="AppName.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
  <Application.Resources>
    <ResourceDictionary>
      <ResourceDictionary.MergedDictionaries>
        <ResourceDictionary Source="Resources/Controls.xaml"/>
      </ResourceDictionary.MergedDictionaries>
    </ResourceDictionary>
  </Application.Resources>
</Application>
```

## App.xaml.cs
```csharp
using System.Windows;
using System.Windows.Threading;
using Serilog;
namespace AppName;

public partial class App : Application
{
    protected override void OnStartup(StartupEventArgs e)
    {
        AppLogger.Init();

        DispatcherUnhandledException += OnDispatcherUnhandledException;
        AppDomain.CurrentDomain.UnhandledException += (_, args) =>
            Log.Fatal(args.ExceptionObject as Exception, "Unhandled exception (process may terminate)");
        TaskScheduler.UnobservedTaskException += (_, args) =>
        {
            Log.Error(args.Exception, "Unobserved task exception");
            args.SetObserved();
        };

        var settings = AppSettings.Load();
        ThemeService.Apply(settings.Theme);

        base.OnStartup(e);
        new MainWindow(settings).Show();
    }

    private void OnDispatcherUnhandledException(object sender, DispatcherUnhandledExceptionEventArgs e)
    {
        Log.Error(e.Exception, "Unhandled UI exception");
        Msg.Error(Strings.UnexpectedError);
        e.Handled = true;
    }

    protected override void OnExit(ExitEventArgs e)
    {
        Log.CloseAndFlush();
        base.OnExit(e);
    }
}
```

## Services/Msg.cs
All dialogs go through here (RTL-aware, one title, consistent icons).
```csharp
using System.Windows;
namespace AppName;

public static class Msg
{
    private static MessageBoxOptions Options =>
        Strings.IsRtl ? MessageBoxOptions.RtlReading | MessageBoxOptions.RightAlign : MessageBoxOptions.None;

    public static void Info(string text) =>
        MessageBox.Show(text, Strings.AppTitle, MessageBoxButton.OK, MessageBoxImage.Information, MessageBoxResult.OK, Options);

    public static void Error(string text) =>
        MessageBox.Show(text, Strings.AppTitle, MessageBoxButton.OK, MessageBoxImage.Error, MessageBoxResult.OK, Options);

    public static bool Confirm(string text) =>
        MessageBox.Show(text, Strings.AppTitle, MessageBoxButton.YesNo, MessageBoxImage.Question, MessageBoxResult.No, Options)
            == MessageBoxResult.Yes;
}
```

## Services/ThemeService.cs
Swaps the theme dictionary at runtime; all brushes are used through `DynamicResource`, so every window updates instantly.
```csharp
using System.Windows;
using System.Windows.Interop;
namespace AppName;

public static class ThemeService
{
    public static AppTheme Current { get; private set; } = AppTheme.Light;

    public static void Apply(AppTheme theme)
    {
        var merged = Application.Current.Resources.MergedDictionaries;
        var old = merged.FirstOrDefault(d =>
            d.Source?.OriginalString.StartsWith("Themes/", StringComparison.Ordinal) == true);
        if (old is not null) merged.Remove(old);
        merged.Add(new ResourceDictionary { Source = new Uri($"Themes/{theme}.xaml", UriKind.Relative) });
        Current = theme;

        foreach (Window window in Application.Current.Windows)
            NativeTheme.SetDarkTitleBar(new WindowInteropHelper(window).Handle, theme == AppTheme.Dark);
    }

    public static AppTheme Toggle() =>
        Current == AppTheme.Dark ? AppTheme.Light : AppTheme.Dark;
}
```

## Themes/Light.xaml
```xml
<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
                    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
  <SolidColorBrush x:Key="WindowBackgroundBrush" Color="#FFF5F6F8"/>
  <SolidColorBrush x:Key="SurfaceBrush" Color="#FFFFFFFF"/>
  <SolidColorBrush x:Key="ForegroundBrush" Color="#FF1F2328"/>
  <SolidColorBrush x:Key="MutedBrush" Color="#FF6B7280"/>
  <SolidColorBrush x:Key="BorderBrushTheme" Color="#FFD0D7DE"/>
  <SolidColorBrush x:Key="AccentBrush" Color="#FF2563EB"/>
  <SolidColorBrush x:Key="HoverBrush" Color="#FFE8EBEF"/>

  <!-- Selection colors used by DataGrid, ListBox, etc. -->
  <SolidColorBrush x:Key="{x:Static SystemColors.HighlightBrushKey}" Color="#FFDBEAFE"/>
  <SolidColorBrush x:Key="{x:Static SystemColors.HighlightTextBrushKey}" Color="#FF1F2328"/>
  <SolidColorBrush x:Key="{x:Static SystemColors.InactiveSelectionHighlightBrushKey}" Color="#FFE5E7EB"/>
  <SolidColorBrush x:Key="{x:Static SystemColors.InactiveSelectionHighlightTextBrushKey}" Color="#FF1F2328"/>
</ResourceDictionary>
```

## Themes/Dark.xaml
```xml
<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
                    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
  <SolidColorBrush x:Key="WindowBackgroundBrush" Color="#FF181A1F"/>
  <SolidColorBrush x:Key="SurfaceBrush" Color="#FF21242B"/>
  <SolidColorBrush x:Key="ForegroundBrush" Color="#FFE6E9EE"/>
  <SolidColorBrush x:Key="MutedBrush" Color="#FF969DA8"/>
  <SolidColorBrush x:Key="BorderBrushTheme" Color="#FF3A3F4A"/>
  <SolidColorBrush x:Key="AccentBrush" Color="#FF60A5FA"/>
  <SolidColorBrush x:Key="HoverBrush" Color="#FF2A2E37"/>

  <SolidColorBrush x:Key="{x:Static SystemColors.HighlightBrushKey}" Color="#FF1D4ED8"/>
  <SolidColorBrush x:Key="{x:Static SystemColors.HighlightTextBrushKey}" Color="#FFFFFFFF"/>
  <SolidColorBrush x:Key="{x:Static SystemColors.InactiveSelectionHighlightBrushKey}" Color="#FF2F3542"/>
  <SolidColorBrush x:Key="{x:Static SystemColors.InactiveSelectionHighlightTextBrushKey}" Color="#FFE6E9EE"/>
</ResourceDictionary>
```

## Resources/Controls.xaml
Shared control styles. They only use `DynamicResource` brushes, so they work for both themes. Implicit styles do not apply to subclasses, so windows use the keyed `AppWindow` style.
```xml
<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
                    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">

  <Style x:Key="AppWindow" TargetType="Window">
    <Setter Property="Background" Value="{DynamicResource WindowBackgroundBrush}"/>
    <Setter Property="Foreground" Value="{DynamicResource ForegroundBrush}"/>
    <Setter Property="FontFamily" Value="Segoe UI, Tahoma"/>
    <Setter Property="FontSize" Value="13"/>
  </Style>

  <Style TargetType="TextBox">
    <Setter Property="Background" Value="{DynamicResource SurfaceBrush}"/>
    <Setter Property="Foreground" Value="{DynamicResource ForegroundBrush}"/>
    <Setter Property="BorderBrush" Value="{DynamicResource BorderBrushTheme}"/>
    <Setter Property="CaretBrush" Value="{DynamicResource ForegroundBrush}"/>
    <Setter Property="Padding" Value="6,4"/>
  </Style>

  <Style TargetType="Button">
    <Setter Property="Background" Value="{DynamicResource SurfaceBrush}"/>
    <Setter Property="Foreground" Value="{DynamicResource ForegroundBrush}"/>
    <Setter Property="BorderBrush" Value="{DynamicResource BorderBrushTheme}"/>
    <Setter Property="Padding" Value="10,5"/>
    <Setter Property="Cursor" Value="Hand"/>
    <Setter Property="Template">
      <Setter.Value>
        <ControlTemplate TargetType="Button">
          <Border x:Name="Bd" Background="{TemplateBinding Background}"
                  BorderBrush="{TemplateBinding BorderBrush}" BorderThickness="1"
                  CornerRadius="4" Padding="{TemplateBinding Padding}">
            <ContentPresenter HorizontalAlignment="Center" VerticalAlignment="Center"/>
          </Border>
          <ControlTemplate.Triggers>
            <Trigger Property="IsMouseOver" Value="True">
              <Setter TargetName="Bd" Property="Background" Value="{DynamicResource HoverBrush}"/>
            </Trigger>
            <Trigger Property="IsKeyboardFocused" Value="True">
              <Setter TargetName="Bd" Property="BorderBrush" Value="{DynamicResource AccentBrush}"/>
            </Trigger>
            <Trigger Property="IsEnabled" Value="False">
              <Setter Property="Opacity" Value="0.5"/>
            </Trigger>
          </ControlTemplate.Triggers>
        </ControlTemplate>
      </Setter.Value>
    </Setter>
  </Style>

  <Style TargetType="DataGrid">
    <Setter Property="Background" Value="{DynamicResource SurfaceBrush}"/>
    <Setter Property="Foreground" Value="{DynamicResource ForegroundBrush}"/>
    <Setter Property="BorderBrush" Value="{DynamicResource BorderBrushTheme}"/>
    <Setter Property="RowBackground" Value="{DynamicResource SurfaceBrush}"/>
    <Setter Property="AlternatingRowBackground" Value="{DynamicResource WindowBackgroundBrush}"/>
    <Setter Property="AlternationCount" Value="2"/>
    <Setter Property="HorizontalGridLinesBrush" Value="{DynamicResource BorderBrushTheme}"/>
    <Setter Property="GridLinesVisibility" Value="Horizontal"/>
    <Setter Property="HeadersVisibility" Value="Column"/>
    <Setter Property="CanUserAddRows" Value="False"/>
  </Style>

  <Style TargetType="DataGridColumnHeader">
    <Setter Property="Background" Value="{DynamicResource WindowBackgroundBrush}"/>
    <Setter Property="Foreground" Value="{DynamicResource ForegroundBrush}"/>
    <Setter Property="BorderBrush" Value="{DynamicResource BorderBrushTheme}"/>
    <Setter Property="BorderThickness" Value="0,0,0,1"/>
    <Setter Property="Padding" Value="8,6"/>
    <Setter Property="FontWeight" Value="SemiBold"/>
  </Style>

  <Style TargetType="DataGridCell">
    <Setter Property="BorderThickness" Value="0"/>
    <Setter Property="Padding" Value="6,4"/>
    <Setter Property="FocusVisualStyle" Value="{x:Null}"/>
  </Style>
</ResourceDictionary>
```
Only the controls above are themed. If the app uses ComboBox, CheckBox, TabControl, Menu, ScrollBar, etc., add matching styles in the same file (Background/Foreground/BorderBrush via `DynamicResource`, and a simple `ControlTemplate` where the default template ignores Background), otherwise they stay light in dark mode.

## MainWindow.xaml (list + form pattern)
```xml
<Window x:Class="AppName.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:local="clr-namespace:AppName"
        Style="{StaticResource AppWindow}"
        Title="{x:Static local:Strings.AppTitle}" Height="500" Width="760" MinHeight="400" MinWidth="600"
        WindowStartupLocation="CenterScreen"
        SourceInitialized="Window_SourceInitialized"
        Loaded="Window_Loaded">
  <Grid Margin="12">
    <Grid.ColumnDefinitions>
      <ColumnDefinition Width="2*"/><ColumnDefinition Width="12"/><ColumnDefinition Width="*"/>
    </Grid.ColumnDefinitions>
    <DataGrid x:Name="ItemsGrid" Grid.Column="0" AutoGenerateColumns="False"
              IsReadOnly="True" SelectionMode="Single" TabIndex="0"
              AutomationProperties.Name="{x:Static local:Strings.Name}"
              SelectionChanged="ItemsGrid_SelectionChanged">
      <DataGrid.Columns>
        <DataGridTextColumn Header="{x:Static local:Strings.Name}" Binding="{Binding Name}" Width="*"/>
      </DataGrid.Columns>
    </DataGrid>
    <StackPanel Grid.Column="2">
      <TextBlock Text="{x:Static local:Strings.Name}" Margin="0,0,0,4"/>
      <TextBox x:Name="NameBox" Margin="0,0,0,12" TabIndex="1"
               AutomationProperties.Name="{x:Static local:Strings.Name}"/>
      <Button Content="{x:Static local:Strings.Add}" Click="AddButton_Click" Margin="0,0,0,6" TabIndex="2"/>
      <Button Content="{x:Static local:Strings.Update}" Click="UpdateButton_Click" Margin="0,0,0,6" TabIndex="3"/>
      <Button Content="{x:Static local:Strings.Delete}" Click="DeleteButton_Click" TabIndex="4"/>
      <Button Content="{x:Static local:Strings.ToggleTheme}" Click="ThemeButton_Click" Margin="0,18,0,0" TabIndex="5"/>
      <TextBlock x:Name="StatusText" Margin="0,12,0,0" Foreground="{DynamicResource MutedBrush}"/>
    </StackPanel>
  </Grid>
</Window>
```

## MainWindow.xaml.cs
```csharp
using System.Windows;
using System.Windows.Controls;
using System.Windows.Interop;
using Serilog;
namespace AppName;

public partial class MainWindow : Window
{
    private readonly DataStore _store = new();
    private readonly AppSettings _settings;
    private List<Item> _items = new();

    public MainWindow(AppSettings settings)
    {
        _settings = settings;
        InitializeComponent();
        FlowDirection = Strings.IsRtl ? FlowDirection.RightToLeft : FlowDirection.LeftToRight;
    }

    private void Window_SourceInitialized(object? sender, EventArgs e) =>
        NativeTheme.SetDarkTitleBar(new WindowInteropHelper(this).Handle, ThemeService.Current == AppTheme.Dark);

    private async void Window_Loaded(object sender, RoutedEventArgs e)
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
        ItemsGrid.ItemsSource = null;
        ItemsGrid.ItemsSource = _items;
        StatusText.Text = Strings.ItemsCount(_items.Count);
    }

    private async void AddButton_Click(object sender, RoutedEventArgs e)
    {
        var name = NameBox.Text.Trim();
        if (name.Length == 0) { Msg.Info(Strings.NameRequired); NameBox.Focus(); return; }
        _items.Add(new Item { Name = name });
        if (await SaveAsync()) NameBox.Clear();
    }

    private async void UpdateButton_Click(object sender, RoutedEventArgs e)
    {
        if (ItemsGrid.SelectedItem is not Item item) return;
        var name = NameBox.Text.Trim();
        if (name.Length == 0) { Msg.Info(Strings.NameRequired); NameBox.Focus(); return; }
        item.Name = name;
        await SaveAsync();
    }

    private async void DeleteButton_Click(object sender, RoutedEventArgs e)
    {
        if (ItemsGrid.SelectedItem is not Item item) return;
        if (!Msg.Confirm(Strings.ConfirmDelete)) return;
        _items.Remove(item);
        await SaveAsync();
    }

    private void ItemsGrid_SelectionChanged(object sender, SelectionChangedEventArgs e)
    {
        if (ItemsGrid.SelectedItem is Item item) NameBox.Text = item.Name;
    }

    private void ThemeButton_Click(object sender, RoutedEventArgs e)
    {
        var next = ThemeService.Toggle();
        ThemeService.Apply(next);
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
Adapt names, fields, and columns to the requested app. Models, `DataStore`, and the other services are in `common.md`.

## WPF gotchas
- `x:Class` must match namespace + class exactly; `MainWindow` has a constructor parameter, so never put `StartupUri` in `App.xaml`.
- Implicit `<Style TargetType="Window">` does not apply to `MainWindow` (a subclass). Use the keyed `AppWindow` style on every window.
- Always use `DynamicResource` (not `StaticResource`) for theme brushes, otherwise the runtime toggle does nothing.
- `System.IO` is not implicitly imported in WPF projects: add `using System.IO;`.
- After changing a plain list, reset `ItemsSource` (or use `ObservableCollection<T>`). Edited properties need `INotifyPropertyChanged` to refresh live; resetting `ItemsSource` is the simple way.
- Arabic UI: handled by `Strings.IsRtl`; see the RTL rules in `common.md`.
- Never enable trimming (`PublishTrimmed`) for WPF.
