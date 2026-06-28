# Bundle component stock: storefront integration

This guide shows how to read your bundles' component data on the storefront and build your own behavior around it, for example warning a shopper when the combined demand for a shared component across several bundles (or a bundle plus the same item bought on its own) is more than the available stock.

The app exposes each bundle's components on the bundle's product variant, so your theme can read the component variants, their per bundle quantity, and their live inventory, then decide what to do.

## Prerequisites

1. **Component references enabled for your store.** Ask your Bundles app contact to enable component references for your shop. Once enabled, the app writes a metafield called `bundles_app.components` on every bundle variant.
2. **Component products published to the Online Store sales channel.** The storefront can only read a component's inventory if its product is available to the Online Store. If a component is hidden or unpublished, its stock cannot be read by any storefront method, so it will be skipped. If you do not want components discoverable, you can publish them to the Online Store but hide them from search, navigation, and collections.
3. **Inventory tracking on the components.** A component with no Shopify inventory tracking reports no quantity, so it cannot be checked.

## What the app gives you

A metafield on each bundle variant:

- Namespace: `bundles_app`
- Key: `components`
- Type: `list.metaobject_reference`

In Liquid you resolve it like this:

```liquid
{% assign components = variant.metafields.bundles_app.components.value %}
{% for c in components %}
  {{ c.variant.value.title }} x {{ c.quantity.value }}
  (in stock: {{ c.variant.value.inventory_quantity }})
{% endfor %}
```

Each entry `c` gives you:

- `c.variant.value`: the component's product variant (so `c.variant.value.id`, `.title`, `.inventory_quantity`, etc.).
- `c.quantity.value`: how many of that component the bundle contains.

## Step 1: add the data on the cart page

Your cart template in an Online Store 2.0 theme is a `.json` file, so you cannot paste Liquid straight into it. Add the snippet through one of these:

- **Custom Liquid block (no file editing):** Online Store, Themes, Customize, select the Cart page, add a Custom Liquid section or block, and paste the snippet.
- **Snippet plus render (recommended for theme repos):** save the snippet as `snippets/bundle-stock.liquid`, then add `{% render 'bundle-stock' %}` inside the section your `templates/cart.json` already uses (often `sections/main-cart-items.liquid`).

This first block renders the cart's bundle data as JSON for the script to read:

```liquid
{%- comment -%} Bundle component data for the cart {%- endcomment -%}
<script type="application/json" id="bundle-stock-data">
{
  "lines": [
    {%- assign line_first = true -%}
    {%- for item in cart.items -%}
      {%- unless line_first %},{% endunless -%}
      {%- assign line_first = false -%}
      {%- assign components = item.variant.metafields.bundles_app.components.value -%}
      {%- assign line_inv = 'null' -%}
      {%- if item.variant.inventory_management != blank -%}
        {%- assign line_inv = item.variant.inventory_quantity -%}
      {%- endif -%}
      {%- assign is_bundle = 'false' -%}
      {%- if components != blank -%}
        {%- assign is_bundle = 'true' -%}
      {%- endif -%}
      {
        "line_variant_id": {{ item.variant.id }},
        "quantity": {{ item.quantity }},
        "title": {{ item.variant.title | json }},
        "inventory": {{ line_inv }},
        "is_bundle": {{ is_bundle }},
        "components": [
          {%- assign comp_first = true -%}
          {%- for c in components -%}
            {%- assign cv = c.variant.value -%}
            {%- if cv != blank -%}
              {%- unless comp_first %},{% endunless -%}
              {%- assign comp_first = false -%}
              {%- assign comp_inv = 'null' -%}
              {%- if cv.inventory_management != blank -%}
                {%- assign comp_inv = cv.inventory_quantity -%}
              {%- endif -%}
              {
                "variant_id": {{ cv.id }},
                "quantity": {{ c.quantity.value }},
                "title": {{ cv.title | json }},
                "inventory": {{ comp_inv }}
              }
            {%- endif -%}
          {%- endfor -%}
        ]
      }
    {%- endfor -%}
  ]
}
</script>
```

## Step 2: add the script

This block reads the data, totals the demand per component across the whole cart, and publishes the result. It does not change the page or the checkout button. You build the behavior in your own listener (Step 3).

