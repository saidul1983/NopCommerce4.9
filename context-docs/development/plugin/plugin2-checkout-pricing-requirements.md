# Plugin 2 — Checkout & Pricing Requirements
**Plugin System Name:** `Marketplace.CheckoutAndPricing`  
**Project Name:** `Nop.Plugin.Misc.CheckoutAndPricing`  
**Output Folder:** `Misc.CheckoutAndPricing`  
**Plugin Type:** `IMiscPlugin`  
**WBS Coverage:** F-4.1, F-4.2, F-5.1, F-5.2, F-5.3, F-5.4

---

## Purpose

This plugin owns the entire buyer-facing order flow for the marketplace. It handles
supplier-aware product display, cart management with supplier identity, checkout price
breakdown, automatic shipment splitting by supplier on order placement, per-supplier
sub-order status management, per-supplier delivery configuration, and per-shipment
delivery option selection at checkout.

This is the highest-risk plugin in the system because it overrides core nopCommerce
checkout services. Regression testing of the standard checkout flow is mandatory after
any change.

---

## nopCommerce Foundation to Build On

- `ShoppingCartItem` (`Nop.Core.Domain.Orders`) — does not carry `VendorId`. This
  plugin must add supplier identity to cart items via a custom table or generic
  attribute. See "Cart Item Supplier Identity" section below.
- `OrderItem` (`Nop.Core.Domain.Orders`) — has no `VendorId` field in the base entity.
  Supplier identity on order items must also be tracked via a plugin table.
- `Shipment` (`Nop.Core.Domain.Shipping`) — exists. This plugin creates one `Shipment`
  per supplier on order placement.
- `ShipmentItem` (`Nop.Core.Domain.Shipping`) — links shipment to order items.
- `IOrderProcessingService` — override `PlaceOrderAsync` post-processing to trigger
  shipment splitting. Use the event `OrderPlacedEvent` instead of overriding the
  service directly where possible.
- `IOrderTotalCalculationService` — override to inject markup, service fee, and
  delivery fee into the order total calculation.
- `IShoppingCartService` — use as-is for cart operations; extend via events.
- `IShipmentService` — use for creating and managing `Shipment` records.
- `IWorkflowMessageService` — use for per-supplier shipment notification emails.
- `IVendorService` — use to resolve vendor details for display and email.
- `IMarketplacePriceService` — from Plugin 1. Resolved via DI. Never reference
  Plugin 1's project directly.

---

## Dependency on Plugin 1

This plugin depends on `Marketplace.SupplierManagement` (Plugin 1) being installed.
Declare this in `plugin.json`:

```json
"DependsOnSystemNames": ["Marketplace.SupplierManagement"]
```

At runtime, resolve `IMarketplacePriceService` and `ISupplierService` via DI — they
are registered by Plugin 1's `NopStartup.cs`.

---

## New Database Tables Required

### `CartItemSupplierMapping` table
Tracks which supplier (vendor) a cart item is associated with, since `ShoppingCartItem`
has no `VendorId` field.

```csharp
public class CartItemSupplierMapping : BaseEntity
{
    public int ShoppingCartItemId { get; set; }  // FK to ShoppingCartItem.Id
    public int VendorId { get; set; }            // FK to Vendor.Id
    public int MasterProductMappingId { get; set; } // FK to MasterProductMapping.Id
}
```

### `OrderItemSupplierMapping` table
Tracks supplier identity on order items after checkout.

```csharp
public class OrderItemSupplierMapping : BaseEntity
{
    public int OrderItemId { get; set; }   // FK to OrderItem.Id
    public int VendorId { get; set; }      // FK to Vendor.Id
    public decimal SupplierBasePrice { get; set; }   // price before markup
    public decimal MarkupPercentage { get; set; }    // markup applied at time of order
    public decimal MarkupAmount { get; set; }        // calculated markup amount
}
```

### `SupplierShipment` table
Represents a per-supplier sub-order within a parent nopCommerce `Order`.

