# Epic 18 — Customer Dashboard — AI Coding Prompts

Repo: `tablogenix-ecommerce-main` (Django + DRF backend in `backend/`, React frontend in `frontend/`)

**How to use this document:** same as prior epics — each task is a standalone prompt, feed one at a time in order, commit/review before the next.

**Assumed prerequisites:** Epics 1–17 are fully merged.

**Important grounded discovery for this epic — read before starting anything:** `frontend/src/pages/Account.jsx` already implements a **complete, working tabbed account UI** — `OrdersTab.jsx` (317 lines, including a full `OrderDetailModal`), `AddressesTab.jsx` (280 lines), `SettingsTab.jsx` (407 lines, **already including a fully working notification-preferences UI for `order_updates`/`promotions`/`newsletter`**), `WishlistTab.jsx`, `ReviewsTab.jsx`, and `PaymentMethodsTab.jsx`. This means most of this epic's tasks are **audit-and-extend** work against real, substantial existing components — not greenfield builds. Read each task's grounding carefully before assuming a full rebuild is needed.

**Three real, concrete bugs/gaps confirmed directly in this existing code, relevant across this epic's tasks:**
1. **`OrdersTab.jsx`'s `fmt()` helper and `PaymentMethodsTab.jsx`'s `fmt()` helper both still format money as USD** — `new Intl.NumberFormat("en-US", { style: "currency", currency: "USD" }).format(n)` — completely unmigrated from Epic 14's Rial-storage/Toman-display work. Every price shown in the account area is currently mislabeled and almost certainly numerically wrong post-Epic-14 (a raw Rial integer formatted as if it were USD dollars-and-cents). This must be fixed as part of this epic's work, not left as-is.
2. **`AddressesTab.jsx` currently still uses the OLD, pre-Epic-5 field names** — `state`, `zip_code`, and a hardcoded `COUNTRIES = [{code: "US", ...}]` list defaulting to `country: "US"`. Epic 5 Task 5.2.1.4 renamed the backend `Address` model's fields to `province` (an `IranProvince`-choice field) and `postal_code` (10-digit validated), defaulting `country` to `"IR"`. **If Epic 5 Task 5.2.1.5's frontend work was completed as specified, this should already be fixed** — but if you're finding this document out of the exact sequence assumed, or that task was skipped/incomplete, Task 18.1.1.3 below is where this gets caught and corrected for good.
3. **`PaymentMethodsTab.jsx` derives its entire "payment methods" concept from historical `order.payment_method` string values** (`card`/`paypal`/`bank_transfer`/`cash_on_delivery`) — a concept that predates Epic 6's real ZarinPal/Zibal/IDPay gateway integration entirely. There is no such thing as a stored "payment method" a customer has on file in this platform anymore (real payment happens via gateway redirect each time, per Epic 6 — nothing is saved). This component is showing the customer a fabricated, increasingly-nonsensical summary of a concept that no longer exists in the actual system. This is flagged for a decision in Task 18.1.1.1.

---

## Phase 18.1 — Self-Service Account Features

### Feature 18.1.1 — Account Pages

---

#### Task 18.1.1.1 — Order history with status timeline

