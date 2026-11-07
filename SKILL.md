---
name: programmatic-planning
description: Use to plan for any type of work.
---

# Programmatic Planning

## Script Forms

- `Task Script`: one fenced `kotlin` pseudocode block.
- Task entry: top-level logic. Reusable workflow or subtask: function.
- Keep context, decisions, state, and results in the script.

### Reusable Workflow

```kotlin
fun Review(change: Change): ReviewResult {
    val findings = `Inspect the change`(change)
    return `Summarize review results`(findings)
}
```

## Imports and Subtasks

- `@file:Import`: relative to the importing document; update paths when tasks move. Import declares functions; a call executes them. Before executing an imported function, recursively read its defining document and transitive `@file:Import` targets.
- Reuse existing skill workflows in subtasks via `@file:Import` and parameterized calls instead of simplified ad-hoc functions.
- Standalone subtask document: invocation context, state, and output. Check completion and output; lock release alone is not completion.
- Give subagents their subtask document; they hold its flock. Model resource ownership as a subtask with acquisition and release.
- Markdown imports are pseudocode, not native Kotlin imports; IDE resolution and navigation are not guaranteed.

### Parent Task

```kotlin
@file:Import("../skills/review/SKILL.md")
@file:Import("../executable/2026-01-01-validate.md")

val change = `Read the task change`()
val review = Review(change)
val validation = Validate(change)
// ^
`Deliver results`(review, validation)
```

### Subtask Document

```kotlin
fun Validate(change: Change): ValidationResult {
    val environment = `Prepare validation environment`(change)
    environment.use {
        try {
            `Check predicate A`()
            `Check predicate B`()
            `Check predicate C`()
        } catch (e: PredicateFailed) {
            return `Handle predicate failure`(e)
        }
        return ValidationResult.`Successful`
    }
}
```

## Execution State

- `// ^`: current or suspended statement, per active invocation or coroutine. Move or remove it as execution advances; evolve the script when replanning, looping, or branching.
- Shared workflow invocation: record the inner location in the calling task, e.g. `// ^ Review: waiting for user confirmation`.
- Completion: successful script and finished scoped child coroutines.
- Interaction: Waiting for user interaction is a suspension.

## Control Flow

```kotlin
try {
    for (module in modules) {
        if (module.requiresTests) {
            `Compile and test module`(module)
        }
    }
} catch (e: CompileOrTestFailed) {
    val response = `Report failure and wait for a decision`(e)
    `Replan based on the decision`(response)
} finally {
    `Clean up task resources`()
}
```

## Concurrency

- `kotlinx.coroutines`: structured concurrency.

```kotlin
coroutineScope {
    val client = async {
        `Build client artifacts`()
        // ^
    }
    val server = async {
        `Build server artifacts`()
        // ^
    }
    client.await()
    server.await()
    `Display build results to user`()
}
```
