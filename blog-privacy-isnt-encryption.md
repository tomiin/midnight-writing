# Privacy isn't encryption: what building on Midnight actually taught me

> Also posted on dev.to: https://dev.to/tminus1s/privacy-isnt-encryption-what-building-on-midnight-actually-taught-me-27j6

When I started building on Midnight, I had the wrong mental model, and I suspect
most people arriving from "normal" blockchains have it too. I assumed a privacy
chain meant my data would sit on-chain, encrypted, and only the right people could
decrypt it. A lockbox on a public ledger.

That model is wrong. And if you design around it, you'll build the wrong thing.
Here's what actually clicked for me after building a few contracts, including a
privacy-preserving health-records system.

## Encryption and zero-knowledge are different tools

They get lumped together, but they solve different problems.

Encryption hides bytes. You scramble data so only someone with the key can
read it. The catch: whoever has the key sees everything. And "encrypted, on a
public ledger, forever" is a liability, not a feature. Keys leak. Keys get lost.
And a ciphertext that sits on-chain for eternity is a standing bet that today's
encryption survives tomorrow's computers.

Zero-knowledge proves statements. It lets you show that something is true —
"I'm over 18," "this vote is valid," "I'm the assigned doctor" — without
revealing the underlying data at all. No key handed over, nothing decrypted. The
sensitive part never has to be shown to anyone.

So privacy on Midnight is not "encrypt the data and put it on-chain." It's almost
the opposite: keep the sensitive data off the ledger entirely, and put only
proofs and commitments on it.

## The real model: selective disclosure

The idea that runs through everything on Midnight is selective disclosure. You
choose exactly which facts become public, and everything else simply never touches
the chain. A couple of examples from things I built:

- A tip jar where the running total is public — so it's trustworthy — but each
  donor is recorded only as a one-way hash. The chain never learns who gave.
- A vote where the tally is public but the voter is a nullifier. Reveal the count,
  hide the identity.

Notice what's not happening: nothing is "encrypted and then decrypted later." The
private part is never on-chain in any form. That's the whole point.

## Where the misconception really bites: health records

I built an access-control contract for medical results, and it's the perfect place
to see the wrong model fail.

If you believe "privacy = encrypt on-chain," you'd try to store encrypted lab
results on the chain and build a key-management scheme around them. You'd spend all
your time on the vault. And you'd have put the most sensitive data humans produce
onto a public ledger forever.

Here's what I built instead:

- The result stays encrypted off-chain. The chain holds only a commitment
  — a one-way hash of it. Tamper-proof, but it reveals nothing.

```compact
// the lab posts only the HASH of the (off-chain, encrypted) result
resultHash.insert(disclose(recordId), disclose(resultCommitment));
```

- Identity never goes on-chain. A caller proves they hold a secret key — a private
  witness — and all the chain ever stores is a hash of it.

```compact
witness localSecretKey(): Bytes<32>;              // private, never disclosed

export pure circuit accountId(sk: Bytes<32>): Bytes<32> {
  return persistentHash<Vector<1, Bytes<32>>>([sk]);   // one-way
}
```

- The chain enforces the rules: only the assigned doctor with the patient's
  consent gets routine access; an emergency responder can break the glass, but
  every override is logged for audit. None of that needs the data — it needs
  proofs.

- And you can still prove a result is the genuine one, without the chain ever
  seeing it:

```compact
export circuit verifyResult(recordId: Uint<64>, candidate: Bytes<32>): Boolean {
  return resultHash.lookup(disclose(recordId)) == disclose(candidate);
}
```

The chain never sees your data. It enforces the rules about your data, and keeps
an honest audit trail. The data itself stays where sensitive data belongs — off
the public ledger.

## The design lesson

Once I stopped thinking "hide the bytes" and started thinking "prove the facts,"
the designs got simpler and better:

- Don't put sensitive data on-chain, even encrypted. Put proofs and commitments
  on-chain; keep data off-chain.
- Let the chain be the referee — rules, consent, audit — not the vault.
- Compact makes this explicit whether you like it or not: the `disclose()` keyword
  forces you to opt in every time you make something public. You can't accidentally
  leak private data; you have to choose to reveal it. That constraint is a gift.

## The honest boundary

Midnight isn't magic, and pretending otherwise is how you ship a demo that
overpromises. The encryption and key management for the actual data still happen
off-chain with ordinary cryptography. What Midnight adds is the layer on top: the
proofs, the rules, and the audit. Being clear about that line is the difference
between a real design and a pitch.

## The takeaway

Privacy on Midnight isn't a lockbox on the chain. It's proving the truth while
keeping the data to yourself. Once that clicked, I stopped trying to hide bytes and
started asking a better question: which facts actually need to be public? Answer
that, put only those on-chain as proofs, and let everything else stay private by
never being there in the first place.
