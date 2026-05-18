# Plugin 4 — Finance & Reconciliation Requirements
**Plugin System Name:** `Marketplace.FinanceAndReconciliation`  
**Project Name:** `Nop.Plugin.Misc.FinanceAndReconciliation`  
**Output Folder:** `Misc.FinanceAndReconciliation`  
**Plugin Type:** `IMiscPlugin`  
**WBS Coverage:** F-2.4, F-6.1, F-7.8

---

## Purpose

This plugin owns all financial tracking for the Al Basher marketplace. It maintains
a per-supplier financial ledger, generates reconciliation reports for admin, handles
the per-shipment refund/return workflow, tracks payout recovery when a supplier has
already been paid, and manages split payment allocation for recurring billing orders.

---

## nopCommerce Foundation to Build On

- `Order` (`Nop.Core.Domain.Orders.Order`) — source of truth for order totals.
- `OrderItem` (`Nop.Core.Domain.Orders.OrderItem`) — line-item prices.
- `ReturnRequest` (`Nop.Core.Domain.Orders.ReturnRequest`) — base for return requests.
  This plugin extends the concept to per-shipment returns.
- `IOrderService` — read order and order item data.
- `IReturnRequestService` — use as reference; this plugin creates its own per-shipment
  return request entity rather than extending `ReturnRequest` directly.
- `IWorkflowMessageService` — send refund approval/rejection emails.
- `IEventPublisher` — publish and consume custom domain events from Plugin 2.
- `IRepository<T>` — all data access to plugin-owned tables.
- `IScheduleTaskService` — register a reconciliation snapshot task.

---

## Dependencies on Other Plugins

This plugin depends on Plugin 1 and Plugin 2:

```json
"DependsOnSystemNames": [
  "Marketplace.SupplierManagement",
  "Marketplace.CheckoutAndPricing"
]
```

- Reads `OrderItemSupplierMapping` (Plugin 2) to determine per-supplier order item
  values. Resolve `ISupplierShipmentService` via DI.
- Reads `SupplierFeeConfig` and `SupplierManagementSettings` (Plugin 1) via
  `IMarketplacePriceService` for markup and fee percentages.
- Consumes `SupplierShipmentDeliveredEvent` and `SupplierShipmentCancelledEvent`
  published by Plugin 2.
- Never reference Plugin 1 or Plugin 2 project assemblies directly.

---

## New Database Tables Required

### `SupplierLedgerEntry` table
One record per financial event per supplier. This is the append-only ledger.

```csharp
public class SupplierLedgerEntry : BaseEntity
{
    public int VendorId { get; set; }              // FK to Vendor.Id
    public int OrderId { get; set; }               // FK to Order.Id
    public int? SupplierShipmentId { get; set; }   // FK to SupplierShipment.Id (Plugin 2)
    public int EntryTypeId { get; set; }           // enum: OrderRevenue=10, MarkupDeducted=20,
                                                   //       ServiceFeeDeducted=30, DeliveryFeeDeducted=40,
                                                   //       RefundDeducted=50, PayoutIssued=60,
                                                   //       RecurringOrderRevenue=70
    public decimal Amount { get; set; }            // positive = credit, negative = debit
    public string Description { get; set; }
    public DateTime CreatedOnUtc { get; set; }
    public bool IsReconciled { get; set; }         // marked true when included in a payout
    public int? ReconciliationReportId { get; set; } // FK to ReconciliationReport.Id
}
```

### `ReconciliationReport` table
A snapshot of a reconciliation period for admin review and payout tracking.

