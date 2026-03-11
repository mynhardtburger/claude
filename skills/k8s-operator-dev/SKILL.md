---
name: k8s-operator-dev
description: >-
  This skill should be used when writing, reviewing, or debugging Kubernetes
  operators in Go using controller-runtime, Kubebuilder, or Operator SDK.
  Covers reconciliation loops, status conditions, finalizers, owner references,
  envtest testing, and performance tuning. Applicable when the user says things
  like "write a reconciler", "add a finalizer", "debug infinite reconciliation
  loop", "set up envtest", "add status conditions to my CRD", or "review my
  operator code". This skill should NOT be used for general Go development,
  Helm/Kustomize templating, kubectl usage, cluster administration, or
  application code running inside operator-managed pods.
user-invocable: false
composable: true
composable-sections:
  - "Review Context for Agents"
  - "Code Review Checklist"
  - "Anti-Pattern Quick Reference"
---

# Kubernetes Operator Development Guide

When writing, reviewing, or debugging Go code that uses controller-runtime, Kubebuilder, or Operator SDK, apply the rules below.

## When to Use

- Writing or modifying a `Reconcile()` function
- Defining CRD types (`*_types.go`)
- Setting up controller watches and predicates
- Writing envtest or operator E2E tests
- Reviewing PR diffs that touch operator code
- Debugging infinite reconciliation loops, stale cache, or resource conflicts

## When NOT to Use

- General Go development unrelated to Kubernetes controllers
- Helm chart or Kustomize template authoring
- kubectl usage or cluster administration
- Application code running inside pods managed by an operator

---

## Core Principles

These 7 rules are universally agreed upon across all authoritative sources. Treat violations as bugs.

### 1. Reconciliation is level-based, not edge-based
Read current state from the cluster. Never branch on event type (create vs update vs delete). The reconcile function receives only a name/namespace -- treat every call identically.

### 2. Reconcile functions MUST be idempotent
Running Reconcile N times with the same cluster state must produce the same result. Compute desired state from scratch every time. Use "if exists and matches, skip" guards before mutations.

### 3. Enforce full world state each reconciliation
Don't rely on incremental state. Assume information is eventually correct but may be stale. Reconstruct status from observed state, not from previously stored status.

### 4. One controller per Kind
Each controller reconciles exactly one GVK. Map child/related object events to the parent using `handler.EnqueueRequestForOwner` or `handler.EnqueueRequestsFromMapFunc`.

### 5. Keep reconciliations short
Never block waiting for long-running operations. Set status, return `RequeueAfter`, and check progress on the next reconcile. Think of requeuing as yielding from a coroutine.

### 6. Use owner references for in-cluster children, finalizers for external cleanup
`ctrl.SetControllerReference()` for same-namespace children (automatic GC + watch propagation). Finalizers only when external resources need cleanup or cross-namespace coordination is required.

### 7. Test state, not API calls
Assert on the state of the world (objects exist, fields match), not that specific API calls were made. Use envtest, not fake clients.

---

## Reconciliation Quick Reference

### Canonical Reconcile Structure

```go
func (r *MyReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // 1. Fetch the CR -- if gone, return success (nothing to do)
    // 2. Handle deletion -- check DeletionTimestamp, run finalizer cleanup
    // 3. Add finalizer if needed (only if external cleanup required)
    // 4. Validate the instance
    // 5. Reconcile child resources to match desired state
    // 6. Update status from observed state
    return ctrl.Result{}, nil
}
```

### Requeue Decision Table

| Situation | Return | Why |
|---|---|---|
| Success, nothing pending | `Result{}, nil` | Done. Remove from queue. |
| Waiting for external operation | `Result{RequeueAfter: 30*time.Second}, nil` | Check again after known interval. |
| Transient error (API conflict, network) | `Result{}, err` | Exponential backoff (5ms -> ~16min). |
| Permanent error (invalid spec) | `Result{}, reconcile.TerminalError(err)` | Do NOT retry. Log + record in metrics. |
| Need re-check at specific time | `Result{RequeueAfter: time.Until(deadline)}, nil` | Deadline-based requeue. |

**Avoid `Requeue: true`** -- it is deprecated. Use `RequeueAfter` with an explicit duration instead.

### After Any Mutation, Re-fetch
After `Status().Update()`, `Update()`, or `Patch()`, re-fetch the object before doing further work. This avoids optimistic concurrency conflicts from stale `resourceVersion`.

---

## Code Review Checklist

When reviewing operator code, check each item:

