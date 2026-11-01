---
name: csharp-desktop-app
description: Generate complete, professional, ready-to-run C# desktop applications (WPF or WinForms) on modern .NET using a simple code-behind style, with Serilog logging, global error handling, light/dark themes, proper RTL support, xUnit tests, and publish/installer scripts. Use this skill whenever the user asks to build, create, scaffold, or generate a C# desktop app, Windows app, WPF app, WinForms app, .NET GUI tool, or a desktop utility (CRUD tool, file manager, calculator, inventory, notes app, etc.), even if they do not name the framework. Also use for Arabic requests such as "تطبيق سطح المكتب" or "برنامج ويندوز بلغة سي شارب".
---

# C# Desktop App Generator (WPF / WinForms)

Goal: produce a **complete, professional project that builds, tests, and runs with `dotnet run`** with no manual fixes. Architecture stays simple on purpose: code-behind, no MVVM, no DI container. "Professional" here means tidy structure, logging, safe error handling, theming, tests, and a one-command publish, not heavier architecture.

## Workflow

1. **Pick the framework.** WPF if the user wants modern/styled UI, data binding, or doesn't care. WinForms if they want quick, classic, simple forms. If unclear, default to WPF and say so in one line.
2. **Read the references** before writing any file:
   - `references/common.md` (always): shared files, defaults, RTL and error-handling rules.
   - `references/wpf.md` or `references/winforms.md`: the framework-specific files.
   - `references/testing-publish.md` (always, unless the user says no tests / no publish): test project, publish script, installer, README, solution fallback.
3. **Plan briefly** (3-5 lines): app purpose, windows/forms, data model, storage, UI language (decides `Strings.IsRtl`).
4. **Generate every file** of the solution in a folder named after the app (PascalCase). Never leave placeholders such as "TODO" or "add your logic here". Generate a real GUID wherever one is needed.
5. **Verify** (details in "Verification"). If `dotnet` is not available, re-read the code once for compile errors (missing usings, mismatched event handler names, `x:Name` vs code references, `x:Class` vs namespace).
6. **Deliver**: present the solution (zip it) and give the run, test and publish commands plus a 2-3 line summary.

## Scale to the app

- Apps with no persistence (calculator, converter): skip `DataStore` and its tests, but keep logging, global error handling, theming, RTL handling, publish script.
- If the user says "no tests", "no installer", or "keep it minimal", drop that part and say so in the summary.
- Never add MVVM, DI containers, or extra UI libraries unless the user explicitly asks.

## Defaults

- Target `net8.0-windows`, `<Nullable>enable</Nullable>`, `<ImplicitUsings>enable</ImplicitUsings>`. Nullable warnings are errors (via `Directory.Build.props`).
- Namespace = project name. One class per file. File-scoped namespaces.
- NuGet: only `Serilog` + `Serilog.Sinks.File` in the app; `xunit` stack in the test project. SQLite (`Microsoft.Data.Sqlite`) only if the user asks.
- Persistence: JSON via `System.Text.Json` in `%AppData%/<AppName>/data.json` (atomic save, corrupt-file backup) unless the user asks for SQLite.
- All paths come from `AppPaths`; all UI text comes from `Strings`; all dialogs go through `Msg`.
- Theme: Light and Dark, follows the Windows setting on first run, toggle button in the UI, choice persisted in `settings.json`.
- UI language = the language the user wrote in. For Arabic set `Strings.IsRtl = true` and follow the RTL rules in `common.md`.
- Business/data logic lives in plain classes under `Services/` so code-behind stays short and testable. This is not MVVM, just tidy.

## Required output structure

```
AppName/
  AppName.sln
  Directory.Build.props
  .editorconfig
  .gitignore
  publish.ps1
  installer.iss              (only when an installer is requested or the app is for end users)
  README.md                  (10 lines max)
  AppName/
    AppName.csproj
    Program.cs                       (WinForms only)
    App.xaml, App.xaml.cs            (WPF only)
    MainWindow.xaml(.cs) | MainForm.cs
    Strings.cs
    Models/Item.cs
    Services/AppPaths.cs, AppSettings.cs, AppLogger.cs, DataStore.cs,
             NativeTheme.cs, ThemeService.cs, Msg.cs
    Themes/Light.xaml, Dark.xaml     (WPF only)
    Resources/Controls.xaml          (WPF only)
  AppName.Tests/
    AppName.Tests.csproj
    DataStoreTests.cs
```

## Verification

When `dotnet` is available, run in this order and fix everything before delivering:

1. `dotnet format AppName.sln` (normalizes whitespace and usings).
2. `dotnet build AppName.sln -c Release` (on Linux/macOS add `-p:EnableWindowsTargeting=true`; this compiles the WPF/WinForms project without running it).
3. `dotnet test AppName.Tests` (plain `net8.0`, runs on any OS).

Say plainly in the summary which of these you actually ran. Never claim a build or test pass you did not run.

## Quality checklist

- Builds with zero nullable warnings; no `#pragma warning disable` to hide problems.
- Every event handler referenced in XAML exists in code; `x:Class` matches namespace + class.
- **No empty `catch`.** Every catch logs with Serilog (full exception) and shows a friendly `Msg.Error` (no stack traces for the user). Global handlers are registered at startup (see `common.md`).
- Event handlers that touch disk or network are `async void` wrappers with their own try/catch; I/O uses `async`/`await`; the UI thread is never blocked.
- Empty-input validation with focus returned to the field; confirm before delete.
- Theme toggle works at runtime in both directions, covers every control used, and the choice survives restart.
- RTL: no hard-coded left/right alignment that breaks when mirrored; LTR islands (emails, URLs, paths, codes) keep `LeftToRight`; message boxes use RTL options.
- Window has `MinWidth/MinHeight`, logical tab order, `Title`, and accessible names on inputs (`AutomationProperties.Name` / `AccessibleName`).
- No UI strings outside `Strings.cs`; no paths outside `AppPaths.cs`.
- No secrets, no hard-coded user paths.
- Do not use deprecated APIs (`BinaryFormatter`, `WebClient`, `Thread.Abort`).
- Tests pass and cover the data layer (round-trip, missing file, corrupt file, atomic save).
- `publish.ps1` produces a single-file `AppName.exe` in `publish/`. Never enable trimming for WPF/WinForms (unsupported).

## When not to use

Console apps, ASP.NET, Unity, or MAUI/Avalonia/WinUI projects: say this skill covers WPF and WinForms and offer to adapt.
