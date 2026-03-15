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

## Snapshot 4 (Week 5): Pseudonymous Channels
Date: 2026-03-13

### Files
- `protocol_w5.AnB`
- `Project_logbook.md`


### Week-5 Design Decision
The week-5 variant replaces transport-level protection with pseudonymous secure channels wherever that does not destroy the authorization logic:
1. `A -> idp` becomes `[A] *->* idp`, so `A` can send `pw(A,idp)` directly over the TLS-style channel instead of hashing it into a message authenticator.
2. `A -> P` becomes `[A] *->* P`, because the token-forwarding step only needs confidential and authenticated transport to the service endpoint.
3. `P -> B` becomes `[P] *->* B`, because the access request is also a client-to-server TLS-style transmission.
4. The `idp` signature on the delegation token is retained, because `B` must still verify an authorization statement that survives forwarding through `A` and `P`.
5. The `B -> P` payload encryption/signature from week 4 is also retained in the final week-5 file, because OFMC found an immediate confidentiality break when that last step was replaced by a pure pseudonymous channel.

### Week-5 Iteration V1: Replace Every Transport Step by Pseudonymous Channels
Command used:
```powershell
ofmc .\protocol_w5.AnB --numSess 2
```

Result:
- `ATTACK_FOUND`
- Goal violated: `secrets`

Protocol version that produced the attack:
```anb
Protocol: Photo_Delegation_W5_Pseudonymous_Channels

# Week-5 delegation protocol:
# - Replace transport cryptography with pseudonymous secure channels
#   where the server side is authenticated
# - Keep only the idp signature that must survive forwarding to B
# - Pseudonymous channels do not themselves guarantee freshness

Types: Agent A,P,B,idp,auth_req_tag,deleg_tok_tag,deleg_msg_tag,access_req_tag,data_msg_tag;
       Number Na,Tid,Scope,Data;
       Function pk,pw

Knowledge: A: A,P,B,idp,auth_req_tag,deleg_tok_tag,deleg_msg_tag,access_req_tag,data_msg_tag,pw(A,idp);
           P: A,P,B,idp,auth_req_tag,deleg_tok_tag,deleg_msg_tag,access_req_tag,data_msg_tag,pk(idp);
           B: A,P,B,idp,auth_req_tag,deleg_tok_tag,deleg_msg_tag,access_req_tag,data_msg_tag,pk(idp);
           idp: A,P,B,idp,auth_req_tag,deleg_tok_tag,deleg_msg_tag,access_req_tag,data_msg_tag,pk(idp),inv(pk(idp)),pw(A,idp)

where A!=P, A!=B, P!=B

Actions:

[A] *->* idp:
  auth_req_tag,A,P,B,Scope,Na,pw(A,idp)

idp *->* [A]:
  {deleg_tok_tag,A,P,B,Scope,Na,Tid}inv(pk(idp))

[A] *->* P:
  deleg_msg_tag,
  {deleg_tok_tag,A,P,B,Scope,Na,Tid}inv(pk(idp))

[P] *->* B:
  access_req_tag,
  P,
  {deleg_tok_tag,A,P,B,Scope,Na,Tid}inv(pk(idp))

B *->* [P]:
  data_msg_tag,
  B,Scope,Data,Tid

Goals:
B authenticates idp on Tid
B authenticates A on Na
P authenticates B on Data
Data secret between B,P
```

Attack trace:
```text
i -> (idp,1): confChCr(i),{{authreqtag,i,x40,x39,x308,x309,pw(i,idp)}_inv(confChCr(i))}_(confChCr(idp))
(idp,1) -> i: {{{delegtoktag,i,x40,x39,x308,x309,Tid(1)}_inv(pk(idp))}_inv(authChCr(idp))}_(confChCr(i))
i -> (x39,1): confChCr(i),{{accessreqtag,x40,{delegtoktag,i,x40,x39,x308,x309,Tid(1)}_inv(pk(idp))}_inv(confChCr(i))}_(confChCr(x39))
(x39,1) -> i: {{datamsgtag,x39,x308,Data(2),Tid(1)}_inv(authChCr(x39))}_(confChCr(i))
```

Interpretation:
- This version replaced even the final `B -> P` response by `B *->* [P]`.
- OFMC shows that `B` then returns `Data` on the same pseudonymous channel that carried the request.
- That is too weak for this protocol: `B` no longer has a cryptographic way to bind the response to the registered delegate `P`.
- So a pure channel-based replacement is not safe here, even though it matches the TLS abstraction locally.

### Week-5 Final Snapshot: Keep Only the Necessary End-to-End Crypto
The final file `[protocol_w5.AnB]` keeps the `idp`-signed token and restores the week-4 encrypted/signed response from `B` to `P`, while still replacing:
- `A -> idp` by `[A] *->* idp`
- `A -> P` by `[A] *->* P`
- `P -> B` by `[P] *->* B`

OFMC results for the final week-5 file:
- `ofmc .\protocol_w5.AnB --numSess 1` -> `NO_ATTACK_FOUND`
- `ofmc .\protocol_w5.AnB --numSess 2` -> `ATTACK_FOUND` on `strong_auth`

