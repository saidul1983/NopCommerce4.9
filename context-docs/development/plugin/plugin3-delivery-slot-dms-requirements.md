# Plugin 3 — Delivery Slot & DMS Requirements
**Plugin System Name:** `Marketplace.DeliverySlotAndDms`  
**Project Name:** `Nop.Plugin.Misc.DeliverySlotAndDms`  
**Output Folder:** `Misc.DeliverySlotAndDms`  
**Plugin Type:** `IMiscPlugin`  
**WBS Coverage:** F-5.5, F-8.1 (API side)

---

## Purpose

This plugin manages platform-managed (Al-Basher) delivery slot scheduling. It provides:
- Admin configuration of available delivery dates, time windows, and capacity limits.
- A calendar date/time picker at checkout when a buyer selects platform delivery.
- Slot booking with capacity enforcement to prevent overbooking.
- A REST API consumed by the DMS mobile app (F-8.1) for driver assignment and
  proof-of-delivery confirmation.

This plugin is self-contained. It does not override any core nopCommerce services.

---

## nopCommerce Foundation to Build On

- `IGenericAttributeService` — store the buyer's selected slot ID on the customer
  entity during checkout, cleared after order placement.
- `IOrderService` — read order and shipment data for the DMS API.
- `IShipmentService` — read shipment records for DMS driver assignment.
- `ICustomerService` — resolve customer details for DMS delivery address.
- `IWorkflowMessageService` — send slot booking confirmation email to buyer.
- `IScheduleTaskService` — register a cleanup task to release expired unconfirmed
  slot bookings.
- `ISettingService` — store admin slot configuration settings.
- `IRepository<T>` — all data access to plugin-owned tables.
- `IConsumer<OrderPlacedEvent>` — confirm slot booking when order is placed.

---

## Dependency on Plugin 2

This plugin depends on `Marketplace.CheckoutAndPricing` (Plugin 2) being installed,
because slot selection only applies when the buyer has chosen "Platform / Al-Basher
Delivery" for a shipment.

```json
"DependsOnSystemNames": ["Marketplace.SupplierManagement", "Marketplace.CheckoutAndPricing"]
```

At runtime, read `SupplierShipment.DeliveryMethodId` to determine if platform delivery
was selected. Resolve `ISupplierShipmentService` via DI — never reference Plugin 2's
project directly.

---

## New Database Tables Required

### `DeliverySlot` table
Admin-configured available delivery slots.

```csharp
public class DeliverySlot : BaseEntity
{
    public DateTime SlotDateUtc { get; set; }       // the calendar date
    public TimeSpan StartTime { get; set; }          // e.g. 09:00
    public TimeSpan EndTime { get; set; }            // e.g. 12:00
    public string Label { get; set; }               // e.g. "Morning (9am–12pm)"
    public int MaxCapacity { get; set; }            // max bookings allowed
    public int CurrentBookingCount { get; set; }    // incremented on booking
    public bool IsActive { get; set; }
    public DateTime CreatedOnUtc { get; set; }
}
```

### `SlotBooking` table
Records a buyer's slot reservation for a specific supplier shipment.

```csharp
public class SlotBooking : BaseEntity
{
    public int DeliverySlotId { get; set; }         // FK to DeliverySlot.Id
    public int OrderId { get; set; }                // FK to Order.Id (0 if pre-order pending)
    public int SupplierShipmentId { get; set; }     // FK to SupplierShipment.Id
    public int CustomerId { get; set; }             // FK to Customer.Id
    public int BookingStatusId { get; set; }        // enum: Pending=10, Confirmed=20, Released=30, Cancelled=40
    public string DeliveryAddress { get; set; }     // snapshot of delivery address at booking time
    public DateTime BookedOnUtc { get; set; }
    public DateTime? ConfirmedOnUtc { get; set; }
    public string DmsNotes { get; set; }            // notes added by driver via DMS app
    public string ProofOfDeliveryPhotoUrl { get; set; } // uploaded by driver
    public DateTime? DeliveredOnUtc { get; set; }
}
```

