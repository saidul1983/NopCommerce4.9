# Plugin Development Rules

> Every rule here is derived from reading the actual plugin implementations in this
> project: `Nop.Plugin.Widgets.GoogleAnalytics`, `Nop.Plugin.Misc.Brevo`,
> `Nop.Plugin.Payments.CheckMoneyOrder`. No rule is based on generic documentation.

---

## Rule 1 — Never Modify Core Code

All customization happens through plugins. If you need to change core behavior, override
the service in your plugin's `NopStartup.cs`. The last registration wins because
`Order = 3000` runs after the core at `Order = 2000`:

```csharp
// Brevo/Infrastructure/NopStartup.cs — replaces the core email service
services.AddScoped<IWorkflowMessageService, BrevoMessageService>();
```

---

## Rule 2 — Dependency Injection Lifetime Rules

Use the correct DI lifetime. Getting this wrong causes runtime bugs that are hard to trace.

| Lifetime | Use For | Real Example in This Project |
|---|---|---|
| `AddScoped` | Services, managers, anything per-HTTP-request | `services.AddScoped<BrevoManager>()` |
| `AddSingleton` | Stateless, thread-safe, long-lived objects | `IStaticCacheManager` (core) |
| `AddTransient` | Short-lived, stateless utilities | `IScheduleTaskRunner` (core) |
| `AddHttpClient<T>` | Typed HTTP clients | `services.AddHttpClient<MarketingAutomationHttpClient>().WithProxy()` |

**Critical constraints:**
- Never register a `Singleton` that depends on a `Scoped` service — this is a captive
  dependency and will cause stale data or exceptions
- Always call `.WithProxy()` on `AddHttpClient<T>()` — this is the pattern used in every
  plugin in this project that makes HTTP calls
- `INopStartup.Order` must be `3000` so all core `Scoped` services are available

```csharp
// Correct pattern from NopStartup.cs in both Brevo and GoogleAnalytics
public void ConfigureServices(IServiceCollection services, IConfiguration configuration)
{
    services.AddHttpClient<GoogleAnalyticsHttpClient>().WithProxy();
    services.AddScoped<BrevoManager>();
    services.AddScoped<MarketingAutomationManager>();
}

public int Order => 3000;
```

---

## Rule 3 — Repository Pattern Usage

- Always use `IRepository<TEntity>` for data access — never write raw SQL for standard CRUD
- Never access the database from a controller or view
- Use `INopDataProvider` only for bulk operations or direct table queries in migrations

```csharp
// Correct — repository injected into service
public class MyService : IMyService
{
    private readonly IRepository<MyEntity> _myEntityRepository;

    public MyService(IRepository<MyEntity> myEntityRepository)
    {
        _myEntityRepository = myEntityRepository;
    }

    public async Task<MyEntity> GetByIdAsync(int id)
        => await _myEntityRepository.GetByIdAsync(id,
            cache => cache.PrepareKeyForDefaultCache(MyDefaults.ByIdCacheKey, id));
}
```

For direct table access in migrations (as Brevo does), use `INopDataProvider`:

```csharp
// From BrevoMigration.cs — direct table query in a migration
var sendinblueSettings = _dataProvider.GetTable<Setting>()
    .Where(x => x.Name.StartsWith("sendinbluesettings.")).ToList();
```

---

## Rule 4 — Event-Driven Architecture

This is the primary mechanism for plugins to react to system events without coupling.
`IConsumer<T>` implementations are **auto-discovered** — no manual registration needed.

### Single-event consumer (GoogleAnalytics pattern)
```csharp
// GoogleAnalytics/Events.cs — one class, multiple event interfaces
public class EventConsumer :
    IConsumer<OrderPlacedEvent>,
    IConsumer<OrderPaidEvent>,
    IConsumer<OrderRefundedEvent>
{
    public async Task HandleEventAsync(OrderPlacedEvent eventMessage)
    {
        // Always check plugin is active first
        if (!await IsPluginEnabledAsync())
            return;

        // Load settings per store — not from constructor injection
        var store = await _storeService.GetStoreByIdAsync(eventMessage.Order.StoreId);
        var settings = await _settingService.LoadSettingAsync<GoogleAnalyticsSettings>(store.Id);

        if (!settings.EnableEcommerce)
            return;

        await SaveCookiesAsync(eventMessage.Order, settings, store);
    }
}
```

### Entity CRUD event consumer (Brevo pattern)
```csharp
// Brevo/Services/EventConsumer.cs — reacts to entity-level events
public class EventConsumer :
    IConsumer<EntityInsertedEvent<ShoppingCartItem>>,
    IConsumer<EntityUpdatedEvent<ShoppingCartItem>>,
    IConsumer<EntityDeletedEvent<ShoppingCartItem>>,
    IConsumer<OrderPaidEvent>,
    IConsumer<OrderPlacedEvent>,
    IConsumer<EntityTokensAddedEvent<Store>>,
    IConsumer<EntityTokensAddedEvent<Customer>>
{
    public async Task HandleEventAsync(EntityInsertedEvent<ShoppingCartItem> eventMessage)
    {
        // Always guard with IsConfigured check
        if (!BrevoManager.IsConfigured(_brevoSettings))
            return;

        await _marketingAutomationManager.HandleShoppingCartChangedEventAsync(eventMessage.Entity);
    }
}
```

### Two mandatory patterns in every event handler:
1. **Always check plugin is active/configured first** — `IsPluginEnabledAsync()` or
   `IsConfigured()` — before doing any work
2. **Load settings per store inside the handler** — use
   `_settingService.LoadSettingAsync<TSettings>(storeId)`, not constructor-injected settings,
   because the event may fire for a different store than the current request context

### Where to place event consumers:
- Simple plugins: `Events.cs` at the project root (GoogleAnalytics pattern)
- Complex plugins: `Services/EventConsumer.cs` (Brevo pattern)

---

## Rule 5 — Background Task / Schedule Task Rule

Implement `IScheduleTask` for background work. The task class must:
- Live in `Services/` folder
- Have a fully qualified type name stored in `Defaults.cs`
- Be registered in the database during `InstallAsync()` and removed in `UninstallAsync()`

```csharp
// Brevo/Services/SynchronizationTask.cs
public class SynchronizationTask : IScheduleTask
{
    protected readonly BrevoManager _brevoEmailManager;

    public SynchronizationTask(BrevoManager brevoEmailManager)
    {
        _brevoEmailManager = brevoEmailManager;
    }

    // Must be async Task — never void, never sync
    public async Task ExecuteAsync()
    {
        await _brevoEmailManager.SynchronizeAsync();
    }
}
```

```csharp
// Brevo/BrevoDefaults.cs — type name stored as constant
public static string SynchronizationTask
    => "Nop.Plugin.Misc.Brevo.Services.SynchronizationTask";
public static string SynchronizationTaskName
    => "Synchronization (Brevo plugin)";
public static int DefaultSynchronizationPeriod => 12; // hours
```

