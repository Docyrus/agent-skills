# Account settings — field reference

Two command groups under `docyrus account`. Convenience flags are camelCase;
`--data` / `--from-file` use the **snake_case key** shown here. Every field is
optional on `update`, and an update that resolves to no fields is rejected.

- **Flag** — the camelCase `update` flag.
- **Key (`--data`)** — the snake_case key to use inside `--data` / `--from-file`.
- **Type** — `string`, or `number` (passed as a bare number, e.g. `--decimalPrecision 2`).

## Contents

- [User profile (`account user-profile`)](#user-profile-account-user-profile)
- [Tenant preferences (`account tenant-preferences`)](#tenant-preferences-account-tenant-preferences)
  - [Locale & region](#locale--region)
  - [Date & time formats](#date--time-formats)
  - [Working hours](#working-hours)
  - [Currency & numbers](#currency--numbers)

## User profile (`account user-profile`)

Self-service fields for the signed-in user. `get` returns the full user record;
the fields below are the ones `update` can change.

| Flag | Key (`--data`) | Type | Notes |
|---|---|---|---|
| `--firstname` | `firstname` | string | First name. |
| `--lastname` | `lastname` | string | Last name. |
| `--mobile` | `mobile` | string | Mobile / telephone number. |
| `--dateOfBirth` | `date_of_birth` | string | Date of birth, `YYYY-MM-DD`. |
| `--gender` | `gender` | string | Gender id. |
| `--timeZone` | `time_zone` | string | IANA time zone id (e.g. `America/New_York`). |
| `--language` | `language` | string | Language id (e.g. `en_US`). |

**Not editable here** (shown by `get` but ignored by the self-service update):
email (has its own change-email flow), photo (separate upload), job title,
primary role, additional roles, and organization/hierarchy unit — those are
admin-managed.

## Tenant preferences (`account tenant-preferences`)

Workspace-wide regional & formatting settings. **Updating requires tenant-admin.**
`get` returns the current preferences as an object with camelCase keys matching
the flag names.

### Locale & region

| Flag | Key (`--data`) | Type | Notes |
|---|---|---|---|
| `--language` | `language` | string | Default workspace language id (e.g. `en_US`). |
| `--locale` | `locale` | string | Locale id (e.g. `en_GB`). |
| `--timeZone` | `time_zone` | string | IANA time zone id (e.g. `Europe/Istanbul`). |
| `--startWeekOn` | `start_week_on` | number | First day of the week: `0`=Sunday … `6`=Saturday. |
| `--fiscalYear` | `fiscal_year` | string | Fiscal year start month, `1`–`12`. |

### Date & time formats

PHP-style format strings (e.g. `Y-m-d`, `H:i`, `d.m.Y H:i`).

| Flag | Key (`--data`) | Type | Notes |
|---|---|---|---|
| `--dateFormat` | `date_format` | string | Date format string. |
| `--dateTimeFormat` | `date_time_format` | string | Combined date + time format string. |
| `--timeFormat` | `time_format` | string | Time format string. |
| `--longDateFormat` | `long_date_format` | string | Long/verbose date format. Nullable — clear by passing `null` via `--data`. |

### Working hours

| Flag | Key (`--data`) | Type | Notes |
|---|---|---|---|
| `--businessHoursBegin` | `business_hours_begin` | string | Day start time, `HH:mm:ss`. |
| `--businessHoursEnd` | `business_hours_end` | string | Day end time, `HH:mm:ss`. Must be after begin. |
| `--dailyWorkingHours` | `daily_working_hours` | number | Daily working time in **seconds** (8h = `28800`). |
| `--weeklyWorkingCapacity` | `weekly_working_capacity` | number | Weekly capacity in **hours** (e.g. `40`). |
| `--timeRounding` | `time_rounding` | string | Time rounding rule (e.g. `15min`). Nullable. |

### Currency & numbers

| Flag | Key (`--data`) | Type | Notes |
|---|---|---|---|
| `--baseCurrency` | `base_currency` | string | Base currency code (e.g. `EUR`). |
| `--currencyAbbrPosition` | `currency_abbr_position` | string | Currency symbol position: `BEFORE` or `AFTER` the amount. |
| `--decimalPrecision` | `decimal_precision` | number | Number of decimals shown, `0`–`8`. |
| `--decimalSeparator` | `decimal_separator` | string | Single character (e.g. `,`). |
| `--thousandSeparator` | `thousand_separator` | string | Single character (e.g. `.`). |
