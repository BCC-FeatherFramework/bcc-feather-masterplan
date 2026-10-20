# BCC on Feather — Master Plan

**Status:** active · **Last updated:** 2026-10-12
**Scope:** porting BryceCanyonCounty (BCC) resources to the Feather Framework (Contract 1)

---

## 1. Where the work lives

All Feather porting work for BCC resources lives in **one place**:
**https://github.com/BCC-FeatherFramework**

| What | Where |
|---|---|
| Feather resources created on the dev server | `BCC-FeatherFramework/<resource>` (see below) |
| Ported resource code | `BCC-FeatherFramework/<resource>`, one repository per resource, same name as the resource |
| Port notes for one resource | Inside that repository (`FEATHER_PORT.md`, or `FEATHER_REWRITE_PLAN.md` for a rewrite) |
| This master plan, standards and status | `BCC-FeatherFramework/bcc-feather-masterplan` |
| Issues, pull requests, releases | The resource's repository in this organization |

Do **not** push BCC Feather ports to BryceCanyonCounty, FeatherFramework, personal
accounts or any other organization. The upstream BCC repositories stay the source
of the original gameplay; this organization holds the Feather versions.

**New Feather resources created on the development server also live here.**
A Feather resource that was built on this server (not taken from
`FeatherFramework`) gets its own repository in this organization, named like the
resource: currently [feather-assets](https://github.com/BCC-FeatherFramework/feather-assets),
[feather-contracts](https://github.com/BCC-FeatherFramework/feather-contracts),
[feather-property](https://github.com/BCC-FeatherFramework/feather-property) and
[feather-licenses](https://github.com/BCC-FeatherFramework/feather-licenses).
Push it here as soon as it is created, before any BCC port depends on it.

**BryceCanyonCounty scripts are forked here too.** 56 `bcc-*` repositories are
GitHub forks of [BryceCanyonCounty](https://github.com/BryceCanyonCounty). The 10 ported
repositories in §3 are not forks; their `main` was rebuilt on top of the original BCC
history, so `git diff upstream/main...main` shows only the Feather port. Nothing is
pushed to BryceCanyonCounty. Not forkable: `bcc-bot`, `bcc-mail`, `bcc-servermenu`,
`bcc-storage`, `bcc-trapper` (forking disabled by the owner) and `bcc-txadmin-recipe` (empty).

**FeatherFramework resources are forked here for traceability.** Every public
`feather-*` repository of [FeatherFramework](https://github.com/FeatherFramework) is
forked into this organization under the same name. Each fork has a `dev-server`
branch holding the development server's code on top of the installed release, so
`compare/<tag>...dev-server` shows exactly what the server changed. Nothing is
pushed to FeatherFramework from here: changes are proposed upstream by hand, and
the local `upstream` remotes have pushing disabled.

---

## 2. Porting standard

Every port follows these rules. They come from the BCC Society rewrite plan and
the Feather master plans.

### 2.1 Dependencies

| Need | Use |
|---|---|
| Database | `feather-mysql` (`@feather-mysql/lib/DB.lua`) |
| Sessions, RPC, locale, events, policy | `feather-core` |
| Character identity | `feather-character` (through Core's `character-profile` provider) |
| Money | `feather-economy` |
| Items and storage | `feather-inventory` |
| Jobs, grades, duty | `feather-jobs` |
| Business, government identity | `feather-organizations` |
| Staff permissions | `feather-authority` |
| Menus | `feather-menu-v2` |
| Prompts, blips, entities | `feather-toolkit` |
| Notifications | `feather-notify` (through Core) |

**Forbidden** in the manifest and the code: `vorp_*`, `oxmysql` / `MySQL.*`,
`bcc-utils` / `BccUtils`, `feather-menu` (v1), `feather-roles` (retired), and
`character.job` / `setJob` / `"off"` duty prefixes.

### 2.2 Ownership

The port keeps BCC's gameplay: menus, NPCs, prompts, configuration and
translations. Shared state moves to the Feather domain that owns it. A BCC
resource never writes another resource's tables.

### 2.3 Required in every repository

1. `fxmanifest.lua` listing only allowed dependencies.
2. `FEATHER_PORT.md` with: status, what moved where, the trust entries and service
   policy grants it needs (with copy-paste examples), console tests, and remaining work.
3. A read-only release smoke test (for example `StablesReleaseSmokeTest`) and, where
   it mutates data, a DevMode-only live test.
4. `Config.DevMode = false` in the committed config.
5. The original `LICENSE` (GPL-3.0) kept, and BryceCanyonCounty credited in the README.
6. No secrets: webhook URLs, tokens and passwords stay empty in config.
7. Default branch `main`.
8. If `.github/workflows/discord.yml` exists, it links to the
   `BCC-FeatherFramework/<resource>` repository and reads `secrets.WEBHOOK_URL`.

### 2.4 Definition of done

- No forbidden dependency left (a `grep` for each returns nothing).
- The release smoke test passes on a clean start and after a restart.
- The live flows listed in `FEATHER_PORT.md` passed on the development server.
- The trust and policy entries it needs are documented and merged upstream (§5).

---

## 3. Repository map and status

Counts are files still using a forbidden dependency, measured on 2026-10-06.

| Repository | Visibility | Port state | Legacy left | Next step |
|---|---|---|---|---|
| [bcc-society](https://github.com/BCC-FeatherFramework/bcc-society) | private (ready for public) | **Done: v2.0.0** (2026-10-11). Every flow passed live with two players: jobs, duty, ledger, bills, payroll, creation-day taxes, storage, dissolve | none | Make public; upstream the feather-economy treasury code (§5) |
| [bcc-banks](https://github.com/BCC-FeatherFramework/bcc-banks) | public | **3.0.0** (2026-10-18): Economy ledger and reserves, Organizations, Authority staff, banker job; loans in installments through feather-contracts; Branch Reserve page; bank actions in feather-audit (`bank.action.recorded`); `BanksAcceptanceTest` 48/48 offline (live items 2-8, 10), item 9 passed in game; no migration needed | none | Release: `devMode = false`, `BanksReleaseSmokeTest` |
| [bcc-shops](https://github.com/BCC-FeatherFramework/bcc-shops) | public | Economy preparation; not ready for live payments | none | Economy trust for bcc-shops; live purchase tests |
| [bcc-stables](https://github.com/BCC-FeatherFramework/bcc-stables) | private | Ported; release test 7/7 on 2026-10-03 | none | Needs `feather-assets` (§5); test remaining flows |
| [bcc-weapons-cosmetics](https://github.com/BCC-FeatherFramework/bcc-weapons-cosmetics) | private | Implemented, offline-tested only | none | Live test; payment through Economy |
| [bcc-craft](https://github.com/BCC-FeatherFramework/bcc-craft) | private | Ported 2.0.0 (2026-10-08); not live-tested | none | Live test; `CraftReleaseSmokeTest` lists missing items and unmapped weapons |
| [bcc-doorlocks](https://github.com/BCC-FeatherFramework/bcc-doorlocks) | private | Ported 2.0.0 (2026-10-08); not live-tested | none | Live test; lockpicking needs a minigame resource |
| [bcc-housing](https://github.com/BCC-FeatherFramework/bcc-housing) | private | 3.0.0 code complete (2026-10-12): feather-property ownership, Economy (no ledger), feather-inventory storage/deeds, doorlocks API, feather-routing interiors; started, `HousingReleaseSmokeTest` 13/13 (2026-10-12) | none | Live acceptance (FEATHER_PORT.md) |
| [bcc-zonenotify](https://github.com/BCC-FeatherFramework/bcc-zonenotify) | private | Clean (bcc-utils version check removed 2026-10-08); started in `server.cfg` | none | Live test zone notifications |
| [bcc-documents](https://github.com/BCC-FeatherFramework/bcc-documents) | private | **Ported 1.0.0** (2026-10-18): licences in feather-licenses (first consumer, L3), fees through Economy as durable orders, items in feather-inventory, `doj` weapon licence by command; release 11/11, offline flow 12/12 | none | Live test with two players |
| [bcc-legendaries](https://github.com/BCC-FeatherFramework/bcc-legendaries) | private | **Ported 4.0.0** (2026-10-18): hunting licence checked with feather-licenses (L4), fees through Economy, rewards through Inventory, trust per character UUID; offline flow 8/8, release 7/8 | none | Define the 89 reward items in feather-inventory; live hunt |
| [bcc-chat](https://github.com/BCC-FeatherFramework/bcc-chat) | public | Feather Chat Contract 1 adapter (2026-09-12) | uses `oxmysql`, `feather-roles` | Decide: retire in favour of `feather-chat`, or update |

---

## 4. Delivery order

### Phase B0: Finish what is running (now)
1. ~~**bcc-society:** two-player live test and the money flows.~~ Done, v2.0.0 (2026-10-11).
2. ~~**bcc-banks:** live tests and balance migration.~~ 3.0.0 (2026-10-18): live acceptance passed (48/48 offline, item 9 in game).
3. **bcc-shops:** Economy trust, then live purchases.
4. **bcc-stables:** get `feather-assets` installed, then the remaining flows.

### Phase B1: Job consumers (Jobs plan J5)
5. **bcc-doorlocks**, then **bcc-craft**: port, and move every `character.job`
   check to `feather-jobs` `CheckAssignment`.

### Phase B2: Ownership domains
6. **bcc-housing** after `feather-property` and `feather-access` exist.
7. Stables, wagons and boats as `feather-assets` consumers.

### Phase B3: Smaller resources
8. **bcc-weapons-cosmetics** live test and payment.
9. **bcc-zonenotify** cleanup.
10. **bcc-chat** decision.

New BCC resources join at the phase whose Feather domains they need.

---

## 5. Upstream dependencies

Changes a BCC port needs in Feather resources. Each is proposed to the owning
`FeatherFramework` repository; none is copied into this organization.

| Feather resource | Needed by | Change | State |
|---|---|---|---|
| `feather-jobs` | society, doorlocks, craft, stables, banks, housing | 0.2.0 on the BCC fork (2026-10-17): references on mutations, integrity check, Contract 1 (CONTRACT.md) proposed for freezing; release 11/11, lifecycle 26/26; bcc-doorlocks on-duty door verified as second consumer. Trusts `bcc-society` upstream; reader entries added locally | Fork changes to propose upstream + local config |
| `feather-authority` | doorlocks | `bcc-doorlocks` in the five access lists (staff.doorlocks.manage) | Local config on dev server |
| `feather-weapons` | craft | `bcc-craft` in `Config.Issuance.trustedResources` (purpose `crafting`) | Local config on dev server |
| `feather-economy` | craft | `bcc-craft` in readers, provisioners, transactors (craft book prices) | Local config on dev server |
| `feather-inventory` | society, stables (storage) | Slot moves dropped the move guard's reason, so moving an equipped weapon showed "The inventory changed; try again."; fixed on fork `dev-server` (reason passed through as code `denied`) | **Upstream fix** to send by hand |
| `feather-inventory` | craft, stables | `ItemsAPI.AdjustCondition` adds a number to a result table and errors; craft uses Get/SetCondition, stables tool wear is affected | **Upstream bug** to report |
| `feather-organizations` | society, banks, shops | `bcc-society` in creators, mutators, readers; `feather-jobs` reader | Local config on dev server |
| `feather-authority` | society, banks | `bcc-society` in the five access lists | Local config on dev server |
| `feather-admin` | society, banks | Service policy for `feather-jobs` and `feather-organizations` | Local config on dev server |
| `feather-economy` | banks | Bank accounts, reserves, `bank.*` reasons, `BankSupply` | **Local code**, outside the Economy master plan (deferred items); needs review and an upstream PR |
| `feather-economy` | society | `EnsureOperatedTreasury`, `society.*` reasons | **Local code**, outside the Economy master plan; needs review and an upstream PR |
| `feather-economy` | stables, shops | `bcc-stables`, `bcc-shops` in readers, provisioners, transactors (not suppliers) | Local config on dev server |
| `feather-admin` | testing (all money flows) | Staff `/givemoney` / `/takemoney` (owner capability `economy.<currency>.add/.remove`, console always); Economy supply as service principal `feather-admin` (`economy.currency.issue/destroy` in `service_policy.lua`); audited. Fork `dev-server` `eaa200f` | **Local code**; propose upstream |
| `feather-assets` | stables (wagons, boats) | The resource itself | Created on the dev server; [BCC-FeatherFramework/feather-assets](https://github.com/BCC-FeatherFramework/feather-assets): 0.2.0, master plan A0–A5 done; Contract 1 (CONTRACT.md) proposed for freezing; `horse`/`wagon`/`boat` enabled; delegation epoch for feather-access; release 10/10, lifecycle 23/23 per type, bcc-stables flow 9/9 |
| `feather-property` | housing | Owners and tenants of houses and hotel rooms (Property master plan P1–P3) | Created on the dev server; [BCC-FeatherFramework/feather-property](https://github.com/BCC-FeatherFramework/feather-property): P0 done, P1–P2 accepted (release 10/10, lifecycle 33/33) |
| `feather-contracts` | banks (loans) | Loan obligations | Created on the dev server; in [BCC-FeatherFramework/feather-contracts](https://github.com/BCC-FeatherFramework/feather-contracts); not live-accepted |
| `feather-licenses` | documents, legendaries | Licences, certifications, permits (Licenses master plan L0–L4) | Created on the dev server 2026-10-18; private [BCC-FeatherFramework/feather-licenses](https://github.com/BCC-FeatherFramework/feather-licenses): 0.1.0, release 10/10, lifecycle 35/35, CONTRACT.md draft |
| `feather-audit` | banks | Audit producer kit | Upstream, A2 not live-tested |
| `feather-justice` | law | Legal records: cases, charges, warrants, citations, custody, rulings, sentences, evidence, sealing (Justice master plan JUS0–JUS7) | Created on the dev server; private [BCC-FeatherFramework/feather-justice](https://github.com/BCC-FeatherFramework/feather-justice): **0.3.0** (2026-10-20) follows the master plan as written (plan tables, all 48 Contract 1 operation names, statuses, events, smoke tests); real-MariaDB suite 160/160; BCC Law must adopt the 0.3.0 calls (`LAW_INTEGRATION.md` §4) before Justice is restarted live |

Economy deviations fixed on 2026-10-08 (still local, still need the upstream PR):
- Bank supply now needs a Core policy decision: `exemptBankSupply = false`, Economy
  sends `subject.resource`, and feather-admin grants
  `['feather-economy']['bcc-banks']['economy.currency.issue']`. Admin's disabled
  prefixes were narrowed to the player money actions (`economy.dollars.`, `.gold.`,
  `.tokens.`, `.xp.`) so that service grant can apply.
- `ListAccountEntries` is limited: `privilegedHistoryReaders` (Economy, Admin) read
  any account; bank transactors read bank accounts; an operator reads only the
  treasury bound to it.

The Economy extension is written up as a proposal for the Economy owners:
[proposals/feather-economy-operated-accounts.md](proposals/feather-economy-operated-accounts.md)
(code on the fork branch `proposal/operated-accounts`, off by default).

Still outside the Economy master plan and awaiting review: bank accounts and
reserves, operated treasuries (`society.*`), bank product reason codes in Economy,
and issuing into bank accounts.

---

## 6. Working in this organization

- **Repositories:** one per resource, named exactly like the resource folder.
- **Branches:** work on a branch, open a pull request to `main`.
- **Commits:** describe the port step (for example "Move duty to feather-jobs").
- **Releases:** tag `vX.Y.Z` on `main`; the Discord workflow announces it when the
  repository has a `WEBHOOK_URL` secret.
- **Visibility:** new repositories start private; make one public when its port
  meets the definition of done (§2.4).
- **This plan:** update §3 and §5 in the same pull request that changes a port's
  state.
