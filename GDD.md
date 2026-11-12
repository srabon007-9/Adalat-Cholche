# আদালত চলছে! — Adalat Cholche!
## Game Design Document (GDD)

**Subtitle:** Kotha Katakati · কথা কাটাকাটি
**Version:** 1.0 · **Date:** 2026-11-12 · **Status:** Ready for implementation (design only — no code in this document)
**Working title:** Adalat Cholche! (আদালত চলছে!) — "The Court Is in Session!"
**Platform:** Mobile-first web (plain HTML/CSS/vanilla JS) · GitHub Pages · Firebase Realtime Database (Spark/free tier)
**Players:** 2–6, online, separate devices, join by 4-letter room code
**Match length:** 6 rounds · ~35–45 minutes
**Language:** Every player-facing string ships in **Banglish** (romanized Bangla) **and Bangla** (বাংলা script), with a persistent in-game toggle.

---

## 1. Executive Summary

### 1.1 Elevator pitch
Your friends already argue about everything — who owes whom 500 taka, who ate the last biryani potato, who is *always* "5 minutes away." **Adalat Cholche!** turns those petty, beloved friend-group disputes into a formal comedy courtroom. Each round, a random **Mamla Card** (fake case) puts one friend on trial. The **Badi** (prosecutor) argues for 90 seconds, the **Bibadi** (accused) replies for 90 seconds, a **Sakkhi** (witness) gives chaotic testimony, objections fly, the court votes **doshi / nirdosh** (guilty / not guilty), and the judge delivers a ridiculous **Ray** (verdict) with a comedy sentence. Everyone earns **Jhogra Meter** points for arguing well, objecting sharply, and stealing the show. After 6 rounds, the loudest, funniest, most persuasive friend is crowned **Adalat-er Raja** (King of the Court).

### 1.2 Design pillars
1. **The argument is the game.** The app never generates humor — it *frames* human humor. Timers, prompts, roles and verdicts exist to make friends perform for each other. All comedy content is hand-written static data.
2. **Petty, relatable, family-friendly.** Every case is a real friend-group "crime": borrowed money, missed adda, biryani betrayal, chronic lateness. Nothing offensive, no politics, no religion, no romance, no real people.
3. **Everyone plays every round.** Nobody is eliminated, nobody waits idle. Non-arguing players are judges, witnesses, voters and hecklers with real mechanical power.
4. **Zero-cost, zero-backend-logic.** No AI, no LLM, no API keys for content, no paid services, no servers to maintain. Static JSON content + Firebase Realtime Database as a dumb shared state board + one host browser as the clock driver.
5. **Bangla first, Banglish always.** Full parity between বাংলা and Banglish at every screen, toggleable per device at any moment — even mid-argument.

### 1.3 Target audience & context
- Bengali friend groups (Bangladesh + West Bengal + diaspora), ages ~15+, comfortable with casual adda humor.
- Two play contexts, both supported identically:
  - **Same room:** friends on their phones, arguing out loud.
  - **Remote:** friends on their own voice call (WhatsApp/Messenger/Discord — out of band, never paid for by the game), app on each device.
- **The app carries no voice or video.** All speech happens out loud between humans. The app is a shared state machine: timers, prompts, buttons, scores. This is a hard architectural constraint that keeps the game free and private.

### 1.4 Session facts
| Fact | Value |
|---|---|
| Players | 2–6 (hard limits) |
| Room code | 4 uppercase letters (A–Z minus I/O → 24 letters, 331,776 codes) |
| Rounds per match | 6 (fixed) |
| Round length | ~6–7.5 min depending on player count |
| Match length | ~35–45 min typical (up to ~48 min at 6P with full timers) |
| Content pack | 30 Mamla cards · 20 Sakkhi cards · 15 Ray lines · 6 Objection lines · full dual-language UI string table |
| Repeat policy | No card repeats within a match (shuffle without replacement) |
| Accounts | None. Nickname + Firebase anonymous auth per device |
| Cost to run | ৳0 / $0 — GitHub Pages + Firebase Spark |

### 1.5 Hard constraints (from the brief, restated as law)
- ✅ Multiplayer online, separate devices, join-by-room-code (4 letters), min 2 / max 6.
- ✅ NO AI, NO API keys for content/services, NO paid services. All content = static data files.
  - *Clarification:* the Firebase web config contains a public `apiKey` field. This is a **public project identifier, not a secret or paid API key** — it is safe to commit and is protected by Realtime Database security rules (§12.6). No other keys exist anywhere in the project.
- ✅ Roles: Badi, Bibadi, Bicharok (auto-rotating judge duties in 2-player mode), Sakkhi (human friends at 3+, built-in Sakkhi-card deck at 2).
- ✅ Round loop: random Mamla → Badi 90s → Bibadi 90s → Sakkhi testimony + questioning → Objections → vote doshi/nirdosh → funny Ray + Jhogra Meter score. Roles rotate every round.
- ✅ Every string in Banglish AND Bangla with in-game toggle.
- ✅ Plain HTML/CSS/vanilla JS · Firebase RTDB Spark · GitHub Pages.

---

## 2. Match Flow (Bird's-Eye View)

```
CREATE / JOIN (room code) → LOBBY (2–6 players, ready check)
        │
        ▼
MATCH START — host shuffles Mamla deck (6 of 30), Sakkhi deck, Ray deck;
              fixed 6-round role schedule is computed (§8)
        │
        ▼
┌────────────────────────── ROUND (×6) ──────────────────────────┐
│ 1. MAMLA REVEAL      20s   card flips, prompts shown, ready-up │
│ 2. BADI ARGUMENT     90s   prosecutor speaks; objections live  │
│ 3. BIBADI REPLY      90s   accused speaks; objections live     │
│ 4. SAKKHI STAND      var   testimony + cross-questioning       │
│ 5. KOTHA KATAKATI    30s   dedicated objection window          │
│ 6. VOTE              25s   secret doshi/nirdosh ballot         │
│    (2P tie → SESH KOTHA 20s+20s → re-vote 15s → gavel)         │
│ 7. RAY               35s   verdict stamp, Ray line, Shasti     │
│ 8. JHOGRA METER      15s   rating gauge + point breakdown      │
│    → roles rotate → next round                               │
└────────────────────────────────────────────────────────────────┘
        │
        ▼
FINAL SCOREBOARD — podium, champion title, superlative awards,
                   rematch (same room) or new room
```

**Voice is out of band.** During every speaking phase, the app shows *who* should talk and *for how long*; friends supply the actual words, out loud. Prompts on the cards are improv starters, not scripts.

---

## 3. Roles

### 3.1 Full role table

| Role | Bangla | Who | Core duties | Mechanical powers | Earns JM |
|---|---|---|---|---|---|
| **Badi** | বাদী (plaintiff / prosecutor) | 1 player per round | Presents the Mamla, argues why Bibadi is doshi. Speaks up to 90s. | Fires objections (2 tokens/round); asks Sakkhi one question (15s); votes only in 2P mode | Verdict win +10 / loss +3; objection points; heat received; Sakkhi lean; chaos bonus (§7) |
| **Bibadi** | বিবাদী (accused / defendant) | 1 player per round | Defends themselves, replies up to 90s. | Same as Badi (2 objection tokens, one 15s question, 2P vote) | Same as Badi |
| **Bicharok** | বিচারক (judge) | 1 player per round at **4+**; **system + rotating gavel at 2–3** | Keeps order, rules on objections (Grohon/Khariz), breaks vote ties, reads the Ray aloud | Objection rulings (+1 JM each, cap +2/round); tie-breaking vote; reads Ray | +2 allowance/round + up to +2 ruling points |
| **Sakkhi** | সাক্ষী (witness) | **3+:** human friends (1 at 3–4P, 2 at 5P, 3 at 6P). **2P:** built-in Sakkhi-card deck, read automatically | Draws a Sakkhi card, testifies in character (30s), answers cross-questions (20s each) | Secret card leaning steers the court; at 3P the Sakkhi is the **sole voter** | +2 allowance/round; +3 Star Witness if verdict matches card lean (+1 if neutral card) |

### 3.2 Role assignment by player count

| Players | Badi | Bibadi | Bicharok | Sakkhi | Voters | Tie handling |
|---|---|---|---|---|---|---|
| **2** | Player A/B (alternates) | The other | **Auto-Bicharok** (system). A ceremonial **gavel** alternates each round: odd rounds → seat 1, even rounds → seat 2 | **Deck mode:** Sakkhi card auto-revealed, testimony auto-read | Both players | Sesh Kotha → re-vote → gavel holder decides |
| **3** | Rotates | Rotates | **Auto-Bicharok** (system); honorary gavel badge rotates for Ray-reading flavor | 1 human friend | The Sakkhi alone (1 vote → no ties possible) | Not applicable |
| **4** | Rotates | Rotates | 1 human | 1 human | Judge + Sakkhi (2) | Judge's vote breaks ties |
| **5** | Rotates | Rotates | 1 human | 2 humans | Judge + 2 Sakkhi (3) | Odd electorate → no ties |
| **6** | Rotates | Rotates | 1 human | 3 humans | Judge + 3 Sakkhi (4) | Judge's vote breaks ties |

**Rules of the bench:**
- Badi and Bibadi are *parties to the case* and never vote (except the 2-player duel, where both must).
- The **gavel** (2P) is independent of who argues: it alternates strictly by round number, so each player holds it exactly 3 times per match. It is the only "judge power" in 2P: breaking a deadlocked vote after Sesh Kotha.
- At 3P there are no vote ties (single voter) and objections auto-stand (§5.4), so the Auto-Bicharok needs no human input at all.
- At 4+, the judge is a real player with real power — and real responsibility to be funny while reading the Ray.
- **Sakkhi card leaning is secret** at 3+ (only the witness sees it; revealed at the Ray). In 2P deck mode the whole card, including leaning, is public.

### 3.3 Role fantasy & tone
- **Badi:** righteous outrage over nothing. "Ostad, ei manush amar 500 taka mereche — ar ami shudhu taka na, bishwash-o hariyechi!"
- **Bibadi:** shameless, creative denial. "Ostad, 500 taka? Kon 500 taka? Ami to 50 taka-o chini na."
- **Bicharok:** exhausted authority. Bangs the gavel, sighs, delivers verdicts like a disappointed parent.
- **Sakkhi:** chaotic neutral energy. A tea-stall uncle who "saw everything" but remembers nothing useful, a nani whose testimony is 80% gossip.

---

## 4. Round Loop & Turn Timing

### 4.1 Phase table (authoritative)

| # | Phase | Duration | Who acts | Early end | Notes |
|---|---|---|---|---|---|
| 1 | **Mamla Reveal** | 20s | Everyone reads; all tap **Prostut** (ready) | All ready → advance | Card flip animation; category emoji; Badi/Bibadi prompts highlighted for the respective players |
| 2 | **Badi Argument** | **90s hard cap** | Badi speaks out loud | Badi taps **Sesh** (done) | Opponent may fire objections (freezes clock 5s each, §5); audience taps **Jhogra!** heat |
| 3 | **Bibadi Reply** | **90s hard cap** | Bibadi speaks | Bibadi taps **Sesh** | Same objection/heat rules |
| 4 | **Sakkhi Stand** | see §4.2 | Witness(es) testify; Badi & Bibadi cross-question | Witness taps **Sesh** per segment | 2P: deck mode, auto-read |
| 5 | **Kotha Katakati** (Objections) | 30s | Badi & Bibadi fire remaining tokens; judge rules | All tokens spent + judge idle 5s → advance | Objections may also be fired *during* phases 2–3; tokens are shared across the round (2 per disputant) |
| 6 | **Vote** | 25s | Eligible voters (§3.2) tap **Doshi** or **Nirdosh** secretly | All votes in → advance | Parties see a "waiting for ray" screen with their round stats |
| 6b | **Sesh Kotha** (2P tie only) | 20s + 20s, then re-vote 15s | Badi final word, then Bibadi final word; both re-vote | — | Still tied → gavel holder's vote decides |
| 7 | **Ray** | 35s | Judge reads the Ray aloud (2–3P: displayed for anyone to read dramatically) | Judge taps **Ray Shesh** | Verdict stamp + gavel sound; Ray line + Shasti shown; Sakkhi lean revealed |
| 8 | **Jhogra Meter** | 15s + 10s auto-next | Everyone watches | Host taps **Next Round** | Rating gauge fills (§7.3); per-player point breakdown; next-round role banner |

**Timing philosophy:** 90s is long enough for a real argument, short enough to stay funny. Early **Sesh** is a legal strategy (bank confidence, deny opponent heat-farming time) with no penalty. All timers are server-time anchored (§12.4) — no client can cheat the clock.

### 4.2 Sakkhi Stand sub-timing

| Player count | Testimony | Cross-questioning | Phase total |
|---|---|---|---|
| **2 (deck mode)** | Card auto-read: 20s | Card's **Pressure Question** shown; Badi answers 20s, then Bibadi answers 20s (each interprets the witness's claim) | 60s |
| **3–4 (1 witness)** | 30s in-character testimony | Badi question 15s → witness answer 20s → Bibadi question 15s → witness answer 20s | 100s |
| **5 (2 witnesses)** | 30s each, seat order | Same single Q&A block (questions may target either witness; the addressed one answers) | 130s |
| **6 (3 witnesses)** | 30s each, seat order | Same single Q&A block | 160s |

- Witness order rotates within the phase by seat order; each witness draws a **different** Sakkhi card (no repeats within a match, §10.2).
- At 3+, testimony is **improv**: the card gives persona + starter line + secret lean; the witness must flavor it toward the Mamla. The starter is a launchpad, not a script.
- In 2P deck mode the card's full testimony text is displayed verbatim (it is written Mamla-agnostic), and the Pressure Question is answered by the players — the "witness" is the court's imagination.

### 4.3 Round length & match length estimates

Fixed phase budget: reveal 20s + arguments 180s + objections 30s + vote 25s + Ray 35s + meter 25s = **315s**, plus the Sakkhi phase (§4.2). "Max" assumes every timer runs full; "typical" assumes early **Sesh** taps and fast ready-ups (~40–60s saved per round in playtests).

| Players | Sakkhi phase | Round (max) | Round (typical) | 6-round match |
|---|---|---|---|---|
| 2 | 60s | 6:15 (+0:55 if Sesh Kotha) | ~5:30 | ~33–42 min |
| 3 | 100s | 6:55 | ~6:10 | ~37–42 min |
| 4 | 100s | 6:55 | ~6:10 | ~37–42 min |
| 5 | 130s | 7:25 | ~6:40 | ~40–45 min |
| 6 | 160s | 7:55 | ~7:10 | ~43–48 min |

### 4.4 What each screen shows during speech phases
- **Speaker:** giant countdown ring, their role badge, their prompt card (private starter line), **Sesh** button, objection-token indicator for the opponent.
- **Opponent:** same minus Sesh, plus **Objection** button (2 tokens) and the 6-line objection menu.
- **Judge/Sakkhi (audience):** speaker view + **Jhogra!** heat button (max 3 taps per speech per tapper) + judge's ruling panel when an objection lands.
- **Everyone:** live objection banner ("⚡ GOLPO GHURACCHEN!" + 5s freeze indicator), emote bar (§11.6).

---

## 5. Objections ("Kotha Katakati")

### 5.1 The six objection lines
Static menu, identical for both sides (full dual-language text in §10.4):

| # | Banglish | Bangla | Flavor |
|---|---|---|---|
| O1 | Golpo ghuracchen! | গল্প ঘোরাচ্ছেন! | Twisting the story |
| O2 | Mithya kotha! | মিথ্যা কথা! | Blatant lies |
| O3 | Amar kotha sesh hoyni! | আমার কথা শেষ হয়নি! | Interrupting me |
| O4 | Eita evidence na! | এটা এভিডেন্স না! | That's not evidence |
| O5 | Ostad, uni haschen! | ওস্তাদ, উনি হাসছেন! | They're laughing, Your Honor |
| O6 | Time waste kortechen! | টাইম ওয়েস্ট করছেন! | Wasting court time |

### 5.2 Tokens & firing
- Each disputant (Badi, Bibadi) has **2 objection tokens per round**.
- Tokens may be fired **during the opponent's argument/reply** (phases 2–3) or during the **dedicated 30s Kotha Katakati window** (phase 5).
- Firing during a speech: the speaker's clock **freezes for 5 seconds** ("ADALAT THAMLO!" banner), the objection line flashes for everyone, and the speaker must keep going after the freeze. Pure comedy interruption, mechanically bounded.
- Firing during phase 5: no clock effect (no speech running); it is a formal exchange of accusations before the vote.
- Tokens are the only spam guard needed: max 4 objections per round total.

### 5.3 Rulings (human judge, 4+ players)
When an objection lands, the judge has **5 seconds** to tap:

| Ruling | Bangla | Effect on objector | Effect on target | Judge |
|---|---|---|---|---|
| **Grohon** (sustained) | গ্রহণ | **+3 JM** | **−1 JM** | +1 JM |
| **Khariz** (overruled) | খারিজ | **−2 JM** | **+1 JM** | +1 JM |
| *(no tap — timeout)* | — | **+2 JM** (auto-stand) | 0 | 0 |

- Judge ruling points are capped at **+2 JM per round** (anti-farming).
- Judges never fire objections (only disputants do), so they never rule on their own.
- Rulings are final. No appeals. (The Ray lines joke about appeals anyway.)

### 5.4 Auto-judge modes (2P and 3P)
- No human judge exists → **every objection auto-stands at +2 JM, target 0**. The 5s speech freeze still applies in 2P/3P.
- This keeps 2–3 player games fast and argument-focused; the gavel's only job is the 2P vote tie-break.

### 5.5 Why this design
Objections are the "kotha katakati" of the subtitle: bounded, theatrical interruptions that reward timing (object when the opponent is actually twisting the story) and give the judge a meaningful, funny power. Deterministic effects mean **no AI or human moderation is needed** to decide whether an objection was "correct" — the judge's taste *is* the mechanic.

---

## 6. Voting & Verdict

### 6.1 Ballot
- Secret, simultaneous, 25s. Eligible voters per §3.2. Two big buttons: **DOSHI** (দোষী — guilty, red ✗) / **NIRDOSH** (নির্দোষ — not guilty, green ✓). Colors always paired with icons (color-blind safe).
- Votes are hidden until the reveal; then chips fly in one by one with a drumroll.
- **Verdict:** majority doshi → **Badi wins the round**; majority nirdosh → **Bibadi wins the round**.
- Timeout = abstain. Abstentions shrink the electorate.

### 6.2 Tie rules (complete)
| Situation | Resolution |
|---|---|
| 4P / 6P tied vote | **Judge's vote breaks the tie** (announced: "Bicharok-er ray-i sesh kotha!" / "বিচারকের রায়ই শেষ কথা!"). The judge always votes, so a tie always has a breaker. |
| 5P / 3P | Odd electorate → ties impossible (barring abstentions; if abstentions create a tie, the judge breaks it at 5P; at 3P a single voter cannot tie). |
| **2P tied vote** | **Sesh Kotha** (final word): Badi 20s, then Bibadi 20s → re-vote 15s. Still tied → **gavel holder's vote decides** (gavel alternates by round, §3.2). |
| **All eligible voters abstain** | **Hung court** ("Ray atke geche!" / "রায় আটকে গেছে!"): no round winner, both disputants get +3 JM, a neutral-tag Ray is read, round counts normally. |
| 3P abstention | Sakkhi abstains → hung court (same as above). |

