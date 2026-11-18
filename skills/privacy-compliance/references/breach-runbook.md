# Breach runbook template

Copy into the project (e.g. `docs/incident-runbook.md`), fill every `<…>` with the user's real
contacts, and rehearse it once. Not legal advice; the clocks below are from the texts in
`sources.md` as last verified in `regimes.md`. The clocks run from **awareness**, so the first
job is to write down when that was.

## Contacts (filled by the user, never guessed)

| Role | Name | Reach |
|---|---|---|
| Incident lead | <…> | <…> |
| CERT-In point of contact *(India)* | <…> | <…> |
| Grievance Officer / DPDP contact | <…> | <…> |
| EU / UK representative *(if any)* | <…> | <…> |
| Lawyer | <…> | <…> |
| Processors' security contacts | <host, database, storage, email, auth> | <…> |

## Clocks

| From awareness | Who | Source |
|---|---|---|
| **6 hours** | CERT-In, for incident types listed in the Directions' Annexure (India) | CERT-In Directions 28.04.2022 [8] |
| **Without delay**, then a detailed report **within 72 hours** | Data Protection Board of India (operative from 13 May 2027) | DPDP Rules r.7(2) [1] |
| **Without delay** | Each affected person in India (from 13 May 2027) | DPDP Rules r.7(1) [1] |
| **72 hours** | EU: the lead supervisory authority only if the company has an EU establishment; otherwise every supervisory authority where affected people are. UK: the ICO | GDPR Art. 33 [13]; EDPB Guidelines 9/2022 [34] |
| **Without undue delay**, if high risk | Affected people in the EU/UK | GDPR Art. 34 [13] |
| By contract (ask for 24–48 h) | Processors must tell you | GDPR Art. 28(3)(f), 33(2) [13]; a contract term under DPDP (r.6(1)(f) requires security terms) [1] |
| Varies by state | US residents and regulators | state breach laws; check before relying [28] |

A late notification with what is known beats a complete one that misses the clock; follow up
with updates.

## Steps

1. **T0: record awareness** (who noticed, when, how) in the incident log. Open a private channel.
2. **Contain** without destroying evidence: revoke leaked credentials and sessions, rotate
   secrets, block the path, snapshot affected systems. Preserve logs (CERT-In: 180 days rolling).
3. **Scope** using the data map: which stores, which fields, which people, which regions, how
   many. Decide whether personal data was involved and whether the risk is high.
4. **≤6 h: CERT-In** report if the incident is a listed type (India).
5. **Notify the regulators** that apply (72 h). Include what happened, data and people affected,
   likely consequences, measures taken, a contact. Send what is known; supplement later.
6. **Notify affected people** in plain language through their account or registered contact:
   what happened (nature, extent, timing), likely consequences for them, what was done to reduce
   risk, what they can do (change password, watch for phishing), and a contact who can answer
   (DPDP r.7(1) lists these five items) [1].
7. **Detailed DPDP Board report** within 72 h (from 13 May 2027): updated description; facts and
   reasons; mitigation; findings about who caused it; measures to prevent recurrence; a report of
   the notifications sent to people (r.7(2)(b)) [1].
8. **Record** every breach, even ones not notified (GDPR Art. 33(5)): facts, effects, decisions
   and reasons. Keep the incident log append-only.
9. **Review** within two weeks: root cause, the control that failed, the fix, and an update to
   this runbook and the data map.

## Notification template (affected people)

```
Subject: Security incident affecting your <product> account

On <date> we found that <what happened, plainly>. It affected <which data>.
What this may mean for you: <likely consequences>.
What we have done: <containment and mitigation>.
What you can do: <specific steps>.
Questions: <named contact, email, postal address>.
```

## Rehearsal (drill)

Once before launch and yearly: run a tabletop with a fictional incident, time each step, confirm
every contact answers, and record the result in the privacy ledger.
