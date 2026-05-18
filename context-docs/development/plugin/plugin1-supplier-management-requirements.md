# Plugin 1 — Supplier Management Requirements
**Plugin System Name:** `Marketplace.SupplierManagement`  
**Project Name:** `Nop.Plugin.Misc.SupplierManagement`  
**Output Folder:** `Misc.SupplierManagement`  
**Plugin Type:** `IMiscPlugin`  
**WBS Coverage:** F-2.1, F-2.2, F-2.3, F-3.1

---

## Overview

### What This Plugin Is

Plugin 1 is the **foundation of the entire Al Basher marketplace**. Every other plugin
depends on the data model and services introduced here. It answers one central question:
*who are the suppliers, what products do they sell, and at what price?*

Without this plugin installed, none of the other marketplace plugins can function.

---

### The Core Problem It Solves

Standard nopCommerce lets vendors create their own products freely, which leads to
duplicate listings, inconsistent naming, and no central control over the catalog.
Al Basher is a B2B marketplace where the platform operator (admin) owns the product
definitions and suppliers simply offer their own price and availability against those
definitions.

This plugin introduces that separation:

```
Admin creates Product  →  Supplier maps to Product with their own price/stock
                       →  Buyer sees all suppliers for that product and picks one
```

---

### How It Works — End to End

**Step 1 — Supplier registers**
A business applies via the standard nopCommerce vendor apply form. The plugin
intercepts this and creates a `SupplierProfile` record with status `Pending`.
The supplier cannot access the portal yet.

**Step 2 — Admin reviews and approves**
Admin sees the supplier in the Supplier Pipeline dashboard (All / Pending / Approved /
Suspended / Rejected tabs). Admin reviews the business registration number, trade
license, and contact details, then clicks Approve. The supplier receives an email and
gains portal access.

**Step 3 — Supplier sets up their profile**
After approval, the supplier logs into their portal and completes their profile:
- Contact person and phone
- **Collection location** — the physical address where buyers can collect orders
  (separate from the registered business address). Includes opening hours and
  collection instructions. This is required before the supplier can enable
  "collection" on any product listing.

**Step 4 — Supplier maps products**
The supplier browses the admin-owned master catalog and maps themselves to products
they can supply. For each product they set:
- Their own **base price** (the platform adds markup on top — supplier never sees
  the final buyer price)
- B2B quantities: minimum and maximum order quantity per transaction
- Delivery and collection availability
- Stock status and indicative quantity
- Their own SKU and lead time

They can do this one by one or via **Excel bulk upload** (up to 500 rows, 5 MB).

**Step 5 — Price engine runs**
When a buyer views a product, the platform calculates the display price:
`display price = supplier base price × (1 + markup%)`. The markup is either the
platform default (3–3.5%) or a per-supplier override set by admin. A service fee
(2.2–2.5%) is added at checkout on the full order total. This calculation is exposed
via `IMarketplacePriceService` and consumed by Plugin 2 at checkout.

**Step 6 — Supplier monitors via dashboard**
The supplier portal dashboard shows live summary cards: active listings, out-of-stock
items, pending orders (from Plugin 2), revenue this month (from Plugin 4), and pending
returns. Each card is independent — if Plugin 2 or Plugin 4 is not installed, those
cards degrade gracefully.

**Step 7 — Returns visibility**
When a buyer submits a return request (handled by nopCommerce core), the supplier
sees it in their "Returns & Replacements" portal page. They can see the reason,
requested action (refund or replacement), and status. They cannot approve or reject —
that is admin-only via Plugin 4. If the action is "replacement", a flag prompts them
to set aside stock.

---

### What Is Deliberately Out of Scope for This Plugin

| Concern | Handled By |
|---|---|
| Buyer submitting a return request | nopCommerce core (`ReturnRequestController`) |
| Refund approval and financial recovery | Plugin 4 (FinanceAndReconciliation) |
| Checkout price breakdown and cart | Plugin 2 (CheckoutAndPricing) |
| Shipment splitting and order status | Plugin 2 (CheckoutAndPricing) |
| Delivery slot scheduling | Plugin 3 (DeliverySlotAndDms) |
| Recurring orders and saved lists | Plugin 5 (CustomerAndRecurring) |

