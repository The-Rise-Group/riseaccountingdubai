# Rise Accounting LP HubSpot Integration: Debugging Timeline

**Session dates:** 2026-09-24 to 2026-09-27 IST
**Total elapsed:** approximately 71 hours across multiple sessions
**Final resolution:** 2026-09-27 at 15:22 IST commit `8100ad7` tag `launch-ready-final-2026-09-27`

## Full chronology with commit SHAs and IST timestamps

### 2026-09-24 evening: First HubSpot wire-up attempts

| Time IST | Commit | Event |
|---|---|---|
| 16:39 | `cf21dd1` | Last commit BEFORE any HubSpot integration attempts. CTA opacity CSS fix. This is the clean pre-work baseline. |
| ~17:00 | (various) | First HubSpot embed integration attempts. Multiple swap and revert cycles. |
| ~17:29 | `6b13a24` | Solo Solo custom POST wire with name split. **First successful contact created in HubSpot from LP form.** |
| ~22:26 | `66288aa` | Last commit of Sep 24. Various CTA styling adjustments. |

### 2026-09-25: Chaos day. Many attempts, few landings.

| Time IST | Commit | Event |
|---|---|---|
| 03:42 | `dc90cd4` | Add Framer thankyou redirect. First introduction of URL-param passthrough. |
| 05:33 | `2253305` | Normalize phone to E.164. |
| 05:49 | `e407858` | Phone under 4 property name variants (shotgun begins here). |
| 06:16 | `92229b7` | Services shotgun with 4 candidate property names. |
| 06:19 | `b9b2f9d` | Add HubSpot tracking script and hubspotutk cookie logic. |
| 14:46 | `530ad04` | Add uppercase FNAME / LNAME / Company / Email / Phone shotgun per Charlotte's Teams reply. |
| 15:26 | `f7b9614` | Remove tracking script due to ghost `#mForm` activities. |
| 20:44 | `f72e761` | Switch to HubSpot native embed. Contacts stopped landing. |
| 21:28 | `34ee81d` | Re-add tracking script for native embed. |
| 21:32 | `5cd85f3` | Revert native embed swap. |
| 22:00 | `0a605ed` | Re-add tracking script per HubSpot AI diagnosis. Testing throttle theory. |
| 22:49 | `b91d166` | **Full rollback to pre-form-rebuild state** (tag `pre-form-rebuild-2026-09-25`) + Charlotte's 6 client content flags applied. |

### 2026-09-26 pre-dawn to morning: Services diagnosis breakthrough

| Time IST | Commit | Event |
|---|---|---|
| 02:28 | `e97c1e8` | Restore Sep 24 Solo Solo custom POST + first services field attempt (with wrong property name). |
| 03:24 | `-` | **Test Cors** contact landed in HubSpot from curl POST. First successful landing today. |
| 03:32 | `1ec6fc5` | User's Claude Chat creates rs-formation page using native HubSpot embed as parallel test. |
| 03:43 | `-` | **Verify Repeat** contact landed. Second successful landing. |
| 03:47 | `329044a` | TEMP embed inspect page pushed to extract form schema via HubSpot's render-definition endpoint. |
| 03:48 | `1c95cab` | Services property name fix: `_dubai_` to `_dubai` (double underscore + trailing removed). Discovered from render-definition JSON. |
| 03:56 | `d92dc34` | Second temp embed inspect. |
| 03:59 | `00f3a1a` | Delete temp embed inspect files. |
| 04:19 | `-` | **Value Match** contact landed. Services value populated correctly. Third successful landing. |
| 04:22 | `152988a` | Add SERVICE_VALUE_MAP for label to enum value conversion. First tag: `form-working-2026-09-26`. |
| **04:22+** | | **Rapid burst of test POSTs begins. HubSpot spam-detection appears to trigger. Nothing lands for the next several hours.** |

### 2026-09-26 morning to evening: Deep struggle

| Time IST | Commit | Event |
|---|---|---|
| 06:00 | `93ae21c` | Add Framer thankyou redirect + UTM passthrough. |
| 06:09 | `d2803ad` | Add FNAME/LNAME/Company/Email/Phone uppercase shotgun. **Reem Al Nuaimi contact landed** shortly after. Tag: `form-services-all-working-2026-09-26`. |
| 06:10-mid | | User's phone test with wife's phone: did not land. User's manual tests: did not land. Playwright tests: did not land. Long back-and-forth about causes. |
| Various | | Testing through 5 different IPRoyal residential IPs. All fail. Multiple hypotheses discarded (throttle, per-IP, domain-allowlist, native embed, etc.). |

### 2026-09-27 early morning: Context object discovery

| Time IST | Commit | Event |
|---|---|---|
| 03:24 | `-` | **Control Test** curl POST (5 fields, no context) landed. Pattern crystallizes: all landings had NO `context` object. All failures had context. |
| 03:29 | `fa912a4` | Remove FNAME/LNAME shotgun to match Value Match landing shape. |
| 14:12 | `754e741` | **Remove `context` object from POST body**. Tag: `form-manual-verified-2026-09-27` after user manual test lands. |
| 14:57 | `985c7bb` | Change redirect URL keys to camelCase (`firstName` / `lastName`) to match Framer's allowedKeys. |
| 15:04 | `bcd92c6` | Revert camelCase because Meetings iframe stopped prefilling entirely during test. |
| 15:07 | `a45cbd6` | Tighten LP validation: 2-word name required, phone digit cap per country. |
| 15:15 | `038e5fb` | Send BOTH lowercase AND camelCase for firstname/lastname. Tag: `launch-ready-2026-09-27`. |
| 15:22 | `8100ad7` | Real-time phone number formatting with country-specific spacing. **Tag: `launch-ready-final-2026-09-27`.** Final working state. |
| 15:33 | `d2d005e` | Add INTEGRATION-NOTES.md documentation. |
| ~15:40 | | User confirmed final flow tested twice, end to end, calendar booking skips confirmation form and confirms on time-slot pick. |

