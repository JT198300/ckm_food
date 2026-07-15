# CKM Nutrition And Keto Playbook v0.2

## Scope

Use this playbook only after the intake stage has produced standardized food-and-amount JSON.

This stage resolves per 100g nutrient fields, keto labels, deterministic derived values, and validation.

It must not identify food from images. It must not infer new foods outside the provided intake item.

## Input Contract

Each item should already contain:

- `item_id`
- `item_name`
- `normalized_name`
- `item_type`
- `estimated_amount`
- `unit`
- `recognition_confidence`
- `source_type`
- `source_image_index`
- `source_text_span`

## Database First Rule

Always prefer existing verified or usable food data over LLM generation.

If a food database match provides usable `nutrition_per_100g` and labels, use that data and do not call the nutrition or keto label LLM for that item.

Only call LLM generation when:

- no usable food database match exists;
- the match is too ambiguous;
- the existing data is missing required core nutrition fields;
- benchmark mode explicitly requests regeneration.

Generated data must be validated before it can be used for reporting or written back as a candidate.

## Nutrition Per 100g Rules

Generate estimated raw nutrient values per 100g only.

Do not output calories, consumed nutrition, or meal totals from the LLM.

Calories must be computed by deterministic code from the per 100g nutrient fields.

Do not calculate net carbs.

Do not output final Carb Impact.

Use the common cooked or served form unless the item name or context clearly indicates otherwise.

For dishes, estimate the average prepared dish per 100g, including typical cooking oil, sauce, moisture, and preparation style when relevant.

Macronutrients should usually be present:

- `protein_g`
- `fat_g`
- `total_carb_g`
- `fiber_g`
- `sugar_g`
- `sugar_alcohol_g`

`total_carb_g` includes sugar alcohol. Sugar alcohol must not be added on top of `total_carb_g` again. Deterministic code excludes `sugar_alcohol_g` from net carbs.

Electrolyte micronutrients should be present when reasonably knowable:

- `sodium_mg`
- `potassium_mg`
- `magnesium_mg`

Use null for electrolyte micronutrients when they are not reasonably knowable from common food data or preparation assumptions.

Detailed fat fields may be null when not reasonably knowable.

Do not invent micronutrients only to make the output look complete.

Set `nutrition_confidence = "high"` when the food is simple, standardized, or has stable common per 100g values.

Set `nutrition_confidence = "low"` when recipe, oil, sauce, batter, sugar, starch, restaurant variation, or dish identity is uncertain.

## Keto Label Rules

CKM logic:

- Carb is a limit.
- Protein is a target.
- Fat is a lever.
- Fat Quality describes fat characteristics, not moral quality or absolute health value.

Do not calculate final Carb Impact. Final Carb Impact is computed by code from net carbs.

Generate:

- `protein_support`
- `fat_support`
- `fatty_acid_profile`
- `fat_processing`
- confidence

Do not output label reasons unless the configured output schema explicitly requests them. The current CKM combined app does not request reasons.

Output short decision descriptions when the configured schema requests `label_descriptions`:

- `protein_support`: use the fixed template defined by `calculateSupportLabels` below;
- `fat_support`: use the fixed template defined by `calculateSupportLabels` below.

Each description must be concise, ideally under 140 characters. It must support the selected label using the generated per 100g values or an explicit food property. Do not output chain-of-thought, long explanations, health advice, or generic restatements of the label.

## Protein Support

- `strong`: final result returned by the executable rule below.
- `moderate`: final result returned by the executable rule below.
- `limited`: final result returned by the executable rule below.

## Fat Support

- `strong`: final result returned by the executable rule below.
- `moderate`: final result returned by the executable rule below.
- `limited`: final result returned by the executable rule below.

## Executable Protein and Fat Support Rules

After generating the per-100g nutrition values for an item, execute the following logic exactly. Do not replace it with qualitative judgement, food-family assumptions, or an alternative threshold.

This calculation happens inside the LLM node. `calculated_kcal_per_100g` is an internal intermediate value used only for these labels and descriptions; do not add it to the structured output. The deterministic output layer independently recomputes final calories and net carbs.

Critical arithmetic constraint: use only `calculated_kcal_per_100g` from this function as the percentage denominator. Do not substitute remembered food calories, database calories, rounded label calories, or any other calorie estimate. Before returning an item, verify that both percentages, both descriptions, and both final labels match the same function execution.

Equivalent final decision table, used as the mandatory self-check:

```text
protein_g >= 20:
  protein_support = moderate if protein_kcal_pct < 25, otherwise strong
10 <= protein_g < 20:
  protein_support = moderate
protein_g < 10:
  protein_support = moderate only if protein_g >= 5 and protein_kcal_pct >= 40, otherwise limited

fat_g >= 20:
  fat_support = moderate if net_carb_g >= 20 or protein_kcal_pct >= 35, otherwise strong
8 <= fat_g < 20:
  fat_support = moderate
fat_g < 8:
  fat_support = moderate only if fat_g >= 5 and fat_kcal_pct >= 50, otherwise limited
```