---

### Key Design Decisions

- **Supplier = nopCommerce Vendor.** The existing `Vendor` entity is extended via
  `SupplierProfile` — no parallel supplier table. This keeps nopCommerce's built-in
  vendor-product assignment working as the base.
- **Master catalog products have `VendorId = 0`.** Admin-owned products are
  distinguished from supplier products by this convention. Suppliers cannot create
  products with `VendorId = 0`.
- **`IMarketplacePriceService` is this plugin's public API** for other plugins.
  Plugin 2 calls it for every price calculation. It is registered as `AddScoped`
  in this plugin's `NopStartup.cs`.
- **Collection address is a separate `Address` record**, not a text field. It uses
  nopCommerce's existing `Address` entity and `IAddressService`, stored as
  `SupplierProfile.CollectionAddressId`.
- **Plugin 1 installs independently.** Dashboard cards that depend on Plugin 2 or
  Plugin 4 use nullable DI resolution (`IServiceProvider.GetService<T>()`) so this
  plugin works even when the others are not yet installed.

---

### What Is Still Missing / Not Planned

The following items are **not covered** by this plugin and have no current plan:

- **Supplier performance metrics** — no rating, fulfilment rate, or on-time delivery
  score tracking. Would require a separate reporting module.
- **Supplier document management** — trade license and business registration are
  stored as text fields only. No file upload for supporting documents during
  onboarding.
- **Supplier bank/payout details** — bank account information for payouts is not
  stored here. Plugin 4 tracks what is owed but does not store payment details.
  This would need a secure vault or integration with a payout provider.
- **Multi-location suppliers** — a supplier can have one collection address. Multiple
  warehouse or collection locations per supplier are not supported.
- **Supplier-to-supplier messaging** — no internal messaging between suppliers or
  between supplier and admin beyond email notifications.
- **Product image upload by supplier** — suppliers map to master catalog products
  and cannot upload their own product images. Images are admin-managed on the master
  product.
- **Automated approval rules** — approval is always manual by admin. No auto-approval
  based on document verification or criteria.

---

## Purpose

This plugin is the core domain foundation of the Al Basher marketplace. It introduces
the supplier onboarding lifecycle, the admin-owned master product catalog, the
supplier-to-product mapping layer, and the platform markup/fee engine. All other
marketplace plugins depend on the data model introduced here.

---

## What nopCommerce Already Provides — Do NOT Re-implement

These features exist in nopCommerce 4.90 out of the box. Build on them, do not replace them.

| Feature | nopCommerce Location | What It Does |
|---|---|---|
| Vendor apply form | `VendorController.ApplyVendor` | Buyer registers as vendor — form, picture upload, vendor attributes |
| Vendor info edit | `VendorController.Info` | Vendor edits own name, email, description, picture |
| Vendor notes | `VendorNote` entity + `IVendorService` | Admin adds internal notes to a vendor record |
| Vendor address | `Vendor.AddressId` FK to `Address` table | Vendor's registered address already stored |
| Return request (buyer) | `ReturnRequestController.ReturnRequest` | Buyer submits return per order item, selects reason + action, uploads file |
| Return request status machine | `ReturnRequestStatus` enum | Pending → Received → ReturnAuthorized → ItemsRepaired → ItemsRefunded → RequestRejected → Cancelled |
| Return reasons & actions | `ReturnRequestReason`, `ReturnRequestAction` | Admin-configurable reason and action lists |
| Return request notifications | `IWorkflowMessageService` | Notifies store owner and customer on submission |
| Vendor attributes | `VendorAttribute`, `VendorAttributeValue` | Custom fields on vendor apply form |

**Key implication for refund/replace:** nopCommerce's `ReturnRequest` is per `OrderItem`
and already handles the buyer-facing submission flow. Plugin 1 does NOT re-implement
this. Instead, Plugin 1 adds supplier-side visibility of return requests in the
supplier portal (read-only view of returns affecting their items). The actual
refund approval and financial recovery logic lives in Plugin 4.