### `DmsApiToken` table
API tokens for DMS mobile app authentication.

```csharp
public class DmsApiToken : BaseEntity
{
    public string Token { get; set; }               // hashed token
    public string DriverName { get; set; }
    public string DriverEmail { get; set; }
    public bool IsActive { get; set; }
    public DateTime CreatedOnUtc { get; set; }
    public DateTime? ExpiresOnUtc { get; set; }
    public DateTime? LastUsedOnUtc { get; set; }
}
```

---

## Settings Class

```csharp
public class DeliverySlotSettings : ISettings
{
    // Maximum days ahead a buyer can book a slot from today
    // Requirement: approximately 2 months (~60 days)
    public int BookingWindowDays { get; set; }  // default: 60

    // How many minutes before a slot's start time to lock it (no new bookings)
    public int SlotLockoutMinutes { get; set; }  // default: 120

    // Whether to show slot selection at checkout (can be toggled off)
    public bool SlotSelectionEnabled { get; set; }  // default: true
}
```

---

## Feature Requirements

### F-5.5 — Platform Delivery Slot Scheduling

**Admin — slot configuration:**
- Admin page: "Delivery Slots" lists all `DeliverySlot` records with date, time window,
  capacity, current booking count, and active status.
- Admin can create slots individually (date picker + time window + capacity).
- Admin can bulk-create slots: select a date range, select days of week, select time
  windows, set capacity — generates all matching slots in one action.
- Admin can deactivate a slot (no new bookings allowed; existing bookings unaffected).
- Admin can view all `SlotBooking` records for a slot, including driver assignment
  status and proof of delivery.
- Admin can edit slot capacity (cannot reduce below current booking count).

**Checkout — slot selection:**
- Slot selection is shown only for supplier shipments where the buyer selected
  "Platform / Al-Basher Delivery" as the delivery method (from Plugin 2).
- If multiple supplier shipments use platform delivery, one slot selection covers
  all of them (single platform delivery run per order). If the client requires
  per-shipment slot selection, this must be confirmed before implementation.
- Inject a slot calendar widget into the checkout flow after the delivery method
  selection step, using a widget zone or a custom checkout step.
- The calendar shows available dates within the booking window
  (`DeliverySlotSettings.BookingWindowDays` from today).
- Dates with no active slots, or where all slots are at capacity, are shown as
  unavailable (greyed out).
- On date selection, show available time windows for that date as radio buttons.
- On time window selection, store the selected `DeliverySlotId` in
  `IGenericAttributeService` on the customer entity.
- Slot is not confirmed (capacity not decremented) until order is placed.

**Order placement — slot confirmation:**
- Listen to `OrderPlacedEvent` via `IConsumer<OrderPlacedEvent>`.
- On order placement:
  1. Read the selected `DeliverySlotId` from `IGenericAttributeService`.
  2. Check slot capacity: if `CurrentBookingCount >= MaxCapacity`, the slot is full.
     In this case, do not block the order — instead, flag the booking as `Pending`
     and notify admin. (Capacity check at checkout should prevent this, but guard
     against race conditions.)
  3. Create a `SlotBooking` record with status `Confirmed`.
  4. Increment `DeliverySlot.CurrentBookingCount`.
  5. Clear the selected slot from `IGenericAttributeService`.
  6. Send slot confirmation email to buyer via `IWorkflowMessageService` using
     message template `Marketplace.SlotBookingConfirmed`.

**Booking window enforcement:**
- Slots more than `BookingWindowDays` days from today must not be shown at checkout.
- Slots within `SlotLockoutMinutes` of their start time must not accept new bookings.

**Cleanup task:**
- Register a `IScheduleTask` (`SlotCleanupTask`) that runs daily.
- Releases `Pending` slot bookings older than 24 hours (order was never placed).
- Decrements `DeliverySlot.CurrentBookingCount` for released bookings.

---

