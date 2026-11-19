# Design memo: mVote on the mBolts relay

Design memo · November 19, 2026 · **Status: proposed. Nothing here is accepted until the author approves it.** It changes no code, no deployment and no Firestore rules. It covers moving mVote's boards and votes from Google Firestore to `@malverdesoftware/relay`, the self-hostable store mPile already runs on, so both apps share one open-source stack that a community can run on its own Cloudflare account or its own server.

## Question and scope

mVote stores boards, vote counts and anonymous participant records in Firestore. The facilitator's Python tool writes boards through the Firebase Admin SDK, and the web app reads and votes through the Firebase web SDK under `firestore.rules`. mPile runs on the mBolts relay: a Cloudflare Worker with a D1 (SQLite) database, the same handler on Node with `node:sqlite`, and content sealed in the browser before it is sent.

This memo answers four questions:

1. What mVote needs that the relay lacks today.
2. Which design closes that gap with the least new surface and the strongest privacy.
3. How live tallies reach every phone on Cloudflare, and how the Python tool talks to the relay.
4. How mvote.app moves over, and how it moves back if something goes wrong.

Out of scope: new voting modes, the creation-key web page (SPEC-distribution Phase 3, item 8), and any change to mPile's session flow.

## Evidence read

| Source | Locator | What it establishes |
|---|---|---|
| mVote web store | [`webapp/src/data/firestoreBoardStore.js`](https://github.com/MalverdeSoftware/mvote/blob/dec6316b90ed446d9fe4c3f72e126cfd233c211b/webapp/src/data/firestoreBoardStore.js) | The seven store functions the UI calls: `loadBoard`, `subscribeBoardStatus`, `subscribeVotes`, `joinParticipant`, `castVote`, `requestMoreDots`, `setDone`. 8 starting dots, 4 more each time a participant runs out. |
| mVote store switch | [`webapp/src/data/index.js`](https://github.com/MalverdeSoftware/mvote/blob/dec6316b90ed446d9fe4c3f72e126cfd233c211b/webapp/src/data/index.js) | The UI imports only from this file, so a third store can sit beside the Firestore and mock stores. |
| mVote rules | [`firestore.rules`](https://github.com/MalverdeSoftware/mvote/blob/dec6316b90ed446d9fe4c3f72e126cfd233c211b/firestore.rules), lines 18 to 21, 75 to 79, 105 | Board reads are public by ID. A browser creates its own participant record with 8 dots, and the rules cannot tell a new browser from a new identity. Extra dots have no upper limit. |
| Dot-budget note | [`README.md`, line 56](https://github.com/MalverdeSoftware/mvote/blob/dec6316b90ed446d9fe4c3f72e126cfd233c211b/README.md) and [`webapp/README.md`, line 48](https://github.com/MalverdeSoftware/mvote/blob/dec6316b90ed446d9fe4c3f72e126cfd233c211b/webapp/README.md) | A skilled participant can get around the dot budget by writing to Firestore directly. |
| mVote tool | [`cli/mvote/firebase_store.py`](https://github.com/MalverdeSoftware/mvote/blob/dec6316b90ed446d9fe4c3f72e126cfd233c211b/cli/mvote/firebase_store.py) and [`cli/mvote/cli.py`](https://github.com/MalverdeSoftware/mvote/blob/dec6316b90ed446d9fe4c3f72e126cfd233c211b/cli/mvote/cli.py) | `create --publish`, `results` (CSV with `board_id, board_title, item_id, label, vote_count`), `status`, `close`, `reopen`. All Firebase code sits in one module by design. |
| mVote hosting | [`wrangler.jsonc`](https://github.com/MalverdeSoftware/mvote/blob/dec6316b90ed446d9fe4c3f72e126cfd233c211b/wrangler.jsonc) | A static-assets Worker on mvote.app with Workers observability **on**. |
| Firestore transport | [`webapp/src/firebase.js`](https://github.com/MalverdeSoftware/mvote/blob/dec6316b90ed446d9fe4c3f72e126cfd233c211b/webapp/src/firebase.js) | Streaming connections already fail behind some school and shared wifi proxies, which is why long-polling detection is on. |
| Relay contract | [SPEC-relay](specs/SPEC-relay.md), HTTP contract v1, Rules, line 125 | Sessions, slots, drafts and submissions only. Close is one way. Expiry at most 30 days. `/v1/sessions/:sessionId/shared` is reserved for mVote and unimplemented. |
| Envelope format | [SPEC-sealed-envelope](specs/SPEC-sealed-envelope.md), lines 46 and 86 | The `"shared"` channel is reserved for mVote. A Python implementation must pass `fixtures/envelope-v1.json`, and that file only changes with a version bump. |
| Session client | [SPEC-session-client](specs/SPEC-session-client.md), line 96 | `publishShared`, `readShared` are reserved as additive methods. Links keep every secret after `#`. |
| Data visibility | [DATA-VISIBILITY](DATA-VISIBILITY.md) | What the relay stores and what Cloudflare sees. Apps should turn off Workers observability. |
| Distribution plan | [SPEC-distribution](specs/SPEC-distribution.md), Phase 2 item 5, Phase 3 items 8 and 9, decisions Q1 and Q3 | The approved plan adds `POST /api/boards` to the mVote Worker, writing to Firestore through its REST API. Creation keys go to collaborators on request. Self-hosting mVote currently means creating a Firebase project. |
| mPile mount | [`app/worker/index.ts`](https://github.com/MalverdeSoftware/mpile/blob/eda0f9999b5eac40821221adb5ae4a2b5172a1c5/app/worker/index.ts) | How an app mounts the relay: `/relay/*` goes to `createRelay` over a dedicated D1 binding, the schema is applied once per isolate, and an hourly cron runs `purgeExpired`. |
| Cloudflare limits | [Workers limits](https://developers.cloudflare.com/workers/platform/limits/), [Durable Objects pricing](https://developers.cloudflare.com/durable-objects/platform/pricing/), [D1 pricing](https://developers.cloudflare.com/d1/platform/pricing/), read November 19, 2026 | Free plan: 100,000 Worker requests a day per account, D1 5 million rows read and 100,000 rows written a day. SQLite-backed Durable Objects are available on the free plan with 100,000 requests a day, incoming WebSocket messages count at 20 to 1, and hibernating WebSockets incur no duration charge while idle. |

## What mVote needs that the relay lacks

| Need | Firestore today | Relay today | Gap |
|---|---|---|---|
| **Live shared tallies** every participant sees | `onSnapshot` on the `votes` collection | Each participant reads only the definition and their own draft. Nothing is shared between participants. | A board-wide tally that anyone holding the link can read, and a way to learn quickly when it changes. |
| **Per-participant dot budget enforced on the server** | Rules tie each vote to a spent dot, but anyone can mint a new participant ID with 8 fresh dots, and extra dots are unlimited | Write tokens per slot, with no notion of a budget or counter | Server-side counters for dots granted and spent, a vote write that is atomic with spending a dot, and a cap on how many identities a board accepts. |
| **Close and reopen** | `closed` flag, both directions | Close is one way, `isClosed: false` returns `422` | A board lifecycle that allows reopening without changing mPile's one-way rule. |
| **Anonymous join with no key file** | Open `/{boardId}`, get a random ID in `localStorage` | Every participant needs a personal link and a six-digit passcode generated by the researcher | One link (and QR code) for the whole room, with no passcode by default. |
| **CSV export from the CLI** | `mvote results` reads Firestore with the admin key | `session-client collect` in Node, with mPile-shaped output | A Python client that reads the tally, opens the sealed item labels and writes the same CSV columns as today. |
| Status for the facilitator | `mvote status`: participants, done, dots placed | `GET /v1/sessions/:id` counts slots | A board summary with participants joined, done and dots placed. |
| "I'm done voting" | `done` flag per participant | None | A per-participant flag. |

## Design options

All three options mount the relay inside mVote's own Worker at `mvote.app/relay` with a dedicated D1 database, as mPile does at `mpile.app/relay`. They share the relay's existing bearer-token roles, boundary validation, headers, no-logging rule, expiry and hourly purge. They differ in what the relay can read.

### Option A: counted board with sealed content (recommended)

The board's title, item labels and descriptions are sealed in the facilitator's tool with a **board key** that travels only in the link fragment, `https://mvote.app/b#v1.<boardId>.<boardKey>` (about 90 characters). Browsers never send the fragment to a server, so the relay stores the sealed definition and cannot read it. Votes name items by **position** (0 to n−1). The relay therefore counts dots on numbered positions, enforces every budget rule itself, and serves the tally to anyone holding the link, while only link holders can turn position 3 back into "Sleep quality".

**Keys from one secret.** The link carries the 32-byte board key. HKDF-SHA-256 derives two values from it, as `deriveSlot` does for participants: a content key for the sealed definition and a **read token** whose hash the relay stores. The read token gates the tally and the definition, so a guessed board ID alone opens nothing. The envelope context is `["malverdesoftware-envelope/v1", "mvote", boardId, "shared", ""]`, which uses the reserved `"shared"` channel and leaves wire format v1 unchanged.

**Joining.** A browser opening the link generates a random participant secret, keeps it in `localStorage` (as `participantId.js` does today) and registers its hash with the relay. The relay grants `startingDots` and refuses new participants past `maxParticipants`, which the facilitator sets to the expected headcount plus a margin. An optional **join code**, shown on the projector and typed once, stops someone outside the room from filling the board with identities. Clearing browser storage still yields a fresh identity, but every identity counts against the cap, so the total number of dots on the board has a hard ceiling for the first time.

**Voting.** Each tap generates a random `voteId` on the phone, stored before sending, so a retry after a lost response replays the first result. The relay performs one atomic write per vote: record the vote ID, add one to the participant's spent count only while it is below their granted count and the board is open, and add one to that position's count. Two simultaneous taps with one dot left give exactly one success, using the same conditional-write discipline the relay already uses for drafts and submissions. A reused `voteId` with a different position fails with `409`. The relay keeps no record linking a participant to the items they chose, only each participant's spent and granted totals.

**Extra dots.** `requestMoreDots` becomes a server rule: allowed only when spent equals granted, adds `reupDots`, and stops after `maxReups` when the facilitator sets one. The default keeps today's behavior (4 more, no limit) so sessions feel the same.

**Proposed resource family.** A new `/v1/boards` family with its own tables, kept apart from sessions so mPile's contract, one-way close and slot rules stay exactly as approved.

| Method and path | Role | Purpose |
|---|---|---|
| `POST /v1/boards` | CREATOR | Create a board: `{ boardId, app, adminTokenHash, readTokenHash, joinCodeHash?, expiresInDays?, definition, rules: { itemCount, startingDots, reupDots, maxReups, maxParticipants } }`. Idempotent on `boardId` with a request hash, as session creation is. |
| `GET /v1/boards/:boardId` | ADMIN | Status: open or closed, expiry, participants, done, dots placed. |
| `PATCH /v1/boards/:boardId` | ADMIN | `{ isClosed, expiresAt? }`, with reopening allowed. |
| `DELETE /v1/boards/:boardId` | ADMIN | Delete everything, `204` also when already gone. |
| `GET /v1/boards/:boardId/definition` | READ | The sealed definition. |
| `GET /v1/boards/:boardId/tally?since=<version>` | READ | `{ version, isClosed, counts: number[] }`, or a short "unchanged" answer when `since` matches. |
| `POST /v1/boards/:boardId/participants` | READ, plus the join code when set | Register `{ participantId, tokenHash }`. `422 LIMIT_EXCEEDED` when full. |
| `GET /v1/boards/:boardId/participants/:participantId` | PARTICIPANT | Own granted, spent and done values after a reload. |
| `PATCH /v1/boards/:boardId/participants/:participantId` | PARTICIPANT | `{ isDone }`. |
| `POST /v1/boards/:boardId/votes` | PARTICIPANT | `{ participantId, voteId, position }`. New codes `NO_DOTS_LEFT` and `BOARD_CLOSED`, both `409`. |
| `POST /v1/boards/:boardId/participants/:participantId/more-dots` | PARTICIPANT | `{ requestId }`, idempotent. |

Storage is three new tables, created with `IF NOT EXISTS` beside the existing two, so the change is additive: `relay_boards` (IDs, hashes, sealed definition, rules, counts, version, created, expiry and closed times), `relay_board_participants` (board and participant IDs, token hash, granted, spent, more-dot requests, done) and `relay_board_requests` (board ID and the vote or more-dot request ID, kept as long as the board for replay). No column holds an IP address, user agent, label or plaintext. `deleteSession`, `purgeExpired` and the custody and no-logging tests extend to these tables.

**Cost of the option.** Two additive exports in `sealed-envelope` (`deriveBoard`, plus a fixture file `fixtures/board-v1.json` so `envelope-v1.json` stays frozen), a new store adapter interface for boards on memory, SQL and D1, a small browser client, and a Python implementation of A256GCM sealing, HKDF and token hashing.

### Option B: sealed ballots on the shared channel

This option builds what the reserved `publishShared` and `readShared` names describe. Each participant publishes a sealed **ballot**, their whole vote vector, under the board key on the `"shared"` channel. Every phone downloads all ballots, opens them and adds them up. The relay stores only ciphertext and cannot read counts.

The budget is then checked by every reader: each phone applies the same rule (a ballot over its budget counts as empty), so honest phones agree on the tally even if one participant writes an oversized ballot. The relay can still cap the number of ballots.

The costs are significant. Every phone downloads every ballot on every refresh, padded to at least 1 KiB each, so 40 participants polling every 3 seconds move about 40 KiB per phone per refresh, on school wifi and older phones. Each participant's full vote vector becomes readable by everyone in the room, which is more than Firestore exposes today (participant records cannot be listed) and enough to link a ballot to the person who just tapped. The facilitator status counts (participants, done) would also have to be computed from ballots.

### Option C: plain board mode in the relay

Port the Firestore data model to D1 as it is, with titles and labels stored readable. This is the fastest path and needs no cryptography in Python. It breaks the promise in [DATA-VISIBILITY](DATA-VISIBILITY.md) that the relay never stores activity content in readable form, so the relay would hold two classes of data with different guarantees, and every self-hoster would have to know which is which. It fits only as a short-lived fallback if Option A's Python work stalls.

### Considered and set aside

Open-source Firebase alternatives that can be self-hosted (Supabase, PocketBase, Appwrite) would remove Google from the picture, but each adds a second server stack, a second auth model and a second self-hosting guide beside mPile's, which is the opposite of the goal of one shared stack.

### Comparison

| | A: counted, sealed content | B: sealed ballots | C: plain board | Firestore today |
|---|---|---|---|---|
| Host can read titles and labels | No | No | Yes | Yes (Google) |
| Host can read counts | Per position only | No | Yes | Yes |
| Host stores who voted for what | No | Sealed ballots per participant | Depends on design | Yes, per participant |
| Other participants can see someone's choices | No | Yes | No | Only with participant colors turned on |
| Budget enforced by | Relay | Every reader | Relay | Rules, with unlimited identities |
| Cap on identities | Yes | Yes | Yes | No |
| Data per refresh on each phone | One small tally | Every ballot | One small tally | Changed documents |
| New crypto in Python | AES-GCM, HKDF, SHA-256 | Same | None | None |

## Live updates on Cloudflare

**Polling is the baseline every host must support.** The relay's rule is that the handler uses only the Fetch API and WebCrypto so it runs unchanged on Workers and on Node. A conditional poll (`GET /tally?since=<version>`) meets that rule everywhere, works through the proxies that already trouble Firestore's streaming transport, and needs no new infrastructure. The browser polls about every 3 seconds while the page is visible, backs off to about 15 seconds after a minute without change, stops while the tab is hidden, and polls once immediately after its own vote.

Rough budget for one 60-minute session with 40 participants, 20 minutes of active voting and 16 dots each:

- Polling: 40 × 400 polls at 3 seconds plus 40 × 160 at 15 seconds ≈ 22,000 requests, plus about 640 vote requests. Static assets do not invoke the Worker script, so they do not count.
- D1: one row read per poll and about three rows written per vote, far below the daily free allowance.
- About a fifth of the free plan's 100,000 daily Worker requests, which are shared across every Worker on the account (mpile.app included). Two large sessions on one day approach the limit. The Workers Paid plan ($5 a month, 10 million requests included) removes the concern, and that choice belongs to the author.

**Durable Objects with hibernating WebSockets are the push upgrade.** One Durable Object per board holds the board's state in its own SQLite storage, applies votes one at a time (which makes the budget rule simple to reason about) and pushes each new tally to every connected phone. The same session costs about 40 connection requests plus 640 incoming messages counted at 20 to 1, so roughly 75 billed requests, with no duration charge while sockets sit idle. Durable Objects exist only on Cloudflare, so this path needs a second board-store adapter, and the Node path keeps polling. Clients fall back to polling whenever a socket fails, which also covers restrictive networks.

**Recommendation:** ship polling first, measure request counts at the first real sessions, and add the Durable Object push only if quotas or perceived lag call for it. The tally response shape stays identical across both, so the push becomes a transport detail behind the same browser function.

## How the Python tool talks to the relay

- **Same commands, new store module.** `cli/mvote/firebase_store.py` keeps its functions, and a new `relay_store.py` implements the same names (`save_board`, `set_board_closed`, `board_status`, `export_board_results`) over HTTPS JSON with the standard library's `urllib`. `board.py` stays backend-agnostic, as its docstring intends.
- **Crypto.** A small `mvote/envelope.py` implements A256GCM sealing, the HKDF derivation and `hashToken`, and its tests run the A256GCM and hash vectors in mBolts' `envelope-v1.json` and the new `board-v1.json`, copied with a recorded SHA-256. It needs the `cryptography` package, a new dependency of the mvote tool, because Python's standard library has no AES-GCM. ECDH sealing is unnecessary for Option A.
- **Creating a board.** `mvote create --config items.csv --title "Week 3" --publish --creation-key <key>` (or `MVOTE_CREATION_KEY` in the environment) generates the board ID, board key and admin token locally, seals the definition once, publishes it and prints the participant link and QR code. Publishing is retried byte for byte, as `publishSession` does.
- **The facilitator's board file.** The tool writes `<title-slug>.mvote-board.md`, a Markdown file in the style of the session key file: the link, the admin token, the board key and the item list in a JSON block (`mvote-board/v1`), under a warning that anyone holding it can close or delete the board. MalverdeSoftware repositories ignore that filename pattern in git. Participants never need it.
- **Results and status.** `mvote results --board-file week-3.mvote-board.md --out results.csv` reads the tally with the read token, opens the definition and writes the same five CSV columns as today. `mvote results --link <participant link>` also works, so a lost board file costs the ability to close or delete early, while results stay recoverable until the board expires. `status`, `close`, `reopen`, plus new `extend` and `delete`, use the admin token.
- **Backend choice during the move.** A `--backend firestore|relay` option, defaulting to `firestore` until cutover, then to `relay`.

## Migration and rollback for mvote.app

The web app chooses its store **by link shape**. A link with a `#v1.` fragment uses the relay store, and a plain `/{boardId}` link uses the Firestore store. Old boards keep working untouched while new boards move, with no copying of board data and no flag day.

1. **mBolts release.** Relay board endpoints, store adapters, `deriveBoard`, browser board client and the new fixtures, with the contract suite on both stores. mPile pins exact versions and needs no change.
2. **Staging rehearsal.** With the author's approval, deploy the mVote Worker with the relay mounted to a separate staging Worker and D1 database. Run a rehearsal on iPhone and Android phones, school-style wifi, and the Python tool from PyPI.
3. **mvote.app gains the relay, unused.** The Worker gets a script entry that sends `/relay/*` to `createRelay` (assets stay free), a dedicated `mvote-relay` D1 database, the hourly purge cron, **Workers observability turned off**, and a content security policy that allows only `'self'` for connections once Firestore is gone. Merging to mvote main redeploys mvote.app, so this step merges only on the author's go-ahead. Old links behave exactly as before.
4. **Cutover.** Release the mvote tool with `--backend relay` as the default. New boards use the relay, and existing Firestore boards stay readable and votable.
5. **Firestore read-only.** After the last planned Firestore session, the author deploys rules that refuse all client writes (rules deploy separately from the site), and exports each board still needed with `mvote results --backend firestore`.
6. **Sunset.** Remove `firestoreBoardStore.js`, the Firebase SDK, `firebase.js`, `firebase.json`, `firestore.rules`, the Firebase values in `.env.production`, `firebase_store.py` and the `firebase-admin` dependency, in a release separate from the cutover. The author deletes the Firebase project after the exports are confirmed.

**Rollback at each step:**

- Before step 4, revert the Worker deploy. Nothing depends on the relay yet.
- After step 4, set the tool's default back to `firestore` in a patch release (or pass `--backend firestore`). Boards already created on the relay keep working as long as the relay mount stays deployed, so a web-app rollback keeps the `/relay/*` route and the relay store, and reverts only what broke.
- If the relay itself must come down, relay boards created since cutover stop working and their facilitators need fresh Firestore boards. Results exported before that point are unaffected. Keeping steps 4 and 5 at least two weeks apart, and checking `mvote status` counts against what facilitators saw in the room, limits that exposure.
- Step 6 is the only step without a quick rollback, which is why it waits until no Firestore board is still in use.

## Privacy and governance trade-offs

- **Who can read what.** Under Option A, Cloudflare stores sealed titles and labels, numbered counts, each participant's spent and granted totals and the board's timing. It sees network metadata as described in DATA-VISIBILITY. Google no longer holds anything. Firestore today holds readable titles, labels and counts that anyone with the board ID can read, and each participant's per-item vote map, readable by anyone who learns that participant's ID.
- **What a request reveals in flight.** A vote request names the participant and a position, so a modified relay could log who voted for which position. The relay code records neither, and it never sees which item a position means. This is the same trust placed in Firestore today, with less exposed.
- **Link holders.** Anyone with the link can read the board and join, as today. The join code and participant cap limit what a forwarded link can do. Per-person budgets that survive clearing a browser require personal links, which the relay's slots already support. That remains a later option for sessions that need it, since it gives up the single shared QR code.
- **Retention.** Relay boards expire after 14 days by default and at most 30, and the hourly purge deletes them. Firestore boards never expire. The shorter life fits data minimization, and facilitators must export results before expiry. D1 Time Travel keeps a recoverable copy of deleted rows for 7 days (free) or 30 days (paid), holding only the data listed above.
- **Participant colors.** The optional `VITE_TRACK_PARTICIPANT_COLORS` view records which participant placed each dot. Option A deliberately stores no participant-to-item link, so this view would not carry over unless the author decides it is worth a separate, opt-in design.
- **Logging.** mvote.app has Workers observability on today. It must be off before the relay is mounted, matching mPile and DATA-VISIBILITY. The open request to remove Cloudflare's Web Analytics script from the site fits the same change.
- **Who can create boards.** Creation keys on mvote.app go to collaborators on request (SPEC-distribution Q1). Anyone else self-hosts, and self-hosting becomes one Worker and one D1 database, the same as mPile, with no Google account or Firebase project. That fits MalverdeSoftware's aim that communities and research partners can run the tools themselves and keep their data under their own governance.
- **Cost exposure.** On the free plan, a busy day can exhaust the shared request allowance and take mpile.app down with mvote.app until midnight UTC. The polling budget above, or the paid plan, addresses that.

## Reconciliation with current decisions

| Record | Relation | Effect |
|---|---|---|
| SPEC-relay, reserved `/v1/sessions/:sessionId/shared` (approved October 25) | Revised | Option A proposes a separate `/v1/boards` family. The reservation can stay for sealed per-participant shared data (Option B) and is not needed for mVote. Adding endpoints is "ask first". |
| SPEC-relay, one-way close | Unchanged | Applies to sessions only. Boards allow reopening. |
| SPEC-sealed-envelope, `"shared"` channel and frozen fixtures | Applied | Option A uses the reserved channel and adds `board-v1.json` beside the frozen file. A new export `deriveBoard` is additive. |
| SPEC-session-client, reserved `publishShared`/`readShared` | Unchanged | Option A adds a separate board client. The reserved names stay reserved. |
| SPEC-distribution Phase 2 item 5 (approved November 12) | Revised | Creation keys go straight to the relay. The planned Firestore REST endpoint becomes unnecessary. Q3 (endpoint and tool option first, web page after) still holds. |
| SPEC-distribution Phase 3 item 9 | Simplified | mVote's self-hosting guide drops the Firebase project. |
| mVote ROADMAP decision "Backend: Firebase" | Superseded if accepted | The roadmap already calls Firebase a default that can be swapped. |
| CONVENTIONS, "ask first" items | Applies | New dependency (`cryptography` in the mvote tool), deploying a relay, DNS for a staging host, and any change to the session file format (none proposed). |

## Validation plan

- Relay contract suite on `memoryStore` and `sqlStore(node:sqlite)` for every new endpoint and error code, including replayed and conflicting vote IDs, two simultaneous votes with one dot left giving exactly one success, the participant cap under concurrent joins, reopen, expiry and purge leaving zero rows in all five tables.
- An extended custody test: a full simulated board with marker strings in the title and labels asserts that no request and no stored row contains them, the board key or the participant secrets.
- The no-logging spy across the new suite.
- Python tests against the fixture files, and an end-to-end test of `create`, vote through the relay handler on Node, `results` and a byte-for-byte CSV comparison with today's columns.
- mVote store tests against the relay handler in-process, mirroring the mock store tests, and a Wrangler local integration test for the D1 binding.
- Staging rehearsal on phones, school-style wifi, and a 40-client polling script whose request count is checked against the budget above.

## Decisions for the author

1. **Option:** A (recommended), B or C.
2. **Board reopening** in the relay for boards only, leaving sessions one-way.
3. **Board lifetime:** keep the relay's 30-day ceiling for boards, or allow longer.
4. **New dependency:** `cryptography` in the mvote tool.
5. **Where the board client lives:** a `./board` subpath of `@malverdesoftware/session-client` (recommended, one release line) or a new package.
6. **Participant colors:** drop on the relay (recommended) or design an opt-in replacement.
7. **Live updates:** polling first with Durable Objects later (recommended), or Durable Objects from the start.
8. **Workers plan:** stay on the free plan with the polling budget, or move to the $5 plan before cutover.
9. **Staging:** approval to deploy a staging Worker and D1 database, and whether it gets a hostname.
10. **Firestore sunset date** and which existing boards to export.
11. **Revising SPEC-distribution Phase 2 item 5** as described above.

## Phased task list

Each phase ends in something reviewable, and nothing ships to mvote.app without the author's go-ahead.

**Phase 0: decide (mBolts)**
- The author reviews this memo and records decisions 1 to 11 here.
- Update SPEC-relay, SPEC-sealed-envelope and SPEC-session-client with the accepted board contract, and revise SPEC-distribution Phase 2 item 5. Done when the author approves the spec changes.

**Phase 1: relay boards (mBolts)**
- `sealed-envelope`: `deriveBoard`, `fixtures/board-v1.json`, tests.
- `relay`: board tables, store adapter on memory, SQL and D1, `/v1/boards` endpoints, purge and deletion coverage, contract, concurrency, custody and no-logging tests.
- `session-client/board`: create, join, vote, request more dots, mark done, poll the tally with backoff.
- DATA-VISIBILITY: a section for boards.
- Done when `npm test` and `npm run typecheck` pass and packages are released through the release workflow.

**Phase 2: mvote tool (mVote)**
- `envelope.py` with fixture tests, `relay_store.py`, the board file, `--creation-key`, `--backend`, `results --link`, `extend`, `delete`.
- Done when the end-to-end CSV test passes against a local relay (`npm run relay:dev`).

**Phase 3: mvote web app (mVote)**
- `relayBoardStore.js` with the seven store functions, link-shape dispatch in `data/index.js`, join-code entry on the board screen following SPEC-ui.
- Worker entry with the relay mount, D1 binding, purge cron, observability off, page security headers.
- Done when store and integration tests pass and `npm run verify` is clean, on a branch that is not merged.

**Phase 4: rehearsal and cutover (author-approved steps)**
- Staging deploy and phone rehearsal, then merge Phase 3 to mvote main (redeploys mvote.app), then the mvote tool release with the relay default.
- Done when a real session runs on the relay and its exported CSV matches what the room saw.

**Phase 5: Firestore sunset**
- Read-only rules deploy, exports, then the removal release, then deletion of the Firebase project by the author.
- Done when mvote.app and the tool contain no Firebase code and the self-hosting guide describes one Worker and one D1 database.

**Later, if measured need appears**
- Durable Object push for live tallies.
- Personal-link boards for sessions that need a budget per person that survives clearing a browser.
