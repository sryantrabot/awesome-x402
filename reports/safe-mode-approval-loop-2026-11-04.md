# Safe Mode approval loop — one self-check send, end to end (2026-11-02 → 2026-11-04)

A short honest run report of our agent driving Nitrosend's held-campaign approval flow on our own
account, with the Mail Action Protocol description checked by the mailschema package. This is a
description of our run, not of anyone's product.

- **Environment and identity.** Debian 12 aarch64 container VM (rust:1-slim-derived, no Node) plus a
  separate browser machine for the account's web session. Everything ran under the org's own
  automation identity (sryantrabot, disclosed as an automated agent wherever it speaks). The Nitrosend
  account is a free self-serve signup on our own address; no card, no spend of any kind.
- **Exact setup commands (2026-11-02).** All API calls went over TLS with `openssl s_client` from the
  VM (no SDK, no CLI installed): `POST /v1/login` (owner JWT, 200); `POST /v1/my/api_keys` (201, key
  stored 0600, never echoed); `GET /v1/my/brands` (200, brand sid read); `POST /v1/my/brands/<sid>/prepare_sending`
  `{"subdomain":"sryantrabot"}` (200 — sender `hello@sryantrabot.nitrosend.net` active);
  `POST /v1/my/messages` (201, one email to proof@dogfood.nitrosend.net, id 5299988, `status: "sent"`,
  `sent_at 2026-11-02T11:35:28.946Z` — the required pre-existing send receipt);
  `PATCH /v1/my/account {"safe_mode_enabled":true}` (403 with the API key, 200 with the owner JWT —
  Safe Mode only accepts the account owner's credential; verified `safe_mode_enabled:true` by
  `GET /v1/my/account`).
- **Campaign composition (2026-11-04, 00:5x–01:0xZ).** Same transport: `POST /v1/my/contacts` (own
  address, subscribed), `POST /v1/my/lists` (one contact), `POST /v1/my/campaigns` (email),
  `PATCH /v1/my/campaigns/16035` with template_attributes (plain subject "Safe Mode approval loop
  check", one-paragraph HTML body, no merge tags, from our own sender) and trigger_attributes
  (audience = that one-contact list), `POST /v1/my/campaigns/16035/send` with `deliver_at` 20 minutes
  out and the expected `campaign_updated_at` + `template_if_version`. Result: held
  `awaiting_approval`, and the "Approve this send" email landed in the account mailbox 01:05:59Z.
- **The MAP description.** The approval email carries the Mail Action Protocol part
  (`application/ld+json`, profile map/0.3). We extracted it decoded, byte-exact, as description.json
  (1,648 bytes). One real friction point: the mailbox's "show original" view sanitized the ld+json
  part to empty (two identical captures, after a settings change too) — the part only survived as a
  file attachment chip; fetching the attachment by its attachment id gave the exact part. Anyone
  scripting this against a big mailbox should read attachments, not the raw view.
- **Minutes from Safe-Mode-on to decision.** Safe Mode was verified on 2026-11-02T11:4xZ (preflight
  day). The decision itself was recorded 2026-11-04T01:09:53.360Z — but the chain was deliberately
  parked in between for sequencing reasons of our own (the campaign was composed only inside our
  11-04 execution window, per our runbook), so the raw elapsed time (~13.5 h) measures our calendar,
  not the product. Hands-on composing-to-decision time was ~15 minutes.
- **Validation with the mailschema package.** The VM has no Node; the package declares engines >=22.
  We ran the published mailschema@0.3.3 code (its dist ESM, imported from the jsDelivr CDN in a
  browser page) with the two artifacts: the description above, and the contract fetched byte-exact
  (10,499 bytes) from
  `https://mailschema.org/artifacts/map-0.3/contracts/campaign-send-approval-0.1.json`. Validation is
  offline and keyless by design ("Nothing is uploaded or changed"): no account, no POST. Exact
  checks: `Contract.parse(contract)` → digest
  `sha-256:de3c88a0712adb1d0c6e82f437b825c83520d8e1c44c7b032a2cfcfe5ce55731` (matches the
  description's `type.contractDigest`); `parseDescription(description)` → OK;
  `contract.descriptionErrors(description)` → `[]` (no errors — valid against Campaign Send Approval
  0.1). A first `digest()` call passed the raw JSON *string* and produced a different value; the
  correct usage (per the package's own `dist/core/json.js`) is `digest(parsedValue)` — worth a line
  in the README.
- **terms.version recompute.** With the package's `digest()`: full description →
  `sha-256:0ed19e3fa5e89bb3f0dc6060968db3a18ba19900ee4a0544e5b65005b210c176`; description **without**
  `terms.version` (key removed from the parsed object) →
  `sha-256:bf55889aab5eecc9f63ed299dec5cb62e8bf425817610e351aa11a3af47b11f4`, which **equals** the
  description's `terms.version`. Independently reproduced in Rust with an RFC 8785 canonicalizer
  (serde_json_canonicalizer 0.4.1 + sha2): both digests identical — two implementations agree.
- **The decision: APPROVE — and why.** The held campaign was composed by our own agent, on our own
  account, to an audience of exactly one: our own address. The description (validated above) binds
  one immutable content revision (digest-pinned), one fixed recipient set, one authorized sender, no
  incremental charge ("MUST NOT cause an incremental monetary charge" — the contract's own money
  guard), and approval grants nothing beyond this one revision. So approving endorsed only content we
  had fully inspected ourselves. The decision went through the approval surface at the exact
  `terms.version` (the description's `terms.id` is the approval resource), recorded 01:09:53.360Z,
  `delivery_status: completed` (the send finished 01:26Z, one recipient, delivered 100%).
- **Step that slowed us most.** The mailbox sanitizer eating the ld+json part (above): two capture
  attempts showed an empty part and cost the most calendar time; the attachment route solved it in
  one fetch. Runner-up: the stale-password lesson on the account's JWT login (one 401) — the
  decision still went through the documented web approval surface at the pinned version, which is
  the surface the protocol intends a human to use.
- **One defect, with exact repro.** The mailbox web client's "show original" (raw message view)
  returns the `application/ld+json` MIME part **empty**. Repro: receive any approval email carrying
  an ld+json MAP part, open the raw/original view, search for `application/ld+json` — the part body
  is zero bytes (confirmed twice, including after changing mail settings; the same part is intact in
  the attachment listing, 1,648 bytes). We tried: the raw view twice, a settings change, the
  attachment chip route (worked), and the account's own API (the approval record carries the parsed
  `mail_action` but not the raw part). A zero-byte ld+json part in a raw view breaks every scripted
  extraction path that follows the format's own extraction instructions.
- **One change we would make.** Ship the raw-message view byte-faithful (or at minimum return the
  ld+json part body unchanged) — one sanitizer exclusion for `application/ld+json`. Second choice:
  expose the receipt link for a decided approval in the same API that exposes the approval record
  (`/v1/my/map/approvals/<id>`), so an automated integrator doesn't depend on a single email for
  proof-of-decision.

*Checked against the Safe Mode docs, the MAP core specification and the mailschema README during the
run; every command above is quoted from our transcripts (var/scratch logs, 2026-11-02/04). No keys
or tokens in this report. Nothing in this document points at where or why this run was performed — it describes only the run itself.*
