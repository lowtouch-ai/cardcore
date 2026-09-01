# Data model

Business entities behind the VSAM files, derived from the record-layout copybooks in `app/cpy/`, the CICS
`DEFINE FILE` entries in `app/csd/CARDDEMO.CSD`, and the `KEYS()` clauses in `app/jcl/*.jcl`.

Copybooks excluded because they are not business-entity record layouts: `COADM02Y`, `COCOM01Y` (CICS
commarea), `CODATECN`, `COMEN02Y`, `COSTM01.CPY` (report layout reusing `CVTRA05Y` fields), `COTTL01Y`,
`CSDAT01Y`, `CSLKPCDY` (lookup tables for validation), `CSMSG01Y`/`CSMSG02Y` (message/abend work areas),
`CSSETATY`, `CSSTRPFY`, `CSUTLDPY`/`CSUTLDWY` (date-validation work areas), `CVCRD01Y` (card-screen CICS
work area, not a card record), `UNUSED1Y` (dead duplicate of the user-security layout). `CVEXPORT.cpy` is a
REDEFINES wrapper for a branch-migration export file restating Customer/Account/Transaction/Card-Xref/Card —
not a distinct entity.

## Customer

**Copybook:** `CVCUS01Y.cpy` (canonical) / `CUSTREC.cpy` (identical duplicate) — `RECLN 500`
**VSAM file:** CICS resource `CUSTDAT`, dataset `AWS.M2.CARDDEMO.CUSTDATA.VSAM.KSDS` (DD/SELECT `CUSTFILE`)

The individual cardholder — identity, address, contact, and underwriting attributes.

| Field | PIC | Meaning |
|---|---|---|
| `CUST-ID` | 9(09) | Primary key |
| `CUST-FIRST-NAME` / `-MIDDLE-NAME` / `-LAST-NAME` | X(25) each | Name |
| `CUST-ADDR-LINE-1/2/3` | X(50) each | Address |
| `CUST-ADDR-STATE-CD` | X(02) | State code |
| `CUST-ADDR-COUNTRY-CD` | X(03) | Country code |
| `CUST-ADDR-ZIP` | X(10) | Postal code |
| `CUST-PHONE-NUM-1` / `-2` | X(15) each | Phone numbers |
| `CUST-SSN` | 9(09) | Social Security Number |
| `CUST-GOVT-ISSUED-ID` | X(20) | Government-issued ID number |
| `CUST-DOB-YYYY-MM-DD` | X(10) | Date of birth |
| `CUST-EFT-ACCOUNT-ID` | X(10) | Linked EFT (bank) account |
| `CUST-PRI-CARD-HOLDER-IND` | X(01) | Primary cardholder indicator |
| `CUST-FICO-CREDIT-SCORE` | 9(03) | Credit score |

**Key:** `CUST-ID` (9 bytes, offset 0) — `KEYS(9 0)` in `CUSTFILE.jcl`. No alternate index.

**Relationships:** Referenced by Card-Cross-Reference (`XREF-CUST-ID`); not keyed directly to Account or Card.

## Account

**Copybook:** `CVACT01Y.cpy` — `RECLN 300`
**VSAM file:** CICS resource `ACCTDAT`, dataset `AWS.M2.CARDDEMO.ACCTDATA.VSAM.KSDS` (DD/SELECT `ACCTFILE`)

The credit-card account — balances, credit limits, lifecycle dates.

| Field | PIC | Meaning |
|---|---|---|
| `ACCT-ID` | 9(11) | Primary key |
| `ACCT-ACTIVE-STATUS` | X(01) | Active/inactive flag |
| `ACCT-CURR-BAL` | S9(10)V99 | Current balance |
| `ACCT-CREDIT-LIMIT` | S9(10)V99 | Credit limit |
| `ACCT-CASH-CREDIT-LIMIT` | S9(10)V99 | Cash-advance credit limit |
| `ACCT-OPEN-DATE` | X(10) | Account open date |
| `ACCT-EXPIRAION-DATE` | X(10) | Account expiration date (sic — as in source) |
| `ACCT-REISSUE-DATE` | X(10) | Card/account reissue date |
| `ACCT-CURR-CYC-CREDIT` | S9(10)V99 | Current billing-cycle credits |
| `ACCT-CURR-CYC-DEBIT` | S9(10)V99 | Current billing-cycle debits |
| `ACCT-ADDR-ZIP` | X(10) | Account billing zip |
| `ACCT-GROUP-ID` | X(10) | Disclosure/pricing group |

**Key:** `ACCT-ID` (11 bytes, offset 0) — `KEYS(11 0)` in `ACCTFILE.jcl`. No alternate index on the account
file itself; the account is reached from Card via the cross-reference file's account-keyed alternate index.