### F-8.1 — DMS REST API

The DMS API is a set of JSON REST endpoints within this plugin, secured by token
authentication. The DMS mobile app (separate React Native / Flutter project) consumes
these endpoints.

**Authentication:**
- All DMS API endpoints require a bearer token in the `Authorization` header.
- Token is validated against the `DmsApiToken` table (hashed comparison).
- If token is invalid or expired, return HTTP 401.
- Admin can generate, view, and revoke DMS tokens from the admin UI.
- Implement token validation as a custom `IActionFilter` or middleware applied to
  the DMS API controller.

**DMS API Endpoints:**

```
GET  /api/dms/shipments/assigned
     Returns all SupplierShipment records where DeliveryMethodId = PlatformDelivery
     and SubOrderStatus = Processing or Shipped, with slot booking details.
     Response includes: shipmentId, orderId, slotDate, slotTime, deliveryAddress,
     items (product name + quantity), supplierName, supplierAddress (pickup).

GET  /api/dms/shipments/{supplierShipmentId}
     Returns full detail for one shipment including slot booking and order items.

POST /api/dms/shipments/{supplierShipmentId}/confirm-delivery
     Body: { "notes": "string", "photoBase64": "string" }
     Marks SlotBooking as delivered (sets ProofOfDeliveryPhotoUrl, DeliveredOnUtc).
     Updates SupplierShipment.SubOrderStatusId to Delivered via ISupplierShipmentService.
     Returns 200 OK or 400 with error message.

GET  /api/dms/slots/today
     Returns all DeliverySlot records for today with their bookings.
     Used by driver to see the day's schedule.

GET  /api/dms/slots/{date}
     Returns all DeliverySlot records for a specific date (format: yyyy-MM-dd).
```

**Photo upload:**
- `photoBase64` in the confirm-delivery endpoint is a base64-encoded image.
- Decode and save using `IPictureService` or write directly to a plugin-managed
  folder under `wwwroot/Plugins/Misc.DeliverySlotAndDms/uploads/`.
- Store the relative URL in `SlotBooking.ProofOfDeliveryPhotoUrl`.
- Limit file size to 5 MB. Validate that the base64 decodes to a valid image
  (JPEG or PNG) before saving.

**API response format:**
- All responses use a consistent envelope:
  ```json
  {
    "success": true,
    "data": { ... },
    "error": null
  }
  ```
- On error: `"success": false`, `"data": null`, `"error": "message"`.
- Use `System.Text.Json` for serialization — do not add Newtonsoft.Json as a
  dependency unless it is already in the solution.

---

## Admin UI Pages Required

| Page | Route | Description |
|---|---|---|
| Plugin Configuration | `Admin/DeliverySlotAndDms/Configure` | Booking window, lockout settings |
| Delivery Slots | `Admin/DeliverySlotAndDms/Slots` | List, create, bulk-create slots |
| Slot Detail | `Admin/DeliverySlotAndDms/SlotDetail/{id}` | View bookings for a slot |
| DMS Tokens | `Admin/DeliverySlotAndDms/Tokens` | Generate and revoke DMS API tokens |

---

## Event Consumers Required

```csharp
public class EventConsumer :
    IConsumer<OrderPlacedEvent>   // confirm slot booking on order placement
```

---

## Install / Uninstall Checklist

**InstallAsync:**
1. Create DB tables via `AutoReversingMigration` tagged `MigrationProcessType.Installation`:
   `DeliverySlot`, `SlotBooking`, `DmsApiToken`.
2. Save default `DeliverySlotSettings`: `BookingWindowDays = 60`,
   `SlotLockoutMinutes = 120`, `SlotSelectionEnabled = true`.
3. Register schedule task `SlotCleanupTask` (runs every 24 hours).
4. Add message template `Marketplace.SlotBookingConfirmed`.
5. Add locale resources with prefix `Plugins.Misc.DeliverySlotAndDms.`.
6. Call `await base.InstallAsync()` last.