Attack trace for the final week-5 file:
```text
i -> (idp,1): confChCr(i),{{authreqtag,i,x53,x501,x410,x411,pw(i,idp)}_inv(confChCr(i))}_(confChCr(idp))
(idp,1) -> i: {{{delegtoktag,i,x53,x501,x410,x411,Tid(1)}_inv(pk(idp))}_inv(authChCr(idp))}_(confChCr(i))
i -> (x501,1): confChCr(i),{{accessreqtag,x53,{delegtoktag,i,x53,x501,x410,x411,Tid(1)}_inv(pk(idp))}_inv(confChCr(i))}_(confChCr(x501))
(x501,1) -> i: datamsgtag,{{x501,x53,x410,Data(2),Tid(1)}_inv(pk(x501))}_(pk(x53))
i -> (x501,2): confChCr(i),{{accessreqtag,x53,{delegtoktag,i,x53,x501,x410,x411,Tid(1)}_inv(pk(idp))}_inv(confChCr(i))}_(confChCr(x501))
(x501,2) -> i: datamsgtag,{{x501,x53,x410,Data(3),Tid(1)}_inv(pk(x501))}_(pk(x53))
```

The final week-5 protocol avoids the immediate secrecy break from the first all-channel variant, but it still inherits the replay weakness already suggested by the lecture: pseudonymous TLS-style channels are useful for replacing transport cryptography, yet they do not provide freshness on their own. As a result, the same valid `idp` token with the same `Tid` can still be reused against `B` in multiple sessions, so the model continues to fail injective authentication (`strong_auth`) on token usage.

## Snapshot 5 (Week 6): Guessable Password Security
Date: 2026-03-15

### Files
- `protocol_w6.AnB`

### Task
Special task of the week: verify that the protocol is secure even if the
password `pw(A,idp)` between A and the identity provider is a guessable
(weak) secret.

### Step 1 — Exposing the Vulnerability

The standard OFMC method for modeling a guessable secret is to expose it on
the network. As a first step, the week-5 authenticated channel to idp was
replaced with a plain public channel and the password sent in plaintext:

```
Was (W5): [A] *->* idp: auth_req_tag,A,P,B,Scope,Na,pw(A,idp)
Try (W6):  A   ->  idp: auth_req_tag,A,P,B,Scope,Na,pw(A,idp)
```

OFMC immediately finds an attack (`--numSess 1`, depth 3):

```text
(x502,1) -> i: authreqtag,x502,x31,x30,Scope(1),Na(1),pw(x502,idp)
i -> (idp,1): authreqtag,x502,x40,x501,x410,x411,pw(x502,idp)
(idp,1) -> i: {delegtoktag,x502,x40,x501,x410,x411,Tid(2)}_inv(pk(idp))
i -> (x501,1): confChCr(i),{{accessreqtag,...}_inv(confChCr(i))}_(confChCr(x501))
(x501,1) -> i: datamsgtag,{{x501,x40,x410,Data(3),Tid(2)}_inv(pk(x501))}_(pk(x40))
```

The intruder intercepts A's plaintext message, copies `pw(A,idp)`, rewrites
P, B, Scope, and Na to intruder-controlled values, and sends the modified
request to idp. IdP cannot detect the substitution and issues a valid token
for the intruder's chosen parameters. B then returns data encrypted for the
intruder's key, violating both `B authenticates A on Na` and data secrecy.

### Step 2 — First Fix Attempt and Why It Fails

The obvious fix is to encrypt the auth request under `pk(idp)` so the password
is no longer visible:

```
A -> idp: {auth_req_tag,A,P,B,Scope,Na,pw(A,idp)}pk(idp)
```

OFMC still finds a `guesswhat` attack. The reason: `Na` also appears in the
signed delegation token returned by idp in step 2, which is only signed (not
encrypted) and therefore readable by anyone. Once the intruder learns `Na`
from the token, they can reconstruct the step-1 ciphertext with a guessed
password and verify the guess offline.

### Step 3 — Cover-Nonce Fix

The fix, following the week-6 slide pattern `A->B: { A, pw(A,B), K }pk(B)`,
is to add a dedicated **cover nonce `Ka`** inside the step-1 encryption:

```
Now (W6): A -> idp: {auth_req_tag,A,P,B,Scope,Na,Ka,pw(A,idp)}pk(idp)
```

`Ka` is fresh and **never appears anywhere else** in the protocol. Even after
the intruder learns `Na` from the signed token, they cannot reconstruct the
ciphertext without also knowing `Ka`, making offline password verification
infeasible. Additional changes: `Ka` declared as `Number`, `pk(idp)` added to
A's initial knowledge, and goal `pw(A,idp) guessable secret between A,idp` added.

### OFMC Results (Final Protocol)

```powershell
ofmc .\protocol_w6.AnB --numSess 1   # NO_ATTACK_FOUND
ofmc .\protocol_w6.AnB --numSess 2   # ATTACK_FOUND (strong_auth, pre-existing Tid replay)
```

**`--numSess 1`** — all five goals satisfied:

| Goal | Result |
|------|--------|
| `B authenticates idp on Tid` | PASS |
| `B authenticates A on Na` | PASS |
| `P authenticates B on Data` | PASS |
| `Data secret between B,P` | PASS |
| `pw(A,idp) guessable secret between A,idp` | PASS |

**`--numSess 2`** — `ATTACK_FOUND` on `strong_auth` only. The intruder
authenticates as itself using `pw(i,idp)`, obtains a valid token with `Tid(1)`,
and replays it to B in a second session. This is the pre-existing Tid replay
vulnerability present since week 3; the `guesswhat` goal is not violated.

### Conclusion
The cover-nonce technique successfully defeats offline dictionary attacks even
when `pw(A,idp)` is declared guessable. The special task of week 6 is
satisfied: the protocol is secure against guessable passwords. The only
remaining failure is the unrelated Tid replay, which would require B to
maintain a replay cache to fix.
