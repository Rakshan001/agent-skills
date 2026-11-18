# Regimes: what applies, from when, and what it asks for

> **Last verified: 2026-11-03.** Not legal advice. Dates and thresholds change: before relying on
> any row, re-check it against the primary source in `sources.md` if this date is more than 90
> days old or the deadline is within 90 days. "(secondary)" marks a law-firm or press summary
> where the primary text could not be retrieved. Bracketed numbers are `sources.md` entries.

## 1. Commencement and deadline table

| Regime / provision | In force | Notes |
|---|---|---|
| DPDP Rules 2025, G.S.R. 846(E), r.1, 2, 17–21 (Board) | 13 Nov 2025 | r.1(2) [1] |
| DPDP Rules r.4 (consent managers) | 13 Nov 2026 | r.1(3): one year after publication [1]. Consent managers are optional for a fiduciary. |
| DPDP Rules r.3, 5–16, 22, 23 (notice, security, breach, retention, contact, children, SDF, rights, transfers) | 13 May 2027 | r.1(4): eighteen months after publication [1]. |
| DPDP Act ss.3–17 (operative duties) | ~May 2027 (secondary) | Commencement notification G.S.R. 843(E) not retrieved; secondary sources say 18 months and disagree on 12/13/14 May 2027 [3]. MeitY floated shortening the timeline (Jan 2026) [5]; PIB still said eighteen months on 5 Aug 2026 [4]. No gazetted change found. |
| DPDP s.17(3) startup exemption | not notified | "Provided for" but none notified [4]. Do not rely on it. |
| IT Act s.43A + SPDI Rules 2011 | in force | Until DPDP s.44(2) repeals s.43A (expected with the operative duties, ~May 2027) [10]. |
| IT (Intermediary Guidelines) Rules 2021, as amended to 10.02.2026 | in force | [7] |
| CERT-In Directions 28.04.2022 | in force | [8], FAQ [9] |
| EU GDPR + ePrivacy Art. 5(3) | in force | [13] |
| UK DUAA: PECR statistics exception | 5 Feb 2026 | [23] |
| UK complaints-handling duty | 19 Jun 2026 | Acknowledge within 30 days (secondary) [24] |
| COPPA amended rule, compliance date | 22 Apr 2026 | [26] |
| CCPA/CPRA | in force | Threshold-gated, see §4 [25] |

## 2. India

### 2.1 DPDP Act 2023 + DPDP Rules 2025 (Rules from 13 May 2027; Act ss.3–17 ~May 2027, secondary)

Applies to digital personal data processed in India, and outside India when offering goods or
services to people in India [2 s.3].

| Duty | Source | What it asks |
|---|---|---|
| Notice | r.3; s.5 [1, 2] | Standalone, clear and plain; itemised personal data; specified purposes and the goods/services or uses enabled; link and means to withdraw consent (as easy as giving it), exercise rights, and complain to the Board. English or an Eighth Schedule language. Existing users get a notice too. |
| Consent | s.6 [2] | Free, specific, informed, unconditional, unambiguous, clear affirmative action, limited to necessary data. |
| Legitimate uses | s.7(a) [2] | Data a person voluntarily gives for a specified purpose, without objecting, may be processed for it without separate consent; likely covers account and publishing data. |
| Children (<18) | s.2(f), s.9; r.10 [1, 2] | Verifiable parental consent; no tracking, behavioural monitoring or targeted advertising directed at children. Practical choice: 18+ accounts with an age declaration. |
| Rights | ss.11–14; r.14 [1, 2] | Access (summary, plus every fiduciary and processor it was shared with), correction, completion, updating, erasure, grievance redressal, nomination. Publish how to ask and which identifier is needed; publish a grievance response period of **at most 90 days**. |
| Contact | r.9 [1] | Publish prominently the contact of the DPO or a person who can answer processing questions; repeat it in every rights response. |
| Retention | s.8(7); r.8 [1, 2] | Erase on withdrawal or when the purpose ends, and make processors erase. r.8(3): keep personal data, traffic data and processing logs **at least one year** for Seventh Schedule purposes, then erase unless another law needs longer. The Third Schedule 3-year-inactivity erasure (with a 48-hour warning, r.8(2)) applies only to listed large e-commerce, online-gaming and social-media classes. |
| Security | r.6 [1] | Encryption, obfuscation, masking or tokens; access control; access logs, monitoring and review; backups for continuity; **logs and personal data kept one year** for detection and investigation; security terms in processor contracts. |
| Breach | r.7 [1] | Each affected person without delay (what happened, likely consequences, mitigation, what they can do, a contact). The Board without delay, then a detailed report **within 72 hours** (or longer if the Board allows in writing). |
| Significant Data Fiduciary | s.10; r.13 [1, 2] | Only if designated: DPO in India, annual DPIA and audit, possible localisation of specified data. |
| Cross-border | s.16; r.15 [1, 2] | Allowed except to restricted countries; none is restricted. |