```
You are working in frontend/src/components/OrdersTab.jsx. Assume
Epics 1–17 are fully merged, including Epic 7's `Shipment` model/
tracking (Task 7.2.2.1-3) and Epic 8's admin order state-machine work
(Task 8.1.1.1).

CONTEXT — READ THIS DOCUMENT'S HEADER BEFORE STARTING
`OrdersTab.jsx` already has a working order list, filtering by status,
pagination, and an `OrderDetailModal` fetching full order data via
`dashboardAPI.getOrder(orderId)`. It currently shows only a flat status
BADGE ("processing", "shipped", etc.) — no visual timeline, no shipment
tracking detail (per Epic 7's `Shipment` model, which by this point has
real carrier/tracking-number/status data available via
`OrderSerializer`'s nested `shipment` field, per Epic 7 Task 7.2.2.3),
and its `fmt()` currency helper is still hardcoded to USD formatting
(a confirmed, real bug per this document's header — Rial integers are
currently being displayed as if they were USD amounts).

TASK
1. Fix the USD currency-formatting bug.
2. Add a visual order-status timeline to the order detail modal.
3. Surface Epic 7's shipment tracking data (carrier, tracking number,
   status) within that same timeline/detail view.
4. Make a deliberate decision about `PaymentMethodsTab.jsx`'s
   obsolescence (per this document's header context).

REQUIREMENTS — Part 1: currency fix
- Replace `OrdersTab.jsx`'s `fmt()` function:
  ```javascript
  import { formatToman } from '../utils/currency'; // Epic 14 Task 14.1.2.3

  const fmt = (rialAmount) => formatToman(rialAmount);
  ```
  Remove the old `Intl.NumberFormat("en-US", { style: "currency", currency: "USD" })`
  implementation entirely. Apply this fix everywhere `fmt()` is called
  in this file (order totals, item line prices in the detail modal).

REQUIREMENTS — Part 2: status timeline
- In `OrderDetailModal`, replace (or augment — your call on visual
  design, but a proper step-indicator reads better than a bare badge
  for this purpose) the current single status badge with a horizontal/
  vertical step timeline showing the order's progression through
  `Order.Status` (`pending → processing → shipped → delivered`, or
  a distinct terminal `cancelled` state rendered differently — e.g.
  a red "Cancelled" indicator replacing the normal progression rather
  than trying to force it into the same linear step sequence):
  ```jsx
  const ORDER_STEPS = ['pending', 'processing', 'shipped', 'delivered'];

  function OrderTimeline({ status }) {
    if (status === 'cancelled') {
      return <div className="text-red-600 font-semibold">این سفارش لغو شده است</div>;
    }
    const currentIndex = ORDER_STEPS.indexOf(status);
    return (
      <div className="flex items-center">
        {ORDER_STEPS.map((step, i) => (
          <div key={step} className="flex items-center flex-1">
            <div className={`w-8 h-8 rounded-full flex items-center justify-center ${i <= currentIndex ? 'bg-teal-600 text-white' : 'bg-gray-200 text-gray-400'}`}>
              {i < currentIndex ? <i className="bi bi-check" /> : i + 1}
            </div>
            {i < ORDER_STEPS.length - 1 && (
              <div className={`flex-1 h-1 ${i < currentIndex ? 'bg-teal-600' : 'bg-gray-200'}`} />
            )}
          </div>
        ))}
      </div>
    );
  }
  ```
  (adjust styling to match this project's established design system —
  this is illustrative structure, not final visual polish; use Persian
  step labels beneath each indicator per Epic 14's localization work —
  "در انتظار", "در حال پردازش", "ارسال شده", "تحویل داده شده" or
  whatever the project's established Persian order-status translations
  are by this point, per Epic 14 Task 14.1.1.3/4's translation work —
  check for existing translated status labels rather than inventing new
  wording).

REQUIREMENTS — Part 3: shipment tracking integration
- `OrderSerializer`'s `shipment` field (Epic 7 Task 7.2.2.3) is already
  included in whatever `dashboardAPI.getOrder(orderId)` returns (this
  is the SAME underlying `Order` API, just consumed here via the
  dashboard app — confirm the dashboard's order-detail serializer ALSO
  includes this nested shipment data; if the `dashboard` app has its
  own separate `AdminOrderSerializer`/customer-facing order serializer
  that DOESN'T yet include the `shipment` field Epic 7 added to the
  main `order` app's `OrderSerializer`, add it here — don't assume it
  automatically propagated just because Epic 7 added it to one
  serializer). When `order.shipment` is present, render carrier name,
  tracking number, and current shipment status within the detail modal
  (reusing/adapting the tracking widget component Epic 7 Task 7.2.2.3
  already built for the main order-detail page, rather than building a
  second, separate tracking-display component — check whether that
  component can be reused directly inside this modal context, or
  extract its rendering logic into a shared sub-component both places
  can use).

REQUIREMENTS — Part 4: PaymentMethodsTab decision
- `PaymentMethodsTab.jsx`'s entire premise (deriving "payment methods on
  file" from historical `order.payment_method` string values) no longer
  corresponds to how this platform actually processes payment since
  Epic 6. Make and implement a deliberate decision:
  (a) REMOVE the tab entirely (simplest, most honest option — this
  platform genuinely has no "saved payment methods" concept anymore,
  and showing a fabricated summary of one is actively misleading), or
  (b) REPURPOSE it into a genuine "Payment History" view — a list of
  actual `PaymentTransaction` records (Epic 6 Task 6.1.1.2) for the
  customer's orders, showing gateway used, amount, status, date — which
  IS a real, meaningful, currently-absent piece of account information.
  RECOMMEND option (b) — repurposing rather than deleting — since
  "payment history" is genuinely useful account information a real
  customer would want (distinct from order history, which is about
  WHAT was ordered, not HOW it was paid for), and the tab/nav slot
  already exists; implement it by adding a
  `GET /dashboard/customer/payment-history/` endpoint (or reusing/
  filtering an existing payments endpoint scoped to
  `request.user`) and rewriting `PaymentMethodsTab.jsx`'s data-fetching
  and rendering to show real `PaymentTransaction` data instead of the
  derived-from-order-history fabrication, using `formatToman()` for
  amounts (closing the SAME USD-formatting bug in this file too, per
  this document's header).

ACCEPTANCE CRITERIA / TESTS
- Manually verify: order amounts throughout `OrdersTab.jsx` now display
  correctly formatted in Toman, not mislabeled/miscalculated USD; the
  status timeline correctly reflects each of the 5 possible order
  statuses; an order with an associated shipment shows tracking detail.
- Add/update component tests for `OrdersTab.jsx`:
  1. `fmt()`/price display correctly uses `formatToman()`.
  2. `OrderTimeline` renders the correct step highlighted for each
     status value, and the distinct cancelled-state rendering for
     `cancelled`.
  3. The detail modal correctly renders shipment tracking info when
     `order.shipment` is present, and doesn't render a broken/empty
     tracking section when it's absent (order not yet shipped).
- Add tests for whichever `PaymentMethodsTab.jsx` decision you
  implemented — if repurposed to real payment history, test that it
  correctly fetches and displays the requesting user's OWN
  `PaymentTransaction` records only (never another user's), with
  correctly formatted Toman amounts.
```

