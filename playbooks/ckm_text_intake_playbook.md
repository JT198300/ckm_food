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

## Food Category

Assign exactly one `food_category` to every returned intake item. The category is a stable icon/index key and must not replace or simplify the extracted food name.

Use only these 18 values, mapped from `03_大类Icon分类` in `高频食物_通用食物分类版.xlsx`:

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
- `other_food`: Other Food / 其他食物

Classify the returned item itself. Use `salad` for named salads and `soup_stew_and_mixed_meals` for cohesive mixed dishes that do not belong to a more specific category. Use `other_food` only when no listed category is defensible. Do not alter item boundaries or food specificity merely to fit a category.

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

For a named dish, output the dish as one item unless the text clearly lists separate foods or amounts.

Use the same general nutrition-granularity test across all food categories:

- Keep a cohesive named dish when it is a stable nutrition lookup candidate.
- Split explicitly listed foods, separately quantified components, or broad combinations that would otherwise average materially different nutrition profiles.
- Output a subtype only when it materially changes nutrition or keto interpretation and the text provides enough evidence.
- When subtype evidence is insufficient, use a useful generic class rather than guessing.
- Do not output both a parent dish and its component foods when that double-counts the same intake.
- Do not add category-specific extraction rules in response to individual benchmark failures.

Use a practical generic food or dish name rather than a brand-heavy product title. Preserve preparation or subtype in the name only when it materially affects nutrition; retain secondary visible or stated details in concise context fields when available.

## Food Name Capitalization

- Output `item_name` as a frontend-facing English name using sentence case: capitalize the first word, not every word.
- Preserve conventional capitalization for proper names and acronyms, such as `Greek yogurt`, `Caesar salad`, `Brussels sprouts`, `MCT oil`, and `BLT sandwich`.
- Do not use Title Case for every word. Prefer `Smoked salmon with cream cheese`, not `Smoked Salmon With Cream Cheese`.
- The validation layer derives lowercase `normalized_name` from `item_name` for lookup and matching.
- Capitalization must not change the food identity, preparation, subtype, or item boundaries extracted from the source text.

For sauces and added fats, keep physically integrated sauce within a cohesive dish. Treat a separately listed or separately quantified sauce, dip, dressing, or added fat as its own item when nutritionally meaningful.

If the text contains meal sections such as breakfast, lunch, dinner, or snack, keep the food items but do not create a fake food item for the meal section itself.

Ignore non-food UI text, button labels, tab names, calorie rings, charts, app navigation, exercise rows, fasting timers, advertisements, and body metrics unless they are directly attached to food rows.

## Amount Rules

Every returned food item must include a usable `estimated_amount` and `unit`.

Use:

- `g` for solid food;
- `ml` for liquids.

Do not use `piece`, `serving`, `cup`, `tbsp`, `egg`, `slice`, or household units in `unit`. Convert them to rough grams/ml.

If the text provides explicit quantity, use it and set `amount_source = "explicit_text"`.

Examples:

- `115g beef` -> `estimated_amount = 115`, `unit = "g"`, `amount_source = "explicit_text"`.
- `30 ml cream` -> `estimated_amount = 30`, `unit = "ml"`, `amount_source = "explicit_text"`.
- `2 fried eggs` -> estimate about `100g`, `amount_source = "explicit_text"` because count was explicit.
- `half avocado` -> estimate about `70g`, `amount_source = "explicit_text"` because fraction was explicit.

If the text names food but gives no explicit amount, provide a rough default portion estimate only when a reasonable meal-serving interpretation exists. Set `recognition_confidence = "low"` and add a warning.

If the text is too vague to estimate amount, return no item or mark the item as failed through validation.

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

## Error Codes

The validation code may return:

- `no_food_text_detected`
- `missing_required_amount`
- `product_package_without_consumed_amount`
- `ambiguous_food_text`
- `asr_text_unusable`
- `unsupported_input_type`
