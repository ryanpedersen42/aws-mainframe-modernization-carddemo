# Ledger Modernization — Migration Guide

Source inventory is taken from the CardDemo legacy tree (`app/cbl`, `app/jcl`, `app/cpy`, `app/cpy-bms`, optional modules under `app/app-*`). Business language below uses CardDemo domain names (account master, card master, customer master, transaction master, disclosure group, transaction category balance, cross-reference) rather than DDNAMES alone.

---

## 1. Program inventory

### Batch programs (core)

| Program | Job / entry | One-line purpose | Copybooks | Files (DD / dataset role) |
|:--------|:------------|:-----------------|:----------|:--------------------------|
| **CBACT01C** | utility / sample | Reads the account master sequentially and writes fixed, array, and variable-block extract files. | `CVACT01Y`, `CODATECN` | Input: `ACCTFILE` (account master). Output: `OUTFILE`, `ARRYFILE`, `VBRCFILE`. |
| **CBACT02C** | `READCARD` | Reads and prints card master records. | `CVACT02Y` | Input: `CARDFILE` (card master). |
| **CBACT03C** | `READXREF` | Reads and prints account–card–customer cross-reference records. | `CVACT03Y` | Input: `XREFFILE` (cross-reference). |
| **CBACT04C** | `INTCALC` | Calculates monthly interest from transaction category balances, writes system interest transactions, updates account master balances and cycle totals. | `CVTRA01Y`, `CVACT03Y`, `CVTRA02Y`, `CVACT01Y`, `CVTRA05Y` | Input: `TCATBALF`, `XREFFILE`, `ACCTFILE`, `DISCGRP`. Output: `TRANSACT` → GDG `SYSTRAN(+1)` (system transactions for later merge into transaction master). |
| **CBCUS01C** | `READCUST` | Reads and prints customer master records. | `CVCUS01Y` | Input: `CUSTFILE` (customer master). |
| **CBTRN01C** | (alternate poster) | Posts daily transactions with customer/card/account validation (earlier posting variant). | `CVTRA06Y`, `CVCUS01Y`, `CVACT03Y`, `CVACT02Y`, `CVACT01Y`, `CVTRA05Y` | `DALYTRAN`, `CUSTFILE`, `XREFFILE`, `CARDFILE`, `ACCTFILE`, `TRANFILE`. |
| **CBTRN02C** | `POSTTRAN` | Posts daily transactions: validates, updates account master and transaction category balances, writes accepted rows to transaction master, writes rejects. | `CVTRA06Y`, `CVTRA05Y`, `CVACT03Y`, `CVACT01Y`, `CVTRA01Y` | Input: `DALYTRAN`, `XREFFILE`. I-O: `TRANFILE` (transaction master), `ACCTFILE`, `TCATBALF`. Output: `DALYREJS`. |
| **CBTRN03C** | `TRANREPT` (from CICS) | Prints transaction detail reports for a date range using type/category lookups. | `CVTRA05Y`, `CVACT03Y`, `CVTRA03Y`, `CVTRA04Y`, `CVTRA07Y` | `TRANFILE`, `CARDXREF`, `TRANTYPE`, `TRANCATG`, `TRANREPT`, `DATEPARM`. |
| **CBSTM03A** | `CREASTMT` | Produces account statements (plain text and HTML) from transaction data; calls CBSTM03B for I/O. | `COSTM01`, `CVACT03Y`, `CUSTREC`, `CVACT01Y` | `STMTFILE`, `HTMLFILE`; via CBSTM03B: `TRNXFILE`, `XREFFILE`, `CUSTFILE`, `ACCTFILE`. |
| **CBSTM03B** | called by CBSTM03A | File open/read/write helper for statement generation. | (linkage from CBSTM03A) | `TRNXFILE`, `XREFFILE`, `CUSTFILE`, `ACCTFILE`. |
| **CBEXPORT** | `CBEXPORT` | Exports customer, account, cross-reference, transaction, and card masters to a pipe-delimited migration file. | `CVCUS01Y`, `CVACT01Y`, `CVACT03Y`, `CVTRA05Y`, `CVACT02Y`, `CVEXPORT` | Inputs: `CUSTFILE`, `ACCTFILE`, `XREFFILE`, `TRANSACT`, `CARDFILE`. Output: `EXPFILE`. |
| **CBIMPORT** | `CBIMPORT` | Imports a branch-migration export file back into split master outputs (with error file). | `CVCUS01Y`, `CVACT01Y`, `CVACT03Y`, `CVTRA05Y`, `CVACT02Y`, `CVEXPORT` | Input: `EXPFILE`. Outputs: `CUSTOUT`, `ACCTOUT`, `XREFOUT`, `TRNXOUT`, `CARDOUT`, `ERROUT`. |
| **COBSWAIT** | `WAITSTEP` | Timed wait step used in batch scheduling. | — | Parameterized wait (assembler `MVSWAIT` support). |
| **CSUTLDTC** | called utility | Date validation / conversion helper (CEEDAYS). | — | Linkage-only utility. |

