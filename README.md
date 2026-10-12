# Socle Twin architecture

> How Socle Twin is designed to build, update and serve an ontology-first digital twin of a small company, and where Claude does the work.

*Version 1, October 2026. The same content as [getsocle.com/architecture](https://getsocle.com/architecture), with the diagrams as files in [`docs/`](docs/).*

## Contents

1. [Summary](#1-summary)
2. [Data path: from source system to agent](#2-data-path-from-source-system-to-agent)
3. [Ontology-first model](#3-ontology-first-model)
4. [Entity resolution](#4-entity-resolution)
5. [Provenance per field, versions validated by people](#5-provenance-per-field-versions-validated-by-people)
6. [Knowledge base](#6-knowledge-base)
7. [MCP gateway, multi-tenant](#7-mcp-gateway-multi-tenant)
8. [Evals: replayed on every new model](#8-evals-replayed-on-every-new-model)
9. [Where Claude sits at each step](#9-where-claude-sits-at-each-step)
10. [Today and 2027](#10-today-and-2027)
11. [Open questions we are working on](#11-open-questions-we-are-working-on)

## 1. Summary

Socle Twin is an ontology-first digital twin of a small or mid-sized company, designed in two parts that point at each other. A typed graph holds what the company is: clients, projects, invoices, people, documents and the links between them. A structured knowledge base holds how it works: rules, procedures, source documents and decisions, attached to the graph objects they govern.

Ingestion connectors are designed to read the tools the company already runs, with read-only credentials. Claude is meant to do the reading, typing and matching, and people will validate what matters. Any agent, Claude or another, will reach the twin through one multi-tenant MCP gateway. The gateway is specified as read-only by default: a write exists only as a proposed action that a person approves.

The company's tools stay the system of record. Each value is designed to carry its provenance, each version to be dated and validated by a person, and each answer to cite its sources or state what is missing.

Nothing described here runs end to end at a client today. What exists is a set of parts built by hand with Claude, listed in section 10.

- **Closed typology.** A finite, versioned set of object and link types per company. Extraction should emit only what the schema allows.
- **Read-only ingestion.** Ingestion connectors will hold read credentials only. Any write will take a separate path and start after a person approves.
- **Provenance per field.** Source, method, validity dates and validation status are meant to travel with each value.
- **Reversible.** Merges, versions and model switches should keep their lineage, so each can be audited or undone.

### How to read the status chips

Sections 2 to 9 each carry a status chip. The plug-and-play platform we are aiming for appears only in section 10, in the future tense.

- **Prototyped.** Done by hand with Claude on a client engagement, a prototype or synthetic data. It does not run as a system yet.
- **In design.** Specified. Not built.

## 2. Data path: from source system to agent

**Status: Prototyped.** A read-only adapter feeding a canonical model ran on one prototype. The rest of the path is a design.

The path has eight stages and runs in one direction. Ingestion is designed to be read-only, so no stage after the connectors can write to a source system. Any write goes through a separate, approval-gated path (section 7). Each stage is meant to be replayable from the one before it. When a source changes, the design calls for a new snapshot and a new graph version (section 5).

![Diagram of the data path: company tools, read-only connectors, raw snapshots, Claude extraction, typed canonical graph, knowledge base, MCP gateway, agents and clients.](docs/data-path.svg)

*Figure 1. The designed data path. Blue marks steps where a model runs, green marks people, amber marks data stores.*

| Stage | Design | Invariant |
|---|---|---|
| 1. Company tools | ERP, CRM, mail, documents and spreadsheets remain the system of record. | The twin build does not modify a source system. |
| 2. Read-only connectors | One adapter per source type, behind a replaceable interface, with a conversion contract per type. Credentials are read-only. | No business object mirrors a single connector row. |
| 3. Raw snapshots | Each pull is stored as captured, with a content hash and a capture time. | Snapshots are immutable, so later stages can rerun without calling the source. |
| 4. Extraction | Claude turns snapshots into candidate objects, links and properties that fit the closed typology. | Output is validated against the schema, and each candidate points to the span it came from. Malformed output is rejected and logged. |
| 5. Typed canonical graph | Entity resolution (section 4) merges candidates into canonical objects, stored as dated versions. | Canonical ids are immutable. Source references stay attached as system and external-id pairs. |
| 6. Knowledge base | Rules, procedures, documents and decisions attach to the objects they govern (section 6). | An item without a valid attachment point is held back from publication. |
| 7. MCP gateway | Tools are generated from each tenant's schema and served per tenant (section 7). | Read-only by default. Writes follow the separate approval path. |
| 8. Agents and clients | Claude, other MCP clients, a staff cockpit. | Any of them can be swapped without changing the twin. |

## 3. Ontology-first model

**Status: Prototyped.** A closed typology was used on one client's documents and in a legal-corpus prototype. The per-company design below is the target.

The ontology comes before the data. For each company we will write a closed typology: a finite list of object types and link types, each with typed properties, identity rules and cardinalities. Extraction will emit only instances of declared types. If the schema cannot place something, it will go to a quarantine list for a person to look at, and no new type will appear on its own. A change to the typology will be a versioned schema change that a person reviews and diffs like code.

### What a type declaration carries

- **Identity.** The fields that identify an instance, and how it is matched across sources.
- **Typed properties.** Scalar types, enumerations, money with currency, dates, references, computed properties.
- **Links.** Named, directed, typed at both ends, with a cardinality.
- **Source mappings.** How each connected system's records map onto the type, with the mapping version.
- **Constraints.** Required fields, allowed ranges and cross-field checks, run at extraction time.
- **Version.** Every change is a new schema version, and data records the version it conforms to.

### Why closed rather than open

- A reviewer can read a company's whole vocabulary in one sitting and diff it between versions. An open-ended vocabulary cannot be reviewed that way.
- With a finite set of types and links, each question will be a parameterised traversal over known shapes. No model will write free-form queries, so the query surface can be listed, tested and permissioned.
- Extraction output will be checked against the schema, so an invented type or field will be rejected before it reaches the graph.
- The gateway will generate its tools from the schema (section 7). The tool list will follow the ontology, and a schema change will change the tools.

The typology can still grow, through reviewed versions only.

## 4. Entity resolution

**Status: Prototyped.** One matching pass ran with Claude on synthetic exports. Scoring, routing and the review queue are specified and unbuilt.

One client can appear as a line in the CRM, an invoice header in the ERP, a short name in a spreadsheet and a sender in a mail thread. Resolution is designed to turn those mentions into one canonical object, with a score, a reason and a way back.

![Diagram of the entity resolution pipeline: blocking, pairwise scoring, routing by score band, Claude arbitration, human review queue, merge with reversible lineage.](docs/entity-resolution.svg)

*Figure 2. Entity resolution as designed. Strong matches link, ambiguous pairs go to Claude, and uncertain verdicts go to a person.*

1. **Exact rules.** Shared identifiers (registration numbers, account ids, verified email addresses) will link mentions deterministically, with no model involved.
2. **Blocking.** Cheap keys (normalised name tokens, identifier prefixes, phonetic and n-gram keys, shared contacts) will group mentions into blocks, and only mentions in the same block will be compared.
3. **Pairwise scoring.** Within a block, each pair will get field-level similarity features chosen per type (names, addresses, amounts, dates, shared relations). They will combine into a calibrated score, with the per-field evidence kept beside it.
4. **Routing by score zone.** The calibrated pairwise score will route each pair into one of three zones: an auto-merge zone, an auto-reject zone, and an ambiguous zone that goes to Claude.
5. **Claude arbitration.** Claude will receive the two mentions, the field evidence and the surrounding graph context, and return a verdict (same, different or unsure) with a written justification citing that evidence. Sonnet is planned for the volume, Opus for contested or high-impact cases.
6. **Verdicts and the human queue.** A "same" verdict that the score supports will merge, and a "different" one will take the reject exit. An "unsure" verdict, or any verdict that goes against the score, will go to a person with the rationale already written. The zone boundaries will apply to the calibrated score from step 3, validated against reviewed pairs, and never to a confidence the model reports about itself. Until enough reviewed pairs exist to calibrate it, every Claude verdict will go to the queue. Each accept, reject or split will be stored as a labelled example.
7. **Merge with reversible lineage.** A merge will be recorded as an event and will overwrite nothing. The canonical object will keep its member mentions and their source references, and the log will record score, evidence, rationale, model version and reviewer. An unmerge will replay from that log.

### Safeguards

- Rejected pairs will go into a cannot-link memory and will never be proposed again. Every merge will be checked against that memory, so a chain of plausible links cannot join two things that a person kept apart.
- A match across languages, regions or legal entities will never be auto-confirmed. It will stay marked "to confirm" until a person validates it.

What ran so far: one matching pass on synthetic exports, with the same client written several ways across an ERP export, a timesheet and a tracking spreadsheet. Claude resolved the variants to one entity, gave a reason and flagged a mismatch in amounts between the sources. We have no accuracy figures yet. Measuring them per client is the job of the eval sets in section 8.

## 5. Provenance per field, versions validated by people

**Status: Prototyped.** Prototypes keep source references and frozen traces. The full per-field model, with both time axes, is specified and not built.

Each value will be stored as a fact with its own history instead of a bare column value.

| Field | Content |
|---|---|
| `value` | the value itself, typed by the schema |
| `source` | system, external id, snapshot id and the span: page, cell or character range |
| `method` | a deterministic rule, or the model and its version |
| `status` | candidate, validated or rejected |
| `valid time` | from and to: when it was true for the company |
| `transaction time` | when the twin recorded it |
| `validation` | who validated it, and when |

### A bitemporal model

The store will be bitemporal. Valid time says when something was true for the company, and transaction time says when the twin recorded it. Two questions then have exact answers: what was true on 3 March, and what did we believe on 3 March. A correction will overwrite nothing. It will add a version with a later transaction time, so earlier answers can be reproduced.

### Validated versions

A fact will start as a candidate and become established once a person validates it. Depending on its role, an agent will see validated facts only, or also labelled candidates. When two sources disagree, both facts will be kept with their provenance.

### Citations

The gateway will return each value with its provenance chain: version, source reference, snapshot, span. Before an answer ships, every cited span will be checked mechanically against the source text: the quoted passage must occur at the stated location and match its stored hash. An answer that cannot be traced will say so. Missing information is a result of its own ("missing fact", "interpretation required"), so the system can abstain instead of filling the gap.

## 6. Knowledge base

**Status: Prototyped.** A legal-corpus prototype exercised verbatim documents and typed rules with four evaluation states. Attachment to graph objects is in design.

The graph says what exists. The knowledge base is meant to say how the company works, and it will attach to graph objects so that one lookup returns both.

| Item | What it is | Attaches to |
|---|---|---|
| Rules | Typed, dated and sourced. Each rule is meant to evaluate, per object, to one of four states: satisfied, not satisfied, missing fact, interpretation required. | An object type, evaluated per object. |
| Procedures | Ordered steps with owners and preconditions. | The process or object type they govern. |
| Documents | Kept verbatim. Passages are anchored by document, page, version and content hash, so a quotation can be checked. | The objects they describe. |
| Decisions | What was decided, by whom, when and why, and which earlier decision it replaces. | The object concerned. |

- Item types are specified as closed on function and open on subject: the type says what role an item plays (rule, procedure, source document, decision), and the subject stays free.
- Validated knowledge will live as human-readable, versioned text files with structured metadata, and those files are the source of truth. Vector indexes, search indexes and graph views will derive from them and can be rebuilt at any time. Retrieval is only an index over the knowledge base.
- For each question, the twin will assemble a bounded context pack: the relevant objects, their link neighbourhood, the attached knowledge and the provenance, within a token budget.
- Facts extracted from a document will stay candidates until a person validates them.

## 7. MCP gateway, multi-tenant

**Status: In design.** A read-only MCP server with schema-generated tools and a human-approval gate on outbound messages was prototyped on the legal corpus. Known gap: an agent with its own mail connector could bypass the send guard. Closing it is a design requirement here.

One gateway is specified to expose each tenant's twin to any MCP-compatible agent. As the only way in, it is where isolation, permissions, approvals and audit will be enforced.

![Diagram of the multi-tenant MCP gateway: tenant binding, tools from the schema, read path to one tenant store, write path through human approval to source systems, audit log.](docs/mcp-gateway.svg)

*Figure 3. The gateway as designed. Reads reach one tenant's store. Writes are proposed actions that a person approves.*

### Transport and authentication

The gateway is designed as a remote MCP server over streamable HTTP, with OAuth 2.1 authorization per tenant. The tenant id will be taken from the access token alone and validated on every request. A session will not be an authorization boundary.

### Read path

- Tools will be generated from the tenant's schema: get an object, search, traverse links, query attached knowledge, describe a type. A role will filter the tools and fields a session sees.
- Each result will carry its provenance and validation status.

### Write path

Writes are specified as proposed actions, in seven steps.

1. **Describe.** The agent reads the action's contract: inputs, preconditions, effects.
2. **Simulate.** The gateway computes a dry-run diff against current data.
3. **Plan.** A plan is created, bound to the requesting actor and to the data version it was computed on.
4. **Approve.** A named person approves or rejects. Nothing runs before that.
5. **Re-check.** At execution time the gateway re-validates its invariants. If the data changed after approval, the action is refused and sent back for review.
6. **Execute once.** Execution is idempotent and goes through a separate action connector. Only that connector holds write credentials, and only for approved actions. No agent will hold another route to a source system.
7. **Receipt and audit.** The outcome is recorded, and a refusal carries a coded reason.

### Indirect prompt injection

Mail and documents that the twin ingests may contain instructions aimed at a model, so the design aims to limit the damage. Ingested content will be treated as data and quoted as material to analyse. The tools a model sees while it extracts or answers will be read-only and scoped to one tenant, and that model will have no outbound channel. The gateway, not the model, will compute the diff of any proposed write, and a person will approve it. A model that reads tool output stays influenceable, so injected text can still bias an answer or a proposed write. That residual risk is why every write needs human approval, and it is listed in the open questions (section 11).

### Per-tenant isolation

Each tenant will get its own graph and knowledge store. No query path will span tenants: the tool layer is designed to receive a tenant-scoped handle, so no shared connection exists. Source-system credentials will be held per tenant, with read and write scopes kept apart. Every eval set is planned to include cross-tenant probes (section 8).

### Audit log

The log will be append-only and kept per tenant. It will record each read, search, proposal, approval, execution and refusal with the actor, tool, arguments, returned facts and outcome, so a team member can reconstruct why an agent said or did something.

## 8. Evals: replayed on every new model

**Status: In design.** Measured test sets were used on a prototype to compare retrieval variants. Per-client eval sets and replay on new models are not built.

Each client twin is meant to have its own eval set, since correct means correct for that company's data and rules. Before a twin's model assignments change, the set will be replayed on the candidate model and compared with the pinned one.

### What an eval set will contain

- Questions across objects and sources, each with the expected answer and the citations it must carry, validated by a person.
- Entity-resolution cases (pairs known to be the same, to differ, or to be ambiguous) and extraction checks against a reviewed sample, per type.
- Abstention cases: questions that must return "missing fact" or "interpretation required" instead of an answer.
- Safety cases: write attempts that must stay proposals, and cross-tenant probes that must return nothing.

### The replay

1. When a new Claude model is released, each tenant's eval set will run on it for every pipeline step, with the current model as baseline.
2. Results will be compared per category against gates. A regression in span existence, claim support, abstention or safety will block the switch, whatever the other gains. Cost and latency will be recorded.
3. If the gates pass, the tenant's model pin for that step will move. The previous pin will stay available for rollback, and the change will go into the audit log.

Two citation checks are planned. The deterministic one confirms that each cited span exists verbatim in the source. The model-graded one judges whether that span supports the claim, against a rubric, and the grader will be checked against samples a person has graded. Safety cases will be deterministic code.

## 9. Where Claude sits at each step

**Status: In design.** This allocation is a design position and has not been measured in production. Each client's eval results will tune it.

| Step | Planned role | Model | Why |
|---|---|---|---|
| Ontology design | Proposes the typology from sample exports, documents and an interview, and lists the open questions. A person reviews every type. | Opus | Judgement-heavy, done once per company. |
| Extraction | Turns snapshots and documents into schema-validated candidates with source pointers. | Sonnet | High volume, bounded task, clear contract. |
| Entity resolution, routine | Arbitrates ambiguous pairs with a written justification. | Sonnet | Volume, with the human queue behind it. |
| Entity resolution, hard cases | Handles contested clusters, conflicting sources and escalations from the routine pass. | Opus | Few cases, and a wrong merge is costly. |
| Classification and routing | Document type, language, duplicate detection, routing into queues. | Haiku | Mechanical, latency-sensitive, cheap. |
| Knowledge authoring | Drafts rules and procedures from documents, with anchored passages. A person validates. | Sonnet | Bounded drafting with verification built in. |

### Tooling and technique

| Item | What it does | Why |
|---|---|---|
| Claude Code and subagents | We use them now to write connectors, mappings, tests and each client prototype. A person reviews every change. | It is the tooling we already build with every day. |
| Prompt caching | The stable company-model prefix (schema, glossary, rule index) will go first and be cached. The changing part, a batch or a question, will follow. | That prefix is large and identical across a tenant's calls, within the cache window. |

The cache is designed around a stable serialisation of the schema, with the breakpoint right after it. Tools sit ahead of the system prompt in the cached prefix, so each tenant's tool list will be deterministic and byte-stable. A schema or tool change will invalidate the cache on purpose. A cached prefix is reused only within the cache lifetime, so a tenant with little traffic pays the cache write more often.

## 10. Today and 2027

### Today: Done by hand, with Claude

Since spring 2026 the two founders have built parts of this by hand with Claude, across three client engagements (property management, joinery, property development).

The pieces so far: a read-only adapter feeding a canonical model on one prototype, a typed extraction over one client's documents, and an audit under way for a third client. A separate legal-corpus prototype exercised the knowledge base and the read-only MCP pattern.

### 2027: Plug and play

In 2027 we aim to offer a plug-and-play platform. A company will connect its tools with read-only credentials. Claude agents will propose the ontology, build the twin and keep it current as sources change, and a person will validate what matters.

One gateway will serve every tenant, and evals will replay automatically on each new model. The founders will stop building each twin by hand and will review the exceptions.

Four things will stay constant: ingestion with read-only credentials, a closed typology, provenance on every field, and a person's approval before any write.

## 11. Open questions we are working on

- How to show that tenant isolation holds, beyond testing it.
- Where deterministic scoring should hand over to model arbitration in entity resolution, and how to evaluate that on messy exports.
- The residual prompt-injection risk: a model that reads ingested text can still be biased, so how far an answer or a proposed write can be trusted.
- Whether shared context or cache can leak across tenants, and how to keep them isolated.
- Cache design for large company models whose schema changes, and its economics at low traffic.

---

This repository is documentation only. It holds no client ontology, prompts, code or client data. See [NOTICE.md](NOTICE.md).

Socle Twin is a project of Derivatives Academy (BCE 0804.022.310), Lasne, Belgium. Contact: info@getsocle.com
