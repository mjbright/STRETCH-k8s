# Kubernetes YAML Repair Lab

A collection of parallel, hands-on repair exercises for introductory Kubernetes training (LFS458/LFD459). Each task starts with a deliberately broken manifest. Students diagnose and correct it with `vi` or another editor, then validate and apply it in a safe training cluster.

The exercises are intentionally independent: assign different exercises to pairs or small groups, then have each group explain the error, the correction, and the Kubernetes concept involved. Exercises 1–8 use ordinary Kubernetes YAML. Exercises 9–11 use Kustomize/kyaml-style configuration (`kustomization.yaml`) alongside ordinary resource YAML.

## Suggested learner workflow

For every exercise:

1.  Copy the broken manifest into a new directory.
2.  Read the error before editing: `kubectl apply --dry-run=client -f <file>`.
3.  Repair it in an editor. In `vi`, turn on useful indentation aids with `:set number list expandtab shiftwidth=2`.
4.  Check the resource shape and API-level validity: `kubectl apply --dry-run=client -f <file>`.
5.  When connected to a disposable cluster, apply it and inspect the result with `kubectl get`, `kubectl describe`, and `kubectl logs`.
6.  Compare with the solution only after recording what was wrong and why.

> **Trainer note:** Use a dedicated namespace or an ephemeral training cluster. Namespaces, image names, and ports in these examples are deliberately simple; adapt them to local cluster policy where necessary.

## Exercise map

| Exercise | Focus | Typical first signal |
| --- | --- | --- |
| 1 | Basic YAML indentation and Pod schema | YAML parse error |
| 2 | Deployment selectors and labels | API validation error |
| 3 | Service selector and target port | Service has no endpoints / traffic fails |
| 4 | ConfigMap references and environment variables | Pod fails to start |
| 5 | Volume mounts and ConfigMap keys | Pod fails to start / file absent |
| 6 | Resource requests and limits | API validation error |
| 7 | Job restart policy and completion | API validation error |
| 8 | Ingress backend schema and Service port | API validation error or 404 |
| 9 | Kustomize resource paths | `kustomize build` path error |
| 10 | Kustomize patches and target identity | Patch appears not to work |
| 11 | Kustomize generators and name references | Missing ConfigMap or stale reference |

---

## Exercise 1 — Repair a Pod: indentation, lists, and required fields

### Goal

Create a running Pod named `web-check` in namespace `yaml-lab`. It should run `nginx:1.27-alpine`, expose container port `8080`, and receive an environment variable named `MESSAGE` with the value `hello from yaml`.

### Broken manifest: `01-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-check
  namespace: yaml-lab
spec:
  containers:
  - name: nginx
    image nginx:1.27-alpine
    ports:
      containerPort: 8080
    env:
    - name: MESSAGE
      value: hello from yaml
```

### Hints

-   A mapping key needs a colon: `key: value`.
-   `containers` is a list; each list item begins with `-`.
-   `ports` is also a list, even when the container has only one port.
-   Use two spaces consistently under each nesting level.

### Check

```bash
kubectl apply --dry-run=client -f 01-pod.yaml
kubectl get pod -n yaml-lab
```

<details>
<summary>Show solution and explanation</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-check
  namespace: yaml-lab
spec:
  containers:                    # A Pod contains a list of containers.
    - name: nginx                # The dash starts the first list item.
      image: nginx:1.27-alpine   # `image:` needs a colon.
      ports:                     # `ports` is a list of port objects.
        - containerPort: 8080
      env:                       # `env` is a list of environment-variable objects.
        - name: MESSAGE
          value: "hello from yaml"
```

**What was fixed**

-   Added the missing colon after `image`.
-   Indented the `- name: nginx` list item beneath `containers`.
-   Changed `ports` from a mapping to a list item: `- containerPort: 8080`.
-   Kept the `env` entries nested under the container rather than beside it.

**Concept to retain:** YAML whitespace expresses structure. Kubernetes first needs valid YAML; only then can it validate the Kubernetes resource schema.

</details>

## Exercise 2 — Repair a Deployment: selectors must match Pod-template labels

### Goal

Create a Deployment named `catalog` with two replicas. The Deployment must manage Pods labelled `app: catalog`.