```csharp
// BrevoPlugin.cs — register in InstallAsync
await _scheduleTaskService.InsertTaskAsync(new ScheduleTask
{
    Enabled = true,
    LastEnabledUtc = DateTime.UtcNow,
    Seconds = BrevoDefaults.DefaultSynchronizationPeriod * 60 * 60,
    Name = BrevoDefaults.SynchronizationTaskName,
    Type = BrevoDefaults.SynchronizationTask,
});

// Remove in UninstallAsync
var task = await _scheduleTaskService.GetTaskByTypeAsync(BrevoDefaults.SynchronizationTask);
if (task != null)
    await _scheduleTaskService.DeleteTaskAsync(task);
```

---

## Rule 6 — Route Registration Rule

Two patterns exist in this project. Choose based on whether the route is admin or public.

### Admin routes — inherit `BaseRouteProvider`, use `AreaNames.ADMIN`
```csharp
// GoogleAnalytics/Infrastructure/RouteProvider.cs
public class RouteProvider : BaseRouteProvider, IRouteProvider
{
    public void RegisterRoutes(IEndpointRouteBuilder endpointRouteBuilder)
    {
        endpointRouteBuilder.MapControllerRoute(
            name: GoogleAnalyticsDefaults.ConfigurationRouteName,
            pattern: "Admin/WidgetsGoogleAnalytics/Configure",
            defaults: new { controller = "WidgetsGoogleAnalytics", action = "Configure",
                            area = AreaNames.ADMIN }
        );
    }

    public int Priority => 0;
}
```

### Public routes — implement `IRouteProvider` directly, no area
```csharp
// Brevo/Infrastructure/RouteProvider.cs — webhook/callback routes
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

**Rules:**
- Route names must be stored in `<Name>Defaults.cs` — never inline strings
- Admin route pattern: `"Admin/{ControllerName}/{ActionName}"`
- Public plugin route pattern: `"Plugins/{PluginFolder}/{ActionName}"`
- Class name is always `RouteProvider` (not `MyPluginRouteProvider`)

---

## Rule 7 — Admin Area Convention

### Two base classes — choose the right one:

| Controller Type | Base Class | Use When |
|---|---|---|
| Admin config page | `BasePluginController` | Plugin configuration in admin |
| Payment admin | `BasePaymentController` | Payment plugin admin pages |
| Public webhook | `Controller` | Public-facing endpoints (no admin) |

### Attribute placement — two patterns exist in this project:

**Pattern A — Class-level attributes** (CheckMoneyOrder — simpler, all actions are admin):
```csharp
[AuthorizeAdmin]
[Area(AreaNames.ADMIN)]
[AutoValidateAntiforgeryToken]
public class PaymentCheckMoneyOrderController : BasePaymentController
{
    // All actions inherit the class-level attributes
    [CheckPermission(StandardPermission.Configuration.MANAGE_PAYMENT_METHODS)]
    public async Task<IActionResult> Configure() { ... }
}
```

**Pattern B — Action-level attributes** (Brevo — mixed admin + public actions):
```csharp
[AutoValidateAntiforgeryToken]
public class BrevoController : BasePluginController
{
    // Each admin action declares its own area and authorization
    [AuthorizeAdmin]
    [Area(AreaNames.ADMIN)]
    public async Task<IActionResult> Configure() { ... }

    [HttpPost]
    [AuthorizeAdmin]
    [Area(AreaNames.ADMIN)]
    public async Task<IActionResult> MessageList(...) { ... }
}
```

**Public webhook controller — no admin attributes at all:**
```csharp
// BrevoWebhookController.cs — inherits plain Controller, no [AuthorizeAdmin]
public class BrevoWebhookController : Controller
{
    [HttpPost]
    public async Task<IActionResult> UnsubscribeWebHook()
    {
        await _brevoEmailManager.HandleWebhookAsync(Request);
        return Ok();
    }
}
```

### Permission checking:
- Use `[CheckPermission(StandardPermission.Configuration.MANAGE_PAYMENT_METHODS)]`
  for payment plugins (CheckMoneyOrder pattern)
- Use `[AuthorizeAdmin]` for general admin access (Brevo pattern)
- Never skip authorization on admin actions

---

## Rule 8 — Plugin Descriptor (`plugin.json`) Rules

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

| Field | Rule |
|---|---|
| `SystemName` | Must be unique. Format: `<Group>.<Name>`. Stored in `Defaults.cs` as `SystemName` property |
| `SupportedVersions` | Must be `["4.90"]` for this system |
| `FileName` | Must exactly match the compiled DLL name including `.dll` |
| `Version` | Format: `"4.90.x"` — major.minor.patch |
| `DisplayOrder` | Controls sort order in admin plugin list |
| `DependsOnSystemNames` | Omit the field entirely if no dependencies (not `[]`) |

The `SystemName` value must also be stored in `Defaults.cs`:
```csharp
public static string SystemName => "Widgets.GoogleAnalytics";
```

---

## Rule 9 — Cache Key Naming Convention

Cache keys are defined in `Defaults.cs` using `CacheKey` from `Nop.Core.Caching`.
Only Brevo uses a cache key in this project:

```csharp
// BrevoDefaults.cs
public static CacheKey SyncKeyCache => new("PLUGINS_MISC_BREVO_SYNCINFO");
```

Used in the controller:
```csharp
// BrevoController.cs
var res = await _staticCacheManager.GetAsync(
    _staticCacheManager.PrepareKeyForDefaultCache(BrevoDefaults.SyncKeyCache),
    () => string.Empty);
