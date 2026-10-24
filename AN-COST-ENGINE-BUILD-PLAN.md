# AN Cost Engine: Combined Build Plan

**One math core, two entry points.** A public client calculator and a Deal side
automation that fires when Consultation outcome is known.

Prepared 2026-09-28. HubSpot portal 244862032 (region na2).
All property facts below were verified live against the portal unless marked otherwise.

---

## 1. What exists today

Three pieces, built separately, that do not currently talk to each other.

| Piece | Where | What it does |
|---|---|---|
| `client-cost-calculator.html` | Repo `marketing-adventurenannies/airtable.js`, branch `claude/client-cost-calculator-xN1FO` | Public page. Visitor enters rate, hours, ZIP, search type. HubSpot form gates the results. Computes employer cost: wages, employer taxes, benefits, agency fee. |
| `Engine.gs` | Apps Script, AN role account | Deterministic candidate compensation math. State overtime and double time rules, live-in exemptions, minimum wage floors, AN $35/hr floor. Produces the PCE compensation sentence. |
| `Runner.gs` | Apps Script, same project | Polls HubSpot Deals every 5 minutes, hashes comp inputs, recalculates only on real input change, batch writes results back. Also exposes a webhook for instant recalculation. |

---

## 2. The problem this build solves

**The two systems compute gross wages differently, and one of them is wrong.**

`client-cost-calculator.html` computes:

```
gross = rate x hours x 52
```

That ignores overtime completely. `Engine.gs` computes the same figure correctly:
straight time plus 1.5x overtime plus 2x double time, applying the right threshold
for the state, the live-in status, and (in California) the personal attendant
classification.

Worked example. A family in California, $45/hr, 50 hours a week, 10 hours a day,
5 days, live-out, personal attendant:

- Calculator today: `45 x 50 x 52` = **$117,000**
- Engine: 40 straight at $45 plus 10 OT at $67.50, weekly $2,475, annual = **$128,700**

A **$11,700 gap** on a single search. The family sees the low number on the website,
then a different number in their PCE. That is a credibility problem, and it gets worse
in daily-overtime states and at higher hour counts.

Everything below follows from fixing this once, in one place.

---

## 3. Target architecture

```
                   an-comp-core.js
                   (single source of truth, no I/O, deterministic)
                   
   Layer 1: WAGE MATH                Layer 2: EMPLOYER COST
   from Engine.gs                    from the calculator
   -------------------               ----------------------
   STATE_NAMES                       Federal tax constants
   MIN_WAGE                          State SUTA and workers comp table
   stateRule()                       Benefits valuation
   weeklyBuild()                     Agency fee tiers
   otSentence()                      employerCost()
   computeComp()                     
         |                                     |
         +------------------+------------------+
                            |
              +-------------+-------------+
              |                           |
   ENTRY POINT A                  ENTRY POINT B
   client-cost-calculator.html    CompCore.gs + Engine.gs + Runner.gs
   Browser, public, gated         Apps Script, Deal records
   Single rate                    Rate range (hr to compmax)
                                  Fires on Consultation outcome
```

**Non negotiable rule:** no math lives anywhere except `an-comp-core.js`. Both entry
points are thin adapters that marshal inputs in and format outputs out.

### Why a build step is needed

Apps Script concatenates all `.gs` files into one global scope, so the core must be
plain top level declarations: no `import`, no `export`, no module wrapper. The HubSpot
embed has to be a single self-contained file, so the core has to be inlined into the
HTML rather than loaded from a second URL.

Both constraints are satisfied by one source file plus a tiny inliner:

```
an-comp-core.js  ──┬──> paste as CompCore.gs in Apps Script
                   └──> node build.js inlines it into client-cost-calculator.html
```

`build.js` replaces everything between two marker comments in the HTML. Roughly 20 lines.
Without it the two copies drift, which is the exact failure this whole plan exists to prevent.

---

## 4. The shared input contract

This is what "same input fields for both" means concretely. Every row is one concept
with one canonical name across all three surfaces.