**Relationships:** `ACCT-GROUP-ID` links to a Disclosure Group (`DIS-ACCT-GROUP-ID`) that sets interest rates
by transaction type/category. One or more Cards via Card-Cross-Reference; one Customer via Card-Cross-Reference.
Transaction Category Balances (`TRANCAT-ACCT-ID`) roll up per account.

## Card

**Copybook:** `CVACT02Y.cpy` — `RECLN 150`
**VSAM file:** CICS resource `CARDDAT`, dataset `AWS.M2.CARDDEMO.CARDDATA.VSAM.KSDS` (DD/SELECT `CARDFILE`);
alternate index `CARDAIX` on `AWS.M2.CARDDEMO.CARDDATA.VSAM.AIX` / path `...AIX.PATH`

A physical/virtual card issued against an account.

| Field | PIC | Meaning |
|---|---|---|
| `CARD-NUM` | X(16) | Primary key — card number (PAN) |
| `CARD-ACCT-ID` | 9(11) | Owning account ID |
| `CARD-CVV-CD` | 9(03) | Card verification value |
| `CARD-EMBOSSED-NAME` | X(50) | Name embossed on the card |
| `CARD-EXPIRAION-DATE` | X(10) | Card expiration date (sic) |
| `CARD-ACTIVE-STATUS` | X(01) | Active/inactive flag |

**Key:** `CARD-NUM` (16 bytes, offset 0) — `KEYS(16 0)` in `CARDFILE.jcl`. Alternate index `CARDAIX`,
`KEYS(11 16)` — `CARD-ACCT-ID` — browse/read cards by account (non-unique, an account can have several cards).

**Relationships:** Belongs to one Account (`CARD-ACCT-ID`). Transactions reference it by `TRAN-CARD-NUM`. The
Card-Cross-Reference file is the authoritative join tying Card ↔ Account ↔ Customer together — the card
record itself carries no customer ID.

## Card Cross-Reference

**Copybook:** `CVACT03Y.cpy` — `RECLN 50`
**VSAM file:** CICS resource `CCXREF`, dataset `AWS.M2.CARDDEMO.CARDXREF.VSAM.KSDS` ("CARD TO ACCOUNT XREF",
DD/SELECT `XREFFILE`/`CARDXREF`); alternate index `CXACAIX`, dataset `...CARDXREF.VSAM.AIX.PATH`
("ALTERNATE INDEX TO CCXREF VIA ACCOUNT KEY")

Pure join record, no business attributes of its own — resolves Card ↔ Account ↔ Customer.

| Field | PIC | Meaning |
|---|---|---|
| `XREF-CARD-NUM` | X(16) | Primary key — card number |
| `XREF-CUST-ID` | 9(09) | Owning customer ID |
| `XREF-ACCT-ID` | 9(11) | Owning account ID |

**Key:** `XREF-CARD-NUM` (16 bytes, offset 0) — `KEYS(16 0)` in `XREFFILE.jcl`. Alternate index `CXACAIX`,
`KEYS(11,25)` — `XREF-ACCT-ID` (offset 25 = 16-byte card num + 9-byte cust id) — look up all cards/customers
for an account.

**Relationships:** Central hub — one record per Card, pointing to exactly one Account and one Customer. This
answers both "which customer owns this card" and "which cards belong to this account."

## Transaction

**Copybook:** `CVTRA05Y.cpy` — `RECLN 350` — posted/master transaction record
**VSAM file:** CICS resource `TRANSACT`, dataset `AWS.M2.CARDDEMO.TRANSACT.VSAM.KSDS` (DD/SELECT
`TRANFILE`/`TRANSACT`); alternate index on processed timestamp, dataset `...TRANSACT.VSAM.AIX`

A posted, individual card transaction (purchase, payment, etc.).

| Field | PIC | Meaning |
|---|---|---|
| `TRAN-ID` | X(16) | Primary key — transaction identifier |
| `TRAN-TYPE-CD` | X(02) | Transaction type code |
| `TRAN-CAT-CD` | 9(04) | Transaction category code |
| `TRAN-SOURCE` | X(10) | Origination source (e.g. POS, online) |
| `TRAN-DESC` | X(100) | Free-text description |
| `TRAN-AMT` | S9(09)V99 | Transaction amount |
| `TRAN-MERCHANT-ID` | 9(09) | Merchant identifier |
| `TRAN-MERCHANT-NAME` | X(50) | Merchant name |
| `TRAN-MERCHANT-CITY` | X(50) | Merchant city |
| `TRAN-MERCHANT-ZIP` | X(10) | Merchant zip |
| `TRAN-CARD-NUM` | X(16) | Card used |
| `TRAN-ORIG-TS` | X(26) | Originating timestamp |
| `TRAN-PROC-TS` | X(26) | Processing timestamp |

