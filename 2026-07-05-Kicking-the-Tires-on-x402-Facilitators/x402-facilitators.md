# Kicking the tires on x402 facilitators

In my [last post](../2026-07-05-Shipping-Real-Payments-over-HTTP-402/x402.md) I called the facilitator the piece that makes x402 practical: it verifies the payment signature, submits the transfer on-chain, and eats the gas, so neither the seller nor the payer touches a blockchain directly. It's also the one centralized piece in an otherwise open protocol, which makes it worth choosing carefully. So I wired up the smallest possible seller — an Express server with one paywalled route, a throwaway testnet wallet, $0.01 a request — and ran it against PayAI and thirdweb. Then, because the real promise of an open protocol is not needing any of these companies at all, I ran a facilitator myself with x402.rs. This is what actually happened.

## PayAI: the five-minute facilitator

PayAI's pitch is "no API keys, one env var," and that held up. The whole server is the foundation's `@x402/express` middleware pointed at PayAI's URL:

```js
const facilitator = new HTTPFacilitatorClient({ url: 'https://facilitator.payai.network' });

app.use(paymentMiddleware(
  { 'GET /premium': { accepts: [{ scheme: 'exact', price: '$0.01', network: 'eip155:84532', payTo }] } },
  new x402ResourceServer(facilitator).register('eip155:84532', new ExactEvmScheme()),
));
```

Curl it with no payment and you get a clean v2 challenge: 402 status, `{}` body, and a `PAYMENT-REQUIRED` header that decodes to the `accepts` array with the Base Sepolia USDC contract. PayAI's `/supported` endpoint lists both v1 and v2 kinds for every chain it covers, so older v1 clients keep settling against the same URL. Point `@x402/fetch` at it with an unfunded wallet and the facilitator does exactly what it should:

```
"error": "invalid_exact_evm_insufficient_balance"
```

though that line took more digging to see than it used to: a v2 rejection re-issues the challenge with the reason in the header's `error` field, and the body stays `{}`. To produce the error at all it had to recover the payer from the signature and check the balance on-chain, before ever attempting settlement. After funding the wallet with faucet USDC, the same request came back `200` with the content and a settlement receipt in the `PAYMENT-RESPONSE` header:

```json
{ "success": true,
  "transaction": "0x363e9ad6ca0975d7e2abc2643c6f2c1c967e9a4b4f6404c4886f52906397a4e1",
  "network": "base-sepolia",
  "payer": "0x4b73e594edfaf51ec43f446cc741450033d4bc7f" }
```

That's a real transaction hash. One small tell in it: I paid on `eip155:84532` and the receipt names the network `base-sepolia` — a v1 name leaking through a v2 receipt. I pulled the receipt from Base Sepolia and the submitting address is PayAI's gas wallet, not mine; my wallet held USDC and zero ETH the whole time. So the gasless thing isn't marketing. The free tier is 10,000 settlements a month, they cover the network fees, and you don't even need an account for basic verify-and-settle. Hard to argue with.

## thirdweb: more capable, sharper edges

thirdweb's facilitator needs a project secret key and a server wallet from their dashboard, and their `settlePayment` API inverts the usual shape. There's no middleware; you call it inside your handler and it tells you whether to serve the content or bounce a 402 back:

```js
const result = await settlePayment({
  resourceUrl: 'http://localhost:4021/premium',
  method: 'GET',
  paymentData: req.header('payment-signature') ?? req.header('x-payment'),
  network: arbitrumSepolia,
  price: '$0.01',
  facilitator: thirdwebX402Facilitator,
});
```

Note the header line, because this is where I lost an hour. thirdweb uses the **v2 transport**: the 402 challenge comes back base64-encoded in a `PAYMENT-REQUIRED` header with an empty body, and the client sends its payment in `PAYMENT-SIGNATURE`, not `X-PAYMENT`. The snippet in their own docs reads `request.headers.get("x-payment")`, which happily returns null when their own v2 client calls you, and `settlePayment` just re-issues the challenge. No error, no hint. Accept both headers and it works.

Two more tripwires, and the first one surprised me: the foundation's own v2 client can't pay a thirdweb seller either. thirdweb's transport is v2, but the requirement objects inside those headers are still v1-shaped — `maxAmountRequired` where the v2 spec says `amount`, resource fields inlined into each entry — so `@x402/fetch` dies mid-signing with `Cannot convert undefined to a BigInt`. It cuts the other way too: point thirdweb's client at a spec-shaped v2 seller and it bounces the challenge with a zod validation error demanding `maxAmountRequired`. Two SDKs, both calling themselves v2, unable to pay each other; for now you pair thirdweb with thirdweb and use their `wrapFetchWithPayment`. And that wrapper takes a `Wallet`, not an `Account`, so a headless private-key setup needs `createWalletAdapter`. At least the error when you get that wrong (`wallet.getAccount is not a function`) says what it means.

Once through all that, the unfunded-wallet test came back with the most helpful rejection of the day:

```
"error": "insufficient_funds",
"errorMessage": "Client does not have enough funds. has 0 but required 10000.
  Use the following link to top up your wallet: https://thirdweb.com/pay?chain=421614&..."
```

