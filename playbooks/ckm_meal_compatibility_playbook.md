# CKM Meal Compatibility Playbook v0.3

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

## Classification Function

Execute this function exactly once using `meal_totals`. Copy its returned `compatibility` and `primary_reason_code` directly into the output. Do not reinterpret or revise the return value using food names or any other context.

```typescript
type Compatibility =
  | "expected_to_support_ketosis"
  | "moderate_impact_expected"
  | "may_challenge_ketosis";

type Result = {
  compatibility: Compatibility;
  primary_reason_code: string;
};

function classifyMeal(totals: {
  net_carb_g: number;
  fiber_g: number;
  protein_g: number;
  fat_g: number;
}): Result {
  const net = totals.net_carb_g;
  const highFiber = totals.fiber_g > 8;
  const proteinDominant =
    totals.protein_g >= 60 && totals.fat_g < totals.protein_g;

  if (net < 7) {
    return {
      compatibility: "expected_to_support_ketosis",
      primary_reason_code: "low_net_carb",
    };
  }

  if (net < 10) {
    return proteinDominant
      ? {
          compatibility: "moderate_impact_expected",
          primary_reason_code: "low_net_carb_protein_dominant",
        }
      : {
          compatibility: "expected_to_support_ketosis",
          primary_reason_code: "low_net_carb",
        };
  }

  if (net < 15) {
    return {
      compatibility: "moderate_impact_expected",
      primary_reason_code: "moderate_net_carb",
    };
  }

  if (net <= 20) {
    if (highFiber && proteinDominant) {
      return {
        compatibility: "moderate_impact_expected",
        primary_reason_code: "moderate_net_carb_balanced_adjustments",
      };
    }
    if (highFiber) {
      return {
        compatibility: "expected_to_support_ketosis",
        primary_reason_code: "moderate_net_carb_high_fiber",
      };
    }
    if (proteinDominant) {
      return {
        compatibility: "may_challenge_ketosis",
        primary_reason_code: "moderate_net_carb_protein_dominant",
      };
    }
    return {
      compatibility: "moderate_impact_expected",
      primary_reason_code: "moderate_net_carb",
    };
  }

  if (net <= 25 && highFiber) {
    return {
      compatibility: "moderate_impact_expected",
      primary_reason_code: "higher_net_carb_high_fiber",
    };
  }

  return {
    compatibility: "may_challenge_ketosis",
    primary_reason_code: "higher_net_carb",
  };
}
```

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