---

## nopCommerce Foundation to Build On

- `Vendor` entity (`Nop.Core.Domain.Vendors.Vendor`) is the base for suppliers.
  Extend it — do not create a parallel supplier table.
- `VendorSettings` controls whether customers can apply to become vendors.
  Set `AllowCustomersToApplyForVendorAccount = true` as a default during install.
- `Product` entity (`Nop.Core.Domain.Catalog.Product`) is the base for master catalog
  products. Admin creates products normally; this plugin adds the mapping layer on top.
- `IVendorService` provides `GetVendorByIdAsync`, `UpdateVendorAsync`, `InsertVendorAsync`.
- `IProductService` provides `GetProductByIdAsync`, `SearchProductsAsync`.
- `ICustomerService` provides customer lookup for linking a customer account to a vendor.
- `ISettingService` for storing platform-wide fee settings.
- `ILocalizationService` for locale resource management.
- `IRepository<T>` for all data access to plugin-owned tables.

---

## New Database Tables Required

### `SupplierProfile` table
Extends vendor with marketplace-specific onboarding fields. Note: `Vendor.AddressId`
already stores the registered business address via nopCommerce's `Address` entity —
do not duplicate it here. `CollectionAddressId` is a separate FK for the physical
location where buyers collect orders, which may differ from the registered address.

```csharp
public class SupplierProfile : BaseEntity
{
    public int VendorId { get; set; }               // FK to Vendor.Id
    public int OnboardingStatusId { get; set; }     // enum: Pending=10, Approved=20, Suspended=30, Rejected=40
    public string BusinessRegistrationNumber { get; set; }
    public string ContactPersonName { get; set; }
    public string ContactPhone { get; set; }
    public string TradeLicenseNumber { get; set; }
    // Collection location — where buyers physically collect orders from this supplier.
    // Stored as a separate Address record (FK to nopCommerce Address table).
    // May differ from Vendor.AddressId (registered business address).
    public int? CollectionAddressId { get; set; }
    public string CollectionInstructions { get; set; } // e.g. "Use rear entrance, ask for John"
    public string CollectionOpeningHours { get; set; } // e.g. "Mon–Fri 8am–5pm"
    public string AdminNotes { get; set; }
    public DateTime SubmittedOnUtc { get; set; }
    public DateTime? ReviewedOnUtc { get; set; }
    public int? ReviewedByCustomerId { get; set; }
    public bool PortalAccessEnabled { get; set; }
}
```

### `SupplierFeeConfig` table
Per-supplier markup and fee overrides.

```csharp
public class SupplierFeeConfig : BaseEntity
{
    public int VendorId { get; set; }                      // FK to Vendor.Id
    public decimal? MarkupPercentageOverride { get; set; } // null = use platform default
    public DateTime CreatedOnUtc { get; set; }
    public DateTime UpdatedOnUtc { get; set; }
}
```

### `MasterProductMapping` table
Links a supplier (vendor) to a master catalog product with their own pricing and availability.

```csharp
public class MasterProductMapping : BaseEntity
{
    public int VendorId { get; set; }              // FK to Vendor.Id
    public int ProductId { get; set; }             // FK to Product.Id (master catalog product)
    public decimal BasePrice { get; set; }         // supplier's own base price (before markup)
    public decimal? MinOrderQuantity { get; set; } // B2B: minimum order quantity (null = no minimum)
    public decimal? MaxOrderQuantity { get; set; } // B2B: maximum order quantity per transaction
    public bool IsAvailableForDelivery { get; set; }
    public bool IsAvailableForCollection { get; set; }
    public bool IsInStock { get; set; }
    public int StockQuantity { get; set; }         // indicative stock level (supplier-managed)
    public bool Published { get; set; }
    public string SupplierSku { get; set; }        // supplier's own SKU/reference code
    public string LeadTimeDays { get; set; }       // e.g. "2–3 days" — shown to buyer
    public DateTime CreatedOnUtc { get; set; }
    public DateTime UpdatedOnUtc { get; set; }
}
```

---

## Settings Class

