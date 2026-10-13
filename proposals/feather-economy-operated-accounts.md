# Proposal: Feather Economy operated accounts (v1.1)

**Status:** proposed, for review by the feather-economy owners (revised 2026-10-13: registered reason codes, table renamed)
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

Add **operated accounts**: an organization treasury bound permanently to one
operating resource, which alone may move it. Economy does not know any consumer's
reason codes. It defines a closed set of **movement shapes**; each operator
registers its own reason codes on start and maps each code to the shapes it may use.

Proposed replacement for the §5.2 paragraph:

> Wallet-to-treasury transfers require a `shop.purchase` reason and UUID `shop_order`
> reference, unless the treasury is an operated treasury. An operated treasury is
> bound once, permanently, to one resource in `trustedTreasuryOperators`; only that
> resource may move it, and only with reason codes it registered, in one of the
> movement shapes Economy defines. Character bank accounts are moved only by
> resources in `trustedBankTransactors`, the same way. Direct issuance stays
> policy-gated and limited to `Config.OperatorSupply` reasons. Every other treasury
> credit, withdrawal, issuance and destruction is rejected.

### 2.1 Off by default

The shipped configuration keeps v1 behaviour: the new access lists are empty or hold
only `feather-economy`, and `Config.OperatorSupply` is empty. A server enables the
extension by trusting its operator resources. Servers that do not are unaffected.

### 2.2 Contract additions

| Export | Access group | Effect |
|---|---|---|
| `EnsureCharacterBankAccounts({ characterId, accountRef })` | `trustedBankProvisioners` | One `bank` account per currency per character and stable `accountRef` (no balances, types or labels from the caller) |
| `EnsureOperatedTreasury({ organizationId })` | `trustedTreasuryOperators` | Binds an active organization's treasuries to the caller, permanently |
| `RegisterOperatedReasons({ reasons })` | `trustedTreasuryOperators` or `trustedBankTransactors` | Replaces the caller's reason codes and their shapes |
| `ListAccountEntries({ accountId, limit, offset })` | `trustedReaders` | Journal history; `privilegedHistoryReaders` read any account, others only accounts they serve; unknown and foreign accounts get the same denial |

Capabilities advertise `bankAccounts`, `operatedTreasuries` and `accountHistory`.

### 2.3 Movement shapes

"Operated" is a treasury bound to the caller; "actor wallet" is the wallet of
`context.actorCharacterId`.

| Shape | From | To | Caller needs |
|---|---|---|---|
| `actor_wallet_to_operated` | actor wallet | operated treasury | `trustedTreasuryOperators` |
| `operated_to_actor_wallet` | operated treasury | actor wallet | `trustedTreasuryOperators` |
| `operated_to_wallet` | operated treasury | any character wallet | `trustedTreasuryOperators` |
| `operated_to_sink` | operated treasury | system sink | `trustedTreasuryOperators` |
| `bank_to_operated` | character bank account | operated treasury | both groups |
| `operated_to_bank` | operated treasury | character bank account | both groups |
| `actor_wallet_to_bank` | actor wallet | character bank account | `trustedBankTransactors` |
| `bank_to_actor_wallet` | character bank account | actor wallet | `trustedBankTransactors` |
| `bank_to_bank` | character bank account | character bank account | `trustedBankTransactors` |
| `bank_to_sink` | character bank account | system sink | `trustedBankTransactors` |
| `issue_to_bank` (supply) | system source | character bank account | `trustedBankTransactors` + `OperatorSupply` + Core policy |
| `issue_to_operated` (supply) | system source | operated treasury | `trustedTreasuryOperators` + `OperatorSupply` + Core policy |

No shape destroys currency. Whether the acting character may act (account access
level, society rank permission) stays the consumer's rule; Economy enforces account
ownership, binding and shape.

The named consumers register:

