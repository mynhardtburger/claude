# Resource Management

## Owner References

### When to Use

Set owner references on **in-cluster child resources** that are in the **same namespace** as the parent. This enables:
- Automatic garbage collection (children deleted when parent is deleted)
- Watch propagation (child changes trigger parent reconciliation via `handler.EnqueueRequestForOwner`)

```go
dep := &appsv1.Deployment{
    ObjectMeta: metav1.ObjectMeta{
        Name:      fmt.Sprintf("%s-worker", parent.Name),
        Namespace: parent.Namespace,
    },
    Spec: desiredSpec,
}

// Set the parent as the controller owner
if err := ctrl.SetControllerReference(parent, dep, r.Scheme); err != nil {
    return ctrl.Result{}, err
}
```

### Cross-Namespace Prohibition

Cross-namespace owner references are **disallowed by design**. A namespaced owner must be in the same namespace as its dependents. Violations cause unpredictable garbage collection behavior.

For cross-namespace relationships, use finalizers and explicit cleanup logic instead.

### Controller Reference vs Owner Reference

- `ctrl.SetControllerReference()`: Sets the owner AND marks it as the controller. Only one controller reference allowed per object. This is what you usually want.
- `controllerutil.SetOwnerReference()`: Sets just the owner reference without the controller flag. Use when multiple parents need to be notified of changes, but only one controls the lifecycle.

*Source: Kubebuilder Controller Implementation, Kubernetes docs, CNCF White Paper*

## Finalizers

### When to Use

Use finalizers ONLY when:
- External resources (cloud storage, DNS, databases) need cleanup on CR deletion
- Cross-namespace resources need cleanup
- Resources not owned by the CR need cleanup

Do NOT use finalizers for in-cluster same-namespace children -- use owner references instead.

### Standard Implementation

```go
const myFinalizer = "mygroup.example.com/finalizer"

func (r *MyReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    var obj myv1.MyResource
    if err := r.Get(ctx, req.NamespacedName, &obj); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // Handle deletion
    if !obj.DeletionTimestamp.IsZero() {
        if controllerutil.ContainsFinalizer(&obj, myFinalizer) {
            // Perform cleanup
            if err := r.cleanupExternalResources(ctx, &obj); err != nil {
                return ctrl.Result{}, err // Retry on failure
            }
            // Remove finalizer ONLY after cleanup succeeds
            controllerutil.RemoveFinalizer(&obj, myFinalizer)
            if err := r.Update(ctx, &obj); err != nil {
                return ctrl.Result{}, err
            }
        }
        return ctrl.Result{}, nil
    }

    // Add finalizer if not present
    if !controllerutil.ContainsFinalizer(&obj, myFinalizer) {
        controllerutil.AddFinalizer(&obj, myFinalizer)
        if err := r.Update(ctx, &obj); err != nil {
            return ctrl.Result{}, err
        }
    }

    // ... rest of reconciliation
}
```

### Key Rules

