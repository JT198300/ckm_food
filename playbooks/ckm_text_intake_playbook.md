# CKM Text Intake Playbook v0.1

## Scope

Use this playbook only in the text intake stage.

The text intake stage receives meal text from:

- direct user text input;
- ASR-transcribed voice input;
- food text extracted from screenshots by the image intake router;
- food log text extracted from other diet, calorie, nutrition, or fasting apps.

It converts text into a standardized food-and-amount JSON structure.

Do not generate nutrition values, keto labels, carb impact, or food database records.

## Human-Edibility Boundary

Only return an intake item when it represents something that exists in the real world and is ordinarily intended or accepted for human consumption as food or drink.

A user's assertion that something was eaten does not make it eligible. Do not return fictional foods or creatures, human tissue, feces or other bodily waste, or any other non-food substance as an intake item. Do not relabel excluded content as `unknown` or `other_food`.

When eligible food or drink appears together with excluded content, keep only the eligible items and add this generic warning without repeating the excluded content:

`Some input content was excluded because it is not a real, ordinarily edible food or drink.`

If no eligible food or drink remains, return an empty `intake_items` array. The validation layer will return `no_food_text_detected`.

## Output Locale

The supported `output_locale` values are:

- `en-US`
- `zh-CN`
- `de-DE`
- `fr-FR`
- `es-ES`

Understand meal text regardless of its source language. Lock explicit food spans and item boundaries before consulting `output_locale`. Then treat `output_locale` as a weak cultural interpretation and naming prior only when one locked item supports multiple similarly plausible food identities. In that limited tie-break case, prefer the common food or dish interpretation used in the requested locale. Locale must never override an explicit food name, ingredient, preparation, amount, or source-language meaning; it must not change the locked item count or boundaries or increase confidence. A meal may come from any cuisine regardless of locale.

```python
locked_items = parse_food_items_from_source_text(meal_text)  # locale-independent
for item in locked_items:
    candidates = interpret_item_from_source_text(item)
    identity = locale_tiebreak(candidates, output_locale) if similarly_plausible(candidates) else candidates[0]
    render_localized_item_name(identity, output_locale)
```

Extract amounts and concise lowercase English canonical `normalized_name` values from the text for each locked item, using locale only for the identity tie-break above. Then render each item one-to-one as frontend-facing `item_name`; localization must never merge, split, add, or remove food items. Both names must preserve the same food identity and major ingredients stated by the user. Apply the category-specific state and preparation rules below instead of requiring every preparation word to appear in both names. Keep `nutrition_relevant_cues` in concise English. Preserve `source_text_span` in the original input language. For `en-US`, `de-DE`, `fr-FR`, and `es-ES`, start `item_name` with an uppercase letter and use natural sentence-style casing, never Title Case for every word. Preserve required local capitalization such as German nouns. For `zh-CN`, use natural Simplified Chinese naming.

## Food Category

Assign exactly one `food_category` to every returned intake item. The category is a stable icon/index key and must not replace or simplify the extracted food name.

Use only these 19 food categories:

- `eggs`: Eggs / 蛋类
- `milk_and_dairy`: Milk & Dairy / 奶及乳制品
- `meat`: Meat / 肉类
- `fish_and_seafood`: Fish & Seafood / 鱼类及海鲜
- `beans_and_soy`: Beans & Soy / 豆类及豆制品
- `vegetables`: Vegetables / 蔬菜
- `fruit`: Fruit / 水果
- `rice_and_grains`: Rice & Grains / 米饭及谷物
- `noodles_and_pasta`: Noodles & Pasta / 面条及意面
- `bread_and_flatbread`: Bread & Flatbread / 面包及饼类
- `potatoes_and_starchy_roots`: Potatoes & Starchy Roots / 薯类
- `nuts_and_seeds`: Nuts & Seeds / 坚果及种子
- `salad`: Salad / 沙拉
- `soups_and_stews`: Soups & Stews / 汤与炖菜
- `solid_mixed_meals`: Solid Mixed Meals / 固体复合餐
- `snacks_and_desserts`: Snacks & Desserts / 零食及甜点
- `drinks`: Drinks / 饮品
- `condiments_and_oils`: Condiments & Oils / 调味品及油脂
- `other_food`: Other Food / 其他食物

