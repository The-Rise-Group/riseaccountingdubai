# Rise Accounting LP integration notes

Locked knowledge from the Sep 25-27 2026 debugging session. Read before making changes to the form flow.

## What ships now (as of commit 8100ad7, tag launch-ready-final-2026-09-27)

- Dubai and Dubai Formation landing pages have a custom modal form
- Form validation: 2-word name required, phone digit count enforced per country with real-time spacing
- Submit fires custom POST to HubSpot Forms API v3 direct-integration endpoint for form `26e3abb8`
- On response (success or failure), LP redirects to `riseaccounting.ae/thankyou` with URL params
- Framer thankyou page reads params and prefills HubSpot Meetings iframe
- Visitor picks a time slot and meeting confirms without a second confirmation form

## HubSpot form config

- Portal ID: `9031287`
- Form GUID: `26e3abb8-e298-40b9-8998-45dcac9f3959`
- Form name in HubSpot: `UAE Site: Formation Form`
- Region: `na1`
- Portal owner: Charlotte Fox
- Contact owner assigned by workflow: Max Raynor

## Fields sent to HubSpot Forms API v3

Exactly these 6 fields. Do not add more.

| POST body key | Source | Notes |
|---|---|---|
| `firstname` | from splitName helper on Full name input | lowercase per HubSpot standard property naming |
| `lastname` | from splitName helper | lowercase |
| `company` | Company name input | |
| `email` | Email input | |
| `phone` | Phone input in E.164-ish format `+CC digits with spaces` | |
| `what_services_are_you_most_interested_in_dubai` | Services pill picker output, mapped via SERVICE_VALUE_MAP | semicolon-delimited HubSpot enum values |

**Do not include `context` object.** Empirically confirmed to cause silent submission drops with no HTTP error. HubSpot returns 200 + redirectUri anyway.

**Do not add FNAME / LNAME / Company / Email / Phone uppercase shotgun variants.** Charlotte's mention of those was mental shorthand for the form fields but HubSpot stores them as lowercase HubSpot standard properties. Shotgun added scoring noise and caused drops.

## Services field label vs value mismatch

Charlotte's HubSpot form services field has 7 options where 4 display labels do not match their stored enum values:

| Label visitor sees on LP pill picker | Enum value LP must send in POST |
|---|---|
| Company Formation | `Year End Accounts` |
| Corporate Tax | `Corporate Tax` |
| Tax Residency Certificates | `Outsourced Finance Function` |
| Accounting & Bookkeeping | `Accounting & Bookkeeping` |
| Annual Accounts | `EMI Options` |
| Management Accounts | `SEIS & EIS` |
| VAT and/or Payroll | `VAT and/or Payroll` |

The `SERVICE_VALUE_MAP` inside `submitLead` in both LP files handles this translation. If Charlotte fixes the option values on her HubSpot form so labels match values, the map becomes redundant. Safe to keep either way.

## Framer thankyou page redirect

LP builds this URL after form submit:

```
https://riseaccounting.ae/thankyou
  ?firstname=X&firstName=X       (both variants, Framer reads camelCase)
  &lastname=Y&lastName=Y         (both variants)
  &email=Z                       (lowercase, matches Framer allowedKeys)
  &phone=W                       (lowercase)
  &company=V                     (lowercase)
  &services=A                    (comma-separated visitor-friendly labels)
  &utm_source=... &utm_medium=... etc.
```

Framer's `HubSpotMeetingPrefill.tsx` reads camelCase keys for `firstName` / `lastName` / `email` / `phone` / `company` and appends them to the Meetings iframe src. HubSpot Meetings then prefills all fields, skipping the confirmation step entirely.

## LP form validation rules

- **Name:** requires 2 words minimum via regex `/^\S+\s+\S+/`. Rejects single-word input at LP level so firstname == lastname fallback never happens downstream.
- **Phone:** exact digit count per country. UAE +971 = 9, UK +44 = 10, US +1 = 10, IE +353 = 9. Other = 7-15 range.
- **Phone real-time formatting:** groups digits with spaces per country pattern. UAE 2-3-4, UK 4-6, US 3-3-4, IE 2-3-4. Non-digits stripped. Paste of too many digits auto-truncates to country max.

## HubSpot debug patterns that WASTED time

Do not re-hit these:

1. **Tracked Site Domains rule does NOT apply to Forms API v3 direct-integration POSTs.** Mavlers article and HubSpot docs confirm the rule only applies to embedded forms (`hs-form-frame` divs), not custom fetch POSTs. Do not ask Charlotte to add domain to Tracked Domains for the custom POST path.
2. **HubSpot does not throttle for < 20 submissions per hour** on legitimate production forms. Assumption killed 4 hours.
3. **Curl vs browser is not a clean discriminator.** Both had mixed results. Real reason was probably the `context` object payload triggering something (empirical, unproven mechanism, just do not include it).
4. **Charlotte's "FNAME, LNAME" note was mental shorthand.** Not required field naming. HubSpot stores as `firstname` / `lastname` per render-definition endpoint.
5. **Native HubSpot embed was tested (commit `3a4dc62`) and failed** for contact creation. Contact only landed via Meetings booking, not form. Custom Forms API v3 POST is the working path.

## Git tag ladder for recovery

- `pre-form-rebuild-2026-09-25` = clean pre-work base
- `form-working-2026-09-26` = first services fix (superseded)
- `form-services-all-working-2026-09-26` = FNAME/LNAME shotgun (superseded)
- `form-manual-verified-2026-09-27` = no-context landing verified
- `form-and-prefill-2026-09-27` = camelCase URL only (broken, do not use)
- `launch-ready-2026-09-27` = both URL variants working
- **`launch-ready-final-2026-09-27` = FINAL WORKING STATE at commit `8100ad7`. Use this as the baseline for future work.**

Recovery command if anything breaks:
```
git reset --hard launch-ready-final-2026-09-27
git push --force-with-lease origin main
```

## Discovery: HubSpot render-definition endpoint

Public URL that returns form field schema JSON:
```
https://forms.hsforms.com/embed/v4/render-definition/{portalId}/{formGuid}
```

Requires loading through the embed script on a real HTTPS origin (not `data:` URL). Response contains `propertyReference` values and `options` arrays. Used to discover:
- Services field's exact internal property name (`what_services_are_you_most_interested_in_dubai`)
- The 4 label vs value mismatches in the services options

Useful for future work if Charlotte's form schema changes.

## HubSpot Meetings iframe URL prefill behavior

When ALL required fields are prefilled via URL params, HubSpot Meetings iframe skips the confirmation form step entirely. Visitor picks a time slot and meeting confirms immediately. This is the optimal LP-to-booked-meeting conversion path.

If any required field is missing, the confirmation form appears and visitor retypes. Retyped values overwrite the LP-submitted contact record fields. Avoid this by always prefilling all 4 required Meetings fields.

## Post-launch housekeeping (optional)

- Delete test contacts on HubSpot filtered by yopmail.com and mailinator.com email domains, and created 26-27 Sep 2026.
- Optional: Charlotte fixes services option value strings so labels match values. Removes need for SERVICE_VALUE_MAP.
- Optional: Framer developer adds UTM param storage on thankyou page so meeting bookings carry Meta / Google Ads attribution back into HubSpot Meetings source.
