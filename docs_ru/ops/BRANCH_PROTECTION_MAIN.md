---
title: "Защита ветки — main"
---

# Защита ветки — `main` (OpenSSF Scorecard: Branch-Protection)

Действие владельца. Применяется через Settings → Branches → Add rule, или:

```bash
gh api -X PUT repos/diegosouzapw/OmniRoute/branches/main/protection \
  --input - <<'JSON'
{ "required_status_checks": { "strict": true, "contexts": ["Quality Ratchet", "Quality Gates (Extended)", "Fast Quality Gates"] },
  "enforce_admins": false,
  "required_pull_request_reviews": { "required_approving_review_count": 0, "dismiss_stale_reviews": true },
  "restrictions": null,
  "required_linear_history": false,
  "allow_force_pushes": false,
  "allow_deletions": false }
JSON
```

Поднимает оценку Scorecard Branch-Protection с 0. `enforce_admins:false` сохраняет работоспособность
существующего потока слияния форвард-мержем; ужесточите до `true`, когда всё стабилизируется.
