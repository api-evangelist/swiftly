---
name: Read a Swiftly loyalty balance and redeem a reward
description: Enroll a shopper in a retailer loyalty program, read their points balance, and spend points on a reward — the only flow on the Swiftly Shopper GraphQL API that moves shopper-owned value.
api: graphql/swiftly-shopper.graphql
endpoint: https://prod.swiftlyapi.net/graphql
operations:
  - loyaltyPrograms
  - enrollInLoyaltyProgram
  - loyaltySummary
  - loyaltyCard
  - availableLoyaltyRewards
  - purchaseReward
  - issuedLoyaltyRewards
  - challenges
generated: '2026-08-05'
method: generated
source: graphql/swiftly-shopper-introspection.json
---

# Swiftly loyalty: points, rewards, redemption

This is the highest-consequence flow on the API. `purchaseReward` spends shopper-owned
points and cannot be undone through the API.

## Before you start

- `POST https://prod.swiftlyapi.net/graphql` with `X-Swiftly-Account-Key` (tenant) and
  `Authorization` (shopper bearer token). See `authentication/swiftly-authentication.yml`.
- `chainId`, `storeId` and `userId` are UUIDs. `programId`, `loyaltyAccountId`, `rewardId`
  and `rewardInstanceId` are plain strings. Sending an integer where a UUID is expected
  returns `Invalid UUID string`.

## Steps

1. **List programs.** `loyaltyPrograms(customRetailerProperties)` returns `LoyaltyProgram`
   with `programId`, `programName`, `enrolled`, and an `account` of type
   `LoyaltyProgramAccount` carrying `accountId` and `accountStatus`.
2. **Enroll if needed.** If `enrolled` is false, call
   `enrollInLoyaltyProgram(programId, customRetailerProperties)`. It returns the
   `LoyaltyProgramAccount`; keep `accountId` — downstream calls need it as
   `loyaltyAccountId`.
3. **Read the balance.** `loyaltySummary(userId)` returns `availablePoints`,
   `lifetimeSavingsCents`, `summaryPoints` and a `ledgerVersion`. Treat `ledgerVersion` as
   the freshness marker for the balance you are about to spend against.
   Do not use `LoyaltySummary.issuedRewards` — it is deprecated; use the
   `issuedLoyaltyRewards` query instead.
4. **Fetch the loyalty card** when the shopper needs to scan in store:
   `loyaltyCard(userId)` returns `loyaltyId`, `loyaltyIdFormat`, `barcodeType` and a
   scannable `payload`.
5. **List what points can buy.**
   `availableLoyaltyRewards(storeId, loyaltyAccountId, loyaltyProgramId)` returns `Reward`
   with `rewardId`, `displayName`, `pointCost`, `status` and `termsAndConditions`.
6. **Confirm before spending.** Check `availablePoints >= pointCost` and surface
   `termsAndConditions` to the shopper. Get explicit confirmation.
7. **Redeem.**
   `purchaseReward(storeId, userId, rewardCanonicalId, loyaltyAccountId)` returns a
   `PurchaseRewardResponse`.
8. **Verify.**
   `issuedLoyaltyRewards(loyaltyAccountId, loyaltyProgramId, chainProvisioningStatus)`
   returns `IssuedReward` with `rewardInstanceId`, `scannablePayload`, `expiration`, and
   both `chainProvisioningStatus` and `storeProvisioningStatus`. A reward is only usable
   once provisioning has landed on both. Read `expiration` (a `DateOrDateTime` union), not
   the deprecated `expiryDateTime`.
9. **Optional: challenges.** `challenges(criteria)` returns `ChallengesQueryResult` with
   `items` and a `pageToken` continuation — a third pagination style on this API.

## Rules

- **`purchaseReward` is not idempotent and not reversible.** No mutation on this API
  accepts a client-supplied request key. If the call times out, do **not** repeat it —
  re-read `issuedLoyaltyRewards` and `loyaltySummary` and decide from the result.
- **Never redeem without explicit shopper confirmation.** This spends value the shopper
  earned.
- **Errors come back as HTTP 200** in an `errors[]` array. Record
  `errors[].extensions.requestId`; it is the only correlation id available and Swiftly
  publishes no support runbook.
- **Provisioning is asynchronous.** A successful `purchaseReward` does not mean the reward
  is scannable yet — poll `issuedLoyaltyRewards` until both provisioning statuses settle.
