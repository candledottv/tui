# The attach protocol

Protocol **1.0**. This is the wire contract for an independent view or agent harness. All shapes
below are JSON (TypeScript notation only expresses field types). `?` means optional, `T[]` an
array, and `Record<string,T>` a JSON object. Timestamps are ISO 8601 strings unless noted.
Clients must ignore unknown fields and event kinds. All requests use an object for `params`;
methods with no parameters accept `{}`. Results below are the `result` of a JSON-RPC response.

## Transport and lifecycle

The core listens on a Unix domain socket at `<config dir>/core/<profile>/core.sock`, mode `0600`
inside a `0700` directory. Config dir is `$CANDLE_CONFIG_DIR`, otherwise `~/.config/candle`.
Profiles share neither sockets nor state. Socket paths must fit in 103 UTF-8 bytes.
Messages are UTF-8 JSON-RPC 2.0, one JSON object per newline, at most 1 MiB per message. No batch
arrays. Use a unique string or number `id` for each request and match responses by `id`; events
may interleave with responses. Authentication must finish within 10 seconds of connecting.

```json
{"jsonrpc":"2.0","id":1,"method":"core.hello","params":{"protocol":"1.0","token":"<token>","clientName":"my-view"}}
```

The first call must be `core.hello`; anything else returns `UNAUTHENTICATED` and closes the
connection. A different or malformed protocol major returns `PROTOCOL_MISMATCH` with
`data.core` and `data.client`, then closes. A missing, unknown or revoked token also closes.
`harnessKind?: string` is a descriptive label, not an authorization request. `clientName` and
`harnessKind` are printable names of 1–64 characters. A valid token determines the role.

Hello returns:

```ts
{ protocol: string, coreVersion: string, profile: string,
  environment: "staging" | "production", tradingOn: boolean,
  approvalPolicy: "ask", clientId: string, role: "view" | "harness" }
```

`clientId` identifies this connection, not the installation or token. Ownership of cards and
operations is per connection. A harness reconnect does not inherit a prior connection's ownership.
The core exits after its last view detaches (allowing a bounded wait for in-flight operations).

## Token handoff: starting a view or harness

A view launcher creates an inherited writable pipe, then starts:

```sh
candle core serve --profile <profile> --json --view-token-fd 3
```

Here descriptor 3 is the child's write end; the parent reads the other end. After the socket is
listening, the core writes **one token plus newline** to that descriptor and closes it. The token
is `chv_` plus a random secret, valid only for this core start, kept only in memory. Read the
pipe to EOF, connect to the socket and send hello with it. Never put it in argv, logs or a file.
Without `--view-token-fd` the core supplies no view token. There is no RPC to retrieve it.
A core already serving the profile refuses a second start with `CORE_ALREADY_RUNNING`; a launcher
cannot recover the existing core's view token. Close the original view before starting another.
`candle tui` implements this lifecycle for the bundled view.

A view calls `harness.register {name}` to create a harness token. The result is:

```ts
{ tokenId: string, name: string, createdAt: string, token: string }
```

This is the **one response that returns a newly minted token**. Pass it privately to the intended
harness (for example an inherited pipe or its child-only environment); save it in a private secret
store if the integration needs to reconnect. The core stores only its hash in `tokens.json`
(mode `0600`). A token is `cht_<12-character id>_<secret>` and survives core restarts until revoked.
The harness sends it in hello and cannot promote itself to view. `candle core mcp`
bridges a harness to MCP over stdio; set `CANDLE_CORE_TOKEN` in its private child environment.
The bundled agent receives `CANDLE_CORE_SOCKET`, `CANDLE_CORE_STATE_DIR` and `CANDLE_CORE_TOKEN`
in its child-only environment. Its launcher revokes its token on close and stale tokens on restart.
Neither `clients.list` nor another method retrieves an existing token, vault key or passphrase.
Revoking a token disconnects all connections using it and withdraws their open cards.

## Roles

`yes` may call; `own` is scoped to the connection's cards, operations or subscriptions; `no` is
refused with `-32001`, `data.code: "FORBIDDEN_FOR_ROLE"` before any handler runs.

