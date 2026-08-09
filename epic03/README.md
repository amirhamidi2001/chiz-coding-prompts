# Epic 3 — Product Catalog & Cosmetics Data Model — AI Coding Prompts

Repo: `chiz-ecommerce-main` (Django + DRF backend in `backend/`, React frontend in `frontend/`)

**How to use this document:** same as Epics 1 and 2 — each task is a standalone prompt, feed them one at a time in order, let each be committed/reviewed before starting the next. This is the largest and most structurally invasive epic in the backlog: it changes what "a product" even means in the data model (single stock/price → variants), so tasks within Feature 3.1.1 in particular MUST run in the exact order given — nothing later in the project can safely touch cart/order/stock logic until the variant migration is fully complete and verified.

**Assumed prerequisites:** Epic 1 (Core Backend Stability) and Epic 2 (Auth/OTP) are fully merged. Epic 1 specifically matters here: `order/serializers.py` `OrderCreateSerializer.create()` currently does `Product.objects.select_for_update()` and decrements `Product.stock` directly inside a `transaction.atomic()` block (built in Epic 1 Tasks 1.1.1.2/1.1.1.3) — several tasks below explicitly repoint that logic at the new `ProductVariant` model instead of `Product`.

---

## Phase 3.1 — Product Variant System

### Feature 3.1.1 — Variant Model

---

#### Task 3.1.1.1 — Design `ProductVariant` model

```
You are working in backend/shop/models.py.

CONTEXT
The current Product model (backend/shop/models.py) carries price and
stock directly on itself:

    class Product(models.Model):
        category = models.ForeignKey(Category, ...)
        brand = models.ForeignKey(Brand, ...)
        name = models.CharField(max_length=300)
        slug = models.SlugField(max_length=320, unique=True, blank=True)
        ...
        price = models.DecimalField(max_digits=10, decimal_places=2)
        original_price = models.DecimalField(..., null=True, blank=True)
        stock = models.PositiveIntegerField(default=0)
        ...

The only variation mechanism today is `ProductColor` (a plain
product-to-color join table with no price/stock/SKU of its own — it's
purely a "this product comes in these colors" tag, not a purchasable
unit). This doesn't work for cosmetics: a real product like a
foundation needs separate stock, price, and SKU per shade, and a
skincare serum needs separate stock/price/SKU per volume size
(30ml/50ml/100ml). Cart, checkout, and inventory all need to operate
on individually-trackable purchasable units, not on the Product row
itself.

TASK
Design and create a `ProductVariant` model representing one
purchasable unit (a specific shade/volume/size combination) of a
Product. This task is schema-design only — do NOT migrate existing
data yet (Task 3.1.1.2) and do NOT touch Cart/Order (Tasks 3.1.1.3/
3.1.1.4) in this task.

REQUIREMENTS
- Add to backend/shop/models.py:
  ```python
  class ProductVariant(models.Model):
      product = models.ForeignKey(
          Product, on_delete=models.CASCADE, related_name="variants"
      )
      sku = models.CharField(max_length=64, unique=True, blank=True)
      barcode = models.CharField(max_length=20, blank=True)
      color = models.ForeignKey(
          Color, on_delete=models.SET_NULL, null=True, blank=True,
          related_name="variants",
      )
      price = models.DecimalField(max_digits=10, decimal_places=2)
      original_price = models.DecimalField(
          max_digits=10, decimal_places=2, null=True, blank=True
      )
      stock = models.PositiveIntegerField(default=0)
      is_active = models.BooleanField(default=True)
      created_at = models.DateTimeField(auto_now_add=True)
      updated_at = models.DateTimeField(auto_now=True)

      class Meta:
          ordering = ["id"]

      def __str__(self):
          return f"{self.product.name} — {self.sku or 'unsaved'}"
  ```
  — reuse the existing `Color` model (from ProductColor) as the color
  FK on the variant directly, rather than inventing a new color-like
  field, since Color already exists and is admin-managed.
- Do NOT add `volume_ml`/`expiration_date`/`batch_number` fields yet —
  those belong to Task 3.2.1.4 and Task 3.2.1.7/3.2.1.8 respectively,
  which run after this task; keep this task's model minimal
  (identity/price/stock/color only) so later tasks can extend it
  cleanly without this task becoming a dumping ground for every future
  field.
- Generate the migration (`python manage.py makemigrations shop`).
  This migration ONLY creates the new `ProductVariant` table — it must
  not alter `Product`, `CartItem`, or `OrderItem` in any way in this
  task.
- Do NOT remove `Product.price`/`Product.stock` yet — Task 3.1.1.2
  handles the data migration and Task 3.1.1.4/3.1.1.5 handle repointing
  dependent logic; removing those fields prematurely would break the
  entire storefront before the migration path is ready. Leave them in
  place for now.

ACCEPTANCE CRITERIA / TESTS
- Migration applies cleanly with zero impact on existing data (it's a
  pure additive new table).
- Add a model test in backend/shop/tests/test_models.py confirming a
  `ProductVariant` can be created linked to an existing `Product`, with
  `str()` producing a readable representation.
- Confirm `Product.objects.get(pk=x).variants.all()` works via the
  `related_name="variants"` reverse accessor.
```

---

#### Task 3.1.1.2 — Migrate existing `Product.stock`/`price` to a default variant

```
You are working in backend/shop/migrations/. Assume Task 3.1.1.1
(ProductVariant model + migration) is already merged.

CONTEXT
Every existing Product row has real `price`, `original_price`, `stock`
data, and possibly related `ProductColor` rows, that all need to be
preserved as the new variant system takes over — this is a live
production dataset (or will be), not a greenfield schema, so this
migration must be lossless.

TASK
Write a Django data migration that creates exactly one `ProductVariant`
per existing `Product`, carrying over its price/original_price/stock,
and — for products that have `ProductColor` entries — creates one
variant PER COLOR (not just one default variant) so color-specific
variants exist from day one wherever color data already exists.

REQUIREMENTS
- Create the migration via
  `python manage.py makemigrations shop --empty --name migrate_products_to_variants`
  and implement the `RunPython` operations using the historical model
  API (`apps.get_model("shop", "Product")`, etc. — never import the
  real model classes into a data migration).
- Migration logic, per existing Product:
  1. If the product has one or more related `ProductColor` rows: create
     one `ProductVariant` per distinct color, each carrying the
     product's current `price`/`original_price`/`stock` — BUT split the
     stock somehow, since a single `stock` integer on Product can't be
     divided into N color variants without a decision: the simplest,
     safest choice is to put the FULL existing stock count on EVERY
     resulting variant is wrong (that would multiply total inventory by
     N colors, inflating stock). Instead, either (a) divide the
     existing stock evenly across the N color variants (integer
     division, remainder going to the first variant), or (b) put the
     full stock on only the FIRST variant and 0 on the rest, flagging
     for manual admin correction post-migration. Prefer (a) — even
     division — since it's less likely to make specific popular colors
     look completely out of stock overnight; document this choice
     clearly in a migration comment since it's a real business decision
     being made in code, not a technical detail.
  2. If the product has NO `ProductColor` rows: create exactly ONE
     `ProductVariant` with `color=None`, carrying the full existing
     price/original_price/stock unchanged.
  3. Generate a placeholder SKU for each created variant using a simple
     deterministic pattern (e.g. `f"LEGACY-{product.id}-{index}"`) —
     real SKU generation logic comes in Task 3.1.2.1, which will run
     AFTER this migration and can be used going forward for new
     variants; this migration just needs any valid unique placeholder
     so `unique=True` doesn't break.
  4. Set `is_active=True` on every created variant.
- Write a corresponding reverse migration function that deletes all
  `ProductVariant` rows whose SKU matches the `LEGACY-` prefix pattern
  used above (a safe, identifiable way to reverse just this migration's
  output without touching any variants created by later, real usage of
  the system after this migration ran).
- Run this migration against a copy of representative test data (via
  the project's existing `seed_shop` management command — run it in a
  scratch environment, then run this migration, then inspect results)
  to sanity-check the output before considering this done, rather than
  reasoning about it purely in the abstract.

ACCEPTANCE CRITERIA / TESTS
Write a migration test:
1. A Product with 3 ProductColor entries and stock=30 results in
   exactly 3 ProductVariant rows after migration, each with a distinct
   color, summing to stock=30 total (verifying the even-division
   logic).
2. A Product with 0 ProductColor entries and stock=15 results in
   exactly 1 ProductVariant with color=None and stock=15.
3. Every created variant's `price`/`original_price` matches the source
   Product's values exactly.
4. Running the reverse migration removes only the LEGACY- prefixed
   variants it created, not any hand-created ones.
```

---

#### Task 3.1.1.3 — Update `Cart`/`CartItem` to reference variant instead of product

