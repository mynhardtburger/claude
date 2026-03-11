# Testing

## envtest Over Fake Clients

The controller-runtime project explicitly recommends envtest over fake clients:

> "In our experience, tests using fake clients gradually re-implement poorly-written impressions of a real API server, which leads to hard-to-maintain, complex test code."

envtest provides a lightweight test environment with a real API server and etcd, without a full cluster.

*Source: controller-runtime FAQ*

## Test State, Not API Calls

Structure tests to check that the **state of the world** matches expectations, not that specific API calls were made. This allows refactoring controller internals without changing tests.

```go
// Good: assert on state
Eventually(func(g Gomega) {
    var dep appsv1.Deployment
    g.Expect(k8sClient.Get(ctx, key, &dep)).To(Succeed())
    g.Expect(dep.Spec.Replicas).To(Equal(ptr.To(int32(3))))
}).Should(Succeed())

// Bad: assert on API calls
Expect(fakeClient.CreateCallCount()).To(Equal(1))
Expect(fakeClient.CreateArgsForCall(0)).To(BeAssignableToTypeOf(&appsv1.Deployment{}))
```

*Source: controller-runtime FAQ*

## Test Suite Setup

### Suite Configuration

```go
var (
    testEnv   *envtest.Environment
    k8sClient client.Client
    ctx       context.Context
    cancel    context.CancelFunc
)

var _ = BeforeSuite(func() {
    ctx, cancel = context.WithCancel(context.TODO())

    testEnv = &envtest.Environment{
        CRDDirectoryPaths:     []string{filepath.Join("..", "config", "crd", "bases")},
        ErrorIfCRDPathMissing: true,
    }

    cfg, err := testEnv.Start()
    Expect(err).NotTo(HaveOccurred())

    // Register scheme
    scheme := runtime.NewScheme()
    Expect(myv1.AddToScheme(scheme)).To(Succeed())
    Expect(clientgoscheme.AddToScheme(scheme)).To(Succeed())

    // Create client for test assertions (direct API client, not cached)
    k8sClient, err = client.New(cfg, client.Options{Scheme: scheme})
    Expect(err).NotTo(HaveOccurred())

    // Start the controller manager
    mgr, err := ctrl.NewManager(cfg, ctrl.Options{Scheme: scheme})
    Expect(err).NotTo(HaveOccurred())

    err = (&MyReconciler{
        Client: mgr.GetClient(),
        Scheme: mgr.GetScheme(),
    }).SetupWithManager(mgr)
    Expect(err).NotTo(HaveOccurred())

    go func() {
        defer GinkgoRecover()
        Expect(mgr.Start(ctx)).To(Succeed())
    }()
})

var _ = AfterSuite(func() {
    cancel()
    Expect(testEnv.Stop()).To(Succeed())
})
```

*Source: Kubebuilder Writing Tests*

## Two-Client Pattern

Use two distinct clients in tests:

| Client | Source | Purpose |
|---|---|---|
| `k8sClient` | `client.New(cfg, ...)` | Direct API reads for test assertions |
| `mgr.GetClient()` | Manager's cached client | Used by the reconciler (production behavior) |

**Why:** "When making assertions in tests, you generally want to assert against the live state of the API server. If you use the client from the manager, you'd end up asserting against the contents of the cache instead, which is slower and can introduce flakiness."

*Source: Kubebuilder Writing Tests*

## Async Assertions

Reconciliation is async. Always use Gomega's async matchers.

### Eventually -- Wait for Expected State

```go
// Wait for the controller to create a Deployment
Eventually(func(g Gomega) {
    var dep appsv1.Deployment
    g.Expect(k8sClient.Get(ctx, types.NamespacedName{
        Name:      "myresource-worker",
        Namespace: ns,
    }, &dep)).To(Succeed())
    g.Expect(dep.Spec.Replicas).To(Equal(ptr.To(int32(3))))
}).WithTimeout(10 * time.Second).WithPolling(250 * time.Millisecond).Should(Succeed())
```

### Consistently -- Verify Stable State

```go
// Verify the controller does NOT create an unwanted resource
Consistently(func(g Gomega) {
    var dep appsv1.Deployment
    err := k8sClient.Get(ctx, types.NamespacedName{
        Name:      "myresource-worker",
        Namespace: ns,
    }, &dep)
    g.Expect(apierrors.IsNotFound(err)).To(BeTrue())
}).WithTimeout(5 * time.Second).WithPolling(500 * time.Millisecond).Should(Succeed())
```

### Best Practices for Async Assertions

- Use the `func(g Gomega)` form so all assertions within are part of the retry loop
- Set explicit timeouts -- don't rely on defaults
- Use `WithPolling` to control check frequency
- Never use `time.Sleep` -- it makes tests slow and flaky

*Source: Kubebuilder Writing Tests*

## envtest Limitations

Know what envtest does NOT provide:

| Missing Feature | Impact | Workaround |
|---|---|---|
| No garbage collection | Objects with OwnerReference won't auto-delete | Assert on owner reference existence, not child object deletion |
| No namespace deletion | Namespaces go to Terminating but never finish | Create a new namespace per test with random suffix |
| No kubelet | Pods won't run, no node operations | Test reconciliation logic, not pod execution |
| No built-in controllers | Deployments won't create ReplicaSets | Test your controller's behavior, not downstream effects |

### Namespace Isolation

```go
var _ = Describe("MyController", func() {
    var ns string

    BeforeEach(func() {
        ns = fmt.Sprintf("test-%s", uuid.New().String()[:8])
        Expect(k8sClient.Create(ctx, &corev1.Namespace{
            ObjectMeta: metav1.ObjectMeta{Name: ns},
        })).To(Succeed())
    })

    // Tests use `ns` for isolation
})
```

*Source: Kubebuilder envtest Reference*

## Test Pyramid

| Level | Tool | What to test | Speed |
|---|---|---|---|
| Unit | `go test` | Pure functions, helpers, validation logic | Fast |
| Integration | envtest | Controller reconciliation, status updates, watches | Medium |
| E2E | KinD + real cluster | Full operator lifecycle, upgrades, multi-component | Slow |

### What to test at each level

**Unit tests:**
- Spec validation logic
- Status condition helpers
- Resource name generation
- Configuration parsing

**Integration tests (envtest):**
- CR creation triggers correct child resource creation
- CR update triggers correct child resource update
- CR deletion triggers finalizer cleanup
- Status conditions set correctly
- Error handling and requeue behavior

**E2E tests (KinD):**
- Full operator deployment and startup
- CRD installation and webhook registration
- Multi-resource interactions (garbage collection, owner references actually working)
- Upgrade scenarios
- Prometheus and cert-manager integration

Reserve Prometheus and cert-manager testing for E2E, not envtest.

*Source: Operator SDK Common Recommendations*

## Detecting Flaky Tests

```bash
# Run tests repeatedly until failure
ginkgo --until-it-fails ./...

# Run with verbose output for debugging
ginkgo -v ./...
```

Common causes of flakiness:
- Missing `Eventually` (asserting immediately after creation)
- Shared state between tests (use namespace isolation)
- Hardcoded timeouts that are too short for CI
- Cache vs direct client confusion (use two-client pattern)

*Source: Kubebuilder, InfraCloud*
