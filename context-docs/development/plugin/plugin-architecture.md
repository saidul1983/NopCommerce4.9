# Plugin Architecture

> Derived from reading the actual plugins in this project:
> `Nop.Plugin.Widgets.GoogleAnalytics`, `Nop.Plugin.Misc.Brevo`,
> `Nop.Plugin.Payments.CheckMoneyOrder`, and the core infrastructure in
> `Nop.Core.Infrastructure` and `Nop.Services.Plugins`.

---

## How Plugins Are Discovered and Loaded

Plugin loading happens in two phases at startup, before the DI container is built.

### Phase 1 — Assembly Loading
`ApplicationPartManager.InitializePlugins(pluginConfig)` runs first:

1. Scans `src/Presentation/Nop.Web/Plugins/` for subdirectories
2. Reads `plugin.json` in each directory
3. Loads the plugin DLL into the `ApplicationPartManager`
4. Plugin controllers, views, and view components become available to MVC routing
5. Checks `App_Data/plugins.json` to mark each plugin as Installed or not
6. Stores all `PluginDescriptor` objects in `Singleton<IPluginsInfo>.Instance`

### Phase 2 — Type Discovery
`WebAppTypeFinder` scans all loaded assemblies (including plugin assemblies) for:

| Interface | What Happens |
|---|---|
| `INopStartup` | Called to register plugin services in DI |
| `IConsumer<T>` | Registered as scoped event consumers automatically |
| `ISettings` | Registered as scoped, loaded per store |
| `IOrderedMapperProfile` | AutoMapper profiles registered automatically |

No manual registration is needed for these — they are auto-discovered.

---

## Plugin Descriptor (`PluginDescriptor`)

Populated from `plugin.json` at startup. Key properties:

```
SystemName                // unique identifier e.g. "Widgets.GoogleAnalytics"
FriendlyName              // display name
Group                     // category e.g. "Widgets"
Version                   // e.g. "4.90.2"
SupportedVersions         // ["4.90"]
DependsOn                 // system names of required plugins
LimitedToStores           // empty = all stores
LimitedToCustomerRoles    // empty = all roles
Installed                 // runtime flag, not in plugin.json
PluginType                // the main plugin class type, resolved at runtime
ReferencedAssembly        // the loaded assembly
```

### Getting a Plugin Instance at Runtime
```csharp
var descriptor = await _pluginService.GetPluginDescriptorBySystemNameAsync<IPaymentMethod>(
    "Payments.CheckMoneyOrder");
var plugin = descriptor.Instance<IPaymentMethod>();
// Instance<T>() resolves through DI — all constructor dependencies are injected
```

---

## Plugin Interface Hierarchy

As seen in this project's actual plugins:

```
IPlugin
  └── BasePlugin (abstract — always inherit this)
        │
        ├── IPaymentMethod              → CheckMoneyOrder, Manual, PayPalCommerce
        │     ProcessPaymentAsync, PostProcessPaymentAsync,
        │     HidePaymentMethodAsync, GetAdditionalHandlingFeeAsync,
        │     ValidatePaymentFormAsync, GetPaymentInfoAsync,
        │     GetPublicViewComponent()
        │     SupportCapture, SupportRefund, SupportVoid,
        │     RecurringPaymentType, PaymentMethodType, SkipPaymentInfo
        │
        ├── IWidgetPlugin               → GoogleAnalytics, FacebookPixel, Swiper, AccessiBe
        │     GetWidgetZonesAsync()
        │     GetWidgetViewComponent(widgetZone)
        │     HideInWidgetList (property)
        │
        ├── IShippingRateComputationMethod → FixedByWeightByTotal, UPS
        │     GetShippingOptionsAsync, GetFixedRateAsync
        │
        ├── ITaxProvider                → FixedOrByCountryStateZip, Avalara
        │     GetTaxRateAsync
        │
        ├── IExternalAuthenticationMethod → ExternalAuth.Facebook
        │     GetPublicViewComponent()
        │
        ├── IMultiFactorAuthenticationMethod → GoogleAuthenticator
        │     GetPublicViewComponent(), IsValidationRequestAsync
        │
        ├── IDiscountRequirementRule    → DiscountRules.CustomerRoles
        │     CheckRequirementAsync, GetConfigurationUrl
        │
        ├── IExchangeRateProvider       → CurrencyExchange.ECB
        │     GetCurrencyLiveRatesAsync
        │
        ├── IPickupPointProvider        → Pickup.PickupInStore
        │     GetPickupPointsAsync
        │
        ├── ISearchProvider             → Search.Lucene
        │     SearchProductsAsync, IndexProductAsync
        │
        └── IMiscPlugin                 → Brevo, Dynamics365, RFQ, Omnisend, etc.
              (no additional interface members — general-purpose)
```

**A plugin can implement multiple interfaces.** For example, `BrevoPlugin` implements
both `IMiscPlugin` and `IWidgetPlugin` simultaneously.

---

## Plugin Lifecycle

