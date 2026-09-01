# Optional modules

Three optional modules extend the base application. Each lives under `app/app-*/`, mirrors the base
directory layout (plus `ddl/`, `dcl/`, `ims/` as needed), and must not be required for the base application
to compile or run. Each has its own detailed README; this page is a pointer, not a duplicate.

| Module | Path | Adds |
|:--|:--|:--|
| VSAM/MQ account extractions | [`app/app-vsam-mq/README.md`](../app/app-vsam-mq/README.md) | `CDRD`/`CDRA` transactions demonstrating VSAM data retrieval over an MQ request/response pattern (system date inquiry, account details inquiry). |
| Transaction type — Db2 | [`app/app-transaction-type-db2/README.md`](../app/app-transaction-type-db2/README.md) | Db2-backed transaction type maintenance, replacing/extending the VSAM `TRANTYPE` reference file with a relational table and screens. |
| Authorization — IMS/Db2/MQ | [`app/app-authorization-ims-db2-mq/README.md`](../app/app-authorization-ims-db2-mq/README.md) | Card authorization flow spanning IMS DB, Db2, and MQ, with its own screen flow and data model — see `diagrams/auth_*.png`. |

See also the screen/flow reference images in [`diagrams/`](../diagrams/): `Admin-Menu.png`,
`Main-Menu.png`, `Signon-Screen.png`, `Application-Flow-Admin.png`, `Application-Flow-User.png`,
`auth_flow.png`, `auth_details.png`, `auth_fraud.png`, `auth_summary.png`, `db2_model.png`, `ims_model.png`,
and `CARDDEMO-DataModel.drawio`.
