# Reconciliation Patterns

## Level-Based vs Edge-Based

Reconciliation is **level-based**: the reconcile function reacts to the current state of the world, not to the specific event that triggered it. The `Request` contains only a name and namespace -- there is no indication of what changed or why.

**Do this:**
```go
func (r *MyReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    var obj myv1.MyResource
    if err := r.Get(ctx, req.NamespacedName, &obj); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    // Observe current state, compute desired state, converge
}
```

**Not this:**
```go
// WRONG: Event-driven reconciliation
func (r *MyReconciler) OnCreate(obj *myv1.MyResource) { ... }
func (r *MyReconciler) OnUpdate(old, new *myv1.MyResource) { ... }
func (r *MyReconciler) OnDelete(obj *myv1.MyResource) { ... }
```

*Source: controller-runtime reconcile.go, Kubebuilder Good Practices, Operator SDK Common Recommendations*

## Idempotency Requirements

Every reconcile call must produce the same outcome given the same cluster state. To verify:

1. **No side effects on repeated runs.** Run Reconcile twice with the same state -- the second run should be a no-op.
2. **No reliance on call order.** Events may be coalesced, skipped, or reordered.
3. **Safe across restarts.** If the operator crashes mid-reconcile, the next reconcile must produce correct results.
4. **Use "if exists and matches, skip" guards** before creating or updating resources.

```go
// Idempotent child resource creation
var dep appsv1.Deployment
err := r.Get(ctx, types.NamespacedName{Name: desired.Name, Namespace: desired.Namespace}, &dep)
if apierrors.IsNotFound(err) {
    return r.Create(ctx, desired)
}
if err != nil {
    return err
}
// Update only if spec differs
if !equality.Semantic.DeepEqual(dep.Spec, desired.Spec) {
    dep.Spec = desired.Spec
    return r.Update(ctx, &dep)
}
```

*Source: controller-runtime FAQ, Kubebuilder Good Practices, Operator SDK Common Recommendations*

## Canonical Reconciliation Structure

Follow this 6-step structure for complex reconcilers:

```go
func (r *MyReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := log.FromContext(ctx)

    // Step 1: Fetch the CR instance
    var obj myv1.MyResource
    if err := r.Get(ctx, req.NamespacedName, &obj); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // Step 2: Handle deletion (finalizer cleanup)
    if !obj.DeletionTimestamp.IsZero() {
        return r.handleDeletion(ctx, &obj)
    }

    // Step 3: Add finalizer if needed
    if !controllerutil.ContainsFinalizer(&obj, myFinalizer) {
        controllerutil.AddFinalizer(&obj, myFinalizer)
        if err := r.Update(ctx, &obj); err != nil {
            return ctrl.Result{}, err
        }
    }

    // Step 4: Validate the instance
    if err := obj.Validate(); err != nil {
        // Set a condition indicating invalid spec
        return ctrl.Result{}, reconcile.TerminalError(err)
    }

    // Step 5: Reconcile child resources
    if result, err := r.reconcileResources(ctx, &obj); err != nil || result.RequeueAfter > 0 {
        return result, err
    }

    // Step 6: Update status from observed state
    return r.updateStatus(ctx, &obj)
}
```

*Source: Red Hat, Operator SDK, Kubebuilder*

## Requeue Patterns

### Result Semantics

| Return | Behavior | Use when |
|---|---|---|
| `ctrl.Result{}, nil` | Remove from queue | Reconciliation complete, nothing pending |
| `ctrl.Result{RequeueAfter: d}, nil` | Requeue after duration `d` | Waiting for external operation, polling |
| `ctrl.Result{}, err` | Exponential backoff (5ms -> ~16min) | Transient errors (API conflicts, network) |
| `ctrl.Result{}, reconcile.TerminalError(err)` | Do NOT retry, log + metrics | Permanent errors (invalid spec, unsupported config) |

### Important Notes

- **`Requeue: true` is deprecated.** It uses the workqueue's rate limiter designed for error retry. Use `RequeueAfter` with an explicit duration.
- **When returning an error**, the Result is ignored entirely. The request goes through exponential backoff.
- **The only exception** to error-based retry is `TerminalError`, which stops retries.
- **Deadline-based requeuing**: Calculate `time.Until(nextEvent)` and return that as `RequeueAfter`. Avoids unnecessary polling.

*Source: controller-runtime reconcile.go, Kubebuilder Controller Implementation*

## Subreconciler Pattern

For complex operators with many child resources, break the monolithic `Reconcile()` into independent subreconcilers:

```go
func (r *MyReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    var obj myv1.MyResource
    if err := r.Get(ctx, req.NamespacedName, &obj); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    subreconcilers := []func(context.Context, *myv1.MyResource) (ctrl.Result, error){
        r.reconcileService,
        r.reconcileDeployment,
        r.reconcileConfigMap,
        r.reconcileStatus,
    }

    for _, sub := range subreconcilers {
        result, err := sub(ctx, &obj)
        if err != nil || result.RequeueAfter > 0 {
            return result, err
        }
    }

    return ctrl.Result{}, nil
}
```

Each subreconciler:
- Handles one concern (one child resource type or status)
- Re-fetches the CR at its start if it mutated the CR in a prior step
- Returns its own Result and error independently

**Trade-off:** More API server reads (one GET per subreconciler), but dramatically improved readability, testability, and maintainability.

*Source: Red Hat (Matthias Goerens), Operator Framework community*

## Re-fetch After Mutations

After any mutation (`Update`, `Status().Update()`, `Patch`), re-fetch the object before further work:

```go
if err := r.Status().Update(ctx, &obj); err != nil {
    return ctrl.Result{}, err
}
// Re-fetch to get the latest resourceVersion
if err := r.Get(ctx, req.NamespacedName, &obj); err != nil {
    return ctrl.Result{}, err
}
// Now safe to continue working with obj
```

This prevents optimistic concurrency conflicts from stale `resourceVersion` values.

*Source: Kubebuilder Controller Implementation*

## Deterministic Naming

Use deterministic names for child resources to leverage Kubernetes optimistic locking:

```go
// Good: deterministic name derived from parent
name := fmt.Sprintf("%s-worker-%d", parent.Name, index)

// Good: hash-based name for config-dependent resources
hash := sha256.Sum256([]byte(configData))
name := fmt.Sprintf("%s-%s", parent.Name, hex.EncodeToString(hash[:8]))
```

When you can't use deterministic names (e.g., `generateName`), track actions taken and assume they need to be repeated if results don't appear after a given time.

*Source: controller-runtime FAQ*
