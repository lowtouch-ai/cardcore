# Screens and transactions

Functional documentation of the online (CICS) part of the base application: transaction inventory, screen
flow, and the business rules implemented by each program. See [data-model.md](data-model.md) for the record
layouts referenced below.

## Transaction inventory

| Tran ID | Program | Function |
|:--|:--|:--|
| `CC00` | `COSGN00C` | Sign on |
| `CM00` | `COMEN01C` | Main menu (regular users) |
| `CA00` | `COADM01C` | Admin menu |
| `CU00` | `COUSR00C` | User list |
| `CU01` | `COUSR01C` | User add |
| `CU02` | `COUSR02C` | User update |
| `CU03` | `COUSR03C` | User delete |
| `CAVW` | `COACTVWC` | Account view |
| `CAUP` | `COACTUPC` | Account update |
| `CCLI` | `COCRDLIC` | Credit card list |
| `CCDL` | `COCRDSLC` | Credit card view/detail |
| `CCUP` | `COCRDUPC` | Credit card update |
| `CT00` | `COTRN00C` | Transaction list |
| `CT01` | `COTRN01C` | Transaction view |
| `CT02` | `COTRN02C` | Transaction add |
| `CB00` | `COBIL00C` | Bill payment |
| `CR00` | `CORPT00C` | Transaction reports |

## Cross-cutting behavior

- **Session context.** Sign-on and the menus set `CDEMO-USER-TYPE` (`U`/`A`) and
  `CDEMO-FROM-PROGRAM`/`CDEMO-FROM-TRANID` in the shared commarea (`COCOM01Y`). Every downstream program uses
  these for "return to caller" navigation on PF3, defaulting to the main menu if unset.
- **Optimistic locking.** `COACTUPC` and `COCRDUPC` both use: fetch → edit → validate → explicit PF5
  confirmation → re-lock and re-compare against the original snapshot → `REWRITE`. A mismatch on re-compare
  ("Record changed by some one else. Please review") aborts the write.
- **Transaction ID generation.** New `TRANSACT` records (`COTRN02C`, `COBIL00C`) get their ID the same way in
  both programs: browse backward from `HIGH-VALUES` to find the current max `TRAN-ID`, then increment by 1.
  There is no separate ID-generator/counter file.
- **Known dead/unused code:** `COTRN02C` declares `ACCTDAT`/`CVACT01Y` but never reads it — transaction add
  performs no account-level balance/credit-limit/active-status check. `COCRDSLC` has an unused alternate card
  lookup by account (`CARDAIX`) never invoked from the main flow.

## COSGN00C — Sign on

- **Tran/Program:** `CC00` / `COSGN00C` (mapset `COSGN00`, map `COSGN0A`).
- **Purpose:** Authenticates a user against the security file and routes to the admin or regular main menu.
- **Screen:** User ID (8 char), Password (8 char).
- **Business rules:**
  - User ID required → "Please enter User ID ...". Password required → "Please enter Password ...".
  - Both are upper-cased before use.
  - Reads `USRSEC` keyed by user ID (`CSUSR01Y`, `SEC-USER-DATA`).
  - RESP 13 (NOTFND) → "User not found. Try again ..."; other non-zero RESP → "Unable to verify the User
    ..."; found but `SEC-USR-PWD` mismatch → "Wrong Password. Try again ...".
  - On success sets `CDEMO-USER-ID`, `CDEMO-USER-TYPE` (from `SEC-USR-TYPE`) and XCTLs to `COADM01C` if
    admin, else `COMEN01C`.
- **Files:** `USRSEC` (VSAM KSDS) — `READ` only, keyed by 8-byte user ID.
- **Navigation:** Enter authenticates; PF3 shows a "thank you" screen and ends the session; other keys →
  "Invalid key pressed...".

## COMEN01C — Main menu (regular users)

