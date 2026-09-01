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

Extract amounts and concise lowercase English canonical `normalized_name` values from the text for each locked item, using locale only for the identity tie-break above. Then render each item one-to-one as frontend-facing `item_name`; localization must never merge, split, add, or remove food items. Both names must preserve the same food identity, major ingredients, and nutrition-relevant preparation stated by the user. Keep `nutrition_relevant_cues` in concise English. Preserve `source_text_span` in the original input language. For `en-US`, `de-DE`, `fr-FR`, and `es-ES`, start `item_name` with an uppercase letter and use natural sentence-style casing, never Title Case for every word. Preserve required local capitalization such as German nouns. For `zh-CN`, use natural Simplified Chinese naming.

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

## Post-Extraction Name Normalization

Extract the food identity, major ingredients, preparation, and amount from the user text before consulting this vocabulary. Then normalize only semantically compatible wording.

Rules:

- Never remove a user-stated ingredient or preparation detail to match a preferred name.
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

These mappings normalize only wording, number, or presentation state. Preserve modifiers such as `with cheese`, `with chicken`, `breaded`, `battered`, `fried`, `roasted`, `smoked`, `sweetened`, `in oil`, `with dressing`, and `with cream sauce`. Do not map a food into a family merely because it belongs to the same category.

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

Use a practical generic food or dish name rather than a brand-heavy product title. Preserve preparation or subtype in the name only when it materially affects nutrition; retain secondary visible or stated details in concise context fields when available.

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

Use `nutrition_relevant_cues` only for a short nutrition-relevant detail that cannot be represented reliably in the practical food name. If an explicit ingredient, preparation, sauce, coating, or formulation is already present in `item_name` and `normalized_name`, do not repeat it in cues. Do not use item count, quoted amount, or the food name itself as a cue. For explicit, high-confidence text with a sufficiently specific food name, return `nutrition_relevant_cues = []`.

## Error Codes

The validation code may return:

- `no_food_text_detected`
- `missing_required_amount`
- `product_package_without_consumed_amount`
- `ambiguous_food_text`
- `asr_text_unusable`
- `unsupported_input_type`
