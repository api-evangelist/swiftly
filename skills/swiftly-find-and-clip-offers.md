---
name: Find and clip Swiftly offers for a shopper
description: Search a retailer's offer catalog on the Swiftly Shopper GraphQL API and clip the ones a shopper wants, then confirm they landed in the clipped set.
api: graphql/swiftly-shopper.graphql
endpoint: https://prod.swiftlyapi.net/graphql
operations:
  - searchOffers
  - rankedOffers
  - productOffers
  - claimOffer
  - claimedOffers
  - declineOffer
generated: '2026-08-05'
method: generated
source: graphql/swiftly-shopper-introspection.json
---

# Find and clip Swiftly offers

Swiftly's offer surface is what shoppers call "digital coupons". You search it, you clip
one, and the clip is later honoured at the register against the shopper's loyalty id.

## Before you start

- The endpoint is `POST https://prod.swiftlyapi.net/graphql`, `Content-Type: application/json`.
- Send the retailer tenant key in `X-Swiftly-Account-Key`. Without a chain context the API
  fails with `IllegalStateException: Chain not available in session context`.
- Send the shopper's token in `Authorization`. Without it, shopper-scoped fields fail with
  `InvalidTokenException: Invalid access token.`
- Swiftly publishes no documentation for any of this. Everything here is grounded in the
  introspected schema (`graphql/swiftly-shopper.graphql`) and in observed live responses.

## Steps

1. **Search.** Call `searchOffers(keyword, offset, size, limit)`. Note that `size` and
   `limit` both exist on this field and overlap — pick one and stay consistent.
2. **Or rank within a category.** Call `rankedOffers(categoryId, offerIds, siteId)` when you
   already know the category or a candidate id set.
3. **Or start from a product.** Call `productOffers(productId)` to get every offer attached
   to a catalog item — pair with `productByBarcode(barcode, storeId)` after a scan.
4. **Read the offer before clipping.** On each `Offer` check `clipStartDate`,
   `clipEndDate`, `expirationDate`, `isExclusive`, `isStackable`, `maxRedeemCount` and
   `daysToRedeemAfterClip`. The value itself lives in the `offerParameters` union — resolve
   it to one of `BoGoParameters`, `CentsOffParameters`, `FixedPriceParameters`,
   `FreeOfferParameters`, `OrderTotalParameters`, `PercentOffParameters`,
   `CompoundCentsOffParameters` or `CompoundFixedPriceParameters`. Do not read
   `Offer.limitedUse` — it is deprecated in favour of `multipleUse`.
5. **Clip.** Call `claimOffer(offerId)`. It returns a `String`, not an object.
6. **Verify.** Call `claimedOffers(offset, size, limit)` and confirm the offer id is
   present. **Do this — there is no idempotency key on `claimOffer`, so you must not retry
   a clip you are unsure about; read back instead.**
7. **Decline** an offer the shopper rejects with `declineOffer(offerId)`, which returns an
   `OfferMutationResponse`.

## Rules

- **Never retry a mutation blindly.** No mutation on this API accepts a client-supplied
  request key. On a timeout, re-read `claimedOffers` and decide from that.
- **Respect age gating.** `OfferV2Response.ageRestricted` marks alcohol and other restricted
  offers. If the shopper is not verified, run the `swiftly-verify-shopper-age` skill first;
  do not clip a restricted offer on an unverified account.
- **Errors arrive with HTTP 200.** Always inspect the `errors[]` array, never the status
  code. Capture `errors[].extensions.requestId` — it is the only correlation handle Swiftly
  exposes. See `errors/swiftly-problem-types.yml`.
- **Two offer models coexist.** `Offer` (v1) and `OfferV2Response` (via `offersv2byStoreId`,
  `offersv2byZipcode`, `offersv2bySourceId`) are different shapes for overlapping data and
  neither is marked deprecated. Pick one per integration and record which.