- **Tran/Program:** `CM00` / `COMEN01C` (mapset `COMEN01`, map `COMEN1A`).
- **Purpose:** Displays the numbered main menu (`COMEN02Y`, 11 options) and dispatches to the chosen function.
- **Options (open to regular users):** 1 Account View (`COACTVWC`) · 2 Account Update (`COACTUPC`) · 3 Credit
  Card List (`COCRDLIC`) · 4 Credit Card View (`COCRDSLC`) · 5 Credit Card Update (`COCRDUPC`) · 6 Transaction
  List (`COTRN00C`) · 7 Transaction View (`COTRN01C`) · 8 Transaction Add (`COTRN02C`) · 9 Transaction Reports
  (`CORPT00C`) · 10 Bill Payment (`COBIL00C`) · 11 Pending Authorization View (`COPAUS0C`).
- **Business rules:**
  - Option must be numeric, 1–option-count, non-zero → "Please enter a valid option number...".
  - Admin-only options blocked for regular users → "No access - Admin Only option...".
  - Target program not installed in CICS → "This option `<name>` is not installed..." (red).
  - Program name starting `DUMMY` → "This option `<name>` is coming soon ..." (green).
- **Navigation:** Enter dispatches via XCTL, carrying `CDEMO-FROM-TRANID`/`CDEMO-FROM-PROGRAM`; PF3 signs off
  to `COSGN00C`.

## COADM01C — Admin menu

- **Tran/Program:** `CA00` / `COADM01C` (mapset `COADM01`, map `COADM1A`).
- **Purpose:** Admin-only menu for user-security maintenance (plus stubbed Db2 transaction-type options not
  present in the base install).
- **Options (`COADM02Y`, 6):** 1 User List (`COUSR00C`) · 2 User Add (`COUSR01C`) · 3 User Update
  (`COUSR02C`) · 4 User Delete (`COUSR03C`) · 5 Transaction Type List/Update — Db2 (`COTRTLIC`) · 6
  Transaction Type Maintenance — Db2 (`COTRTUPC`).
- **Business rules:**
  - Numeric-range validation identical to `COMEN01C`.
  - `PGMIDERR` on a missing program (e.g. options 5/6 without the Db2 module) → "This option is not installed
    ..." (green), instead of an abend.
- **Navigation:** Enter dispatches via XCTL; PF3 returns to sign-on.

## COUSR00C — User list

- **Tran/Program:** `CU00` / `COUSR00C` (mapset `COUSR00`, map `COUSR0A`).
- **Purpose:** Browse/page `USRSEC` (10 rows/page); select a user to update or delete.
- **Screen:** optional "search User ID" filter; 10-row grid with selection flag (`U`=update, `D`=delete),
  User ID, first/last name, type.
- **Business rules:**
  - Selection flag must be `U`/`u` or `D`/`d` → "Invalid selection. Valid values are U and D".
  - Pagination via `STARTBR`/`READNEXT`/`READPREV`/`ENDBR` keyed on `SEC-USR-ID`, tracking first/last ID and
    page number in the commarea (`CDEMO-CU00-INFO`).
  - Messages: "You are at the top of the page...", "You have reached the bottom/top of the page...", "You
    are already at the top/bottom of the page...", "Unable to lookup User...".
- **Files:** `USRSEC` (`CSUSR01Y`) — browse only.
- **Navigation:** Enter + selected row XCTLs to `COUSR02C` (update) or `COUSR03C` (delete) with the selected
  user ID; PF7/PF8 page; PF3 returns to `COADM01C`.

## COUSR01C — User add

- **Tran/Program:** `CU01` / `COUSR01C` (mapset `COUSR01`, map `COUSR1A`).
- **Purpose:** Create a new `USRSEC` record.
- **Screen:** First Name, Last Name, User ID, Password, User Type — all mandatory.
- **Business rules:** each field checked in order → "`<field>` can NOT be empty...".
- **Files:** `WRITE` to `USRSEC` keyed by `SEC-USR-ID`. `DUPKEY`/`DUPREC` → "User ID already exist...";
  other error → "Unable to Add User...". Success → "User `<id>` has been added ..." (green), fields cleared.
