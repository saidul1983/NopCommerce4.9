# Plugin Folder Structure and File Conventions

> Derived from reading the actual plugin projects in this codebase:
> `Nop.Plugin.Widgets.GoogleAnalytics`, `Nop.Plugin.Misc.Brevo`,
> `Nop.Plugin.Payments.CheckMoneyOrder`.

---

## Actual Plugin Layout (from this project)

### Simple Plugin — `Nop.Plugin.Payments.CheckMoneyOrder`
No custom DB tables, no Infrastructure folder, no Services folder:

```
Nop.Plugin.Payments.CheckMoneyOrder/
├── plugin.json
├── Nop.Plugin.Payments.CheckMoneyOrder.csproj
├── CheckMoneyOrderPaymentProcessor.cs   ← BasePlugin + IPaymentMethod
├── CheckMoneyOrderPaymentSettings.cs    ← ISettings
├── logo.png                             ← shown in admin plugin list
├── Notes.txt                            ← optional developer notes
├── Components/
│   └── CheckMoneyOrderViewComponent.cs  ← public checkout UI
├── Controllers/
│   └── PaymentCheckMoneyOrderController.cs
├── Models/
│   └── ConfigurationModel.cs
└── Views/
    ├── Configure.cshtml
    └── PaymentInfo.cshtml
```

### Widget Plugin — `Nop.Plugin.Widgets.GoogleAnalytics`
Has Infrastructure and Migrations folders, no custom DB entities:

```
Nop.Plugin.Widgets.GoogleAnalytics/
├── plugin.json
├── Nop.Plugin.Widgets.GoogleAnalytics.csproj
├── GoogleAnalyticsPlugin.cs             ← BasePlugin + IWidgetPlugin
├── GoogleAnalyticsSettings.cs           ← ISettings
├── GoogleAnalyticsDefaults.cs           ← static constants class
├── Events.cs                            ← IConsumer<T> implementations
├── logo.png
├── Notes.txt
├── Api/
│   └── GoogleAnalyticsHttpClient.cs     ← typed HttpClient
├── Components/
│   └── WidgetsGoogleAnalyticsViewComponent.cs
├── Controllers/
│   └── WidgetsGoogleAnalyticsController.cs
├── Infrastructure/
│   ├── NopStartup.cs                    ← INopStartup: DI registrations
│   └── RouteProvider.cs                 ← IRouteProvider: custom routes
├── Migrations/
│   ├── UpgradeTo460/
│   └── UpgradeTo470/
├── Models/
│   └── ConfigurationModel.cs
└── Views/
    ├── _ViewImports.cshtml
    ├── Configure.cshtml
    └── PublicInfo.cshtml
```

### Complex Misc Plugin — `Nop.Plugin.Misc.Brevo`
Has Infrastructure, Data, Domain, Services, and Content folders:

```
Nop.Plugin.Misc.Brevo/
├── plugin.json
├── Nop.Plugin.Misc.Brevo.csproj
├── BrevoPlugin.cs                       ← BasePlugin + IMiscPlugin + IWidgetPlugin
├── BrevoSettings.cs                     ← ISettings
├── BrevoDefaults.cs                     ← static constants class
├── logo.png
├── Notes.txt
├── Components/
│   └── WidgetsBrevoViewComponent.cs
├── Content/
│   └── styles.css                       ← plugin-specific CSS
├── Controllers/
│   ├── BrevoController.cs               ← admin controller
│   └── BrevoWebhookController.cs        ← public webhook controller
├── Data/
│   └── BrevoMigration.cs                ← MigrationBase: update migration
├── Domain/
│   └── (domain entities if any)
├── Infrastructure/
│   ├── NopStartup.cs                    ← INopStartup
│   └── RouteProvider.cs                 ← IRouteProvider
├── MarketingAutomation/
│   └── MarketingAutomationHttpClient.cs ← typed HttpClient
├── Models/
│   └── ConfigurationModel.cs
├── Services/
│   ├── BrevoManager.cs
│   └── MarketingAutomationManager.cs
└── Views/
    ├── _ViewImports.cshtml
    ├── Configure.cshtml
    └── PublicInfo.cshtml
```

---

## `plugin.json` — Exact Format Used in This Project

```json
{
  "Group": "Widgets",
  "FriendlyName": "Google Analytics",
  "SystemName": "Widgets.GoogleAnalytics",
  "Version": "4.90.2",
  "SupportedVersions": [ "4.90" ],
  "Author": "nopCommerce team",
  "DisplayOrder": 1,
  "FileName": "Nop.Plugin.Widgets.GoogleAnalytics.dll",
  "Description": "This plugin integrates with Google Analytics"
}
```

---

## `NopStartup.cs` — Exact Pattern Used in This Project