```csharp
public class ReconciliationReport : BaseEntity
{
    public int VendorId { get; set; }              // FK to Vendor.Id
    public DateTime PeriodStartUtc { get; set; }
    public DateTime PeriodEndUtc { get; set; }
    public int TotalOrders { get; set; }
    public decimal TotalOrderRevenue { get; set; }
    public decimal TotalMarkupCollected { get; set; }
    public decimal TotalServiceFeesCollected { get; set; }
    public decimal TotalDeliveryFeesCollected { get; set; }
    public decimal TotalRefundsIssued { get; set; }
    public decimal NetPayableToSupplier { get; set; }  // revenue minus all deductions
    public int PayoutStatusId { get; set; }            // enum: Pending=10, Paid=20, PartiallyPaid=30
    public decimal AmountPaid { get; set; }
    public DateTime? PaidOnUtc { get; set; }
    public string AdminNotes { get; set; }
    public DateTime GeneratedOnUtc { get; set; }
}
```

### `ShipmentReturnRequest` table
Per-shipment return/refund request. Separate from nopCommerce's `ReturnRequest`
which is per order item.

```csharp
public class ShipmentReturnRequest : BaseEntity
{
    public int SupplierShipmentId { get; set; }    // FK to SupplierShipment.Id (Plugin 2)
    public int OrderId { get; set; }               // FK to Order.Id
    public int CustomerId { get; set; }            // FK to Customer.Id
    public int VendorId { get; set; }              // FK to Vendor.Id
    public string Reason { get; set; }
    public string CustomerComments { get; set; }
    public decimal RequestedRefundAmount { get; set; }
    public decimal ApprovedRefundAmount { get; set; }
    public int ReturnStatusId { get; set; }        // enum: Pending=10, Approved=20, Rejected=30, Recovered=40
    public bool SupplierAlreadyPaid { get; set; }  // true = recovery needed; false = direct refund
    public decimal RecoveredAmount { get; set; }   // amount recovered from supplier so far
    public string AdminNotes { get; set; }
    public DateTime CreatedOnUtc { get; set; }
    public DateTime? ReviewedOnUtc { get; set; }
    public int? ReviewedByCustomerId { get; set; }
}
```

### `RecurringOrderPaymentSplit` table
Tracks per-supplier payment allocation for recurring billing orders.

```csharp
public class RecurringOrderPaymentSplit : BaseEntity
{
    public int OrderId { get; set; }               // FK to Order.Id (the auto-placed recurring order)
    public int VendorId { get; set; }              // FK to Vendor.Id
    public decimal GrossAmount { get; set; }       // supplier's items total before deductions
    public decimal MarkupDeducted { get; set; }
    public decimal ServiceFeeDeducted { get; set; }
    public decimal DeliveryFeeDeducted { get; set; }
    public decimal NetPayableToSupplier { get; set; }
    public int AllocationStatusId { get; set; }    // enum: Pending=10, Reconciled=20
    public DateTime CreatedOnUtc { get; set; }
}
```

---

## Feature Requirements

### F-2.4 — Reconciliation Reporting

**Ledger population:**
- Listen to `SupplierShipmentDeliveredEvent` (from Plugin 2) via
  `IConsumer<SupplierShipmentDeliveredEvent>`.
- On delivery confirmed, create `SupplierLedgerEntry` records for the supplier:
  1. `OrderRevenue` entry: sum of `OrderItem.PriceExclTax` for items in the shipment
     (from `OrderItemSupplierMapping`).
  2. `MarkupDeducted` entry: negative amount = markup collected by platform.
  3. `ServiceFeeDeducted` entry: negative amount = service fee portion attributable
     to this supplier's items.
  4. `DeliveryFeeDeducted` entry: negative amount = delivery fee for this shipment.
- The net of these four entries = amount owed to the supplier for this shipment.

**Reconciliation report generation (admin):**
- Admin page: "Reconciliation Reports" with date range filter and supplier filter.
- Admin clicks "Generate Report" for a supplier + period:
  1. Query all unreconciled `SupplierLedgerEntry` records for the supplier within
     the period.
  2. Aggregate into a `ReconciliationReport` record.
  3. Mark all included ledger entries as `IsReconciled = true` and set
     `ReconciliationReportId`.