| Method | View | Harness |
| --- | --- | --- |
| `core.hello`, `core.status` | yes | yes |
| `tools.list`, `tools.call`, `cards.propose` | yes | yes |
| `cards.list`, `cards.get`, `operations.list`, `operations.get` | yes | own |
| `cards.approve`, `cards.reject`, `operations.resolve` | yes | no |
| `cards.withdraw` | own | own |
| `clients.list`, `clients.kill`, `harness.register`, `harness.revoke` | yes | no |
| `settings.get`, `settings.set`, `layout.get`, `layout.set` | yes | no |
| `launch.uploadImage`, `launch.dryRun` | yes | no |
| `transfer.sources`, `transfer.destinations`, `transfer.prepare` | yes | no |
| `quant.status`, `quant.send`, `quant.cancel`, `quant.history` | yes | no |
| `subscribe`, `unsubscribe` | yes | own |

## Shared result types

```ts
type Factor = {type:"passphrase", value:string} | {type:"security-key", pin?:string}
  | {type:"touch-id"} | {type:"passkey"};
type Kind = "trade" | "launch" | "swap" | "stake" | "claim" | "perps" | "profile" | "keyaccess" | "transfer";
type Origin = {clientId:string, role:"view"|"harness", clientName:string, harnessKind?:string};
type SuggestedBy = {agent:"quant", turnId:string};
type Card = {
  cardId:string, opId:string, idKey:string, kind:Kind, tool:string,
  params:Record<string,unknown>, cardHash:string, origin:Origin, reason?:string,
  display:{title:string, contract:string, chain:string, amount:string,
    environment:"staging"|"production", amountUsd?:string, keyLimitUsd?:string,
    estimate?:Record<string,unknown>, checks?:Record<string,unknown>,
    transfer?:TransferDisplay, suggestedBy?:SuggestedBy},
  policy:"ask", needsFactor:boolean, createdAt:string, expiresAt:string,
  state:"proposed"|"awaiting_approval"|"approved"|"rejected"|"withdrawn"|"expired",
  proposalKey?:string, operation?:Operation
};
type Operation = {
  opId:string, cardId:string, kind:Kind, tool:string, idKey:string, ownerClientId:string,
  state:"submitting"|"pending"|"succeeded"|"failed"|"refused"|"unknown"
    |"resolved_resent"|"resolved_abandoned"|"attested_sent"|"attested_not_sent",
  createdAt:string, updatedAt:string, result?:unknown, error?:{code:string,message:string},
  resolution?:"resent"|"abandoned", readNeedsFactor?:boolean,
  evidence?:{before?:unknown,after?:unknown}, lateRecord?:unknown,
  reconcile?:{actions:string[],message:string},
  transfer?:{source:"vault"|"embedded",chain:"solana"|"hood",from:string,to:string,
    id?:string,blockhash?:string,nonce?:string}
};
type Client = {clientId:string, role:"view"|"harness", clientName:string,
  harnessKind?:string, tokenId?:string, connectedAt:string,lastCall?:string,lastCallAt?:string};
type HarnessToken = {tokenId:string,name:string,createdAt:string,revokedAt?:string};
type Window = {open:false} | {open:true,viewClientId:string,closesAt:string};
type Withdraw = {status:"off",reason?:string,prefix?:string} | {status:"on",prefix:string,
  limitUsdMicros:number,reset:"daily"|"weekly"|"monthly"|"never",expiresAt:string|null};
type Trading = {tradingOn:boolean,approvalPolicy:"ask",timedWindow:Window,paperWindow:Window,
  approvalsLockedUntil:string|null,withdraw:Withdraw};
type Settings = Trading & {factors:{type:Factor["type"],usable:boolean,reason?:string}[],
  agents?:{id:"quant",kind:"hosted-quant",dataSources:string[],key:"core-read",
    canPropose:false,memory:"none",on:boolean}[],agentsRefused?:string};
type Status = {profile:string,environment:"staging"|"production",tradingOn:boolean,
  approvalPolicy:"ask",viewCount:number,clientCount:number};
```

A view must draw the full parameters, environment and hash of the card it approves. Send a factor
only with the one request that needs it; do not store it. `suggestedBy` is view-only display
attribution and never changes execution parameters. An `unknown` outcome means money may have
moved: offer only the actions in `operation.reconcile.actions`.

## Core, client and layout calls

