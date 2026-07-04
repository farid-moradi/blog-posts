# Self-hosting an x402 facilitator

The facilitator is the one centralized piece in an otherwise open payment protocol: it verifies signatures and submits transfers on-chain, which means it sees every payment and can, in principle, decline to process them. Coinbase runs one, PayAI runs one, thirdweb runs one. But x402 is just a spec, and [x402.rs](https://github.com/x402-rs/x402-rs) is an open-source Rust implementation you can run yourself. Their hosted mainnet facilitator has a waitlist; the code doesn't. I stood one up locally against Base Sepolia to see how much ceremony "run your own" actually involves.

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

Note the `x402Version: 2`. Like thirdweb, x402.rs has moved to the v2 wire format, so pair it with v2-capable client SDKs or budget for the interop quirks I covered in the previous post.

The part nobody hosts for you: the signer. That `FACILITATOR_PRIVATE_KEY` wallet submits every `transferWithAuthorization` on-chain, so it needs native gas on every chain you settle, it needs monitoring, and it needs the same key hygiene as any hot wallet. This is exactly what you pay Coinbase or PayAI their $0.001 to worry about. Whether that trade is worth it comes down to volume and trust: past a million settlements a month the fees are real money, and if your product's story is "no intermediary can freeze payments," a facilitator you don't run is an intermediary.

For us, staying on a hosted facilitator is still the right call; the signer-ops burden lands on the same team that already runs treasury wallets, and they have enough hot keys to babysit. But I like knowing the exit exists, that it's a config file and a container, and that I've run it. An open payment protocol where self-hosting is theoretical isn't really open. This one checks out.

*The working config and run commands are in [github.com/farid-moradi/x402-facilitator-demos](https://github.com/farid-moradi/x402-facilitator-demos), under `selfhost/`.*