### 6.3 The Ray
- After the verdict, the host draws the next unused Ray line whose tag matches the verdict (**Doshi**, **Nirdosh**, or **Any**; pools: 5+5+5, §10.3). "Any" lines are eligible for both verdicts, hung courts, and forfeits.
- The judge reads it aloud with maximum drama; 2–3P: anyone may read it (the app suggests the gavel holder).
- Every Ray carries a **Shasti** (comedy sentence) — a light, family-friendly forfeit performed on the honor system ("pay the next adda's tea bill", "sing 10 seconds", "serve everyone's plates at the next dawat"). The app displays it; it never enforces it. Sentences may target the loser, the winner, or both — always gentle.
- The Sakkhi card's secret lean is revealed here (for Star Witness scoring, §7.2).

---

## 7. Jhogra Meter — Scoring

**Jhogra Meter (JM)** is the match-long score currency. Every player earns JM every round, in every role. The per-round **Jhogra Rating** (0–100) is a separate drama gauge that flavors the round and pays a chaos bonus.

### 7.1 Disputant (Badi / Bibadi) round score

```
JM_disputant = Verdict + Objections + Heat + Lean + Chaos        (floored at 0)

Verdict:      win  +10   |   loss  +3 ("jhogra allowance")   |   hung court  +3   |   forfeit  0
Objections:   per fired token → sustained +3 (target −1)
                              → overruled −2 (target +1)
                              → auto-stand +2 (target 0)      [2P/3P, or judge timeout]
Heat:         +1 per "Jhogra!" tap received during your speech,
              capped at +5 per speech. Tappers: judge + Sakkhi only (3+ players),
              max 3 taps per tapper per speech. No heat in 2P.
Lean:         Sakkhi card leans your side → +2. Neutral card → +1 to both.
              (2P: lean public from the start. 3+: revealed at Ray.
               Multiple witnesses at 5–6P: each leaning card gives +2 to its side,
               each neutral card +1 to both; capped at +4 per disputant.)
Chaos:        round Jhogra Rating ≥ 61 → +2 to both disputants.
```

### 7.2 Support-role round score

```
JM_judge  = +2 allowance  + 1 per ruling made (cap +2)          → max +4
JM_sakkhi = +2 allowance  + 3 Star Witness (verdict matches your card's lean)
                          + 1 instead if your card is neutral   → max +5
```
- Star Witness rewards *performing* the lean convincingly — the court voting your way means you steered it.
- Allowances keep judges/witnesses in championship contention even in quiet rounds.
- Forfeited rounds (§13): no support-role allowances are paid (round skipped), except the judge still gets +1 if they had already made rulings.

### 7.3 Round Jhogra Rating (the drama gauge)

```
R = min(100, 15 + 10·O + 2·min(H, 20) + 15·T + 10·C)

O = total objections fired this round            (0–4)
H = total heat taps this round                   (counted up to 20)
T = 1 if the first vote was tied, else 0
C = 1 if the final verdict margin was exactly 1
    (includes 2P gavel-decided and judge-decided ties), else 0
```

| Rating | Tier (Banglish) | Tier (Bangla) | Vibe |
|---|---|---|---|
| 0–30 | Shanto Adda | শান্ত আড্ডা 😴 | A polite disagreement. Yawn. |
| 31–60 | Kotha Katakati | কথা কাটাকাটি 😤 | Now we're talking. |
| 61–85 | Ragaragiri | রাগারাগি 🔥 | Voices raised. Chaos bonus active. |
| 86–100 | Adalat Uchol! | আদালত উচ্ছল! 🌋 | The court has lost control. Beautiful. |

