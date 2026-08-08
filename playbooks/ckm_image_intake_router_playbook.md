# CKM Image Intake Router Playbook v0.2

## Scope

Use this playbook only in the image intake router stage.

The image router receives one or more images and must decide whether the images already contain complete food-and-amount intake information, contain food text that should be handed to the text intake service, or contain no usable meal intake content.

Do not generate nutrition values, keto labels, carb impact, or food database records.

## Output Locale

The supported `output_locale` values are:

- `en-US`
- `zh-CN`
- `de-DE`
- `fr-FR`
- `es-ES`

Understand image content and visible text regardless of source language. First determine item boundaries, amounts, and concise lowercase English canonical `normalized_name` values without using locale. Then translate each item one-to-one into frontend-facing `item_name`; translation must never merge, split, add, or remove food items. Both names must preserve the same food identity, major ingredients, and nutrition-relevant preparation. Keep `nutrition_relevant_cues` in concise English. Preserve screenshot text in its original language when returning `extracted_text`. For `en-US`, use sentence case for `item_name`; for other locales, use normal local capitalization.

## Food Category

Assign exactly one `food_category` to every returned intake item. This field is a stable icon/index key, not a substitute for `item_name` and not a reason to make the food name more generic.

Use only these 17 business categories, mapped from `03_大类Icon分类` in `高频食物_通用食物分类版.xlsx`:

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
- `soup_stew_and_mixed_meals`: Soup, Stew & Mixed Meals / 汤、炖菜及混合餐
- `snacks_and_desserts`: Snacks & Desserts / 零食及甜点
- `drinks`: Drinks / 饮品
- `condiments_and_oils`: Condiments & Oils / 调味品及油脂

Classify by the practical identity of the returned item. Use `salad` for named salads and `soup_stew_and_mixed_meals` for cohesive mixed dishes that do not belong to a more specific category. Every edible item must use the closest defensible business category. `Other Food` is an internal fallback icon in the source workbook, not a runtime output value. Do not change item boundaries, confidence, or nutrition cues merely to fit a category.

## Output Modes

Return exactly one top-level routing result:

- `completed_food_intake`: direct meal photo recognition is complete and `intake_items` can be used by the next nutrition stage.
- `extracted_food_text`: the image is primarily a screenshot or text image containing food intake text. Put the extracted text in `extracted_text`; do not convert it to `intake_items` here.
- `failed`: no usable meal intake information is present, or the image type is unsupported without additional user input.

## Input Classes

Use one of these `input_class` values:

- `meal_photo`: a real food or meal photo where food and portions are visible.
- `food_text_screenshot`: a screenshot or image that primarily contains user-entered meal text.
- `food_log_screenshot`: a screenshot from another food, diet, calorie, or nutrition tracking app with logged food entries.
- `fasting_screenshot`: a fasting timer/status screen without meal intake content.
- `product_or_package`: a product, package, product card, nutrition label, or shopping page.
- `non_food_image`: logo, chart, body metric page, dashboard, empty UI screen, or unrelated image.
- `uncertain`: image type is unclear.

## Priority Routing Patch

Apply this order before the detailed rules below:

```python
def route_image(image):
    if readable_food_log_or_meal_text(image):
        return extracted_food_text()
    candidates = detect_visible_intake_candidates(image)
    edible = [x for x in candidates if not supplement_medication_or_energy_shot(x)]
    if edible:
        return completed_food_intake(classify_food_input(edible), edible)
    if candidates and all(supplement_medication_or_energy_shot(x) for x in candidates):
        return failed("unsupported_input_type")
    return failed_after_final_food_evidence_check()
```

Food may occupy only a small image region. A clear edible region, food package, grocery item, or food-log text takes precedence over non-food background. When exact subtype is unclear but a useful generic food class exists, return that class with low confidence instead of failing.

Supplement pills, capsules, vitamin products, creatine products, electrolyte supplements, medication, and energy shots are never food intake items. If supplements appear beside edible food or an edible ingredient, omit the supplements and keep the edible items. Return `failed` with `unsupported_input_type` only when no edible item remains. Food ingredients such as butter, edible oils, MCT oil, and visibly prepared protein or meal-replacement drinks remain valid intake.

## Meal Photo Rules

For `meal_photo`, identify visible food items and output `result_type = "completed_food_intake"`.

If a meaningful portion of the image clearly shows plated or served food, classify it as `meal_photo` even when people, furniture, packaging, or other non-food background occupies more of the frame.

Determine intended intake scope before extracting items. Prioritize food that is plated, served, opened for consumption, or explicitly presented as the meal. Do not automatically include background packages, storage containers, decorative food, or unrelated edible objects merely because they are visible.

