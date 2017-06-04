    BSIP: 0017a
    Title: Partial Solution to Reviving BitAssets after Global Settlement
    Authors: Peter Conrad <conrad@quisquis.de>
    Status: Draft
    Type: Protocol
    Created: 2017-06-04
    Discussion: tbd
    Copyright: Public Domain

# Abstract

This proposal is a simplified version of BSIP-0017. In addition, it proposes
an alternative way to revive SmartCoins, i. e. by re-collateralization.

# Motivation

The necessity of reviving bitassets has already been discussed in BSIP-0017, and
is unquestioned.

BSIP-0017 aims to provide a complete solution that is applicable to all kinds of
bitassets, in all situations. This results in a very complex proposal that
covers all corner cases. However, its complexity makes it difficult to
implement, and even though the issue has been talked about for more than half a
year now, no known attempts have been made to implement the proposal.

This proposal, in contrast, aims at a simple solution that can be implemented
with comparatively little effort. The price for simplicity is that it sacrifices
completeness, i. e. it does not cover all possible situations.

# Rationale

Bitassets come with two kinds of "guarantee":

1. After global settlement, bitassets can be exchanged for the backing asset
   at a fixed price.

2. Before global settlement, SmartCoins can be exchanged for the backing asset
   at the feed price.

The ideas presented here ensure that one of these guarantees always holds. The
only new thing is that SmartCoins can switch from guarantee #1 back to
guarantee #2.

In addition, the steps described below do not make the implementation of the
full BSIP-0017 proposal more difficult. In fact, the "Long Squeeze" discussed
below is a partial implementation of BSIP-0017 that can later be extended to
a full implementation.

# Specification

Like in BSIP-0017, let SWAN be an asset that has seen global settlement, and
let BACK be the asset backing SWAN.

### Bugfix: MPAs that have seen a black swan cannot be settled after the price feed expires

It has turned out that force-settling an MPA requires a valid price feed even
when the MPA has a settlement_price set. This is clearly a bug, since in that
case the settlement price is independent from the price feed. Furthermore,
publishing price feeds is no longer possible after a black swan, so the time
when settlement is possible at all is limited to the expiration period of the
price feed of the MPA.

Resolution:

This bug will be fixed. See https://github.com/cryptonomex/graphene/issues/664#issuecomment-254056746
for a discussion.

## Auto-revive empty bitassets

A bitasset is "empty" if nobody is holding a positive amount of it anymore. The
only reasonable exception to this rule is the pool of accumulated fees belonging
to the asset itself. This situation can occur after all holders of SWAN have
settled their position via forced settlement.

Resolution:

The emptiness of a bitasset can easily be determined. When SWAN is empty,
the remainder of the settlement fund will be paid out to the issuer, the
accumulated fees and the current supply are reset to zero, and the settlement
price is cleared.

## Auto-revive after increase of settlement fund value

This applies only to SmartCoins, not to Prediction Markets.

A price increase of BACK can lead to the situation where SWAN is worth much more
than it was originally intended to be. I. e. the value of the settlement fund
becomes much greater than the nominal value of the existing supply of SWAN.

Resolution:

When the value of the settlement fund reaches the minimum required collateral
(in terms of price feed and MCR), a call_order_object owned by the issuer of
SWAN is created (or updated) that takes the settlement_fund as collateral and
covers the full debt. The settlement_fund and the settlement_price will then
be cleared, which revives the asset.

The condition can easily be checked at the time the price feed is updated.
Obviously, this requires a price feed. Currently it is not possible to
publish a price feed for assets that have seen global settlement. This
restriction will be removed.

## Revival by re-collateralization

This applies only to SmartCoins, not to Prediction Markets.

The idea of turning the settlement fund into a short position when its value
has increased sufficiently can easily be extended. If the value of the
settlement fund itself is not sufficient to create a sufficiently collateralized
short position (in terms of price feed and MCR), an investor could volunteer to
add the required amount of BACK to the fund and take ownership of the resulting
short position.

Resolution:

For this, a new operation "recollateralize" will be introduced. Its parameters
are the SWAN asset id, the investor's account id, and the amount of collateral
to add. During evaluation of this operation it will be checked that SWAN has
a settlement price, that the investor owns the specified amount of BACK, and
that the resulting short position is sufficiently collateralized (in terms of
the price feed and MCR). If the checks are successful, a call_order_object
belonging to the investor will be created or updated as described above. Then,
the settlement price and settlement_fund will be cleared.

