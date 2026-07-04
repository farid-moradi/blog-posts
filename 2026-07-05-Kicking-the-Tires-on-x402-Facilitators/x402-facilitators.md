# Kicking the tires on x402 facilitators

In my [last post](../2026-07-05-Shipping-Real-Payments-over-HTTP-402/x402.md) I called the facilitator the piece that makes x402 practical: it verifies the payment signature, submits the transfer on-chain, and eats the gas, so neither the seller nor the payer touches a blockchain directly. It's also the one centralized piece in an otherwise open protocol, which makes it worth choosing carefully. So I wired up the smallest possible seller — an Express server with one paywalled route, a throwaway testnet wallet, $0.01 a request — and ran it against PayAI and thirdweb. Then, because the real promise of an open protocol is not needing any of these companies at all, I ran a facilitator myself with x402.rs. This is what actually happened.

## PayAI: the five-minute facilitator

PayAI's pitch is "no API keys, one env var," and that held up. The whole server is Coinbase's own `x402-express` middleware pointed at PayAI's URL:

```js
app.use(paymentMiddleware(
  payTo,
  { 'GET /premium': { price: '$0.01', network: 'base-sepolia' } },
  { url: 'https://facilitator.payai.network' },
));
```

Curl it with no payment and you get a clean x402 v1 challenge: 402 status, JSON body, `accepts` array with the Base Sepolia USDC contract. Point `x402-fetch` at it with an unfunded wallet and the facilitator does exactly what it should:

```
"error": "invalid_exact_evm_insufficient_balance",
"payer": "0x4B73E594edFaf51eC43F446cC741450033D4Bc7F"
```

It recovered the payer address from the signature and checked the balance on-chain before ever attempting settlement. After funding the wallet with faucet USDC, the same request came back `200` with the content and a settlement receipt in the `X-PAYMENT-RESPONSE` header:

```json
{ "success": true,
  "transaction": "0x71114fb4a5b9c939ed840d5031d526f0c742a3952482e421e222e0f3aa2c3e57",
  "network": "base-sepolia",
  "payer": "0x4b73e594edfaf51ec43f446cc741450033d4bc7f" }
```

That's a real transaction hash. I pulled the receipt from Base Sepolia and the submitting address is PayAI's gas wallet, not mine; my wallet held USDC and zero ETH the whole time. So the gasless thing isn't marketing. The free tier is 10,000 settlements a month, they cover the network fees, and you don't even need an account for basic verify-and-settle. Hard to argue with.

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

Note the header line, because this is where I lost an hour. thirdweb speaks **x402 v2**: the 402 challenge comes back base64-encoded in a `PAYMENT-REQUIRED` header with an empty body, CAIP-2 network IDs and all, and the client sends its payment in `PAYMENT-SIGNATURE`, not `X-PAYMENT`. The snippet in their own docs reads `request.headers.get("x-payment")`, which happily returns null when their own v2 client calls you, and `settlePayment` just re-issues the challenge. No error, no hint. Accept both headers and it works.

Two more v2 tripwires. Coinbase's `x402-fetch` client cannot talk to a thirdweb seller at all. It's v1-only, so it goes looking for the `accepts` array in the response body, finds `{}`, and dies; you need thirdweb's own `wrapFetchWithPayment`. And that wrapper takes a `Wallet`, not an `Account`, so a headless private-key setup needs `createWalletAdapter`. At least the error when you get that wrong (`wallet.getAccount is not a function`) says what it means.

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

Note the `x402Version: 2`. Like thirdweb, x402.rs speaks the v2 wire format, so pair it with v2-capable client SDKs or budget for the interop quirks above.

The part nobody hosts for you: the signer. That `FACILITATOR_PRIVATE_KEY` wallet submits every `transferWithAuthorization` on-chain, so it needs native gas on every chain you settle, it needs monitoring, and it needs the same key hygiene as any hot wallet. This is exactly what you pay Coinbase or PayAI their $0.001 to worry about. Whether that trade is worth it comes down to volume and trust: past a million settlements a month the fees are real money, and if your product's story is "no intermediary can freeze payments," a facilitator you don't run is an intermediary.

## What I took away

The v1/v2 split is not theoretical. Run one seller and one client from the same vendor and you'll never notice; mix vendors and things fail with empty bodies and nulls rather than errors. Until v2 is everywhere, accept both payment headers on the server and match your client SDK to whatever your seller speaks.

On picking one: PayAI is the path of least resistance, has the best free tier in the business, and gives you a proper on-chain receipt; for selling API access I'd start there and not feel like I was settling. thirdweb buys you 170+ chains, any permit-capable ERC-20, and polished failure modes, at the cost of an account, a v2-only client stack, and a settlement receipt that points into their infrastructure instead of the chain. Coinbase's CDP facilitator stays the pick when compliance screening on settlements matters, since KYT baked into the payment rail isn't something the others offer. And self-hosting is genuinely a config file and a container away, if you have the appetite to babysit one more hot wallet. An open payment protocol where self-hosting is theoretical isn't really open; this one checks out.

*Everything I ran here — the PayAI and thirdweb sellers and clients, a Go seller that works with any standard facilitator, and the working x402.rs config under `selfhost/` — is in [github.com/farid-moradi/x402-facilitator-demos](https://github.com/farid-moradi/x402-facilitator-demos).*
