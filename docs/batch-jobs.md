# Batch jobs

Functional documentation of the batch (non-CICS) programs in `app/cbl/` and the JCL/scheduler definitions
that run them. See [data-model.md](data-model.md) for the record layouts referenced below.

## Batch cycle overview

The scheduler definitions (`app/scheduler/CardDemo.ca7`, `app/scheduler/CardDemo.controlm`) describe three
recurring cycles, each bracketed by `CLOSEFIL` (close VSAM for exclusive batch access) / `OPENFIL` (reopen
for CICS) and a `WAITSTEP` sync point between phases:

1. **Daily transaction-posting cycle** (Control-M folder `DAILY-TransactionBackup`; CA7 chain
   `CLOSEFIL → CBPAUP0J → POSTTRAN → WAITSTEP → OPENFIL`):
   - `CBPAUP0J` (not in this source tree) presumably stages the day's transaction extract.
   - **POSTTRAN** (`CBTRN02C`) validates and posts the day's transactions, updating balances and rejecting
     bad records.
   - The same CA7 chain also runs, on the same daily cadence: a reference-file snapshot phase
     (`CLOSEFIL → READACCT → READCARD → READCUST → READXREF → WAITSTEP → OPENFIL`), statement generation
     (`CLOSEFIL → CREASTMT → TXT2PDF1 → WAITSTEP → OPENFIL`), and the category-balance report
     (`CLOSEFIL → PRTCATBL → WAITSTEP → OPENFIL`).

2. **Monthly interest-calculation cycle** (Control-M folder `MONTHLY-InterestCalculation`:
   `CLOSEFIL → INTCALC → COMBTRAN → WAITSTEP → OPENFIL`):
   - **INTCALC** (`CBACT04C`) computes interest per account/category and writes it as new transactions to a
     new `SYSTRAN` generation, adding interest directly to each account's current balance.
   - **COMBTRAN** sorts and merges the day's posted-transaction backup (`TRANSACT.BKUP`) with the
     interest-generated `SYSTRAN` file and reloads the combined set into the `TRANSACT` VSAM master.

3. **Weekly reference-data refresh cycles** (`WEEKLY-TransactionTypesDBRefresh`,
   `WEEKLY-DisclosureGroupsRefresh`) reload `TRANTYPE`/`TRANCATG` and `DISCGRP` — IDCAMS/extract-based, not
   COBOL-driven.

`TRANREPT.jcl` (ad hoc, not in the CA7/Control-M chains) backs up the transaction VSAM file, sorts/filters it
by date range and card number, then runs **CBTRN03C** to print the transaction detail report.
`CBEXPORT`/`CBIMPORT` (branch-migration jobs) are standalone, outside the scheduled chains.

## Programs

### CBACT01C — Account file dump/export utility

- **JCL:** `READACCT.jcl`, job `READACCT`, step `STEP05`.
- **Purpose:** Sequentially reads the Account Master VSAM KSDS and re-emits it in three output formats for
  downstream tooling/testing.
- **Logic:** For every account record (`CVACT01Y`):
  - Writes a fixed 107-byte flattened record to `OUTFILE` (`AWS.M2.CARDDEMO.ACCTDATA.PSCOMP`), converting
    `ACCT-REISSUE-DATE` via the assembler subroutine `COBDATFT`. If `ACCT-CURR-CYC-DEBIT` is zero, the output
    field is forced to the literal `2525.00` instead of the real value — a data-quality/test-fixture quirk,
    not a real financial rule.
  - Writes a 5-element array record to `ARRYFILE` with hardcoded demo values (`1005.00`/`1525.00`/`-1025.00`/
    `-2500.00` in elements 1–3, seeded from `ACCT-CURR-BAL`) — demonstration data, not derived business logic.
  - Writes two variable-length sub-records to `VBRCFILE`: a 12-byte "VB1" record (account id + active status)
    and a 39-byte "VB2" record (account id, balance, credit limit, reissue year).
- **I/O:** Input `ACCTFILE` (VSAM KSDS, keyed `FD-ACCT-ID`). Outputs: `OUTFILE` (PS, LRECL 107), `ARRYFILE`
  (PS, LRECL 110), `VBRCFILE` (variable PS, LRECL 84).