| Method | Parameters | Result |
| --- | --- | --- |
| `core.status` | `{}` | `Status` |
| `clients.list` | `{}` | `{clients:Client[],harnessTokens:HarnessToken[]}` |
| `clients.kill` | `{clientId:string}` | `{killed:string,revokedTokenId:string|null}` |
| `harness.register` | `{name:string}` | `HarnessToken & {token:string}` (new secret, shown once) |
| `harness.revoke` | `{tokenId:string}` | `{revoked:string,disconnected:string[]}` |
| `settings.get` | `{}` | `Settings` |
| `settings.set` | `{path:string,value:unknown,factor?:Factor}` | updated `Settings` |
| `layout.get` | `{}` | `{layout:Record<string,unknown>}` (default `{preset:"C"}`) |
| `layout.set` | `{layout:Record<string,unknown>}` | `{layout:Record<string,unknown>}` |

Killing yourself is invalid; detach instead. Killing a harness also revokes its token.
Unknown clients return `CLIENT_NOT_FOUND`; unknown/already revoked tokens return `TOKEN_NOT_FOUND`.
Layout is a client-defined object of at most 64 KiB, persisted atomically.

Settable settings paths:

| Path | Value | Authorization |
| --- | --- | --- |
| `trading` | boolean | turning on requires a factor |
| `window` | `{minutes:number}` or `false` | opening requires a factor; capped at 15 minutes |
| `paperWindow` | `{minutes:number}` or `false` | opening requires a factor; capped at 15 minutes |
| `withdraw` | `{limitUsd:number,reset?:"daily"|"weekly"|"monthly"|"never"}` or `false` | both directions require a fresh factor; positive limit; default daily |
| `quant` | boolean | hides/enables Quant; no factor |

`approvalPolicy` is fixed to `"ask"`; attempts to set it return `POLICY_FIXED`.
Invalid paths/values return `INVALID_SETTING`. Factor rejection, unavailability and lockout are
separate errors (`FACTOR_REJECTED`, `VAULT_FACTOR_UNAVAILABLE`, `APPROVALS_LOCKED`).
Five rejected factors in ten minutes lock approvals for five minutes.

## Tools, cards and operations

| Method | Parameters | Result |
| --- | --- | --- |
| `tools.list` | `{}` | `{tools:{name:string,class:"read"|"card",kind?:string,idKey?:string}[]}` |
| `tools.call` | `{name:string,args?:Record<string,unknown>}` | `ToolResult` below |
| `cards.propose` | `{tool:string,params:Record<string,unknown>,reason?:string,suggestedBy?:SuggestedBy}` | `Card` (with `operation` if already executed) |
| `cards.list` | `{}` | `{cards:Card[]}` |
| `cards.get` | `{cardId:string}` | `Card` |
| `cards.approve` | `{cardId:string,cardHash:string,factor?:Factor,confirm?:string}` | `{card:Card,operation:Operation}` |
| `cards.reject` | `{cardId:string}` | updated `Card` |
| `cards.withdraw` | `{cardId:string}` | updated `Card`, proposer only |
| `operations.list` | `{}` | `{operations:Operation[]}` |
| `operations.get` | `{opId:string}` | `Operation` |
| `operations.resolve` | `{opId:string,action:"check"|"resend"|"abandon"|"rebroadcast"|"attest_sent"|"attest_not_sent",factor?:Factor}` | updated `Operation` |

