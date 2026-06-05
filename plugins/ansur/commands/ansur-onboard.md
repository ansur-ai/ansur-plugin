---
description: Build or iterate an AI employee on Ansur — drives the `ansur` CLI through connect → create → author → ship → channel → trace.
argument-hint: [business or role to build an employee for]
---

Use the `ansur` skill to build an Ansur AI employee.

1. Run `ansur whoami`. If it errors with `no_tenant` (or `ansur` isn't installed /
   logged in), this is a first-time setup — follow `references/initial-setup.md`
   first, then continue.
2. Otherwise walk the build loop in the skill (`SKILL.md`): connect systems →
   `guards init` → `bundle create` → author bundle + guards policy → `git push`
   both repos → `channel bind` → `trace` and iterate. If writes need human approval,
   set guards `mode: gated` + `approve_if` and bundle `approvals.notify`.

The user's description of the business / role: $ARGUMENTS
