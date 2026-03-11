# Anti-Patterns

Common anti-patterns in Kubernetes operator development. Each entry explains what it is, why it's wrong, and what to do instead.

---

## Reconciliation Anti-Patterns

### 1. Event-Driven Reconciliation

**What:** Writing separate handlers for create, update, and delete events instead of a single level-based reconcile function.

**Why it's wrong:** Breaks the fundamental controller-runtime design. Events may be coalesced, skipped, or reordered. Event-specific logic leads to resources becoming stuck and requiring manual intervention.

**Do instead:** Read current state in every reconcile call. Compute desired state. Converge. The Request only contains a name/namespace -- treat every call identically.

*Source: Kubebuilder Good Practices, Operator SDK Common Recommendations, controller-runtime FAQ*

### 2. Long-Running Reconciliations

**What:** Blocking the reconcile loop waiting for operations to complete (polling until a Pod is ready, waiting for an external API).

**Why it's wrong:** Blocks processing of other resources. With default `MaxConcurrentReconciles: 1`, one stuck reconciliation halts the entire controller.

**Do instead:** Check current state, set status, return `RequeueAfter` with an appropriate interval. Check progress on the next reconcile.

*Source: Red Hat, Operator Framework Google Group, cert-manager scaling docs*

### 3. Infinite Reconciliation Loops

**What:** Status update triggers a watch event, which triggers a reconcile, which updates status, endlessly.

**Why it's wrong:** Wastes CPU and API server bandwidth. Can cascade to impact other controllers.

**Common causes:**
- Updating status on every reconcile without checking for changes
- Non-deterministic map serialization (Go map iteration order is random)
- Missing `GenerationChangedPredicate`

**Do instead:** Compare status before updating. Use deterministic serialization. Apply `GenerationChangedPredicate` on the primary resource watch.

*Source: ArgoCD issue #19675, controller-runtime docs*

### 4. Reading Status to Decide Actions

**What:** Using `obj.Status.Phase` or `obj.Status.Conditions` to determine what the reconciler should do next.

**Why it's wrong:** Status should be reconstructable from world state. If the status is lost or incorrect, the controller should still work correctly by observing actual state.

**Do instead:** Observe the actual state of child resources and external systems. Compute what needs to happen. Update status as the last step to reflect what you observed.

*Source: Kubebuilder Controller Implementation*

### 5. Using `Requeue: true`

**What:** Returning `ctrl.Result{Requeue: true}` to re-process an item.

**Why it's wrong:** Deprecated by controller-runtime. Uses the workqueue's rate limiter (designed for error retry), which is confusing and unpredictable. Developers usually know exactly when they want to re-check.

**Do instead:** Use `ctrl.Result{RequeueAfter: duration}` with an explicit interval.

*Source: controller-runtime reconcile.go*

### 6. Reconciling Without Predicates

**What:** Processing every watch event without filtering.

**Why it's wrong:** Causes unnecessary reconciliations on status-only or metadata-only changes. Wastes CPU and increases API server load. Particularly bad for cluster-wide watches.

**Do instead:** Apply `GenerationChangedPredicate` as a starting point, but **audit your reconcile function for metadata dependencies first**. `GenerationChangedPredicate` filters out all Update events where generation didn't change, which includes DeletionTimestamp being set (breaks finalizers), label changes, and annotation changes. If the controller uses finalizers or reads metadata fields, combine `GenerationChangedPredicate` with predicates that pass those changes through. See the [performance reference](performance.md#generationchangedpredicate) for details and the compound predicate pattern.

*Source: controller-runtime docs, OuterByte 2025 Guide*

---

## Architecture Anti-Patterns

### 7. One Controller Managing Multiple CRDs

**What:** A single reconciler handling multiple GVKs (e.g., reconciling both Databases and DatabaseUsers in one controller).

**Why it's wrong:** Violates encapsulation, single responsibility, and cohesion. Causes scalability bottlenecks, race conditions, error cascades across unrelated resources, and maintenance difficulty.

**Do instead:** One controller per Kind. Compose multiple controllers in the same manager binary. Use watches with `EnqueueRequestsFromMapFunc` for cross-resource coordination.

*Source: Kubebuilder Good Practices, Operator SDK Common Recommendations, controller-runtime FAQ*

### 8. Multiple Operators Managing the Same CRD

**What:** Two operators watching and reconciling the same CRD on a cluster.

**Why it's wrong:** Causes reconciliation conflicts, unpredictable behavior, and potential infinite update loops as each operator tries to enforce its view.

**Do instead:** Only one operator should control a CRD on a cluster.

*Source: Operator SDK Best Practices, CNCF White Paper*

### 9. Meta/Super Operators

**What:** An operator that deploys or manages other operators.

**Why it's wrong:** Operator lifecycle management is OLM's job. Embedding this in an operator creates circular dependencies, upgrade complexity, and RBAC escalation.

**Do instead:** Use OLM's Dependency Resolution for operator dependencies.

*Source: Operator SDK Best Practices*

### 10. Writing CR Spec from the Operator

**What:** The operator modifying the `.spec` of the CR it's reconciling.

**Why it's wrong:** Violates the user/controller contract (spec = user intent, status = controller observation). Breaks GitOps workflows. Can trigger infinite reconciliation loops.