### 2.2 IT Act intermediary duties (in force now)

A service that hosts content users upload or publish is likely an intermediary. Rule 3 of the
Intermediary Rules [7]:

- Publish rules, privacy policy and user agreement prominently, including the r.3(1)(b) list of
  prohibited content.
- Remind users **at least every 3 months** of the right to suspend accounts or remove content, and
  of their liability (amended 10.02.2026).
- Act on a court order or an authorised officer's reasoned notice within **3 hours** (cut from 36
  in 2026).
- Preserve removed content and associated records **180 days**; registration data **180 days
  after account deletion** or cancellation.
- Provide information to authorised agencies within **72 hours** of a written order.
- **Grievance Officer**: name, contact and complaint mechanism on, or one click from, the home
  page. Acknowledge in **24 hours**, resolve in **7 days**; most removal requests in **36
  hours**; non-consensual intimate imagery in **2 hours**. Users may appeal to the Grievance
  Appellate Committee within 30 days.
- Significant social media intermediary duties apply only above the notified user threshold
  (widely reported as 50 lakh users; not re-verified).
- Synthetic-media labelling covers AI-generated audio, image and video, so it may reach
  user-uploaded AI images (open question).

### 2.3 CERT-In Directions 28.04.2022 (in force now) [8, 9]

- Report listed cyber incidents to CERT-In within **6 hours** of noticing them.
- Name a point of contact for CERT-In.
- Sync system clocks to NIC/NPL NTP or a source traceable to them.
- Keep ICT system logs for a rolling **180 days**. The Directions say "within the Indian
  jurisdiction"; FAQ Q35 allows storage abroad if produced in reasonable time. Conflict: lawyer.

### 2.4 SPDI Rules 2011 (until s.43A is repealed) [10]

Privacy policy; consent for sensitive personal data (passwords, financial, health, biometric and
similar); a grievance officer who resolves within one month.

### 2.5 Data residency

DPDP does not require hosting in India: transfers are allowed unless a country is restricted, and
none is [2 s.16; 1 r.15; 4]. Localisation applies only to data specified for designated SDFs
[1 r.13(4)]. CERT-In log location: see 2.3. RBI localisation binds payment system operators (the
gateway), not a merchant [11]; never store card data [12].

## 3. EU GDPR, ePrivacy, UK GDPR and PECR

| Topic | Source | What it asks |
|---|---|---|
| Scope | Art. 3(2) [13, 14] | Applies if the service targets people in the EU, paid or free. Whether a generic English site "targets" the EU is a lawyer's call. |
| Lawful bases | Art. 6 [13] | Typical: contract (account, publishing, notifications); legitimate interests (security, moderation, activity log); consent (marketing, non-exempt tracking). |
| Notice | Art. 13 [13] | See `pages-checklist.md` §1. |
| Rights | Arts 12, 15–22 [13] | Answer within one month, extendable by two months if complex. |
| Records of processing | Art. 30 [13] | Needed; the under-250-staff exemption is lost when processing is not occasional. |
| Processor contracts | Art. 28 [13] | A DPA with every processor (hosting, database, storage, email, auth, analytics). |
| Breach | Arts 33, 34 [13] | Regulator within **72 hours** of awareness; affected people without undue delay if high risk. |
| Representative | Art. 27 [13, 14] | Needed for a non-EU controller targeting the EU unless processing is occasional; separate EU and UK representatives. |
| Transfers | Ch. V [15, 16, 17] | Disclosure to a processor outside the EEA is a transfer. Check each vendor's EU-US DPF certification or SCCs; the DPF stands while appeal C-703/25 P is pending; no adequacy for India or Singapore. |
| Erasure limits | Art. 17(3) [13] | Refusal only on listed grounds, e.g. freedom of expression, a legal obligation under EU or member-state law, legal claims. An Indian retention duty is not an Art. 17(3)(b) ground. |
| Erasure practice | EDPB CEF 2025 [18]; EDPB 02/2025 [19] | Basic pseudonymisation or masking is not deletion; record erasure requests and re-apply them after restoring backups. Keep personal data out of immutable layers. |
| UK complaints | DUAA [24] (secondary) | From 19 Jun 2026: a complaints process; acknowledge within 30 days. |