- [ ] Reconcile is idempotent -- no event-type branching, no side effects on repeated runs
- [ ] No blocking waits -- uses RequeueAfter for pending operations
- [ ] Status updated via `Status().Update()` or `Status().Patch()`, not `Update()`
- [ ] Status reconstructed from world state, not read from the object's own status
- [ ] `observedGeneration` set on conditions
- [ ] Owner references set on in-cluster children (`ctrl.SetControllerReference`)
- [ ] No cross-namespace owner references
- [ ] Finalizer present only when external cleanup is needed
- [ ] Finalizer removed only AFTER cleanup succeeds
- [ ] Predicates applied to reduce unnecessary reconciliations
- [ ] `GenerationChangedPredicate` used where status-only changes should not trigger reconciliation
- [ ] Deterministic resource naming where possible (for optimistic locking)
- [ ] Errors classified: transient -> return error, permanent -> TerminalError
- [ ] No `Requeue: true` -- use `RequeueAfter` instead
- [ ] RBAC annotations match actual API calls (no over-privileged roles)
- [ ] Tests use envtest, not fake clients
- [ ] Async assertions use `Eventually`/`Consistently`, not `time.Sleep`

---

## Reference Files

For detailed guidance on specific topics, consult these references:

| Topic | File | When to consult |
|---|---|---|
| Reconciliation patterns | [references/reconciliation.md](references/reconciliation.md) | Writing/restructuring a Reconcile function, debugging requeue behavior |
| Status and conditions | [references/status-and-conditions.md](references/status-and-conditions.md) | Adding conditions, debugging infinite loops, CRD type annotations |
| Resource management | [references/resource-management.md](references/resource-management.md) | Finalizers, owner refs, SSA, patching strategies |
| Testing | [references/testing.md](references/testing.md) | Writing envtest tests, test suite setup, async assertions |
| Performance | [references/performance.md](references/performance.md) | Cache tuning, predicates, indexing, concurrency |
| Anti-patterns | [references/anti-patterns.md](references/anti-patterns.md) | Reviewing code, identifying common mistakes |

---

## Anti-Pattern Quick Reference

| Anti-pattern | Fix |
|---|---|
| Event-driven reconciliation (create/update/delete branching) | Level-based: read state, compute diff, converge |
| `Requeue: true` | `RequeueAfter` with explicit duration |
| Reading own status to decide actions | Reconstruct status from world state |
| One controller managing multiple CRDs | One controller per Kind |
| `client.Update()` for status changes | `client.Status().Update()` or `client.Status().Patch()` |
| Fake client in tests | envtest with real API server |
| Blocking reconcile waiting for pods/jobs | Return `RequeueAfter`, check on next reconcile |
| Full PUT when patch suffices | Use `Patch()` or Server-Side Apply |
| Cross-namespace owner references | Use finalizers instead |
| Hardcoded namespaces/resource names | Make configurable, use deterministic naming formulas |
| Updating status every reconcile even when unchanged | Compare before updating, use helpers that return changed bool |
| No predicates on watches | Add `GenerationChangedPredicate` at minimum |

---

When sources disagree, prefer: controller-runtime source code > Kubebuilder Book > community guides. See [SOURCES.md](SOURCES.md) for the full authority hierarchy.

---

## Behavioral Rules for Claude

When helping with operator code:

1. **Always read the existing code first.** Understand the current reconciliation structure before suggesting changes.

2. **Check for the anti-patterns table above.** If you spot one, flag it explicitly with the fix.

3. **When writing a new Reconcile function**, follow the canonical 6-step structure. Include comments marking each step.

4. **When adding status conditions**, use `[]metav1.Condition` with proper struct tags. Set `observedGeneration`. Use `Status().Update()`.

5. **When writing tests**, use envtest. Use `Eventually` for state assertions. Create isolated namespaces per test.

6. **When adding watches**, always consider whether a predicate is needed. Default to including `GenerationChangedPredicate` for the primary resource.

7. **When creating child resources**, set owner references via `ctrl.SetControllerReference`. Use deterministic naming.

8. **When debugging infinite reconciliation loops**, check: (a) status updates without change detection, (b) missing predicates, (c) non-deterministic serialization (map ordering), (d) SyncPeriod interaction with predicates.

9. **Consult reference files** for detailed patterns when the quick reference above isn't sufficient. Look up the correct pattern rather than guessing.

10. **Cite the rule** when flagging an issue. E.g., "This violates Core Principle #2 (idempotency) because..."

---

## Review Context for Agents

> **Purpose:** This section is designed for injection by orchestration skills (e.g., dev-team). It provides a self-contained operator review checklist and condensed anti-patterns without requiring agents to read the full skill or reference files.