**Do instead:** Set defaults via admission webhooks. Communicate state via status conditions. Never modify spec from the reconciler.

*Source: CNCF White Paper, Kubernetes API Conventions*

---

## Resource Management Anti-Patterns

### 11. Deleting CRs Without Finalizers (When External Resources Exist)

**What:** Not using finalizers when the CR manages external resources (cloud storage, DNS, databases).

**Why it's wrong:** Without a finalizer, deletion = gone. The object disappears from the cache, and you have no opportunity to clean up external resources. Orphaned cloud resources are expensive and insecure.

**Do instead:** Add finalizers before creating external resources. Remove finalizers only after cleanup succeeds.

*Source: Operator SDK Best Practices, CNCF White Paper*

### 12. Cross-Namespace Owner References

**What:** Setting owner references across namespace boundaries (e.g., a namespaced resource owned by a resource in a different namespace).

**Why it's wrong:** Disallowed by Kubernetes design. Causes unpredictable garbage collection behavior.

**Do instead:** Use finalizers and explicit cleanup logic for cross-namespace relationships.

*Source: Kubernetes docs, CNCF White Paper*

### 13. Full PUT When Patch Would Suffice

**What:** Using `Update()` (HTTP PUT) to modify objects when only specific fields need to change.

**Why it's wrong:** Clobbers fields set by other controllers, users, or admission webhooks. Creates race conditions in multi-controller environments.

**Do instead:** Use `Patch()` or Server-Side Apply. These modify only the fields you specify, respecting other controllers' field ownership.

*Source: Kubernetes API Conventions*

### 14. Hardcoded Namespaces and Resource Names

**What:** Hardcoding namespace names, resource names, or other environmental assumptions.

**Why it's wrong:** Breaks portability. Prevents multi-tenant deployments. Makes testing harder.

**Do instead:** Make namespace watching configurable. Derive resource names from the parent CR. Use environment variables or CR spec for configuration.

*Source: Operator SDK Best Practices*

---

## Security Anti-Patterns

### 15. Cluster-Scoped RBAC When Namespace-Scoped Suffices

**What:** Using `ClusterRole`/`ClusterRoleBinding` when the operator only needs access to one or a few namespaces.

**Why it's wrong:** Amplifies security blast radius. A compromised operator with cluster-wide permissions can access all namespaces.

**Do instead:** Use namespace-scoped `Role`/`RoleBinding` whenever possible. Only escalate to ClusterRole when genuinely needed (e.g., cluster-scoped CRDs).

*Source: CNCF White Paper, Operator SDK Best Practices*

### 16. Running Containers as Root

**What:** Not setting security context, running with default (root) user, keeping all Linux capabilities.

**Why it's wrong:** Critical security vulnerability. A container escape gives root access to the node.

**Do instead:**
```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
  readOnlyRootFilesystem: true
```

*Source: OuterByte 2025 Guide, CNCF White Paper*

### 17. Self-Registering CRDs

**What:** The operator installs or updates its own CRD definitions at startup.

**Why it's wrong:** CRDs are global resources requiring careful consideration. Self-registration requires elevated privileges and creates chicken-and-egg problems during upgrades.

**Do instead:** Ship CRDs separately (Helm chart, Kustomize, OLM bundle). Install CRDs before deploying the operator.

*Source: Operator SDK Best Practices*

---

## Testing Anti-Patterns

### 18. Using Fake Clients

**What:** Using `client.NewFakeClient()` or mock clients instead of envtest.

**Why it's wrong:** Fake clients gradually re-implement a poorly-written API server. Tests become complex, fragile, and don't catch real API server behavior (validation, defaulting, conflict detection).

**Do instead:** Use envtest for integration tests. Use pure unit tests for business logic functions that don't interact with the API.

*Source: controller-runtime FAQ*

### 19. Asserting Immediately After Creation

**What:** Creating a resource and immediately asserting on its effects without waiting for reconciliation.

**Why it's wrong:** Reconciliation is async. The controller may not have processed the event yet.

**Do instead:** Use `Eventually` for all assertions that depend on reconciliation:
```go
Eventually(func(g Gomega) {
    var obj myv1.MyResource
    g.Expect(k8sClient.Get(ctx, key, &obj)).To(Succeed())
    g.Expect(obj.Status.Ready).To(BeTrue())
}).Should(Succeed())
```

*Source: Kubebuilder Writing Tests*

### 20. Using `time.Sleep` in Tests

**What:** Using `time.Sleep` to wait for reconciliation instead of async assertions.

**Why it's wrong:** Too short = flaky tests. Too long = slow tests. Neither adapts to actual completion time.

**Do instead:** Use `Eventually` with appropriate timeout and polling intervals.

*Source: Kubebuilder Writing Tests*

### 21. Unfiltered High-Cardinality Metrics

**What:** Tagging Prometheus metrics with unbounded labels like pod UID, full resource paths, or user-supplied values.

**Why it's wrong:** Metric cardinality grows unboundedly. Overwhelms monitoring systems (Prometheus, Grafana). Increases memory usage and query times.

**Do instead:** Use bounded labels: namespace, controller name, CR kind, error category. Never use UIDs or user-supplied strings as metric labels.

*Source: OuterByte 2025 Guide*
