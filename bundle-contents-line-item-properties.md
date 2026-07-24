# Persisting bundle contents on order lines (line item properties)

## What this is

A theme integration that copies a bundle's components into **line item properties** when the bundle is added to the cart. The properties travel with the cart line into checkout, the order, email notifications, and anything that reads order line items (fulfillment apps, packing slips, order printers). The result: whoever picks the order sees "2x Red Mug (MUG-R)" on the bundle line without opening the app.

The data source is the `bundles_app.content` metafield on the bundle's ProductVariant, the standard json metafield the app writes for every bundle. No opt in feature is required.

**This is a guide, not a copy paste snippet.** Product forms, variant pickers, and cart rendering differ per theme. The snippet below shows the working pattern; placement and the variant switching behavior have to be adapted to the theme at hand.

## How it works

1. Liquid reads the bundle variant's `bundles_app.content` metafield (a json list of components with `title`, `sku`, `quantity`).
2. For each component it renders a hidden `<input name="properties[...]">` **inside the product form**.
3. When the shopper submits the form, Shopify attaches the properties to the cart line, and from there to the order line item.

Property names start with an underscore (`_Bundle Contents`). By convention, modern themes and checkout hide underscore prefixed properties from the customer while they stay visible on the order in the admin and to apps. Drop the underscore if the customer should see them in cart and checkout too.

## Prerequisites

The snippet assumes the **default content shape**: a json list where each component has `title`, `sku`, and `quantity` keys. That is what the app writes when the store has no `metafieldsfields` setting configured.

Stores with a custom `metafieldsfields` value need the snippet adapted:

- A comma separated field list (for example `title,quantity`) produces only those keys, so references to missing keys render blank.
- A `SINGLE::` template produces a plain string instead of a list. The whole loop is unnecessary then, output the metafield value into one property directly.

Check the store's setting before installing, and inspect the metafield on a real bundle variant (Shopify admin, product, variant, metafields) to confirm the shape.

## The pattern

### Recommended: one joined property

One property holds the whole contents. This avoids numbered property collisions and keeps orders tidy.

Create a snippet, for example `snippets/bundle-line-properties.liquid`:

```liquid
{% comment %}
  bundles.app: attach a bundle's components to the cart line as a
  line item property. Must be rendered INSIDE the product form.
  Renders nothing for non bundle variants.
{% endcomment %}

{%- liquid
  assign bundle_variant = variant | default: product.selected_or_first_available_variant
  assign lines = bundle_variant.metafields.bundles_app.content.value
-%}

{%- if lines != blank -%}
  {%- capture summary -%}
    {%- for line in lines -%}
      {{- line.quantity }}x {{ line.title }} ({{ line.sku }}){%- unless forloop.last %}, {% endunless -%}
    {%- endfor -%}
  {%- endcapture -%}
  <input type="hidden" name="properties[_Bundle Contents]" value="{{ summary | strip | escape }}">
{%- endif -%}
```

Note the direct field access (`line.title`). The json metafield's `.value` is a list of objects, so there is no need to iterate key value pairs the way older examples do.

### Alternative: one property per component

If the consumer of the properties (a fulfillment integration, an order printer template) wants separate fields, replace the capture block with:

```liquid
{%- for line in lines -%}
  <input type="hidden"
         name="properties[_Component {{ forloop.index }}]"
         value="{{ line.quantity }}x {{ line.title }} ({{ line.sku }})">
{%- endfor -%}
```

Keep this variant scoped to a single bundle variant (as above). Rendering it per product across all variants makes `forloop.index` restart for every variant, so properties from different bundle variants overwrite each other under the same name.

## Placing it in the theme

The inputs only work inside the `<form>` that posts to `/cart/add`. Where that form lives differs per theme:

- **Dawn and most OS 2.0 themes**: the form is created in `snippets/buy-buttons.liquid` (`{%- form 'product', product ... -%}`). Render the snippet inside that form block:

  ```liquid
  {% render 'bundle-line-properties', variant: product.selected_or_first_available_variant %}
  ```

- **Older themes**: look for `{% form 'product' %}` in `sections/product-template.liquid` or `templates/product.liquid`.
- **Quick add, quick view, featured product sections**: these have their own product forms. Every add to cart path that can add a bundle needs the snippet, otherwise some lines carry the properties and others do not, and the cart shows the same variant split across two lines (properties make lines distinct).

### Variant switching

The hidden inputs are rendered for the variant selected at page load. What happens when the shopper picks another variant depends on the theme:

- Themes that re render the product info section on variant change (Dawn does, via the Section Rendering API) refresh the inputs automatically, as long as the snippet sits inside the re rendered section.
- Themes that only swap the variant id input via JavaScript leave the properties stale. Then either re render the form section on variant change, or update the inputs from JavaScript using a per variant data attribute you render alongside the form.

If all bundles in the store are single variant products (common), this concern disappears.

## Verifying

1. Add a bundle to the cart. In the cart, request `/cart.js` and check the line's `properties`.
2. Place a test order and confirm the properties show on the order line in the admin.
3. Add the same bundle through every other add to cart path (quick add, ajax drawers) and confirm the cart does not split the variant into two lines.

## Limitations

- **Snapshot, not live.** The property captures the contents at the moment of add to cart. If the bundle definition changes while the item sits in a cart, the property keeps the old text. The order itself is expanded by the app from the definition at order time, so treat the property as informational, not authoritative.
- **Customer visible unless hidden.** Themes that print all properties in the cart (some older ones do) will show underscore properties too; filter them in the cart template if needed (`{% unless property.first contains '_' %}`).
- **Line splitting.** Identical variants with different properties become separate cart lines. Roll the snippet out to all add to cart paths at once.

## Better alternatives, depending on the goal

- **Live component data on the storefront.** If the goal is showing contents on the product or cart page (rather than persisting them on the order), read the metafields directly at render time instead of copying them into properties. Use `bundles_app.content`, or the component reference metaobjects for live inventory, see `docs/bundle-component-references.md` and `docs/bundle-components-snippet.md`.
- **Order level needs.** If the information is only needed after the order exists (packing slips, ERP export), skip the theme entirely and read the bundle definition from the app or the order expansion, properties add nothing there.
