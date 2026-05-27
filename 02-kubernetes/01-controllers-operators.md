# 2.1 · Custom Controllers & Operators

## Overview

A Kubernetes Operator is an application-specific controller that extends the Kubernetes API to create, configure, and manage instances of complex stateful applications. Built with the Operator pattern: CRD + controller loop.

---

## Core Concepts

### The Operator Pattern
- **CRD (Custom Resource Definition)** — extends the Kubernetes API with a new resource type
- **CR (Custom Resource)** — an instance of a CRD
- **Controller** — a reconcile loop that watches resources and drives actual state → desired state
- **Reconcile loop** — idempotent function called whenever relevant state changes

### Reconciliation
```
Observe (List/Watch) → Compare (desired vs actual) → Act (create/update/delete) → Update Status
```
- Must be **idempotent** — same input always produces same result
- Handle **transient errors** with exponential backoff (requeue)
- Use **owner references** + **finalizers** for lifecycle management

### CRD Structure
- `spec` — desired state (user-defined)
- `status` — observed state (controller-written)
- `metadata.finalizers` — prevents deletion until cleanup is done
- `metadata.ownerReferences` — garbage-collects child resources

### Operator SDK / controller-runtime
- `manager.Manager` — runs controllers, caches, webhooks
- `reconcile.Reconciler` — implement `Reconcile(ctx, req) (Result, error)`
- `client.Client` — CRUD for Kubernetes objects
- `Result{Requeue: true}` / `Result{RequeueAfter: duration}` — control requeue

---

## Key Commands / Code Snippets

```go
// Minimal Reconciler
func (r *MyReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := log.FromContext(ctx)

    // 1. Fetch the CR
    var obj myv1.MyResource
    if err := r.Get(ctx, req.NamespacedName, &obj); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 2. Handle deletion (finalizer)
    if !obj.DeletionTimestamp.IsZero() {
        return r.handleDelete(ctx, &obj)
    }

    // 3. Add finalizer
    if !controllerutil.ContainsFinalizer(&obj, myFinalizer) {
        controllerutil.AddFinalizer(&obj, myFinalizer)
        return ctrl.Result{}, r.Update(ctx, &obj)
    }

    // 4. Reconcile logic
    if err := r.ensureDeployment(ctx, &obj); err != nil {
        return ctrl.Result{}, fmt.Errorf("ensureDeployment: %w", err)
    }

    // 5. Update status
    obj.Status.Phase = "Ready"
    return ctrl.Result{}, r.Status().Update(ctx, &obj)
}
```

```bash
# Scaffold with operator-sdk
operator-sdk init --domain example.com --repo github.com/org/my-operator
operator-sdk create api --group app --version v1 --kind MyResource --resource --controller

# Install CRDs
make install

# Run locally (against cluster in current kubeconfig)
make run

# Build & deploy
make docker-build IMG=my-operator:latest
make deploy IMG=my-operator:latest
```

---

## Common Interview Questions

**Q: What is the difference between a Controller and an Operator?**
> All Operators use Controllers, but not all Controllers are Operators. An Operator specifically encodes operational knowledge about a stateful application (e.g., how to scale, backup, upgrade it). A plain controller just manages resource lifecycle.

**Q: What is idempotency and why must reconcilers be idempotent?**
> Idempotency means the same operation applied multiple times produces the same result. Reconcilers can be called repeatedly (retries, re-watches) so they must safely handle "already reconciled" state without side effects.

**Q: How do finalizers work?**
> A finalizer is a string in `metadata.finalizers`. Kubernetes refuses to delete a resource while it has finalizers. The controller performs cleanup, then removes the finalizer to allow deletion.

**Q: What causes infinite reconcile loops?**
> Updating the resource inside the reconciler triggers another watch event → another reconcile. Use `Status().Update()` (separate subresource) and check `ResourceVersion` or desired==actual before updating.

---

## Gotchas & Best Practices

- Always use `client.IgnoreNotFound` on `Get` — resource may have been deleted between watch event and reconcile
- Never update `spec` from a controller (user-owned); only update `status`
- Use server-side apply or `patch` instead of `update` to avoid conflicts
- Set `OwnerReferences` on child resources for automatic garbage collection
- Use `predicates` to filter watch events and reduce unnecessary reconciles
- Log with structured fields: `log.Info("reconciling", "name", req.Name, "namespace", req.Namespace)`

---

## Resources

- [ ] [Kubernetes Controller Runtime](https://github.com/kubernetes-sigs/controller-runtime)
- [ ] [Operator SDK Docs](https://sdk.operatorframework.io/docs/)
- [ ] [Kubebuilder Book](https://book.kubebuilder.io/)
- [ ] [Writing a Controller (k8s.io)](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)

---

## My Notes

<!-- ADD YOUR PERSONAL NOTES HERE -->