Prefer dish-level recognition before ingredient-level decomposition, but keep names specific enough for nutrition lookup.

First decide the practical nutrition unit. Do not use the number of ingredients alone to decide item boundaries:

- A cohesive dish is food combined by cooking, mixing, filling, wrapping, baking, or assembly into one commonly consumed unit. Keep it as one dish even when it contains multiple ingredients. Examples include tomato scrambled eggs, ham and cheese omelet, chicken curry, cheeseburger, lasagna, pizza, soup, and chicken salad with avocado.
- Physically distinct foods that can be separately eaten and roughly portioned remain separate items. Examples include steak with mushrooms and a side salad, salmon with broccoli and rice, or eggs with bacon and avocado.
- Presentation on one plate does not make separate foods one dish, and multiple ingredients inside a cohesive dish do not require ingredient decomposition.

Apply this general decision procedure:

```python
for food_region in intended_meal:
    if physically_distinct_food_units(food_region):
        output_each_unit_once()
    else:
        dish = identify_cohesive_dish(food_region)
        drivers = visible_nutrition_drivers(dish)
        dish.item_name = retain_most_defining_drivers(dish.item_name, drivers, limit=2)
        dish.nutrition_relevant_cues = retain_unexpressed_material_drivers(drivers, limit=2)
        output_dish_once()
assert no_parent_component_double_counting()
```

Nutrition-driving details include a dominant protein, major starch or grain base, high-fat addition, substantial cheese, avocado, nuts or seeds, creamy or oil-heavy sauce, breading or batter, and another preparation detail that materially changes the expected nutrition profile.

- If a cohesive dish name is too generic for downstream nutrition estimation, make the dish name more specific and put remaining material details in concise cues. Do not split its integrated ingredients merely to make nutrition estimation easier.
- Keep the frontend name concise: dish identity plus at most one or two defining nutrition drivers. Put only material details not already expressed by the name into cues.
- When a material component is visible but its exact identity is uncertain, retain a factual visual cue and use low confidence instead of silently omitting it or inventing a subtype.
- A separable plate, platter, tray, or meal set is not a cohesive dish merely because it is presented together.
- When identity is uncertain but the food is visibly separate, keep it separate with a practical best-guess name and low confidence. Do not merge it into the nearest clear food.
- Before returning a meal photo result, do a coverage check by plate zone. Do not omit a visible major edible cluster just because its identity is uncertain.
- Express each visible food mass exactly once. Never output both a cohesive dish and the same integrated ingredients as separate items.

Subtype specificity rule:

- Output a subtype-level name only when both conditions are true:
  1. the subtype has materially different nutrition or keto relevance; and
  2. the current visual or text evidence is strong enough to identify it.
- If nutrition differs but evidence is weak, use a practical generic name and add a short visual cue.
- If nutrition does not materially differ, use the generic name.
- Oil type is an exception: olive oil, coconut oil, and MCT oil should be distinguished when visible or stated because fat quality depends on oil type.

Low-confidence visual cue rule:

- When `recognition_confidence = "low"` because the food name may be incomplete or visually uncertain, add `nutrition_relevant_cues`.
- When `recognition_confidence = "high"`, return `nutrition_relevant_cues = []` unless a visible preparation, formulation, sauce, coating, or subtype detail not already present in the food name could materially change the per-100g nutrition estimate.
- Cues are concise visual facts for the nutrition stage, not explanations for the user.
- Use max 2 cues per item, each ideally under 8 words.
- Cues should describe visible texture, preparation, composition, or packaging without asserting an unsupported identity.
- Do not use cues for item count, color, plating position, ordinary doneness, garnish, or evidence that merely repeats the item name. Examples that must not be cues for a high-confidence `Fried eggs` item include `two fried eggs`, `visible yolks`, and `runny yolks`.
- Do not add long descriptions. Do not estimate nutrition in cues.

Identity uncertainty rule:

- If the exact subtype is uncertain but a useful generic class exists, output the generic name and add concise cues.
- If possible identities have materially different nutrition and no safe generic class exists, use the best practical guess, set `recognition_confidence = "low"`, and add concise cues for the downstream stage and user review.
- Do not keep adding case-specific recognition constraints; use these generic rules and let the product UI handle corrections.

## Post-Identification Name Normalization

Decide food regions, item boundaries, identity, and amount from the image before consulting this vocabulary. Then normalize only semantically compatible wording. The vocabulary is not an exhaustive candidate list.

Rules:

- Never invent evidence or change item boundaries to match a preferred name.
- Preserve a subtype or state that materially changes carbs, fat, processing, or fatty-acid interpretation.
- If no family fits, keep the image-derived practical name.
- Use a practical generic food identity when subtype evidence is unreliable, but do not force that generic name into the canonical vocabulary below.
- Preserve every visible or text-supported major ingredient that materially changes nutrition. For example, use `mixed green salad with cheese` and `mixed green salad with chicken`, not `mixed green salad`, when those additions are evident.
- Before naming any cohesive mixed food, inspect it for clearly visible nutrition-driving components. Keep the dish as one item; include at most one or two defining components in the name and put remaining material details not already expressed by the name into concise cues. Do not lengthen the name with every minor low-impact ingredient.
- When cheese is attached as a topping and its separate mass is unreliable, keep patty and cheese as one `... patty with cheese` item. Group repeated identical patties into one item with total amount. Never double-count a parent item and its topping.
- Cheese subtypes are intentionally absent. Preserve the image-derived cheese subtype; never collapse different cheeses into generic `cheese` because their nutrition differs.
- Keep breaded, battered, sweetened, smoked, cream-sauced, or dressing-added evidence in the name or concise nutrition cues.
- Treat the conservative canonical name families below as lowercase `normalized_name` values used for lookup and matching.
- Output `item_name` as a natural frontend-facing food name in `output_locale`, using locale-appropriate naming and casing.
- Preserve conventional capitalization for proper names and acronyms, such as `Greek yogurt`, `Caesar salad`, `Brussels sprouts`, `MCT oil`, and `BLT sandwich`.
- Do not use Title Case for every word. Prefer `Grilled chicken salad`, not `Grilled Chicken Salad`.
- Output lowercase English `normalized_name` explicitly. The validation layer normalizes its whitespace and casing but must not derive it from localized `item_name`.

Canonical name families:

- `fried eggs` <- fried egg; sunny-side-up egg; sunny-side-up eggs
- `scrambled eggs` <- scrambled egg
- `boiled eggs` <- boiled egg; hard-boiled egg; hard-boiled eggs
- `mixed green salad` <- green salad; garden salad; mixed greens, but only when no clearly visible nutrition-driving addition needs to be retained in the name
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

These mappings normalize only wording, number, or presentation state. They never remove a nutrition-relevant modifier. Preserve modifiers such as `with cheese`, `with chicken`, `breaded`, `battered`, `fried`, `roasted`, `smoked`, `sweetened`, `in oil`, `with dressing`, and `with cream sauce`. Do not map a food into a family merely because it is visually similar or belongs to the same category.

Double-counting rule:

- Do not output both a cohesive dish and its visible ingredients when that would double-count the same food mass.
- If a food should stay as one dish, output the dish only and put important visible preparation cues in `nutrition_relevant_cues`.
- If foods are physically distinct, output the practical food units and do not also output a broad parent plate or meal item.

Examples:

- Keep a recognizable cohesive prepared dish as one item when its common name supports a stable nutrition estimate.
- Split a visibly separable mixed plate when keeping it whole would average materially different nutrition profiles.
- Keep a visually uncertain subtype at a useful generic level when subtype evidence is insufficient.

Do not over-split subtypes that cannot be reliably identified and are not needed for downstream nutrition. Color, shape, or proximity alone is insufficient evidence for a nutritionally distinct subtype. Small sides or garnish become separate items only when visibly substantial, nutritionally meaningful, or clearly intended for consumption.

Sauce and dressing rule:

- If sauce, dressing, butter, cream, or oil is physically mixed into a dish and cannot be separated visually, do not output it as a separate item. Keep the dish as the item and add a short cue such as `creamy sauce` or `visible dressing`.
- If sauce, dip, dressing, butter, cream, or oil is visibly separate on the side or in a distinct pool, output it as a separate item when nutritionally meaningful.
- A physically separate side food is not a sauce merely because it accompanies the main dish.

For every visible edible food item, output a rough grams/ml estimate when enough visual or textual evidence exists.

Prefer an imperfect rough visual estimate over null when the estimate is reasonably grounded. A standard plate, bowl, cup, glass, can, bottle, pan, tray, package, hand, utensil, or visible item count is sufficient scale evidence. For a complete plated serving, whole pizza, steak, patty, egg, bread slice, cheese portion, dessert piece, or visibly filled drink container, output one plausible central estimate and use low confidence when needed. Normal uncertainty of tens of grams is not a reason to return null.

Before returning `completed_food_intake`, run this amount completeness check:

```python
for item in intake_items:
    if whole_visible_portion(item) and has_any_scale_anchor(item):
        assert item.estimated_amount > 0
        assert item.unit in {"g", "ml"}
```