### Broken manifest: `02-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: catalog
  namespace: yaml-lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: catalogue
  template:
    metadata:
      labels:
        app: catalog
    spec:
      containers:
        - name: catalog
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80
```

### Hints

-   There are two label locations to compare.
-   `spec.selector.matchLabels` tells a Deployment which Pods it owns.
-   `spec.template.metadata.labels` is copied onto newly created Pods.

### Check

```bash
kubectl apply --dry-run=client -f 02-deployment.yaml
kubectl get deployment,pods -n yaml-lab -l app=catalog
```

<details>
<summary>Show solution and explanation</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: catalog
  namespace: yaml-lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: catalog                # Must exactly match the Pod-template label.
  template:
    metadata:
      labels:
        app: catalog              # Pods created by this Deployment receive this label.
    spec:
      containers:
        - name: catalog
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80
```

**What was fixed**

-   Changed `catalogue` to `catalog` under `spec.selector.matchLabels`.

**Concept to retain:** A Deployment selector is an identity contract. It must match the labels on the Pod template. A small spelling difference is not cosmetic; it changes which Pods the controller can manage.

</details>

## Exercise 3 — Repair a Service: selectors and named target ports

### Goal

Expose the `catalog` Pods from Exercise 2 through a ClusterIP Service called `catalog`. The Service listens on port `80` and forwards traffic to the container port named `http`.

### Broken manifest: `03-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: catalog
  namespace: yaml-lab
spec:
  selector:
    app: catalogue
  ports:
    - name: http
      port: 80
      targetPort: web
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: catalog
  namespace: yaml-lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: catalog
  template:
    metadata:
      labels:
        app: catalog
    spec:
      containers:
        - name: catalog
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80
              name: http
```

### Hints

-   A Service selector finds Pods by labels.
-   `targetPort` may be a number or the name of a container port.
-   When a named `targetPort` is used, it must match exactly.

### Check

```bash
kubectl apply -f 03-service.yaml
kubectl get endpointslice -n yaml-lab -l kubernetes.io/service-name=catalog
kubectl describe service catalog -n yaml-lab
```

<details>
<summary>Show solution and explanation</summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: catalog
  namespace: yaml-lab
spec:
  selector:
    app: catalog                  # Must match the labels on the Pods.
  ports:
    - name: http
      port: 80                     # Port exposed by the Service.
      targetPort: http             # Refers to the named container port below.
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: catalog
  namespace: yaml-lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: catalog
  template:
    metadata:
      labels:
        app: catalog
    spec:
      containers:
        - name: catalog
          image: nginx:1.27-alpine
          ports:
            - name: http           # The Service's targetPort name resolves here.
              containerPort: 80
```

**What was fixed**

-   Corrected the Service selector from `app: catalogue` to `app: catalog`.
-   Corrected `targetPort: web` to `targetPort: http`.

**Concept to retain:** A Service is not connected to a Deployment by name. It discovers Pods by labels. It then forwards to the numbered or named container port specified by `targetPort`.

</details>

## Exercise 4 — Repair a ConfigMap environment-variable reference

### Goal

Run a Pod that reads `APP_MODE=training` from a ConfigMap named `lab-settings`.

### Broken manifest: `04-configmap-env.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: lab-settings
  namespace: yaml-lab
data:
  APP_MODE: training
---
apiVersion: v1
kind: Pod
metadata:
  name: env-reader
  namespace: yaml-lab
spec:
  containers:
    - name: reader
      image: busybox:1.36
      command: ["sh", "-c", "echo APP_MODE=$APP_MODE; sleep 3600"]
      env:
        - name: APP_MODE
          valueFrom:
            configMapKeyRef:
            name: lab-setting
            key: app_mode
```

### Hints

-   Look carefully at the indentation beneath `configMapKeyRef`.
-   ConfigMap names and data keys are exact strings.
-   The environment variable's `name` is independent from the ConfigMap data key.

### Check

```bash
kubectl apply -f 04-configmap-env.yaml
kubectl get pod env-reader -n yaml-lab
kubectl logs env-reader -n yaml-lab
```

<details>
<summary>Show solution and explanation</summary>

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: lab-settings
  namespace: yaml-lab
data:
  APP_MODE: training               # This is the ConfigMap data key.
---
apiVersion: v1
kind: Pod
metadata:
  name: env-reader
  namespace: yaml-lab
spec:
  containers:
    - name: reader
      image: busybox:1.36
      command: ["sh", "-c", "echo APP_MODE=$APP_MODE; sleep 3600"]
      env:
        - name: APP_MODE           # Name of the variable inside the container.
          valueFrom:
            configMapKeyRef:
              name: lab-settings   # Exact ConfigMap object name.
              key: APP_MODE         # Exact key under ConfigMap `data`.
```

