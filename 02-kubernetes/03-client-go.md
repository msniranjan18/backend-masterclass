# 2.3 · client-go

## Overview

`client-go` is the official Go client library for the Kubernetes API. It provides typed clients, dynamic clients, informers, work queues, and utilities for building controllers and tools.

---

## Core Concepts

### Client Types
| Client | Use Case |
|--------|----------|
| `kubernetes.Clientset` | Typed CRUD for core and named API groups |
| `dynamic.Interface` | Untyped — works with any resource via `unstructured.Unstructured` |
| `controller-runtime client.Client` | Higher-level; used in Operators |
| `rest.RESTClient` | Low-level HTTP; base for all others |

### Authentication
- In-cluster: `rest.InClusterConfig()` — uses service account token + CA cert
- Out-of-cluster: `clientcmd.BuildConfigFromFlags("", kubeconfig)`

### Key Packages
- `k8s.io/client-go/kubernetes` — typed clientset
- `k8s.io/client-go/dynamic` — dynamic client
- `k8s.io/client-go/tools/cache` — informers, indexers, listers
- `k8s.io/client-go/util/workqueue` — rate-limiting queues
- `k8s.io/client-go/tools/record` — event recording
- `k8s.io/client-go/util/retry` — retry on conflict

---

## Key Commands / Code Snippets

```go
// Build config & clientset
config, err := rest.InClusterConfig()
if err != nil {
    config, err = clientcmd.BuildConfigFromFlags("", os.Getenv("KUBECONFIG"))
}
clientset, err := kubernetes.NewForConfig(config)

// List pods
pods, err := clientset.CoreV1().Pods("default").List(ctx, metav1.ListOptions{
    LabelSelector: "app=myapp",
})

// Watch deployments
watcher, err := clientset.AppsV1().Deployments("").Watch(ctx, metav1.ListOptions{})
for event := range watcher.ResultChan() {
    deploy := event.Object.(*appsv1.Deployment)
    fmt.Printf("%s: %s\n", event.Type, deploy.Name)
}

// Update with retry on conflict
retryErr := retry.RetryOnConflict(retry.DefaultRetry, func() error {
    deploy, err := clientset.AppsV1().Deployments(ns).Get(ctx, name, metav1.GetOptions{})
    if err != nil { return err }
    deploy.Spec.Replicas = pointer.Int32(3)
    _, err = clientset.AppsV1().Deployments(ns).Update(ctx, deploy, metav1.UpdateOptions{})
    return err
})

// Dynamic client for CRDs
dynClient, _ := dynamic.NewForConfig(config)
gvr := schema.GroupVersionResource{Group: "example.com", Version: "v1", Resource: "myresources"}
obj, err := dynClient.Resource(gvr).Namespace("default").Get(ctx, "my-obj", metav1.GetOptions{})
```

---

## Common Interview Questions

**Q: When would you use the dynamic client over the typed clientset?**
> Use the dynamic client when the resource type is not known at compile time (e.g., a generic tool that operates on any CRD), or when working with resources not in the main API groups.

**Q: How does `retry.RetryOnConflict` work?**
> On a 409 Conflict (optimistic concurrency via `resourceVersion`), it re-fetches the latest object and retries the update. This is the standard pattern for safe updates in controllers.

**Q: What is the role of `workqueue.RateLimitingQueue`?**
> It deduplicates reconcile requests (multiple events for the same key only trigger one reconcile) and applies rate limiting / exponential backoff on failures.

---

## Gotchas & Best Practices

- Never modify the object returned from a lister — it's shared; `DeepCopy()` first
- Use `Patch` (strategic merge or server-side apply) over `Update` to reduce conflicts
- Set `QPS` and `Burst` on `rest.Config` to avoid throttling in high-throughput controllers
- Record Kubernetes Events (`record.EventRecorder`) for user-visible status messages
- Use `metav1.ListOptions{ResourceVersion: "0"}` for cached list from etcd

---

## Resources

- [ ] [client-go GitHub](https://github.com/kubernetes/client-go)
- [ ] [client-go Examples](https://github.com/kubernetes/client-go/tree/master/examples)
- [ ] [Programming Kubernetes (book)](https://www.oreilly.com/library/view/programming-kubernetes/9781492047094/)

---

## My Notes

<!-- ADD YOUR PERSONAL NOTES HERE -->