### 7.4 Worked example (4 players, round 3)
A = Badi, B = Bibadi, C = judge, D = Sakkhi (card leans Badi, secret).
- A fires 2 objections: C sustains one (+3 A, −1 B, +1 C), overrules one (−2 A, +1 B, +1 C).
- B fires 1 objection: C sustains (+3 B, −1 A, +1 C → C's ruling cap reached at +2).
- Heat: D taps 3× on A's speech (+3 A), C taps 2× on B's speech (+2 B). H = 5.
- Vote: C doshi, D nirdosh → 1–1 tie → judge C's vote breaks → **doshi, A wins**. T = 1, C = 1.
- Rating: R = 15 + 10·3 + 2·5 + 15 + 10 = **70 → Ragaragiri 🔥** → chaos +2 both.
- Lean: card leans Badi → +2 A (revealed now). Verdict doshi = Badi wins = the card's lean → **D's Star Witness succeeds**, +3 D.

| Player | Verdict | Objections | Heat | Lean | Chaos | Allowance/Ruling/Star | **Round JM** |
|---|---|---|---|---|---|---|---|
| A (Badi, win) | +10 | +3 −2 −1 = 0 | +3 | +2 | +2 | — | **17** |
| B (Bibadi, loss) | +3 | −1 +1 +3 = +3 | +2 | 0 | +2 | — | **10** |
| C (judge) | — | — | — | — | — | +2 +2 (cap) | **4** |
| D (Sakkhi) | — | — | — | — | — | +2 +3 | **5** |

### 7.5 Score integrity rules
- Round JM is floored at 0 (nobody loses their shirt; match totals can still plateau).
- All point events are written to the room log by the host driver with type + amount, so the breakdown screen is fully auditable and reconnecting clients rebuild identical totals.
- No discretionary points exist anywhere — every number above is deterministic. (The judge's *taste* chooses rulings; the *values* are fixed.)

---

## 8. Rotation & Fairness (6-Round Schedules)

Seats are fixed at lobby time (seat 1 = host, then join order). Schedules are computed at match start from seat order — deterministic, shown in the lobby ("Ebar ke ki" preview), and immune to host migration.

### 8.1 Two players (A, B)
| Round | Badi | Bibadi | Gavel (tie-break) | Sakkhi |
|---|---|---|---|---|
| 1 | A | B | A | Deck |
| 2 | B | A | B | Deck |
| 3 | A | B | A | Deck |
| 4 | B | A | B | Deck |
| 5 | A | B | A | Deck |
| 6 | B | A | B | Deck |

Each: 3× Badi, 3× Bibadi, 3× gavel. Perfectly fair.

### 8.2 Three players (A, B, C) — cycle shifts by one
| Round | Badi | Bibadi | Sakkhi (sole voter) |
|---|---|---|---|
| 1 | A | B | C |
| 2 | B | C | A |
| 3 | C | A | B |
| 4 | A | B | C |
| 5 | B | C | A |
| 6 | C | A | B |

Each: 2× Badi, 2× Bibadi, 2× Sakkhi. Perfectly fair.

### 8.3 Four players (A, B, C, D)
| Round | Badi | Bibadi | Bicharok | Sakkhi |
|---|---|---|---|---|
| 1 | A | B | C | D |
| 2 | C | D | A | B |
| 3 | B | A | D | C |
| 4 | D | C | B | A |
| 5 | A | C | D | B |
| 6 | B | D | A | C |

Each: exactly **3 disputes** (Badi/Bibadi split 2/1), judge 1–2×, Sakkhi 1–2×, judge+Sakkhi = 3 each. Dispute count perfectly fair; support-role mix varies by at most one round.

### 8.4 Five players (A, B, C, D, E)
| Round | Badi | Bibadi | Bicharok | Sakkhi ×2 |
|---|---|---|---|---|
| 1 | A | B | C | D, E |
| 2 | C | D | A | B, E |
| 3 | E | A | B | C, D |
| 4 | B | C | D | A, E |
| 5 | D | E | A | B, C |
| 6 | C | A | E | B, D |

Disputes: A 3, C 3, B 2, D 2, E 2. **Inherent variance:** 12 dispute slots ÷ 5 players cannot be equal; the two extra disputes go to seats 1 and 3 by fixed rule, and the lobby schedule preview makes this transparent. Judge: A 2, others 1. Sakkhi counts fill the remainder (each player's non-dispute rounds). This is the one player count with structural imbalance — documented, deterministic, and visible before the match starts.

### 8.5 Six players (A, B, C, D, E, F)
| Round | Badi | Bibadi | Bicharok | Sakkhi ×3 |
|---|---|---|---|---|
| 1 | A | B | C | D, E, F |
| 2 | C | D | A | B, E, F |
| 3 | F | E | B | A, C, D |
| 4 | B | A | D | C, E, F |
| 5 | E | C | F | A, B, D |
| 6 | D | F | E | A, B, C |

Each: exactly **2 disputes (1× Badi, 1× Bibadi), 1× judge, 3× Sakkhi**. Perfectly fair.

### 8.6 Rotation under player-count changes
If players leave mid-match (≥2 remain), the schedule for **future rounds** is recomputed from the surviving seats using the same tables (a 4P match that drops to 3P continues with the 3P cycle from the next round). Past rounds and scores are never rewritten. If a departing player held a current-round role, §13 edge rules apply immediately.

---

## 9. Win Conditions & Awards

### 9.1 The championship
- After round 6, **highest total JM wins** → crowned **Adalat-er Raja / Adalat-er Rani** (আদালতের রাজা / রানী — gender-neutral display offers both; the player picks their crown flavor in the lobby, default "Raja"). Also earns the static title line **"Moha Jhograbaj"** (মহা ঝগড়াবাজ — Grand Arguer).
- **Match tie-break ladder** (applied in order):
  1. Most round wins (verdicts won as a disputant).
  2. Most objections sustained (Grohon).
  3. Most heat taps received.
  4. Fewest objections overruled.
  5. Still tied → **shared crown**: "Jugalbondi Champion" (যুগলবন্দি চ্যাম্পিয়ন) — the court declares a historic friendship. No sudden-death round; the match always ends at 6.

### 9.2 Superlative awards (deterministic, from match stats; shared if tied)
| Award (Banglish) | Award (Bangla) | Criterion |
|---|---|---|
| Adalat-er Bagh 🐯 | আদালতের বাঘ | Most round wins |
| Objection-er Raja ⚡ | অবজেকশনের রাজা | Most objections sustained |
| Jhogra Magnet 🧲 | ঝগড়া ম্যাগনেট | Most heat taps received |
| Sakkhi-er Sera 🎭 | সাক্ষীর সেরা | Most Star Witness bonuses |
| Khariz King 😅 | খারিজ কিং | Most objections overruled (consolation comedy) |
| Shanti Dut 🕊️ | শান্তি দূত | Lowest total JM (consolation: "apni jhogra koren ni, shanti diyechen") |

### 9.3 End-of-match flow
Final scoreboard → podium animation (top 3) → champion crowned with gavel fanfare → superlatives dealt one by one → full stats table → buttons: **Rematch** (same room, same seats, scores reset, decks reshuffled), **Notun Adalat** (new room code), **Biday** (exit to splash). Rematch keeps the room alive; the 2-hour idle TTL (§12.3) resets on any activity.

### 9.4 Round-level "win"
Winning a round = verdict in your favor (+10 JM + bragging rights + the loser performs the Shasti on honor system). Round wins feed tie-breaker #1 but do not directly win the match — **JM is the only championship currency**, so a graceful loser who steals every scene can still take the crown. This is intentional: the game rewards *performance*, not just verdicts.

---

## 10. Complete Content Pack

All content lives in static JSON files (§10.6 schema). Every text field is an object `{ "bl": "...", "bn": "..." }`. IDs are stable forever (save-file/room compatibility). Tone rules for all content: family-friendly, friend-group petty humor, no politics/religion/romance/alcohol/real people, money amounts ≤ ৳500, exaggeration over insult.

### 10.1 Mamla Cards (30)

Each card: **title**, **case** (the accusation, shown to all), **Badi prompt** (prosecutor's improv starter, highlighted for Badi), **Bibadi prompt** (defense starter, highlighted for Bibadi), plus a category emoji for the card face. Prompts are launchpads — players improvise the rest.

---

**M01 — Biryani-r Alu | বিরিয়ানির আলু** 🍛
- **Mamla (BL):** Dawat-er biryani-te Badi ekta alu tule rekhechilo shesh khabar jonno — "shukher shesh kamor." Bibadi sei alu kheye feleche, ar bolche "extra alu, keu khabe na."
- **মামলা (BN):** দাওয়াতের বিরিয়ানিতে বাদী একটা আলু তুলে রেখেছিল শেষ কামড়ের জন্য — "সুখের শেষ কামড়।" বিবাদী সেই আলু খেয়ে ফেলেছে, আর বলছে "এক্সট্রা আলু, কেউ খাবে না।"
- **Badi (BL):** "Ostad, alu chilo na — oita chilo amar biryani-r mukut. Ami plate-er dike takiye chilam, ar uni..."
- **বাদী (BN):** "ওস্তাদ, আলু ছিল না — ওটা ছিল আমার বিরিয়ানির মুকুট। আমি প্লেটের দিকে তাকিয়ে ছিলাম, আর উনি..."
- **Bibadi (BL):** "Ostad, alu niye eto boro mamla? Manush biryani khay, alu na. Ami to shudhu... shudhu..."
- **বিবাদী (BN):** "ওস্তাদ, আলু নিয়ে এত বড় মামলা? মানুষ বিরিয়ানি খায়, আলু না। আমি তো শুধু... শুধু..."

**M02 — 500 Taka Dhara | ৫০০ টাকা ধার** 💸
- **Mamla (BL):** Tin mash age Bibadi 500 taka dhara niyechilo — "kalkei diye debo." Ekhon Bibadi bolche "kon 500? Ami to 50 taka-o chini na." Badi-r kache screenshot ache.
- **মামলা (BN):** তিন মাস আগে বিবাদী ৫০০ টাকা ধার নিয়েছিল — "কালকেই দিয়ে দেব।" এখন বিবাদী বলছে "কোন ৫০০? আমি তো ৫০ টাকাও চিনি না।" বাদীর কাছে স্ক্রিনশট আছে।
- **Badi (BL):** "Ostad, 'kalke' bolte uni tin mash bujhiyechen. Ei screenshot dekhun — its written, 'kalke debo'!"
- **বাদী (BN):** "ওস্তাদ, 'কালকে' বলতে উনি তিন মাস বুঝিয়েছেন। এই স্ক্রিনশট দেখুন — লেখা আছে, 'কালকে দেব'!"
- **Bibadi (BL):** "Ostad, taka ami niyechi — kintu bondhu-r kache taka ki asolei taka? Oita to bishwash-er len-den."
- **বিবাদী (BN):** "ওস্তাদ, টাকা আমি নিয়েছি — কিন্তু বন্ধুর কাছে টাকা কি আসলেই টাকা? ওটা তো বিশ্বাসের লেনদেন।"

**M03 — Adda Miss | আড্ডা মিস** 🫖
- **Mamla (BL):** Shukrobar-er adda-e Bibadi asheni — bolchen "traffic chilo." Kintu thik oi shomoye Bibadi-ke onno ekta adda-e selfie tul-te dekha geche. Badi-r dabi: eta adda-droho.
- **মামলা (BN):** শুক্রবারের আড্ডায় বিবাদী আসেনি — বলছে "ট্রাফিক ছিল।" কিন্তু ঠিক ওই সময়ে বিবাদীকে অন্য একটা আড্ডায় সেলফি তুলতে দেখা গেছে। বাদীর দাবি: এটা আড্ডা-দ্রোহ।
- **Badi (BL):** "Ostad, traffic chilo na — chilo onno adda. Selfie-r caption chilo 'chill'. Amar chokhe jol chole elo."
- **বাদী (BN):** "ওস্তাদ, ট্রাফিক ছিল না — ছিল অন্য আড্ডা। সেলফির ক্যাপশন ছিল 'চিল'। আমার চোখে জল চলে এল।"
- **Bibadi (BL):** "Ostad, oi selfie purono! Date... date ta thik ache, kintu mood ta purono chilo."
- **বিবাদী (BN):** "ওস্তাদ, ওই সেলফি পুরোনো! তারিখ... তারিখটা ঠিক আছে, কিন্তু মুডটা পুরোনো ছিল।"

**M04 — "5 Minute" | "৫ মিনিট"** ⏰
- **Mamla (BL):** Bibadi protibar bole "5 min, ber hochi" — tarpor 45 min pore ashe. Aaj to 1 ghonta! Badi dabi kore, "5 minute" ekhon Bibadi-r kache ekta time zone.
- **মামলা (BN):** বিবাদী প্রতিবার বলে "৫ মিন, বের হচ্ছি" — তারপর ৪৫ মিনিট পরে আসে। আজ তো ১ ঘণ্টা! বাদী দাবি করে, "৫ মিনিট" এখন বিবাদীর কাছে একটা টাইম জোন।
- **Badi (BL):** "Ostad, ami ghori niye bose chilam. Uni 'ber hochi' bolchen — ami bhablam, manush ber hochilo, kintu asole..."
- **বাদী (BN):** "ওস্তাদ, আমি ঘড়ি নিয়ে বসে ছিলাম। উনি 'বের হচ্ছি' বলছেন — আমি ভাবলাম, মানুষ বের হচ্ছিল, কিন্তু আসলে..."
- **Bibadi (BL):** "Ostad, 5 min bolle ki 5 min hote hoy? Eta ekta expression — shomoyer jonno ekta ashar bani."
- **বিবাদী (BN):** "ওস্তাদ, ৫ মিন বললে কি ৫ মিনিট হতে হয়? এটা একটা এক্সপ্রেশন — সময়ের জন্য একটা আশার বাণী।"

**M05 — Cha-er Bill | চায়ের বিল** ☕
- **Mamla (BL):** Cha-er dokane 14 diner bill baki — Bibadi protidin bole "kalke dibo." Dokaner Mamu ekhon Badi-ke dhorche, ar Badi Bibadi-ke dhoreche. Mamu bolchen: "dui jon-er ekjon-ke to ditei hobe."
- **মামলা (BN):** চায়ের দোকানে ১৪ দিনের বিল বাকি — বিবাদী প্রতিদিন বলে "কালকে দেব।" দোকানের মামু এখন বাদীকে ধরছে, আর বাদী বিবাদীকে ধরেছে। মামু বলছেন: "দুই জনের একজনকে তো দিতেই হবে।"
- **Badi (BL):** "Ostad, ami cha khaichi 2 cup, uni 12 cup. Hisab ta ki kore amar upor pore, bolun to?"
- **বাদী (BN):** "ওস্তাদ, আমি চা খেয়েছি ২ কাপ, উনি ১২ কাপ। হিসাবটা কী করে আমার ওপর পড়ে, বলুন তো?"
- **Bibadi (BL):** "Ostad, cha to dui jon-i khaichi. Ar Mamu-r khata-e amar naam... thik ache, ache. Kintu baki to bondhu-ra mile..."
- **বিবাদী (BN):** "ওস্তাদ, চা তো দুই জনই খেয়েছি। আর মামুর খাতায় আমার নাম... ঠিক আছে, আছে। কিন্তু বাকি তো বন্ধুরা মিলে..."

**M06 — Netflix Password | নেটফ্লিক্স পাসওয়ার্ড** 📺
- **Mamla (BL):** Dui bondhu mile Netflix share korto. Bibadi hotath password change kore finale nijei dekhe feleche — ar Badi-ke bolche "subscription sesh." Badi ekhon season-er shesh episode chara adalat-e.
- **মামলা (BN):** দুই বন্ধু মিলে নেটফ্লিক্স শেয়ার করত। বিবাদী হঠাৎ পাসওয়ার্ড চেঞ্জ করে ফিনালে নিজেই দেখে ফেলেছে — আর বাদীকে বলছে "সাবস্ক্রিপশন শেষ।" বাদী এখন সিজনের শেষ এপিসোড ছাড়া আদালতে।
- **Badi (BL):** "Ostad, 7 season eksathe dekhchi, ar finale-er age uni password palte dilen. Eta ki bondhu-tto, na eta cyber crime?"
- **বাদী (BN):** "ওস্তাদ, ৭ সিজন একসাথে দেখছি, আর ফিনালের আগে উনি পাসওয়ার্ড পাল্টে দিলেন। এটা কি বন্ধুত্ব, না এটা সাইবার ক্রাইম?"
- **Bibadi (BL):** "Ostad, password change hoye gechilo... nijer theke. System-er dosh. Ar finale to... spoiler chara bola jay na, tai chup chilam."
- **বিবাদী (BN):** "ওস্তাদ, পাসওয়ার্ড চেঞ্জ হয়ে গিয়েছিল... নিজে থেকে। সিস্টেমের দোষ। আর ফিনালে তো... স্পয়লার ছাড়া বলা যায় না, তাই চুপ ছিলাম।"

**M07 — Charger | চার্জার** 🔌
- **Mamla (BL):** Bibadi dui shoptaho age Badi-r charger niyechilo. Ekhon bolche "eita to amar charger, tomar ta kobe dilam?" Badi-r kache charger-er gaye nijer naam lekha ache — permanent marker-e.
- **মামলা (BN):** বিবাদী দুই সপ্তাহ আগে বাদীর চার্জার নিয়েছিল। এখন বলছে "এটা তো আমার চার্জার, তোমারটা কবে দিলাম?" বাদীর কাছে চার্জারের গায়ে নিজের নাম লেখা আছে — পার্মানেন্ট মার্কারে।
- **Badi (BL):** "Ostad, charger-er gaye amar naam lekha. Permanent marker. Uni bolchen eta 'design'."
- **বাদী (BN):** "ওস্তাদ, চার্জারের গায়ে আমার নাম লেখা। পার্মানেন্ট মার্কার। উনি বলছেন এটা 'ডিজাইন'।"
- **Bibadi (BL):** "Ostad, naam lekha thakle ki seta tar hoy? Tahole to school-er bench-gula sobar..."
- **বিবাদী (BN):** "ওস্তাদ, নাম লেখা থাকলে কি সেটা তার হয়? তাহলে তো স্কুলের বেঞ্চগুলো সবার..."

**M08 — Group Chat Leave | গ্রুপ চ্যাট লিভ** 📱
- **Mamla (BL):** Group chat-e ekta joke hoyechilo Bibadi-ke niye. Bibadi "Left the group" kore chole gechilo — drama kore. Ekhon abar add hote chay, kintu shart: shobai "sorry" bolbe. Badi bolche: age adalat-e sorry bolte hobe.
- **মামলা (BN):** গ্রুপ চ্যাটে একটা জোক হয়েছিল বিবাদীকে নিয়ে। বিবাদী "Left the group" করে চলে গিয়েছিল — ড্রামা করে। এখন আবার অ্যাড হতে চায়, কিন্তু শর্ত: সবাই "সরি" বলবে। বাদী বলছে: আগে আদালতে সরি বলতে হবে।
- **Badi (BL):** "Ostad, ekta joke-e uni group chere dilen. Amra 3 din 'ke add korbe' kore bose chilam. Eta emotional blackmail."
- **বাদী (BN):** "ওস্তাদ, একটা জোকে উনি গ্রুপ ছেড়ে দিলেন। আমরা ৩ দিন 'কে অ্যাড করবে' করে বসে ছিলাম। এটা ইমোশনাল ব্ল্যাকমেইল।"
- **Bibadi (BL):** "Ostad, joke ta chilo... thik ache, funny chilo. Kintu amar dignity? Dignity ki joke-er cheye choto?"
- **বিবাদী (BN):** "ওস্তাদ, জোকটা ছিল... ঠিক আছে, ফানি ছিল। কিন্তু আমার ডিগনিটি? ডিগনিটি কি জোকের চেয়ে ছোট?"

**M09 — Last Leg Piece | শেষ লেগ পিস** 🍗
- **Mamla (BL):** Dawat-e biryani-r shesh leg piece-ta Badi chokh diye mark korechilo. Bibadi "ami to shudhu nite chilam... " bole piece-ta niye feleche. Badi-r dabi: eta biryani-niyom-er lankhon.
- **মামলা (BN):** দাওয়াতে বিরিয়ানির শেষ লেগ পিসটা বাদী চোখ দিয়ে মার্ক করেছিল। বিবাদী "আমি তো শুধু নিতে ছিলাম..." বলে পিসটা নিয়ে ফেলেছে। বাদীর দাবি: এটা বিরিয়ানি-নিয়মের লঙ্ঘন।
- **Badi (BL):** "Ostad, ami piece-ta 10 min dhore chokh diye dhore rekhechilam. Eta dawat-er niyom — je age dekhe, seta tar!"
- **বাদী (BN):** "ওস্তাদ, আমি পিসটা ১০ মিনিট ধরে চোখ দিয়ে ধরে রেখেছিলাম। এটা দাওয়াতের নিয়ম — যে আগে দেখে, সেটা তার!"
- **Bibadi (BL):** "Ostad, chokh diye reserve kora jay na. Hat diye nite hoy. Ami hat diyechi — legal process."
- **বিবাদী (BN):** "ওস্তাদ, চোখ দিয়ে রিজার্ভ করা যায় না। হাত দিয়ে নিতে হয়। আমি হাত দিয়েছি — লিগ্যাল প্রসেস।"

**M10 — Picnic Fund | পিকনিক ফান্ড** 🧺
- **Mamla (BL):** Choy mash age Bibadi shobar kach theke 200 taka kore picnic-er fund tulechilo. Picnic aj porjonto hoyni. Bibadi bolche "plan cholche" — kintu fund-er taka diye notun headphone kineche.
- **মামলা (BN):** ছয় মাস আগে বিবাদী সবার কাছ থেকে ২০০ টাকা করে পিকনিকের ফান্ড তুলেছিল। পিকনিক আজ পর্যন্ত হয়নি। বিবাদী বলছে "প্ল্যান চলছে" — কিন্তু ফান্ডের টাকা দিয়ে নতুন হেডফোন কিনেছে।
- **Badi (BL):** "Ostad, picnic-er naam-e taka nilo, headphone kineche. Ekhon uni headphone-e gaan shune 'plan cholche' bolchen."
- **বাদী (BN):** "ওস্তাদ, পিকনিকের নামে টাকা নিল, হেডফোন কিনেছে। এখন উনি হেডফোনে গান শুনে 'প্ল্যান চলছে' বলছেন।"
- **Bibadi (BL):** "Ostad, headphone ta picnic-er jonno! Gaan chara picnic hoy? Eta investment, churi na."
- **বিবাদী (BN):** "ওস্তাদ, হেডফোনটা পিকনিকের জন্য! গান ছাড়া পিকনিক হয়? এটা ইনভেস্টমেন্ট, চুরি না।"

**M11 — Kharap Photo | খারাপ ফটো** 📸
- **Mamla (BL):** Bibadi Badi-r ekta photo tulechilo — thik hachi-er muhurte. Photo-ta group-e post koreche, ar ekhon bolche "delete korbo na, eta art." Badi-r dabi: eta manushikotar oporadh.
- **মামলা (BN):** বিবাদী বাদীর একটা ফটো তুলেছিল — ঠিক হাঁচির মুহূর্তে। ফটোটা গ্রুপে পোস্ট করেছে, আর এখন বলছে "ডিলিট করব না, এটা আর্ট।" বাদীর দাবি: এটা মানবিকতার অপরাধ।
- **Badi (BL):** "Ostad, amar chokh dui-ta bondho, mukh pura khola — ar caption 'model'. Eta delete na hole ami adda-e ashbo na."
- **বাদী (BN):** "ওস্তাদ, আমার চোখ দুইটা বন্ধ, মুখ পুরা খোলা — আর ক্যাপশন 'মডেল'। এটা ডিলিট না হলে আমি আড্ডায় আসব না।"
- **Bibadi (BL):** "Ostad, photo ta candid. Candid mane sotti. Sotti ki oporadh? Art gallery-te ei photo-r dam hobe."
- **বিবাদী (BN):** "ওস্তাদ, ফটোটা ক্যান্ডিড। ক্যান্ডিড মানে সত্যি। সত্যি কি অপরাধ? আর্ট গ্যালারিতে এই ফটোর দাম হবে।"

**M12 — Cricket Umpiring | ক্রিকেট আম্পায়ারিং** 🏏
- **Mamla (BL):** Para-r cricket-e Bibadi umpire chilo. Badi-r clear six-ke uni "out" diyechen — karon "ball ta amar chilo, ar amar ball out hole out." Badi dabi kore: eta match-fixing, para-level.
- **মামলা (BN):** পাড়ার ক্রিকেটে বিবাদী আম্পায়ার ছিল। বাদীর ক্লিয়ার ছক্কাকে উনি "আউট" দিয়েছেন — কারণ "বলটা আমার ছিল, আর আমার বল আউট হলে আউট।" বাদী দাবি করে: এটা ম্যাচ-ফিক্সিং, পাড়া-লেভেল।
- **Badi (BL):** "Ostad, ball roof-er upor diye geche. SIX. Uni bolchen 'amar ball, amar iccha'. Umpire na, raja?"
- **বাদী (BN):** "ওস্তাদ, বল রুফের ওপর দিয়ে গেছে। সিক্স। উনি বলছেন 'আমার বল, আমার ইচ্ছা'। আম্পায়ার না, রাজা?"
- **Bibadi (BL):** "Ostad, ball amar chilo, risk amar chilo. Nijer ball-e karo six hole kemon lage, bujhlen?"
- **বিবাদী (BN):** "ওস্তাদ, বল আমার ছিল, রিস্ক আমার ছিল। নিজের বলে কারো সিক্স হলে কেমন লাগে, বুঝলেন?"

**M13 — WiFi Password | ওয়াইফাই পাসওয়ার্ড** 📶
- **Mamla (BL):** Badi-r exam week-e Bibadi WiFi password change koreche — bolche "router reset hoye gechilo." Kintu Bibadi nijei oi WiFi-te 3 din series dekheche. Badi exam-e fail koreche (almost).
- **মামলা (BN):** বাদীর এক্সাম উইকে বিবাদী ওয়াইফাই পাসওয়ার্ড চেঞ্জ করেছে — বলছে "রাউটার রিসেট হয়ে গিয়েছিল।" কিন্তু বিবাদী নিজেই ওই ওয়াইফাইতে ৩ দিন সিরিজ দেখেছে। বাদী এক্সামে ফেল করেছে (প্রায়)।
- **Badi (BL):** "Ostad, amar exam chilo, ar uni password niye boshe chilo. Result ashche 2.9 — eta ki amar dosh, na WiFi-er?"
- **বাদী (BN):** "ওস্তাদ, আমার এক্সাম ছিল, আর উনি পাসওয়ার্ড নিয়ে বসে ছিল। রেজাল্ট এসেছে ২.৯ — এটা কি আমার দোষ, না ওয়াইফাইয়ের?"
- **Bibadi (BL):** "Ostad, router reset hoyechilo — sotti. Ar notun password... mone chilo na. Nijer-o mone chilo na, tai likhe rekhechilam."
- **বিবাদী (BN):** "ওস্তাদ, রাউটার রিসেট হয়েছিল — সত্যি। আর নতুন পাসওয়ার্ড... মনে ছিল না। নিজেরও মনে ছিল না, তাই লিখে রেখেছিলাম।"

**M14 — Tiffin | টিফিন** 🍱
- **Mamla (BL):** Office-e Badi-r tiffin-er porota-gula Bibadi kheye feleche — "bhabchilam tumi share korba." Badi ekhon dupure bhooka, ar Bibadi bolche "tumi to boloni je khabe na."
- **মামলা (BN):** অফিসে বাদীর টিফিনের পরোটাগুলো বিবাদী খেয়ে ফেলেছে — "ভাবছিলাম তুমি শেয়ার করবা।" বাদী এখন দুপুরে ভুখা, আর বিবাদী বলছে "তুমি তো বলোনি যে খাবে না।"
- **Badi (BL):** "Ostad, tiffin box-e amar naam lekha chilo. Porota 3-ta chilo. Ekhon box khali, ar uni bolchen 'share'."
- **বাদী (BN):** "ওস্তাদ, টিফিন বক্সে আমার নাম লেখা ছিল। পরোটা ৩টা ছিল। এখন বক্স খালি, আর উনি বলছেন 'শেয়ার'।"
- **Bibadi (BL):** "Ostad, bondhu-r tiffin ki bondhu-r na? Ami to bhabchilam, amra ek poribar. Ek poribar-e ki hisab hoy?"
- **বিবাদী (BN):** "ওস্তাদ, বন্ধুর টিফিন কি বন্ধুর না? আমি তো ভাবছিলাম, আমরা এক পরিবার। এক পরিবারে কি হিসাব হয়?"

**M15 — Spoiler | স্পয়লার** 🎬
- **Mamla (BL):** Group chat-e Bibadi series-er finale-r spoiler diyeche — caption chilo "keu kichu bolis na." Badi 6 season dhore dekhchilo, ekhon shesh episode dekhte parche na — karon shob jana hoye geche.
- **মামলা (BN):** গ্রুপ চ্যাটে বিবাদী সিরিজের ফিনালের স্পয়লার দিয়েছে — ক্যাপশন ছিল "কেউ কিছু বলিস না।" বাদী ৬ সিজন ধরে দেখছিল, এখন শেষ এপিসোড দেখতে পারছে না — কারণ সব জানা হয়ে গেছে।
- **Badi (BL):** "Ostad, 6 season-er porishrom ek message-e sesh. 'Keu kichu bolis na' — eta spoiler-er age bosale ki spoiler bondho hoy?"
- **বাদী (BN):** "ওস্তাদ, ৬ সিজনের পরিশ্রম এক মেসেজে শেষ। 'কেউ কিছু বলিস না' — এটা স্পয়লারের আগে বসালে কি স্পয়লার বন্ধ হয়?"
- **Bibadi (BL):** "Ostad, ami to warning diyechilam! 'Keu kichu bolis na' — je poreche, seta tar porar dosh."
- **বিবাদী (BN):** "ওস্তাদ, আমি তো ওয়ার্নিং দিয়েছিলাম! 'কেউ কিছু বলিস না' — যে পড়েছে, সেটা তার পড়ার দোষ।"

---

**M16 — Shortcut | শর্টকাট** 🗺️
- **Mamla (BL):** Bibadi bolchilo "ami rasta chini, shortcut ache." Oi shortcut-e shobai 1.5 ghonta hariye gechilo — ekta dead-end-e. Bibadi ekhon bolchen "Google Maps-er dosh."
- **মামলা (BN):** বিবাদী বলছিল "আমি রাস্তা চিনি, শর্টকাট আছে।" ওই শর্টকাটে সবাই ১.৫ ঘণ্টা হারিয়ে গিয়েছিল — একটা ডেড-এন্ডে। বিবাদী এখন বলছে "গুগল ম্যাপসের দোষ।"
- **Badi (BL):** "Ostad, uni bolchen 'ei to, ekdom kache.' Tarpor amra ekta goru-r sathe 20 min chokh-a-chokhi kore chilam."
- **বাদী (BN):** "ওস্তাদ, উনি বলছেন 'এই তো, একদম কাছে।' তারপর আমরা একটা গরুর সাথে ২০ মিনিট চোখাচোখি করে ছিলাম।"
- **Bibadi (BL):** "Ostad, shortcut chilo — kintu rasta ta notun kore... onno dike chole gechilo. Rasta-r dosh, amar na."
- **বিবাদী (BN):** "ওস্তাদ, শর্টকাট ছিল — কিন্তু রাস্তাটা নতুন করে... অন্য দিকে চলে গিয়েছিল। রাস্তার দোষ, আমার না।"

**M17 — Umbrella | ছাতা** ☂️
- **Mamla (BL):** Brishti-r dine Bibadi Badi-r chata niyechilo. Feroat diyeche dui-ta dandi bhanga obosthay — "hawa-e bhenge geche." Badi-r dabi: chata hawa-e bhange na, bondhu-r hate bhange.
- **মামলা (BN):** বৃষ্টির দিনে বিবাদী বাদীর ছাতা নিয়েছিল। ফেরত দিয়েছে দুইটা দাঁড়ি ভাঙা অবস্থায় — "হাওয়ায় ভেঙে গেছে।" বাদীর দাবি: ছাতা হাওয়ায় ভাঙে না, বন্ধুর হাতে ভাঙে।
- **Badi (BL):** "Ostad, chata ta notun chilo. Ekhon seta ekta art piece — dui dandi, tin kon. Uni bolchen 'hawa'."
- **বাদী (BN):** "ওস্তাদ, ছাতাটা নতুন ছিল। এখন সেটা একটা আর্ট পিস — দুই দাঁড়ি, তিন কোণ। উনি বলছেন 'হাওয়া'।"
- **Bibadi (BL):** "Ostad, hawa sotti chilo! Ami to chata dhore chilam, hawa amake dhore... fele chilo. Ami victim."
- **বিবাদী (BN):** "ওস্তাদ, হাওয়া সত্যি ছিল! আমি তো ছাতা ধরে ছিলাম, হাওয়া আমাকে ধরে... ফেলেছিল। আমি ভিকটিম।"

**M18 — Biye-r Dawat | বিয়ের দাওয়াত** 💌
- **Mamla (BL):** Bibadi-r biye-te group-er shobai dawat peyeche — shudhu Badi payni. Bibadi bolche "card chilo na, list-e naam chilo na." Kintu Badi-r naam list-e chilo — cross kora.
- **মামলা (BN):** বিবাদীর বিয়েতে গ্রুপের সবাই দাওয়াত পেয়েছে — শুধু বাদী পায়নি। বিবাদী বলছে "কার্ড ছিল না, লিস্টে নাম ছিল না।" কিন্তু বাদীর নাম লিস্টে ছিল — ক্রস করা।
- **Badi (BL):** "Ostad, amar naam cross kora! Keno? Uni bolchen 'jodi asho, biryani kom porbe.' Eta ki logic?"
- **বাদী (BN):** "ওস্তাদ, আমার নাম ক্রস করা! কেন? উনি বলছেন 'যদি আসো, বিরিয়ানি কম পড়বে।' এটা কি লজিক?"
- **Bibadi (BL):** "Ostad, biye-te khoroch onek. Ami budget-er jonno... thik ache, Badi biryani onek khay. Eta planning, betrayal na."
- **বিবাদী (BN):** "ওস্তাদ, বিয়েতে খরচ অনেক। আমি বাজেটের জন্য... ঠিক আছে, বাদী বিরিয়ানি অনেক খায়। এটা প্ল্যানিং, বেট্রেয়াল না।"

**M19 — Notes | নোটস** 📚
- **Mamla (BL):** Exam-er age Bibadi Badi-r hate-lekha notes niyechilo. Feroat diyeche cha-er daag soho — ar 3 page gayeb. Bibadi bolche "page-gula... important chilo na."
- **মামলা (BN):** এক্সামের আগে বিবাদী বাদীর হাতে-লেখা নোটস নিয়েছিল। ফেরত দিয়েছে চায়ের দাগ সহ — আর ৩ পৃষ্ঠা গায়েব। বিবাদী বলছে "পৃষ্ঠাগুলো... ইম্পর্ট্যান্ট ছিল না।"
- **Badi (BL):** "Ostad, oi 3 page-e chilo shob formula. Uni 'important na' bolchen — amar result dekhle bujhben, koto important chilo."
- **বাদী (BN):** "ওস্তাদ, ওই ৩ পৃষ্ঠায় ছিল সব ফর্মুলা। উনি 'ইম্পর্ট্যান্ট না' বলছেন — আমার রেজাল্ট দেখলে বুঝবেন, কত ইম্পর্ট্যান্ট ছিল।"
- **Bibadi (BL):** "Ostad, cha-er daag to accident. Ar page... ami bhabchilam oigula revision-er por lagbe na. Ami to positive chintar manush."
- **বিবাদী (BN):** "ওস্তাদ, চায়ের দাগ তো অ্যাক্সিডেন্ট। আর পৃষ্ঠা... আমি ভাবছিলাম ওইগুলো রিভিশনের পর লাগবে না। আমি তো পজিটিভ চিন্তার মানুষ।"

**M20 — Snoring | নাক ডাকা** 😴
- **Mamla (BL):** Sleepover-e Bibadi emon nake dakche je shobai jagga chilo. Shokal-e Bibadi bolche "ami to ghumai ni, ami chokh bondho kore chintaa korchilam." Badi-r chokh-er niche kalir dosh.
- **মামলা (BN):** স্লিপওভারে বিবাদী এমন নাক ডেকেছে যে সবাই জাগা ছিল। সকালে বিবাদী বলছে "আমি তো ঘুমাইনি, আমি চোখ বন্ধ করে চিন্তা করছিলাম।" বাদীর চোখের নিচে কালির দোষ।
- **Badi (BL):** "Ostad, uni bolchen 'chinta korchilam.' Kon chinta manushke truck-er moto shonay, bolun to?"
- **বাদী (BN):** "ওস্তাদ, উনি বলছেন 'চিন্তা করছিলাম।' কোন চিন্তা মানুষকে ট্রাকের মতো শোনায়, বলুন তো?"
- **Bibadi (BL):** "Ostad, nak daka? Ami? Impossible. Ami to khub halka ghumai — ekta pata porle uthe jai. Seti recording thakle... na, thakbe na."
- **বিবাদী (BN):** "ওস্তাদ, নাক ডাকা? আমি? ইম্পসিবল। আমি তো খুব হালকা ঘুমাই — একটা পাতা পড়লে উঠে যাই। সেটা রেকর্ডিং থাকলে... না, থাকবে না।"

**M21 — "Ami To Bolchilam" | "আমি তো বলছিলাম"** 🗣️
- **Mamla (BL):** Badi-r notun recipe experiment fail korar por Bibadi ek dine 17 bar "ami to bolchilam" boleche. Badi-r dabi: eta mental torture, ar protibar-er jonno alada kore shasti chai.
- **মামলা (BN):** বাদীর নতুন রেসিপি এক্সপেরিমেন্ট ফেল করার পর বিবাদী এক দিনে ১৭ বার "আমি তো বলছিলাম" বলেছে। বাদীর দাবি: এটা মেন্টাল টর্চার, আর প্রতিবারের জন্য আলাদা করে শাস্তি চাই।
- **Badi (BL):** "Ostad, 17 bar! Ami count korechi. Ranna ta kharap hoyechilo — kintu uni ki doctor? Uni ki ranna bujhen? Na. Shudhu bolchilen."
- **বাদী (BN):** "ওস্তাদ, ১৭ বার! আমি কাউন্ট করেছি। রান্নাটা খারাপ হয়েছিল — কিন্তু উনি কি ডাক্তার? উনি কি রান্না বোঝেন? না। শুধু বলছিলেন।"
- **Bibadi (BL):** "Ostad, ami to sotti-i bolchilam. Sotti bola ki oporadh? Tahole to adalat-er naam 'Mithya Adalat' hobe."
- **বিবাদী (BN):** "ওস্তাদ, আমি তো সত্যিই বলছিলাম। সত্যি বলা কি অপরাধ? তাহলে তো আদালতের নাম 'মিথ্যা আদালত' হবে।"

**M22 — Aam | আম** 🥭
- **Mamla (BL):** Pasher barir aam gach theke Bibadi aam perechilo. Dhora porar shomoy uni aam-gula Badi-r bag-e dhukiye diyeche — "kichu khon-er jonno." Badi-ke shobai chor bolche.
- **মামলা (BN):** পাশের বাড়ির আম গাছ থেকে বিবাদী আম পেড়েছিল। ধরা পড়ার সময় উনি আমগুলো বাদীর ব্যাগে ঢুকিয়ে দিয়েছে — "কিছুক্ষণের জন্য।" বাদীকে সবাই চোর বলছে।
- **Badi (BL):** "Ostad, amar bag-e aam, amar hate aam-er atha, ar uni bolchen 'tui to niyechili.' Ami to gach-er dike takai ni!"
- **বাদী (BN):** "ওস্তাদ, আমার ব্যাগে আম, আমার হাতে আমের আঠা, আর উনি বলছেন 'তুই তো নিয়েছিলি।' আমি তো গাছের দিকে তাকাইনি!"
- **Bibadi (BL):** "Ostad, bag to tar chilo. Aam bag-e chilo. Logic simple. Ar ami to... aam-gula rakhar jayga khujchilam."
- **বিবাদী (BN):** "ওস্তাদ, ব্যাগ তো তার ছিল। আম ব্যাগে ছিল। লজিক সিম্পল। আর আমি তো... আমগুলো রাখার জায়গা খুঁজছিলাম।"

**M23 — Football Window | ফুটবল জানালা** ⚽
- **Mamla (BL):** Para-r football-e Bibadi-r shot-e ekta janala bhengeche. Bibadi ekhon bolche "Badi kick korechilo, ami to shudhu dariye chilam." Badi-r dabi: uni goalkeeper chilo — shot uni-i mareche.
- **মামলা (BN):** পাড়ার ফুটবলে বিবাদীর শটে একটা জানালা ভেঙেছে। বিবাদী এখন বলছে "বাদী কিক করেছিল, আমি তো শুধু দাঁড়িয়ে ছিলাম।" বাদীর দাবি: উনি গোলকিপার ছিল — শট উনিই মেরেছে।
- **Badi (BL):** "Ostad, uni goalkeeper. Goalkeeper hat diye ball mare — eta niyom. Uni hat diye mareche, janala bhengeche, ar naam amar."
- **বাদী (BN):** "ওস্তাদ, উনি গোলকিপার। গোলকিপার হাত দিয়ে বল মারে — এটা নিয়ম। উনি হাত দিয়ে মেরেছে, জানালা ভেঙেছে, আর নাম আমার।"
- **Bibadi (BL):** "Ostad, ami dariye chilam — eta sotti. Ball amar hat-e eshechilo, kintu ball-er dosh. Ball jodi na ashto, ami ki kortam?"
- **বিবাদী (BN):** "ওস্তাদ, আমি দাঁড়িয়ে ছিলাম — এটা সত্যি। বল আমার হাতে এসেছিল, কিন্তু বলের দোষ। বল যদি না আসত, আমি কী করতাম?"

**M24 — Phone Gallery | ফোন গ্যালারি** 📱
- **Mamla (BL):** Bibadi Badi-r phone niyechilo "ekta call korbo" bole. 30 min por phone ferot — gallery pura scroll kora. Badi-r 200 selfie ar ekta purono cringe video ekhon Bibadi-r mone ache.
- **মামলা (BN):** বিবাদী বাদীর ফোন নিয়েছিল "একটা কল করব" বলে। ৩০ মিনিট পর ফোন ফেরত — গ্যালারি পুরা স্ক্রল করা। বাদীর ২০০ সেলফি আর একটা পুরোনো ক্রিঞ্জ ভিডিও এখন বিবাদীর মনে আছে।
- **Badi (BL):** "Ostad, 'ekta call' bolte uni 30 min bujhiyechen. Ar oi video-ta... oi video-ta niye kono kotha hobe na. Shudhu shasti chai."
- **বাদী (BN):** "ওস্তাদ, 'একটা কল' বলতে উনি ৩০ মিনিট বুঝিয়েছেন। আর ওই ভিডিওটা... ওই ভিডিওটা নিয়ে কোনো কথা হবে না। শুধু শাস্তি চাই।"
- **Bibadi (BL):** "Ostad, call korte giye... gallery khule gechilo. Slip hoye gechilo. Ar selfie-gula to valo chilo — ami to compliment-i korchilam."
- **বিবাদী (BN):** "ওস্তাদ, কল করতে গিয়ে... গ্যালারি খুলে গিয়েছিল। স্লিপ হয়ে গিয়েছিল। আর সেলফিগুলো তো ভালো ছিল — আমি তো কমপ্লিমেন্টই করছিলাম।"

**M25 — Loudspeaker | লাউডস্পিকার** 🔊
- **Mamla (BL):** Exam week-e rat 2-tay Bibadi gaan bajiyechilo full volume-e. Badi-r exam chilo shokal 8-tay. Bibadi bolche "volume to kom chilo — para-r lokera-i complain koreche, ami na."
- **মামলা (BN):** এক্সাম উইকে রাত ২টায় বিবাদী গান বাজিয়েছিল ফুল ভলিউমে। বাদীর এক্সাম ছিল সকাল ৮টায়। বিবাদী বলছে "ভলিউম তো কম ছিল — পাড়ার লোকেরাই কমপ্লেইন করেছে, আমি না।"
- **Badi (BL):** "Ostad, rat 2-ta. Exam 8-ta. Ganer kotha-gula ekhon-o amar mathay ghurche — exam-er formula na, gaan."
- **বাদী (BN):** "ওস্তাদ, রাত ২টা। এক্সাম ৮টা। গানের কথাগুলো এখনও আমার মাথায় ঘুরছে — এক্সামের ফর্মুলা না, গান।"
- **Bibadi (BL):** "Ostad, gaan ta chilo... motivational. Ami to shobaike exam-er jonno motivate korchilam. Para-r lokera bujhe ni."
- **বিবাদী (BN):** "ওস্তাদ, গানটা ছিল... মোটিভেশনাল। আমি তো সবাইকে এক্সামের জন্য মোটিভেট করছিলাম। পাড়ার লোকেরা বোঝেনি।"

**M26 — Plan Cancel | প্ল্যান ক্যান্সেল** 📅
- **Mamla (BL):** "Kalke dekha hobe" — Bibadi ei kotha bole 5 bar plan cancel koreche, shob last minute-e. 6th bar Badi ar bishwash koreni, ar adalat-e esheche. Badi-r dabi: protibar-er jonno ekta kore cha.
- **মামলা (BN):** "কালকে দেখা হবে" — বিবাদী এই কথা বলে ৫ বার প্ল্যান ক্যান্সেল করেছে, সব লাস্ট মিনিটে। ৬ষ্ঠ বার বাদী আর বিশ্বাস করেনি, আর আদালতে এসেছে। বাদীর দাবি: প্রতিবারের জন্য একটা করে চা।
- **Badi (BL):** "Ostad, ami protibar ready hoyechilam. Juta porechilam, ber hoyechilam — tarpor message: 'ajke na, kalke.' 5 bar!"
- **বাদী (BN):** "ওস্তাদ, আমি প্রতিবার রেডি হয়েছিলাম। জুতা পরেছিলাম, বের হয়েছিলাম — তারপর মেসেজ: 'আজকে না, কালকে।' ৫ বার!"
- **Bibadi (BL):** "Ostad, cancel to korechi — kintu 'kalke' to bolechi! Mane ashar iccha chilo. Iccha ta ki count hoy na?"
- **বিবাদী (BN):** "ওস্তাদ, ক্যান্সেল তো করেছি — কিন্তু 'কালকে' তো বলেছি! মানে আসার ইচ্ছা ছিল। ইচ্ছাটা কি কাউন্ট হয় না?"

**M27 — Cake-er Cream | কেকের ক্রিম** 🎂
- **Mamla (BL):** Birthday cake cut korar age Bibadi cake-er corner-er cream ar dui-ta cherry kheye feleche — "shudhu taste korchilam." Badi-r dabi: cake cut-er age cake khawa birthday-ain-er lankhon.
- **মামলা (BN):** বার্থডে কেক কাটার আগে বিবাদী কেকের কর্নারের ক্রিম আর দুইটা চেরি খেয়ে ফেলেছে — "শুধু টেস্ট করছিলাম।" বাদীর দাবি: কেক কাটার আগে কেক খাওয়া বার্থডে-আইনের লঙ্ঘন।
- **Badi (BL):** "Ostad, cake-er photo tulbo, tarpor cut — eta niyom. Uni photo-r age-i cream... cherry... shob. Ekhon cake ta dekhte 'already opened' lagche."
- **বাদী (BN):** "ওস্তাদ, কেকের ফটো তুলব, তারপর কাট — এটা নিয়ম। উনি ফটোর আগেই ক্রিম... চেরি... সব। এখন কেকটা দেখতে 'অলরেডি ওপেনড' লাগছে।"
- **Bibadi (BL):** "Ostad, taste kora to quality check! Ami to nishchit korchilam cake ta valo. Na hole birthday ta kharap hoto."
- **বিবাদী (BN):** "ওস্তাদ, টেস্ট করা তো কোয়ালিটি চেক! আমি তো নিশ্চিত করছিলাম কেকটা ভালো। না হলে বার্থডেটা খারাপ হতো।"

**M28 — Auto Fare | অটো ভাড়া** 🛺
- **Mamla (BL):** Auto-r bhara 90 taka. Dui jon mile dewar kotha — Bibadi bolche "ami 50 diyechi." Badi bolche "ami 50 diyechi." Auto mama bolchen "dui jon-i 40 kore diyecho." Hisab milche na.
- **মামলা (BN):** অটোর ভাড়া ৯০ টাকা। দুই জন মিলে দেওয়ার কথা — বিবাদী বলছে "আমি ৫০ দিয়েছি।" বাদী বলছে "আমি ৫০ দিয়েছি।" অটো মামা বলছেন "দুই জনই ৪০ করে দিয়েছ।" হিসাব মিলছে না।
- **Badi (BL):** "Ostad, 40+40 = 80. Bhara 90. Baki 10 taka ke debe? Ar uni bolchen 50 diyechen — calculator ki mithya bolche?"
- **বাদী (BN):** "ওস্তাদ, ৪০+৪০ = ৮০। ভাড়া ৯০। বাকি ১০ টাকা কে দেবে? আর উনি বলছেন ৫০ দিয়েছেন — ক্যালকুলেটর কি মিথ্যা বলছে?"
- **Bibadi (BL):** "Ostad, ami 50-i diyechi. Mama-r hisab bhul. Ar jodi 40-i hoy... tobe seta amar na, onno karo 40."
- **বিবাদী (BN):** "ওস্তাদ, আমি ৫০ই দিয়েছি। মামার হিসাব ভুল। আর যদি ৪০ই হয়... তবে সেটা আমার না, অন্য কারো ৪০।"

**M29 — Regift | রিগিফট** 🎁
- **Mamla (BL):** Badi Bibadi-ke birthday-te ekta gift diyechilo. Bibadi sei gift-i onno ek bondhu-ke diyeche — ar shei bondhu abar Badi-ke. Gift ta ekhon Badi-r hate, original box soho.
- **মামলা (BN):** বাদী বিবাদীকে বার্থডেতে একটা গিফট দিয়েছিল। বিবাদী সেই গিফটই অন্য এক বন্ধুকে দিয়েছে — আর সেই বন্ধু আবার বাদীকে। গিফটটা এখন বাদীর হাতে, অরিজিনাল বক্স সহ।
- **Badi (BL):** "Ostad, gift ta ghure firure amar kache esheche. Box-e amar nijer handwriting: 'Happy Birthday, Bibadi.' Eta ki gift, na boomerang?"
- **বাদী (BN):** "ওস্তাদ, গিফটটা ঘুরে ফিরে আমার কাছে এসেছে। বক্সে আমার নিজের হ্যান্ডরাইটিং: 'হ্যাপি বার্থডে, বিবাদী।' এটা কি গিফট, না বুমেরাং?"
- **Bibadi (BL):** "Ostad, gift to bondhu-der moddhe ghurte-i thake. Eta recycling — environment bachchi. Ar gift ta to valo chilo, tai na?"
- **বিবাদী (BN):** "ওস্তাদ, গিফট তো বন্ধুদের মধ্যে ঘুরতেই থাকে। এটা রিসাইক্লিং — এনভায়রনমেন্ট বাঁচাচ্ছি। আর গিফটটা তো ভালো ছিল, তাই না?"

**M30 — Order | অর্ডার** 🍔
- **Mamla (BL):** Bibadi Badi-r phone theke food order koreche — Badi-r saved card diye. "Tumi to order korte bolchila." Kintu order-er 70% Bibadi kheyeche, ar Badi peyeche shudhu ekta fry.
- **মামলা (BN):** বিবাদী বাদীর ফোন থেকে ফুড অর্ডার করেছে — বাদীর সেভড কার্ড দিয়ে। "তুমি তো অর্ডার করতে বলছিলা।" কিন্তু অর্ডারের ৭০% বিবাদী খেয়েছে, আর বাদী পেয়েছে শুধু একটা ফ্রাই।
- **Badi (BL):** "Ostad, amar phone, amar card, amar taka — ar amar plate-e ekta fry. Uni bolchen 'share'. Ekta fry ki share?"
- **বাদী (BN):** "ওস্তাদ, আমার ফোন, আমার কার্ড, আমার টাকা — আর আমার প্লেটে একটা ফ্রাই। উনি বলছেন 'শেয়ার'। একটা ফ্রাই কি শেয়ার?"
- **Bibadi (BL):** "Ostad, order korte to tumi bolchila! Ami to shudhu... list ta boro korechilam. Ar bhook ta chilo dui jon-er, kintu stomach ta..."
- **বিবাদী (BN):** "ওস্তাদ, অর্ডার করতে তো তুমি বলছিলা! আমি তো শুধু... লিস্টটা বড় করেছিলাম। আর ভুখটা ছিল দুই জনের, কিন্তু স্টমাকটা..."

---

### 10.2 Sakkhi Cards (20)

Each card: **persona name**, **persona line** (who this witness is), **lean** (Badi / Bibadi / Neutral), **testimony starter** (Mamla-agnostic improv launchpad — at 3+ only the witness sees the card; in 2P deck mode the full text is read aloud), **pressure question** (the cross-exam prompt; in 2P both players answer it in turn).

**Lean distribution:** Badi ×7 (S01, S04, S07, S10, S13, S16, S20) · Bibadi ×7 (S03, S06, S09, S11, S12, S15, S18) · Neutral ×6 (S02, S05, S08, S14, S17, S19).
**Deck math:** max draws per match = 6 rounds × 3 witnesses = 18 ≤ 20 → no repeats within a match. ✔

---

**S01 — Cha-er Dokaner Mamu | চায়ের দোকানের মামু** ☕ · Lean: **Badi**
- **Persona (BL):** Cha banate banate shob shonen. Tar theory: je beshi cha khay, shey beshi mithya bole.
- **চরিত্র (BN):** চা বানাতে বানাতে সব শোনেন। তার থিওরি: যে বেশি চা খায়, সে বেশি মিথ্যা বলে।
- **Testimony (BL):** "Ami kettle-r pichon theke shob dekhechi. Oi din... dui jon-er moddhe ekjon-er cha thanda hoye gechilo. Ar je cha thanda korte dey, shey..."
- **সাক্ষ্য (BN):** "আমি কেটলির পিছন থেকে সব দেখেছি। ওই দিন... দুই জনের মধ্যে একজনের চা ঠান্ডা হয়ে গিয়েছিল। আর যে চা ঠান্ডা করতে দেয়, সে..."
- **Pressure (BL):** "Mamu, apnar kettle ki shob shone? Tahole amader kotha-gula ki record hoyeche?"
- **চাপ প্রশ্ন (BN):** "মামু, আপনার কেটলি কি সব শোনে? তাহলে আমাদের কথাগুলো কি রেকর্ড হয়েছে?"

**S02 — Parar Nani | পাড়ার নানি** 👵 · Lean: **Neutral**
- **Persona (BL):** Shob kichu "pasher barir bou" theke shonen. Tar kotha-r 80% gossip, 20% ashirbad.
- **চরিত্র (BN):** সব কিছু "পাশের বাড়ির বউ" থেকে শোনেন। তার কথার ৮০% গসিপ, ২০% আশীর্বাদ।
- **Testimony (BL):** "Baba, ami shob jani. Pasher barir bou bolchilo... ki jeno bolchilo... ha, mone poreche — na, poreni. Kintu ekta kotha bolchi, shuno..."
- **সাক্ষ্য (BN):** "বাবা, আমি সব জানি। পাশের বাড়ির বউ বলছিল... কী যেন বলছিল... হ্যাঁ, মনে পড়েছে — না, পড়েনি। কিন্তু একটা কথা বলছি, শোনো..."
- **Pressure (BL):** "Nani, pasher barir bou ki asolei dekhechilo, naki uni-o onno karo kach theke shunechilo?"
- **চাপ প্রশ্ন (BN):** "নানি, পাশের বাড়ির বউ কি আসলেই দেখেছিল, নাকি উনিও অন্য কারো কাছ থেকে শুনেছিল?"

**S03 — Building-er Darwan | বিল্ডিংয়ের দারোয়ান** 🚪 · Lean: **Bibadi**
- **Persona (BL):** "Ami shob dekhi" — kintu tar CCTV nei. Gate-e boshe thaken, shobai-ke chenen, kichu-i mone rakhen na.
- **চরিত্র (BN):** "আমি সব দেখি" — কিন্তু তার সিসিটিভি নেই। গেটে বসে থাকেন, সবাইকে চেনেন, কিছুই মনে রাখেন না।
- **Testimony (BL):** "Ami gate-e chilam. Shob dekhechi. CCTV-te... ache. Mane, thakle thakto. Kintu amar chokh-i CCTV. Oi din je gechilo, shey..."
- **সাক্ষ্য (BN):** "আমি গেটে ছিলাম। সব দেখেছি। সিসিটিভিতে... আছে। মানে, থাকলে থাকত। কিন্তু আমার চোখই সিসিটিভি। ওই দিন যে গিয়েছিল, সে..."
- **Pressure (BL):** "Darwan kaka, apnar CCTV kothay? Dekhan to ekbar — na thakle chokh-er CCTV ki kore kaj kore?"
- **চাপ প্রশ্ন (BN):** "দারোয়ান কাকা, আপনার সিসিটিভি কোথায়? দেখান তো একবার — না থাকলে চোখের সিসিটিভি কী করে কাজ করে?"

**S04 — Rickshaw Mama | রিকশা মামা** 🛺 · Lean: **Badi**
- **Persona (BL):** Traffic jam-e manush ja bole, shob shonen. Tar dabi: jam-e manush shotti kotha bole.
- **চরিত্র (BN):** ট্রাফিক জ্যামে মানুষ যা বলে, সব শোনেন। তার দাবি: জ্যামে মানুষ সত্যি কথা বলে।
- **Testimony (BL):** "Jam-e chilam, 40 minute. Pichon-er seat-e dui jon boshechilo. Ja bolchilo... ami to rickshaw chalai, kintu kaan dui-ta khola rakhi."
- **সাক্ষ্য (BN):** "জ্যামে ছিলাম, ৪০ মিনিট। পিছনের সিটে দুই জন বসেছিল। যা বলছিল... আমি তো রিকশা চালাই, কিন্তু কান দুইটা খোলা রাখি।"
- **Pressure (BL):** "Mama, apni ki shob ride-er kotha mone rakhen, naki shudhu ei mamla-r kotha-i mone ache?"
- **চাপ প্রশ্ন (BN):** "মামা, আপনি কি সব রাইডের কথা মনে রাখেন, নাকি শুধু এই মামলার কথাই মনে আছে?"

**S05 — Class-er Monitor | ক্লাসের মনিটর** 🤓 · Lean: **Neutral**
- **Persona (BL):** Shobar naam board-e likhte chan. Adalat-ke ekta class bhabchen. Shobai-ke "chup" korate chan.
- **চরিত্র (BN):** সবার নাম বোর্ডে লিখতে চান। আদালতকে একটা ক্লাস ভাবছেন। সবাইকে "চুপ" করাতে চান।
- **Testimony (BL):** "Ami attendance register-e shob likhechi. Oi din ke present chilo, ke late chilo, ke kotha bolchilo — shob. Naam bolbo? Board ta kothay?"
- **সাক্ষ্য (BN):** "আমি অ্যাটেনডেন্স রেজিস্টারে সব লিখেছি। ওই দিন কে প্রেজেন্ট ছিল, কে লেট ছিল, কে কথা বলছিল — সব। নাম বলব? বোর্ডটা কোথায়?"
- **Pressure (BL):** "Monitor, apnar register-e ki amar naam-o ache? Ki lekha ache amar naam-er pashe?"
- **চাপ প্রশ্ন (BN):** "মনিটর, আপনার রেজিস্টারে কি আমার নামও আছে? কী লেখা আছে আমার নামের পাশে?"

**S06 — Pasher Barir Aunty | পাশের বাড়ির আন্টি** 🏘️ · Lean: **Bibadi**
- **Persona (BL):** Balcony theke shob dekhen. Tar "amar chele to emon na" energy shobai-ke bachay — shudhu tar chele na, shobai-ke.
- **চরিত্র (BN):** ব্যালকনি থেকে সব দেখেন। তার "আমার ছেলে তো এমন না" এনার্জি সবাইকে বাঁচায় — শুধু তার ছেলে না, সবাইকে।
- **Testimony (BL):** "Ami balcony-te kapor melchilam. Shob dekhechi. Ei chele-ta... emon na. Mane, jai hok, ami bolchi na je emon. Kintu oi din..."
- **সাক্ষ্য (BN):** "আমি ব্যালকনিতে কাপড় মেলছিলাম। সব দেখেছি। এই ছেলেটা... এমন না। মানে, যাই হোক, আমি বলছি না যে এমন। কিন্তু ওই দিন..."
- **Pressure (BL):** "Aunty, apnar balcony theke asolei dekha jay, naki shudhu onuman koren?"
- **চাপ প্রশ্ন (BN):** "আন্টি, আপনার ব্যালকনি থেকে আসলেই দেখা যায়, নাকি শুধু অনুমান করেন?"

**S07 — Muudi Dokaner Kaka | মুদি দোকানের কাকা** 🏪 · Lean: **Badi**
- **Persona (BL):** Tar udhar khata-i shob evidence. Je kichu-i hok, khata-r ekta page-e tar prooman ache.
- **চরিত্র (BN):** তার উধার খাতাই সব এভিডেন্স। যে কিছুই হোক, খাতার একটা পাতায় তার প্রমাণ আছে।
- **Testimony (BL):** "Khata dekhen. Ei page-e... ha, ei je. Hisab shob lekha. Taka, cha, biscuit — ar oi din-er ghotona-o. Mane, ghotona ta lekha nei, kintu hisab theke bojha jay."
- **সাক্ষ্য (BN):** "খাতা দেখেন। এই পাতায়... হ্যাঁ, এই যে। হিসাব সব লেখা। টাকা, চা, বিস্কুট — আর ওই দিনের ঘটনাও। মানে, ঘটনাটা লেখা নেই, কিন্তু হিসাব থেকে বোঝা যায়।"
- **Pressure (BL):** "Kaka, khata-e ki shotti oi din-er kichu lekha ache, naki shudhu biscuit-er hisab?"
- **চাপ প্রশ্ন (BN):** "কাকা, খাতায় কি সত্যি ওই দিনের কিছু লেখা আছে, নাকি শুধু বিস্কুটের হিসাব?"

**S08 — Library-r Sir | লাইব্রেরির স্যার** 📚 · Lean: **Neutral**
- **Persona (BL):** Shob kichu fushfush kore bolen. Adalat-ke shush korate chan. Tar kotha shunte hole kaan pete hoy.
- **চরিত্র (BN):** সব কিছু ফিসফিস করে বলেন। আদালতকে শুশ করাতে চান। তার কথা শুনতে হলে কান পেতে হয়।
- **Testimony (BL):** "(fushfush) Ami library-te chilam. Shob shunechi. Kintu jore bola jay na... (aro choto) oi din je kotha bolchilo, tar shobdo... (shush!)"
- **সাক্ষ্য (BN):** "(ফিসফিস) আমি লাইব্রেরিতে ছিলাম। সব শুনেছি। কিন্তু জোরে বলা যায় না... (আরো ছোট) ওই দিন যে কথা বলছিল, তার শব্দ... (শুশ!)"
- **Pressure (BL):** "Sir, ektu jore bolun — amra shunte pacchi na. Apni ki asolei kichu shunechilen?"
- **চাপ প্রশ্ন (BN):** "স্যার, একটু জোরে বলুন — আমরা শুনতে পাচ্ছি না। আপনি কি আসলেই কিছু শুনেছিলেন?"

**S09 — Para-r Cricket Captain | পাড়ার ক্রিকেট ক্যাপ্টেন** 🏏 · Lean: **Bibadi**
- **Persona (BL):** Shob kichu-ke cricket analogy banan. DRS chan. Toss niye kotha bolen.
- **চরিত্র (BN):** সব কিছুকে ক্রিকেট অ্যানালজি বানান। ডিআরএস চান। টস নিয়ে কথা বলেন।
- **Testimony (BL):** "Dekhen, cricket-e ki hoy? Ball ekbar, bat ekbar. Ekhane-o same. Oi din je out hoyechilo... mane, je doshi chilo... DRS nite hobe. Amar review ache."
- **সাক্ষ্য (BN):** "দেখেন, ক্রিকেটে কী হয়? বল একবার, ব্যাট একবার। এখানেও সেম। ওই দিন যে আউট হয়েছিল... মানে, যে দোষী ছিল... ডিআরএস নিতে হবে। আমার রিভিউ আছে।"
- **Pressure (BL):** "Captain, apnar review ki asolei kichu dekheche, naki shudhu match-er moto feel hocche?"
- **চাপ প্রশ্ন (BN):** "ক্যাপ্টেন, আপনার রিভিউ কি আসলেই কিছু দেখেছে, নাকি শুধু ম্যাচের মতো ফিল হচ্ছে?"

**S10 — Group Chat Admin | গ্রুপ চ্যাট অ্যাডমিন** 📱 · Lean: **Badi**
- **Persona (BL):** Tar kache screenshot ache. Shob kichur screenshot ache. Kintu relevant screenshot ekta-o nei.
- **চরিত্র (BN):** তার কাছে স্ক্রিনশট আছে। সব কিছুর স্ক্রিনশট আছে। কিন্তু রেলিভেন্ট স্ক্রিনশট একটাও নেই।
- **Testimony (BL):** "Screenshot ache. Ei dekhun... na, eta onno din-er. Eta... eta to cha-er photo. Kintu ekta screenshot ache jeta... mone hocche... thak, pore pathabo."
- **সাক্ষ্য (BN):** "স্ক্রিনশট আছে। এই দেখুন... না, এটা অন্য দিনের। এটা... এটা তো চায়ের ফটো। কিন্তু একটা স্ক্রিনশট আছে যেটা... মনে হচ্ছে... থাক, পরে পাঠাব।"
- **Pressure (BL):** "Admin, ekhon-i dekhun — relevant screenshot ta ki asolei ache, naki 'pore pathabo' forever?"
- **চাপ প্রশ্ন (BN):** "অ্যাডমিন, এখনই দেখুন — রেলিভেন্ট স্ক্রিনশটটা কি আসলেই আছে, নাকি 'পরে পাঠাব' ফরএভার?"

**S11 — Tailor Master | টেইলার মাস্টার** 🧵 · Lean: **Bibadi**
- **Persona (BL):** Fita diye shob mapen — jomi na, kotha. Tar dabi: map thik thakle kotha thik.
- **চরিত্র (BN):** ফিতা দিয়ে সব মাপেন — জমি না, কথা। তার দাবি: মাপ ঠিক থাকলে কথা ঠিক।
- **Testimony (BL):** "Ami fita niye eshechi. Kotha-r map nite hobe. Ei je... dui jon-er kotha-r length alada. Ekjon-er kotha lomba, arekjon-er choto. Je choto, shey..."
- **সাক্ষ্য (BN):** "আমি ফিতা নিয়ে এসেছি। কথার মাপ নিতে হবে। এই যে... দুই জনের কথার লেংথ আলাদা। একজনের কথা লম্বা, আরেকজনের ছোট। যে ছোট, সে..."
- **Pressure (BL):** "Master, kotha mapar niyom ta ke banalo? Apni ki age karo kotha meter korechen?"
- **চাপ প্রশ্ন (BN):** "মাস্টার, কথা মাপার নিয়মটা কে বানাল? আপনি কি আগে কারো কথা মাপ করেছেন?"

**S12 — Photocopy Dokaner Da | ফটোকপি দোকানের দা** 📄 · Lean: **Bibadi**
- **Persona (BL):** "Original-er copy amar kache ache" — kintu original kar, janen na. Copy machine-i tar shob.
- **চরিত্র (BN):** "অরিজিনালের কপি আমার কাছে আছে" — কিন্তু অরিজিনাল কার, জানেন না। কপি মেশিনই তার সব।
- **Testimony (BL):** "Copy ache. Original... original ta karo kache chilo. Ami copy korechi. Copy te shob ache — shudhu original-er je jaygay fold chilo, shei jaygay... ache."
- **সাক্ষ্য (BN):** "কপি আছে। অরিজিনাল... অরিজিনালটা কারো কাছে ছিল। আমি কপি করেছি। কপিতে সব আছে — শুধু অরিজিনালের যে জায়গায় ফোল্ড ছিল, সেই জায়গায়... আছে।"
- **Pressure (BL):** "Da, original ta kothay? Copy jodi thake, original to ekhono karo kache ache — ke shei?"
- **চাপ প্রশ্ন (BN):** "দা, অরিজিনালটা কোথায়? কপি যদি থাকে, অরিজিনাল তো এখনো কারো কাছে আছে — কে সেই?"

**S13 — Ice-cream Wala | আইসক্রিম ওয়ালা** 🍦 · Lean: **Badi**
- **Persona (BL):** Tar chaos theory: je ice-cream khay na, shey nirdosh hote pare na. Ghonta bajay, kintu shomoy janen na.
- **চরিত্র (BN):** তার কাওস থিওরি: যে আইসক্রিম খায় না, সে নির্দোষ হতে পারে না। ঘণ্টা বাজায়, কিন্তু সময় জানেন না।
- **Testimony (BL):** "Ami ghonta bajai, shob shuni. Oi din... ekjon ice-cream khaiche, arekjon khayni. Je khayni... hmm. Amar theory ache. Shunben?"
- **সাক্ষ্য (BN):** "আমি ঘণ্টা বাজাই, সব শুনি। ওই দিন... একজন আইসক্রিম খেয়েছে, আরেকজন খায়নি। যে খায়নি... হুম। আমার থিওরি আছে। শুনবেন?"
- **Pressure (BL):** "Ice-cream wala, apnar theory ki science, naki shudhu ice-cream bechar upay?"
- **চাপ প্রশ্ন (BN):** "আইসক্রিম ওয়ালা, আপনার থিওরি কি সায়েন্স, নাকি শুধু আইসক্রিম বেচার উপায়?"

**S14 — Night Guard | নাইট গার্ড** 🔦 · Lean: **Neutral**
- **Persona (BL):** Rat 2-tay "kichu ekta" dekhechilen. Torch-ta jhuki-jhuki kore. Kichu-i thik mone nei.
- **চরিত্র (BN):** রাত ২টায় "কিছু একটা" দেখেছিলেন। টর্চটা ঝুকি-ঝুকি করে। কিছুই ঠিক মনে নেই।
- **Testimony (BL):** "Rat chilo. Torch chilo. Kichu ekta... move korchilo. Manush, naki biral, naki... torch-ta thik kaj kore na. Kintu ekta chhaya chilo. Chhaya ta..."
- **সাক্ষ্য (BN):** "রাত ছিল। টর্চ ছিল। কিছু একটা... মুভ করছিল। মানুষ, নাকি বিড়াল, নাকি... টর্চটা ঠিক কাজ করে না। কিন্তু একটা ছায়া ছিল। ছায়াটা..."
- **Pressure (BL):** "Guard, chhaya ta ki manush chilo? Apni ki torch thik kore age kichu dekhechen?"
- **চাপ প্রশ্ন (BN):** "গার্ড, ছায়াটা কি মানুষ ছিল? আপনি কি টর্চ ঠিক করে আগে কিছু দেখেছেন?"

**S15 — Barber Kaka | নাপিত কাকা** 💈 · Lean: **Bibadi**
- **Persona (BL):** Chair-e boshe manush shob shotti kotha bole. Tar chair-i adalat-er shreshtho sakkhi.
- **চরিত্র (BN):** চেয়ারে বসে মানুষ সব সত্যি কথা বলে। তার চেয়ারই আদালতের শ্রেষ্ঠ সাক্ষী।
- **Testimony (BL):** "Chair-e boshe manush ja bole... haircut-er shomoy manush nirdosh hoy. Ami onek kotha shunechi. Oi din-er kotha... chair ta ekhon khali, kintu kotha-gula ache."
- **সাক্ষ্য (BN):** "চেয়ারে বসে মানুষ যা বলে... হেয়ারকাটের সময় মানুষ নির্দোষ হয়। আমি অনেক কথা শুনেছি। ওই দিনের কথা... চেয়ারটা এখন খালি, কিন্তু কথাগুলো আছে।"
- **Pressure (BL):** "Kaka, chair-e ke boshechilo oi din? Naam bolun — chair ki shob mone rakhe?"
- **চাপ প্রশ্ন (BN):** "কাকা, চেয়ারে কে বসেছিল ওই দিন? নাম বলুন — চেয়ার কি সব মনে রাখে?"

**S16 — Bus Conductor | বাস কন্ডাক্টর** 🚌 · Lean: **Badi**
- **Persona (BL):** "Ticket-er moto kotha — ekbar kata gele ar ferot paben na." Shob kichu-ke ticket-er sathe tulona koren.
- **চরিত্র (BN):** "টিকিটের মতো কথা — একবার কাটা গেলে আর ফেরত পাবেন না।" সব কিছুকে টিকিটের সাথে তুলনা করেন।
- **Testimony (BL):** "Ticket kata hoyeche. Mane, kotha bola hoyeche. Ekhon ferot pawar upay nei. Oi din je ticket kateche... mane, je kotha boleche... seta tar. Hisab simple."
- **সাক্ষ্য (BN):** "টিকিট কাটা হয়েছে। মানে, কথা বলা হয়েছে। এখন ফেরত পাওয়ার উপায় নেই। ওই দিন যে টিকিট কেটেছে... মানে, যে কথা বলেছে... সেটা তার। হিসাব সিম্পল।"
- **Pressure (BL):** "Conductor, ticket-er sathe kotha-r ki somporko? Apni ki asolei kichu dekhechen, naki shudhu analogy?"
- **চাপ প্রশ্ন (BN):** "কন্ডাক্টর, টিকিটের সাথে কথার কী সম্পর্ক? আপনি কি আসলেই কিছু দেখেছেন, নাকি শুধু অ্যানালজি?"

**S17 — Mistir Dokaner Meye | মিষ্টির দোকানের মেয়ে** 🍬 · Lean: **Neutral**
- **Persona (BL):** Mishti kore bolen, kintu tar kotha devastating. Hashi-mukhe shobai-ke dhore falen.
- **চরিত্র (BN):** মিষ্টি করে বলেন, কিন্তু তার কথা ডেভাস্টেটিং। হাসি-মুখে সবাইকে ধরে ফেলেন।
- **Testimony (BL):** "(hashi) Ami to kichu bolbo na. Mane, bolbo — kintu mishti kore. Oi din ja hoyechilo... (aro hashi) thak, na bolle-i valo. Apnara to shob-i janen."
- **সাক্ষ্য (BN):** "(হাসি) আমি তো কিছু বলব না। মানে, বলব — কিন্তু মিষ্টি করে। ওই দিন যা হয়েছিল... (আরো হাসি) থাক, না বললেই ভালো। আপনারা তো সবই জানেন।"
- **Pressure (BL):** "Apni haschen keno? Ja janen, sheta bolun — hasi diye dhaka jabe na."
- **চাপ প্রশ্ন (BN):** "আপনি হাসছেন কেন? যা জানেন, সেটা বলুন — হাসি দিয়ে ঢাকা যাবে না।"

**S18 — Postman | পিয়ন** 📮 · Lean: **Bibadi**
- **Persona (BL):** Thikana thik na thakle chithi ferot pathan. Proshno-gula-o ferot pathan — "thikana nei."
- **চরিত্র (BN):** ঠিকানা ঠিক না থাকলে চিঠি ফেরত পাঠান। প্রশ্নগুলোও ফেরত পাঠান — "ঠিকানা নেই।"
- **Testimony (BL):** "Chithi eshechilo. Thikana... thik chilo na. Ferot geche. Oi din-er ghotona-o ekta chithi-r moto — thikana thik na thakle, ferot jay. Ami shudhu deliver kori."
- **সাক্ষ্য (BN):** "চিঠি এসেছিল। ঠিকানা... ঠিক ছিল না। ফেরত গেছে। ওই দিনের ঘটনাও একটা চিঠির মতো — ঠিকানা ঠিক না থাকলে, ফেরত যায়। আমি শুধু ডেলিভার করি।"
- **Pressure (BL):** "Postman, chithi-r thikana ke bhul korechilo? Apni ki janen shei thikana ta ki chilo?"
- **চাপ প্রশ্ন (BN):** "পিয়ন, চিঠির ঠিকানা কে ভুল করেছিল? আপনি কি জানেন সেই ঠিকানাটা কী ছিল?"

**S19 — Para-r Kukur-er Malok | পাড়ার কুকুরের মালিক** 🐕 · Lean: **Neutral**
- **Persona (BL):** Tar kukur naki mithya bolle ghew-ghew kore. Kukur-ta ekhon ghumacche.
- **চরিত্র (BN):** তার কুকুর নাকি মিথ্যা বললে ঘেউ-ঘেউ করে। কুকুরটা এখন ঘুমাচ্ছে।
- **Testimony (BL):** "Amar Tommy shob bojhe. Mithya bolle ghew-ghew kore. Oi din... kichu ekta hoyechilo. Tommy ekhon ghumacche — mane, shob shotti. Naki... ghum mane kichu na."
- **সাক্ষ্য (BN):** "আমার টমি সব বোঝে। মিথ্যা বললে ঘেউ-ঘেউ করে। ওই দিন... কিছু একটা হয়েছিল। টমি এখন ঘুমাচ্ছে — মানে, সব সত্যি। নাকি... ঘুম মানে কিছু না।"
- **Pressure (BL):** "Tommy-ke ekhon jagiye jiggesh korun — shey ki bolbe? Ghumonto kukur ki sakkhi hote pare?"
- **চাপ প্রশ্ন (BN):** "টমিকে এখন জাগিয়ে জিজ্ঞেস করুন — সে কী বলবে? ঘুমন্ত কুকুর কি সাক্ষী হতে পারে?"

**S20 — Ochena Ekjon | অচেনা একজন** 🕶️ · Lean: **Badi**
- **Persona (BL):** Ke janen ke. Kothay theke eshechen, janen na. Shudhu "ekta jinish" dekhechilen.
- **চরিত্র (BN):** কে জানে কে। কোথা থেকে এসেছেন, জানেন না। শুধু "একটা জিনিস" দেখেছিলেন।
- **Testimony (BL):** "Ami... ekhane chilam. Keno chilam, bolbo na. Shudhu ekta jinish dekhechi. Ekta. Jinish ta... chilo. Ar ekhon... ache. Eta-i amar sakkhi."
- **সাক্ষ্য (BN):** "আমি... এখানে ছিলাম। কেন ছিলাম, বলব না। শুধু একটা জিনিস দেখেছি। একটা। জিনিসটা... ছিল। আর এখন... আছে। এটাই আমার সাক্ষ্য।"
- **Pressure (BL):** "Apni ke? Oi 'ekta jinish' ki chilo — ek kothay bolun. Na bolle apnar sakkhi batil."
- **চাপ প্রশ্ন (BN):** "আপনি কে? ওই 'একটা জিনিস' কী ছিল — এক কথায় বলুন। না বললে আপনার সাক্ষ্য বাতিল।"

---

### 10.3 Ray Lines (15)

Tags: **Doshi ×5** (R01–R05) · **Nirdosh ×5** (R06–R10) · **Any ×5** (R11–R15, eligible for both verdicts and hung courts). Draw: next unused line matching the verdict (pools of 10 each ≥ max 6 verdicts per match → no repeats). Every Ray is **Mamla-agnostic** and carries a **Shasti** — a gentle, family-friendly, honor-system forfeit. Shasti targets the loser unless stated otherwise.

---

**R01 — Bishwash Bhokkhon | বিশ্বাস ভক্ষণ** · Tag: **Doshi**
- **Ray (BL):** "Doshi! Tumi shudhu mamla na, bondhu-r bishwash-o kheye felecho. Kintu chinta nei — cha-er dokane abar dekha hobe."
- **রায় (BN):** "দোষী! তুমি শুধু মামলা না, বন্ধুর বিশ্বাসও খেয়ে ফেলেছ। কিন্তু চিন্তা নেই — চায়ের দোকানে আবার দেখা হবে।"
- **Shasti (BL):** "Next adda-er cha-er bill tumi debe."
- **শাস্তি (BN):** "নেক্সট আড্ডার চায়ের বিল তুমি দেবে।"

**R02 — Expression Sotti | এক্সপ্রেশন সত্যি** · Tag: **Doshi**
- **Ray (BL):** "Doshi! Tomar kotha-r cheye tomar mukh-er expression beshi sotti bolchilo. Ei adalat chokh diye dekhte pay."
- **রায় (BN):** "দোষী! তোমার কথার চেয়ে তোমার মুখের এক্সপ্রেশন বেশি সত্যি বলছিল। এই আদালত চোখ দিয়ে দেখতে পায়।"
- **Shasti (BL):** "Ekhon theke 'kalke debo' bolte gele age adalat-er onumoti nite hobe."
- **শাস্তি (BN):** "এখন থেকে 'কালকে দেব' বলতে গেলে আগে আদালতের অনুমতি নিতে হবে।"

**R03 — Hasi-r Shasti | হাসির শাস্তি** · Tag: **Doshi**
- **Ray (BL):** "Doshi! Tumi nirdosh hote parte — kintu tomar hasi tomar hoye gelo. Heshe case hara, eta notun record."
- **রায় (BN):** "দোষী! তুমি নির্দোষ হতে পারতে — কিন্তু তোমার হাসি তোমার হয়ে গেল। হেসে কেস হারা, এটা নতুন রেকর্ড।"
- **Shasti (BL):** "Shobar samne 3 bar bolo: 'Ami ar emon korbo na' — hasi chara."
- **শাস্তি (BN):** "সবার সামনে ৩ বার বলো: 'আমি আর এমন করব না' — হাসি ছাড়া।"

**R04 — Chokh-er Adalat | চোখের আদালত** · Tag: **Doshi**
- **Ray (BL):** "Doshi! Evidence chilo na, kintu tomar chokh-er dike takiyei bojha jacchilo. Ei adalat-er chokh ache, ar memory-o ache."
- **রায় (BN):** "দোষী! এভিডেন্স ছিল না, কিন্তু তোমার চোখের দিকে তাকিয়েই বোঝা যাচ্ছিল। এই আদালতের চোখ আছে, আর মেমোরিও আছে।"
- **Shasti (BL):** "Next dawat-e shob plate tumi poribeshon korbe."
- **শাস্তি (BN):** "নেক্সট দাওয়াতে সব প্লেট তুমি পরিবেশন করবে।"

**R05 — Appeal-er Bill | আপিলের বিল** · Tag: **Doshi**
- **Ray (BL):** "Doshi! Ei ray-er biruddhe appeal korte parbe — kintu cha-er bill diye. Appeal-er form cha-er dokane pawa jay."
- **রায় (BN):** "দোষী! এই রায়ের বিরুদ্ধে আপিল করতে পারবে — কিন্তু চায়ের বিল দিয়ে। আপিলের ফর্ম চায়ের দোকানে পাওয়া যায়।"
- **Shasti (BL):** "Group chat-e 'ami doshi' likhe 10 min rakhte hobe."
- **শাস্তি (BN):** "গ্রুপ চ্যাটে 'আমি দোষী' লিখে ১০ মিনিট রাখতে হবে।"

**R06 — Nijei Hara Case | নিজেই হারা কেস** · Tag: **Nirdosh**
- **Ray (BL):** "Nirdosh! Badi-r kotha shune mone hoyechilo uni nijei nijer case harben. Case closed — Badi-r kotha-i evidence chilo."
- **রায় (BN):** "নির্দোষ! বাদীর কথা শুনে মনে হয়েছিল উনি নিজেই নিজের কেস হারবেন। কেস ক্লোজড — বাদীর কথাই এভিডেন্স ছিল।"
- **Shasti (BL):** "Badi next adda-e shobar prothom cha banabe."
- **শাস্তি (BN):** "বাদী নেক্সট আড্ডায় সবার প্রথম চা বানাবে।"

**R07 — Kharap Acting, Valo Mon | খারাপ অ্যাক্টিং, ভালো মন** · Tag: **Nirdosh**
- **Ray (BL):** "Nirdosh! Tumi eto kharap acting korecho je... asolei nirdosh. Acting kharap, kintu mon valo — ei dui-ta alada jinish."
- **রায় (BN):** "নির্দোষ! তুমি এত খারাপ অ্যাক্টিং করেছ যে... আসলেই নির্দোষ। অ্যাক্টিং খারাপ, কিন্তু মন ভালো — এই দুইটা আলাদা জিনিস।"
- **Shasti (BL):** "Badi-ke next dawat-e biryani-r last piece dite hobe."
- **শাস্তি (BN):** "বাদীকে নেক্সট দাওয়াতে বিরিয়ানির লাস্ট পিস দিতে হবে।"

**R08 — Itihash Rochito | ইতিহাস রচিত** · Tag: **Nirdosh**
- **Ray (BL):** "Nirdosh! Ei court-er itihash-e eto kharap deny-er por-o nirdosh howa prothom. Itihash rochito holo — keu hat-tali dao."
- **রায় (BN):** "নির্দোষ! এই কোর্টের ইতিহাসে এত খারাপ ডিনাইয়ের পরও নির্দোষ হওয়া প্রথম। ইতিহাস রচিত হলো — কেউ হাততালি দাও।"
- **Shasti (BL):** "Badi-ke agami 3 adda-e shomoy moto ash-te hobe — '5 minute' bole na."
- **শাস্তি (BN):** "বাদীকে আগামী ৩ আড্ডায় সময় মতো আসতে হবে — '৫ মিনিট' বলে না।"

**R09 — Kopal Kharap | কপাল খারাপ** · Tag: **Nirdosh**
- **Ray (BL):** "Nirdosh! Sakkhi-r ek kothay case ulte gelo. Badi, tomar kopal kharap — ar sakkhi-r acting valo. Dui-tai sotti."
- **রায় (BN):** "নির্দোষ! সাক্ষীর এক কথায় কেস উল্টে গেল। বাদী, তোমার কপাল খারাপ — আর সাক্ষীর অ্যাক্টিং ভালো। দুইটাই সত্যি।"
- **Shasti (BL):** "Badi shobaike cha kheyalbe."
- **শাস্তি (BN):** "বাদী সবাইকে চা খাওয়াবে।"

**R10 — Para-r Adalat Bhule Na | পাড়ার আদালত ভোলে না** · Tag: **Nirdosh**
- **Ray (BL):** "Nirdosh! Kintu mone rekho — para-r adalat kichu bhule na. Aaj beche gele, kalke abar mamla hote pare. Eti sotorkobani."
- **রায় (BN):** "নির্দোষ! কিন্তু মনে রেখো — পাড়ার আদালত কিছু ভোলে না। আজ বেঁচে গেলে, কালকে আবার মামলা হতে পারে। এটি সতর্কবাণী।"
- **Shasti (BL):** "Nirdosh ekhon 10 second-er ekta nach debe — khushi-te."
- **শাস্তি (BN):** "নির্দোষ এখন ১০ সেকেন্ডের একটা নাচ দেবে — খুশিতে।"

**R11 — Dui Pokkhei Kharap | দুই পক্ষই খারাপ** · Tag: **Any**
- **Ray (BL):** "Adalat aj khushi! Dui pokkhei eto kharap argument koreche je ray dewa kothin. Thik ache, ray hoye gelo — kintu keu-i impressed na."
- **রায় (BN):** "আদালত আজ খুশি! দুই পক্ষই এত খারাপ আর্গুমেন্ট করেছে যে রায় দেওয়া কঠিন। ঠিক আছে, রায় হয়ে গেল — কিন্তু কেউই ইমপ্রেসড না।"
- **Shasti (BL):** "Dui jon-e mile cha-er bill debe."
- **শাস্তি (BN):** "দুই জনে মিলে চায়ের বিল দেবে।"

**R12 — Thanda Cha | ঠান্ডা চা** · Tag: **Any**
- **Ray (BL):** "Ei mamla eto complicated je judge-er cha thanda hoye gelo. Thanda cha-er shokti ononto — ar ei ray-o thanda, kintu sotti."
- **রায় (BN):** "এই মামলা এত কমপ্লিকেটেড যে জাজের চা ঠান্ডা হয়ে গেল। ঠান্ডা চায়ের শক্তি অনন্ত — আর এই রায়ও ঠান্ডা, কিন্তু সত্যি।"
- **Shasti (BL):** "Je harbe, shey judge-ke notun cha debe."
- **শাস্তি (BN):** "যে হারবে, সে জাজকে নতুন চা দেবে।"

**R13 — Age Haso | আগে হাসো** · Tag: **Any**
- **Ray (BL):** "Ray hocche... kintu tar age sobai ekbar haso. Hasa sesh? Thik ache, ekhon ray. Ei adalat-e hasa ain-er part."
- **রায় (BN):** "রায় হচ্ছে... কিন্তু তার আগে সবাই একবার হাসো। হাসা শেষ? ঠিক আছে, এখন রায়। এই আদালতে হাসা আইনের পার্ট।"
- **Shasti (BL):** "Je shobcheye beshi hasbe, shey next adda-er location thik korbe."
- **শাস্তি (BN):** "যে সবচেয়ে বেশি হাসবে, সে নেক্সট আড্ডার লোকেশন ঠিক করবে।"

**R14 — Mamu Labhban | মামু লাভবান** · Tag: **Any**
- **Ray (BL):** "Tomra dui jon-i bondhu. Ar bondhu-der jhogra-e sudhu ekjon labhban — cha-er dokaner Mamu. Mamu aj onek cha bikri koreche."
- **রায় (BN):** "তোমরা দুই জনই বন্ধু। আর বন্ধুদের ঝগড়ায় শুধু একজন লাভবান — চায়ের দোকানের মামু। মামু আজ অনেক চা বিক্রি করেছে।"
- **Shasti (BL):** "Dui jon mile ekta group selfie tulbe — hasi mukhe."
- **শাস্তি (BN):** "দুই জন মিলে একটা গ্রুপ সেলফি তুলবে — হাসি মুখে।"

**R15 — Kotha Sesh Na | কথা শেষ না** · Tag: **Any**
- **Ray (BL):** "Mamla sesh, kintu kotha sesh na. Ei ray appeal kora jabe na — karon cha-er bill already dewa hoye geche. Adalat uthlo, adda shuru."
- **রায় (BN):** "মামলা শেষ, কিন্তু কথা শেষ না। এই রায় আপিল করা যাবে না — কারণ চায়ের বিল অলরেডি দেওয়া হয়ে গেছে। আদালত উঠল, আড্ডা শুরু।"
- **Shasti (BL):** "Shobai mile bolo: 'Adalat cholche!' — tarpor adda shuru."
- **শাস্তি (BN):** "সবাই মিলে বলো: 'আদালত চলছে!' — তারপর আড্ডা শুরু।"

---

### 10.4 Objection Lines (6)

Full dual-language text (mechanics in §5). Shown as a 6-button menu; each button displays the line in the player's active language.

| ID | Banglish | Bangla | When to fire (flavor hint, shown as tooltip) |
|---|---|---|---|
| O1 | Golpo ghuracchen! | গল্প ঘোরাচ্ছেন! | They're twisting the story / ঘুরিয়ে কথা বলছে |
| O2 | Mithya kotha! | মিথ্যা কথা! | Blatant lie / সরাসরি মিথ্যা |
| O3 | Amar kotha sesh hoyni! | আমার কথা শেষ হয়নি! | They interrupted you / কথা কেড়ে নিয়েছে |
| O4 | Eita evidence na! | এটা এভিডেন্স না! | Fake proof / ভুয়া প্রমাণ |
| O5 | Ostad, uni haschen! | ওস্তাদ, উনি হাসছেন! | They're laughing at the court / আদালতে হাসছে |
| O6 | Time waste kortechen! | টাইম ওয়েস্ট করছেন! | Stalling / সময় নষ্ট করছে |

### 10.5 UI String Table (complete, dual-language)

Convention: keys are `snake_case`; every value is `{bl, bn}`. Language names ("Banglish", "বাংলা") are never translated. Emojis are shared across languages. This table is the full launch string set — implementation must not invent new player-facing strings outside it.

**Global / shell**
| Key | Banglish | Bangla |
|---|---|---|
| app_title | Adalat Cholche! | আদালত চলছে! |
| app_subtitle | Kotha Katakati | কথা কাটাকাটি |
| tagline | Bondhu-der jhogra, ekhon adalat-e! | বন্ধুদের ঝগড়া, এখন আদালতে! |
| btn_back | Pichone | পিছনে |
| btn_close | Bondho | বন্ধ |
| btn_ok | Thik ache | ঠিক আছে |
| loading | Adalat sajano hocche... | আদালত সাজানো হচ্ছে... |
| waiting | Opekkha korchi... | অপেক্ষা করছি... |
| connection_lost | Connection geche! Adalat thame ache. | কানেকশন গেছে! আদালত থেমে আছে। |
| reconnecting | Abar connect korchi... | আবার কানেক্ট করছি... |
| reconnected | Abar eshechi! | আবার এসেছি! |
| paused_title | Adalat Sthagit | আদালত স্থগিত |
| paused_body | Player-er opekkhay adalat theme ache. | প্লেয়ারের অপেক্ষায় আদালত থেমে আছে। |
| resumed | Adalat abar cholche! | আদালত আবার চলছে! |
| lang_label | Bhasha | ভাষা |

**Splash / create / join**
| Key | Banglish | Bangla |
|---|---|---|
| splash_create | Notun Adalat Khulo | নতুন আদালত খোলো |
| splash_join | Adalat-e Dhuko | আদালতে ঢুকো |
| name_label | Tomar naam | তোমার নাম |
| name_placeholder | Jemon: Rafi | যেমন: রাফি |
| create_title | Notun Adalat | নতুন আদালত |
| join_title | Adalat-e Dhuko | আদালতে ঢুকো |
| code_label | Room Code (4 okkhor) | রুম কোড (৪ অক্ষর) |
| code_placeholder | ABCD | ABCD |
| btn_create | Adalat Khulo | আদালত খোলো |
| btn_join | Dhuko | ঢুকো |
| btn_share | Bondhu-der Dako (link copy) | বন্ধুদের ডাকো (লিংক কপি) |
| share_copied | Link copy hoyeche! | লিংক কপি হয়েছে! |
| room_code_label | Ei code-ta bondhu-der dao | এই কোডটা বন্ধুদের দাও |
| err_code_invalid | Ei code-e kono adalat nei! | এই কোডে কোনো আদালত নেই! |
| err_room_full | Adalat full hoye geche! (6 jon max) | আদালত ফুল হয়ে গেছে! (৬ জন ম্যাক্স) |
| err_room_closed | Ei adalat bondho hoye geche. | এই আদালত বন্ধ হয়ে গেছে। |
| err_name_empty | Naam likho age! | নাম লেখো আগে! |
| err_same_device | Ek device, ek seat! Onno tab-e already boshe acho. | এক ডিভাইস, এক সিট! অন্য ট্যাবে আগেই বসে আছো। |

**Lobby**
| Key | Banglish | Bangla |
|---|---|---|
| lobby_title | Lobby — Adalat Sajano | লবি — আদালত সাজানো |
| lobby_players | Player ({n}/6) | প্লেয়ার ({n}/৬) |
| lobby_seat | Seat {n} | সিট {n} |
| lobby_host | Host | হোস্ট |
| btn_ready | Ami Prostut | আমি প্রস্তুত |
| lobby_ready | Prostut | প্রস্তুত |
| lobby_not_ready | Ekhono prostut na | এখনো প্রস্তুত না |
| btn_start | Adalat Shuru! | আদালত শুরু! |
| start_need_two | Minimum 2 jon lagbe! | মিনিমাম ২ জন লাগবে! |
| lobby_schedule | Ebar ke ki — 6 round-er schedule | এবার কে কী — ৬ রাউন্ডের শিডিউল |
| lobby_leave | Ber hoye jao | বের হয়ে যাও |
| crown_label | Tomar crown | তোমার ক্রাউন |
| crown_raja | Raja 👑 | রাজা 👑 |
| crown_rani | Rani 👑 | রানি 👑 |
| rules_title | Kivabe Khelbe | কীভাবে খেলবে |
| rules_body | Prot round-e ekta mamla. Badi 90s argument korbe, Bibadi 90s reply korbe. Sakkhi sakkhi debe, objection hobe, tarpor vote — doshi na nirdosh. Ray-er por Jhogra Meter score. 6 round sesh-e je shobcheye beshi score pabe, shei Adalat-er Raja! | প্রতি রাউন্ডে একটা মামলা। বাদী ৯০ সেকেন্ড আর্গুমেন্ট করবে, বিবাদী ৯০ সেকেন্ড রিপ্লাই করবে। সাক্ষী সাক্ষ্য দেবে, অবজেকশন হবে, তারপর ভোট — দোষী না নির্দোষ। রায়ের পর ঝগড়া মিটার স্কোর। ৬ রাউন্ড শেষে যে সবচেয়ে বেশি স্কোর পাবে, সেই আদালতের রাজা! |
| rules_voice_note | Kotha bolo nije-der moddhe — same room-e ba nijer voice call-e. App kono audio record kore na. | কথা বলো নিজেদের মধ্যে — একই রুমে বা নিজের ভয়েস কলে। অ্যাপ কোনো অডিও রেকর্ড করে না। |

**Roles**
| Key | Banglish | Bangla |
|---|---|---|
| role_badi | Badi | বাদী |
| role_bibadi | Bibadi | বিবাদী |
| role_bicharok | Bicharok | বিচারক |
| role_sakkhi | Sakkhi | সাক্ষী |
| role_auto_bicharok | Auto-Bicharok | অটো-বিচারক |
| role_gavel | Gavel (tie-break) | গ্যাভেল (টাই-ব্রেক) |
| role_deck_sakkhi | Sakkhi Card | সাক্ষী কার্ড |

**Mamla reveal**
| Key | Banglish | Bangla |
|---|---|---|
| phase_reveal | Mamla Prostut | মামলা প্রস্তুত |
| mamla_label | Mamla #{id} | মামলা #{id} |
| btn_prostut | Prostut! | প্রস্তুত! |
| reveal_ready_count | {x}/{n} prostut | {x}/{n} প্রস্তুত |
| badi_prompt_label | Badi-r shuru | বাদীর শুরু |
| bibadi_prompt_label | Bibadi-r shuru | বিবাদীর শুরু |

**Argument / Sakkhi stand**
| Key | Banglish | Bangla |
|---|---|---|
| phase_badi | Badi-r Argument | বাদীর আর্গুমেন্ট |
| phase_bibadi | Bibadi-r Reply | বিবাদীর রিপ্লাই |
| btn_sesh | Sesh! | শেষ! |
| timer_label | Shomoy | সময় |
| objection_btn | Objection! | অবজেকশন! |
| objection_tokens | Token: {n}/2 | টোকেন: {n}/২ |
| freeze_banner | ADALAT THAMLO! (5s) | আদালত থামল! (৫ সে) |
| heat_btn | Jhogra! 🔥 | ঝগড়া! 🔥 |
| heat_cap | Tap sesh! (max 3 per speech) | ট্যাপ শেষ! (ম্যাক্স ৩ প্রতি স্পিচ) |
| speaker_hint | Jore bolo! Shobai shunchhe. | জোরে বলো! সবাই শুনছে। |
| audience_hint | Jhogra button diye heat dao! | ঝগড়া বাটন দিয়ে হিট দাও! |
| phase_sakkhi | Sakkhi Stand | সাক্ষী স্ট্যান্ড |
| sakkhi_testimony | Sakkhi-r Sakkhi | সাক্ষীর সাক্ষ্য |
| sakkhi_lean_secret | Tomar lean gopon — acting koro! | তোমার লিন গোপন — অ্যাক্টিং করো! |
| sakkhi_q_badi | Badi-r Proshno | বাদীর প্রশ্ন |
| sakkhi_q_bibadi | Bibadi-r Proshno | বিবাদীর প্রশ্ন |
| sakkhi_answer | Sakkhi-r Uttor | সাক্ষীর উত্তর |
| sakkhi_pressure | Sakkhi-r Chaper Proshno | সাক্ষীর চাপের প্রশ্ন |
| sakkhi_auto_read | Sakkhi card pora hocche... | সাক্ষী কার্ড পড়া হচ্ছে... |
| lean_badi | Ei sakkhi Badi-r dike jhuke | এই সাক্ষী বাদীর দিকে ঝুঁকে |
| lean_bibadi | Ei sakkhi Bibadi-r dike jhuke | এই সাক্ষী বিবাদীর দিকে ঝুঁকে |
| lean_neutral | Ei sakkhi neutral — nijer iccha | এই সাক্ষী নিউট্রাল — নিজের ইচ্ছা |

**Objections / voting**
| Key | Banglish | Bangla |
|---|---|---|
| phase_kothakatakati | Kotha Katakati! | কথা কাটাকাটি! |
| ruling_panel | Bicharok-er Ruling | বিচারকের রুলিং |
| ruling_grohon | Grohon ✓ | গ্রহণ ✓ |
| ruling_khariz | Khariz ✗ | খারিজ ✗ |
| ruling_timeout | Ruling hoyni — auto-stand | রুলিং হয়নি — অটো-স্ট্যান্ড |
| objection_feed_empty | Ekhono kono objection hoyni... shobai shanto? | এখনো কোনো অবজেকশন হয়নি... সবাই শান্ত? |
| phase_vote | Vote — Doshi na Nirdosh? | ভোট — দোষী না নির্দোষ? |
| vote_doshi | DOSHI ✗ | দোষী ✗ |
| vote_nirdosh | NIRDOSH ✓ | নির্দোষ ✓ |
| vote_cast | Vote dewa hoyeche! | ভোট দেওয়া হয়েছে! |
| vote_waiting_party | Tomar ray-er opekkha... | তোমার রায়ের অপেক্ষা... |
| vote_reveal | Vote Khulche! | ভোট খুলছে! |
| vote_tie_judge | Tie! Bicharok-er ray-i sesh kotha! | টাই! বিচারকের রায়ই শেষ কথা! |
| vote_tie_2p | Tie! Ekhon Sesh Kotha! | টাই! এখন শেষ কথা! |
| seshkotha_badi | Badi-r Sesh Kotha (20s) | বাদীর শেষ কথা (২০ সে) |
| seshkotha_bibadi | Bibadi-r Sesh Kotha (20s) | বিবাদীর শেষ কথা (২০ সে) |
| revote_label | Abar Vote! | আবার ভোট! |
| gavel_decides | Gavel-er malik thik korlo! | গ্যাভেলের মালিক ঠিক করল! |
| vote_hung | Ray atke geche! Keu-i vote dey ni. | রায় আটকে গেছে! কেউই ভোট দেয়নি। |
| vote_abstain | Abstain (vote dey ni) | অ্যাবস্টেইন (ভোট দেয়নি) |

**Ray / Jhogra Meter**
| Key | Banglish | Bangla |
|---|---|---|
| phase_ray | RAY! | রায়! |
| verdict_doshi | DOSHI! | দোষী! |
| verdict_nirdosh | NIRDOSH! | নির্দোষ! |
| ray_label | Bicharok-er Ray | বিচারকের রায় |
| shasti_label | Shasti | শাস্তি |
| btn_ray_sesh | Ray Shesh | রায় শেষ |
| ray_read_hint | Bicharok, jore poro! | বিচারক, জোরে পড়ো! |
| lean_reveal_label | Sakkhi-r gopon lean | সাক্ষীর গোপন লিন |
| phase_meter | Jhogra Meter | ঝগড়া মিটার |
| tier_shanto | Shanto Adda 😴 | শান্ত আড্ডা 😴 |
| tier_kotha | Kotha Katakati 😤 | কথা কাটাকাটি 😤 |
| tier_raga | Ragaragiri 🔥 | রাগারাগি 🔥 |
| tier_uchol | Adalat Uchol! 🌋 | আদালত উচ্ছল! 🌋 |
| chaos_bonus | Chaos Bonus! | ক্যাওস বোনাস! |
| pts_verdict_win | Round jit | রাউন্ড জিত |
| pts_verdict_loss | Jhogra allowance | ঝগড়া অ্যালাউন্স |
| pts_obj_sustained | Objection grohon | অবজেকশন গ্রহণ |
| pts_obj_overruled | Objection khariz | অবজেকশন খারিজ |
| pts_obj_stand | Objection stand | অবজেকশন স্ট্যান্ড |
| pts_heat | Jhogra heat | ঝগড়া হিট |
| pts_lean | Sakkhi lean | সাক্ষী লিন |
| pts_chaos | Chaos bonus | ক্যাওস বোনাস |
| pts_allowance_judge | Bicharok allowance | বিচারক অ্যালাউন্স |
| pts_ruling | Ruling | রুলিং |
| pts_allowance_sakkhi | Sakkhi allowance | সাক্ষী অ্যালাউন্স |
| pts_star | Star Witness! | স্টার উইটনেস! |
| next_round_banner | Poroborti round: {badi} vs {bibadi} | পরবর্তী রাউন্ড: {badi} বনাম {bibadi} |
| btn_next_round | Poroborti Round | পরবর্তী রাউন্ড |
| round_label | Round {n}/6 | রাউন্ড {n}/৬ |

**Final scoreboard**
| Key | Banglish | Bangla |
|---|---|---|
| final_title | Adalat-er Ray Sesh! | আদালতের রায় শেষ! |
| champion_line | Adalat-er {crown} — Moha Jhograbaj! | আদালতের {crown} — মহা ঝগড়াবাজ! |
| shared_crown | Jugalbondi Champion! Dui jon-i raja! | যুগলবন্দি চ্যাম্পিয়ন! দুই জনই রাজা! |
| award_bagh | Adalat-er Bagh 🐯 | আদালতের বাঘ 🐯 |
| award_obj | Objection-er Raja ⚡ | অবজেকশনের রাজা ⚡ |
| award_magnet | Jhogra Magnet 🧲 | ঝগড়া ম্যাগনেট 🧲 |
| award_sakkhi | Sakkhi-er Sera 🎭 | সাক্ষীর সেরা 🎭 |
| award_khariz | Khariz King 😅 | খারিজ কিং 😅 |
| award_shanti | Shanti Dut 🕊️ | শান্তি দূত 🕊️ |
| stats_wins | Round jit | রাউন্ড জিত |
| stats_obj | Objection grohon | অবজেকশন গ্রহণ |
| stats_heat | Heat peyechi | হিট পেয়েছি |
| stats_star | Star Witness | স্টার উইটনেস |
| btn_rematch | Abar Khelo! | আবার খেলো! |
| btn_new_room | Notun Adalat | নতুন আদালত |
| btn_exit | Biday | বিদায় |

**Edge-case messages**
| Key | Banglish | Bangla |
|---|---|---|
| player_disconnected | {name} disconnect hoyeche! Opekkha korchi... | {name} ডিসকানেক্ট হয়েছে! অপেক্ষা করছি... |
| player_returned | {name} phire esheche! | {name} ফিরে এসেছে! |
| forfeit_banner | {name} phire asheni — round forfeit! | {name} ফিরে আসেনি — রাউন্ড ফরফিট! |
| judge_left_promote | Bicharok chole geche! {name} ekhon notun Bicharok. | বিচারক চলে গেছে! {name} এখন নতুন বিচারক। |
| sakkhi_left_deposition | Sakkhi chole geche — tar card deposition hishebe pora hocche. | সাক্ষী চলে গেছে — তার কার্ড ডিপোজিশন হিসেবে পড়া হচ্ছে। |
| host_migrated | Notun host: {name} | নতুন হোস্ট: {name} |
| room_paused_waiting | Player kom! 2 jon-er opekkhay adalat theme ache. | প্লেয়ার কম! ২ জনের অপেক্ষায় আদালত থেমে আছে। |
| room_closed_solo | Keu-i nei. Adalat bondho — notun code nao. | কেউই নেই। আদালত বন্ধ — নতুন কোড নাও। |
| afk_warning | {name}, bolo! {n}s-er moddhe na bolle forfeit. | {name}, বলো! {n} সেকেন্ডের মধ্যে না বললে ফরফিট। |
| gavel_passed | Gavel ekhon {name}-er kache | গ্যাভেল এখন {name}-এর কাছে |

### 10.6 Content file schemas (static JSON, no code)

| File | Fields per record | Notes |
|---|---|---|
| `data/mamla.json` | `id` (M01–M30), `emoji`, `title{bl,bn}`, `case{bl,bn}`, `badi{bl,bn}`, `bibadi{bl,bn}` | 30 records; prompts are first-person starters |
| `data/sakkhi.json` | `id` (S01–S20), `emoji`, `name{bl,bn}`, `persona{bl,bn}`, `lean` (`badi`\|`bibadi`\|`neutral`), `testimony{bl,bn}`, `pressure{bl,bn}` | 20 records; lean hidden from non-witnesses at 3+ |
| `data/ray.json` | `id` (R01–R15), `tag` (`doshi`\|`nirdosh`\|`any`), `title{bl,bn}`, `ray{bl,bn}`, `shasti{bl,bn}` | 15 records; tag pools drawn without replacement |
| `data/objections.json` | `id` (O1–O6), `text{bl,bn}`, `hint{bl,bn}` | 6 records |
| `data/ui.json` | `key` → `{bl,bn}` | Full §10.5 table |
| `data/awards.json` | `key` → `{bl,bn}` | Superlatives + champion lines (§9.2) |

Rules: UTF-8; stable IDs forever; every player-facing text field is a `{bl,bn}` pair — a build-time lint check rejects any string missing either language; no runtime text generation of any kind.

---

## 11. UI / UX — Screen List & Visual Identity

### 11.1 Visual identity
- **Theme:** "Para-r Adalat" — a neighborhood court run out of a tea stall. Manila case-file folders, rubber stamps, a wooden gavel, chai stains. Dramatic but silly.
- **Palette:** Maroon `#7B2D26` (bench/headers) · Mustard `#E8A33D` (accents, gavel) · Cream `#FAF3E3` (paper background) · Ink `#2B2118` (text) · Verdict red `#C0392B` + ✗ / verdict green `#2E7D4F` + ✓ (always icon-paired, color-blind safe).
- **Typography:** Bangla — *Hind Siliguri* (Google Fonts, OFL, free) with *Noto Sans Bengali* fallback; Banglish — system sans stack. Minimum body 16px, timers 48px+, buttons ≥ 48px touch targets. Mobile-first (360px up), tablet/desktop centered column.
- **Motion:** card flip on Mamla reveal, gavel bang + stamp on Ray, meter fill on scoring, vote chips flying in. All animations respect `prefers-reduced-motion` (instant states instead).
- **Sound:** WebAudio-synthesized gavel bang, drumroll, stamp thud — generated in code, **zero audio files, zero licenses**. Mute toggle in header.
- **Persistent header:** room code · round n/6 · language toggle (Banglish | বাংলা) · mute · connection dot.

### 11.2 Screen inventory (13 screens + overlays)

| # | Screen | Purpose | Key elements | Entry → Exit |
|---|---|---|---|---|
| 1 | **Splash** | First launch; language choice; create/join | Title + tagline, language toggle, name field, two big buttons, "Kivabe Khelbe" rules modal | App open → Create (2) or Join (3) |
| 2 | **Create Room** | Host opens a court | Generated 4-letter code (huge), share/copy-link button (`?room=CODE` deep link), player list filling live, Start button (enabled ≥2) | → Lobby (4) |
| 3 | **Join Room** | Enter code + name | 4 letter-boxes (auto-advance, paste support), name field, error states (invalid/full/closed) | → Lobby (4) |
| 4 | **Lobby** | Gather 2–6, ready up, preview fairness | Seat list with connection dots + host badge, ready checks, crown flavor picker (Raja/Rani), 6-round role schedule preview (§8), rules modal, leave | Host Start → Mamla Reveal (5) |
| 5 | **Mamla Reveal** | The case lands | Card flip animation, category emoji, title + case text, role-highlighted prompts (Badi sees Badi starter emphasized; Bibadi likewise), 20s ring, Prostut buttons + ready count | All ready / timeout → Argument (6) |
| 6 | **Argument** (Badi, then Bibadi) | The 90s speeches | Giant countdown ring, speaker name + role badge, prompt card, Sesh button (speaker only), Objection button + token pips (opponent), Jhogra! heat button (audience, 3-tap cap), freeze banner, emote bar | Timer/Sesh → next speech → Sakkhi Stand (7) |
| 7 | **Sakkhi Stand** | Testimony + cross-exam | Witness persona card (secret lean badge visible only to witness at 3+), testimony timer, Q&A sub-phase indicator (who asks/answers), pressure-question display (2P), auto-read mode (2P/deposition) | → Kotha Katakati (8) |
| 8 | **Kotha Katakati** | Dedicated 30s objection window | Both sides' token counts, 6-objection menu, judge ruling panel (Grohon/Khariz + 5s ring), objection feed, 30s timer | → Vote (9) |
| 9 | **Vote** | Secret ballot | Doshi/Nirdosh giant buttons (voters), party waiting screen with round stats, 25s ring, reveal animation; **Sesh Kotha sub-screen** (2P tie): 20s+20s final words → re-vote → gavel result | → Ray (10) |
| 10 | **Ray** | Verdict + comedy sentence | Gavel bang, DOSHI!/NIRDOSH! stamp, Ray line (judge reads), Shasti card, Sakkhi lean reveal, hung-court variant | → Jhogra Meter (11) |
| 11 | **Jhogra Meter** | Round scoring + rotation | 0–100 gauge fill with tier label, per-player point breakdown rows (auditable, §7), running totals, next-round role banner, 10s auto-next + host Next button | → next round's Mamla Reveal (5) |
| 12 | **Final Scoreboard** | Championship | Podium (top 3), champion crown + title line, superlative awards dealt one-by-one, full stats table, Rematch / Notun Adalat / Biday | Rematch → Lobby (4) with reset scores |
| 13 | **Rules Modal** (overlay screen) | How to play, any time | Role table, loop summary, scoring summary, voice-out-of-band note, language toggle | Overlay on any screen |

**Global overlays:** connection-lost banner + reconnecting spinner · pause screen ("Adalat Sthagit") when <2 connected · rejoin toast · host-migration toast · AFK warning countdown · room-closed screen.

### 11.3 Per-role screen logic (Argument screen example)
- **Badi (speaking):** timer + prompt + Sesh. No objection button (can't object to yourself), no heat button.
- **Bibadi (opponent):** timer + Objection button (2 pips) + emote bar. No Sesh, no heat.
- **Bicharok / Sakkhi (audience):** timer + Jhogra! button (3-tap cap per speech) + emote bar; judge additionally gets the ruling panel whenever an objection fires (5s decision ring).
- All views show the same live objection banner and freeze state — the court experiences the interruption together.

### 11.4 Emote bar
Six static reactions anyone can send anytime: 😂 🔥 👏 😱 🤔 💤. Rate-limited to 1 per 2s per player, purely visual (float across the screen), **zero scoring effect**. Keeps non-speakers expressive without chat moderation needs.

### 11.5 Accessibility & comfort
- Icon + color for every state (verdicts, ready, connection); no color-only signals.
- Bangla font with generous line-height (1.6); Banglish never auto-corrected.
- Reduced-motion mode; mute toggle; timers also shown as numeric text (not only rings).
- One-hand mobile use: all primary actions in bottom 40% of screen.
- No flashing >3Hz; gavel animation is a single thud.

---

## 12. Technical Architecture (design only — no code)

### 12.1 Stack & constraints
| Layer | Choice | Why |
|---|---|---|
| Frontend | Plain HTML + CSS + vanilla JS (ES modules) | No build step, no framework, runs from GitHub Pages as-is |
| State sync | Firebase Realtime Database (Spark/free) | Free realtime pub/sub + built-in presence (`onDisconnect`) + server timestamps; no server logic needed |
| Identity | Firebase Anonymous Auth (free on Spark) | Per-device persistent uid, no accounts, no PII |
| Content | Static JSON in `data/` (§10.6) | No AI, no APIs, no keys, no paid services — humor is hand-written |
| Hosting | GitHub Pages (HTTPS) | Free, global CDN, Firebase-compatible |
| Audio | WebAudio synthesis | Zero asset files/licenses |
| Fonts | Google Fonts (Hind Siliguri, OFL) | Free Bangla rendering |

**The Firebase `apiKey` clarification (restated):** the web config's `apiKey` is a public project identifier shipped to every Firebase web app — it is not a secret, not a paid API key, and grants nothing by itself. Access control lives entirely in RTDB security rules (§12.6). No other keys, tokens, or third-party services exist in this project.

### 12.2 Repository layout (described)
```
index.html            — single page; screens are sections toggled by JS
styles.css            — theme, screens, animations, reduced-motion
js/app.js             — boot, screen router, language manager
js/net.js             — Firebase init, anonymous auth, room join/create, presence
js/state.js           — room state mirror, subscriptions, resync
js/driver.js          — host phase-driver + lease failover + deterministic draws
js/screens/*.js       — one module per screen (§11.2)
js/scoring.js         — pure functions implementing §7 (unit-testable)
js/i18n.js            — {bl,bn} lookup, toggle, localStorage persistence
data/*.json           — §10.6 content pack
```

### 12.3 Room lifecycle & limits
- **Create:** client generates 4-letter code (alphabet A–Z minus I/O), checks existence via transaction, writes `meta`. Collision → regenerate (331,776 codes; collisions negligible).
- **Join:** deep link `?room=CODE` pre-fills; membership write adds `players/{uid}` with seat = join order.
- **TTL:** `meta.expiresAt = lastActivity + 2h`, refreshed on any write. Stale-room cleanup: the next *creating* client sweeps rooms past `expiresAt` (bounded read of `meta` only) — no cron, no server.
- **Spark tier budget (documented, monitored):** 100 simultaneous connections ≈ **16 full 6-player rooms** at once; 1 GB stored (rooms are ~5 KB each + TTL sweep → effectively unlimited); 10 GB/month download ≈ tens of thousands of matches (a full match transfers ~200–300 KB per client). Graceful "server busy" error if limits are hit; the game never silently corrupts state.
- **Privacy:** nicknames only; no chat text stored (objections are enum IDs, votes are enums); rooms auto-expire; codes are unguessable-enough for private friend groups and rules require membership anyway.

### 12.4 State model (RTDB tree, described)
```
rooms/{CODE}/
  meta/        hostUid, driverUid, leaseExpires, status(lobby|match|results|closed),
               createdAt, expiresAt
  players/{uid}/  name, seat, connected, lastSeen, crown, jm, stats{...}
  match/       roundIndex(0–5), schedule (computed §8), decks{mamla[6], sakkhi[≤18], rayOrder[15]}
  round/       phase, phaseEndsAt (server timestamp), pausedRemaining,
               mamlaId, roles{badi,bibadi,judge,sakkhi[]}, sakkhiCards{uid→cardId},
               heat{speech→{uid→count}}, objections[{by,type,at,ruling}],
               votes{uid→doshi|nirdosh}, verdict, rayId, rating, points{uid→breakdown}
```
- **Timers:** every phase writes `phaseEndsAt` using Firebase server timestamps; clients compute remaining time using `/.info/serverTimeOffset`. No client clock is trusted; backgrounded tabs resync on `visibilitychange`.
- **Votes/heat/objections:** each client writes only its own leaf (rules-enforced); the driver aggregates.
- **Point log:** every JM event is appended with type + amount; totals are recomputable from the log (reconnect-safe, audit-safe).

### 12.5 Host driver, randomness & failover
- The **host client** is the phase driver: when `phaseEndsAt` passes, it writes the next phase (idempotent transition guarded by a transaction on `round.phase`).
- **All randomness happens once, at match start:** the driver shuffles (Fisher–Yates) the Mamla deck (draw 6 of 30), Sakkhi deck (draw up to 18 of 20), and Ray order (15), storing the results in `match/decks`. Mid-match draws are just "next unused item" — so failover never changes content and replays are deterministic.
- **Lease failover:** `meta.driverUid` + `leaseExpires` (renewed every 5s by the driver). If the lease expires (host tab closed, network gone), any connected client may claim it via transaction (lowest seat wins ties). The new driver continues from `round.phase` — players see only a "Notun host: X" toast.
- **Host migration ≠ role change:** hosting is infrastructure; roles follow §8 schedules regardless.

### 12.6 Security rules (described, not code)
- All reads/writes require `auth != null` (anonymous auth).
- Room creation: any authenticated user may create a room whose `meta.hostUid == auth.uid`.
- Room reads: members only (`players/{auth.uid}` exists) — plus the creating client during collision check.
- `players/{uid}`: writable only by that uid (name/seat/crown), except `connected/lastSeen` which presence writes.
- `round/votes/{uid}`, `round/heat/.../{uid}`: self-only writes.
- `round.phase`, `match/*`, `round.verdict/rayId/points`: driver-only (`meta.driverUid == auth.uid`), with the lease-claim transaction as the only exception path.
- No field accepts free text from clients (all content is enum/ID references into static JSON) → injection and moderation surface ≈ zero.

### 12.7 Presence & disconnects
- `onDisconnect()` sets `connected=false` + `lastSeen` (RTDB built-in, fires on socket death — no heartbeat cost).
- Grace windows (§13) are computed from `lastSeen` by the driver; pauses write `pausedRemaining` so resumption is exact.
- Rejoin: anonymous auth persists in localStorage → same uid → same seat, scores intact, state resubscribed. A second tab on the same device is detected (same uid) and blocked with `err_same_device`.

### 12.8 Deployment & dev workflow
- GitHub Pages from `main` (or `/docs`); no build step; HTTPS automatic.
- Local dev: Firebase Emulator Suite (free) for RTDB + Auth; content lint script checks `{bl,bn}` parity, ID uniqueness, deck-size invariants (30/20/15/6), and lean distribution (7/7/6).
- Release checklist: run §15.3 test matrix on real devices before tagging.

### 12.9 Explicit non-goals (architecture-level)
No voice/video transport · no text chat · no accounts/profiles · no AI/LLM anywhere (content, moderation, or "smart" rulings) · no analytics SDKs · no payments · no server functions (Spark has none — the design needs none).

---

## 13. Edge Cases & Handling (complete)

| # | Situation | Handling |
|---|---|---|
| E1 | **Disputant disconnects mid-argument** | Presence fires → driver **pauses the phase clock** (`pausedRemaining` stored), banner "{name} disconnect hoyeche!". **60s grace**: return → clock resumes exactly where it froze. No return → **round forfeit**: opponent +10 JM, forfeiter 0, no vote/Ray contest (a neutral-tag Ray is still read for closure), support allowances skipped (§7.2), roles rotate normally. |
| E2 | **Disputant AFK at speech start** (connected but silent) | 30s "Start grace" with `afk_warning` countdown; no Sesh/start activity → forfeit as E1. (Connected-and-talking-but-quiet is *not* AFK — silence is a legal argument style; the court will judge it.) |
| E3 | **Judge leaves (4+)** | First Sakkhi (seat order) is **promoted to judge** immediately; their Sakkhi card becomes an **auto-deposition** (read aloud, lean revealed) so testimony isn't lost; remaining witnesses continue. If the judge drops during a ruling window → that objection auto-stands (+2). If no Sakkhi exists to promote (already left) → Auto-Bicharok mode for the rest of the round (objections auto-stand). If no non-party player remains at all, the round converts to 2P duel voting (both disputants vote; tie → Sesh Kotha → gavel by round parity). |
| E4 | **Sakkhi leaves mid-testimony** | Their card becomes an auto-deposition (20s read), phase continues. All witnesses gone → deck mode for the rest of the round. Role map recomputed next round (§8.6). |
| E5 | **2P: one player leaves** | Gavel passes to the remaining player; round pauses (E1 grace). If only 1 connected for **2 minutes** → room closes to `room_closed_solo` screen (create/join again). Rejoin within grace resumes everything. |
| E6 | **Vote tie** | §6.2 ladder: 4/6P judge breaks; 2P Sesh Kotha → re-vote → gavel; 3/5P structurally impossible; abstention-induced ties fall back to judge (5P) or hung court (3P). |
| E7 | **All voters abstain** | Hung court: no winner, both disputants +3, neutral Ray, round counts normally. |
| E8 | **Match JM tie** | §9.1 ladder: round wins → sustained objections → heat received → fewest overruled → shared crown. Never extends past 6 rounds. |
| E9 | **Host leaves / host tab dies** | Lease failover (§12.5): lowest-seat connected client claims driver; toast "Notun host: X". Roles/schedule unchanged. If host was also a disputant, E1 applies to their speech independently. |
| E10 | **Room full / bad code / expired room** | Join rejected with `err_room_full` / `err_code_invalid` / `err_room_closed` + create suggestion. No partial joins. |
| E11 | **Same device, two tabs** | Same anonymous uid detected → second tab blocked (`err_same_device`). One device, one seat. |
| E12 | **Player rejoins after forfeit** | Seat + total JM preserved; re-enters rotation next round (schedule recomputed if count changed). No score penalty beyond the forfeited round. |
| E13 | **Player count drops mid-match (≥2 remain)** | Match continues; future rounds use the recomputed schedule for the new count (§8.6); past scores untouched. Below 2 → pause screen (`room_paused_waiting`), 2-minute solo timeout → close (E5). |
| E14 | **Language toggle mid-round** | Instant re-render from `{bl,bn}` fields; per-device localStorage; zero state impact; room never forces a language. |
| E15 | **Objection / heat / ruling spam** | Hard caps: 2 tokens per disputant per round; 3 heat taps per tapper per speech, +5 JM cap per speech; judge ruling points capped +2/round. Emotes rate-limited 1/2s. |
| E16 | **Card exhaustion** | Impossible within a match: 6 Mamla ≤ 30; ≤18 Sakkhi ≤ 20; Ray tag pools of 10 ≥ 6 verdicts. Rematch reshuffles everything. Lint checks enforce deck invariants at build time. |
| E17 | **Timer drift / backgrounded tabs** | Server-timestamp `phaseEndsAt` + `serverTimeOffset`; `visibilitychange` resync; driver lease failover covers a stalled driver (any client can claim an expired lease and continue idempotently). |
| E18 | **Refresh mid-round** | Anonymous auth persists → auto-rejoin same seat; full state resubscribe; point log rebuilds identical totals. |
| E19 | **Firebase Spark limits hit** | Connection cap → join shows "server busy" retry message (never corrupts an existing room); download quota → same graceful error. Room TTL sweep keeps storage bounded. |
| E20 | **No WebAudio / old browser** | Sound silently disabled; visual gavel/stamp remain. ES-module-capable evergreen browsers are the supported floor; a static "browser too old" notice covers the rest. |
| E21 | **Judge rules on an objection after the 5s window** | Late taps ignored; auto-stand already applied (idempotent ruling write keyed by objection index). |
| E22 | **Two clients claim the driver lease simultaneously** | Transaction guarantees exactly one winner; loser re-subscribes as normal client. |
| E23 | **Early Sesh spam / instant forfeits** | Sesh is legal strategy (no penalty, no bonus). A disputant who Seshes at 0s simply gives the court nothing to vote for — the mechanic self-punishes. |
| E24 | **Shasti refusal** | Honor system by design; the app displays, never enforces. Social contract stated in the rules modal. |

---

## 14. Localization (i18n) Specification

- **Data model:** every player-facing string is `{bl, bn}` (§10.6). No concatenated sentences — full strings per language (word order differs); placeholders use `{name}`, `{n}`, `{id}` tokens identical in both languages.
- **Toggle:** persistent header control, per-device, stored in localStorage, applies instantly to all screens including live rounds. Default on first launch: **Banglish** (fastest to read for mixed groups), with a one-tap Bangla switch on the splash screen.
- **Numerals:** Bangla mode renders Bengali numerals (০–৯) for timers/scores via a display formatter; Banglish mode uses 0–9. Underlying state always stores ASCII numbers.
- **Content parity rule:** a card/Ray/objection/UI key missing either language fails the build lint. Translation drift is prevented by keeping both versions in the same record, authored together (§10).
- **Names:** player nicknames are never translated or transliterated.
- **Font loading:** Hind Siliguri loaded with `font-display: swap`; Bangla text never renders in a fallback that breaks conjuncts (Noto Sans Bengali fallback chain).

---

## 15. Production Plan

### 15.1 Milestones
| Milestone | Scope | Exit criteria |
|---|---|---|
| **M0 — Skeleton** | Pages deploy, anonymous auth, room create/join, lobby, presence, host driver + timers (2P only) | Two real devices complete a lobby→timer→lobby cycle; host kill fails over |
| **M1 — Core loop** | Full round loop at 4P with placeholder content; voting, Ray, scoring (§7 pure functions + unit tests) | 4 devices play 6 rounds; totals match hand-computed §7.4 example |
| **M2 — All counts + content** | 2/3/5/6P modes (deck Sakkhi, gavel, Sesh Kotha, multi-witness), full §10 content pack loaded, i18n toggle everywhere | Test matrix §15.3 green; content lint green |
| **M3 — Polish** | Animations, WebAudio, emotes, accessibility pass, edge-case hardening (§13), rules modal | Playtest with 6 friends: round length within §4.3 estimates, zero state desyncs |
| **M4 — Launch** | GitHub Pages public, README with house rules, feedback loop | A stranger group completes a match unassisted |

### 15.2 Content QA checklist
- [ ] 30 Mamla / 20 Sakkhi / 15 Ray / 6 Objections — counts exact, IDs unique, every field has `bl` + `bn`.
- [ ] Family-friendly scan: no politics, religion, romance, alcohol, real people, slurs; money ≤ ৳500.
- [ ] Read-aloud test: every case + prompt readable in ≤ 20s; testimony starters fit any Mamla.
- [ ] Lean distribution 7/7/6; Ray tags 5/5/5; deck-size invariants (§13 E16).
- [ ] Banglish↔Bangla meaning parity reviewed by a native speaker; humor survives both versions.

### 15.3 Device test matrix
2/3/4/5/6 real devices × {airplane mode mid-argument, host kill mid-round, judge leave, witness leave, refresh mid-vote, double-tab, slow 3G, language toggle mid-round, all-abstain vote, 2P double-tie → gavel, rematch}. Each run must end with identical point totals on every device (log-rebuild check).

### 15.4 Risks & mitigations
| Risk | Mitigation |
|---|---|
| Spark 100-connection cap at viral moment | Graceful busy message; rooms are tiny; document the cap; upgrade path exists but is *not* required for launch |
| Friends don't improvise (dead air) | Prompt starters on every card; 90s cap; Sesh button; emotes keep energy; rules modal coaches "exaggerate, don't insult" |
| Bangla font rendering on low-end devices | Hind Siliguri + Noto fallback, swap loading, tested on budget Android |
| Host tab sleeps (mobile OS) | Lease failover + visibility resync; driver duties are tiny writes |
| Verdict arguments (meta-arguments!) | That's the product. Objection tokens bound it; Ray is final; Shasti is honor-system |

---

## 16. Appendices

### 16.1 Glossary
| Term | Bangla | Meaning |
|---|---|---|
| Mamla | মামলা | Case / lawsuit |
| Badi | বাদী | Plaintiff / prosecutor |
| Bibadi | বিবাদী | Accused / defendant |
| Bicharok | বিচারক | Judge |
| Sakkhi | সাক্ষী | Witness |
| Ray | রায় | Verdict |
| Doshi / Nirdosh | দোষী / নির্দোষ | Guilty / not guilty |
| Shasti | শাস্তি | Sentence / punishment (comedy forfeit) |
| Jhogra | ঝগড়া | Argument / quarrel |
| Kotha Katakati | কথা কাটাকাটি | Bickering / back-and-forth |
| Adda | আড্ডা | Hangout / chat session |
| Para | পাড়া | Neighborhood |
| Dawat | দাওয়াত | Feast / invitation |
| Ostad / Janab | ওস্তাদ / জনাব | "Your Honor" (playful) |
| Grohon / Khariz | গ্রহণ / খারিজ | Sustain / overrule |
| Sesh Kotha | শেষ কথা | Final word |
| Gavel | গ্যাভেল | Judge's mallet; 2P tie-break power |

### 16.2 Out of scope (v1)
Voice/video · text chat · accounts · AI anything · payments · custom card editor · spectator mode · tournaments · moderation tooling (private rooms + enum-only writes make it unnecessary) · native apps.

### 16.3 Future ideas (explicitly not required, zero-cost only if ever built)
Community Mamla packs (more static JSON, same schema) · seasonal cards (Eid/picnic/biryani festivals — still family-friendly) · "Adalat League" local leaderboard in localStorage · printable Shasti cards · spectator emote-only seats. None of these change the core loop or the free architecture.

### 16.4 Design sign-off checklist (against the brief)
- [x] Multiplayer online, separate devices, 4-letter room code, 2–6 players (§3, §12)
- [x] No AI, no API keys, no paid services; all content static (§1.5, §10, §12)
- [x] Badi / Bibadi / Bicharok (auto-rotating duties in 2P) / Sakkhi (human 3+, deck 2P) (§3)
- [x] Round loop exactly: random Mamla → Badi 90s → Bibadi 90s → Sakkhi + questioning → Objections → doshi/nirdosh vote → funny Ray + Jhogra Meter; roles rotate every round (§2, §4, §8)
- [x] Every string Banglish + Bangla with in-game toggle (§10.5, §14)
- [x] Plain HTML/CSS/vanilla JS + Firebase RTDB Spark + GitHub Pages (§12)
- [x] Full role table (§3.1–3.2), turn timing (§4), Jhogra Meter formula (§7), 6-round win conditions (§9), edge cases incl. mid-argument disconnect / judge leaves / ties (§13), UI screen list incl. lobby, Mamla reveal, argument, Sakkhi stand, voting, Ray, scoreboard (§11)
- [x] Complete content pack: 30 Mamla + 20 Sakkhi + 15 Ray, each written twice, family-friendly Bengali friend-group humor (§10.1–10.3)

---

*End of document. Next step: implementation (M0) — intentionally not started here.*
