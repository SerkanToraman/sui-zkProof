# zkLogin on Sui: Beginner-Friendly Summary

## 1. What is zkLogin?

zkLogin lets someone log in with a Web2 identity (like Google) and act on Sui **without exposing their identity on-chain**. It combines:
- A Web2 login (OAuth/OpenID),
- A secret (salt),
- A short-lived session (ephemeral key + nonce),
- A zero-knowledge proof that ties it all together.

> **Test it:** https://sui-zklogin.vercel.app/

---

## 2. Key Pieces (Simple)

### OAuth JWT (ID Token)
- **Role:** Proof that the user logged in with Google (or other OpenID).  
- **Why:** Establishes “who” the user is.  
- **Lifetime:** Short (expires soon).  
- **If missing/invalid:** Cannot authenticate.

### Salt
- **Role:** Secret value combined with the login identity to create the Sui address.  
- **Why:** Prevents linking your Web2 account directly to the on-chain address and adds a secret factor.  
- **Lifetime:** Persistent if reused.  
- **If lost:** You lose the original address.  
- **If leaked with login:** Account can be impersonated.

### zkLogin Address
- **Role:** The on-chain Sui account derived from identity + salt.  
- **Why:** Where you send/receive assets.  
- **Stable** if same salt + identity are reused.

### Ephemeral Key Pair
- **Role:** Temporary key for this session, used to sign transactions.  
- **Why:** Binds the action to the login without long-term keys.  
- **Lifetime:** Short.  
- **If reused improperly:** Weakens security.

### Nonce
- **Role:** Binds the ephemeral key, session expiry, and randomness into the login.  
- **Why:** Ensures the JWT is tied to this specific session.  
- **Prevents:** Replay of old tokens.

### Extended Ephemeral Public Key
- **Role:** Transforms the ephemeral public key into the format the proof system needs.  
- **Why:** So the zero-knowledge proof can include the session key.

### Zero-Knowledge Proof (PartialZkLoginSignature)
- **Role:** Cryptographically proves everything (login, salt, ephemeral key) is linked correctly without revealing secrets.  
- **Why:** Lets Sui trust the login and authorization.

### Final zkLogin Signature
- **Role:** Combines the proof, ephemeral signature, and address seed to authorize a transaction.  
- **Submitted to:** Sui for execution.

---

## 3. Simplified Flow

1. **Generate session:**  
   - Create an ephemeral key pair.  
   - Pick a validity (`maxEpoch`) and randomness.  
   - Compute a `nonce` = hash(ephemeral public key, maxEpoch, randomness).

2. **Login via OpenID:**  
   - Redirect user to provider (e.g., Google) including the nonce.  
   - Receive a JWT (`id_token`) that includes that nonce and identity (`iss`, `aud`, `sub`).

3. **Get or reuse salt:**  
   - Secret value stored by user, client, or backend.  
   - Needed to derive a consistent zkLogin address.

4. **Compute zkLogin address:**  
   - Derive address from JWT identity + salt.

5. **Get ZK proof:**  
   - Extend ephemeral public key into proof format.  
   - Send JWT, extended key, nonce parameters, salt, and `maxEpoch` to the prover service.  
   - Receive the partial zkLogin signature (proof + public inputs).

6. **Prepare transaction:**  
   - Build the Sui transaction (set sender = zkLogin address).  
   - Sign it with ephemeral key to get ephemeral signature.

7. **Assemble full signature:**  
   - Combine proof, address seed (from salt + identity), ephemeral signature, and `maxEpoch` into the final zkLogin signature.

8. **Submit to Sui:**  
   - Execute the transaction with the composite signature.  
   - Ensure the zkLogin address has enough SUI for gas.

---

## 4. Security Highlights

- **Two factors:** Web2 login (JWT) + salt. Both needed.  
- **Session binding:** Ephemeral key + nonce stop reuse of old proofs.  
- **Privacy:** Identity isn’t exposed on-chain; only its validity is proven.  
- **Address stability:** Same salt + identity ⇒ same address.  
- **Limited risk:** Ephemeral keys expire quickly.

---

## 5. Salt Storage Options

- **User-managed:** Show it once; user saves it. (If lost, address is gone.)  
- **Client-side:** Keep encrypted in browser/mobile storage.  
- **Backend:** Store encrypted and return only after verifying a fresh JWT.

---

## 6. Summary

To act as a zkLogin user you need:  
- A **salt** (stable secret),  
- A **fresh login session** (JWT + nonce + ephemeral key),  
- A **zero-knowledge proof** tying it all together,  
- A **signed transaction** with the ephemeral key,  
- A **composite signature** submitted to Sui.

Missing or mismatched pieces cause failure.

---