- **Navigation:** PF3 returns to `COADM01C`; PF4 clears the screen.

## COUSR02C — User update

- **Tran/Program:** `CU02` / `COUSR02C` (mapset `COUSR02`, map `COUSR2A`).
- **Purpose:** Look up a user by ID and edit/save name, password, and type.
- **Business rules:**
  - Lookup requires User ID.
  - Save requires all 5 fields non-blank (same per-field messages as Add).
  - New values compared to the value just read; if nothing changed → "Please modify to update ..." (red), no
    write performed.
  - Read uses `READ ... UPDATE` (lock) then `REWRITE`; `NOTFND` → "User ID NOT found...".
  - Success → "User `<id>` has been updated ..." (green).
- **Files:** `USRSEC`, `READ UPDATE` / `REWRITE`.
- **Navigation:** PF3/PF12 save then return to caller (default `COADM01C`); PF4 clears; PF5 saves without
  leaving.

## COUSR03C — User delete

- **Tran/Program:** `CU03` / `COUSR03C` (mapset `COUSR03`, map `COUSR3A`).
- **Purpose:** Look up a user by ID and delete the `USRSEC` record.
- **Business rules:** User ID required. Read locks the record and prompts "Press PF5 key to delete this user
  ..."; `NOTFND` → "User ID NOT found...".
- **Files:** `USRSEC`, `READ UPDATE` then `DELETE`. Success → "User `<id>` has been deleted ..." (green);
  other error → "Unable to Update User...".
- **Navigation:** PF5 deletes; PF3/PF12 return to caller (default `COADM01C`); PF4 clears.

## COACTVWC — Account view

- **Tran/Program:** `CAVW` / `COACTVWC` (mapset `COACTVW`, map `CACTVWA`).
- **Purpose:** Look up an account by ID and display full account + primary-customer details, read-only.
- **Screen:** Account ID filter; outputs active status, current balance, credit limit, cash credit limit,
  current-cycle credit/debit, open/expiration/reissue dates, group ID, plus customer ID, SSN, FICO score,
  DOB, name, address, phones, govt ID, EFT account ID, primary-cardholder flag.
- **Business rules:** account ID required ("Account number not provided" / "No input received"); must be
  numeric, non-zero, 11 digits ("Account Filter must be a non-zero 11 digit number").
- **Files (read-only):** `CXACAIX` (card-xref by account, `CVACT03Y`) → customer ID; `ACCTDAT` (`CVACT01Y`)
  by account ID; `CUSTDAT` (`CVCUS01Y`) by customer ID.
- **Navigation:** PF3 returns to caller (default `COMEN01C`).

## COACTUPC — Account update

- **Tran/Program:** `CAUP` / `COACTUPC` (mapset `COACTUP`, map `CACTUPA`).
- **Purpose:** Fetch account + customer + card-xref records for an account, let the operator edit account and
  customer fields, validate every field, require explicit PF5 confirmation, and commit both records with
  optimistic-locking protection.
- **Screen fields:**
  - Search key: 11-digit account number.
  - Account: Active Status (Y/N), Credit Limit, Cash Credit Limit, Current Balance, Current Cycle
    Credit/Debit, Open/Expiration/Reissue Date, Account Group Id.
  - Customer: Customer Id (display), SSN, FICO score, DOB, First/Middle/Last Name, Address Line 1/2, City,
    State, Country, Zip, Phone 1/2, Government-issued Id, EFT Account Id, Primary Cardholder flag (Y/N).