```
You are working in backend/cart/models.py, backend/cart/serializers.py,
backend/cart/views.py (view/serializer file names — verify exact
filenames first via `find backend/cart -name "*.py"`, they weren't
directly enumerated in this prompt's context, so confirm before
editing). Assume Tasks 3.1.1.1 and 3.1.1.2 are already merged (every
Product now has at least one ProductVariant).

CONTEXT
The current CartItem model is:

    class CartItem(models.Model):
        cart = models.ForeignKey(Cart, on_delete=models.CASCADE, related_name="items")
        product = models.ForeignKey("shop.Product", on_delete=models.CASCADE, related_name="cart_items")
        quantity = models.PositiveIntegerField(default=1, validators=[MinValueValidator(1)])
        added_at = models.DateTimeField(auto_now_add=True)
        updated_at = models.DateTimeField(auto_now=True)

        class Meta:
            unique_together = ("cart", "product")

        @property
        def unit_price(self):
            return self.product.price

        @property
        def subtotal(self):
            return self.unit_price * self.quantity

This points directly at Product, which no longer holds authoritative
price/stock now that ProductVariant exists. A cart needs to reference a
SPECIFIC variant (a specific shade/size), not just "this product,
whichever variant happens to apply" — a customer choosing "Foundation —
Shade 320" must have their cart line tied to exactly that variant, not
to the Foundation product in the abstract.

TASK
Change `CartItem.product` to `CartItem.variant`, pointing at
`ProductVariant` instead of `Product`, and update all dependent
properties, serializers, and views.

REQUIREMENTS — model
- Change:
  `product = models.ForeignKey("shop.Product", on_delete=models.CASCADE, related_name="cart_items")`
  to:
  `variant = models.ForeignKey("shop.ProductVariant", on_delete=models.CASCADE, related_name="cart_items")`
- Change `unique_together = ("cart", "product")` to
  `unique_together = ("cart", "variant")` (two lines of the SAME
  variant can't both exist in a cart, but two DIFFERENT variants of the
  same product now correctly CAN coexist as separate cart lines — e.g.
  two different shades of the same foundation in one cart).
- Update `unit_price` property to `return self.variant.price`.
- Update `subtotal` property (unchanged logic, just reads through
  `variant` now).
- Update `__str__` to reference `self.variant.product.name` and
  whatever variant-identifying detail is available (color name if set).
- Generate a migration. THIS IS A SCHEMA-BREAKING CHANGE for any
  existing CartItem rows — decide explicitly how to handle existing
  data: the safest approach is a two-step migration (add nullable
  `variant` FK, data-migrate existing CartItems to point at each
  product's now-existing default/first variant from Task 3.1.1.2, THEN
  make `variant` non-nullable and drop `product` in a second migration)
  rather than a single destructive step — write it as such, don't try
  to cram this into one migration operation.

REQUIREMENTS — serializers/views
- Update whatever serializer currently exposes `product` on a cart item
  (find it via `grep -rn "CartItem\|class Cart" backend/cart/serializers.py`)
  to expose `variant` instead, including whatever nested
  product/color/price representation the frontend cart UI needs (check
  frontend/src for how the cart currently renders items — e.g.
  frontend/src/pages/Cart.jsx or a CartContext — to understand what
  shape of data it expects, and adjust the serializer to still supply
  everything the frontend needs, just nested one level deeper under
  `variant.product` instead of flat `product`).
- Update whatever view/endpoint accepts "add to cart" requests (find it
  via `grep -rn "CartItem" backend/cart/views.py`) to accept a
  `variant_id` in the request body instead of `product_id`.
- Update `backend/cart/tests/` (find the exact test file locations)
  accordingly — every existing cart test that references `.product` on
  a CartItem needs updating to `.variant`, and every test that creates
  a CartItem via a Product fixture needs to instead create/reference a
  ProductVariant fixture.

ACCEPTANCE CRITERIA / TESTS
- Full cart test suite passes with the new variant-based model.
- Add a NEW test proving two different variants of the SAME product can
  both exist as separate lines in one cart (the exact scenario the old
  `unique_together=("cart","product")` would have prevented).
- Add a test proving the SAME variant can't be added to a cart twice as
  two separate rows (quantity should increment on the existing row
  instead — verify whatever the existing "add to cart" endpoint's
  actual behavior is for duplicate adds today, and confirm it's
  preserved with variant instead of product).
```

---

#### Task 3.1.1.4 — Update `OrderItem` to snapshot variant SKU

```
You are working in backend/order/models.py and backend/order/serializers.py
(OrderCreateSerializer). Assume Task 3.1.1.3 (Cart/CartItem now
variant-based) is already merged.

CONTEXT
The current OrderItem model:

    class OrderItem(models.Model):
        order = models.ForeignKey(Order, on_delete=models.CASCADE, related_name="items")
        product = models.ForeignKey("shop.Product", on_delete=models.SET_NULL, null=True, related_name="order_items")
        product_name = models.CharField(max_length=255)
        product_slug = models.SlugField(max_length=255)
        product_image = models.URLField(blank=True)
        unit_price = models.DecimalField(max_digits=10, decimal_places=2, validators=[MinValueValidator(0)])
        quantity = models.PositiveIntegerField(validators=[MinValueValidator(1)])

freezes product-level data at order time, but has no way to record
WHICH variant (shade/size) was actually purchased — this loses
critical fulfillment information (a warehouse worker fulfilling
"Foundation x1" has no idea which of 20 shades to pick and ship).

TASK
Add variant-level fields to OrderItem, add a `variant` FK alongside the
existing `product` FK, and update order creation (in
`order/serializers.py`, built out across Epic 1 Feature 1.1.1) to
populate them from cart items' variants.

REQUIREMENTS
- Add to OrderItem:
  `variant = models.ForeignKey("shop.ProductVariant", on_delete=models.SET_NULL, null=True, related_name="order_items")`
  (same `SET_NULL` pattern as the existing `product` FK, for the same
  reason — an order must survive the deletion of the variant/product it
  references).
  `variant_sku = models.CharField(max_length=64, blank=True)` — frozen
  snapshot, same pattern as the existing `product_name`/`product_slug`
  snapshot fields.
  `variant_attributes_json = models.JSONField(default=dict, blank=True)`
  — frozen snapshot of variant-identifying attributes at order time
  (e.g. `{"color": "Shade 320 - Warm Beige"}`), so the order remains a
  fully self-contained record even if the variant is later deleted or
  its color changes.
- Keep the existing `product`/`product_name`/`product_slug`/
  `product_image` fields exactly as they are — this task is additive,
  not a replacement of product-level snapshotting.
- Generate the migration (nullable/blank-safe additive fields, no data
  migration needed here since this only affects NEW orders going
  forward — existing OrderItem rows simply have these new fields blank,
  which is fine and expected for historical pre-variant orders).
- In `order/serializers.py` `OrderCreateSerializer.create()` (built out
  in Epic 1 Feature 1.1.1 — locate the loop that creates OrderItem rows
  from cart items), update it to read from `cart_item.variant` instead
  of `cart_item.product` (following Task 3.1.1.3's CartItem change),
  and populate the new fields:
  `variant=cart_item.variant`,
  `variant_sku=cart_item.variant.sku`,
  `variant_attributes_json={"color": cart_item.variant.color.name if cart_item.variant.color else None}`
  — keep populating `product`/`product_name`/etc. from
  `cart_item.variant.product` (the parent product), unchanged in
  spirit, just accessed one level deeper through the variant now.

ACCEPTANCE CRITERIA / TESTS
- Migration applies cleanly.
- Update the order test suite: every test that creates an order via a
  cart needs its cart fixture updated to use variant-based CartItems
  (per Task 3.1.1.3), and new assertions added confirming the resulting
  OrderItem's `variant`, `variant_sku`, and `variant_attributes_json`
  are populated correctly from the cart item's variant.
- Add a specific test: ordering two different color variants of the
  same product results in two separate OrderItem rows with different
  `variant_sku`/`variant_attributes_json` values, both correctly linked
  back to the same underlying `product`.
```

---

#### Task 3.1.1.5 — Update stock-decrement logic for variants

