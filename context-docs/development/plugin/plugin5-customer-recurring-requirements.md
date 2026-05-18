# Plugin 5 — Customer & Recurring Orders Requirements
**Plugin System Name:** `Marketplace.CustomerAndRecurring`  
**Project Name:** `Nop.Plugin.Misc.CustomerAndRecurring`  
**Output Folder:** `Misc.CustomerAndRecurring`  
**Plugin Type:** `IMiscPlugin`  
**WBS Coverage:** F-7.1, F-7.2, F-7.3, F-7.4, F-7.5, F-7.6, F-7.7

---

## Purpose

This plugin owns the buyer-side experience for the Al Basher marketplace. It separates
the registration flow for wholesalers and end consumers, extends the consumer
registration form, introduces named saved order lists as the foundation for recurring
orders, implements recurring order scheduling with automatic placement, and integrates
with a payment gateway for stored payment methods and automatic billing.

---

## nopCommerce Foundation to Build On

- `Customer` (`Nop.Core.Domain.Customers.Customer`) — base entity. Do not add columns
  to this table. Use `IGenericAttributeService` for extended customer fields.
- `CustomerRole` (`Nop.Core.Domain.Customers.CustomerRole`) — use existing role system.
  Create two new roles during `InstallAsync`: `Wholesaler` and `Consumer`.
- `ShoppingCartItem` with `ShoppingCartType.Wishlist` — nopCommerce wishlist exists
  but does not support named lists or push-to-cart. This plugin builds on top of it
  with a separate entity.
- `RecurringPayment` (`Nop.Core.Domain.Orders.RecurringPayment`) — exists in nopCommerce
  but is tied to a single product. Do not use it for marketplace recurring orders.
  This plugin implements its own recurring schedule entity.
- `ICustomerService` — use for customer lookup, role assignment.
- `IShoppingCartService` — use `AddToCartAsync` when pushing a saved list to cart.
- `IOrderProcessingService` — use `PlaceOrderAsync` for automatic recurring order
  placement.
- `IWorkflowMessageService` — send price change and availability alert emails.
- `IScheduleTaskService` — register the recurring order placement task.
- `IGenericAttributeService` — store extended registration fields on the customer.
- `ISettingService` — store plugin settings.
- `IRepository<T>` — all data access to plugin-owned tables.

---

## Dependency on Plugin 1

```json
"DependsOnSystemNames": ["Marketplace.SupplierManagement"]
```

- Reads `MasterProductMapping` to check product availability for alerts (F-7.6).
- Reads `IMarketplacePriceService` for price change detection (F-7.5).
- Never reference Plugin 1's project assembly directly.

---

## New Database Tables Required

### `SavedOrderList` table
A named, reusable list of products saved by a buyer.

```csharp
public class SavedOrderList : BaseEntity
{
    public int CustomerId { get; set; }        // FK to Customer.Id
    public string Name { get; set; }           // buyer-given name, e.g. "Weekly Groceries"
    public bool IsRecurringEnabled { get; set; }
    public int RecurringScheduleId { get; set; } // enum: Weekly=10, Biweekly=20, Monthly=30; 0 if not recurring
    public DateTime? NextOrderDateUtc { get; set; }
    public DateTime? LastOrderDateUtc { get; set; }
    public bool IsActive { get; set; }
    public DateTime CreatedOnUtc { get; set; }
    public DateTime UpdatedOnUtc { get; set; }
}
```

### `SavedOrderListItem` table
Individual product entries within a saved order list.

```csharp
public class SavedOrderListItem : BaseEntity
{
    public int SavedOrderListId { get; set; }      // FK to SavedOrderList.Id
    public int ProductId { get; set; }             // FK to Product.Id (master catalog)
    public int VendorId { get; set; }              // preferred supplier (FK to Vendor.Id)
    public int Quantity { get; set; }
    public decimal PriceSnapshotAtSave { get; set; }  // price when item was saved (for change detection)
    public DateTime AddedOnUtc { get; set; }
}
```

### `RecurringOrderLog` table
Audit trail of automatic recurring order placements.