- **Business rules (exact validations):**
  - Search key: blank → "Account number not provided"; non-numeric/zero → "Account Number if supplied must
    be a 11 digit Non-Zero Number".
  - Active Status / Primary Cardholder: blank → "`<field>` must be supplied."; not Y/N → "`<field>` must be
    Y or N.".
  - Credit Limit / Cash Credit Limit / Current Balance / Current-Cycle Credit/Debit: signed numeric via
    `TEST-NUMVAL-C`; blank → "`<field>` must be supplied."; invalid → "`<field>` is not valid".
  - Dates (Open/Expiry/Reissue/DOB): CCYY/MM/DD validated via `CSUTLDPY`/`CSUTLDWY` (numeric, month 1–12, day
    valid for month/leap year); DOB additionally checked as not-in-future.
  - FICO Score: 3 numeric digits, non-zero, then range 300–850 → "FICO Score: should be between 300 and
    850".
  - First/Last Name: mandatory, alphabetic + spaces only. Middle Name: same alpha rule but optional.
  - Address Line 1: mandatory; Address Line 2 optional. City (stored in `CUST-ADDR-LINE-3`): mandatory,
    alpha-only.
  - State: mandatory, alpha-only, checked against `CSLKPCDY` valid US state codes → "`<field>`: is not a
    valid state code".
  - Zip: mandatory, all-numeric, non-zero. State+Zip cross-checked against `CSLKPCDY` → "Invalid zip code
    for state".
  - Country: mandatory, alpha-only.
  - Phone 1/2: optional as a whole; if any part supplied, area code/prefix/line-number each required,
    numeric, non-zero, area code checked against the North America area-code table in `CSLKPCDY`.
  - SSN: part 1 (3 digits) required/numeric/non-zero, excluded from `000`, `666`, `900`–`999` → "SSN: First
    3 chars: should not be 000, 666, or between 900 and 999"; parts 2/3 required/numeric/non-zero only.
  - EFT Account Id: mandatory, all-numeric, non-zero. Government Issued Id: no explicit edit paragraph
    (passes through unchecked).
  - Change detection: new values compared field-by-field to originally fetched values; no change →
    "No change detected with respect to values fetched.".
  - Confirmation gate: validated + real change → "Changes validated.Press F5 to save"; only PF5 commits.
  - Optimistic concurrency check on PF5 (after locking both records): re-compares every field against the
    values captured when first shown; mismatch → "Record changed by some one else. Please review", write
    abandoned.
- **Files:** `ACCTDAT` — `READ` then `READ UPDATE`/`REWRITE`; `CUSTDAT` — same pattern; `CXACAIX` —
  read-only. Write order: lock account → lock customer → concurrency check → rewrite account → rewrite
  customer (rewrite failure on customer triggers `SYNCPOINT ROLLBACK` to undo the account rewrite).
- **Navigation:** Enter validates/redisplays; PF3 syncpoints and exits to caller; PF5 saves; PF12 cancels and
  re-fetches original data.

## COCRDLIC — Card list

- **Tran/Program:** `CCLI` / `COCRDLIC` (mapset `COCRDLI`, map `CCRDLIA`).
- **Purpose:** Lists cards in a scrollable 7-row grid, optionally filtered by account/card number; rows can
  be flagged `S` (view) or `U` (update).
- **Screen:** Account Number filter (11 digits), Credit Card Number filter (16 digits), both optional; 7 rows
  with select field, account #, card #, active status.
- **Business rules:**
  - Account filter non-blank but not numeric → "ACCOUNT FILTER,IF SUPPLIED MUST BE A 11 DIGIT NUMBER". Card
    filter non-blank but not numeric → "CARD ID FILTER,IF SUPPLIED MUST BE A 16 DIGIT NUMBER".
  - More than one row selected → "PLEASE SELECT ONLY ONE RECORD TO VIEW OR UPDATE". Select value other than
    S/U/blank → "INVALID ACTION CODE".
  - Page size fixed at 7 rows; forward paging via `STARTBR`/`READNEXT` (GTEQ) with exact-match filtering,
    peeking ahead for another page; backward paging via `READPREV` from the remembered first key.
  - PF7 on first page → "NO PREVIOUS PAGES TO DISPLAY"; PF8 at end → "NO MORE PAGES TO DISPLAY"; zero rows on
    first page → "NO RECORDS FOUND FOR THIS SEARCH CONDITION.".
