# CKM Food

This repository stores the CKM food AI playbook used by the Dify internal benchmark workflow.

## Dify Playbook URL

Use this raw URL in the Dify HTTP Request node `load_ckm_food_ai_playbook`:

```text
https://raw.githubusercontent.com/JT198300/ckm_food/main/playbooks/ckm_food_ai_playbook.md
```

A/B1/B2 should reference:

```text
{{load_ckm_food_ai_playbook.body}}
```
