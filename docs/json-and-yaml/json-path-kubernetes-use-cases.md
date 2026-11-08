# JSONPath Use Cases in Kubernetes

`kubectl` can format resource data with JSONPath. Kubernetes expressions are enclosed in braces, and paths are relative to the object returned by the command.

## Get Pod Names

The result of `kubectl get pods -o json` contains an `items` array. Select each item's Pod name:

```bash
kubectl get pods -o=jsonpath='{.items[*].metadata.name}'
```

To print one Pod name per line, iterate over the items:

```bash
kubectl get pods -o=jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'
```

## Get Names and Node Assignments

```bash
kubectl get pods -o=jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\n"}{end}'
```

This prints each Pod's name followed by the node it is assigned to.

## Inspect One Resource

When querying a single resource, address its fields directly:

```bash
kubectl get pod nginx -o=jsonpath='{.spec.containers[*].image}'
```

Use `kubectl get <resource> -o json` to inspect the available fields when constructing a query. JSONPath output is useful for targeted inspection; use `kubectl describe` when you also need events and human-readable status details.