**What was fixed**

-   Indented `name` and `key` underneath `configMapKeyRef`.
-   Corrected the referenced ConfigMap name from `lab-setting` to `lab-settings`.
-   Corrected the key from `app_mode` to `APP_MODE`.

**Concept to retain:** Kubernetes references are schema-shaped nested objects. Correct indentation determines which object owns `name` and `key`; correct spelling determines which real resource and data entry are read.

</details>

## Exercise 5 — Repair a ConfigMap volume and mount

### Goal

Mount a ConfigMap key as `/etc/lab/message.txt` inside a busybox Pod, then print the file.

### Broken manifest: `05-configmap-volume.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: message-config
  namespace: yaml-lab
data:
  message.txt: "Kubernetes YAML repaired successfully."
---
apiVersion: v1
kind: Pod
metadata:
  name: volume-reader
  namespace: yaml-lab
spec:
  containers:
    - name: reader
      image: busybox:1.36
      command: ["sh", "-c", "cat /etc/lab/message.txt; sleep 3600"]
      volumeMounts:
        - name: message-volume
          mountPath: /etc/lab
  volumes:
    - name: message-vol
      configMap:
        name: message-config
        items:
          - key: message
            path: message.txt
```

### Hints

-   The volume mount name and the volume name form a connection.
-   The ConfigMap key is not necessarily the same as the target filename.
-   `items` selectively maps ConfigMap keys to files.

### Check

```bash
kubectl apply -f 05-configmap-volume.yaml
kubectl logs volume-reader -n yaml-lab
```

<details>
<summary>Show solution and explanation</summary>

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: message-config
  namespace: yaml-lab
data:
  message.txt: "Kubernetes YAML repaired successfully."
---
apiVersion: v1
kind: Pod
metadata:
  name: volume-reader
  namespace: yaml-lab
spec:
  containers:
    - name: reader
      image: busybox:1.36
      command: ["sh", "-c", "cat /etc/lab/message.txt; sleep 3600"]
      volumeMounts:
        - name: message-volume      # Refers to the volume object below.
          mountPath: /etc/lab
  volumes:
    - name: message-volume           # Must match volumeMounts[].name.
      configMap:
        name: message-config
        items:
          - key: message.txt         # Existing key in ConfigMap `data`.
            path: message.txt         # Filename created under /etc/lab.
```

**What was fixed**

-   Changed the volume name from `message-vol` to `message-volume` so the mount can find it.
-   Changed the ConfigMap key from `message` to `message.txt`.

**Concept to retain:** A `volumeMount` attaches to a `volume` by name. A ConfigMap volume then maps ConfigMap data keys to files. There are three different names in play: volume name, ConfigMap name, and ConfigMap data key.

</details>

## Exercise 6 — Repair container resource requests and limits

### Goal

Create a Deployment whose container requests 100 millicores of CPU and 64 MiB memory, with limits of 250 millicores and 128 MiB.

### Broken manifest: `06-resources.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resource-demo
  namespace: yaml-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: resource-demo
  template:
    metadata:
      labels:
        app: resource-demo
    spec:
      containers:
        - name: app
          image: nginx:1.27-alpine
          resources:
            request:
              cpu: 100m
              memory: 64Mi
            limits:
              cpu: "250m"
              memory: "128Mi"
```

### Hints

-   Kubernetes resource settings use the plural field names `requests` and `limits`.
-   CPU values such as `100m` are quantities, not plain integers.
-   Quote values when doing so makes the YAML type unambiguous.

### Check

```bash
kubectl apply --dry-run=client -f 06-resources.yaml
kubectl describe pod -n yaml-lab -l app=resource-demo
```

<details>
<summary>Show solution and explanation</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resource-demo
  namespace: yaml-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: resource-demo
  template:
    metadata:
      labels:
        app: resource-demo
    spec:
      containers:
        - name: app
          image: nginx:1.27-alpine
          resources:
            requests:               # Plural schema field.
              cpu: "100m"           # 100 millicores = 0.1 CPU.
              memory: "64Mi"
            limits:
              cpu: "250m"
              memory: "128Mi"
```

