# Tally Aloft — Design Docs

These documents are the source of truth for what Tally Aloft is and how it's built. They describe the current agreed design, and they change as implementation settles questions or improves on the original plan. An approved change that hasn't been built yet lives in its epic's spec until the story that builds it updates these documents. When the code and a document disagree, one of them is wrong, and the disagreement is a problem to raise, not a choice to make silently.

Read this index first, then read only the documents your work needs. The "Read when" column says which those are. If your work changes scope, come back here and read what the new scope needs.

## Documents

| Document | Covers | Read when |
| --- | --- | --- |
| [Product overview](product/overview.md) | Purpose, target pilots, positioning, the feature list with each feature's status, business model, product risks and open product questions | You need to know what a feature must do for pilots, whether it's in v1 or decided, or why the product makes a choice. Most feature work needs its row in the feature tables. |
| [Market analysis](product/market-analysis.md) | The October 2026 research behind the product: pilot population, competitors and their complaints, the feature ranking, pricing math, and regulatory and privacy risks, with sources | Background only. Read it for product strategy, positioning, pricing or competitor questions, or to check the source of a claim in the product overview. Not needed for implementation. |
| [Glossary](glossary.md) | Aviation and regulatory terms: REED, LADD, PIA, AC 120-78B, the FAA Parts and others | You meet a term you don't know. |
| [Technical overview](technical/overview.md) | System diagram, hosts and domains, and how the design keeps each product promise (no lost hours, tamper-evident signatures, identical totals, export, disaster recovery) | Any technical work, including infrastructure (`tally-aloft-infra`). It's short. |
| [Client contract](technical/client-contract.md) | The rules the API and every app follow: schema and operation IDs, breaking changes, client-version headers, idempotency keys, version conflicts, jobs, CORS, tracing, error shape and rate limits | Any work on an API endpoint, on the generated client, or on app code that calls the API. Backend and client work both need it. |
| [Backend tech stack](technical/backend/tech-stack.md) | Backend libraries and vendors and why each was chosen; authentication, storage, digitization, PDFs, subscriptions, push and email; observability (Logfire) and what stays out of traces; signatures; data ownership, export and deletion; AWS infrastructure; operations; developer tooling; the deferred offline-sync plan | Choosing or using a backend library or vendor, or working on logging and tracing, signatures, account deletion, export, retention, deploys or migrations. Infrastructure work needs its AWS sections. |
| [Backend structure](technical/backend/structure.md) | Hexagonal layers, domains and their levels, authentication flow, background jobs, where each client-contract rule lives in the code, idempotency, a "where things go" table, testing, Import Linter contracts and conventions | Writing or reviewing any backend code. Start with "Where things go". |
| [Client tech stack](technical/clients/tech-stack.md) | Libraries for the web app, the mobile app and the marketing site, and why each was chosen; error monitoring (Sentry); hosting on S3 and CloudFront; the site's required pages; cross-cutting app behavior (environments, unsent drafts, entitlements, PDFs); the deferred offline plan | Choosing or using a client library, or working on error monitoring, environments, the marketing site's pages, drafts or entitlements. Infrastructure work needs its hosting rows. |
| [Client structure](technical/clients/structure.md) | The monorepo and its rules, shared packages, Feature-Sliced Design layers and slices, data flow, auth, routing, configuration, version skew, UI kits, the marketing site's layout, a "where things go" table, enforced boundaries, tests and CI | Writing or reviewing any web or mobile code. Start with "Where things go". Marketing-site work needs only "The marketing site". |

### Docs in the code repositories

Some documentation belongs next to the code it describes, such as the list of shared UI controls that the [client structure](technical/clients/structure.md#ui-kits-and-visual-consistency) keeps in `tally-aloft-clients/docs/`. Each such document is listed here so it can be found from this index.

| Document | Covers | Read when |
| --- | --- | --- |
| None yet | | |

## Maintaining these docs

Anyone who changes a document, human or agent, follows these rules.

- **Describe the current design, not its history.** When the design changes, change the document so it reads as if it had always said the new thing, and let git keep the old version. Keep the reason for a choice next to it in a sentence ("Chosen over X, because …"), so nobody reopens a settled question without knowing why it was settled.
- **Keep the status honest.** Mark what is decided, what is proposed and what is open, as the existing documents do with *decided*, statuses and "Open questions". When a question is settled, move it out of the open questions and into the text.
- **One home per topic.** Each topic has one document that owns it, and other documents link to it rather than restate it. When you add something, put it in the document whose "Covers" column fits, and change that column if the document now covers more.
- **Keep this index complete.** In the same change that adds, renames, splits, merges or removes a document, update its row here and fix every link to it (search the whole workspace for the old file name, including `.claude/` and the code repositories). A new document gets a row with all three columns, and its "Read when" must let a reader tell, from the task alone, whether they need it.
- **Split by when a document is read.** Split a document when one part is often needed without the rest, or when it grows past about 30 KB. Don't split a topic into pieces that always have to be read together.
- **Keep the header line.** Each document starts with a one-line header of its area and topic, ending in the month and year of its last substantive change, such as `Backend · Structure · October 2026`. Update the date when you change the document's substance.
- **Docs repo or code repo.** Design that spans repositories or explains why the system works as it does belongs here. A reference tied to one repository's code, such as a list of its controls, can live in that repository's `docs/`, with a row in the table above.