```csharp
// SupplierManagementSettings.cs — implements ISettings, auto-registered per store
public class SupplierManagementSettings : ISettings
{
    // Platform-wide default markup applied to all supplier base prices
    // Typical range: 3.0 – 3.5 %
    public decimal DefaultMarkupPercentage { get; set; }

    // Platform service fee applied on the full order total at checkout
    // Typical range: 2.2 – 2.5 %
    public decimal ServiceFeePercentage { get; set; }

    // Flat delivery fee added per order at checkout (platform-managed delivery)
    public decimal FlatDeliveryFee { get; set; }

    // Threshold % below which a price reduction triggers a confirmation warning
    // to the supplier before saving. Default: 50 (i.e. warn if new price is
    // more than 50% lower than current price).
    public decimal PriceChangeWarningThresholdPercent { get; set; }
}
```

---

## Feature Requirements

### F-2.1 — Supplier Onboarding & Approval

**Onboarding status machine:**
- States: `Pending` → `Approved` or `Rejected`. `Approved` → `Suspended` → `Approved`.
- A supplier registers via the standard nopCommerce vendor apply form
  (`/vendor/applyvendor`). On submission, a `SupplierProfile` record is created with
  status `Pending` and `PortalAccessEnabled = false`.
- Admin sees a dedicated **Supplier Pipeline** page in the admin area listing all
  suppliers grouped by status with filter tabs (All / Pending / Approved / Suspended /
  Rejected).
- Admin actions per supplier: Approve, Reject, Suspend, Edit Profile, View Orders.
- On **Approve**: set `SupplierProfile.OnboardingStatusId = Approved`,
  `PortalAccessEnabled = true`, activate the linked `Vendor` record (`Active = true`),
  send approval email to supplier using nopCommerce message template system
  (`IWorkflowMessageService`).
- On **Reject**: set status to `Rejected`, `PortalAccessEnabled = false`,
  `Vendor.Active = false`, send rejection email with admin notes as reason.
- On **Suspend**: set status to `Suspended`, `PortalAccessEnabled = false`,
  `Vendor.Active = false`. Supplier portal becomes inaccessible.
- Admin can edit any supplier's profile fields (contact info, business details, admin notes).
- Admin can re-approve a suspended supplier.

**Portal access gating:**
- Supplier portal pages must check `SupplierProfile.PortalAccessEnabled` before rendering.
  If `false`, redirect to an "account pending approval" page.
- Use a custom `IActionFilter` or base controller check — do not rely solely on the
  `Vendor` role check that nopCommerce provides.

**Collection location management:**
- During onboarding (or after approval), supplier sets their collection location via
  the supplier portal "My Profile" page.
- Collection location is stored as a separate nopCommerce `Address` record, with its
  `Id` saved in `SupplierProfile.CollectionAddressId`.
- Use `IAddressService` to insert/update the `Address` record.
- Fields: full address (street, city, postcode, country), `CollectionInstructions`,
  `CollectionOpeningHours`.
- This address is shown to buyers at checkout when they select "Customer Collection"
  as the delivery method for this supplier's shipment (consumed by Plugin 2).
- Admin can also set/override the collection address from the supplier detail page.

**Email notifications:**
- Use `IWorkflowMessageService` and nopCommerce message templates.
- Add two message templates during `InstallAsync`:
  - `Marketplace.SupplierApproved` — sent to supplier on approval
  - `Marketplace.SupplierRejected` — sent to supplier on rejection with reason
  - `Marketplace.SupplierSuspended` — sent to supplier on suspension with reason

---

### F-2.1a — Supplier Dashboard Summary (Supplier Portal)

This is a new section not in the original WBS but required for a functional supplier
portal. The supplier dashboard is the landing page after login.

**Dashboard summary cards (read-only, supplier sees only their own data):**
- **Active Listings** — count of `MasterProductMapping` records where `Published = true`.
- **Out of Stock** — count of mappings where `IsInStock = false`.
- **Pending Orders** — count of `SupplierShipment` records (Plugin 2) where
  `SubOrderStatus = Processing`. Resolved via `ISupplierShipmentService` (Plugin 2,
  registered in DI).