### Online (CICS) programs (core)

| Program | Tran | BMS | One-line purpose | Copybooks (domain + map) | Files (CICS) |
|:--------|:-----|:----|:-----------------|:-------------------------|:-------------|
| **COSGN00C** | CC00 | COSGN00 | Sign-on: authenticate against user security and route by role. | `COCOM01Y`, `COSGN00`, `COTTL01Y`, `CSDAT01Y`, `CSMSG01Y`, `CSUSR01Y` | `USRSEC` |
| **COMEN01C** | CM00 | COMEN01 | Regular-user main menu. | `COCOM01Y`, `COMEN02Y`, `COMEN01`, `COTTL01Y`, `CSDAT01Y`, `CSMSG01Y`, `CSUSR01Y` | `USRSEC` |
| **COADM01C** | CA00 | COADM01 | Admin menu. | `COCOM01Y`, `COADM02Y`, `COADM01`, `COTTL01Y`, `CSDAT01Y`, `CSMSG01Y`, `CSUSR01Y` | `USRSEC` |
| **COACTVWC** | CAVW | COACTVW | View account master joined with customer master and cards via cross-reference. | `CVCRD01Y`, `COCOM01Y`, `COACTVW`, `CVACT01Y`, `CVACT02Y`, `CVACT03Y`, `CVCUS01Y`, … | `ACCTDAT`, `CARDDAT`, `CUSTDAT`, `CARDAIX`, `CXACAIX` |
| **COACTUPC** | CAUP | COACTUP | Update account master and related customer master fields. | `CVCRD01Y`, `COACTUP`, `CVACT01Y`, `CVACT03Y`, `CVCUS01Y`, `CSUTLDWY`, … | `ACCTDAT`, `CUSTDAT`, `CARDDAT`, `CARDAIX`, `CXACAIX` |
| **COCRDLIC** | CCLI | COCRDLI | List cards for an account. | `CVCRD01Y`, `COCRDLI`, `CVACT02Y`, … | `CARDDAT`, `CARDAIX` |
| **COCRDSLC** | CCDL | COCRDSL | View card detail (and customer). | `CVCRD01Y`, `COCRDSL`, `CVACT02Y`, `CVCUS01Y`, … | card / customer files via CICS |
| **COCRDUPC** | CCUP | COCRDUP | Update card master fields. | `CVCRD01Y`, `COCRDUP`, `CVACT02Y`, `CVCUS01Y`, … | `CARDDAT`, `CARDAIX` |
| **COTRN00C** | CT00 | COTRN00 | List transactions from the transaction master. | `COCOM01Y`, `COTRN00`, `CVTRA05Y`, … | `TRANSACT` |
| **COTRN01C** | CT01 | COTRN01 | View one transaction master record. | `COCOM01Y`, `COTRN01`, `CVTRA05Y`, … | `TRANSACT` |
| **COTRN02C** | CT02 | COTRN02 | Add a transaction to the transaction master (updates account when applicable). | `COCOM01Y`, `COTRN02`, `CVTRA05Y`, `CVACT01Y`, `CVACT03Y`, … | `TRANSACT`, account / cross-reference |
| **CORPT00C** | CR00 | CORPT00 | Request transaction reports (submits batch `TRANREPT`). | `COCOM01Y`, `CORPT00`, `CVTRA05Y`, … | `TRANSACT` (+ batch submit) |
| **COBIL00C** | CB00 | COBIL00 | Bill payment: pay account balance in full and write a payment transaction to the transaction master. | `COCOM01Y`, `COBIL00`, `CVACT01Y`, `CVACT03Y`, `CVTRA05Y`, … | `TRANSACT`, `ACCTDAT`, `CXACAIX` |
| **COUSR00C** | CU00 | COUSR00 | List users from user security. | `COCOM01Y`, `COUSR00`, `CSUSR01Y`, … | `USRSEC` |
| **COUSR01C** | CU01 | COUSR01 | Add a user to user security. | `COCOM01Y`, `COUSR01`, `CSUSR01Y`, … | `USRSEC` |
| **COUSR02C** | CU02 | COUSR02 | Update a user in user security. | `COCOM01Y`, `COUSR02`, `CSUSR01Y`, … | `USRSEC` |
| **COUSR03C** | CU03 | COUSR03 | Delete a user from user security. | `COCOM01Y`, `COUSR03`, `CSUSR01Y`, … | `USRSEC` |