| Concept | Calculator field | Deal property | Contact property | Status |
|---|---|---|---|---|
| Hourly rate (low) | `f-rate` | `hr` | `anticipated_hourly_rate` | Deal and Contact exist |
| Hourly rate (high) | not collected, equals low | `compmax` | not needed | Deal exists |
| Hours per week | `f-hrs` | `guaranteed_hours_per_week` | `calc_hours_per_week` | Contact prop to create |
| Hours per day | **new input** | `working_hours_per_day` | `calc_hours_per_day` | Contact prop to create |
| Days per week | **new input** | `working_days_per_week` | `calc_days_per_week` | Contact prop to create |
| Live in or out | **new input** | `live_in_or_out` | `calc_live_in_or_out` | Contact prop to create |
| CA personal attendant | **new input**, only shown for CA | `ca_personal_attendant` | `calc_ca_personal_attendant` | Contact prop to create |
| State | derived from ZIP | `state` or `physical_address_state_region` | `state` | All exist |
| ZIP code | `f-zip` | not stored on Deal | `zip` | Contact exists |
| Search type | `f-hire` | `search_type_cleaned` | `search_type` | Both exist, values differ, see 4.1 |

**The calculator needs four new inputs** to reach parity: hours per day, days per week,
live in or out, and the California personal attendant question. Without hours per day
the engine cannot apply daily overtime, which is required in California and Colorado.

### 4.1 Search type value mapping

The two objects use different vocabularies for the same concept. Verified values:

**Deal `search_type_cleaned`:** `nanny`, `pe`, `ncs`, `temporary`, `short_term`,
`hybrid`, `retainer_search`, `test_record`, `recruitment`, `cobroke`, `an_open_call`,
`magnolia_agency`

**Contact `search_type`:** `Long Term Nanny`, `Travel / Temporary Nanny`,
`Short Term Nanny (90-365 days)`, `Private Educator`, `Newborn Care Specialist`,
`Rotational Nanny/Team`, `Household Manager`, `Family Assistant`,
`I'm not sure yet, I'd like to discuss with your team`

Required mapping table in the core:

| Calculator option | Writes to Contact `search_type` | Maps to Deal `search_type_cleaned` | Fee tier |
|---|---|---|---|
| Long-Term Nanny | `Long Term Nanny` | `nanny` | 25% |
| Short-Term Nanny (90-365 days) | `Short Term Nanny (90-365 days)` | `short_term` | 25% |
| Temporary or Travel Nanny | `Travel / Temporary Nanny` | `temporary` | 25% |
| Household Manager | `Household Manager` | `nanny` | 25% |
| Family Assistant | `Family Assistant` | `nanny` | 25% |
| Private Educator | `Private Educator` | `pe` | 30% |
| Newborn Care Specialist | `Newborn Care Specialist` | `ncs` | 30% |
| Rotational Nanny / Team | `Rotational Nanny/Team` | rota flag | 30% |

### 4.2 Known gap: NCS and ROTA

`Engine.gs` deliberately skips both. `HOURLY_TYPES` is
`["nanny", "pe", "hybrid", "temporary", "short_term", ""]`, and any Deal with the `rota`
flag or a `rota_pattern` value returns status `skipped`.

But the public calculator currently offers both Newborn Care Specialist and Rotational
Nanny / Team in its dropdown. So a visitor can get a confident number on the website for
a search type the Deal side refuses to price. That inconsistency has to be resolved.
See open decision D3.

---

## 5. Build phases

### Phase 0: Reconcile the math (blocking)

Confirm the client calculator adopts overtime aware gross. This changes quoted totals
for every schedule above 40 hours a week, always upward. Nothing else can proceed until
this is agreed, because every downstream number depends on it.

### Phase 1: Extract the shared core

Create `an-comp-core.js`. Plain top level declarations only.

**Moves in from `Engine.gs` unchanged:**
`STATE_NAMES`, `MIN_WAGE`, `stateRule()`, `weeklyBuild()`, `otSentence()`, `num()`,
`stateCode()`, `liveStatus()`, `bool()`, the formatting helpers, `computeComp()`,
`WEEKS_PER_YEAR`, `AN_FLOOR`

**Moves in from the calculator:**
Federal tax constants (`SS_BASE` 168600, `SS_RATE` .062, `MED_RATE` .0145,
`FUTA_RATE` .006, `FUTA_BASE` 7000), the per state SUTA and workers comp table,
the ZIP3 to state lookup, the fee tier table

**New, written for this build:**
`computeEmployerCost(compResult, options)`. Takes the output of `computeComp()` and
layers employer taxes, benefits, and agency fee on the overtime aware gross. Returns
min and max where the input was a range.