- **Files:** `CARDDAT` (`CVACT02Y`) — browse only, keyed by 16-byte card number.
- **Navigation:** PF3 → `COMEN01C`; PF7/PF8 page; Enter with `S` → XCTL `COCRDSLC`; Enter with `U` → XCTL
  `COCRDUPC`, passing selected account/card.

## COCRDSLC — Card view/detail

- **Tran/Program:** `CCDL` / `COCRDSLC` (mapset `COCRDSL`, map `CCRDSLA`).
- **Purpose:** Displays full detail of a single card (embossed name, active status, expiry month/year) by
  account+card number key; read-only.
- **Screen:** Account Number (11 digits), Card Number (16 digits) as search keys; output-only embossed name,
  active status, expiry month/year.
- **Business rules:** if arriving from the Card List, passed-in keys are trusted; if entered fresh,
  account/card fields are validated (blank → "Account number not provided" / "Card number not provided";
  non-numeric → digit-count messages; both blank → "No input received"). Card lookup `NOTFND` → "Did not find
  cards for this search condition".
- **Files:** `CARDDAT` (`CVACT02Y`) — `READ` keyed by card number.
- **Navigation:** PF3 returns to caller (default `COMEN01C`); Enter re-validates/re-displays.

## COCRDUPC — Card update

- **Tran/Program:** `CCUP` / `COCRDUPC` (mapset `COCRDUP`, map `CCRDUPA`).
- **Purpose:** Fetch a card by account+card number, edit embossed name / active status / expiry month / year,
  validate, require explicit PF5 confirmation, then `REWRITE` with optimistic-locking protection.
- **Screen:** Account Number, Card Number (search keys); Embossed Name (50 chars, alpha+space only), Active
  Status (Y/N), Expiry Month, Expiry Year (expiry day is not user-editable).
- **Business rules:**
  - Search key edits same as `COCRDSLC`.
  - Change detection compares new (uppercased) values to old; no change → "No change detected with respect
    to values fetched.".
  - Name: blank → "Card name not provided"; non-alpha/non-space → "Card name can only contain alphabets and
    spaces".
  - Status: blank or not Y/N → "Card Active Status must be Y or N".
  - Expiry month: blank or not 1–12 → "Card expiry month must be between 1 and 12". Expiry year: blank or
    outside 1950–2099 → "Invalid card expiry year".
  - Once validated → "Changes validated.Press F5 to save"; only PF5 commits.
  - On PF5: lock (fail → "Could not lock record for update"); optimistic-lock re-comparison against
    originally fetched values (mismatch → "Record changed by some one else. Please review", write aborted);
    `REWRITE` (fail → "Update of record failed"); success → "Changes committed to database".
- **Files:** `CARDDAT` (`CVACT02Y`) — `READ`, `READ UPDATE`, `REWRITE`, keyed by card number.
- **Navigation:** PF3 exits to caller; PF5 saves confirmed changes; PF12 cancels and re-fetches the original
  record.

## COTRN00C — Transaction list

- **Tran/Program:** `CT00` / `COTRN00C` (mapset `COTRN00`, map `COTRN0A`).
- **Purpose:** Browse `TRANSACT` 10 rows/page, optionally starting from a given Transaction ID; select a row
  to view detail.
- **Screen:** "Search Tran ID" (16 chars, optional); 10-row grid with select flag, Transaction ID, Date,
  Description, Amount; page number.
- **Business rules:**
  - Row selection value must be `S`/`s` → "Invalid selection. Valid value is S".
  - Search Tran ID, if supplied, must be numeric → "Tran ID must be Numeric ..."; blank starts from the
    beginning of the file.
  - PF7 on page 1 → "You are already at the top of the page..."; PF8 with no more records → "You are already
    at the bottom of the page...".
- **Files:** `TRANSACT` (`CVTRA05Y`) — browse only, keyed by `TRAN-ID`.
- **Navigation:** PF3 → `COMEN01C`; PF7/PF8 page; Enter + `S` selection → XCTL `COTRN01C` with the selected
  Tran ID.

## COTRN01C — Transaction view