```
You are working in backend/order/serializers.py
(OrderCreateSerializer.create()) and backend/order/views.py
(OrderDetailView.patch, order cancellation). Assume Task 3.1.1.4 is
already merged and that Epic 1 Feature 1.1.1 (atomic order creation,
select_for_update locking, stock decrement/restoration) is already
merged and currently operates on `Product.stock` directly.

CONTEXT — THIS IS THE MOST IMPORTANT TASK IN THIS EPIC
Epic 1 built real, tested, atomic stock-safety logic
(transaction.atomic + select_for_update + decrement-on-create +
restore-on-cancel), but it operates on `Product.stock`. Now that
`ProductVariant.stock` is the real, authoritative, per-shade/size
inventory count (per Tasks 3.1.1.1–3.1.1.4), that entire safety system
needs to be repointed at variants — otherwise the codebase will have
TWO different stock numbers (Product.stock, now stale/unused, and
ProductVariant.stock, the real one) and Epic 1's carefully-built race-
condition protection will be silently protecting the WRONG field,
leaving the real variant stock completely unprotected from overselling
again.

TASK
Repoint every piece of stock-locking/decrement/restoration logic built
in Epic 1 Feature 1.1.1 from `Product` to `ProductVariant`.

REQUIREMENTS
- In `OrderCreateSerializer.create()`:
  - Change the `Product.objects.select_for_update().filter(id__in=...)`
    query (from Epic 1 Task 1.1.1.2) to
    `ProductVariant.objects.select_for_update().filter(id__in=<variant ids from cart>)`,
    building the locked-instances dict keyed by VARIANT id instead of
    product id.
  - Change every reference to `locked_product` in the stock-check/
    decrement loop (from Epic 1 Task 1.1.1.3) to use the locked
    `ProductVariant` instance instead — the stock check
    (`if locked_variant.stock < cart_item.quantity: raise ...`) and the
    decrement (`locked_variant.stock -= cart_item.quantity;
    locked_variant.save(update_fields=["stock"])`) both now operate on
    the variant, not the product.
  - Update the error message in the stock-check ValidationError to
    include which SPECIFIC variant is short (e.g. include the color/SKU,
    not just the base product name, so the customer/support team knows
    exactly which shade is out of stock) — pull the display detail from
    `locked_variant.product.name` plus
    `locked_variant.color.name if locked_variant.color else locked_variant.sku`.
- In `OrderDetailView.patch()` (order cancellation, from Epic 1 Task
  1.1.1.4):
  - Change the stock-restoration loop to iterate
    `order.items.all()` and restore `order_item.variant.stock` (not
    `order_item.product.stock`) — but remember `order_item.variant` can
    be `None` (SET_NULL, same as `product`) if the variant was later
    deleted, so keep the existing graceful-skip behavior for that case,
    now checking `if order_item.variant is None: continue` instead of
    checking `order_item.product is None`.
  - Use `select_for_update()` on the `ProductVariant` being restored,
    same locking pattern as before, just retargeted.
- Do NOT remove the old `Product.stock` field yet in this task (that's
  a separate future cleanup decision — leaving it in place but unused
  avoids a bigger destructive migration bundled into this already-large
  task; flag this in a code comment: `# Product.stock is now
  superseded by ProductVariant.stock and unused in the order flow —
  candidate for removal in a future cleanup task`).

ACCEPTANCE CRITERIA / TESTS
- Re-run EVERY test from Epic 1 Feature 1.1.1 (atomic creation, locking,
  decrement, restoration, AND the concurrency test suite from Task
  1.1.1.5) — update their fixtures to use ProductVariant-based carts
  (per Task 3.1.1.3) and confirm every assertion still passes, now
  checking `ProductVariant.stock` values instead of `Product.stock`.
- Specifically re-verify the concurrency test (Task 1.1.1.5's
  `test_stock_concurrency.py`) end-to-end against variant stock — this
  is the single most important regression check in this whole epic:
  prove overselling is still impossible after the variant migration,
  don't just assume the repointing preserved the guarantee.
- Add one new test: two customers concurrently ordering the LAST unit
  of two DIFFERENT variants of the SAME product (e.g. the last unit of
  Shade A and the last unit of Shade B, both in stock=1) should BOTH
  succeed, since they're different variants and don't actually contend
  for the same inventory — proving the locking is correctly scoped per
  variant, not accidentally over-broad at the product level.
```

---

#### Task 3.1.1.6 — Admin UI for managing variants (Django admin)

```
You are working in backend/shop/admin.py. Assume Feature 3.1.1 Tasks
3.1.1.1 through 3.1.1.5 are already merged.

CONTEXT
The current ProductAdmin has inlines for `ProductImageInline` and
`ProductColorInline`:

    @admin.register(Product)
    class ProductAdmin(admin.ModelAdmin):
        list_display = (...)
        list_filter = ("category", "brand", "is_new", "is_sale")
        search_fields = ("name", "slug", "short_description")
        prepopulated_fields = {"slug": ("name",)}
        ordering = ("-created_at",)
        inlines = [ProductImageInline, ProductColorInline]
        readonly_fields = ("created_at",)
        list_per_page = 25

There's no way for store admins to view or manage `ProductVariant`
records at all yet — every variant currently exists only via the
Task 3.1.1.2 migration or direct DB access.

TASK
Add a `ProductVariantInline` to `ProductAdmin`, and register a
standalone `ProductVariantAdmin` for direct variant management/search.

REQUIREMENTS
- Add:
  ```python
  class ProductVariantInline(admin.TabularInline):
      model = ProductVariant
      extra = 1
      fields = ("sku", "barcode", "color", "price", "original_price", "stock", "is_active")
  ```
  and add `ProductVariantInline` to `ProductAdmin.inlines` (alongside
  the existing `ProductImageInline`, `ProductColorInline`).
- Register a standalone admin:
  ```python
  @admin.register(ProductVariant)
  class ProductVariantAdmin(admin.ModelAdmin):
      list_display = ("id", "product", "sku", "color", "price", "stock", "is_active")
      list_filter = ("is_active", "color")
      search_fields = ("sku", "barcode", "product__name")
      autocomplete_fields = ("product", "color")
      ordering = ("product__name", "id")
  ```
  — using `autocomplete_fields` for `product` since the product list
  will be large; confirm `ProductAdmin` has `search_fields` set (it
  does) since Django admin autocomplete requires the target model's
  ModelAdmin to define `search_fields` for autocomplete to work.
- Import `ProductVariant` into the existing `from .models import (...)`
  line at the top of admin.py alongside the other model imports.
- Since `ProductAdmin` now has THREE inlines
  (Image/Color/Variant), verify the admin page still renders
  sensibly (not requiring code changes necessarily, just a manual
  sanity check via `python manage.py runserver` and visiting the admin
  product change page — note in your task summary whether the layout
  is still usable or whether you'd recommend collapsing/reordering
  inlines, but don't over-engineer this beyond what's asked).

ACCEPTANCE CRITERIA / TESTS
- Manually verify in the Django admin: a Product's change page shows
  its variants inline and allows adding a new variant directly from the
  product page; the standalone Product Variant admin list page allows
  searching by SKU and filtering by active status.
- No automated test is strictly required for admin.py configuration
  changes (Django admin registration isn't typically unit-tested in
  this codebase — confirm by checking whether backend/shop/tests/
  contains any admin-specific tests before deciding whether to add one;
  if the project has no precedent for testing admin config, skip
  writing tests for this task rather than inventing a new testing
  pattern for admin-only changes).
```

---

### Feature 3.1.2 — SKU & Barcode Generation

---

#### Task 3.1.2.1 — Auto-generate SKU on variant save

```
You are working in backend/shop/models.py (ProductVariant.save()).
Assume Feature 3.1.1 is fully merged.

CONTEXT
`ProductVariant.sku` is currently a plain `CharField(max_length=64,
unique=True, blank=True)` with no auto-generation — admins creating a
new variant through the Django admin (Task 3.1.1.6) have to type a SKU
by hand every time, which is tedious and error-prone (typos causing
accidental uniqueness collisions, inconsistent formats across
products). The legacy migration (Task 3.1.1.2) used a placeholder
`LEGACY-{product.id}-{index}` pattern for existing data, but there's no
equivalent auto-generation for variants created going forward through
normal admin/API use.

TASK
Auto-generate a deterministic, readable SKU when a ProductVariant is
saved without one already set.

REQUIREMENTS
- Override `ProductVariant.save()`, mirroring the existing pattern
  already used on `Product.save()` and `Category.save()` in this same
  file (auto-slugify-if-blank pattern) for consistency:
  ```python
  def save(self, *args, **kwargs):
      if not self.sku:
          self.sku = self._generate_sku()
      super().save(*args, **kwargs)

  def _generate_sku(self) -> str:
      category_code = (
          self.product.category.name[:3].upper()
          if self.product.category else "GEN"
      )
      brand_code = (
          self.product.brand.name[:3].upper()
          if self.product.brand else "UNK"
      )
      base = f"{category_code}-{brand_code}-{self.product.id}"
      sku = base
      counter = 1
      while ProductVariant.objects.filter(sku=sku).exclude(pk=self.pk).exists():
          sku = f"{base}-{counter}"
          counter += 1
      return sku
  ```
  — adjust the exact format if you find a cleaner convention, but it
  must (a) be deterministic given the same product/category/brand
  inputs, (b) guarantee uniqueness via the same collision-loop pattern
  already used by `Product.save()`'s slug generation (look at that
  method for the established style before writing this one, and match
  it), and (c) require `self.product` to already be set/saved before
  this runs (a ProductVariant always has its `product` FK set before
  save since it's non-nullable, so this is safe).
- This must NOT overwrite an admin-provided SKU — only generate one
  when the field is blank, exactly like the existing slug-generation
  pattern.

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/shop/tests/test_models.py:
1. Creating a ProductVariant with no `sku` provided results in a
   non-blank, uniquely-generated SKU after save.
2. Creating a ProductVariant WITH an explicit `sku` provided preserves
   that exact value (no override).
3. Creating two variants for products with the same category+brand+id
   collision scenario (construct this deliberately) results in
   distinct SKUs (proving the collision-avoidance loop works,
   mirroring the equivalent existing test for `Product.slug` generation
   if one exists — check backend/shop/tests/test_models.py for that
   precedent test and mirror its structure for consistency).
```