```
[DLL on disk, not in plugins.json]
        │  Admin uploads DLL + plugin.json to Plugins/ folder
        ▼
[Discovered — Not Installed]
        │  Admin clicks Install → app restart required
        ▼
  FluentMigrator up-migrations run (MigrationProcessType.Installation)
  plugin.InstallAsync() runs:
    - saves default settings via ISettingService
    - adds widget to ActiveWidgetSystemNames (widget plugins only)
    - registers schedule tasks (if needed)
    - adds locale resources via ILocalizationService
    - calls await base.InstallAsync()
        │
        ▼
[Installed — Active]
        │  Admin clicks Uninstall → app restart required
        ▼
  plugin.UninstallAsync() runs:
    - removes from ActiveWidgetSystemNames (widget plugins)
    - deletes settings via ISettingService.DeleteSettingAsync<T>()
    - deletes schedule tasks (if registered)
    - removes locale resources via ILocalizationService.DeleteLocaleResourcesAsync()
    - calls await base.UninstallAsync()
  FluentMigrator down-migrations run
        │
        ▼
[Discovered — Not Installed]
        │  Admin clicks Delete → app restart required
        ▼
[DLL removed from disk]
```

### Update Flow
When a new plugin version is uploaded:
1. FluentMigrator runs new up-migrations tagged `MigrationProcessType.Update`
2. `plugin.UpdateAsync(oldVersion, newVersion)` is called
3. `plugins.json` is updated with the new version

The Brevo plugin demonstrates this — `BrevoMigration` is tagged `MigrationProcessType.Update`
and handles renaming locale keys and settings from the old "Sendinblue" name.

---

## Plugin Manager Pattern

Each plugin type has a typed `IPluginManager<T>`. Use these in services — never query
`IPluginsInfo` directly.

```csharp
// Example: get all active payment methods for a customer in a store
var methods = await _paymentPluginManager.LoadActivePluginsAsync(customer, storeId);
```

Available plugin managers registered in DI:

| Manager | Plugin Type |
|---|---|
| `IPaymentPluginManager` | `IPaymentMethod` |
| `IShippingPluginManager` | `IShippingRateComputationMethod` |
| `ITaxPluginManager` | `ITaxProvider` |
| `IWidgetPluginManager` | `IWidgetPlugin` |
| `IAuthenticationPluginManager` | `IExternalAuthenticationMethod` |
| `IMultiFactorAuthenticationPluginManager` | `IMultiFactorAuthenticationMethod` |
| `IDiscountPluginManager` | `IDiscountRequirementRule` |
| `IExchangeRatePluginManager` | `IExchangeRateProvider` |
| `IPickupPluginManager` | `IPickupPointProvider` |
| `ISearchPluginManager` | `ISearchProvider` |

---

## Service Override Pattern

Plugins can **replace core services** by re-registering them in `NopStartup.cs`.
The Brevo plugin does this to replace the email sending service:

```csharp
// Brevo/Infrastructure/NopStartup.cs
services.AddScoped<IWorkflowMessageService, BrevoMessageService>();
```

Because `NopStartup.Order = 3000` runs after the core registration at `Order = 2000`,
the last registration wins in the DI container. This is the correct way to override
core behavior without modifying core code.

---

## Plugin Dependencies

If Plugin B requires Plugin A to be installed first:

```json
// Plugin B's plugin.json
{
  "DependsOnSystemNames": ["MyGroup.PluginA"]
}
```

`PluginService` validates this before queuing installation. If Plugin A is not installed,
an error is shown and Plugin B cannot be installed. The same check runs on uninstall.

---

## AutoMapper Profile Registration

If a plugin needs to map between domain entities and view models, create a profile
that implements `IOrderedMapperProfile`. It is auto-discovered — no manual registration:

```csharp
public class MapperConfiguration : Profile, IOrderedMapperProfile
{
    public int Order => 1;

    public MapperConfiguration()
    {
        CreateMap<ShippingByWeightByTotalRecord, ShippingByWeightByTotalModel>();
        CreateMap<ShippingByWeightByTotalModel, ShippingByWeightByTotalRecord>();
    }
}
```

Place in `Infrastructure/MapperConfiguration.cs`.

---

## `HideInWidgetList` Property

Widget plugins (`IWidgetPlugin`) have a `HideInWidgetList` property that controls
whether the plugin appears in the admin widget list page:

```csharp
// GoogleAnalyticsPlugin.cs — shown in widget list
public bool HideInWidgetList => false;

// BrevoPlugin.cs — hidden from widget list (it's primarily a misc plugin)
public bool HideInWidgetList => true;
```

Set `true` for plugins that implement `IWidgetPlugin` only as a secondary interface
(e.g., a misc plugin that also renders a tracking script). Set `false` for dedicated
widget plugins.

---

## `GetConfigurationPageUrl()` — Two Patterns

```csharp
// Pattern 1 — named route via INopUrlHelper (preferred, GoogleAnalytics)
public override string GetConfigurationPageUrl()
{
    return _nopUrlHelper.RouteUrl(GoogleAnalyticsDefaults.ConfigurationRouteName);
}

// Pattern 2 — string concatenation via IWebHelper (Brevo, CheckMoneyOrder)
public override string GetConfigurationPageUrl()
{
    return $"{_webHelper.GetStoreLocation()}Admin/Brevo/Configure";
}
```

Use Pattern 1 when a named route is registered in `RouteProvider.cs`.
Use Pattern 2 only when no named route exists.