Assign `food_category` only after the returned food items have already been identified. Classification must not split, merge, rename, add, or omit food items.

Classify by the practical identity of each returned item. Prefer a specific food category when it clearly defines the item. A vegetable-named, vegetable-dominant dish remains `vegetables` when meat, cheese, or sauce is only an accompaniment. An item with a clear retained staple base remains in that base category when another food is only a topping or accompaniment, such as rice with egg → `rice_and_grains`, pasta with sauce → `noodles_and_pasta`, or toast with cheese → `bread_and_flatbread`. Use `salad` for every named salad, including tuna salad, egg salad, chicken salad, and potato salad. Use `soups_and_stews` only when the returned item itself is a soup, broth, chowder, hot pot, liquid curry, stew, or another dish in which consumable liquid is a defining part of the item. A single meat item with cooking or braising liquid may remain `meat`; do not recombine separately returned meat, egg, or vegetable items merely to create a soup. Use `solid_mixed_meals` for customary cohesive combination dishes such as burgers, sandwiches, pizza, wraps, mixed rice bowls, fried rice, and casseroles, or when no more specific category defines the item. Use `other_food` only when an edible item cannot be reliably assigned to any of the other 18 categories.

Edible insects such as crickets or grasshoppers are `other_food`, not meat or seafood.

## Category-Specific Name And State Contract

Apply naming semantics only after food identity, item boundaries, `item_type`, and `food_category` are independently determined. Use this precedence so category labels never erase dish identity:

1. A cohesive dish follows the dish rule, even when its `food_category` is `vegetables`, `eggs`, `meat`, or another ingredient category.
2. A simple animal-protein food or ingredient follows the animal-protein state rule.
3. A simple vegetable or vegetable ingredient follows the vegetable state rule.
4. A standardized prepared product follows the conventional-product rule.
5. Other foods retain a concise practical canonical identity supported by the source text.

`item_name` is localized for display. `normalized_name`, `food_category`, and `nutrition_relevant_cues` are stable machine semantics and must remain one-to-one with that display item.

### Cohesive-Dish Naming Hard Rule

For `item_type = "dish"`, `normalized_name` must retain the full practical dish identity rather than collapse to a generic component. For example, use `tomato scrambled eggs`, not `scrambled eggs`, and `chicken curry`, not `chicken`. Keep material ingredients or preparation already expressed by the dish name in the name. Use `nutrition_relevant_cues` only for material nutrition drivers stated by the user but not already expressed by either name.

### Raw/Cooked Animal-Protein Naming Hard Rule

Apply this rule to simple animal-protein foods and ingredients, including meat, poultry, fish, and shellfish. This is a mandatory output constraint, not an example set or a suggestion.

- When the source explicitly says the food is raw, `normalized_name` must be `raw <base food>`.
- When the source explicitly says the food is cooked but gives no supported method, `normalized_name` must be `cooked <base food>`.
- A specific cooking method may replace `cooked` only when the text explicitly supports it and the exact method is in the global allowlist: `grilled`, `roasted`, `boiled`, `steamed`, `fried`, or `braised`.
- Collapse unsupported or overly specific method words such as `pan-fried`, `pan-seared`, `seared`, `sauteed`, `poached`, `air-fried`, or `barbecued` to `cooked <base food>` unless a food-specific allowed-name combination explicitly permits that exact name.
- Never remove all state meaning during normalization. For example, do not map `raw shrimp` to `shrimp`, `grilled lamb chops` to `lamb chops`, or `cooked salmon` to `salmon`.
- For `en-US`, `item_name` must use the same allowed state-bearing food name as `normalized_name`, with natural sentence casing. For another `output_locale`, `item_name` must be its direct localized semantic equivalent.
- If the source does not state or support raw versus cooked state, keep the practical base food name and do not invent a state.

Food-specific allowed-name combinations override the global method allowlist. They are output constraints only and must never be used as identity candidates. Independently identify the base food first. Never promote generic fish, white fish, or another uncertain protein to salmon merely because salmon has an allowed-name list.