---

#### Task 18.1.1.2 — Downloadable invoice from order detail

```
You are working in frontend/src/components/OrdersTab.jsx. Assume Epic
8 Task 8.1.1.5's `GET /api/orders/{id}/invoice/` endpoint (PDF
generation, permission-checked for the order's owner or staff) is
already merged, and Task 18.1.1.1 is already merged.

CONTEXT
The backend invoice-generation endpoint has existed since Epic 8, but
nothing in the frontend account UI links to it — per Epic 8 Task
8.1.1.5's own note at the time, it explicitly flagged that "a plain
anchor-tag download would not carry the auth header" and that a
`responseType: 'blob'` fetch-then-download approach was needed instead
— that frontend wiring was never actually built as part of Epic 8
(which was correctly scoped to the backend endpoint only); this task
is where it finally happens.

TASK
Add a "Download Invoice" button to the order detail modal, correctly
handling the authenticated binary-PDF-download pattern.

REQUIREMENTS
- Add to frontend/src/services/api.js (check for the existing
  `dashboardAPI` object's conventions and match them):
  ```javascript
  downloadInvoice: (orderId) =>
    api.get(`/orders/${orderId}/invoice/`, { responseType: 'blob' }),
  ```
  Note this correctly goes through the SAME `api` axios instance
  already configured with the Authorization header interceptor
  established since early epics — `responseType: 'blob'` tells axios
  to treat the response as binary data rather than attempting to parse
  it as JSON, which is essential for a PDF response.
- In `OrderDetailModal`, add a "Download Invoice" button that, on
  click:
  ```javascript
  const handleDownloadInvoice = async () => {
    setDownloading(true);
    try {
      const response = await dashboardAPI.downloadInvoice(order.id);
      const blob = new Blob([response.data], { type: 'application/pdf' });
      const url = window.URL.createObjectURL(blob);
      const link = document.createElement('a');
      link.href = url;
      link.download = `invoice_${order.order_number}.pdf`;
      document.body.appendChild(link);
      link.click();
      link.remove();
      window.URL.revokeObjectURL(url);
    } catch (err) {
      // show a toast/error state — don't fail silently
      setDownloadError('دانلود فاکتور با خطا مواجه شد. لطفاً دوباره تلاش کنید.');
    } finally {
      setDownloading(false);
    }
  };
  ```
  This create-object-URL-then-programmatic-click pattern is the
  standard, correct way to trigger a browser file download from a
  blob response obtained via an authenticated API call (a plain
  `<a href="/api/orders/1/invoice/">` would NOT carry the
  Authorization header and would 401, exactly as flagged back in Epic
  8's own note) — `window.URL.revokeObjectURL(url)` afterward is
  important cleanup, releasing the temporary object URL's memory
  rather than leaking it.
- Only show the "Download Invoice" button for orders in a state where
  an invoice actually makes sense — a `PENDING` (unpaid, per Epic 6's
  order-status flow) order shouldn't offer an invoice download at all,
  since no payment has actually been confirmed yet; show the button
  only for `processing`/`shipped`/`delivered` orders (i.e. anything
  past confirmed payment).
- Add a loading/disabled state on the button while the download is in
  progress (PDF generation, per Epic 8 Task 8.1.1.5, involves real
  server-side rendering work via WeasyPrint and isn't instant — the UI
  should clearly indicate it's working, not appear unresponsive).

ACCEPTANCE CRITERIA / TESTS
Add component tests for the invoice-download button:
1. Renders and is clickable for a `processing`/`shipped`/`delivered`
   order; is HIDDEN for a `pending` order.
2. Clicking it calls `dashboardAPI.downloadInvoice` with the correct
   order ID (mock the API call — actual blob/browser-download behavior
   isn't meaningfully testable in a JSDOM test environment, so mock at
   the point of the API call and verify it was invoked correctly,
   rather than trying to assert real browser download behavior).
3. A failed download (mocked API rejection) shows the error state
   rather than failing silently, and re-enables the button for retry.
Manually verify in a real browser: clicking the button actually
downloads a real PDF file with the correct filename, and the PDF opens
correctly showing accurate order details.
```

