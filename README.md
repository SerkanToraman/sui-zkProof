# zkLogin on Sui: A Beginner’s Guide

## 1. Overview

zkLogin allows users to authenticate with a familiar Web2 identity (like Google) and act on the Sui blockchain **without exposing that identity publicly**. It combines:
- A Web2 login (OAuth/OpenID),
- A secret (salt),
- A short-lived session binding (ephemeral key + nonce),
- And a zero-knowledge proof that ties everything together securely and privately.

Note: Visit https://sui-zklogin.vercel.app/ for testing.
---

## 2. Core Components (What They Are, Why They Exist, Failure Modes, Lifetimes)

### OAuth JWT (ID Token)
- Definition / Role: Token from an identity provider (e.g., Google) asserting the user logged in; contains iss, aud, sub, nonce, expiration, etc.
- Why it’s needed: Proves the user’s Web2 identity and that they recently authenticated.
- Failure / Risk if missing or invalid: No proof of identity; expired, forged, or mismatched token causes authentication to fail.
- Lifetime: Short-lived.

### Salt
- Definition / Role: Secret random value combined with identity fields (iss, aud, sub) to derive the on-chain zkLogin address.
- Why it’s needed: Breaks direct one-to-one mapping between Web2 identity and on-chain address (privacy) and provides a second secret factor.
- Failure / Risk: Changing it yields a different address (loses continuity); leaking it together with OAuth control enables impersonation; losing it makes the original address unrecoverable.
- Lifetime: Persistent (if reused consistently).

### zkLogin Address
- Definition / Role: The derived Sui account address computed deterministically from (issuer, audience, sub, salt).
- Why it’s needed: The stable on-chain identity the user controls via zkLogin.
- Failure / Risk: Any divergence in inputs (e.g., different salt) produces a different address; derivation is one-way.
- Lifetime: Deterministic given identical inputs.

### Ephemeral Key Pair
- Definition / Role: A short-lived key pair generated per session, used to sign the transaction.
- Why it’s needed: Binds the transaction submission to the specific authenticated session and limits blast radius if compromised.
- Failure / Risk: Without it, you cannot prove the session’s action; if instead long-lived, compromise yields persistent control.
- Lifetime: Short-lived / session-scoped.

### Nonce
- Definition / Role: A hash that binds the ephemeral public key, expiry (maxEpoch), and randomness into the OAuth login request.
- Why it’s needed: Ensures the returned JWT is session-specific and tied to the ephemeral key, preventing replay.
- Failure / Risk: Broken session linkage or replay attacks if missing/malformed; proof cannot validate.
- Lifetime: Session-scoped (short-lived).

### Extended Ephemeral Public Key
- Definition / Role: The ephemeral public key transformed into the numeric format the zero-knowledge circuit consumes.
- Why it’s needed: Allows the proving system to incorporate the ephemeral key into the zk proof.
- Failure / Risk: If not extended correctly, the proof cannot bind the session key and will fail.
- Lifetime: Session-scoped.

### Zero-Knowledge Proof (PartialZkLoginSignature)
- Definition / Role: Cryptographic proof plus public inputs that attest the JWT, salt, nonce, and ephemeral key are properly linked without revealing sensitive data.
- Why it’s needed: Lets the blockchain trust the composite login/session identity while preserving privacy.
- Failure / Risk: If invalid or absent, validators reject the transaction; forged or inconsistent proofs fail verification.
- Lifetime: Short-lived (tied to current login/session).

### Final zkLogin Signature
- Definition / Role: The assembled signature combining the proof, address seed, ephemeral signature, and metadata (e.g., maxEpoch).
- Why it’s needed: This is what is submitted to Sui to authenticate and authorize the transaction.
- Failure / Risk: Any inconsistency (wrong seed, stale proof, invalid ephemeral signature) causes rejection.
- Lifetime: Per transaction / per session usage.

---

## 3. Step-by-Step Flow

### Step 1: Generate Ephemeral Key Pair
- Create a short-lived key pair for this login session.
- Choose a validity window (maxEpoch, e.g., current epoch + 10) and generate fresh randomness.
- These will be ingredients for the nonce that binds the session.

### Step 2: Fetch JWT from OpenID Provider
Construct the OpenID/OAuth login request with:
1. $CLIENT_ID – your registered client ID.
2. $REDIRECT_URL – configured redirect callback in the provider.
3. $NONCE – generated from the ephemeral public key, maxEpoch, and randomness.

Example nonce generation:
import { generateNonce } from "@mysten/zklogin";

const nonce = generateNonce(
  ephemeralKeyPair.getPublicKey(),
  maxEpoch,
  randomness
);
Perform the login (e.g., redirect to Google with response_type=id_token, scope=openid, and the nonce). On success, you receive an id_token (JWT) in the redirect.

### Step 3: Decode JWT
Decode the ID token to extract claims:
- iss (issuer),
- aud (audience/client ID),
- sub (user identifier),
- nonce (ensures session binding),
- iat, exp (timestamps),
- jti (token ID), etc.

These values give you the stable identity tuple and confirm the login session.

