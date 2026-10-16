# NevDEX Developer Hub: Meme Orderbook Sprint API ⚔️

*On Robinhood Chain. Built for bots. Stream the book, place signed orders, earn liquidity points.*

Three workflows to get started: **read depth → build/sign/place → track points**.
This guide documents the implemented production API. Order acceptance, matching,
and on-chain settlement are separate steps; settlement latency is variable.

| Connection | Value |
| --- | --- |
| REST origin | `https://nevdex.xyz` |
| Public market WebSocket | `wss://nevdex.xyz/api/v1/ws?marketID=CASHCAT-USDG` |
| Account orders WebSocket | `wss://nevdex.xyz/api/v1/orders/ws?marketID=CASHCAT-USDG&address=YOUR_ADDRESS&openOnly=1` |
| Chain ID | `4663` |
| JSON-RPC | `https://rpc.mainnet.chain.robinhood.com` |

Market identifiers are case-sensitive, for example `CASHCAT-USDG`, `AI-USDG`,
`PONS-USDG`, and `Kudo-USDG`. Use EVM wallet addresses.

## Minute 1 — Get orderbook depth

**`POST /api/order/orderbook`**

```bash
curl --fail-with-body -sS https://nevdex.xyz/api/order/orderbook \
  -H 'Content-Type: application/json' \
  -d '{"marketID":"CASHCAT-USDG"}'
```

Illustrative response; prices and sequence are examples:

```json
{
  "code": 20000,
  "data": {
    "sequence": 1234,
    "bids": [["0.1199", "10"], ["0.1198", "20"]],
    "asks": [["0.1201", "15"], ["0.1202", "25"]]
  }
}
```

Each level is `[price, remaining base-token amount]`, aggregated at that price.
`bids` are highest-first; `asks` are lowest-first. Best bid is `bids[0][0]`,
best ask is `asks[0][0]`. Either side may be empty; missing sides can also be
`null` when no snapshot is available. Do not interpret an unavailable book as a
reliable zero-price market. Prices are in USDG per base token.

Use decimal arithmetic for prices, amounts and points. Fetch precision, minimum
size and fee rates from market metadata instead of hardcoding them:

```bash
curl --fail-with-body -sS https://nevdex.xyz/api/getMarket \
  -H 'Content-Type: application/json' \
  -d '{"marketID":"CASHCAT-USDG"}'
```

Read `data.priceDecimals`, `amountDecimals`, `minOrderSize`, `baseTokenDecimals`,
`quoteTokenDecimals`, `makerFeeRate` and `takerFeeRate`. Token decimals describe
on-chain integer units; market decimals describe permitted order precision.
REST order amounts use human-readable decimal strings, not token integers.

For continuous updates, subscribe to the public WebSocket described below.

## Minute 2 — Place a limit order

**`POST /api/order/build` → local wallet signature → `POST /api/order/place`**

Before placing orders, connect an EOA wallet on chain `4663`, approve the relevant
token to the Vault, and deposit trading funds through the app's Deposit dialog.
Wallet token holdings alone are not trading account funds. Buy orders need
available quote funds including fees; sell orders need available base funds.
Existing orders lock funds across markets.

### Build

```bash
# Replace the address, price and amount with your intended order.
curl --fail-with-body -sS https://nevdex.xyz/api/order/build \
  -H 'Content-Type: application/json' \
  -d '{
    "address":"YOUR_EVM_ADDRESS",
    "marketID":"CASHCAT-USDG",
    "side":"buy",
    "orderType":"limit",
    "postOnly":true,
    "price":"0.1000",
    "amount":"0.1000",
    "expires":120
  }'
```

| Field | Meaning |
| --- | --- |
| `address` | EOA that owns the funds and signs the order |
| `marketID` | Published market identifier |
| `side` | `buy` or `sell` |
| `orderType` | `limit` for this workflow |
| `postOnly` | `true` prevents taking liquidity, including if the book changes before processing |
| `price` | Positive decimal string, in quote tokens per base token |
| `amount` | Positive decimal string, in base tokens for limit orders |
| `expires` | Order lifetime in seconds (1–86400), not a Unix timestamp; omitted/out-of-range values default to 3600 |
| `referrer` | Optional referring EVM address for eligible first-time users |