---

#### Task 18.1.1.3 — Address book management UI

```
You are working in frontend/src/components/AddressesTab.jsx. Assume
Epic 5 Task 5.2.1.4 (backend `Address` model with `province`/
`postal_code`/`country="IR"` default) is merged, and Epic 5 Task
5.2.1.5 (which was SUPPOSED to build/fix this exact frontend component)
may or may not have actually been completed depending on your real
build sequence.

CONTEXT — VERIFY BEFORE ASSUMING EITHER STATE
Per this document's header: `AddressesTab.jsx`, AS ORIGINALLY WRITTEN
before any Epic 5 frontend work, uses the OLD `state`/`zip_code` field
names and a hardcoded US-only `COUNTRIES` list. Epic 5 Task 5.2.1.5
was explicitly scoped to fix exactly this. YOUR FIRST STEP in this task
is to check the ACTUAL CURRENT STATE of this file: if it already
correctly uses `province` (with an `IranProvince` dropdown) and
`postal_code` (with 10-digit validation), Epic 5's task was completed
and THIS task becomes pure verification/polish. If it STILL shows the
old `state`/`zip_code`/US-country fields, Epic 5's frontend task was
skipped or incomplete, and this task must do the full fix now — the
account section literally cannot function correctly for checkout
address-book use (per Epic 5 Task 5.2.1.3's `address_id`-based
checkout, which expects `province`/`postal_code` data) until this is
corrected.

TASK
Ensure `AddressesTab.jsx` is fully aligned with the backend `Address`
model's actual current fields (`province`, `postal_code`, `country`
defaulting to `"IR"`), and polish the overall address-book management
UX.

REQUIREMENTS (apply whichever subset is still actually needed, per
your verification above)
- Replace the `state` field with a `province` `<select>` populated from
  Iran's 31 provinces, matching the EXACT `IranProvince` choices/values
  the backend expects (per Epic 5 Task 5.2.1.4's enum) — either
  hardcode the list matching the backend exactly (documenting that
  these two lists must be kept in sync, per the same tradeoff already
  discussed in Epic 5 Task 5.2.1.5), or fetch valid choices dynamically
  via a DRF `OPTIONS` request against the addresses endpoint (preferred
  if not meaningfully more complex, since it stays automatically in
  sync with the backend and can't drift).
- Replace `zip_code` with `postal_code`, with client-side validation
  matching the backend's 10-digit format requirement (immediate
  feedback before submission, while the backend remains the
  authoritative validator).
- Replace the hardcoded `COUNTRIES = [{code: "US", ...}]` list and
  `country: "US"` default with an Iran-appropriate default
  (`country: "IR"`) — since this platform is explicitly single-market
  (Iran) per the project's stated goal, consider whether the country
  FIELD needs to remain user-EDITABLE in the UI at all, or whether it's
  simpler/more honest to just fix it as a non-editable `"IR"` value in
  the form (removing the country selector entirely) — RECOMMEND
  removing the editable country selector, since offering a dropdown of
  countries for a platform that only ships within Iran is confusing UX
  that implies false capability; if the backend `Address.country` field
  itself should also become a fixed/non-editable value rather than a
  free CharField at this point, that's a reasonable small backend
  follow-up to flag, but isn't required to fix on the frontend
  regardless — the frontend can simply stop exposing a country picker.
- Update the address CARD display (`{addr.city}, {addr.state} {addr.zip_code}`
  per the confirmed old code) to `{addr.city}, {addr.province} {addr.postal_code}`,
  using the province's human-readable label (not its raw choice-key
  value, e.g. show "تهران" not `"tehran"`) — if you're consuming
  choices dynamically via `OPTIONS`, use the label that response
  provides; if hardcoded, maintain a key→label mapping alongside the
  key list.
- General UX polish while in this file: confirm the add/edit/delete
  flow works smoothly, the "set as default" action (per the existing
  `Address.is_default` field/`save()` override logic from before Epic
  5) is clearly indicated in the UI, and error states (e.g. backend
  validation rejecting a malformed postal code) surface clearly to the
  user rather than failing silently.

ACCEPTANCE CRITERIA / TESTS
Add/update component tests for `AddressesTab.jsx`:
1. The province field renders as a select populated with all 31 Iran
   provinces, and submitting a new address sends the correct
   `province`/`postal_code` field names (not the old `state`/`zip_code`)
   to the API.
2. Client-side postal code validation rejects a non-10-digit value
   before submission.
3. Existing addresses (fetched with `province`/`postal_code` data,
   matching the current real backend shape) display correctly on the
   address cards, showing the province's human-readable label.
4. "Set as default" correctly reflects and updates the default address
   state.
Manually verify end-to-end: adding a new address through this UI,
selecting it during checkout (per Epic 5 Task 5.2.1.5's checkout
picker), and confirming the resulting order's shipping fields correctly
reflect the chosen address's real data.
```