- **Tran/Program:** `CT01` / `COTRN01C` (mapset `COTRN01`, map `COTRN1A`).
- **Purpose:** Look up and display full detail of one transaction by Tran ID; read-only inquiry.
- **Screen:** "Enter Tran ID" (16 chars, input); display-only Card Number, Type CD, Category CD, Source,
  Description, Amount, Orig/Proc Date, Merchant ID/Name/City/Zip.
- **Business rules:** blank Tran ID → "Tran ID can NOT be empty..." (no numeric-format check, unlike
  List/Add). `NOTFND` → "Transaction ID NOT found...". Read uses the CICS `UPDATE` option though the program
  never explicitly rewrites (lock released at task-end syncpoint).
- **Files:** `TRANSACT` (`CVTRA05Y`) — `READ UPDATE`, single record.
- **Navigation:** PF3 returns to caller (typically `COTRN00C`, else `COMEN01C`); PF4 clears; PF5 → XCTL
  `COTRN00C` (back to list).

## COTRN02C — Transaction add

- **Tran/Program:** `CT02` / `COTRN02C` (mapset `COTRN02`, map `COTRN2A`).
- **Purpose:** Create a new transaction record, with cross-reference lookup, full field validation, and
  explicit Y/N confirmation before write.
- **Screen:** Account # or Card # (exactly one used as lookup key), Type CD, Category CD, Source,
  Description, Amount, Orig Date, Proc Date, Merchant ID/Name/City/Zip, Confirm (Y/N).