Success returns `code: 20000` and `data.id`, `marketID`, `side`, `type`, `price`,
`amount`, `json`, `asMakerFeeRate`, `asTakerFeeRate`, `makerRebateRate` and
`gasFeeAmount`. `json` contains the contract order's token addresses, integer
amounts, relayer and packed data. Inspect these against your intended order before
signing. The unsigned build cache lasts **60 seconds**, independently of the
signed order lifetime. Build a fresh order if that cache expires.

### Sign and submit

Sign the **32 raw bytes** of `data.id` using EIP-191 personal signing. Signing the
literal 66-character hex text produces a different signature. The API expects a
**96-byte** signature, not the wallet's ordinary 65-byte output:

```text
32-byte config | 32-byte r | 32-byte s
config[0] = v (27 or 28)
config[1] = 0 (personal signing)
config[2..31] = 0
```

The following browser-console example runs on `https://nevdex.xyz` with an
injected EVM wallet. **Running it requests a wallet signature and submits a real
order.** Set your intended price and amount first; example prices are not live
quotes. EIP-1271/6492 contract-wallet signatures are not supported by this flow.

```javascript
const intent = {
  marketID: "CASHCAT-USDG", side: "buy", orderType: "limit",
  postOnly: true, price: "0.1000", amount: "0.1000", expires: 120,
};
async function post(path, body) {
  const res = await fetch(path, {
    method: "POST", headers: { "Content-Type": "application/json" },
    body: JSON.stringify(body),
  });
  const result = await res.json();
  if (!res.ok || result.code !== 20000)
    throw new Error(result.desc || result.msg || `HTTP ${res.status}`);
  return result.data;
}
if (Number(await ethereum.request({ method: "eth_chainId" })) !== 4663)
  throw new Error("Switch to Robinhood Chain (4663)");
const [address] = await ethereum.request({ method: "eth_requestAccounts" });
const order = await post("/api/order/build", { ...intent, address });
if (!/^0x[0-9a-f]{64}$/i.test(order.id)) throw new Error("Invalid order ID");
console.log("Review built order:", order);
if (!confirm("Sign and submit the displayed order?")) throw new Error("Canceled");
const raw = await ethereum.request({
  method: "personal_sign", params: [order.id, address],
});
if (!/^0x[0-9a-f]{130}$/i.test(raw)) throw new Error("Unsupported signature");
let v = parseInt(raw.slice(-2), 16);
if (v < 2) v += 27;
if (v !== 27 && v !== 28) throw new Error("Invalid recovery ID");
const config = v.toString(16).padStart(2, "0") + "00".repeat(31);
const signature = "0x" + config + raw.slice(2, 130);
await post("/api/order/place", { address, orderID: order.id, signature });
console.log("Order queued:", order.id);
```

