# Pages checklist: what each public page must contain

Every page: plain language, a version number and "last updated" date, linked from the footer
(and the grievance page on or one click from the home page). Never state that the product is
"compliant", "safe" or "secure"; describe what is done. Every fact about the company (entity,
address, officer, representative, governing law, agent) comes from the user, never a guess.
Bracketed numbers are `sources.md` entries.

## 1. Privacy notice

One page can serve several regimes if it carries the union of their contents. Build it from the
data map: one section per purpose, each naming the data, source, basis, recipients and retention.

| Content | DPDP r.3 / s.5 [1, 2] | GDPR Art. 13 [13] | SPDI [10] | CCPA [25] | Google OAuth [29] |
|---|---|---|---|---|---|
| Who we are: legal entity, contact | contact (r.9) | yes, plus EU/UK representative | yes | yes | yes |
| Grievance officer / DPO or contact person | r.9; IT Rules officer [7] | DPO if any | grievance officer | — | — |
| Itemised data collected | itemised | categories (and source if not from the person, Art. 14) | yes, incl. sensitive data | categories, at collection | Google user data specifically |
| Purposes, per data item | specific purposes + goods/services enabled | purposes + lawful basis per purpose | purpose | purposes | how Google data is used |
| Legitimate interests relied on | — | yes | — | — | — |
| Recipients / processors | (access right lists them) | recipients or categories | disclosure | categories of third parties; sold/shared? | sharing of Google data |
| Transfers outside the country and safeguard | — | yes, with safeguard | — | — | — |
| Retention per category | — | period or criteria | — | per category | — |
| Rights and how to use them, identifier needed | r.14 | access, rectification, erasure, restriction, portability, objection, withdrawal | review and correct | know, delete, correct, opt out, limit | — |
| Withdraw consent, as easy as giving it | link (r.3(c)) | yes | yes | — | revoke access |
| Complaint route | to the Board | to a supervisory authority | — | — | — |
| Whether data is required, and what happens if not given | — | yes | — | — | — |
| Automated decision-making / profiling | — | yes, with logic | — | — | — |
| Children / age | 18+ rule | — | — | under-16 sale opt-in | — |
| Grievance response period | at most 90 days (r.14(3)) | one month for rights | one month | 45 days | — |
| Language | English or Eighth Schedule language | — | — | — | — |
| Cookies | link to cookie policy | link | — | — | — |

Common omissions found in real audits: session IP and user agent; user agents on tokens or
devices; free-text answers (onboarding, feedback); referral or attribution cookies; staff action
logs; every processor and its region; and statements the schema contradicts ("your content is
deleted when you delete it" while an immutability trigger forbids it; "we do not store IPs" while
a sessions table does).

## 2. Cookie policy

A table generated from the inventory, plus how to change the choice:

| Name / key | Set by (first/third party) | Purpose | Lifetime | Category | Needs consent? |
|---|---|---|---|---|---|

Categories: strictly necessary (exempt), preferences, analytics (exempt only where a national
exemption applies: CNIL audience measurement, UK statistics exception with opt-out [22, 23]),
marketing/advertising (consent). Include `localStorage` keys and scripts that send data back
[20]. A persistent "keep me signed in" cookie is a separate choice [21].

## 3. Terms and content policy

- The prohibited-content list from IT Rules r.3(1)(b) [7]; consequences (removal, suspension).
- The quarterly reminder mechanism (an email or in-app notice at least every 3 months).
- Age requirement (18+, or the chosen rule) and what happens if it is false.
- Acceptable use, liability cap, governing law and venue (from the user), changes to terms.
- Copyright: link to the DMCA page *(US)*; repeat-infringer policy.
- Synthetic-media labelling rule if users can upload AI-generated media (open question).

## 4. Grievance and contact page

- Grievance Officer: name, designation, email, postal address, and the complaint mechanism (a
  form or address) [7]. Published timelines: acknowledge in 24 hours, resolve in 7 days, removal
  requests in 36 hours, intimate imagery in 2 hours; appeal to the Grievance Appellate Committee
  within 30 days.
- DPDP contact person or DPO (r.9) and the grievance response period (at most 90 days) [1].
- How to make a rights request, which identifier to give (r.14) [1].
- EU and UK representatives *(if targeting)* [13].
- Security contact (also in `security.txt`).

## 5. Sub-processor list

| Processor | Purpose | Data | Region | Transfer safeguard | DPA signed |
|---|---|---|---|---|---|

The "DPA signed" column is filled from the user's answer, never assumed.

## 6. Account deletion and export help

What is deleted, what is kept (each exception with its period and basis from
`erasure-patterns.md` §2), what happens to published content, how long the job takes, how to
export first. Google Play requires a web link to this when an Android app ships [30].

## 7. DMCA page *(US-hosted user content)*

Designated agent (registered with the Copyright Office, $6, renewed every 3 years) [31]; how to
send a takedown and a counter-notice; the repeat-infringer policy.

## 8. `/.well-known/security.txt` [32]

```
Contact: mailto:security@example.com
Expires: <run date + under one year, ISO 8601, e.g. 2027-05-01T00:00:00.000Z>
Preferred-Languages: en
Canonical: https://example.com/.well-known/security.txt
Policy: https://example.com/security
```

`Contact` and `Expires` are required; `Expires` must be in the future (under a year is
recommended). Replace the example domain and address with the user's real ones.

## 9. Consent and age-gate UI

- Nothing non-exempt loads before an affirmative choice; "reject" is as prominent as "accept";
  no pre-ticked boxes; withdrawing is one click from every page (footer link).
- Record the choice: what, when, which notice/policy version, how. No personal data is needed
  for an anonymous visitor's record beyond the choice itself.
- Age declaration at sign-up (a neutral question, not a pre-filled "I am over 18"), recorded with
  the notice version; a "no" stops sign-up and creates no account.
- Sign-up text links the terms and privacy notice; where consent is the basis, a separate
  affirmative action records it.

## 10. Email

- Non-transactional mail: an unsubscribe link and `List-Unsubscribe` / `List-Unsubscribe-Post`
  headers (one-click, RFC 8058 [33]); opt-outs honoured within 10 business days; postal address;
  ad label on promotions; honest subjects [27].
- Transactional mail: accurate headers and routing; no promotional content that turns it into
  commercial mail.