**What was fixed**

-   Changed `request` to `requests`.

**Concept to retain:** Kubernetes schemas are precise. Valid-looking YAML with an incorrect field name is still invalid Kubernetes configuration. Requests influence scheduling; limits constrain the container at runtime.

</details>

## Exercise 7 — Repair a Job: restart policy and command layout

### Goal

Run a Job named `yaml-check` that prints a message once and completes successfully.

### Broken manifest: `07-job.yaml`

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: yaml-check
  namespace: yaml-lab
spec:
  completions: 1
  template:
    spec:
      restartPolicy: Always
      containers:
        - name: checker
          image: busybox:1.36
          command:
            - sh -c
            - echo "Job completed"
```

### Hints

-   A Job's Pod template permits only certain restart policies.
-   `command` is a list of command arguments, not a list of shell lines.
-   `sh -c` requires the shell command as a separate argument.

### Check

```bash
kubectl apply --dry-run=client -f 07-job.yaml
kubectl apply -f 07-job.yaml
kubectl get job,pod -n yaml-lab
kubectl logs job/yaml-check -n yaml-lab
```

<details>
<summary>Show solution and explanation</summary>

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: yaml-check
  namespace: yaml-lab
spec:
  completions: 1
  template:
    spec:
      restartPolicy: Never          # Jobs use Never or OnFailure, not Always.
      containers:
        - name: checker
          image: busybox:1.36
          command:
            - sh                   # First argument: executable.
            - -c                   # Second argument: shell option.
            - 'echo "Job completed"' # Third argument: shell command string.
```

**What was fixed**

-   Changed `restartPolicy: Always` to `restartPolicy: Never`.
-   Split `sh -c` into two arguments.
-   Kept the shell program as one final command string.

**Concept to retain:** Controllers impose their own constraints on Pod templates. A Job tracks completion, while a Pod with `restartPolicy: Always` is intended to keep running.

</details>

## Exercise 8 — Repair an Ingress: service backend schema and named port

### Goal

Route requests for `catalog.lab.example` to the `catalog` Service on its port named `http`.

### Broken manifest: `08-ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: catalog
  namespace: yaml-lab
spec:
  ingressClassName: nginx
  rules:
    - host: catalog.lab.example
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              serviceName: catalog
              servicePort: http
```

### Hints

-   The `networking.k8s.io/v1` Ingress API uses a nested `service` object.
-   A Service port can be selected by `name` or `number`.
-   This exercise assumes an Ingress controller and the `nginx` IngressClass are installed; client-side validation does not require them.

### Check

```bash
kubectl apply --dry-run=client -f 08-ingress.yaml
kubectl describe ingress catalog -n yaml-lab
```

<details>
<summary>Show solution and explanation</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: catalog
  namespace: yaml-lab
spec:
  ingressClassName: nginx
  rules:
    - host: catalog.lab.example
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:             # Ingress v1 nests the Service reference here.
                name: catalog       # Name of the Service resource.
                port:
                  name: http        # Name under Service `spec.ports[].name`.
```

**What was fixed**

-   Replaced legacy-style `serviceName` and `servicePort` fields with the current nested `service` structure.
-   Used `service.port.name: http` to match the Service port name.

**Concept to retain:** Kubernetes API versions can change a resource's schema. When an error involves an unknown field, verify that the manifest structure matches the API version declared at the top.

</details>

## Exercise 9 — Repair a Kustomization: resource paths and YAML list syntax

### Goal

Build an overlay that includes a base Deployment and Service, then adds the `training` namespace and a common label.

### Intended directory layout

```text
09-kustomize-paths/
├── base/
│   ├── deployment.yaml
│   └── service.yaml
└── overlays/
    └── training/
        └── kustomization.yaml
```

### Broken manifest: `overlays/training/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: training
commonLabels:
  audience: students
resources:
  - base/deployment.yaml
  - ../base/service.yaml