- **Batch position:** Daily, in the `READACCT → READCARD → READCUST → READXREF` snapshot chain, after the
  daily posting close/open bracket.

### CBACT02C — Card file reader

- **JCL:** `READCARD.jcl`, job `READCARD`, step `STEP05`.
- **Purpose:** Sequential read-and-display verification utility for the Card Master VSAM KSDS (`CVACT02Y`,
  keyed `FD-CARD-NUM`) — no output file, records are `DISPLAY`ed to SYSOUT.
- **Logic:** No business transformation; straight read loop to end-of-file, abends via `CEE3ABD` on
  unexpected file status.
- **I/O:** Input only — `CARDFILE`.
- **Batch position:** Daily snapshot chain, after `READACCT`, before `READCUST`.

### CBACT03C — Card cross-reference file reader

- **JCL:** `READXREF.jcl`, job `READXREF`, step `STEP05`.
- **Purpose:** Sequential read-and-display of the Card↔Account↔Customer cross-reference VSAM KSDS
  (`CVACT03Y`, keyed `FD-XREF-CARD-NUM`).
- **Logic:** No transformation; same read/display pattern as CBACT02C.
- **I/O:** Input only — `XREFFILE`.
- **Batch position:** Daily snapshot chain, last in `READACCT → READCARD → READCUST → READXREF`.

### CBCUS01C — Customer file reader

- **JCL:** `READCUST.jcl`, job `READCUST`, step `STEP05`.
- **Purpose:** Sequential read-and-display of the Customer Master VSAM KSDS (`CVCUS01Y`, keyed `FD-CUST-ID`).
- **Logic:** No transformation; identical read/display pattern.
- **I/O:** Input only — `CUSTFILE`.
- **Batch position:** Daily snapshot chain, between `READCARD` and `READXREF`.

### CBACT04C — Interest calculator (INTCALC)

- **JCL:** `INTCALC.jcl`, job `INTCALC`, step `STEP15 EXEC PGM=CBACT04C,PARM='2022071800'` (the parm is a
  10-char processing date used to build generated transaction IDs).
- **Purpose:** Computes monthly interest per account/transaction-category from the transaction-category
  balance file, posts it as new system-generated transactions, and updates account balances.
- **Business logic (exact):**
  - Reads `TCATBALF` sequentially (keyed acct-id + type-cd + category-cd). At each account boundary
    (`TRANCAT-ACCT-ID` change), finalizes the prior account (`1050-UPDATE-ACCOUNT`) and reads that account's
    master (`ACCTFILE`, random) and cross-reference (`XREFFILE`, keyed by alternate key `FD-XREF-ACCT-ID`).
  - For each category-balance record, looks up the interest rate from `DISCGRP` (keyed `ACCT-GROUP-ID +
    TRANCAT-TYPE-CD + TRANCAT-CD`, field `DIS-INT-RATE`). If no group-specific record is found
    (`DISCGRP-STATUS = '23'`), falls back to a record keyed `'DEFAULT'` + type + category
    (`1200-A-GET-DEFAULT-INT-RATE`).
  - **Interest formula** (`1300-COMPUTE-INTEREST`):
    ```
    WS-MONTHLY-INT = (TRAN-CAT-BAL * DIS-INT-RATE) / 1200
    ```
    Category balance × annual rate (whole-number percent, e.g. `18.00` = 18%), divided by 1200 (÷100 to a
    fraction, ÷12 for monthly). Skipped entirely when `DIS-INT-RATE = 0`.
  - `WS-MONTHLY-INT` accumulates into `WS-TOTAL-INT` per account across all its categories.
  - Each computed interest amount is written as a new transaction to `TRANSACT` (a new GDG generation,
    `AWS.M2.CARDDEMO.SYSTRAN(+1)`): `TRAN-TYPE-CD = '01'`, `TRAN-CAT-CD = '05'`, `TRAN-SOURCE = 'System'`,
    description `'Int. for a/c ' + ACCT-ID`, `TRAN-AMT = WS-MONTHLY-INT`, no merchant data, DB2-format
    timestamps. `TRAN-ID` = job parm date + a running 6-digit sequence suffix, unique within the run.
  - `1050-UPDATE-ACCOUNT` (at each account boundary and end-of-file): adds `WS-TOTAL-INT` to `ACCT-CURR-BAL`
    and resets `ACCT-CURR-CYC-CREDIT`/`ACCT-CURR-CYC-DEBIT` to zero (new billing cycle), then rewrites the
    account record.
  - `1400-COMPUTE-FEES` is an unimplemented stub — no fee logic exists despite the paragraph name.
