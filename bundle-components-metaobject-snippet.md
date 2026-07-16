# Showing bundle components on the product page

A theme snippet that shows the components of a bundle on the product template, using the component reference metaobjects the app maintains. The list resolves the component variants live at render time, so titles, images and availability ("Sold out") are always current.

Products without components render nothing, so the snippet is safe to add to every product template unconditionally.

## Prerequisites

1. The store has the component reference feature enabled and synced:

   ```bash
   art metaobjects:component-references {store} --enable
   ```

2. The store has accepted the required scopes (`write_metaobjects`, `write_metaobject_definitions`, `write_products`). Check with:

   ```bash
   art shop:scopes {store}
   ```

3. Storefront access needs no extra setup: the app creates both the metaobject definition and the metafield definition with `PUBLIC_READ`.

## Install (merchant instructions)

1. In the Shopify admin go to Online Store, then Themes, then Edit code.
2. Under Snippets choose "Add a new snippet" and name it `bundle-components`.
3. Paste the snippet below into the new file and save.
4. Add this line to the product template or section where the components should appear:

   ```liquid
   {% render 'bundle-components' %}
   ```

   Optional parameters:

   ```liquid
   {% render 'bundle-components', heading: 'What is inside' %}
   {% render 'bundle-components', variant: product.selected_or_first_available_variant %}
   ```

## The snippet

```liquid
{% comment %}
  bundles.app: show a bundle's components on the product page.

  Reads the bundle variant's bundles_app.components metafield (a list of
  metaobjects that each hold a component variant reference and a quantity)
  and renders nothing for products without components.
{% endcomment %}

{%- liquid
  assign bundle_variant = variant | default: product.selected_or_first_available_variant
  assign component_refs = bundle_variant.metafields.bundles_app.components.value
  assign heading = heading | default: 'This bundle contains'
-%}

{%- if component_refs and component_refs.size > 0 -%}
  <div class="bundle-components">
    <h3 class="bundle-components__heading">{{ heading }}</h3>

    <ul class="bundle-components__list" role="list">
      {%- for component in component_refs -%}
        {%- assign component_variant = component.variant.value -%}
        {%- if component_variant -%}
          {%- assign component_product = component_variant.product -%}
          {%- assign component_url = component_product.url | append: '?variant=' | append: component_variant.id -%}
          {%- assign component_image = component_variant.image | default: component_product.featured_image -%}

          <li class="bundle-components__item">
            {%- if component_image -%}
              <a href="{{ component_url }}" class="bundle-components__media" tabindex="-1" aria-hidden="true">
                {{ component_image | image_url: width: 120 | image_tag: loading: 'lazy', class: 'bundle-components__image' }}
              </a>
            {%- endif -%}

            <div class="bundle-components__info">
              <a class="bundle-components__title" href="{{ component_url }}">
                {{- component_product.title -}}
                {%- unless component_variant.title == 'Default Title' %} ({{ component_variant.title }}){% endunless -%}
              </a>

              <span class="bundle-components__quantity">
                {{- component.quantity.value }}&times;
              </span>

              {%- unless component_variant.available -%}
                <span class="bundle-components__sold-out">Sold out</span>
              {%- endunless -%}
            </div>
          </li>
        {%- endif -%}
      {%- endfor -%}
    </ul>
  </div>

  <style>
    .bundle-components {
      margin: 1.5rem 0;
    }

    .bundle-components__heading {
      margin: 0 0 0.75rem;
      font-size: 1rem;
    }

    .bundle-components__list {
      list-style: none;
      margin: 0;
      padding: 0;
      display: flex;
      flex-direction: column;
      gap: 0.75rem;
    }

    .bundle-components__item {
      display: flex;
      align-items: center;
      gap: 0.75rem;
    }

    .bundle-components__image {
      width: 60px;
      height: 60px;
      object-fit: cover;
      border-radius: 6px;
      display: block;
    }

    .bundle-components__info {
      display: flex;
      align-items: baseline;
      gap: 0.5rem;
      flex-wrap: wrap;
    }

    .bundle-components__title {
      color: inherit;
      text-decoration: none;
    }

    .bundle-components__title:hover {
      text-decoration: underline;
    }

    .bundle-components__quantity {
      font-variant-numeric: tabular-nums;
      opacity: 0.7;
    }

    .bundle-components__sold-out {
      font-size: 0.85em;
      color: #b00020;
    }
  </style>
{%- endif -%}
```

## How it works

The app stores the components of a bundle on the bundle variant:

| Piece | Value |
| --- | --- |
| Metafield on the bundle variant | `bundles_app.components`, type `list.metaobject_reference` |
| Metaobject type | `$app:bundle_component` (app reserved, hidden from the merchant) |
| Metaobject field `variant` | Reference to the component product variant |
| Metaobject field `quantity` | Number of units of the component in the bundle |

The snippet iterates the metafield list, resolves each referenced variant, and renders image, linked title, quantity and availability. All data is read live from the storefront, so component availability follows the actual inventory.

## Limitations

1. The snippet renders the components of the variant selected at page load. Themes that switch variants without a page reload will not update the list. Typical bundle products have a single variant, so this is usually not an issue.
2. Text in the snippet ("This bundle contains", "Sold out") is plain English. Merchants with multilingual themes can replace these strings with translation keys from their own locale files.
3. The snippet was validated with Shopify Theme Check.