For Node.js bots, the equivalent signing operation in ethers v6 is
`wallet.signMessage(getBytes(order.id))`; unpack `r`, `s`, `v` with
`Signature.from(...)` and use the same 96-byte format. See the official
[ethers signer documentation](https://docs.ethers.org/v6/api/providers/#Signer-signMessage).
Never send a private key to the API. A complete Go implementation is available
in the repository's [post-only maker bot](../bot/README.md).

A place response with `code: 20000` acknowledges queue submission. A crossing
post-only order can be rejected by the engine after this response. Observe the
account stream for persisted order state; successful matching still needs
on-chain settlement. After a network timeout, reconcile the original order ID
before building a replacement. Retrying a consumed build does not enqueue it twice.

## Minute 3 — Fetch live points and rank

**`GET /api/v1/leaderboard?sort=total&address=YOUR_EVM_ADDRESS`**

```bash
curl --fail-with-body -sS \
  'https://nevdex.xyz/api/v1/leaderboard?sort=total&address=YOUR_EVM_ADDRESS'
```

The response is `{ "code": 20000, "data": { ... } }`. Relevant fields:

| Field under `data` | Meaning |
| --- | --- |
| `scope` | `"global"`: points aggregated across all markets |
| `quoteToken` | `"USDG"` |
| `me.makerPoints` | This wallet's resting-liquidity points |
| `me.takerPoints` | This wallet's settled taker points |
| `me.referralPoints` | This wallet's referral rewards |
| `me.totalPoints` | Maker + Taker + Referral for this wallet |
| `me.rank` | Global rank for the requested sort; `null` without positive ranked points |
| `entries` | Top 100 wallets for the requested sort |
| `stats` | Whole-site totals, not your wallet's points |
| `startedAt`, `updatedAt` | Season start and response-generation time, Unix milliseconds |

`sort` accepts `total`, `maker`, `taker` or `referral`. Omit `address` for public
rankings only (`me` is then `null`). Supplying an address returns its personal
totals and global rank even when it is outside the top 100. Points and volumes
are decimal strings. `updatedAt` is not a last-settlement timestamp.

## Stream updates instead of polling

In the browser console on the app origin:

```javascript
const marketSocket = new WebSocket(
  "wss://nevdex.xyz/api/v1/ws?marketID=CASHCAT-USDG"
);
marketSocket.onmessage = ({ data }) => {
  const event = JSON.parse(data);
  if (event.type === "depth_update") {
    // Full replacement snapshot, at most 20 levels per side; not a delta.
    const { bids, asks, sequence } = event.data;
    console.log({ bestBid: bids?.[0], bestAsk: asks?.[0], sequence });
  }
  if (event.type === "leaderboard_update") {
    // Invalidation notice: coalesce concurrent notices, then fetch the global
    // leaderboard once. This event does not contain the wallet's score.
    console.log("Points changed:", event.marketID, event.data.updatedAt);
  }
};
```

The event envelope is `{type, marketID, data}`. Public streams also carry
`ticker_update`, `trade_match` and `account_update` events. A cached depth
snapshot is sent when available; fetch the REST snapshot if you need to bootstrap
before the first streamed depth event. A newly connected stream also sends ticker
data. Keep the latest streamed full book rather than appending snapshots.

Open one public connection for each market contributing to your global points;
`leaderboard_update` is market-scoped. Fetch the leaderboard on initial connection,
on these invalidation notices, and after reconnect. Coalesce requests and serialize
refreshes to avoid older responses overwriting newer state. There is no HTTP
timer required. If using only CASHCAT, the connection above covers that market.

For order state, connect to
`/api/v1/orders/ws?marketID=CASHCAT-USDG&address=YOUR_ADDRESS&openOnly=1`.
`orders_snapshot` events contain `data.address`, `version`, `openOrder`, and an
optional `blockNumber`. These are full, DB-backed snapshots of pending/settling
orders. Omit `openOnly=1` to include recent trade history. `version` is monotonic
within one connection and resets on reconnect. Open order records use capitalized
fields such as `Id`, `Side`, `Status`, `AvailableAmount` and `PendingAmount`.

Servers send WebSocket ping control frames every 20 seconds; libraries should
respond with pong (browsers do this automatically). Slow clients are disconnected;
reconnect with backoff and replace local state from the new snapshot. Pause trading
while required market/account streams are disconnected or stale.

## Cancel an order

**`POST /api/order/cancel`**

```bash
curl --fail-with-body -sS https://nevdex.xyz/api/order/cancel \
  -H 'Content-Type: application/json' \
  -d '{"address":"YOUR_EVM_ADDRESS","marketID":"CASHCAT-USDG","orderID":"YOUR_ORDER_ID"}'
```

Cancellation is asynchronous. It releases only unmatched quantity; already
matched quantity can still settle. Observe the account snapshot before reusing
funds. The current cancel endpoint requires no wallet signature and does not
authenticate the caller; matching the supplied address to the order is not proof
of caller ownership.

## Responses and limits

- Always check both HTTP status and JSON `code`. Business errors can use HTTP 200;
  **only `code: 20000` indicates success**. Errors may include `desc`, `msg`, or an
  HTTP error `message`. Do not assume every successful response has `data`.
- Build failures include unavailable funds, invalid precision, minimum size and
  open-order capacity. Read market configuration before choosing size and price.
  Reserve fees as well as principal; account for locks across markets.
- Ordinary accounts default to 10 pending/settling orders across markets, checked
  at build time. The operator can change this limit or explicitly allowlist bot
  wallets; clients cannot grant themselves an exemption. No public request-rate
  guarantee or latency SLA is advertised by this guide.
- Market orders are outside this limit-order quickstart. In particular, a market
  **buy** uses quote-token spend for `amount`, rather than base-token size.
- `/api/v1/depth`, `/api/v1/order` and `/api/v1/user/stats` are **not implemented
  aliases**. Use the exact routes and `marketID` parameter documented here.

