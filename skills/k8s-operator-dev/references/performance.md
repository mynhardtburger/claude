# Performance

## Split Client Architecture

The default controller-runtime client reads from a local shared cache (backed by informers) and writes directly to the API server.

```
GET/LIST -> Cache (informers) -> eventually consistent
CREATE/UPDATE/DELETE/PATCH -> API Server -> strongly consistent
```

**Key implication:** Objects retrieved from the cache are shared pointers. **Never modify them without copying first**, or you corrupt the cache for all controllers in the manager.

```go
// WRONG: modifying cache object directly
var obj appsv1.Deployment
_ = r.Get(ctx, key, &obj)
obj.Spec.Replicas = ptr.To(int32(5)) // Corrupts cache!
r.Update(ctx, &obj)

// Correct: deep copy first
var obj appsv1.Deployment
_ = r.Get(ctx, key, &obj)
updated := obj.DeepCopy()
updated.Spec.Replicas = ptr.To(int32(5))
r.Update(ctx, updated)
```

*Source: Operator SDK Common Recommendations, Tim Ebert's blog*

## Cache Optimization

### Strip Managed Fields

If you don't access `managedFields`, strip them to reduce memory:

```go
mgr, err := ctrl.NewManager(cfg, ctrl.Options{
    Cache: cache.Options{
        DefaultTransform: cache.TransformStripManagedFields(),
    },
})
```

### Metadata-Only Watches

For large objects where you only need metadata (e.g., watching Secrets for name/namespace only):

```go
ctrl.NewControllerManagedBy(mgr).
    For(&myv1.MyResource{}).
    WatchesMetadata(&corev1.Secret{}, handler.EnqueueRequestsFromMapFunc(r.mapSecretToParent)).
    Build(r)
```

This stores only `PartialObjectMetadata` in the cache, dramatically reducing memory for large or numerous objects.

### Per-Type Cache Configuration

```go
mgr, err := ctrl.NewManager(cfg, ctrl.Options{
    Cache: cache.Options{
        ByObject: map[client.Object]cache.ByObject{
            &corev1.ConfigMap{}: {
                Namespaces: map[string]cache.Config{
                    "my-namespace": {},
                },
            },
        },
    },
})
```

### Disable Caching Per Type

For objects with high update rates or large memory footprints, bypass the cache:

```go
// Use APIReader for direct API reads (last resort)
var obj myv1.LargeResource
if err := r.APIReader.Get(ctx, key, &obj); err != nil {
    return ctrl.Result{}, err
}
```

*Source: controller-runtime cache_options.md, controller-runtime use-selectors-at-cache.md*

## Field Indexing

Create indexes for efficient lookups of related resources. Without indexing, lookups require scanning all objects.

### Setup

```go
func SetupIndexes(mgr ctrl.Manager) error {
    return mgr.GetFieldIndexer().IndexField(
        context.Background(),
        &batchv1.Job{},
        jobOwnerKey,
        func(obj client.Object) []string {
            job := obj.(*batchv1.Job)
            owner := metav1.GetControllerOf(job)
            if owner == nil || owner.Kind != "MyResource" {
                return nil
            }
            return []string{owner.Name}
        },
    )
}
```

### Usage

```go
var jobs batchv1.JobList
if err := r.List(ctx, &jobs,
    client.InNamespace(req.Namespace),
    client.MatchingFields{jobOwnerKey: req.Name},
); err != nil {
    return ctrl.Result{}, err
}
```

This is O(1) lookup instead of O(n) filtering through all jobs.

*Source: Kubebuilder Controller Implementation*

## Predicates

### GenerationChangedPredicate

The most important predicate. Skips reconciliation when only metadata or status changed (not spec):

```go
ctrl.NewControllerManagedBy(mgr).
    For(&myv1.MyResource{}, builder.WithPredicates(predicate.GenerationChangedPredicate{})).
    Build(r)
```

`metadata.generation` is incremented by the API server only when writes are made to the spec. Status-only updates do not change generation.

**Caveat: metadata-only changes are filtered out.** `GenerationChangedPredicate` only overrides `Update` events — `Create` and `Delete` events pass through unchanged. However, some important changes are delivered as Update events with no generation increment:

- **DeletionTimestamp** — When an object with a finalizer is deleted, the API server sets `DeletionTimestamp` in metadata. The object is NOT removed (the finalizer keeps it alive), so the watch delivers this as an **Update** event, not a Delete event. `GenerationChangedPredicate` filters it out because generation didn't change. **This breaks finalizer cleanup and makes the object permanently undeletable.**
- **Labels and annotations** — If the controller reads labels or annotations from the primary resource to make decisions (e.g., pause annotations, opt-out flags), changes to those fields will be missed.
- **Finalizers** — If an external actor modifies finalizers on the primary resource, the controller won't see it.

**Rule of thumb:** Before applying `GenerationChangedPredicate` to a resource watch, audit the reconcile function for any dependency on metadata fields (DeletionTimestamp, labels, annotations, finalizers). If any exist, use a compound predicate:

```go
pred := predicate.Or(
    predicate.GenerationChangedPredicate{},
    // Pass through updates where DeletionTimestamp was just set
    predicate.Funcs{
        UpdateFunc: func(e event.UpdateEvent) bool {
            return e.ObjectOld.GetDeletionTimestamp().IsZero() &&
                !e.ObjectNew.GetDeletionTimestamp().IsZero()
        },
    },
    // Add further predicates for any other metadata dependencies
)
```

In practice, **any controller that uses finalizers MUST handle the DeletionTimestamp case** — combine `GenerationChangedPredicate` with a deletion-aware predicate, or write a custom predicate that passes both.

### Other Built-in Predicates

| Predicate | Skips when |
|---|---|
| `GenerationChangedPredicate` | Only metadata/status changed (not spec) |
| `AnnotationChangedPredicate` | Annotations didn't change |
| `LabelChangedPredicate` | Labels didn't change |
| `LabelSelectorPredicate` | Object doesn't match a label selector |

### Combining Predicates

```go
// Reconcile if generation OR annotations changed
pred := predicate.Or(
    predicate.GenerationChangedPredicate{},
    predicate.AnnotationChangedPredicate{},
)

ctrl.NewControllerManagedBy(mgr).
    For(&myv1.MyResource{}, builder.WithPredicates(pred)).
    Build(r)
```

### Custom Predicates

```go
pred := predicate.Funcs{
    CreateFunc: func(e event.CreateEvent) bool {
        return true // Always reconcile on create
    },
    UpdateFunc: func(e event.UpdateEvent) bool {
        // Only reconcile if a specific field changed
        old := e.ObjectOld.(*myv1.MyResource)
        new := e.ObjectNew.(*myv1.MyResource)
        return old.Spec.Replicas != new.Spec.Replicas
    },
    DeleteFunc: func(e event.DeleteEvent) bool {
        return true
    },
    GenericFunc: func(e event.GenericEvent) bool {
        return true
    },
}
```

### SyncPeriod Interaction

`SyncPeriod` (default: 10 hours) fires artificial Update events where `ObjectOld == ObjectNew`. Predicates that compare old vs new (like `GenerationChangedPredicate`) filter these out. Custom predicates that always return true will process them.

If you want periodic re-reconciliation, prefer `RequeueAfter` from your Reconcile function over reducing `SyncPeriod`.

*Source: controller-runtime predicate package, Operator SDK Event Filtering*

## MaxConcurrentReconciles

Controls maximum concurrent reconciliations per controller. Default: 1.

```go
ctrl.NewControllerManagedBy(mgr).
    For(&myv1.MyResource{}).
    WithOptions(controller.Options{
        MaxConcurrentReconciles: 5,
    }).
    Build(r)
```

**When to increase:**
- Controller manages many independent objects
- Reconciliation involves external API calls (I/O bound, not CPU bound)
- Individual reconciliations are fast but there are many objects

**When to keep at 1:**
- Reconciliations for different objects can interfere with each other
- Controller manages cluster-scoped singletons
- CPU-bound reconciliation logic

*Source: controller-runtime controller.go*

## Rate Limiting

### Default Behavior

The default rate limiter combines:
1. **Per-item exponential backoff:** 5ms initial, doubling up to ~16 minutes. Reset on success.
2. **Overall bucket rate limit:** 10 requeues per second average.

The actual delay is the **maximum** of these two.

### Customization

```go
import "sigs.k8s.io/controller-runtime/pkg/ratelimiter"
import "k8s.io/client-go/util/workqueue"

ctrl.NewControllerManagedBy(mgr).
    For(&myv1.MyResource{}).
    WithOptions(controller.Options{
        RateLimiter: workqueue.NewTypedMaxOfRateLimiter(
            workqueue.NewTypedItemExponentialFailureRateLimiter[reconcile.Request](
                1*time.Second,  // base delay
                30*time.Second, // max delay
            ),
            &workqueue.TypedBucketRateLimiter[reconcile.Request]{
                Limiter: rate.NewLimiter(rate.Limit(20), 100), // 20/sec, burst 100
            },
        ),
    }).
    Build(r)
```

*Source: controller-runtime controller.go*

## Metric Cardinality

When exposing custom Prometheus metrics:

```go
// Good: bounded cardinality
var reconcileErrors = prometheus.NewCounterVec(
    prometheus.CounterOpts{
        Name: "myoperator_reconcile_errors_total",
    },
    []string{"controller", "namespace"}, // bounded labels
)

// Bad: unbounded cardinality
var reconcileErrors = prometheus.NewCounterVec(
    prometheus.CounterOpts{
        Name: "myoperator_reconcile_errors_total",
    },
    []string{"pod_uid"}, // unbounded! grows with every pod
)
```

Avoid tagging metrics with high-cardinality labels (pod UID, full resource name in clusters with dynamic names). Use namespace, controller name, or CR kind as labels.

*Source: OuterByte 2025 Guide*

## Dealing with Stale Cache

Three strategies, in order of preference:

1. **Optimistic locking with deterministic names.** Use deterministic names so `Create` returns `AlreadyExists` if the object exists.

2. **Track and requeue.** Track actions taken, use `RequeueAfter` to verify they completed. If not, repeat.

3. **Direct API reads (last resort).** Use `APIReader` to bypass the cache. This increases API server load.

*Source: controller-runtime FAQ*
