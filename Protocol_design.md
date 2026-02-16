# Protocol Design
This is a preliminary protocol design in natural language following the project description given on DTU learn.

## 1. Scenario and Goal
The goal is for 'A' to let service 'P' access only a limited set of 'A''s photos on server 'B', without sharing 'A''s credentials and without granting access to other resources.

'A' authenticates to the trusted identity provider ('IdP') using the shared password (for authentication only, not as an encryption key). After successful authentication, 'IdP' issues a signed delegation token that states: subject 'A', delegate 'P', audience 'B', allowed scope (for example a specific folder), allowed action ('read'), and validity period.

'A' sends this token to 'P'. 'P' then requests data directly from 'B' and presents the token. 'B' verifies the 'IdP' signature and checks that the token is intended for 'B', assigned to 'P', within scope, and still valid. Only then does 'B' return the requested photos.

## 2. Parties and Trust Assumptions
Parties: 'A' (user), 'P' (photo-book service), 'B' (resource server), 'IdP' (identity provider), 'I' (network intruder).

Trust assumptions:
1. 'IdP' is honest and trusted by all parties.
2. Statements signed by 'IdP' are accepted as true (for example key bindings and delegation token claims).
3. 'A' shares a password only with 'IdP'; the password is used only for authentication, not encryption.
4. 'A' has no long-term public/private key pair.
5. 'B', 'P', and 'IdP' have public/private key pairs, and private keys remain secret.
6. 'A' initially knows 'pk(IdP)'; 'IdP' knows the public keys of all parties.
7. 'B' is trusted to enforce token checks: valid 'IdP' signature, audience='B', delegate='P', scope, rights, and expiry.
8. The network is controlled by intruder 'I' (Dolev-Yao), while cryptographic primitives are assumed ideal.

## 3. Security Goals


## 4. High-Level Protocol Steps


## 5. Token and Message Structure


## 6. Threats and Limitations


## 7. Diagram Notes


## 8. Planned AnB Formalization