```csharp
public class RecurringOrderLog : BaseEntity
{
    public int SavedOrderListId { get; set; }      // FK to SavedOrderList.Id
    public int CustomerId { get; set; }            // FK to Customer.Id
    public int? OrderId { get; set; }              // FK to Order.Id (null if placement failed)
    public int PlacementStatusId { get; set; }     // enum: Success=10, Failed=20, SkippedUnavailable=30
    public string FailureReason { get; set; }
    public DateTime AttemptedOnUtc { get; set; }
}
```

### `StoredPaymentMethod` table
Stores a tokenised payment method reference for recurring billing.

```csharp
public class StoredPaymentMethod : BaseEntity
{
    public int CustomerId { get; set; }            // FK to Customer.Id
    public string PaymentMethodSystemName { get; set; }  // e.g. "Payments.Stripe"
    public string GatewayCustomerId { get; set; }  // gateway's customer ID (e.g. Stripe cus_xxx)
    public string GatewayPaymentMethodId { get; set; }  // gateway's payment method ID (e.g. Stripe pm_xxx)
    public string CardBrand { get; set; }          // e.g. "Visa" — display only
    public string CardLast4 { get; set; }          // last 4 digits — display only
    public string CardExpiryMonth { get; set; }
    public string CardExpiryYear { get; set; }
    public bool IsDefault { get; set; }
    public DateTime CreatedOnUtc { get; set; }
}
```

---

## Settings Class

```csharp
public class CustomerAndRecurringSettings : ISettings
{
    // System name of the payment gateway plugin used for recurring billing
    // e.g. "Payments.Stripe" — set by admin after gateway plugin is installed
    public string RecurringPaymentGatewaySystemName { get; set; }

    // Whether wholesaler registration requires admin approval before login
    public bool WholesalerRequiresApproval { get; set; }  // default: true

    // Maximum number of saved order lists per customer
    public int MaxSavedOrderListsPerCustomer { get; set; }  // default: 10

    // Maximum number of items per saved order list
    public int MaxItemsPerSavedOrderList { get; set; }  // default: 50
}
```

---

## Feature Requirements

### F-7.1 — Registration Separation (Wholesaler vs Consumer)

**Login page — role selector:**
- Inject a radio button group ("I am a: Wholesaler / Consumer") into the login page
  using a widget zone (`login_page_top` or similar).
- The selection routes the user to the correct registration page:
  - Wholesaler → `/register/wholesaler`
  - Consumer → `/register/consumer`
- If a user navigates directly to `/register`, redirect to the login page with the
  role selector visible.

**Customer roles:**
- Create two `CustomerRole` records during `InstallAsync`:
  - `Wholesaler` (system name: `Marketplace.Wholesaler`)
  - `Consumer` (system name: `Marketplace.Consumer`)
- On registration completion, assign the appropriate role to the new customer.
- Wholesaler registration: if `WholesalerRequiresApproval = true`, set
  `Customer.Active = false` after registration. Admin must activate the account.
  Send admin notification email using message template `Marketplace.WholesalerRegistered`.
- Consumer registration: `Customer.Active = true` immediately (standard flow).

**Admin — user type management:**
- Admin can filter the customer list by role (Wholesaler / Consumer) using the
  existing nopCommerce customer list page's role filter. No new admin page needed
  for this — the existing `CustomerRole` filter covers it.
- Admin can manually assign or remove the Wholesaler/Consumer role from any customer.

---

### F-7.2 — Consumer Registration Form

**Extended registration fields:**
- The consumer registration page (`/register/consumer`) uses the standard nopCommerce
  registration form as a base and adds marketplace-specific fields.
- Additional fields (store in `IGenericAttributeService` on the customer):
  - `DeliveryAddress` (full address — can use nopCommerce address entity)
  - `PreferredDeliveryInstructions` (text)
  - `BusinessName` (optional — for small business consumers)
  - `HowDidYouHearAboutUs` (dropdown: Social Media / Referral / Search / Other)
- All additional fields are stored via `IGenericAttributeService` using keys prefixed
  with `Marketplace.Customer.`.
- Implement the extended form as a view component injected into the registration page
  widget zone, or as a standalone registration controller action that wraps the core
  registration service.