- Report detail page shows:
  - Period summary: total orders, total revenue, markup collected, service fees,
    delivery fees, refunds, net payable.
  - Line-by-line ledger entries for the period.
  - Payout status and amount paid.
- Admin can mark a report as Paid (enter amount paid, date, notes).
- Admin can export the report to CSV/Excel.

**Dashboard summary widget:**
- Admin dashboard widget showing: total platform commission earned (current month),
  total service fees collected, total payout balance owed across all suppliers.
- Implement as a widget injected into the admin dashboard using `IWidgetPlugin`
  and the `admin_dashboard_top` widget zone.
- This means the plugin implements both `IMiscPlugin` and `IWidgetPlugin`.

---

### F-6.1 — Manual Refund / Return

**Buyer — submit return request:**
- Buyer account page: "My Orders" shows each `SupplierShipment` with a "Request
  Return" button (shown only for shipments in `Delivered` status).
- Return request form: reason (dropdown), comments (text), requested refund amount
  (pre-filled with shipment total, editable).
- On submission, create a `ShipmentReturnRequest` record with status `Pending`.
- Determine `SupplierAlreadyPaid`:
  - Check if a `ReconciliationReport` exists for this supplier and order with
    `PayoutStatus = Paid`. If yes, `SupplierAlreadyPaid = true`.
  - Otherwise, `SupplierAlreadyPaid = false`.
- Notify admin by email using message template `Marketplace.ReturnRequestSubmitted`.

**Admin — review return request:**
- Admin page: "Return Requests" lists all `ShipmentReturnRequest` records with
  status filter (Pending / Approved / Rejected / Recovered).
- Admin actions:
  - **Approve**: set status to `Approved`, set `ApprovedRefundAmount`.
    - If `SupplierAlreadyPaid = false`: trigger direct refund via the payment gateway
      (call `IPaymentService.RefundAsync` on the original order). Create a
      `SupplierLedgerEntry` of type `RefundDeducted` for the supplier.
    - If `SupplierAlreadyPaid = true`: do NOT issue a gateway refund. Instead, log
      the amount in the supplier's ledger as a `RefundDeducted` entry. The amount
      will be recovered from the supplier's next payout. Set status to `Recovered`
      once the deduction appears in a reconciliation report.
  - **Reject**: set status to `Rejected`. Send rejection email to buyer using
    message template `Marketplace.ReturnRequestRejected`.
- Admin can add notes to any return request.

**Recovery tracking:**
- When a reconciliation report is generated for a supplier, include any outstanding
  `ShipmentReturnRequest` records where `SupplierAlreadyPaid = true` and
  `ReturnStatus = Approved` as `RefundDeducted` ledger entries.
- Update `ShipmentReturnRequest.RecoveredAmount` and set status to `Recovered`
  once included in a paid reconciliation report.

---

### F-7.8 — Split Payment for Recurring Billing

**Trigger:**
- Listen to `OrderPlacedEvent` via `IConsumer<OrderPlacedEvent>`.
- Check if the order originated from a recurring schedule (Plugin 5 sets a custom
  order attribute `IsRecurringOrder = true` on auto-placed orders).
- If it is a recurring order, create `RecurringOrderPaymentSplit` records.

