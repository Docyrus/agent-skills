# tenant_brand — complete field reference

Every column of `tenant_brand` that the `account brands` API exposes, grouped by
section. Columns:

- **Flag** — the camelCase `account brands create|update` flag.
- **Column** — the `snake_case` key in the record / `--data` payload (the flag maps 1:1 onto this).
- **Type** — CLI flag type: `string`, `number`, `boolean`, `list` (comma-separated → array), `json` (a JSON string parsed into an object/array).
- **Notes** — constraints and meaning.

`--name` is the only field required on `create`. Every flag is also reachable as
a key inside `--data` / `--from-file` (use the **Column** name there). Flags win
over `--data` keys when both are given. `json` and `list` columns can only carry
structured values through the `json`/`list` flags or `--data`.

## Contents

- [Identity & status](#identity--status)
- [Colors](#colors)
- [Typography](#typography)
- [Spacing](#spacing)
- [Components](#components)
- [Images](#images)
- [Visual system](#visual-system)
- [Voice & messaging](#voice--messaging)
- [Slides / presentations](#slides--presentations)
- [Charts & data viz](#charts--data-viz)
- [Illustration](#illustration)
- [Content constraints](#content-constraints)
- [Server-managed fields](#server-managed-fields-read-only)

## Identity & status

| Flag | Column | Type | Notes |
|---|---|---|---|
| `--name` | `name` | string | **Required on create.** Brand display name. |
| `--description` | `description` | string | Free-text description. |
| `--websiteUrl` | `website_url` | string | Brand site; also the source URL for `fetch-from-website`. |
| `--isDefault` | `is_default` | boolean | Marks the tenant default brand. Only **one** default per tenant (enforced by a unique partial index — promoting a new default is the platform's job, not a second insert). |
| `--archived` | `archived` | boolean | **Update only.** Soft-archive. `list` returns only non-archived brands; archived brands are still reachable by `get`. |

## Colors

All `string` (hex like `#1d4ed8`, CSS color, or token).

| Flag | Column | Notes |
|---|---|---|
| `--colorScheme` | `color_scheme` | e.g. `light` / `dark` / `system`. |
| `--colorPrimary` | `color_primary` | Primary brand color. |
| `--colorSecondary` | `color_secondary` | Secondary color. |
| `--colorAccent` | `color_accent` | Accent color. |
| `--colorBackground` | `color_background` | Page/background color. |
| `--colorTextPrimary` | `color_text_primary` | Primary text color. |
| `--colorTextSecondary` | `color_text_secondary` | Secondary/muted text color. |
| `--colorLink` | `color_link` | Hyperlink color. |
| `--colorSuccess` | `color_success` | Success state color. |
| `--colorWarning` | `color_warning` | Warning state color. |
| `--colorError` | `color_error` | Error state color. |

## Typography

| Flag | Column | Type | Notes |
|---|---|---|---|
| `--fontFamilyPrimary` | `font_family_primary` | string | Body font family. |
| `--fontFamilyHeading` | `font_family_heading` | string | Heading font family. |
| `--fontFamilyCode` | `font_family_code` | string | Monospace/code font family. |
| `--fontSizeH1` | `font_size_h1` | string | H1 size (e.g. `2.5rem`). |
| `--fontSizeH2` | `font_size_h2` | string | H2 size. |
| `--fontSizeH3` | `font_size_h3` | string | H3 size. |
| `--fontSizeBody` | `font_size_body` | string | Body size. |
| `--fontWeightLight` | `font_weight_light` | number | Light weight (e.g. `300`). |
| `--fontWeightRegular` | `font_weight_regular` | number | Regular weight (e.g. `400`). |
| `--fontWeightMedium` | `font_weight_medium` | number | Medium weight (e.g. `500`). |
| `--fontWeightBold` | `font_weight_bold` | number | Bold weight (e.g. `700`). |
| `--lineHeightHeading` | `line_height_heading` | string | Heading line height (e.g. `1.2`). |
| `--lineHeightBody` | `line_height_body` | string | Body line height (e.g. `1.6`). |
| `--fonts` | `fonts` | json | Array of font definitions, e.g. `[{"family":"Inter","weights":[400,700]}]`. |

## Spacing

| Flag | Column | Type | Notes |
|---|---|---|---|
| `--spacingBaseUnit` | `spacing_base_unit` | number | Base spacing unit in px (e.g. `4`). |
| `--spacingBorderRadius` | `spacing_border_radius` | string | Default border radius (e.g. `8px`). |
| `--spacingPadding` | `spacing_padding` | json | Padding scale object, e.g. `{"sm":"8px","md":"16px"}`. |
| `--spacingMargins` | `spacing_margins` | json | Margin scale object. |

## Components

Component style objects (`json`). Shape is open — store whatever the renderer expects.

| Flag | Column | Type | Notes |
|---|---|---|---|
| `--componentButtonPrimary` | `component_button_primary` | json | Primary button style, e.g. `{"bg":"#1d4ed8","radius":"8px"}`. |
| `--componentButtonSecondary` | `component_button_secondary` | json | Secondary button style. |
| `--componentInput` | `component_input` | json | Input field style. |

## Images

All `string` URLs.

| Flag | Column | Notes |
|---|---|---|
| `--logoUrl` | `logo_url` | Primary logo. |
| `--faviconUrl` | `favicon_url` | Favicon. |
| `--ogImageUrl` | `og_image_url` | Open Graph / social share image. |

## Visual system

Open-shaped `json` objects.

| Flag | Column | Type | Notes |
|---|---|---|---|
| `--iconStyle` | `icon_style` | json | Icon style guidance (weight, corner style, set). |
| `--animations` | `animations` | json | Motion/animation guidance. |
| `--layout` | `layout` | json | Layout/grid guidance. |
| `--personality` | `personality` | json | Brand personality descriptors. |

## Voice & messaging

| Flag | Column | Type | Notes |
|---|---|---|---|
| `--activeVoicePreference` | `active_voice_preference` | string | Active vs passive voice preference. |
| `--directness` | `directness` | string | Directness of tone. |
| `--emojiPolicy` | `emoji_policy` | string | When/whether emoji are allowed. |
| `--formalityLevel` | `formality_level` | string | Formality (e.g. `casual`, `formal`). |
| `--grammarDo` | `grammar_do` | list | Grammar rules to follow. |
| `--grammarDont` | `grammar_dont` | list | Grammar rules to avoid. |
| `--metricEmphasis` | `metric_emphasis` | string | How to present metrics/numbers. |
| `--messagingFramework` | `messaging_framework` | json | Structured messaging framework. |
| `--toneByContext` | `tone_by_context` | json | Tone overrides per context, e.g. `{"support":"warm","legal":"precise"}`. |
| `--voiceIntent` | `voice_intent` | list | Voice intents (e.g. `bold,concise,friendly`). |
| `--preferredTerminology` | `preferred_terminology` | list | Preferred terms. |
| `--restrictedTerminology` | `restricted_terminology` | list | Terms to avoid. |
| `--prohibitedPhrases` | `prohibited_phrases` | list | Phrases never to use. |
| `--slogansAndTaglines` | `slogans_and_taglines` | list | Approved slogans/taglines. |
| `--punctuationRules` | `punctuation_rules` | string | Punctuation guidance (e.g. Oxford comma). |
| `--styleRules` | `style_rules` | string | General writing style rules. |
| `--preferredSentenceLength` | `preferred_sentence_length` | number | Target sentence length (words). |

## Slides / presentations

| Flag | Column | Type | Notes |
|---|---|---|---|
| `--coverSlideLayout` | `cover_slide_layout` | string | Cover slide layout guidance. |
| `--sectionDividerSlides` | `section_divider_slides` | string | Section divider guidance. |
| `--closingAndCtaSlides` | `closing_and_cta_slides` | string | Closing/CTA slide guidance. |
| `--slideDensity` | `slide_density` | string | Content density per slide. |
| `--textVisualBalance` | `text_visual_balance` | string | Text-to-visual balance. |
| `--maxTextPerSlide` | `max_text_per_slide` | number | Max text characters per slide. |
| `--headingToBodyRatio` | `heading_to_body_ratio` | number | Heading-to-body size ratio. Stored as `numeric(4,2)`; may come back as a string in responses. |

## Charts & data viz

| Flag | Column | Type | Notes |
|---|---|---|---|
| `--preferredChartColors` | `preferred_chart_colors` | list | Preferred chart color values. (Column is `uuid[]` in the table but the API validates a string list — pass color tokens/ids as strings.) |
| `--preferredChartTypes` | `preferred_chart_types` | list | Preferred chart types. (Same `uuid[]`-column caveat — pass as strings.) |
| `--annotationLabelingRules` | `annotation_labeling_rules` | string | Annotation/labeling rules. |
| `--competitiveComparisonRules` | `competitive_comparison_rules` | string | Rules for competitive comparisons. |

## Illustration

| Flag | Column | Type | Notes |
|---|---|---|---|
| `--illustrationStyle` | `illustration_style` | string | Illustration style. |
| `--illustrationNotes` | `illustration_notes` | string | Free-text illustration notes. |
| `--useHumanFigures` | `use_human_figures` | boolean | Whether human figures are allowed. |
| `--visualRealismLevel` | `visual_realism_level` | number | Realism level (smallint scale). |

## Content constraints

| Flag | Column | Type | Notes |
|---|---|---|---|
| `--industryConstraints` | `industry_constraints` | list | Industry-specific constraints. |
| `--regulatoryConsiderations` | `regulatory_considerations` | list | Regulatory considerations. |
| `--culturalSensitivityGuidelines` | `cultural_sensitivity_guidelines` | string | Cultural sensitivity guidance. |
| `--legalEthicalBoundaries` | `legal_ethical_boundaries` | string | Legal/ethical boundaries. |

## Server-managed fields (read-only)

These appear in `get`/`list` responses (or only in the database) and are **not**
settable through `create`/`update` flags:

| Column | Notes |
|---|---|
| `id` | UUID primary key (returned). |
| `tenant_id` | Set automatically from the active session. |
| `created_on` / `created_by` | Set on insert (returned). |
| `last_modified_on` / `last_modified_by` | Maintained on update (returned). |
| `workspace_slug` | Unique workspace slug; not part of the create/update DTO and not returned by the brand response — not settable via `account brands`. |
