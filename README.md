# katiban

Debt ledger between two people, recorded and attested by witnesses.

> **Not legal or financial advice.** Katiban is a record-keeping aid. It is not a notary, and there is no guarantee that a court or any authority will accept a record. It never holds, moves or collects money.

---

## What it is

Loans between friends, family and community members often rest on trust and memory. Months later, people disagree about the amount, the date, or who was present.

Katiban lets two parties (a debtor and a creditor) write down a loan that carries **no interest, in any form**. Witnesses attest to it. Katiban acts as the **scribe**: it records exactly what the parties state, never edits it, and never takes a side.

## Principles

1. **Neutrality.** Both parties see the same record at the same time. The scribe favors no one.
2. **Always available.** Recording a debt is never locked behind a paywall or a minimum amount.
3. **No harm.** No selling or mining of debt data, no pressure features, no public shaming.
4. **The debtor states the terms.** Katiban never proposes amounts, dates or conditions. It writes what the debtor states and the creditor confirms.
5. **No interest, ever.** No fees based on the debt amount or on repayments.

## Roles

| Role                     | What they do                                                     |
| ------------------------ | ---------------------------------------------------------------- |
| **Debtor**               | States the terms: amount, currency, due date, notes              |
| **Creditor**             | Confirms the terms                                               |
| **Witnesses**            | Each attests separately, after seeing the terms                  |
| **Scribe**               | Katiban itself: append-only, neutral                             |
| **Guardian** _(planned)_ | States the terms on behalf of a debtor who cannot, acting fairly |

## Debt lifecycle

```
draft -> proposed -> confirmed -> witnessing -> active
                                                 |
                  extended <---- (hardship) -----+
                                                 +--> partially_repaid --> settled
                                                 +--> forgiven (creditor remits) --> settled
                                                 +--> disputed
settled --(both confirm + grace period)--> purged
```

- `proposed`: the debtor stated the terms.
- `confirmed`: the creditor agreed. Terms are now frozen.
- `witnessing`: both parties accepted a witness list, and witnesses attest one by one.
- `active`: the witness policy is satisfied and the record is sealed.
- `settled`: **both** parties confirm full repayment (or the creditor forgives).
- `purged`: possible **only** from `settled`, after a grace period (default 30 days). The code rejects purging any unpaid debt.
- Corrections are never edits. They are new entries that reference the old one.

## Design rules

| #   | Rule                                                                  | Behavior in Katiban                                                                                         | Status       |
| --- | --------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- | ------------ |
| 1   | A debt is written down                                                | A debt cannot become `active` without a written record                                                      | Strict       |
| 2   | A due date is set                                                     | `due_date` is required                                                                                      | Strict       |
| 3   | The scribe is neutral                                                 | Append-only, hash-chained log, no edits, equal access for both parties                                      | Strict       |
| 4   | Recording is always possible                                          | No minimum amount, no paywall on recording                                                                  | Strict       |
| 5   | The debtor states the terms                                           | Debtor drafts, creditor confirms, Katiban never suggests terms                                              | Strict       |
| 6   | A guardian can act for a debtor who cannot                            | Optional guardian role                                                                                      | Planned      |
| 7   | Witnesses attest to the debt                                          | Configurable **witness policy** (see below). Both parties must accept each witness                          | Configurable |
| 8   | Witnesses can be reached and their testimony kept                     | Invitations and reminders. Testimony stays retrievable while the debt is open. The app cannot compel anyone | Strict       |
| 9   | Hardship is allowed for                                               | `extended` state when the debtor asks, `forgiven` state when the creditor remits                            | Strict       |
| 10  | Nobody is harmed by taking part                                       | Data minimization, no data sales, witnesses see only what they need                                         | Strict       |
| 11  | Pledges and collateral                                                | Pledge notes                                                                                                | Later        |
| 12  | Whether a record is needed between parties who fully trust each other | Open design question                                                                                        | Open         |
| 13  | Small cash exchanges that need no record                              | Out of scope                                                                                                | Out of scope |

### Witness policy

The witness policy is a **configuration profile**, so it can be adapted without changing the core.

- The **default profile** requires **two male witnesses, or one man and two women**.
- A **neutral profile** requires **two adult witnesses, with no requirement about sex**. The parties choose the profile when the debt is created, and the record shows which one was chosen.
- Both parties must accept every witness before the record can become active.
- Katiban cannot verify who a person really is in the real world. Both parties accepting each witness is the protection against fake witness accounts.
- Eligibility details (age, trustworthiness, relationship to the parties) are left to the parties' agreement. Katiban does not judge anyone's character.
- Additional profiles can be added, with documented reasoning and review.

## Stack

- **Language:** Go 1.27 or later (standard `net/http`, `database/sql`, `encoding/json/v2`)
- **Database:** **PostgreSQL**
- **Front end:** SvelteKit
- **Auth:** passkeys or OpenID Connect login (see below)
- **Format:** JSON handled with `encoding/json/v2`. Content that gets hashed is first put in **canonical form (RFC 8785)**, because the same data must always produce the same bytes.

## Authentication

- **Accounts:** people can sign in with a **passkey** (no password stored by Katiban) or with an **external identity provider** through OpenID Connect ("Sign in with Google", Microsoft, Apple, or a self-hosted Keycloak).
- **OIDC (as a relying party):** Authorization Code flow with PKCE, following the **OAuth 2.1 profile** (no implicit flow, no password grant, exact redirect matching). Katiban does **not** build its own authorization server.
- **Passkeys (WebAuthn):** planned library: `go-webauthn/webauthn`.
- **Witnesses:** one-time invitation link, verified by email, then a short-lived session.
- **API tokens for third-party apps** are not part of v1. If added later, they follow the same OAuth 2.1 profile.
- **Idea to explore:** each attestation could be a WebAuthn assertion over the entry hash. The witness's own device would sign the exact content they saw, so the proof does not depend on trusting the server.

## Data and privacy

- Each debt has its **own hash chain**. Purging a debt deletes its whole chain, so nothing is left half-verifiable.
- Personal data (names, emails) is stored apart from the chain and encrypted with a **per-debt key**. Purge deletes the key and the rows.
- Parties can export a **signed PDF** of the record before purge. The proof then lives with them, not on the server.
- GDPR: this project is a tool. Whoever hosts an instance is responsible for the data. Handling of erasure requests is still to be designed. The intended direction: delete what is not needed for an open debt, and keep only the minimum the other party needs for their claim until the debt is settled.
- Witness gender is needed only by the default witness profile, not by the neutral one. Collect it as **self-declared and minimal**, and treat it as sensitive.

## Contributing

Issues and discussion are welcome, especially from people with legal or domain knowledge. Please cite sources when proposing changes to the design rules.

## License

Apache-2.0 license
