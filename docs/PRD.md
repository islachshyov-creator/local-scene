# PRD: LOCAL SCENE — Cinema Aggregator for Independent Prague Cinemas
*(PRD = Product Requirements Document — the single source of truth for what we're building, why, and how.)*

**Version:** 0.2 (In Review)
**Author:** Ivan Slachshyov
**Status:** In Review
**Last updated:** June 2026

---

## 1. Problem Statement

People in Prague who want to visit independent cinemas have no convenient single tool to discover showtimes, understand what's playing, and evaluate films — especially in English. Existing solutions (kinobox.cz) are technically mobile-accessible but the UX is overwhelming and unenjoyable — too many options, difficult navigation, low sense of discovery. They also lack meaningful filtering by cinema type and provide no reliable English-language experience. *Hypothesis to validate in user interviews (Phase 0).*

**Who feels this most acutely:**
- Expats and long-term foreign residents (~100k+ in Prague) who have limited Czech
- International tourists looking for cultural experiences beyond mainstream cinema
- Czech cinephiles who prefer arthouse / independent venues

**Evidence of demand:**
- Founder ran a manual Telegram channel curating independent cinema schedules for expat communities — sustained weekly effort of 1+ hour per issue, with organic growth from expat groups
- This project is the direct product evolution of that manual workflow

**Scope communication (UX requirement):**
> LOCAL SCENE covers independent and arthouse cinemas only. This must be communicated at the entry point — users should never discover mid-session that mainstream chains are not listed. Suggested: tagline makes it clear ("Prague's independent cinema guide"), with a small note directing multiplex seekers to kinobox.cz.

---

## 2. Vision

> LOCAL SCENE is the go-to platform for discovering independent cultural venues in Prague — starting with cinema.

LOCAL SCENE is built as an **umbrella brand** from day one, designed to expand beyond cinema into theatre, live music, and galleries. The cinema vertical is the MVP because it has the clearest data structure, the strongest personal founder insight, and the most immediate expat/tourist use case.

**Geographic expansion (future):**
- v2 → Brno, Plzeň
- v3 → Other Czech cities (Olomouc, Ostrava, České Budějovice)
- v4 → Full Czech Republic coverage
- *Strategic decision pending: depth in Prague (more verticals) vs breadth (more cities) — to be discussed post-MVP.*

---

## 3. Goals & Success Metrics

### One Metric That Matters (OMTM) — Pre-launch
> Number of target users who confirm they would use this weekly (from interviews + Reddit research). Target: 15+ unprompted positive signals before build starts.

### Product Goals (MVP — 3 months post-launch)
| Goal | Metric | Target | Notes |
|------|--------|--------|-------|
| Become a useful weekly tool | Weekly Active Users | 300+ WAU | Baseline estimate, to be revised after 30 days of real data |
| Data reliability | Pipeline uptime | >95% | Automated monitoring, not manual spot-checks |
| Data freshness | Average data lag | <4 hours | Pipeline runs every 3–4 hours |
| Mobile usage | Device mix | Tracked from launch | Hypothesis: mobile >70%; target set after 30 days of data |

*Device mix tracked across three segments: mobile / desktop / tablet.*

### What success is NOT
- Revenue in the first 6 months (this is a portfolio + community project first)
- Comprehensive coverage of all Prague cinemas (multiplex chains are explicitly out of scope)

---

## 4. Target Users

### Primary: The Expat Cinephile
- Lives in Prague 1–3 years+, conversational Czech but not fluent
- Wants to experience local culture, not just Palác Cinemas
- Frustrated by Czech-only interfaces and inability to quickly evaluate films (ratings, trailers)
- **Frequency:** Weekly or bi-weekly cinema-goer

### Secondary: The International Tourist
- 3–7 day stay, culturally curious
- Wants something authentic, not a multiplex
- Needs English entirely — no fallback to Czech
- **Frequency:** One-time, but high intent

### Tertiary: The Czech Cinephile
- Knows the scene, but values aggregation convenience
- Appreciates having IMDb/RT scores alongside ČSFD
- **Frequency:** Weekly

---

## 5. Scope — MVP

### In Scope
- [ ] Showtimes aggregated from 8–10 independent Prague cinemas (see Appendix A)
- [ ] Automated data pipeline every 3–4 hours: scraping/parsing + AI normalisation + confidence scoring
- [ ] TMDb API integration: posters, EN descriptions, trailers, IMDb rating, genres
- [ ] DeepL integration: CZ → EN and CZ → RU for titles and descriptions where TMDb has no native translation
- [ ] Filters: by cinema / genre / date / language (VO / titulky) — all on one mobile screen, no scroll required; live filtering with no apply button
- [ ] Film detail page:
  - TMDb description (concise, factual — no AI-generated opinion summaries)
  - Three ratings with brand icons, all clickable to source: ČSFD → IMDb → RT (order adapts to user language)
  - Trailer embed
  - Age restriction (format: 12+, 15+, 18+ — universal numeric, prominent placement)
- [ ] Cinema page: name, address, map (Google Maps / Mapy.cz toggle, language-aware default), short description, transport guide (see note below)
- [ ] Languages: CZ + EN + RU (UI + content)
- [ ] Language detection: browser/device language → auto-select UI language; manual override saved to localStorage; default EN if undetected
- [ ] Language switcher: persistent text labels (CZ / EN / RU) in sticky header, always visible
- [ ] Mobile-first responsive design
- [ ] "Getting there" section on cinema pages: directions deep link + ticket buying guide (tap contactless card or Apple/Google Pay on validator — no app needed for tourists)
- [ ] Feedback contact: persistent footer link "Have suggestions or ideas? Let me know →" (Tally.so form or email)
- [ ] LanguageTool API integration: grammar and diacritics check on all text output (CZ/EN/RU)
- [ ] All outbound data points (ratings, reviews links) open source pages — no dead ends

### Out of Scope (MVP)
- Online ticket purchase / booking → deep link to cinema's own ticket page instead
- User accounts, watchlists (v2 — with Google/Apple one-tap login when implemented)
- UA language version (v2)
- Theatre, music, gallery verticals (v2+)
- Personalisation / recommendations
- Monetisation features
- IP geolocation for language detection (overkill for MVP)

### UX North Star
> **Minimum steps to value.** Every flow is evaluated by how few taps/clicks it takes to get the user what they need. Any feature that adds steps without clear user value gets cut or redesigned. This principle overrides feature completeness.

### One-window philosophy
> Users should get 80% of what they need without leaving LOCAL SCENE. For the remaining 20% — clearly labelled deep links out.

---

## 6. Technical Approach

### Data Pipeline Architecture

```
Source sites (cinema websites) or API/feed if available
        ↓
Scraper / Parser layer (Playwright or fetch+cheerio, per-cinema module)
        ↓
Raw data → AI Normalisation (Claude API)
  • Extract: film title (CZ), showtime, cinema, hall, language flag (VO/titulky)
  • Confidence score per record (0.0–1.0)
  • CZ diacritics normalised for matching only (display always preserves diacritics)
        ↓
≥ 0.85 confidence → auto-write to Supabase
< 0.85 confidence → flagged queue (manual review)
        ↓
TMDb title matching (multi-source pipeline):
  1. TMDb search by CZ title → check year match
  2. If no match → ČSFD page scrape (EN title listed below CZ name)
  3. If no match → Google search "{CZ title} site:imdb.com"
  • Confidence score: ≥2 sources agree → auto-match; 0–1 → flagged
        ↓
TMDb metadata pull (description EN, poster, trailer, IMDb rating, genres)
        ↓
DeepL: EN → RU for descriptions; CZ → RU/EN only for films with no TMDb coverage
        ↓
LanguageTool API: grammar + diacritics check on all text output
        ↓
Frontend (Next.js on Vercel) reads from Supabase
```

### Scheduled runs
- Pipeline: every 3–4 hours via GitHub Actions cron (`0 */4 * * *`)
- Alert: email/Slack notification if pipeline error rate >10%
- Manual trigger: available for urgent corrections (e.g. cancelled screening)

### Stack
| Layer | Tool | Cost |
|-------|------|------|
| Frontend | Next.js / Vercel | Free |
| Database | Supabase (free tier) | Free |
| Scraping | Playwright (Node.js) | Free |
| AI normalisation | Claude API (claude-haiku) | ~$5–15/mo |
| Film metadata | TMDb API | Free |
| Translation | DeepL API (free tier) | Free |
| Grammar/diacritics | LanguageTool API | Free tier |
| Domain | localscene.cz | ~300 CZK/yr |
| Monitoring | UptimeRobot | Free |

### Key Technical Risks
- **Parser fragility:** Cinema sites change layouts. *Mitigation: first ask all cinemas for API/feed (Phase 0); modular per-cinema parser files isolate breakage; confidence scoring auto-flags when a parser breaks.*
- **TMDb matching accuracy:** Czech film titles + diacritics may not match TMDb. *Mitigation: multi-source matching pipeline (TMDb → ČSFD → Google); diacritics stripped for matching only, preserved for display.*
- **Facebook-only venues:** Some smaller venues post schedules only on Facebook. *Mitigation: direct partnership outreach ("we'll feature you free, share your schedule"); RSS from public Facebook pages as fallback.*

---

## 7. User Experience — Key Flows

*Note: flows below are hypotheses. To be tested personally in Figma prototype before dev, then validated with 2–3 real users.*

### Three user mental models (to validate in Phase 0):
- **Showtime-first:** "What's on tonight?" → starts with date
- **Cinema-first:** "What's Aero showing this month?" → starts with venue
- **Discovery-first:** "Find me a great film" → starts with ratings/genre *(lowest priority — limited value if film is >3 days away)*

### Flow 1: "What's on tonight?"
Home → filter by date (today) → browse results → tap film → ratings / trailer / age restriction → tap through to cinema site for ticket

### Flow 2: "I want something in English"
Home → filter language = VO → browse by IMDb rating → pick film → confirm showtime → directions

### Flow 3: "What's my favourite cinema showing?"
Cinema page → upcoming programme → tap film → details

---

## 8. Launch Plan (Phased)

### Phase 0 — Lean Validation (Weeks 1–2, before any code)

**Week 1: Desk research (~8–10 hours)**
- Full competitive analysis: kinobox.cz, csfd.cz, GoOut.net, individual cinema sites, analogues in Warsaw/Budapest/Vienna/Berlin
  - *Method: use each as a target user for 20–30 min; check App Store reviews; search Reddit for complaints; note in comparison table: Strengths / Weaknesses / Missing entirely / User complaints*
  - *Google keywords: "prague cinema expats", "what's on prague english", "indie cinema prague app", "kinobox alternatives", "prague film guide english"*
  - *Platforms: Reddit (r/prague, r/czechrepublic), ProductHunt ("cinema guide")*
- Check ČSFD ToS (footer → Podmínky užití; email if unclear)
- Check RT enforcement posture for small sites → decision: link only vs display score
- TMDb coverage test: manually check 10 Czech indie films → confirm matching viability
- Check if any target cinemas have existing public data feeds (iCal, XML, JSON)

**Week 2: Primary research (~6–8 hours)**
- Reddit/Facebook expat groups: search for existing threads about Prague cinema/culture tools
  - *Groups: Prague Expats, Americans in Prague, British Expats Prague, Prague International Community*
  - *Keywords inside groups: "cinema", "movie", "film", "what's on", "kinobox", "GoOut"*
  - *Post Option A (problem-focused): "How do you find out what's playing at independent cinemas in Prague? I always spend 20 minutes clicking through individual sites — wondering if there's something better I'm missing."*
- 5 user interviews (expats in Prague, 20 min each)
  - *Purpose: understand WHY people behave as they do, not statistical validation*
  - *Key questions:*
    1. What's your current workflow for finding indie cinema in Prague?
    2. Have you ever missed a film because you didn't know it was showing?
    3. What language do you default to when searching for Prague events?
    4. Have you used kinobox / GoOut? What did you think?
    5. Would you check a weekly newsletter vs an app vs a website?
    6. When you want to go out, do you start with "what's on tonight" or "what's my favourite cinema showing"?
    - *Free space: "Anything else you want to add?"*
- Email all target cinemas (see Appendix A + B):
  - *Questions to ask:*
    1. Do you have an API or data feed for your programme?
    2. Do you have an XML/JSON/iCal export of showtimes?
    3. Do you use a third-party ticketing platform (GoOut, Ticketmaster, etc.)?
    4. Would you be willing to share a Google Calendar or spreadsheet feed?
    5. Do you have an English version of your programme/descriptions?
    6. Who is the right contact for ongoing data/partnership questions?
- Compile all findings → validate or kill top assumptions (see below)

**Decision gate (end of Week 2):**
> Review findings. Do they support building MVP as specced?
> - Yes → proceed to Phase 1
> - Partially → revise PRD scope before building
> - No → pivot before spending 6 weeks building the wrong thing

**Top assumptions to validate in Phase 0:**
- [ ] Expats actively want this (Telegram = proxy, not hard validation)
- [ ] Independent cinemas will cooperate with data sharing
- [ ] TMDb has adequate Czech film coverage
- [ ] Users are showtime-first or cinema-first (not discovery-first)
- [ ] kinobox UX is the real problem (vs low awareness of indie cinemas generally)
- [ ] English is the right primary language (vs Czech-first)

---

### Phase 1 — Build MVP (Weeks 3–8)
- Sprint 1 (W3): Setup + architecture + 2–3 parsers (or feed integrations if cinemas responded)
- Sprint 2 (W4–5): All parsers + TMDb matching pipeline + DeepL
- Sprint 3 (W6–7): Full frontend — home, filters, film card, language switcher
- Sprint 4 (W8): Cinema pages + maps + transport guide + LanguageTool + QA

### Phase 2 — Soft Launch (Week 9)
- Deploy to production
- Post Reddit Option B: "I built this — would you use it?"
- Post in expat Facebook groups
- Email all cinemas: "You're listed — here's your page" + invite feedback
- Personal outreach to 3–5 engaged cinema managers for relationship building (meeting or call)

### Phase 3 — Iterate (Week 10+, ongoing)
- Weekly: analytics review (device mix, language split, top pages, drop-off points)
- Weekly: check feedback widget (Tally.so) + Reddit/Facebook mentions
- Bi-weekly sprint: compile top findings → prioritise top 3 fixes → implement
- Month 2: compile findings → revise roadmap → begin partnership conversations
  - *Partnership model: free featured listing in exchange for venue promoting LOCAL SCENE to their audience (social post, newsletter, badge on their site)*

---

## 9. Open Questions

| Question | Owner | Due | Status |
|----------|-------|-----|--------|
| Do any target cinemas have XML/JSON/iCal feeds or APIs? | Ivan | Phase 0 | Open |
| Is ČSFD rating scrapeable without ToS issues? | Ivan | Phase 0 | Open |
| RT: link only or display score? Check enforcement posture for small sites | Ivan | Phase 0 | Open |
| Showtime-first vs cinema-first vs discovery-first? | User research | Phase 0 | Open |
| Depth (more Prague verticals) vs breadth (more Czech cities) post-MVP? | Ivan | Post-launch | Open |

---

## 10. Future Vision (Post-MVP)

```
LOCAL SCENE
├── 🎬 Cinema         ← MVP
├── 🎭 Theatre        ← v2 (same data pipeline pattern)
├── 🎵 Live Music     ← v2
└── 🖼  Galleries      ← v3
```

**User accounts (v2):**
- Google / Apple one-tap login (no email/password forms)
- Saved filter preferences
- Personalised weekly newsletter ("your saved search, every week")
- Watchlist

**Geographic expansion (v2+):**
- Brno → Plzeň → other Czech cities → full Czech Republic
- *Strategic priority to be decided post-MVP: more Prague verticals vs more cities*

**Monetisation path (year 1+):**
- Featured venue listings: 500–1500 CZK/month per venue
- Sponsored event promotion: 200–1000 CZK per event
- Newsletter sponsorship
- Realistic ceiling (Prague, niche): 20–50k CZK/month at maturity

---

## Appendix A — Target Cinemas (Prague MVP)

| Cinema | Neighbourhood | Notes |
|--------|---------------|-------|
| Kino Aero | Žižkov | Largest arthouse, strong programme |
| Bio Oko | Holešovice | Club cinema, popular with expats |
| Kino Světozor | Nové Město | Central, strong repertoire |
| Kino Pilotů | Smíchov | Community cinema |
| Kino 35 | Nové Město | Non-profit, unusual programme |
| Evald | Nové Město | Small, central |
| Kino Přítomnost | Žižkov | Newer, arthouse focus |
| Kasárna Karlín | Karlín | Cultural venue, occasional screenings |
| Ponrepo | Staré Město | Cinematheque, loyal audience, future programme focus |

## Appendix B — Competitive Analysis Template

*(To be completed in Phase 0)*

| Competitor | Type | Strengths | Weaknesses | Missing entirely | User complaints |
|------------|------|-----------|------------|-----------------|-----------------|
| kinobox.cz | Direct | | | | |
| csfd.cz | Direct | | | | |
| GoOut.net | Indirect | | | | |
| Individual cinema sites | Indirect | | | | |
| [Warsaw/Budapest analogue] | Analogue | | | | |

---

*This PRD is a living document. Updated as research findings and technical spikes inform scope decisions.*
