---
description: "Execute plan in batches with review checkpoints. Use --resume to continue from last pause point."
disable-model-invocation: true
args: "[plan_path] [--resume]"
---

Invoke the superpowers:executing-plans skill and follow it exactly as presented to you.

**If `--resume` flag is present:**

RESUME INSTRUCTIONS: Examine the plan's acceptance criteria. Tasks with `passes: true` are already complete. Start execution from the first task whose acceptance criteria has `passes: false` or is missing. Do not re-execute completed tasks.

**Resume workflow:**
1. Read the plan file at the specified path
2. Scan all tasks for acceptance criteria status
3. Identify first incomplete task (no `passes: true`)
4. Continue execution from that task
5. Skip all tasks marked as complete
