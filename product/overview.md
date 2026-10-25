Product · Web · iOS · Android · October 2026

# Tally Aloft — Product Overview

Tally Aloft is a pilot logbook for the web, iOS and Android. It aims to be the record pilots trust most: one that never loses hours, holds up at a checkride or an airline interview, and can always be exported. This document summarizes what the product is for, who it serves and what it does for pilots. The market research behind it is in the [market analysis](market-analysis.md), and how it's built is in the [technical overview](../technical/overview.md). Terms such as REED and LADD are explained in the [glossary](../glossary.md).

## Purpose

A pilot logbook is a legal record (14 CFR 61.51), and for a career pilot it's also a hiring document. The market analysis found that pilots' biggest complaints about current logbooks are about trust, not appearance: sync bugs that lose hours, support that never answers, prices that keep climbing, and lock-in to one platform or EFB subscription. Pilots rarely switch logbooks. When they do, it's at a few moments: when they digitize years of paper history, when they start instructing, and when they get ready for an airline interview.

Tally Aloft is built for those moments. Its job is to be **the trustworthy keeper of a pilot's legal record**. It gets that record in cleanly, checks it, protects it, and turns it into the documents an examiner or interviewer asks for.

Now is a good time to try, for three reasons:

- The U.S. pilot population is growing: 887,519 active pilots at the end of 2025, with commercial certificates up 7.8%.
- The incumbents are consolidating. ForeFlight is now owned by private equity and raised prices in 2026. LogTen belongs to a flight-school scheduling company and is still Apple-only. That gives pilots a reason to want a vendor-neutral record.
- No current product combines a modern design, iOS, Android and web support, reliable airline schedule import, a good support reputation and a fair price.

## Who it's for

**Primary segment:** U.S. career-track pilots, from CFIs and time-builders through the first five years as a regional first officer. The emphasis is on Android users, whom the premium apps don't serve. These pilots pay for their tools, and the hiring market rewards precisely documented multi-engine, turbine and PIC time.

**Students** come in through a free tier. That reaches pilots years before the airline transition, ideally before they commit to LogTen.

**Not targeted at launch:**

- GA hobbyists, who are loyal to the free MyFlightbook or to EFB bundles they already pay for.
- EASA markets, where logbook acceptance is decided country by country.

## Positioning

The pitch is **"the interview-ready logbook."** The analysis reduces it to three steps: digitize and audit your history, keep logging in a logbook you trust, and produce an interview packet with one tap.

These commitments support the positioning:

- **Data integrity first.** A dropped connection or a retry never loses or duplicates an entry, every edit is kept, and a signed entry can never be quietly changed. Backups are kept in a separate location and tested regularly.
- **Guaranteed export.** A full CSV and PDF export is available on every plan, including after a subscription lapses. This answers the "what if the developer disappears" concern raised about MyFlightbook.
- **Honest compliance claims.** Electronic signatures are designed around the attributes in AC 120-78B. The plan is to publish an attribute-by-attribute compliance mapping. The product is never marketed as "FAA-approved," because no such approval exists for Part 61 and Part 91 records.
- **"AI-drafted, pilot-verified."** Digitized entries stay drafts until the pilot confirms each one against the source page.

## Features

No source document defines the first release yet, so most statuses below are proposals.

- **v1**: proposed for the first release. Only rows marked *decided* reflect a decision.
- **Staged**: ships in steps, and the timing of the first step isn't decided.
- **Later**: comes after the first release, and its design is still open.