**Deduplication to handle:** both files carry state name data. `Engine.gs` has
`STATE_NAMES`, the calculator has an `n` field inside its tax table. Keep `STATE_NAMES`
as the only source and strip `n` from the tax table. Note the two disagree on DC:
Engine says "District of Columbia", calculator says "Washington D.C.". Pick one,
recommend the Engine value since it is already writing to live Deal records.

**Range handling:** `computeComp()` already collapses a single rate into a range where
min equals max. The calculator passes one rate, the Deal passes `hr` and `compmax`.
No special casing needed.

### Phase 2: Build step

`build.js`. Reads `an-comp-core.js`, replaces the block between
`/* CORE:START */` and `/* CORE:END */` in `client-cost-calculator.html`, writes the
file back. Run it after any core change. Add a note in the HTML header saying the
block is generated and must not be hand edited.

### Phase 3: Client calculator updates

1. Add the four new inputs from section 4. Show the California personal attendant
   question only when the ZIP resolves to CA.
2. Replace the naive gross with `computeComp()` from the core.
3. Surface overtime in the results: show overtime hours per week and the premium
   hours as their own line in the breakdown, the way `comp_calc_math` does.
4. Carry `comp_calc_flags` through to the UI. If someone enters a rate below the
   state minimum wage or below the AN $35/hr floor, say so rather than returning a number.
5. Add hidden HubSpot form fields for the four new inputs.
6. Write both `search_type` and the mapped `search_type_cleaned` value.

### Phase 4: Apps Script employer cost layer

1. Paste `an-comp-core.js` as `CompCore.gs`.
2. Strip the moved functions out of `Engine.gs`, leaving it as the wage layer entry point.
3. New `EmployerCost.gs` calling `computeEmployerCost()`.
4. Extend `Runner.gs`:
   - Add `consultation_outcome` to `INPUT_PROPS` so a change to it triggers recalculation
     through the existing signature mechanism.
   - Add the new employer cost properties to `OUTPUT_PROPS`.
   - Gate the employer cost layer on outcome, see section 6.
   - Write everything in the same batch call, preserving the existing rule that
     `comp_calc_completed_at` changes only when every output is already in place.

The existing signature hash, watermark, lock, and retry logic all carry over unchanged.

### Phase 5: Parity test

The test that makes this whole plan worth doing. Run a fixture set of inputs through
both entry points and assert the outputs match to the cent.

Minimum fixture coverage:
- Texas live-out 50 hrs (federal overtime only)
- California live-out personal attendant 50 hrs, 10 hrs/day (daily overtime)
- California live-out standard 60 hrs, 13 hrs/day (double time path)
- California live-in personal attendant (45 hr threshold)
- New York live-in 50 hrs (44 hr threshold)
- Texas live-in (federal exemption, no overtime at all)
- Colorado 50 hrs, 13 hrs/day (12 hr daily rule)
- A rate below state minimum wage (expect review status, no number)
- A rate below the $35 AN floor (expect review status)
- An NCS search (expect skipped, see D3)

Extend the existing `testEngine()` pattern in `Runner.gs`. It already asserts
`128700 / 143000 / 10` for a Texas fixture, so keep that as the regression anchor.

### Phase 6: HubSpot setup

Properties and form, section 7.

---

## 6. The Consultation outcome trigger

**Verified** `consultation_outcome` values on the Deal object:

| Value | Label |
|---|---|
| `held_good_fit` | Good Fit |
| `held_not_a_fit` | NAF |
| `Bad Timing/Follow Up` | Bad Timing/Follow Up |
| `consultation_held` | Maybe |
| `consultation_not_held` | Not Held |

Note the value `consultation_held` carries the label "Maybe", which is confusing to read
in code. Comment it wherever it appears.

### Two trigger paths, both already built

**Polling, up to 5 minutes:** add `consultation_outcome` to `INPUT_PROPS`. It enters the
signature hash, so setting it marks the Deal as changed and the next poll recalculates.
No new infrastructure.

**Webhook, instant:** `Runner.gs` already exposes `doPost` behind a `WEBHOOK_KEY`. Create
a HubSpot workflow, enrollment trigger "Consultation outcome is known", action "Send a
webhook" pointing at the deployed web app URL. The handler re-reads the Deal from the API
and never trusts the payload for math, which is the right design and needs no change.

