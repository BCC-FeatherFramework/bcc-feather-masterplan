# Proposal: Feather Economy operated accounts (v1.1)

**Status:** proposed, for review by the feather-economy owners
**Amends:** [Feather Economy Master Plan](https://github.com/FeatherFramework/feather-framework-docs/blob/main/feather-economy/Feather_Economy_Master_Plan.md) §5.2, §7, §13
**Code:** [BCC-FeatherFramework/feather-economy `proposal/operated-accounts`](https://github.com/BCC-FeatherFramework/feather-economy/compare/main...proposal/operated-accounts)
**Named consumers:** bcc-banks, bcc-society
**Date:** 2026-10-11

## 1. Problem

The master plan keeps organization treasuries receive-only in v1:

> Wallet-to-treasury transfers require a `shop.purchase` reason and UUID `shop_order`
> reference. Generic treasury credits, withdrawals, direct issuance, and destruction
> are rejected. (§5.2)

and defers bank accounts, general treasury spending, payroll, Society management and
taxes (§13), each until it has "a named consumer and separate contract review".

Two BCC resources need exactly those deferred features:

- **bcc-society:** a society ledger must take deposits and bills, and pay withdrawals,
  wages and taxes.
- **bcc-banks:** characters hold bank accounts; branches pay out loans, checks and
  gold, and take repayments and fees.

Keeping balances in the consumer, or creating and destroying currency for every
movement, would break stronger rules (Economy owns all balances; supply is
policy-gated). The deferred features need to be in Economy.

## 2. Proposal

Add **operated accounts**: an account bound permanently to one operating resource,
which alone may move it, and only with a closed set of reason codes.

Proposed replacement for the §5.2 paragraph:

> Wallet-to-treasury transfers require a `shop.purchase` reason and UUID `shop_order`
> reference, unless the treasury is an operated account. An operated account is bound
> once, permanently, to one resource in an operator access group; only that resource
> may move it, and only with the reason codes of its kind (`bank.*` for bank reserves,
> `society.*` for operated treasuries). Character bank accounts are moved only by
> bank transactors with `bank.*` reasons. Direct issuance and destruction stay
> policy-gated and limited to `Config.BankSupply` reasons. Every other treasury credit,
> withdrawal, issuance and destruction is rejected.

### 2.1 Off by default

The shipped configuration keeps v1 behaviour: the new access lists are empty or hold
only `feather-economy`, and `Config.BankSupply` is empty. A server enables the
extension by trusting its operator resources. Servers that do not are unaffected.

### 2.2 Contract additions

| Export | Access group | Effect |
|---|---|---|
| `EnsureCharacterBankAccounts({ characterId, accountRef })` | `trustedBankProvisioners` | One `bank` account per currency per character and stable `accountRef` (no balances, types or labels from the caller) |
| `EnsureBankReserve({ organizationId })` | `trustedBankProvisioners` | Binds an active organization's treasuries to the caller as a bank reserve |
| `EnsureOperatedTreasury({ organizationId })` | `trustedTreasuryOperators` | Binds an active organization's treasuries to the caller as an operated treasury |
| `ListAccountEntries({ accountId, limit, offset })` | `trustedReaders` | Journal history; `privilegedHistoryReaders` read any account, others only accounts they serve; unknown and foreign accounts get the same denial |

Capabilities advertise `bankAccounts`, `bankReserves` and `accountHistory`.

### 2.3 Reason codes

| Reason | From | To | Who |
|---|---|---|---|
| `bank.deposit` | acting character's wallet | character bank account | bank transactor |
| `bank.withdraw` | character bank account | acting character's wallet | bank transactor |
| `bank.transfer` | character bank account | character bank account | bank transactor |
| `bank.fee` | character bank account | system sink | bank transactor |
| `bank.loan.repayment`, `bank.check.issue`, `bank.box.fee` | character funds | caller's bank reserve | bank transactor |
| `bank.loan.disbursement`, `bank.check.cash`, `bank.box.refund`, `bank.compensation` | caller's bank reserve | character funds | bank transactor |
| `bank.gold.exchange` | either direction between character funds and the caller's reserve | | bank transactor |
| `society.deposit`, `society.bill` | acting character's wallet | caller's operated treasury | treasury operator |
| `society.withdraw` | caller's operated treasury | acting character's wallet | treasury operator |
| `society.payroll`, `society.refund` | caller's operated treasury | character wallet | treasury operator |
| `society.tax` | caller's operated treasury | system sink | treasury operator |
| `bank.migration.import`, `bank.reserve.funding` (supply) | system source | bank account / caller's reserve | only if listed in `Config.BankSupply` **and** a Core policy grant exists |

"Acting character" is `context.actorCharacterId`. Whether that character may act
(account access level, society rank permission) stays the consumer's rule; Economy
enforces account ownership, binding and shape.

### 2.4 Security properties

- An operated account has exactly one operator, set on first binding and never
  changed. A second resource binding the same organization gets
  `reserve_operator_conflict` (bank reserve) or `treasury_operator_conflict` (operated treasury).
- A treasury operator cannot touch a shop, bank or government treasury; a bank
  cannot drain a society treasury, and the reverse.
- No reason code moves money from one character's wallet unless that character is
  the actor.
- Supply is unchanged in spirit: exact `BankSupply` reasons, plus a Core policy
  decision with `subject.resource` set to the caller. `exemptBankSupply` exists for
  testing only and ships `false`.
- History reads cannot enumerate accounts: an unknown account and a foreign account
  return the same denial.

### 2.5 Persistence

- Migration 005: `account_ref` column, `bank` account type, constraints updated so
  bank accounts need a reference and other types must not have one.
- Migration 006: `economy_bank_reserves (organization_id, operator_resource)`, the
  binding table for both bank reserves and operated treasuries.

Both are additive and keep existing rows and earlier migration checksums.

## 3. Evidence

| Consumer | Result |
|---|---|
| bcc-society 2.0.0 | Live with two players (2026-10-11): deposit 2948 + 1232 = 4180; withdraw 4180 - 1542 = 2638; bills paid 1500 and 100 (declined bill moved nothing); payroll accrued and collected; monthly tax paid once, replay charged nothing, an empty treasury failed with `insufficient_funds` and rolled back its idempotency reservation. Every movement was one transfer. |
| bcc-banks 2.0.0 | Code complete; read-only `BanksReleaseSmokeTest` and the Economy bank gate/contract checks. Live acceptance list in its FEATHER_PORT.md, not yet run. |
| feather-economy | `EconomyBankAccountGateSmokeTest`, `EconomyBankContractSmokeTest`, `EconomyJournalAuditSmokeTest`, `EconomyReleaseContractSmokeTest` |

## 4. Questions for reviewers

1. **Binding table name.** `economy_bank_reserves` also holds operated treasuries.
   Rename it (for example `economy_operated_accounts`) before this lands upstream, while
   no production server has it.
2. **History paging.** `ListAccountEntries` uses `limit`/`offset`. Should it follow a
   cursor convention to match the other list exports?
3. **Permanent binding.** Is a one-time, permanent operator binding acceptable, or is an
   administrative re-binding path needed (for example when a resource is replaced)?
4. **Reason namespaces.** `bank.*` and `society.*` are fixed in Economy. Should operators
   instead register their reason codes, so new operated-account kinds need no Economy
   change?

## 5. Also open in the master plan (not part of this extension)

- E6: a repository test proving no first-party resource writes Economy tables.
- Release gates not yet run: history pagination, migration failure, policy-provider failure.
- Admin integration: a staff money command (`/givemoney`, `/takemoney`) is on the
  BCC-FeatherFramework feather-admin fork; it issues and destroys through Economy as the
  `feather-admin` service with an explicit grant, as §10 describes.