1. **Add finalizer early** -- before any external resources are created.
2. **Remove finalizer last** -- only after all cleanup has succeeded.
3. **Return error on cleanup failure** -- this retries the reconciliation. The finalizer prevents the object from being garbage collected until cleanup succeeds.
4. **Without a finalizer, deletion = gone.** The object disappears from the cache. You get a reconcile where `Get` returns NotFound, with no opportunity to run cleanup.
5. **Multiple finalizers are allowed.** Each represents a different cleanup responsibility.
6. **Predicate compatibility.** `GenerationChangedPredicate` blocks the Update event that sets `DeletionTimestamp` (generation does not change on deletion). If you use finalizers, do NOT use a bare `GenerationChangedPredicate` on the primary resource — combine it with a predicate that passes deletion-timestamp changes through. See the [performance reference](performance.md#generationchangedpredicate) for the compound predicate pattern.

### Async Cleanup

If cleanup triggers an async operation (e.g., a Job):

```go
if !obj.DeletionTimestamp.IsZero() {
    if controllerutil.ContainsFinalizer(&obj, myFinalizer) {
        // Check if cleanup job completed
        if !r.cleanupComplete(ctx, &obj) {
            // Ensure cleanup job exists
            if err := r.ensureCleanupJob(ctx, &obj); err != nil {
                return ctrl.Result{}, err
            }
            // Requeue to check later
            return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
        }
        // Cleanup done, remove finalizer
        controllerutil.RemoveFinalizer(&obj, myFinalizer)
        if err := r.Update(ctx, &obj); err != nil {
            return ctrl.Result{}, err
        }
    }
    return ctrl.Result{}, nil
}
```

*Source: Operator SDK Best Practices, Operator SDK Advanced Topics, Kubebuilder*

## Server-Side Apply (SSA)

### When to Use SSA

SSA is the recommended approach for controllers managing child resources. It eliminates the need for separate Get/Create and Get/Update patterns.

**Advantages:**
- No read-modify-write race conditions
- No clobbering of fields owned by other controllers
- A single `Patch` call handles both create and update
- Clear field ownership via `FieldManager`

### How to Use SSA

```go
desired := &appsv1.Deployment{
    TypeMeta: metav1.TypeMeta{
        APIVersion: "apps/v1",
        Kind:       "Deployment",
    },
    ObjectMeta: metav1.ObjectMeta{
        Name:      fmt.Sprintf("%s-worker", parent.Name),
        Namespace: parent.Namespace,
    },
    Spec: appsv1.DeploymentSpec{
        // ... compute desired spec from scratch
    },
}

if err := r.Patch(ctx, desired, client.Apply, client.FieldOwner("mycontroller"), client.ForceOwnership); err != nil {
    return ctrl.Result{}, err
}
```

### Key Rules

1. **Reconstruct desired state from scratch** on every reconcile. Do not read the existing object and modify it.
2. **Always set `Force: true`** (`client.ForceOwnership`) for objects your controller fully owns.
3. **Use a unique FieldManager name** per controller (not per reconcile).
4. **TypeMeta is required** -- SSA needs APIVersion and Kind set on the object.
5. **Use apply configuration types** when available (e.g., `appsv1ac.Deployment()`) for better zero-value handling.

### SSA with GitOps

When operators coexist with GitOps controllers (ArgoCD, Flux):
- Let the GitOps controller own the CR spec (FieldManager: "argocd")
- Let the operator own child resources (FieldManager: "myoperator")
- Distinct FieldManager values prevent reconciliation wars

*Source: Kubernetes blog, OuterByte 2025 Guide, cert-manager, controller-runtime*

## Patch vs Update

| Method | Use when | Pros | Cons |
|---|---|---|---|
| `Update` (PUT) | You own the entire object | Simple | Clobbers fields set by others |
| `Patch` (strategic merge) | Modifying specific fields | Doesn't clobber other fields | More complex to construct |
| `Patch` (SSA) | Managing child resources | No read-modify-write, clear ownership | Requires TypeMeta, FieldManager |
| `Status().Update()` | Updating status only | Targets subresource, doesn't touch spec | Full status replacement |
| `Status().Patch()` | Updating status fields | Targeted, avoids conflicts | More complex |

**General rule:** Prefer `Patch` over `Update` to facilitate composition and concurrency of controllers.

*Source: Kubernetes API Conventions*

## Deterministic Naming

Use deterministic names for child resources to leverage Kubernetes optimistic locking:

```go
// Derive name from parent + purpose
name := fmt.Sprintf("%s-config", parent.Name)

// Derive name from content hash for config-dependent resources
hash := fnv.New32a()
hash.Write([]byte(configData))
name := fmt.Sprintf("%s-%x", parent.Name, hash.Sum32())
```

Benefits:
- `Create` will return `AlreadyExists` if the object exists -- no need to check first
- Avoids creating duplicate resources on retries
- Makes it easy to identify which parent owns which child

When you must use `generateName`, track which resources you created (e.g., via labels or annotations) and verify they exist on subsequent reconciles.

*Source: controller-runtime FAQ*