### Optional modules (inventory summary)

| Area | Programs | Notes |
|:-----|:---------|:------|
| Transaction type (Db2) | `COTRTLIC`, `COTRTUPC`, `COBTUPDT` | Online list/add/edit of transaction types; batch maintain (`MNTTRDB2`). |
| Pending authorizations (IMS/Db2/MQ) | `COPAUS0C`, `COPAUS1C`, `COPAUA0C`, `CBPAUP0C`, load/unload helpers | MQ-triggered auth, IMS pending store, Db2 audit, batch purge. |
| MQ account/date inquiry | `COACCT01`, `CODATE01` | Request/response over MQ against account master / system date. |

### Fee-quote screen (legacy status)

There is **no** BMS map or CICS program named “fee-quote” in this tree. Related legacy facts:

- `INTCALC` JCL states the step should “compute interest and fees.”
- `CBACT04C` performs `1400-COMPUTE-FEES` after interest, but that paragraph is a stub (`To be implemented`, lines 518–520).
- Closest existing online money screen is **Bill Payment** (`COBIL00C` / CB00), which quotes the current balance and posts a payment to the transaction master—not a fee schedule quote.

Slice 2 therefore targets a **new** fee-quote online surface in the Java ledger service, fed by disclosure group rates (and eventually the unfinished fee rules).

---

## 2. Proposed service boundaries (Java target)

Keep services aligned with existing batch/online seams so each slice can cut over with clear ownership of masters and jobs.

| Target service | Owns | Migrates from | Notes |
|:---------------|:-----|:--------------|:------|
| **Interest & fees (batch)** | Interest calculation, future fee assessment, system transaction generation, cycle reset on account master | `CBACT04C` / `INTCALC`; stub `1400-COMPUTE-FEES` | Slice 1. Reads transaction category balances + disclosure groups; writes system transactions; updates account master. |
| **Fee-quote (online)** | Quote interest/fees for an account (and optional category) before posting | New UI/API; rates from disclosure group; related UX to bill payment display patterns | Slice 2. No legacy screen to port line-for-line. |
| **Transaction posting (batch)** | Daily post, rejects, transaction category balance maintenance, transaction master writes | `CBTRN02C` / `POSTTRAN` (`CBTRN01C` as legacy alternate) | Upstream of interest. |
| **Account & customer online** | Account view/update, customer fields | `COACTVWC`, `COACTUPC` | Account master + customer master + cross-reference. |
| **Card online** | Card list / view / update | `COCRDLIC`, `COCRDSLC`, `COCRDUPC` | Card master. |
| **Transaction online** | List / view / add | `COTRN00C`, `COTRN01C`, `COTRN02C` | Transaction master. |
| **Payments online** | Bill pay | `COBIL00C` | Updates account master + transaction master. |
| **Statements & reports** | Statement PDF/HTML, transaction reports | `CBSTM03A`/`B`, `CBTRN03C`, `CORPT00C` | Downstream of posting + interest merge (`COMBTRAN`). |
| **Identity / admin** | Sign-on, menus, user admin | `COSGN00C`, `COMEN01C`, `COADM01C`, `COUSR0xC` | User security file. |
| **Reference data** | Disclosure groups, transaction types/categories | `DISCGRP`, `TRANTYPE`, `TRANCATG` (+ optional Db2 module) | Shared read models for posting, interest, fee-quote. |
| **Branch migration I/O** | Export/import masters | `CBEXPORT`, `CBIMPORT` | Operational, not core ledger path. |
| **Optional: authorizations** | Pending auth, MQ | IMS/Db2/MQ module | Keep separate bounded context. |

Shared persistence recommendation: one logical ledger database with tables for account master, customer master, card master, cross-reference, transaction master, transaction category balance, and disclosure group—accessed through the service APIs above rather than cross-writing VSAM from multiple apps.

---

## 3. Migration order (with justification)