**Recommendation:** wire the webhook, keep polling as the safety net. The webhook gives
the PCE a live number the moment the rep sets the outcome. Polling catches anything the
workflow misses.

### Gate logic

```
Compute employer cost when:
  comp_calc_status is "ok"
  AND consultation_outcome is in the priced set
  
Priced set, pending D1: held_good_fit, consultation_held ("Maybe")
```

Deals with outcome `held_not_a_fit`, `consultation_not_held`, or blank get
`employer_cost_status = "not_applicable"` and no figures. Writing an explicit status
rather than leaving fields blank means a rep can tell "not calculated yet" from
"calculated, came back empty".

---

## 7. HubSpot properties

### 7.1 Deal properties to create

None of these exist. Verified.

| Internal name | Label | Type |
|---|---|---|
| `employer_cost_status` | Employer cost status | Single line text |
| `employer_cost_first_year_min` | Employer cost first year min | Number |
| `employer_cost_first_year_max` | Employer cost first year max | Number |
| `employer_cost_annual_min` | Employer cost year 2+ min | Number |
| `employer_cost_annual_max` | Employer cost year 2+ max | Number |
| `employer_cost_taxes_min` | Employer taxes annual min | Number |
| `employer_cost_taxes_max` | Employer taxes annual max | Number |
| `employer_cost_agency_fee_min` | Agency fee min | Number |
| `employer_cost_agency_fee_max` | Agency fee max | Number |
| `employer_cost_math` | Employer cost math | Multi line text |
| `employer_cost_plain_pce` | Employer cost PCE sentence | Multi line text |
| `employer_cost_completed_at` | Employer cost completed at | Single line text |

Mirrors the existing `comp_*` naming so the two sets read as one system.

### 7.2 Deal properties that already exist

Verified present: `comp_calc_status`, `comp_calc_flags`, `comp_calc_completed_at`,
`comp_calc_math`, `comp_calc_input_signature`, `comp_weekly_min`, `comp_weekly_max`,
`ot_hrs_wk`, `consultation_outcome`, `search_type_cleaned`, `live_in_or_out`,
`liveinliveout`, `ca_personal_attendant`, `guaranteed_hours_per_week`,
`working_hours_per_day`, `working_days_per_week`, `overtime_expected`

**Verify before building:** `comp_annualized_min`, `comp_annualized_max`,
`comp_plain_pce`, `comp_budget_band` appear in `OUTPUT_PROPS` in `Runner.gs` and are
presumably live, but the keyword search did not surface them. Confirm they exist before
assuming.

### 7.3 Contact properties to create

Verified absent. All Number type on the Contact object except where noted.

| Internal name | Label | Type |
|---|---|---|
| `calc_hours_per_week` | Calculator hours per week | Number |
| `calc_hours_per_day` | Calculator hours per day | Number |
| `calc_days_per_week` | Calculator days per week | Number |
| `calc_live_in_or_out` | Calculator live in or out | Dropdown, mirror the Deal options |
| `calc_ca_personal_attendant` | Calculator CA personal attendant | Checkbox |
| `calc_gross_annual_wages` | Calculator gross annual wages | Number |
| `calc_overtime_hours_week` | Calculator overtime hours per week | Number |
| `calc_agency_fee` | Calculator estimated agency fee | Number |
| `calc_first_year_total` | Calculator first year total | Number |
| `calc_ongoing_annual` | Calculator year 2+ annual cost | Number |

HubSpot auto generates an internal name from the label and its guess will not match.
Click Edit beside the internal name on each one and paste the exact value.

### 7.4 Contact properties that already exist

`firstname`, `lastname`, `email`, `anticipated_hourly_rate`, `search_type`, `zip`,
`state`, `high_value_lead`

`high_value_lead` is already defined in the portal as "greater than $45/hr", so the
calculator can set it directly with no new logic.

### 7.5 Form

Form GUID `c8989759-617c-4855-8fca-55083b6059d4`. Three visible fields: `firstname`,
`lastname`, `email`, all required. Every property in 7.3 and 7.4 added as a Hidden field.

Post submit must be "Display a thank you message", never "Redirect to another page".
A redirect navigates away from the calculator and the visitor never sees the breakdown
they traded their email for.

The calculator already logs a self audit to the browser console naming any hidden field
that is missing from the form, so setup errors surface immediately rather than silently
discarding data.

---

## 8. Open decisions

These need a person, not a build agent. Each one changes behaviour.