### Operator Review Checklist

When reviewing Kubernetes operator code (Go, controller-runtime, Kubebuilder, Operator SDK), check every item below. Flag violations with the rule ID.

**Reconciliation (R1-R6)**
- **R1** Reconcile is level-based — no branching on event type (create/update/delete). Request contains only name/namespace.
- **R2** Reconcile is idempotent — running N times with same state produces same result. Uses "if exists and matches, skip" guards.
- **R3** Status reconstructed from world state — never reads own `.status` to decide what to do next.
- **R4** One controller per Kind — does not reconcile multiple GVKs in one controller.
- **R5** No blocking waits — returns `RequeueAfter` for pending operations, checks progress on next reconcile.
- **R6** No `Requeue: true` — uses `RequeueAfter` with explicit duration instead.

**Status & Conditions (S1-S5)**
- **S1** Uses `[]metav1.Condition` (not single `.status.phase` as primary mechanism).
- **S2** Condition struct has all required tags: `+listType=map`, `+listMapKey=type`, `+patchStrategy=merge`, `+patchMergeKey=type`, `+optional`, and JSON/patch struct tags.
- **S3** `ObservedGeneration` set to `obj.Generation` on every condition update.
- **S4** Status updated via `Status().Update()` or `Status().Patch()` — never `Update()` on the main object.
- **S5** Status compared before updating — avoids writes when nothing changed (prevents infinite reconciliation loops).

**Resource Management (M1-M4)**
- **M1** Owner references set on in-cluster children via `ctrl.SetControllerReference()`.
- **M2** No cross-namespace owner references — uses finalizers instead.
- **M3** Finalizers present when external resources need cleanup — added before creating external resources, removed only after cleanup succeeds.
- **M4** Uses `Patch()` or Server-Side Apply over full `Update()` where possible.

**Watches & Performance (W1-W3)**
- **W1** `GenerationChangedPredicate` applied on primary resource watch (prevents reconciliation on status-only changes).
- **W2** Child/related object events mapped to parent via `handler.EnqueueRequestForOwner` or `handler.EnqueueRequestsFromMapFunc`.
- **W3** No unfiltered cluster-wide watches without predicates.

**Testing (T1-T4)**
- **T1** Integration tests use envtest — not fake clients (`client.NewFakeClient()` is an anti-pattern).
- **T2** Async assertions use `Eventually`/`Consistently` — never `time.Sleep`.
- **T3** Two-client pattern: direct `client.New()` for test assertions, `mgr.GetClient()` for the reconciler.
- **T4** Namespace isolation: each test creates a unique namespace.

**Security (X1-X2)**
- **X1** RBAC annotations match actual API calls — no over-privileged roles.
- **X2** Containers run as non-root with `allowPrivilegeEscalation: false` and capabilities dropped.

### Key Anti-Patterns to Flag

| ID | Anti-Pattern | What to Look For | Fix |
|---|---|---|---|
| AP1 | Event-driven reconciliation | Separate create/update/delete handlers or event-type branching | Single level-based reconcile: read state, compute diff, converge |
| AP2 | Infinite reconciliation loop | Status updated every reconcile without change detection; non-deterministic map serialization; missing predicates | Compare before updating; sort maps; add `GenerationChangedPredicate` |
| AP3 | Reading status to decide actions | `if obj.Status.Phase == "Running"` driving reconciler logic | Observe actual child resource state; update status as last step |
| AP4 | Fake client tests | `client.NewFakeClient()`, mock clients, asserting on API call counts | envtest with real API server; assert on state, not calls |
| AP5 | Long-running reconcile | Polling loops, `time.Sleep`, waiting for external API inside Reconcile | Return `RequeueAfter`; check progress on next reconcile |
| AP6 | Writing CR spec from operator | Controller modifying `.spec` of its own CR | Use admission webhooks for defaults; communicate via status |
| AP7 | Multiple operators for same CRD | Two controllers reconciling the same GVK | One operator per CRD on a cluster |
| AP8 | Missing finalizer for external resources | No finalizer when CR manages cloud/external resources | Add finalizer before creating external resources |

### Severity Guide for Reviewers

- **Critical**: R1-R3 violations, AP1-AP3, S4 (wrong status update method), M2 (cross-namespace owner refs), AP6
- **Important**: R5-R6, S1-S3, S5, M1, M3-M4, W1, T1-T2, AP4-AP5, AP8
- **Suggestion**: W2-W3, T3-T4, X1-X2, AP7, deterministic naming, error classification (transient vs permanent)