```csharp
public class SupplierShipment : BaseEntity
{
    public int OrderId { get; set; }           // FK to Order.Id
    public int VendorId { get; set; }          // FK to Vendor.Id
    public int ShipmentId { get; set; }        // FK to Shipment.Id (nopCommerce shipment)
    public int SubOrderStatusId { get; set; }  // enum: Processing=10, Shipped=20, Delivered=30, Cancelled=40
    public int DeliveryMethodId { get; set; }  // enum: SupplierDelivery=10, PlatformDelivery=20, Collection=30, UberDelivery=40
    public decimal DeliveryFee { get; set; }
    public DateTime CreatedOnUtc { get; set; }
    public DateTime UpdatedOnUtc { get; set; }
    public string SupplierNotes { get; set; }
}
```

### `SupplierDeliveryConfig` table
Admin-configured delivery settings per supplier.

```csharp
public class SupplierDeliveryConfig : BaseEntity
{
    public int VendorId { get; set; }                  // FK to Vendor.Id
    public bool OffersOwnDelivery { get; set; }
    public bool OffersCollection { get; set; }
    public int DeliveryFeeTypeId { get; set; }         // enum: Fixed=10, PerMile=20, Custom=30
    public decimal FixedDeliveryFee { get; set; }      // used when DeliveryFeeType = Fixed
    public decimal PerMileRate { get; set; }           // used when DeliveryFeeType = PerMile
    public string CustomFeeRule { get; set; }          // JSON or expression for custom rules
    public string StandardDeliveryRange { get; set; }  // e.g. "3–5 days"
    public string CollectionAddress { get; set; }
    public DateTime UpdatedOnUtc { get; set; }
}
```

---

## Feature Requirements

### F-4.1 — Browse & Display: Supplier-wise Pricing & Add to Cart

**Product listing page:**
- The standard nopCommerce product listing page shows one card per master catalog
  product. No change to the listing page layout is required.
- Product cards show the **lowest available supplier price** (after markup) as the
  display price. This is calculated by querying all `MasterProductMapping` records
  for the product where `Published = true` and `IsInStock = true`, applying markup
  via `IMarketplacePriceService.CalculateDisplayPriceAsync`, and taking the minimum.

**Product detail page:**
- Override or extend the product detail page to show a **supplier comparison table**
  below the main product info.
- The table lists all suppliers who have an active, in-stock mapping for this product.
  Columns: Supplier Name, Price (after markup), Delivery Available, Collection Available.
- Buyer selects a supplier from the table before clicking "Add to Cart".
- If only one supplier is available, pre-select them automatically.
- The selected `VendorId` and `MasterProductMappingId` are passed as hidden form
  fields on the add-to-cart form.
- Implement this as a `IWidgetPlugin` widget zone injection into
  `productdetails_overview_bottom` widget zone, or override the product detail
  view component — choose the approach that avoids modifying core views.

**Add to Cart:**
- Intercept the add-to-cart action. After `IShoppingCartService.AddToCartAsync`
  creates the `ShoppingCartItem`, immediately insert a `CartItemSupplierMapping`
  record linking the new cart item to the selected vendor and mapping.
- If no supplier is selected (buyer bypassed the selection), return a validation
  error: "Please select a supplier before adding to cart."
- Use `IConsumer<EntityInsertedEvent<ShoppingCartItem>>` to react to cart item
  creation and create the `CartItemSupplierMapping` record. Pass the selected
  vendor ID via a temporary mechanism (e.g., `IHttpContextAccessor` request data
  or a custom checkout attribute).

---

### F-4.2 — Multi-Supplier Checkout

**Cart page:**
- Group cart items by supplier in the cart display. Each supplier group shows:
  - Supplier name
  - Items with their display prices (base price + markup)
  - Subtotal for that supplier's items
- Implement via a widget zone injection into the cart summary area or a view
  component override.

**Checkout — order total calculation:**
- Override `IOrderTotalCalculationService` by registering a replacement in
  `NopStartup.cs`:
  ```csharp
  services.AddScoped<IOrderTotalCalculationService, MarketplaceOrderTotalCalculationService>();
  ```
- `MarketplaceOrderTotalCalculationService` wraps the original service and adds:
  1. For each cart item, replace the product price with the markup-applied price
     from `IMarketplacePriceService.CalculateDisplayPriceAsync`.
  2. Add the service fee: `IMarketplacePriceService.CalculateServiceFeeAsync(subtotal)`.
  3. Add delivery fees per supplier shipment (from `SupplierDeliveryConfig` or
     platform flat fee from `SupplierManagementSettings`).
