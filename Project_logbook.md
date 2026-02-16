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
