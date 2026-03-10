# Project Logbook

## Snapshot 1 (Week 2): Initial Simplified AnB Model
Date: 2026-02-16

### Files
- `protocol_w2_initial.AnB`
- `Protocol_design.md`
- `Protocol_design_UML.puml`

### Purpose of This Version
Build the simplest possible AnB model that compiles and captures the main delegation flow:
1. `A` creates signed delegation information for `P`.
2. `P` forwards delegation data to `B`.
3. `B` returns scoped data to `P`.

### Simplifications / Requirement Changes (Intentional)
- `IdP` is omitted in this first version.
- `A` is modeled with a key pair (simplification), even though final scenario says `A` should rely on `IdP`.
- Delegation is represented as signed message content instead of a richer token format.
- `Scope` and `Data` are modeled as abstract `Number` values.
- No explicit timestamp, token-id, or replay cache at `B`.

### Assumptions in This Model
- Public-key cryptography is ideal (Dolev-Yao model).
- `A`, `P`, and `B` know all relevant public keys.
- Private keys remain secret.
- Network is under intruder control for active analysis.

### Static Analysis Note (Week-2 Special Task Context)
If the intruder is passive (only observes, does not send):
- The intruder learns message metadata and any signed plaintext fields that are not encrypted.
- The intruder cannot decrypt data encrypted for `P` without `inv(pk(P))`.
- The replay attack below does not occur in purely passive mode because it requires intruder message injection.

### OFMC Check
Command:
```powershell
ofmc .\protocol_w2_initial.AnB --numSess 2
```
Result:
- `ATTACK_FOUND`
- Goal violated: `strong_auth`

### Replay Attack Trace (Logged)
```text
ATTACK TRACE:
(x502,1) -> i: {x502,x43,x501,Scope(1),Na(1)}_inv(pk(x502))
i -> (x501,1): x43,{x502,x43,x501,Scope(1),Na(1)}_inv(pk(x502))
(x501,1) -> i: {{x501,x43,Scope(1),Data(2),Na(1)}_inv(pk(x501))}_(pk(x43))
i -> (x501,2): x43,{x502,x43,x501,Scope(1),Na(1)}_inv(pk(x502))
(x501,2) -> i: {{x501,x43,Scope(1),Data(3),Na(1)}_inv(pk(x501))}_(pk(x43))
```

### Interpretation of Attack
- Intruder replays the same signed delegation message to `B` in a second run.
- `B` accepts both runs and produces two authenticated responses.
- This breaks injective-style authentication (`strong_auth`) for goal `B authenticates A on Na`.
- This replay is expected at this stage, because this first protocol version is intentionally oversimplified to establish a working baseline model in AnB.

## Snapshot 2 (Week 3): IdP-Based Delegation + Lazy Intruder Executability
Date: 2026-03-09

### Files
- `protocol_w3.AnB`
- `Project_logbook.md`

### What Changed from Previous Version (and Why)
- Added an explicit identity provider participant `idp` (fixed honest agent constant).
- Removed the simplification that `A` has a public/private key pair.
- Modeled password knowledge as `pw(A,idp)` and authentication evidence as `h(pw(A,idp),A,P,B,Scope,Na)`.
- Delegation token is now signed by `idp` and contains `A,P,B,Scope,Na,Tid`.
- `B` now authorizes based on an `idp`-signed delegation token instead of directly trusting an `A` signature.

### Modeling Considerations (Assumptions and Goals)
- In line with Week 3 instructions, `A` still knows all public keys initially (temporary simplification).
- The shared password term `pw(A,idp)` is **not** used as an encryption key; it is only used for authentication evidence.
- `idp` is modeled as honest and trusted by all parties for signed statements.
- Goals in this version:
1. `B authenticates idp on Tid`
2. `B authenticates A on Na`
3. `P authenticates B on Data`
4. `Data secret between B,P`

### Special Task (Week 3): Lazy Intruder Proof That Role `P` Is Executable
Following the lecture method, instantiate `P` as intruder `i` and keep `A,B,idp` honest.

`P`-role strand in this model:
1. Incoming from `A`: `Tok = {A,i,B,Scope,Na,Tid}inv(pk(idp))`
2. Outgoing to `B`: `i,Tok`

Constraint sequence:
1. Initial intruder knowledge (`M0`) from role setup:
   `{A,i,B,idp,pk(i),inv(pk(i)),pk(B),pk(idp)}`
2. After incoming message, knowledge is:
   `M1 = M0 U {Tok}`
3. Outgoing obligation:
   `M1 |- (i,Tok)`
4. By composition, reduce to sub-goals:
   `M1 |- i` and `M1 |- Tok`
5. Both are solved by axiom:
   `i in M1` and `Tok in M1`

Conclusion:
- The constraint is solved, so the intruder can produce every required outgoing message of role `P`.
- Therefore role `P` is executable under the lazy intruder method.

### Problems / Open Issues
- Replay handling at `B` is still abstract; under multi-session analysis, the same valid token can be replayed to trigger repeated acceptances.
- This currently violates injective authentication (`strong_auth`) on the token identifier (`Tid`).

### Week 3 OFMC Iteration Log (Executable Versions with Attack Traces)
Command used:
```powershell
ofmc protocol_w3.AnB --numSess 2
```