- `salmon`: `raw salmon`; `cooked salmon`

### Simple-Vegetable State Naming Hard Rule

Apply this rule only to simple vegetable foods and vegetable ingredients, not to cohesive dishes.

- `normalized_name` must be the lowercase English base vegetable identity, independent of whether the current food is raw or cooked. Use `spinach`, `chayote`, `tomato`, or `napa cabbage`, not `raw spinach`, `cooked chayote`, or `steamed napa cabbage`.
- When raw state is stated or clearly supported, include exactly one `raw` cue in `nutrition_relevant_cues`. When cooked state is stated or clearly supported, include exactly one `cooked` cue. Do not include both.
- State cue control words are exactly `raw` and `cooked`; collapse individual cooking-method words such as steamed, boiled, roasted, grilled, fried, or sauteed to `cooked` for a simple vegetable.
- If raw versus cooked state is genuinely uncertain, do not invent a state cue. Keep the base vegetable identity, set `recognition_confidence = "low"`, and record the uncertainty in `ambiguities`.
- `item_name` may naturally preserve the stated raw or cooked wording in `output_locale`, but the machine `normalized_name` remains the base vegetable identity.
- `estimated_amount` is the current described-state weight. A `raw` cue means the amount represents raw edible weight; a `cooked` cue means it represents cooked edible weight.

### Standardized Prepared-Product Naming Hard Rule

For a stable, conventionally named prepared product such as bread, rye bread, toast, plain yogurt, or a specific cheese, use its conventional ready-to-eat canonical identity without a redundant raw/cooked modifier. Minor toasting, seeds, moisture, brand, or similar ordinary variation does not create a cue merely to alter nutrition. For example, `toasted rye bread` normalizes to `rye bread` with no `toasted` cue. Preserve a distinct stable product identity when the source supports one; keep material separate additions as separate items or retain a cohesive combination as a dish.

## Post-Extraction Name Normalization

Extract the food identity, major ingredients, preparation, and amount from the user text before consulting this vocabulary. Then normalize only semantically compatible wording.

Rules:

- Never remove a user-stated major ingredient. Handle preparation wording according to the category-specific contract above.
- Preserve every major ingredient that materially changes nutrition. For example, `mixed green salad with cheese` and `mixed green salad with chicken` remain different names.
- If no family fits, keep a concise practical name derived from the text.
- Treat the conservative canonical name families below as lowercase `normalized_name` values used for lookup and matching.
- Output `item_name` as a natural frontend-facing food name in `output_locale`, using locale-appropriate naming and casing.

Canonical name families:

- `fried eggs` <- fried egg; sunny-side-up egg; sunny-side-up eggs
- `scrambled eggs` <- scrambled egg
- `boiled eggs` <- boiled egg; hard-boiled egg; hard-boiled eggs
- `avocado` <- avocado half; avocado slices; sliced avocado
- `green olives` <- green olive
- `red bell pepper` <- red bell peppers; sliced red bell pepper
- `cherry tomatoes` <- cherry tomato
- `tomato` <- tomatoes; tomato slices; sliced tomato
- `lettuce` <- lettuce leaves
- `red onion` <- red onion slices; sliced red onion
- `onion` <- onion slices; sliced onion
- `radishes` <- radish; radish slices; sliced radishes
- `spinach` <- spinach leaves
- `celery` <- celery sticks
- `cucumber` <- cucumber slices; sliced cucumber; raw cucumber
- `carrots` <- carrot; carrot slices; sliced carrots
- `strawberries` <- strawberry
- `blueberries` <- blueberry
- `raspberries` <- raspberry
- `blackberries` <- blackberry
- `butter` <- butter pat; butter pats
- `cottage cheese` <- cottage cheese curds
- `feta cheese` <- feta
- `mayonnaise` <- mayo
- `mct oil` <- MCT oil
- `olive oil` <- extra virgin olive oil; EVOO
- `walnuts` <- walnut
- `pecans` <- pecan
- `hazelnuts` <- hazelnut
- `white rice` <- cooked white rice; steamed white rice
- `toast` <- toasted bread
- `grilled chicken breast` <- grilled chicken breasts
- `pork chops` <- pork chop
- `crab` <- crab meat
- `sardines` <- sardine
- `french fries` <- fries