---

#### Task 18.1.1.4 — Notification preferences UI (SMS/email toggles)

```
You are working in frontend/src/components/SettingsTab.jsx. Assume
Epic 16 (unified `notify()` system, channel selection via
`Profile.order_updates`/`promotions`/`newsletter`) is already merged.

CONTEXT — READ THIS DOCUMENT'S HEADER BEFORE STARTING, THIS TASK IS
SMALLER THAN IT LOOKS
`SettingsTab.jsx` ALREADY has a fully working notification-preferences
UI for exactly the three `Profile` boolean fields
(`order_updates`/`promotions`/`newsletter`) this task's backlog
description calls for — confirmed present, with correct fetch/save
logic and clear labels/descriptions already in place. This task is
NOT a build-from-scratch. Its actual remaining scope: confirm this
existing UI correctly reflects how Epic 16's `notify()` system ACTUALLY
uses these three flags, and consider whether finer-grained CHANNEL
control (SMS vs. email specifically, rather than one blanket toggle
per event category) is worth adding.

TASK
Audit the existing notification-preferences UI against Epic 16's real
`_default_channels_for()` logic, and decide whether to extend it with
per-channel granularity.

REQUIREMENTS
- Re-read Epic 16 Task 16.1.1.4's `_default_channels_for()` function:
  it currently treats `order_updates=True` as "send BOTH SMS and email"
  for order-related events, and `promotions=True` as "send email only"
  for back-in-stock/price-drop events — meaning the CURRENT three-
  checkbox UI can't actually express "I want order SMS but not order
  email" or vice versa; it's coarser than what the backend's channel
  logic conceptually supports extending to. Decide: is this coarser
  three-checkbox UX ACCEPTABLE as the permanent design (simpler,
  fewer decisions for the customer, matches how most e-commerce sites
  actually do this), or does this platform genuinely need finer
  SMS-vs-email control per category? RECOMMEND keeping the existing
  three-checkbox UX as-is for now — it's simpler, it's already built
  and working, and Iranian consumers' actual SMS/email preference
  granularity needs aren't established by any real user research at
  this point in the project — adding complexity here without evidence
  it's needed would be premature. Document this as a deliberate
  "kept as-is, evaluate later if requested" decision rather than
  silently doing nothing without reasoning about it.
- If you DO decide finer granularity is warranted (e.g. if by this
  point in your real project other signals suggest it's needed), this
  would require: new `Profile` fields (`order_updates_sms`,
  `order_updates_email`, splitting the current single boolean), a
  corresponding backend migration, updated `_default_channels_for()`
  logic in Epic 16 Task 16.1.1.4, and updated UI — a genuinely larger,
  cross-cutting change spanning multiple epics' work; only take this on
  if you're confident it's warranted, not as a default choice for this
  task.
- Regardless of the granularity decision, verify the EXISTING save
  flow correctly persists changes and that the account's actual
  notification behavior (send a test order-status-changing action,
  per Epic 8 Task 8.1.1.2's signal, and confirm it correctly
  respects the customer's ACTUAL saved preference) works end-to-end —
  this is a genuine, valuable verification step even if no UI changes
  are needed: confirm the EXISTING toggle in `SettingsTab.jsx`
  genuinely, functionally controls Epic 16's real notification
  dispatch, not just that it saves a database value that nothing
  actually reads correctly.
- While reviewing this file, apply the SAME currency-formatting
  audit as Task 18.1.1.1 if `SettingsTab.jsx` displays any monetary
  values anywhere (check — a settings page may not have any, but
  confirm rather than assume, given how pervasive the USD-formatting
  bug turned out to be elsewhere in the account area).

ACCEPTANCE CRITERIA / TESTS
- Add an END-TO-END test (or as close to one as this project's testing
  setup allows) confirming: a user with `promotions=False` genuinely
  does NOT receive a `PRICE_DROP`/`BACK_IN_STOCK` notification (per
  Epic 16's `_default_channels_for()` logic) when the underlying event
  fires, and a user with `promotions=True` DOES — this is the real
  functional proof that this existing UI's toggle actually controls
  real behavior, not just a cosmetic settings-page checkbox.
- If finer channel granularity was added, add tests covering the new
  fields' save/fetch/dispatch-integration behavior with the same
  rigor.
```

