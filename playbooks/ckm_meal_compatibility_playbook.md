# CKM Meal Compatibility Playbook v0.1

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

## Classification Procedure

Use the following procedure exactly.

```text
net = meal_totals.net_carb_g
fiber = meal_totals.fiber_g
protein = meal_totals.protein_g
fat = meal_totals.fat_g

if net < 10:
    base = expected_to_support_ketosis
else if net <= 20:
    base = moderate_impact_expected
else:
    base = may_challenge_ketosis

positive_adjustment =
    fiber > 8
    and ((15 <= net <= 20) or (20 < net <= 25))

negative_adjustment =
    protein >= 60
    and fat < protein
    and ((7 <= net < 10) or (15 <= net <= 20))

if positive_adjustment and negative_adjustment:
    final = base
else if positive_adjustment:
    promote base by one level
else if negative_adjustment:
    demote base by one level
else:
    final = base
```

Promotion means:

- `may_challenge_ketosis` becomes `moderate_impact_expected`
- `moderate_impact_expected` becomes `expected_to_support_ketosis`

Demotion means:

- `expected_to_support_ketosis` becomes `moderate_impact_expected`
- `moderate_impact_expected` becomes `may_challenge_ketosis`

Net carbohydrate is always the primary driver. Fiber and protein-dominant structure are supporting adjustments only when the conditions above are met. Do not apply subjective health, ingredient-quality, food-processing, calorie, meal-time, or user-preference adjustments.

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

