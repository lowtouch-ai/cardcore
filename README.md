# CardCore

**COBOL to Java, story by story.**

How Krithi ports CardCore: PRD extraction, sprint planning and governed delivery, two stories at a time.

`github.com/lowtouch-ai/cardcore`

---

## The codebase

CardCore is a fork of AWS CardDemo into the lowtouch-ai GitHub org, renamed. It is a credit card
management system originally built by AWS to exercise modernization tooling.

| | |
|:--|:--|
| **Real mainframe stack** | COBOL, CICS, VSAM and JCL, batch and online — the same shape as the client estate. |
| **Apache 2.0 licensed** | Forkable and demonstrable with no licensing conversation. |
| **Built for this purpose** | AWS wrote it specifically to exercise modernization tooling, so the scenarios are credible. |
| **Known to the field** | AWS Transform demos run on the same codebase, which makes results comparable. |

Synthetic data by design: the full pipeline can be shown without touching anyone's production logic.

## One pipeline, source to pull request

Krithi runs the whole loop inside the client tenancy.

```
  CardCore                    Krithi                      Modern target
  legacy source        agentic SDLC engine          Spring Boot on Postgres
  ─────────────        ───────────────────          ──────────────────────
  COBOL                REQ agent      ──▶ PRD
  CICS          ──▶    Planning       ──▶ stories   ──▶   Spring Boot
  VSAM                 Coding         ──▶ code            Postgres
  JCL                  Testing        ──▶ evidence        Playwright
                                                          Pull requests
  read only,                                          one PR per story
  never modified
```

- Legacy source is **read only** and never modified.
- The engine runs **in the client tenancy**.
- **One pull request per story.**
- Models — Claude Code, Codex — routed per stage.
- **Human sign-off between every stage.**

## The delivery loop

Five stages, each producing an artifact the client can inspect. Every completed ticket carries the full
trail, so any story can be walked through as a demo at any time.

| Stage | | What happens |
|:--|:--|:--|
| 1 | **PRD extraction** | The REQ agent reads the CardCore source and produces a PRD, with no documentation as input. |
| 2 | **Review and PRD v2** | A human reviewer comments on the PRD; the agent folds the feedback into a revised version. |
| 3 | **Sprint plan** | The planning agent turns the approved PRD into a sprint backlog of plain-English stories. |
| 4 | **Two stories to done** | Each story is coded in Spring Boot on Postgres, 45 to 90 minutes end to end. |
| 5 | **Evidence and PRs** | Equivalence tests, Playwright screenshots and pull requests attached to every ticket. |
| ↻ | **Repeat** | Stages 4 and 5, next two stories — until every story is done and CardCore is fully ported. |

## PRD from source, no documents needed

The REQ agent reads the COBOL, CICS screens and JCL directly and writes the PRD the original team never
left behind.

- **Source is the input.** COBOL programs, copybooks, CICS maps and JCL. Nothing else exists for this
  estate.
- **Reviewable output.** A structured PRD the client's architects can mark up like any requirements
  document.
- **Feedback loop shown live.** One reviewer comment accepted and folded into PRD v2 during the
  walkthrough.
- **Why it matters.** Incomplete business logic mapping is the top cause of failed migrations. This step
  attacks it directly.

> This root README is deliberately free of functional detail. It describes how the migration runs, not
> what the application does. Behaviour lives in the source tree, and the source tree is the REQ agent's
> only input.

## Plan the sprint, build two stories

The planning agent turns the approved PRD into a sprint backlog. Two stories are taken to done in Spring
Boot on Postgres.

- **Plain-English tickets.** Developers write the story; Krithi generates every prompt internally.
- **Story 1, online path.** One CICS screen flow rebuilt as a Spring Boot service, its UI verified by
  Playwright.
- **Story 2, batch path.** One batch job rebuilt as a scheduled service, output diffed against the COBOL
  original.
- **Governed merges.** The dev branch is merged autonomously; the main branch PR waits for human review.

A story takes 45 to 90 minutes end to end. Because every ticket keeps its commits, test runs and
screenshots, walking through any completed story is a demo — no live run needed.

## Cadence

Two stories per cycle over 4 to 8 weeks, until all of CardCore is on the Java stack. Every cycle ends in
reviewed pull requests.

| When | Milestone |
|:--|:--|
| Week 1 | **First cycle** — PRD extracted, sprint planned, first two stories done. |
| Weeks 2–3 | **Online paths** — CICS screen flows ported two stories at a time. |
| Weeks 4–5 | **Batch paths** — batch jobs rebuilt as scheduled services, each output diffed against the COBOL run. |
| Weeks 6–8 | **Full application** — remaining stories, cumulative equivalence suite green, CardCore fully on Spring Boot and Postgres. |

## Evidence

Every artifact stays in the repository, timestamped, ready to open on demand.

| Artifact | Produced by |
|:--|:--|
| **PRD v1 and v2** | Extracted from source, revised through human review. |
| **Sprint backlog** | Plain-English stories with acceptance criteria, generated by the planning agent. |
| **Equivalence tests** | Generated per story against the legacy behaviour, run on every commit. |
| **Playwright evidence** | Screenshots attached to each ticket, produced by the test agent. |

Per cycle: **2** stories taken to done · **45–90 min** per story, end to end · **2 PRs** — dev merged
autonomously, main human-reviewed.

## Repository layout

Source only. Nothing in this repository compiles or runs locally; the mainframe artifacts are built and
executed on z/OS or an equivalent runtime.

| Path | Member type |
|:--|:--|
| `app/cbl/` | COBOL programs |
| `app/cpy/` | Copybooks |
| `app/bms/` | BMS map source |
| `app/cpy-bms/` | Generated symbolic maps |
| `app/jcl/`, `app/proc/` | Batch jobs and procedures |
| `app/asm/`, `app/maclib/` | Assembler utilities and macros |
| `app/csd/`, `app/ctl/`, `app/catlg/` | CICS resource definitions, control cards, catalog listings |
| `app/scheduler/` | Job scheduler definitions |
| `app/data/` | Sample data, EBCDIC and ASCII |
| `app/app-*/` | Optional modules, each mirroring the layout above |
| `samples/` | Sample compile JCL, procs and runtime bundles |
| `scripts/` | Upload, compile and job-submission helpers |
| `diagrams/` | Reference images |

## License

Apache 2.0. See [LICENSE](LICENSE) and [NOTICE](NOTICE).

CardCore is derived from AWS CardDemo, © Amazon.com, Inc. or its affiliates. Source artifacts retain the
upstream `CardDemo` naming in program banners, dataset names and CICS resource groups.