await _staticCacheManager.RemoveAsync(BrevoDefaults.SyncKeyCache);
```

**Naming convention for cache keys:**
- Format: `"PLUGINS_{GROUP}_{NAME}_{DESCRIPTION}"` — all uppercase
- Define in `Defaults.cs`, never inline
- Use `_staticCacheManager.PrepareKeyForDefaultCache()` to create the final key
- Always remove the key after reading when it is a one-time notification cache

---

## Rule 10 — Async-Only Rule

Every method that does I/O must be `async Task` or `async Task<T>`. This is enforced
throughout every plugin in this project without exception.

```csharp
// Correct — every handler is async
public async Task HandleEventAsync(OrderPaidEvent eventMessage) { ... }
public async Task ExecuteAsync() { ... }                          // IScheduleTask
public async Task<IViewComponentResult> InvokeAsync(...) { ... } // ViewComponent
public async Task<IActionResult> Configure() { ... }             // Controller action
public override async Task InstallAsync() { ... }                // Plugin lifecycle
public override async Task UninstallAsync() { ... }              // Plugin lifecycle
```

**Exception — synchronous token handlers** (Brevo pattern for `EntityTokensAddedEvent`):
```csharp
// When no async work is needed, return Task.CompletedTask — do not use async void
public Task HandleEventAsync(EntityTokensAddedEvent<Store> eventMessage)
{
    if (!BrevoManager.IsConfigured(_brevoSettings))
        return Task.CompletedTask;

    eventMessage.Tokens.Add(new Token("Store.Id", eventMessage.Entity.Id));
    return Task.CompletedTask;
}
```

Never use `async void`. Never use `.Result` or `.Wait()` on tasks.

---

## Rule 11 — Localization Loading Timing Rule

Settings must be loaded **inside the method** using `_settingService.LoadSettingAsync<T>(storeId)`,
not from constructor-injected settings, when the store context may differ from the
current request. This is critical in event consumers and schedule tasks.

```csharp
// Correct — load settings per store inside the handler (GoogleAnalytics pattern)
public async Task HandleEventAsync(OrderPaidEvent eventMessage)
{
    var order = eventMessage.Order;
    var store = await _storeService.GetStoreByIdAsync(order.StoreId)
                ?? await _storeContext.GetCurrentStoreAsync();

    // Load settings for the ORDER's store, not the current request's store
    var settings = await _settingService.LoadSettingAsync<GoogleAnalyticsSettings>(store.Id);

    if (!settings.EnableEcommerce)
        return;
}
```

```csharp
// Correct — load settings for active store scope in admin controller (CheckMoneyOrder pattern)
public async Task<IActionResult> Configure()
{
    var storeScope = await _storeContext.GetActiveStoreScopeConfigurationAsync();
    var settings = await _settingService.LoadSettingAsync<CheckMoneyOrderPaymentSettings>(storeScope);
    // ...
}
```

**For per-store setting overrides**, use `SaveSettingOverridablePerStoreAsync`:
```csharp
// CheckMoneyOrder — saves with per-store override support
await _settingService.SaveSettingOverridablePerStoreAsync(
    settings, x => x.DescriptionText,
    model.DescriptionText_OverrideForStore,
    storeScope,
    clearCache: false   // batch saves — clear cache once at the end
);
// ...
await _settingService.ClearCacheAsync(); // clear once after all saves
```

---

## Rule 12 — View Location Convention

Plugin views are **not** discovered automatically by the Razor engine. They must be
referenced by their full virtual path from the web root.

### Full path format used in this project:
```
~/Plugins/{OutputFolderName}/Views/{ViewName}.cshtml
```

```csharp
// CheckMoneyOrder controller — full path, not just view name
return View("~/Plugins/Payments.CheckMoneyOrder/Views/Configure.cshtml", model);

// Brevo controller — full path
return View("~/Plugins/Misc.Brevo/Views/Configure.cshtml", model);
```

```csharp
// GoogleAnalytics ViewComponent — full path in InvokeAsync
public async Task<IViewComponentResult> InvokeAsync(string widgetZone, object additionalData)
{
    var script = await GetScriptAsync();
    return View("~/Plugins/Widgets.GoogleAnalytics/Views/PublicInfo.cshtml", script);
}
```

The path segment `Plugins/{OutputFolderName}` must match the `OutputPath` in the `.csproj`:
```xml
<OutputPath>$(SolutionDir)\Presentation\Nop.Web\Plugins\Widgets.GoogleAnalytics</OutputPath>
```
So the view path is `~/Plugins/Widgets.GoogleAnalytics/Views/PublicInfo.cshtml`.

### `_ViewImports.cshtml` — required in every Views folder:
```cshtml
@inherits Nop.Web.Framework.Mvc.Razor.NopRazorPage<TModel>
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
@addTagHelper *, Nop.Web.Framework

@using Microsoft.AspNetCore.Mvc.ViewFeatures
@using Nop.Web.Areas.Admin.Components
@using Nop.Web.Framework.UI
@using Nop.Web.Framework.Extensions
@using System.Text.Encodings.Web
```

The `@inherits Nop.Web.Framework.Mvc.Razor.NopRazorPage<TModel>` line is mandatory —
it gives views access to nopCommerce-specific Razor helpers.

---

## Rule 13 — Plugin Isolation Rule

Plugins must not reference each other's code directly. Communication between plugins
happens only through:

1. **Core service interfaces** — `IOrderService`, `IProductService`, etc.
2. **Core events** — `IConsumer<OrderPlacedEvent>`, `IConsumer<EntityInsertedEvent<T>>`, etc.
3. **Core entity models** — `Order`, `Customer`, `ShoppingCartItem`, etc.
4. **`IWidgetPluginManager`** — to check if another widget plugin is active

```csharp
// Correct — GoogleAnalytics checks its own active status via IWidgetPluginManager
protected async Task<bool> IsPluginEnabledAsync()
{
    return await _widgetPluginManager.IsPluginActiveAsync(GoogleAnalyticsDefaults.SystemName);
}
```

```csharp
// Correct — Brevo checks its own configuration state via a static method on its own manager
if (!BrevoManager.IsConfigured(_brevoSettings))
    return;
```

**Never** add a `<ProjectReference>` to another plugin project. If shared logic is needed,
it belongs in a shared library or in `Nop.Services`.

---

## Rule 14 — Widget Plugin Must Manage `ActiveWidgetSystemNames`

Widget plugins must explicitly add/remove themselves from `WidgetSettings.ActiveWidgetSystemNames`.
Without this, the widget will never render even if the plugin is installed.

```csharp
// InstallAsync — add to active widgets
if (!_widgetSettings.ActiveWidgetSystemNames.Contains(GoogleAnalyticsDefaults.SystemName))
{
    _widgetSettings.ActiveWidgetSystemNames.Add(GoogleAnalyticsDefaults.SystemName);
    await _settingService.SaveSettingAsync(_widgetSettings);
}

// UninstallAsync — remove from active widgets
if (_widgetSettings.ActiveWidgetSystemNames.Contains(GoogleAnalyticsDefaults.SystemName))
{
    _widgetSettings.ActiveWidgetSystemNames.Remove(GoogleAnalyticsDefaults.SystemName);
    await _settingService.SaveSettingAsync(_widgetSettings);
}
```

---

## Rule 15 — Always Call `base.InstallAsync()` and `base.UninstallAsync()` Last

```csharp
public override async Task InstallAsync()
{
    // all setup code first...
    await base.InstallAsync();   // always last
}