- **Business rules:**
  - Account ID (if entered): must be numeric → looked up in `CXACAIX` to auto-fill card number; `NOTFND` →
    "Account ID NOT found...".
  - Else Card Number (if entered): must be numeric → looked up in `CCXREF` to auto-fill account ID; `NOTFND`
    → "Card Number NOT found...".
  - Neither entered → "Account or Card Number must be entered...".
  - `ACCTDAT`/`CVACT01Y` are declared but never actually read — no balance/credit-limit/active-status check
    against the account master occurs here (see cross-cutting notes above).
  - Type CD, Category CD, Source, Description, Amount, Orig Date, Proc Date, Merchant ID, Merchant Name,
    Merchant City, Merchant Zip are all mandatory.
  - Type CD and Category CD must be numeric; no lookup against a reference table is performed.
  - Amount must match `[+-]NNNNNNNN.NN` → "Amount should be in format -99999999.99", then renormalized via
    `NUMVAL-C`.
  - Orig/Proc Date must match `YYYY-MM-DD` → "`<field>` should be in format YYYY-MM-DD", then validated via
    `CSUTLDTC` → "`<field>` - Not a valid date...".
  - Merchant ID must be numeric.
  - Confirm must be Y/y to proceed ("Confirm to add this transaction..." if blank/N; "Invalid value. Valid
    values are (Y/N)..." otherwise).
  - New Transaction ID: `STARTBR`+`READPREV` from `HIGH-VALUES` to find the current max `TRAN-ID`, +1.
  - Write success → "Transaction added successfully.  Your Tran ID is `<id>`." (green); `DUPKEY`/`DUPREC` →
    "Tran ID already exist..."; other error → "Unable to Add Transaction...".
- **Files:** `TRANSACT` (`CVTRA05Y`) — browse (max-ID) + `WRITE`; `CXACAIX`/`CCXREF` (`CVACT03Y`) — read-only
  cross-reference lookups; `ACCTDAT`/`CVACT01Y` declared but unused.
- **Navigation:** PF3 → caller (default `COMEN01C`); PF4 clears screen; PF5 "Copy Last Tran." pre-fills
  fields from the most recent transaction on file. Redisplays itself after a successful add.

## COBIL00C — Bill payment

- **Tran/Program:** `CB00` / `COBIL00C` (mapset `COBIL00`, map `COBIL0A`).
- **Purpose:** Pays off an account's entire current balance in one online transaction — always full balance,
  no partial-payment option.
- **Screen:** Account ID (may be pre-filled from a prior screen); Current Balance (display); Confirm Y/N.
- **Business rules:**
  - Account ID blank → "Acct ID can NOT be empty...".
  - Confirm: `Y`/`y` → proceeds to look up account and pay; `N`/`n` → clears screen; blank (first pass) →
    looks up account and shows balance without paying; other → "Invalid value. Valid values are (Y/N)...".
  - `ACCT-CURR-BAL <= 0` → "You have nothing to pay...".
  - If confirm not yet given → "Confirm to make a bill payment...".
  - On confirmed pay: looks up card number via `CXACAIX`; determines next `TRAN-ID` via `STARTBR`/`READPREV`
    from `HIGH-VALUES` (+1; starts at 1 if empty); builds a transaction with `TRAN-TYPE-CD='02'`,
    `TRAN-CAT-CD=2`, `TRAN-SOURCE='POS TERM'`, `TRAN-DESC='BILL PAYMENT - ONLINE'`, `TRAN-AMT` = full current
    balance, merchant fields hardcoded (`999999999`/`'BILL PAYMENT'`/`'N/A'`/`'N/A'`).
  - Writes the transaction, decrements `ACCT-CURR-BAL` to zero, rewrites `ACCTDAT`.
  - Success → "Payment successful.  Your Transaction ID is `<id>`." (green); `DUPKEY`/`DUPREC` on write →
    "Tran ID already exist...".
- **Files:** `ACCTDAT` — `READ UPDATE` (locked from first confirm-prompt read through commit) then
  `REWRITE`; `CXACAIX` — read-only; `TRANSACT` — browse for max-ID, then `WRITE`.
- **Navigation:** PF3 → caller (default `COMEN01C`); PF4 clears screen.

## CORPT00C — Transaction reports

- **Tran/Program:** `CR00` / `CORPT00C` (mapset `CORPT00`, map `CORPT0A`).
- **Purpose:** Lets the user request a Monthly, Yearly, or Custom-date-range transaction report. The program
  does **not** run the report itself — it assembles a JCL job stream and submits it to the internal reader
  via a CICS extra-partition transient data queue named `JOBS` (feeding batch program `TRANREPT` — see
  [batch-jobs.md](batch-jobs.md)).
- **Screen:** report-type selectors (Monthly / Yearly / Custom); for Custom, Start/End Date MM/DD/YYYY;
  Confirm Y/N.
- **Business rules:**
  - No report type selected → "Select a report type to print report...".
  - Monthly: 1st through last day of the current month.
  - Yearly: Jan 1 – Dec 31 of the current year.
  - Custom: each of the 6 date subfields checked for blank in sequence with its own message; then
    numeric/range checks; then both dates validated via `CSUTLDTC`.
  - Confirmation required — blank → "Please confirm to print the `<report>` report..."; `N`/`n` clears and
    aborts; other → `"<value>" is not a valid value to confirm...`.
  - On confirmed submission, builds an 80-column JCL stream (job `TRNRPT00`, JOBLIB `AWS.M2.CARDDEMO.PROC`,
    `EXEC PROC=TRANREPT`) and writes it to TDQ `JOBS` via `WRITEQ TD`. Failure → "Unable to Write TDQ
    (JOBS)...".
  - Success → "`<Report Name>` report submitted for printing ..." (green).
- **Files:** No VSAM files read/written directly; writes to TDQ `JOBS`; calls `CSUTLDTC` for date validation.
- **Navigation:** PF3 always XCTLs to `COMEN01C` (hardcoded, unlike other programs which honor
  `CDEMO-FROM-PROGRAM`).

## Source files referenced

`app/cbl/{COSGN00C,COMEN01C,COADM01C,COUSR00C,COUSR01C,COUSR02C,COUSR03C,COACTVWC,COACTUPC,COCRDLIC,COCRDSLC,COCRDUPC,COTRN00C,COTRN01C,COTRN02C,COBIL00C,CORPT00C}.cbl`,
the corresponding `app/bms/*.bms` maps, `app/csd/CARDDEMO.CSD`, and the copybooks in `app/cpy/`.
