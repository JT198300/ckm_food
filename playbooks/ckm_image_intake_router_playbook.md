# CKM Image Intake Router Playbook v0.1

## Scope

Use this playbook only in the image intake router stage.

The image router receives one or more images and must decide whether the images already contain complete food-and-amount intake information, contain food text that should be handed to the text intake service, or contain no usable meal intake content.

Do not generate nutrition values, keto labels, carb impact, or food database records.

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

## Meal Photo Rules

For `meal_photo`, identify visible food items and output `result_type = "completed_food_intake"`.

If a meaningful portion of the image clearly shows plated or served food, classify it as `meal_photo` even when people, furniture, packaging, or other non-food background occupies more of the frame.

Determine intended intake scope before extracting items. Prioritize food that is plated, served, opened for consumption, or explicitly presented as the meal. Do not automatically include background packages, storage containers, decorative food, or unrelated edible objects merely because they are visible.

Prefer dish-level recognition before ingredient-level decomposition, but keep names specific enough for nutrition lookup.

Dish-first is not blanket merging. If a dish name is too broad, too complex, or too generic for the downstream nutrition stage to query or estimate, split the visible food into the smallest practical nutrition-calculable units.

Do not output a presentation-level name when the image supports more specific nutrition-calculable items and amounts.

Use the downstream nutrition test to decide whether to split or merge:

- If one item would require averaging clearly different nutrition profiles, split it.
- If the image shows visible boundaries between major food groups, split them into separate items.
- A named cohesive dish can stay as one item only when it has a stable nutrition lookup candidate.
- A cohesive mixed dish name or its short cues must retain any clearly visible dominant component that materially changes the nutrition estimate. Do not use a narrower dish name that silently omits such a component.
- A separable plate, platter, tray, or meal set is not a cohesive dish merely because it is presented together.
- When identity is uncertain but the food is visibly separate, keep it separate with a practical best-guess name and low confidence. Do not merge it into the nearest clear food.
- Before returning a meal photo result, do a coverage check by plate zone. Do not omit a visible major edible cluster just because its identity is uncertain.
- If adjacent clusters have materially different nutrition profiles, output both separately rather than choosing only one.

Subtype specificity rule:

- Output a subtype-level name only when both conditions are true:
  1. the subtype has materially different nutrition or keto relevance; and
  2. the current visual or text evidence is strong enough to identify it.
- If nutrition differs but evidence is weak, use a practical generic name and add a short visual cue.
- If nutrition does not materially differ, use the generic name.
- Oil type is an exception: olive oil, coconut oil, and MCT oil should be distinguished when visible or stated because fat quality depends on oil type.

Low-confidence visual cue rule:

- When `recognition_confidence = "low"` because the food name may be incomplete or visually uncertain, add `nutrition_relevant_cues`.
- Cues are concise visual facts for the nutrition stage, not explanations for the user.
- Use max 2 cues per item, each ideally under 8 words.
- Cues should describe visible texture, preparation, composition, or packaging without asserting an unsupported identity.
- Do not add long descriptions. Do not estimate nutrition in cues.

Identity uncertainty rule:

- If the exact subtype is uncertain but a useful generic class exists, output the generic name and add concise cues.
- If possible identities have materially different nutrition and no safe generic class exists, use the best practical guess, set `recognition_confidence = "low"`, and add concise cues for the downstream stage and user review.
- Do not keep adding case-specific recognition constraints; use these generic rules and let the product UI handle corrections.

Double-counting rule:

- Do not output both a cohesive dish and its visible ingredients when that would double-count the same food mass.
- If a food should stay as one dish, output the dish only and put important visible preparation cues in `nutrition_relevant_cues`.
- If a food is too broad for nutrition calculation, split it into practical items and do not also output the broad parent item.

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

Prefer an imperfect rough visual estimate over null when the estimate is reasonably grounded. Use low confidence to express uncertainty.

If the food identity is useful but the amount cannot be estimated responsibly, keep the food item with `estimated_amount = null`, `amount_source = "unknown"`, and low confidence. The business UI can ask the user to enter or correct the amount before nutrition calculation.

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

When visible, use package net weight or capacity as the item amount and set `amount_source = "package_label"`. If the amount is not readable but the package size is visually clear enough, provide a rough visual estimate and set `amount_source = "visual_estimate"`.

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