- **I/O:** Input `TCATBAL-FILE` (VSAM, sequential), `XREF-FILE` (VSAM, random), `DISCGRP-FILE` (VSAM,
  random), `ACCOUNT-FILE` (VSAM, I-O/random). Output: `TRANSACT-FILE` (new sequential GDG generation).
- **Batch position:** Monthly cycle, first step after `CLOSEFIL`; output (`SYSTRAN`) feeds `COMBTRAN`.

### CBTRN01C — Daily transaction lookup/verification (not wired into any job)

- **JCL:** none in `app/jcl/` — not part of any current job stream; functionally superseded by `CBTRN02C`.
- **Purpose:** Reads `DALYTRAN` and, per transaction, verifies the card cross-reference and account master
  exist — a lookup/verification pass only, with `DISPLAY` diagnostics. No posting, no balance updates, no
  output file.
- **Logic:** Per record: look up `XREF-FILE` by card number (`2000-LOOKUP-XREF`); if found, look up
  `ACCOUNT-FILE` by the resolved account id (`3000-READ-ACCOUNT`); displays "ACCOUNT NOT FOUND" or "CARD
  NUMBER ... COULD NOT BE VERIFIED. SKIPPING TRANSACTION ID-" on failure. Nothing is rejected to a file or
  posted.
- **I/O:** Inputs `DALYTRAN`, `CUSTOMER-FILE`, `XREF-FILE`, `CARD-FILE`, `ACCOUNT-FILE`, `TRANSACT-FILE` all
  opened, but `CUSTFILE`/`CARDFILE`/`TRANSACT-FILE` are never actually read by the logic. No outputs.
- **Batch position:** Not scheduled — an earlier/simpler prototype of the daily transaction pipeline, kept
  in source as a variant.

### CBTRN02C — Daily transaction validation & posting (POSTTRAN)

- **JCL:** `POSTTRAN.jcl`, job `POSTTRAN`, step `STEP15`.
- **Purpose:** The daily transaction-posting engine — validates each transaction on the daily input file,
  posts accepted ones to account and category balances, rejects bad ones to a reject file.
- **Validation logic** (`1500-VALIDATE-TRAN`), exact:
  1. **XREF lookup** (`1500-A-LOOKUP-XREF`): card number must exist in `XREFFILE`; else reason **100**
     "INVALID CARD NUMBER FOUND".
  2. **Account lookup** (`1500-B-LOOKUP-ACCT`): resolved account must exist in `ACCTFILE`; else reason
     **101** "ACCOUNT RECORD NOT FOUND".
  3. **Credit-limit check:** `WS-TEMP-BAL = ACCT-CURR-CYC-CREDIT − ACCT-CURR-CYC-DEBIT + DALYTRAN-AMT`; if
     `ACCT-CREDIT-LIMIT < WS-TEMP-BAL` → reason **102** "OVERLIMIT TRANSACTION".
  4. **Expiration check:** if `ACCT-EXPIRAION-DATE < DALYTRAN-ORIG-TS(1:10)` → reason **103** "TRANSACTION
     RECEIVED AFTER ACCT EXPIRATION".
  5. Account-record rewrite failure during posting sets reason **109** "ACCOUNT RECORD NOT FOUND" (defensive,
     late-binding check).
  - Any non-zero `WS-VALIDATION-FAIL-REASON` rejects the whole transaction.