| Slice | Scope | Justification |
|:------|:------|:--------------|
| **1** | **CBACT04C — interest calculation** | Self-contained batch with a closed formula, explicit inputs/outputs, and high business value. Depends on posted transaction category balances and disclosure groups already produced by `POSTTRAN`, so it can run against migrated reference + balance data without rewriting all online screens first. Proves decimal money handling, system transaction generation, and account master updates end-to-end. |
| **2** | **Fee-quote screen (online)** | Builds on Slice 1’s disclosure group rate lookup and the unfinished fee hook (`1400-COMPUTE-FEES`). Delivers an early interactive product surface while fee *assessment* batch rules are still being defined. Does not block on full posting rewrite. |
| **3** | Transaction posting (`CBTRN02C`) | Supplies transaction category balances and transaction master rows that interest and statements require; largest correctness surface (reject codes, credit limit, cycle credit/debit). |
| **4** | Account / customer / card online | Day-to-day servicing; read/write account master and related masters used by posting and interest. |
| **5** | Transaction online + bill payment | Completes online ledger UX against the transaction master. |
| **6** | Statements, reports, `COMBTRAN` merge path | Downstream consumers of posted + system (interest) transactions. |
| **7** | Identity/admin, export/import, optional modules | Supporting and extension contexts. |

**Batch dependency to preserve during coexistence:** close online files → refresh/reference loads as needed → `POSTTRAN` → `INTCALC` → backup → `COMBTRAN` (merge `SYSTRAN` into transaction master) → statements/reports → reopen files.

---

## 4. Detailed analysis: CBACT04C (interest calculation)

Job: `INTCALC` (`app/jcl/INTCALC.jcl`) runs `PGM=CBACT04C` with `PARM` date prefix (sample `2022071800`) used to build transaction IDs.

### Inputs

| Input | Copybook / record | Role |
|:------|:------------------|:-----|
| Transaction category balance file (`TCATBALF`) | `CVTRA01Y` — account id, transaction type, transaction category, balance | Driving sequential read (indexed file, sequential access). |
| Account master (`ACCTFILE`) | `CVACT01Y` | Random read for group id and balances; rewritten after interest for an account. |
| Cross-reference (`XREFFILE`) | `CVACT03Y` | Random read by account id to obtain card number for the interest transaction. |
| Disclosure group (`DISCGRP`) | `CVTRA02Y` | Random read by (account group, transaction type, transaction category) for interest rate; falls back to group `DEFAULT`. |
| Linkage `PARM-DATE` | 10-character prefix | Prefixed onto a numeric suffix to form `TRAN-ID`. |

### Outputs

| Output | Copybook / record | Role |
|:-------|:------------------|:-----|
| System transaction sequential file (`TRANSACT` DD → `SYSTRAN(+1)`) | `CVTRA05Y` | One interest transaction per category balance with non-zero rate. |
| Account master rewrite | `CVACT01Y` | Add total interest to current balance; zero cycle credit and cycle debit. |

`COMBTRAN` later sorts/merges `SYSTRAN` with the transaction master backup and `REPRO`s into the transaction master VSAM (`TRANSACT.VSAM.KSDS`).

### Business rules (plain English)

1. Walk every transaction category balance row in key order.
2. When the account id changes, finish the previous account (add accumulated interest to the account master current balance, clear cycle credit and cycle debit, rewrite the account master), then load the new account master and cross-reference.
3. Look up the disclosure group rate for (account group id, transaction type, transaction category). If the key is missing (status `23`), retry with account group `DEFAULT`.
4. If the interest rate is not zero, compute monthly interest for that category balance, accumulate it for the account, and write a system interest transaction:
   - type `01`, category `05`, source `System`
   - description `Int. for a/c ` + account id
   - amount = monthly interest
   - card number from cross-reference
   - origin/process timestamps from `CURRENT-DATE` formatted like a Db2 timestamp
5. Fee calculation is invoked whenever interest is computed, but is not implemented yet.
6. Interest is only applied for accounts that appear in the transaction category balance file (accounts with no category rows are never read or cycle-reset by this program).

### Exact interest formula and ROUNDED

From `1300-COMPUTE-INTEREST` (lines 462–465):

```cobol
COMPUTE WS-MONTHLY-INT
 = ( TRAN-CAT-BAL * DIS-INT-RATE) / 1200
```

- **Formula:** `monthlyInterest = (transactionCategoryBalance × interestRate) / 1200`  
  (annual percentage rate on the category balance, converted to a monthly amount by dividing by `12 × 100`).
