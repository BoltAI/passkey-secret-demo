# Passkey Secret Demo

A tiny demo of protecting browser-stored secrets with passkeys.

[Live Demo](https://passkey-demo.boltai.com) • [Source Code](https://github.com/BoltAI/passkey-secret-demo) • Made by [@daniel_nguyenx](https://x.com/daniel_nguyenx) for [BoltAI](https://boltai.com)

This is a small, browser-first demo for one specific idea:

- encrypt a user secret locally in the browser
- store only ciphertext in browser storage
- use a WebAuthn passkey as a local unwrap mechanism
- keep a recovery passphrase as fallback

This repo is not a package, password manager, or auth product. It is a focused demo and OSS reference for discussing one security pattern clearly.

## Why This Exists

BYOK flows in web apps are awkward.

If a product asks users to paste an API key into a browser app, the obvious thing to do is store it in `localStorage` or IndexedDB. That is simple, but it also means the secret is sitting there as plaintext.

This demo explores a narrower, more defensible approach:

- generate a random data-encryption key locally
- encrypt the secret with that key
- persist only ciphertext
- wrap that key with a recovery passphrase
- optionally wrap the same key with a passkey-derived PRF output

The result is still "just a browser app," but the stored value is encrypted rather than a raw API key.

## What This Demo Claims

Allowed claim:

- this protects browser-stored secrets at rest better than plaintext `localStorage` or IndexedDB

Also true:

- storage exfiltration is less useful if the attacker gets ciphertext instead of plaintext
- passkey unlock adds a user-mediated local unwrap step
- the demo is fully local and does not require a backend

What this demo does not claim:

- it does not prevent XSS
- it does not make browser memory safe
- it does not solve browser security
- it is not a production WebAuthn login system

A sufficiently capable same-origin XSS can still compromise a live session or decrypted in-memory data. This demo is mainly an encrypted-at-rest and user-presence improvement over plaintext browser storage.

## Current Demo UX

The current app is intentionally simple and chat-like.

1. Open the page.
2. Type a message and press `Send`.
3. If no local setup exists yet, a setup modal asks for:
   - a fake API key
   - a recovery passphrase
4. On submit, the app:
   - encrypts the API key locally
   - stores the encrypted envelope in `localStorage`
   - registers a passkey wrap for this browser
5. Refresh the page.
6. Send another message.
7. The app opens an unlock modal where you can choose:
   - `Unlock with passkey`
   - `Use passphrase instead`
8. After unlock, the assistant reply shows:
   - the plaintext API key currently held in memory
   - the exact encrypted JSON envelope stored locally

The point of the demo is to make the data flow inspectable in under a minute.

## How It Works

There is one stored envelope in `localStorage`.

- `ciphertext`
  The encrypted API key.
- `ciphertextIv`
  The AES-GCM IV for the encrypted payload.
- `recoveryWrap`
  An AES-GCM-wrapped copy of the DEK, where the wrapping key is derived from the recovery passphrase using `scrypt`.
- `passkeyWrap`
  An AES-GCM-wrapped copy of the same DEK, where the wrapping key is derived from a WebAuthn PRF result using HKDF.

There is also transient in-memory session state.

- decrypted plaintext is held only after a successful unlock
- refreshing the page clears that in-memory plaintext
- the stored envelope remains ciphertext at rest

## Technical Notes

Current crypto shape:

- payload encryption: `AES-GCM`
- recovery KDF: `scrypt`
- current scrypt params:
  - `N = 32768`
  - `r = 8`
  - `p = 1`
  - `dkLen = 32`
- passkey unwrap source: WebAuthn PRF extension
- passkey wrap derivation: `HKDF-SHA-256`

Relevant files:

- [src/lib/crypto.ts](src/lib/crypto.ts)
- [src/lib/passkey.ts](src/lib/passkey.ts)
- [src/lib/types.ts](src/lib/types.ts)
- [src/main.ts](src/main.ts)

## Why The Design Looks Like This

This repo is deliberately narrow.

It is trying to support this story:

1. plaintext browser storage is a weak default for BYOK secrets
2. passkey unlock can improve the local unwrap story
3. recovery still matters as fallback
4. the stored data should be visibly ciphertext, not marketing copy about security

That is why the code is explicit, the UI is small, and the app shows the exact stored JSON envelope instead of hiding it behind abstractions.

## Browser Support

This demo requires a secure context for passkeys:

- `http://localhost`
- `https://...`

Best results today are in a recent Chromium-based browser with WebAuthn PRF support.

Important caveats:

- passkey support exists before PRF support does
- some browsers support WebAuthn but not the PRF extension needed for this flow
- if PRF is unavailable, the recovery passphrase flow still demonstrates encrypted-at-rest storage, but passkey wrap registration/unlock will not work

## Local Development

Install and run:

```bash
npm install
npm run dev
```

Open [http://localhost:5173](http://localhost:5173).

Other commands:

```bash
npm test
npm run build
npm run preview
```

## What To Look At During The Demo

When the app is working correctly, you should be able to verify:

- the secret entered during setup is not stored as plaintext
- the stored envelope contains ciphertext plus wrap metadata
- passkey unlock requires explicit user interaction
- recovery passphrase can be used as fallback
- decrypted plaintext only appears after unlock and is held in memory

## Non-Goals

This repo is not trying to become:

- a full password manager
- a synced secret vault
- a backend-dependent auth system
- a general-purpose crypto SDK

If something does not help explain, demonstrate, or audit this one pattern, it is probably out of scope.

## Source

GitHub: [BoltAI/passkey-secret-demo](https://github.com/BoltAI/passkey-secret-demo)

## License

[MIT](LICENSE)