```html
<script>
(function () {
  function compute() {
    var el = document.getElementById('bundle-stock-data');
    if (!el) { return null; }

    var data;
    try { data = JSON.parse(el.textContent); } catch (e) { return null; }

    var demand = {};
    var info = {};

    data.lines.forEach(function (line) {
      if (line.is_bundle) {
        line.components.forEach(function (c) {
          var prev = demand[c.variant_id];
          if (prev == null) { prev = 0; }
          demand[c.variant_id] = prev + (c.quantity * line.quantity);
          info[c.variant_id] = { inventory: c.inventory, title: c.title };
        });
      } else {
        var lid = line.line_variant_id;
        var prevLine = demand[lid];
        if (prevLine == null) { prevLine = 0; }
        demand[lid] = prevLine + line.quantity;
        if (!info[lid]) {
          info[lid] = { inventory: line.inventory, title: line.title };
        } else if (info[lid].inventory == null) {
          info[lid].inventory = line.inventory;
        }
      }
    });

    var components = [];
    var problems = [];

    Object.keys(demand).forEach(function (id) {
      var meta = info[id];
      var available = meta ? meta.inventory : null;
      var title = 'variant ' + id;
      if (meta) { if (meta.title) { title = meta.title; } }

      var entry = { variant_id: id, title: title, demanded: demand[id], available: available };
      components.push(entry);

      if (available != null) {
        if (demand[id] > available) {
          entry.short_by = demand[id] - available;
          problems.push(entry);
        }
      }
    });

    return { ok: problems.length === 0, problems: problems, components: components };
  }

  function run() {
    var result = compute();
    if (!result) { return null; }
    window.bundleStockGuard.result = result;
    document.dispatchEvent(new CustomEvent('bundle-stock:checked', { detail: result }));
    return result;
  }

  window.bundleStockGuard = { run: run, result: null };

  run();
})();
</script>
```

## The data you can build on

Once Step 2 runs, you have two ways to read the result.

A custom event on `document`:

```js
document.addEventListener('bundle-stock:checked', function (e) {
  var result = e.detail;
  // ...
});
```

A global object, useful for re reading later:

```js
window.bundleStockGuard.result; // the latest result
window.bundleStockGuard.run();  // re check and fire the event again
```

The result shape:

- `ok`: boolean. True when nothing in the cart is oversold.
- `problems`: array of components whose combined demand is more than the available stock. Each item has:
  - `variant_id`: the component variant id.
  - `title`: the component title.
  - `demanded`: total units needed across the whole cart.
  - `available`: units in stock.
  - `short_by`: how many units short.
- `components`: every aggregated component (the same fields, without `short_by`), in case you want to show all of them.

## Step 3: examples

### Show a warning banner

```html
<script>
document.addEventListener('bundle-stock:checked', function (e) {
  var old = document.getElementById('bundle-stock-notice');
  if (old) { old.remove(); }
  if (e.detail.ok) { return; }

  var box = document.createElement('div');
  box.id = 'bundle-stock-notice';
  box.setAttribute('role', 'alert');
  box.style.cssText = 'margin:1rem 0;padding:.75rem 1rem;border:1px solid #c00;background:#fff5f5;color:#900;';

  var rows = e.detail.problems.map(function (p) {
    return 'Only ' + p.available + ' of "' + p.title + '" left, but your cart needs ' + p.demanded + '.';
  });
  box.innerHTML = '<strong>Not enough stock for everything in your cart</strong><br>' + rows.join('<br>');

  var anchor = document.querySelector('form[action$="/cart"]');
  if (!anchor) { anchor = document.body; }
  anchor.prepend(box);
});
</script>
```

### Disable the checkout button

```html
<script>
document.addEventListener('bundle-stock:checked', function (e) {
  var blocked = e.detail.ok === false;
  var selectors = '[name="checkout"], #checkout, .cart__checkout-button';
  document.querySelectorAll(selectors).forEach(function (b) {
    b.disabled = blocked;
    b.style.opacity = blocked ? '0.5' : '';
    b.style.pointerEvents = blocked ? 'none' : '';
  });
  // Express checkout buttons (Shop Pay, PayPal, etc.)
  document.querySelectorAll('.shopify-payment-button, [data-shopify="dynamic-checkout"]').forEach(function (elx) {
    elx.style.display = blocked ? 'none' : '';
  });
});
</script>
```

## Ajax carts

The data block is rendered when the page loads. If your theme updates the cart without a full reload (quantity changes, ajax add to cart), re render the cart section so the data block is fresh, then call `window.bundleStockGuard.run()` to recompute and fire the event again.

## Troubleshooting

Add this temporary block to the cart to inspect what the storefront sees:

```liquid
<pre style="background:#f6f6f6;border:1px solid #888;padding:.75rem;font-size:12px;white-space:pre-wrap;">DEBUG
{%- for item in cart.items -%}
{%- assign comps = item.variant.metafields.bundles_app.components.value -%}
line {{ item.variant.id }} | components: {% if comps %}yes{% else %}no{% endif %} | size: {{ comps.size }}
{%- for c in comps -%}
  {{ c.variant.value.title }} | qty {{ c.quantity.value }} | stock {{ c.variant.value.inventory_quantity }}
{%- endfor -%}
{%- endfor -%}
</pre>
```

Read it like this:

- **`components: no`** on a bundle line: the metafield is not on that variant. Ask your Bundles app contact to confirm component references are enabled and backfilled for your store.
- **`size` is correct but title and stock are blank:** the component product is not published to the Online Store sales channel, so the storefront cannot resolve it. Publish it (you may hide it from search and navigation).
- **`stock` is blank for one component:** that component does not have inventory tracking on.

## Limitations

- This runs in the shopper's browser, so it is a helpful guard, not a hard guarantee. A shopper can bypass it, and two shoppers racing for the last unit is not covered. Treat it as a better cart experience, not a replacement for inventory rules.
- It can only see components that are published to the Online Store and have inventory tracking on, as described above.
