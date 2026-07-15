# CKM Meal Compatibility Playbook v0.2

## Scope

Use this playbook only after the meal's consumed nutrition has been calculated.

The task is to predict how compatible the meal's nutrition structure is with maintaining ketosis and to produce one short user-facing sentence. This is a prediction based on the meal itself. It is not an observed metabolic outcome and must not use language claiming that ketosis was maintained, interrupted, or improved.

Do not estimate or recalculate nutrition. Treat `meal_totals` as authoritative. Use `meal_items` only to identify the foods that materially drive the explanation.

## Required Input

The input must contain:

- `meal_totals.net_carb_g`
- `meal_totals.fiber_g`
- `meal_totals.protein_g`
- `meal_totals.fat_g`
- at least one `meal_items` entry with an item ID, food name, and consumed nutrient contribution

If these values are incomplete, the workflow must fail before classification rather than infer missing values.

## Compatibility Values

Return exactly one:

- `expected_to_support_ketosis`
- `moderate_impact_expected`
- `may_challenge_ketosis`

## Classification Decision Table

First calculate these two booleans from `meal_totals` without using food names:

```text
high_fiber = fiber_g > 8
protein_dominant = protein_g >= 60 and fat_g < protein_g
```

Then select exactly one row from this mutually exclusive table. The selected row directly determines both `compatibility` and `primary_reason_code`.

| Net carbohydrate | Additional condition | Compatibility | Primary reason code |
|---|---|---|---|
| `net_carb_g < 7` | any | `expected_to_support_ketosis` | `low_net_carb` |
| `7 <= net_carb_g < 10` | `protein_dominant = true` | `moderate_impact_expected` | `low_net_carb_protein_dominant` |
| `7 <= net_carb_g < 10` | `protein_dominant = false` | `expected_to_support_ketosis` | `low_net_carb` |
| `10 <= net_carb_g < 15` | any | `moderate_impact_expected` | `moderate_net_carb` |
| `15 <= net_carb_g <= 20` | `high_fiber = true` and `protein_dominant = true` | `moderate_impact_expected` | `moderate_net_carb_balanced_adjustments` |
| `15 <= net_carb_g <= 20` | `high_fiber = true` and `protein_dominant = false` | `expected_to_support_ketosis` | `moderate_net_carb_high_fiber` |
| `15 <= net_carb_g <= 20` | `high_fiber = false` and `protein_dominant = true` | `may_challenge_ketosis` | `moderate_net_carb_protein_dominant` |
| `15 <= net_carb_g <= 20` | `high_fiber = false` and `protein_dominant = false` | `moderate_impact_expected` | `moderate_net_carb` |
| `20 < net_carb_g <= 25` | `high_fiber = true` | `moderate_impact_expected` | `higher_net_carb_high_fiber` |
| `20 < net_carb_g <= 25` | `high_fiber = false` | `may_challenge_ketosis` | `higher_net_carb` |
| `net_carb_g > 25` | any | `may_challenge_ketosis` | `higher_net_carb` |

Do not apply high fiber or protein dominance outside the row in which it is explicitly listed. In particular:

- between 10g and less than 15g net carbohydrate, neither factor changes the result;
- above 20g net carbohydrate, protein dominance does not change the result;
- above 25g net carbohydrate, high fiber does not change the result.

Net carbohydrate is always the primary driver. Do not apply subjective health, ingredient-quality, food-processing, calorie, meal-time, or user-preference adjustments.

## Primary Reason Codes

Select the code that matches the final decision path:

- `low_net_carb`
- `moderate_net_carb`
- `higher_net_carb`
- `moderate_net_carb_high_fiber`
- `higher_net_carb_high_fiber`
- `low_net_carb_protein_dominant`
- `moderate_net_carb_protein_dominant`
- `moderate_net_carb_balanced_adjustments`

Use `moderate_net_carb_balanced_adjustments` only when both adjustments apply and cancel each other.

## User-Facing Sentence

Return one concise sentence in the requested locale.

- Keep English output at 30 words or fewer.
- Use predictive wording such as `expected`, `may`, or `likely`.
- Mention net carbohydrate first because it is the primary driver.
- Mention fiber or protein-dominant structure only when it changed the result or the two adjustments cancelled.
- You may name up to two foods that materially contribute to the primary driver.
- Do not list every food.
- Do not provide advice, recommendations, warnings, or metabolic outcome claims.
- Do not mention thresholds or the classification procedure.

## Key Item IDs

Return zero to two input `item_id` values that materially support the sentence.

- Prefer the largest net-carbohydrate contributors when net carbohydrate drives the result.
- If an adjustment is described, include a major fiber or protein contributor when useful.
- Never invent an item ID.
- An empty array is valid when naming a food would not improve the explanation.