- The checkout order summary must display a clear breakdown:
  - Supplier subtotals (one line per supplier)
  - Platform markup (shown as included in price or as a separate line — confirm with client)
  - Service fee (line item)
  - Delivery fee per supplier shipment (line item per supplier)
  - Order total

**Checkout — price breakdown display:**
- Inject a custom order summary component into the checkout confirmation page using
  a widget zone (`checkout_confirm_top` or similar).
- The component reads `CartItemSupplierMapping` records for the current cart and
  renders the full breakdown.

---

### F-5.1 — Supplier-wise Shipment Splitting

**Trigger:**
- Listen to `OrderPlacedEvent` via `IConsumer<OrderPlacedEvent>`.
- On order placement, group `OrderItem` records by supplier using
  `OrderItemSupplierMapping` records.

**Splitting logic:**
- For each distinct `VendorId` in the order:
  1. Create a nopCommerce `Shipment` record via `IShipmentService`.
  2. Add `ShipmentItem` records for all `OrderItem` records belonging to that vendor.
  3. Create a `SupplierShipment` record linking the `Order`, `Vendor`, and `Shipment`
     with initial status `Processing`.
  4. Populate `OrderItemSupplierMapping` records for each order item with the
     supplier base price and markup percentage captured at time of order.

**Supplier email notification:**
- After creating each `SupplierShipment`, send an email to the supplier's vendor
  email address containing only their items.
- Use `IWorkflowMessageService` with a custom message template
  `Marketplace.SupplierNewShipment` added during `InstallAsync`.
- The email must include: order number, list of items (product name, quantity, price),
  buyer delivery address, and selected delivery method for their shipment.

---

### F-5.2 — Shipment Status Management

**Supplier portal — shipment management:**
- Supplier portal page: "My Orders" lists all `SupplierShipment` records for the
  current supplier, grouped by status.
- Supplier can update the status of their shipment:
  - `Processing` → `Shipped` (requires tracking number input)
  - `Shipped` → `Delivered`
  - Any non-terminal state → `Cancelled` (partial cancellation)
- Status transitions are validated — no skipping states.

**Overall order completion:**
- After any `SupplierShipment` status update, check if all `SupplierShipment` records
  for the parent `Order` have reached a terminal state (`Delivered` or `Cancelled`).
- If all are terminal, update the nopCommerce `Order.OrderStatus` to `Complete`
  via `IOrderProcessingService.CheckOrderStatusAsync`.

**Partial cancellation:**
- If a supplier sets their shipment to `Cancelled`, trigger a partial refund workflow:
  - Calculate the refund amount for that supplier's items (sum of `OrderItem.PriceInclTax`
    for items in that shipment).
  - Create a note on the `Order` via `IOrderService.InsertOrderNoteAsync` recording
    the partial cancellation.
  - Notify the buyer by email using a custom message template
    `Marketplace.PartialShipmentCancelled`.
  - The actual refund processing is handled by Plugin 4 (FinanceAndReconciliation).
    Raise a custom domain event `SupplierShipmentCancelledEvent` that Plugin 4 consumes.

---

### F-5.3 — Delivery Method Configuration Per Supplier (Admin)

