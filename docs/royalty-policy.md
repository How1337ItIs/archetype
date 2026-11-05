# Marketplace royalty recipient policy

This page explains the public `RoyaltyPolicyRegistry` implementation in this
repository. It is a source-code guide, **not a deployment registry, audit report,
or guarantee about a live marketplace transaction**.

## Recipient resolution

[`RoyaltyPolicyRegistry.sol`](../src/RoyaltyPolicyRegistry.sol) exposes
`royaltyRecipient(address collection)`, which resolves one recipient address
for a collection, in this order:

1. A configured override wins. An override to the zero address deliberately
   disables the recipient; it does not fall through to the default policy.
2. Without an override, a registered Scatter collection resolves to its current
   `owner()`, including after an ownership transfer.
3. Without an override or registration, the result is the zero address.

Clearing an override restores the default policy. If a registered collection's
`owner()` call reverts, resolution reverts too. If that call returns the zero
address, the result is the zero address.

The registry does not query ERC-2981 `royaltyInfo(tokenId, salePrice)`. Its
collection-level recipient lookup should not be described as implementing that
interface or as discovering a token-specific royalty amount.

## Who controls the policy?

The registry inherits OpenZeppelin `Ownable`. Its owner can:

- approve or revoke factories with `setApprovedFactory`;
- register existing collections with `registerScatterCollections`;
- set a recipient override, including a disabling zero override, with
  `setRoyaltyOverride`;
- clear an override with `clearRoyaltyOverride`.

Approved factories can register collections with `registerScatterCollection`.
Registration requires contract code at the collection address and cannot be
repeated. Revoking a factory prevents its future registrations; the function
does not remove collections it already registered. Collection ownership and
registry ownership are separate roles: a collection owner does not thereby gain
permission to change registry overrides.

The events `ApprovedFactoryUpdated`, `ScatterCollectionRegistered`, and
`RoyaltyOverrideUpdated` expose these configuration changes. The inherited
`owner()` getter identifies the registry administrator, not the collection's
recipient.

## Recipient policy is not the marketplace fee schedule

This contract returns an address. It does **not** calculate a fee percentage,
split fees between a creator and a referrer, select referral attribution, or
execute marketplace settlement. A resolved recipient is not proof that the
recipient receives a payment on every trade.

For the published product rules, see Scatter's
[marketplace fees](https://docs.scatter.art/platform/fees#marketplace-fees) and
[royalties](https://docs.scatter.art/marketplace/royalties) documentation.
In particular, the documented active-referral rule replaces the collection's
share of the marketplace fee; this registry alone does not implement that rule.
Do not confuse secondary-market rules with minting fees or minting affiliate
settings.

The source comment also warns that router fills can revert when the recipient
rejects an ETH or ERC-20 transfer. That settlement behavior needs to be checked
against the actual router version and transaction, not inferred solely from a
successful registry lookup.

## What to verify for a live deployment

Before treating this source as applicable to a live trade, establish:

- the chain ID, registry address, explorer link, verified source, and source
  commit/build corresponding to its deployed bytecode;
- any proxy and implementation addresses, upgrade authority, and current
  registry owner;
- the collection's registration, override history, current resolved recipient,
  and the collection ownership used by the default policy;
- the exchange/router actually used and how it consumes this recipient, applies
  referral and fee policy, and handles failed transfers;
- any audit report's URL, exact reviewed commit, scope, date, findings and
  remediation status. Test files with "Audit" in their name are not themselves
  independent audit reports. If independent review has not been published,
  state that explicitly rather than imply an audit.

These are verification requirements, not claims that any specific deployment or
report is present or absent. This repository page intentionally supplies no
unverified production addresses.

## Existing executable coverage

[`RoyaltyPolicyRegistry.t.sol`](../test/RoyaltyPolicyRegistry.t.sol) covers
unregistered and registered collections, factory authorization, duplicate
registration, batch registration, current-owner changes, nonzero and zero
overrides, clearing overrides, reverting/zero `owner()` results, owner-only
configuration, and non-contract addresses. With the repository's Foundry
requirements installed, run:

```sh
forge test --match-contract RoyaltyPolicyRegistryTest
```

Those unit tests exercise source behavior. They do not establish the deployment
mapping, the exchange fee schedule, or an independent security review.