```csharp
// From Nop.Plugin.Misc.Brevo/Infrastructure/NopStartup.cs
public class NopStartup : INopStartup
{
    public void ConfigureServices(IServiceCollection services, IConfiguration configuration)
    {
        // Register typed HttpClients
        services.AddHttpClient<MarketingAutomationHttpClient>().WithProxy();

        // Register plugin-specific services
        services.AddScoped<BrevoManager>();
        services.AddScoped<MarketingAutomationManager>();

        // Override a core service (last registration wins)
        services.AddScoped<IWorkflowMessageService, BrevoMessageService>();
    }

    public void Configure(IApplicationBuilder application) { }

    // Must be > 2002 so all core services are available
    public int Order => 3000;
}
```

---

## `RouteProvider.cs` — Two Patterns Used in This Project

### Pattern 1 — Inherits `BaseRouteProvider` (GoogleAnalytics)
```csharp
// Uses BaseRouteProvider base class and AreaNames.ADMIN constant
public class RouteProvider : BaseRouteProvider, IRouteProvider
{
    public void RegisterRoutes(IEndpointRouteBuilder endpointRouteBuilder)
    {
        endpointRouteBuilder.MapControllerRoute(
            name: GoogleAnalyticsDefaults.ConfigurationRouteName,
            pattern: "Admin/WidgetsGoogleAnalytics/Configure",
            defaults: new { controller = "WidgetsGoogleAnalytics", action = "Configure", area = AreaNames.ADMIN }
        );
    }

    public int Priority => 0;
}
```

### Pattern 2 — Direct `IRouteProvider` (Brevo — public routes)
```csharp
// Used for public-facing webhook/callback routes
public class RouteProvider : IRouteProvider
{
    public void RegisterRoutes(IEndpointRouteBuilder endpointRouteBuilder)
    {
        endpointRouteBuilder.MapControllerRoute(
            BrevoDefaults.ImportContactsRoute,
            "Plugins/Brevo/ImportContacts",
            new { controller = "Brevo", action = "ImportContacts" }
        );
    }

    public int Priority => 0;
}
```

Use `BaseRouteProvider` for admin routes. Use `IRouteProvider` directly for public routes.
Route names should be stored in the plugin's `Defaults.cs` constants class.

---

## `Defaults.cs` — Constants Class Pattern

Every plugin should have a static constants class at the root:

```csharp
// From GoogleAnalyticsDefaults.cs
public static class GoogleAnalyticsDefaults
{
    public static string SystemName => "Widgets.GoogleAnalytics";
    public static string ConfigurationRouteName => "Plugin.Widgets.GoogleAnalytics.Configure";
    // ... other constants
}
```

This avoids magic strings scattered across the plugin.

---

## `.csproj` — Exact Pattern Used in This Project

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <!-- Output path uses $(SolutionDir) — not relative ..\ paths -->
    <OutputPath>$(SolutionDir)\Presentation\Nop.Web\Plugins\Widgets.GoogleAnalytics</OutputPath>
    <OutDir>$(OutputPath)</OutDir>
    <!-- false = do NOT copy NuGet DLLs (plugin has no NuGet packages) -->
    <!-- true  = DO copy NuGet DLLs (plugin has its own NuGet packages, e.g. Brevo) -->
    <CopyLocalLockFileAssemblies>false</CopyLocalLockFileAssemblies>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

  <ItemGroup>
    <!-- All views and assets must be explicitly listed as Content -->
    <Content Include="logo.png">
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
    </Content>
    <Content Include="plugin.json">
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
    </Content>
    <Content Include="Views\Configure.cshtml">
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
    </Content>
    <Content Include="Views\_ViewImports.cshtml">
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
    </Content>
  </ItemGroup>

  <ItemGroup>
    <!-- Reference Nop.Web (not individual libraries) — this is the actual pattern used -->
    <ProjectReference Include="$(SolutionDir)\Presentation\Nop.Web\Nop.Web.csproj" />
    <!-- ClearPluginAssemblies removes duplicate core DLLs from plugin output -->
    <ClearPluginAssemblies Include="$(SolutionDir)\Build\ClearPluginAssemblies.proj" />
  </ItemGroup>

  <!-- This MSBuild target runs after Build to clean up duplicate DLLs -->
  <Target Name="NopTarget" AfterTargets="Build">
    <MSBuild Projects="@(ClearPluginAssemblies)" Properties="PluginPath=$(OutDir)" Targets="NopClear" />
  </Target>
