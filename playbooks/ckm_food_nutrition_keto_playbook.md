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

Output `nutrition_profile_stability` separately from `nutrition_confidence`:

- `stable`: the named food has a reasonably repeatable per 100g nutrition profile across ordinary servings, so one reusable food-library profile is appropriate. Examples include fried egg, boiled egg, plain cooked salmon, avocado, and a specific cheese type such as cheddar.
- `variable`: the same name can represent materially different ingredient ratios, added fats, sauces, sugar, starch, cuts, formulations, or preparation styles, so one reusable profile could be misleading. Examples include mixed green salad, bowl meal, homemade soup, generic cream sauce, generic beef without a cut or fat level, and mixed restaurant dishes.

Minor normal cooking variation alone does not make a common standardized food `variable`. Use `variable` when variation can materially change the macro profile or when the food name is too broad to select one representative profile.

`nutrition_profile_stability` describes the food profile, while `nutrition_confidence` describes confidence in the current numeric estimate. Evaluate and output both fields independently.

Do not copy or mechanically downgrade `nutrition_confidence` from the upstream `recognition_confidence`. The business system evaluates recognition confidence separately. For a specific stable food name such as `boiled egg`, nutrition confidence should remain high even when upstream recognition confidence is low. For a broad variable name such as `cooked beef` without a cut or fat level, nutrition confidence should be low unless reliable subtype, recipe, brand, or formulation details make the numeric estimate specific.

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

`fatty_acid_profile` is a single-choice field describing the highest-priority characteristic fat type supported by the evidence.

When `fat_support = "limited"`, always set `fatty_acid_profile = null`, including for foods with clear omega-3 or MCT content. Do not assign any fatty-acid profile to a limited-fat-support food.

When `fat_support` is `moderate` or `strong`, first evaluate which profiles meet their evidence standards, then return exactly one using this first-match priority:

1. `mct_rich`
2. `omega_3_rich`
3. `monounsaturated_rich`
4. `saturated_rich`
5. null when none qualifies

Priority is only a tie-break between profiles that independently qualify. It must not make an unsupported higher-priority profile eligible.

For fish and seafood:

- Use `omega_3_rich` when `fat_support` is moderate or strong and salmon, sardines, mackerel, herring, trout, tuna belly, or other fatty fish is clearly identified.
- Use `omega_3_rich` when `fat_support` is moderate or strong, nutrition estimates show `omega_3_g >= 0.5g` per 100g, and the item is fish or seafood.
- Use low fatty acid profile confidence for assorted sashimi or mixed fish platters when exact fish composition varies, but return `omega_3_rich` when omega-3 evidence is still meaningful and no qualifying MCT evidence exists.

For MCT-rich foods:

- Use `mct_rich` for MCT oil, coconut oil, coconut cream, coconut milk, or coconut meat when coconut or MCT is a meaningful part of the food.
- Do not use `mct_rich` for coconut flavor, small coconut garnish, or a mixed dish where coconut is minor or uncertain.
- Do not assign `mct_rich` when `fat_support = "limited"`; return `fatty_acid_profile = null`.

For monounsaturated-rich foods:

- Use `monounsaturated_rich` for olive oil, olives, avocado, macadamia nuts, almonds, pecans, or foods clearly dominated by these fat sources.
- Use `monounsaturated_rich` when total fat is meaningful and monounsaturated fat is clearly dominant.
- Do not use `monounsaturated_rich` only because monounsaturated fat is the largest subtype in an otherwise lean food.
- For mixed cooked dishes with unknown oil, do not select `monounsaturated_rich` unless the oil or fat source is visually or contextually clear.

For saturated-rich foods:

- Use `saturated_rich` for butter, cream, high-fat cheese, coconut-heavy foods, palm oil-heavy foods, fatty red meat, pork belly, bacon, sausage, or other clearly saturated-fat-dominant foods.
- Use `saturated_rich` when total fat is meaningful and saturated fat is clearly high relative to total fat.
- Do not use `saturated_rich` for lean red meat, lean pork, skinless poultry, low-fat dairy, or small amounts of cheese or cream in a mixed dish.
- For mixed restaurant dishes where the fat source is unclear, do not select `saturated_rich` unless the evidence supports it.

Do not assign a fatty acid profile only because one fat subtype is numerically the largest.

For eggs, mixed cooked dishes, or foods where the cooking fat is unknown, return null unless there is strong visual or contextual evidence for a profile.

## Fat Processing

