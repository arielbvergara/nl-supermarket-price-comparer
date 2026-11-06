# Collecting AH and Jumbo data for private use: legal review

> **Checked on:** 2026-11-06. Terms, robots.txt files and case law change, so re-check them before each phase (§6).
>
> **Not legal advice:** this is a developer's reading of public sources, and this review found no ruling on this exact use. For certainty, get the store's written permission or ask a lawyer.

The [MVP plan](MVP-plan.md) §2 leaves one question open: may this project collect AH's and Jumbo's product and promotion data and use it on your own computer? This review answers it.

## 1. Short answer

| Store and host | Verdict | Why |
|---|---|---|
| AH, `www.ah.nl` paths that robots.txt allows | **Likely yes** | AH's terms allow copying for personal use and say nothing about automated access. robots.txt explicitly allows `/zoeken/api/` and the REST API under `/service/rest`. No opt-out covers these paths, so the text-and-data-mining (TDM) exception is available, and likely covers even the full promotions list (§4.3). |
| AH, `api.ah.nl` (the app's backend) | **No** | Its robots.txt shuts out every bot, which likely counts as an opt-out, and nothing licenses third-party clients of the app's backend. |
| Jumbo | **No, not under Jumbo's terms** | The terms ban scraping, crawling and repeated extraction. Mandatory law limits that ban, and the realistic risk is being blocked, but only Jumbo's written consent makes the use permitted. |

For both stores:
- Using the data on your own computer isn't the problem; collecting it is. Nothing is made available to the public, so the only act in question is extraction (§4.2).
- The database law's private-use exception doesn't cover online databases, so private use alone doesn't allow copying a large part of a catalogue (§4.2).
- Getting past a block, a bot check or a login turns a civil risk into a criminal one (§4.5).
- Publishing the data changes the answer (§8).

## 2. What this review assumes

The answer holds for the MVP as planned in [MVP plan](MVP-plan.md) §2:
- One person, private and non-commercial, on a laptop.
- Current promotions, watchlist searches, pinned products and EAN look-ups: a few hundred requests a day in total, one at a time, 1–2 seconds apart.
- No login, no account and no personal data.
- Nothing published. The app stays on `localhost`, and the public repository holds code and a few trimmed fixtures.

If any of these change, re-read §8.

## 3. What the stores say

### 3.1 AH

**Terms** ([ah.nl/algemene-voorwaarden](https://www.ah.nl/algemene-voorwaarden); Part II, version of 26 August 2024, last changed 28 June 2026):
- Part II, clause 14: publishing, copying or editing information from the site needs AH's consent, *"behalve voor persoonlijk gebruik"* (except for personal use).
- Part III covers the AH app. §1 licenses it for use on your own device and bans decompiling and reverse engineering it; §4 repeats the personal-use exception.
- No clause mentions robots, crawlers, scrapers, automated access, data mining or databases.

**robots.txt on `www.ah.nl`** (last updated 5 December 2025), for `User-agent: *`:
- Disallowed: account, login and ordering pages; internal paths such as `/data/`, `/ui/` and `/website/`; and URLs with a query string for search (`/zoeken?*`), product listings (`/producten?*` and `/producten/*?*`, apart from pagination and a few filters such as `Bonus=true`) and promotion groups (`/bonus/groep/*?*`).
- Explicitly allowed: `/zoeken/api/`, and `/service/rest`, which the file labels as the REST API.
- Product and promotion pages without a query string have no rule, so they're allowed.

**robots.txt on `api.ah.nl`:** `Disallow: /` for every user agent. This is the host the community clients in [MVP plan](MVP-plan.md) §13 call.

**Bot filter:** on 2026-11-06, `www.ah.nl` refused curl's default client (403 Access Denied) and answered a client whose User-Agent names this project (200).

### 3.2 Jumbo

**Terms** ([jumbo.com/service/algemene-voorwaarden](https://www.jumbo.com/service/algemene-voorwaarden); printable version dated 19 May 2026). Part II covers the website and the apps:
- Art. 1 counts database rights among Jumbo's intellectual property rights.
- Art. 9(1)–(2): Jumbo keeps all rights and grants a limited, non-transferable licence to use the site and apps for their intended purposes.
- Art. 9(3): no copying or publishing unless Part II or mandatory law allows it. It expressly bans extracting a substantial part of the content, repeatedly and systematically extracting insubstantial parts, using tools aimed at taking over information, and crawling, spidering or scraping with robots.

**robots.txt on `www.jumbo.com`**, for `User-agent: *`: disallows keyword-search and recipe-search URLs, the basket (`/mandje/`), account pages (`/mijn/`), orders (`/bestellingen/`) and recipe favourites. It has no rule for `/api`, product pages or promotion pages.

**`mobileapi.jumbo.com`** (the app's backend): no robots.txt found (one 404, one timeout).

### 3.3 Machine-readable opt-outs

Neither store has a TDM reservation file (`/.well-known/tdmrep.json`: AH 204, Jumbo 404), sends a `tdm-reservation` header, or names AI crawlers in robots.txt. The robots.txt rules above are the only machine-readable signals.

## 4. The law

### 4.1 Facts and copyright

Prices, promotion terms, product names, EANs and unit prices are facts. Copyright doesn't protect facts, and the database right isn't copyright in facts either: it protects the investment behind a collection (Database Directive 96/9/EC, recital 45; §4.2).

Product descriptions and photos may be works under the *Auteurswet* (Aw). The MVP stores image URLs, not images, and keeps raw responses for 30 days on your laptop ([MVP plan](MVP-plan.md) §5.2). Two exceptions cover those copies:
- Art. 16b Aw: a natural person may make a few copies for their own use, with no commercial aim.
- Art. 15o Aw: copies made for text and data mining may be kept as long as the mining needs them (§4.3).

### 4.2 Database right

The *Databankenwet* (Dbw) protects a collection whose obtaining, verification or presentation took a substantial investment (art. 1(1)(a)). Its maker controls:
- extraction (*opvragen*) and re-utilisation (*hergebruiken*) of the whole or a substantial part (art. 2(1)(a));
- repeated and systematic extraction of insubstantial parts, but only where it conflicts with the database's normal exploitation or unjustifiably harms the maker's legitimate interests (art. 2(1)(b)).

Extraction is transferring content to another medium, permanently or temporarily (art. 1(1)(c)), so saving responses in Postgres counts. Re-utilisation is making content available to the public (art. 1(1)(d)), which the MVP never does.

Is a supermarket's online catalogue protected? Probably, given the work of gathering and checking supplier data and keeping it current. But under the CJEU's *BHB* ruling (C-203/02, 9 November 2004), investment in *creating* the data, such as a store setting its own prices, doesn't count, so protection is less certain than it looks. This review assumes the catalogues are protected, which is the cautious reading.

The rules that decide the answer:
- **No private-use exception online.** The Dbw allows extracting a substantial part for private purposes only from a non-electronic database (art. 5(1)(a)).
- **Lawful users may always take insubstantial parts.** The maker of a public database can't stop a lawful user (*rechtmatige gebruiker*) from extracting insubstantial parts, and no contract can take that right away (art. 3). In return, the user mustn't endanger the database's normal exploitation or cause the maker unjustified harm (art. 4).
- **Text and data mining** by someone with lawful access is allowed unless the maker has reserved it (art. 4a(b); §4.3).
- **Harm to the investment.** In *CV-Online* (C-762/19, 3 June 2021), the CJEU held that a maker can stop extraction only where it adversely affects the maker's investment, that is, its chance to recoup it.

Applied to the MVP ([MVP plan](MVP-plan.md) §2.3):
- A watchlist search fetches one page of results (20 products by default), and a pin refresh or EAN look-up fetches one product. These are insubstantial parts, which art. 3 protects. Repeating them daily matters only if it conflicts with the store's normal exploitation or unjustifiably harms it (art. 2(1)(b) and art. 4). A private tool that sells nothing and publishes nothing is unlikely to, though no ruling confirms it.
- The daily list of all current promotions may be a substantial part of a store's promotions. It needs the TDM exception (available on AH's allowed paths, doubtful for Jumbo; §4.3), or the argument that a private tool harms no investment (*CV-Online*).
- Keeping and using the data on your own computer adds nothing: without a public, there's no re-utilisation.

### 4.3 Text and data mining and opt-outs

Art. 15o Aw and art. 4a(b) Dbw implement art. 4 of the DSM Directive (2019/790). Anyone with lawful access may copy and extract content for text and data mining, unless the rightholder has expressly reserved that right in an appropriate way; for content online, only by machine-readable means (DSM art. 4(3) and recital 18). Content that's freely available online counts as lawfully accessed (DSM recital 14). The copies may be kept as long as the mining needs them.

- **What counts as a reservation.** DSM recital 18 says machine-readable means include metadata and a website's terms and conditions.
- **Case law.** In *DPG Media e.a. v HowardsHome* (Rb. Amsterdam, 30 October 2024), the publishers' robots.txt files blocked only some AI crawlers (GPTBot, ChatGPT-User, CCBot and anthropic-ai). The court held that the publishers hadn't shown an express, machine-readable reservation, so the TDM exception applied, and it dismissed the database-right claim on the same ground. It's a first-instance ruling, and an appeal was filed (status as of April 2026).
- **Is a price tracker TDM?** Plausibly. The DSM Directive (art. 2(2)) defines TDM as any automated analysis of digital text and data that generates information, such as patterns, trends and correlations; price histories and promotion patterns fit. This review found no ruling on a price tracker, though.

Applied:
- **AH, allowed `www.ah.nl` paths:** no reservation applies, so the exception is available.
- **AH, `api.ah.nl`:** `Disallow: /` for every bot is a general reservation, unlike the partial one in *HowardsHome*.
- **Jumbo:** the terms ban automated extraction, and recital 18 counts website terms as a way to reserve, so the exception is likely unavailable. *HowardsHome* was argued over robots.txt, so it doesn't settle whether a ban in written terms is enough.

### 4.4 Contract

Terms of use bind you only as part of an agreement (art. 6:217 BW; for general terms, art. 6:231 ff. BW). Having an account or placing an order means you accepted the store's terms. Whether they bind an anonymous visitor who only browses is doubtful.

Where they bind:
- In *Ryanair* (C-30/14, 15 January 2015), the CJEU held that terms can restrict the use of a database that neither copyright nor the database right protects. For a protected database, a lawful user's rights under art. 3 Dbw can't be contracted away.
- AH's terms help you: they allow personal use.
- Jumbo's terms are against you. Art. 3 Dbw protects taking insubstantial parts, but it's unclear whether that also overrides a ban on a *method* (robots), and it does nothing against the ban on substantial parts. If Jumbo's catalogue isn't protected (§4.2), the contract alone decides.
- A breach would realistically mean being blocked, or losing a Jumbo account. In theory a store could seek an injunction or damages, but a private tool causes little damage to show.

### 4.5 Criminal law

*Computervredebreuk* (art. 138ab Sr) is intentionally and unlawfully intruding into a computer system. Access counts as intrusion in any case when it's gained by breaking a security measure, by a technical intervention, with false signals or a false key, or by assuming a false identity (*valse hoedanigheid*). The maximum is 2 years in prison, or 4 if the intruder then copies data (art. 138ab(2)).

- Requesting public pages that need no login is unlikely to be intrusion: there's no barrier to get past.
- Getting past a block, a bot check, a CAPTCHA or a login, or posing as a browser or the store's app to get access, could be. That's why stopping when blocked ([MVP plan](MVP-plan.md) §2.1) is a legal line, not just etiquette.
- An honest User-Agent that names this project isn't a false identity. If a store refuses it, don't answer by copying a browser's.

### 4.6 Privacy

The MVP collects product and price data, not personal data, so the GDPR (*AVG*) doesn't come into it.

## 5. Verdict per store

| | AH, allowed `www.ah.nl` paths | AH, `api.ah.nl` | Jumbo |
|---|---|---|---|
| Terms | allow personal use; silent on automation | nothing licenses third-party clients of the app's backend | ban scraping, crawling and repeated extraction |
| robots.txt | allowed; `/zoeken/api/` and `/service/rest` explicitly | everything disallowed | no rule for API, product or promotion pages |
| TDM exception | available | likely reserved | likely reserved by the terms |
| Database right | small look-ups are insubstantial; the promotions list falls under TDM | doesn't arise | small look-ups are protected by art. 3; the promotions list needs consent or the no-harm argument |
| **Verdict** | **likely permitted** | **don't use** | **not permitted by Jumbo's terms**: low exposure, but needs consent |

## 6. Rules the MVP adopts

[MVP plan](MVP-plan.md) §2 and §13 carry these rules:
1. **Robots-allowed paths only.** For AH, that means `www.ah.nl` paths such as `/zoeken/api/`, not `api.ah.nl`. Phase 0 checks every endpoint against its host's robots.txt.
2. **An honest, fixed User-Agent** that names this project; never a browser's or a store app's identity.
3. **Never get past a block, a bot check, a CAPTCHA or a login.** Stop and decide what to do.
4. **No login and no account data.**
5. **Publish nothing but trimmed fixtures:** facts such as names, prices and promotion labels, without descriptions or images.
6. **Re-check** both stores' terms and robots.txt before each phase and whenever a store starts blocking, and update this review's date.

## 7. Decision needed: Jumbo

Before any automated Jumbo request, including the Phase 0 spike, choose one:
- **A. Ask Jumbo for written permission (recommended).** It's the only way to make the use permitted. The AH work goes ahead meanwhile, and a refusal settles the question.
- **B. Go ahead without asking,** within the rules in §6, knowingly against Jumbo's terms. The legal exposure is low (§4.2, §4.4), and the realistic outcome is being blocked.
- **C. Replace Jumbo** with another store or source, after the same review.

**Status:** open.

## 8. What would change the answer

- **Publishing the data,** or running a public comparison site, is re-utilisation, which the TDM exception doesn't cover. Licensed access exists: Supermarktscanner says it has direct data connections with AH, Jumbo and PLUS.
- **Commercial use:** get legal advice and a licence first.
- **Logging in:** the terms then clearly bind you, and a breach can cost you the account.
- **Changed terms or robots.txt:** redo §3 and §5.
- **The *HowardsHome* appeal:** a different view of what counts as a reservation would change §4.3.

## 9. Sources

All retrieved on 2026-11-06.

**Stores**
- AH terms: <https://www.ah.nl/algemene-voorwaarden>
- AH robots.txt: <https://www.ah.nl/robots.txt> and <https://api.ah.nl/robots.txt>
- Jumbo terms: <https://www.jumbo.com/service/algemene-voorwaarden>; printable version: <https://www.jumbo.com/dam/service/jumbo-extra/20260519-Algemene-en-aanvullende-voorwaarden-Jumbo-B2C.pdf>
- Jumbo robots.txt: <https://www.jumbo.com/robots.txt>
- Supermarktscanner, about page: <https://www.supermarktscanner.nl/over-ons>

**Legislation**
- *Databankenwet*: <https://wetten.overheid.nl/BWBR0010591/>
- *Auteurswet*: <https://wetten.overheid.nl/BWBR0001886/>
- *Wetboek van Strafrecht*, art. 138ab: <https://wetten.overheid.nl/BWBR0001854/>
- *Burgerlijk Wetboek* Book 6: <https://wetten.overheid.nl/BWBR0005289/>
- Database Directive 96/9/EC, recitals: <https://www.legislation.gov.uk/eudr/1996/9/introduction/adopted>
- DSM Directive 2019/790, recitals: <https://www.legislation.gov.uk/eudr/2019/790/introduction/adopted>; art. 2: <https://www.legislation.gov.uk/eudr/2019/790/article/2/adopted>; art. 4: <https://www.legislation.gov.uk/eudr/2019/790/article/4/adopted>

**Case law** (this review relies on summaries, not the full judgments)
- *BHB* (C-203/02, 9 November 2004): <https://curia.europa.eu/juris/liste.jsf?num=C-203/02>
- *Ryanair* (C-30/14, 15 January 2015): <https://curia.europa.eu/juris/liste.jsf?num=C-30/14>; summary by Pinsent Masons: <https://www.pinsentmasons.com/out-law/news/website-operators-can-prohibit-screen-scraping-of-unprotected-data-via-terms-and-conditions-says-eu-court-in-ryanair-case>
- *CV-Online* (C-762/19, 3 June 2021): <https://curia.europa.eu/juris/liste.jsf?num=C-762/19>; summary by SCL: <https://www.scl.org/12290-cjeu-search-engine-copying-of-databases-infringes-sui-generis-right-where-it-adversely-affects-database-maker-investment/>
- *DPG Media e.a. v HowardsHome* (Rb. Amsterdam, 30 October 2024), CMS case tracker: <https://cms.law/en/nld/publication/artificial-intelligence-and-copyright-case-tracker/the-netherlands-dpg-media-b.v.-mediahuis-nederland-b.v.-mediahuis-nrc-b.v.-v-knowledge-exchange-b.v.-trading-as-howards-home-30-october-2024>
