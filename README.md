# Automatic Testing in Integration Hub Beta

A guide for software engineers who create, run, and maintain automatic tests in the NHS Wales
[Integration Hub Beta](https://github.com/DHCW-Digital-Health-and-Care-Wales/Integration-Hub-Beta)
codebase.

## Summary

Integration Hub Beta is a Python 3.13 microservices platform that receives HL7 v2, SOAP/XML and
REST messages from NHS Wales clinical systems, validates and transforms them, and routes them
through Azure Service Bus to destinations such as the Master Patient Index (MPI). Its automatic
testing is built from many small, independent suites. 37 components each have their own `tests/`
directory, about 212 test modules and about 2,420 test functions. They run per component inside
each component's own `uv` virtual environment, mostly with the standard library `unittest`
runner. Five components use pytest. Nearly all tests are offline unit tests that replace Azure
Service Bus, Cosmos DB, SQL, sockets and HTTP with `unittest.mock`. Three opt-in integration tests
talk to real local emulators, but only when an environment variable switches them on. Every pull
request into `main`, `develop` or `release-*` runs one Azure DevOps job that checks 27 components
in sequence. Each component gets ruff, bandit, `uv audit`, mypy and its unit tests. Every merge to
`main` runs a coverage pipeline over 32 components and publishes a merged Cobertura report and a
trend dashboard. Each service's container build pipeline runs the same quality gate again before
it builds an image. There is no automated end-to-end suite in this repository. End-to-end testing
is done by hand against a Docker/Podman Compose stack that `just` drives, with a Service Bus
emulator, mock receivers and sample messages. A separate, private `Integration-Hub-tests`
repository is linked as the home of "automated tests" but could not be examined for this document.
The test suites are large and generally good, but the configuration around them has drifted. The
list of components each pipeline tests differs between pipelines. `check.sh` scripts and CI run
different commands. There is a mix of unittest and pytest, some test files are duplicated, and
several components are never tested in pull-request validation. **Part 1 (Description)** records
exactly how things are. **Part 2 (Recommendations)** gives opinionated advice on how to write,
run and maintain tests and how to close those gaps.

---

## About this document

### Sources and snapshot

Everything in Part 1 was gathered by reading the repository at:

- **Repository:** `DHCW-Digital-Health-and-Care-Wales/Integration-Hub-Beta`
- **Branch:** `main`
- **Commit:** [`636a050`](https://github.com/DHCW-Digital-Health-and-Care-Wales/Integration-Hub-Beta/tree/636a050fe5f9f0e5dd10045e1af99214c681637f)
  ("feat: add LIMS WRRS HL7 transformer to classify A31/A40 messages (#418)"), committed 2026-10-27
- **Researched on:** 2026-10-28

The codebase moves quickly, so counts, file names and pipeline lists will drift. Wherever a number
appears (for example "98 tests"), it is a snapshot at that commit.

### Method

- File and directory listings, `pyproject.toml` files, `check.sh` scripts, Azure DevOps YAML,
  Docker Compose files, `justfile`s and READMEs were read directly.
- **Test counts** were made with a static search for `def test_` and `async def test_` in each
  `tests/` directory. This gives a close approximation of what the test runners collect. It can
  differ slightly from the runners: it does not count parameterised expansions or `subTest`
  iterations, and it counts tests that are skipped at runtime.
- **The tests were not executed** while preparing this document. Statements such as "this test
  is skipped unless…" come from reading the code, not from observed runs.
- The private `Integration-Hub-tests` repository could not be read. GitHub returned *"The
  'DHCW-Digital-Health-and-Care-Wales' organization has enabled or enforced SAML SSO"*. See
  [§14](#14-the-external-integration-hub-tests-repository).

### Conventions used here

- Paths are relative to the repository root, for example `servers/hl7_server/tests/`.
- "Component" means any directory with its own `pyproject.toml`: a service, a shared library, the
  dashboard, the lookup service or a developer tool.
- Shell commands assume you are in the component's directory unless stated otherwise. They assume
  `uv` is installed.
- **Part 1** contains only facts that can be checked against the repository. **Part 2** contains
  opinions and advice, and is labelled as such.

---

## Contents

**Part 1 — Description**

1. [The system under test](#1-the-system-under-test)
2. [Testing at a glance](#2-testing-at-a-glance)
3. [Toolchain](#3-toolchain)
4. [Component inventory](#4-component-inventory)
5. [How a component's tests are organised](#5-how-a-components-tests-are-organised)
6. [Test patterns used in the codebase](#6-test-patterns-used-in-the-codebase)
7. [Component-by-component description](#7-component-by-component-description)
8. [Opt-in integration tests](#8-opt-in-integration-tests)
9. [Local quality gates and repo-wide scripts](#9-local-quality-gates-and-repo-wide-scripts)
10. [Continuous integration pipelines](#10-continuous-integration-pipelines)
11. [Code coverage](#11-code-coverage)
12. [The local end-to-end environment](#12-the-local-end-to-end-environment)
13. [Manual and exploratory testing tools](#13-manual-and-exploratory-testing-tools)
14. [The external Integration-Hub-tests repository](#14-the-external-integration-hub-tests-repository)
15. [Test data in the repository](#15-test-data-in-the-repository)
16. [Observed inconsistencies and gaps](#16-observed-inconsistencies-and-gaps)

**Part 2 — Recommendations**

17. [Guiding principles](#17-guiding-principles)
18. [Day-to-day workflows](#18-day-to-day-workflows)
19. [Writing new tests: recipes by component type](#19-writing-new-tests-recipes-by-component-type)
20. [Test data guidance](#20-test-data-guidance)
21. [Mocking guidance](#21-mocking-guidance)
22. [Integration test strategy](#22-integration-test-strategy)
23. [Automating end-to-end tests](#23-automating-end-to-end-tests)
24. [CI recommendations](#24-ci-recommendations)
25. [Coverage recommendations](#25-coverage-recommendations)
26. [Maintaining the test estate](#26-maintaining-the-test-estate)
27. [Checklist: adding a new component](#27-checklist-adding-a-new-component)
28. [Checklist: reviewing a pull request's tests](#28-checklist-reviewing-a-pull-requests-tests)
29. [Troubleshooting](#29-troubleshooting)
30. [Prioritised roadmap](#30-prioritised-roadmap)

**Appendices**

- [A. Command cheat sheet](#appendix-a-command-cheat-sheet)
- [B. Glossary](#appendix-b-glossary)
- [C. Environment variables that affect tests](#appendix-c-environment-variables-that-affect-tests)
- [D. Key files index](#appendix-d-key-files-index)
- [E. Open questions for the team](#appendix-e-open-questions-for-the-team)
- [F. Complete test module inventory](#appendix-f-complete-test-module-inventory)

---

# Part 1 — Description

This part describes the repository as it is. It does not recommend anything.

## 1. The system under test

You need a working model of what the platform does to test it well. This section is a short,
test-oriented summary. The authoritative descriptions are the root `README.md`, `CLAUDE.md`,
`.github/copilot-instructions.md` (which is also the content of `AGENTS.md`) and
`docs/TRAINING.md`.

### 1.1 Purpose

The Integration Hub carries clinical messages between disparate NHS Wales systems. Its main
destination today is the **MPI (Master Patient Index)**. It also delivers to downstream systems
that subscribe to MPI updates, to the WRRS database, and (in development) to PROMS. It replaces
proprietary integration engines and runs on **Azure Container Apps**, with **Azure Service Bus**
as the backbone between microservices.

### 1.2 The message pipeline

Most flows follow the same four-stage pattern:

```
Source system ──► Server ──► Service Bus ──► Transformer ──► Service Bus ──► Sender ──► Destination
               (MLLP/REST/  (pre-transform                 (post-transform           (MLLP/SOAP/
                SOAP ingress) queue or topic)               queue or topic)           HTTP/SQL)
```

1. **Servers** (`servers/`) accept inbound traffic.
   - `hl7_server` accepts HL7 v2 over **MLLP** (TCP framing with `0x0B … 0x1C 0x0D`).
   - `rest_server` accepts HL7-over-REST, SOAP/XML and plain XML.
   - `hl7_rest_server` and `hl7_soap_server` are older ingress services that are being replaced by
     `rest_server`.

   A server validates each message, publishes it to Service Bus and sends back an HL7
   acknowledgement: `ACK` with `MSA-1 = AA` on success, or a negative acknowledgement (NACK).
2. **Transformers** (`transformers/`) read from a pre-transform queue or topic, apply
   source-specific mapping rules, and publish to a post-transform queue or topic. Examples are
   date format changes for PHW, WRDS reference-data lookups for Chemocare and PIMS, and
   XML-to-FHIR conversion for PROMS. Several flows have no transformer.
3. **Senders** (`senders/`) read from Service Bus and deliver to the destination:
   - `hl7_sender` and `hl7_subscription_sender` send MLLP to MPI or downstream systems and
     interpret the ACK that comes back.
   - `soap_sender` and `soap_subscription_sender` send SOAP over HTTP.
   - `db_sender` writes to SQL.
4. **Message processing** (`message_processing/`) supports the pipeline.
   - `message_store_service` persists every message to SQL.
   - `message_replay_job` replays stored messages onto a priority queue.

Cross-cutting behaviour lives in **shared libraries** (`shared_libs/`): Service Bus clients,
health checks, event and audit logging, metrics, OpenTelemetry, HL7 validation, HL7 field
utilities, the transformer base class, the processing loop, and WRDS reference-data access.

### 1.3 Integration flows

The table below combines the root `README.md` and `local/README.md`.

| Flow | Ingress | Transformer | Sender | Destination |
|---|---|---|---|---|
| PHW → MPI | `hl7_server` (MLLP, port 2575) | `hl7_phw_transformer` | `hl7_sender` | MPI |
| CAV Paris → MPI | `hl7_server` (MLLP, 2577) | none | `hl7_sender` (topic-based) | MPI |
| ChemoCare → MPI | `hl7_server` (MLLP, 2578) | `hl7_chemo_transformer` | `hl7_sender` | MPI |
| PIMS → MPI | `hl7_server` (MLLP, 2579) | `hl7_pims_transformer` | `hl7_sender` | MPI |
| Mosaiq → MPI | `hl7_server` (MLLP, 2583) | none | `hl7_sender` (topic-based) | MPI |
| WDS → MPI | `hl7_server` (MLLP, 2582) | none locally (`hl7_wds_transformer` exists) | `hl7_sender` | MPI |
| RISP → MPI | `rest_server` (HL7-over-REST) | `hl7_core_reference_transformer` | `hl7_sender` | MPI (also fans out to WRRS) |
| RISP → WRRS | `rest_server` | none | `db_sender` | WRRS database |
| LIMS → MPI | `rest_server` (SOAP/XML; assigning authority `328`) | `hl7_lims_wrrs_transformer` (classification) | `hl7_subscription_sender` | MPI |
| PMS → MPI | `rest_server` (SOAP/XML) | none | `hl7_sender` | MPI |
| WPAS → PROMS | `rest_server` (XML) | `xml_fhir_proms_transformer` (in development) | — | PROMS |
| MPI → outbound | `hl7_server` (MLLP, 2580, from MPI) | none | `hl7_subscription_sender` | Downstream systems (for example SWW Chemocare) |

### 1.4 Properties that make testing interesting

These are facts about the design that shape what tests have to cover:

- **Ordering.** Service Bus queues are configured with sessions (FIFO). Several tests assert
  session IDs on received and sent messages.
- **Acknowledgements are behaviour.** The source system decides whether to resend based on the
  ACK or NACK it receives, so ACK content (`MSA-1`, `MSA-2`, `MSH-10` echo) is tested directly.
- **At-least-once delivery.** Senders complete, abandon or dead-letter Service Bus messages
  depending on the outcome, and tests check which of these happens.
- **Sensitive data.** Logs must not contain patient-identifiable data. `event_logger_lib` has a
  redaction module with its own tests (`test_redaction.py`), and `hl7_validation` tests check
  that validation errors are redacted.
- **Configuration through environment variables only.** Every service reads `os.getenv` in an
  `app_config.py` or `config.py`. Almost every component has a `test_app_config.py`.
- **Throttling and retries.** Senders throttle to a configured messages-per-minute rate, and
  receivers retry with exponential backoff. Both have unit tests, and throttling also has an
  opt-in integration test.

---

## 2. Testing at a glance

### 2.1 Layers

The repository's automatic testing has seven layers. From fastest and most isolated to slowest
and most realistic:

| # | Layer | Where | Runs automatically? | What it catches |
|---|---|---|---|---|
| 1 | **Static analysis**: ruff (lint), mypy (types), bandit (security), `uv audit` and `pip-audit` (vulnerable dependencies) | Every component | Yes: PR validation, build pipelines, `check.sh` | Style, typing and security defects, known CVEs |
| 2 | **Unit tests** with mocked I/O | `<component>/tests/` | Yes: PR validation (27 components), build pipelines, coverage pipeline | Logic errors in mappers, validators, handlers, clients, config |
| 3 | **In-process API tests** (Flask test client, FastAPI `TestClient`) | dashboard, lookup_service, rest servers, http_mock_receiver | Yes, as part of the unit suites | Route wiring, status codes, request and response shapes |
| 4 | **Opt-in integration tests** against real local emulators | 3 test files | **No.** Skipped unless an environment variable is set | Behaviour against the real Service Bus or Cosmos emulator |
| 5 | **Local end-to-end flows** on a Compose stack | `local/` | **No.** Started by hand with `just` | Whole-flow wiring: ports, queues, env files, container builds |
| 6 | **Manual and exploratory tools** | `dev_tools/` (Component Tester, Ultra7, BusWatch), HAPI TestPanel | **No** | Ad-hoc investigation and reproducing bugs |
| 7 | **Deployed environments** (dev, load, DTE, TST, UAT) | Azure, via `release-apps.yml` | Deployment is automated; no test steps were found in the release YAML | Environment-specific behaviour |

A separate private repository, `Integration-Hub-tests`, is described in the root README as
"Automated tests". It probably adds to layers 5 and 7, but that could not be confirmed
([§14](#14-the-external-integration-hub-tests-repository)).

### 2.2 Key numbers (snapshot)

| Measure | Value |
|---|---|
| Components with a `pyproject.toml` | 37 |
| Components with a `tests/` directory | 36 of those 37 (only `dev_tools/integration_hub_tester` has none), plus `dev_tools/scripts/tests`, which has no `pyproject.toml` |
| Test modules (`test_*.py`) | ~212 |
| Test functions (static count) | ~2,420 |
| Largest suite | `dashboard` (30 modules, ~673 tests) |
| Smallest suite | `transformers/hl7_wds_transformer` (1 module, 4 tests) |
| Components using unittest as their runner | 32 |
| Components using pytest as their runner | 5: `dashboard`, `mock_receivers/http_mock_receiver`, `senders/soap_sender`, `senders/soap_subscription_sender`, `transformers/hl7_wds_transformer` |
| Components with a `check.sh` | 23 |
| Components tested in PR validation | 27 |
| Components measured by the coverage pipeline | 32 |
| Per-service `*-build.yml` pipelines that run the quality gate | 21 (plus `networktestapp-build.yml`, which has no tests) |
| Opt-in integration test files | 3 |

### 2.3 Where tests run

```
                  ┌─────────────────────────┐
 developer laptop │ uv run python -m        │  per component, by hand or via check.sh
                  │   unittest discover     │  or dev_tools/scripts/run-all-tests.sh
                  │ uv run pytest           │  or dev_tools/scripts/quality_report.py
                  └───────────┬─────────────┘
                              │ git push / PR
                              ▼
                  ┌─────────────────────────┐
 Azure DevOps     │ pr-validation.yml       │  PR → main / develop / release-*  (27 components)
                  └───────────┬─────────────┘
                              │ merge to main
                              ▼
                  ┌─────────────────────────┐   ┌──────────────────────────────┐
                  │ <service>-build.yml     │   │ code-coverage.yml            │
                  │  CodeQuality stage      │   │  coverage run per component  │
                  │  (code-quality-template)│   │  ReportGenerator merge       │
                  │  then Docker build/push │   │  ADO Coverage tab + trend    │
                  └───────────┬─────────────┘   └──────────────────────────────┘
                              ▼
                  ┌─────────────────────────┐
                  │ release-apps.yml        │  dev, load, DTE → TST → UAT (manual trigger)
                  └─────────────────────────┘
```

---

## 3. Toolchain

### 3.1 Python and uv

- **Python:** every component declares `requires-python = ">=3.13"`. Two components pin a newer
  interpreter locally with a `.python-version` file containing `3.14`: `dashboard/` and
  `dev_tools/ultra7/`. CI uses Python `3.13`.
- **uv:** all dependency management goes through [uv](https://docs.astral.sh/uv/). Each
  component has its own `pyproject.toml` and committed `uv.lock`, and gets its own `.venv` when
  you run `uv sync`. The repository has no root-level workspace, so there is no single
  environment and no single test command.
- **Shared libraries as path dependencies.** Services refer to shared libraries through
  `[tool.uv.sources]`, for example:

  ```toml
  [tool.uv.sources]
  message-bus-lib = { path = "../../shared_libs/message_bus_lib" }
  ```

  A change to a shared library is therefore picked up by a service's tests only after that
  service's environment is re-synced. `dev_tools/scripts/run-all-tests.sh` uses
  `uv sync --locked --all-groups --reinstall` for this reason. Its comment says this "bypasses
  uv's build cache, which otherwise can serve stale wheels for local shared_libs/ path
  dependencies after they change".

### 3.2 Test runners

| Runner | Used by | Invocation |
|---|---|---|
| `unittest` (standard library) | 32 components | `uv run python -m unittest discover tests` (CI adds `-v`) |
| `pytest` (`>=8.3.0`) | 5 components | `uv run pytest` (CI uses `python -m pytest tests -v`) |

Only `dashboard/pyproject.toml` has a `[tool.pytest.ini_options]` section
(`testpaths = ["tests"]`). Only `dashboard/tests/conftest.py` exists as a conftest.

**The repository documents its runner inconsistently.**

- `CLAUDE.md` and the root `README.md` say "Unit testing framework (unittest)" and give
  `uv run python -m unittest discover tests` as the test command.
- `.github/copilot-instructions.md` (the same text as `AGENTS.md`) says "Tests: pytest" and shows
  a `check.sh` pattern that ends with `uv run pytest`. It also lists `pytest>=8.3.0` in the
  standard `dev` dependency group.

In practice most components do not depend on pytest (31 of 36 lack it in their `dev` group) and
most `check.sh` files run `unittest`.

The four pytest components other than the dashboard (`http_mock_receiver`, `soap_sender`,
`soap_subscription_sender`, `hl7_wds_transformer`) contain only `unittest.TestCase` classes and
use no pytest-only features: no fixtures, `monkeypatch`, `tmp_path` or markers. Their tests can be
collected by either runner. The dashboard is different. It has 18 module-level `def test_…`
functions, uses `pytest.fixture` (including an autouse fixture in `conftest.py`) and uses
`pytest.mark.skipif`, so **it needs pytest**.

### 3.3 Mocking and helpers

- `unittest.mock` is the only mocking library in use. A static count across test files finds
  roughly 930 uses of `MagicMock`, 560 `@patch` decorators and 370 `patch.object` calls.
- No third-party mocking or test-data library is used. There is no `responses`, `respx`,
  `requests-mock`, `freezegun`, `testcontainers`, `hypothesis` or `factory_boy`. HTTP clients
  (`requests`), database drivers (`pyodbc`), Azure SDK clients and `time.sleep` are all replaced
  with `unittest.mock.patch`, or with small hand-written fakes such as `_FakeHttp` in the PIMS
  transformer tests and `FakeClock` in `lookup_service`.
- `@patch("time.sleep")` appears 33 times. Retry and backoff logic is tested without real delays.
- `subTest` is used about 68 times for table-driven cases.
- `unittest.IsolatedAsyncioTestCase` is used in `lookup_service`, which has an async Cosmos store.

### 3.4 HL7 libraries used by tests

- [`hl7apy`](https://github.com/crs4/hl7apy) is a dependency of about 20 components. Tests use it
  to build messages (`Message("ADT_A01")`, `msg.msh.msh_10 = …`) and to parse them
  (`parse_message(er7)`).
- [`python-hl7`](https://python-hl7.readthedocs.io/) (the `hl7` package) is used by two
  components. It also provides the `mllp_send` command-line tool that the local `justfile` uses to
  send MLLP messages.

### 3.5 Static analysis

| Tool | Pinned version in most `dev` groups | How CI runs it | Notes |
|---|---|---|---|
| ruff | `==0.14.9` (24 components); `==0.15.0`, `==0.15.1` or `>=0.14.8` elsewhere | `uv tool run ruff check --output-format=github .` | `[tool.ruff] line-length = 120`; lint rules `E,F,W,A,PLC,PLE,PLW,I` |
| mypy | `==1.18.2` (26 components); `==1.19.1` or `>=1.18.2` elsewhere | `uv tool run mypy <pkg> tests --ignore-missing-imports` | Root `mypy.ini` sets `ignore_missing_imports = True`; three shared libraries have a `[tool.mypy]` section |
| bandit | `==1.9.2` (25 components); `==1.9.3` or `>=1.9.1` elsewhere | `uv tool run bandit -r <pkg> tests --severity-level medium` | `[tool.bandit.assert_used] skips = ["*/test_*.py", "*/*_test.py"]`, so `assert` in tests is allowed |
| uv audit | — | `uv audit --locked --ignore GHSA-5239-wwwm-4pmq` | The ignore is commented as "Temporary ignore for no-fix advisory on transitive Pygments" |
| pip-audit | — | Build pipelines only: `uv tool run pip-audit --desc --ignore-vuln GHSA-5239-wwwm-4pmq` | Not run in PR validation |

CI installs ruff, mypy and bandit with `uv tool install <name>` **without a version**. CI
therefore uses whatever version is current when the tool cache is filled, not the version pinned
in each component's `dev` group. `check.sh` uses `uv run <tool>`, which runs the pinned version
from the component's own environment.

### 3.6 Coverage

- `coverage>=7.6.0` is in the `dev` group of 34 components. The dashboard gets it through
  `pytest-cov>=6.0.0` instead.
- 36 `pyproject.toml` files contain this identical configuration:

  ```toml
  [tool.coverage.run]
  branch = true
  omit = ["tests/*", ".venv/*"]

  [tool.coverage.report]
  skip_empty = true
  ```

  See [§11](#11-code-coverage) for the pipeline.

---

## 4. Component inventory

The table lists every component that has tests, with its runner and where it is exercised.

Column key:

- **Modules / Tests:** test module and test function counts (static).
- **`__init__`:** whether `tests/__init__.py` exists.
- **check.sh:** the runner `check.sh` invokes. "—" means there is no `check.sh`.
- **PR:** the runner used in `pipeline-ado/pr-validation.yml`. "—" means the component is not
  in PR validation.
- **Cov:** the runner used in `pipeline-ado/code-coverage.yml`. "—" means it is not measured.
- **Build:** the per-service `*-build.yml` that runs `code-quality-template.yml` for this
  component. "(pytest)" means it passes `usePytest: true`.

| Component | Modules | Tests | `__init__` | check.sh | PR | Cov | Build |
|---|---:|---:|:-:|:-:|:-:|:-:|---|
| `dashboard` | 30 | 673 | y | pytest | — | pytest | `dashboard-build.yml` (pytest) |
| `dev_tools/buswatch` | 2 | 15 | n | — | unittest | — | — |
| `dev_tools/scripts` (no pyproject) | 2 | 75 | n | — | — | — | — |
| `dev_tools/ultra7` | 12 | 106 | y | unittest | — | — | — |
| `lookup_service` | 12 | 160 | y | unittest | — | — | — |
| `message_processing/message_replay_job` | 4 | 58 | y | unittest | — | unittest | `messagereplayjob-build.yml` |
| `message_processing/message_store_service` | 6 | 39 | y | unittest | unittest | unittest | `messagestoreservice-build.yml` |
| `mock_receivers/hl7_mock_receiver` | 5 | 14 | y | — | unittest | unittest | `hl7mockreceiver-build.yml` |
| `mock_receivers/hl7_topic_mock_receiver` | 5 | 17 | y | — | unittest | — | `hl7topicmockreceiver-build.yml` |
| `mock_receivers/http_mock_receiver` | 2 | 33 | y | pytest | pytest | pytest | `httpmockreceiver-build.yml` |
| `senders/db_sender` | 4 | 29 | y | unittest | unittest | unittest | `dbsender-build.yml` |
| `senders/hl7_sender` | 7 | 60 | y | unittest | unittest | unittest | `hl7sender-build.yml` |
| `senders/hl7_subscription_sender` | 5 | 44 | y | unittest | unittest | unittest | `hl7subscriptionsender-build.yml` |
| `senders/soap_sender` | 3 | 39 | y | pytest | — | pytest | `soapsender-build.yml` |
| `senders/soap_subscription_sender` | 3 | 32 | y | pytest | — | pytest | — |
| `servers/hl7_rest_server` | 7 | 48 | y | unittest | unittest | unittest | `hl7restserver-build.yml` |
| `servers/hl7_server` | 12 | 101 | y | unittest | unittest | unittest | `hl7server-build.yml` |
| `servers/hl7_soap_server` | 4 | 16 | y | unittest | unittest | unittest | `hl7soapserver-build.yml` |
| `servers/rest_server` | 14 | 130 | y | unittest | unittest | unittest | `restserver-build.yml` |
| `shared_libs/event_logger_lib` | 2 | 31 | n | — | unittest | unittest | — |
| `shared_libs/field_utils_lib` | 1 | 32 | y | — | unittest | unittest | — |
| `shared_libs/health_check_lib` | 1 | 7 | n | — | unittest | unittest | — |
| `shared_libs/hl7_message_processor` | 6 | 59 | y | unittest | — | unittest | — |
| `shared_libs/hl7_validation` | 11 | 89 | n | — | unittest | unittest | — |
| `shared_libs/message_bus_lib` | 7 | 98 | n | unittest | unittest | unittest | — |
| `shared_libs/metric_sender_lib` | 1 | 16 | n | — | unittest | unittest | — |
| `shared_libs/otel_lib` | 1 | 24 | n | unittest | unittest | unittest | — |
| `shared_libs/processor_manager_lib` | 1 | 11 | n | — | unittest | unittest | — |
| `shared_libs/transformer_base_lib` | 3 | 16 | y | — | unittest | unittest | — |
| `shared_libs/wrds_service` | 1 | 26 | y | — | — | unittest | — |
| `transformers/hl7_chemo_transformer` | 8 | 33 | y | — | unittest | unittest | `hl7chemotransformer-build.yml` |
| `transformers/hl7_core_reference_transformer` | 3 | 20 | y | unittest | unittest | unittest | `hl7corereferencetransformer-build.yml` |
| `transformers/hl7_lims_wrrs_transformer` | 3 | 35 | y | unittest | unittest | unittest | `hl7limswrrstransformer-build.yml` |
| `transformers/hl7_phw_transformer` | 9 | 33 | y | unittest | unittest | unittest | `hl7phwtransformer-build.yml` |
| `transformers/hl7_pims_transformer` | 11 | 72 | y | — | unittest | unittest | `hl7pimstransformer-build.yml` |
| `transformers/hl7_wds_transformer` | 1 | 4 | y | pytest | — | pytest | `hl7wdstransformer-build.yml` |
| `transformers/xml_fhir_proms_transformer` | 3 | 125 | y | unittest | unittest | unittest | `promsfhirtransformer-build.yml` |

Components with **no tests**: `dev_tools/integration_hub_tester` (a GUI with a `pyproject.toml`
but no `tests/`) and `network_test_app` (no `pyproject.toml`; it has its own build pipeline).

### 4.1 Reading the inventory

Three facts stand out:

- **Not in PR validation.** These components have tests that no pull request ever runs:
  - `dashboard`
  - `lookup_service`
  - `message_replay_job`
  - `soap_sender`
  - `soap_subscription_sender`
  - `hl7_wds_transformer`
  - `hl7_message_processor`
  - `wrds_service`
  - `ultra7`
  - `dev_tools/scripts`

  Some of them are tested after merge, either by their build pipeline or by the coverage
  pipeline.
- **Not in the coverage pipeline:** `hl7_topic_mock_receiver`, `buswatch`, `lookup_service`,
  `ultra7` and `dev_tools/scripts`.
- **Not tested by any pipeline:** `lookup_service`, `ultra7` and `dev_tools/scripts`. Their
  tests run only when a developer runs them.

The full list of inconsistencies is in [§16](#16-observed-inconsistencies-and-gaps).

---

## 5. How a component's tests are organised

### 5.1 Directory layout

Every component follows the same convention, which `.github/copilot-instructions.md` describes as
"always match this when adding or modifying services":

```
service_name/
├── Dockerfile
├── pyproject.toml          # dev group: ruff, bandit, mypy, coverage (and pytest for some)
├── uv.lock
├── check.sh                # optional; local quality gate
├── README.md
├── service_name/           # the importable package (same name as the folder)
│   ├── __init__.py
│   ├── application.py | app.py
│   └── app_config.py | config.py
└── tests/
    ├── __init__.py         # present in most, missing in 9 (see below)
    ├── test_<module>.py    # usually one per source module
    └── <helpers>.py        # optional non-test modules: shared fixtures and fakes
```

Tests import the package under test with absolute imports. Because the component is installed
into its own `.venv` by `uv sync`, `from hl7_sender.app_config import AppConfig` resolves to the
local source.

### 5.2 Test discovery

- **unittest** components are discovered with `python -m unittest discover tests`. The default
  pattern is `test*.py`, and discovery starts in `tests/`. Discovery works whether or not
  `tests/__init__.py` exists. Without it, the test modules are imported as top-level modules
  rather than as `tests.<module>`.
- **Missing `tests/__init__.py`** in 9 components: `dev_tools/buswatch`, `event_logger_lib`,
  `health_check_lib`, `hl7_validation`, `message_bus_lib`, `metric_sender_lib`, `otel_lib`,
  `processor_manager_lib` and `dev_tools/scripts`.
- **Nested test packages.** `servers/rest_server/tests/hl7/` is a sub-package with its own
  `__init__.py`. It holds the tests for the HL7 pipeline inside `rest_server`, alongside the
  top-level tests for the generic pipeline.
- **File naming.** Test modules are `test_<subject>.py`. One exception in casing:
  `shared_libs/message_bus_lib/tests/test_MessageReceiverClient.py` and
  `test_MessageSenderClient.py` use CamelCase.
- **Non-test helper modules** inside `tests/` (not collected, because they don't match `test*.py`):

| File | Purpose |
|---|---|
| `servers/hl7_rest_server/tests/helpers.py` | Shared builders for the HL7 REST pipeline tests (valid ER7 and RISP sample messages, processor factories) |
| `servers/rest_server/tests/hl7/helpers.py` | The same role for `rest_server`'s `hl7` pipeline |
| `lookup_service/tests/helpers.py` | `MemoryTableStore` (in-memory store double), `SeedDirTestCase` (base class with a temporary seed directory), `make_settings`, `make_row`, `codes_definition` |
| `transformers/hl7_chemo_transformer/tests/chemo_messages.py` | Sample Chemocare ER7 messages |
| `transformers/hl7_pims_transformer/tests/pims_messages.py` | Sample PIMS ER7 messages |
| `transformers/hl7_pims_transformer/tests/fakes.py` | `IdentityLookupClient` and `StubLookupClient` test doubles for the reference-data API |
| `transformers/hl7_core_reference_transformer/tests/messages.py` | "Sample RISP-style ADT messages used across hl7_core_reference_transformer tests" |
| `transformers/hl7_lims_wrrs_transformer/tests/messages.py` | "HL7 ER7 fixtures for the LIMS WRRS transformer tests" |
| `transformers/hl7_lims_wrrs_transformer/tests/lims_xml.py` | Builds LIMS HL7 v2 XML shaped like the real LIMS SOAP payload (`Identifier`, `Patient`, `LimsMessage` dataclasses) and converts it the way the REST server does |
| `transformers/hl7_wds_transformer/tests/wds_messages.py` | "Sample WDS HL7 ER7 messages for use in tests" |
| `transformers/xml_fhir_proms_transformer/tests/wpas_messages.py` | "WPAS XML fixtures for the PROMS transformer tests" |
| `senders/hl7_sender/tests/run_integration_test.py` | A script, not a test: it runs the throttling integration test against already-running containers (see [§8.1](#81-hl7-sender-throttling-against-the-service-bus-emulator)) |
| `dashboard/tests/conftest.py` | pytest fixtures, including the autouse `_disable_cosmos` fixture |

- **File-based fixtures.** Only one non-Python file lives in any `tests/` directory:
  `transformers/hl7_phw_transformer/tests/phw-valid-message.hl7`. Every other fixture is
  defined as a Python string or builder.

### 5.3 Test class and method conventions

- Test classes are named `Test<Subject>` (for example `TestGenericHandler`, `TestAppConfig`) or
  `<Subject>Tests` (for example `HL7AckBuilderTests` and `MessageRouteTests` in the REST servers,
  and `LookupApiTests` in `lookup_service`).
- Test methods are `test_<behaviour>`, often written as a sentence, for example
  `test_valid_a28_message_returns_ack` or `test_healthz_does_not_query_azure`.
- Most modules end with the usual guard so that they can be run directly:

  ```python
  if __name__ == "__main__":
      unittest.main()
  ```

- Type annotations appear on test methods (`def test_x(self) -> None:`). CI runs mypy over
  `tests/` (see [§10.1](#101-pr-validation-pr-validationyml)), so the annotations are checked.

---

## 6. Test patterns used in the codebase

This section catalogues the recurring techniques in the existing tests, with real excerpts. When
you write a new test, these are the idioms reviewers will expect to see.

### 6.1 HL7 messages as inline ER7 strings

Most HL7 tests build their input as a Python string in **ER7** ("pipe and hat") encoding, with
segments separated by carriage return (`\r`). Two styles are common.

Concatenated string literals, each ending in `\r` (from
`servers/hl7_server/tests/test_generic_handler.py`):

```python
VALID_A28_MESSAGE = (
    "MSH|^~\\&|252|252|100|100|2025-05-05 23:23:32||ADT^A31^ADT_A05|202505052323364444|P|2.5|||||GBR||EN\r"
    "PID|1||123456^^^Hospital^MR||Doe^John\r"
)
```

A list joined with `"\r"` (from `shared_libs/hl7_validation/tests/test_phw.py`):

```python
er7 = "\r".join([
    "MSH|^~\\&|252|252|100|100|2025-05-05 23:23:30||ADT^A31^ADT_A05|"
    "202505052323300000000000|P|2.5|||||GBR||EN",
    "EVN|A05|20250502092900|20250505232330|||20250505232330",
    "PID|||8888888^^^252^PI~4444444444^^^NHS^NH||SURNAME^FORENAME",
    "PV1||"
])
```

Things to notice:

- `MSH-2` must be written as `^~\\&` in Python source, because the backslash is an escape
  character.
- `MSH-3` and `MSH-4` carry numeric **sending application and facility codes** (`252` for PHW,
  `224` for Chemocare, `169` for Paris, `285` for Mosaiq, `328` for LIMS, `140` for PMS, `349` and
  `350` for RISP, `100` for MPI). Validation and routing depend on these codes, so tests choose them
  deliberately.
- `MSH-9` (message type) and `MSH-12` (version) select the validation schema and the
  transformation path.
- Larger message sets live in helper modules (`chemo_messages.py`, `pims_messages.py`,
  `wds_messages.py`, `messages.py`) so that tests can share them.

### 6.2 HL7 messages built with hl7apy

When a test only needs a structurally valid message with a few known fields, it builds one with
hl7apy (from `senders/hl7_sender/tests/test_application.py`):

```python
from hl7apy.core import Message

hl7_message = Message("ADT_A01")
hl7_message.msh.msh_10 = "MSGID1234"
hl7_string = hl7_message.to_er7()
service_bus_message = ServiceBusMessage(body=hl7_string)
```

Transformer tests often parse a base message and then mutate one field per case:

```python
original_message = parse_message(self.base_hl7_message)
new_message = Message(version="2.5")
original_message.pid.pid_29.ts_1.value = original_value
map_pid(original_message, new_message, self.lookup_client)
self.assertEqual(get_hl7_field_value(new_message.pid, "pid_29.ts_1"), expected_value)
```

### 6.3 Mocked collaborators injected through constructors

Most services are written so that their collaborators are passed in: the Service Bus sender,
event logger, metric sender, message store client and validator. Tests give them `MagicMock`s (from
`servers/hl7_server/tests/test_generic_handler.py`):

```python
class TestGenericHandler(unittest.TestCase):
    def setUp(self) -> None:
        self.mock_sender = MagicMock()
        self.mock_event_logger = MagicMock()
        self.mock_metric_sender = MagicMock()
        self.mock_message_store = MagicMock()
        self.validator = MagicMock()
        self.handler = GenericHandler(
            VALID_A28_MESSAGE,
            self.mock_sender,
            self.mock_event_logger,
            self.mock_metric_sender,
            self.validator,
            workflow_id="test-workflow",
            sending_app="252",
            message_store_client=self.mock_message_store,
            egress_session_id="test-session",
        )
```

Assertions then check both the return value (the ACK) and the interactions:
`assert_called_once()`, `assert_not_called()` and `assert_called_once_with(...)`.

### 6.4 Patching at the point of use

When a collaborator is created inside the code under test, tests patch it where it is looked up,
as a dotted path string:

```python
ACK_BUILDER_ATTRIBUTE = "hl7_server.generic_handler.HL7AckBuilder"

with patch(ACK_BUILDER_ATTRIBUTE) as mock_builder:
    mock_instance = mock_builder.return_value
    mock_ack_message = MagicMock()
    mock_ack_message.to_mllp.return_value = "\x0bACK_CONTENT\x1c\r"
    mock_instance.build_ack.return_value = mock_ack_message
    result = self.handler.reply()
```

The most frequently patched targets across the repository are:

| Target | Approx. uses | Purpose |
|---|---:|---|
| `time.sleep` | 33 | Remove real delays from retry and backoff paths |
| `message_replay_job.db_client.pyodbc` | 19 | Fake SQL Server access |
| `message_replay_job.message_replay_job.ServiceBusClientFactory` | 18 | Fake Service Bus |
| `message_store_service.db_client.pyodbc` | 15 | Fake SQL Server access |
| `pyodbc.connect` | 6 | Fake SQL Server access (`db_sender`) |
| `message_bus_lib.servicebus_client_factory.ServiceBusClient` | 4 | Fake the Azure SDK client |

### 6.5 Mocking the Azure Service Bus SDK

`shared_libs/message_bus_lib/tests/test_MessageReceiverClient.py` shows how the SDK's
context-manager receivers are faked. It configures `__enter__.return_value` on the mock that
`get_queue_receiver()` returns:

```python
def create_message(message_id: str) -> MagicMock:
    message = MagicMock(spec=ServiceBusMessage)
    message.message_id = message_id
    return message

class TestMessageReceiverClient(unittest.TestCase):
    def setUp(self) -> None:
        self.service_bus_client = MagicMock()
        self.service_bus_receiver_client = (
            self.service_bus_client.get_queue_receiver.return_value.__enter__.return_value
        )
        self.message_receiver_client = MessageReceiverClient(self.service_bus_client, "test-queue")

    @patch("time.sleep", return_value=None)
    def test_receive_messages_calls_complete_when_valid_message(self, sleep_mock: MagicMock) -> None:
        # Arrange
        message = create_message("123")
        self.service_bus_receiver_client.receive_messages.return_value = [message]

        def processor(msg: Any) -> bool:
            return True

        # Act
        self.message_receiver_client.receive_messages(1, processor)

        # Assert
        self.service_bus_receiver_client.complete_message.assert_called_once_with(message)
        self.service_bus_receiver_client.abandon_message.assert_not_called()
        sleep_mock.assert_not_called()
```

The same file and `test_subscription_receiver_client.py` cover abandon, dead-letter, session
lock (`SessionCannotBeLockedError`), `ServiceBusError` recovery, auto-lock renewer lifecycle, and
trace-context propagation.

### 6.6 Using `side_effect` to assert ordering and failure handling

To check that one call happens before another, tests use `side_effect` callbacks that record
state. This is from `servers/hl7_server/tests/test_generic_handler.py`:

```python
def test_message_stored_before_service_bus_send(self) -> None:
    message_stored = {"value": False}

    def _store_side_effect(*args: object, **kwargs: object) -> None:
        message_stored["value"] = True

    def _send_side_effect(*args: object, **kwargs: object) -> None:
        self.assertTrue(message_stored["value"], "Message should be stored before sending to Service Bus")

    self.mock_message_store.send_to_store.side_effect = _store_side_effect
    self.mock_sender.send_text_message.side_effect = _send_side_effect

    self.handler.reply()
```

Failure paths are tested by making a collaborator raise, for example
`self.mock_message_store.send_to_store.side_effect = Exception("Store unavailable")`. The test then
asserts that an ACK is still returned (`test_message_store_failure_does_not_block_ack`).

`side_effect=AssertionError(...)` is also used to prove that something is **never** called. This
is from `dashboard/tests/test_app.py`:

```python
def test_healthz_does_not_query_azure(self, client: FlaskClient) -> None:
    with patch("dashboard.app._get_cached_status", side_effect=AssertionError("healthz should not query Azure")):
        ...
```

### 6.7 Configuration tests

Almost every service has a `test_app_config.py`. Two techniques are used.

Patching `os.getenv` with a lookup table (from `senders/hl7_sender/tests/test_app_config.py`):

```python
@patch("hl7_sender.app_config.os.getenv")
def test_read_env_config_returns_config(self, mock_getenv: Mock) -> None:
    def getenv_side_effect(name: str) -> Optional[str]:
        values = {
            "SERVICE_BUS_CONNECTION_STRING": "conn_str",
            "INGRESS_QUEUE_NAME": "ingress_queue",
            "INGRESS_SESSION_ID": "ingress_session",
            "RECEIVER_MLLP_HOST": "localhost",
            "RECEIVER_MLLP_PORT": "1234",
            "ACK_TIMEOUT_SECONDS": "30",
            # ...
        }
        return values.get(name)

    mock_getenv.side_effect = getenv_side_effect
    config = AppConfig.read_env_config()
    self.assertEqual(config.receiver_mllp_port, 1234)
```

Using `patch.dict(os.environ, {...}, clear=True)`, as in
`shared_libs/event_logger_lib/tests/test_event_logger.py`.

`servers/rest_server/tests/test_app_config.py` also has a `TestDotenvLoading` class. It covers the
`python-dotenv` behaviour that `rest_server` uses only for local runs.

### 6.8 Table-driven tests with `subTest`

There are about 68 `subTest` blocks. The usual shape (from
`transformers/hl7_pims_transformer/tests/test_pid_mapper.py`) is:

```python
for original_value, expected_value in test_cases:
    with self.subTest(original_value=original_value):
        original_message = parse_message(self.base_hl7_message)
        new_message = Message(version="2.5")
        original_message.pid.pid_29.ts_1.value = original_value
        map_pid(original_message, new_message, self.lookup_client)
        self.assertEqual(get_hl7_field_value(new_message.pid, "pid_29.ts_1"), expected_value)
```

Each iteration re-parses the base message so that one case cannot leak state into the next.

### 6.9 Hand-written fakes for external services

Instead of mocking HTTP, the PIMS transformer injects a fake lookup client (from
`transformers/hl7_pims_transformer/tests/fakes.py`):

```python
class StubLookupClient:
    """A ``ReferenceDataLookupClient`` stand-in returning preconfigured codes per (dataset, source).

    Unknown (dataset, source) pairs raise :class:`ReferenceDataLookupError`, mirroring the real
    client's "no mapping / invalid" behaviour.
    """

    def __init__(self, mappings: Dict[Tuple[ReferenceDataset, str], str]) -> None:
        self._mappings = mappings

    def lookup(self, dataset: ReferenceDataset, source_code: str) -> str:
        key = (dataset, source_code.strip())
        if key not in self._mappings:
            raise ReferenceDataLookupError(...)
        return self._mappings[key]
```

`IdentityLookupClient` in the same file echoes the source code back, so tests that don't care
about lookups stay deterministic. Other fakes:

- `lookup_service/tests/helpers.py`: `MemoryTableStore`, `SeedDirTestCase`.
- `lookup_service/tests/test_table_cache.py`: `FakeClock`.
- `dashboard/tests/test_azure_monitor.py` and `test_traces.py`: `FakeResponse` and `FakeTable`.
- `dev_tools/buswatch/tests/test_servicebus_reader.py`: `_FakeMessage`.
- `transformers/hl7_pims_transformer/tests/test_reference_data_client.py`: `_FakeResponse` and
  `_FakeHttp`.

### 6.10 Fixture-file mode for WRDS

`shared_libs/wrds_service` (the Welsh Reference Data Service client) has a built-in offline mode.
Passing `fixture_file_path` makes `WRDSService` parse a saved SOAP `GetResultSetResponse` from disk
instead of making an HTTP call:

```python
wrds = WRDSService(endpoint_url=WRDS_ENDPOINT_URL, fixture_file_path="wrds_local_fixture.xml")
rows = wrds.get_result_set(...)  # parses the fixture file, makes no network call
```

`transformers/hl7_core_reference_transformer` exposes the same mode through the
`WRDS_LOCAL_FIXTURE_PATH` environment variable. The Component Tester GUI uses it as well
([§13.1](#131-component-tester-dev_toolsintegration_hub_tester)). In fixture mode
`FromSystem`/`ToSystem` filtering is not applied.

### 6.11 In-process HTTP API tests

**FastAPI** components (`rest_server`, `hl7_rest_server`, `http_mock_receiver`, `lookup_service`)
use `fastapi.testclient.TestClient`. This is from
`servers/rest_server/tests/hl7/test_message_route.py`:

```python
class MessageRouteTests(unittest.TestCase):
    def _client(self, **kwargs: object) -> tuple[TestClient, MagicMock, MagicMock]:
        context, sender, store, _ = build_test_context(**kwargs)
        return TestClient(create_hl7_app(context)), sender, store

    def test_valid_er7_returns_201_with_raw_ack_and_sends_once(self) -> None:
        client, sender, _ = self._client()
        response = client.post("/hl7MessageReceiver", json={"messageContent": VALID_ER7_MESSAGE})
        self.assertEqual(response.status_code, 201)
        self.assertIn("MSA|AA|MSGID12345", response.text)
        sender.send_text_message.assert_called_once()

    def test_validation_failure_returns_422_envelope_without_send(self) -> None:
        client, sender, _ = self._client(hl7_version="2.9")
        response = client.post("/hl7MessageReceiver", json={"messageContent": VALID_ER7_MESSAGE})
        self.assertEqual(response.status_code, 422)
        self.assertIn("MSA|AR|MSGID12345", response.json()["ErrorMessage"])
        sender.send_text_message.assert_not_called()
```

The docstring calls these "End-to-end tests". They are end to end only inside the process: the
Service Bus sender and message store are still `MagicMock`s.

The **Flask** dashboard uses a pytest fixture around `app.test_client()` (from
`dashboard/tests/test_app.py`):

```python
@pytest.fixture()
def client() -> Generator[FlaskClient, None, None]:
    app.config["TESTING"] = True
    with app.test_client() as c:
        yield c
```

### 6.12 Testing socket and MLLP handlers without sockets

`hl7_server`'s MLLP handler derives from the `socketserver` request handler. Its tests create the
handler with `object.__new__` so that the socket-reading `__init__` never runs, then attach mock
`server`, `request` and `wfile` objects. This is from
`servers/hl7_server/tests/test_size_limited_mllp_request_handler.py`:

```python
def _create_handler_instance(self, max_size: int) -> SizeLimitedMLLPRequestHandler:
    handler = object.__new__(SizeLimitedMLLPRequestHandler)
    mock_server = Mock()
    mock_server.max_message_size_bytes = max_size
    mock_server.event_logger = self.mock_event_logger
    mock_server.ack_builder = self.ack_builder
    handler.server = mock_server
    handler.request = Mock()
    handler.wfile = Mock()
    handler.sb = b'\x0b'   # MLLP start block
    handler.eb = b'\x1c'   # MLLP end block
    handler.cr = b'\r'
    handler.encoding = 'utf-8'
    return handler
```

The sender client tests (`senders/hl7_sender/tests/test_hl7_sender_client.py`) use
`Mock(spec=socket.socket)` to simulate closed sockets, timeouts and partial reads. The mock
receivers' application tests patch `threading.Thread` so that the server loop is never started.

### 6.13 Concurrency tests

`shared_libs/message_bus_lib/tests/test_MessageSenderClient.py` starts real `threading.Thread`s
to call the sender at the same time and gathers any errors. It checks that sender recovery is
thread-safe.

### 6.14 Async tests

`lookup_service/tests/test_cosmos_store.py` uses `unittest.IsolatedAsyncioTestCase`, with
`asyncSetUp` and `asyncTearDown` around the async Azure Cosmos client. It is also an opt-in
integration test ([§8.3](#83-lookup-service-cosmos-store-against-the-cosmos-emulator)).

### 6.15 Characterisation (parity) tests

`servers/rest_server/tests/test_hl7_soap_server_parity.py` pins the behaviour of the new generic
`rest_server` pipeline to the legacy `hl7_soap_server`, which it is replacing:

> Characterization tests: rest_server's `generic` pipeline (configured per docs/rest_merge.md §9
> as CONTENT_ADAPTER=soap, VALIDATOR_TYPE=hl7-xsd, VALIDATION_SCHEMA=phw,
> ALLOWED_SOURCE_IDENTIFIERS=328) against hl7_soap_server's SoapMessageProcessor behaviour.
> hl7_soap_server is a separate deployable package (own pyproject.toml/venv) and is intentionally
> *not* imported here - the "known good" behaviour below is captured directly from reading
> hl7_soap_server/hl7_soap_server/soap_processor.py, and the sample LIMS payloads are the same
> ones used by hl7_soap_server/tests/test_soap_processor.py.

It contains `RestServerSoapParityTests` and `SoapParityGapFixesTests`. The second class records
places where the new server deliberately differs from the old one.

### 6.16 Redaction and privacy tests

`shared_libs/event_logger_lib/tests/test_redaction.py` checks that personal data is masked in
log output while the message stays debuggable:

```python
def test_pii_segments_masked(self):
    redacted = redact_hl7_message(self._adt_message())
    for pii in ("SMITH", "JOHN", "19870101", "123456", "CARDIFF", "CF10 1AA", "JANE", "WARD"):
        self.assertNotIn(pii, redacted)

def test_segment_structure_preserved(self):
    redacted = redact_hl7_message(self._adt_message())
    segments = redacted.split("\r")
    self.assertEqual([seg[:3] for seg in segments], ["MSH", "PID", "NK1", "PV1"])
    self.assertTrue(segments[1].startswith(f"PID|{REDACTION_MASK}|"))
```

`shared_libs/hl7_validation/tests/test_validate.py` has a `TestSchemaValidationErrorRedaction`
class that checks the same property for validation error messages.

### 6.17 Hermetic-by-default pytest fixtures

`dashboard/tests/conftest.py` makes every dashboard test hermetic by default:

```python
@pytest.fixture(autouse=True)
def _disable_cosmos() -> Generator[None, None, None]:
    """Disable Cosmos persistence for every test and reset the cached client."""
    cosmos_store._reset_client_for_tests()
    with patch.object(cosmos_store.config, "COSMOS_ENDPOINT", ""):
        yield
    cosmos_store._reset_client_for_tests()
```

Its module docstring explains why: "Because `dashboard/.env` configures `COSMOS_ENDPOINT` for
local development, an autouse fixture disables Cosmos persistence for every test by default".
`dashboard/tests/test_app.py` has another autouse fixture, `_stub_retry_delay_metrics`, which
keeps route tests offline. Both are pytest-only mechanisms.

### 6.18 Test-only hooks in production code

Some production modules expose functions that exist only to reset state between tests. One
example is `cosmos_store._reset_client_for_tests()` in the dashboard. They are named with a
leading underscore and a `_for_tests` suffix.

### 6.19 Tests that call private functions

Several tests import and call private helpers directly. Examples are
`hl7_sender.application._calculate_batch_size`, `_process_message`, and dashboard functions such
as `_applicable_threshold`. `docs/deep_eval.md` lists this as "Private methods tested directly…
refactoring hazard".

---

## 7. Component-by-component description

This section describes what each suite covers. Module lists come from the files. Coverage notes
come from module and class names and docstrings, plus the partly outdated observations in
`docs/deep_eval.md` (dated 2026-07-11), which is flagged where it no longer matches.

### 7.1 Shared libraries (`shared_libs/`)

Shared libraries are dependencies of every service, so a defect in one affects many flows. All 11
have tests. None has a container build pipeline: their tests run in PR validation (9 of 11) and in
the coverage pipeline (all 11). A shared library's tests do **not** run in a service's build
pipeline, even though the build pipelines trigger on `shared_libs/**` changes.

| Library | Modules → test classes | Tests | Notes |
|---|---|---:|---|
| `event_logger_lib` | `test_event_logger.py` → `TestEventLogger`; `test_redaction.py` → `TestRedactHl7Message`, `TestRedactWarningDetails` | 31 | Uses `patch.dict(os.environ)` heavily; redaction is security-relevant |
| `field_utils_lib` | `test_field_utils.py` → `TestGetHl7FieldValue`, `TestSetNestedField`, `TestCopySegmentFieldsInRange`, `TestGetCx4Hd1SegmentCodes`, … | 32 | The canonical copy (411 lines); see duplication in [§16](#16-observed-inconsistencies-and-gaps) |
| `health_check_lib` | `test_health_check_server.py` → `TestTCPHealthCheckServer` | 7 | TCP health-check endpoint |
| `hl7_message_processor` | `test_converter.py`, `test_escaping.py`, `test_processor.py`, `test_schema_resolver.py`, `test_trigger_structure_map.py`, `test_validator.py` | 59 | ER7↔XML conversion depth (ADT_A39, ORU_R01), escape sequences round-trip, schema and version resolution, strict and tolerant validation, report-file cleanup. **Not in PR validation** |
| `hl7_validation` | `test_chemo.py`, `test_mosaiq.py`, `test_paris.py`, `test_phw.py`, `test_pims.py`, `test_wds.py`, `test_standard_validation.py`, `test_validate.py`, `test_validation_result.py`, `test_convert_no_data_loss.py`, `test_convert_write_file.py` | 89 | One module per flow schema; conversion must not lose data; error redaction |
| `message_bus_lib` | `test_MessageReceiverClient.py`, `test_MessageSenderClient.py`, `test_subscription_receiver_client.py`, `test_servicebus_client_factory.py`, `test_audit_service_client.py`, `test_message_store_client.py`, `test_metadata_utils.py` | 98 | Complete, abandon and dead-letter; batching; session locks; sender recovery; concurrency; trace context |
| `metric_sender_lib` | `test_metrics_sender.py` → `TestMetricSender` | 16 | Patches `DefaultAzureCredential` |
| `otel_lib` | `test_otel.py` → `TestConfigureOtel`, `TestGetTracer`, `TestInjectExtractTraceContext`, `TestOtelCorrelationFilter` | 24 | OpenTelemetry setup and no-op fallback |
| `processor_manager_lib` | `test_processor_manager.py` → `TestProcessorManager`, `TestWrapHandler` | 11 | Processing loop and handler wrapping |
| `transformer_base_lib` | `test_app_config.py`, `test_base_transformer.py`, `test_message_processor.py` | 16 | `BaseTransformer` plumbing; `run_transformer.py` is patched in one place (`transformer_base_lib.run_transformer.ServiceBusClientFactory`) |
| `wrds_service` | `test_wrds_service.py` → `TestBuildGetResultSetEnvelope`, `TestXmlEscape`, `TestParseGetResultSetResponse`, `TestWRDSServiceGetResultSet`, … | 26 | SOAP envelope build and parse, filtering. **Not in PR validation** |

### 7.2 Servers (`servers/`)

| Server | Tests | What is covered |
|---|---:|---|
| `hl7_server` (MLLP) | 101 in 12 modules | `GenericHandler` (20 tests: ACK and NACK paths, store-before-send ordering, store failure not blocking the ACK, MPI outbound checks); `HL7AckBuilder` (`TestHL7AckBuilder`, `TestHL7AckBuilderNacks`); `HL7Validator`; PHW-specific validation; MPI outbound validation; `ErrorHandler`; custom Service Bus message properties; metadata extraction and correlation IDs; size-limited MLLP request handler and server; application wiring for queue and topic modes; `AppConfig` |
| `rest_server` (generic + HL7) | 130 in 14 modules (`tests/` + `tests/hl7/`) | Top level: `AppConfig`, `PipelineConfig`, dotenv loading; ASGI ingest, health and OpenAPI endpoints; SOAP and raw-XML content adapters; validators (`NoOp`, `Hl7Xsd`, `Hl7Apy`, `Xsd`); `RestMessageProcessor`; building adapters and validators from config; parity with `hl7_soap_server`. `tests/hl7/`: health routes and Swagger gating, ACK builder, `Hl7MessageProcessor` (including the RISP flow), ER7/XML input adapter, `POST /hl7MessageReceiver` route, RISP fan-out routing (MPI + WRRS), RISP sender, version and type validation |
| `hl7_rest_server` (legacy) | 48 in 7 modules | Nearly the same subjects as `rest_server/tests/hl7/`. The files have the same names, but every one differs a little (see [§16](#16-observed-inconsistencies-and-gaps)) |
| `hl7_soap_server` (legacy) | 16 in 4 modules | `AppConfig`, WSDL endpoint, `SoapMessageProcessor`, WSDL document builder |

The root README says `hl7_rest_server` and `hl7_soap_server` are "Due to be decommissioned and
consolidated into `servers/rest_server`".

### 7.3 Transformers (`transformers/`)

| Transformer | Tests | What is covered |
|---|---:|---|
| `hl7_phw_transformer` | 33 in 9 modules | One module per mapper (MSH, EVN, PID, additional segments), datetime transformer, date-of-death transformer, `PhwTransformer` segment order, application `process_phw_message`, `AppConfig`. Uses the only file fixture, `phw-valid-message.hl7` |
| `hl7_chemo_transformer` | 33 in 8 modules | MSH, EVN, PID, PD1 and NK1 mappers; `ChemocareTransformer`; application; **its own `test_field_utils.py`** (15 tests) |
| `hl7_pims_transformer` | 72 in 11 modules | MSH, EVN, PID (including reference lookups via `StubLookupClient`), PD1, PV1 and MRG mappers; timezone removal; reference-data HTTP client (`_FakeHttp`); `PimsTransformer`; application; **its own `test_field_utils.py`** (18 tests) |
| `hl7_core_reference_transformer` | 20 in 3 modules | `WRDSConfig`; `CoreReferenceTransformer` message-type filter and WRDS wiring; `pid_reference_mapper` (field mapping, empty and HL7-null rules, in-place mutation) |
| `hl7_lims_wrrs_transformer` | 35 in 3 modules | `MappingTypeConfig`; A31/A40 classifier, including "compacted ER7 from REST server"; message processor. Fixtures built with `lims_xml.py` mirror the real SOAP payload and the REST server's XML→ER7 conversion. Newest component at the snapshot commit |
| `hl7_wds_transformer` | 4 in 1 module | `TestWdsTransformer` only. **Not in PR validation**; tested in its build pipeline and the coverage pipeline |
| `xml_fhir_proms_transformer` | 125 in 3 modules | `test_proms_transformer.py` alone has 96 tests across classes per FHIR resource (`TestBundleEnvelope`, `TestMessageHeader`, `TestPatient`, `TestServiceRequest`, `TestPractitionerRole`, …); WPAS XML parser; wiki-derived reference tables |

### 7.4 Senders (`senders/`)

| Sender | Tests | What is covered |
|---|---:|---|
| `hl7_sender` | 60 in 7 modules | `get_ack_result` (`AA`, `AE`, `AR` outcomes); `AppConfig`; `_process_message`, main ingress setup, batch sizing; `HL7SenderClient` (socket closed detection, send and receive with `Mock(spec=socket.socket)`); `MessageThrottler`; **opt-in** throttling integration test |
| `hl7_subscription_sender` | 44 in 5 modules | Same structure as `hl7_sender`, for topic subscriptions |
| `soap_sender` | 39 in 3 modules | `MessageThrottler` (interval calculation, validation, wait); SOAP ACK processor; `SOAPSenderClient` (envelope building, WIS envelope, HTTP error mapping with patched `requests`). **Not in PR validation** |
| `soap_subscription_sender` | 32 in 3 modules | Same structure as `soap_sender`. **Not in PR validation and has no build pipeline** |
| `db_sender` | 29 in 4 modules | `AppConfig`; application; `DatabaseClient` with patched `pyodbc.connect`; MSH extractor |

### 7.5 Mock receivers (`mock_receivers/`)

These are test infrastructure: they stand in for MPI and other destinations in local Compose
flows. They have their own unit tests.

| Mock receiver | Tests | What is covered |
|---|---:|---|
| `hl7_mock_receiver` | 14 in 5 modules | `AppConfig`; `ErrorHandler`; `GenericHandler` with and without Service Bus; application with and without Service Bus (thread patched) |
| `hl7_topic_mock_receiver` | 17 in 5 modules | Same structure, publishing to a topic. **Not in the coverage pipeline** |
| `http_mock_receiver` | 33 in 2 modules | FastAPI app (health, SOAP endpoint, forwarding to Service Bus) via `TestClient`; SOAP request parser and response builder |

### 7.6 Message processing (`message_processing/`)

| Component | Tests | What is covered |
|---|---:|---|
| `message_store_service` | 39 in 6 modules | `AppConfig`; application; `DatabaseClient` (patched `pyodbc`); message record builder; message record; service init, run and `process_messages` |
| `message_replay_job` | 58 in 4 modules | `AppConfig`; `DatabaseClient` (22 tests); job run, fetch-retry, send-retry and loader (`ServiceBusClientFactory` patched 18 times); `ReplayRecord`. **Not in PR validation** |

### 7.7 Dashboard (`dashboard/`)

The NOC (Network Operations Centre) monitoring dashboard is a Flask 3 application. It has **the
largest suite in the repository**: 30 modules and about 673 tests, all run with pytest. They cover:

- **Routes.** `test_app.py` has 89 tests: pages, language switching, environment label, API
  routes and network test routes.
- **Alarms.**
  - `test_alarm1.py`, `test_alarm2.py` (36 tests) and `test_alarm3.py`.
  - `test_alarm_base.py`, `test_alarm_flow_config.py`, `test_alarm_time_utils.py`.
  - Pauses: `test_alarm_pauses.py` (68 tests), `test_alarm_pause_routes.py` and
    `test_alarm_pause_evaluation.py`.
  - `test_email_ooh_guard.py` (the out-of-hours email guard).
- **Azure integrations, all faked:**
  - `test_azure_monitor.py`, `test_traces.py`.
  - `test_service_bus.py`, `test_container_apps.py`, `test_arm.py`.
  - `test_cosmos_store.py` (38 tests), `test_credentials.py`.
  - `test_email_service.py`, including retry-with-backoff on ACS 429 responses.
- **Domain logic:**
  - `test_flows.py` (flow health calculation).
  - `test_flow_sources.py`, `test_contacts_store.py`.
  - `test_network_test.py`, `test_status_builder.py`.
  - `test_cache.py` (TTL and stale-while-revalidate).
  - `test_template_filters.py`, `test_form_utils.py`, `test_config.py`.
  - `test_seed_example_alarms.py` (tests a script under `dashboard/scripts/`).
- **Opt-in emulator integration.** `test_alarm_pause_emulator.py`; see [§8.2](#82-dashboard-alarm-pauses-against-the-cosmos-emulator).

The dashboard is **not in PR validation**. Its tests run in `dashboard-build.yml` with
`usePytest: true`, and in the coverage pipeline.

### 7.8 Lookup service (`lookup_service/`)

`lookup_service` is a reference-data lookup REST API backed by files or Cosmos DB, with an admin
GUI. It has 160 tests in 12 modules:

| Module | Subject |
|---|---|
| `test_api.py` (32) | `LookupApiTests`, `UploadApiTests`, `SettingsApiTests`, `WritesDisabledTests`, `HealthTests`, `BackendFailureTests`, Swagger gating |
| `test_gui.py` (25) | Overview, table, paging and upload pages, settings form, HTML escaping |
| `test_file_store.py` (30) | Repo seed data, file-backed store loading and validation |
| `test_keys.py` (23) | Key normalisation, canonical keys, row ID encoding, table definitions |
| `test_csv_import.py` (15) | CSV reading, row import, building definitions, transforms |
| `test_resolver.py` (10) | Lookup resolution |
| `test_config.py`, `test_seed.py`, `test_ttl.py`, `test_table_cache.py`, `test_cosmos_batching.py` | Settings, seeding, TTL, cache with `FakeClock`, Cosmos batching |
| `test_cosmos_store.py` (3) | **Opt-in** integration against the Cosmos emulator |

`lookup_service` has a `check.sh`, but **no CI pipeline runs its tests**. It is in none of PR
validation, the coverage pipeline or any build pipeline.

### 7.9 Developer tools (`dev_tools/`)

| Tool | Tests | Notes |
|---|---:|---|
| `buswatch` | 15 in 2 modules | Config helpers; Service Bus reader (emulator config parsing, receiver selection, message formatting, session error detection). In PR validation; no coverage run; no build pipeline |
| `ultra7` | 106 in 12 modules | Tk GUI for sending test messages. Tests for format detection, export, iteration specs, models, pretty printing, MLLP/REST/SOAP senders, settings, storage and themes. GUI widget tests are decorated `@unittest.skipUnless(_tk_available(), "requires a Tk display")`. Not run by any pipeline |
| `scripts` | 75 in 2 modules | Tests for `quality_report.py` (parsers for ruff, mypy, unittest, JUnit and bandit output; pytest detection) and `quality_fix.py` (version updates, requirement updates, unused-ignore removal, suppressions, `-> None` annotation). Not run by any pipeline |
| `integration_hub_tester` | none | GUI tool; see [§13.1](#131-component-tester-dev_toolsintegration_hub_tester) |

---

## 8. Opt-in integration tests

Three test files talk to real infrastructure. Each one is **skipped by default** and switched on
with environment variables. None of them is switched on in any pipeline.

| File | Switch | Talks to | Runner |
|---|---|---|---|
| `senders/hl7_sender/tests/test_throttling_local.py` | `INGRESS_QUEUE_NAME` set (plus the rest of the sender's env config) | Service Bus emulator + running sender + mock receiver | unittest |
| `dashboard/tests/test_alarm_pause_emulator.py` | `RUN_COSMOS_EMULATOR_TESTS=1` and `COSMOS_*` in `dashboard/.env` | Cosmos DB emulator | pytest |
| `lookup_service/tests/test_cosmos_store.py` | `LOOKUP_IT_COSMOS_ENDPOINT` and `LOOKUP_IT_COSMOS_KEY` | Cosmos DB emulator | unittest (`IsolatedAsyncioTestCase`) |

### 8.1 HL7 sender throttling against the Service Bus emulator

```python
@unittest.skipUnless(
    os.environ.get("INGRESS_QUEUE_NAME"),
    "Skipping integration test - environment not configured for local Docker testing"
)
class TestMessageThrottlingIntegration(unittest.TestCase):
    """Integration test for message throttling functionality using docker containers."""
```

- **What it does.** It reads the sender's real `AppConfig` from the environment. It then sends 30
  HL7 messages to the ingress queue on the Service Bus emulator and checks how long processing
  takes.
- **Expected timing.** From the docstring: "For 30 messages at 30 messages/minute with 2-second
  intervals: Expected processing time: 29 intervals * 2 seconds = 58 seconds. Acceptable range:
  46-87 seconds (80%-150% of expected)".
- **Runner script.** `senders/hl7_sender/tests/run_integration_test.py` drives the test. It
  expects these containers to be running already: `sb-emulator`, `mpi-hl7-sender` and
  `mpi-hl7-mock-receiver`.
- **Profile mismatch.** No single Compose profile in `local/docker-compose.yml` starts all three
  of those containers. `mpi-hl7-sender` is in `phw-to-mpi`, `chemo-to-mpi` and others, while
  `mpi-hl7-mock-receiver` is only in `mpi-to-topic`. Since the topic-based mock receiver was
  introduced, the main flows use `mpi-hl7-topic-mock-receiver`.
- **Duration.** The test takes about a minute.

### 8.2 Dashboard alarm pauses against the Cosmos emulator

From the module docstring:

> Unlike the unit tests, nothing in the pause/config/state storage path is mocked — pauses are
> created and cancelled through the Flask routes, persisted to the emulator, read back by the
> Alarm 3 evaluator, and the alert email call is captured. Only Log Analytics (failure counts) and
> the outbound email transport are stubbed.

How to run it:

```bash
docker compose --profile dashboard up -d cosmos-emulator    # from local/
RUN_COSMOS_EMULATOR_TESTS=1 uv run pytest tests/test_alarm_pause_emulator.py -v   # from dashboard/
```

According to `dashboard/README.md`, the tests have these safeguards:

- They refuse to run unless `COSMOS_ENDPOINT` is a local emulator address. The allowed hosts are
  `localhost`, `127.0.0.1` and `cosmos-emulator`.
- They skip if the emulator is unreachable.
- Each test uses a throwaway `itest-<id>` flow and removes its rule, alarm state and pauses
  afterwards.
- They skip if an active all-flows pause exists, for example one created by
  `seed_example_alarms.py --with-pauses`.

### 8.3 Lookup service Cosmos store against the Cosmos emulator

```bash
LOOKUP_IT_COSMOS_ENDPOINT=https://localhost:8081 \
LOOKUP_IT_COSMOS_KEY=<emulator key> \
uv run python -m unittest tests.test_cosmos_store
```

- Each run creates a throwaway database named `lookup-it-<8 hex chars>` and deletes it in
  `asyncTearDown`.
- TLS verification is disabled (`cosmos_disable_ssl_verify=True`, `connection_verify=False`)
  because the emulator uses a self-signed certificate.

### 8.4 What is not covered by integration tests

No automated test in this repository runs against:

- a real or emulated SQL Server (`message_store_service`, `message_replay_job`, `db_sender` all
  patch `pyodbc`);
- a real MLLP socket pair, meaning a server and a sender talking to each other over TCP;
- a real HTTP SOAP endpoint (for example `soap_sender` → `http_mock_receiver`);
- WRDS or the PIMS reference-data API;
- a whole flow, from server through Service Bus and transformer to sender and receiver.

These are exercised by hand through the local Compose stack ([§12](#12-the-local-end-to-end-environment)),
and possibly by the external tests repository ([§14](#14-the-external-integration-hub-tests-repository)).

---

## 9. Local quality gates and repo-wide scripts

### 9.1 Per-component `check.sh`

23 components have a `check.sh` at their root. `.github/copilot-instructions.md` describes it as
the "Local quality gate: ruff → bandit → mypy → pytest… run before committing". Each script
starts with `#!/bin/bash` and `set -e`, so the first failing step stops it. Run it from the
component's directory:

```bash
cd servers/hl7_server
bash check.sh
```

The documented pattern is:

```bash
#!/bin/bash
set -e
uv run ruff check
uv run bandit -r service_name/ tests/
uv run mypy --ignore-missing-imports service_name/
uv run pytest
```

In practice, all 23 scripts are slightly different from one another: no two have the same
checksum. They fall into these groups:

| Variant | Components | Differences from the documented pattern |
|---|---|---|
| A. unittest, mypy over package **and tests** | `hl7_server`, `hl7_rest_server`, `hl7_soap_server`, `rest_server`, `hl7_sender`, `hl7_subscription_sender`, `db_sender`, `lookup_service` | Last line is `uv run python -m unittest discover tests` |
| B. unittest, mypy over **package only** | `message_store_service`, `message_replay_job`, `hl7_core_reference_transformer`, `hl7_lims_wrrs_transformer`, `hl7_phw_transformer`, `xml_fhir_proms_transformer` | mypy does not check `tests/`. `message_replay_job` also runs `uv sync` first |
| C. `PACKAGE=` variable, unittest, mypy over package and tests | `hl7_message_processor`, `message_bus_lib`, `otel_lib` | Defines `PACKAGE=<name>` and reuses it |
| D. pytest, mypy over package only | `dashboard`, `http_mock_receiver`, `soap_sender`, `soap_subscription_sender`, `hl7_wds_transformer` | Last line is `uv run pytest` |
| E. no bandit | `dev_tools/ultra7` | ruff → mypy → unittest only |

**Components without a `check.sh`:**

- `dev_tools/buswatch`, `hl7_mock_receiver`, `hl7_topic_mock_receiver`
- `event_logger_lib`, `field_utils_lib`, `health_check_lib`, `hl7_validation`,
  `metric_sender_lib`, `processor_manager_lib`, `transformer_base_lib`, `wrds_service`
- `hl7_chemo_transformer`, `hl7_pims_transformer`

**How `check.sh` differs from CI.** It is not an exact local copy of the CI gate:

| Aspect | `check.sh` | CI (`pr-validation.yml`, `code-quality-template.yml`) |
|---|---|---|
| Tool versions | Pinned versions from the component's `dev` group (`uv run`) | Unpinned latest from `uv tool install` / `uv tool run` |
| ruff output | Default | `--output-format=github` |
| bandit severity | Default (reports low, medium and high) | `--severity-level medium` |
| mypy scope | Package only in variants B and D | Always `<package> tests` when `tests/` exists |
| Dependency audit | Not run | `uv audit --locked` (and `pip-audit` in build pipelines) |
| Test verbosity | Default | `-v` |

Two consequences follow:

- A component can pass `check.sh` and still fail CI on **mypy errors in `tests/`** (variants B
  and D) or on **dependency advisories**.
- A component can fail `check.sh` on a **low-severity bandit finding** that CI would ignore.

### 9.2 Repo-wide scripts in `dev_tools/scripts/`

| Script | What it does |
|---|---|
| `run-all-tests.sh` | Finds every directory up to depth 3 with both a `pyproject.toml` and a `tests/` directory. For each one, runs `uv sync --locked --all-groups --reinstall && uv run python -m unittest discover tests -v`, then prints `ALL PASSED` or `FAILED: <list>`. Written for bash 3.2 (the macOS default) |
| `run-all-ruff.sh` | `uv run ruff check .` in every component |
| `run-all-mypy.sh` | `uv tool run --native-tls --python 3.13 mypy <pkg> tests --ignore-missing-imports` in every component, matching CI |
| `run-all-security.sh` | bandit at `--severity-level medium` over `<pkg> tests`, plus `uv audit` when it is available |
| `uv_sync_all.sh [dir]` | `uv sync` in every directory with a `pyproject.toml` (useful after pulling) |
| `upgrade-package.ps1` | PowerShell helper for upgrading a package across components |
| `quality_report.py` | All checks (ruff, mypy, tests, security) for all or chosen components, written to one HTML report with a JSON sidecar |
| `quality_fix.py` | Reads the JSON report and applies safe automatic fixes |

**`run-all-tests.sh` and pytest components.** The script always uses `unittest`. That is harmless
for the four pytest components that contain only `TestCase` classes. For the **dashboard** it
means:

- the 18 module-level `def test_…` functions are not collected;
- the `conftest.py` autouse fixture (`_disable_cosmos`) and the other pytest fixtures are not
  applied;
- tests that take a pytest `client` fixture argument would fail if they were collected.

`quality_report.py` does not have this problem. Its docstring says: "Tests run with pytest where
the component's check.sh does (as CI's usePytest does), otherwise unittest".

### 9.3 `quality_report.py`

`quality_report.py` is a standard-library-only script with an inline `uv run --script` header, so
it needs nothing except `uv`:

```bash
dev_tools/scripts/quality_report.py                                  # all checks, all components
dev_tools/scripts/quality_report.py --checks ruff,security dashboard senders/hl7_sender
dev_tools/scripts/quality_report.py --output /tmp/report.html
```

- The check groups are `ruff`, `mypy`, `tests` and `security` (bandit + audit).
- The default output is `dev_tools/reports/quality-report.html` with
  `dev_tools/reports/quality-report.json` beside it.
- It asks each tool for machine-readable output and renders a self-contained HTML file with no
  CDN assets.

### 9.4 `quality_fix.py`

`quality_fix.py` reads the JSON report and fixes what can be fixed safely. Its own manual is
`dev_tools/scripts/quality_fix.md`.

```bash
dev_tools/scripts/quality_report.py                          # 1. generate or refresh the report
dev_tools/scripts/quality_fix.py --safe                      # 2. dry run: show the plan
dev_tools/scripts/quality_fix.py --safe --apply --recheck    # 3. apply, then re-run affected checks
git diff                                                     # 4. review
```

Fixers:

| Flag | Effect |
|---|---|
| `--safe` | Shortcut for `--ruff --audit --audit-pin --unused-ignores` |
| `--ruff` / `--ruff-unsafe` | `ruff check --fix` (optionally with unsafe fixes) |
| `--format` | `ruff format` the files changed by this run |
| `--audit`, `--audit-pin`, `--audit-transitive`, `--audit-override`, `--allow-major` | Upgrade vulnerable packages in `uv.lock` and pin or override as needed |
| `--unused-ignores` | Remove unused `# type: ignore` comments |
| `--annotate-none` | Add `-> None` return annotations |
| `--bandit-timeouts [--timeout-seconds N]` | Add `timeout=` to HTTP calls (bandit B113) |
| `--baseline` | Suppress remaining mypy and bandit findings with `# type: ignore[...]` / `# nosec ...` |
| `--triage FILE` | Write findings left for a person to a `.md` or `.json` file |
| `--recheck` | After `--apply`, re-run the affected checks |
| `--allow-dirty` | Allow `--apply` with uncommitted changes |

Safety properties stated in its manual:

- **Dry run by default.** Nothing changes without `--apply`.
- **Clean tree required.** `--apply` refuses to run when there are uncommitted changes,
  including untracked files, unless you pass `--allow-dirty`.
- **Edits are verified.** Python edits are re-parsed with `ast` and must keep the same line
  count. TOML edits are re-parsed with `tomllib`.
- **No git operations.** It never commits, pushes or creates branches.
- **Some things are never fixed automatically:** most mypy errors, most bandit findings, **test
  failures**, advisories that have no fixed release, and "adverse-status" audit entries.

The script's own tests are in `dev_tools/scripts/tests/` (75 tests). Nothing in CI runs them.

---

## 10. Continuous integration pipelines

All CI runs in **Azure DevOps** using YAML in `pipeline-ado/`. The repository has no GitHub
Actions workflows. `.github/` contains only `copilot-instructions.md`. Jobs run on the
self-hosted agent pool `UKS-DHCW-IH-DEV-ADOManagedPool`, which the YAML calls
`POOL_NAME_NON_PROD` or `POOL_NAME`.

### 10.1 PR validation (`pr-validation.yml`)

- **Trigger:** pull requests targeting `main`, `develop` or `release-*`, when the change touches
  one of the included paths. `trigger: none`, so pushes alone don't run it.
- **Included paths:**
  - servers: `hl7_server`, `hl7_rest_server`, `rest_server`, `hl7_soap_server`
  - transformers: `hl7_phw_transformer`, `hl7_chemo_transformer`, `hl7_pims_transformer`,
    `hl7_core_reference_transformer`, `hl7_lims_wrrs_transformer`, `xml_fhir_proms_transformer`
  - senders: `hl7_sender`, `hl7_subscription_sender`, `db_sender`
  - mock receivers: all three
  - `dev_tools/buswatch`, `message_processing/message_store_service`
  - `shared_libs/**`, `pipeline-ado/**`
- **Paths not included,** so a PR touching only these does not trigger it:
  - `dashboard/`, `lookup_service/`
  - `message_processing/message_replay_job/`
  - `senders/soap_sender/`, `senders/soap_subscription_sender/`
  - `transformers/hl7_wds_transformer/`
  - `network_test_app/`, `dev_tools/` (except BusWatch)
  - `local/`, `docs/`
- **Shape:** one stage (`PRValidation`) and one job (`CodeQualityAll`) with a 30-minute timeout.
  The header comment says: "Consolidates 16 code quality checks into one job to avoid 16× agent
  allocation overhead." The list has since grown to 27.

Steps:

1. Checkout with `fetchDepth: 1`, then `UsePythonVersion@0` with 3.13.
2. `Cache@2` restores the uv cache at `$(Pipeline.Workspace)/.uv-cache`, with the key
   `uv-cache | "python-3.13" | "$(Agent.OS)" | "pr-validation"`.
3. `pip install uv`, then `uv tool install ruff`, `bandit` and `mypy`.
4. A bash loop over an `APPS` array. Each entry is
   `"appDir|displayName|banditDir|testRunner"`. For each app:
   1. Create an isolated venv at `$(Pipeline.Workspace)/.pr-validation-envs/<app>` and activate it.
   2. If `pyproject.toml` has a non-empty `dependency-groups.dev`, run
      `uv sync --active --locked --group dev`. Otherwise run `--no-dev`.
   3. **Ruff:** `uv tool run ruff check --output-format=github .`
   4. **Bandit:** `uv tool run bandit -r <banditDir> tests --severity-level medium`
   5. **UV audit:** `uv audit --locked --ignore GHSA-5239-wwwm-4pmq`
   6. **MyPy:** `uv tool run mypy <banditDir> tests --ignore-missing-imports`
   7. **Tests:** `python -m pytest tests -v` or `python -m unittest discover tests -v`, inside the
      activated venv. Tests are skipped, and the app marked failed, if dependency installation
      failed.
   8. Record pass or fail.
5. If any app failed, print the list and `exit 1`. All apps are always attempted, so one run
   reports every failure.

The `APPS` array at the snapshot has 27 entries:

```
servers/hl7_server                    servers/hl7_rest_server            servers/rest_server
transformers/hl7_phw_transformer      senders/hl7_sender                 senders/hl7_subscription_sender
senders/db_sender                     mock_receivers/hl7_mock_receiver   mock_receivers/http_mock_receiver (pytest)
mock_receivers/hl7_topic_mock_receiver transformers/hl7_chemo_transformer transformers/hl7_pims_transformer
transformers/hl7_core_reference_transformer transformers/hl7_lims_wrrs_transformer
transformers/xml_fhir_proms_transformer servers/hl7_soap_server          message_processing/message_store_service
dev_tools/buswatch                    shared_libs/message_bus_lib        shared_libs/event_logger_lib
shared_libs/health_check_lib          shared_libs/hl7_validation         shared_libs/metric_sender_lib
shared_libs/otel_lib                  shared_libs/processor_manager_lib  shared_libs/transformer_base_lib
shared_libs/field_utils_lib
```

Each step uses `##[group]` and `##[error]` logging commands, so the ADO log shows one collapsible
group per tool per app, and failures appear as errors.

### 10.2 Per-service build pipelines (`*-build.yml`)

There are 22 build pipelines, one per deployable. 21 of them run the quality gate as their first
stage:

```yaml
stages:
  - stage: CodeQuality
    jobs:
      - template: templates/code-quality-template.yml
        parameters:
          appName: 'servers/hl7_server'
          appDisplayName: 'HL7 Server'
          pythonVersion: '3.13'
          poolName: ${{ variables.POOL_NAME }}
  - stage: BuildAndPushHL7Server
    dependsOn: CodeQuality
    jobs:
      - template: templates/docker-build-push-template.yml
        ...
```

- **Triggers:** pushes to `main` and `release-*` (`pr: none`) that touch the component's
  directory, `shared_libs/**`, `pipeline-ado/templates/**` or `ca-certs/**`.
- **Path filter syntax varies.** `dashboard-build.yml`, `dbsender-build.yml`,
  `soapsender-build.yml` and `networktestapp-build.yml` use single-star patterns such as
  `dashboard/*` or `shared_libs/*`. The others use `**`.
- **Order matters.** Because Docker build depends on the CodeQuality stage, a failing test stops
  the image from being built and pushed to the Azure Container Registry (`uksdhcwihsharedcr`).
- **Runner.** Only `dashboard-build.yml` passes `usePytest: true`. `httpmockreceiver-build.yml`,
  `soapsender-build.yml` and `hl7wdstransformer-build.yml` build pytest components with the
  default unittest runner. That still collects their `TestCase` classes.
- **No build pipeline:** `soap_subscription_sender`, `lookup_service`, `dev_tools/*` and the
  shared libraries.

### 10.3 The consolidated build (`build-apps.yml`)

`build-apps.yml` has `trigger: none` and `pr: none`, so it is run by hand. Its `buildApps`
parameter lists 19 deployables. Each selected app gets its own `CodeQuality_<app>` stage (using
the same template) and a `Build_<app>` stage that depends only on that gate, so the gates run in
parallel. As in the per-service pipelines, only the dashboard sets `usePytest: true`. The comment
says: "Individual *-build.yml files remain for per-path CI triggers."

### 10.4 The quality template (`templates/code-quality-template.yml`)

Parameters: `appName`, `appDisplayName`, `pythonVersion` (default `3.13`), `poolName`,
`banditSourceDir` (defaults to the last path segment of `appName`) and `usePytest` (default
`false`).

Steps:

1. Checkout and Python setup. Restore the uv cache, keyed on the component's `pyproject.toml`.
2. `pip install --upgrade pip`, `pip install uv`, then `uv tool install` ruff, bandit, mypy and
   **pip-audit**.
3. **Install shared dependencies** into the *system* interpreter with
   `uv pip install --system -e shared_libs/<lib>/` for seven libraries: `message_bus_lib`,
   `health_check_lib`, `processor_manager_lib`, `event_logger_lib`, `field_utils_lib`,
   `transformer_base_lib` and `hl7_validation`. Then check that each one imports.
4. Install the app into the system interpreter with `uv pip install --system .` and
   `uv pip install --system --group dev .`. The second command is allowed to fail ("No dev
   dependencies found").
5. **Dependency scan:** `uv tool run pip-audit --desc --ignore-vuln GHSA-5239-wwwm-4pmq`. This
   fails the job.
6. **Code quality:** ruff, bandit (`--severity-level medium`), `uv audit --locked`, then mypy
   over `<src> tests`. Each sets `VALIDATION_FAILED`, and the step exits non-zero at the end if
   any failed.
7. **Unit tests:** with `usePytest`, `uv run --no-sync pytest tests -v`. Otherwise
   `python -m unittest discover tests -v`, using the system interpreter where everything was
   installed.

This template installs into the **system** interpreter, while PR validation uses **per-app
virtual environments with `uv sync --locked`**. The two pipelines can therefore resolve slightly
different dependency sets for the same component. The template's installs are not `--locked`.

### 10.5 Code coverage pipeline (`code-coverage.yml`)

See [§11](#11-code-coverage).

### 10.6 Release pipeline (`release-apps.yml`)

`release-apps.yml` is run by hand. It takes a `selectedApp` parameter (default `all`) and deploys
images built by the build pipelines to Azure Container Apps environments:

| Environment | Display name | Depends on previous |
|---|---|---|
| `dev` | Development | no |
| `load` | Load Testing | no |
| `dte` | DTE | yes |
| `tst` | Test | yes |
| `uat` | UAT | yes |

No test steps (smoke, health or end-to-end checks) were found in `release-apps.yml` or its
templates. A **Load Testing** environment exists, but no load-test definitions (JMeter, Locust,
k6 or similar) were found in this repository.

### 10.7 What CI does not do (facts)

- It does not publish test results (no `PublishTestResults@2`), so ADO has no Tests tab or test
  trend for unit tests.
- It does not run any test that needs a container, an emulator or a network service.
- It does not enforce a coverage threshold.
- It does not run the tests of `lookup_service`, `dev_tools/ultra7` or `dev_tools/scripts`.
- It does not check that the `APPS` arrays in `pr-validation.yml` and `code-coverage.yml` match
  each other or match the components on disk.

---

## 11. Code coverage

### 11.1 Background

`docs/code_coverage.md` is the plan for this work. It records that "before this rollout, no
coverage tooling was installed anywhere in the repo". It sets these constraints:

1. Coverage runs **after merge to `main`**, not on every PR, "keeps PR validation fast".
2. The tool is `coverage.py`, which works with both unittest and pytest.
3. The output format is **Cobertura XML**, which ADO understands natively.
4. Per-component reports are merged with **ReportGenerator**.
5. The dashboard is a set of **ADO widgets**: no external SaaS such as Codecov or Coveralls.

`docs/deep_eval.md` (2026-07-11) says "No evidence of pytest-cov or any coverage measurement in
CI". That statement predates the rollout and is **out of date**.

### 11.2 Per-component configuration

As described in [§3.6](#36-coverage), each component's `pyproject.toml` has:

```toml
[tool.coverage.run]
branch = true
omit = ["tests/*", ".venv/*"]

[tool.coverage.report]
skip_empty = true
```

- **Branch coverage is on.** A line counts as fully covered only when every branch from it has
  been taken.
- **No `source`.** The plan proposed `source = ["<service_pkg>"]`, but the files do not set it.
  The pipeline passes `--source=.` on the command line instead.
- **No `fail_under`.** No component sets a minimum.

### 11.3 The pipeline

**Trigger:**

- Merge to `main` with `batch: true`. The comment explains that this "collapses several rapid
  merges to main into a single queued run instead of one run per push (which is what caused the
  burst of ~60 runs when this pipeline was added)".
- Changes under `docs/**`, `**/*.md` or `pipeline-ado/**` are excluded.
- `pr: none`.

**Stage 1, `CollectCoverage`** (45-minute timeout). This loops over its own `APPS` array of 32
entries. A comment says it should be kept "in sync with pipeline-ado/pr-validation.yml's APPS
list", but it is not the same list ([§16](#16-observed-inconsistencies-and-gaps)). For each app:

1. Skip the app if it has no `tests/`.
2. Create and activate a venv, then `uv sync --active --locked --group dev`. If there is no dev
   group, install with `--no-dev` and add `coverage>=7.6.0` with `uv pip install`.
3. Run either `coverage run --source=. -m pytest tests -v` or
   `coverage run --source=. -m unittest discover tests -v`.
4. Run `coverage xml -o $(Build.ArtifactStagingDirectory)/coverage-raw/<app>/coverage.xml`
   **even if tests failed**: "so a single flaky/broken suite doesn't blank out the whole
   dashboard".
5. Run `coverage report` to print a table in the log.
6. A test failure produces a **warning, not a failed pipeline**: "pr-validation.yml already
   gates merges to main".
7. Publish `coverage-raw` as a pipeline artifact.

**Stage 2, `AggregateCoverage`:**

1. Download `coverage-raw`.
2. Download the previous run's `coverage-history` artifact from `main` with
   `allowPartiallySucceededBuilds: true` and `allowFailedBuilds: true`. A long comment explains
   that these flags prevent "a self-perpetuating loop where history is never found again".
3. Install .NET SDK 8 and `dotnet-reportgenerator-globaltool`.
4. Run:

   ```bash
   reportgenerator \
     -reports:"coverage-raw/**/coverage.xml" \
     -targetdir:"coverage-report" \
     -reporttypes:"HtmlInline_AzurePipelines;Cobertura;Badges;TextSummary" \
     -historydir:"coverage-history"
   ```

5. Print `Summary.txt` to the log.
6. Publish `Cobertura.xml` with `PublishCodeCoverageResults@2`. This fills the build's
   **Coverage** tab.
7. Publish `coverage-report/` (an HTML drill-down) and `coverage-history/` as artifacts.

### 11.4 Viewing coverage

Following `docs/code_coverage.md` Phase 4:

- **Trend:** an ADO dashboard ("Integration Hub — Code Quality" was the proposed name) with the
  native **Code Coverage** widget pointing at the pipeline on `main`.
- **Per-file detail:** the **Coverage** tab of any run, or the `coverage-report` artifact (open
  `index.html`).
- **Text summary:** the `AggregateCoverage` log.

Phase 5 ("Guardrails") proposed a later, opt-in floor with `coverage report --fail-under=X`,
first as a warning and later as a blocking PR gate. That had not been done at the snapshot.

### 11.5 Measuring coverage locally

The documented local commands (from `docs/code_coverage.md` Phase 1) are:

```bash
uv run coverage run -m unittest discover tests   # or: -m pytest tests
uv run coverage xml -o coverage.xml
uv run coverage report
```

`.gitignore` already excludes `htmlcov/`, `.coverage`, `.coverage.*` and `coverage.xml`.

---

## 12. The local end-to-end environment

There are no automated end-to-end tests in this repository. The **local Compose stack** in
`local/` lets a developer run whole flows on a laptop or in a Codespace, send real messages, and
watch them arrive at a mock destination. This section describes that environment because it is
the platform any future automated end-to-end suite would build on.

### 12.1 Ingredients

| File | Role |
|---|---|
| `local/docker-compose.yml` | Defines about 32 services, grouped by Compose **profiles** |
| `local/justfile` | [just](https://github.com/casey/just) recipes using **`podman-compose`** ("Podman Edition") |
| `local/justfile.docker` | The same core recipes using **`docker compose`** |
| `local/ServiceBusEmulatorConfig.json` | Queues, topics and subscriptions created in the Service Bus emulator |
| `local/env/**.env` | One env file per service (36 files), grouped into `servers/`, `transformers/`, `senders/`, `mock_receivers/`, `message_processing/`, `lookup/`, `dev_tools/` |
| `local/generate_secrets.py` + `.secrets-template` | Writes `.secrets` by appending `secrets.token_urlsafe(16)` to each template line |
| `local/sample_messages/` | Sample input messages (see [§15](#15-test-data-in-the-repository)) |
| `local/sql-scripts/` | SQL for the local Azure SQL Edge database |
| `.devcontainer/devcontainer.json` | Codespaces/Dev Container with docker-from-docker and `just` |

### 12.2 Infrastructure containers

| Service | Image | Host port | Purpose |
|---|---|---|---|
| `sb-emulator` | `mcr.microsoft.com/azure-messaging/servicebus-emulator:latest` | 5672 (AMQP) | Azure Service Bus emulator |
| `sqledge` | `mcr.microsoft.com/azure-sql-edge:latest` | 14333 → 1433 | SQL Server for the message store and the WRRS database |
| `cosmos-emulator` | Cosmos DB emulator | 8081, 1234 | Backs the dashboard and `lookup_service` |

Both images use the `:latest` tag. To connect to the Service Bus emulator from the host:

```
Endpoint=sb://127.0.0.1;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=SAS_KEY_VALUE;UseDevelopmentEmulator=true;
```

### 12.3 Service Bus emulator entities

From `local/ServiceBusEmulatorConfig.json`:

**Queues** (session-enabled unless stated):

- `local-inthub-phw-ingress`
- `local-inthub-phw-transformer-ingress`
- `local-inthub-chemo-transformer-ingress`
- `local-inthub-pims-transformer-ingress`
- `local-inthub-priority-messagequeue`
- `local-inthub-mpi-mock-receiver` (no sessions)
- `local-inthub-messagestore` (no sessions)

**Topics → subscriptions:**

| Topic | Subscriptions |
|---|---|
| `mpihl7input` | `mpihl7input-test-subscription` |
| `local-inthub-lims-transformer-ingress` | `…-ingress-sub`, `local-inthub-lims-wrrs-transformer-sub` |
| `local-inthub-lims-wrrs-classified` | `local-inthub-lims-wrrs-nhs-id-sub`, `local-inthub-lims-wrrs-a40-sub` |
| `local-inthub-pms-sender-ingress` | `…-sub` |
| `local-inthub-wpas-transformer-ingress` | `…-sub` |
| `local-inthub-core-reference-transformer-ingress` | `local-inthub-core-reference-transformer-sub` |
| `local-inthub-wrrs-outbound` | `…-sub` |
| `local-inthub-mpi-sender-ingress` | `local-inthub-mpi-sender-sub` |
| `local-inthub-risp-mpi-sender-ingress` | `…-sub` |
| `local-inthub-mosaiq-mpi-sender-ingress` | `…-sub` |
| `local-inthub-cav-paris-mpi-sender-ingress` | `…-sub` |
| `local-inthub-hl7-topic-mock-receiver` | `local-inthub-hl7-topic-mock-receiver-verifier` |

The `…-verifier` subscription on the topic mock receiver's output topic is where a flow's
**final output** can be read. That makes it the natural assertion point for an automated
end-to-end test ([§23](#23-automating-end-to-end-tests)).

### 12.4 Profiles

The Compose file uses these profile names: `phw-to-mpi`, `lims-to-mpi`, `pms-soap-to-mpi`,
`paris-to-mpi`, `chemo-to-mpi`, `pims-to-mpi`, `mosaiq-to-mpi`, `wds-to-mpi`, `risp-to-mpi`,
`risp-to-wrrs`, `hl7-rest`, `rest-to-mpi`, `wpas-to-proms`, `mpi-to-topic`,
`topic-mock-receiver`, `dashboard`, `lookup` and `everything`.

| Profile | Services |
|---|---|
| `phw-to-mpi` | `phw-hl7-server` (2575), `phw-hl7-transformer`, `mpi-hl7-sender`, `mpi-hl7-topic-mock-receiver`, `bus-watch` |
| `paris-to-mpi` | `cav-paris-hl7-server` (2577), `paris-mpi-hl7-sender`, `mpi-hl7-topic-mock-receiver`, `bus-watch` |
| `chemo-to-mpi` | `chemo-hl7-server` (2578), `chemo-hl7-transformer`, `mpi-hl7-sender`, `mpi-hl7-topic-mock-receiver`, `bus-watch` |
| `pims-to-mpi` | `pims-hl7-server` (2579), `pims-hl7-transformer`, `mpi-hl7-sender`, `mpi-hl7-topic-mock-receiver`, `bus-watch` (+ external reference-data API on host port 8085) |
| `mosaiq-to-mpi` | `mosaiq-hl7-server` (2583), `mosaiq-mpi-hl7-sender`, `mpi-hl7-topic-mock-receiver`, `bus-watch` |
| `wds-to-mpi` | `wds-hl7-server` (2582), `mpi-hl7-sender`, `mpi-hl7-topic-mock-receiver`, `bus-watch` |
| `lims-to-mpi` | `hl7-soap-server` (8082; built from `rest_server`), `lims-wrrs-hl7-transformer`, `lims-mpi-hl7-sender`, `mpi-hl7-topic-mock-receiver`, `bus-watch` |
| `pms-soap-to-mpi` | `pms-hl7-soap-server` (8086), `mpi-hl7-sender`, `mpi-hl7-topic-mock-receiver`, `bus-watch` |
| `risp-to-mpi` | `risp-hl7-rest-server` (8085), `core-reference-hl7-transformer`, `risp-mpi-hl7-sender`, `mpi-hl7-topic-mock-receiver`, `bus-watch` |
| `risp-to-wrrs` | `risp-hl7-rest-server`, `wrrs-db-sender` |
| `hl7-rest` | `risp-hl7-rest-server` (8085), `rest-hl7-server` (8083) |
| `rest-to-mpi` | `rest-hl7-server`, `mpi-hl7-sender` |
| `wpas-to-proms` | `wpas-rest-server` (8084), `mpi-hl7-sender`, `mpi-hl7-topic-mock-receiver`, `bus-watch` |
| `mpi-to-topic` | `mpi-hl7-server` (2580), `mpi-hl7-chemo-sender` (2581), `mpi-hl7-mock-receiver` (2576), `bus-watch` |
| `topic-mock-receiver` | `mpi-hl7-topic-mock-receiver` (2584 → 2581) |
| `dashboard` | `cosmos-emulator` |
| `lookup` | `cosmos-emulator`, `lookup-service` (8090) |
| `everything` | Nearly every service, **except `phw-hl7-server` and `chemo-hl7-transformer`**, which have no `everything` tag |

BusWatch runs on host port 8080. `sb-emulator` and `sqledge` have no profile, so they start with
any profile.

**Discrepancies between the Compose file and its documentation:**

- `local/README.md` says "all the listed profiles will start the **message-store-service** as
  well as it is not tagged with a profile". It also lists a `replay` profile that runs
  `message-replay-job`. **Neither service is defined in `local/docker-compose.yml` at the
  snapshot.** The justfile's `all_profiles_flags` still includes `--profile replay`, and
  `CLAUDE.md` still suggests `just run replay`.
- `local/README.md`'s profile list omits `risp-to-wrrs`, `rest-to-mpi`, `lookup` and
  `everything`. It describes `hl7-rest` differently from the Compose file.
- The README's quick-start table says `docker compose --profile "*" up -d` starts all profiles.
  The justfile has a comment that says the opposite: "`--profile "*"` is NOT a wildcard (it's
  treated as a literal, nonexistent profile name)". The Compose file provides an `everything`
  profile.

### 12.5 just recipes

`local/justfile` (Podman) and `local/justfile.docker` (Docker) share these recipes:

| Recipe | Effect |
|---|---|
| `just` / `just default` | List recipes |
| `just profiles` | Print the profile list |
| `just install` | `python3 -m pip install hl7` (provides `mllp_send`) |
| `just secrets` | `python3 generate_secrets.py > .secrets` |
| `just build <profile>` | `podman-compose --profile <p> build` |
| `just start <profile>` | Ensure secrets exist, then `up -d` |
| `just stop` | `down` across every profile |
| `just logs [services…]` | Follow logs |
| `just run [profile]` | install + secrets + start |
| `just restart <profile>` | build + stop + start (use after code changes) |
| `just clean` | stop + delete `.secrets` |

These three recipes are **only in the Podman `justfile`**:

| Recipe | Effect |
|---|---|
| `just send <file> [port=2575]` | `mllp_send --loose --file <file> --port <port> 127.0.0.1`, with CR→LF translation so the ACK is readable |
| `just send-soap [file] [port=8082]` | POST the file as `text/xml` to `http://127.0.0.1:<port>/soap`. Default file: `./sample_messages/lims-to-mpi.sample.xml` |
| `just send-rest [file] [port=8084]` | POST the file as `text/plain` to `http://127.0.0.1:<port>/hl7MessageReceiver`. Default file: `./sample_messages/wpas-to-proms.sample.hl7`, **which does not exist** (the sample is `wpas-to-proms.xml`) |

The justfile also sets `DOCKER_BUILDKIT=1` and `COMPOSE_DOCKER_CLI_BUILD=1`. Compose's
`additional_contexts` (used to bring `shared_libs` and `ca-certs` into each image) need
BuildKit.

### 12.6 A manual end-to-end run

This is the documented manual procedure, using the Docker variant:

```bash
cd local
just secrets                      # once
just start phw-to-mpi             # or: docker compose --profile phw-to-mpi up -d
just logs phw-hl7-server          # watch ingress
just send ./sample_messages/phw-to-mpi.sample.hl7          # MLLP to 2575; prints the ACK
just logs phw-hl7-transformer mpi-hl7-sender mpi-hl7-topic-mock-receiver
# then open BusWatch at http://127.0.0.1:8080 to inspect queues and topics
just stop
```

Things to check in such a run:

1. The ACK returned to `mllp_send` has `MSA|AA|<MSH-10>`.
2. The server log shows the message was validated and published.
3. The transformer log shows the message was received, transformed and published.
4. The sender log shows it was delivered and that the ACK from the mock receiver was `AA`.
5. The topic mock receiver log shows the message arrived. The message is also on the `…-verifier`
   subscription, visible in BusWatch.

After a code change, run `just restart <profile>`. This rebuilds the images, because the
containers don't mount source code.

### 12.7 Other manual senders

- **HAPI TestPanel** (Java): `local/README.md` has step-by-step instructions for connecting a
  "Single Port MLLP" sending connection to `localhost:2575`.
- **SoapUI:** `local/sample_messages/hl7_soap_server.wsdl` is a hand-written WSDL "for SoapUI
  import / discovery only… NOT generated or enforced by the server at runtime".
- **Ultra7** and the **Component Tester**: see [§13](#13-manual-and-exploratory-testing-tools).

### 12.8 External dependencies of local flows

- **PIMS reference-data API.** The `pims-to-mpi` flow needs a separately supplied reference-data
  lookup service on host port 8085. The transformer reaches it through
  `REFERENCE_DATA_API_BASE_URL`, which defaults to `http://host.docker.internal:8085`. You can
  override this in an untracked `env/transformers/pims-hl7-transformer.local.env`. Note that
  `risp-hl7-rest-server` also publishes host port 8085, so the two cannot run at the same time,
  for example under the `everything` profile.
- **WRDS.** The core reference transformer needs WRDS, or a fixture file set with
  `WRDS_LOCAL_FIXTURE_PATH`.
- **Corporate TLS.** `ca-certs/` holds corporate CA certificates that are injected into images.
  `docs/PODMAN_CORPORATE_CA_FIX.md` covers Podman specifically.

### 12.9 Dev Container and Codespaces

`.devcontainer/devcontainer.json` uses `mcr.microsoft.com/vscode/devcontainers/python:3` with the
`docker-from-docker` and `just` features. It forwards ports 2575, 2577, 2578 and 2579 and sets
`DOCKER_API_VERSION=1.43`. `local/README.md` suggests `just run phw-to-mpi` as a one-command
start inside a Codespace.

---

## 13. Manual and exploratory testing tools

### 13.1 Component Tester (`dev_tools/integration_hub_tester/`)

This is a Tk desktop GUI for "testing and debugging **all Integration Hub services** from a
single tabbed window, without running any Azure infrastructure, Service Bus connections, or MLLP
ports".

- **Run it:**

  ```bash
  cd dev_tools/integration_hub_tester
  uv sync
  uv run python app.py
  ```

  Or use the VS Code launch configuration **"Component Tester GUI"** in `.vscode/launch.json`
  (`justMyCode: false`, so breakpoints inside service code work).
- **Tabs:** PHW, Chemo, PIMS, Core Reference and PROMS transformers; HL7 Server; HL7 Sender; HL7
  Sender ACK; REST Server (HL7); HL7 SOAP Server; SOAP Sender and SOAP Sub Sender. A header bar
  starts and stops an MLLP or SOAP mock receiver in a separate console.
- **How it reaches the real code.** The tool's `pyproject.toml` declares the real services as
  live path dependencies, so edits take effect immediately. Each tab is a `ServicePlugin` with a
  `run(input_text) -> (output_text, status)` method. The plugins avoid infrastructure as follows:

| Service | Approach |
|---|---|
| Chemo, PIMS, Core Reference | Call the module-level function directly (`transform_chemocare_message`, `transform_pims_message`, `apply_core_reference_mapping`) |
| PHW | `object.__new__(PhwTransformer)` to skip `__init__`, then set two attributes |
| PROMS | `transform_proms_xml_to_fhir_bundle()` |
| HL7 Server | `HL7Validator`, `HL7AckBuilder`, `ErrorHandler` used directly |
| HL7 Sender ACK | `get_ack_result()` (a pure function) |
| REST and SOAP servers | Real processors, with `MagicMock()` for the Service Bus sender and message store, "the same pattern each service's own test suite uses" |
| Core Reference WRDS | `WRDSService(fixture_file_path=…)` |

- **Known quirk.** `wrds-service` is a non-editable path dependency, so edits to
  `shared_libs/wrds_service` need `uv sync`.
- **Status.** It is not built or tested by any pipeline and has no tests of its own.

### 13.2 Ultra7 (`dev_tools/ultra7/`)

Ultra7 is "A local developer GUI for sending HL7 v2.x / XML / JSON test messages to MLLP, REST,
and SOAP endpoints". It has projects, message lists, field iteration (incrementing a field across
repeated sends), pretty printing and themes.

- **Requirements:** Python with Tk. `.python-version` pins 3.14. The README has a note on
  Homebrew Python lacking `_tkinter` and suggests `uv python install 3.14`.
- **Run it:** `cd dev_tools/ultra7 && uv sync && uv run python app.py`.
- **Tests:** `uv run python -m unittest discover tests` (106 tests). GUI widget tests skip when
  Tk is not available.
- **Design documents:** `dev_tools/ultra7_plan.md`. The README also refers to an
  `ultra7_spec.md`, which is not present in `dev_tools/` at the snapshot.

### 13.3 BusWatch (`dev_tools/buswatch/`)

BusWatch is a FastAPI web UI for inspecting messages on Service Bus queues and subscriptions,
against either the emulator or the cloud. It runs at `http://127.0.0.1:8080` in most local
profiles. It reads queue names from `BUSWATCH_QUEUE_NAMES`, or discovers them from
`ServiceBusEmulatorConfig.json` when that variable is empty. Its own tests (15) run with
`uv run python -m unittest discover tests`.

### 13.4 Network Test App (`network_test_app/`)

The Network Test App is a container image for checking network paths and firewall rules from
inside Azure Container Apps. Its "Local Testing" section is:

```bash
docker build -t network-test-app .
docker run -it network-test-app /bin/sh
```

It has no Python tests. It has a build pipeline (`networktestapp-build.yml`) and is deployed by
`release-apps.yml`.

### 13.5 Dashboard network tests

The dashboard has a network-test feature (`dashboard/services/network_test.py`, tested in
`test_network_test.py`). It runs latency tests against configured endpoints and stores the
history in Cosmos. This is an operational tool, not part of the test suite.

---

## 14. The external `Integration-Hub-tests` repository

The root `README.md` ends with:

> ## Related repositories
> - [Terraform](https://github.com/DHCW-Digital-Health-and-Care-Wales/Integration-Hub-Terraform)
> - [Automated tests](https://github.com/DHCW-Digital-Health-and-Care-Wales/Integration-Hub-tests)

The repository at
`https://github.com/DHCW-Digital-Health-and-Care-Wales/Integration-Hub-tests` could not be read
for this document. GitHub returned HTTP 403: *"The 'DHCW-Digital-Health-and-Care-Wales'
organization has enabled or enforced SAML SSO. To access this repository, you must re-authorize
the OAuth Application 'GitHub CLI'."* Nothing is known about its contents, framework, triggers or
target environments. Nothing in `Integration-Hub-Beta` (no pipeline, script or doc) refers to it
apart from that link.

If you have access, you can authorise the GitHub CLI for the organisation and clone it:

```bash
gh auth refresh -h github.com -s read:org   # then approve SSO for the organisation in the browser
gh repo clone DHCW-Digital-Health-and-Care-Wales/Integration-Hub-tests
```

---

## 15. Test data in the repository

### 15.1 Where it lives

| Location | Format | Used by |
|---|---|---|
| Inline strings in `test_*.py` | ER7, XML, SOAP | Most unit tests |
| `tests/*_messages.py`, `tests/messages.py`, `tests/lims_xml.py` | ER7 and XML builders | Transformer tests |
| `transformers/hl7_phw_transformer/tests/phw-valid-message.hl7` | ER7 file | PHW transformer |
| `local/sample_messages/*.hl7`, `*.xml` | ER7, SOAP, XML | Manual end-to-end runs (`just send`), the Component Tester, Ultra7 |
| `lookup_service` seed data | CSV / JSON | `test_file_store.py`, `test_seed.py` |
| WRDS fixture XML | SOAP response | Fixture mode (§6.10) |

### 15.2 Sample messages

| File | Type | Sending app (`MSH-3`) | Version |
|---|---|---|---|
| `phw-to-mpi.sample.hl7` | `ADT^A31^ADT_A05` | 252 | 2.5 |
| `chemocare-to-mpi.sample.hl7` | `ADT^A28` | 224 | 2.4 |
| `paris-to-mpi.sample.hl7` | `ADT^A31^ADT_A05` | 169 | 2.5.1 |
| `mosaiq-to-mpi.sample.hl7` | `ADT^A31^ADT_A05` | 285 | 2.5 |
| `pims-to-mpi.sample.hl7` | `ADT^A40^ADT_A39` | PIMS | 2.3.1 |
| `mpi-outbound-pharmacy.sample.hl7` | `ADT^A31^ADT_A05` | 100 | 2.5 |
| `training-test-a28.hl7` | `ADT^A28` | 100 | 2.5 |
| `lims-to-mpi.sample.xml` | SOAP-wrapped HL7 v2 XML `ADT_A05` | 328 | — |
| `pms-to-mpi.sample.xml` | SOAP-wrapped HL7 v2 XML `ADT_A05` | 140 | — |
| `wpas-to-proms.xml` | `PromsEventRequest` XML | `system_id` 149 | — |
| `hl7_soap_server.wsdl` | WSDL | — | — |

### 15.3 Character of the data

Most sample data is obviously synthetic:

- names such as `TEST^TEST`, `MYSURNAME^MYFNAME`, `SURNAME^FORENAME`, `Doe^John`;
- addresses such as `1 TEST STREET`, `99, MY ROAD`;
- telephone numbers such as `07000000001`;
- NHS-number-shaped identifiers such as `4444444444`, `6666666666` and `1000000001`.

A few files have realistic-looking names, addresses and ten-digit identifiers, for example
`mpi-outbound-pharmacy.sample.hl7` and `wpas-to-proms.xml`. Nothing in the repository says
whether these are synthetic. The repository also has no documented convention for test NHS
numbers or other identifiers.

---

## 16. Observed inconsistencies and gaps

This is a factual list of mismatches and gaps found while preparing Part 1. Recommendations for
each are in Part 2.

### 16.1 Pipeline coverage of components

| # | Finding |
|---|---|
| G1 | The PR-validation `APPS` list (27) and the coverage `APPS` list (32) differ. The coverage YAML says to keep them "in sync". |
| G2 | **Not in PR validation:** `dashboard` (673 tests), `lookup_service` (160), `message_replay_job` (58), `hl7_message_processor` (59), `soap_sender` (39), `soap_subscription_sender` (32), `wrds_service` (26), `hl7_wds_transformer` (4), `ultra7` (106), `dev_tools/scripts` (75). That is about **1,232 tests, roughly half the repository's total**, none of which gates a merge. |
| G3 | **Not in the coverage pipeline:** `hl7_topic_mock_receiver`, `buswatch`, `lookup_service`, `ultra7`, `dev_tools/scripts`. |
| G4 | **Tested by no pipeline at all:** `lookup_service`, `ultra7`, `dev_tools/scripts`. |
| G5 | The PR-validation `paths` filter leaves out several components, so a PR that touches only them never triggers validation (§10.1). |
| G6 | `soap_subscription_sender` has no build pipeline. `lookup_service` has none either, even though it appears in the Compose stack. |
| G7 | Build pipelines for three pytest components (`http_mock_receiver`, `soap_sender`, `hl7_wds_transformer`) don't set `usePytest`. This works today only because those suites contain nothing but `TestCase` classes. |
| G8 | Path filters use `*` in some build pipelines and `**` in others (§10.2). |

### 16.2 Local and CI parity

| # | Finding |
|---|---|
| G9 | CI installs ruff, mypy and bandit **unpinned**. `check.sh` uses the pinned versions. Versions also differ between components (ruff 0.14.9, 0.15.0, 0.15.1). |
| G10 | 11 of the 23 `check.sh` scripts (variants B and D) run mypy without `tests/`, while CI runs mypy over `tests/`. |
| G11 | `check.sh` runs bandit at the default severity, and CI at `medium`. |
| G12 | `check.sh` doesn't run `uv audit` or `pip-audit`. CI does. |
| G13 | 13 components have no `check.sh`. |
| G14 | `run-all-tests.sh` runs every component with unittest, including the dashboard, whose pytest-only functions and autouse fixtures are then skipped (§9.2). |
| G15 | PR validation uses `uv sync --locked` into per-app venvs. The build template uses unlocked `uv pip install --system`. |

### 16.3 Test code

| # | Finding |
|---|---|
| G16 | **`test_field_utils.py` is duplicated.** The canonical copy is `shared_libs/field_utils_lib/tests/test_field_utils.py` (411 lines, 32 tests). Smaller, diverged copies exist in `hl7_chemo_transformer` (135 lines, 15 tests) and `hl7_pims_transformer` (177 lines, 18 tests). `docs/deep_eval.md` calls this a "maintenance hazard". |
| G17 | **`hl7_rest_server/tests/` and `rest_server/tests/hl7/` are near-duplicates.** All eight shared filenames differ, by between 4 and 74 changed lines. This is expected while `hl7_rest_server` is being decommissioned, but fixes have to be made twice. |
| G18 | `docs/deep_eval.md` notes mis-ordered mock parameter names in `test_pims_transformer.py`: "assertions pass but the naming would be misleading". This was not re-checked here. |
| G19 | Tests call private functions (`_process_message`, `_calculate_batch_size`, `_applicable_threshold`, …). |
| G20 | 9 test directories lack `__init__.py`. Two test files use CamelCase names (`test_MessageReceiverClient.py`, `test_MessageSenderClient.py`). |
| G21 | Some files in `docs/deep_eval.md`'s "thin tests" table have since gained tests. `hl7_server/tests/test_hl7_ack_builder.py` now exists with 11 tests, while the doc says "No dedicated test file". Others are unverified: `PhwTransformer` still has 1 test in `test_phw_transformer.py`, and `transformer_base_lib/tests/test_message_processor.py` has 2. |
| G22 | The `hl7_wds_transformer` suite has only 4 tests. |

### 16.4 Documentation

| # | Finding |
|---|---|
| G23 | `CLAUDE.md` and the root README say unittest. `AGENTS.md` and `.github/copilot-instructions.md` say pytest and show a pytest `check.sh`. |
| G24 | `docs/deep_eval.md` says there is no coverage in CI. That is outdated (§11). |
| G25 | `local/README.md` describes `message-store-service` and a `replay` profile that aren't in the Compose file. `CLAUDE.md` suggests `just run replay`. |
| G26 | `local/README.md` says `--profile "*"` starts everything. The justfile says it doesn't. |
| G27 | `just send-rest`'s default file doesn't exist. `justfile.docker` lacks the `send*` recipes. |
| G28 | `CLAUDE.md`'s "Repository Structure Reference" lists a root `TRAINING.md`. The file is actually at `docs/TRAINING.md`. |

### 16.5 Missing test types

| # | Finding |
|---|---|
| G29 | No automated end-to-end or flow tests in this repository. The external tests repository could not be checked. |
| G30 | No automated contract or compatibility tests between services. Examples would be server output vs transformer input, and transformer output vs sender input. The REST/SOAP parity test is the only cross-service characterisation test. |
| G31 | No SQL integration tests. No real-socket MLLP round-trip tests. |
| G32 | No property-based or fuzz tests, even though HL7 parsing and escaping are good candidates. |
| G33 | No smoke or health checks after deployment in `release-apps.yml`. No load tests, despite a `load` environment. |
| G34 | No test-result publishing (JUnit) to ADO. No coverage threshold. |
| G35 | No documented policy for synthetic patient identifiers in test data. |

---

# Part 2 — Recommendations

Everything in this part is **opinion and advice**, based on the facts in Part 1 and on common
practice for Python microservices that handle clinical messages. Where advice conflicts with a
decision the team has already made, the team's decision wins. Treat these as proposals to discuss.
Code samples are **sketches**. Names marked `<like_this>` are placeholders, and real constructor
signatures should be checked against the current source.

## 17. Guiding principles

These principles sit behind the rest of the advice.

1. **One component, one environment, one command.** Every component should be testable with a
   single command run from its own directory, giving the same result on a laptop and in CI. The
   repository is already mostly like this. The aim is to make it fully true.
2. **CI is the specification. `check.sh` is its local mirror.** If CI and `check.sh` disagree,
   developers lose trust in both. Make them run the same tools, at the same versions, with the
   same flags.
3. **The list of components should be discovered, not written by hand.** Hand-maintained
   `APPS` arrays have drifted (G1–G4). The repository's own `run-all-*.sh` scripts already
   discover components with `find`. CI should do the same, or read one shared manifest.
4. **Hermetic by default, real on purpose.** Unit tests must never touch the network, Azure, a
   database or the clock. Tests that do touch them must say so in their name or location, skip
   cleanly when their dependency is missing, and run on a known schedule.
5. **Test behaviour at the boundaries that matter clinically.** For this platform the important
   behaviours are:
   - which ACK is returned;
   - whether a message is completed, abandoned or dead-lettered;
   - which fields are mapped, and how;
   - that nothing patient-identifiable is logged;
   - that ordering and sessions are preserved.

   Prefer assertions on those outcomes over assertions about how the code got there.
6. **Synthetic data only, and recognisably synthetic.** Test data should never be mistaken for
   real patient data, by a person or by a scanner.
7. **Fix duplication at the source.** If a test is copied between components, the code under
   test is probably copied too. Remove both copies together.
8. **Small, steady improvement beats a big-bang rewrite.** Change one pipeline or one component
   at a time, keep `main` green, and measure progress with coverage and with the count of
   components in each pipeline.

---

## 18. Day-to-day workflows

### 18.1 First-time setup

```bash
# Install uv (macOS: brew install uv; otherwise see https://docs.astral.sh/uv/)
uv --version

# Sync every component once (creates each component's .venv)
dev_tools/scripts/uv_sync_all.sh
```

To sync just one component:

```bash
cd transformers/hl7_pims_transformer
uv sync --all-groups
```

On a corporate network that intercepts TLS, add `--native-tls` to `uv` commands, as the
Dockerfiles and `run-all-mypy.sh` do.

### 18.2 Running tests for one component

From the component directory:

```bash
# unittest components (most of them)
uv run python -m unittest discover tests -v

# pytest components (dashboard, http_mock_receiver, soap_sender,
# soap_subscription_sender, hl7_wds_transformer)
uv run pytest -v
```

Which runner a component uses is shown by the last line of its `check.sh`, by its row in
[§4](#4-component-inventory), or by whether `pytest` appears in its `pyproject.toml` `dev` group.

### 18.3 Running a subset

**unittest:**

```bash
uv run python -m unittest tests.test_generic_handler -v                                   # one module
uv run python -m unittest tests.test_generic_handler.TestGenericHandler -v                # one class
uv run python -m unittest tests.test_generic_handler.TestGenericHandler.test_valid_a28_message_returns_ack
uv run python -m unittest discover tests -k ack -v                                        # by name pattern (Python 3.7+)
uv run python -m unittest discover tests -f                                               # stop at first failure
```

Dotted module names such as `tests.test_generic_handler` work only when `tests/__init__.py`
exists. In the 9 components without one (§5.2), use `discover` with `-p`:

```bash
uv run python -m unittest discover tests -p "test_MessageReceiverClient.py" -v
```

**pytest:**

```bash
uv run pytest tests/test_app.py -v
uv run pytest tests/test_app.py::TestApiRoutes -v
uv run pytest -k "pause and not emulator" -v
uv run pytest -x --lf                  # stop at first failure; re-run last failures
```

### 18.4 Running everything locally

```bash
dev_tools/scripts/run-all-tests.sh       # all components, unittest (see caveat in §9.2)
dev_tools/scripts/quality_report.py      # all checks; picks pytest where appropriate; HTML report
```

**Recommendation:** use `quality_report.py --checks tests` as the "run all tests" command until
`run-all-tests.sh` learns about pytest (§26.3). It already picks the right runner for each
component and gives you a readable report.

### 18.5 Running the full local gate before pushing

```bash
cd <component>
bash check.sh                 # if it exists
```

For a CI-identical gate on any component, including those without a `check.sh`, run the same
commands CI runs, from the component directory:

```bash
PKG=$(basename "$PWD")
uv tool run ruff check --output-format=github .
uv tool run bandit -r "$PKG" tests --severity-level medium
uv audit --locked --ignore GHSA-5239-wwwm-4pmq
uv tool run mypy "$PKG" tests --ignore-missing-imports
uv run python -m unittest discover tests -v      # or: uv run pytest tests -v
```

### 18.6 Measuring coverage locally

```bash
uv run coverage run --source=. -m unittest discover tests     # or: -m pytest tests
uv run coverage report -m                                     # -m lists missing lines
uv run coverage html && open htmlcov/index.html               # browsable report
```

`--source=.` matches CI. The `omit` setting in `pyproject.toml` keeps `tests/` and `.venv/` out
of the numbers.

### 18.7 Debugging a failing test

- **VS Code:** add a configuration like this to `.vscode/launch.json`, or to a personal launch
  file:

  ```jsonc
  {
    "name": "unittest: current component",
    "type": "debugpy",
    "request": "launch",
    "module": "unittest",
    "args": ["discover", "tests", "-v"],
    "cwd": "${fileDirname}/..",
    "python": "${fileDirname}/../.venv/bin/python",
    "justMyCode": false
  }
  ```

- **pdb:** put `breakpoint()` in the test, then run it alone. With pytest, add `-s` so stdin
  isn't captured.
- **Logging:** many modules log through `logging`. To see the output while a unittest runs, wrap
  the code in `with self.assertLogs("<logger name>", level="DEBUG") as cm:` or set
  `logging.basicConfig(level=logging.DEBUG)` temporarily. With pytest, use `-o log_cli=true
  --log-cli-level=DEBUG`.
- **Stale shared library:** if a service test fails in a way that suggests old shared-library
  code, re-sync that service with `uv sync --reinstall` (§3.1).

### 18.8 Running the opt-in integration tests

| Test | Commands |
|---|---|
| HL7 sender throttling | Start `sb-emulator`, `mpi-hl7-sender` and a mock receiver, export the sender's env (`set -a; . ../../local/env/senders/mpi-hl7-sender.env; set +a`, with the connection string adjusted for the host), then run `uv run python -m unittest tests.test_throttling_local -v` from `senders/hl7_sender`, or `python tests/run_integration_test.py` |
| Dashboard alarm pauses | `cd local && docker compose --profile dashboard up -d cosmos-emulator`, then `cd dashboard && RUN_COSMOS_EMULATOR_TESTS=1 uv run pytest tests/test_alarm_pause_emulator.py -v` |
| Lookup Cosmos store | `cd local && docker compose --profile lookup up -d cosmos-emulator`, then `cd lookup_service && LOOKUP_IT_COSMOS_ENDPOINT=https://localhost:8081 LOOKUP_IT_COSMOS_KEY=<emulator key> uv run python -m unittest tests.test_cosmos_store -v` |

The emulator key is the well-known Cosmos emulator key from Microsoft's documentation. It is
typically already in `local/env/lookup/lookup-service.env`. Never commit a key for a real account.

### 18.9 Running tests in the Dev Container or Codespaces

The Dev Container (`.devcontainer/devcontainer.json`) gives you Python, Docker (through
docker-from-docker) and `just`. It does **not** install uv. Install it once per container:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh     # or: pipx install uv
```

After that, every command in this section works unchanged. Two differences from a laptop:

- **Ports.** Only 2575, 2577, 2578 and 2579 are forwarded automatically. To reach BusWatch
  (8080) or the REST servers (8083–8086) from your browser, forward them in the VS Code *Ports*
  panel.
- **Volume paths.** Because Docker runs on the host, bind mounts must use host paths. The
  container sets `HOST_WORKSPACE_PATH` for this purpose. Integration or end-to-end scripts that
  mount files should use it rather than `$PWD`.

---

## 19. Writing new tests: recipes by component type

Each recipe gives (a) what to test, (b) a skeleton in the house style (unittest `TestCase`,
`-> None` annotations, `unittest.mock`), and (c) pitfalls.

### 19.1 House style (applies to every recipe)

**Recommended conventions for new test code:**

- **Runner:** write `unittest.TestCase` classes. They run under both unittest and pytest, which
  keeps every component compatible with whichever runner its pipeline uses. Use pytest-only
  features (fixtures, `parametrize`, `tmp_path`) only in components that already depend on
  pytest, which today means the dashboard.
- **Layout:** one `tests/test_<module>.py` per source module. Put shared builders in a non-test
  module such as `tests/helpers.py`, `tests/messages.py` or `tests/fakes.py`. Add
  `tests/__init__.py`.
- **Names:**
  - Classes: `Test<Subject>`.
  - Methods: `test_<condition>_<expected_outcome>`, for example
    `test_missing_pid_returns_ar_nack`.
  - Keep names readable as sentences. They are what you see in CI logs.
- **Structure:** Arrange / Act / Assert, separated by a blank line or `# Arrange` comments as in
  `test_MessageReceiverClient.py`.
- **Types:** annotate test methods `-> None` and mock parameters `MagicMock`. CI runs mypy over
  `tests/`.
- **One behaviour per test.** Use `subTest` for many inputs that share one behaviour.
- **No sleeping.** Patch `time.sleep`. Inject clocks.
- **No network or filesystem outside a temporary directory.** Use `tempfile.TemporaryDirectory()`
  or `self.enterContext(...)` (Python 3.11+) for file I/O.
- **Synthetic data:** see [§20](#20-test-data-guidance).
- End each module with `if __name__ == "__main__": unittest.main()`.

### 19.2 Shared library functions (pure logic)

**Test:** every public function's documented behaviour, edge cases (empty fields, HL7 nulls `""`,
repetitions `~`, components `^`, sub-components `&`, escape sequences `\F\ \S\ \T\ \R\ \E\`),
and errors.

```python
import unittest

from field_utils_lib.field_utils import get_hl7_field_value  # adjust to the real module path
from hl7apy.parser import parse_message

MESSAGE = (
    "MSH|^~\\&|252|252|100|100|20250101120000||ADT^A31^ADT_A05|MSG0001|P|2.5\r"
    "PID|||9990000018^^^NHS^NH||TESTPATIENT^ALPHA\r"
)


class TestGetHl7FieldValue(unittest.TestCase):
    def setUp(self) -> None:
        self.message = parse_message(MESSAGE)

    def test_returns_first_component_of_patient_name(self) -> None:
        self.assertEqual(get_hl7_field_value(self.message.pid, "pid_5.xpn_1"), "TESTPATIENT")

    def test_returns_empty_string_for_absent_field(self) -> None:
        self.assertEqual(get_hl7_field_value(self.message.pid, "pid_7.ts_1"), "")


if __name__ == "__main__":
    unittest.main()
```

**Pitfalls:**

- Don't add a new copy of a shared library's tests to a service (G16). Test the shared function
  once, in `shared_libs/`. In the service, test only how the service uses it.
- When you change a shared library, run the tests of **every service that depends on it**. To
  find them:

  ```bash
  grep -l "<lib-name>" */*/pyproject.toml */pyproject.toml
  ```

### 19.3 Transformer mappers

Mappers (`map_msh`, `map_pid`, `map_evn`, …) take an original message and a new message and copy
or transform fields. They are the most valuable thing to test in a transformer, because they
encode the business rules agreed with each source system.

**Test:**

- every field the mapper sets, including fields it deliberately leaves empty;
- each reference-data lookup outcome: mapped, unmapped, and lookup error;
- date and time formats, timezone handling, and invalid values (with `assertRaises`);
- repeating fields such as `PID-3` identifier lists, including the order of repetitions.

```python
import unittest

from hl7apy.core import Message
from hl7apy.parser import parse_message

from hl7_pims_transformer.mappers.pid_mapper import map_pid       # adjust to the real path
from tests.fakes import StubLookupClient
from tests.pims_messages import <BASE_MESSAGE>                     # reuse shared fixtures


class TestPIDMapperGender(unittest.TestCase):
    def setUp(self) -> None:
        self.lookup = StubLookupClient({(<ReferenceDataset.GENDER>, "F"): "F", (<ReferenceDataset.GENDER>, "M"): "M"})

    def test_maps_each_known_gender_code(self) -> None:
        for source, expected in [("F", "F"), ("M", "M")]:
            with self.subTest(source=source):
                original = parse_message(<BASE_MESSAGE>)
                original.pid.pid_8.value = source
                new = Message(version="2.5")

                map_pid(original, new, self.lookup)

                self.assertEqual(new.pid.pid_8.value, expected)

    def test_unknown_gender_code_raises(self) -> None:
        original = parse_message(<BASE_MESSAGE>)
        original.pid.pid_8.value = "Q"
        with self.assertRaises(<ReferenceDataLookupError>):
            map_pid(original, Message(version="2.5"), self.lookup)
```

**Pitfalls:**

- Re-parse the base message inside each `subTest`. hl7apy objects are mutable, so one case can
  otherwise change the next.
- Prefer injected fakes (`StubLookupClient`) over patching an HTTP library. Fakes survive
  refactoring.
- When a mapping rule comes from a specification or wiki page, put the link or reference in the
  test's docstring. The PROMS reference-data tests ("wiki-derived lookup tables") are a good
  model.

### 19.4 Whole-message transformer tests

Add at least one **golden-message** test per message type a transformer supports: a realistic
input, the complete expected output, and a comparison of the two.

```python
import unittest
from pathlib import Path

from hl7apy.parser import parse_message

from hl7_phw_transformer.phw_transformer import PhwTransformer

FIXTURES = Path(__file__).parent / "fixtures"


def _segments(er7: str) -> list[str]:
    return [s for s in er7.replace("\n", "\r").split("\r") if s]


class TestPhwGoldenMessages(unittest.TestCase):
    def test_a31_matches_expected_output(self) -> None:
        source = (FIXTURES / "phw_a31_input.hl7").read_text()
        expected = (FIXTURES / "phw_a31_expected.hl7").read_text()

        result = PhwTransformer().transform_message(parse_message(source.replace("\n", "\r")))

        self.assertEqual(_segments(result.to_er7()), _segments(expected))
```

**Why:** mapper unit tests prove each rule on its own. Golden tests catch interactions between
rules, segment ordering (already a concern in `test_phw_transformer.py`), and unintended changes
anywhere in the message. When a golden test fails, `assertEqual` on lists of segments shows a
readable diff per segment.

**Pitfalls:**

- Fields that change on every run (timestamps, `MSH-10` control IDs) make golden output unstable.
  Either inject a clock or ID generator, or blank those fields in both strings before comparing.
- Keep fixture files small and synthetic. Store them in `tests/fixtures/` with `.hl7` endings and
  keep `\r` segment endings. `.gitattributes` may need a `-text` rule for them, so that line-ending
  conversion doesn't damage them.
- `PhwTransformer.__init__` may need configuration. If it does, construct it the way the
  Component Tester does (`object.__new__` plus the required attributes), or better, refactor so
  that the transform logic is a pure function ([§26.5](#265-make-code-easier-to-test)).

### 19.5 Server handlers and ACK behaviour

For MLLP and REST servers, the **ACK is the contract with the source system**. Every validation
rule should have a test that asserts the resulting `MSA-1` code and that the message was **not**
published.

```python
import unittest
from unittest.mock import MagicMock

from hl7_server.generic_handler import GenericHandler

VALID = (
    "MSH|^~\\&|252|252|100|100|20250101120000||ADT^A31^ADT_A05|MSG0001|P|2.5|||||GBR||EN\r"
    "PID|||9990000018^^^NHS^NH||TESTPATIENT^ALPHA\r"
)


class TestGenericHandlerAckCodes(unittest.TestCase):
    def _handler(self, message: str, validator: MagicMock | None = None) -> tuple[GenericHandler, MagicMock]:
        sender = MagicMock()
        handler = GenericHandler(
            message, sender, MagicMock(), MagicMock(), validator or MagicMock(),
            workflow_id="test-workflow", sending_app="252",
            message_store_client=MagicMock(), egress_session_id="test-session",
        )
        return handler, sender

    def test_valid_message_is_published_and_acknowledged_aa(self) -> None:
        handler, sender = self._handler(VALID)

        ack = handler.reply()

        self.assertIn("MSA|AA|MSG0001", ack)
        sender.send_text_message.assert_called_once()

    def test_validation_failure_is_not_published(self) -> None:
        validator = MagicMock()
        validator.validate.side_effect = <ValidationException>("bad")
        handler, sender = self._handler(VALID, validator)

        ack = handler.reply()

        self.assertRegex(ack, r"MSA\|A[ER]\|MSG0001")
        sender.send_text_message.assert_not_called()
```

**Also test:**

- `MSH-10` is echoed into `MSA-2`.
- The sending and receiving application and facility are swapped in the ACK's MSH.
- The processing ID (`MSH-11`) of the ACK. `docs/deep_eval.md` says it is hard-coded to `P`;
  pin whichever behaviour is intended with a test.
- Message size limits, at, just below and just above the configured maximum.
- The store-before-publish ordering and store-failure behaviour (existing examples in §6.6).
- Metrics and event-log calls for success and failure. Assert the event type and workflow ID,
  and check that no patient data appears in what is logged.

### 19.6 REST and SOAP ingress routes (FastAPI)

Use `TestClient` with an application built from a test context, as `rest_server/tests/hl7/`
does:

```python
import unittest
from unittest.mock import MagicMock

from fastapi.testclient import TestClient

from rest_server.hl7.app import create_hl7_app
from tests.hl7.helpers import VALID_ER7_MESSAGE, build_test_context


class TestMessageRouteContentTypes(unittest.TestCase):
    def setUp(self) -> None:
        context, self.sender, self.store, _ = build_test_context()
        self.client = TestClient(create_hl7_app(context))

    def test_missing_message_content_returns_4xx_without_publishing(self) -> None:
        response = self.client.post("/hl7MessageReceiver", json={})

        self.assertGreaterEqual(response.status_code, 400)
        self.assertLess(response.status_code, 500)
        self.sender.send_text_message.assert_not_called()
```

**Test for each route:**

- status code, body shape and content type;
- that the message is published exactly once on success and never on failure;
- health, liveness and readiness routes;
- that Swagger/OpenAPI is disabled when configured off (existing `SwaggerGatingTests`);
- for SOAP: the envelope namespace, the fault structure, and XXE safety. The code uses
  `defusedxml`. Add a test that posts an XML document with an external entity and asserts it is
  rejected.

**Where new REST tests belong:** `servers/rest_server` is the replacement. Write new REST tests
there, not in `hl7_rest_server` or `hl7_soap_server` (G17).

### 19.7 Flask routes (dashboard)

The dashboard uses pytest, so use its fixtures:

```python
from flask.testing import FlaskClient


class TestNewApiRoute:
    def test_returns_json_list(self, client: FlaskClient) -> None:
        response = client.get("/api/<new-route>")

        assert response.status_code == 200
        assert response.is_json
        assert isinstance(response.get_json(), list)
```

**Pitfalls:**

- Rely on the autouse `_disable_cosmos` fixture to stay offline. If your route calls a new Azure
  service, add an autouse stub for it in `conftest.py` (as `_stub_retry_delay_metrics` does), so
  that every test is hermetic without having to remember.
- `dashboard/app.py` has a 30-second in-memory cache for `/api/status`. Clear or patch it
  between tests that depend on fresh data. `test_cache.py` shows how the cache works.

### 19.8 Senders: MLLP client

Socket code should be tested without sockets, using `Mock(spec=socket.socket)` as
`test_hl7_sender_client.py` does. Cover:

- connect, send, receive, then close;
- a timeout while waiting for the ACK (asserting the configured `ACK_TIMEOUT_SECONDS` is used);
- a partial frame followed by the rest (`recv` returning chunks);
- the peer closing the connection (`recv` returning `b""`);
- reconnecting after a broken pipe.

```python
import socket
import unittest
from unittest.mock import Mock, patch

from hl7_sender.hl7_sender_client import HL7SenderClient  # check the real constructor


class TestHl7SenderClientPartialFrames(unittest.TestCase):
    @patch("hl7_sender.hl7_sender_client.socket.socket")
    def test_ack_split_across_two_reads_is_reassembled(self, socket_cls: Mock) -> None:
        sock = Mock(spec=socket.socket)
        sock.recv.side_effect = [b"\x0bMSH|^~\\&|100|100|252|252|", b"20250101||ACK|1|P|2.5\rMSA|AA|MSG0001\r\x1c\r"]
        socket_cls.return_value = sock
        client = HL7SenderClient(<host>, <port>, <timeout>)

        ack = client.send_message("<er7>")

        self.assertIn("MSA|AA|MSG0001", ack)
```

### 19.9 Senders: message processing outcomes

The most important sender behaviour is what happens to the Service Bus message for each outcome.
Make a table of outcomes and test every row:

| ACK outcome (`AckOutcome`) | Expected Service Bus action | Expected metric or event |
|---|---|---|
| `SUCCESS` (`AA`/`CA`) | complete | delivered |
| `AE` (application error, retryable) | abandon (retry) | retry / failure event |
| `AR` (application reject) | dead-letter | rejected event |
| `INVALID` (no `MSA`, malformed) | (as designed; pin it with a test) | — |
| Connection error or timeout | abandon (retry) | failure event |

`senders/hl7_sender/tests/test_application.py` already has an `_assert_error_handling` helper.
Extend it rather than starting a second style.

### 19.10 Service Bus client code

If you change `message_bus_lib`, follow the pattern in §6.5. Pay particular attention to:

- **Session handling.** `NEXT_AVAILABLE_SESSION`, `SessionCannotBeLockedError`, and session IDs
  on sent messages (FIFO ordering depends on these).
- **Lock renewal.** Start and stop the `AutoLockRenewer` around long processing.
- **Batch behaviour.** Partial failure within a batch.
- **Recovery.** The client is recreated after `ServiceBusError`, and concurrent callers don't
  create two clients (the existing threaded test in §6.13).

### 19.11 Configuration (`app_config.py`)

Every new environment variable should have three tests:

1. it is read and converted to the right type;
2. its default when unset (or the error when required);
3. its validation error for a bad value, for example a non-integer port.

`docs/deep_eval.md` notes that a `_read_int_env` failure path was untested. This is the kind of
gap these three tests close.

```python
import os
import unittest
from unittest.mock import patch

from <service>.app_config import AppConfig

BASE_ENV = {"SERVICE_BUS_NAMESPACE": "ns", "INGRESS_QUEUE_NAME": "q", "WORKFLOW_ID": "wf", "MICROSERVICE_ID": "ms"}


class TestAppConfigMaxMessagesPerMinute(unittest.TestCase):
    def test_reads_integer(self) -> None:
        with patch.dict(os.environ, {**BASE_ENV, "MAX_MESSAGES_PER_MINUTE": "30"}, clear=True):
            self.assertEqual(AppConfig.read_env_config().max_messages_per_minute, 30)

    def test_defaults_to_none_when_unset(self) -> None:
        with patch.dict(os.environ, BASE_ENV, clear=True):
            self.assertIsNone(AppConfig.read_env_config().max_messages_per_minute)

    def test_rejects_non_integer(self) -> None:
        with patch.dict(os.environ, {**BASE_ENV, "MAX_MESSAGES_PER_MINUTE": "thirty"}, clear=True):
            with self.assertRaises(ValueError):
                AppConfig.read_env_config()
```

**Pitfall:** `patch.dict(os.environ, …, clear=True)` is better than patching `os.getenv` with a
`side_effect`. It also catches code that reads `os.environ[...]` directly, and it ensures that a
developer's own shell environment, such as an exported `INGRESS_QUEUE_NAME`, cannot leak into the
test.

**Keep env files in sync.** When you add a variable, also add it to the matching
`local/env/**/*.env` file, so that the local stack keeps working.

### 19.12 Database clients (`pyodbc`)

The existing tests patch `<module>.pyodbc`. That works, but it ties tests to the exact sequence
of driver calls. Recommendations:

- Keep unit tests at that level for error handling: connection failure, retry and transaction
  rollback.
- **Assert the SQL parameters, not just that `execute` was called.** Parameterised SQL is a
  security property (`docs/deep_eval.md` notes "parameterised SQL" as a strength). Add a test
  that the query string contains `?` placeholders and that the values arrive as a separate
  argument.
- Add one **opt-in integration test** per database client against `sqledge` from the Compose
  stack ([§22](#22-integration-test-strategy)). Only a real database can confirm that the SQL
  matches the schema in `local/sql-scripts/`.

### 19.13 Async code

Use `unittest.IsolatedAsyncioTestCase`, as `lookup_service` does:

```python
import unittest
from unittest.mock import AsyncMock

from lookup_service.resolver import <Resolver>


class TestResolverAsync(unittest.IsolatedAsyncioTestCase):
    async def asyncSetUp(self) -> None:
        self.store = AsyncMock()
        self.resolver = <Resolver>(self.store)

    async def test_returns_mapped_value(self) -> None:
        self.store.get.return_value = {"to_code": "X"}

        self.assertEqual(await self.resolver.resolve("<table>", "A"), "X")
```

### 19.14 Writing a new opt-in integration test

Use the pattern the three existing tests share, made consistent:

```python
import os
import unittest

RUN = os.getenv("RUN_INTEGRATION_TESTS") == "1"
SB_CONN = os.getenv("IT_SERVICE_BUS_CONNECTION_STRING", "")


@unittest.skipUnless(RUN and SB_CONN, "set RUN_INTEGRATION_TESTS=1 and IT_SERVICE_BUS_CONNECTION_STRING to run")
class TestPublishToEmulatorIntegration(unittest.TestCase):
    """Integration: publishes to the local Service Bus emulator. Never point this at a real namespace."""

    def setUp(self) -> None:
        if "UseDevelopmentEmulator=true" not in SB_CONN:
            self.skipTest("refusing to run against a non-emulator namespace")
        ...
```

Rules:

1. **Opt in with a single, repo-wide switch** (`RUN_INTEGRATION_TESTS=1`) plus the specific
   connection details. Today there are three different switches.
2. **Refuse to run against non-local targets.** The dashboard's emulator test already does this
   (`_LOCAL_HOSTS`).
3. **Use throwaway names** (`it-<uuid>`) and clean up in `tearDown`, even after a failure.
4. Name the file `test_<subject>_integration.py`, or put it in `tests/integration/`, so it is easy
   to find and to exclude.
5. Document how to run it at the top of the file, as all three existing tests do.

### 19.15 Tests for bug fixes

Every bug fix should start with a failing test that reproduces the bug. Put the ticket ID
(`INTHUB-<n>`) in the test's docstring. The branch naming convention already uses `INTHUB-` IDs.

```python
def test_a40_without_mrg_returns_ar_nack(self) -> None:
    """INTHUB-123456: A40 messages with no MRG segment were ACKed AA and silently dropped."""
```

---

### 19.16 Worked example: adding a validation rule, test first

This example walks through a realistic change from start to finish, to show how the recipes fit
together. Suppose the team agrees a new rule:

> **Rule:** PHW `ADT^A40` (merge) messages must contain an `MRG` segment. Without one, the server
> must reply `AR` and must not publish the message.

**Step 1: find where the behaviour lives.** PHW-specific validation lives in `hl7_server`. Its
tests are in `servers/hl7_server/tests/test_hl7_phw_specific_validation.py` (class
`TestPhwHL7Validator`, 6 tests). The schema-level rules live in `shared_libs/hl7_validation`
(`test_phw.py`). Pick the layer that owns the rule. A structural "segment must be present" rule
could belong in either. Ask the team which layer owns PHW structure rules. Here we assume the
server-level validator owns it.

**Step 2: write the failing unit test first.**

```python
# servers/hl7_server/tests/test_hl7_phw_specific_validation.py
A40_WITHOUT_MRG = (
    "MSH|^~\\&|252|252|100|100|20250101120000||ADT^A40^ADT_A39|MSGA40001|P|2.5|||||GBR||EN\r"
    "EVN||20250101120000\r"
    "PID|||TESTPI0001^^^252^PI~9990000018^^^NHS^NH||TESTPATIENT^ALPHA\r"
)

class TestPhwHL7Validator(unittest.TestCase):
    ...
    def test_a40_without_mrg_is_rejected(self) -> None:
        """INTHUB-<ticket>: A40 merges must carry an MRG segment."""
        validator = <PhwHL7Validator()>

        with self.assertRaises(<ValidationException>) as ctx:
            validator.validate(<parse>(A40_WITHOUT_MRG))

        self.assertIn("MRG", str(ctx.exception))
```

Run it and watch it fail for the right reason:

```bash
cd servers/hl7_server
uv run python -m unittest tests.test_hl7_phw_specific_validation -v
```

**Step 3: pin the observable behaviour at the handler level.** The source system sees the ACK,
not the validator's exception. So add a handler test that checks the ACK code and that nothing is
published, following §19.5:

```python
# servers/hl7_server/tests/test_generic_handler.py
def test_phw_a40_without_mrg_returns_ar_and_is_not_published(self) -> None:
    handler = GenericHandler(A40_WITHOUT_MRG, self.mock_sender, self.mock_event_logger,
                             self.mock_metric_sender, <real PHW validator>, workflow_id="test-workflow",
                             sending_app="252", message_store_client=self.mock_message_store,
                             egress_session_id="test-session")

    ack = handler.reply()

    self.assertIn("MSA|AR|MSGA40001", ack)
    self.mock_sender.send_text_message.assert_not_called()
```

Using the **real** validator here is deliberate. It makes this a small, in-process integration
test of the validator and handler together, while I/O stays mocked.

**Step 4: implement the rule.** Make both tests pass. Then run the whole component:

```bash
bash check.sh
```

**Step 5: check privacy.** The rejection path logs a warning. Add `assert_no_pii(...)` (§21.3)
to the handler test, so that the warning cannot include `TESTPATIENT` or `9990000018`.

**Step 6: check the edges.**

- an A40 *with* `MRG` is still accepted (a regression guard);
- other message types without `MRG` are unaffected;
- non-PHW sources are unaffected (`sending_app="169"`).

Use one `subTest` loop for the "unaffected" cases.

**Step 7: run dependent suites.** `hl7_server` is a leaf service, so no other component's tests
depend on it. Had the change been in `shared_libs/hl7_validation`, you would also run the tests
of every service that depends on that library (§19.2).

**Step 8: update the local stack and samples, if relevant.** If there is a PHW A40 sample in
`local/sample_messages/`, make sure it contains an `MRG`. Otherwise `just send` will now be
rejected and confuse the next person.

**Step 9: open the PR.** In the description, list:

- the rule and its ticket;
- the tests added;
- the commands you ran;
- confirmation that `servers/hl7_server` is in `pr-validation.yml`'s `APPS` array. It is, so CI
  will run the new tests.

**Step 10: after merge.** Check the coverage pipeline's report for `hl7_server`. The new branch
in the validator should show as covered.

---

## 20. Test data guidance

### 20.1 Rules

1. **Never use real patient data** in tests, fixtures, sample messages, screenshots, logs pasted
   into issues, or commit messages. This applies even to "anonymised" extracts. Re-identification
   from a combination of date of birth, postcode and sex is well documented.
2. **Make synthetic data obviously synthetic.**
   - Names: `TESTPATIENT^ALPHA`, `ZZTEST^ONE`.
   - Addresses: `1 TEST STREET`.
   - Postcodes: an obviously fictitious one, or a test postcode agreed with the team.
   - Telephone numbers: `07000000001`.
3. **Use reserved test NHS numbers.** NHS numbers are ten digits, with a Modulus 11 check digit.
   NHS England reserves the range beginning `999` for testing; those numbers are never issued to
   patients. Confirm with DHCW Information Governance that this range is the right choice for NHS
   Wales and record the decision in the repository. Then **use only numbers from that range**,
   and make sure they have a valid check digit, so that validation code paths are exercised
   properly.
4. **Keep identifiers consistent within a message.** If a test cares about the relationship
   between `PID-3` repetitions (local identifier, NHS number) and `MRG-1`, use distinct, readable
   values: `TESTPI0001`, `TESTPI0002`.
5. **Review the existing samples.** A few sample files have realistic-looking data
   (`mpi-outbound-pharmacy.sample.hl7`, `wpas-to-proms.xml`; §15.3). Either confirm they are
   synthetic and say so in a comment or README, or replace them with values from the reserved
   range.

### 20.2 A helper for valid test NHS numbers

Consider adding a small helper to a shared, dev-only place (see §20.4):

```python
def nhs_check_digit(nine_digits: str) -> int | None:
    """Modulus 11 check digit for the first nine digits of an NHS number, or None if invalid (10)."""
    total = sum(int(d) * w for d, w in zip(nine_digits, range(10, 1, -1)))
    check = 11 - (total % 11)
    if check == 11:
        return 0
    if check == 10:
        return None
    return check


def test_nhs_number(seq: int) -> str:
    """Return the seq-th valid NHS number in the 999 test range, e.g. test_nhs_number(1) -> '9990000018'."""
    n = seq
    while True:
        nine = f"999{n:06d}"
        check = nhs_check_digit(nine)
        if check is not None:
            return nine + str(check)
        n += 1
```

`9990000018` is valid. It is the first such number and is used in the examples above.

### 20.3 Message builders instead of copy-pasted strings

The repository repeats near-identical MSH and PID lines across many files. A small builder keeps
tests readable and makes the important field in each test stand out:

```python
from dataclasses import dataclass, field


@dataclass
class Er7:
    sending_app: str = "252"
    message_type: str = "ADT^A31^ADT_A05"
    control_id: str = "MSG0001"
    version: str = "2.5"
    timestamp: str = "20250101120000"
    segments: list[str] = field(default_factory=lambda: ["PID|||9990000018^^^NHS^NH||TESTPATIENT^ALPHA"])

    def build(self) -> str:
        msh = (f"MSH|^~\\&|{self.sending_app}|{self.sending_app}|100|100|{self.timestamp}||"
               f"{self.message_type}|{self.control_id}|P|{self.version}|||||GBR||EN")
        return "\r".join([msh, *self.segments]) + "\r"


# Usage: the field under test is the only thing that stands out
Er7(version="2.9").build()
Er7(sending_app="328", message_type="ADT^A40^ADT_A39", segments=[..., "MRG|TESTPI0002^^^328^PI"]).build()
```

### 20.4 Where shared test helpers should live

Today each component has its own helpers (§5.2). The trade-off:

- **Per-component helpers (status quo).** These are isolated and simple. They also drift: the
  same MSH builder appears many times.
- **A shared dev-only library**, for example `shared_libs/test_support/` with a `pyproject.toml`
  that is added only to each component's `dev` group. It removes duplication. It also adds a
  dependency that must stay out of production images. The Dockerfiles install with `--no-dev`,
  so a dev-group dependency would not reach production.

**Recommendation:** create `shared_libs/test_support` only for genuinely cross-cutting helpers:
the ER7 builder, the NHS test number generator, Service Bus mock factories, and an
`assert_no_pii_logged` helper. Keep flow-specific fixtures (`pims_messages.py` and similar) in
their components.

### 20.5 Golden files

If you adopt golden-message tests (§19.4):

- put them in `tests/fixtures/<flow>_<type>_{input,expected}.<ext>`;
- protect `.hl7` line endings in `.gitattributes`, for example `*.hl7 -text`, so `\r` is kept;
- regenerate expected files only on purpose, and review the diff like code. Never regenerate them
  automatically in CI.

---

## 21. Mocking guidance

### 21.1 What to mock

| Dependency | In unit tests | How |
|---|---|---|
| Azure Service Bus (`ServiceBusClient`, senders, receivers) | **Always mock** | Inject `MagicMock`; for receivers configure `get_queue_receiver.return_value.__enter__.return_value` (§6.5) |
| Message store client, audit client, metric sender, event logger | **Always mock** | Inject `MagicMock`; assert calls |
| SQL (`pyodbc`) | **Mock** in unit tests; real in opt-in integration | `patch("<module>.pyodbc")` |
| HTTP (`requests`) to WRDS, the reference-data API or SOAP endpoints | **Fake** at your own client boundary; mock `requests` only in that client's own tests | `StubLookupClient`-style fakes; WRDS fixture mode |
| Sockets | **Mock** | `Mock(spec=socket.socket)`; build handlers with `object.__new__` |
| Time (`time.sleep`, `datetime.now`) | **Always control** | `patch("time.sleep")`; inject a clock (`FakeClock`) |
| Random and UUIDs | Control when the output is asserted | Inject a generator, or patch `uuid.uuid4` |
| Environment | **Always isolate** | `patch.dict(os.environ, {...}, clear=True)` |
| Filesystem | Use real files in a temp directory | `tempfile.TemporaryDirectory()` |
| hl7apy, defusedxml, your own pure functions | **Never mock** | Use the real thing |

### 21.2 Rules of thumb

- **Patch where the name is looked up, not where it is defined.**
  `patch("hl7_server.generic_handler.HL7AckBuilder")` is right.
  `patch("hl7_server.hl7_ack_builder.HL7AckBuilder")` would not affect `generic_handler`, which
  has already imported the name.
- **Use `spec=` or `autospec=True`** for mocks of real classes:
  `MagicMock(spec=ServiceBusMessage)` or `patch(..., autospec=True)`. A mock without a spec
  accepts any attribute or method, so a typo or a renamed method passes silently. This is the
  most common way mocked tests drift from reality.
- **Watch the order of mock parameters with stacked `@patch` decorators.** They are applied
  bottom-up, so the **lowest** decorator supplies the **first** mock argument. `docs/deep_eval.md`
  found mis-named parameters in `test_pims_transformer.py` (G18). Name parameters after what they
  patch, and prefer `with patch(...) as name:` blocks when there are more than two.
- **Don't assert on mocks you configured.** `mock.return_value = 5; self.assertEqual(mock(), 5)`
  tests nothing.
- **Prefer asserting outcomes over interactions.** Assert the returned ACK, the
  completed/abandoned/dead-lettered call, or the published body. Assert incidental calls such as
  "logger.info was called with…" only when that call *is* the behaviour, as with audit events and
  PII redaction.
- **Avoid patching private functions.** If a test needs to, the seam is in the wrong place
  (§26.5).

### 21.3 Asserting that personal data is not logged

Add this kind of assertion to handler and sender tests that log on failure:

```python
def assert_no_pii(test: unittest.TestCase, mock_logger: MagicMock, *values: str) -> None:
    logged = " ".join(str(c) for c in mock_logger.mock_calls)
    for value in values:
        test.assertNotIn(value, logged, f"PII value {value!r} leaked into log calls")
```

Then call `assert_no_pii(self, mock_event_logger, "9990000018", "TESTPATIENT")` after the failing
path runs. It is cheap and guards one of the platform's most important non-functional
requirements.

### 21.4 Common mocking mistakes, with fixes

**Mistake 1: patching the definition site.**

```python
# Wrong: generic_handler already did "from hl7_server.hl7_ack_builder import HL7AckBuilder"
@patch("hl7_server.hl7_ack_builder.HL7AckBuilder")
# Right: patch the name in the namespace that uses it
@patch("hl7_server.generic_handler.HL7AckBuilder")
```

**Mistake 2: decorator order versus parameter order.**

```python
@patch("svc.app.ServiceBusClientFactory")   # applied last  → second parameter
@patch("svc.app.EventLogger")               # applied first → first parameter
def test_x(self, mock_event_logger: MagicMock, mock_factory: MagicMock) -> None: ...
```

Swapping the names here still passes if both mocks are only asserted as "called". That is exactly
the trap `docs/deep_eval.md` describes. Prefer `with patch(...) as mock_factory:` blocks when the
order is not obvious.

**Mistake 3: an unspecced mock hides a renamed method.**

```python
sender = MagicMock()
sender.send_text_mesage.assert_not_called()      # typo: passes for ever
sender = MagicMock(spec=MessageSenderClient)
sender.send_text_mesage.assert_not_called()      # AttributeError: the typo is caught
```

Note that `assert_*` methods with typos such as `assert_called_once_wiht` raise
`AttributeError` in modern Python only for names that begin with `assert`/`assret`. Other typos
on unspecced mocks still pass silently, so `spec` matters.

**Mistake 4: forgetting the context manager.**

```python
client.get_queue_receiver.return_value.receive_messages.return_value = [msg]          # never used
client.get_queue_receiver.return_value.__enter__.return_value.receive_messages.return_value = [msg]
```

The code under test does `with client.get_queue_receiver(...) as receiver:`, so the receiver
is whatever `__enter__` returns.

**Mistake 5: an infinite receive loop.**

```python
receiver.receive_messages.return_value = [msg]               # loop receives msg for ever
receiver.receive_messages.side_effect = [[msg], [], KeyboardInterrupt()]   # one batch, then stop
```

Use a `side_effect` list, or a stop flag the code under test checks, so that loops terminate.

**Mistake 6: leaking environment.**

```python
os.environ["INGRESS_QUEUE_NAME"] = "q"                        # leaks into every later test
with patch.dict(os.environ, {"INGRESS_QUEUE_NAME": "q"}, clear=True): ...   # restored afterwards
```

A leaked `INGRESS_QUEUE_NAME` would even switch on the throttling integration test (§8.1) for the
rest of the run.

**Mistake 7: mocking what you own and could run.** Mocking `parse_message`, `map_pid` or
`HL7AckBuilder` in tests of their callers removes the logic you most want to exercise. Mock at
the **I/O boundary**, not at internal seams. The exception is a deliberate collaboration test,
like the existing ACK-builder patch in `test_generic_handler.py`.

---

## 22. Integration test strategy

### 22.1 What integration tests are for here

Unit tests here mock every external system. That is right for speed, but it means **nothing
automated confirms that the mocks behave like the real systems**. Integration tests should close
exactly that gap, one dependency at a time:

| Dependency | Integration test to add | Target |
|---|---|---|
| Service Bus | Send, receive, complete and dead-letter with sessions through `message_bus_lib` | `sb-emulator` |
| SQL | Insert and read with `message_store_service` and `message_replay_job` against the schema in `local/sql-scripts/` | `sqledge` |
| Cosmos | Already present for the dashboard and lookup service | `cosmos-emulator` |
| MLLP | A real `hl7_server` on an ephemeral port, plus a real `HL7SenderClient` sending to it | localhost sockets |
| SOAP over HTTP | `soap_sender` → `http_mock_receiver` running in-process with uvicorn | localhost |
| WRDS | Contract test against recorded responses (fixture mode) | fixtures |

### 22.2 Conventions

- **Location:** `tests/integration/` inside each component, with an `__init__.py`. Unit
  discovery (`unittest discover tests`) will still find them, so they **must** skip themselves
  (§19.14).
- **Switch:** a single `RUN_INTEGRATION_TESTS=1`, plus target-specific variables:
  `IT_SERVICE_BUS_CONNECTION_STRING`, `IT_SQL_CONNECTION_STRING`, `IT_COSMOS_ENDPOINT` and
  `IT_COSMOS_KEY`. Keep the three existing switches working, but document the new one as the
  standard.
- **Safety:** refuse to run unless the target is local. For Service Bus, require
  `UseDevelopmentEmulator=true`. For SQL and Cosmos, require `localhost`, `127.0.0.1` or a
  Compose service name.
- **Isolation:** each test creates uniquely named resources (sessions, rows, databases) and
  cleans up afterwards.
- **Speed:** keep each integration test under about 10 seconds. The throttling test (about 60
  seconds) is better placed in a separate "slow" group.

### 22.3 Making real MLLP round-trips cheap

A real-socket test needs no containers:

```python
import socket
import threading
import unittest


def free_port() -> int:
    with socket.socket() as s:
        s.bind(("127.0.0.1", 0))
        return s.getsockname()[1]


class TestMllpRoundTripIntegration(unittest.TestCase):
    def setUp(self) -> None:
        self.port = free_port()
        self.server = <build hl7_server's SizeLimitedMLLPServer on ("127.0.0.1", self.port) with mocked Service Bus>
        self.thread = threading.Thread(target=self.server.serve_forever, daemon=True)
        self.thread.start()

    def tearDown(self) -> None:
        self.server.shutdown()
        self.server.server_close()

    def test_sender_receives_aa_ack_over_real_socket(self) -> None:
        client = <HL7SenderClient("127.0.0.1", self.port, timeout=5)>
        ack = client.send_message(<valid ER7>)
        self.assertIn("MSA|AA|", ack)
```

Because it uses only localhost, this kind of test can run **in normal PR validation**. It checks
MLLP framing (start block, end block, carriage return), size limits and timeouts end to end
across the server and sender code bases. Today nothing does. There is a packaging complication:
`hl7_server` and `hl7_sender` are separate packages with separate environments. Put the test in
whichever component can depend on the other's package in its `dev` group, or in the end-to-end
harness (§23).

### 22.4 Running integration tests in CI

Proposed: a separate, **non-blocking at first** ADO job (or a nightly pipeline) that:

1. starts `sb-emulator`, `sqledge` and `cosmos-emulator` with
   `docker compose -f local/docker-compose.yml up -d sb-emulator sqledge cosmos-emulator`;
2. waits for each to become healthy;
3. runs each component's integration tests with `RUN_INTEGRATION_TESTS=1` and the `IT_*`
   variables;
4. publishes JUnit results;
5. always tears the containers down.

Make it blocking once it has been stable for a few weeks. The self-hosted pool
(`UKS-DHCW-IH-DEV-ADOManagedPool`) needs Docker for this. Check with the DevOps lead.

### 22.5 Contract tests between services

Each service is tested in isolation, against mocks that encode one side's *assumptions* about its
neighbours. A **contract test** checks those assumptions against the real neighbour's code, with
no infrastructure. Useful contracts here:

| Producer → consumer | Contract to check |
|---|---|
| `hl7_server` → transformer | The Service Bus message body is ER7 the transformer can parse. The application properties the transformer reads (workflow ID, correlation ID, session ID) are set. |
| transformer → `hl7_sender` | The transformed message validates against the MPI schema used by `hl7_validation`. The session ID is preserved. |
| `rest_server` (SOAP adapter) → `hl7_lims_wrrs_transformer` | The "compacted ER7 from REST server" that the classifier expects is what the adapter produces. `lims_xml.py` already mirrors this, and a contract test would check it against the adapter itself. |
| `hl7_sender` ↔ `hl7_mock_receiver` | The ACK the mock produces is classified `SUCCESS` by `get_ack_result()`. |

**How:** keep a small set of **canonical intermediate messages** (`contracts/<flow>/<stage>.hl7`)
that the producer's tests assert they produce and the consumer's tests assert they accept. When
either side changes the format, one of the two test suites fails in the same PR. This needs no
containers and runs in PR validation.

---

## 23. Automating end-to-end tests

### 23.1 Goal

Turn the manual procedure in §12.6 into a repeatable suite. For each flow:

1. start the profile;
2. send a known message;
3. assert the ACK;
4. assert the transformed message arrived at the destination;
5. tear down.

This is the only kind of test that catches wiring defects: wrong queue names in an env file, a
missing subscription in `ServiceBusEmulatorConfig.json`, a Dockerfile that forgets a shared
library, or a transformer and sender that disagree on session IDs.

### 23.2 Where it should live

Decide this first. It depends on what `Integration-Hub-tests` already does (§14):

- **If that repository already has flow tests,** extend it, and add a pipeline here that checks
  it out and runs it against the local stack.
- **If it doesn't,** add an `e2e/` directory here with its own `pyproject.toml`. Keeping the
  flow tests next to the Compose file and env files means one PR can change a flow and its test
  together.

### 23.3 Assertion point: the verifier subscription

Most flows end at `mpi-hl7-topic-mock-receiver`. It publishes what it receives to the
`local-inthub-hl7-topic-mock-receiver` topic, which has a
`local-inthub-hl7-topic-mock-receiver-verifier` subscription (§12.3). An end-to-end test can read
that subscription with the Azure SDK and the emulator connection string. This needs no log
scraping.

### 23.4 Harness sketch

```python
# e2e/tests/test_phw_to_mpi.py  (sketch)
import os
import subprocess
import time
import unittest
from pathlib import Path

import hl7
from hl7.client import MLLPClient
from azure.servicebus import ServiceBusClient

LOCAL = Path(__file__).resolve().parents[2] / "local"
CONN = "Endpoint=sb://127.0.0.1;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=SAS_KEY_VALUE;UseDevelopmentEmulator=true;"
VERIFIER = ("local-inthub-hl7-topic-mock-receiver", "local-inthub-hl7-topic-mock-receiver-verifier")


def compose(*args: str) -> None:
    subprocess.run(["docker", "compose", *args], cwd=LOCAL, check=True)  # nosec B603 B607


@unittest.skipUnless(os.getenv("RUN_E2E_TESTS") == "1", "set RUN_E2E_TESTS=1 to run end-to-end flow tests")
class TestPhwToMpi(unittest.TestCase):
    @classmethod
    def setUpClass(cls) -> None:
        compose("--profile", "phw-to-mpi", "up", "-d", "--build", "--wait")

    @classmethod
    def tearDownClass(cls) -> None:
        if os.getenv("E2E_KEEP_STACK") != "1":
            compose("--profile", "phw-to-mpi", "down")

    def test_a31_is_acknowledged_and_delivered_to_mpi(self) -> None:
        control_id = f"E2E{int(time.time())}"
        message = (LOCAL / "sample_messages" / "phw-to-mpi.sample.hl7").read_text().replace(
            "202505052323364444444444", control_id)

        with MLLPClient("127.0.0.1", 2575) as client:
            ack = client.send_message(message.replace("\n", "\r")).decode()
        self.assertIn(f"MSA|AA|{control_id}", ack)

        delivered = self._wait_for_delivery(control_id, timeout=60)
        self.assertIsNotNone(delivered, "message did not reach the MPI mock receiver")
        # Assert on the PHW transformation, e.g. MSH-7 reformatted for MPI:
        self.assertNotIn("2025-05-05 23:23:32", delivered)

    def _wait_for_delivery(self, control_id: str, timeout: int) -> str | None:
        deadline = time.monotonic() + timeout
        with ServiceBusClient.from_connection_string(CONN) as sb:
            with sb.get_subscription_receiver(*VERIFIER, max_wait_time=5) as receiver:
                while time.monotonic() < deadline:
                    for msg in receiver.receive_messages(max_message_count=10):
                        body = str(msg)
                        receiver.complete_message(msg)
                        if control_id in body:
                            return body
        return None
```

Notes:

- Whether the verifier subscription needs a session receiver depends on how the topic mock
  receiver publishes. Check `env/mock_receivers/mpi-hl7-topic-mock-receiver.env` and its code.
- `docker compose up --wait` needs health checks in the Compose file. Where none are defined, add
  a readiness poll: a TCP connect to the MLLP port, and a check that the receiver is listening.
- Give each run a unique control ID (`MSH-10`) so that leftover messages from earlier runs can't
  produce false passes.
- Cover at least one negative path per flow, such as an invalid message returning an `AR`/`AE`
  ACK and never reaching the receiver.

### 23.5 Flow matrix

Aim for one happy-path test and one rejection test per flow:

| Flow | Send via | Sample | Assert at |
|---|---|---|---|
| PHW → MPI | MLLP 2575 | `phw-to-mpi.sample.hl7` | verifier subscription |
| Paris → MPI | MLLP 2577 | `paris-to-mpi.sample.hl7` | verifier |
| Chemo → MPI | MLLP 2578 | `chemocare-to-mpi.sample.hl7` | verifier |
| PIMS → MPI | MLLP 2579 | `pims-to-mpi.sample.hl7` | verifier (needs the reference-data API or a stub) |
| Mosaiq → MPI | MLLP 2583 | `mosaiq-to-mpi.sample.hl7` | verifier |
| WDS → MPI | MLLP 2582 | (add a sample) | verifier |
| LIMS → MPI | SOAP POST 8082 `/soap` | `lims-to-mpi.sample.xml` | verifier + `local-inthub-lims-wrrs-classified` |
| PMS → MPI | SOAP POST 8086 | `pms-to-mpi.sample.xml` | verifier |
| RISP → MPI | REST POST 8085 `/hl7MessageReceiver` | (add a sample) | verifier (needs WRDS fixture mode) |
| RISP → WRRS | REST POST 8085 | (add a sample) | row in `sqledge` |
| WPAS → PROMS | REST POST 8084 | `wpas-to-proms.xml` | `local-inthub-wpas-transformer-ingress` |
| MPI → outbound | MLLP 2580 | `mpi-outbound-pharmacy.sample.hl7` | `mpi-hl7-mock-receiver` (2576) logs or its queue |

### 23.6 Running end-to-end tests in CI

Run them nightly and on demand rather than on every PR at first: building every image takes
time. When they are stable, run them on PRs that change `local/`, `pipeline-ado/` or any
`Dockerfile`, for just the affected profiles.

Before relying on the Compose stack in CI, fix the discrepancies in §16.4: the missing
message-store and replay services, the `everything` profile omissions, and the port 8085
collision.

---

## 24. CI recommendations

### 24.1 One list of components, discovered automatically

Replace the hand-written `APPS` arrays in `pr-validation.yml` and `code-coverage.yml` with
discovery. Discovery already works in `dev_tools/scripts/run-all-*.sh`. Let each component
declare its own test settings in its `pyproject.toml`:

```toml
[tool.inthub]
test-runner = "unittest"     # or "pytest"
source-dir  = "hl7_server"   # replaces the hard-coded banditDir column
ci = true                    # set false to opt out explicitly, with a comment saying why
```

Then:

```bash
# discover.sh — prints "dir|runner|srcdir" for every component that opts in
find . -maxdepth 3 -name pyproject.toml -not -path '*/.venv/*' | sort | while read -r f; do
  d=$(dirname "$f"); d=${d#./}
  python3 - "$f" "$d" <<'PY'
import sys, tomllib, pathlib
f, d = sys.argv[1], sys.argv[2]
t = tomllib.loads(pathlib.Path(f).read_text()).get("tool", {}).get("inthub", {})
if t.get("ci", True) and pathlib.Path(d, "tests").is_dir():
    print(f'{d}|{t.get("test-runner", "unittest")}|{t.get("source-dir", pathlib.Path(d).name)}')
PY
done
```

Benefits:

- New components are tested automatically.
- The two pipelines can no longer disagree (G1–G4).
- The runner is declared once, next to the code (G7).
- Opting out becomes visible in code review.

### 24.2 Include every component in PR validation

Until discovery lands, add these to `pr-validation.yml`'s `APPS` array and `paths`:

- `dashboard` (pytest), `lookup_service`, `message_processing/message_replay_job`
- `senders/soap_sender` and `senders/soap_subscription_sender` (pytest)
- `transformers/hl7_wds_transformer` (pytest)
- `shared_libs/hl7_message_processor`, `shared_libs/wrds_service`
- `dev_tools/ultra7`, if a Tk-less agent is acceptable (its GUI tests skip themselves), and
  `dev_tools/scripts`

That adds about 1,230 tests (G2). The dashboard alone has 673. It is worth it: these suites
already exist and cost nothing extra to write.

### 24.3 Consider dropping the `paths` filter, or run only affected components

The `paths` filter saves agent time but has already left components unprotected (G5). There are
two better options:

- **Simplest:** remove the `paths` filter and validate everything on every PR. Check that this
  fits the 30-minute timeout. If it doesn't, use the next option.
- **Faster:** compute the changed components and their dependents. Shared-library changes affect
  every service that lists the library in `[tool.uv.sources]`. Validate only those. Sketch:

  ```bash
  CHANGED=$(git diff --name-only origin/$(System.PullRequest.TargetBranch)...HEAD)
  # map each changed file to its component dir; if under shared_libs/<lib>, add every
  # component whose pyproject.toml references "<lib>"; if under pipeline-ado/ or local/, run all
  ```

### 24.4 Run components in parallel

The single sequential job (§10.1) will get slower as components are added. Use a matrix: one job
per component, generated from discovery (§24.1). Run them in parallel on the managed pool, with a
final job that aggregates results. The original reason for one job, avoiding "16× agent
allocation overhead", deserves re-measuring: cached uv environments make per-job setup cheap.

### 24.5 Same tools, same versions, same flags, everywhere

- **Run tools from the component's environment** (`uv run ruff`, `uv run mypy`, `uv run bandit`)
  in CI as well as in `check.sh`. This makes the pinned `dev` group versions authoritative
  (G9), so a version bump becomes a deliberate PR, not a surprise on the next cache refresh.
- **Align the pinned versions** across components (ruff, mypy and bandit each have 2–4 versions
  today). `quality_fix.py` has version-update fixers that can help.
- **Use the same flags:** bandit `--severity-level medium` and mypy over `tests/` in `check.sh`
  too (G10, G11).
- **Use the same installation method:** the build template should use the PR pipeline's
  approach, `uv sync --locked --group dev` in a venv, instead of unlocked
  `uv pip install --system` (G15).

### 24.6 Publish test results

Publish JUnit XML so that ADO shows a **Tests** tab, failure history and flaky-test detection.

- **Simplest route:** use pytest as the *runner* everywhere. pytest collects `unittest.TestCase`
  classes unchanged, so no test needs rewriting. Add `pytest>=8.3.0` to each `dev` group, then:

  ```bash
  python -m pytest tests -v --junitxml="$(Build.ArtifactStagingDirectory)/junit/$SAFE_NAME.xml"
  ```

  ```yaml
  - task: PublishTestResults@2
    condition: always()
    inputs:
      testResultsFormat: JUnit
      testResultsFiles: '$(Build.ArtifactStagingDirectory)/junit/*.xml'
      mergeTestResults: true
      testRunTitle: 'PR validation — unit tests'
  ```

- **If the team prefers to keep the unittest runner,** use `unittest-xml-reporting`
  (`python -m xmlrunner discover tests -o junit/`).

Using pytest as the runner also resolves G14 (`run-all-tests.sh`), simplifies G7 (`usePytest`),
and gives `-k`, `-x` and `--lf` everywhere. **This is the single highest-leverage CI change in
this document.** The existing unittest-style tests stay as they are.

### 24.7 Fix the smaller pipeline issues

- Set `usePytest: true` (or follow §24.6) for `http_mock_receiver`, `soap_sender` and
  `hl7_wds_transformer` (G7).
- Normalise path filters to `**` (G8). In ADO, check whether `dashboard/*` matches nested files
  as intended.
- Add a build pipeline for `soap_subscription_sender`, and decide whether `lookup_service` needs
  one (G6).
- Review the `GHSA-5239-wwwm-4pmq` ignore periodically, as its comment asks. Put a review date in
  the comment.
- Add a cheap "pipeline lint" step that fails if a component directory exists that discovery
  doesn't find, or that `pr-validation.yml` doesn't list, until §24.1 is done.

### 24.8 Add post-deployment smoke tests

After each `release-apps.yml` deployment to dev, DTE, TST or UAT, run a smoke stage:

- **HTTP services:** health, liveness and readiness routes return 200.
- **MLLP services:** a TCP health-check connect, through `health_check_lib`'s TCP server.
- **Optionally, in non-production only:** one synthetic message per flow, asserted at a test
  subscription.

Make the deployment fail if the smoke stage fails. The **load** environment suggests performance
testing is intended. If it is, keep load-test definitions (for example Locust for REST and a
small MLLP load generator) in version control next to the pipeline that runs them.

---

## 25. Coverage recommendations

### 25.1 Make coverage complete before making it strict

1. Add the missing components to the coverage pipeline (G3), or let discovery add them
   (§24.1).
2. Leave the pipeline as it is (post-merge, non-blocking) for a few weeks to establish a
   **baseline per component**.

### 25.2 Then ratchet, per component

Set `fail_under` per component, just below its baseline, in `pyproject.toml`:

```toml
[tool.coverage.report]
skip_empty = true
fail_under = 82     # baseline 83.4% on 2026-11-xx; raise when coverage improves, never lower
show_missing = true
```

Then:

- Run `coverage report` in PR validation (it exits non-zero below `fail_under`). This keeps
  coverage from going down without needing a global target.
- Raise the number whenever coverage improves. A small script can propose the new value.
- **Avoid a single global target.** It hides weak components behind strong ones. The 673-test
  dashboard could carry a weak shared library.

`docs/deep_eval.md` suggested "75%+ for services, 60%+ for libraries" as eventual targets. Given
how many services depend on each library, the opposite weighting is easier to justify: shared
libraries should be at least as well covered as services.

### 25.3 Prefer diff coverage on PRs

A per-PR rule such as "new and changed lines must be at least 80% covered" (for example with
`diff-cover` against `coverage.xml`) focuses effort on the code being changed. It doesn't punish
a developer for old gaps they didn't touch. Run it as a warning first.

### 25.4 Use branch coverage to guide reviews

Branch coverage is already on (`branch = true`). In the HTML report, partially covered branches
are shown in yellow. For validators, ACK builders and sender outcome handling, **every branch
corresponds to a clinically meaningful outcome**, so aim for 100% branch coverage of those
specific modules, whatever the component-level number.

### 25.5 Coverage is not correctness

High coverage of mapper code with weak assertions proves little. Pair coverage with:

- golden-message tests (§19.4);
- mutation testing on a few critical modules, run occasionally rather than in CI. Good candidates
  are `hl7_ack_builder`, `ack_processor` and the validators, using a tool such as `mutmut`. It
  shows whether the tests would actually notice a changed `==` or a dropped `not`.

---

## 26. Maintaining the test estate

### 26.1 Keep the documentation consistent

- **One source of truth for testing conventions.** Put the agreed conventions in one place,
  either this document or a `docs/TESTING.md` derived from it. Then make `CLAUDE.md`, `AGENTS.md`,
  `.github/copilot-instructions.md` and the root `README.md` link to it instead of restating it.
  Today they contradict one another on the runner (G23). AI coding assistants read those files,
  so a contradiction there produces inconsistent tests at scale.
- **Mark `docs/deep_eval.md` as a dated snapshot.** Add a note at the top listing which findings
  have since been addressed (coverage in CI; `test_hl7_ack_builder.py`) (G21, G24).
- **Fix `local/README.md`** for the missing message-store and replay services, the profile list
  and `--profile "*"` (G25, G26). Fix the `justfile` `send-rest` default and port the `send*`
  recipes to `justfile.docker` (G27).

### 26.2 Standardise `check.sh`

Replace the 23 hand-written variants (§9.1) with one shared script, plus a one-line `check.sh`
in each component:

```bash
#!/bin/bash
# <component>/check.sh
exec "$(git rev-parse --show-toplevel)/dev_tools/scripts/check-component.sh" "$(dirname "$0")"
```

The shared `check-component.sh` would read `[tool.inthub]` (§24.1) and run exactly what CI runs:
the same tools, versions and flags, including `uv audit` and mypy over `tests/`. Add a `check.sh`
to the 13 components that lack one (G13), or just document
`dev_tools/scripts/check-component.sh <dir>` as the command.

### 26.3 Teach `run-all-tests.sh` about pytest

Until §24.6 (pytest as runner everywhere) is adopted, change the test line in
`run-all-tests.sh` so that it uses pytest when the component's `check.sh` does, as
`quality_report.py` does:

```bash
if grep -q 'pytest' check.sh 2>/dev/null; then
  uv sync --locked --all-groups --reinstall && uv run pytest tests -v
else
  uv sync --locked --all-groups --reinstall && uv run python -m unittest discover tests -v
fi
```

### 26.4 Remove duplicated tests

**`test_field_utils.py` (G16).**

1. Diff the chemo and PIMS copies against `shared_libs/field_utils_lib/tests/test_field_utils.py`.
2. For each case that exists only in a copy, decide whether it tests shared behaviour or
   transformer-specific behaviour:
   - shared → move it into the canonical file;
   - specific → keep it in a renamed module (for example `test_chemo_field_handling.py`) that
     tests the transformer's own functions.
3. Check whether the transformers still have **their own copy of `field_utils.py`**. If they do,
   switch them to `field_utils_lib` first, then delete their copies of both code and tests.

**REST server tests (G17).** While `hl7_rest_server` is still deployed, any behaviour fix must be
made in both packages. Add a short note at the top of each duplicated test module naming its twin.
When `hl7_rest_server` is decommissioned:

1. confirm that every test in `servers/hl7_rest_server/tests/` has an equivalent in
   `servers/rest_server/tests/hl7/`;
2. delete the old package, its build pipeline, and its entries in the `APPS` arrays.

Do the same for `hl7_soap_server` against `test_hl7_soap_server_parity.py`.

### 26.5 Make code easier to test

Several existing tests work around constructors that do I/O:

- `object.__new__(SizeLimitedMLLPRequestHandler)`;
- `object.__new__(PhwTransformer)` in the Component Tester;
- patching `ServiceBusClientFactory` 18 times in `message_replay_job`.

These are signs that the code would benefit from refactoring:

- **Separate pure transformation from plumbing.** The Chemo and PIMS transformers already expose
  module-level functions (`transform_chemocare_message`, `transform_pims_message`). Do the same
  for PHW, and make the class a thin wrapper.
- **Inject factories and clients** instead of constructing them inside `run()` or `main()`.
  `main()` should be the only place that wires real Azure clients together.
- **Inject a clock** (`Callable[[], float]`) into throttlers and retry loops instead of calling
  `time.monotonic`, `time.sleep` or `datetime.now` directly.
- **Expose behaviour through public functions,** so tests don't need to call `_private`
  helpers (G19).

### 26.6 Handle flaky tests

- **Policy:** a flaky test is a failing test. Either fix it within a few days, or quarantine it
  with `@unittest.skip("FLAKY INTHUB-<ticket>: <reason>")` and a ticket. Never re-run CI until it
  passes and carry on.
- **Likely sources here:**
  - real-time assertions (the throttling integration test, any test asserting elapsed time);
  - threads (`test_MessageSenderClient.py`'s concurrency test);
  - shared state, such as module-level caches like the dashboard's `/api/status` cache or a
    cached Cosmos client;
  - the developer's environment leaking in when `os.environ` isn't cleared.
- **Detection:** once JUnit results are published (§24.6), ADO's flaky-test detection can flag
  tests that both pass and fail on the same commit.

### 26.7 Dependency upgrades

- After any `uv lock` change, run the component's tests and then the tests of every component
  that depends on it.
- Use `quality_fix.py --audit` for vulnerable-package upgrades. It edits `uv.lock` only, leaves
  everything for review, and `--recheck` re-runs the affected checks.
- When upgrading **hl7apy**, run all transformer and validation suites. Parsing and
  serialisation changes show up there first. Golden-message tests (§19.4) are the best defence.
- When upgrading the **Azure SDK** (`azure-servicebus`, `azure-cosmos`), remember that unit tests
  mock it, so they **will not** catch behaviour changes. Run the integration tests (§22) or a
  local end-to-end flow before merging.

### 26.8 Normalise layout and naming

- Add `tests/__init__.py` to the 9 directories that lack one (G20). Dotted test selection then
  works everywhere (§18.3).
- Rename `test_MessageReceiverClient.py` and `test_MessageSenderClient.py` to snake case, in one
  PR with no other changes so the history stays readable.

### 26.9 Check that the local stack and the code agree

Add a fast, offline consistency check to PR validation:

```text
For each service in local/docker-compose.yml:
  - every env_file it references exists;
  - every *_QUEUE_NAME / *_TOPIC_NAME / *_SUBSCRIPTION_NAME value in those env files
    exists in local/ServiceBusEmulatorConfig.json;
  - every variable that the component's app_config.py reads (os.getenv("...")) is either set in
    its env file or has a default in code.
For each sample in local/sample_messages:
  - it parses (ER7 via hl7apy, XML via defusedxml).
For each recipe in local/justfile*:
  - default file arguments exist.
```

This would have caught G25 and G27 automatically, and it would catch the most common local-stack
breakage: a renamed queue or a new required environment variable.

### 26.10 Ownership and cadence

- **Every component has an owner** who reviews changes to its tests.
- **Quarterly:**
  - re-run `quality_report.py`;
  - review the skip list (`grep -rn "skipUnless\|unittest.skip" */*/tests */tests`);
  - review the audit ignore list;
  - refresh this document's numbers.
- **When a component is added or removed:** follow §27, and check that it appears in or
  disappears from every pipeline.

### 26.11 Measure the health of the test estate

Track a few numbers over time, for example in the quarterly review (§26.10) or on the ADO
dashboard next to the coverage widget:

| Metric | How to compute | Target direction |
|---|---|---|
| Components gated by PR validation ÷ components with tests | `APPS` length ÷ count of `*/tests` dirs | → 100% |
| Tests gated by PR validation ÷ all tests | Static count (§2.2 method) | → 100% |
| Components with `check.sh` identical to CI | Diff against the shared script (§26.2) | → 100% |
| Skipped tests | `grep -rn "skip" */*/tests */tests`, or JUnit `skipped` counts | Stable; each one has a ticket |
| Duplicated test modules | Same filename in two components with >50% similar content | → 0 |
| Per-component coverage (line and branch) | Coverage pipeline | ↑, never ↓ (ratchet) |
| Mean PR-validation duration | ADO analytics | Stable as tests grow (parallelise if not) |
| Flaky test count | ADO flaky-test detection (after JUnit publishing) | → 0 |
| Flows with an automated end-to-end test ÷ flows | §23.5 matrix | → 100% |

At the snapshot, the first two ratios were 27/37 test directories (73%) and about 1,190/2,420 tests
(about 49%). Those are the quickest numbers to move (§30, step 1).

---

## 27. Checklist: adding a new component

Use this when you create a new service, transformer, sender or shared library.

**Structure**

- [ ] Directory follows the standard layout (§5.1), with the package name matching the directory
      name.
- [ ] `pyproject.toml`:
  - [ ] `requires-python = ">=3.13"`;
  - [ ] a `dev` group with ruff, bandit, mypy and `coverage>=7.6.0`, at the **same pinned
        versions** as the rest of the repository;
  - [ ] pytest, if pytest is adopted as the runner (§24.6);
  - [ ] the standard `[tool.ruff]`, `[tool.ruff.lint]`, `[tool.bandit.assert_used]`,
        `[tool.coverage.run]` and `[tool.coverage.report]` sections;
  - [ ] `[tool.inthub]`, if adopted (§24.1).
- [ ] `uv lock` run and `uv.lock` committed.
- [ ] `tests/__init__.py` and at least one `tests/test_*.py`.
- [ ] `check.sh` (the standard one-liner, if §26.2 is adopted).

**Tests**

- [ ] `test_app_config.py` covering every environment variable (§19.11).
- [ ] Unit tests for each module, following the recipes in §19.
- [ ] For a server: ACK and NACK tests for every validation rule (§19.5).
- [ ] For a sender: an outcome table test (§19.9).
- [ ] For a transformer: mapper tests plus at least one golden-message test (§19.3–§19.4).
- [ ] No real network, Azure, SQL or clock in unit tests.
- [ ] Synthetic data only (§20).

**CI**

- [ ] Added to `pipeline-ado/pr-validation.yml`, in both `APPS` and `paths` (unless discovery
      is in place).
- [ ] Added to `pipeline-ado/code-coverage.yml` `APPS` (unless discovery is in place).
- [ ] If deployable:
  - [ ] a `<name>-build.yml` using `code-quality-template.yml`, with `usePytest` set correctly;
  - [ ] added to `build-apps.yml`;
  - [ ] added to `release-apps.yml`.
- [ ] The first PR shows the new component's tests running in the PR-validation log.

**Local stack**

- [ ] Service added to `local/docker-compose.yml` with the right profiles (including
      `everything`).
- [ ] `local/env/<group>/<name>.env` created.
- [ ] Queues, topics and subscriptions added to `local/ServiceBusEmulatorConfig.json`.
- [ ] A sample message in `local/sample_messages/` if it is an ingress flow.
- [ ] `local/README.md` profile table updated.
- [ ] End-to-end flow test added, if an e2e suite exists (§23).

**Documentation**

- [ ] Component `README.md` has a "Running tests" section.
- [ ] This document's inventory (§4) updated.

---

## 28. Checklist: reviewing a pull request's tests

**Does the change have the right tests?**

- [ ] A bug fix includes a test that fails without the fix (§19.15).
- [ ] New behaviour has tests at the lowest sensible level. Mapper logic goes in mapper tests,
      not only in an application test.
- [ ] Changed environment variables have config tests, **and** the `local/env` files are
      updated.
- [ ] Changes to a shared library list the dependent components whose tests were run.

**Are the tests good?**

- [ ] Each test name says what is being checked and what should happen.
- [ ] Assertions check outcomes (ACK code, complete/abandon/dead-letter, mapped value), not only
      `assert_called()`.
- [ ] Mocks of real classes use `spec`/`autospec`. Stacked `@patch` parameters are in the right
      order (bottom-up).
- [ ] No `time.sleep`, real sockets, real network or unpatched `datetime.now()` in unit tests.
- [ ] `os.environ` is isolated with `patch.dict(..., clear=True)`.
- [ ] `subTest` cases rebuild their inputs.
- [ ] Failure paths are tested, not only happy paths.
- [ ] Logs on failure paths contain no patient-identifiable data (§21.3).
- [ ] Test data is synthetic and uses reserved identifiers (§20).

**Will CI actually run them?**

- [ ] The component is in PR validation, or the PR adds it.
- [ ] The tests are discoverable by the component's runner (file named `test_*.py`, class
      derives from `TestCase` or is collected by pytest).
- [ ] mypy passes over `tests/`. Run `uv tool run mypy <pkg> tests --ignore-missing-imports`
      locally.
- [ ] Integration tests skip themselves cleanly without their environment.

**Maintenance**

- [ ] No copy-pasted test modules from another component (G16, G17).
- [ ] No new tests added to `hl7_rest_server` or `hl7_soap_server` unless the same change is
      made in `rest_server`.
- [ ] Coverage did not drop for the component (check the post-merge coverage run, or run
      coverage locally).

---

## 29. Troubleshooting

**`ModuleNotFoundError: No module named '<shared_lib>'` when running tests**

Run tests through the component's environment (`uv run …`) and from the component's directory.
If you just pulled, run `uv sync --all-groups` first. For a fresh machine, run
`dev_tools/scripts/uv_sync_all.sh`.

**A service's tests behave as if a shared-library change isn't there**

uv may be serving a cached build of the path dependency. Run `uv sync --all-groups --reinstall`
in the service directory, as `run-all-tests.sh` does.

**`Ran 0 tests` with unittest**

- You ran it from the wrong directory. Run from the component root, not from `tests/`.
- The file isn't named `test*.py`.
- The tests are bare pytest functions, not methods on a `TestCase`. Use `uv run pytest`.
- An import error in the module was reported as a failed `unittest.loader._FailedTest`. Scroll
  up in the output.

**Dotted test names fail (`ModuleNotFoundError: No module named 'tests.test_x'`)**

That component has no `tests/__init__.py` (§5.2). Use `discover tests -p "test_x.py"`, or add
the `__init__.py` (§26.8).

**mypy passes locally (via `check.sh`) but fails in CI**

CI runs mypy over `tests/` as well, and with the newest mypy (§9.1, G9, G10). Reproduce with:

```bash
uv tool run mypy <pkg> tests --ignore-missing-imports
```

Common causes:

- a test method without `-> None`, when it calls something that mypy then checks strictly;
- a mock passed where a typed argument is expected. Annotate it as `MagicMock`, or use
  `cast(...)`.

**bandit fails locally but CI passes, or the other way round**

`check.sh` runs bandit at the default severity (low), and CI at `medium` (G11). `assert` in tests
is allowed by `[tool.bandit.assert_used] skips`. Subprocess use in test runners needs
`# nosec B603 B607` with a justification, as in `run_integration_test.py`.

**`uv audit` fails in CI on a dependency you didn't touch**

A new advisory was published. Run `dev_tools/scripts/quality_report.py --checks security
<component>`, then `quality_fix.py --audit --apply --recheck <component>`. If there is no fixed
version, discuss an ignore with the team. Don't add one alone.

**Dashboard tests try to contact Cosmos DB**

You ran them with unittest (for example through `run-all-tests.sh`), so the autouse
`_disable_cosmos` fixture in `conftest.py` didn't apply (§9.2). Use `uv run pytest`.

**Tests pass alone but fail in the full run, or the other way round**

Shared state is leaking between tests. Common culprits:

- module-level caches;
- a cached client (`cosmos_store._reset_client_for_tests()` exists for this reason);
- `os.environ` changes without `clear=True`;
- hl7apy message objects reused across cases.

Run the failing pair in a fixed order to confirm, then isolate the state.

**A test hangs**

Look for an unpatched `time.sleep` in a retry loop, a real socket waiting on `recv`, a
`threading.Thread` that was never joined, or a Service Bus receiver mock whose
`receive_messages` returns messages forever. A `side_effect` list ending in `[]` stops the loop.

**`_tkinter` not found (Ultra7, Component Tester)**

Homebrew Python lacks Tk. Run `uv python install 3.14` and let `.python-version` select it (see
`dev_tools/ultra7/README.md`). On headless CI agents, Ultra7's GUI tests skip themselves.

**Cosmos emulator TLS errors**

The emulator uses a self-signed certificate. The opt-in tests disable verification
(`COSMOS_DISABLE_SSL_VERIFY=true`, `connection_verify=False`). Never do this against a real
account.

**`the classic builder doesn't support additional contexts` when building images**

BuildKit is needed for Compose `additional_contexts` (`shared_libs`, `ca-certs`). The justfile
exports `DOCKER_BUILDKIT=1` and `COMPOSE_DOCKER_CLI_BUILD=1`. If you call Compose directly, export
them yourself.

**`mllp_send: command not found`**

Run `just install`, or `python3 -m pip install hl7`, in the interpreter that `just` uses.

**The ACK printed by `mllp_send` looks garbled**

HL7 separates segments with CR only, and terminals overwrite the line. `just send` already
translates CR to LF. When calling `mllp_send` directly, pipe it through `tr '\r' '\n'`.

**`docker compose --profile "*" up` starts nothing useful**

`"*"` is not a wildcard (§12.4). Use the `everything` profile, but note that it omits
`phw-hl7-server` and `chemo-hl7-transformer`, or pass several `--profile` flags.

**Port 8085 already in use**

`risp-hl7-rest-server` and the external PIMS reference-data service both use host port 8085
(§12.8). Don't run both at once, or remap one of them.

**The Service Bus emulator or SQL Edge won't start on Apple Silicon**

See the "Apple Silicon (M series) Mac Setup" section of `local/README.md`. Docker needs x86
emulation enabled, as `CLAUDE.md`'s troubleshooting list says.

**A `.hl7` fixture stops parsing after a commit**

Line endings were converted. Make sure fixtures keep `\r` (add `*.hl7 -text` to
`.gitattributes`), or normalise in the test with `.replace("\n", "\r")`.

**The coverage dashboard shows a drop, but PR validation was green**

The coverage pipeline runs more components than PR validation (G1) and doesn't fail on test
failures (§11.3). Check its log for `##[warning]Some apps had failing tests`. The drop may be a
component that PR validation never runs.

---

## 30. Prioritised roadmap

A suggested order, balancing value against effort. Each step can be done on its own.

| # | Step | Closes | Effort | Value |
|---|---|---|---|---|
| 1 | Add the 10 missing components to PR validation `APPS` and `paths` | G2, G5 | S | **Very high**: about 1,230 tests start gating merges |
| 2 | Make `CLAUDE.md`, `AGENTS.md` and `.github/copilot-instructions.md` agree on testing conventions; link to one testing document | G23 | S | High |
| 3 | Use pytest as the runner everywhere (no test rewrites) and publish JUnit results | G7, G14, G34 | S–M | High |
| 4 | Run ruff, mypy and bandit from the pinned `dev` groups in CI; align versions; same flags in `check.sh` | G9–G12 | M | High |
| 5 | Replace the `APPS` arrays with discovery plus `[tool.inthub]` | G1–G4, G7 | M | High |
| 6 | Add the local-stack consistency check (§26.9) and fix the documented drifts | G25–G27 | S–M | Medium |
| 7 | Agree a synthetic data policy (reserved NHS numbers) and review the samples | G35 | S | High (governance) |
| 8 | Deduplicate `test_field_utils.py` (and the underlying code) | G16 | S | Medium |
| 9 | Add a real-socket MLLP round-trip test to PR validation | G31 | M | High |
| 10 | Add emulator-backed integration tests for Service Bus and SQL, run nightly | G31 | M | High |
| 11 | Automate end-to-end flow tests on the Compose stack, run nightly (§23) | G29 | L | **Very high** |
| 12 | Per-component `fail_under` ratchet, then diff coverage on PRs | G34 | S–M | Medium |
| 13 | Post-deployment smoke tests in `release-apps.yml` | G33 | M | High |
| 14 | Golden-message tests for every transformer and message type | — | M | High |
| 15 | Contract tests between adjacent services (server → transformer → sender message shape and Service Bus properties) | G30 | M | Medium |
| 16 | Property-based tests (for example with `hypothesis`) for HL7 escaping and ER7↔XML conversion | G32 | M | Medium |
| 17 | Decommission `hl7_rest_server` and `hl7_soap_server` and their duplicated tests | G17 | M | Medium |
| 18 | Load tests for the `load` environment | G33 | L | Depends on the team's non-functional requirements |

---

# Appendices

## Appendix A. Command cheat sheet

```bash
# ── One component (run from its directory) ────────────────────────────────────
uv sync --all-groups                                   # create/refresh .venv
uv run python -m unittest discover tests -v            # unittest components
uv run pytest -v                                       # pytest components
uv run python -m unittest tests.test_x.TestY.test_z    # one test (needs tests/__init__.py)
uv run python -m unittest discover tests -k ack        # by name
uv run pytest -k "ack and not integration" -x          # pytest selection
bash check.sh                                          # local gate (if present)
uv run coverage run --source=. -m unittest discover tests && uv run coverage report -m
uv run coverage html                                   # htmlcov/index.html

# ── CI-identical checks for one component ─────────────────────────────────────
PKG=$(basename "$PWD")
uv tool run ruff check --output-format=github .
uv tool run bandit -r "$PKG" tests --severity-level medium
uv audit --locked --ignore GHSA-5239-wwwm-4pmq
uv tool run mypy "$PKG" tests --ignore-missing-imports

# ── Whole repository (run from the root) ──────────────────────────────────────
dev_tools/scripts/uv_sync_all.sh
dev_tools/scripts/run-all-tests.sh                     # unittest everywhere (see §9.2)
dev_tools/scripts/run-all-ruff.sh
dev_tools/scripts/run-all-mypy.sh
dev_tools/scripts/run-all-security.sh
dev_tools/scripts/quality_report.py                    # HTML + JSON report
dev_tools/scripts/quality_report.py --checks tests     # just tests, right runner per component
dev_tools/scripts/quality_fix.py --safe                # dry run
dev_tools/scripts/quality_fix.py --safe --apply --recheck

# ── Opt-in integration tests ──────────────────────────────────────────────────
cd local && docker compose --profile dashboard up -d cosmos-emulator
cd dashboard && RUN_COSMOS_EMULATOR_TESTS=1 uv run pytest tests/test_alarm_pause_emulator.py -v
cd lookup_service && LOOKUP_IT_COSMOS_ENDPOINT=https://localhost:8081 LOOKUP_IT_COSMOS_KEY=<key> \
  uv run python -m unittest tests.test_cosmos_store -v

# ── Local end-to-end (from local/) ────────────────────────────────────────────
just secrets
just start phw-to-mpi
just send ./sample_messages/phw-to-mpi.sample.hl7            # MLLP 2575
just send ./sample_messages/chemocare-to-mpi.sample.hl7 2578
just send-soap ./sample_messages/lims-to-mpi.sample.xml 8082
just send-rest ./sample_messages/wpas-to-proms.xml 8084       # pass the file: default is wrong
just logs phw-hl7-transformer mpi-hl7-sender mpi-hl7-topic-mock-receiver
open http://127.0.0.1:8080                                   # BusWatch
just restart phw-to-mpi                                      # after code changes
just stop
```

## Appendix B. Glossary

| Term | Meaning |
|---|---|
| **ACK / NACK** | HL7 acknowledgement message. `MSA-1` is `AA` (accept), `AE` (error, retryable) or `AR` (reject). `CA`/`CE`/`CR` are the enhanced-mode equivalents. A NACK is an ACK with `AE` or `AR` |
| **ADT** | Admit/Discharge/Transfer: the HL7 message family used here, for example `ADT^A28` (add person), `ADT^A31` (update person), `ADT^A40` (merge patient) |
| **Assigning authority** | The organisation that issued an identifier (`CX-4`), for example `NHS` or `328` for LIMS |
| **Cobertura** | The XML coverage format produced by `coverage xml` and read by ADO and ReportGenerator |
| **DLQ / dead-letter** | The Service Bus sub-queue for messages that can't be processed |
| **DTE / TST / UAT** | Deployment environments after dev, in `release-apps.yml` |
| **ER7** | HL7 v2 "pipe and hat" text encoding (`MSH|^~\&|…`), segments separated by `\r` |
| **FHIR** | HL7 Fast Healthcare Interoperability Resources. The PROMS transformer produces FHIR bundles |
| **Golden test** | A test that compares complete output with a stored expected output |
| **Hermetic test** | A test that depends on nothing outside the process: no network, clock or shared state |
| **hl7apy** | The Python HL7 v2 library used to parse and build messages |
| **MLLP** | Minimal Lower Layer Protocol: HL7 over TCP, framed by `0x0B` … `0x1C 0x0D` |
| **MPI** | Master Patient Index, the main destination of most flows |
| **MSH / PID / PV1 / PD1 / EVN / MRG / NK1** | HL7 segments: message header, patient identification, patient visit, additional demographics, event type, merge information, next of kin |
| **NOC** | Network Operations Centre: the audience for the dashboard |
| **Profile (Compose)** | A named group of services in `docker-compose.yml`, started with `--profile` |
| **PROMS** | Patient Reported Outcome Measures, the destination of the WPAS flow |
| **ReportGenerator** | The .NET tool that merges coverage reports |
| **Session (Service Bus)** | A grouping key that guarantees FIFO processing for messages with the same session ID |
| **WRDS** | Welsh Reference Data Service, a SOAP service for code mappings |
| **WRRS** | The destination database for RISP and LIMS classification flows |

## Appendix C. Environment variables that affect tests

**Existing (at the snapshot):**

| Variable | Used by | Effect |
|---|---|---|
| `INGRESS_QUEUE_NAME` (+ the sender's full env) | `hl7_sender/tests/test_throttling_local.py` | Enables the throttling integration test |
| `RUN_COSMOS_EMULATOR_TESTS=1` | `dashboard/tests/test_alarm_pause_emulator.py` | Enables the Cosmos emulator tests |
| `COSMOS_ENDPOINT`, `COSMOS_KEY`, `COSMOS_DATABASE`, `COSMOS_CONTAINER`, `COSMOS_DISABLE_SSL_VERIFY` | dashboard | Cosmos connection. Must be a local emulator for the emulator tests. Disabled for unit tests by `conftest.py` |
| `LOOKUP_IT_COSMOS_ENDPOINT`, `LOOKUP_IT_COSMOS_KEY` | `lookup_service/tests/test_cosmos_store.py` | Enable the Cosmos integration test |
| `WRDS_LOCAL_FIXTURE_PATH` | `hl7_core_reference_transformer` (runtime) | WRDS fixture mode |
| `REFERENCE_DATA_API_BASE_URL` | `hl7_pims_transformer` (runtime, local stack) | Reference-data API location |
| `BUSWATCH_QUEUE_NAMES` | BusWatch | Queues to show. Empty means discover them from the emulator config |
| `UV_CACHE_DIR` | CI | uv cache location for `Cache@2` |
| `DOCKER_BUILDKIT`, `COMPOSE_DOCKER_CLI_BUILD` | local image builds | Needed for `additional_contexts` |

**Proposed in this document:**

| Variable | Purpose |
|---|---|
| `RUN_INTEGRATION_TESTS=1` | Single switch for all emulator-backed integration tests (§22.2) |
| `IT_SERVICE_BUS_CONNECTION_STRING`, `IT_SQL_CONNECTION_STRING`, `IT_COSMOS_ENDPOINT`, `IT_COSMOS_KEY` | Integration test targets (local only) |
| `RUN_E2E_TESTS=1` | Enable end-to-end flow tests (§23) |
| `E2E_KEEP_STACK=1` | Leave the Compose stack running after e2e tests, for debugging |

## Appendix D. Key files index

All links are pinned to commit `636a050`.

Base: `https://github.com/DHCW-Digital-Health-and-Care-Wales/Integration-Hub-Beta/blob/636a050fe5f9f0e5dd10045e1af99214c681637f/`

| File | Why it matters for testing |
|---|---|
| [`pipeline-ado/pr-validation.yml`](https://github.com/DHCW-Digital-Health-and-Care-Wales/Integration-Hub-Beta/blob/636a050fe5f9f0e5dd10045e1af99214c681637f/pipeline-ado/pr-validation.yml) | The merge gate: the `APPS` list and the per-app steps |
| [`pipeline-ado/code-coverage.yml`](https://github.com/DHCW-Digital-Health-and-Care-Wales/Integration-Hub-Beta/blob/636a050fe5f9f0e5dd10045e1af99214c681637f/pipeline-ado/code-coverage.yml) | The post-merge coverage pipeline |
| [`pipeline-ado/templates/code-quality-template.yml`](https://github.com/DHCW-Digital-Health-and-Care-Wales/Integration-Hub-Beta/blob/636a050fe5f9f0e5dd10045e1af99214c681637f/pipeline-ado/templates/code-quality-template.yml) | The quality gate in build pipelines |
| [`pipeline-ado/build-apps.yml`](https://github.com/DHCW-Digital-Health-and-Care-Wales/Integration-Hub-Beta/blob/636a050fe5f9f0e5dd10045e1af99214c681637f/pipeline-ado/build-apps.yml) | Manual consolidated build |
| [`pipeline-ado/release-apps.yml`](https://github.com/DHCW-Digital-Health-and-Care-Wales/Integration-Hub-Beta/blob/636a050fe5f9f0e5dd10045e1af99214c681637f/pipeline-ado/release-apps.yml) | Deployment environments |
| [`docs/code_coverage.md`](https://github.com/DHCW-Digital-Health-and-Care-Wales/Integration-Hub-Beta/blob/636a050fe5f9f0e5dd10045e1af99214c681637f/docs/code_coverage.md) | Coverage plan and ADO dashboard setup |
| [`docs/deep_eval.md`](https://github.com/DHCW-Digital-Health-and-Care-Wales/Integration-Hub-Beta/blob/636a050fe5f9f0e5dd10045e1af99214c681637f/docs/deep_eval.md) | 2026-07 evaluation, including the testing section (partly outdated) |
| [`dev_tools/scripts/run-all-tests.sh`](https://github.com/DHCW-Digital-Health-and-Care-Wales/Integration-Hub-Beta/blob/636a050fe5f9f0e5dd10045e1af99214c681637f/dev_tools/scripts/run-all-tests.sh) | Run every suite locally |
| [`dev_tools/scripts/quality_report.py`](https://github.com/DHCW-Digital-Health-and-Care-Wales/Integration-Hub-Beta/blob/636a050fe5f9f0e5dd10045e1af99214c681637f/dev_tools/scripts/quality_report.py) | Whole-repo quality report |
| [`dev_tools/scripts/quality_fix.md`](https://github.com/DHCW-Digital-Health-and-Care-Wales/Integration-Hub-Beta/blob/636a050fe5f9f0e5dd10045e1af99214c681637f/dev_tools/scripts/quality_fix.md) | Manual for the automatic fixer |
| [`local/README.md`](https://github.com/DHCW-Digital-Health-and-Care-Wales/Integration-Hub-Beta/blob/636a050fe5f9f0e5dd10045e1af99214c681637f/local/README.md) | Local stack documentation |
| [`local/docker-compose.yml`](https://github.com/DHCW-Digital-Health-and-Care-Wales/Integration-Hub-Beta/blob/636a050fe5f9f0e5dd10045e1af99214c681637f/local/docker-compose.yml) | Services and profiles |
| [`local/justfile`](https://github.com/DHCW-Digital-Health-and-Care-Wales/Integration-Hub-Beta/blob/636a050fe5f9f0e5dd10045e1af99214c681637f/local/justfile) | `just` recipes (Podman), including `send*` |
| [`local/ServiceBusEmulatorConfig.json`](https://github.com/DHCW-Digital-Health-and-Care-Wales/Integration-Hub-Beta/blob/636a050fe5f9f0e5dd10045e1af99214c681637f/local/ServiceBusEmulatorConfig.json) | Emulator queues and topics |
| [`dashboard/tests/conftest.py`](https://github.com/DHCW-Digital-Health-and-Care-Wales/Integration-Hub-Beta/blob/636a050fe5f9f0e5dd10045e1af99214c681637f/dashboard/tests/conftest.py) | Hermetic-by-default fixture |
| [`servers/hl7_server/tests/test_generic_handler.py`](https://github.com/DHCW-Digital-Health-and-Care-Wales/Integration-Hub-Beta/blob/636a050fe5f9f0e5dd10045e1af99214c681637f/servers/hl7_server/tests/test_generic_handler.py) | Model for handler and ACK tests |
| [`shared_libs/message_bus_lib/tests/test_MessageReceiverClient.py`](https://github.com/DHCW-Digital-Health-and-Care-Wales/Integration-Hub-Beta/blob/636a050fe5f9f0e5dd10045e1af99214c681637f/shared_libs/message_bus_lib/tests/test_MessageReceiverClient.py) | Model for Service Bus mocking |
| [`servers/rest_server/tests/hl7/test_message_route.py`](https://github.com/DHCW-Digital-Health-and-Care-Wales/Integration-Hub-Beta/blob/636a050fe5f9f0e5dd10045e1af99214c681637f/servers/rest_server/tests/hl7/test_message_route.py) | Model for FastAPI route tests |
| [`transformers/hl7_pims_transformer/tests/fakes.py`](https://github.com/DHCW-Digital-Health-and-Care-Wales/Integration-Hub-Beta/blob/636a050fe5f9f0e5dd10045e1af99214c681637f/transformers/hl7_pims_transformer/tests/fakes.py) | Model for hand-written fakes |
| [`shared_libs/event_logger_lib/tests/test_redaction.py`](https://github.com/DHCW-Digital-Health-and-Care-Wales/Integration-Hub-Beta/blob/636a050fe5f9f0e5dd10045e1af99214c681637f/shared_libs/event_logger_lib/tests/test_redaction.py) | Model for privacy assertions |
| [`servers/rest_server/tests/test_hl7_soap_server_parity.py`](https://github.com/DHCW-Digital-Health-and-Care-Wales/Integration-Hub-Beta/blob/636a050fe5f9f0e5dd10045e1af99214c681637f/servers/rest_server/tests/test_hl7_soap_server_parity.py) | Model for characterisation tests |
| [`senders/hl7_sender/tests/test_throttling_local.py`](https://github.com/DHCW-Digital-Health-and-Care-Wales/Integration-Hub-Beta/blob/636a050fe5f9f0e5dd10045e1af99214c681637f/senders/hl7_sender/tests/test_throttling_local.py) | Opt-in Service Bus integration test |
| [`dev_tools/integration_hub_tester/README.md`](https://github.com/DHCW-Digital-Health-and-Care-Wales/Integration-Hub-Beta/blob/636a050fe5f9f0e5dd10045e1af99214c681637f/dev_tools/integration_hub_tester/README.md) | Component Tester GUI |

## Appendix E. Open questions for the team

These came up while writing this document and need a team decision or knowledge the repository
doesn't contain:

1. **What does `Integration-Hub-tests` contain,** and how is it run (against the local stack,
   dev, or TST)? Should end-to-end tests live there or here (§23.2)?
2. **Was leaving components out of PR validation deliberate?** For example, the dashboard is
   validated only in its build pipeline. If so, record why next to the `APPS` array.
3. **Which runner should be the standard:** unittest, pytest, or pytest-as-runner over
   unittest-style tests (§24.6)?
4. **Is Docker available on the self-hosted ADO pool** for emulator-backed integration and
   end-to-end jobs (§22.4, §23.6)?
5. **Which test NHS number range** does DHCW Information Governance want used, and are the
   realistic-looking sample messages synthetic (§20.1, §15.3)?
6. **What should the coverage targets be,** and should shared libraries carry higher targets than
   services (§25.2)?
7. **Should the release pipeline include smoke tests,** and is there a load-testing plan for the
   `load` environment (§24.8)?
8. **When will `hl7_rest_server` and `hl7_soap_server` be decommissioned,** so that their
   duplicated tests can be removed (§26.4)?
9. **Are `message-store-service` and `message-replay-job` meant to be in the local Compose
   stack** (G25)?

## Appendix F. Complete test module inventory

Generated from the repository at commit `636a050`. For each component, every `tests/` module is listed
with its static test count, its `TestCase` (or other) class names, and the first line of its module
docstring where one exists. Helper modules (no tests) are included because other tests import them.

### `dashboard` — 31 files, 673 tests

| Module | Tests | Classes | Docstring |
|---|---:|---|---|
| `conftest.py` | 0 | — | Shared pytest fixtures for the dashboard test suite. |
| `test_alarm1.py` | 2 | `TestSendAlarmEmail` | Unit tests for Alarm 1 (inactivity) service logic. |
| `test_alarm2.py` | 36 | `TestApplicableThreshold`, `TestBuildRow`, `TestFormatDuration`, `TestGetAlarm2ConfigPageData`, `TestGenerateRuleId`, `TestSendAlarm2Email` | Unit tests for Alarm 2 service logic. |
| `test_alarm3.py` | 2 | `TestSendAlarm3Email` | Unit tests for Alarm 3 (message processing failures) service logic. |
| `test_alarm_base.py` | 29 | `TestLoadSaveConfig`, `TestLoadSaveState`, `TestResolvePause`, `TestClearExpiredLegacyPause`, `TestPauseRuleNowAndResume`, `TestPauseUnpauseRule`, `TestParseLogAnalyticsDatetime`, `TestFormatDuration` | Unit tests for dashboard.services.alarm_base — the shared Cosmos DB config/ |
| `test_alarm_flow_config.py` | 27 | `TestFlowConfigPageGet`, `TestFlowConfigPagePost`, `TestFlowConfigScoping`, `TestLegacyConfigRedirects`, `TestScopedForm` | Tests for the per-flow alarm configuration screen (''/alarms/config''). |
| `test_alarm_pause_emulator.py` | 5 | `EmulatorRule`, `TestBaseline`, `TestFlowPause`, `TestScheduledPause`, `TestSingleRulePause` | Local integration tests: alarm pauses against the real Cosmos DB emulator. |
| `test_alarm_pause_evaluation.py` | 9 | `TestAlarm1PauseGating`, `TestAlarm2PauseGating`, `TestAlarm3PauseGating` | Evaluator tests: pause records gate Alarms 1, 2 and 3. |
| `test_alarm_pause_routes.py` | 22 | `TestCreatePauseApi`, `TestCancelPauseApi`, `TestLegacyRuleEndpoints`, `TestPatchCachedRows`, `TestPausePagesAndListing` | Route tests for the alarm pause endpoints in dashboard.routes.alarms. |
| `test_alarm_pauses.py` | 68 | `TestPauseStatus`, `TestCovers`, `TestActiveAndScheduledLookup`, `TestRowHelpers`, `TestGroupForDisplay`, `TestParseLondonDatetime`, `TestBuildPause`, `TestPersistence`, `TestCancelPause`, `TestCancelRulePause` | Unit tests for dashboard.services.alarm_pauses — scheduled alarm pause records. |
| `test_alarm_time_utils.py` | 31 | `TestWeekendPeriod`, `TestDayPeriod`, `TestEveningPeriod`, `TestUTCNormalisation` | Unit tests for alarm_time_utils.get_current_period(). |
| `test_app.py` | 89 | `TestPageRoutes`, `TestSetLanguage`, `TestNavEnvLabel`, `TestEmailAlertsConfigured`, `TestApiRoutes`, `TestNetworkTestRoutes`, `TestNetworkTestConfigRoutes`, `TestAlarmsByFlowRoutes`, `TestRowsForFlow`, `TestFlowOptions`, `TestContactsRoutes`, `TestEnvLoading` | Unit tests for the Flask routes. |
| `test_arm.py` | 14 | `TestClassifyApp`, `TestBuildFlow` | Unit tests for Container App discovery (arm.py). |
| `test_azure_monitor.py` | 30 | `FakeResponse`, `FakeTable`, `TestGetContainerAppMetricHistoryValidation`, `TestGetContainerAppMetricHistoryIntervalSelection`, `TestGetContainerAppMetricHistoryParsing`, 14 module-level test functions | — |
| `test_cache.py` | 12 | `TestCached`, `TestCachedNowait`, `TestIsCacheStale`, `TestMultiCachedNowait` | Unit tests for dashboard.services.cache — the generic TTL / stale-while- |
| `test_config.py` | 40 | `TestConfigDefaults`, `TestEnvironmentLabel`, `TestEnvironmentColour` | Unit tests for configuration defaults. |
| `test_contacts_store.py` | 44 | `TestAddContact`, `TestUpdateContact`, `TestDeleteContact`, `TestRestoreContact`, `TestPurgeExpiredContacts`, `TestListContacts`, `TestContactsForFlowAndByFlow`, `TestParseCsv`, `TestImportContacts` | Unit tests for dashboard.services.contacts_store. |
| `test_container_apps.py` | 2 | `TestBuildAppFlowMap` | — |
| `test_cosmos_store.py` | 38 | `TestIsConfigured`, `TestGetDocument`, `TestQueryDocuments`, `TestUpsertDocument`, `TestCreateDocument`, `TestDeleteDocument`, `TestGetClient`, `TestGetContainer` | Unit tests for the Cosmos DB persistence layer (dashboard.services.cosmos_store). |
| `test_credentials.py` | 3 | `TestAzureCredentialSelection` | Unit tests for Azure credential selection. |
| `test_email_ooh_guard.py` | 12 | `TestAlarm1EmailOohGuard`, `TestAlarm2EmailOohGuard`, `TestAlarm3EmailOohGuard` | Unit tests for the Email OOH guard on alarm config POST handlers. |
| `test_email_service.py` | 13 | `TestGetAcsConnectionString`, `TestSendAlertEmail`, `TestSendAlertEmailRetry` | Unit tests for dashboard.services.email_service. |
| `test_flow_sources.py` | 35 | `TestAddSource`, `TestUpdateSource`, `TestDeleteSource`, `TestListSources`, `TestListSourcesAsEndpointOptions`, `TestParseCsv`, `TestImportSources` | Unit tests for dashboard.services.flow_sources. |
| `test_flows.py` | 30 | `TestQueueHealth`, `TestFlowHealth`, `TestOverallHealth`, `TestBuildFlowData`, `TestMergeSubscriptionRecords` | Unit tests for flow health calculation logic. |
| `test_form_utils.py` | 6 | `TestParseIntFormField` | Unit tests for dashboard.services.form_utils. |
| `test_network_test.py` | 33 | `TestValidateHost`, `TestValidatePort`, `TestResolveHost`, `TestRunLatencyTest`, `TestHistoryPersistence`, `TestListTestedEndpoints`, `TestBuildEndpointOptions` | Unit tests for dashboard.services.network_test. |
| `test_seed_example_alarms.py` | 11 | `TestEnsureLocalEndpoint`, `TestBuildExampleRules`, `TestMergeAndRemove`, `TestBuildExamplePauses` | Unit tests for scripts/seed_example_alarms.py — the local-emulator example data seeder. |
| `test_service_bus.py` | 1 | 1 module-level test functions | — |
| `test_status_builder.py` | 3 | 3 module-level test functions | — |
| `test_template_filters.py` | 11 | `TestFormatBytes`, `TestHealthBadge` | Unit tests for dashboard.template_filters — the Jinja2 filters extracted |
| `test_traces.py` | 15 | `FakeTable`, `FakeResponse`, `TestOperationIdValidation`, `TestNoWorkspaceConfigured`, `TestQueryScoping`, `TestParseSpans`, `TestDurationComputation`, `TestErrorHandling` | Tests for dashboard.services.traces. |

### `dev_tools/buswatch` — 2 files, 15 tests

| Module | Tests | Classes | Docstring |
|---|---:|---|---|
| `test_config.py` | 3 | `TestConfigHelpers`, `TestGetSettings` | — |
| `test_servicebus_reader.py` | 12 | `_FakeMessage`, `TestEmulatorConfigParsing`, `TestReceiverSelection`, `TestMessageFormatting`, `TestSessionErrorDetection` | — |

### `dev_tools/scripts` — 2 files, 75 tests

| Module | Tests | Classes | Docstring |
|---|---:|---|---|
| `test_quality_fix.py` | 55 | `TempDirTestCase`, `TestVersions`, `TestUpdateRequirement`, `TestUnusedIgnore`, `TestSuppressions`, `TestAnnotateNone`, `TestRequestTimeout`, `TestManifestEdits`, `TestPlanning`, `TestCli` | — |
| `test_quality_report.py` | 20 | `TestHelpers`, `TestParseRuff`, `TestParseMypy`, `TestParseUnittest`, `TestParseJunit`, `TestUsesPytest`, `TestParseBandit`, `TestParseUvAudit`, `TestRenderHtml` | — |

### `dev_tools/ultra7` — 12 files, 106 tests

| Module | Tests | Classes | Docstring |
|---|---:|---|---|
| `test_detect.py` | 8 | `TestDetectFormat` | — |
| `test_editor_pane.py` | 24 | `TestEditorPaneDisabledMessages`, `TestEditorPaneMultiSelect`, `TestEditorPaneLoadFromDisk`, `TestEditorPaneNewMessage` | — |
| `test_export.py` | 5 | `TestExtensionFor`, `TestSanitizeFilename`, `TestUniqueFilename` | — |
| `test_iteration.py` | 9 | `TestApplyIterationIncrement`, `TestApplyIterationList`, `TestApplyIterationTimestamp`, `TestApplyIterationBounds` | — |
| `test_log_panel.py` | 8 | `TestLogPanel` | — |
| `test_models.py` | 11 | `TestMessage`, `TestIterationSpec`, `TestEndpoint`, `TestProject` | — |
| `test_pretty_print.py` | 5 | `TestPrettyPrintJson`, `TestPrettyPrintXml`, `TestPrettyPrintUnsupportedFormat` | — |
| `test_send_controls.py` | 10 | `TestSendWorkerLabels` | — |
| `test_senders.py` | 10 | `TestMllpSender`, `TestRestSender`, `TestSoapSender` | — |
| `test_settings.py` | 3 | `TestSettings` | — |
| `test_storage.py` | 6 | `TestProjectFilename`, `TestProjectStore` | — |
| `test_themes.py` | 7 | `TestThemes` | — |

### `lookup_service` — 13 files, 160 tests

| Module | Tests | Classes | Docstring |
|---|---:|---|---|
| `helpers.py` | 0 | `MemoryTableStore`, `SeedDirTestCase` | — |
| `test_api.py` | 32 | `LookupApiTests`, `UploadApiTests`, `SettingsApiTests`, `WritesDisabledTests`, `HealthTests`, `BackendFailureTests`, `SwaggerGatingTests` | — |
| `test_config.py` | 6 | `SettingsTests` | — |
| `test_cosmos_batching.py` | 3 | `BatchingTests` | — |
| `test_cosmos_store.py` | 3 | `CosmosTableStoreIntegrationTests` | Opt-in integration tests against a real Cosmos DB (the local emulator). |
| `test_csv_import.py` | 15 | `ReadCsvTests`, `ImportRowsTests`, `BuildDefinitionTests`, `TransformTests` | — |
| `test_file_store.py` | 30 | `RepoSeedTests`, `FileTableStoreLoadTests`, `FileTableStoreValidationTests` | — |
| `test_gui.py` | 25 | `OverviewPageTests`, `TablePageTests`, `TablePagingTests`, `UploadPageTests`, `TableSettingsFormTests`, `EscapingTests` | — |
| `test_keys.py` | 23 | `NormalisePartTests`, `CanonicalKeyTests`, `EncodeRowIdTests`, `TableDefinitionTests` | — |
| `test_resolver.py` | 10 | `ResolverTests` | — |
| `test_seed.py` | 5 | `SeedImportTests`, `SeedValidationTests` | — |
| `test_table_cache.py` | 4 | `FakeClock`, `TableCacheTests` | — |
| `test_ttl.py` | 4 | `TtlTests` | — |

### `message_processing/message_replay_job` — 4 files, 58 tests

| Module | Tests | Classes | Docstring |
|---|---:|---|---|
| `test_app_config.py` | 9 | `TestAppConfig` | — |
| `test_db_client.py` | 22 | `TestDatabaseClient` | — |
| `test_message_replay_job.py` | 23 | `TestMessageReplayJobRun`, `TestMessageReplayJobFetchRetry`, `TestMessageReplayJobSendRetry`, `TestMessageReplayJobLoadedUpdateFailure`, `TestBuildMessages`, `TestMessageReplayJobOversizedMessage`, `TestMessageReplayJobOperationTimeout`, `TestMessageReplayJobServiceBusError`, `TestRetryOnce` | — |
| `test_replay_record.py` | 4 | `TestReplayRecord` | — |

### `message_processing/message_store_service` — 6 files, 39 tests

| Module | Tests | Classes | Docstring |
|---|---:|---|---|
| `test_app_config.py` | 4 | `TestAppConfig` | — |
| `test_application.py` | 1 | `TestApplication` | — |
| `test_db_client.py` | 18 | `TestDatabaseClient` | — |
| `test_message_record.py` | 2 | `TestMessageRecord` | — |
| `test_message_record_builder.py` | 9 | `TestBuildMessageRecord`, `TestBuildMessageRecords` | — |
| `test_message_store_service.py` | 5 | `TestMessageStoreServiceInit`, `TestMessageStoreServiceRun`, `TestProcessMessages` | — |

### `mock_receivers/hl7_mock_receiver` — 5 files, 14 tests

| Module | Tests | Classes | Docstring |
|---|---:|---|---|
| `test_app_config.py` | 3 | `TestAppConfig` | — |
| `test_error_handler.py` | 2 | `TestErrorHandler` | — |
| `test_generic_handler.py` | 4 | `TestGenericHandler` | — |
| `test_generic_handler_no_service_bus.py` | 1 | `TestGenericHandlerNoServiceBus` | — |
| `test_hl7_mock_receiver_application.py` | 4 | `TestHl7ServerApplication`, `TestHl7ServerApplicationWithoutServiceBus` | — |

### `mock_receivers/hl7_topic_mock_receiver` — 5 files, 17 tests

| Module | Tests | Classes | Docstring |
|---|---:|---|---|
| `test_app_config.py` | 4 | `TestAppConfig` | — |
| `test_error_handler.py` | 2 | `TestErrorHandler` | — |
| `test_generic_handler.py` | 4 | `TestGenericHandler` | — |
| `test_generic_handler_no_service_bus.py` | 1 | `TestGenericHandlerNoServiceBus` | — |
| `test_hl7_topic_mock_receiver_application.py` | 6 | `TestHl7TopicServerApplication`, `TestHl7TopicServerApplicationWithoutServiceBus` | — |

### `mock_receivers/http_mock_receiver` — 2 files, 33 tests

| Module | Tests | Classes | Docstring |
|---|---:|---|---|
| `test_application.py` | 13 | `TestHealthEndpoint`, `TestSoapEndpoint`, `TestServiceBusForwarding` | Integration tests for the FastAPI application endpoints. |
| `test_soap_handler.py` | 20 | `TestSoapHandler`, `TestSoapResponseBuilder` | Tests for the SOAP handler (parse_soap_request) and response builder. |

### `senders/db_sender` — 4 files, 29 tests

| Module | Tests | Classes | Docstring |
|---|---:|---|---|
| `test_app_config.py` | 5 | `TestAppConfig` | — |
| `test_application.py` | 6 | `TestApplication` | — |
| `test_db_client.py` | 12 | `TestDatabaseClient` | — |
| `test_msh_extractor.py` | 6 | `TestMshExtractor` | — |

### `senders/hl7_sender` — 7 files, 60 tests

| Module | Tests | Classes | Docstring |
|---|---:|---|---|
| `run_integration_test.py` | 0 | — | Integration test runner for message throttling functionality. |
| `test_ack_processor.py` | 7 | `TestGetAckResult` | — |
| `test_app_config.py` | 10 | `TestAppConfig` | — |
| `test_application.py` | 18 | `TestProcessMessage`, `TestMainIngressSetup`, `TestBatchSizing` | — |
| `test_hl7_sender_client.py` | 17 | `TestIsSocketClosed`, `TestHL7SenderClient` | — |
| `test_message_throttler.py` | 7 | `TestMessageThrottler` | — |
| `test_throttling_local.py` | 1 | `TestMessageThrottlingIntegration` | — |

### `senders/hl7_subscription_sender` — 5 files, 44 tests

| Module | Tests | Classes | Docstring |
|---|---:|---|---|
| `test_ack_processor.py` | 7 | `TestGetAckResult` | — |
| `test_app_config.py` | 2 | `TestAppConfig` | — |
| `test_application.py` | 9 | `TestProcessMessage`, `TestBatchSizing` | — |
| `test_hl7_subscription_sender_client.py` | 19 | `TestIsSocketClosed`, `TestHL7SenderClient` | — |
| `test_message_throttler.py` | 7 | `TestMessageThrottler` | — |

### `senders/soap_sender` — 3 files, 39 tests

| Module | Tests | Classes | Docstring |
|---|---:|---|---|
| `test_message_throttler.py` | 10 | `TestMessageThrottler` | Tests for MessageThrottler — interval calculation, validation, and wait behaviour. |
| `test_soap_ack_processor.py` | 10 | `TestGetAckResult` | Tests for get_ack_result — SOAP response parsing logic. |
| `test_soap_sender_client.py` | 19 | `TestBuildSoapEnvelope`, `TestSOAPSenderClientSendMessage`, `TestBuildWisSoapEnvelope` | Tests for SOAPSenderClient — envelope building and HTTP error mapping. |

### `senders/soap_subscription_sender` — 3 files, 32 tests

| Module | Tests | Classes | Docstring |
|---|---:|---|---|
| `test_message_throttler.py` | 10 | `TestMessageThrottler` | Tests for MessageThrottler — interval calculation, validation, and wait behaviour. |
| `test_soap_ack_processor.py` | 10 | `TestGetAckResult` | Tests for get_ack_result — SOAP response parsing logic. |
| `test_soap_subscription_sender_client.py` | 12 | `TestBuildSoapEnvelope`, `TestSOAPSubscriptionSenderClientSendMessage` | Tests for SOAPSubscriptionSenderClient — envelope building and HTTP error mapping. |

### `servers/hl7_rest_server` — 8 files, 48 tests

| Module | Tests | Classes | Docstring |
|---|---:|---|---|
| `helpers.py` | 0 | — | Shared test helpers for the hl7_rest_server unit tests. |
| `test_health_routes.py` | 4 | `HealthRouteTests`, `SwaggerGatingTests` | Tests for the health/liveness/readiness routes and swagger gating. |
| `test_hl7_ack_builder.py` | 4 | `HL7AckBuilderTests` | Tests for HL7AckBuilder ACK/NACK string construction. |
| `test_hl7_message_processor.py` | 10 | `Hl7MessageProcessorTests`, `Hl7MessageProcessorRispFlowTests` | Unit tests for Hl7MessageProcessor pipeline behaviour. |
| `test_message_input_adapter.py` | 6 | `ToEr7Tests` | Tests for the message input adapter (ER7 passthrough and XML → ER7). |
| `test_message_route.py` | 8 | `MessageRouteTests` | End-to-end tests for the POST /hl7MessageReceiver route via the FastAPI TestClient. |
| `test_risp_routing.py` | 6 | `RispFlowRouterTests` | Tests for RispFlowRouter's multi-destination fan-out (plan §3a). |
| `test_risp_validation.py` | 10 | `ValidateRispMessageTests` | Tests for RISP sender/version/message-type validation (plan §3a). |

### `servers/hl7_server` — 12 files, 101 tests

| Module | Tests | Classes | Docstring |
|---|---:|---|---|
| `test_app_config.py` | 8 | `TestAppConfig` | — |
| `test_custom_message_properties.py` | 16 | `TestCustomMessageProperties` | — |
| `test_error_handler.py` | 7 | `TestErrorHandler` | — |
| `test_generic_handler.py` | 20 | `TestGenericHandler` | — |
| `test_hl7_ack_builder.py` | 11 | `TestHL7AckBuilder`, `TestHL7AckBuilderNacks` | — |
| `test_hl7_phw_specific_validation.py` | 6 | `TestPhwHL7Validator` | — |
| `test_hl7_server_application.py` | 6 | `TestHl7ServerApplicationQueue`, `TestHl7ServerApplicationTopic` | — |
| `test_hl7_validator.py` | 11 | `TestHL7Validator` | — |
| `test_metadata_utils.py` | 6 | `TestExtractMetadata`, `TestGetMetadataLogValues`, `TestCorrelationIdForLogger` | — |
| `test_mpi_outbound_validation.py` | 4 | `TestMpiOutboundValidation` | — |
| `test_size_limited_mllp_request_handler.py` | 5 | `TestSizeLimitedMLLPRequestHandler` | — |
| `test_size_limited_mllp_server.py` | 1 | `TestSizeLimitedMLLPServer` | — |

### `servers/hl7_soap_server` — 4 files, 16 tests

| Module | Tests | Classes | Docstring |
|---|---:|---|---|
| `test_app_config.py` | 4 | `TestAppConfig` | — |
| `test_hl7_soap_server_application.py` | 4 | `TestWsdlEndpoint` | — |
| `test_soap_processor.py` | 5 | `TestSoapMessageProcessor` | — |
| `test_wsdl_service.py` | 3 | `TestBuildWsdlDocument` | — |

### `servers/rest_server` — 15 files, 130 tests

| Module | Tests | Classes | Docstring |
|---|---:|---|---|
| `hl7/helpers.py` | 0 | — | Shared test helpers for the rest_server ''hl7'' pipeline unit tests. |
| `hl7/test_health_routes.py` | 4 | `HealthRouteTests`, `SwaggerGatingTests` | Tests for the health/liveness/readiness routes and swagger gating. |
| `hl7/test_hl7_ack_builder.py` | 4 | `HL7AckBuilderTests` | Tests for HL7AckBuilder ACK/NACK string construction. |
| `hl7/test_hl7_message_processor.py` | 11 | `Hl7MessageProcessorTests`, `Hl7MessageProcessorRispFlowTests` | Unit tests for Hl7MessageProcessor pipeline behaviour. |
| `hl7/test_message_input_adapter.py` | 6 | `ToEr7Tests` | Tests for the message input adapter (ER7 passthrough and XML → ER7). |
| `hl7/test_message_route.py` | 8 | `MessageRouteTests` | End-to-end tests for the POST /hl7MessageReceiver route via the FastAPI TestClient. |
| `hl7/test_risp_routing.py` | 6 | `RispFlowRouterTests` | Tests for RispFlowRouter's multi-destination fan-out. |
| `hl7/test_risp_validation.py` | 10 | `ValidateRispMessageTests` | Tests for RISP sender/version/message-type validation. |
| `test_app_config.py` | 23 | `TestAppConfig`, `TestPipelineConfig`, `TestDotenvLoading` | — |
| `test_asgi_app.py` | 9 | `TestIngestEndpoint`, `TestHealthEndpoint`, `TestOpenApiDocs` | — |
| `test_content_adapters.py` | 13 | `TestSoapContentAdapter`, `TestXmlRawContentAdapter` | — |
| `test_hl7_soap_server_parity.py` | 6 | `RestServerSoapParityTests`, `SoapParityGapFixesTests` | Characterization tests: rest_server's 'generic' pipeline (configured per docs/rest_merge.md §9 |
| `test_message_processor.py` | 8 | `TestRestMessageProcessor` | — |
| `test_rest_server_application.py` | 8 | `TestBuildContentAdapter`, `TestBuildValidator` | — |
| `test_validators.py` | 14 | `TestNoOpValidator`, `TestHl7XsdValidator`, `TestHl7ApyValidator`, `TestXsdValidator` | — |

### `shared_libs/event_logger_lib` — 2 files, 31 tests

| Module | Tests | Classes | Docstring |
|---|---:|---|---|
| `test_event_logger.py` | 20 | `TestEventLogger` | — |
| `test_redaction.py` | 11 | `TestRedactHl7Message`, `TestRedactWarningDetails` | — |

### `shared_libs/field_utils_lib` — 1 files, 32 tests

| Module | Tests | Classes | Docstring |
|---|---:|---|---|
| `test_field_utils.py` | 32 | `TestGetHl7FieldValue`, `TestSetNestedField`, `TestCopySegmentFieldsInRange`, `TestGetCx4Hd1SegmentCodes`, `TestExtractCx4Hd1`, `TestNormalizeRepetitions` | — |

### `shared_libs/health_check_lib` — 1 files, 7 tests

| Module | Tests | Classes | Docstring |
|---|---:|---|---|
| `test_health_check_server.py` | 7 | `TestTCPHealthCheckServer` | — |

### `shared_libs/hl7_message_processor` — 6 files, 59 tests

| Module | Tests | Classes | Docstring |
|---|---:|---|---|
| `test_converter.py` | 15 | `Adt39DepthTests`, `OruR01DepthTests`, `SubcomponentSeparatorTests`, `Er7EscapeSequenceTests`, `XmlToEr7EscapeSequenceTests`, `XmlToEr7RoundTripTests`, `ObxVariableValueTypeTests` | — |
| `test_escaping.py` | 14 | `DecodeHl7EscapesTests`, `EncodeHl7EscapesTests`, `RoundTripTests` | — |
| `test_processor.py` | 5 | `ProcessEr7WithExplicitStructureTests`, `ProcessEr7WithTriggerAliasTests`, `ProcessEr7FailureTests` | — |
| `test_schema_resolver.py` | 8 | `NormalizeVersionTests`, `SchemaResolverTests` | — |
| `test_trigger_structure_map.py` | 7 | `TriggerStructureMapTests` | — |
| `test_validator.py` | 10 | `ValidateHl7MessageStrictTests`, `ValidateHl7MessageTolerantTests`, `ValidateHl7MessageReportFileCleanupTests` | — |

### `shared_libs/hl7_validation` — 11 files, 89 tests

| Module | Tests | Classes | Docstring |
|---|---:|---|---|
| `test_chemo.py` | 8 | `TestChemoMessages` | — |
| `test_convert_no_data_loss.py` | 10 | `TestConvertEr7ToXmlNoDataLoss` | — |
| `test_convert_write_file.py` | 1 | `TestConvertAndWriteFile` | — |
| `test_mosaiq.py` | 5 | `TestMosaiqMessages` | — |
| `test_paris.py` | 4 | `TestParisValidation` | — |
| `test_phw.py` | 6 | `TestPhwMessages` | — |
| `test_pims.py` | 9 | `TestPPimsValidation` | — |
| `test_standard_validation.py` | 17 | `TestStandardValidation`, `TestXmlValidationWithHl7apy` | — |
| `test_validate.py` | 12 | `TestSchemaPathResolutionAndTriggerParsing`, `TestSchemaValidationErrorRedaction` | — |
| `test_validation_result.py` | 11 | `TestValidationResult`, `TestValidateAndConvert`, `TestConvertWithoutValidation`, `TestConvertEr7ToXmlComprehensive` | — |
| `test_wds.py` | 6 | `TestWdsMessages` | — |

### `shared_libs/message_bus_lib` — 7 files, 98 tests

| Module | Tests | Classes | Docstring |
|---|---:|---|---|
| `test_MessageReceiverClient.py` | 23 | `TestMessageReceiverClient`, `TestReceiveMessagesBatch`, `TestAutoLockRenewerLifecycle`, `TestInvokeWithTraceContext` | — |
| `test_MessageSenderClient.py` | 22 | `TestMessageSenderClient`, `TestSendMessageBatch` | — |
| `test_audit_service_client.py` | 7 | `TestAuditServiceClient` | — |
| `test_message_store_client.py` | 11 | `TestMessageStoreClient` | — |
| `test_metadata_utils.py` | 8 | `TestExtractMetadata`, `TestGetMetadataLogValues`, `TestCorrelationIdForLogger` | — |
| `test_servicebus_client_factory.py` | 13 | `TestServiceBusClientFactoryClose`, `TestCreateMessageStoreClient`, `TestServiceBusClientFactorySenderRecovery` | — |
| `test_subscription_receiver_client.py` | 14 | `TestSubscriptionReceiverClient`, `TestMessageReceiverClient`, `TestAutoLockRenewerLifecycle` | — |

### `shared_libs/metric_sender_lib` — 1 files, 16 tests

| Module | Tests | Classes | Docstring |
|---|---:|---|---|
| `test_metrics_sender.py` | 16 | `TestMetricSender` | — |

### `shared_libs/otel_lib` — 1 files, 24 tests

| Module | Tests | Classes | Docstring |
|---|---:|---|---|
| `test_otel.py` | 24 | `TestConfigureOtel`, `TestGetTracer`, `TestInjectExtractTraceContext`, `TestOtelCorrelationFilter` | — |

### `shared_libs/processor_manager_lib` — 1 files, 11 tests

| Module | Tests | Classes | Docstring |
|---|---:|---|---|
| `test_processor_manager.py` | 11 | `TestProcessorManager`, `TestWrapHandler` | — |

### `shared_libs/transformer_base_lib` — 3 files, 16 tests

| Module | Tests | Classes | Docstring |
|---|---:|---|---|
| `test_app_config.py` | 11 | `TestAppConfig` | — |
| `test_base_transformer.py` | 3 | `TestTransformer`, `TestBaseTransformer` | — |
| `test_message_processor.py` | 2 | `TestMessageProcessor` | — |

### `shared_libs/wrds_service` — 1 files, 26 tests

| Module | Tests | Classes | Docstring |
|---|---:|---|---|
| `test_wrds_service.py` | 26 | `TestBuildGetResultSetEnvelope`, `TestXmlEscape`, `TestParseGetResultSetResponse`, `TestWRDSServiceGetResultSet`, `TestWRDSServiceGetToCode`, `TestWRDSServiceContextManager`, `TestWRDSServiceLocalFixture` | Tests for WRDSService — envelope building, response parsing, and get_to_code filtering. |

### `transformers/hl7_chemo_transformer` — 9 files, 33 tests

| Module | Tests | Classes | Docstring |
|---|---:|---|---|
| `chemo_messages.py` | 0 | — | — |
| `test_application.py` | 5 | `TestChemocareTransformer` | — |
| `test_chemocare_transformer.py` | 3 | `TestChemocareTransformer` | — |
| `test_evn_mapper.py` | 1 | `TestEVNMapper` | — |
| `test_field_utils.py` | 15 | `TestGetHL7FieldValue`, `TestSetNestedField` | — |
| `test_msh_mapper.py` | 2 | `TestMSHMapper` | — |
| `test_nk1_mapper.py` | 1 | `TestNK1Mapper` | — |
| `test_pd1_mapper.py` | 1 | `TestPD1Mapper` | — |
| `test_pid_mapper.py` | 5 | `TestPIDMapper` | — |

### `transformers/hl7_core_reference_transformer` — 4 files, 20 tests

| Module | Tests | Classes | Docstring |
|---|---:|---|---|
| `messages.py` | 0 | — | Sample RISP-style ADT messages used across hl7_core_reference_transformer tests. |
| `test_app_config.py` | 3 | `TestWRDSConfig` | Tests for WRDSConfig — env-var reading and defaults. |
| `test_core_reference_transformer.py` | 4 | `TestCoreReferenceTransformer` | Tests for CoreReferenceTransformer — message-type filter and WRDSService wiring. |
| `test_pid_reference_mapper.py` | 13 | `TestApplyCoreReferenceMapping`, `TestIsEmptyOrHl7Null`, `TestFieldTypeMap`, `TestSetFieldValue` | Tests for pid_reference_mapper — field mapping, empty-value rules, and in-place mutation. |

### `transformers/hl7_lims_wrrs_transformer` — 5 files, 35 tests

| Module | Tests | Classes | Docstring |
|---|---:|---|---|
| `lims_xml.py` | 0 | `Identifier`, `Patient`, `LimsMessage` | Builds LIMS HL7v2 XML (shaped like the real LIMS SOAP payload) and converts it with the real xml_to_er7. |
| `messages.py` | 0 | — | HL7 ER7 fixtures for the LIMS WRRS transformer tests. |
| `test_app_config.py` | 4 | `TestMappingTypeConfig` | — |
| `test_classifier.py` | 21 | `TestClassifier`, `TestCompactedEr7FromRestServer` | — |
| `test_message_processor.py` | 10 | `TestLimsWrrsMessageProcessor` | — |

### `transformers/hl7_phw_transformer` — 9 files, 33 tests

| Module | Tests | Classes | Docstring |
|---|---:|---|---|
| `test_additional_segment_mapper.py` | 5 | `TestAdditionalSegmentMapper` | — |
| `test_app_config.py` | 2 | `TestAppConfig` | — |
| `test_application.py` | 4 | `TestProcessPhwMessage` | — |
| `test_date_of_death_transformer.py` | 2 | `TestDateOfDeathTransformer` | — |
| `test_datetime_transformer.py` | 5 | `TestDatetimeTransformer` | — |
| `test_evn_mapper.py` | 1 | `TestEVNMapper` | — |
| `test_msh_mapper.py` | 5 | `TestMSHMapper` | — |
| `test_phw_transformer.py` | 1 | `TestPhwTransformer` | — |
| `test_pid_mapper.py` | 8 | `TestPIDMapper` | — |

### `transformers/hl7_pims_transformer` — 13 files, 72 tests

| Module | Tests | Classes | Docstring |
|---|---:|---|---|
| `fakes.py` | 0 | `IdentityLookupClient`, `StubLookupClient` | Test doubles for the PIMS transformer tests. |
| `pims_messages.py` | 0 | — | — |
| `test_application.py` | 4 | `TestProcessPimsMessage` | — |
| `test_evn_mapper.py` | 3 | `TestEVNMapper` | — |
| `test_field_utils.py` | 18 | `TestGetHL7FieldValue`, `TestSetNestedField` | — |
| `test_mrg_mapper.py` | 2 | `TestMRGMapper` | — |
| `test_msh_mapper.py` | 4 | `TestMSHMapper` | — |
| `test_pd1_mapper.py` | 2 | `TestPD1Mapper` | — |
| `test_pid_mapper.py` | 13 | `TestPIDMapper`, `TestPIDMapperReferenceLookups` | — |
| `test_pims_transformer.py` | 5 | `TestPimsTransformer` | — |
| `test_pv1_mapper.py` | 2 | `TestPV1Mapper` | — |
| `test_reference_data_client.py` | 13 | `_FakeResponse`, `_FakeHttp`, `TestReferenceDataLookupClient` | — |
| `test_remove_timezone_from_datetime.py` | 6 | `TestRemoveTimezoneFromDatetime` | — |

### `transformers/hl7_wds_transformer` — 2 files, 4 tests

| Module | Tests | Classes | Docstring |
|---|---:|---|---|
| `test_wds_transformer.py` | 4 | `TestWdsTransformer` | — |
| `wds_messages.py` | 0 | — | Sample WDS HL7 ER7 messages for use in tests. |

### `transformers/xml_fhir_proms_transformer` — 4 files, 125 tests

| Module | Tests | Classes | Docstring |
|---|---:|---|---|
| `test_proms_parser.py` | 12 | `TestNormaliseKey`, `TestParseProsXml`, `TestGetMessageType` | — |
| `test_proms_transformer.py` | 96 | `TestMessageTypeRouting`, `TestBundleEnvelope`, `TestMessageHeader`, `TestPatient`, `TestServiceRequest`, `TestPractitionerRole`, `TestPractitioner`, `TestOrganization`, `TestLocation`, `TestProcedure`, `TestAppointment`, `TestEncounter`, `TestParsingRobustness`, `TestPromsFhirTransformer` | Tests for the WPAS -> Promptly FHIR bundle transformer. |
| `test_reference_data.py` | 17 | `TestNhsCertification`, `TestDhaCode`, `TestSourceSystems`, `TestStaticReferenceDataResolver` | Tests for the wiki-derived lookup tables. |
| `wpas_messages.py` | 0 | — | WPAS XML fixtures for the PROMS transformer tests. |


---

*This document reflects `Integration-Hub-Beta` at commit `636a050` (2026-10-27). Part 1 is
descriptive and should be refreshed when the repository changes. Part 2 is a set of
recommendations for the team to adopt, adapt or reject.*