```

### Hints

-   Resource paths are relative to the directory containing `kustomization.yaml`.
-   A resource directory can be listed when that directory itself contains a `kustomization.yaml`.
-   `commonLabels` is valid in common Kustomize versions, but current projects may prefer `labels`; the repair below keeps the exercise focused on paths.

### Check

Run this from `09-kustomize-paths/overlays/training`:

```bash
kubectl kustomize .
```

<details>
<summary>Show solution and explanation</summary>

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: training
commonLabels:
  audience: students
resources:
  - ../../base/deployment.yaml  # Go up from overlays/training to the lab root.
  - ../../base/service.yaml     # Both resource paths use the same relative base.
```

**What was fixed**

-   Corrected both relative paths. From `overlays/training`, the `base` directory is two levels up: `../../base/`.

**Concept to retain:** Kustomize resolves files relative to the `kustomization.yaml` file, not the terminal's current directory. Draw the directory tree before editing paths.

</details>

## Exercise 10 — Repair a Kustomize patch: target identity and patch indentation

### Goal

Patch the `catalog` Deployment from a base directory so that its replica count becomes `3`.

### Base resource: `base/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: catalog
spec:
  replicas: 1
  selector:
    matchLabels:
      app: catalog
  template:
    metadata:
      labels:
        app: catalog
    spec:
      containers:
        - name: catalog
          image: nginx:1.27-alpine
```

### Broken Kustomization: `overlays/scale/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
patches:
  - path: scale.yaml
    target:
      kind: Deployment
      name: catalogue
```

### Broken patch: `overlays/scale/scale.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
name: catalog
spec:
replicas: 3
```

### Hints

-   The patch target is matched using the existing resource identity.
-   `metadata.name` must be indented below `metadata`.
-   `spec.replicas` must be indented below `spec`.

### Check

Run this from `overlays/scale`:

```bash
kubectl kustomize .
```

Look for `replicas: 3` in the rendered Deployment.

<details>
<summary>Show solution and explanation</summary>

`overlays/scale/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base                  # Directory reference; base contains its own kustomization.yaml.
patches:
  - path: scale.yaml
    target:
      kind: Deployment
      name: catalog             # Must match the existing Deployment name exactly.
```

`overlays/scale/scale.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: catalog                 # Nested under metadata.
spec:
  replicas: 3                   # Nested under spec.
```

**What was fixed**

-   Corrected the target name from `catalogue` to `catalog`.
-   Repaired the YAML indentation in the patch.

**Concept to retain:** Kustomize patches do not search for near matches. The target identifies the object exactly; the patch must also be valid YAML and use a Kubernetes-compatible shape.

</details>

## Exercise 11 — Repair Kustomize-generated ConfigMap references

### Goal

Use `configMapGenerator` to create a ConfigMap from a literal and inject its generated name into a Deployment through a name reference.

### Broken manifest: `kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
configMapGenerator:
  - name: app-settings
    literals:
      - LOG_LEVEL=info
```

### Broken manifest: `deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: settings-demo
spec:
  selector:
    matchLabels:
      app: settings-demo
  template:
    metadata:
      labels:
        app: settings-demo
    spec:
      containers:
        - name: app
          image: busybox:1.36
          command: ["sh", "-c", "echo $LOG_LEVEL; sleep 3600"]
          envFrom:
            - configMapRef:
                name: app-config
```

### Hints

-   Kustomize can add a hash suffix to a generated ConfigMap name.
-   Kustomize rewrites recognised name-reference fields when their starting name is correct.
-   Do not hard-code the hash suffix.

### Check

```bash
kubectl kustomize .
```

Look for the same generated ConfigMap name in both the `ConfigMap` object and the Deployment's `configMapRef`.

<details>
<summary>Show solution and explanation</summary>

`kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
configMapGenerator:
  - name: app-settings            # Stable base name; Kustomize may add a hash suffix.
    literals:
      - LOG_LEVEL=info
```

`deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: settings-demo
spec:
  selector:
    matchLabels:
      app: settings-demo
  template:
    metadata:
      labels:
        app: settings-demo
    spec:
      containers:
        - name: app
          image: busybox:1.36
          command: ["sh", "-c", "echo $LOG_LEVEL; sleep 3600"]
          envFrom:
            - configMapRef:
                name: app-settings # Kustomize rewrites this recognised reference.
```

**What was fixed**

-   Changed `configMapRef.name` from the nonexistent `app-config` to the generator's base name, `app-settings`.

**Concept to retain:** In source manifests, reference the generator's declared name. Kustomize handles the generated hash suffix and updates recognised references during the build. Do not copy a rendered suffix back into source files.

</details>

