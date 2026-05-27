# 2.2 · Kube Indexers & Informers

## Overview

Informers and Indexers are the foundational caching and eventing layer in client-go. They allow controllers to react to Kubernetes resource changes without polling the API server directly.

---

## Core Concepts

### Informer
- Combines a **ListWatch** (initial list + streaming watch) with an in-memory **store**
- Delivers events to registered **ResourceEventHandlers** (`OnAdd`, `OnUpdate`, `OnDelete`)
- Handles **reconnection** and **re-list** automatically
- Backed by a **DeltaFIFO queue** that buffers events

### SharedInformer / SharedIndexInformer
- Multiple controllers share one underlying ListWatch → reduces API server load
- `SharedInformerFactory` creates and manages all informers for a process

### Indexer
- Thread-safe in-memory cache (implements `Store`)
- Supports **indexes** — lookup objects by arbitrary fields (e.g., all pods by node name)
- `cache.NewIndexer(keyFunc, indexers)` — key function + index functions

### Work Queue
- `workqueue.NewRateLimitingQueue` — deduplicates and rate-limits reconcile requests
- Informer handler enqueues key → worker dequeues key → fetch from indexer → reconcile

---

## Key Commands / Code Snippets

```go
// SharedInformerFactory usage
factory := informers.NewSharedInformerFactory(client, 30*time.Second)
podInformer := factory.Core().V1().Pods()

podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc: func(obj interface{}) {
        pod := obj.(*v1.Pod)
        fmt.Printf("Pod added: %s/%s\n", pod.Namespace, pod.Name)
    },
    UpdateFunc: func(old, new interface{}) { /* handle update */ },
    DeleteFunc: func(obj interface{}) { /* handle delete */ },
})

ctx, cancel := context.WithCancel(context.Background())
defer cancel()
factory.Start(ctx.Done())
factory.WaitForCacheSync(ctx.Done())

// Custom Index — pods by node name
indexer := podInformer.Informer().GetIndexer()
indexer.AddIndexers(cache.Indexers{
    "byNode": func(obj interface{}) ([]string, error) {
        pod := obj.(*v1.Pod)
        return []string{pod.Spec.NodeName}, nil
    },
})

// Look up pods on a specific node
objs, _ := indexer.ByIndex("byNode", "node-1")
```

---

## Common Interview Questions

**Q: Why use an Informer instead of calling the API server directly?**
> Informers maintain a local cache from a single ListWatch connection. This avoids thundering-herd problems (all controllers polling) and reduces API server load significantly.

**Q: What is the DeltaFIFO queue?**
> A FIFO queue that stores ordered deltas (Added, Updated, Deleted, Replaced, Sync) for each object. It deduplicates: if an object is updated multiple times, only the latest state is kept, but all delta types are preserved.

**Q: How do you handle a "stale cache" problem with informers?**
> After enqueueing, always re-fetch the object from the lister/indexer (not from the event payload) to get the latest cached version. Never act on the object passed to the handler — it may be stale.

---

## Gotchas & Best Practices

- Always wait for `WaitForCacheSync` before starting reconcile loops
- Use `lister.Get()` inside the worker, not the object from the handler closure
- `DeleteFunc` receives a `cache.DeletedFinalStateUnknown` wrapper when the watch misses a delete — always type-assert safely
- Indexes add memory overhead — only create indexes you actually query
- Use `SharedInformerFactory` with a common resync period across controllers

---

## Resources

- [ ] [client-go Informer Example](https://github.com/kubernetes/client-go/tree/master/examples/workqueue)
- [ ] [How Informers Work (Deep Dive)](https://macias.info/entry/202109081800_k8s_informers.md)
- [ ] [controller-runtime Source](https://github.com/kubernetes-sigs/controller-runtime/tree/main/pkg/cache)

---

## My Notes

<!-- ADD YOUR PERSONAL NOTES HERE -->