- **Orders This Month** — count of `SupplierShipment` records created in the current
  calendar month.
- **Revenue This Month** — sum of `SupplierLedgerEntry.Amount` where `EntryType =
  OrderRevenue` for the current month (Plugin 4, resolved via `ILedgerService`).
  If Plugin 4 is not installed, show "N/A" gracefully — do not throw.
- **Pending Returns** — count of `ShipmentReturnRequest` records (Plugin 4) where
  `ReturnStatus = Pending` and `VendorId` matches the current supplier. Resolved via
  `IReturnRequestService` (Plugin 4). If Plugin 4 is not installed, hide this card.

**Implementation:**
- Dashboard is the default landing page at `Vendor/Dashboard`.
- Implement as a view with multiple `ViewComponent` calls so each card loads
  independently and a failure in one does not break the others.
- All data is read-only. No actions on the dashboard — actions are on their
  respective pages.
- Guard every cross-plugin service call with a null check after DI resolution.
  Use `IServiceProvider.GetService<T>()` (returns null if not registered) rather
  than constructor injection for Plugin 2 and Plugin 4 services, so Plugin 1 can
  function even if those plugins are not yet installed.

---

### F-2.2 — Centralised Product Catalog

**Admin-owned master catalog:**
- Products in the master catalog are created by admin only. Suppliers cannot create
  new products — they can only map themselves to existing master catalog products.
- Enforce this by overriding the vendor portal product creation flow. In the supplier
  portal, replace the "Add new product" button with "Map to existing product".
- Master catalog products are standard nopCommerce `Product` entities with
  `VendorId = 0` (platform-owned). This distinguishes them from supplier-specific
  product copies.
- Admin manages master catalog products through the standard nopCommerce admin product
  pages. No new admin UI is needed for product creation itself.
- Admin can mark a product as "active in master catalog" using a custom flag stored
  in a plugin setting or product generic attribute if a full new table is not warranted.

**Preventing duplicate products:**
- In the supplier portal, the product search/mapping UI searches only master catalog
  products (`VendorId = 0`, `Published = true`).
- Suppliers cannot publish a product directly — only their `MasterProductMapping`
  record controls their listing visibility.

---

### F-2.3 — Markup Fee & Price Calculation

**Platform-wide settings (admin):**
- Admin configures `DefaultMarkupPercentage`, `ServiceFeePercentage`, and
  `FlatDeliveryFee` on the plugin configuration page.
- These are stored in `SupplierManagementSettings` via `ISettingService`.

**Per-supplier markup override (admin):**
- On the supplier detail page, admin can set a `MarkupPercentageOverride` in
  `SupplierFeeConfig`. If set, this overrides the platform default for that supplier.
- If `MarkupPercentageOverride` is null, the platform default applies.

**Price calculation service:**
- Expose a service interface `IMarketplacePriceService` with:
  ```csharp
  Task<decimal> GetEffectiveMarkupPercentageAsync(int vendorId);
  Task<decimal> CalculateDisplayPriceAsync(int vendorId, decimal basePrice);
  Task<decimal> CalculateServiceFeeAsync(decimal orderTotal);
  Task<decimal> GetFlatDeliveryFeeAsync();
  ```
- `CalculateDisplayPriceAsync` = `basePrice * (1 + markupPercentage / 100)`.
- This service is consumed by Plugin 2 (CheckoutAndPricing) for cart and checkout
  calculations. Register it as `AddScoped` in `NopStartup.cs`.

---

### F-3.1 — Product Mapping & Upload

**Supplier portal — map to product:**
- Supplier portal page: "My Products" lists all `MasterProductMapping` records for
  the current supplier.
- "Add Mapping" page: supplier searches the master catalog by name/category, selects
  a product, and sets:
  - `BasePrice` (their own price — required, must be > 0)
  - `SupplierSku` (optional — their internal reference code)
  - `MinOrderQuantity` (optional — B2B minimum, e.g. 10 units)
  - `MaxOrderQuantity` (optional — B2B maximum per transaction)
  - `LeadTimeDays` (optional — e.g. "2–3 days")
  - `IsAvailableForDelivery` (toggle)
  - `IsAvailableForCollection` (toggle — only shown if `SupplierProfile.CollectionAddressId` is set)
  - `IsInStock` (toggle)
  - `StockQuantity` (numeric — indicative, not enforced by platform)
  - `Published` (show/hide their listing)