Obviously, this operation is only possible if SWAN has a valid feed price.

The fee for this operation will be paid by the investor. The fee is equal to
the fee of the call_order_update operation.

# Long squeeze

For various reasons, it is unlikely that an asset will become "empty" (in the
sense described above) all by itself. This is, however, a necessary precondition
for reviving a prediction market. For an investor who wants to re-collateralize
an asset, it may also be desirable to have a large portion of the supply
settled, because that will keep the required amount of additional collateral
low.

Resolution:

Therefore, a new operation "long_squeeze" will be introduced. It takes as
parameters the asset id of SWAN, and a flag indicating if a full or a partial
squeeze is intended. In this proposal, only the partial squeeze will be allowed,
the full squeeze is an extension for the implementation of BSIP-0017.

The operation is only allowed on assets that have a settlement_price. The
operation requires authorization from the issuer of SWAN, who will also pay the
fee.

The long squeeze is computationally expensive. Therefore, the operation will
only mark SWAN for the long squeeze. The actual squeeze will happen in the
next maintenance interval (unless SWAN has already been revived by that time).
The fee for that operation will also have to be comparatively expensive. For the
sake of simplicity, it will be set to half the fee of creating a "cheap" asset.

The long squeeze encompasses the following actions:

### force_settlement_object

Resolution: Pending forced settlements will be executed immediately.

### limit_order_object

Limit orders can either buy or sell SWAN. In the case of a sell, the order holds
some amount of SWAN.

Resolution: Sell orders will be cancelled, so the contained SWAN can be settled
in the next step.

### account_balance_object

This is what holds the actual balances of an account. Each such object refers to
a specific account and asset. Account_balances of SWAN can be converted into
BACK at any time via the forced settlement feature.

Resolution: Account balances of SWAN are converted into BACK immediately.

### balance_object

Balance objects (not to be confused with account_balance objects) contain
genesis balances, possibly with linear vesting. This should be a rare case in
BitShares-2.0, but it is a possible case and requires resolution.

Resolution: All balances of SWAN are replaced with equivalent balances of BACK.

# Discussion

## Fees

As has been mentioned in the discussion about BSIP-0017, adding new types of
fees to the blockchain's fee data structure is non-trivial. Therefore, in this
proposal existing fee types are re-used for the new operations. If necessary,
separate fee types can be introduced at a later time. 

## Re-collateralization

Re-collateralization is deliberately not restricted to the issuer of SWAN. The
intent here is to incentivize potential investors. Effectively, during an
uptrend in the value of BACK, this works like a reverse auction. A higher value
of BACK results in a lower required amount for re-collateralization, i. e.
the chance/risk ratio increases. This incentive makes sense, because
additional collateral is in the best interest of the SWAN holders.

This "reverse auction" ends when SWAN is auto-revived by creating a short
position for the issuer. At that point, an "investor" could re-collateralize
with zero risk, which is no longer in the interest of the holders.

## Affected Parties

### Exchanges

Exchanges will be specifically affected by the long squeeze.

An exchange holding SWAN will have its SWAN exchanged for BACK just like anyone
else. The exchange will probably have SWAN balances in its internal ledgers,
which belong to the users of the exchange. The exchange MUST be notified of the
settlement and it MUST modify its internal ledger, replacing SWAN balances
with BACK balances.

Exchanges SHOULD perform this operation independently from the revival, i. e.
before the revival is triggered. They can do this by force-settling their SWAN
holdings.

### (Other) Users

The issuer of SWAN has a choice between squeezing holders at the settlement
price, or re-collateralizing SWAN and thereby reactivating the feed price as the
settlement price. It is possible that the issuer will take the choice that is
economically in his own favour. It is, however, also likely that he will act in
the best interest of the users of SWAN, because he will be interested in keeping
them as users.

# Summary

This proposal discusses possibilities to bring back a "stuck" asset into a
usable state:

* when all longs have been settled,
* when the value of the settlement fund has increased considerably,
* when an investor puts additional coins into the settlement fund.

This is supplemented by the possibility to force-settle some of the long
positions.

Not all shareholders need to understand the technical details of the proposal,
however, they should be aware of the implications of these changes. It is
particularly important to understand how the revival of an asset affects the
different parties, i. e. holders, shorters, traders and issuers.