### 3.1 Cookies and similar technologies [20–23]

- EDPB Guidelines 2/2023: Art. 5(3) covers any instruction that makes the browser store or send
  information back, so pixels, tracking URLs and "cookieless" scripts can be in scope [20 ¶33,
  50–51].
- Exempt: storage strictly necessary for a service the user asked for, e.g. a login session
  cookie. WP29 04/2012: persistent "remember me" cookies are not automatically exempt, which
  suggests an explicit "keep me signed in" choice [21].
- No EU-wide analytics exemption. CNIL exempts strictly limited audience measurement [22]. The UK
  has a statistics exception since 5 Feb 2026, with clear information and an easy opt-out [23].
- Typical outcome: strictly necessary cookies plus first-party aggregate analytics with an
  opt-out and a cookie policy, no banner; any third-party ad pixel or cross-site tracker needs
  prior consent.

## 4. United States

| Law | Source | Applies when | What it asks |
|---|---|---|---|
| CCPA/CPRA | [25] | Over $26.625M annual revenue, or buys/sells/shares data of 100,000+ California consumers or households, or 50%+ of revenue from selling/sharing | Notice at collection; "Do Not Sell or Share" (third-party ad tracking counts as sharing); honour Global Privacy Control; answer requests within 45 days; opt-outs within 15 business days. |
| COPPA | [26] | Directed to under-13s, or actual knowledge of an under-13 user | Verifiable parental consent; neutral age screen; amended rule compliance date 22 Apr 2026. |
| State privacy laws | [28] | Threshold-gated, state by state | Check thresholds before scaling. |
| CAN-SPAM | [27] | Commercial email | Accurate headers and routing for all mail; promotional mail needs an ad label, a postal address, honest subjects, and a working opt-out honoured within 10 business days. |
| DMCA safe harbour | [31] | Hosting user content | Designated agent registered with the Copyright Office ($6, renew every 3 years); takedown, counter-notice and repeat-infringer policy. |

## 5. Platform rules that act like law

- **Google OAuth** [29]: a privacy policy on a verified domain, on its own page, linked from a
  homepage that is not only a sign-in page, explaining how Google user data is used.
- **Google Play** [30] *(if an Android app ships)*: account deletion reachable in the app and by a
  web link, stating what is kept, why and for how long.

## 6. Open questions to put to a lawyer (copy the ones that apply)

1. Does a generic English-language site target the EU/UK (Art. 3(2))?
2. Do Indian retention duties justify keeping EU users' data after erasure, given Art. 17(3)(b)
   only recognises EU/member-state law? Is Art. 17(3)(e) enough?
3. Is a legal-redaction path through an immutability rule needed, or does Art. 17(3)(a) (free
   expression) cover published content?
4. Which SCC module, while clauses for Art. 3(2) importers are still missing [16]?
5. Follow CERT-In FAQ Q35 on log location, or keep an Indian copy of host and database logs?
6. Does DPDP reach non-Indian users' data processed by an Indian fiduciary abroad?
7. Is an 18+ self-declaration enough due diligence under DPDP r.10?
8. Are long-lived session cookies exempt in each EU member state (WP29 04/2012)?
9. Does an ad or sponsorship model "share" data (CCPA) or need consent (ePrivacy)?
10. Do synthetic-media labelling duties reach user-uploaded AI images?
11. Is a UK ICO data protection fee payable?
