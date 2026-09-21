# ItemService affix augmentation (proposed host API)

**Status: declaration only. D2RLoader host implementation and native D2R build qualification are required before this function is available.** No SoE currency should invoke this service until the loader publishes the implemented function pointer.

## Purpose

`ItemService::augmentItemAffix` adds **exactly one** eligible native MagicPrefix or MagicSuffix modifier **in place** to an existing rare item. It supports `RandomEligible` and `ExplicitId` selection. The host must never regenerate the item or reroll existing affixes.

This is intended as a general item operation, not a special-case Exalted Orb function.

## ABI compatibility

The existing ItemService ABI V1 functions and offsets are unchanged. The optional new pointer is appended at byte offset 64, making the new published table 72 bytes. `ItemServiceRequiredSize` remains **64** for existing consumers; `ItemServiceAffixAugmentRequiredSize` is **72**.

Plugins MUST feature-check both the size and the pointer, for example:

```cpp
if (!D2RL::HasItemServiceField(items, D2RL::ItemServiceAffixAugmentRequiredSize) ||
    items->augmentItemAffix == nullptr) {
    // Unsupported loader; do not use an item-regeneration fallback.
}
```

The host MUST NOT publish `serviceSize = 72` until it sets the new function pointer to an implemented, qualified host function. A 64-byte host table remains a valid ItemService V1.

## Request behavior

* `selection=RandomEligible` requires `affixId=0` and supports `kind=Either/Prefix/Suffix`.
* `selection=ExplicitId` requires `kind=Prefix/Suffix` and a one-based native `affixId` from the active MagicPrefix/MagicSuffix bank.
* `maxPrefixes/maxSuffixes/maxAffixes=0` means defaults of 3/3/6, respectively. Nonzero maxima must stay within those native limits.
* The target must be an eligible Rare item with an empty permitted native affix slot. Item level, affix group conflicts, type applicability and other native rules must be checked against the active data tables.
* `paymentItem=InvalidItemHandle, paymentQuantity=0` requests an unpaid operation. Otherwise the payment handle must be distinct from the target and owned by the same player, with the specified available quantity.
* Mutations require the authoritative game thread, active plugin, owner/handle validation and a local or TCP/IP host (remote clients: `NotAuthoritative`).

## Native backend requirements

The loader host must:

1. Acquire/validate both handles and payment, including transient cursor ownership, quantity and allowed containers, **before** mutation.
2. Select a valid affix with game-compatible RNG, or validate the explicit native ID. Reject unavailable side/capacity/group/level/type combinations before mutation.
3. Capture a reversible native snapshot including current affix slots, full existing magic stat-list entries, item identity/seed/name/state and payment quantity/identity.
4. Apply one new affix through the build-qualified native affix/stat machinery, keeping existing rolls and affix slots unchanged and adding one ID.
5. Audit the postcondition: old affix IDs and values identical, new eligible ID in exactly one previously empty slot, newly applied stats consistent even when they overlap an existing stat ID, original native item identity and ordinary item properties retained.
6. Debit payment **only after** the new affix passes validation; on any failure, restore all native changes and verify restoration. On irreversible or unverified rollback, return `RollbackFailed`, quarantine further mutations in this game session, and never report an ordinary safe failure.
7. Synchronize the actual native item and inventory after successful mutation and debit.

Do not implement augmentation using `createItem` with an affix array containing zero placeholders: current host rare generation fills unspecified slots, as observed by Sanctuary of Exile on D2R build 93847. Do not subtract unwanted aggregate stats from six-affix generated items.

## Required native acceptance cases

* Rare ring with 3 affixes: 3→4, preserve every old ID and roll.
* Rare ring with 5 affixes: 5→6; full 6→fail without debit.
* Explicit prefix/suffix, wrong side, wrong item type, duplicate group, low item level.
* Overlapping stat IDs: additive and nonadditive contribution cases; no old-roll changes.
* Optional no-payment call; one unit from stack; last unit of stack; insufficient payment.
* Stale target/payment handles; target == payment; non-owner; client authority rejection.
* Fault after new affix applied, before payment debit and during debit; rollback or session quarantine.
* Save/reload persistence and same-game inventory/tooltip synchronization.

The declaration and contract tests on this branch do **not** prove native host behavior.