---

#### Task 3.1.2.2 — Add barcode field with format validation (EAN-13)

```
You are working in backend/shop/models.py (ProductVariant). Assume
Task 3.1.2.1 is already merged.

CONTEXT
`ProductVariant.barcode` already exists as a plain
`CharField(max_length=20, blank=True)` with zero format validation
(added back in Task 3.1.1.1). Barcodes matter for cosmetics
specifically for warehouse/POS scanning integration and eventual
Iranian regulatory barcode requirements — an unvalidated free-text
field risks admins entering malformed values that silently fail to
scan later.

TASK
Add EAN-13 format validation to the existing `barcode` field, applied
only when a value is actually provided (the field remains optional).

REQUIREMENTS
- Create a validator function/RegexValidator, either inline in
  models.py or in the backend/accounts/validators.py-style pattern if
  this project has established a `validators.py` convention elsewhere
  (check whether Epic 2's Task 2.1.1.2 already created a project
  convention for standalone validator modules per-app — if so, create
  `backend/shop/validators.py` to match that convention rather than
  inlining validators directly in models.py).
- EAN-13 validation should check: (a) exactly 13 digits, and (b) the
  correct check-digit (the last digit is a checksum computed from the
  first 12 — implement the standard EAN-13 checksum algorithm: sum of
  digits at odd positions + 3× sum of digits at even positions,
  mod 10, and the check digit must make that total a multiple of 10).
  Don't just regex-match "13 digits" without validating the actual
  checksum — a barcode field that accepts any 13 digits regardless of
  checksum validity provides false confidence and won't actually catch
  common typos (transposed digits, etc.) that a real checksum would
  catch.
- Apply the validator via `validators=[validate_ean13]` on the
  `barcode` field. Since the field is `blank=True`, Django's validator
  execution only runs when a non-empty value is provided AND
  `full_clean()`/form validation is actually invoked — as established
  in Task 2.1.1.2's investigation, confirm whether this validation
  actually fires on bare `.save()` calls (likely NOT, per Django's
  normal behavior) vs. only through ModelForms/DRF serializers/explicit
  `full_clean()` calls, and make sure the admin form (Task 3.1.1.6's
  `ProductVariantInline`/`ProductVariantAdmin`, which uses standard
  Django ModelForms) DOES enforce it, since that's the primary place
  where malformed barcodes would currently be entered by hand.
- If a DRF-facing variant-creation API exists or will soon (check
  whether admin bulk import / variant CRUD API endpoints exist yet at
  this point in the project — if not, note in a comment that any future
  variant API serializer must also trigger this validation, since DRF
  serializers by default DO run model field validators during
  `is_valid()`, unlike bare `.save()`).

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/shop/tests/test_models.py (or test_validators.py
if you created a dedicated validators module):
1. A valid, correctly-checksummed 13-digit EAN-13 barcode passes
   `full_clean()`.
2. A 13-digit string with an incorrect checksum digit fails
   `full_clean()` with a validation error.
3. A string that isn't 13 digits (too short, too long, contains
   letters) fails `full_clean()`.
4. An empty/blank barcode passes `full_clean()` without error (the
   field remains genuinely optional — validation only applies when a
   value IS provided).
```

---

## Phase 3.2 — Cosmetics-Specific Attributes

*Note on sequencing: Tasks 3.2.1.1–3.2.1.13 are all small, independent, additive field additions to `Product` (or, where noted, `ProductVariant`) and can technically be done in any order or even batched into fewer migrations by a human team — they're kept as separate tasks here because each is independently reviewable and revertable, matching the backlog's "one task, one sitting" sizing rule. Tasks 3.2.1.14 and 3.2.1.15 MUST run after all field-addition tasks are complete, since they depend on every field existing.*

### Feature 3.2.1 — Product Attribute Fields

---

#### Task 3.2.1.1 — Add `skin_type` choice field

```
You are working in backend/shop/models.py (Product model). Assume all
of Phase 3.1 is merged.

CONTEXT
Product currently has zero cosmetics-specific attributes — it's a
generic e-commerce product model (name, price, stock, category, brand,
description). Skin type is one of the most commonly filtered attributes
on any real cosmetics/skincare storefront (Sephora, Ulta, etc. all
support this as a primary filter facet) and is currently entirely
absent.

TASK
Add a `skin_type` field to `Product` (not `ProductVariant` — skin-type
suitability is a property of the product formulation itself, not of a
specific shade/size, so it belongs at the product level, unlike
price/stock which genuinely vary per variant).

REQUIREMENTS
- Add:
  ```python
  class SkinType(models.TextChoices):
      OILY = "oily", "Oily"
      DRY = "dry", "Dry"
      COMBINATION = "combination", "Combination"
      SENSITIVE = "sensitive", "Sensitive"
      NORMAL = "normal", "Normal"
      ALL = "all", "All Skin Types"

  # on Product:
  skin_type = models.CharField(
      max_length=20, choices=SkinType.choices, blank=True,
  )
  ```
  — `blank=True` and no default, since not every product category
  (e.g. haircare, fragrance) has a meaningful skin type; leaving it
  blank must be valid, not force an arbitrary default like "all" onto
  products where the concept doesn't apply.
- Define `SkinType` as a module-level class in shop/models.py, near the
  top of the file (before `Product`), following the same
  `models.TextChoices`/`models.IntegerChoices` pattern already
  established in backend/accounts/models.py's `UserType` for
  consistency across the codebase.
- Generate the migration.
- Add `skin_type` as a read-only field on the Django admin's
  `ProductAdmin.list_filter` (add `"skin_type"` to the existing
  `list_filter = ("category", "brand", "is_new", "is_sale")` tuple) so
  admins can immediately filter the product list by skin type.

ACCEPTANCE CRITERIA / TESTS
- Migration applies cleanly.
- Add a model test confirming a Product can be saved with each valid
  `skin_type` choice, and that leaving it blank is valid (no
  full_clean error).
- Do NOT add this to `ProductFilter`/serializers yet — that's
  consolidated into Tasks 3.2.1.14 and 3.2.1.15 at the end of this
  Feature, to avoid touching the same serializer/filter files in 12
  separate tiny commits.
```

---

#### Task 3.2.1.2 — Add `hair_type` choice field

```
You are working in backend/shop/models.py (Product model). Assume Task
3.2.1.1 is merged (for the established TextChoices pattern to mirror).

CONTEXT
Same rationale as skin_type (Task 3.2.1.1) but for haircare products —
shampoo, conditioner, styling products are commonly filtered by hair
type on cosmetics storefronts.

TASK
Add a `hair_type` field to `Product`, following the exact same pattern
as `skin_type` from Task 3.2.1.1.

REQUIREMENTS
- Add:
  ```python
  class HairType(models.TextChoices):
      STRAIGHT = "straight", "Straight"
      WAVY = "wavy", "Wavy"
      CURLY = "curly", "Curly"
      COILY = "coily", "Coily"
      ALL = "all", "All Hair Types"

  # on Product:
  hair_type = models.CharField(
      max_length=20, choices=HairType.choices, blank=True,
  )
  ```
- Generate the migration (can be combined with Task 3.2.1.1's migration
  if you're running these back-to-back in one session and
  `makemigrations` batches them — either one combined migration or two
  separate sequential ones is fine, just don't manually hand-edit
  migration files to force a specific grouping; let Django's migration
  system do what it naturally does based on when you run
  `makemigrations`).
- Add `"hair_type"` to `ProductAdmin.list_filter`.

ACCEPTANCE CRITERIA / TESTS
Same pattern as Task 3.2.1.1: migration applies cleanly, model test
confirms valid choices save correctly and blank is valid.
```

---

#### Task 3.2.1.3 — Add `spf` integer field

```
You are working in backend/shop/models.py (Product model).

CONTEXT
SPF (Sun Protection Factor) is a critical, commonly-filtered attribute
for sunscreen and many daily-wear skincare/makeup products with
built-in sun protection, and is currently completely absent from the
schema.

TASK
Add an `spf` field to `Product`.

REQUIREMENTS
- Add:
  `spf = models.PositiveSmallIntegerField(null=True, blank=True)`
  — nullable (not just blank) since this is a numeric field where
  "0"/blank would be ambiguous (does 0 mean "no SPF" or "not
  specified"?); `null=True` lets the field genuinely be "not
  applicable" for products where SPF doesn't apply at all (e.g. most
  makeup remover, most haircare).
- Add a `validators=[MaxValueValidator(100)]` (import from
  `django.core.validators`, already used elsewhere in this codebase
  e.g. `MinValueValidator` in cart/order models) since real-world SPF
  ratings realistically top out around SPF 50-100 — this is a sanity
  bound, not a hard business rule, so pick a generous ceiling rather
  than trying to encode exact real-world SPF product-labeling
  regulations.
- Generate the migration.
- Add `spf` as a `list_display` column on `ProductAdmin` if it fits
  reasonably (the existing `list_display` tuple is already fairly long
  — use judgment on whether adding it there clutters the list view too
  much versus just leaving it available in the detail/change form; if
  in doubt, don't add it to list_display, just leave it in the
  standard admin change form via the default field rendering).

ACCEPTANCE CRITERIA / TESTS
Add a model test confirming valid SPF values (e.g. 30, 50) save
correctly, `None`/blank is valid, and a value above the configured max
validator bound fails `full_clean()`.
```