### Step 4: Generate / Obtain User Salt
The salt is a secret that, together with iss, aud, and sub, deterministically defines the zkLogin Sui address.

Example salt:
164969034301553664818867886335076681282

Storage options:
- User-managed: Show once so the user backs it up.
- Client-side: Persist securely (e.g., encrypted storage).
- Backend: Store encrypted and release only after validating the JWT.

### Step 5: Derive the zkLogin Sui Address
Compute the on-chain address from the stable identity and salt:
import { jwtToAddress } from "@mysten/zklogin";

const zkLoginUserAddress = jwtToAddress(id_token, userSalt);
This address remains consistent across sessions if iss/aud/sub and salt stay the same.

### Step 6: Fetch Zero-Knowledge Proof (Groth16)
#### a. Extended Ephemeral Public Key
Prepare the session key for the proof:
import { getExtendedEphemeralPublicKey } from "@mysten/zklogin";

const extendedEphemeralPublicKey = getExtendedEphemeralPublicKey(
  ephemeralKeyPair.getPublicKey()
);

#### b. Request Proof
Send the necessary inputs to the proving service:
const zkProofResult = await axios.post(
  "https://prover-dev.mystenlabs.com/v1",
  {
    jwt: id_token,
    extendedEphemeralPublicKey,
    maxEpoch,
    jwtRandomness: randomness,
    salt: userSalt,
    keyClaimName: "sub",
  },
  { headers: { "Content-Type": "application/json" } }
).then(res => res.data);

const partialZkLoginSignature = zkProofResult;

This returns the PartialZkLoginSignature containing the proof and public inputs tying together login, salt, and ephemeral key.

### Step 7: Assemble zkLogin Signature and Submit Transaction
- Build the transaction block (e.g., transfer 1 SUI) and set sender to zkLoginUserAddress.
- Sign the transaction with the ephemeral key to get userSignature.
- Derive the address seed from userSalt, "sub", decodedJwt.sub, and decodedJwt.aud.
- Combine everything into the final zkLogin signature:
const zkLoginSignature = getZkLoginSignature({
  inputs: {
    ...partialZkLoginSignature,
    addressSeed,
  },
  maxEpoch,
  userSignature,
});
- Execute the transaction:
suiClient.executeTransactionBlock({
  transactionBlock: bytes,
  signature: zkLoginSignature,
});
Note: Ensure the zkLogin account has a small SUI balance to pay gas before invoking.

---

## 4. Security Properties

- Two-factor-like control: Both the OAuth proof and the secret salt are required to control the address.
- Session binding: Ephemeral key + nonce prevent replay attacks and tie actions to a specific login session.
- Privacy: Identity details (sub, etc.) aren’t exposed on-chain; only their validity is proven via the zk proof.
- Consistent address: Reusing the same salt and identity yields the same address; changing the salt produces a new, unlinkable address.
- Limited exposure: Ephemeral session artifacts expire (by epoch), bounding risk if compromised.

---

## 5. Salt Storage Guidance

Backend (recommended for UX):
- Encrypt the salt at rest (e.g., envelope encryption with a KMS).
- Only retrieve/use it after validating a fresh OAuth JWT.
- Monitor access patterns, rate-limit, and alert on anomalies.

Client-side:
- Store in secure local storage (e.g., encrypted IndexedDB or mobile secure store).
- Provide clear instructions for backup; losing the salt loses recoverability of the address.

User-managed:
- Display as a one-time recovery secret. If lost and no backup exists, the original address cannot be reconstructed.

---

## 6. Improved Analogy: Secure Courier Pickup System

Imagine a high-security courier locker where users retrieve sensitive packages. Access requires multiple pieces that collectively prove authorization, but the system never publicly exposes the user’s identity or full secret.

- OAuth login (JWT): The verified digital pickup request—proves the user is allowed to collect a package.
- Salt: The secret pickup code (like a PIN); combining it with the request determines the specific locker (the zkLogin address).
- zkLogin Address: The locker number derived from request + secret.
- Ephemeral Key Pair: A temporary access badge issued for that visit, proving the person using the locker is the one who initiated the pickup.
- Nonce: The session ticket embedding badge details and expiry into the request—binding everything to a single visit.
- Extended Ephemeral Public Key: The badge’s internals transformed for the internal verification engine.
- Zero-Knowledge Proof: Sealed confirmation that the request, code, and badge align for that locker—without revealing the code itself.
- Final zkLogin Signature: The combination of the confirmation, the live badge signature, and locker identity that opens the locker securely.

If any piece is missing, mismatched, or expired, the locker stays locked. Observers only see valid access—they don’t see the secret code or internal identity.

---

## 7. Summary

To control a zkLogin identity and successfully submit a transaction you need:
- A persistent secret (salt) to define and stabilize the address.
- A fresh login session (OAuth JWT + nonce + ephemeral key) to prove current authorization.
- A zero-knowledge proof that binds identity, salt, and session without leaking private data.
- A signed transaction using the ephemeral key.
- A composite signature combining all pieces, submitted to Sui.

Any inconsistency, expiration, or missing piece causes the transaction to be rejected.
