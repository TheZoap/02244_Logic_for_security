# Protocol Design
This is the current natural-language design for the week-4 version of the protocol.

## 1. Scenario and Goal
The goal is for `A` to let service `P` access a limited subset of `A`'s photos on server `B`, without giving `P` `A`'s password and without letting `P` access any other resources.

The week-4 version separates two concerns:
1. A key-discovery protocol where `A` asks the trusted identity provider `idp` for the public key of another party.
2. A delegation protocol where `A` authenticates to `idp`, receives an `idp`-signed delegation token, and forwards that token to `P` for presentation to `B`.

## 2. Parties and Trust Assumptions
Parties: `A` (user), `P` (photo-book service), `B` (resource server), `idp` (identity provider), `I` (network intruder).

Trust assumptions:
1. `idp` is honest and trusted by all parties.
2. Statements signed by `idp` are accepted as authoritative for both key bindings and delegation claims.
3. `A` shares a password only with `idp`; the password is used only for authentication evidence, not as an encryption key.
4. `A` has no long-term public/private key pair.
5. `P`, `B`, and `idp` have public/private key pairs, and their private keys remain secret.
6. `A` initially knows only `pk(idp)`.
7. `idp` knows the public keys of the registered parties.
8. `B` is trusted to verify the `idp` signature and enforce the audience, delegate, scope, and freshness checks encoded in the token.
9. The network is controlled by the Dolev-Yao intruder, while cryptographic primitives are assumed ideal.

## 3. Week-4 Design Choices
To address the week-4 requirements, each message carries an explicit tag that identifies its format. In the OFMC model, this is encoded with dedicated public tag constants rather than nested function terms, because that keeps the protocol executable while still making message types distinguishable:
- `key_req_tag`, `key_resp_tag`
- `auth_req_tag`, `deleg_tok_tag`, `deleg_msg_tag`, `access_req_tag`, `data_msg_tag`

These tagged formats are intended to prevent type-flaw attacks where one message shape could otherwise be misread as another.

## 4. Protocol Sketch
Key-discovery protocol:
1. `A -> idp`: request the public key of some target party.
2. `idp -> A`: tagged response carrying a signed binding between the requester, the requested identity, and its public key.

Delegation protocol:
1. `A -> idp`: authentication request carrying password-derived evidence over the full delegation request.
2. `idp -> A`: signed delegation token.
3. `A -> P`: forwarding wrapper containing the signed delegation token.
4. `P -> B`: access request carrying the same signed token.
5. `B -> P`: encrypted and signed data response bound to the token identifier.

## 5. Diagram Notes
See [Protocol_design_UML.puml](/C:/Users/olive/Documents/02244_Logic_for_security/Protocol_design_UML.puml).
