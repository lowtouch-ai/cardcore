# Documentation

Functional documentation of the CardCore application — what it does, not how it's built or how the
migration pipeline works. (For pipeline/mechanics, see the root [README.md](../README.md) and
[CLAUDE.md](../CLAUDE.md)/[AGENTS.md](../AGENTS.md).)

This documentation was reconstructed directly from the COBOL, BMS, JCL, and copybook source in `app/` — it
is not sourced from any AWS CardDemo documentation. Keep it in sync with the source when programs, screens,
batch jobs, or record layouts change.

| Document | Covers |
|:--|:--|
| [screens-and-transactions.md](screens-and-transactions.md) | Every CICS transaction: screen fields, validation rules, error messages, navigation, and the files each program reads/writes. |
| [batch-jobs.md](batch-jobs.md) | Every batch COBOL program: the daily/monthly/weekly batch cycle, business logic (interest formula, posting validation rules, statement generation), and the JCL that runs each. |
| [data-model.md](data-model.md) | The VSAM business entities (Customer, Account, Card, Card Cross-Reference, Transaction, and reference tables), their key fields, key structures, and how they relate to each other. |
| [optional-modules.md](optional-modules.md) | Pointer to the three optional modules (VSAM/MQ, transaction-type Db2, authorization IMS/Db2/MQ) and their own READMEs. |

## What CardCore is, in one paragraph

CardCore (AWS CardDemo) is a credit-card management system: customers hold accounts, accounts have one or
more cards, and cards accrue transactions. Online CICS screens let a bank operator sign on, browse/maintain
accounts, cards, and transactions, pay a bill, and submit transaction reports; a security-user menu (admin
only) manages who can sign on. Batch jobs post the day's transactions against account balances and credit
limits, calculate monthly interest per account/category against a rate table, generate customer statements,
and produce printed reports — see [batch-jobs.md](batch-jobs.md) for the full cycle.