public override async Task UninstallAsync()
{
    // all cleanup code first...
    await base.UninstallAsync(); // always last
}
```

---

## Rule 16 — Check `DataSettingsManager.IsDatabaseInstalled()` in Migrations

Data migrations that query the database must guard against running before the DB exists:

```csharp
// BrevoMigration.cs
public override void Up()
{
    if (!DataSettingsManager.IsDatabaseInstalled())
        return;

    // safe to query DB here
}
```

---

## Rule 17 — Migration Type Selection

Three migration base classes exist. Choose the right one based on what the migration does:

| Base Class | Use When | Auto-generates Down()? |
|---|---|---|
| `AutoReversingMigration` | Pure schema — `Create.TableFor<T>()` only | Yes — drops the table |
| `MigrationBase` | Data operations + schema together | No — write `Down()` manually |
| `Migration` | Schema alterations (`Alter.Table`, column changes) | No — write `Down()` manually |

```csharp
// AutoReversingMigration — schema only (FixedByWeightByTotal/SchemaMigration.cs)
[NopMigration("2020/02/03 08:40:55:1687541",
    "Shipping.FixedByWeightByTotal base schema",
    MigrationProcessType.Installation)]
public class SchemaMigration : AutoReversingMigration
{
    public override void Up()
    {
        Create.TableFor<ShippingByWeightByTotalRecord>();
    }
    // Down() is auto-generated: Drop.Table(...)
}
```

```csharp
// Migration — column alterations (FixedByWeightByTotal/UpgradeTo450.cs)
[NopMigration("2021-10-29 11:00:00",
    "Shipping.FixedByWeightByTotal change decimal precision",
    MigrationProcessType.Update)]
public class ChangeDecimalPrecision : Migration
{
    public override void Up()
    {
        Alter.Table(nameof(ShippingByWeightByTotalRecord))
            .AlterColumn(nameof(ShippingByWeightByTotalRecord.WeightFrom))
            .AsDecimal(18, 4);
    }

    public override void Down()
    {
        // add downgrade logic if necessary
    }
}
```

```csharp
// MigrationBase — data + settings operations (BrevoMigration.cs)
[NopMigration("2023-05-22 17:00:00",
    "Misc.Brevo 4.70.4. Rename Sendinblue to Brevo.",
    MigrationProcessType.Update)]
public class BrevoMigration : MigrationBase
{
    public override void Up() { /* rename settings, locales, schedule tasks */ }
    public override void Down() { }
}
```

### Migration timestamp formats used in this project:
- `"2020/02/03 08:40:55:1687541"` — legacy format (older plugins)
- `"2021-10-29 11:00:00"` — standard format (newer plugins)
- Both are valid. Use `"YYYY-MM-DD HH:MM:SS"` for new migrations.
- Timestamps must be **globally unique** across all migrations in the entire solution.

### Migration folder organisation:
- Installation migrations: flat in `Migrations/` folder
- Update migrations: in `Migrations/UpgradeTo{Version}/` subfolder (GoogleAnalytics pattern)
  or flat with descriptive class name (FixedByWeightByTotal pattern)

---

## Rule 18 — Entity and Data Mapping Pattern

Plugin entities that need their own DB table follow this exact pattern from
`FixedByWeightByTotal`:

### Entity — inherits `BaseEntity`, no `partial`, XML doc comments on every property
```csharp
// Domain/ShippingByWeightByTotalRecord.cs
public class ShippingByWeightByTotalRecord : BaseEntity
{
    /// <summary>Gets or sets the store identifier</summary>
    public int StoreId { get; set; }

    /// <summary>Gets or sets the zip</summary>
    public string Zip { get; set; }

    /// <summary>Gets or sets the "Weight from" value</summary>
    public decimal WeightFrom { get; set; }
}
```

### Entity Builder — inherits `NopEntityBuilder<T>`, only override non-default columns
```csharp
// Data/ShippingByWeightByTotalRecordBuilder.cs
public class ShippingByWeightByTotalRecordBuilder
    : NopEntityBuilder<ShippingByWeightByTotalRecord>
{
    public override void MapEntity(CreateTableExpressionBuilder table)
    {
        table
            // Decimal precision must be explicit — default is (18,2)
            .WithColumn(nameof(ShippingByWeightByTotalRecord.WeightFrom))
                .AsDecimal(18, 4)
            .WithColumn(nameof(ShippingByWeightByTotalRecord.WeightTo))
                .AsDecimal(18, 4)
            // Nullable strings must be declared
            .WithColumn(nameof(ShippingByWeightByTotalRecord.Zip))
                .AsString(400).Nullable();
        // Int, bool columns with defaults do NOT need to be listed here
    }
}
```

**Rules:**
- Only declare columns that differ from defaults (non-default precision, nullable, max length)
- `int`, `bool`, non-nullable `string` columns are handled automatically
- Always use `nameof()` — never hardcode column name strings
- The builder is auto-discovered — no manual registration needed

---

## Rule 19 — FluentValidation Pattern

Validators inherit `BaseNopValidator<TModel>` (not `AbstractValidator<T>` directly).
Validation messages use `WithMessageAwait` with `ILocalizationService` — never hardcoded strings.

```csharp
// PayPalCommerce/Validators/ConfigurationValidator.cs
public class ConfigurationValidator : BaseNopValidator<ConfigurationModel>
{
    public ConfigurationValidator(ILocalizationService localizationService)
    {
        RuleFor(model => model.ClientId)
            .NotEmpty()
            .WithMessageAwait(localizationService.GetResourceAsync(
                "Plugins.Payments.PayPalCommerce.Fields.ClientId.Required"))
            .When(model => !model.UseSandbox && model.SetCredentialsManually);
    }
}
```

**Rules:**
- Always inherit `BaseNopValidator<T>` — not `AbstractValidator<T>`
- Always use `WithMessageAwait(localizationService.GetResourceAsync(...))` for messages
- Use `.When(condition)` for conditional validation
- Validators are **auto-discovered** — no manual registration needed
- Place in `Validators/` folder, named `<ModelName>Validator.cs`

---

## Rule 20 — Settings Migration Pattern

When a new setting is added to an existing plugin via an update migration, use
`EngineContext.Current.Resolve<ISettingService>()` — not constructor injection —
because DI is not fully available during migration execution:

```csharp
// FixedByWeightByTotal/Migrations/AddLoadAllRecordSetting.cs
[NopMigration("2023-08-17 15:00:00",
    "Shipping.FixedByWeightByTotal add LoadAllRecord setting",
    MigrationProcessType.Update)]
public class AddLoadAllRecordSetting : Migration
{
    public override void Up()
    {
        // Use EngineContext — not constructor injection — in migrations
        var settingService = EngineContext.Current.Resolve<ISettingService>();
        var pluginSettings = settingService.LoadSetting<FixedByWeightByTotalSettings>();

        // Always check if setting already exists before adding
        if (!settingService.SettingExists(pluginSettings, settings => settings.LoadAllRecord))
        {
            pluginSettings.LoadAllRecord = true;
            settingService.SaveSetting(pluginSettings, settings => settings.LoadAllRecord);
        }
    }