**Admin UI:**
- On the supplier detail page (in Plugin 1's admin area), add a "Delivery Settings"
  tab that loads a partial view from this plugin.
- Alternatively, add a standalone admin page:
  `Admin/CheckoutAndPricing/DeliveryConfig/{vendorId}`.
- Admin configures per supplier:
  - `OffersOwnDelivery` (checkbox)
  - `OffersCollection` (checkbox)
  - `DeliveryFeeType` (dropdown: Fixed / Per Mile / Custom)
  - `FixedDeliveryFee` (shown when Fixed selected)
  - `PerMileRate` (shown when Per Mile selected)
  - `CustomFeeRule` (text/JSON when Custom selected)
  - `StandardDeliveryRange` (text, e.g. "3–5 days")
  - `CollectionAddress` (text, shown when collection is enabled)

**Delivery fee calculation service:**
- Expose `ISupplierDeliveryService` with:
  ```csharp
  Task<SupplierDeliveryConfig> GetDeliveryConfigAsync(int vendorId);
  Task<decimal> CalculateSupplierDeliveryFeeAsync(int vendorId, decimal orderWeight, decimal distanceMiles);
  Task<IList<DeliveryOption>> GetAvailableDeliveryOptionsAsync(int vendorId, decimal orderWeight);
  ```
- `DeliveryOption` is a model (not an entity) with: `MethodId`, `Label`, `Fee`, `EstimatedRange`.

---

### F-5.4 — Checkout Delivery Option Selection Per Shipment

**Checkout — per-shipment delivery selection:**
- At the checkout shipping step, replace the single global shipping method selection
  with a per-supplier-shipment section.
- For each `SupplierShipment` being created (grouped by vendor), show available
  delivery options:
  1. **Supplier Delivery** — shown only if `SupplierDeliveryConfig.OffersOwnDelivery = true`.
     Fee from `ISupplierDeliveryService.CalculateSupplierDeliveryFeeAsync`.
  2. **Platform / Al-Basher Delivery** — always available. Fee from
     `SupplierManagementSettings.FlatDeliveryFee`.
  3. **Customer Collection** — shown only if `SupplierDeliveryConfig.OffersCollection = true`.
     Fee = 0. Shows collection address.
  4. **Uber Delivery** — deferred (F-4.3). Do not implement; reserve the enum value.
- Buyer must select one option per supplier shipment before proceeding.
- Selected delivery method per shipment is stored in session/checkout attributes
  until order placement, then persisted to `SupplierShipment.DeliveryMethodId` and
  `SupplierShipment.DeliveryFee`.

**Checkout — shipping cost summary:**
- Below the per-shipment delivery selection, show a summary table:
  - One row per supplier: Supplier Name | Delivery Method | Fee
  - Total shipping cost row
- This summary feeds into the order total calculation override.

**Implementation approach:**
- Override the checkout shipping step by registering a replacement for the shipping
  view component, or inject into the `checkout_shipping_method_top` widget zone.
- Store per-shipment delivery selections in `IGenericAttributeService` on the customer
  entity, keyed by `VendorId`, until order placement.

---

## Admin UI Pages Required

| Page | Route | Description |
|---|---|---|
| Delivery Config | `Admin/CheckoutAndPricing/DeliveryConfig/{vendorId}` | Per-supplier delivery settings |
| Sub-Order Monitor | `Admin/CheckoutAndPricing/SubOrders` | Admin view of all supplier shipments |

## Supplier Portal UI Pages Required

| Page | Route | Description |
|---|---|---|
| My Orders | `Vendor/MyOrders` | List supplier's shipments with status |
| Order Detail | `Vendor/OrderDetail/{supplierShipmentId}` | View items, update status |

---

## Service Overrides in NopStartup.cs

```csharp
// Infrastructure/NopStartup.cs
public void ConfigureServices(IServiceCollection services, IConfiguration configuration)
{
    // Override core order total calculation to inject marketplace fees
    services.AddScoped<IOrderTotalCalculationService, MarketplaceOrderTotalCalculationService>();

    // Register plugin services
    services.AddScoped<ISupplierDeliveryService, SupplierDeliveryService>();
    services.AddScoped<ISupplierShipmentService, SupplierShipmentService>();
}

public int Order => 3000;
```

---

## Event Consumers Required

```csharp
public class EventConsumer :
    IConsumer<OrderPlacedEvent>,                          // trigger shipment splitting
    IConsumer<EntityInsertedEvent<ShoppingCartItem>>,     // create CartItemSupplierMapping
    IConsumer<EntityDeletedEvent<ShoppingCartItem>>       // clean up CartItemSupplierMapping
```

---

## Custom Domain Events

Define these in the plugin's `Domain/` folder. Other plugins consume them.

```csharp
// Consumed by Plugin 4 (FinanceAndReconciliation)
public class SupplierShipmentCancelledEvent
{
    public SupplierShipment SupplierShipment { get; set; }
    public decimal RefundAmount { get; set; }
}

// Consumed by Plugin 4
public class SupplierShipmentDeliveredEvent
{
    public SupplierShipment SupplierShipment { get; set; }
}
```

Publish via `IEventPublisher.PublishAsync(event)`.

---

## Install / Uninstall Checklist

**InstallAsync:**
1. Create DB tables via `AutoReversingMigration` tagged `MigrationProcessType.Installation`:
   `CartItemSupplierMapping`, `OrderItemSupplierMapping`, `SupplierShipment`,
   `SupplierDeliveryConfig`.
2. Add message templates: `Marketplace.SupplierNewShipment`,
   `Marketplace.PartialShipmentCancelled`.
3. Add locale resources with prefix `Plugins.Misc.CheckoutAndPricing.`.
4. Call `await base.InstallAsync()` last.

**UninstallAsync:**
1. Delete locale resources.
2. Remove message templates.
3. Call `await base.UninstallAsync()` last.
4. Do NOT drop DB tables on uninstall.

---

## Folder Structure

```
Nop.Plugin.Misc.CheckoutAndPricing/
├── plugin.json
├── Nop.Plugin.Misc.CheckoutAndPricing.csproj
├── CheckoutAndPricingPlugin.cs          ← BasePlugin + IMiscPlugin
├── CheckoutAndPricingDefaults.cs        ← static constants
├── Events.cs                            ← IConsumer<T> implementations
├── logo.png
├── Controllers/
│   ├── CheckoutAndPricingAdminController.cs
│   └── SupplierOrderController.cs       ← supplier portal order management
├── Components/
│   ├── SupplierComparisonViewComponent.cs   ← product detail supplier table
│   ├── CartSupplierGroupViewComponent.cs    ← cart grouped by supplier
│   └── PerShipmentDeliveryViewComponent.cs  ← checkout delivery selection
├── Domain/
│   ├── CartItemSupplierMapping.cs
│   ├── OrderItemSupplierMapping.cs
│   ├── SupplierShipment.cs
│   ├── SupplierDeliveryConfig.cs
│   ├── SubOrderStatus.cs                ← enum
│   ├── DeliveryMethodType.cs            ← enum
│   ├── DeliveryFeeType.cs               ← enum
│   ├── SupplierShipmentCancelledEvent.cs
│   └── SupplierShipmentDeliveredEvent.cs
├── Data/
│   ├── CartItemSupplierMappingBuilder.cs
│   ├── OrderItemSupplierMappingBuilder.cs
│   ├── SupplierShipmentBuilder.cs
│   └── SupplierDeliveryConfigBuilder.cs
├── Factories/
│   ├── ICheckoutModelFactory.cs
│   └── CheckoutModelFactory.cs
├── Infrastructure/
│   ├── NopStartup.cs                    ← service overrides registered here
│   └── RouteProvider.cs
├── Migrations/
│   └── SchemaMigration.cs
├── Models/
│   ├── SupplierComparisonModel.cs
│   ├── CartSupplierGroupModel.cs
│   ├── PerShipmentDeliveryModel.cs
│   ├── DeliveryOption.cs                ← non-entity model
│   ├── SupplierOrderListModel.cs
│   └── SupplierOrderDetailModel.cs
├── Services/
│   ├── ISupplierDeliveryService.cs
│   ├── SupplierDeliveryService.cs
│   ├── ISupplierShipmentService.cs
│   ├── SupplierShipmentService.cs
│   └── MarketplaceOrderTotalCalculationService.cs  ← overrides core service
└── Views/
    ├── _ViewImports.cshtml
    ├── DeliveryConfig.cshtml
    ├── SubOrders.cshtml
    ├── MyOrders.cshtml
    └── OrderDetail.cshtml
```

---

## Key Constraints and Rules

- Follow all rules in `plugin-rule.md` without exception.
- `MarketplaceOrderTotalCalculationService` must call the original
  `OrderTotalCalculationService` methods via constructor injection (wrap, don't replace
  all logic) to preserve tax, discount, and gift card calculations.
- Never modify `ShoppingCartItem` or `OrderItem` core entities. Use the plugin's own
  mapping tables for supplier identity.
- The `CartItemSupplierMapping` record must be cleaned up when a `ShoppingCartItem`
  is deleted — handle via `IConsumer<EntityDeletedEvent<ShoppingCartItem>>`.
- Per-shipment delivery selections stored in `IGenericAttributeService` must be
  cleared after order placement to avoid stale data on the next checkout.
- All locale resource keys must use the prefix `Plugins.Misc.CheckoutAndPricing.`.
- `plugin.json` must declare `"DependsOnSystemNames": ["Marketplace.SupplierManagement"]`.
