---
name: Resolve a Swiftly store and look up products and circulars
description: Find the right retailer store on the Swiftly Shopper GraphQL API, then search the catalog, scan a barcode, and pull the weekly digital circular for that store.
api: graphql/swiftly-shopper.graphql
endpoint: https://prod.swiftlyapi.net/graphql
operations:
  - chain
  - stores
  - store
  - storeByNumber
  - products
  - product
  - productByBarcode
  - taxonomy
  - flyersByDate
generated: '2026-08-05'
method: generated
source: graphql/swiftly-shopper-introspection.json
---

# Resolve a store, then read the catalog

Almost everything on this API is store-scoped. Pricing, inventory, offers and circulars all
change per store, so store resolution is step one for every other flow.

## Before you start

- `POST https://prod.swiftlyapi.net/graphql` with `X-Swiftly-Account-Key` for the retailer
  tenant. Catalog reads do not need a shopper token; shopper-scoped fields do.
- The Swiftly web client also sends `X-Swiftly-Lat` and `X-Swiftly-Lng`, which participate
  in store resolution.
- `chainId` and `storeId` are UUIDs. Passing `"1"` returns `Invalid UUID string: 1`.

## Steps

1. **Get the chain.** `chain(chainId)` returns `branding` and `configuration` — use it to
   render retailer-correct labels rather than hardcoding them.
2. **Resolve the store.** Pick one:
   - `stores(chainId)` → `Stores` with `all` (every `StoreEntry`) and `nearby`
     (`StoreEntryDistance`, distance-ordered — this is the one to use with lat/lng).
   - `store(chainId, storeId)` when the store is already known.
   - `storeByNumber(storeNumber)` when you have the retailer's own store number.
3. **Read what the store supports.** `StoreEntry` is a GraphQL **interface**, so shape
   varies by retailer. Query the fields you need explicitly: `address1`, `city`, `state`,
   `postalCode`, `latitude`, `longitude`, `timeZone`, `isOpenNow`, `amenities`,
   `shoppingModes`, `payments`, `accessibility`, `swiftlyServices`, `primaryDetails`,
   `departmentDetails`. Do not assume a concrete store type.
4. **Browse the taxonomy.** `taxonomy(id)` returns the category `graph`;
   `offerBrowseCategories(siteId)` and `sortDimensions(siteId)` give the browse and sort
   surfaces.
5. **Search the catalog.**
   `products(match, taxonomyId, storeId, sortBy, offset, limit, cookie)` returns
   `ProductResult` with `products`, `count`, `fieldFacets`, `spelling` and a `cookie`.
   **Paginate with `cookie`, not by incrementing `offset`** — the cookie is the continuation
   token this query is built around.
6. **Fetch one product.** `product(productId, storeId)` returns `Product` with `name`,
   `brand`, `unitSize`, `soldBy`, `inventoryQuantity`, `images`, `price` (`PriceResult`),
   `webPrice`, `tags`, `notices`, `eligibilities` and attached `offers`. On `PriceResult`
   use `success`/`failure`, not the deprecated `ok`/`err`. Do not read
   `PriceDetail.price` — deprecated so callers stop treating money as a bare number.
7. **Scan.** `productByBarcode(barcode, storeId)` resolves a scanned UPC to the same
   `Product` shape.
8. **Pull the circular.** `flyersByDate(storeId, date)` returns `Flyer` with `title`,
   `startDate`, `endDate`, `activationDate` and `images`. On `ImageRef`, use `image` — the
   plural `images` field is deprecated in favour of a single mandated density.

## Rules

- **Never cache a price across stores.** Price, inventory and offers are store-scoped; a
  `Product` fetched for one `storeId` is not valid for another.
- **Errors arrive with HTTP 200** in `errors[]`. Log `errors[].extensions.requestId`.
- **Responses are marked uncacheable** (`cache-control: no-cache, no-store,
  must-revalidate`). Do not build a shared cache layer on top of these reads without the
  retailer's agreement.
- **This API is undocumented.** Swiftly ships no reference for it, so the schema in
  `graphql/swiftly-shopper.graphql` is the contract — and it can change without notice.
  There is no changelog, no deprecation policy and no status page
  (`lifecycle/swiftly-lifecycle.yml`).
