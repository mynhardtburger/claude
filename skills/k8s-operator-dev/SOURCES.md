# Sources

Research that informed the k8s-operator-dev skill, organized by authority tier.

## Source Authority Hierarchy

When sources disagree, prefer higher-tier sources:

**Tier 1 -- Canonical** (defines the framework behavior):
- controller-runtime source code and FAQ
- Kubernetes API Conventions

**Tier 2 -- Official Guides** (teaches the framework):
- Kubebuilder Book
- Operator SDK Documentation
- CNCF Operator White Paper

**Tier 3 -- Production Evidence** (real-world validation):
- cert-manager, Prometheus Operator, ArgoCD patterns
- Google Cloud, Red Hat operator guides

---

## Tier 1 -- Canonical (defines framework behavior)

### controller-runtime (kubernetes-sigs)
- FAQ: https://github.com/kubernetes-sigs/controller-runtime/blob/main/FAQ.md
- reconcile.go (Reconciler interface, Result struct): https://github.com/kubernetes-sigs/controller-runtime/blob/main/pkg/reconcile/reconcile.go
- controller.go (MaxConcurrentReconciles, RateLimiter): https://github.com/kubernetes-sigs/controller-runtime/blob/main/pkg/controller/controller.go
- Cache Options Design Doc: https://github.com/kubernetes-sigs/controller-runtime/blob/main/designs/cache_options.md
- Cache Selectors Design Doc: https://github.com/kubernetes-sigs/controller-runtime/blob/main/designs/use-selectors-at-cache.md
- Predicate Package: https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/predicate
- Cache Package: https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/cache
- Builder Package: https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/builder
- Go Docs: https://pkg.go.dev/sigs.k8s.io/controller-runtime

### Kubernetes API Conventions
- https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md

### Kubernetes Official Documentation
- Operator Pattern: https://kubernetes.io/docs/concepts/extend-kubernetes/operator/
- Controllers: https://kubernetes.io/docs/concepts/architecture/controller/
- Custom Resources: https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/
- Garbage Collection: https://kubernetes.io/docs/concepts/architecture/garbage-collection/
- Finalizers: https://kubernetes.io/docs/concepts/overview/working-with-objects/finalizers/

---

## Tier 2 -- Official Guides (teaches the framework)

### Kubebuilder Book
- Good Practices: https://book.kubebuilder.io/reference/good-practices
- Controller Implementation (CronJob Tutorial): https://book.kubebuilder.io/cronjob-tutorial/controller-implementation.html
- Controller Overview: https://book.kubebuilder.io/cronjob-tutorial/controller-overview.html
- Writing Tests: https://book.kubebuilder.io/cronjob-tutorial/writing-tests
- Configuring envtest: https://book.kubebuilder.io/reference/envtest
- Using Finalizers: https://book.kubebuilder.io/reference/using-finalizers

### Operator SDK Documentation
- Operator Best Practices: https://sdk.operatorframework.io/docs/best-practices/best-practices/
- Common Recommendations: https://sdk.operatorframework.io/docs/best-practices/common-recommendation/
- Advanced Topics (Go): https://sdk.operatorframework.io/docs/building-operators/golang/advanced-topics/
- Event Filtering with Predicates: https://sdk.operatorframework.io/docs/building-operators/golang/references/event-filtering/
- Controller Runtime Client API: https://sdk.operatorframework.io/docs/building-operators/golang/references/client/
- Operator Capability Levels: https://sdk.operatorframework.io/docs/overview/operator-capabilities/
- Testing: https://sdk.operatorframework.io/docs/building-operators/golang/testing/

### CNCF Operator White Paper
- https://tag-app-delivery.cncf.io/whitepapers/operator/
- PDF: https://www.cncf.io/wp-content/uploads/2021/07/CNCF_Operator_WhitePaper.pdf
- GitHub: https://github.com/cncf/tag-app-delivery/blob/main/operator-whitepaper/v1/Operator-WhitePaper_v1-0.md