---

#### Task 3.2.1.4 — Add `volume_ml`/`weight_g` fields

```
You are working in backend/shop/models.py (ProductVariant model, NOT
Product). Assume Phase 3.1 is fully merged.

CONTEXT
Unlike skin_type/hair_type/SPF (product-formulation-level attributes,
correctly placed on `Product` in the preceding tasks), volume/weight
genuinely DIFFERS per purchasable unit — a serum sold in 30ml and 50ml
sizes needs each size tracked as its own variant with its own volume,
price, and stock. This field belongs on `ProductVariant`, not
`Product`, matching the backlog's explicit note: "On variant, not
product (differs per SKU)."

TASK
Add `volume_ml` and `weight_g` fields to `ProductVariant`.

REQUIREMENTS
- Add to `ProductVariant`:
  ```python
  volume_ml = models.PositiveIntegerField(null=True, blank=True)
  weight_g = models.PositiveIntegerField(null=True, blank=True)
  ```
  — both nullable/optional since a given variant is typically measured
  in EITHER volume (liquids: serums, toners, perfume) OR weight
  (solids/powders: pressed powder, lipstick, some skincare balms), not
  both, and plenty of variants (e.g. a color-only variant of an
  eyeshadow palette with no size variation) may need neither.
- Do NOT add any validation forcing "exactly one of volume_ml/weight_g
  must be set" — that's an unnecessary constraint that would break
  perfectly valid variants that have neither dimension recorded (yet,
  or ever); keep this permissive.
- Generate the migration.
- Add both fields to `ProductVariantInline.fields` and
  `ProductVariantAdmin.list_display` (from Task 3.1.1.6 — update that
  existing admin config to include the two new fields) so they're
  immediately editable/visible in the admin.

ACCEPTANCE CRITERIA / TESTS
Add a model test confirming a ProductVariant can be created with
`volume_ml` set and `weight_g` null (or vice versa, or both null), and
all combinations save without error.
```

---

#### Task 3.2.1.5 — Add `ingredients` long-text field (INCI list)

```
You are working in backend/shop/models.py (Product model).

CONTEXT
Cosmetics products are commonly required (and always expected by
informed shoppers) to disclose a full ingredients list, formatted as an
INCI (International Nomenclature of Cosmetic Ingredients) list — a
long, comma-separated technical ingredient list. This is entirely
absent today.

TASK
Add an `ingredients` field to `Product`.

REQUIREMENTS
- Add: `ingredients = models.TextField(blank=True)` — a plain long-text
  field is sufficient for now (a structured, individually-searchable
  ingredient list with allergen-flagging is a much bigger feature,
  explicitly out of scope here; this task just needs the raw INCI text
  to be storable and displayable).
- Generate the migration.
- Do NOT make this searchable via `ProductFilter`/full-text search in
  this task — that's out of scope (tracked separately under the
  project's Search epic if/when ingredient-based search becomes a
  requirement). This task is storage only.

ACCEPTANCE CRITERIA / TESTS
Add a minimal model test confirming a Product can store and retrieve a
long ingredients string (e.g. a realistic multi-hundred-character INCI
list) without truncation.
```

---

#### Task 3.2.1.6 — Add `country_of_origin` field

```
You are working in backend/shop/models.py (Product model).

CONTEXT
Country of origin is a commonly-displayed and sometimes
regulatorily-relevant attribute for imported cosmetics in the Iranian
market context this platform targets, and is currently absent.

TASK
Add a `country_of_origin` field to `Product`.

REQUIREMENTS
- Add: `country_of_origin = models.CharField(max_length=100, blank=True)`
  — a plain CharField with a curated choices list is tempting, but
  cosmetics products are sourced from a very wide range of countries
  and a hardcoded choices list would need constant maintenance; use
  free text for now rather than over-engineering a full ISO
  country-code dropdown in this task (that's a reasonable future
  enhancement, but out of scope here — note this decision in a code
  comment).
- Generate the migration.
- Add `"country_of_origin"` to `ProductAdmin.search_fields` (extend the
  existing `search_fields = ("name", "slug", "short_description")`
  tuple) so admins can search products by origin country.

ACCEPTANCE CRITERIA / TESTS
Minimal model test confirming the field saves/retrieves correctly and
is searchable in the admin (the search_fields addition can be verified
via a quick Django admin manual check rather than requiring a dedicated
automated test, consistent with how this project doesn't appear to
unit-test admin search_fields configuration elsewhere).
```

---

#### Task 3.2.1.7 — Add `expiration_date`/`manufacture_date` fields

```
You are working in backend/shop/models.py (ProductVariant model, NOT
Product). Assume Phase 3.1 is fully merged.

CONTEXT
Per the backlog: expiration/manufacture dates are batch-specific, and a
single Product can have multiple variants (and, in a fuller
implementation, multiple batches per variant) with different
expiration dates — this belongs on `ProductVariant`, matching the
volume_ml/weight_g placement decision from Task 3.2.1.4. This is
functionally important for cosmetics specifically (shelf-life,
regulatory compliance, return/recall handling) in a way it wouldn't be
for most other e-commerce categories.

TASK
Add `expiration_date` and `manufacture_date` fields to `ProductVariant`.

REQUIREMENTS
- Add to `ProductVariant`:
  ```python
  manufacture_date = models.DateField(null=True, blank=True)
  expiration_date = models.DateField(null=True, blank=True)
  ```
  — both nullable/optional, since not every product category has
  meaningful expiration data (though most cosmetics/skincare will).
- Add a `clean()` method override on `ProductVariant` validating that
  `expiration_date`, if set, is after `manufacture_date`, if also set
  (skip the check entirely if either is None) — raise
  `ValidationError({"expiration_date": "Expiration date must be after manufacture date."})`.
  Note per the established pattern from earlier tasks (Task 2.1.1.2's
  investigation) that `clean()` only runs via `full_clean()` /
  ModelForm validation, not bare `.save()` — that's fine here since the
  primary entry point for setting these dates is the admin form (Task
  3.1.1.6), which DOES run full clean via Django's ModelForm machinery.
- Generate the migration.
- Add both fields to `ProductVariantInline.fields` and
  `ProductVariantAdmin.list_display`/`list_filter` (add
  `expiration_date` to `list_filter` so admins can filter/sort by it —
  useful groundwork for Task 3.3.1.1's near-expiry report).

ACCEPTANCE CRITERIA / TESTS
Add model tests:
1. A variant with `expiration_date` after `manufacture_date` passes
   `full_clean()`.
2. A variant with `expiration_date` BEFORE `manufacture_date` fails
   `full_clean()` with the expected error.
3. A variant with only one of the two dates set (or neither) passes
   `full_clean()` without triggering the comparison check.
```

---

#### Task 3.2.1.8 — Add `batch_number` field

```
You are working in backend/shop/models.py (ProductVariant model).
Assume Task 3.2.1.7 is merged.

CONTEXT
Batch/lot number tracking is standard cosmetics industry practice for
recall management and quality control, and pairs naturally with the
expiration/manufacture dates just added.

TASK
Add a `batch_number` field to `ProductVariant`.

REQUIREMENTS
- Add: `batch_number = models.CharField(max_length=50, blank=True)`
  — plain optional text field; real batch number formats vary
  significantly by manufacturer, so free text is appropriate (no
  format validation needed, unlike the SKU/barcode fields which have
  this platform's own generation/checksum logic).
- Generate the migration.
- Add `batch_number` to `ProductVariantInline.fields` and
  `ProductVariantAdmin.search_fields` (create this attribute on
  `ProductVariantAdmin` if it doesn't already have one from Task
  3.1.1.6 — check that task's admin config first) so admins can look up
  a variant by batch number, e.g. during a recall.

ACCEPTANCE CRITERIA / TESTS
Minimal model test confirming the field saves/retrieves correctly.
```

---

#### Task 3.2.1.9 — Add `usage_instructions` and `warnings` text fields

```
You are working in backend/shop/models.py (Product model).

CONTEXT
Cosmetics products commonly need clear usage instructions (how/when to
apply) and safety warnings (patch-test recommendations, discontinue-use
conditions, allergen callouts) displayed to the customer — both
entirely absent from the current schema.

TASK
Add `usage_instructions` and `warnings` fields to `Product`.

REQUIREMENTS
- Add:
  ```python
  usage_instructions = models.TextField(blank=True)
  warnings = models.TextField(blank=True)
  ```
  — two separate TextFields (not one combined field), since these are
  semantically distinct and will likely be displayed in visually
  distinct sections on the product detail page (usage instructions as
  routine informational content, warnings often styled with more
  visual prominence/urgency — e.g. an icon or colored callout box).
- Generate the migration.

ACCEPTANCE CRITERIA / TESTS
Minimal model test confirming both fields save/retrieve independently
without interfering with each other.
```

