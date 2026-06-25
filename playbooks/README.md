# CKM Food AI Playbooks

## V1 Compatibility

```text
ckm_food_ai_playbook.md
```

This is the original combined playbook used by the existing V1 workflow.

Do not remove it while the old Dify workflow is still in use.

Raw URL:

```text
https://raw.githubusercontent.com/JT198300/ckm_food/main/playbooks/ckm_food_ai_playbook.md
```

## V2 Split Playbooks

V2 uses two playbooks.

### Intake Playbook

```text
ckm_food_intake_playbook.md
```

Use for:

- image input;
- text input;
- dish-first food extraction;
- amount estimation;
- unified `intake_items` JSON.

Raw URL:

```text
https://raw.githubusercontent.com/JT198300/ckm_food/main/playbooks/ckm_food_intake_playbook.md
```

### Nutrition And Keto Playbook

```text
ckm_food_nutrition_keto_playbook.md
```

Use for:

- nutrition per 100g generation;
- keto label generation;
- confidence rules;
- deterministic validation boundaries.

Raw URL:

```text
https://raw.githubusercontent.com/JT198300/ckm_food/main/playbooks/ckm_food_nutrition_keto_playbook.md
```

## Dify Config

V1 only needs one HTTP Request node:

```text
load_ckm_food_ai_playbook
```

V2 should use two HTTP Request nodes:

```text
load_ckm_food_intake_playbook
load_ckm_food_nutrition_keto_playbook
```
