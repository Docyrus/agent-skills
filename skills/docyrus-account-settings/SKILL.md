---
name: docyrus-account-settings
description: Read and update a Docyrus user's own profile and the active tenant's (workspace's) regional & formatting preferences using the `docyrus account user-profile` and `docyrus account tenant-preferences` CLI commands. Use when the user wants to view or change their personal profile (first/last name, mobile, date of birth, gender, time zone, language) or the workspace-wide regional settings (default language, locale, time zone, date/time format strings, first day of week, fiscal year, business/working hours, base currency, currency position, decimal/thousand separators, decimal precision). Triggers on "update my profile", "change my name/phone/timezone/language", "set my date of birth", "view my account info", "change the workspace language/locale/timezone", "set the date or time format", "set business/working hours", "change base currency", "decimal/thousand separator", "first day of the week", "fiscal year", "regional settings", "tenant preferences", `docyrus account user-profile`, `docyrus account tenant-preferences`, or any personal-account or tenant-wide regional/formatting settings task. For the full CLI command index see docyrus-cli-app; for tenant brand identity see docyrus-tenant-brand-management.
---

# Docyrus Account Settings

Two related settings surfaces, both under `docyrus account`:

- **User profile** (`account user-profile`) — the **signed-in user's own** personal
  details. Self-service: no admin rights needed, and it always acts on the current
  user (there is no id/selector).
- **Tenant preferences** (`account tenant-preferences`) — the **whole workspace's**
  regional & formatting defaults (language, dates, numbers, currency, working
  hours). Anyone can read them; **updating requires tenant-admin**.

Both are **read + update only** — there is no create or delete (each record exists
for the lifetime of the user / workspace).

For the complete field list (every flag, accepted values, and the snake_case key
to use inside `--data`) read [references/fields.md](references/fields.md).

## Workflow

1. **Confirm auth and the right workspace** — preferences are workspace-wide, so
   make sure the active tenant is the intended one.
   ```bash
   docyrus auth who --json
   ```
   No session → stop and ask the user to run `docyrus auth login`.

2. **Read current values first**, so an update only changes what's intended.
   ```bash
   docyrus account user-profile get --json
   docyrus account tenant-preferences get --json
   ```

3. **Update only the fields that change.** Updates are partial — unsent fields
   keep their current value. An update with no fields is rejected.

4. **Re-read** to confirm the values landed.

## Commands

Add `--json` for machine-readable output. Run `docyrus account <group> <cmd> --help`
for the live flag list.

### User profile

```bash
docyrus account user-profile get --json

docyrus account user-profile update \
  --firstname "Jane" \
  --lastname "Doe" \
  --mobile "+1 555 0100" \
  --dateOfBirth "1990-04-15" \
  --timeZone "America/New_York" \
  --language "en_US" \
  --json
```

Editable fields: `--firstname`, `--lastname`, `--mobile`, `--dateOfBirth`
(`YYYY-MM-DD`), `--gender`, `--timeZone`, `--language`. These are the only things a
user can change about themselves — email, photo, job title, roles, and
organization unit are managed elsewhere (admin tools), so they are not editable
here even though `get` may show them.

### Tenant preferences

```bash
docyrus account tenant-preferences get --json

# Regional
docyrus account tenant-preferences update \
  --language "en_US" --locale "en_GB" --timeZone "Europe/Istanbul" \
  --startWeekOn 1 --fiscalYear 1 --json

# Date / time format strings (PHP-style, e.g. Y-m-d, d.m.Y H:i)
docyrus account tenant-preferences update \
  --dateFormat "Y-m-d" --timeFormat "H:i" --dateTimeFormat "Y-m-d H:i" --json

# Working hours
docyrus account tenant-preferences update \
  --businessHoursBegin "09:00:00" --businessHoursEnd "17:00:00" \
  --dailyWorkingHours 28800 --weeklyWorkingCapacity 40 --json

# Currency & numbers
docyrus account tenant-preferences update \
  --baseCurrency "EUR" --currencyAbbrPosition AFTER \
  --decimalPrecision 2 --decimalSeparator "," --thousandSeparator "." --json
```

### Bulk / scripted payloads

Both `update` commands also accept `--data '<json>'` / `--from-file <file.json>`
(JSON only). Convenience flags win over `--data` keys when both set the same
field. Inside `--data`, use the **snake_case** keys (e.g. `time_zone`,
`date_format`, `business_hours_begin`, `currency_abbr_position`,
`decimal_precision`) — the full mapping is in
[references/fields.md](references/fields.md).

```bash
docyrus account tenant-preferences update \
  --from-file prefs.json --json
docyrus account user-profile update \
  --data '{"firstname":"Jane","time_zone":"America/New_York"}' --json
```

## Key rules

- **Only `get` and `update`** — no create, no delete. An empty update (no fields
  resolved) is rejected before any request is sent.
- **Profile is self-only.** `account user-profile` always targets the signed-in
  user; there is no selector and no way to edit another user here.
- **Preference updates need tenant-admin.** A non-admin caller gets a permission
  error from the server.
- **Time units matter:** `--businessHoursBegin` / `--businessHoursEnd` are
  `HH:mm:ss` (end must be after begin); `--dailyWorkingHours` is in **seconds**
  (8h = 28800); `--weeklyWorkingCapacity` is in **hours**.
- **Value ranges:** `--startWeekOn` is 0 (Sunday) – 6 (Saturday); `--fiscalYear`
  is the start month `1`–`12`; `--decimalPrecision` is 0–8;
  `--decimalSeparator` / `--thousandSeparator` are single characters;
  `--currencyAbbrPosition` is `BEFORE` or `AFTER`.
- **Reading preferences returns just the preferences record** with camelCase keys
  that line up with the flag names (e.g. `timeZone`, `dateFormat`,
  `currencyAbbrPosition`).
- **Clearing a nullable field** (`longDateFormat`, `timeRounding`): pass `null`
  via `--data` — an empty flag value is ignored, not cleared.

## Related skills

- **docyrus-cli-app** — full `docyrus` CLI command index and global conventions (`--json`, auth, environments).
- **docyrus-tenant-brand-management** — tenant brand identity (`account brands`).
- **docyrus-platform** — Docyrus platform concepts and building blocks.