**Split calculation:**
- For each distinct `VendorId` in the order (from `OrderItemSupplierMapping`):
  1. `GrossAmount` = sum of `OrderItem.PriceExclTax` for that vendor's items.
  2. `MarkupDeducted` = `GrossAmount - (sum of supplier base prices for those items)`.
     Base prices come from `OrderItemSupplierMapping.SupplierBasePrice`.
  3. `ServiceFeeDeducted` = proportional share of the order's service fee
     (vendor's gross / order subtotal * total service fee).
  4. `DeliveryFeeDeducted` = delivery fee for that vendor's shipment from
     `SupplierShipment.DeliveryFee`.
  5. `NetPayableToSupplier` = `GrossAmount - MarkupDeducted - ServiceFeeDeducted - DeliveryFeeDeducted`.
- Create one `RecurringOrderPaymentSplit` record per vendor.
- These records feed into the reconciliation report when the supplier's shipment
  is delivered.

**Ledger integration:**
- When `SupplierShipmentDeliveredEvent` fires for a recurring order's shipment,
  the ledger population logic (F-2.4) applies identically. The `RecurringOrderPaymentSplit`
  record provides the pre-calculated split amounts for accuracy.

---

## Admin UI Pages Required

| Page | Route | Description |
|---|---|---|
| Reconciliation Reports | `Admin/FinanceAndReconciliation/Reports` | List reports, generate new |
| Report Detail | `Admin/FinanceAndReconciliation/ReportDetail/{id}` | Full ledger view, mark paid |
| Return Requests | `Admin/FinanceAndReconciliation/ReturnRequests` | List and review return requests |
| Return Request Detail | `Admin/FinanceAndReconciliation/ReturnDetail/{id}` | Approve/reject, add notes |
| Ledger | `Admin/FinanceAndReconciliation/Ledger/{vendorId}` | Raw ledger entries per supplier |

## Buyer Portal UI Pages Required

| Page | Route | Description |
|---|---|---|
| Submit Return | `Customer/SubmitReturn/{supplierShipmentId}` | Return request form |
| My Returns | `Customer/MyReturns` | Buyer's return request history |

---

## Widget Zone Integration (Dashboard)

```csharp
// FinanceAndReconciliationPlugin.cs — implements both IMiscPlugin and IWidgetPlugin
public Task<IList<string>> GetWidgetZonesAsync()
{
    return Task.FromResult<IList<string>>(new List<string>
    {
        AdminWidgetZones.DashboardTop  // or "admin_dashboard_top"
    });
}

public string GetWidgetViewComponent(string widgetZone)
{
    return "FinanceDashboardSummary";
}

public bool HideInWidgetList => true;  // it's primarily a misc plugin
```

Register in `InstallAsync`:
```csharp
if (!_widgetSettings.ActiveWidgetSystemNames.Contains(FinanceAndReconciliationDefaults.SystemName))
{
    _widgetSettings.ActiveWidgetSystemNames.Add(FinanceAndReconciliationDefaults.SystemName);
    await _settingService.SaveSettingAsync(_widgetSettings);
}
```

---

## Event Consumers Required

```csharp
public class EventConsumer :
    IConsumer<SupplierShipmentDeliveredEvent>,    // populate ledger on delivery
    IConsumer<SupplierShipmentCancelledEvent>,    // create refund deduction entry
    IConsumer<OrderPlacedEvent>                   // create split records for recurring orders
```

Note: `SupplierShipmentDeliveredEvent` and `SupplierShipmentCancelledEvent` are
defined in Plugin 2's `Domain/` folder. Consume them via the shared event bus
(`IEventPublisher`) — no direct project reference to Plugin 2.

---

## Install / Uninstall Checklist

**InstallAsync:**
1. Create DB tables via `AutoReversingMigration` tagged `MigrationProcessType.Installation`:
   `SupplierLedgerEntry`, `ReconciliationReport`, `ShipmentReturnRequest`,
   `RecurringOrderPaymentSplit`.
2. Add message templates: `Marketplace.ReturnRequestSubmitted`,
   `Marketplace.ReturnRequestRejected`.
3. Register as active widget (for dashboard summary).
4. Add locale resources with prefix `Plugins.Misc.FinanceAndReconciliation.`.
5. Call `await base.InstallAsync()` last.

**UninstallAsync:**
1. Remove from active widgets.
2. Delete locale resources.
3. Remove message templates.
4. Call `await base.UninstallAsync()` last.
5. Do NOT drop DB tables on uninstall.

---

## Folder Structure

```
Nop.Plugin.Misc.FinanceAndReconciliation/
├── plugin.json
├── Nop.Plugin.Misc.FinanceAndReconciliation.csproj
├── FinanceAndReconciliationPlugin.cs    ← BasePlugin + IMiscPlugin + IWidgetPlugin
├── FinanceAndReconciliationDefaults.cs  ← static constants
├── Events.cs                            ← IConsumer<T> implementations
├── logo.png
├── Controllers/
│   ├── FinanceAdminController.cs        ← reconciliation and return request admin
│   └── CustomerReturnController.cs      ← buyer return submission
├── Components/
│   └── FinanceDashboardSummaryViewComponent.cs  ← admin dashboard widget
├── Domain/
│   ├── SupplierLedgerEntry.cs
│   ├── ReconciliationReport.cs
│   ├── ShipmentReturnRequest.cs
│   ├── RecurringOrderPaymentSplit.cs
│   ├── LedgerEntryType.cs               ← enum
│   ├── PayoutStatus.cs                  ← enum
│   └── ReturnStatus.cs                  ← enum
├── Data/
│   ├── SupplierLedgerEntryBuilder.cs
│   ├── ReconciliationReportBuilder.cs
│   ├── ShipmentReturnRequestBuilder.cs
│   └── RecurringOrderPaymentSplitBuilder.cs
├── Factories/
│   ├── IReconciliationModelFactory.cs
│   └── ReconciliationModelFactory.cs
├── Infrastructure/
│   ├── NopStartup.cs
│   └── RouteProvider.cs
├── Migrations/
│   └── SchemaMigration.cs
├── Models/
│   ├── ReconciliationReportListModel.cs
│   ├── ReconciliationReportDetailModel.cs
│   ├── ReturnRequestListModel.cs
│   ├── ReturnRequestDetailModel.cs
│   ├── SubmitReturnModel.cs
│   ├── LedgerEntryModel.cs
│   └── DashboardSummaryModel.cs
├── Services/
│   ├── ILedgerService.cs
│   ├── LedgerService.cs
│   ├── IReconciliationService.cs
│   ├── ReconciliationService.cs
│   ├── IReturnRequestService.cs         ← plugin's own, not nopCommerce's
│   └── ReturnRequestService.cs
└── Views/
    ├── _ViewImports.cshtml
    ├── Reports.cshtml
    ├── ReportDetail.cshtml
    ├── ReturnRequests.cshtml
    ├── ReturnDetail.cshtml
    ├── Ledger.cshtml
    ├── SubmitReturn.cshtml
    ├── MyReturns.cshtml
    └── DashboardSummary.cshtml
```

---

## Key Constraints and Rules

- Follow all rules in `plugin-rule.md` without exception.
- `SupplierLedgerEntry` is append-only. Never update or delete ledger entries —
  corrections are made by adding new entries with opposite signs.
- The `ReconciliationReport` generation must be idempotent: if a report already
  exists for a supplier + period, do not create a duplicate. Check before generating.
- Payment gateway refund calls (`IPaymentService.RefundAsync`) must only be made
  when `SupplierAlreadyPaid = false`. Double-check this condition before calling.
- The plugin implements `IWidgetPlugin` as a secondary interface. Set
  `HideInWidgetList = true` so it does not appear in the widget list page.
- All locale resource keys must use the prefix `Plugins.Misc.FinanceAndReconciliation.`.
- `plugin.json` must declare:
  ```json
  "DependsOnSystemNames": ["Marketplace.SupplierManagement", "Marketplace.CheckoutAndPricing"]
  ```
- The `IsRecurringOrder` order attribute key must be agreed with Plugin 5 and stored
  as a constant in a shared location. Since plugins cannot reference each other,
  define the key string in both plugins' `Defaults.cs` files with the same value:
  `"Marketplace.IsRecurringOrder"`.