- A supplier can only have one active mapping per master product. Enforce uniqueness
  on `(VendorId, ProductId)` in the database.
- Supplier can edit or delete their own mappings.

**Validation rules for product mapping:**
- `BasePrice` must be greater than 0. Error: "Base price must be greater than zero."
- `BasePrice` must be a valid decimal with at most 2 decimal places.
- `MinOrderQuantity`, if set, must be a positive integer.
- `MaxOrderQuantity`, if set, must be >= `MinOrderQuantity`. Error: "Maximum order
  quantity cannot be less than minimum order quantity."
- `StockQuantity` must be >= 0.
- `IsAvailableForCollection` can only be set to `true` if
  `SupplierProfile.CollectionAddressId` is not null. If the supplier has no collection
  address set, show a warning: "Please set your collection address in My Profile before
  enabling collection for this product."
- A supplier cannot set `Published = true` if `IsInStock = false`. Warn: "You cannot
  publish a listing that is out of stock. Set stock status to In Stock first."
  (This is a soft warning — allow override if supplier explicitly confirms.)

**Validation rules for supplier profile:**
- `ContactPhone` must match a valid phone format (allow international format).
- `BusinessRegistrationNumber` is required before the supplier can be approved.
- `CollectionOpeningHours` max length: 200 characters.
- `CollectionInstructions` max length: 500 characters.
- `BasePrice` changes on existing mappings: if the new price is more than 50% lower
  than the current price, show a confirmation warning to the supplier before saving.
  This prevents accidental price errors. (Threshold is configurable in
  `SupplierManagementSettings`.)

**Excel bulk upload:**
- Supplier portal page: "Bulk Upload" with a downloadable Excel template.
- Template columns: `ProductName`, `SupplierSku`, `BasePrice`, `MinOrderQuantity`,
  `MaxOrderQuantity`, `LeadTimeDays`, `IsAvailableForDelivery` (Y/N),
  `IsAvailableForCollection` (Y/N), `IsInStock` (Y/N), `StockQuantity`, `Published` (Y/N).
- On upload, match `ProductName` to master catalog products (case-insensitive exact
  match). If no match found, log the row as an error and continue processing others.
- Apply the same validation rules as the single-mapping form to each row.
- If a mapping already exists for a product, update it (upsert behaviour).
- Use `IExportManager` / `IImportManager` patterns from `Nop.Services.ExportImport`
  as reference for the Excel processing approach. Use `ClosedXML` or `EPPlus` — check
  which NuGet package is already referenced in the solution before adding a new one.
- Return a summary report: rows processed, rows succeeded, rows failed with reasons.
- Maximum file size: 5 MB. Maximum rows per upload: 500.

---

### F-3.1b — Supplier-side Return & Replace Visibility

nopCommerce's `ReturnRequest` system handles the buyer submission and admin review
flow. Plugin 4 handles the financial recovery. This plugin adds supplier-side
visibility only — suppliers need to know about returns affecting their items so they
can prepare for replacements or stock adjustments.

**Supplier portal — Returns page:**
- Page: "Returns & Replacements" lists all `ReturnRequest` records (nopCommerce core
  entity) where the `OrderItem.ProductId` maps to one of the supplier's
  `MasterProductMapping.ProductId` records.
- Query via `IReturnRequestService.GetReturnRequestsAsync` filtered by the supplier's
  product IDs.
- Display columns: Return #, Order #, Product, Quantity, Reason, Requested Action,
  Status, Submitted Date.
- Supplier can see the status but **cannot approve or reject** — that is admin-only
  (Plugin 4).
- If the requested action is "Replacement" (`ReturnRequest.RequestedAction` contains
  "replacement"), show a "Prepare Replacement" flag so the supplier knows to set
  aside stock.
