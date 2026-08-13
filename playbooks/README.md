# CKM Playbooks

## Current Runtime Playbooks

Only the following files are current rule sources. New Dify apps and backend
services must load one of these files according to their responsibility.

| Application responsibility | Current Playbook |
|---|---|
| Image Intake | `ckm_image_intake_router_playbook.md` |
| Text Intake | `ckm_text_intake_playbook.md` |
| Nutrition + Keto | `ckm_food_nutrition_keto_playbook.md` |
| About This Food | `ckm_about_this_food_playbook.md` |
| Meal Compatibility | `ckm_meal_compatibility_playbook.md` |

The current Dify DSL files use commit-pinned GitHub Raw URLs. Do not replace a
pinned URL with `main` during routine migration because that makes an imported
workflow change behavior without a DSL change.

## Archive

`archive/` contains superseded combined-flow Playbooks and prompt experiments.
They are retained for traceability and benchmark reproduction only.

Do not inject files under `archive/` into a production workflow. When an
experiment is approved, promote its final content into the corresponding
current Playbook above, publish a commit, and update the DSL to that commit's
Raw URL.

Archive groups:

- `archive/about_this_food_experiments_20260814/`: About This Food prompt
  iterations evaluated before the current writer was promoted.
- `archive/image_intake_experiments/`: historical Image Intake P1 candidates.
- `archive/legacy_combined_flows/`: the original combined Food AI Playbook and
  the superseded shared Intake Playbook.
