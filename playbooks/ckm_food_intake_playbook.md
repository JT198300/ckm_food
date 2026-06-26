# CKM Food Intake Playbook v0.2

## Scope

Use this playbook only in the intake extraction stage.

The intake stage converts meal images and/or user text into a unified food-and-amount JSON structure.

It must not generate nutrition values, keto labels, carb impact, or food database records.

## Inputs

The intake stage may receive:

- one or more meal images;
- user text such as "2 fried eggs and 100g salmon";
- meal context such as breakfast, lunch, dinner, or unknown;
- user context such as cuisine preference, usual foods, or serving habits;
- benchmark mode.

Images and text can both be present. Use text as context when it appears to describe the same meal. If text adds foods not visible in the image, include them with `source_type = "text"`.

## Output Contract

Return only the JSON required by the Dify node schema.

Output a top-level object:

```json
{
  "intake_items": [],
  "intake_warnings": []
}
```

Each `intake_items` element must represent one food, dish, simple food, ingredient, meal set, or unknown item.

## Dish-First Rules

Prefer dish-level recognition before ingredient-level recognition.

For a named dish, output the dish as one item unless decomposition is explicitly required.

Examples:

- If the image likely shows a cheeseburger, output `cheeseburger` as one dish.
- Do not output bun + beef patty + cheese + lettuce + tomato unless decomposition is explicitly required.
- If the image likely shows pepperoni pizza, output `pepperoni pizza` as one dish.
- If the image shows clearly separated grilled steak, asparagus, and mashed potatoes, output separate items.
- If the image shows a barbecue platter, buffet plate, sashimi platter, or mixed visible components, use `meal_set` or separate items depending on visual separation.

## Item Types

- `dish`: cooked or prepared dish treated as one nutrition unit.
- `simple_food`: single recognizable food such as boiled egg, apple, avocado, plain rice.
- `ingredient`: visible raw or component ingredient that is not itself the dish.
- `meal_set`: multiple foods presented together where no single dish name is adequate.
- `unknown`: image or text is too ambiguous.

Use `unknown` only when the food identity is too unclear to produce a useful nutrition lookup candidate.

When the item is clearly edible but the exact identity is uncertain, output the best practical guess as `item_name` and `normalized_name`, set `recognition_confidence = "low"`, and put alternatives in `ambiguities`.

Example:

- Prefer `roasted beetroot or dark red vegetables` with low confidence over `unknown` when the image clearly shows a dark red roasted food item.

## Portion Rules

Estimate portion only when visual or textual evidence supports a rough grams/ml estimate.

Use:

- `g` for solid food;
- `ml` for liquid;
- `unknown` when amount cannot be estimated.

Do not use `piece`, `serving`, or household units in `unit`.

If the user says "2 fried eggs", estimate total grams and put the original phrase in `source_text_span`.

Use `estimated_amount = null` and `unit = "unknown"` when scale is unclear.

## Source Tracking

For every item, indicate where it came from:

- `source_type = "image"` when identified from an image;
- `source_type = "text"` when parsed from user text;
- `source_type = "image_and_text"` when image and text refer to the same item.

For multiple images, set:

- `source_image_index = 1` for the first image;
- `source_image_index = 2` for the second image;
- and so on.

If an item comes only from text, set `source_image_index = null`.

Use `source_image_filename` only when available; otherwise null.

Use `source_text_span` for the exact text fragment that supports the item. Use null when no text span exists.

## Confidence

Set `recognition_confidence = "high"` when the item identity and amount are clear enough for benchmark use.

Set `recognition_confidence = "low"` when:

- food identity is ambiguous;
- amount is weakly estimated;
- multiple foods are visually merged;
- text is vague, such as "some fish" or "a plate of meat";
- image and text conflict.

Low confidence is not a failure. It should be recorded for benchmark analysis.

## Strict Exclusions

Do not output:

- calories;
- protein;
- fat;
- carbohydrates;
- net carbs;
- keto labels;
- food database IDs;
- validation results.