| Consumer | Codes → shapes |
|---|---|
| bcc-society | `society.deposit`, `society.bill` → `actor_wallet_to_operated`; `society.withdraw` → `operated_to_actor_wallet`; `society.payroll` → `operated_to_wallet`; `society.tax` → `operated_to_sink` |
| bcc-banks | `bank.deposit` → `actor_wallet_to_bank`; `bank.withdraw` → `bank_to_actor_wallet`; `bank.transfer` → `bank_to_bank`; `bank.fee` → `bank_to_sink`; loan repayment, check issue, box fee → into the reserve from a wallet or bank account; loan disbursement, check cash, box refund, compensation → out of the reserve to a wallet or bank account; `bank.gold.exchange` → both; `bank.migration.import` → `issue_to_bank`; `bank.reserve.funding` → `issue_to_operated` |

### 2.4 Security properties

- An operated treasury has exactly one operator, set on first binding and never
  changed. A second resource binding the same organization gets `operator_conflict`.
- Every treasury shape requires the treasury to be bound to the caller: an operator
  cannot touch a shop or government treasury, or another operator's treasury.
- A reason code belongs to one resource at a time (`reason_conflict`), so a code in
  the journal always has one meaning. `shop.` and `economy.` are reserved. A
  stopped resource's codes are released.
- No shape moves money from a character's wallet unless that character is the actor.
- Supply is unchanged in spirit: exact `OperatorSupply` reasons, plus a Core policy
  decision with `subject.resource` set to the caller. There is no exemption switch.
- History reads cannot enumerate accounts: an unknown account and a foreign account
  return the same denial.

### 2.5 Persistence

- Migration 005: `account_ref` column, `bank` account type, constraints updated so
  bank accounts need a reference and other types must not have one.
- Migration 006: `economy_treasury_operators (organization_id, operator_resource)`,
  the binding table.
- Reason registrations are in memory: operators register on every start.

Both migrations are additive and keep existing rows and earlier migration checksums.

## 3. Evidence

| Consumer | Result |
|---|---|
| bcc-society 2.0.0 | Live with two players (2026-10-11): deposit 2948 + 1232 = 4180; withdraw 4180 - 1542 = 2638; bills paid 1500 and 100 (declined bill moved nothing); payroll accrued and collected; monthly tax paid once, replay charged nothing, an empty treasury failed with `insufficient_funds` and rolled back its idempotency reservation. Every movement was one transfer. |
| bcc-banks 2.0.0 | Code complete; read-only `BanksReleaseSmokeTest` and the Economy bank gate/contract checks. Live acceptance list in its FEATHER_PORT.md, not yet run. |
| feather-economy | `EconomyBankAccountGateSmokeTest` (test-only operators, no database writes), `EconomyBankContractSmokeTest`, `EconomyJournalAuditSmokeTest`, `EconomyReleaseContractSmokeTest` |

## 4. Questions for reviewers

1. **History paging.** `ListAccountEntries` uses `limit`/`offset`. Should it follow a
   cursor convention to match the other list exports?
2. **Permanent binding.** Is a one-time, permanent operator binding acceptable, or is an
   administrative re-binding path needed (for example when a resource is replaced)?
   Reason codes are already released when a resource stops; treasury bindings are not.
3. **Registration lifetime.** Registrations are in memory and repeated on every start.
   Should they be persisted instead, so a code keeps one owner across restarts?

Resolved in this revision: the binding table is named `economy_treasury_operators`
(it held operated treasuries under the name `economy_bank_reserves`), and operators
register their own reason codes, so a new kind of operated account needs no Economy
change.

## 5. Also open in the master plan (not part of this extension)

- E6: a repository test proving no first-party resource writes Economy tables.
- Release gates not yet run: history pagination, migration failure, policy-provider failure.
- Admin integration: a staff money command (`/givemoney`, `/takemoney`) is on the
  BCC-FeatherFramework feather-admin fork; it issues and destroys through Economy as the
  `feather-admin` service with an explicit grant, as §10 describes.
