> ⚠️ **TEMPLATE — NOT LEGAL ADVICE**
>
> This document was generated from a template and has not been reviewed by an
> attorney. Do not publish or rely on it until reviewed by qualified counsel
> in the relevant jurisdiction(s). Version v0.1.0 (unreviewed draft — do not publish).

# Privacy Policy — DDT (Databricks Data Tools)

**Plain-language summary:** Here's what data DDT (Databricks Data Tools) collects from you, why, who else sees it, how long we keep it, and how to get rid of it.

## 1. Who we are

Varol Consulting LLC ("we", "us", "our") operates DDT (Databricks Data Tools) (the "Service"). This Privacy Policy explains how we handle personal information.

Contact for privacy matters: sdt.ddt.tools@gmail.com
Mailing address: <<< DECISION NEEDED: postal address — none published (email-only contact); no legal entity formed yet. Counsel to advise whether a postal/agent-for-service address is required for the targeted jurisdictions. >>>


**Data controller (GDPR):** Varol Consulting LLC is the data controller for personal data processed in connection with the Service. <<< DECISION NEEDED: eu_representative — whether you have an EU representative under GDPR Art. 27 (required if processing EU residents' data and not established in the EU) >>>



**Data controller (UK GDPR):** Varol Consulting LLC is the data controller. <<< DECISION NEEDED: uk_representative — whether you have a UK representative under UK GDPR Art. 27 >>>


## 2. What we collect

We collect only the personal information we need to provide the Service. Here's the complete list:

| What | Required? | Why | Retention |
|---|---|---|---|
| Sanitized error diagnostics (opt-in error reporting): error name, sanitized error message (quoted identifiers / string literals / credentials redacted client-side before sending), sanitized stack frames (app-only, package-relative paths), error fingerprint (sha256), occurrence counts, timestamps | no | Diagnose and fix bugs. Opt-in via explicit first-run consent. NEVER includes your SQL, identifiers, catalog/schema/table/column names, file contents, credentials, or absolute paths — these are redacted client-side before transmission. | Until the bug is fixed and a fix-notification is sent, or 12 months, whichever is sooner |
| Environment context attached to error reports: operating system, Node.js version, product version, surface (CLI or VS Code) | no | Reproduce and triage errors against the right platform. Sent only with an error report when error reporting is enabled. | Same as the error record it accompanies |
| Reporter email (optional) | no | Notify you when the bug you reported is fixed. Collected only if you provide it via `ddt feedback --contact` or it comes from an installed license. Never required. | Until the related bug is fixed and notification sent, or removed on request (GDPR erasure endpoint), or 12 months |
| Anonymous usage ping (opt-in): a random install UUID (never derived from hardware or user identity), product name, version, OS platform, surface (cli/vscode) | no | Count active installs and platforms to prioritize support. Opt-in via the same first-run consent. Sent at most once per 24h and aggregated daily. | Aggregated; aggregates kept 12 months |
| Feedback submissions (explicit user action via `ddt feedback` or VS Code Report-a-Bug): the sanitized text you type, CLI version, OS, optional contact email, the names (not contents) of your last 5 commands | no | Act on the feature requests and bug reports you deliberately send us. Only sent when you run the feedback command or use Report-a-Bug. | Until resolved, or 12 months, whichever is sooner |

We do not knowingly collect any data field that is not in this list. If we add a new field, we will update this policy before turning on collection.

## 3. How we use it

We use your personal information to:
- Provide, maintain, and improve the Service
- Authenticate you and secure your account
- Process payments and detect fraud (if applicable to your plan)
- Communicate with you about the Service (service announcements, security notices, billing)
- Comply with legal obligations and respond to lawful requests from authorities
- Aggregate or anonymize for analytics (after anonymization, the data is no longer personal information)



## 4. Who we share it with

We share personal information with the following third parties (subprocessors) only as needed to provide the Service:

| Subprocessor | Purpose | Data shared | Location | Privacy notice |
|---|---|---|---|---|
| Cloudflare, Inc. | Hosting and storage of error reports, usage-ping aggregates, and feedback (Cloudflare Workers + Cloudflare KV, global edge network) | All opt-in telemetry and feedback data listed above | Global edge network; US-based company | [link](https://www.cloudflare.com/privacypolicy/) |
| MailChannels (or successor email service) | Sending fix-notification emails to reporters who provided a contact email | Reporter email address and the notification message | USA | [link](https://www.mailchannels.com/privacy-policy/) |
| Microsoft Corporation (Visual Studio Code Marketplace) | Distribution of the VS Code extension. We receive no personal data from Microsoft. | None received by us | USA | [link](https://privacy.microsoft.com/privacystatement) |
| npm, Inc. (GitHub / Microsoft) | Distribution of the `@ddt-tools/cli` npm package. We receive no personal data from npm. | None received by us | USA | [link](https://docs.npmjs.com/policies/privacy) |

We do not sell personal information to anyone.


**For California residents:** "Sell" and "share" have the specific meanings given by the CCPA/CPRA. We do not sell or share your personal information for cross-context behavioral advertising as defined by the CPRA. See Section 8 for your California rights.


We may also disclose personal information when required by law, court order, or government request; to protect our rights or the safety of users; or in connection with a merger, acquisition, or sale of assets (in which case we will give notice before your information is transferred and becomes subject to a different privacy policy).

## 5. International transfers


**For EU users:** Some of our subprocessors are located outside the European Economic Area. When we transfer your personal data outside the EEA, we rely on the European Commission's Standard Contractual Clauses (SCCs) and, where applicable, the EU–US Data Privacy Framework, to provide an adequate level of protection. <<< DECISION NEEDED: international_transfer_mechanism — specify the exact legal mechanism (SCCs 2021/914 + supplementary measures, DPF, adequacy decision, etc.) >>>



**For UK users:** We use the UK International Data Transfer Agreement (IDTA) or UK Addendum to the EU SCCs for transfers outside the UK, as applicable.


## 6. Cookies and tracking

The Service uses cookies and similar technologies. See the [Cookie Notice](./cookie-notice.md) for the complete list of cookies, their purposes, and how to manage them.

## 7. Data retention

We keep personal information only as long as needed for the purposes described in this policy or as required by law. Specifically:

- **Account data** (email, login credentials, profile): kept while your account is active and deleted within 30 days after account closure (a 30-day backup window).
- **Usage logs and analytics:** kept according to the retention values in the table in Section 2.
- **Payment records:** kept for the period required by tax and accounting law (typically 7 years in most jurisdictions). <<< DECISION NEEDED: payment_retention_law — the specific tax-retention period that applies to your business >>>
- **Backup copies:** kept in encrypted form for up to 90 days for disaster recovery, then permanently destroyed.

You can request earlier deletion through the rights process in Section 8.

## 8. Your rights

Depending on where you live, you have some or all of the following rights:

- **Access:** ask for a copy of the personal information we hold about you
- **Rectification:** ask us to correct inaccurate information
- **Erasure:** ask us to delete your personal information
- **Restriction:** ask us to stop processing your information temporarily
- **Portability:** receive your information in a machine-readable format
- **Objection:** object to processing based on legitimate interests
- **Withdraw consent:** where processing is based on consent, withdraw it at any time

**To exercise any of these rights**, email sdt.ddt.tools@gmail.com from the address associated with your account. We will respond within 30 days (or sooner if required by law). We may ask for additional verification if your identity is unclear.


**California rights (CCPA/CPRA):** California residents also have the right to know what categories of personal information we have collected; the right to opt out of any "sale" or "sharing" (we do neither, but the right exists); the right to non-discrimination for exercising any of these rights; and the right to limit use of sensitive personal information. The verifiable-consumer-request process is the same as above. We do not respond to Do-Not-Track signals because there is no industry-standard meaning, but we honor the Global Privacy Control (GPC) signal as an opt-out request where applicable.



**EU rights (GDPR):** You have the right to lodge a complaint with your local supervisory authority. The list of authorities is available at https://edpb.europa.eu/about-edpb/about-edpb/members_en.

The legal bases on which we process your personal data are:
- **Contract** — processing necessary to provide the Service you signed up for (Art. 6(1)(b))
- **Legitimate interests** — security, fraud prevention, product analytics (Art. 6(1)(f))
- **Legal obligation** — tax records, lawful requests (Art. 6(1)(c))
- **Consent** — optional analytics cookies and marketing emails (Art. 6(1)(a)) — you may withdraw at any time



**UK rights (UK GDPR + DPA 2018):** You have the right to lodge a complaint with the Information Commissioner's Office (ICO) at https://ico.org.uk/.



**Canadian rights (PIPEDA):** Canadian residents may contact the Office of the Privacy Commissioner of Canada at https://www.priv.gc.ca/ if a privacy concern is not resolved with us first. Quebec residents have additional rights under Law 25, including the right to data portability and the right to know about automated decision-making.


## 9. Security

We protect personal information with industry-standard technical and organizational measures, including:
- Encryption in transit (TLS) and at rest (where technically feasible)
- Access controls and least-privilege provisioning for internal accounts
- Regular security reviews and dependency patching
- A documented incident-response process

No system is perfectly secure. If we become aware of a personal-data breach that is likely to result in a risk to your rights, we will notify you and the appropriate authorities within the timelines required by applicable law (typically 72 hours under GDPR).

## 10. Children

The Service is not directed to children under **16 (the Service is a professional developer tool, not directed to children)**. We do not knowingly collect personal information from children under this age. If you believe a child has provided us with personal information, contact us at sdt.ddt.tools@gmail.com and we will delete it.


**California minors:** Users under 18 in California may request removal of content they themselves posted by emailing sdt.ddt.tools@gmail.com.


## 11. Changes to this policy

We may update this Privacy Policy from time to time. If we make material changes, we will give you at least 30 days' notice by email or in-product notice before they take effect. The "Effective" date at the bottom of this document tells you when the current version started.

## 12. Contact

Privacy questions, requests, or complaints: sdt.ddt.tools@gmail.com

Mailing address:
Varol Consulting LLC
<<< DECISION NEEDED: postal address — none published (email-only contact); no legal entity formed yet. Counsel to advise whether a postal/agent-for-service address is required for the targeted jurisdictions. >>>

---

# Appendix A — DDT-specific disclosures (product addendum)

> **Why this appendix exists.** Sections 1–12 above are generated from a shared
> template. The `legal-compliance` builder has no mechanism to inject
> product-specific prose into the body, so the DDT-specific facts — and
> corrections to template language that does not fit this product — are
> collected here. Where this appendix conflicts with the templated body above,
> **this appendix controls.** A counsel pass should fold these into the body.

## A.1 No accounts, no passwords, no payments

DDT does not create user accounts, does not collect passwords, and does not
process payments. The beta is free and there is no payment processing in the
product. Accordingly:

- The reference to authenticating accounts and processing payments/billing in
  **Section 3** does not apply to DDT — we do none of those things.
- The "Account data … 30 days" and "Payment records … 7 years" bullets in
  **Section 7** do not apply. The only retention periods that govern DDT are the
  ones in the **Section 2** table (error records until fixed + notified or 12
  months; usage-ping aggregates 12 months; feedback until resolved or 12 months).

## A.2 Everything we collect is opt-in

DDT transmits **nothing** to us unless you choose to enable it. On first run you
are shown an explicit consent prompt for error reporting and the anonymous usage
ping; until you choose, nothing is sent. Feedback submissions are only sent when
you deliberately run `ddt feedback` or use VS Code's Report-a-Bug.

You can turn telemetry off, or keep it off, at any time:

- Set `DDT_TELEMETRY=0` in your environment.
- Set `DO_NOT_TRACK=1` (honored as a global opt-out).
- In VS Code, set `telemetry.telemetryLevel` to `off` (we respect the editor
  setting).
- Continuous-integration (CI) environments are auto-detected and telemetry is
  disabled there automatically.

## A.3 Client-side sanitization — your SQL and identifiers never leave your machine

Before any error report is sent, DDT redacts it **on your machine**: quoted
identifiers, string literals, and anything resembling a credential are stripped,
stack frames are reduced to application-only package-relative paths, and the
error message is sanitized. DDT **never** transmits your SQL, your catalog /
schema / table / column names, your file contents, your credentials, or absolute
filesystem paths. The error fingerprint is a one-way SHA-256 hash used only to
group duplicate reports.

## A.4 AI features are bring-your-own-key — we never see your AI traffic

DDT's AI features require **your own API key** for the AI provider you choose
(Anthropic, OpenAI, Azure OpenAI, an OpenAI-compatible endpoint, or a self-hosted
model). When you use an AI feature, your prompt goes **directly from your machine
to your chosen provider under your own account**. DDT does **not** see, proxy,
store, log, or route that traffic, and there is **no AI subprocessor on our
side**. Whatever you send to your AI provider is governed by that provider's
terms and privacy policy, not this one.

## A.5 Fix-notification emails and the right to be forgotten

If you give us a contact email (via `ddt feedback --contact`, or because it came
from an installed license), we use it for exactly one purpose: to notify you when
the bug you reported is fixed. We send those emails through MailChannels (or a
successor email service) — see **Section 4**.

You can have your email removed from all of our records at any time:

- **Email us** at sdt.ddt.tools@gmail.com and ask for deletion, or
- **Use the automated GDPR erasure endpoint** — a `DELETE` request to
  `/errors/reporter/:email` removes your email from every record that references
  it. This satisfies the erasure right described in **Section 8** for the
  reporter-email data category.

## A.6 No cookies

DDT is a VS Code extension and a command-line tool. It does **not** set cookies
or use web-tracking technologies. The reference to a Cookie Notice in
**Section 6** does not apply; no Cookie Notice is published for DDT.

## A.7 Public beta framing

DDT is distributed as a **30-day public beta**. During the beta every feature is
free. This policy is published by **Varol Consulting LLC**, the entity that
operates the Service; it is a `v0.1.0` unreviewed draft pending counsel review.
We may update it as the product and its legal posture evolve, with the notice
described in **Section 11**.

## A.8 Australia and Brazil

The templated body does not yet include tailored sections for two in-scope
jurisdictions; they are recorded here pending a counsel pass:

- **Australia (Privacy Act 1988 / Australian Privacy Principles):** Australian
  users may complain to the Office of the Australian Information Commissioner
  (OAIC) at https://www.oaic.gov.au/ if a privacy concern is not resolved with
  us first. The access and correction rights in **Section 8** apply.
- **Brazil (LGPD):** Brazilian users have the rights granted by the Lei Geral de
  Proteção de Dados, including confirmation of processing, access, correction,
  anonymization or deletion of unnecessary data, portability, and information
  about sharing. Requests go to sdt.ddt.tools@gmail.com. The supervisory
  authority is the ANPD (https://www.gov.br/anpd/).
  <<< DECISION NEEDED: Brazil — confirm whether an LGPD data-protection officer (encarregado) designation is required and, if EU/UK Art. 27-style representation is needed, the analogous LGPD posture. >>>

---

**Document metadata:** version v0.1.0 · effective 2026-06-02 · generated 2026-06-02T15:15:02 · jurisdictions: US-federal, US-California, EU, UK, Canada, Australia, Brazil