- **Posting logic** (`2000-POST-TRANSACTION`, accepted transactions only):
  - Maps `DALYTRAN-*` into a `TRAN-RECORD`, generates a DB2-format processing timestamp
    (`Z-GET-DB2-FORMAT-TIMESTAMP`, `FUNCTION CURRENT-DATE`), keeps `DALYTRAN-ORIG-TS` as `TRAN-ORIG-TS`.
  - `2700-UPDATE-TCATBAL`: looks up (or creates, `WS-CREATE-TRANCAT-REC`) the category-balance record for the
    account/type/category and adds `DALYTRAN-AMT` to `TRAN-CAT-BAL`.
  - `2800-UPDATE-ACCOUNT-REC`: adds `DALYTRAN-AMT` to `ACCT-CURR-BAL`. If `DALYTRAN-AMT >= 0`, adds to
    `ACCT-CURR-CYC-CREDIT`; if negative, adds to `ACCT-CURR-CYC-DEBIT` — the sign of the transaction amount
    classifies it as a credit (payment/refund) vs. debit (purchase/charge) for cycle totals. Rewrites the
    account record.
  - `2900-WRITE-TRANSACTION-FILE`: writes the posted transaction to `TRANSACT-FILE` (keyed `FD-TRANS-ID`).
- **Rejects** (`2500-WRITE-REJECT-REC`): original transaction record plus a validation trailer (fail reason
  code + 76-char description) written to `DALYREJS`.
- **End-of-job:** displays `WS-TRANSACTION-COUNT` processed and `WS-REJECT-COUNT` rejected; sets
  `RETURN-CODE = 4` if any rejects occurred, so downstream JCL/scheduler steps can detect partial failure.
- **I/O:** Input `DALYTRAN-FILE` (sequential PS), `XREF-FILE` (VSAM, random), `ACCOUNT-FILE` (VSAM, I-O
  random). Output `TRANSACT-FILE` (VSAM KSDS, new), `DALYREJS-FILE` (new sequential PS, LRECL 430),
  `TCATBAL-FILE` (VSAM, I-O random, updated in place).
- **Batch position:** Daily cycle — the core posting job, run right after `CLOSEFIL`/`CBPAUP0J`, before file
  reopen and the snapshot/report/statement phases.

### CBTRN03C — Transaction detail report

- **JCL:** `TRANREPT.jcl`, step `STEP10R` (preceded by `STEP05R`, a REPRO backup, and a `SORT` step that
  filters/sorts the transaction file by date range and card number).
- **Purpose:** Formatted, paginated transaction detail report (text, 133-byte lines) for a date range,
  grouped by card number, with running page/account/grand totals.
- **Logic:**
  - Reads a date-parameter record (`DATEPARM`, `WS-START-DATE`/`WS-END-DATE`) once at start.
  - Reads posted transactions (pre-sorted by card number, pre-filtered to the date window by the JCL `SORT`
    step) from `TRANFILE`, additionally re-checking `TRAN-PROC-TS(1:10)` against the date window in-program.
  - Per transaction: looks up card cross-reference (`CARDXREF`) on card-number change (writing an
    account-total break line first if not the first group), looks up transaction-type description
    (`TRANTYPE`) and category description (`TRANCATG`) by key, writes a detail line.
  - Report layout: header block, detail lines (Tran ID, Tran Details, Tran Amount), page totals every
    `WS-PAGE-SIZE` (20) lines, account-total break lines on card-number change, grand total at end-of-file.
  - Unlike `CBTRN02C`'s soft-reject handling, any missing xref/type/category lookup key here is fatal
    (`INVALID KEY` → abend via `CEE3ABD`).
- **I/O:** Inputs `TRANSACT-FILE` (sequential, pre-sorted/filtered copy `TRANSACT.DALY`), `XREF-FILE`
  (`CARDXREF`, VSAM random), `TRANTYPE-FILE` (VSAM random), `TRANCATG-FILE` (VSAM random), `DATE-PARMS-FILE`
  (sequential). Output: `REPORT-FILE` (`TRANREPT`, sequential, LRECL 133).
- **Batch position:** Ad hoc/on-demand — not in the CA7/Control-M chains examined, but logically runs after
  `POSTTRAN` (its JCL takes a backup+sorted copy of the transaction VSAM master).

### CBSTM03A — Account statement generator

- **JCL:** `CREASTMT.JCL`, step `STEP040` (preceded by steps that rebuild `TRXFL`, a card-number+trans-id
  keyed copy of the transaction VSAM, via SORT + IDCAMS REPRO).