1. Executable version V1 (before constraining trusted `IdP`):
- Result: `ATTACK_FOUND`, goal: `weak_auth`.
- Attack trace (excerpt):
```text
i -> (x301,1): x38,{x302,x38,x301,x210,x211,x212}_inv(pk(i))
(x301,1) -> i: {{x301,x38,x210,Data(1),x212}_inv(pk(x301))}_(pk(x38))
```
- Comment on failure:
  - OFMC can instantiate the identity provider as intruder (`IdP = i`) in this version.
  - `B` therefore accepts attacker-signed tokens, violating the trusted-IdP requirement.

2. Executable version V2 (after enforcing honest IdP, but before binding full request fields):
- Result: `ATTACK_FOUND`, goal: `weak_auth`.
- Attack trace (excerpt):
```text
(x502,1) -> i: x502,x28,x27,Scope(1),Na(1),h(pw(x502,x37),Na(1))
i -> (x37,1): x502,x38,x501,x410,Na(1),h(pw(x502,x37),Na(1))
(x37,1) -> i: {x502,x38,x501,x410,Na(1),Tid(2)}_inv(pk(x37))
i -> (x501,1): x38,{x502,x38,x501,x410,Na(1),Tid(2)}_inv(pk(x37))
```
- Comment on failure:
  - Intruder rewrites delegation fields (`P`, `B`, `Scope`) while preserving a valid authenticator on `Na`.
  - This version does not satisfy Week 3 intent that `A`'s authenticated request to IdP binds the authorization details.

3. Executable version V3 (current syntax-aligned model with `idp` and bound request fields):
- Result: `ATTACK_FOUND`, goal: `strong_auth`.
- Attack trace (excerpt):
```text
i -> (idp,1): i,x49,x501,x409,x410,h(pw(i,idp),i,x49,x501,x409,x410)
(idp,1) -> i: {i,x49,x501,x409,x410,Tid(1)}_inv(pk(idp))
i -> (x501,1): x49,{i,x49,x501,x409,x410,Tid(1)}_inv(pk(idp))
(x501,1) -> i: {{x501,x49,x409,Data(2),Tid(1)}_inv(pk(x501))}_(pk(x49))
i -> (x501,2): x49,{i,x49,x501,x409,x410,Tid(1)}_inv(pk(idp))
(x501,2) -> i: {{x501,x49,x409,Data(3),Tid(1)}_inv(pk(x501))}_(pk(x49))
```
- Comment on failure:
  - A valid token with the same `Tid` is replayed to `B` in multiple sessions.
  - `B` accepts repeated token use, breaking injective authentication (`strong_auth`) on token usage.

## Snapshot 3 (Week 4): Typed Formats + Separate Key Lookup
Date: 2026-03-10

### Files
- `protocol_w4.AnB`
- `protocol_w4_key_lookup.AnB`
- `Protocol_design.md`

### What Changed from Previous Version (and Why)
- Added a separate key-discovery protocol so `A` no longer starts with the public key of every party.
- Restricted `A`'s initial knowledge in the main delegation model to `pk(idp)` plus the shared password term.
- Replaced the previous untagged messages with explicit message-format tags:
  - `key_req_tag`, `key_resp_tag`
  - `auth_req_tag`, `deleg_tok_tag`, `deleg_msg_tag`, `access_req_tag`, `data_msg_tag`
- Used tag constants instead of nested function-format terms because the latter made the protocol non-executable in OFMC when responders needed to reuse parsed fields.

### Security Rationale
- Distinct message tags reduce the risk that one message can be reinterpreted as another protocol message.
- The key-discovery protocol isolates trust in key bindings to signed `idp` responses.
- The main protocol now matches the intended trust model more closely: `A` relies on `idp` for both authentication and public-key discovery.

### Week-4 OFMC Results

Results:
- `protocol_w4_key_lookup.AnB`: `NO_ATTACK_FOUND`
- `protocol_w4.AnB`: `ATTACK_FOUND` on `strong_auth`

### Week-4 Attack Status
- The separate key-lookup protocol verifies successfully once the signed response binds the requester identity `A` together with `(C, pk(C), Nk)`.
- The main delegation protocol remains vulnerable to replay of the same valid `idp` token at `B`.
- This is not a type-flaw attack; it is a reuse / injectivity problem on `Tid`.

### Week-4 Type-Flaw Resistance Argument
To show type-flaw resistance, consider any two non-variable messages `m1` and `m2` in the set of protocol messages `SMP`. Assume `m1` and `m2` have a unifier. Then their outermost constructors must be identical.

In the week-4 models, every protocol message format is distinguished by a unique top-level tag:
- `key_req_tag` for key lookup requests
- `key_resp_tag` for key lookup responses
- `auth_req_tag` for authentication requests from `A` to `idp`
- `deleg_tok_tag` for `idp`-signed delegation tokens
- `deleg_msg_tag` for token forwarding from `A` to `P`
- `access_req_tag` for access requests from `P` to `B`
- `data_msg_tag` for responses from `B` to `P`

Therefore, if two messages have different protocol types, then they have different top-level tags, and unification fails immediately. For example:
- `auth_req_tag, A,P,B,Scope,Na,h(...)` cannot unify with `deleg_msg_tag, Tok`
- `access_req_tag, P, Tok` cannot unify with `data_msg_tag, Resp`
- `key_req_tag, A,C,Nk` cannot unify with `key_resp_tag, Resp`

The only remaining case is that two messages have the same top-level tag. But then they already belong to the same protocol format by construction, and hence they have the same type.

So for any two non-variable messages in `SMP`, if they have a unifier, they must have the same type. This shows that the week-4 protocols are type-flaw resistant.