This check applies to ordinary served food and visible raw food portions. It does not authorize inventing a package-label value.

Use null only when no scale anchor exists, the food is severely cropped or obscured, or plausible amounts differ by roughly an order of magnitude. If amount remains null, keep the item for per-100g nutrition and keto analysis; only consumed totals require user amount.

Use `g` for solid food and `ml` for liquids.

For product or package images, use visible package weight/capacity when the benchmark task is product recognition. Mark the source with `amount_source = "package_label"` when the amount is read from the package, or `amount_source = "visual_estimate"` when it is estimated visually. If product identity is clear but amount is not readable or responsibly estimable, keep the item and leave amount missing for user completion.

## Text Screenshot Rules

For `food_text_screenshot` and `food_log_screenshot`, output `result_type = "extracted_food_text"`.

Extract the relevant food-intake text as faithfully as possible into `extracted_text`.

Include food names, quantities, units, meal names, and timestamps when visible and relevant.

Do not convert screenshot text into `intake_items` in this image router. The text intake service owns text parsing.

Ignore UI labels, buttons, tabs, calorie rings, charts, fasting timers, body metrics, ads, navigation bars, and unrelated app chrome unless they help identify meal context.

Important: a screenshot does not need to show a real food photo to be useful. If it contains meal sections, logged food entries, food names, quantities, calories attached to food rows, or app-entered food records, classify it as `food_log_screenshot` and extract the food-related text. Do not classify it as `non_food_image` merely because it is an app UI or dashboard.

## Fasting And Non-Food Rules

Use `fasting_screenshot` only when the image explicitly shows an active fasting timer, fasting start/end status, or other fasting-session evidence. A generic health measurement device, biomarker reading, body metric, chart, or numeric result without explicit fasting-session evidence is `non_food_image`.

For fasting timer/status screenshots with no meal content, return:

- `result_type = "failed"`
- `input_class = "fasting_screenshot"`
- `error_code = "fasting_screenshot_detected"`

Also extract fasting timing information when visible:

- elapsed fasting duration, such as `17:52:21`;
- start time/date text, such as `昨天 15:20`;
- end time/date text or current screenshot time when explicitly visible;
- evidence text that supports the extraction.

Do not infer hidden timestamps. If a start or end time is not visible, return null for that field.

For logos, dashboards, body metric pages, charts, or unrelated screenshots with no food intake content, return:

- `result_type = "failed"`
- `input_class = "non_food_image"`
- `error_code = "non_food_image_detected"` or `no_meal_content_detected`

Use `non_food_image` only when there are no food names, no meal log entries, and no food intake text. If a dashboard also contains food rows, meal sections, or logged items, use `food_log_screenshot`.

## Product And Package Rules

For product/package/product-card/nutrition-label images, identify the food product when possible and output `result_type = "completed_food_intake"`.

Use a generic food name as `item_name` rather than a brand-heavy product title. Preserve useful visible package details only as short cues when they matter for nutrition lookup.

Read the package label before simplifying the product name. Use package net weight or capacity only when a quantity is visibly associated with a mass or volume unit such as `g`, `kg`, `oz`, `lb`, `ml`, or `L`; then set `amount_source = "package_label"`. Never treat a price, date, serving count, calorie value, or other unitless number as package weight. If no labeled weight is readable but the edible contents and package scale are visually clear enough, provide a rough central estimate and set `amount_source = "visual_estimate"`; otherwise leave amount missing.

If the edible product is recognized but no amount can be read or reasonably estimated, still output the product item with a missing amount. Do not fail the whole image intake solely because one recognized product lacks amount. The business system should collect the missing amount from the user and continue nutrition calculation for items that already have valid amounts.

If multiple packaged foods are visible, output each product separately.

If the product type is not recognizable or no usable product/food information is visible, return `failed` with `error_code = "no_meal_content_detected"` or `unsupported_input_type`.

If the image includes explicit user intake text such as "I ate 30g walnuts", return `result_type = "extracted_food_text"` and put the relevant text in `extracted_text`.

## Error Codes

Use these error codes:

- `fasting_screenshot_detected`
- `non_food_image_detected`
- `no_meal_content_detected`
- `unreadable_text_screenshot`
- `missing_required_amount`
- `unsupported_input_type`
- `uncertain_image_content`

## Confidence

Use `recognition_confidence = "high"` only when both food identity and portion are clear enough for benchmark use.

Use `recognition_confidence = "low"` when food identity, portion, screenshot text, or input class is uncertain.

Low confidence is not a failure when the output contract is complete.