---

## Tier 3 -- Production Evidence (real-world validation)

### Production Operator References
- Prometheus Operator Design: https://prometheus-operator.dev/docs/getting-started/design/
- Prometheus Operator (GitHub): https://github.com/prometheus-operator/prometheus-operator
- cert-manager Best Practices: https://cert-manager.io/docs/installation/best-practice/
- cert-manager Scaling: https://cert-manager.io/docs/devops-tips/scaling-cert-manager/
- cert-manager issuer-lib (GitHub): https://github.com/cert-manager/issuer-lib
- ArgoCD Architecture: https://argo-cd.readthedocs.io/en/stable/operator-manual/architecture/
- ArgoCD Reconcile Optimization: https://argo-cd.readthedocs.io/en/stable/operator-manual/reconcile/
- ArgoCD Operator (Go Package): https://pkg.go.dev/github.com/argoproj-labs/argocd-operator/controllers/argocd

### Google / GKE
- Best Practices for Building Kubernetes Operators: https://cloud.google.com/blog/products/containers-kubernetes/best-practices-for-building-kubernetes-operators-and-stateful-apps
- Google Config Connector Overview: https://cloud.google.com/config-connector/docs/overview
- Google Config Connector Source (GitHub): https://github.com/GoogleCloudPlatform/k8s-config-connector

### Red Hat
- Kubernetes Operators Best Practices: https://www.redhat.com/en/blog/kubernetes-operators-best-practices
- Subreconciler Pattern in Action: https://www.redhat.com/en/blog/subreconciler-patterns-in-action
- OpenShift Operators in Golang: https://www.redhat.com/en/blog/red-hat-openshift-operators-concept-and-working-example-golang
- OpenShift Condition Management: https://deepwiki.com/openshift/custom-resource-status/2.3-condition-management-best-practices

### Server-Side Apply
- Kubernetes Blog (Advanced SSA): https://kubernetes.io/blog/2022/10/20/advanced-server-side-apply/
- Kubernetes Blog (SSA GA): https://kubernetes.io/blog/2021/08/06/server-side-apply-ga/
- Andreas Karis (SSA with controller-runtime): https://andreaskaris.github.io/blog/coding/server-side-apply/

### Status Conditions
- Maelvls (Conditions Deep Dive): https://maelvls.dev/kubernetes-conditions/
- Niklas Heidloff (Storing State with Conditions): https://heidloff.net/article/storing-state-status-kubernetes-resources-conditions-operators-go/

### Testing
- Marc Nuri (EnvTest in Go): https://blog.marcnuri.com/go-testing-kubernetes-applications-envtest
- InfraCloud (Testing with EnvTest): https://www.infracloud.io/blogs/testing-kubernetes-operator-envtest/
- envtest Go Package: https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/envtest

### Community Discussions and Analysis
- Operator Framework Google Group (Architecture Best Practices): https://groups.google.com/g/operator-framework/c/K7zwQiCJVYg/m/03NT2HYeCAAJ
- Tim Ebert (Controllers at Scale): https://medium.com/@timebertt/kubernetes-controllers-at-scale-clients-caches-conflicts-patches-explained-aa0f7a8b4332
- Diving Into Controller Runtime: https://vivilearns2code.github.io/k8s/2021/03/12/diving-into-controller-runtime.html
- OuterByte (Kubernetes Operators in 2025): https://outerbyte.com/kubernetes-operators-2025-guide/
- The New Stack (When to Use/Avoid Operator Pattern): https://thenewstack.io/kubernetes-when-to-use-and-when-to-avoid-the-operator-pattern/
- CloudARK Operator Maturity Model: https://cloudark.medium.com/introducing-kubernetes-operator-maturity-model-for-multi-operator-platforms-952d2e637a82
- Kubernetes Reconciliation Patterns: https://hkassaei.com/posts/kubernetes-and-reconciliation-patterns/
