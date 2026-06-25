# CKM Food AI Playbook v0.1

## Scope

This playbook is used by the CKM internal meal image benchmark workflow.

It covers:

- dish-first food recognition;
- per 100g nutrition estimation;
- keto profile label generation;
- deterministic validation boundaries.

Use the section relevant to the current Dify node. Do not apply rules from another section when they conflict with the node's responsibility.

## Global Rules

Prefer dish-level recognition before ingredient-level recognition.

For a named dish, treat the dish as the nutrition unit unless decomposition is explicitly required.

User context may help identify the dish, cuisine, serving style, or rough portion. User context must not change nutrition facts, keto label thresholds, or validation rules.

Return only the JSON required by the Dify node schema. Do not include markdown or commentary in model outputs.

## A. Recognition Rules

Use in the food recognition LLM node.

Identify food items from meal images.

Prefer a whole dish when a common dish name is likely.

Examples:

- If the image likely shows a cheeseburger, output `cheeseburger` as one dish.
- Do not output bun + beef patty + cheese + lettuce + tomato unless decomposition is explicitly required.
- If the image likely shows pepperoni pizza, output `pepperoni pizza` as one dish.
- Do not output crust + tomato sauce + cheese + pepperoni unless decomposition is explicitly required.
- If the image shows clearly separated grilled steak, asparagus, and mashed potatoes, output separate items.
- If the image shows a barbecue platter, buffet plate, or mixed visible components, use `meal_set` or separated items depending on visual separation.

Item types:

- `dish`: cooked or prepared dish treated as one nutrition unit.
- `simple_food`: single recognizable food such as boiled egg, apple, avocado, plain rice.
- `ingredient`: visible raw or component ingredient that is not itself the dish.
- `meal_set`: multiple foods presented together where no single dish name is adequate.
- `unknown`: image is too ambiguous.

Set `should_decompose = false` when a dish name is available, the food is mixed/prepared as one dish, or ingredient quantity cannot be reliably estimated.

Set `should_decompose = true` only when foods are clearly separated, no dish name is reasonably possible, the input is a platter/meal set, or benchmark mode asks for decomposition.

For each returned item, `should_decompose` means whether that returned item should be decomposed further. Most returned items should have `should_decompose = false`.

If `should_decompose = false`, set `decomposition_reason = null`.

Estimate portion only when visual evidence supports a rough grams/ml estimate. Use `g` for solid food and `ml` for liquid.

Do not use `piece` for `unit` in this workflow. If the image shows two fried eggs, estimate their total grams and mention the count in `visible_evidence`.

Use `estimated_amount = null` and `unit = "unknown"` when scale is unclear.

## B1. Nutrition Rules

Use in the nutrition per 100g LLM node.

Generate estimated raw nutrition values per 100g only.

Do not calculate net carbs.

Do not output final Carb Impact.

Use the common cooked or served form unless the item name or context clearly indicates otherwise.

For dishes, estimate the average prepared dish per 100g, including typical cooking oil, sauce, moisture, and preparation style when relevant.

Examples:

- Cheeseburger should be estimated as the prepared dish per 100g, including typical bun, beef patty, cheese, condiments, and vegetables when visible or typical.
- Pepperoni pizza should be estimated as the prepared dish per 100g, including typical crust, tomato sauce, cheese, and pepperoni.
- Macaroni and cheese should be estimated as the prepared dish per 100g, including typical pasta, cheese sauce, and moisture assumptions.
- Plain `chicken breast` should use cooked plain chicken breast if the image shows cooked chicken.

Macronutrients should usually be present:

- `calories_kcal`
- `protein_g`
- `fat_g`
- `total_carb_g`
- `fiber_g`
- `sugar_g`
- `sugar_alcohol_g`

Micronutrients and detailed fat fields may be null when not reasonably knowable. Do not invent micronutrients only to make the output look complete.

Set `nutrition_confidence = "high"` when the food is simple, standardized, or has stable common per 100g values.

Set `nutrition_confidence = "low"` when recipe, oil, sauce, batter, sugar, starch, restaurant variation, or dish identity is uncertain.

## B2. Keto Label Rules

Use in the keto label generation LLM node.

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
- reasons
- confidence

Protein Support:

- `strong`: practical protein source with high protein density and reasonable calorie efficiency.
- `moderate`: meaningful protein contribution but not primarily a protein source.
- `limited`: protein is low, incidental, or impractical as a protein target.

Fat Support:

- `strong`: high fat density and practical as a fat lever with limited carb burden.
- `moderate`: meaningful fat but not dominant, or comes with significant protein load.
- `limited`: low fat or not useful as a fat lever.

Fatty Acid Profile:

- `mct_rich`: coconut or MCT-rich foods.
- `omega_3_rich`: fatty fish or clear omega-3 evidence.
- `monounsaturated_rich`: olive oil, avocado, many nuts.
- `saturated_rich`: butter, cream, fatty red meat, coconut-heavy foods.
- null: evidence is unclear or no fatty acid profile clearly characterizes the food.

Fat Support and Fatty Acid Profile are related but not identical.

`fat_support` answers whether the food is useful as a practical keto fat lever.

`fatty_acid_profile` answers whether the food has a characteristic fat type.

Therefore, a food can have `fat_support = "limited"` and still have `fatty_acid_profile = "omega_3_rich"` when fatty fish is clearly present or nutrition estimates show meaningful omega-3 content.

For fish and seafood:

- Use `omega_3_rich` when salmon, sardines, mackerel, herring, trout, tuna belly, or other fatty fish is clearly identified.
- Use `omega_3_rich` when B1 estimates `omega_3_g >= 0.5g` per 100g and the item is fish or seafood.
- Use low fatty acid profile confidence for assorted sashimi or mixed fish platters when exact fish composition varies, but do not set the profile to null if omega-3 evidence is still meaningful.

For MCT-rich foods:

- Use `mct_rich` for MCT oil, coconut oil, coconut cream, coconut milk, or coconut meat when coconut or MCT is a meaningful part of the food.
- Do not use `mct_rich` for coconut flavor, small coconut garnish, or a mixed dish where coconut is minor or uncertain.
- `mct_rich` may be shown even when `fat_support = "limited"` if MCT evidence is clear, but this should be uncommon.

For monounsaturated-rich foods:

- Use `monounsaturated_rich` for olive oil, olives, avocado, macadamia nuts, almonds, pecans, or foods clearly dominated by these fat sources.
- Use `monounsaturated_rich` when B1 estimates meaningful total fat and monounsaturated fat is clearly dominant.
- Do not use `monounsaturated_rich` only because monounsaturated fat is the largest subtype in an otherwise lean food.
- For mixed cooked dishes with unknown oil, prefer `fatty_acid_profile = null` or low confidence unless the oil or fat source is visually or contextually clear.

For saturated-rich foods:

- Use `saturated_rich` for butter, cream, high-fat cheese, coconut-heavy foods, palm oil-heavy foods, fatty red meat, pork belly, bacon, sausage, or other clearly saturated-fat-dominant foods.
- Use `saturated_rich` when B1 estimates meaningful total fat and saturated fat is clearly high relative to total fat.
- Do not use `saturated_rich` for lean red meat, lean pork, skinless poultry, low-fat dairy, or small amounts of cheese or cream in a mixed dish.
- For mixed restaurant dishes where the fat source is unclear, prefer `fatty_acid_profile = null` or low confidence.

Do not assign a fatty acid profile only because one fat subtype is numerically the largest. The profile should describe a clear food-level characteristic.

For eggs, mixed cooked dishes, or foods where the cooking fat is unknown, prefer `fatty_acid_profile = null` unless there is strong visual or contextual evidence.

Fat Processing:

- `whole_food`: intact foods such as eggs, fish, meat, avocado, plain vegetables, nuts, including simple cooked forms such as fried eggs, boiled eggs, grilled meat, or plain fish.
- `minimally_processed`: simple cooked mixed dishes or basic dairy where the food is still recognizable and not industrially processed.
- `processed`: packaged, refined, breaded, battered, sweetened, ultra-processed, or industrially prepared foods.

Use low label confidence when B1 nutrition confidence is low, the food is a mixed restaurant dish, fat source is unclear, or label choice depends on preparation assumptions.

## Validation Boundaries

Use deterministic code, not LLM judgement, for:

- `net_carb_g_per_100g = total_carb_g - fiber_g`
- final Carb Impact
- consumed nutrition by portion
- numeric validation
- enum validation

Final Carb Impact:

- `< 5g net carbs per 100g`: `very_low`
- `< 10g`: `low`
- `< 20g`: `moderate`
- otherwise: `high`

The Code Node must check:

- all numeric nutrition values are non-negative when present;
- `fiber_g <= total_carb_g`;
- `sugar_g <= total_carb_g`;
- `protein_g + fat_g + total_carb_g <= 100`;
- fat subtype sum does not exceed total fat with small tolerance;
- `omega_3_g <= polyunsaturated_fat_g` when both are present;
- `mct_g <= saturated_fat_g` when both are present;
- label enum values are valid;
- calorie estimate is within MVP tolerance.

MVP calorie formula:

```text
calories ~= protein_g * 4 + fat_g * 9 + net_carb_g * 4 + sugar_alcohol_g * 2
```

MVP tolerance:

```text
max(20 kcal, 15% of declared calories)
```

Low confidence is not the same as validation failure. Low confidence should be recorded for review and benchmark analysis.