</Project>
```

**Key differences from generic nopCommerce docs:**
- Reference `Nop.Web.csproj` directly — not individual `Nop.Services`, `Nop.Core` etc.
- Use `$(SolutionDir)` for `OutputPath` — not relative `..\..\` paths
- Use `ClearPluginAssemblies` MSBuild target to remove duplicate DLLs — this replaces the `<Private>False</Private>` approach
- Set `CopyLocalLockFileAssemblies=true` only if the plugin has its own NuGet packages

---

## `InstallAsync` / `UninstallAsync` — Exact Pattern Used in This Project

```csharp
// From GoogleAnalyticsPlugin.cs — widget plugin install pattern
public override async Task InstallAsync()
{
    // 1. Save default settings
    await _settingService.SaveSettingAsync(new GoogleAnalyticsSettings { ... });

    // 2. Register as active widget (widget plugins only)
    if (!_widgetSettings.ActiveWidgetSystemNames.Contains(GoogleAnalyticsDefaults.SystemName))
    {
        _widgetSettings.ActiveWidgetSystemNames.Add(GoogleAnalyticsDefaults.SystemName);
        await _settingService.SaveSettingAsync(_widgetSettings);
    }

    // 3. Add locale resources
    await _localizationService.AddOrUpdateLocaleResourceAsync(new Dictionary<string, string>
    {
        ["Plugins.Widgets.GoogleAnalytics.GoogleId"] = "ID",
        // ...
    });

    // 4. Always call base last
    await base.InstallAsync();
}

public override async Task UninstallAsync()
{
    // 1. Remove from active widgets (widget plugins only)
    if (_widgetSettings.ActiveWidgetSystemNames.Contains(GoogleAnalyticsDefaults.SystemName))
    {
        _widgetSettings.ActiveWidgetSystemNames.Remove(GoogleAnalyticsDefaults.SystemName);
        await _settingService.SaveSettingAsync(_widgetSettings);
    }

    // 2. Delete settings
    await _settingService.DeleteSettingAsync<GoogleAnalyticsSettings>();

    // 3. Delete locale resources
    await _localizationService.DeleteLocaleResourcesAsync("Plugins.Widgets.GoogleAnalytics");

    // 4. Always call base last
    await base.UninstallAsync();
}
```

---

## Naming Conventions (from actual plugins)

| Item | Pattern | Real Example |
|---|---|---|
| Project folder | `Nop.Plugin.<Group>.<Name>` | `Nop.Plugin.Widgets.GoogleAnalytics` |
| Output folder | `<Group>.<Name>` | `Widgets.GoogleAnalytics` |
| System name | `<Group>.<Name>` | `Widgets.GoogleAnalytics` |
| Main plugin class | `<Name>Plugin` | `GoogleAnalyticsPlugin`, `BrevoPlugin` |
| Payment processor | `<Name>PaymentProcessor` | `CheckMoneyOrderPaymentProcessor` |
| Settings class | `<Name>Settings` | `GoogleAnalyticsSettings` |
| Defaults class | `<Name>Defaults` | `GoogleAnalyticsDefaults` |
| NopStartup class | `NopStartup` (always this name) | `NopStartup` |
| RouteProvider class | `RouteProvider` (always this name) | `RouteProvider` |
| Locale key prefix | `Plugins.<Group>.<Name>.` | `Plugins.Widgets.GoogleAnalytics.` |
| Entity class | `<Description>Record` | `ShippingByWeightByTotalRecord` |
| Entity builder | `<EntityName>Builder` | `ShippingByWeightByTotalRecordBuilder` |
| Schema migration | `SchemaMigration` | `SchemaMigration` |
| Update migration | descriptive class name | `ChangeDecimalPrecision`, `AddLoadAllRecordSetting` |
| Validator class | `<ModelName>Validator` | `ConfigurationValidator` |
| Event consumer | `EventConsumer` (always this name) | `EventConsumer` |
| Schedule task | `<Description>Task` | `SynchronizationTask` |

---

## Migration Folder Organisation (from actual plugins)

Two patterns exist in this project:

### Flat migrations (FixedByWeightByTotal)
```
Migrations/
  SchemaMigration.cs              ← MigrationProcessType.Installation
  UpgradeTo450.cs                 ← MigrationProcessType.Update
  AddLoadAllRecordSetting.cs      ← MigrationProcessType.Update
```

### Versioned subfolders (GoogleAnalytics)
```
Migrations/
  UpgradeTo460/
    SomeMigration.cs
  UpgradeTo470/
    LocalizationMigration.cs      ← MigrationProcessType.Update
```

Both patterns are valid. Use versioned subfolders when a plugin has many update migrations.

---

## Factories Folder (PayPalCommerce pattern)

Complex plugins that need to prepare view models from multiple services use a `Factories/`
folder — the same model factory pattern used in the core web project:

```
Factories/
  IPayPalCommerceModelFactory.cs
  PayPalCommerceModelFactory.cs
```

Use this when a controller action needs to build a complex view model from many services.
Register the factory in `NopStartup.cs` as `AddScoped`.

---

## Content Folder (Brevo pattern)

Plugin-specific CSS or JS files go in a `Content/` folder:

```
Content/
  styles.css
```

These must be explicitly listed in the `.csproj` as `<Content>` items and referenced
in views using the plugin's output path:

```html
<link rel="stylesheet" href="~/Plugins/Misc.Brevo/Content/styles.css" />