- **`ROUNDED`:** **not used.** The `COMPUTE` has no `ROUNDED` phrase. `WS-MONTHLY-INT` is `PIC S9(09)V99`, so excess fractional digits are truncated to two decimal places per COBOL rules for `COMPUTE` without `ROUNDED`.
- Each monthly amount is `ADD`ed into `WS-TOTAL-INT` for the account, then written as `TRAN-AMT` on the system transaction.

### How results flow into the transaction master

1. **Immediate write (CBACT04C):** each interest line is `WRITE`n to the sequential `TRANSACT` DD, cataloged as `AWS.M2.CARDDEMO.SYSTRAN(+1)` (`1300-B-WRITE-TX`, lines 473–515; JCL lines 37–41).
2. **Account master side effect:** when an account break is processed, `1050-UPDATE-ACCOUNT` adds `WS-TOTAL-INT` to `ACCT-CURR-BAL` and clears cycle credit/debit (lines 350–370).
3. **Merge into transaction master (downstream JCL):** `COMBTRAN` concatenates the transaction master backup with `SYSTRAN(0)`, sorts by transaction id, and `REPRO`s into `TRANSACT.VSAM.KSDS` — the online transaction master used by CICS list/view/add and statements.

---

## 5. Open decisions

### D1 — Last account’s `1050-UPDATE-ACCOUNT` skipped at EOF (**real defect**)

**Finding: real.** The last account processed in the transaction category balance file does **not** receive `1050-UPDATE-ACCOUNT` (balance add + cycle reset), even though its interest transactions were already written.

**Evidence — read loop** (`app/cbl/CBACT04C.cbl`):

```188:222:app/cbl/CBACT04C.cbl
           PERFORM UNTIL END-OF-FILE = 'Y'
               IF  END-OF-FILE = 'N'
                   PERFORM 1000-TCATBALF-GET-NEXT
                   IF  END-OF-FILE = 'N'
                     ...
                     IF TRANCAT-ACCT-ID NOT= WS-LAST-ACCT-NUM
                       IF WS-FIRST-TIME NOT = 'Y'
                          PERFORM 1050-UPDATE-ACCOUNT
                       ...
                     END-IF
                     ...
                   END-IF
               ELSE
                    PERFORM 1050-UPDATE-ACCOUNT
               END-IF
           END-PERFORM.
```

**Why the `ELSE` never runs for EOF:**

1. Loop entry requires `END-OF-FILE ≠ 'Y'` (line 188).
2. Inside the loop, the outer `IF END-OF-FILE = 'N'` (line 189) is therefore always true on entry, so control never takes the `ELSE` at lines 219–221 on that iteration.
3. `1000-TCATBALF-GET-NEXT` sets `END-OF-FILE = 'Y'` on status `10` (lines 325–340).
4. The inner `IF END-OF-FILE = 'N'` (line 191) then skips processing; no update is performed.
5. The next `UNTIL` test sees `END-OF-FILE = 'Y'` and exits — still without calling `1050-UPDATE-ACCOUNT`.
6. There is no post-loop call to `1050-UPDATE-ACCOUNT` before the closes (lines 224–228).

Account-break updates for *prior* accounts do run at lines 194–196 when the account id changes. Only the **final** account is left without the rewrite.

**Decision required:** **replicate** this EOF skip for strict behavioral parity with production CardDemo, or **fix** it in the Java port (call the account update once after the read loop for the last account). Recommendation for Ledger Modernization: **fix**, with an explicit compatibility test toggle if golden-file compares against unrepaired COBOL output are required.

### D2 — Fee rules undefined

`1400-COMPUTE-FEES` is empty (lines 518–520). Slice 2 fee-quote needs a product decision: which fees, bases, and disclosure fields apply before implementing batch assessment.

### D3 — Rounding policy

Legacy truncates via `COMPUTE` without `ROUNDED`. Decide whether the Java service keeps truncation to two decimals or adopts banker’s / half-up rounding and documents the delta.

### D4 — Accounts with no transaction category balance rows

CBACT04C never opens those accounts, so their cycle credit/debit are not cleared during `INTCALC`. Confirm whether modernization should keep that scope or reset all accounts each cycle.

### D5 — Transaction id scheme

Interest `TRAN-ID` = `PARM-DATE` (10) + six-digit suffix. Decide how the Java job supplies the run date/prefix and avoids collisions with online and daily posted ids when merging into the transaction master.