No other transition is allowed. In particular, a `moderate` base can never become `strong`, and Fat Support must not be upgraded by general food intuition. If a description states `base moderate`, its final support label must be `moderate`.

```typescript
type SupportTier = "strong" | "moderate" | "limited";

type SupportResult = {
  protein_support: SupportTier;
  fat_support: SupportTier;
  protein_description: string;
  fat_description: string;
};

function calculateSupportLabels(n: {
  protein_g: number;
  fat_g: number;
  total_carb_g: number;
  fiber_g: number;
  sugar_alcohol_g: number;
}): SupportResult {
  const protein_g = Math.max(n.protein_g, 0);
  const fat_g = Math.max(n.fat_g, 0);
  const total_carb_g = Math.max(n.total_carb_g, 0);
  const fiber_g = Math.max(n.fiber_g, 0);
  const sugar_alcohol_g = Math.max(n.sugar_alcohol_g, 0);

  // Sugar alcohol is included in total carbs but excluded from net carbs.
  const net_carb_g = Math.max(total_carb_g - fiber_g - sugar_alcohol_g, 0);
  const protein_kcal = protein_g * 4;
  const fat_kcal = fat_g * 9;
  const calculated_kcal_per_100g =
    protein_kcal + fat_kcal + net_carb_g * 4 + sugar_alcohol_g * 2;
  const protein_kcal_pct = calculated_kcal_per_100g > 0
    ? protein_kcal / calculated_kcal_per_100g * 100
    : 0;
  const fat_kcal_pct = calculated_kcal_per_100g > 0
    ? fat_kcal / calculated_kcal_per_100g * 100
    : 0;

  let protein_base: SupportTier;
  if (protein_g >= 20) protein_base = "strong";
  else if (protein_g >= 10) protein_base = "moderate";
  else protein_base = "limited";

  let protein_support = protein_base;
  if (protein_base === "strong" && protein_kcal_pct < 25) {
    protein_support = "moderate";
  } else if (
    protein_base === "limited" &&
    protein_g >= 5 &&
    protein_kcal_pct >= 40
  ) {
    protein_support = "moderate";
  }

  let fat_base: SupportTier;
  if (fat_g >= 20) fat_base = "strong";
  else if (fat_g >= 8) fat_base = "moderate";
  else fat_base = "limited";

  let fat_support = fat_base;
  if (
    fat_base === "strong" &&
    (net_carb_g >= 20 || protein_kcal_pct >= 35)
  ) {
    fat_support = "moderate";
  } else if (
    fat_base === "limited" &&
    fat_g >= 5 &&
    fat_kcal_pct >= 50
  ) {
    fat_support = "moderate";
  }

  const protein_description =
    `${protein_g.toFixed(1)}g protein/100g -> base ${protein_base}; ` +
    `${protein_kcal_pct.toFixed(1)}% of calculated kcal -> ${protein_support}.`;
  const fat_description =
    `${fat_g.toFixed(1)}g fat/100g -> base ${fat_base}; ` +
    `net carbs ${net_carb_g.toFixed(1)}g and protein ${protein_kcal_pct.toFixed(1)}% of kcal -> ${fat_support}.`;

  return {
    protein_support,
    fat_support,
    protein_description,
    fat_description,
  };
}
```

Map the function output to the schema exactly:

- `labels.protein_support = protein_support`
- `labels.fat_support = fat_support`
- `label_descriptions.protein_support = protein_description`
- `label_descriptions.fat_support = fat_description`

If the generated nutrition values are uncertain, still execute the same function using the generated numeric values. Express uncertainty through `nutrition_confidence` and `label_confidence`; do not change the label outside this function.

## Fatty Acid Profile

- `mct_rich`: coconut or MCT-rich foods.
- `omega_3_rich`: fatty fish or clear omega-3 evidence.
- `monounsaturated_rich`: olive oil, avocado, many nuts.
- `saturated_rich`: butter, cream, fatty red meat, coconut-heavy foods.
- null: evidence is unclear or no fatty acid profile clearly characterizes the food.

Fat Support gates Fatty Acid Profile.

`fat_support` answers whether the food is useful as a practical keto fat lever.

`fatty_acid_profile` answers whether a food that already provides meaningful fat support has a characteristic fat type.

When `fat_support = "limited"`, always set `fatty_acid_profile = null`, including for foods with clear omega-3 or MCT content. Do not assign any fatty-acid profile to a limited-fat-support food.

For fish and seafood:

- Use `omega_3_rich` when `fat_support` is moderate or strong and salmon, sardines, mackerel, herring, trout, tuna belly, or other fatty fish is clearly identified.
- Use `omega_3_rich` when `fat_support` is moderate or strong, nutrition estimates show `omega_3_g >= 0.5g` per 100g, and the item is fish or seafood.
- Use low fatty acid profile confidence for assorted sashimi or mixed fish platters when exact fish composition varies, but do not set the profile to null if omega-3 evidence is still meaningful.