    public override void Down() { }
}
```

**Rules:**
- Use `EngineContext.Current.Resolve<T>()` in migrations — not constructor injection
- Always check `settingService.SettingExists()` before adding a new setting in an update migration
- Use synchronous `LoadSetting` / `SaveSetting` in migrations — not the async versions

---

## Rule 21 — Localization Migration Pattern

When locale resources need to be added or renamed in an update migration, use
`EngineContext.Current.Resolve<ILocalizationService>()` and the synchronous
`AddOrUpdateLocaleResource` method:

```csharp
// GoogleAnalytics/Migrations/UpgradeTo470/LocalizationMigration.cs
[NopMigration("2023-03-01 17:00:00",
    "Widgets.GoogleAnalytics 2.00. Update localizations",
    MigrationProcessType.Update)]
public class LocalizationMigration : MigrationBase
{
    public override void Up()
    {
        if (!DataSettingsManager.IsDatabaseInstalled())
            return;

        var localizationService = EngineContext.Current.Resolve<ILocalizationService>();
        var (languageId, _) = this.GetLanguageData();

        // Synchronous version — not async — in migrations
        localizationService.AddOrUpdateLocaleResource(new Dictionary<string, string>
        {
            ["Plugins.Widgets.GoogleAnalytics.UseSandbox"] = "UseSandbox",
        }, languageId);
    }

    public override void Down() { }
}
```

**Rules:**
- Use `EngineContext.Current.Resolve<ILocalizationService>()` — not constructor injection
- Use synchronous `AddOrUpdateLocaleResource` — not the async version
- Call `this.GetLanguageData()` extension method to get the default language ID
- Always guard with `DataSettingsManager.IsDatabaseInstalled()`
- For renaming locale keys, use `this.RenameLocales(dictionary, languages, service)`
  (Brevo migration pattern)

---

## Rule 22 — `ISettings` Auto-Registration Behaviour

`ISettings` implementations are **auto-discovered and registered as Scoped** by
`WebAppTypeFinder`. They are loaded per-store automatically. This has important implications:

```csharp
// Settings are injected directly — no need to call LoadSettingAsync in the constructor
public class GoogleAnalyticsPlugin : BasePlugin, IWidgetPlugin
{
    protected readonly GoogleAnalyticsSettings _settings; // injected per-store automatically

    public GoogleAnalyticsPlugin(GoogleAnalyticsSettings settings, ...)
    {
        _settings = settings;
    }
}
```

**However** — in event consumers and schedule tasks, the settings injected via constructor
reflect the **current HTTP request's store**. If the event is for a different store,
load settings explicitly:

```csharp
// In event consumers — load per the event's store, not the request's store
var settings = await _settingService.LoadSettingAsync<GoogleAnalyticsSettings>(store.Id);
```

**Rules:**
- Constructor-injected `ISettings` is correct for controllers and plugin main class
- Always use `LoadSettingAsync(storeId)` in event consumers and schedule tasks
- Never call `new MySettings()` — always get settings through DI or `ISettingService`

---

## Rule 23 — `protected readonly` Field Convention

All field declarations in every plugin in this project use `protected readonly`,
not `private readonly`:

```csharp
// Correct — from every plugin in this project
protected readonly ILocalizationService _localizationService;
protected readonly ISettingService _settingService;
protected readonly GoogleAnalyticsSettings _googleAnalyticsSettings;
```

This allows subclasses to access fields without re-injecting them, which is important
because nopCommerce controllers and services are designed to be overridable.

---

## Rule 24 — `Schema.Table().Column().Exists()` Guard in Alter Migrations

When altering columns in an update migration, always check if the column exists first
to make the migration safe to run on databases that may already have the change:

```csharp
// FixedByWeightByTotal/Migrations/UpgradeTo450.cs
foreach (var columnName in new[] { nameof(ShippingByWeightByTotalRecord.WeightFrom), ... })
{
    if (!Schema.Table(tableName).Column(columnName).Exists())
        continue;

    Alter.Table(tableName).AlterColumn(columnName).AsDecimal(18, 4);
}
```

---

## Rule 25 — Anti-Forgery Token Enforcement

`[AutoValidateAntiforgeryToken]` is **mandatory on every controller class** in this
project — without exception. It is applied at the class level, not per-action.

```csharp
// Every admin controller in this project — class-level, not action-level
[AuthorizeAdmin]
[Area(AreaNames.ADMIN)]
[AutoValidateAntiforgeryToken]                    // ← always present
public class WidgetsGoogleAnalyticsController : BasePluginController { }

// Every public controller in this project
[AutoValidateAntiforgeryToken]                    // ← always present
public class RfqCustomerController : BasePublicController { }

// Webhook/callback controllers that receive external POST requests
[AutoValidateAntiforgeryToken]                    // ← still present at class level
public class BrevoController : BasePluginController { }
```

### The one exception — `[IgnoreAntiforgeryToken]` on specific actions

When a specific POST action receives data from an external source (file upload, AJAX
without token, external callback), use `[IgnoreAntiforgeryToken]` on that action only:

```csharp
// Swiper plugin — file upload action bypasses token check
[IgnoreAntiforgeryToken]
[CheckPermission(StandardPermission.Configuration.MANAGE_WIDGETS)]
[HttpPost, ActionName("Configure")]
[FormValueRequired("add-slide")]
public async Task<IActionResult> AddSlide(ConfigurationModel model) { ... }
```

### Webhook controllers that receive external POST — no anti-forgery at all

Public webhook controllers that receive POST from external services (Brevo, PayPal)
inherit plain `Controller` and have no anti-forgery attributes:

```csharp
// BrevoWebhookController.cs — external webhook, no anti-forgery
public class BrevoWebhookController : Controller
{
    [HttpPost]
    public async Task<IActionResult> UnsubscribeWebHook() { ... }
}
```

**Rule summary:**
- `[AutoValidateAntiforgeryToken]` on every controller class
- `[IgnoreAntiforgeryToken]` only on specific actions that receive external data
- Webhook controllers that receive external POST inherit `Controller` directly (not
  `BasePluginController`) and have no anti-forgery

---

## Rule 26 — Permission System Must Be Explicitly Defined Per Action

Every admin action must declare its required permission using `[CheckPermission]`.
`[AuthorizeAdmin]` alone is not sufficient — it only checks that the user is an admin,
not that they have the specific permission for the resource.

### Permission constants by plugin type (from actual plugins):

```csharp
// Widget plugins
[CheckPermission(StandardPermission.Configuration.MANAGE_WIDGETS)]

// Payment plugins
[CheckPermission(StandardPermission.Configuration.MANAGE_PAYMENT_METHODS)]

// Tax plugins
[CheckPermission(StandardPermission.Configuration.MANAGE_TAX_SETTINGS)]

// Shipping plugins
[CheckPermission(StandardPermission.Configuration.MANAGE_SHIPPING_SETTINGS)]