**Wholesaler registration form:**
- The wholesaler registration page (`/register/wholesaler`) captures:
  - Standard nopCommerce registration fields (name, email, password)
  - `BusinessRegistrationNumber`
  - `BusinessName`
  - `ContactPhone`
  - `BusinessAddress`
  - `TradeLicenseNumber` (optional)
- Store extended fields via `IGenericAttributeService`.
- On submission, create the customer account and set `Active = false` if approval
  is required. Create a notification for admin.

---

### F-7.3 — Saved Order List (Favourite / Frequent Orders)

**Buyer account — saved lists:**
- Buyer account page: "Saved Orders" lists all `SavedOrderList` records for the
  current customer.
- Each list shows: name, item count, recurring status, next order date (if recurring).
- Actions per list: View / Edit / Delete / Push to Cart / Enable Recurring.

**Create / edit saved list:**
- Buyer can create a new named list.
- Buyer can add items to a list from the product detail page ("Save to List" button)
  or from the active cart ("Save Cart as List" button).
- When adding an item, capture the currently selected supplier (`VendorId`) as the
  preferred supplier for that list item.
- Capture `PriceSnapshotAtSave` = current display price (after markup) at the time
  of saving. This is used for price change detection (F-7.5).
- Buyer can edit quantity or remove items from a list.
- Buyer can rename a list.

**Push to cart:**
- "Push to Cart" action on a saved list:
  1. For each `SavedOrderListItem`, check if the product is still available
     (`MasterProductMapping.Published = true` and `IsInStock = true` for the
     preferred vendor).
  2. If available, call `IShoppingCartService.AddToCartAsync` for the item.
  3. If unavailable, skip the item and show a warning to the buyer listing skipped
     items.
  4. After pushing, redirect to the cart page.
- Do not clear the saved list after pushing — it remains for future reuse.

**Availability check on view:**
- When a buyer views a saved list, show a status indicator per item:
  - Green: available from preferred supplier
  - Yellow: available but from a different supplier (preferred supplier's mapping
    is out of stock or unpublished)
  - Red: not available from any supplier

---

### F-7.4 — Recurring Order Scheduling

**Enable recurring on a saved list:**
- From the saved list detail page, buyer can enable recurring and set:
  - Schedule: Weekly / Biweekly / Monthly (radio buttons)
  - Start date (date picker — must be in the future)
- On save, set `SavedOrderList.IsRecurringEnabled = true`,
  `SavedOrderList.RecurringScheduleId`, and calculate `NextOrderDateUtc` from the
  start date.

**Automatic order placement (schedule task):**
- Register `RecurringOrderPlacementTask` as a `IScheduleTask` running every hour.
- Task logic:
  1. Query all `SavedOrderList` records where `IsRecurringEnabled = true`,
     `IsActive = true`, and `NextOrderDateUtc <= DateTime.UtcNow`.
  2. For each list:
     a. Load the customer and their `StoredPaymentMethod` (default).
     b. If no stored payment method, log failure in `RecurringOrderLog` with reason
        "No stored payment method" and skip.
     c. Push all available items to a temporary cart for the customer.
     d. If no items are available, log as `SkippedUnavailable` and advance
        `NextOrderDateUtc` to the next recurrence date.
     e. Call `IOrderProcessingService.PlaceOrderAsync` with the customer's stored
        payment method token.
     f. Set a custom order attribute `Marketplace.IsRecurringOrder = true` on the
        placed order (via `IGenericAttributeService`) so Plugin 4 can identify it.
     g. On success: create `RecurringOrderLog` with `PlacementStatus = Success`,
        update `SavedOrderList.LastOrderDateUtc`, advance `NextOrderDateUtc`.
     h. On failure: create `RecurringOrderLog` with `PlacementStatus = Failed` and
        the exception message. Do NOT advance `NextOrderDateUtc` — retry on next run.
     i. Send order confirmation email to buyer via `IWorkflowMessageService`.

**Next order date calculation:**
- Weekly: `NextOrderDateUtc = LastOrderDateUtc + 7 days`
- Biweekly: `NextOrderDateUtc = LastOrderDateUtc + 14 days`
- Monthly: `NextOrderDateUtc = LastOrderDateUtc.AddMonths(1)`