**Key:** `TRAN-ID` (16 bytes, offset 0) — `KEYS(16 0)` in `TRANFILE.jcl`. Non-unique alternate index
`KEYS(26 304)` on `TRAN-PROC-TS` for date-range browsing/reporting.

**Relationships:** Belongs to a Card (resolved to Account/Customer via Card-Cross-Reference). Classified by
`TRAN-TYPE-CD`/`TRAN-CAT-CD` against Transaction Type / Transaction Category Type. Aggregated into Transaction
Category Balance. Loaded from the Daily Transaction staging file by the posting batch job (`POSTTRAN.jcl`,
program `CBTRN02C`) — see [batch-jobs.md](batch-jobs.md).

### Daily Transaction (staging copy)

**Copybook:** `CVTRA06Y.cpy` — `RECLN 350`, field-for-field identical to `CVTRA05Y`, prefixed `DALYTRAN-`
**File:** sequential dataset `AWS.M2.CARDDEMO.DALYTRAN.PS` / `.PS.INIT` (DD `DALYTRAN`)

Pre-posting staging file that daily-capture jobs write to; the posting job reads, validates, and posts these
records into `TRANSACT`, updating account balances in the process.

## Transaction Type (reference data)

**Copybook:** `CVTRA03Y.cpy` — `RECLN 60`
**VSAM file:** dataset `AWS.M2.CARDDEMO.TRANTYPE.VSAM.KSDS` (DD/SELECT `TRANTYPE`), batch-only

| Field | PIC | Meaning |
|---|---|---|
| `TRAN-TYPE` | X(02) | Primary key — type code |
| `TRAN-TYPE-DESC` | X(50) | Description |

**Key:** `TRAN-TYPE` (2 bytes) — `KEYS(2 0)` in `TRANTYPE.jcl`. Referenced by Transaction, Transaction
Category Type, Transaction Category Balance, and Disclosure Group.

## Transaction Category Type (reference data)

**Copybook:** `CVTRA04Y.cpy` — `RECLN 60`
**VSAM file:** dataset `AWS.M2.CARDDEMO.TRANCATG.VSAM.KSDS` (DD/SELECT `TRANCATG`), batch-only

| Field | PIC | Meaning |
|---|---|---|
| `TRAN-TYPE-CD` | X(02) | Composite key part — parent type |
| `TRAN-CAT-CD` | 9(04) | Composite key part — category code |
| `TRAN-CAT-TYPE-DESC` | X(50) | Description |

**Key:** composite `TRAN-TYPE-CD` + `TRAN-CAT-CD` (6 bytes) — `KEYS(6 0)` in `TRANCATG.jcl`. Child of
Transaction Type.

## Transaction Category Balance

**Copybook:** `CVTRA02Y.cpy` — `RECLN 50`
**VSAM file:** dataset `AWS.M2.CARDDEMO.TCATBALF.VSAM.KSDS` (DD/SELECT `TCATBALF`), batch-only

Running balance an account carries within a transaction type/category — cycle-to-date summarization input to
interest calculation.

| Field | PIC | Meaning |
|---|---|---|
| `TRANCAT-ACCT-ID` | 9(11) | Composite key part — account |
| `TRANCAT-TYPE-CD` | X(02) | Composite key part — transaction type |
| `TRANCAT-CD` | 9(04) | Composite key part — transaction category |
| `TRAN-CAT-BAL` | S9(09)V99 | Balance accumulated for the account/type/category |

**Key:** composite (17 bytes) — `KEYS(17 0)` in `TCATBALF.jcl`. Feeds the interest-calculation batch job
(`INTCALC.jcl`) together with Disclosure Group rates.

## Disclosure Group

**Copybook:** `CVTRA01Y.cpy` — `RECLN 50`
**VSAM file:** dataset `AWS.M2.CARDDEMO.DISCGRP.VSAM.KSDS` (DD/SELECT `DISCGRP`), batch-only

Interest rate applicable to a pricing group of accounts for a given transaction type/category.

| Field | PIC | Meaning |
|---|---|---|
| `DIS-ACCT-GROUP-ID` | X(10) | Composite key part — matches Account's `ACCT-GROUP-ID` |
| `DIS-TRAN-TYPE-CD` | X(02) | Composite key part — transaction type |
| `DIS-TRAN-CAT-CD` | 9(04) | Composite key part — transaction category |
| `DIS-INT-RATE` | S9(04)V99 | Interest rate for this group/type/category |

**Key:** composite (16 bytes) — `KEYS(16 0)` in `DISCGRP.jcl`. Used with Transaction Category Balance by the
interest-calculation job to determine accrual per account/category.