These mappings normalize only wording, number, or presentation state. Preserve material modifiers such as `with cheese`, `with chicken`, `breaded`, `battered`, `smoked`, `sweetened`, `in oil`, `with dressing`, and `with cream sauce`, except where the category-specific contract places state in a cue or treats a preparation as minor product variation. Do not map a food into a family merely because it belongs to the same category.

## Output Contract

Return only the JSON required by the Dify node schema.

Output a top-level object:

```json
{
  "intake_items": [],
  "intake_warnings": []
}
```

## Food Parsing Rules

Extract food names, dish names, meal sections, quantities, units, and text spans.

Prefer dish-level recognition before ingredient-level decomposition, but keep names specific enough for nutrition lookup.

For a named cohesive dish, output the dish as one item. Do not split its integrated ingredients merely because they have different nutrition profiles. Tomato scrambled eggs, ham and cheese omelet, chicken curry, cheeseburger, lasagna, pizza, soup, and chicken salad with avocado are each one dish when described as such.

Use the same practical nutrition-unit test across all food categories:

- Split foods only when the text presents them as independently eaten, independently portioned, or separately quantified foods, such as steak with mushrooms and a side salad.
- Keep ingredients combined by cooking, mixing, filling, wrapping, baking, or assembly as one dish.
- If a cohesive dish name is too generic for nutrition estimation, retain at most one or two defining nutrition drivers in the name and put remaining material details not already expressed by the name into concise cues.
- Nutrition drivers include dominant protein, major starch or grain base, high-fat additions, substantial cheese, avocado, nuts or seeds, creamy or oil-heavy sauces, breading or batter, and preparation details that materially change nutrition.
- Output a subtype only when it materially changes nutrition or keto interpretation and the text provides enough evidence.
- When subtype evidence is insufficient, use a useful generic class rather than guessing.
- Do not output both a parent dish and its integrated ingredients when that double-counts the same intake.
- Do not add category-specific extraction rules in response to individual benchmark failures.

Use a practical generic food or dish name rather than a brand-heavy product title. Preserve preparation or subtype in the machine field specified by the category-specific contract; retain secondary stated details in concise context fields when available.

## Food Name Capitalization

- Output `item_name` as a natural frontend-facing food name in `output_locale`, using locale-appropriate naming and casing.
- Preserve conventional capitalization for proper names and acronyms, such as `Greek yogurt`, `Caesar salad`, `Brussels sprouts`, `MCT oil`, and `BLT sandwich`.
- Do not use Title Case for every word. Prefer `Smoked salmon with cream cheese`, not `Smoked Salmon With Cream Cheese`.
- Output lowercase English `normalized_name` explicitly. The validation layer normalizes its whitespace and casing but must not derive it from localized `item_name`.
- Capitalization must not change the food identity, preparation, subtype, or item boundaries extracted from the source text.

For sauces and added fats, keep physically integrated sauce within a cohesive dish. Treat a separately listed or separately quantified sauce, dip, dressing, or added fat as its own item when nutritionally meaningful.

If the text contains meal sections such as breakfast, lunch, dinner, or snack, keep the food items but do not create a fake food item for the meal section itself.

Ignore non-food UI text, button labels, tab names, calorie rings, charts, app navigation, exercise rows, fasting timers, advertisements, and body metrics unless they are directly attached to food rows.

## Amount Rules

Every returned food item must include an `estimated_amount` and a form-appropriate `unit`.

`estimated_amount` always represents the food in its currently described state. For example, an amount attached to raw spinach is raw edible weight, while an amount attached to cooked spinach is cooked edible weight. Do not reinterpret a current-state amount as a pre-cooking or post-cooking equivalent.

Use:

- `g` for solid food;
- `ml` for liquids.

Do not use `piece`, `serving`, `cup`, `tbsp`, `egg`, `slice`, or household units in `unit`. Convert them to rough grams/ml.