A funding link, in the error, addressed to the payer. For agent-facing APIs that's a nice touch; the agent can hand the link to whoever holds the purse.

Funded, it settled: `200`, content served, one cent of USDC moved on Arbitrum Sepolia. But look closely at the receipt in the `PAYMENT-RESPONSE` header:

```json
{ "success": true,
  "transaction": "0ed308b2-35e3-46d2-b869-552c519415cf",
  "network": "eip155:421614", ... }
```

That `transaction` is not a transaction hash. It's a UUID, thirdweb's internal queue ID for the transaction their infrastructure submitted. PayAI handed me the on-chain hash; thirdweb hands me a reference into their system. I confirmed the USDC actually moved by checking balances directly, but if your bookkeeping wants a chain-verifiable receipt at settlement time, that's an extra API round trip against their Engine. It's the kind of thing you only find out by running it.

## x402.rs: running it yourself

The facilitator sees every payment and can, in principle, decline to process one, and every option above is somebody else's service. But x402 is just a spec, and [x402.rs](https://github.com/x402-rs/x402-rs) is an open-source Rust implementation you can run yourself. Their hosted mainnet facilitator has a waitlist; the code doesn't. I stood one up locally against Base Sepolia to see how much ceremony "run your own" actually involves.

Not much, it turns out. One config file and one container:

```json
{
  "port": 8080,
  "host": "0.0.0.0",
  "chains": {
    "eip155:84532": {
      "eip1559": true,
      "signers": ["$FACILITATOR_PRIVATE_KEY"],
      "rpc": [{ "http": "https://sepolia.base.org", "rate_limit": 100 }]
    }
  },
  "schemes": [
    { "id": "v2-eip155-exact", "scheme": "v2-eip155-exact", "chains": "eip155:84532" }
  ]
}
```

```bash
docker run -d -p 8402:8080 \
  -e FACILITATOR_PRIVATE_KEY="$YOUR_SIGNER_KEY" \
  -v "$(pwd)/config.json:/app/config.json" \
  ghcr.io/x402-rs/x402-facilitator:latest
```

One honest caveat: the config format in the project README didn't match what the published image accepts. The README shows scheme entries as `{"scheme": "...", "chains": ["..."]}`; the `:latest` image wants an `id` field and a plain string for `chains`, and tells you so one field at a time through parse errors. That's life a quarter ahead of a v1.0 spec freeze. Pin your image version and keep your config next to it.

Once up, it announces itself properly:

```
Registered scheme handler chain_id=eip155:84532 scheme=exact id="v2-eip155-exact"
Starting server at http://0.0.0.0:8080
```

and `/supported` returns the same shape the hosted facilitators serve:

```json
{
  "kinds": [{ "x402Version": 2, "scheme": "exact", "network": "eip155:84532", "extra": {} }],
  "signers": { "eip155:84532": ["0x4B73E594edFaf51eC43F446cC741450033D4Bc7F"] }
}
```

Note the `x402Version: 2`. x402.rs speaks the spec's v2 — the same shapes the `@x402/*` reference SDKs emit, not thirdweb's dialect of it.

The part nobody hosts for you: the signer. That `FACILITATOR_PRIVATE_KEY` wallet submits every `transferWithAuthorization` on-chain, so it needs native gas on every chain you settle, it needs monitoring, and it needs the same key hygiene as any hot wallet. This is exactly what you pay Coinbase or PayAI their $0.001 to worry about. Whether that trade is worth it comes down to volume and trust: past a million settlements a month the fees are real money, and if your product's story is "no intermediary can freeze payments," a facilitator you don't run is an intermediary.

## What I took away

The version split is not theoretical, and adopting v2 didn't close it. PayAI now answers v1 and v2 clients from the same URL and the reference SDKs are solid, yet a spec-conformant v2 client still can't pay a thirdweb seller, because the two sides disagree about what belongs inside the headers. Mixed pairs fail with empty bodies, nulls, and missing-field complaints rather than anything that says "version mismatch." Until that settles, match your client SDK to whatever your seller speaks — and if your settle API parses both formats (thirdweb's does), read both payment headers; it costs one line. The wire-format differences themselves are laid out in [the protocol post](../2026-07-05-Shipping-Real-Payments-over-HTTP-402/x402.md).

On picking one: PayAI is the path of least resistance, has the best free tier in the business, and gives you a proper on-chain receipt; for selling API access I'd start there and not feel like I was settling. thirdweb buys you 170+ chains, any permit-capable ERC-20, and polished failure modes, at the cost of an account, a client stack that currently only pairs with itself, and a settlement receipt that points into their infrastructure instead of the chain. Coinbase's CDP facilitator stays the pick when compliance screening on settlements matters, since KYT baked into the payment rail isn't something the others offer. And self-hosting is genuinely a config file and a container away, if you have the appetite to babysit one more hot wallet. An open payment protocol where self-hosting is theoretical isn't really open; this one checks out.

*Everything I ran here — the PayAI and thirdweb sellers and clients, a Go seller that works with any standard facilitator, and the working x402.rs config under `selfhost/` — is in [github.com/farid-moradi/x402-facilitator-demos](https://github.com/farid-moradi/x402-facilitator-demos).*