- Supplier can add a note to a return request via `VendorNote` on their own vendor
  record — this is a workaround since `ReturnRequest.StaffNotes` is admin-only.
  Alternatively, use `IGenericAttributeService` to store a supplier note keyed by
  `ReturnRequest.Id`.

**No new DB table needed** — this feature reads from nopCommerce's existing
`ReturnRequest` table and the plugin's own `MasterProductMapping` table.

---

## Admin UI Pages Required

| Page | Route | Description |
|---|---|---|
| Plugin Configuration | `Admin/SupplierManagement/Configure` | Platform fee settings, price change warning threshold |
| Supplier Pipeline | `Admin/SupplierManagement/Suppliers` | List all suppliers with status filter tabs |
| Supplier Detail | `Admin/SupplierManagement/SupplierDetail/{id}` | View/edit profile, approve/reject/suspend, set collection address |
| Supplier Fee Override | `Admin/SupplierManagement/FeeConfig/{vendorId}` | Set per-supplier markup override |
| Master Catalog | `Admin/SupplierManagement/MasterCatalog` | View all master catalog products and their supplier mapping counts |

---

## Supplier Portal UI Pages Required

| Page | Route | Description |
|---|---|---|
| Dashboard | `Vendor/Dashboard` | Summary cards: listings, orders, revenue, returns |
| My Profile | `Vendor/MyProfile` | Edit contact info, collection address, opening hours |
| My Products | `Vendor/MyProducts` | List supplier's product mappings with status indicators |
| Add Mapping | `Vendor/AddProductMapping` | Search master catalog and create mapping |
| Edit Mapping | `Vendor/EditProductMapping/{id}` | Edit base price, B2B quantities, availability, stock |
| Bulk Upload | `Vendor/BulkUpload` | Excel upload for product mappings |
| Returns | `Vendor/Returns` | Read-only view of return requests affecting supplier's products |
| Account Status | `Vendor/AccountStatus` | Shown when portal access is disabled (pending/suspended) |

---

## Event Consumers Required

```csharp
public class EventConsumer :
    IConsumer<EntityInsertedEvent<Vendor>>,   // create SupplierProfile on vendor creation
    IConsumer<EntityUpdatedEvent<Vendor>>     // sync Vendor.Active with PortalAccessEnabled
```

---

## Install / Uninstall Checklist

**InstallAsync:**
1. Create DB tables: `SupplierProfile`, `SupplierFeeConfig`, `MasterProductMapping`
   via `AutoReversingMigration` tagged `MigrationProcessType.Installation`.
2. Save default `SupplierManagementSettings`:
   `DefaultMarkupPercentage = 3.0m`, `ServiceFeePercentage = 2.2m`,
   `FlatDeliveryFee = 0m`, `PriceChangeWarningThresholdPercent = 50m`.
3. Add message templates: `Marketplace.SupplierApproved`, `Marketplace.SupplierRejected`,
   `Marketplace.SupplierSuspended`.
4. Add all locale resources with prefix `Plugins.Misc.SupplierManagement.`.
5. Call `await base.InstallAsync()` last.

**UninstallAsync:**
1. Delete `SupplierManagementSettings`.
2. Delete locale resources with prefix `Plugins.Misc.SupplierManagement.`.
3. Remove message templates.
4. Call `await base.UninstallAsync()` last.
5. Do NOT drop DB tables on uninstall — data must be preserved for reinstall.

---

## Folder Structure