---

### F-7.5 — Price Change Notifications

**Detection:**
- The `RecurringOrderPlacementTask` (or a separate daily `PriceCheckTask`) checks
  each `SavedOrderListItem.PriceSnapshotAtSave` against the current display price
  from `IMarketplacePriceService.CalculateDisplayPriceAsync`.
- If the current price differs from the snapshot by more than a configurable
  threshold (default: any change), trigger a notification.

**Notification:**
- Send email to the buyer using message template `Marketplace.PriceChangeAlert`.
- Email content: product name, old price (snapshot), new price, saved list name.
- Update `SavedOrderListItem.PriceSnapshotAtSave` to the new price after sending
  the notification (to avoid repeated alerts for the same price change).
- Do not send more than one price change notification per item per day.

**Implementation note:**
- The WBS marks this as "Free PWA Plugin" — check if a PWA plugin is installed in
  the solution that provides email notification hooks. If so, integrate with it.
  If not, implement the email notification directly using `IWorkflowMessageService`.

---

### F-7.6 — Product Availability Alerts

**Detection:**
- Listen to `EntityUpdatedEvent<MasterProductMapping>` (Plugin 1 entity) via
  `IConsumer<EntityUpdatedEvent<MasterProductMapping>>`.
- When a `MasterProductMapping` is updated and `Published` changes to `false` or
  `IsInStock` changes to `false`, check if any `SavedOrderListItem` references
  this product + vendor combination.

**Notification:**
- For each affected `SavedOrderListItem`, send an email to the buyer using message
  template `Marketplace.ProductUnavailableAlert`.
- Email content: product name, supplier name, saved list name, link to the saved
  list to update it.
- Do not send duplicate alerts: track last alert sent per `(SavedOrderListItem.Id,
  date)` using `IGenericAttributeService` or a simple flag on the item.

**Implementation note:**
- Same note as F-7.5 regarding PWA plugin integration.

---

### F-7.7 — Recurring Billing & Payment Gateway Integration

**Gateway selection:**
- The gateway is configured by admin via `CustomerAndRecurringSettings.RecurringPaymentGatewaySystemName`.
- This plugin does not implement a payment gateway itself. It integrates with an
  existing nopCommerce payment plugin (e.g., Stripe plugin).
- If the Stripe plugin is installed, use its API to create a Stripe Customer and
  attach a Payment Method during the buyer's stored payment method setup flow.

**Stored payment method setup:**
- Buyer account page: "Payment Methods" shows saved cards (from `StoredPaymentMethod`).
- "Add Card" flow:
  1. Redirect buyer to the gateway's hosted payment method setup page (e.g., Stripe
     SetupIntent flow).
  2. On return, receive the gateway's payment method token.
  3. Create a `StoredPaymentMethod` record with the token, card brand, last 4, expiry.
  4. Never store raw card numbers — only gateway tokens.
- Buyer can set a default card and delete saved cards.
- Deleting a card calls the gateway API to detach the payment method before removing
  the `StoredPaymentMethod` record.

**Automatic charge during recurring order placement:**
- In `RecurringOrderPlacementTask`, when calling `IOrderProcessingService.PlaceOrderAsync`,
  pass the stored payment method token as the payment info.
- The exact mechanism depends on the gateway plugin's API. For Stripe: create a
  PaymentIntent with `customer` and `payment_method` parameters and `confirm = true`.
- If the charge fails (card declined, insufficient funds), log the failure in
  `RecurringOrderLog` and send a payment failure notification to the buyer using
  message template `Marketplace.RecurringPaymentFailed`.

**Gateway plugin dependency:**
- Do not hardcode Stripe-specific code in this plugin. Abstract the gateway
  interaction behind an `IRecurringPaymentGateway` interface:
  ```csharp
  public interface IRecurringPaymentGateway
  {
      Task<string> CreateCustomerAsync(Customer customer);
      Task<StoredPaymentMethodResult> AttachPaymentMethodAsync(string gatewayCustomerId, string paymentMethodToken);
      Task<PaymentResult> ChargeAsync(string gatewayCustomerId, string paymentMethodId, decimal amount, string currency, string orderReference);
      Task DetachPaymentMethodAsync(string gatewayPaymentMethodId);
  }
  ```