If the text provides an explicit weight, volume, count, fraction, household portion, or serving container, convert it to a reasonable grams/ml estimate and set `amount_source = "explicit_text"`.

HARD TEXT AMOUNT RULE: An explicit count, fraction, household portion, or serving container is sufficient amount evidence and must receive a reasonable total grams/ml estimate. It must not return zero, null, or unknown merely because conversion is approximate. A count is not metric mass or volume: never copy the count numeral directly into `estimated_amount` and relabel it as `g` or `ml`. Estimate the total metric amount represented by all counted items. For example, `2 burgers` or `2个汉堡` must never become `2g`; estimate the combined weight of two burgers. Likewise, `1 egg` must receive a representative one-egg weight rather than `1g`.

Examples:

- `115g beef` -> `estimated_amount = 115`, `unit = "g"`, `amount_source = "explicit_text"`.
- `30 ml cream` -> `estimated_amount = 30`, `unit = "ml"`, `amount_source = "explicit_text"`.
- `2 fried eggs` -> estimate about `100g`, `amount_source = "explicit_text"` because count was explicit.
- `half avocado` -> estimate about `70g`, `amount_source = "explicit_text"` because fraction was explicit.
- `one bowl of rice` -> estimate a common bowl portion in grams and use `amount_source = "explicit_text"` because the serving container was explicit.
- `one cup of coffee` -> estimate a common cup volume in ml and use `amount_source = "explicit_text"` because the serving container was explicit.

An amount must be inferred from an explicit portion expression such as `one egg`, `half an avocado`, `one slice`, `one bowl`, `one cup`, or `two bites`. The wording is explicit evidence even though conversion to grams/ml is approximate. Do not output zero merely because the household portion lacks an exact metric size.

Final amount check: when the source span contains an explicit count or household portion but no explicit metric amount, `estimated_amount` must not equal the raw count numeral. Re-estimate the total grams/ml before returning.

If the text only states that a food or drink was consumed and contains no weight, volume, count, fraction, household portion, serving container, or other amount evidence, do not invent a default serving. Keep the recognized item and return `estimated_amount = 0`, use `ml` for liquids or `g` for solids, and set `amount_source = "unknown"`. Zero means the business system must ask the user for an amount; it is not an estimated consumed amount.

## Product And Package Rules

Product package net weight, product card size, package specification, shopping quantity, and nutrition label serving size are not consumed amounts.

Do not treat `1kg walnuts`, `500g bag`, `serving size 30g`, or product display weight as intake unless the text explicitly says the user ate or logged that amount.

If only product/package information is present without consumed amount, the text intake service should fail with `product_package_without_consumed_amount`.

## Source Tracking

Use `source_type = "text"` for direct user text, ASR text, and text extracted from screenshots after it enters this service.

Use `source_text_span` for the exact text fragment supporting the item whenever possible.

Use `source_image_index = null` and `source_image_filename = null` in this text service.

## Confidence

Set `recognition_confidence = "high"` when food identity and amount are explicit or clear.

Set `recognition_confidence = "low"` when:

- amount is estimated from a vague serving phrase;
- food identity is ambiguous;
- text is OCR-like, incomplete, multilingual, or noisy;
- product text may be confused with consumed amount;
- rows from a food log are partially cut off.

Low confidence is not a failure when the output contract is complete.

## Nutrition-Relevant Cues

Use `nutrition_relevant_cues` only for a short nutrition-relevant detail that cannot be represented reliably in the practical food name. If an explicit ingredient, preparation, sauce, coating, or formulation is already present in `item_name` and `normalized_name`, do not repeat it in cues. The controlled `raw` or `cooked` cue for a simple vegetable is a required exception. Do not use item count, quoted amount, or the food name itself as a cue. For explicit, high-confidence text with a sufficiently specific food name, return `nutrition_relevant_cues = []` unless the simple-vegetable state rule requires one state cue.

## Error Codes

The validation code may return:

- `no_food_text_detected`
- `missing_required_amount`
- `product_package_without_consumed_amount`
- `ambiguous_food_text`
- `asr_text_unusable`
- `unsupported_input_type`