// Multi-permission — action requires BOTH permissions
[CheckPermission(StandardPermission.Catalog.PRODUCTS_VIEW)]
[CheckPermission(StandardPermission.Configuration.MANAGE_TAX_SETTINGS)]
public async Task<IActionResult> ProductToClassification() { ... }
```

### Two placement patterns:

**Pattern A — class-level** (CheckMoneyOrder — all actions need same permission):
```csharp
[AuthorizeAdmin]
[Area(AreaNames.ADMIN)]
[AutoValidateAntiforgeryToken]
public class PaymentCheckMoneyOrderController : BasePaymentController
{
    [CheckPermission(StandardPermission.Configuration.MANAGE_PAYMENT_METHODS)]
    public async Task<IActionResult> Configure() { ... }

    [HttpPost]
    [CheckPermission(StandardPermission.Configuration.MANAGE_PAYMENT_METHODS)]
    public async Task<IActionResult> Configure(ConfigurationModel model) { ... }
}
```

**Pattern B — action-level** (Brevo — mixed permissions across actions):
```csharp
[AutoValidateAntiforgeryToken]
public class BrevoController : BasePluginController
{
    [AuthorizeAdmin]
    [Area(AreaNames.ADMIN)]
    public async Task<IActionResult> Configure() { ... }  // uses AuthorizeAdmin only
}
```

**Rule:** Use `[CheckPermission]` with the most specific `StandardPermission` constant
that matches the resource being managed. Never rely on `[AuthorizeAdmin]` alone for
resource-specific operations.

---

## Rule 27 — Query Optimization: Use LINQ Inside Repository Lambdas

All filtering, ordering, and joining must happen inside the LINQ lambda passed to
`GetAllAsync` / `GetAllPagedAsync` — not after the result is returned. This ensures
the query executes in the database, not in memory.

```csharp
// Correct — filtering happens in the DB query
return await _repository.GetAllAsync(
    query => query
        .Where(r => r.StoreId == storeId && r.Active)
        .OrderBy(r => r.DisplayOrder),
    cache => cache.PrepareKeyForDefaultCache(MyDefaults.AllKey, storeId)
);

// Wrong — loads all records then filters in memory
var all = await _repository.GetAllAsync();
return all.Where(r => r.StoreId == storeId).ToList();
```

### Paged queries — always use `GetAllPagedAsync`

```csharp
// Correct — paging happens in DB
return await _repository.GetAllPagedAsync(
    query => query
        .Where(r => r.CustomerId == customerId)
        .OrderByDescending(r => r.CreatedOnUtc),
    pageIndex,
    pageSize
);
```

### `includeDeleted` parameter — always pass explicitly for soft-deleted entities

```csharp
// Correct — explicitly exclude deleted records
await _repository.GetByIdAsync(id, cache => ..., includeDeleted: false);
await _repository.GetAllAsync(query => ..., cache => ..., includeDeleted: false);
```

The default is `includeDeleted: true`. For soft-deleted entities, always pass
`includeDeleted: false` unless you specifically need deleted records.

### `WithNoLock` — SQL Server only, configured in `appsettings.json`

The `WithNoLock` setting in `appsettings.json` is a global SQL Server hint. Plugins
do not control this — it is a deployment configuration decision.

---

## Rule 28 — Cache Pattern Is Mandatory, Not Optional

Caching is not optional for frequently-read data. The `IRepository<T>` methods accept
a `getCacheKey` parameter — always provide it for data that is read on every request.

### Three cache tiers — use the right one:

```csharp
// Tier 1: IStaticCacheManager — long-lived, survives across requests
// Use for: settings, plugin lists, reference data
return await _repository.GetAllAsync(
    query => query.Where(r => r.Active),
    cache => cache.PrepareKeyForDefaultCache(MyDefaults.AllActiveKey)
);

// Tier 2: IShortTermCacheManager (PerRequestCacheManager) — per HTTP request only
// Use for: data needed multiple times in one request
return await _repository.GetByIdAsync(id,
    cache => cache.PrepareKeyForDefaultCache(MyDefaults.ByIdKey, id),
    useShortTermCache: true
);

// Tier 3: No cache — pass null
// Use for: write operations, admin grids, one-time reads
return await _repository.GetAllAsync(query => query.Where(...), getCacheKey: null);
```

### Cache invalidation is automatic via events

When `InsertAsync`, `UpdateAsync`, or `DeleteAsync` is called on the repository,
`EntityInsertedEvent<T>`, `EntityUpdatedEvent<T>`, `EntityDeletedEvent<T>` are
published automatically. Your `IConsumer<T>` cache event consumer handles invalidation:

```csharp
public class MyEntityCacheConsumer :
    IConsumer<EntityInsertedEvent<MyEntity>>,
    IConsumer<EntityUpdatedEvent<MyEntity>>,
    IConsumer<EntityDeletedEvent<MyEntity>>
{
    public async Task HandleEventAsync(EntityInsertedEvent<MyEntity> e)
        => await _staticCacheManager.RemoveByPrefixAsync(MyDefaults.AllPrefix);

    public async Task HandleEventAsync(EntityUpdatedEvent<MyEntity> e)
        => await _staticCacheManager.RemoveByPrefixAsync(MyDefaults.AllPrefix);

    public async Task HandleEventAsync(EntityDeletedEvent<MyEntity> e)
        => await _staticCacheManager.RemoveByPrefixAsync(MyDefaults.AllPrefix);
}
```

**Never** call `_staticCacheManager.RemoveByPrefixAsync()` directly inside a service
method after a write. Always let the event consumer handle it.

---

## Rule 29 — Soft Delete / Data Integrity Model

nopCommerce uses soft delete for entities that implement `ISoftDeletedEntity`.
The repository enforces this automatically — you never need to check it manually.

### How it works (from `EntityRepository.cs`):

```csharp
// DeleteAsync — automatically detects ISoftDeletedEntity
public virtual async Task DeleteAsync(TEntity entity, bool publishEvent = true)
{
    switch (entity)
    {
        case ISoftDeletedEntity softDeletedEntity:
            softDeletedEntity.Deleted = true;          // soft delete
            await _dataProvider.UpdateEntityAsync(entity);
            break;
        default:
            await _dataProvider.DeleteEntityAsync(entity); // hard delete
            break;
    }
    if (publishEvent)
        await _eventPublisher.EntityDeletedAsync(entity);
}
```

### Rules for plugin entities:

- If your entity represents auditable business data (orders, transactions, records),
  implement `ISoftDeletedEntity` — add a `bool Deleted { get; set; }` property
- If your entity is configuration/reference data that can be safely removed, do not
  implement `ISoftDeletedEntity`
- Never set `entity.Deleted = true` manually — call `_repository.DeleteAsync(entity)`
  and let the repository handle it
- When querying, always pass `includeDeleted: false` to exclude soft-deleted records
  from normal queries

```csharp
// Plugin entity with soft delete
public class MyRecord : BaseEntity, ISoftDeletedEntity
{
    public bool Deleted { get; set; }
    // ... other properties
}