---

#### Task 3.2.1.10 — Add `gender` choice field

```
You are working in backend/shop/models.py (Product model). Assume the
TextChoices pattern from Task 3.2.1.1 is established.

CONTEXT
Many cosmetics/personal-care products are marketed toward or formulated
for a specific gender demographic (or explicitly unisex), and this is
a common storefront filter facet, currently absent.

TASK
Add a `gender` field to `Product`.

REQUIREMENTS
- Add:
  ```python
  class ProductGender(models.TextChoices):
      UNISEX = "unisex", "Unisex"
      FEMALE = "female", "Female"
      MALE = "male", "Male"

  # on Product:
  gender = models.CharField(
      max_length=10, choices=ProductGender.choices, default=ProductGender.UNISEX,
  )
  ```
  — unlike skin_type/hair_type (which default to blank since they
  often don't apply), `gender` defaults to `UNISEX` rather than blank,
  since every product realistically has SOME applicable answer here
  (even if it's "unisex") and defaulting to the most inclusive option
  avoids accidentally mis-filtering products that haven't been
  explicitly tagged yet.
- Generate the migration — note this migration needs a `default` value
  supplied for existing rows since the field isn't nullable/blank
  (Django's `makemigrations` will prompt for this interactively when
  you add a non-nullable field with a Python-level default to a model
  with existing rows; confirm it correctly applies `ProductGender.UNISEX`
  as the migration-time default for all pre-existing Product rows).
- Add `"gender"` to `ProductAdmin.list_filter`.

ACCEPTANCE CRITERIA / TESTS
Add a model test confirming the default value is applied when
`gender` isn't explicitly set, and all three choices save correctly.
```

---

#### Task 3.2.1.11 — Add `is_cruelty_free`/`is_vegan`/`is_organic` booleans

```
You are working in backend/shop/models.py (Product model).

CONTEXT
Cruelty-free, vegan, and organic certifications are increasingly
important purchase-decision filters for cosmetics shoppers and are
commonly surfaced as prominent badges/filters on modern cosmetics
storefronts (this is explicitly called out in the project's review as
a suitability gap). None of these flags currently exist.

TASK
Add three boolean flags to `Product`.

REQUIREMENTS
- Add:
  ```python
  is_cruelty_free = models.BooleanField(default=False)
  is_vegan = models.BooleanField(default=False)
  is_organic = models.BooleanField(default=False)
  ```
  — all default `False` (unverified/unclaimed by default; a product
  should only show these badges once genuinely confirmed by an admin,
  not opt-in-by-default).
- Generate the migration.
- Add all three to `ProductAdmin.list_filter` (extend the existing
  tuple) so admins can filter the product list by any combination of
  these flags.

ACCEPTANCE CRITERIA / TESTS
Add a model test confirming all three flags default to `False` on a
freshly created Product and can each be independently toggled to
`True`.
```

---

#### Task 3.2.1.12 — Add `irc_regulatory_code` field

```
You are working in backend/shop/models.py (Product model).

CONTEXT
Per the project's Iranian-market-suitability review: many cosmetics
require an IRC (Iran Food & Drug Administration) registration code for
legal retail sale in Iran. This is a compliance-relevant field, not
just a nice-to-have display attribute — selling a cosmetics product in
Iran without a valid, verified registration code where one is legally
required is a real regulatory risk for the business this platform is
being built for.

TASK
Add an `irc_regulatory_code` field to `Product`, laying groundwork for
Task 3.2.1.13's admin verification workflow.

REQUIREMENTS
- Add: `irc_regulatory_code = models.CharField(max_length=50, blank=True)`
  — plain optional text field in this task (format validation isn't
  specified by the backlog and Iranian IRC code formats aren't
  something to guess at without authoritative documentation — leave
  format-checking out of scope here rather than inventing an incorrect
  regex).
- Generate the migration.
- Add `"irc_regulatory_code"` to `ProductAdmin.search_fields`.

ACCEPTANCE CRITERIA / TESTS
Minimal model test confirming the field saves/retrieves correctly and
is genuinely optional (blank Product instances remain valid).
```

---

#### Task 3.2.1.13 — Admin verification workflow for regulatory code

```
You are working in backend/shop/models.py and backend/shop/admin.py.
Assume Task 3.2.1.12 (irc_regulatory_code field) is already merged.

CONTEXT
Having the code field alone isn't enough — the backlog specifically
calls for an admin VERIFICATION workflow: a way to flag that a given
code has actually been checked/confirmed valid by a staff member (not
just that someone typed something into the field), plus the ability to
filter the product list by verification status, and — per the backlog
— optionally prevent publishing a product without a verified code
(configurable, not a hard requirement).

TASK
Add a `regulatory_verified` boolean to `Product`, an admin action to
mark products as verified, a filterable admin column, and an optional
configurable publish-guard.

REQUIREMENTS
- Add: `regulatory_verified = models.BooleanField(default=False)` to
  `Product`. Generate the migration.
- Add a Django admin action to `ProductAdmin`:
  ```python
  @admin.action(description="Mark selected products as regulatory-verified")
  def mark_regulatory_verified(modeladmin, request, queryset):
      updated = queryset.update(regulatory_verified=True)
      modeladmin.message_user(request, f"{updated} product(s) marked as verified.")

  # in ProductAdmin:
  actions = [mark_regulatory_verified]
  ```
  (adjust the exact action-registration syntax to match whatever
  Django version this project uses — check `Django==5.2.11` in
  requirements.txt and use the modern `@admin.action` decorator syntax,
  which is supported from Django 3.2+, so this is fine).
- Add `"regulatory_verified"` to `ProductAdmin.list_display` and
  `list_filter` so admins can see and filter verification status
  directly from the product list.
- Add a settings-driven, OFF-by-default configurable guard: introduce
  `REQUIRE_REGULATORY_VERIFICATION = config("REQUIRE_REGULATORY_VERIFICATION", default=False, cast=bool)`
  in backend/core/settings/base.py (matching the existing
  python-decouple config() pattern), and — ONLY if this setting is
  True — add validation somewhere in the product-publishing path (the
  most sensible place is a `clean()` override on `Product`, checking
  `if settings.REQUIRE_REGULATORY_VERIFICATION and self.irc_regulatory_code and not self.regulatory_verified: raise ValidationError(...)`
  — note this only blocks products that HAVE a code entered but aren't
  yet verified; a product with NO code entered at all is a separate
  business decision about whether IRC codes are mandatory for ALL
  products vs. only claimed-but-unverified ones, and the backlog says
  "optional (configurable)", so keep this guard narrowly scoped to the
  has-code-but-unverified case rather than making IRC codes mandatory
  for every product, which isn't what was asked).
  Import `from django.conf import settings` at the top of models.py if
  not already imported.

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/shop/tests/test_models.py:
1. With `REQUIRE_REGULATORY_VERIFICATION=False` (the default), a
   Product with a code but `regulatory_verified=False` still passes
   `full_clean()`.
2. With `REQUIRE_REGULATORY_VERIFICATION=True` (use
   `@override_settings` for this specific test), a Product with a code
   but `regulatory_verified=False` FAILS `full_clean()`, and the same
   Product with `regulatory_verified=True` passes.
3. With `REQUIRE_REGULATORY_VERIFICATION=True`, a Product with NO code
   at all still passes `full_clean()` (proving the guard doesn't make
   codes universally mandatory).
Manually verify the admin action works: selecting multiple products in
the admin list and running "Mark selected products as
regulatory-verified" sets the flag on all of them and shows the
confirmation message.
```

---

#### Task 3.2.1.14 — Serializer updates to expose new attributes

