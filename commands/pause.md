---
description: Pause plan execution gracefully with resume command
---

# Pause Execution

Complete your current in-progress task, then:

1. **Update acceptance criteria** for all completed tasks (passes: true)
2. **Output resume command** in this exact format:

---
**Session paused.** To continue in a fresh session, copy-paste:

```
/superpowers:execute-plan $PLAN_PATH --resume
```

Where `$PLAN_PATH` is the absolute path to the plan file.

The `--resume` flag will automatically:
- Scan tasks for acceptance criteria status
- Skip tasks with `passes: true`
- Start from first incomplete task
---

3. **Stop execution** - do not continue to next task
