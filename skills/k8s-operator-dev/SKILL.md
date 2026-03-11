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

9. **Consult reference files** for detailed patterns when the quick reference above isn't sufficient. Don't guess -- look up the correct pattern.

10. **Cite the rule** when flagging an issue. E.g., "This violates Core Principle #2 (idempotency) because..."