```
You are working in backend/shop/serializers.py. Assume ALL of Tasks
3.2.1.1 through 3.2.1.13 are already merged (every new field exists on
Product/ProductVariant).

CONTEXT
`ProductDetailSerializer` and `ProductListSerializer` currently expose
only the original, pre-cosmetics field set (name, price, stock, rating,
category, brand, colors, images, reviews, etc.) — none of the 13 new
attribute fields added across this Feature are visible via the API
yet, meaning the frontend has no way to display or filter on any of
them despite the data now existing in the database.

TASK
Add all new Product fields to the appropriate serializers, and add a
new `ProductVariantSerializer` exposing the new ProductVariant fields
(volume_ml, weight_g, expiration_date, manufacture_date, batch_number,
sku, barcode, color, price, stock) — replacing/supplementing the
existing `colors`/`ProductColorSerializer` nested representation on
`ProductDetailSerializer` with real variant data, since variants are
now the authoritative purchasable-unit representation.

REQUIREMENTS
- Create:
  ```python
  class ProductVariantSerializer(serializers.ModelSerializer):
      color = ColorSerializer(read_only=True)

      class Meta:
          model = ProductVariant
          fields = (
              "id", "sku", "barcode", "color", "price", "original_price",
              "stock", "volume_ml", "weight_g", "manufacture_date",
              "expiration_date", "batch_number", "is_active",
          )
  ```
  Import `ProductVariant` into the existing
  `from .models import (...)` line at the top of serializers.py.
- On `ProductListSerializer`: decide thoughtfully which new fields
  belong on the LIGHTWEIGHT list view — list serializers exist
  specifically to keep payload size small for grid/browse pages, so
  don't blindly add all 13 new fields here. Add the ones genuinely
  useful for browse/filter display and badges:
  `skin_type`, `hair_type`, `gender`, `is_cruelty_free`, `is_vegan`,
  `is_organic` (these drive badges/filter chips shown directly on
  product cards). Do NOT add `ingredients`, `usage_instructions`,
  `warnings`, `country_of_origin`, `irc_regulatory_code`,
  `regulatory_verified` to the LIST serializer — these are detail-page-
  only fields with no browse-page use case and would bloat every list
  response unnecessarily.
- On `ProductDetailSerializer`: add ALL new Product fields (`skin_type`,
  `hair_type`, `spf`, `ingredients`, `country_of_origin`,
  `usage_instructions`, `warnings`, `gender`, `is_cruelty_free`,
  `is_vegan`, `is_organic`, `irc_regulatory_code`,
  `regulatory_verified`) — the detail page is exactly where a customer
  needs the full ingredient list, usage instructions, and warnings, and
  where regulatory transparency matters most.
- On `ProductDetailSerializer`: add
  `variants = ProductVariantSerializer(many=True, read_only=True)` —
  keep the existing `colors = ProductColorSerializer(many=True, read_only=True)`
  field in place too for now rather than deleting it outright (removing
  it is a frontend-breaking API change that deserves its own deliberate
  task/coordination with frontend work, not a silent side effect of
  this serializer task — flag this clearly: `# TODO: colors field is
  superseded by variants and should be removed once frontend fully
  migrates to variant-based rendering`).
- Update the existing `ProductAdmin`/wherever else new fields might
  need admin exposure — you already handled admin display in each
  individual field task (3.2.1.1 through 3.2.1.13); this task is
  serializer-only, don't duplicate that work.

ACCEPTANCE CRITERIA / TESTS
Add/update tests in backend/shop/tests/test_serializers.py:
1. `ProductListSerializer` output includes the 6 browse-relevant new
   fields and does NOT include the 6 detail-only fields.
2. `ProductDetailSerializer` output includes ALL new Product fields
   AND a `variants` array with each variant's new attribute fields
   correctly nested and serialized.
3. Re-run the full shop test suite (test_views.py, test_filters.py) to
   confirm no existing consumer of these serializers breaks from the
   additive field changes (additive fields shouldn't break anything,
   but confirm rather than assume, especially around the `colors` vs
   `variants` coexistence).
```

---

#### Task 3.2.1.15 — Update `ProductFilter` for new filterable attributes

```
You are working in backend/shop/filters.py. Assume Task 3.2.1.14 is
already merged (fields are now visible via the API).

CONTEXT
The current `ProductFilter` supports min_price/max_price, category,
brand, color, is_new, is_sale — none of the new cosmetics attributes
(skin_type, hair_type, spf, gender, cruelty-free/vegan/organic flags)
are filterable via the API yet, meaning the frontend filter sidebar
(planned separately under the project's Search epic, Task 12.2.1.1) has
nothing to actually query against for these facets today.

TASK
Extend `ProductFilter` to support filtering by the new cosmetics
attributes.

REQUIREMENTS
- Add to `ProductFilter`:
  ```python
  skin_type = django_filters.CharFilter(field_name="skin_type")
  hair_type = django_filters.CharFilter(field_name="hair_type")
  gender = django_filters.CharFilter(field_name="gender")
  min_spf = django_filters.NumberFilter(field_name="spf", lookup_expr="gte")
  max_spf = django_filters.NumberFilter(field_name="spf", lookup_expr="lte")
  is_cruelty_free = django_filters.BooleanFilter(field_name="is_cruelty_free")
  is_vegan = django_filters.BooleanFilter(field_name="is_vegan")
  is_organic = django_filters.BooleanFilter(field_name="is_organic")
  ```
  — mirror the exact style already used for `min_price`/`max_price`/
  `is_new`/`is_sale` in the existing file for consistency (simple
  direct field filters, not the more complex `method=` pattern used
  for `category`/`brand`/`color` which need comma-separated multi-value
  parsing — skin_type/hair_type/gender are single-value choice fields
  and don't need that added complexity).
  Add all new field names to `Meta.fields` (extend the existing list).
- Consider whether `skin_type`/`hair_type` should support
  comma-separated multi-select like `brand`/`color` do (e.g.
  `?skin_type=oily,combination` to show products suitable for either) —
  this is a genuinely reasonable UX improvement given a product might
  reasonably want to show up under multiple skin type searches, but
  the current model only stores ONE skin_type value per product
  (single CharField, not a multi-select field), so multi-VALUE
  filtering (matching against several selected skin types at once via
  `__in`) is still very possible even with a single-value field — add
  `skin_type = django_filters.CharFilter(method="filter_skin_type")`
  using the same comma-split-then-`__in` pattern as the existing
  `filter_brand`/`filter_color` methods, rather than the plain
  single-value filter suggested above. Do the same for `hair_type`.
  Keep `gender` as a simple single-value filter (rarely useful to
  select multiple genders at once, but adjust if you disagree — use
  judgment, this isn't a hard requirement either way).

ACCEPTANCE CRITERIA / TESTS
Add tests to backend/shop/tests/test_filters.py (mirroring the existing
test structure/fixtures used for the current filter tests):
1. Filtering by `?skin_type=oily` returns only products with
   `skin_type="oily"`.
2. Filtering by `?skin_type=oily,dry` (comma-separated) returns
   products matching EITHER value.
3. Filtering by `?min_spf=30` returns only products with `spf >= 30`,
   correctly excluding products with `spf=None` (confirm Django's
   `__gte` lookup correctly excludes NULL values by default — it
   should, but verify with an explicit test rather than assuming).
4. Filtering by `?is_vegan=true` returns only vegan-flagged products.
5. Combining multiple new filters simultaneously (e.g.
   `?skin_type=oily&is_vegan=true&min_spf=30`) correctly ANDs them
   together (standard django-filter behavior, but confirm with a real
   combined-filter test since this is the realistic way a filter
   sidebar would actually query).
```

---

## Phase 3.3 — Expiration & Batch Tracking

### Feature 3.3.1 — Expiry Management

---

#### Task 3.3.1.1 — Add "near expiry" admin filter/report

```
You are working in backend/shop/admin.py. Assume Task 3.2.1.7
(expiration_date on ProductVariant) is already merged.

CONTEXT
`expiration_date` exists on `ProductVariant` and is already in
`ProductVariantAdmin.list_filter` (added in Task 3.2.1.7), but Django's
default date filter widget only offers broad buckets (today, past 7
days, this month, this year) — not a specific, business-relevant
"expiring within the next 90 days" view that inventory staff would
actually want to check regularly.

TASK
Add a custom Django admin `SimpleListFilter` offering a "Near Expiry
(90 days)" quick-filter option on `ProductVariantAdmin`.

REQUIREMENTS
- Implement:
  ```python
  from django.utils import timezone
  from datetime import timedelta

  class NearExpiryFilter(admin.SimpleListFilter):
      title = "expiry status"
      parameter_name = "expiry_status"

      def lookups(self, request, model_admin):
          return (
              ("near", "Expiring within 90 days"),
              ("expired", "Already expired"),
          )

      def queryset(self, request, queryset):
          today = timezone.now().date()
          if self.value() == "near":
              return queryset.filter(
                  expiration_date__isnull=False,
                  expiration_date__gte=today,
                  expiration_date__lte=today + timedelta(days=90),
              )
          if self.value() == "expired":
              return queryset.filter(
                  expiration_date__isnull=False,
                  expiration_date__lt=today,
              )
          return queryset
  ```
- Add `NearExpiryFilter` to `ProductVariantAdmin.list_filter` (alongside
  the existing plain `expiration_date` filter — keep both, since the
  plain date filter is still useful for browsing by specific date
  ranges, while this new filter offers the specific business-relevant
  quick views).
- Sort `ProductVariantAdmin`'s default ordering to surface the most
  urgent items first when this filter is active — this is optional
  polish; if you do it, use `get_ordering()` override checking
  `request.GET.get("expiry_status")`, but don't over-engineer this if
  it adds meaningful complexity — a static ordering by
  `expiration_date` ascending is a reasonable default regardless of
  which filter is active, so consider just setting
  `ProductVariantAdmin.ordering = ("expiration_date", "product__name")`
  more simply instead.

ACCEPTANCE CRITERIA / TESTS
Add a test (Django admin filter classes can be tested directly by
calling `.queryset()` with a constructed request and pre-built test
data) confirming:
1. A variant expiring in 30 days is included when `expiry_status=near`.
2. A variant expiring in 200 days is EXCLUDED when `expiry_status=near`.
3. A variant with `expiration_date` in the past is included when
   `expiry_status=expired` and excluded from `expiry_status=near`.
4. A variant with `expiration_date=None` is excluded from BOTH filter
   options (shouldn't show up as "near" or "expired" if it has no date
   at all).
```