## Key learnings by session pain point

### Pain: HubSpot returns HTTP 200 with `redirectUri` on every submission, but contacts do not always create

**What made it hard:** the HTTP response is identical whether the submission creates a contact or gets silent-dropped by HubSpot's spam-detection. Impossible to distinguish success from silent-drop without a HubSpot CRM lookup on the exact email.

**What we tried that failed:**
- Assumed HubSpot's "Tracked Site Domains" spam-routing was the cause (only applies to native embeds, not Forms API v3 direct POSTs per Mavlers article and HubSpot own docs)
- Assumed per-IP throttling (rules out per 5+ different residential IPs)
- Assumed FNAME/LNAME uppercase field naming was required (Charlotte's mental shorthand, not actual)
- Assumed shotgun-duplicate fields (firstname AND FNAME with same value) would help (added spam-score noise)
- Assumed HubSpot's spam-detection was rate-limited on submissions per hour (< 20 does not trigger)
- Assumed native HubSpot embed would work better than custom POST (embed failed to create contacts even worse)

**What actually worked:**
- Custom POST to Forms API v3 with EXACTLY 6 fields (firstname, lastname, company, email, phone, services enum value)
- NO `context` object in POST body (empirically confirmed cause, mechanism unclear)
- NO uppercase shotgun variants
- Services value must be one of the 7 accepted enum strings, not the display label. Uses SERVICE_VALUE_MAP.

### Pain: Framer thankyou page not prefilling Meetings iframe First and Last name

**What made it hard:** Framer's `HubSpotMeetingPrefill.tsx` reads URL params using camelCase keys (`firstName`, `lastName`). Our LP sent lowercase (`firstname`, `lastname`). Email and Company worked because both sides happened to use lowercase for those.

**What actually worked:**
- LP sends BOTH lowercase and camelCase variants for firstname/lastname in redirect URL
- Framer picks up camelCase from URL, appends to Meetings iframe URL
- HubSpot Meetings receives all required fields prefilled
- **Confirmation form step SKIPS entirely.** Visitor picks time slot and meeting confirms immediately.

### Pain: Charlotte's services field label vs value mismatch

**What made it hard:** LP visitor picks a pill labeled "Company Formation" and the LP would send "Company Formation" as the value. HubSpot rejected because "Company Formation" is not in the enum values list. Silent drop of the services field (contact still creates but services empty).

**What actually worked:**
- Extract the render-definition JSON from `https://forms.hsforms.com/embed/v4/render-definition/9031287/26e3abb8-e298-40b9-8998-45dcac9f3959`
- Parse the options array to find each label's corresponding value
- 4 of 7 have mismatches: Company Formation → Year End Accounts, Tax Residency Certificates → Outsourced Finance Function, Annual Accounts → EMI Options, Management Accounts → SEIS & EIS
- LP applies SERVICE_VALUE_MAP inside submitLead to translate labels to values before POST

### Pain: LP accepting invalid phone numbers and single-word names

**What actually worked (Sep 27 15:07 commit `a45cbd6`):**
- Name validator: `/^\S+\s+\S+/` regex requires 2 words separated by whitespace with non-empty tokens on both sides
- Phone validator: exact digit count per country. UAE=9, UK=10, US=10, IE=9, Other=7-15
- Real-time phone formatting: non-digits stripped, digits truncated to country max, spacing added per country pattern (UAE 2-3-4, UK 4-6, US 3-3-4, IE 2-3-4)

## Milestone tag ladder (chronological)

| Tag | Commit | State |
|---|---|---|
| `pre-form-rebuild-2026-09-25` | `b91d166` | Clean pre-form-work base. Rollback here if entire form work needs undo. |
| `form-working-2026-09-26` | `152988a` | First services fix. Superseded. |
| `form-services-all-working-2026-09-26` | `d2803ad` | FNAME/LNAME shotgun. Superseded (shotgun found harmful). |
| `form-manual-verified-2026-09-27` | `754e741` | No-context landing verified via manual test. |
| `form-and-prefill-2026-09-27` | `985c7bb` | camelCase URL only. Broken. Do not use. |
| `launch-ready-2026-09-27` | `038e5fb` | Both URL variants working. |
| **`launch-ready-final-2026-09-27`** | **`8100ad7`** | **FINAL WORKING STATE.** Full validation, phone formatting, both URL variants, no context, services mapping. |

Recovery command:
```
git reset --hard launch-ready-final-2026-09-27
git push --force-with-lease origin main
```

## What Charlotte and Framer developer still need to do (post-launch, optional)

- **Charlotte:** fix her HubSpot form's services option value strings so labels match values. Removes need for SERVICE_VALUE_MAP in LP code. Not blocking. 5 minutes of her time.
- **Framer developer:** add UTM param storage on thankyou page so meeting bookings carry Meta / Google Ads attribution back into HubSpot Meetings source data. Currently UTMs pass through to the thankyou page URL but do not flow into the Meetings booking record. Not blocking. 30 minutes of work.

## When starting the next session on this codebase

Read `INTEGRATION-NOTES.md` and this file (`DEBUGGING-TIMELINE.md`) BEFORE making any changes to the form flow. Do not re-run the debug patterns that failed. The locked answers are non-obvious and cost ~48 hours to discover.

Session cost breakdown by phase:
- 24-25 Sep chaotic wiring attempts: ~10 hours
- 26 Sep morning services diagnosis: ~4 hours (productive)
- 26 Sep morning to evening struggle with context object: ~14 hours
- 27 Sep resolution and hardening: ~4 hours (productive)
