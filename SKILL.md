---
name: agent-script
description: Use to author, plan, and execute structured agent tasks and workflow scripts.
---

# AgentScript

## Import

```kotlin
// The same `@file:Import` syntax carries two different meanings:
// - Subtask document: this task depends on that task instance. Calling its function
//   runs it as a blocking call; the subtask records its own state and output.
// - Skill: a reference to stateless definitions. Run them through a subtask, or inline them.
// Paths are relative to the importing document; update them when tasks move.
// Before executing an imported function, read its document and transitive imports.
@file:Import("../executable/2026-11-05-restack-app-tracing.md") // subtask
@file:Import("../../.agents/skills/review-infra/SKILL.md")     // skill
```

## Subtask

```kotlin
// A subtask is a standalone document defining one function with parameters.
// The parent calls it like a plain blocking function and receives its deliverable.

// File: kanban/active/2026-11-05-port-c0-libclang-runtime.md (parent)
@file:Import("../executable/2026-11-05-port-c0a-libclang-bindings.md")

val bindings = PortLibclangBindings(upstream = pinnedUpstream, toolchain = verifiedToolchain)
`Replace JVM/JNI calls with the Native bindings`(bindings)

// File: kanban/executable/2026-11-05-port-c0a-libclang-bindings.md (subtask)
fun PortLibclangBindings(upstream: Revision, toolchain: Toolchain): ClangBindings {
    `Copy the pinned upstream modules, keeping clang.def and Linux conditions`(upstream)
    `Build clangext and generate Native bindings`(toolchain)
    `Parse a real C header and verify cursors, callbacks, and resource release`()
    return `Hand the pinned KLIB and link dependencies to the parent`()
}
```

## Skill

```kotlin
// Call a skill in one of two ways.

// 1. As a subtask: a subtask document imports the skill, pins its parameters, and calls it.
//    The parent calls the subtask as usual. Narrow the scope of this run in a trailing comment.
// File: kanban/executable/2026-11-05-review-pr-44505.md
@file:Import("../../.agents/skills/review-infra/SKILL.md")

fun ReviewPr44505(pr: String): ReviewReport {
    val target = InfraPR(url = pr, useParallelExternalModel = false, authorizesMechanicalCIFixes = false)
    return ReviewInfra(target) // read-only review and report; skip public submission steps
}

// 2. Inlined: copy the skill's function declaration together with its body into the task,
//    then call it. Adapt the copy to this task without dropping steps or guards.

// Inlined from ../../.agents/skills/document-review-workflow/SKILL.md
fun DocumentReviewWorkflow(document: Document, reviewFocus: String): MarkdownFile {
    var file = `Put document in editable Markdown file`(document)
    do {
        askUser("Review ${file.path} (focus: $reviewFocus) and leave FIXME notes")
        file = `Reread document preserving user edits`(file)
        `Resolve every FIXME marker`(file)
    } while (file.`has remaining review points or FIXME markers`())
    return file
}

val reviewed = DocumentReviewWorkflow(design, reviewFocus = "API boundaries")
```

## Subagent Delegation

```kotlin
// Delegate a subtask by handing its document to a subagent; the subagent holds its flock.
// Launch delegated subtasks inside `coroutineScope`; the scope returns only after all finish.
// Accept results by checking each subtask's completion and output, not lock release.
coroutineScope {
    launch { AdaptAppTracing(layer = 7) }   // subagent
    launch { AdaptAgentTracing(layer = 7) } // subagent
    `Build the App, Agent, and Workflow regression baseline`()
}
`Accept subtask outputs against their deliverables`()
```

## Suspend Function

```kotlin
// Asking the user and long-running operations of uncertain duration are suspend functions.
// Calling one suspends the task: park `/*PC*/` on it and release held locks while waiting.
// Authorizations and decisions are results of these calls; bind them as values.
// `askUser` calls the ask-user tool.
val draft = `Draft the APPROVE text`()
val authorization = askUser("Submit APPROVE with this text?\n$draft", listOf("Approve", "Edit", "Cancel"))
if (authorization == "Approve") {
    `Submit APPROVE against the reviewed head and read it back`(draft)
}
```

## Resource Scope

```kotlin
// Hold resources with scope functions; they release on every exit path.
// Prefer them over try/finally for cleanup.
taskFlock.withLock {
    `Create an isolated temporary worktree`().use { worktree ->
        `Run contract and integration regressions`(worktree)
    }
}
```