**UninstallAsync:**
1. Delete `DeliverySlotSettings`.
2. Remove schedule task `SlotCleanupTask`.
3. Remove message template.
4. Delete locale resources.
5. Call `await base.UninstallAsync()` last.
6. Do NOT drop DB tables on uninstall.

---

## Folder Structure

```
Nop.Plugin.Misc.DeliverySlotAndDms/
├── plugin.json
├── Nop.Plugin.Misc.DeliverySlotAndDms.csproj
├── DeliverySlotAndDmsPlugin.cs          ← BasePlugin + IMiscPlugin
├── DeliverySlotSettings.cs              ← ISettings
├── DeliverySlotAndDmsDefaults.cs        ← static constants
├── Events.cs                            ← IConsumer<OrderPlacedEvent>
├── logo.png
├── Controllers/
│   ├── DeliverySlotAdminController.cs   ← admin slot management
│   └── DmsApiController.cs             ← REST API for mobile app (no [AuthorizeAdmin])
├── Components/
│   └── SlotCalendarViewComponent.cs    ← checkout calendar widget
├── Domain/
│   ├── DeliverySlot.cs
│   ├── SlotBooking.cs
│   ├── DmsApiToken.cs
│   └── BookingStatus.cs                ← enum
├── Data/
│   ├── DeliverySlotBuilder.cs
│   ├── SlotBookingBuilder.cs
│   └── DmsApiTokenBuilder.cs
├── Filters/
│   └── DmsTokenAuthFilter.cs           ← IActionFilter for DMS API auth
├── Infrastructure/
│   ├── NopStartup.cs
│   └── RouteProvider.cs                ← registers both admin and /api/dms/* routes
├── Migrations/
│   └── SchemaMigration.cs
├── Models/
│   ├── ConfigurationModel.cs
│   ├── DeliverySlotListModel.cs
│   ├── DeliverySlotModel.cs
│   ├── BulkSlotCreateModel.cs
│   ├── SlotBookingModel.cs
│   ├── DmsTokenModel.cs
│   └── Api/
│       ├── DmsShipmentResponse.cs
│       ├── DmsSlotResponse.cs
│       └── ConfirmDeliveryRequest.cs
├── Services/
│   ├── IDeliverySlotService.cs
│   ├── DeliverySlotService.cs
│   ├── IDmsTokenService.cs
│   ├── DmsTokenService.cs
│   └── SlotCleanupTask.cs              ← IScheduleTask
└── Views/
    ├── _ViewImports.cshtml
    ├── Configure.cshtml
    ├── Slots.cshtml
    ├── SlotDetail.cshtml
    └── Tokens.cshtml
```

---

## Key Constraints and Rules

- Follow all rules in `plugin-rule.md` without exception.
- The `DmsApiController` must NOT have `[AuthorizeAdmin]` or `[Area(AreaNames.ADMIN)]`
  attributes. It is a public API secured by the custom `DmsTokenAuthFilter`.
- Route pattern for DMS API: `"api/dms/{action}"` — registered in `RouteProvider.cs`
  using `IRouteProvider` directly (not `BaseRouteProvider`).
- `DeliverySlot.CurrentBookingCount` updates must be atomic. Use a database-level
  increment or a pessimistic lock pattern to prevent race conditions when two buyers
  book the same slot simultaneously. Consider using `IRepository<T>` with a
  `UPDATE ... WHERE CurrentBookingCount < MaxCapacity` approach via `INopDataProvider`
  for the atomic increment.
- Photo uploads must be validated for file type and size before saving. Never trust
  the base64 content without validation.
- `DmsApiToken` stores a hashed token — never store the raw token value. Use
  `HashHelper` from `Nop.Core` for hashing.
- All locale resource keys must use the prefix `Plugins.Misc.DeliverySlotAndDms.`.
- `plugin.json` must declare:
  `"DependsOnSystemNames": ["Marketplace.SupplierManagement", "Marketplace.CheckoutAndPricing"]`.