Which features each pricing tier includes is also undecided (see [Open questions](#open-questions)).

### The wedge: getting a pilot's history in, clean

| Feature | Status | What it does |
| --- | --- | --- |
| Paper-logbook digitization | v1 | The pilot photographs paper logbook pages. The phone finds the page edges and straightens the photo. AI drafts the entries, and the pilot confirms each one against the page image. Sold as a one-time purchase together with the audit. It replaces human services that charge hundreds of dollars. |
| Pre-interview audit | v1 | Flags errors: column sums that don't match page totals, carry-forward mistakes, dual entries missing CFI signatures or certificate data, entries logged as cross-country that don't meet the definition, and gaps in dates. Audit quality, not OCR alone, is the edge over the AI scanning apps that launched in 2025–26. Whether the audit also runs on imported logbooks is open. |
| Imports from other logbooks | v1 | Imports ForeFlight, LogTen, MyFlightbook and Garmin CSV files, then **recalculates and shows currency** so the pilot can see whether the import came through intact. Signatures made in other apps can't be verified again, so they're imported as images, with a record of where they came from. |

### Core logbook

| Feature | Status | What it does |
| --- | --- | --- |
| Flight entry | v1 | Fast entry on the phone, and a full logbook grid on the web with sorting, filtering and column choices. Aircraft details prefill from the FAA registry by tail number. Every edit is kept in the entry's history. |
| Totals | v1 | Totals by aircraft and by time column, identical on every device. Duty-time tracking is an open question. |
| 8710 / IACRA totals | v1 | The time summary an applicant enters on FAA Form 8710 or in IACRA for each certificate or rating. Competitors treat this as standard. |
| Currency and expiry reminders | v1 | Currency status in the app, plus push and email reminders before deadlines. Instructor rules are based on the REED, because flight instructor certificates no longer expire. |
| Printable logbook | v1 | PDF logbooks for any date range. The app prepares the file in the background and offers it for download when it's ready. |
| Interview packet | v1 | A PDF with tabbed milestones, an appendix of endorsements and reconciled totals. No competitor offers this. |
| Export and account deletion | v1, every plan | Export as above. Account deletion in every app, with an export offered first. The website explains how to delete an account without the mobile app. |
| Sign-in | v1 | Passkeys, MFA and social sign-in. Sign in with Apple is offered wherever iOS offers another social login, as App Store rules require. |

### Instructors

| Feature | Status | What it does |
| --- | --- | --- |
| Correct-and-sign workflow | v1, *decided* | The CFI can fix a student's entry and then sign it. ForeFlight makes the CFI reject the entry instead. Signing requires the instructor to sign in again with MFA and to confirm explicitly. |
| Tamper-evident signatures | v1, *decided* | Each signature covers one specific version of an entry, so any change to a signed entry can be detected. A correction creates a new version that needs a new signature, and the earlier version keeps its valid signature. |
| Endorsements | v1, *decided* | Endorsements record the instructor's certificate number and REED. From March 1, 2027, training entries must show the instructor's REED (14 CFR 61.51(h)(2)(ii)), so this must be ready by then. Instructor certificates are checked against the FAA airmen registry. |
| Endorsements-given report | v1 | A PDF of every endorsement an instructor has given, which ForeFlight lacks. |
| Instructor record retention | v1, *decided* | Instructors must keep endorsement records for three years (14 CFR 61.189), so the instructor's copy survives even if the student deletes their account. |
| Check-airman sign-offs | Later (B2B), *decided* | These require the operator's FAA authorization (OpSpec A025), so they'll be sold to Part 121 and Part 135 operators rather than shipped as an app feature. |

### Professional pilots

| Feature | Status | What it does |
| --- | --- | --- |
| Airline schedule import | Staged, Pro tier, *decided* | Imports a month of flying in one step, which is the main reason airline pilots pay. It works from files, calendar (ICS) feeds and the share sheet. It **never stores airline credentials**, both for crew security and because airlines have shut down credential-based apps before. Formats are added one airline at a time, which makes this the longest piece of work. |

### Platforms

There are three apps: web, iOS and Android, all showing the same logbook. Whether they launch together is open. Each app is designed for its device rather than sharing screens: the web app handles bulk entry and review, and the phone handles quick logging and page capture. Version 1 needs a connection to view or save entries (see Deferred), but whatever the pilot types stays on the device until it's saved, so a dropped connection loses nothing. Older app versions keep working for months, and one too old to support asks the pilot to update.

The web app is at tallyaloft.app. A public website at tallyaloft.com presents the product: features, pricing, principles, support, the privacy policy and terms, and how to delete an account.

### Deferred

| Feature | Notes |
| --- | --- |
| Offline use | Viewing and logging with no connection. Each device keeps a copy of the pilot's logbook, and changes made offline upload when the connection returns. Version 1 is built so this can be added later. |
| Crew social features | Following friends and same-city layover alerts. If built, they'll be opt-in and city-level only, will never show hotels, and will expire after a set time. Airlines treat crew location as a security issue. |
| Sharing flights with family | A live view of the pilot's flight for people they invite. How the flight is tracked is undecided. The analysis suggests sharing the pilot's own phone GPS through expiring links. Any design that shows FAA-derived data must follow the LADD and PIA rules. |

## Business model

This is the current direction, and the numbers aren't final.

| Tier | Who | Direction |
| --- | --- | --- |
| Free | Student pilots | Free through the private pilot certificate or a set number of hours, whichever comes first. This is LogTen's model, and the limits aren't set. |
| CFI | Instructors and time-builders | A discounted tier to win the time-building years. |
| Pro | Professional pilots | About $69–79 a year with schedule import included, below LogTen Pro's $129.99. |
| Digitization and audit | Anyone with paper history | A one-time purchase of about $99–299, depending on the number of entries. |

- **Billing:** on the web where possible, plus the App Store and Google Play. A subscription bought on any of them works in every app.
- **No lifetime licenses.** CrewLounge's conversion of lifetime licenses to subscriptions still draws accusations years later.
- **No generous freemium tier.** A free tier limited by usage converts better.
- **Distribution:** flight-school bulk licenses, trials through pilot associations such as NGPA, a booth at AirVenture, and aviation podcasts.
- **Viability target:** at $69–79 a year, about 900–2,050 paying subscribers produce $5k–$10k in monthly revenue. That figure is after store fees and before VAT and operating costs. It's under half a percent of U.S. certificated pilots.
- **Validate early:** the analysis suggests pre-selling digitization audits to CFIs and regional applicants before the full logbook is built.

## Risks

- **Sync and data loss.** This is the category's top complaint. One data-loss bug would bring the reviews that hurt CrewLounge and FlyLog.
- **Online-only v1.** The analysis says the cross-platform advantage only counts if sync works offline and never loses data, but offline use is deferred. Until then, the apps keep unsent input on the device, but pilots can't view or log without signal.
- **Calculation errors.** A wrong currency or duty calculation would destroy credibility.
- **Signature compliance.** Signatures must meet the AC 120-78B attributes, and the product must never claim FAA approval. A draft change to AC 120-78B was out for industry comment in February 2026 and may change the requirements.
- **Crew privacy.** Any feature that reveals where crews will be is a liability. So is schedule import, if it ever stored airline credentials.
- **The digitization lead is shrinking.** Several AI scanning apps launched in 2025–26, so the advantage has to come from audit quality and verification.
- **Hiring cycles.** Professional demand rises and falls with airline hiring, which fell about 60% in 2024. CFIs and time-builders keep logging through downturns.
- **Incumbents can catch up.** ForeFlight or LogTen could close the printing, CFI-editing or iOS schedule-import gaps. LogTen's iOS schedule import is already in beta.
- **Solo-developer risk.** Guaranteed export and a published commitment to data continuity are the answer.

## Open questions

- **Launch scope and order.** Which v1 rows ship first? Does a pre-sold digitization and audit service come before the full logbook, as the analysis suggests? Do all three apps launch together?
- **Offline in v1.** Version 1 needs a connection. Is a downloaded PDF enough for students who must carry their endorsements on solo cross-countries?
- **Tier contents.** Which features are free, which are in the CFI tier and which are Pro? Is the interview packet paid? Does the audit run on imported logbooks?
- **Duty tracking.** The original feature list included totals by duty. Is Part 117 or Part 135 duty and rest tracking in scope? A wrong calculation is a credibility risk, and competitors cover Part 135 unevenly.
- **Family flight sharing.** Will flights be tracked by phone GPS, ADS-B or other data, or something else? The answer decides whether the LADD rules apply.
- **Final pricing.** The tier prices, the free tier's limits and the size of the CFI discount.
