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

Prefer dish-level recognition before ingredient-level decomposition, but keep names specific enough for nutrition lookup.

For every visible edible food item, output a rough grams/ml estimate.

Prefer an imperfect rough visual estimate over null when food is visible. Use low confidence to express uncertainty.

Use `g` for solid food and `ml` for liquids.

Do not use packaging net weight, product weight, or nutrition label serving size as consumed amount.

## Text Screenshot Rules

For `food_text_screenshot` and `food_log_screenshot`, output `result_type = "extracted_food_text"`.

Extract the relevant food-intake text as faithfully as possible into `extracted_text`.

Include food names, quantities, units, meal names, and timestamps when visible and relevant.

Do not convert screenshot text into `intake_items` in this image router. The text intake service owns text parsing.

Ignore UI labels, buttons, tabs, calorie rings, charts, fasting timers, body metrics, ads, navigation bars, and unrelated app chrome unless they help identify meal context.

## Fasting And Non-Food Rules

For fasting timer/status screenshots with no meal content, return:

- `result_type = "failed"`
- `input_class = "fasting_screenshot"`
- `error_code = "fasting_screenshot_detected"`

For logos, dashboards, body metric pages, charts, or unrelated screenshots with no food intake content, return:

- `result_type = "failed"`
- `input_class = "non_food_image"`
- `error_code = "non_food_image_detected"` or `no_meal_content_detected`

## Product And Package Rules

For product/package/product-card/nutrition-label images, do not treat product net weight or package size as consumed amount.

If the image does not say how much the user ate, return:

- `result_type = "failed"`
- `input_class = "product_or_package"`
- `error_code = "product_package_without_consumed_amount"`

If the image includes explicit consumed intake text such as "I ate 30g walnuts", return `result_type = "extracted_food_text"` and put the relevant text in `extracted_text`.

## Error Codes

Use these error codes:

- `fasting_screenshot_detected`
- `non_food_image_detected`
- `product_package_without_consumed_amount`
- `no_meal_content_detected`
- `unreadable_text_screenshot`
- `missing_required_amount`
- `unsupported_input_type`
- `uncertain_image_content`

## Confidence

Use `recognition_confidence = "high"` only when both food identity and portion are clear enough for benchmark use.

Use `recognition_confidence = "low"` when food identity, portion, screenshot text, or input class is uncertain.

Low confidence is not a failure when the output contract is complete.