// Querying — exclude deleted
var records = await _repository.GetAllAsync(
    query => query.Where(r => r.CustomerId == customerId),
    cache => ...,
    includeDeleted: false   // ← always explicit for soft-deleted entities
);
```

---

## Rule 30 — Event System Guarantees and Constraints

The event system in `EntityRepository.cs` has specific guarantees that affect how
you design plugin code.

### Guarantee 1 — Events fire AFTER the DB write succeeds

```csharp
// From EntityRepository.InsertAsync:
await _dataProvider.InsertEntityAsync(entity);   // DB write first
if (publishEvent)
    await _eventPublisher.EntityInsertedAsync(entity); // event after
```

This means: when your `IConsumer<EntityInsertedEvent<T>>` fires, the entity is
already committed to the database. You can safely query it.

### Guarantee 2 — Bulk operations use TransactionScope

```csharp
// From EntityRepository.InsertAsync(IList<TEntity>):
using var transaction = new TransactionScope(TransactionScopeAsyncFlowOption.Enabled);
await _dataProvider.BulkInsertEntitiesAsync(entities);
transaction.Complete();
// Events fire AFTER transaction.Complete()
```

Bulk insert/delete/update are wrapped in `TransactionScope`. Events fire after the
transaction commits. If the transaction fails, no events are published.

### Guarantee 3 — `publishEvent: false` suppresses events

```csharp
// Use publishEvent: false for internal/batch operations that should not trigger
// cache invalidation or side effects
await _repository.InsertAsync(entity, publishEvent: false);
await _repository.UpdateAsync(entities, publishEvent: false);
```

Use `publishEvent: false` when:
- Performing bulk data migrations where cache invalidation would be expensive
- Inserting seed data during plugin installation
- Internal state updates that should not trigger external side effects

### Guarantee 4 — Event consumers run synchronously in sequence

`EventPublisher.PublishAsync` iterates consumers in registration order and awaits each
one. If a consumer throws, the error is logged but subsequent consumers still run.
Your consumer must not throw unhandled exceptions.

```csharp
// Always wrap consumer logic in try-catch for non-critical operations
public async Task HandleEventAsync(OrderPlacedEvent eventMessage)
{
    try
    {
        if (!await IsPluginEnabledAsync())
            return;
        await _myService.ProcessOrderAsync(eventMessage.Order);
    }
    catch (Exception ex)
    {
        await _logger.ErrorAsync($"{MyDefaults.SystemName} error processing order", ex);
        // Do NOT rethrow — other consumers must still run
    }
}
```

---

## Rule 31 — Transaction Behavior (Implicit Unit of Work)

nopCommerce does not use an explicit Unit of Work pattern. Each repository operation
is its own transaction unless you wrap multiple operations in `TransactionScope`.

### Single entity operations — auto-committed immediately

```csharp
// Each call is its own transaction
await _repository.InsertAsync(entity1);   // committed
await _repository.UpdateAsync(entity2);   // committed
// If second call fails, first is already committed — no rollback
```

### Multiple operations that must be atomic — use TransactionScope

```csharp
// Wrap in TransactionScope when multiple writes must succeed or fail together
using var transaction = new TransactionScope(TransactionScopeAsyncFlowOption.Enabled);

await _repository.InsertAsync(entity1, publishEvent: false);
await _repository.UpdateAsync(entity2, publishEvent: false);

transaction.Complete(); // commits both
// Publish events manually after commit
await _eventPublisher.EntityInsertedAsync(entity1);
await _eventPublisher.EntityUpdatedAsync(entity2);
```

**Rules:**
- Use `TransactionScopeAsyncFlowOption.Enabled` — always, for async code
- Pass `publishEvent: false` inside the transaction — publish events manually after
  `transaction.Complete()` so events only fire on successful commit
- Keep transactions short — do not call external services inside a `TransactionScope`

---

## Rule 32 — HttpClient Usage Pattern

Every plugin that makes HTTP calls uses a **typed HttpClient** class. Never use
`HttpClient` directly or `new HttpClient()`.

### Typed HttpClient structure (from GoogleAnalytics and FacebookPixel):

```csharp
// Api/GoogleAnalyticsHttpClient.cs — typed client class
public class GoogleAnalyticsHttpClient
{
    protected readonly HttpClient _httpClient;

    // HttpClient is injected by IHttpClientFactory — never new'd up
    public GoogleAnalyticsHttpClient(HttpClient httpClient)
    {
        _httpClient = httpClient;
    }