```
Nop.Plugin.Misc.SupplierManagement/
├── plugin.json
├── Nop.Plugin.Misc.SupplierManagement.csproj
├── SupplierManagementPlugin.cs          ← BasePlugin + IMiscPlugin
├── SupplierManagementSettings.cs        ← ISettings
├── SupplierManagementDefaults.cs        ← static constants
├── Events.cs                            ← IConsumer<T> implementations
├── logo.png
├── Controllers/
│   ├── SupplierManagementAdminController.cs   ← admin pages
│   └── SupplierPortalController.cs            ← supplier portal pages
├── Components/
│   ├── DashboardActiveListingsViewComponent.cs
│   ├── DashboardPendingOrdersViewComponent.cs
│   ├── DashboardRevenueViewComponent.cs
│   └── DashboardPendingReturnsViewComponent.cs
├── Domain/
│   ├── SupplierProfile.cs
│   ├── SupplierFeeConfig.cs
│   ├── MasterProductMapping.cs
│   └── OnboardingStatus.cs              ← enum
├── Data/
│   ├── SupplierProfileBuilder.cs
│   ├── SupplierFeeConfigBuilder.cs
│   └── MasterProductMappingBuilder.cs
├── Factories/
│   ├── ISupplierModelFactory.cs
│   └── SupplierModelFactory.cs
├── Infrastructure/
│   ├── NopStartup.cs
│   └── RouteProvider.cs
├── Migrations/
│   └── SchemaMigration.cs               ← AutoReversingMigration, Installation
├── Models/
│   ├── SupplierListModel.cs
│   ├── SupplierDetailModel.cs
│   ├── ProductMappingModel.cs
│   ├── BulkUploadModel.cs
│   ├── BulkUploadResultModel.cs
│   ├── DashboardModel.cs
│   ├── SupplierReturnListModel.cs
│   └── ConfigurationModel.cs
├── Services/
│   ├── ISupplierService.cs
│   ├── SupplierService.cs
│   ├── IMarketplacePriceService.cs      ← consumed by Plugin 2
│   └── MarketplacePriceService.cs
├── Validators/
│   ├── ProductMappingValidator.cs       ← BaseNopValidator<ProductMappingModel>
│   └── SupplierProfileValidator.cs      ← BaseNopValidator<SupplierDetailModel>
└── Views/
    ├── _ViewImports.cshtml
    ├── Configure.cshtml
    ├── Suppliers.cshtml
    ├── SupplierDetail.cshtml
    ├── MasterCatalog.cshtml
    ├── Dashboard.cshtml
    ├── MyProfile.cshtml
    ├── MyProducts.cshtml
    ├── AddProductMapping.cshtml
    ├── EditProductMapping.cshtml
    ├── BulkUpload.cshtml
    ├── Returns.cshtml
    └── AccountStatus.cshtml
```

---

## Key Constraints and Rules

- Follow all rules in `plugin-rule.md` without exception.
- `IMarketplacePriceService` must be registered in this plugin's `NopStartup.cs` as
  `AddScoped` so Plugin 2 can resolve it via DI without a direct project reference.
- Never add a `<ProjectReference>` to Plugin 2, 3, 4, or 5. Communication is via
  shared core interfaces and events only.
- The `MasterProductMapping` uniqueness constraint `(VendorId, ProductId)` must be
  enforced at the database level in the entity builder, not only in application code.
- All admin controller actions must have `[AuthorizeAdmin]`, `[Area(AreaNames.ADMIN)]`,
  and appropriate `[CheckPermission]` attributes.
- Supplier portal controller actions must check `SupplierProfile.PortalAccessEnabled`
  before proceeding — redirect to `AccountStatus` page if false.
- Use `nameof()` for all column references in entity builders — never hardcode strings.
- All locale resource keys must use the prefix `Plugins.Misc.SupplierManagement.`.
- Validators must inherit `BaseNopValidator<T>` (not `AbstractValidator<T>`) and use
  `WithMessageAwait(localizationService.GetResourceAsync(...))` for all messages.
- The supplier dashboard uses `IServiceProvider.GetService<T>()` (nullable resolution)
  for Plugin 2 and Plugin 4 services — never constructor-inject cross-plugin services
  in this plugin. This ensures Plugin 1 installs and runs independently.
- `CollectionAddressId` in `SupplierProfile` references nopCommerce's `Address` table.
  Use `IAddressService` to create/update the address record. Never store address
  fields as raw strings in `SupplierProfile` — always use the `Address` entity.
- The `Returns` portal page is read-only. Suppliers cannot change `ReturnRequest`
  status — that is admin-only via Plugin 4. Enforce this with no POST actions on
  the Returns page.
- `PriceChangeWarningThresholdPercent` validation is a client-side + server-side
  soft warning, not a hard block. The supplier must explicitly confirm before the
  save proceeds if the threshold is exceeded.