## Error Handling

```kotlin
// try/catch expresses failure semantics, not cleanup.
// Unexpected situation: stop, report to the user, and replan from their decision.
try {
    `Merge the verified lower layer and resolve conflicts`(layer)
    `Run related regressions and static checks`(layer)
} catch (e: Exception) {
    val decision = askUser("Unexpected failure: ${e.message}. Completed and remaining scope attached. How to proceed?")
    `Replan the remaining script from the decision`(decision)
}

// Live-state change: back up first; on validation failure restore and rethrow.
try {
    `Back up live ~/.config/environment.d/90-fcitx.conf`()
    `Install the new environment file and sync the live session`()
    `Validate the environment generator, systemctl --user, and Fcitx5 debug info`()
} catch (e: ValidationFailed) {
    `Restore the backup`()
    throw e
}
```

## Control Flow

```kotlin
// Express decisions and repetition with control flow instead of a linear list of steps.
when (review.verdict) {
    Verdict.Approve -> `Merge change`(change)
    Verdict.RequestChanges -> `Request changes from author`(review.findings)
}
if (module.requiresTests) `Compile and test module`(module)
for (module in modules) {
    `Migrate module`(module)
}
while (!`All checks pass`()) {
    `Fix next failing check`()
}

// Guard: refresh external state right before an irreversible or public action,
// and throw when it drifted from what was reviewed.
val gates = `Refresh head, discussion, CI, and repository rules`()
if (gates.head != reviewedHead) throw ReviewOutdated()
`Submit APPROVE against reviewedHead and read it back`()

// Iterative loops carry an explicit bound.
var round = 1
do {
    val errors = `Score reviews against ground truth`(instructions)
    if (errors.isNotEmpty()) instructions = `Refine instructions without overfitting`(instructions, errors)
    round++
} while (errors.isNotEmpty() && round <= 6)
```

## Progress

```kotlin
// `/*PC*/` sits directly before the current or suspended call; one per active coroutine.
// Calls before it are done; calls after it are pending.
// Trailing comments on finished steps record receipts and coverage limits.
// The trailing comment on `/*PC*/` states why it is parked: what it waits for, or progress so far.
`Submit confirmed REQUEST_CHANGES review with 7 inline comments`() // https://github.com/langgenius/dify/pull/44483#pullrequestreview-5763860849
`Verify streaming, recycling, and real entry points`() // automated and isolated entries tested; other hosts untested
/*PC*/`Run macOS Native host regression`() // roundtrip interrupted; waiting for a free window
`Archive and update inbound links`()

// While a subtask runs, the parent's `/*PC*/` stays on the blocking subtask call.
// The subtask document tracks its own `/*PC*/`; the parent comment summarizes it.
/*PC*/ReviewPr44505(pr) // waiting for user confirmation (APPROVE "LGTM" + close competing #45714)

// Replan by rewriting pending steps; the script is the living plan.

// Done means the script finished and every scoped child coroutine finished.
// Then remove `/*PC*/` and move the task to Done; lock release alone is not completion.
```

## Task State

```kotlin
// Local variables and runtime state live in a `# Task State` section of the same document,
// parallel to `# Task Script`. Record each bound value under its variable name.
// Write the value as soon as the call returns; read it back when resuming.

// # Task Script
val pinned = `Pin main and the remote stack heads`()
val feedback = askUser("Review docs/plan.md and leave FIXME notes")
/*PC*/`Revise the plan from feedback`(pinned, feedback)

// # Task State
// - pinned: main = 262158cf, top = fb01efbfd5 (26 layers)
// - feedback:
//   - "Merge Steps into Control Flow"
//   - "Prefer flock.withLock {} over try/finally for cleanup"
```

## Execution Context

```kotlin
// Run every step that targets another machine, privilege, or workspace inside `withContext`.
// Elements combine with `+`; steps and subtasks inside inherit the context.
// Functions never take the environment as a parameter.
for (device in listOf(Marble, Prague)) {
    withContext(device + Sudo) {
        `Back up baseline packages and settings`()
        `Disable the confirmed telemetry packages`()
        `Verify disabled states and zero crashes`()
    }
}
coroutineScope {
    launch(RemoteMacBook) { `Run macOS Native host regression`() }
    launch(XiaoxinUbuntu) { `Run Linux Native host regression`() }
}
`Compare host results`()
```
