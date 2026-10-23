---
name: programmatic-planning
description: Use to plan for any type of work.
---

# Programmatic Planning

## Task Tree Structure

- Express `Task Script` as Kotlin script pseudocode in nested Markdown lists, with top-level logic and descriptive backtick-quoted action names.

```markdown
# Task Script

- `Review task requirements and relevant code`()
- **`Implement the requested behavior`()**
- `Run validation checks for the change`()

# Details

```

**This is only the simplest example. Planing with rich control flows is suggested.**

## Execution State

- Bold the current executing or suspended statement in each active invocation or coroutine; concurrent branches may have multiple bold positions.
- A task completes when its script finishes successfully and its scoped child coroutines have finished.

## Control Flow Examples

- Represent standard branches and loops with Kotlin blocks:

```markdown
- `if (needsMigration()) {`
  - `Run database schema migration`()
- `} else {`
  - `Verify existing schema integrity`()
- `}`
- `try {`
    - `for (module in modules) {`
      - `Compile and test module`(module)
    - `}`
- `} catch (CompileOrTestFailed) {`
    - val userResponse = `Report compile or test failure to the user`()
    - `replan with`(userResponse)
- `}`
- `while (hasPendingQueue()) {`
  - `Process next message`()
- `}`
```

## Concurrency Primitives

- Use `kotlinx.coroutines` primitives for structured-concurrency semantics.
- Bold active statements across concurrent branches:

```markdown
- `coroutineScope {`
  - `val taskA = async {`
    - **`Build client artifacts`()**
  - `}`
  - `val taskB = async {`
    - **`Build server artifacts`()**
  - `}`
  - `taskA.await()`
  - `taskB.await()`
  - `launch {`
    - `Report background metrics`()
  - `}`
  - `Deploy bundled artifacts`()
- `}`
```

## Exception Handling

- Express error handling and cleanup via `try` / `catch` / `finally` blocks:

```markdown
- `try {`
  - `Acquire deployment lock`()
  - **`Apply risky database migration`()**
- `} catch (e: MigrationException) {`
  - `Rollback migration step`(e)
  - `Report to user`()
- `} finally {`
  - `Release deployment lock`()
- `}`
```

## Subtasks

- Keep each subtask in a standalone task document and reference it from the parent task document.
- Express the subtask as a function call and link the call name to its document in `Details`.
- Holding a resource should be a subtask to avoid forgetting to release it.
- When invoking subagents, a subtask should be provided to the subagent to share context and collaborate. The subagent holds the subtask flock and can be waited.

