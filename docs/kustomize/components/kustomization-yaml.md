# Component `kustomization.yaml`

- **Official Documentation:** [Kustomize Components](https://kubectl.docs.kubernetes.io/guides/config_management/components/)

A component's `kustomization.yaml` declares the component, lists the resources it adds, and references patch files it applies to resources from the base or the component.

## Example

```yaml
# k8s/components/auth/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component
resources:
- auth-config.yaml
patches:
- path: api-auth-patch.yaml
```

Paths under `resources:` and `patches:` are relative to the component directory.

The component adds a ConfigMap:

```yaml
# k8s/components/auth/auth-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: auth-config
data:
  AUTH_ENABLED: "true"
```

The patch adds an environment variable to the `api` container in the base Deployment:

```yaml
# k8s/components/auth/api-auth-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  template:
    spec:
      containers:
      - name: api
        env:
        - name: AUTH_ENABLED
          valueFrom:
            configMapKeyRef:
              name: auth-config
              key: AUTH_ENABLED
```

An overlay includes this component under `components:`:

```yaml
# k8s/overlays/dev/kustomization.yaml
resources:
- ../../base
components:
- ../../components/auth
```

Build the overlay to render the base and the enabled component together:

```bash
kustomize build k8s/overlays/dev
```

## Quick Reference

- `kind: Component` identifies the component's `kustomization.yaml`.
- Use `resources:` for manifests the component adds.
- Use `patches:` with `path:` to reference patch files.
- The overlay enables a component with `components:`.
