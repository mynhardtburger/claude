# Status and Condition Management

## Use Conditions, Not State Machines

Do NOT use a single `.status.phase` field with state machine semantics. Use `[]metav1.Condition` for an open-world model where multiple orthogonal conditions can be true simultaneously.

```go
// Good: conditions array
type MyResourceStatus struct {
    // +listType=map
    // +listMapKey=type
    // +patchStrategy=merge
    // +patchMergeKey=type
    // +optional
    Conditions []metav1.Condition `json:"conditions,omitempty" patchStrategy:"merge" patchMergeKey:"type"`
}

// Avoid: single phase field as primary status
type MyResourceStatus struct {
    Phase string `json:"phase"` // Don't rely on this as primary mechanism
}
```

Status must be 100% reconstructable by observation. Any history kept through conditions is just an optimization, not required for correct operation.

*Source: Kubernetes API Conventions, Kubebuilder, Maelvls blog, cert-manager*

## Start with "Ready"

Begin with a single `Ready` condition. Add more condition types only when they serve a specific downstream consumer.

```go
const (
    ConditionReady = "Ready"
)
```

For more complex operators, consider the OpenShift pattern:
- `Available` -- the operand is functional
- `Progressing` -- the operator is actively working toward desired state
- `Degraded` -- something is wrong but the operand may still be partially functional

*Source: Maelvls blog, Kubernetes community, OpenShift CVO*

## Condition Field Rules

Every condition must include all standard fields:

| Field | Rules |
|---|---|
| `Type` | CamelCase. Describes current observed state, not transitions. Use adjectives (`Ready`, `OutOfDisk`) or past-tense verbs (`Succeeded`, `Failed`). NOT present-tense (`Deploying`). |
| `Status` | `True`, `False`, or `Unknown`. |
| `Reason` | One-word CamelCase machine-readable category (e.g., `MinimumReplicasAvailable`). |
| `Message` | Human-readable explanation. |
| `LastTransitionTime` | When Status last **changed** (not when condition was last evaluated). |
| `ObservedGeneration` | MUST be set. Set to `obj.Generation`. Tells consumers if the condition is stale. |

### Polarity

Neither positive nor negative polarity is universally prescribed. Choose what makes sense:
- "MemoryExhausted" (negative) is clearer than "SufficientMemory"
- "Ready" (positive) is clearer than "NotFailed"

### Absence Means Unknown

If a known condition type is absent from the array, interpret it as `Unknown` -- reconciliation hasn't finished yet.

### Apply on First Visit

Set conditions (even with `Unknown` status) the first time a controller processes a resource. This signals that the controller is aware of the resource.

*Source: Kubernetes API Conventions*

## Standard Struct Annotations

Always use these struct tags and markers on your Conditions field:

```go
// +listType=map
// +listMapKey=type
// +patchStrategy=merge
// +patchMergeKey=type
// +optional
Conditions []metav1.Condition `json:"conditions,omitempty" patchStrategy:"merge" patchMergeKey:"type"`
```

These enable:
- Strategic merge patch to work correctly
- Server-side apply to merge conditions by type
- Proper CRD schema generation

*Source: Kubernetes API Conventions*

## Status Updates: Use the Subresource

Always update status via the status subresource:

```go
// Correct: targets the /status subresource
if err := r.Status().Update(ctx, &obj); err != nil {
    return ctrl.Result{}, err
}

// WRONG: updates the entire object, can clobber spec changes
if err := r.Update(ctx, &obj); err != nil {
    return ctrl.Result{}, err
}
```

The status subresource:
- Prevents accidental spec overwrites in read-modify-write scenarios
- Reduces conflicts with other controllers or users modifying spec
- Is enforced by RBAC separately (you can grant status-update without spec-update)

*Source: Kubebuilder Controller Implementation, Kubernetes API Conventions*

## Setting Conditions with ObservedGeneration

```go
import "k8s.io/apimachinery/pkg/api/meta"

meta.SetStatusCondition(&obj.Status.Conditions, metav1.Condition{
    Type:               ConditionReady,
    Status:             metav1.ConditionTrue,
    ObservedGeneration: obj.Generation,
    Reason:             "ReconcileSucceeded",
    Message:            "All child resources are healthy",
})
```

`meta.SetStatusCondition` handles:
- No duplicate conditions (replaces existing condition with same Type)
- Automatic `LastTransitionTime` (only updates when Status changes)
- Deterministic ordering

*Source: Kubernetes API Conventions, Operator SDK*

## Avoiding Infinite Reconciliation Loops

Infinite loops are one of the most common operator bugs. They occur when a status update triggers a new reconciliation, which updates status again, endlessly.

### Root Causes

1. **Updating status without change detection.** Always compare before updating:
```go
// Check if conditions actually changed before updating
existingCondition := meta.FindStatusCondition(obj.Status.Conditions, ConditionReady)
if existingCondition != nil &&
    existingCondition.Status == newCondition.Status &&
    existingCondition.Reason == newCondition.Reason &&
    existingCondition.Message == newCondition.Message {
    // No change, skip update
    return ctrl.Result{}, nil
}
```

2. **Non-deterministic serialization.** Map iteration order in Go is random. If status contains a map, the serialized JSON may differ each time even when the content is identical. This causes the API server to see a "change" and increment resourceVersion, triggering a new watch event.

   **Fix:** Sort maps before serialization, or use slices with deterministic ordering.

3. **Missing predicates.** Without `GenerationChangedPredicate`, a status-only update triggers a new reconcile:
```go
ctrl.NewControllerManagedBy(mgr).
    For(&myv1.MyResource{}, builder.WithPredicates(predicate.GenerationChangedPredicate{})).
    Build(r)
```
   **Warning:** A bare `GenerationChangedPredicate` filters out ALL metadata-only Update events, including the one that sets `DeletionTimestamp` (which breaks finalizer cleanup). If the controller uses finalizers or depends on metadata fields, use a compound predicate — see the [performance reference](performance.md#generationchangedpredicate).

4. **SyncPeriod interaction.** `SyncPeriod` fires artificial update events. Predicates that compare old vs new will filter these out, but custom predicates that always return true will not.

### The ArgoCD Lesson

ArgoCD ApplicationSet controller had an infinite reconciliation loop (issue #19675) caused by always writing status even when nothing changed. Map field ordering caused serialization differences on every write. Fix: compare status before updating, ensure deterministic serialization.

*Source: ArgoCD issue #19675, controller-runtime docs*

## Deterministic Condition Sorting

Use `meta.SetStatusCondition` from `k8s.io/apimachinery/pkg/api/meta` or Operator SDK's condition utilities. These ensure:
- No duplicate conditions
- Deterministic sort order (prevents spurious updates)
- Automatic `LastTransitionTime` management

*Source: Operator SDK Advanced Topics*