## User Security (internal operator, not a customer)

**Copybook:** `CSUSR01Y.cpy`
**VSAM file:** CICS resource `USRSEC`, dataset `AWS.M2.CARDDEMO.USRSEC.VSAM.KSDS`

Internal CardDemo application user (bank staff) — used for CICS signon and menu authorization, unrelated to
the Customer entity.

| Field | PIC | Meaning |
|---|---|---|
| `SEC-USR-ID` | X(08) | Primary key — user ID |
| `SEC-USR-FNAME` | X(20) | First name |
| `SEC-USR-LNAME` | X(20) | Last name |
| `SEC-USR-PWD` | X(08) | Password |
| `SEC-USR-TYPE` | X(01) | `'A'` = Admin, `'U'` = regular User (`CDEMO-USRTYP-ADMIN`/`CDEMO-USRTYP-USER` 88-levels in `COCOM01Y.cpy`) — drives which menu is shown at signon |

**Key:** `SEC-USR-ID` (8 bytes) — `KEYS(8,0)` in `DUSRSECJ.jcl`. No alternate index. Standalone — not linked
to Customer/Account/Card.

## Entity relationship overview

```
Customer (CUSTDAT)                     Disclosure Group (DISCGRP)
  CUST-ID (PK)                           DIS-ACCT-GROUP-ID + DIS-TRAN-TYPE-CD
       ^                                 + DIS-TRAN-CAT-CD  (PK, composite)
       | XREF-CUST-ID                          ^
       |                                       | ACCT-GROUP-ID = DIS-ACCT-GROUP-ID
  Card-Cross-Reference (CCXREF)                |
    XREF-CARD-NUM (PK)  --------------->   Account (ACCTDAT)
    XREF-CUST-ID                             ACCT-ID (PK)
    XREF-ACCT-ID  <--(alt idx CXACAIX)--------^
       |
       | XREF-CARD-NUM = CARD-NUM
       v
     Card (CARDDAT)
       CARD-NUM (PK)
       CARD-ACCT-ID  --(alt idx CARDAIX)--> Account.ACCT-ID
       |
       | TRAN-CARD-NUM = CARD-NUM
       v
   Transaction (TRANSACT) <--- posted from --- Daily Transaction (DALYTRAN, staging)
     TRAN-ID (PK)
     TRAN-TYPE-CD ------> Transaction Type (TRANTYPE): TRAN-TYPE (PK)
     TRAN-TYPE-CD+TRAN-CAT-CD -> Transaction Category Type (TRANCATG): composite PK

   Transaction Category Balance (TCATBALF)
     TRANCAT-ACCT-ID   ------> Account.ACCT-ID
     TRANCAT-TYPE-CD + TRANCAT-CD -> Transaction Type / Category Type

User Security (USRSEC) — standalone, not linked to the above (internal operator accounts)
```

- **Customer** is the person; **Account** is their credit line; **Card** is a physical/virtual instrument
  drawn on an account. Card and Account never point directly to Customer — **Card-Cross-Reference** is the
  sole join, mapping one card number to exactly one account and one customer, with an account-keyed alternate
  index (`CXACAIX`) enabling "all cards for this account/customer" lookups. Card also carries its own
  account-keyed alternate index (`CARDAIX`) for the same purpose, independent of the cross-reference file.
- **Transaction** records a posted charge/payment against a card (and transitively its account), classified
  by a two-level reference hierarchy: **Transaction Type** and, within it, **Transaction Category Type**. New
  transactions land first in the sequential **Daily Transaction** file and are posted into the VSAM
  Transaction file by a batch job that also updates balances.
- **Transaction Category Balance** rolls up activity per account/type/category — the working figures for
  billing/interest processing.
- **Disclosure Group** is a rate table keyed by account-group + type + category; an Account's `ACCT-GROUP-ID`
  selects which disclosure group's rates apply, and the interest-calculation job combines Disclosure Group
  rates with Transaction Category Balance amounts to compute interest.
- **User Security** is unrelated to the customer/account/card graph — it governs CICS signon and whether a
  session gets the admin or general-user menu.

**Source files referenced:** `app/cpy/{CVCUS01Y,CUSTREC,CVACT01Y,CVACT02Y,CVACT03Y,CVTRA01Y,CVTRA02Y,CVTRA03Y,CVTRA04Y,CVTRA05Y,CVTRA06Y,CSUSR01Y,CVEXPORT}.cpy`,
`app/csd/CARDDEMO.CSD`, `app/jcl/{ACCTFILE,CARDFILE,CUSTFILE,XREFFILE,TRANFILE,TRANIDX,TCATBALF,DISCGRP,TRANTYPE,TRANCATG,DUSRSECJ,POSTTRAN,INTCALC}.jcl`.