The offered list classifies tools; their argument schemas and API result bodies follow the public
[Candle tool reference](https://github.com/candledottv/agentic). Unclassified tools are refused.
Read tools relay API results. A paper trade or launch dry run uses an open paper/live window.
Live money writes propose a card and return immediately; they do not wait for the human.
Transfers are absent from `tools.list` and refused by `tools.call`/`cards.propose`.

```ts
type ToolResult = {content:{type:"text",text:string}[],isError?:boolean};
```

Parse `content[].text` as JSON. Refusals have `isError:true` and
`{success:false,error:{code:string,message:string},...details}`. A proposed live action has
`isError` absent and the following body (APPROVAL_REQUIRED is not an execution failure):

```json
{"success":false,"error":{"code":"APPROVAL_REQUIRED","message":"A card awaits the user's approval. This is not a refusal and not a failure."},"cardId":"card_…","clientTradeId":"…","opId":"…","kind":"trade","state":"awaiting_approval","expiresAt":"…","next":"…"}
```

The dynamic id field is the tool's `idKey`, also holding `opId`. Keep it for all retries.
Reproposing the same id with changed parameters returns `OP_ID_CONFLICT`. A repeated identical
proposal returns the held card/operation. Read outcomes with
`tools.call {name:"candle_get_operation",args:{clientId:opId,kind}}`:
`{success:true,...ids,state:"succeeded",result}` or
`{success:false,error,...ids,state,result?,next}`. Pending uses `OPERATION_PENDING`, uncertain
outcomes `OUTCOME_UNKNOWN`, closed cards `CARD_REJECTED`/`CARD_WITHDRAWN`/`CARD_EXPIRED`.
Never turn an uncertain result into another request under a fresh id.

Approvals require the displayed `cardHash`; a stale hash returns `CARD_HASH_MISMATCH`. Transfers
always require typed `confirm:"confirm"` and a fresh factor in the same call. Other actions may
use a live window when allowed; launch, transfer and key-access actions still need a fresh factor. A proposal with no view is `NO_VIEW_ATTACHED`. Card and operation lookup errors
are `CARD_NOT_FOUND`/`OPERATION_NOT_HELD`; foreign harness ownership is refused.
`resend` needs a fresh factor and reuses exactly the same id and parameters. A vault transfer may
rebroadcast only the same signed bytes; an embedded transfer cannot be re-sent after an unknown
outcome. Attestations record what the user says, not a chain confirmation.

## Launch and transfer calls (view only)

| Method | Parameters | Result |
| --- | --- | --- |
| `launch.uploadImage` | `{path:string,confirmPublic:true,factor?:Factor}` | `{imageUrl:string,contentType:string,size:number}` |
| `launch.dryRun` | `{cardId:string,factor?:Factor}` | updated launch `Card` |
| `transfer.sources` | `{factor?:Factor}` | `{sources:TransferSource[],vaultListed:boolean,withdraw:Withdraw}` |
| `transfer.destinations` | `{source:{kind:"vault"|"embedded"},chain:"solana"|"hood"}` | `{destinations:Destination[]}` |
| `transfer.prepare` | `TransferInput` (also accepted as `{params:TransferInput}`) | transfer `Card` |

Upload accepts PNG, JPEG, GIF or WebP up to 10 MB. It is public and permanent even without a
launch; ask the user first. Missing confirmation returns `UPLOAD_NOT_CONFIRMED`. With no open
window, a provided factor opens a paper window for upload/dry run. Launch cards begin `proposed`;
a successful dry run sets `display.checks.dryRun:"passed"` and `awaiting_approval`. A failed
check withdraws the card; a transport problem leaves it proposed for retry.

```ts
type Balance = {asset:string,contract:string,raw:string,decimals:number};
type TransferSource = {kind:"vault"|"embedded",walletId?:string,label:string,role?:string,
  chain:"solana"|"hood",address:string,on:boolean,reason?:string,balances:Balance[]|null};
type Destination = {address:string,label?:string,sentBefore:boolean,own?:boolean,
  serverClass?:"own"|"approved"|"linked"};
type TransferInput = {
  source:{kind:"vault"|"embedded",walletId?:string,address:string},
  chain:"solana"|"hood",asset:{symbol:"SOL"|"USDC"|"CNDL"|"ETH"|"USDG"}
    |{mint:string}|{token:string},
  amount:{raw:string}|{decimal:string}|{max:true},to:string
};
type TransferDisplay = {
  sourceKind:"vault"|"embedded",source:string,sourceLabel?:string,to:string,toGrouped:string,
  toLabel?:string,serverClass?:"own"|"approved"|"linked"|"unlisted",newDestination:boolean,
  asset:{label:string,contract:string},amount:{decimal:string,raw:string,max?:boolean,
    balanceAtPreview?:string},feeEstimate?:string,previewLines?:string[],describeSend:string
};
```

Raw amounts are positive base-unit strings; decimal amounts respect asset precision. `max`
requests the full balance at execution time. Base symbols must match the chain (SOL/USDC/CNDL
on Solana; ETH/USDG on Hood). Solana mints use `mint`, Hood token contracts use `token`. The core
normalizes addresses and fills the card with the exact source/destination/amount and preview.
List vault wallets with a factor before preparing a vault send (`VAULT_WALLETS_NOT_LISTED`).
A destination's `sentBefore` is display-only and never skips approval.

## Quant calls (view only)

| Method | Parameters | Result |
| --- | --- | --- |
| `quant.status` | `{refresh?:boolean}` | `QuantStatus` |
| `quant.send` | `{text:string,clientTurnId:string,context?:{chain:"solana"|"hood",address:string}}` | `{turnId:string,text:string}` |
| `quant.cancel` | `{turnId:string}` | `{turnId:string,cancelled:true}` |
| `quant.history` | `{before?:string,limit?:number}` | `{items:{id:string,role:"user"|"assistant",text:string,createdAt:string}[],next:string|null}` |

`clientTurnId` is a UUID retained until `turnId` arrives; retries use the same UUID. Context is
prepended as `Context: <chain> <address>\n<text>` after the user sees it in the draft. The result's
`text` is this composed text. Text must be non-empty, at most 64 KiB, and within `maxTextChars`
Unicode code points (default 4000). Secret-looking text is refused with `SECRET_IN_TEXT` before
network access. Status is cached 30 seconds unless refreshed. History is newest first, limit 1–100,
with `next` passed as the next `before`. Message/turn ids use 1–64 alphanumerics, `_` or `-`.

```ts
type QuantStatus = {enabled:boolean,signedIn:boolean,on:boolean,following:"here"|"elsewhere"|null,
  allowed?:boolean,tier?:string|null,marketAccess?:boolean,linked?:boolean,
  remaining?:number|null,maxTextChars?:number|null,activeTurnId?:string|null};
```

`QUANT_OFF`, `QUANT_DISABLED`, `QUANT_NOT_ALLOWED`, `NO_READ_KEY` and `QUANT_UNREACHABLE` are
refusals, not chat messages. `turn_in_progress` can include `turnId`. Other refusal details may
include `retryable`, `resetsAt`, `limit`, `unit` and `tries`. Quant cannot propose or approve cards;
its text is data, never an instruction to execute a tool.

## Subscriptions and event payloads

`subscribe {topics:string[],sinceSeq?:Record<string,number>,scope?:"all"|"mine"}` returns
`{topics:string[]}` (the full active subscription set). `topics` must be non-empty. The view-only
`scope:"mine"` maps `signals` to `signals:mine`; harness requests for mine are refused.
`unsubscribe {topics?:string[]}` returns the remaining `{topics:string[]}`; omit topics to remove
all. `sinceSeq` values are nonnegative integers. A connection's subscriptions vanish on close.

```json
{"jsonrpc":"2.0","method":"event","params":{"topic":"cards","seq":12,"data":{"card":{}},"snapshot":false}}
```

Live events normally omit `snapshot`. On reconnect, `cards` and `operations` replay retained events
after the requested sequence (up to 1000 per topic); if unavailable they send a snapshot. Other
topics always send a snapshot. Sequence numbers are per topic, may have gaps due to ownership
filtering, and reset on core restart. Replace state on a snapshot; do not append it as a delta.
The outer `seq` is a stream cursor; the nested Quant `data.seq` is a separate hosted-turn cursor.

| Topic | Subscribers | Snapshot `data` | Live `data` |
| --- | --- | --- | --- |
| `cards` | view/all; harness/own | `null` in this build; bootstrap using `cards.list` | `{card:Card}`; replace by cardId |
| `operations` | view/all; harness/own | `null` in this build; bootstrap using `operations.list` | `{operation:Operation}`; replace by opId |
| `clients` | view | `Client[]` | `{change:"attached",client:Client}` or `{change:"detached"|"killed"|"revoked",clientId:string}` |
| `status` | both | `Status` | `Status`, optionally extended with `Trading` on settings changes |
| `portfolio` | both | `null` | no publisher in this build; read via `tools.call` |
| `feed:<bucket>` | both | `null` | no publisher in this build; read via `tools.call` |
| `token:<chain>:<address>` | both | `null` | no publisher in this build; read via `tools.call` |
| `quant` | view | `{status:QuantStatus|null,turn:QuantTurn|null}` | Quant events below |
| `signals` | both | `SignalSnapshot` | `SignalSnapshot`, `SignalStatus`, or `{type:"signal",row:TapeRow}` |
| `signals:mine` | view | `SignalSnapshot & {watchlist:string}` | same variants with watchlist state on snapshot/status |

Feed bucket syntax is `[a-z]{1,32}`. Token topic chains use `[a-z0-9]{1,16}` and addresses
`[A-Za-z0-9]{20,64}`. These topics are reserved transport slots, not a promise of market updates;
use classified read tools such as `candle_get_portfolio`, `candle_get_feed`, and `candle_get_token`.

```ts
type TapeRow = {haccId:number,pattern:string,summary:string,chain:"solana"|"hood",mint:string,
  createdAt:number,symbol?:string,marketCap?:number,walletLabel?:string,wallet?:string};
type TapeStatus = {state:"closed"|"connecting"|"live"|"backoff",paused:boolean,cached:number};
type SignalSnapshot = TapeStatus & {type:"snapshot",rows:TapeRow[]};
type SignalStatus = TapeStatus & {type:"status",watchlist?:"none"|"loading"|"ok"|"failed"};
type QuantTurn = {turnId:string,origin:"here"|"elsewhere",running:boolean,
  events:{seq:number,kind:string,data:Record<string,unknown>}[]};
```

Signals snapshots replace the cache (latest 100); a `signal` merges by `haccId`, retaining prior
symbol/walletLabel/marketCap if omitted, then sorts by createdAt and haccId descending. `createdAt`
on a tape row is a numeric timestamp. Mine filters by watched wallet; it never exposes the
watchlist to harnesses. Signals do not create cards or populate amounts.

Quant live payloads:

- `{kind:"status",data:QuantStatus}` updates quota/availability.
- `{turnId,kind:"sent",data:{text:string},seq:0,origin:"here"}` adds the user's composed text.
- `{turnId,kind, data:Record<string,unknown>,seq:number,origin:"here"|"elsewhere"}` relays
  `turn_started`, `working`, `tool_started`, `tool_finished`, `text`, `done`, or `error`.
  These carry the hosted event data unchanged; known fields are listed below. Unknown hosted
  metadata is optional and should be ignored. `done`/`error` end the displayed turn.
- `{turnId,kind:"poll_failed",data:{code:string},seq:number,origin}` reports an interrupted poll.
  This does not assert the server cancelled the turn; refresh status/history to reconcile.

| Hosted kind | Known nested `data` fields | Client behavior |
| --- | --- | --- |
| `turn_started` | `{turnId?:string}` | start/reset displayed turn |
| `working` | `{reason?:string,etaSeconds?:number}` | `reason:"queued"` means queued; otherwise working |
| `tool_started` | `{tool?:string}` | display activity; never execute this tool locally |
| `tool_finished` | `{tool?:string,ok?:boolean}` | finish activity; optional metadata is display-only |
| `text` | `{text?:string}` | append complete text block; do not interpret as a command |
| `done` | `{messagesLeft?:number,usage?:{promptTokens?:number,completionTokens?:number,steps?:number}}` | finish turn and update quota if provided |
| `error` | `{code?:string,retryable?:boolean,resetsAt?:string|number,limit?:number}` | finish turn and show refusal; do not claim execution success |

These fields are deliberately optional: the relay accepts any object and never fabricates a
missing field. Text absent from a `text` event adds nothing. Missing error codes display as
unknown. `poll_failed` is generated by the core rather than the hosted stream.

## Errors and compatibility

```json
{"jsonrpc":"2.0","id":2,"error":{"code":-32001,"message":"A harness may not call cards.approve.","data":{"code":"FORBIDDEN_FOR_ROLE"}}}
```

Transport/shape errors use JSON-RPC codes `-32700` (parse), `-32600` (request), `-32601` (method),
`-32602` (params), `-32603` (internal). Role refusals use `-32001`, other refusals `-32000`.
Branch on `error.data.code`, not message text or the numeric code alone. Unknown methods are
`METHOD_NOT_FOUND`; a known but unavailable handler is `METHOD_NOT_AVAILABLE`. Details are
code-specific, e.g. `core/client` on protocol mismatch, `state` on a closed card,
`step/differsAt/triesLeft/withdrawn` on transfer confirmation, or `turnId` on an active Quant turn.
Tool-level refusals can instead be inside `ToolResult` as described above.

Protocol majors change for incompatible existing calls/events; minor versions add fields, methods
or topics. Retain unknown fields when relaying, ignore unknown event kinds when rendering, and
never infer approval or a successful transaction from an event you do not understand.