---

#### Task 3.3.1.2 — Prevent checkout of expired variants

```
You are working in backend/cart/ (add-to-cart endpoint) and
backend/order/serializers.py (OrderCreateSerializer). Assume Task
3.2.1.7 (expiration_date) and Epic 3 Feature 3.1.1 (variant-based cart/
order, including Task 3.1.1.5's variant-based stock locking) are
already merged.

CONTEXT
Nothing currently stops a customer from adding an already-expired
variant to their cart, or from completing checkout with an expired
variant in their cart — a real gap for a cosmetics business where
selling expired product is both a customer-trust and potential
regulatory problem.

TASK
Add expiration validation at TWO points: when adding a variant to a
cart, and again (defense in depth) at checkout submission — mirroring
the existing "re-validate stock at checkout, don't just trust the cart"
principle already established elsewhere in this codebase's order flow.

REQUIREMENTS
- In the cart app's "add to cart" view/serializer (the same location
  updated in Task 3.1.1.3 to accept `variant_id`), add a validation
  check: if the requested variant's `expiration_date` is set AND is in
  the past (`expiration_date < timezone.now().date()`), reject the
  request with a 400 and a clear message, e.g.
  `{"variant": "This product has expired and is no longer available for purchase."}`.
  A variant with `expiration_date=None` is unaffected (no expiration
  data means no expiration-based restriction — don't treat missing data
  as "expired").
- In `OrderCreateSerializer.create()` (inside the same
  `transaction.atomic()` block from Epic 1/Task 3.1.1.5, alongside the
  existing stock-check-before-decrement loop), add the identical
  expiration check for every cart item's variant BEFORE creating any
  OrderItem or decrementing any stock — if ANY item in the cart is
  expired at the moment of checkout (e.g. it was valid when added to
  the cart days ago but has since expired), the entire order must fail
  with a clear error identifying which item, and (per the existing
  atomic-block guarantee) nothing should be partially committed.
- Also consider: should an admin be able to sell expired stock
  deliberately (e.g. a clearance/close-out sale with clear customer
  disclosure)? The backlog doesn't call for this exception, so do NOT
  build an override mechanism — keep the block absolute for now, but
  leave a comment noting this as a possible future business decision
  if it comes up, rather than silently deciding either way without
  documenting the choice.

ACCEPTANCE CRITERIA / TESTS
Add tests:
1. Adding an expired variant to a cart returns 400 and does not create
   a CartItem.
2. Adding a non-expired (or no-expiration-date) variant succeeds
   normally (regression check).
3. A cart item that was valid when added but whose variant has since
   expired (simulate by creating the CartItem first, then updating the
   variant's `expiration_date` into the past) causes checkout to fail
   with a 400 identifying the expired item, and — critically — confirm
   NO Order/OrderItem rows exist afterward and NO other (non-expired)
   item's stock in the same cart was decremented (proving the
   all-or-nothing atomic behavior holds for this new check too, not
   just for the pre-existing stock check).
```

---

#### Task 3.3.1.3 — Celery task: nightly expiry sweep

```
You are working in backend/shop/tasks.py (new file). Assume Task
3.3.1.2 is merged, AND assume Celery infrastructure already exists in
this project (per the project's Epic 22, "Celery & Async Tasks" —
CONFIRM this is actually true before starting: check for
backend/core/celery.py and a configured `CELERY_BROKER_URL` in
settings; if Celery is NOT yet set up in this codebase at the time you
do this task, STOP and flag that this task has an unmet dependency on
Epic 22 Phase 22.1 rather than trying to bootstrap Celery from scratch
as a side effect of this task — that infrastructure work belongs in
its own dedicated task, not bundled invisibly into this one).

CONTEXT
Task 3.3.1.2 prevents NEW purchases of expired variants at the moment
of add-to-cart/checkout, but doesn't proactively deactivate expired
stock from browse/search/listing pages — an expired variant with
`is_active=True` would still appear in product listings and be visible
on the product detail page (just not purchasable), which is confusing
UX and looks like a bug rather than intentional behavior.

TASK
Create a scheduled Celery task that runs nightly, finds all
`ProductVariant` rows whose `expiration_date` has passed and are still
`is_active=True`, and deactivates them.

REQUIREMENTS
- Create `backend/shop/tasks.py`:
  ```python
  from celery import shared_task
  from django.utils import timezone
  from .models import ProductVariant

  @shared_task
  def deactivate_expired_variants():
      today = timezone.now().date()
      updated = ProductVariant.objects.filter(
          expiration_date__isnull=False,
          expiration_date__lt=today,
          is_active=True,
      ).update(is_active=False)
      return f"Deactivated {updated} expired variant(s)."
  ```
- Register this as a periodic task using whatever scheduling mechanism
  Epic 22's Celery setup established (`django-celery-beat`'s
  `CrontabSchedule`/`PeriodicTask` via the admin, or a
  `CELERY_BEAT_SCHEDULE` dict in settings — check which pattern the
  existing Celery infrastructure uses and match it exactly rather than
  introducing a second, inconsistent scheduling mechanism), scheduled
  to run once nightly (e.g. 2 AM).
- Since `.update()` is a bulk queryset operation, it does NOT trigger
  `save()` or any model signals — confirm this is acceptable here
  (deactivation doesn't need to trigger any side effects like
  notifications at this point in the project; if a future task adds a
  "notify admin when stock is auto-deactivated" requirement, that would
  need a different implementation using individual `.save()` calls or
  an explicit signal dispatch, but that's out of scope for this task).

ACCEPTANCE CRITERIA / TESTS
Add a test in backend/shop/tests/ (a new test_tasks.py, or added to
test_models.py — match whatever convention the project's Celery task
tests elsewhere use, if any exist yet from Epic 22 work) that:
1. Calls `deactivate_expired_variants()` directly (Celery tasks
   decorated with `@shared_task` can be called as plain Python
   functions in tests via `.run()` or by calling them directly without
   `.delay()`, avoiding the need for a real broker in tests — use
   whichever invocation style the project's existing Celery task tests
   (if any) already establish).
2. A variant with `expiration_date` in the past and `is_active=True`
   is deactivated (`is_active=False`) after the task runs.
3. A variant with `expiration_date` in the future is left untouched.
4. A variant with `expiration_date=None` is left untouched (never
   auto-deactivated on expiry grounds, since it has no expiry data).
5. A variant already `is_active=False` before the task runs is
   unaffected (no error, no redundant write — the `.filter(is_active=True)`
   clause already excludes it from the update, but confirm this with an
   explicit test).
```

---

## Summary checklist

| Task | Feature | Status |
|---|---|---|
| 3.1.1.1 | Design ProductVariant model | ☐ |
| 3.1.1.2 | Migrate existing stock/price to default variant | ☐ |
| 3.1.1.3 | Update Cart/CartItem to reference variant | ☐ |
| 3.1.1.4 | Update OrderItem to snapshot variant SKU | ☐ |
| 3.1.1.5 | Update stock-decrement logic for variants | ☐ |
| 3.1.1.6 | Admin UI for managing variants | ☐ |
| 3.1.2.1 | Auto-generate SKU on variant save | ☐ |
| 3.1.2.2 | Barcode field with EAN-13 validation | ☐ |
| 3.2.1.1 | Add skin_type field | ☐ |
| 3.2.1.2 | Add hair_type field | ☐ |
| 3.2.1.3 | Add spf field | ☐ |
| 3.2.1.4 | Add volume_ml/weight_g fields | ☐ |
| 3.2.1.5 | Add ingredients field | ☐ |
| 3.2.1.6 | Add country_of_origin field | ☐ |
| 3.2.1.7 | Add expiration_date/manufacture_date fields | ☐ |
| 3.2.1.8 | Add batch_number field | ☐ |
| 3.2.1.9 | Add usage_instructions/warnings fields | ☐ |
| 3.2.1.10 | Add gender field | ☐ |
| 3.2.1.11 | Add is_cruelty_free/is_vegan/is_organic flags | ☐ |
| 3.2.1.12 | Add irc_regulatory_code field | ☐ |
| 3.2.1.13 | Admin verification workflow for regulatory code | ☐ |
| 3.2.1.14 | Serializer updates for new attributes | ☐ |
| 3.2.1.15 | ProductFilter updates for new attributes | ☐ |
| 3.3.1.1 | Near-expiry admin filter/report | ☐ |
| 3.3.1.2 | Prevent checkout of expired variants | ☐ |
| 3.3.1.3 | Nightly Celery expiry sweep | ☐ |

Once Epic 3 is fully merged, the next epic to generate prompts for is
**Epic 4 — Inventory Management**, which builds `StockMovement` audit
logging and low-stock/back-in-stock features directly on top of the
`ProductVariant.stock` field this epic just established as the single
source of truth for inventory.
