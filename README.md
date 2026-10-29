# Dust Identity

<!-- API-EVANGELIST-PROVENANCE:BEGIN -->
> ### About this repository
>
> **This is not our API.** This repository is an independent, third-party profile of a company's
> **publicly available** API surface, maintained by [API Evangelist](https://apievangelist.com).
> API Evangelist does not operate, host, resell, or support this company's APIs, and is not
> affiliated with or endorsed by the company unless stated on the profile.
>
> **Where the information came from.** Everything here is assembled from material a member of the
> public can reach with a browser and no credentials — the company's own website, developer portal
> and documentation, the specifications it publishes for public use (OpenAPI, AsyncAPI, JSON Schema,
> `apis.json`, `llms.txt` and similar), its public repositories, and its public status, pricing and
> changelog pages. **Nothing here is obtained by breaching a system, defeating an access control, or
> using credentials of any kind.**
>
> **The rating is an independent assessment.** The Kin Score and Agent Readiness rating are
> independently calculated scores of a company's *public* API artifacts, produced by API Evangelist
> against a published rubric. They are not certifications, endorsements, security assessments, or
> audits, and they score published artifacts — not the quality, safety, or security of the software.
>
> **Corrections, re-scores, and removal are free.** No partnership, contract, or purchase is
> required, and you do not need to justify the request.
>
> - **Something wrong?** Open an issue on this repository, or email
>   [info@apievangelist.com](mailto:info@apievangelist.com).
> - **Published something new?** Ask for a re-score and we will re-run the rating.
> - **Want the listing taken down?** Say so and we will honor it. The profile is reduced to your
>   company name, a factual description, and a link to your own site, and the company is recorded as
>   **unrated** — never scored zero for having asked.
>
> **Response times.** Acknowledgement within **one business day**; removal or restriction within
> **two business days**; corrections and re-scores within **five business days**.
>
> **Not from the company, and here with a question?** You are welcome here — we would rather be the
> front line and point you the right way than have a good report go nowhere. What this repository
> can answer is narrow, though, so it is worth knowing who you are actually looking for:
>
> - **A question about how the API works, an account, billing, or a bug in the service** — that is
>   the company's own support, not us. We profile this API; we do not operate it and cannot see
>   your account.
> - **A bug in an open-source project we only catalog** — file it on that project's own repository.
>   This has happened with a real and correct bug report that reached us instead of the people who
>   could fix it, which helped nobody.
> - **Anything about this listing itself** — the description, the tags, the rating, a missing or
>   wrong artifact — is ours. Open an issue here.
> - **Not sure, or something general about API Evangelist or APIs.io** — open an issue on the
>   [APIs.io Inbox](https://github.com/api-search/inbox) and we will route it.
>
> **This repository contains no software, and we will never ask you to download anything.** There is
> no build, release, installer, or binary here — only text and machine-readable API descriptions, so
> there is nothing here that can be "corrupt" or need "repairing". Any issue, comment, or email
> claiming otherwise and offering a download link is not from us and is hostile. Do not follow the
> link; it is a lure. Report it to GitHub and, if you like, tell us at
> [info@apievangelist.com](mailto:info@apievangelist.com) so we can take it down.
>
> **On a security or compliance team?** Email
> [info@apievangelist.com](mailto:info@apievangelist.com) with *security* in the subject line and
> you will get a person, not a form. We will tell you exactly which public URLs this profile was
> built from so your team can see the same surface we did, and we will take the listing down on
> request while you work through it.
>
> Full detail: **[Where this data comes from](https://apievangelist.com/about/where-our-data-comes-from)**
<!-- API-EVANGELIST-PROVENANCE:END -->

DUST Identity (Newton, Massachusetts; founded 2018 as an MIT spinout) binds physical objects to trusted digital records using the Diamond Unclonable Security Tag — an applied coating of engineered nano-diamonds in a polymer matrix that forms an unclonable optical fingerprint. The DUST Platform (DICE) tracks each marked item as a "Thread": a digital record carrying typed fields, files, physical identifiers (DUST, QR, barcode, Data Matrix, NFC), an immutable event history, and Fabric lineage across slices, shipments and organization boundaries. It is used for aerospace and defense raw-material traceability, secure electronics, certified-pre-owned authentication, return-fraud detection, security labeling and high-value asset identity. The company publishes a public OpenAPI 3.1 contract for the DUST API (apid) covering 177 operations, a Starlight documentation site, llms.txt, and two provider-authored agent skills.

**Areas:** Authentication · Identity · Supply Chain · Traceability · Provenance · Anti-Counterfeiting · Asset Tracking · Aerospace and Defense · Manufacturing · Security

## Dust Identity on APIs.io

Everything profiled for Dust Identity is published on APIs.io rather than in this repository. Start at the provider page — **[apis.io/providers/dust-identity/](https://apis.io/providers/dust-identity/)** — or go straight to one of the APIs or properties below.

### APIs (24)

- [DUST AuthD](https://apis.io/apis/dust-identity/dust-authd/) — The DUST account service — organizations, users, sessions, Service Accounts and OpenID Connect. Issues the short-lived JWTs the DUST API consumes, via an…
- [Dust Identity Auth API](https://apis.io/apis/dust-identity/dust-identity-auth-api/) — Authentication and authorization
- [Dust Identity Bundles API](https://apis.io/apis/dust-identity/dust-identity-bundles-api/) — Unordered teams of threads
- [Dust Identity Certificate Forms API](https://apis.io/apis/dust-identity/dust-identity-certificate-forms-api/) — Create and manage certificate form designs
- [Dust Identity Certificates API](https://apis.io/apis/dust-identity/dust-identity-certificates-api/) — Generate, list, and void certificates
- [Dust Identity Connections API](https://apis.io/apis/dust-identity/dust-identity-connections-api/) — Connections (team links) between Teams; they gate sharing and shipments
- [Dust Identity Events API](https://apis.io/apis/dust-identity/dust-identity-events-api/) — Work with system events
- [Dust Identity Fabric API](https://apis.io/apis/dust-identity/dust-identity-fabric-api/) — The Fabric API from Dust Identity — 25 operation(s) for fabric.
- [Dust Identity Files API](https://apis.io/apis/dust-identity/dust-identity-files-api/) — Upload and manage files
- [Dust Identity Metrics API](https://apis.io/apis/dust-identity/dust-identity-metrics-api/) — Organization and team metrics.
- [Dust Identity Notifications API](https://apis.io/apis/dust-identity/dust-identity-notifications-api/) — Manage user notifications.
- [Dust Identity Org Admin API](https://apis.io/apis/dust-identity/dust-identity-org-admin-api/) — Org-admin management of teams and memberships
- [Dust Identity Organizations API](https://apis.io/apis/dust-identity/dust-identity-organizations-api/) — Manage organizations and their members.
- [Dust Identity Sharing API](https://apis.io/apis/dust-identity/dust-identity-sharing-api/) — The Sharing API from Dust Identity — 4 operation(s) for sharing.
- [Dust Identity Slices API](https://apis.io/apis/dust-identity/dust-identity-slices-api/) — The Slices API from Dust Identity — 3 operation(s) for slices.
- [Dust Identity System API](https://apis.io/apis/dust-identity/dust-identity-system-api/) — Health and status checks
- [Dust Identity Tags API](https://apis.io/apis/dust-identity/dust-identity-tags-api/) — Manage identifiers and their metadata
- [Dust Identity Teams API](https://apis.io/apis/dust-identity/dust-identity-teams-api/) — Manage user teams and permissions
- [Dust Identity Templates API](https://apis.io/apis/dust-identity/dust-identity-templates-api/) — Manage thread templates
- [Dust Identity Thread Links API](https://apis.io/apis/dust-identity/dust-identity-thread-links-api/) — Link threads with relations
- [Dust Identity Thread Relations API](https://apis.io/apis/dust-identity/dust-identity-thread-relations-api/) — The ways threads can be linked
- [Dust Identity Threads API](https://apis.io/apis/dust-identity/dust-identity-threads-api/) — Create and manage threads
- [Dust Identity Transfers API](https://apis.io/apis/dust-identity/dust-identity-transfers-api/) — Transfer ownership of threads across organizations
- [Dust Identity Users API](https://apis.io/apis/dust-identity/dust-identity-users-api/) — Manage user accounts and profiles

### Published properties

Each property below is a page on APIs.io carrying the machine-readable artifact and what it says.

#### AsyncAPI (1)

Event-driven and webhook contracts.

- [Dust Identity Events](https://apis.io/asyncapis/dust-identity/dust-identity-events/)

#### MCP servers (1)

Model Context Protocol servers that expose the API to agents.

- [dust-identity-mcp.yml](https://apis.io/servers/dust-identity/dust-identity-mcp-yml/)

#### Agentic access (1)

Which operations an agent can call, which act, and which need a human in the loop.

- [Dust Identity Agentic Access](https://apis.io/agentic-access/dust-identity/dust-identity-agentic-access/)

#### Collections (1)

Postman and Open Collection request collections.

- [DUST API](https://apis.io/collections/dust-identity/open-dust-identity-apid/)

#### Overlays (1)

OpenAPI overlays, including conversational phrasing for each operation.

- [Dust Identity Apid Overlay](https://apis.io/overlays/dust-identity/dust-identity-apid-overlay/)

#### Security (2)

Authentication, trust centre, and vulnerability disclosure.

- [Dust Identity Authentication](https://apis.io/security/dust-identity/dust-identity-authentication/)
- [Dust Identity Domain Security](https://apis.io/security/dust-identity/dust-identity-domain-security/)

#### OAuth scopes (1)

The permissions the API asks for.

- [Dust Identity Scopes](https://apis.io/scopes/dust-identity/dust-identity-scopes/)

#### Plans and pricing (1)

Published plans, tiers and prices.

- [Dust Identity Plans Pricing](https://apis.io/plans/dust-identity/dust-identity-plans-pricing/)

#### Rate limits (1)

Published request limits.

- [Dust Identity Rate Limits](https://apis.io/rate-limits/dust-identity/dust-identity-rate-limits/)

## Where to find this API

Links to the company's own pages. This repository publishes no copy of their specifications — the canonical document is theirs and lives at the URLs below.

- [Website](https://www.dustidentity.com/)
- [Documentation](https://docs.dustid.io/)
- [Getting started](https://docs.dustid.io/api/quickstart/)
- [API reference](https://docs.dustid.io/reference/api/)
- [Sign up](https://dice.dustid.io/login)
- [Support](https://www.dustidentity.com/contact)
- [Blog](https://www.dustidentity.com/blog)
- [Terms of service](https://www.dustidentity.com/legal-pages/terms-of-service)
- [Privacy policy](https://www.dustidentity.com/legal-pages/privacy-policy)
- [GitHub](https://github.com/dustid)

## API Evangelist rating

**Kin Score 45.8** — developing · rubric 0.25.0 · scored 2026-10-23

| Facet | Score |
|---|---|
| Discoverability | 73.2 |
| Contract quality | 59.1 |
| Contract governance | 4.5 |
| Operational transparency | 10.5 |
| Developer ergonomics | 71.4 |
| Access clarity | 21.1 |

**Agent readiness 44.3** — agent-ready (gated down from agent-native)

| Agent-readiness dimension | Result |
|---|---|
| Spec Presence | yes |
| Agentic Access | derived (partial credit) |
| Reversibility Documented | documented (partial credit) |
| MCP Server | — |
| Auth Clarity | served (partial credit) |
| Idempotency | — |
| Error Semantics | verified |
| OpenAPI Examples | documented (partial credit) |
| Rate Limit Signal | documented (partial credit) |
| Event Surface Described | yes |
| Agent Skills | yes |
| Well Known Catalog | — |
| Consent Identity | — |
| Agent Card | — |
| Dry Run Mode | — |
| Delegated Identity | served (partial credit) |
| Protected Resource Metadata | — |
| Dynamic Client Registration | — |
| Agentic Commerce | — |
| Pagination Ratio | — |
| Operation Distinctiveness | distinct |

**Regulatory alignment 34.3** — Horizontal (data, software, accessibility, platform)

The rubric is published at [github.com/api-evangelist/kin-score](https://github.com/api-evangelist/kin-score). The per-check breakdown behind these numbers — which checks were earned, partial or missed — along with score history and how this provider compares to its peers, is available at [apis.io/providers/dust-identity/](https://apis.io/providers/dust-identity/).

## How this profile is made

API Evangelist profiles a company's public API surface the way a developer would find it: the website, developer portal and documentation, the specifications it publishes, its public repositories, and its pricing, status and changelog pages. From that it derives the properties above — contracts split per API, workflows, schemas, collections, governance rules, plans and rate limits — and scores them against the published [Kin Score rubric](https://github.com/api-evangelist/kin-score).

The artifacts are published on [APIs.io](https://apis.io), not in this repository. This repository is the public record for Dust Identity: the place to say something is wrong, missing or out of date, and to ask for a re-score or a removal.

## Open an issue

- **Something about this listing** — a wrong description, a missing API, a stale artifact or a question about the rating: [open an issue on this repository](https://github.com/api-evangelist/dust-identity/issues/new).
- **A new API, or one that is not in the catalog yet** — [submit it on APIs.io](https://apis.io/add/).
- **Anything else** — [the APIs.io inbox](https://github.com/api-search/inbox/issues/new) or [info@apievangelist.com](mailto:info@apievangelist.com).