---

#### Task 18.1.1.5 — Reorder ("buy again") button on past orders

```
You are working in frontend/src/components/OrdersTab.jsx and
backend/order/ (or cart/). Assume Task 18.1.1.1 is already merged.

CONTEXT
No "reorder" capability exists — a customer wanting to buy the same
items again from a past order currently has to manually search for and
re-add each product individually.

TASK
Add a "Reorder" button on past orders that re-adds all of that order's
items to the customer's current cart.

REQUIREMENTS — backend
- Add `POST /api/orders/{id}/reorder/` in backend/order/views.py:
  ```python
  class OrderReorderView(APIView):
      permission_classes = [IsAuthenticated]

      def post(self, request, pk):
          order = get_object_or_404(Order, pk=pk, user=request.user)
          cart = get_or_create_cart(request)  # Epic 5's cart-resolution helper

          added, unavailable = [], []
          for item in order.items.select_related("variant", "variant__product").all():
              variant = item.variant
              if variant is None or not variant.is_active or variant.stock <= 0:
                  unavailable.append({
                      "product_name": item.product_name,
                      "reason": "no longer available" if variant is None or not variant.is_active else "out of stock",
                  })
                  continue
              cart_item, created = CartItem.objects.get_or_create(
                  cart=cart, variant=variant, defaults={"quantity": item.quantity}
              )
              if not created:
                  cart_item.quantity += item.quantity
                  cart_item.save(update_fields=["quantity"])
              added.append(item.product_name)

          return Response(
              {"added_count": len(added), "added": added, "unavailable": unavailable},
              status=status.HTTP_200_OK,
          )
  ```
  Import `get_or_create_cart` from `cart.views`, `CartItem` from
  `cart.models`. Note this correctly handles EVERY failure mode a
  historical order's items can hit by this point in the project: a
  deleted variant (`item.variant is None`, per the `SET_NULL`
  relationship established since Epic 3), a deactivated variant (e.g.
  expired stock, per Epic 3 Task 3.3.1.3's sweep, or discontinued), or
  simply out-of-stock — in ALL these cases, the reorder should still
  succeed for whatever items ARE available, clearly reporting which
  ones weren't, rather than failing the entire operation over one
  unavailable item — mirroring the exact same "partial success with
  clear reporting" pattern already established for Epic 11 Task
  11.1.1.3's wishlist bulk "move all to cart" action.
  Register the URL:
  `path("<int:pk>/reorder/", views.OrderReorderView.as_view(), name="order-reorder"),`
  in order/urls.py.

REQUIREMENTS — frontend
- Add `reorderOrder: (orderId) => api.post(`/orders/${orderId}/reorder/`)`
  to api.js.
- Add a "Reorder" button on each order card/detail view in
  `OrdersTab.jsx` (reasonable to show for `delivered` orders primarily
  — reordering a still-`pending`/`processing` order is a less common,
  though not unreasonable, use case; decide whether to show it for
  every order status or just completed ones, and be consistent).
- On click, call the endpoint and show a clear result: how many items
  were added, and — importantly — which ones were NOT (with the
  reason, e.g. "این محصول دیگر موجود نیست" / "no longer available"),
  so the customer isn't confused about why their cart doesn't have
  everything from the original order.
- On success (at least one item added), offer a direct link/button to
  go to the cart (`navigate('/cart')`) so the customer can immediately
  proceed rather than having to separately find the cart icon.
- Trigger a `CartContext` refetch after a successful reorder so the
  cart badge/count updates immediately, consistent with every other
  cart-mutating action established across this project since Epic 5.

ACCEPTANCE CRITERIA / TESTS
Add backend tests to backend/order/tests/:
1. Reordering an order where all items are still available adds every
   item to the cart with correct quantities, summing correctly with
   any quantity ALREADY in the cart for the same variant.
2. Reordering an order with one deleted variant (`variant=None`) and
   one still-available variant correctly adds the available one and
   reports the unavailable one, without erroring the whole request.
3. Reordering an order with an out-of-stock variant reports it as
   unavailable with the correct reason.
4. A user attempting to reorder ANOTHER user's order gets 404 (the
   ownership filter).
Add frontend component tests for the reorder button: successful
reorder shows the correct added/unavailable summary; clicking "go to
cart" navigates correctly; the cart context refetches after a
successful reorder (mock and assert).
```