    public async Task RequestAsync(EventRequest request, GoogleAnalyticsSettings settings)
    {
        _httpClient.BaseAddress = new Uri(settings.EndPointUrl);
        _httpClient.Timeout = TimeSpan.FromSeconds(10);  // always set timeout

        var content = new StringContent(
            JsonConvert.SerializeObject(request),
            Encoding.Default,
            MimeTypes.ApplicationJson);

        var response = await _httpClient.SendAsync(...);
        response.EnsureSuccessStatusCode();
    }
}
```

### Registration — always `.WithProxy()` (from every plugin in this project):

```csharp
// NopStartup.cs — always use .WithProxy()
services.AddHttpClient<GoogleAnalyticsHttpClient>().WithProxy();
services.AddHttpClient<MarketingAutomationHttpClient>().WithProxy();
services.AddHttpClient<FacebookConversionsHttpClient>().WithProxy();
```

`.WithProxy()` is a nopCommerce extension that applies the system proxy configuration.
Omitting it means the HTTP client ignores the store's proxy settings.

### Naming convention:

```
{PluginName}HttpClient.cs       → GoogleAnalyticsHttpClient
{ServiceName}HttpClient.cs      → MarketingAutomationHttpClient
```

Place in `Api/` folder (GoogleAnalytics pattern) or `Services/` folder (Brevo pattern).

**Rules:**
- Always use typed HttpClient — never `IHttpClientFactory` directly
- Always register with `.WithProxy()`
- Always set `Timeout` — never rely on the default (infinite)
- Always use `MimeTypes.ApplicationJson` for content type — never hardcode `"application/json"`

---

## Rule 33 — Plugin Lifecycle Data Integrity

Plugin install and uninstall must leave the database in a clean state. Incomplete
cleanup causes orphaned data that breaks reinstallation.

### Complete install checklist (from Brevo and GoogleAnalytics):

```csharp
public override async Task InstallAsync()
{
    // 1. Save default settings
    await _settingService.SaveSettingAsync(new MyPluginSettings { ... });

    // 2. Add to ActiveWidgetSystemNames (widget plugins only)
    if (!_widgetSettings.ActiveWidgetSystemNames.Contains(MyDefaults.SystemName))
    {
        _widgetSettings.ActiveWidgetSystemNames.Add(MyDefaults.SystemName);
        await _settingService.SaveSettingAsync(_widgetSettings);
    }

    // 3. Register schedule tasks (if any)
    if (await _scheduleTaskService.GetTaskByTypeAsync(MyDefaults.TaskType) == null)
    {
        await _scheduleTaskService.InsertTaskAsync(new ScheduleTask
        {
            Enabled = true,
            LastEnabledUtc = DateTime.UtcNow,
            Seconds = MyDefaults.TaskPeriodHours * 60 * 60,
            Name = MyDefaults.TaskName,
            Type = MyDefaults.TaskType,
        });
    }

    // 4. Add locale resources
    await _localizationService.AddOrUpdateLocaleResourceAsync(new Dictionary<string, string>
    {
        ["Plugins.MyGroup.MyPlugin.Key"] = "Value",
    });

    // 5. Always last
    await base.InstallAsync();
}
```

### Complete uninstall checklist:

```csharp
public override async Task UninstallAsync()
{
    // 1. Clean up plugin-specific data (email accounts, generic attributes, etc.)
    // Brevo deletes email accounts it created:
    foreach (var store in await _storeService.GetAllStoresAsync())
    {
        var emailAccountId = await _settingService.GetSettingByKeyAsync<int>(
            $"{nameof(BrevoSettings)}.{nameof(BrevoSettings.EmailAccountId)}",
            storeId: store.Id, loadSharedValueIfNotFound: true);
        var emailAccount = await _emailAccountService.GetEmailAccountByIdAsync(emailAccountId);
        if (emailAccount != null)
            await _emailAccountService.DeleteEmailAccountAsync(emailAccount);
    }

    // 2. Remove from ActiveWidgetSystemNames (widget plugins only)
    if (_widgetSettings.ActiveWidgetSystemNames.Contains(MyDefaults.SystemName))
    {
        _widgetSettings.ActiveWidgetSystemNames.Remove(MyDefaults.SystemName);
        await _settingService.SaveSettingAsync(_widgetSettings);
    }

    // 3. Delete settings
    await _settingService.DeleteSettingAsync<MyPluginSettings>();

    // 4. Delete schedule tasks
    var task = await _scheduleTaskService.GetTaskByTypeAsync(MyDefaults.TaskType);
    if (task != null)
        await _scheduleTaskService.DeleteTaskAsync(task);

    // 5. Delete locale resources
    await _localizationService.DeleteLocaleResourcesAsync("Plugins.MyGroup.MyPlugin");

    // 6. Always last
    await base.UninstallAsync();
}
```

**Rule:** Every resource created in `InstallAsync` must be destroyed in `UninstallAsync`.
Missing cleanup causes `InvalidOperationException` on reinstall.

---

## Rule 34 — Logging System

All plugins use `ILogger` (nopCommerce's own logger, not `Microsoft.Extensions.Logging.ILogger`).
It writes to the database and is visible in Admin → System → Log.

### Correct import and usage:

```csharp
using Nop.Services.Logging;  // ← nopCommerce ILogger, not Microsoft's

protected readonly ILogger _logger;

// Error with exception — always include the customer context when available
await _logger.ErrorAsync(
    $"{MyDefaults.SystemName} error: {errorMessage}",
    exception,
    await _workContext.GetCurrentCustomerAsync()  // optional but preferred
);

// Error without customer context (event consumers, background tasks)
await _logger.InsertLogAsync(
    LogLevel.Error,
    "Google Analytics. Error canceling transaction from server side",
    ex.ToString()
);

// Warning
await _logger.WarningAsync($"{MyDefaults.SystemName}. {warning}");

// Information (only when explicitly enabled in settings)
if (_settings.LogMessages)
    await _logger.InformationAsync($"{MyDefaults.SystemName} info: {message}");
```

### Two logging method styles used in this project:

| Method | Use When |
|---|---|
| `_logger.ErrorAsync(message, exception, customer)` | Service methods with customer context |
| `_logger.InsertLogAsync(LogLevel.Error, title, body)` | Event consumers, ViewComponents |
| `_logger.WarningAsync(message)` | Non-critical issues |
| `_logger.InformationAsync(message)` | Debug/trace, gated by a settings flag |

### Static context logging (UPS HttpClientExtensions pattern):

When `ILogger` cannot be constructor-injected (static helpers, extension methods):

```csharp
var logger = EngineContext.Current.Resolve<ILogger>();
logger.Information($"UPS rates. Request: {request}");
```

**Rules:**
- Always use `Nop.Services.Logging.ILogger` — not `Microsoft.Extensions.Logging.ILogger`
- Always include the exception object in `ErrorAsync` — not just `ex.Message`
- Gate `InformationAsync` behind a plugin setting — never log info unconditionally
- Never log sensitive data (API keys, passwords, card numbers)

---

## Rule 35 — Settings Security

Plugin settings that contain API keys, secrets, or credentials must never be exposed
in view models or logs.

### Correct pattern — settings stored via `ISettingService`, never in source code:

```csharp
// Settings class — API key is a plain string property
public class GoogleAnalyticsSettings : ISettings
{
    public string GoogleId { get; set; }
    public string ApiSecret { get; set; }   // ← sensitive
    public string TrackingScript { get; set; }
}
```

### In admin views — use password input type for secrets:

```cshtml
@* Configure.cshtml — API secret rendered as password field *@
<input asp-for="ApiSecret" type="password" class="form-control" />
```

### In controllers — load settings per store scope, never hardcode:

```csharp
// Always load from ISettingService with store scope
var storeScope = await _storeContext.GetActiveStoreScopeConfigurationAsync();
var settings = await _settingService.LoadSettingAsync<GoogleAnalyticsSettings>(storeScope);

// Save individual properties — not the whole object — to support per-store overrides
await _settingService.SaveSettingAsync(settings, s => s.ApiSecret, clearCache: false);
await _settingService.ClearCacheAsync();
```

### Per-store override pattern (CheckMoneyOrder pattern):

```csharp
// Saves with per-store override flag — standard for all configurable settings
await _settingService.SaveSettingOverridablePerStoreAsync(
    settings,
    x => x.DescriptionText,
    model.DescriptionText_OverrideForStore,
    storeScope,
    clearCache: false   // batch — clear once at end
);
await _settingService.ClearCacheAsync();
```

**Rules:**
- Never store API keys or secrets in `plugin.json`, source code, or `appsettings.json`
- Always use `ISettingService` for plugin configuration storage
- Use `clearCache: false` on individual saves, then call `ClearCacheAsync()` once at the end
- Use `SaveSettingOverridablePerStoreAsync` for settings that can differ per store
- Render secret fields as `type="password"` in admin views