- Provide a `StripeRecurringPaymentGateway` implementation if Stripe is the chosen
  gateway. If a different gateway is selected, a new implementation is added without
  changing the interface.
- Register the correct implementation in `NopStartup.cs` based on
  `CustomerAndRecurringSettings.RecurringPaymentGatewaySystemName`.

---

## Admin UI Pages Required

| Page | Route | Description |
|---|---|---|
| Plugin Configuration | `Admin/CustomerAndRecurring/Configure` | Gateway selection, approval settings |
| Wholesaler Approvals | `Admin/CustomerAndRecurring/WholesalerApprovals` | Pending wholesaler accounts |
| Recurring Order Logs | `Admin/CustomerAndRecurring/RecurringLogs` | Audit trail of auto-placed orders |

## Buyer Account UI Pages Required

| Page | Route | Description |
|---|---|---|
| Saved Orders | `Customer/SavedOrders` | List saved order lists |
| Saved Order Detail | `Customer/SavedOrderDetail/{id}` | View/edit items, push to cart, enable recurring |
| Payment Methods | `Customer/PaymentMethods` | Manage stored payment methods |
| My Returns | `Customer/MyReturns` | Return request history (shared with Plugin 4) |

## Public Registration Pages Required

| Page | Route | Description |
|---|---|---|
| Wholesaler Registration | `register/wholesaler` | Extended wholesaler signup form |
| Consumer Registration | `register/consumer` | Extended consumer signup form |

---

## Event Consumers Required

```csharp
public class EventConsumer :
    IConsumer<CustomerRegisteredEvent>,                        // assign role after registration
    IConsumer<EntityUpdatedEvent<MasterProductMapping>>        // trigger availability alerts (F-7.6)
```

Note: `MasterProductMapping` is defined in Plugin 1. Consume its update event via
the shared event bus. No direct project reference to Plugin 1.

---

## Schedule Tasks Required

| Task Class | Type Name Constant | Default Interval |
|---|---|---|
| `RecurringOrderPlacementTask` | `Marketplace.CustomerAndRecurring.Services.RecurringOrderPlacementTask` | Every 60 minutes |
| `PriceCheckTask` | `Marketplace.CustomerAndRecurring.Services.PriceCheckTask` | Every 24 hours |

---

## Install / Uninstall Checklist

**InstallAsync:**
1. Create DB tables via `AutoReversingMigration` tagged `MigrationProcessType.Installation`:
   `SavedOrderList`, `SavedOrderListItem`, `RecurringOrderLog`, `StoredPaymentMethod`.
2. Create `CustomerRole` records: `Wholesaler` (system name `Marketplace.Wholesaler`),
   `Consumer` (system name `Marketplace.Consumer`).
3. Save default `CustomerAndRecurringSettings`.
4. Register schedule tasks: `RecurringOrderPlacementTask`, `PriceCheckTask`.
5. Add message templates: `Marketplace.WholesalerRegistered`,
   `Marketplace.PriceChangeAlert`, `Marketplace.ProductUnavailableAlert`,
   `Marketplace.RecurringPaymentFailed`.
6. Add locale resources with prefix `Plugins.Misc.CustomerAndRecurring.`.
7. Call `await base.InstallAsync()` last.

**UninstallAsync:**
1. Remove schedule tasks.
2. Delete locale resources.
3. Remove message templates.
4. Call `await base.UninstallAsync()` last.
5. Do NOT drop DB tables on uninstall.
6. Do NOT delete the `Wholesaler` and `Consumer` customer roles on uninstall —
   customers may already be assigned to them.

---

## Folder Structure