**D1. Which consultation outcomes get priced?**
Recommend `held_good_fit` plus `consultation_held` ("Maybe"). Open question is whether
`Bad Timing/Follow Up` should also price, since those Deals often revive months later
and a stored number goes stale.

**D2. Where does the agency fee percentage come from on the Deal side?**
The calculator derives it from search type at 25% or 30%. The Deal carries
`type_of_search__launch_fee`, a number. If a Deal has a negotiated fee, that should win
over the tier default. Confirm which is authoritative.

**D3. What does the calculator do for NCS and Rotational searches?**
`Engine.gs` skips both. Three options: remove them from the public dropdown, keep them
but show a "contact us for a custom quote" panel instead of numbers, or extend the engine
to price them. Recommend the middle option, it captures the lead without publishing a
number the Deal side will not stand behind.

**D4. Fee tier confirmation.**
Newborn Care Specialist and Rotational Nanny / Team are currently assigned 30% on the
reading that they count as specialized searches. Household Manager and Family Assistant
are at 25%. Both are assumptions, one line each to change.

**D5. Does the public calculator show a range or a single number?**
The engine is built for ranges. The calculator asks for one rate. Offering a rate range
on the public page would match the Deal side exactly, at the cost of a slightly heavier form.

**D6. Benefits defaults on the Deal side.**
The calculator collects paid vacation weeks, sick days, and a health stipend under
Advanced Options, defaulting to 2 weeks, 5 days, and $0. Deals have no equivalent fields.
Either add them, or have the Deal side use the same defaults and say so in
`employer_cost_math`.

---

## 9. Acceptance criteria

1. `an-comp-core.js` contains every formula in the system. Neither entry point contains
   arithmetic on rates, hours, taxes, or fees.
2. The parity fixture suite passes: identical outputs to the cent from both entry points
   across all ten cases in Phase 5.
3. `testEngine()` still returns its existing Texas values of 128700, 143000, 10.
4. Setting Consultation outcome to Good Fit on a Deal with complete comp inputs produces
   populated `employer_cost_*` properties within 5 minutes by polling, or within seconds
   by webhook.
5. Setting it to NAF produces `employer_cost_status = "not_applicable"` and no figures.
6. The public calculator's first year total for the California fixture matches
   `employer_cost_first_year_min` on an equivalent Deal.
7. The browser console reports all hidden fields captured.
8. Running `node build.js` after a core change updates the HTML, and the page still
   works when opened directly from disk.

---

## 10. Deployment order

Order matters. Each step depends on the one before it.

1. Resolve D1 through D6.
2. Create the Deal properties (7.1) and Contact properties (7.3).
3. Build and pass the parity suite locally. Nothing ships until it is green.
4. Paste `CompCore.gs`, update `Engine.gs` and `Runner.gs` in Apps Script.
5. Run `dryRunRecent(14)` with `write: false`. Compare the log against what reps expect
   on recent live Deals. This is the existing QA path and it should be used here.
6. Turn on the Apps Script side.
7. Add the hidden fields to the HubSpot form.
8. Build and publish the client calculator.
9. Verify criterion 6, the public number and the Deal number agree.
10. Wire the HubSpot workflow for the instant webhook.

Steps 4 through 6 touch live Deal records. Step 5 is not optional.

---

## 11. Notes for whoever builds this

- Apps Script has no modules. `an-comp-core.js` must be plain top level declarations.
  Adding `export` anywhere breaks the Apps Script side silently.
- `Engine.gs` avoids locale APIs on purpose, with a hand rolled comma formatter, so that
  Node and Apps Script print identically. Keep that. `toLocaleString()` will diverge.
- The calculator currently uses `toLocaleString('en-US')`. Replace it with the Engine's
  `commas()` helper as part of Phase 1, or the two surfaces will format differently.
- Money in the Engine is handled in integer cents and rounded once at the end. The
  calculator rounds with `Math.round()` at several points. Adopt the cents approach
  throughout, it is the reason the parity test can assert exact equality.
- The webhook handler re-reads the Deal from the API rather than trusting the payload.
  Preserve that.
- `Runner.gs` writes outputs in one batch call and only then updates
  `comp_calc_completed_at`. Downstream automation enrols on that timestamp, never on the
  comp fields themselves. The employer cost fields must follow the same discipline.
- Use they/them for nannies, caregivers, and educators in every user facing string.