- `whole_food`: intact foods such as eggs, fish, meat, avocado, plain vegetables, nuts, including simple cooked forms such as fried eggs, boiled eggs, grilled meat, or plain fish.
- `minimally_processed`: simple cooked mixed dishes or basic dairy where the food is still recognizable and not industrially processed.
- `processed`: packaged, refined, breaded, battered, sweetened, ultra-processed, or industrially prepared foods.

Use low label confidence when nutrition confidence is low, the food is a mixed restaurant dish, fat source is unclear, or label choice depends on preparation assumptions.

## About This Food Writer

After an item's per-100g nutrition and Keto labels are finalized, generate a
static food-level `about_this_food` description from those exact final fields.
The description is cached against the food and locale. It must remain true
regardless of serving amount, meal, user, date, or measured ketone response.

### Language

Write in the language selected by `output_locale`:

- `en-US`: English
- `zh-CN`: Simplified Chinese
- `de-DE`: German
- `fr-FR`: French
- `es-ES`: Spanish

Use the localized input `item_name` naturally. The prose must be entirely in
the selected language except for food names that are normally borrowed and
universal units such as g and mg. Machine enums remain English in their schema
fields but must never appear verbatim in user-facing prose.

### Evidence Boundary

- Use only the item's identity, `nutrition_relevant_cues`, generated per-100g
  nutrition, and generated labels.
- Do not mention serving amount, the rest of a meal, user context, user goals,
  measured ketones, or a predicted outcome for this meal or person.
- Do not recommend, warn, praise, criticize, or call a food healthy, unhealthy,
  good, bad, clean, junk, `keto-friendly`, or `keto-approved`.
- Do not invent nutrients, ingredients, preparation details, or physiology.
- Do not mention calories. Calories are computed by deterministic code after
  this LLM call and are not authoritative evidence for this writer.
- Do not expose implementation metadata such as confidence, stability,
  variability, caching, reuse, references, databases, model generation, or
  suitability for storage.

### Content Contract

Return exactly two short paragraphs through separate structured fields:

- `paragraph_1`: position what the food is, then cite only the per-100g macro
  facts needed to establish that position. Do not explain a label or
  fatty-acid profile in this paragraph.
- `paragraph_2`: explain exactly one most informative supplied label in natural
  language, then end with exactly one neutral sentence about the role this kind
  of food usually plays in a Keto meal.

Do not put line breaks inside either field. Do not output headings, bullets,
field names, enum names, or markdown. Do not list multiple support tiers,
processing classes, or fatty-acid labels. Natural expressions include `rich in
omega-3 fats`, `mostly monounsaturated fat`, and `primarily a protein food`.
Forbidden prose includes `protein_support`, `fat_support`, `omega_3_rich`,
`mct_rich`, `monounsaturated_rich`, `saturated_rich`, `whole_food`,
`minimally_processed`, and translated equivalents used as product labels.
Never translate a support field and tier literally. For example, do not write
`protein support is strong`, `strong protein support`, `蛋白质支持为强`, or
their equivalents in another language. State the underlying food fact
naturally, such as `it provides substantial protein per 100g` or
`每100g可提供较多蛋白质`.

### Fat-Quality Consistency

- When `fat_support = "limited"`, `fatty_acid_profile` must be null and the
  description must not state or imply omega-3-rich, MCT-rich,
  monounsaturated-fat-rich, or saturated-fat-rich. `fat_processing` may be
  explained only when it materially describes what the food is.
- When `fat_support` is `moderate` or `strong`, a supported non-null
  `fatty_acid_profile` may be explained. Do not force it into the description
  when another supplied label is more informative.

### Fiber And Micronutrients

- Fiber may explain net carbohydrate when total carbohydrate and fiber are
  present.
- Mention at most one micronutrient and only when its per-100g value reaches:
  potassium >= 400 mg, magnesium >= 80 mg, or sodium >= 500 mg.
- When a micronutrient is mentioned, state its per-100g numeric value. Never
  round up to reach a threshold. Null means omit.

### Length And Final Check

- `en-US`, `de-DE`, `fr-FR`, `es-ES`: target 42-60 words, hard maximum 70.
- `zh-CN`: target 70-130 Chinese characters, hard maximum 140 Chinese
  characters excluding punctuation.

A shorter complete description is better than padded text. Before returning
each item, verify that both paragraph fields match that same item's final
nutrition and labels, explain only one supplied label, contain one general
Keto-role sentence, and obey the selected language and hard length limit.

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
