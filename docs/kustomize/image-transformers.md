# Image Transformers

- **Official Documentation:** [Kustomize images Reference](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/images/)

The `images` transformer changes the image (and/or tag) used by containers, **without editing the Deployment**. Useful when the same app needs a different image per environment.

---

## Example Setup

Directory:

```text
web/
├── deployment.yaml
└── kustomization.yaml
```

```yaml
# web/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx
```

---

## Fields

| Field | What it does | Example |
| :--- | :--- | :--- |
| **`name`** | The image to find. Matches the image name, not the container name. | `nginx` |
| **`newName`** | Replaces the image name. | `haproxy` |
| **`newTag`** | Replaces the tag. | `2.4` |
| **`digest`** | Replaces the tag with a digest. | `sha256:...` |

---

## Example 1: Change the Image

```yaml
# web/kustomization.yaml
resources:
- deployment.yaml

images:
- name: nginx
  newName: haproxy
```

Result: `image: haproxy`

---

## Example 2: Change Only the Tag

```yaml
# web/kustomization.yaml
resources:
- deployment.yaml

images:
- name: nginx
  newTag: "1.25"
```

Result: `image: nginx:1.25`

---

## Example 3: Change Image and Tag

```yaml
# web/kustomization.yaml
resources:
- deployment.yaml

images:
- name: nginx
  newName: haproxy
  newTag: "2.4"
```

Result: `image: haproxy:2.4`

Run `kubectl kustomize web/` to see the result.

---

## Notes

- **`name` is the image, not the container.** In the Deployment, the container is called `web`. The `name: nginx` in `images` matches the image `nginx`, so ignore the container name.
- **Quote tags** such as `"2.4"` and `"1.25"`. Otherwise YAML may read them as numbers.
- **No match, no change.** If no container uses the image named in `name`, nothing happens and there is no error.

---

## CKA Quick Reference

- Use `images` to change image names or tags without touching the Deployment file.
- `name` = the original image. `newName` / `newTag` / `digest` = what to change to.
- Check your work with `kubectl kustomize <dir>`.