---

## Summary checklist

| Task | Feature | Status |
|---|---|---|
| 18.1.1.1 | Order history + status timeline (+ currency fix + PaymentMethodsTab decision) | ☐ |
| 18.1.1.2 | Downloadable invoice from order detail | ☐ |
| 18.1.1.3 | Address book management UI (province/postal_code alignment) | ☐ |
| 18.1.1.4 | Notification preferences UI (audit existing, verify integration) | ☐ |
| 18.1.1.5 | Reorder ("buy again") button | ☐ |

**Outstanding item carried over from Epic 17's grounding, still unresolved:** `dashboard/services.py`'s `get_user_summary()` computes a customer's `reviews_count` via `Review.objects.filter(name=full_name)` — a fragile string match that should use `Review.objects.filter(user=user).count()` (the real FK, present since Epic 1 Task 1.3.1.1). This feeds the `OverviewTab` component in `Account.jsx` (confirmed to consume `getUserSummary`-shaped data). If not already fixed, this is a natural, small addition to Task 18.1.1.1 or 18.1.1.4's session, since both already touch account-summary-adjacent code.

Once Epic 18 is fully merged, the next epics to generate prompts for
are **Epic 19 — Blog & Content** and **Epic 20 — Media Management**,
both lower-urgency epics that can run independently of each other and
of most other in-progress work at this point in the backlog.