```
Nop.Plugin.Misc.CustomerAndRecurring/
├── plugin.json
├── Nop.Plugin.Misc.CustomerAndRecurring.csproj
├── CustomerAndRecurringPlugin.cs        ← BasePlugin + IMiscPlugin
├── CustomerAndRecurringSettings.cs      ← ISettings
├── CustomerAndRecurringDefaults.cs      ← static constants
├── Events.cs                            ← IConsumer<T> implementations
├── logo.png
├── Controllers/
│   ├── CustomerAndRecurringAdminController.cs
│   ├── RegistrationController.cs        ← wholesaler + consumer registration pages
│   └── SavedOrderController.cs          ← buyer saved order list management
├── Components/
│   ├── LoginRoleSelectorViewComponent.cs    ← login page radio button injection
│   └── SaveToListButtonViewComponent.cs     ← product detail "Save to List" button
├── Domain/
│   ├── SavedOrderList.cs
│   ├── SavedOrderListItem.cs
│   ├── RecurringOrderLog.cs
│   ├── StoredPaymentMethod.cs
│   ├── RecurringSchedule.cs             ← enum: Weekly=10, Biweekly=20, Monthly=30
│   └── PlacementStatus.cs               ← enum
├── Data/
│   ├── SavedOrderListBuilder.cs
│   ├── SavedOrderListItemBuilder.cs
│   ├── RecurringOrderLogBuilder.cs
│   └── StoredPaymentMethodBuilder.cs
├── Factories/
│   ├── ISavedOrderModelFactory.cs
│   └── SavedOrderModelFactory.cs
├── Infrastructure/
│   ├── NopStartup.cs
│   └── RouteProvider.cs
├── Migrations/
│   └── SchemaMigration.cs
├── Models/
│   ├── ConfigurationModel.cs
│   ├── WholesalerRegistrationModel.cs
│   ├── ConsumerRegistrationModel.cs
│   ├── SavedOrderListModel.cs
│   ├── SavedOrderListDetailModel.cs
│   ├── SavedOrderListItemModel.cs
│   ├── StoredPaymentMethodModel.cs
│   └── RecurringOrderLogModel.cs
├── Services/
│   ├── ISavedOrderService.cs
│   ├── SavedOrderService.cs
│   ├── IRecurringPaymentGateway.cs      ← gateway abstraction interface
│   ├── StripeRecurringPaymentGateway.cs ← Stripe implementation (if Stripe chosen)
│   ├── RecurringOrderPlacementTask.cs   ← IScheduleTask
│   └── PriceCheckTask.cs               ← IScheduleTask
└── Views/
    ├── _ViewImports.cshtml
    ├── Configure.cshtml
    ├── WholesalerApprovals.cshtml
    ├── RecurringLogs.cshtml
    ├── WholesalerRegistration.cshtml
    ├── ConsumerRegistration.cshtml
    ├── SavedOrders.cshtml
    ├── SavedOrderDetail.cshtml
    └── PaymentMethods.cshtml
```

---

## Key Constraints and Rules

- Follow all rules in `plugin-rule.md` without exception.
- Never store raw card numbers or CVV in `StoredPaymentMethod` or anywhere in the
  database. Store only gateway tokens and display-safe metadata (brand, last 4, expiry).
- `RecurringOrderPlacementTask` must be idempotent: if an order was already placed
  for a given `SavedOrderList` on the current recurrence date (check `LastOrderDateUtc`),
  do not place a duplicate order.
- The `Marketplace.IsRecurringOrder` generic attribute key must match exactly the
  constant used in Plugin 4's `Defaults.cs`. Value: `"Marketplace.IsRecurringOrder"`.
- `IRecurringPaymentGateway` must be registered conditionally in `NopStartup.cs`
  based on `CustomerAndRecurringSettings.RecurringPaymentGatewaySystemName`. If the
  setting is empty or the gateway plugin is not installed, log a warning and skip
  registration — do not throw on startup.
- The `PriceCheckTask` must not send more than one price change notification per
  `SavedOrderListItem` per 24-hour period. Use `IGenericAttributeService` to track
  the last notification timestamp per item.
- All locale resource keys must use the prefix `Plugins.Misc.CustomerAndRecurring.`.
- `plugin.json` must declare:
  ```json
  "DependsOnSystemNames": ["Marketplace.SupplierManagement"]
  ```
- If the Stripe NuGet package is added for `StripeRecurringPaymentGateway`, set
  `CopyLocalLockFileAssemblies = true` in the `.csproj` and pin the Stripe package
  to an exact version.