- **Purpose:** Produces per-account customer statements in two parallel formats — fixed-width plain text
  (`STMTFILE`) and HTML (`HTMLFILE`) — from transaction, cross-reference, customer and account data.
- **Logic:**
  - Deliberately uses low-level techniques (per its own header comment, for tooling-exercise purposes):
    reads PSA/TCB/TIOT mainframe control blocks to display DD names from the JCL, uses `ALTER`/`GO TO`
    state-machine control flow instead of structured `PERFORM`, and delegates all file I/O to a companion
    subroutine `CBSTM03B` via a generic linkage-passed operation code (`M03B-OPEN`/`READ`/`READ-K`/`CLOSE`).
  - Main flow: for each cross-reference record (`XREFFILE`, sequential), looks up the customer (`CUSTFILE`,
    keyed `XREF-CUST-ID`) and account (`ACCTFILE`, keyed `XREF-ACCT-ID`), builds a statement header
    (`5000-CREATE-STATEMENT`) with customer name/address, account id, current balance, and FICO score.
  - Transactions for the account's cards are pre-loaded into an in-memory table (`WS-TRNX-TABLE`, up to 51
    cards × 10 transactions each) keyed by card number while `TRNXFILE` is read sequentially; a matching
    card's transactions are then written to the statement (`4000-TRNXFILE-GET`/`6000-WRITE-TRANS`) — each
    showing transaction id, description, and amount — accumulating a running `WS-TOTAL-AMT` printed as
    "Total EXP" at the end.
  - No interest, rate, or threshold logic — a formatted transcript of transactions per account with a running
    total; balance and FICO score come straight from the account/customer master records.
- **I/O:** Inputs (via `CBSTM03B`): `TRNXFILE` (VSAM KSDS, card+trans-id keyed transaction copy),
  `XREFFILE`, `CUSTFILE`, `ACCTFILE`. Outputs: `STMTFILE` (text, LRECL 80), `HTMLFILE` (HTML, LRECL 100).
- **Batch position:** Daily cycle, `CREASTMT` step, after the daily posting/reload bracket; followed by
  `TXT2PDF1` (statement PDF conversion, not COBOL).

### CBSTM03B — Statement file-I/O subroutine

- **JCL:** none — called only as a subroutine (dynamic `CALL 'CBSTM03B'`) from `CBSTM03A`, same `STEPLIB` in
  `CREASTMT.JCL`.
- **Purpose:** Generic file-access helper used exclusively by `CBSTM03A`, keeping that program's mainline
  free of direct file verbs. No business logic of its own.
- **Logic:** Dispatches on the caller-supplied DD name (`TRNXFILE`/`XREFFILE`/`CUSTFILE`/`ACCTFILE`) and
  operation code (open/read/read-by-key/close), performs the corresponding native file verb, returns the
  file status via `LK-M03B-RC`.
- **I/O:** `TRNX-FILE`, `XREF-FILE` (VSAM, sequential), `CUST-FILE`, `ACCT-FILE` (VSAM, random) — all opened
  INPUT only.
- **Batch position:** Same step as `CBSTM03A`; not independently scheduled.

### CBEXPORT — Branch-migration data export

- **JCL:** `CBEXPORT.jcl`; `STEP01` defines a new VSAM KSDS export cluster
  (`AWS.M2.CARDDEMO.EXPORT.DATA`) via IDCAMS, `STEP02` runs the program.
- **Purpose:** Reads the five core master files and produces a single multi-record-type export file (fixed
  500-byte records) for migrating customer data to another system/branch.
- **Logic:**
  - Sequentially reads `CUSTOMER-INPUT`, `ACCOUNT-INPUT`, `XREF-INPUT`, `TRANSACTION-INPUT`, `CARD-INPUT` in
    that order.
  - For every record, builds an `EXPORT-RECORD` (layout `CVEXPORT`) tagged with a one-character record type
    (`'C'` customer, `'A'` account, `'X'` xref, `'T'` transaction, `'D'` card), a common timestamp built from
    `ACCEPT ... FROM DATE/TIME`, a monotonically increasing sequence number (also the VSAM record key), and
    hardcoded branch/region metadata (`EXPORT-BRANCH-ID = '0001'`, `EXPORT-REGION-CODE = 'NORTH'` — fixed
    placeholders, not derived from source data).
  - No filtering, validation, or business-rule logic — field-by-field remapping, with per-type and total
    counters displayed at the end.