For MCT-rich foods:

- Use `mct_rich` for MCT oil, coconut oil, coconut cream, coconut milk, or coconut meat when coconut or MCT is a meaningful part of the food.
- Do not use `mct_rich` for coconut flavor, small coconut garnish, or a mixed dish where coconut is minor or uncertain.
- Do not assign `mct_rich` when `fat_support = "limited"`; return `fatty_acid_profile = null`.

For monounsaturated-rich foods:

- Use `monounsaturated_rich` for olive oil, olives, avocado, macadamia nuts, almonds, pecans, or foods clearly dominated by these fat sources.
- Use `monounsaturated_rich` when total fat is meaningful and monounsaturated fat is clearly dominant.
- Do not use `monounsaturated_rich` only because monounsaturated fat is the largest subtype in an otherwise lean food.
- For mixed cooked dishes with unknown oil, prefer `fatty_acid_profile = null` or low confidence unless the oil or fat source is visually or contextually clear.

For saturated-rich foods:

- Use `saturated_rich` for butter, cream, high-fat cheese, coconut-heavy foods, palm oil-heavy foods, fatty red meat, pork belly, bacon, sausage, or other clearly saturated-fat-dominant foods.
- Use `saturated_rich` when total fat is meaningful and saturated fat is clearly high relative to total fat.
- Do not use `saturated_rich` for lean red meat, lean pork, skinless poultry, low-fat dairy, or small amounts of cheese or cream in a mixed dish.
- For mixed restaurant dishes where the fat source is unclear, prefer `fatty_acid_profile = null` or low confidence.

Do not assign a fatty acid profile only because one fat subtype is numerically the largest.

For eggs, mixed cooked dishes, or foods where the cooking fat is unknown, prefer `fatty_acid_profile = null` unless there is strong visual or contextual evidence.

## Fat Processing

- `whole_food`: intact foods such as eggs, fish, meat, avocado, plain vegetables, nuts, including simple cooked forms such as fried eggs, boiled eggs, grilled meat, or plain fish.
- `minimally_processed`: simple cooked mixed dishes or basic dairy where the food is still recognizable and not industrially processed.
- `processed`: packaged, refined, breaded, battered, sweetened, ultra-processed, or industrially prepared foods.

Use low label confidence when nutrition confidence is low, the food is a mixed restaurant dish, fat source is unclear, or label choice depends on preparation assumptions.

## Deterministic Validation Boundaries

Use deterministic code, not LLM judgement, for:

- `net_carb_g_per_100g = total_carb_g - fiber_g - sugar_alcohol_g`
- `calories_kcal_per_100g = protein_g * 4 + fat_g * 9 + net_carb_g_per_100g * 4 + sugar_alcohol_g * 2`
- final Carb Impact
- consumed nutrition by portion
- consumed calories by portion
- meal-level nutrition and calorie totals
- numeric validation
- enum validation
- display label rules

Final Carb Impact:

- `< 5g net carbs per 100g`: `very_low`
- `< 10g`: `low`
- `< 20g`: `moderate`
- otherwise: `high`

The code stage must check:

- all numeric nutrition values are non-negative when present;
- `fiber_g <= total_carb_g`;
- `fiber_g + sugar_alcohol_g <= total_carb_g`;
- `sugar_g <= total_carb_g`;
- `protein_g + fat_g + total_carb_g <= 100`;
- fat subtype sum does not exceed total fat with small tolerance;
- `omega_3_g <= polyunsaturated_fat_g` when both are present;
- `mct_g <= saturated_fat_g` when both are present;
- label enum values are valid.

MVP calorie formula used by code:

```text
calories_kcal_per_100g = protein_g * 4 + fat_g * 9 + net_carb_g_per_100g * 4 + sugar_alcohol_g * 2
```

Low confidence is not the same as validation failure.

## Fat Quality Output Rule

The model returns:

- `fat_processing`
- `fatty_acid_profile`

The deterministic output layer enforces:

- when `fat_support = "limited"`, set `fatty_acid_profile = null` and output only `fat_processing` as Fat Quality;
- when `fat_support = "moderate"` or `"strong"`, display `fat_processing` and also display `fatty_acid_profile` when it is not null;

## Label Conflict Precedence

Apply category-specific exclusions after all general fatty-acid rules. A specific exclusion overrides a generic numeric or food-family heuristic.

- For plain eggs, boiled eggs, poached eggs, scrambled eggs, or fried eggs with no explicit dominant cooking fat, set `fatty_acid_profile = null`.
- The word `fried` does not prove butter, coconut oil, palm oil, or another saturated-fat source.
- Do not use `saturated_rich` for eggs merely because saturated fat is the largest reported fat subtype.
- Low confidence does not make an otherwise unsupported fatty-acid profile acceptable. When evidence is insufficient, return null with the appropriate confidence.