- **I/O:** Inputs: `CUSTFILE`, `ACCTFILE`, `XREFFILE`, `TRANSACT`, `CARDFILE` (VSAM KSDS, sequential).
  Output: `EXPFILE` (new VSAM KSDS, keyed by sequence number, RECORDSIZE 500).
- **Batch position:** Standalone/on-demand migration job, not in the recurring scheduler chains.

### CBIMPORT — Branch-migration data import

- **JCL:** `CBIMPORT.jcl`, `STEP01`.
- **Purpose:** Reverse of CBEXPORT — reads the multi-record export file and splits it back into five
  separate normalized sequential output files, flagging unrecognized record types.
- **Logic:**
  - Reads `EXPORT-INPUT` sequentially; dispatches on `EXPORT-REC-TYPE` (`2200-PROCESS-RECORD-BY-TYPE`):
    `'C'`→customer, `'A'`→account, `'X'`→xref, `'T'`→transaction, `'D'`→card, anything else →
    `2700-PROCESS-UNKNOWN-RECORD`.
  - Each handler does a straight field-by-field reverse mapping back into the native record layout and
    writes it to a plain sequential output file (`CUSTOUT`/`ACCTOUT`/`XREFOUT`/`TRNXOUT`/`CARDOUT`).
  - Unknown record types are logged to a pipe-delimited `ERROUT` file (timestamp, record type, sequence
    number, "Unknown record type encountered") and counted; processing continues without abending.
  - `3000-VALIDATE-IMPORT` is a stub — despite the program header claiming "Validate data integrity using
    checksums," no checksum or validation logic is implemented; it unconditionally displays "No validation
    errors detected".
- **I/O:** Input: `EXPFILE` (VSAM KSDS, sequential). Outputs: `CUSTOUT`, `ACCTOUT`, `XREFOUT`, `TRNXOUT`,
  `CARDOUT` (new sequential PS, fixed lengths 500/300/50/350/150), `ERROUT` (sequential, LRECL 132).
- **Batch position:** Standalone/on-demand, paired with `CBEXPORT`, not in the recurring scheduler chains.

### CSUTLDTC — Date validation utility (callable subroutine)

- **JCL:** none — invoked via `CALL 'CSUTLDTC'` from online/batch programs needing date validation (callers
  in source: `COTRN02C.cbl`, `CORPT00C.cbl`, and copybook `CSUTLDPY.cpy`).
- **Purpose:** Validates whether a passed date string is a legitimate calendar date in a given format, using
  the Language Environment `CEEDAYS` callable service.
- **Logic:** Wraps the input date and format mask into varying-length strings, calls `CEEDAYS` to convert to
  a Lilian date, and inspects the returned `FEEDBACK-CODE` token to classify the result into one of nine
  outcomes (valid date, insufficient data, bad date value, invalid era, unsupported range, invalid month, bad
  picture string, non-numeric data, year-in-era-zero, or a catch-all "date is invalid"), returning a
  15-character result string and the LE severity code as `RETURN-CODE`.
- **I/O:** None — pure linkage-section, in-memory validation (`LS-DATE`, `LS-DATE-FORMAT` in; `LS-RESULT`
  out).
- **Batch position:** Not scheduled itself; a shared utility called by whichever program needs to validate a
  date field.

## Key source files

- COBOL: `app/cbl/{CBACT01C,CBACT02C,CBACT03C,CBACT04C,CBCUS01C,CBTRN01C,CBTRN02C,CBTRN03C,CBSTM03A,CBSTM03B,CBEXPORT,CBIMPORT,CSUTLDTC}`
- JCL: `app/jcl/{READACCT,READCARD,READXREF,READCUST,INTCALC,POSTTRAN,TRANREPT,CREASTMT,CBEXPORT,CBIMPORT,COMBTRAN}`
- Scheduler: `app/scheduler/CardDemo.ca7`, `app/scheduler/CardDemo.controlm`
- Rate/copybook detail: `app/cpy/CVTRA01Y.cpy`, `app/cpy/CVTRA02Y.cpy`
