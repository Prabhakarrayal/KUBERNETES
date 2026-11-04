# Kubernetes Production Deployment & Operations Runbook

## A first-principles, step-by-step guide from pre-deployment checks to production validation

> **Purpose:** This guide turns the 40 Kubernetes production-readiness
> practices in the supplied source into an operational learning path. It
> follows the order an engineer should normally think and act: prepare
> the cluster, configure the application, secure it, deploy it, expose
> it, observe it, protect its data, and test its resilience.
>
> **Important distinction:** The source contains 40 best practices, not
> 40 standalone `kubectl` commands. Some topics require YAML manifests,
> application code, CI/CD tools, Prometheus, Grafana, Velero, Trivy, k6,
> a CSI driver, or a cloud autoscaler. For those topics, this guide
> gives the relevant configuration and the `kubectl` commands used to
> apply or verify it. Terminal output is representative; names,
> versions, addresses, and counts will differ in a real cluster.

------------------------------------------------------------------------

## How to use this guide

Work through the phases in order. Do not treat a successful
`kubectl apply` as proof that an application is production-ready.
Applying a manifest only means the API server accepted the object. We
must also verify scheduling, container startup, readiness, service
routing, resource behavior, security, observability, storage, and
recovery.

### Example values used throughout

The examples use these values consistently:

  Item             Example
  ---------------- ------------------------------
  Namespace        `production`
  Application      `myapp`
  Container port   `3000`
  Service          `myapp-service`
  ConfigMap        `myapp-config`
  Secret           `myapp-secrets`
  ServiceAccount   `myapp-sa`
  Deployment       `myapp`
  Image            `myregistry.io/myapp:v1.2.3`
  Ingress host     `app.example.com`

Replace these values with the names used in your environment.

### Command-to-output reading convention

Whenever a section contains more than one command, the expected-output code block repeats each command as a `$ command` label immediately before its corresponding sample output. Read each pair together:

```text
$ kubectl get pods -n production
NAME       READY   STATUS
myapp-...  1/1     Running

$ kubectl get services -n production
NAME            TYPE        PORT(S)
myapp-service   ClusterIP   80/TCP
```

This makes it clear which terminal output belongs to which command. The output is illustrative, not a promise that every cluster will display identical names or values.

### Safety rules before running commands

-   Start with read-only inspection commands. Use mutating commands only
    when you understand their effect.
-   Confirm the current context and namespace before applying or
    deleting anything.
-   Do not print Secret values into terminal logs, screenshots, tickets,
    or chat.
-   Do not run `kubectl drain`, delete PVCs, change traffic selectors,
    or perform restore tests in production without an approved change
    plan.
-   A `Running` Pod is not necessarily ready to serve traffic. Check
    `READY`, probes, Service endpoints, and application behavior.
-   A successful manifest apply is not the same as a successful rollout.
-   Keep manifests in version control. Prefer reviewed declarative
    changes over one-off imperative edits.

------------------------------------------------------------------------

# Phase 1 --- Establish the environment and prepare the namespace

The first task is to make sure we are talking to the intended cluster
and that the target namespace is ready. This prevents a common and
serious mistake: deploying into the wrong cluster or namespace.

## 1. Confirm the active Kubernetes context

### 🎯 The Goal --- Concept

A kubeconfig can contain access details for multiple clusters and users.
The current context tells `kubectl` which cluster, identity, and default
namespace it will use.

Think of a context as the destination and identity written on an
envelope. Before sending a deployment, verify the address.

### 💻 The Command

```bash
kubectl config current-context
kubectl config get-contexts
kubectl cluster-info
```

### 🖥️ Expected Terminal Output

```text
$ kubectl config current-context
production-admin@prod-cluster

$ kubectl config get-contexts
CURRENT   NAME                           CLUSTER       AUTHINFO
*         production-admin@prod-cluster  prod-cluster  production-admin
          staging-admin@staging-cluster  staging       staging-admin

$ kubectl cluster-info
Kubernetes control plane is running at https://10.0.0.10:6443
CoreDNS is running at https://10.0.0.10:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

### 🔍 What to Verify

-   `current-context` must identify the intended cluster.
-   In `get-contexts`, the `*` must be beside the expected context.
-   `cluster-info` should return the control-plane endpoint without
    connection or authentication errors.
-   **Failure signals:** an unexpected context, `connection refused`,
    timeout, certificate errors, or `Unauthorized`.
-   Do not continue until the target cluster is unambiguous.

------------------------------------------------------------------------

## 2. Check cluster nodes and their readiness

### 🎯 The Goal --- Concept

Nodes are the machines that run our Pods. A cluster may have a healthy
API server but still be unable to run workloads because worker nodes are
unavailable, under pressure, or not ready.

Think of nodes as the workers in a warehouse. Before assigning new work,
we check that the workers are present and able to work.

### 💻 The Command

```bash
kubectl get nodes -o wide
kubectl describe nodes
```

### 🖥️ Expected Terminal Output

```text
$ kubectl get nodes -o wide
NAME             STATUS   ROLES           AGE   VERSION   INTERNAL-IP
control-plane-1  Ready    control-plane   30d   v1.30.4   10.0.0.11
worker-1         Ready    <none>          30d   v1.30.4   10.0.0.21
worker-2         Ready    <none>          30d   v1.30.4   10.0.0.22

$ kubectl describe nodes
Conditions:
  Type             Status
  MemoryPressure   False
  DiskPressure     False
  PIDPressure      False
  Ready            True
```

### 🔍 What to Verify

-   Every node expected to host application workloads should show
    `Ready`.
-   Check `MemoryPressure`, `DiskPressure`, and `PIDPressure` in
    `describe`.
-   Check node versions and architecture compatibility with your
    workload images.
-   **Failure signals:** `NotReady`, pressure conditions set to `True`,
    frequent node flapping, or insufficient worker capacity.
-   A control-plane node being `Ready` does not prove that worker
    capacity is sufficient.

------------------------------------------------------------------------

## 3. Check cluster-wide system components and API availability

### 🎯 The Goal --- Concept

Kubernetes depends on system components such as DNS and networking
components. If these are unhealthy, applications may fail even when
their own containers are correct.

The API server is the front desk of the cluster. DNS and networking are
essential internal services. We check the platform before blaming the
application.

### 💻 The Command

```bash
kubectl get --raw='/readyz?verbose'
kubectl get pods -n kube-system -o wide
kubectl get services -n kube-system
```

### 🖥️ Expected Terminal Output

```text
$ kubectl get --raw='/readyz?verbose'
[+]ping ok
[+]log ok
[+]etcd ok
[+]poststarthook/start-kube-apiserver-admission-initializer ok
readyz check passed

$ kubectl get pods -n kube-system -o wide
NAME                       READY   STATUS    RESTARTS
coredns-6f6b679f8f-abcde    1/1     Running   0
coredns-6f6b679f8f-fghij    1/1     Running   0
kube-proxy-x1y2z            1/1     Running   0

$ kubectl get services -n kube-system
NAME       TYPE        CLUSTER-IP    PORT(S)
kube-dns   ClusterIP   10.96.0.10    53/UDP,53/TCP
```

### 🔍 What to Verify

-   The API readiness endpoint should report `readyz check passed`.
-   CoreDNS Pods should be `Running` and fully ready.
-   Review CNI and other cluster networking components according to the
    distribution in use.
-   **Failure signals:** failed readiness checks, system Pods in
    `Pending`, `CrashLoopBackOff`, or repeated restarts.
-   Component names differ between managed Kubernetes distributions. Do
    not assume every cluster uses the same Pod names.

------------------------------------------------------------------------

## 4. Create and verify a dedicated application namespace

### 🎯 The Goal --- Concept

A namespace is a logical boundary for grouping Kubernetes resources. It
helps separate production, staging, monitoring, and other workloads.

Think of namespaces as separate rooms in one building. They help
organize resources and apply policies, quotas, and access rules. They
are not a complete security boundary by themselves.

### 💻 The Command

```bash
kubectl create namespace production
kubectl get namespaces
kubectl get namespace production
```

If the namespace already exists, do not recreate it blindly. Inspect it
and apply the required labels and policies.

### 🖥️ Expected Terminal Output

```text
$ kubectl create namespace production
namespace/production created

$ kubectl get namespaces
NAME              STATUS   AGE
default           Active   30d
kube-system       Active   30d
kube-public       Active   30d
kube-node-lease   Active   30d
production        Active   2s

$ kubectl get namespace production
NAME         STATUS   AGE
production   Active   2s
```

### 🔍 What to Verify

-   The namespace must show `Active`.
-   Confirm that the spelling and environment are correct.
-   Confirm that the deployment account is authorized to create
    resources there.
-   **Failure signals:** `Forbidden`, `AlreadyExists` when the workflow
    assumes a new namespace, or a namespace stuck in `Terminating`.
-   Avoid placing production workloads in `default`.

------------------------------------------------------------------------

## 5. Apply consistent labels to resources

### 🎯 The Goal --- Concept

Labels are key-value metadata used to identify and group Kubernetes
objects. Selectors use labels to connect resources, such as a Service
selecting the Pods that belong to an application.

Think of labels as standardized tags on equipment. If every team uses
different tags for the same application, it becomes difficult to find
and manage the right objects.

### 💻 The Command

Example labels in a workload manifest:

``` yaml
metadata:
  labels:
    app: myapp
    version: v1.2.3
    component: api
    environment: production
    team: backend
    managed-by: helm
```

Inspect labels:

```bash
kubectl get pods -n production --show-labels
kubectl get deployments -n production --show-labels
kubectl get pods -n production -l app=myapp
```

### 🖥️ Expected Terminal Output

```text
NAME                     READY   STATUS    LABELS
myapp-7d8f9c6b5d-abcde   1/1     Running   app=myapp,version=v1.2.3,component=api
myapp-7d8f9c6b5d-fghij   1/1     Running   app=myapp,version=v1.2.3,component=api
```

### 🔍 What to Verify

-   Pods belonging to the application should have the expected `app`
    label.
-   Deployment selectors and Pod-template labels must match.
-   Service selectors must match the intended Pod labels.
-   **Failure signals:** a selector returning no Pods, inconsistent
    application labels, or a Service selecting the wrong version.
-   Labels are not merely documentation; selectors can depend on them
    for actual traffic routing.

------------------------------------------------------------------------

## 6. Configure namespace resource defaults with a LimitRange

### 🎯 The Goal --- Concept

A LimitRange defines default and maximum resource values for objects in
a namespace. It is a safety net for containers whose manifests omit
resource settings.

Think of it as a set of house rules: every container receives sensible
default resource values, and no single container can request an
unreasonable amount.

### 💻 The Command

Create `limit-range.yaml`:

``` yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:
        memory: "256Mi"
        cpu: "500m"
      defaultRequest:
        memory: "128Mi"
        cpu: "100m"
      max:
        memory: "2Gi"
        cpu: "2"
```

Apply and inspect:

```bash
kubectl apply -f limit-range.yaml
kubectl describe limitrange default-limits -n production
```

### 🖥️ Expected Terminal Output

```text
limitrange/default-limits created

Name:       default-limits
Namespace:  production
Type        Resource  Min  Max   Default Request  Default Limit
Container   cpu       -    2     100m             500m
Container   memory    -    2Gi   128Mi            256Mi
```

### 🔍 What to Verify

-   The LimitRange exists in the intended namespace.
-   CPU and memory defaults match the team's policy.
-   Maximum values are realistic for the workload.
-   **Failure signals:** unexpected defaults, values that are too
    restrictive, or Pod admission errors caused by exceeding maximums.
-   Defaults do not replace explicit resource sizing. Set requests and
    limits in application manifests.

------------------------------------------------------------------------

## 7. Configure namespace-wide ResourceQuota

### 🎯 The Goal --- Concept

A ResourceQuota limits the total resource consumption or object count in
a namespace. It prevents one team or application from consuming all
cluster resources.

Think of it as a department budget. Each namespace receives a defined
allowance for CPU, memory, Pods, Services, and storage claims.

### 💻 The Command

Create `resource-quota.yaml`:

``` yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"
    pods: "100"
    services: "20"
    persistentvolumeclaims: "30"
```

Apply and inspect:

```bash
kubectl apply -f resource-quota.yaml
kubectl describe resourcequota production-quota -n production
```

### 🖥️ Expected Terminal Output

```text
Name:            production-quota
Namespace:       production
Resource         Used   Hard
--------         ----   ----
limits.cpu       4      40
limits.memory    8Gi    80Gi
persistentvolumeclaims  2  30
pods             6      100
requests.cpu     2      20
requests.memory  4Gi    40Gi
services         2      20
```

### 🔍 What to Verify

-   `Used` should be within `Hard` for every quota resource.
-   Leave enough headroom for rollout surges, autoscaling, and
    operational Pods.
-   **Failure signals:** a resource at its hard limit or Pod creation
    rejected with `exceeded quota`.
-   A quota is a namespace ceiling, not a guarantee that the cluster has
    that much capacity.

------------------------------------------------------------------------

# Phase 2 --- Configure application behavior and security

This phase defines how the application receives configuration, accesses
credentials, identifies itself to the API, and operates with least
privilege.

## 8. Create a ConfigMap for non-secret configuration

### 🎯 The Goal --- Concept

A ConfigMap stores non-sensitive configuration separately from the
container image. This lets us change settings without rebuilding the
application image.

Think of the image as the appliance and the ConfigMap as its settings
panel. The same image can run in staging and production with different
configuration.

### 💻 The Command

Create `myapp-config.yaml`:

``` yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
  namespace: production
data:
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"
  CACHE_TTL: "300"
  config.yaml: |
    server:
      port: 3000
      timeout: 30s
    cache:
      ttl: 5m
```

Apply and inspect:

```bash
kubectl apply -f myapp-config.yaml
kubectl get configmap myapp-config -n production
kubectl describe configmap myapp-config -n production
```

### 🖥️ Expected Terminal Output

```text
configmap/myapp-config created

NAME           DATA   AGE
myapp-config   4      5s

Name:         myapp-config
Namespace:    production
Data
====
CACHE_TTL:
----
300
LOG_LEVEL:
----
info
MAX_CONNECTIONS:
----
100
config.yaml:
----
server:
  port: 3000
  timeout: 30s
```

### 🔍 What to Verify

-   The ConfigMap exists in the same namespace as the consuming Pod.
-   Expected keys are present and correctly spelled.
-   Values match the application's expected configuration format.
-   **Failure signals:** missing keys, invalid configuration syntax, or
    an application failing to start because a required setting is
    absent.
-   ConfigMap values injected as environment variables are generally
    read when the container starts. Updating a ConfigMap does not
    automatically restart the application.

------------------------------------------------------------------------

## 9. Create and verify Kubernetes Secrets safely

### 🎯 The Goal --- Concept

A Secret stores sensitive values such as passwords, tokens, and
certificates. Kubernetes Secret data is Base64-encoded, which is
encoding---not encryption.

Think of a Secret as a locked container only if the cluster is
configured with appropriate encryption, access controls, and operational
safeguards. Base64 alone does not protect the value.

### 💻 The Command

Create a Secret interactively or from a protected file. Avoid putting
real credentials directly in shell history.

```bash
kubectl create secret generic myapp-secrets \
  --namespace production \
  --from-file=database-password=./database-password.txt
```

Inspect metadata without displaying the secret data:

```bash
kubectl get secret myapp-secrets -n production
kubectl describe secret myapp-secrets -n production
```

Example Pod volume configuration:

``` yaml
volumes:
  - name: secrets
    secret:
      secretName: myapp-secrets
      defaultMode: 0400

containers:
  - name: myapp
    volumeMounts:
      - name: secrets
        mountPath: /etc/secrets
        readOnly: true
```

### 🖥️ Expected Terminal Output

```text
secret/myapp-secrets created

NAME             TYPE     DATA   AGE
myapp-secrets    Opaque   1      5s

Name:         myapp-secrets
Namespace:    production
Type:         Opaque
Data
====
database-password:  24 bytes
```

### 🔍 What to Verify

-   The Secret exists in the correct namespace.
-   The `DATA` count matches the expected number of keys.
-   The Pod references the exact Secret name and mounts it at the
    intended path.
-   **Failure signals:** `secret not found`, mount errors, permission
    errors, or application authentication failures.
-   Never use `kubectl get secret -o yaml` in shared logs or screenshots
    unless the output is safely redacted.
-   Use encryption at rest for etcd, restrict RBAC access, and consider
    an external secrets manager for production credentials.
-   The source mentions checking API-server encryption configuration.
    The exact method depends on cluster distribution and control-plane
    access; `kubectl get apiserver` is not a portable standard
    Kubernetes check.

------------------------------------------------------------------------

## 10. Create a dedicated ServiceAccount for the application

### 🎯 The Goal --- Concept

A ServiceAccount gives a Pod an identity when it communicates with the
Kubernetes API. It should be separate from a human user's identity.

Think of it as an application ID badge. The badge should grant only the
permissions the application needs---not administrator access.

### 💻 The Command

Create `service-account.yaml`:

``` yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: production
```

Apply and inspect:

```bash
kubectl apply -f service-account.yaml
kubectl get serviceaccount myapp-sa -n production
kubectl describe serviceaccount myapp-sa -n production
```

Use it in the Pod template:

``` yaml
spec:
  serviceAccountName: myapp-sa
```

### 🖥️ Expected Terminal Output

```text
serviceaccount/myapp-sa created

NAME       SECRETS   AGE
myapp-sa   0         5s
```

### 🔍 What to Verify

-   The ServiceAccount exists in the application's namespace.
-   The Deployment specifies `serviceAccountName: myapp-sa`.
-   The application does not use the default ServiceAccount
    unnecessarily.
-   **Failure signals:** `service account not found`, or unexpected API
    access by a workload.
-   Modern Kubernetes commonly mounts short-lived projected tokens when
    token automounting is enabled; do not assume a long-lived Secret
    token will appear in the `SECRETS` column.

------------------------------------------------------------------------

## 11. Apply least-privilege RBAC permissions

### 🎯 The Goal --- Concept

RBAC decides who can perform which actions on which Kubernetes
resources. A Role defines namespace-scoped permissions; a RoleBinding
attaches those permissions to a user, group, or ServiceAccount.

Think of it as an access card. An application that only reads ConfigMaps
should not receive permission to delete Pods or modify cluster-wide
settings.

### 💻 The Command

Create `myapp-rbac.yaml`:

``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: myapp-role
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: myapp-rolebinding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: myapp-sa
    namespace: production
roleRef:
  kind: Role
  name: myapp-role
  apiGroup: rbac.authorization.k8s.io
```

Apply and test:

```bash
kubectl apply -f myapp-rbac.yaml
kubectl auth can-i get configmaps \
  --as=system:serviceaccount:production:myapp-sa \
  -n production

kubectl auth can-i delete pods \
  --as=system:serviceaccount:production:myapp-sa \
  -n production
```

### 🖥️ Expected Terminal Output

```text
role.rbac.authorization.k8s.io/myapp-role created
rolebinding.rbac.authorization.k8s.io/myapp-rolebinding created

yes
no
```

### 🔍 What to Verify

-   The ServiceAccount can perform required operations.
-   It cannot perform unnecessary operations.
-   The RoleBinding subject and namespace match the intended
    ServiceAccount.
-   **Failure signals:** `no` for a required permission, `yes` for an
    unnecessary powerful permission, or `Forbidden` in application logs.
-   `kubectl auth can-i --list` can help audit the permissions visible
    to an identity, but it does not replace reviewing RoleBindings and
    ClusterRoleBindings.

------------------------------------------------------------------------

## 12. Audit high-privilege RBAC bindings

### 🎯 The Goal --- Concept

A RoleBinding grants permissions inside one namespace. A
ClusterRoleBinding can grant permissions across the cluster. We should
regularly inspect cluster-wide bindings and identify who has
`cluster-admin`.

Think of this as reviewing master keys. A small number of trusted
administrators may need them, but ordinary applications should not.

### 💻 The Command

```bash
kubectl get clusterrolebindings -o wide
kubectl get clusterrolebindings \
  -o jsonpath='{range .items[?(@.roleRef.name=="cluster-admin")]}{.metadata.name}{"\t"}{.subjects}{"\n"}{end}'
```

Optional tool from the source:

```bash
kubectl krew install rbac-lookup
kubectl rbac-lookup myapp-sa -n production
```

### 🖥️ Expected Terminal Output

```text
$ kubectl get clusterrolebindings -o wide
NAME                     ROLE                        AGE
cluster-admin-binding    ClusterRole/cluster-admin   30d
system:node              ClusterRole/system:node      30d

$ kubectl get clusterrolebindings -o jsonpath='{range .items[?(@.roleRef.name=="cluster-admin")]}{.metadata.name}{"\t"}{.subjects}{"\n"}{end}'
cluster-admin-binding    [ServiceAccount:admin]

$ kubectl rbac-lookup myapp-sa -n production
SUBJECT             NAMESPACE    ROLE
myapp-sa            production   myapp-role
```

### 🔍 What to Verify

-   Every `cluster-admin` subject should be known, approved, and
    necessary.
-   Application ServiceAccounts should not receive cluster-admin access.
-   Investigate unexpected users, groups, and ServiceAccounts.
-   **Failure signals:** broad cluster-admin bindings for application
    identities, unknown subjects, or permissions that exceed the
    workload's needs.
-   Treat RBAC auditing as a recurring control, not a one-time
    deployment task.

------------------------------------------------------------------------

## 13. Enforce Pod Security Standards at the namespace level

### 🎯 The Goal --- Concept

Pod Security Standards define security levels for Pods. The `restricted`
profile applies strong security requirements, including restrictions on
privilege escalation, root execution, and Linux capabilities.

Think of it as a building safety code. A workload must satisfy the rules
before it is allowed into the production area.

### 💻 The Command

Label the namespace:

```bash
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted \
  --overwrite
```

Verify:

```bash
kubectl get namespace production --show-labels
```

### 🖥️ Expected Terminal Output

```text
namespace/production labeled

NAME         STATUS   AGE   LABELS
production   Active   30d   pod-security.kubernetes.io/enforce=restricted,pod-security.kubernetes.io/audit=restricted,pod-security.kubernetes.io/warn=restricted
```

### 🔍 What to Verify

-   All three labels are set to the intended profile.
-   Workload manifests satisfy the restricted requirements.
-   **Failure signals:** Pod admission rejections with messages about
    privileged containers, running as root, capabilities, or host
    namespaces.
-   Test enforcement in a non-production namespace first. Enabling
    `restricted` can block existing workloads that do not meet the
    policy.

------------------------------------------------------------------------

## 14. Configure container security contexts

### 🎯 The Goal --- Concept

A security context controls how a container or Pod runs. It can prevent
root execution, privilege escalation, writable root filesystems, and
unnecessary Linux capabilities.

Think of it as limiting what a process can do even if an attacker
manages to compromise the application.

### 💻 The Command

Add a security context to the Pod template:

``` yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1001
    runAsGroup: 1001
    fsGroup: 1001
  containers:
    - name: myapp
      image: myregistry.io/myapp:v1.2.3
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL
```

After applying the Deployment, inspect it:

```bash
kubectl apply -f deployment.yaml
kubectl get deployment myapp -n production
kubectl get pods -n production -l app=myapp -o yaml
```

### 🖥️ Expected Terminal Output

```text
deployment.apps/myapp configured

NAME    READY   UP-TO-DATE   AVAILABLE   AGE
myapp   3/3     3            3           2m
```

### 🔍 What to Verify

-   `runAsNonRoot: true` and the intended non-zero UID are configured.
-   Privilege escalation is disabled.
-   All unnecessary capabilities are dropped.
-   The root filesystem is read-only if the application supports it.
-   **Failure signals:** container startup failures caused by
    file-permission assumptions or attempts to write to the read-only
    root filesystem.
-   Some applications need writable mounted volumes for temporary files.
    Provide narrowly scoped writable mounts rather than disabling the
    control without analysis.

------------------------------------------------------------------------

## 15. Define NetworkPolicies for Pod traffic

### 🎯 The Goal --- Concept

A NetworkPolicy controls which network connections are allowed to or
from selected Pods. Without policies, many Kubernetes networking setups
allow broad Pod-to-Pod communication by default.

Think of it as a firewall between rooms. The application should accept
traffic from approved sources and reach only the dependencies it needs.

### 💻 The Command

Create `myapp-network-policy.yaml`:

``` yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: myapp-network-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: myapp
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: nginx-ingress
      ports:
        - protocol: TCP
          port: 3000
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: UDP
          port: 53
```

Apply and inspect:

```bash
kubectl apply -f myapp-network-policy.yaml
kubectl get networkpolicy -n production
kubectl describe networkpolicy myapp-network-policy -n production
```

### 🖥️ Expected Terminal Output

```text
networkpolicy.networking.k8s.io/myapp-network-policy created

NAME                  POD-SELECTOR   AGE
myapp-network-policy  app=myapp      5s
```

### 🔍 What to Verify

-   The policy selects the intended Pods.
-   Required ingress sources and ports are allowed.
-   Required egress destinations and ports are allowed.
-   DNS traffic is allowed to the actual cluster DNS service.
-   **Failure signals:** application timeouts, failed DNS lookups, or
    blocked dependency connections after policy enforcement.
-   NetworkPolicy enforcement depends on the cluster network plugin. A
    created policy does not prove that the CNI enforces it.
-   The example is illustrative. Production policies should restrict DNS
    destinations and namespaces more precisely where practical.

------------------------------------------------------------------------

# Phase 3 --- Validate images, resources, storage, and workload design

Before applying the main Deployment, validate the image, resource model,
health checks, storage requirements, and availability strategy. These
decisions strongly affect whether the workload can start and remain
healthy.

## 16. Scan the container image for vulnerabilities

### 🎯 The Goal --- Concept

A container image contains the application and its operating-system
packages and libraries. Vulnerable packages can create security risks
even when our Kubernetes YAML is correct.

Think of this as inspecting building materials before construction. We
should not deploy a known vulnerable image just because it starts
successfully.

### 💻 The Command

The source uses Trivy in GitHub Actions:

``` yaml
- name: Scan Image with Trivy
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: myregistry.io/myapp:${{ github.sha }}
    severity: CRITICAL,HIGH
    exit-code: 1
```

A local CLI example is:

```bash
trivy image --severity HIGH,CRITICAL --exit-code 1 \
  myregistry.io/myapp:v1.2.3
```

### 🖥️ Expected Terminal Output

```text
myregistry.io/myapp:v1.2.3 (debian 12)

Total: 0 (HIGH: 0, CRITICAL: 0)

No vulnerabilities found
```

### 🔍 What to Verify

-   The scan targets the exact image digest or immutable version
    intended for deployment.
-   Critical and high findings are reviewed according to the
    organization's policy.
-   The CI job fails when policy-blocking findings are detected.
-   **Failure signals:** critical vulnerabilities, unreviewed
    high-severity findings, or a scan that did not actually run.
-   A clean scan is not proof of total security. It is one control in
    the image supply chain.

------------------------------------------------------------------------

## 17. Pin the image to a version or immutable digest

### 🎯 The Goal --- Concept

A Deployment must run a predictable image. The `latest` tag can point to
different image contents over time, making rollbacks and incident
investigations harder.

Think of it as recording an exact software release number rather than
saying "install the newest one."

### 💻 The Command

Prefer an exact version:

``` yaml
image: myregistry.io/myapp:v1.2.3
```

For stronger immutability, use a digest:

``` yaml
image: myregistry.io/myapp@sha256:a3b4c5d6e7f8...
```

Inspect the image used by the Deployment:

```bash
kubectl get deployment myapp -n production \
  -o jsonpath='{.spec.template.spec.containers[*].image}{"\n"}'
```

### 🖥️ Expected Terminal Output

```text
myregistry.io/myapp:v1.2.3
```

### 🔍 What to Verify

-   The image reference identifies the intended release.
-   Avoid `:latest` for production.
-   Confirm that the registry and image path are correct.
-   **Failure signals:** mutable tags, unexpected image versions, or
    `ImagePullBackOff`.
-   A digest is immutable; a tag is only a name that can be moved to
    another image.

------------------------------------------------------------------------

## 18. Set imagePullPolicy intentionally

### 🎯 The Goal --- Concept

`imagePullPolicy` tells Kubernetes when to pull an image from the
registry.

-   `IfNotPresent`: use a local image if it is already available on the
    node.
-   `Always`: ask the registry to resolve and pull the image when the
    container starts.
-   `Never`: use only a locally available image.

Think of it as deciding whether the machine should check the store for
the image every time or use the copy it already has.

### 💻 The Command

For a pinned version, a common setting is:

``` yaml
containers:
  - name: myapp
    image: myregistry.io/myapp:v1.2.3
    imagePullPolicy: IfNotPresent
```

Inspect the applied value:

```bash
kubectl get deployment myapp -n production \
  -o jsonpath='{.spec.template.spec.containers[*].imagePullPolicy}{"\n"}'
```

### 🖥️ Expected Terminal Output

```text
IfNotPresent
```

### 🔍 What to Verify

-   The pull policy matches the image-tag strategy.
-   Registry authentication is configured where required.
-   **Failure signals:** `ErrImagePull`, `ImagePullBackOff`, registry
    authorization errors, or missing images when using `Never`.
-   Kubernetes sets a default pull policy when the field is omitted. Be
    explicit when reproducibility matters.

------------------------------------------------------------------------

## 19. Define CPU and memory requests and limits

### 🎯 The Goal --- Concept

Requests tell the scheduler how much CPU and memory a container needs
for placement. Limits restrict how much it may consume.

Think of requests as reserving a minimum amount of workspace and limits
as setting the maximum space the process may use.

-   CPU is compressible: exceeding a CPU limit causes throttling.
-   Memory is not compressible: exceeding a memory limit can result in
    an OOM kill.

### 💻 The Command

Add resource settings to every application container:

``` yaml
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "500m"
```

Apply and inspect:

```bash
kubectl apply -f deployment.yaml
kubectl describe pod -n production -l app=myapp
kubectl get pods -n production
```

### 🖥️ Expected Terminal Output

```text
Containers:
  myapp:
    Requests:
      cpu:      100m
      memory:   128Mi
    Limits:
      cpu:      500m
      memory:   256Mi

NAME                     READY   STATUS    RESTARTS
myapp-7d8f9c6b5d-abcde   1/1     Running   0
```

### 🔍 What to Verify

-   Every container has realistic CPU and memory requests.
-   Limits are based on measured workload behavior, not guesses alone.
-   Pods schedule successfully without `Insufficient cpu` or
    `Insufficient memory`.
-   **Failure signals:** `Pending` Pods, CPU throttling, `OOMKilled`, or
    repeated restarts.
-   The source suggests CPU limits around 2--4 times requests and memory
    limits relatively close to requests. Treat this as a starting
    heuristic, not a universal rule. Load-test and measure the
    application.

------------------------------------------------------------------------

## 20. Configure health probes: startup, liveness, and readiness

### 🎯 The Goal --- Concept

Kubernetes uses probes to understand the state of an application.

-   **Startup probe:** Has the application finished starting?
-   **Liveness probe:** Is the process still healthy, or is it stuck and
    should be restarted?
-   **Readiness probe:** Can the application receive traffic now?

Think of a restaurant: - Startup means the restaurant has finished
opening. - Liveness means it is still operating. - Readiness means it is
ready to accept customers.

A failed readiness probe removes a Pod from normal Service traffic. A
failed liveness probe can restart its container. A startup probe
protects slow-starting applications from premature liveness failures.

### 💻 The Command

Example Deployment probe configuration:

``` yaml
containers:
  - name: myapp
    image: myregistry.io/myapp:v1.2.3
    ports:
      - containerPort: 3000
    startupProbe:
      httpGet:
        path: /healthz
        port: 3000
      failureThreshold: 30
      periodSeconds: 10
    livenessProbe:
      httpGet:
        path: /healthz
        port: 3000
      initialDelaySeconds: 30
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /ready
        port: 3000
      initialDelaySeconds: 10
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 3
      successThreshold: 1
```

Apply and inspect:

```bash
kubectl apply -f deployment.yaml
kubectl describe pod -n production -l app=myapp
```

### 🖥️ Expected Terminal Output

```text
Containers:
  myapp:
    State:          Running
    Ready:          True
    Restart Count:  0
    Startup:        http-get http://:3000/healthz delay=0s timeout=1s period=10s
    Liveness:       http-get http://:3000/healthz delay=30s timeout=5s period=10s
    Readiness:      http-get http://:3000/ready delay=10s timeout=3s period=5s

Events:
  Normal  Started  kubelet  Started container myapp
```

### 🔍 What to Verify

-   Startup succeeds before normal liveness and readiness checking takes
    over.
-   Readiness is `True` before expecting traffic to reach the Pod.
-   Liveness checks whether the process is alive; avoid making it depend
    on every external service.
-   Readiness may check whether essential dependencies are available.
-   **Failure signals:** `Unhealthy` events, repeated restarts,
    readiness remaining false, or probe timeouts.
-   Probe endpoints must return success only when their intended health
    condition is met.

------------------------------------------------------------------------

## 21. Implement meaningful health endpoints in the application

### 🎯 The Goal --- Concept

A probe is only useful if the application endpoint reports the correct
state. The application should provide separate endpoints for process
health and traffic readiness.

Think of the difference between asking "Is the building standing?" and
"Is the service desk ready to serve customers?"

### 💻 The Command

The source gives this Node.js-style example:

``` javascript
app.get('/healthz', (req, res) => {
  // Liveness: confirm the process is alive.
  res.status(200).json({
    status: 'ok',
    uptime: process.uptime()
  });
});

app.get('/ready', async (req, res) => {
  try {
    await db.query('SELECT 1');
    await redis.ping();
    res.status(200).json({ status: 'ready' });
  } catch (err) {
    res.status(503).json({
      status: 'not ready',
      error: err.message
    });
  }
});
```

Test from inside the Pod:

```bash
kubectl exec -n production deploy/myapp -- \
  sh -c 'wget -qO- http://127.0.0.1:3000/healthz'

kubectl exec -n production deploy/myapp -- \
  sh -c 'wget -qO- http://127.0.0.1:3000/ready'
```

### 🖥️ Expected Terminal Output

``` json
{"status":"ok","uptime":842.4}
{"status":"ready"}
```

### 🔍 What to Verify

-   `/healthz` returns HTTP success when the process is functioning.
-   `/ready` returns success only when the application can serve
    traffic.
-   Dependency checks have sensible timeouts.
-   **Failure signals:** HTTP 500/503, connection refused, timeouts, or
    a readiness endpoint that reports ready while the application cannot
    serve requests.
-   Do not make liveness depend on a temporary database outage unless
    restarting the process is genuinely the correct recovery action.

------------------------------------------------------------------------

## 22. Configure persistent storage with a StorageClass

### 🎯 The Goal --- Concept

A StorageClass describes how Kubernetes should dynamically provision
storage. Its provisioner connects Kubernetes to a storage system such as
a cloud disk or CSI-backed storage platform.

Think of it as selecting the type of disk and the rules for what happens
to it when a claim is released.

### 💻 The Command

The source's example is AWS EBS-specific:

``` yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: production-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  encrypted: "true"
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

Apply and inspect:

```bash
kubectl apply -f storageclass.yaml
kubectl get storageclass
kubectl describe storageclass production-ssd
```

### 🖥️ Expected Terminal Output

```text
storageclass.storage.k8s.io/production-ssd created

NAME             PROVISIONER          RECLAIMPOLICY   VOLUMEBINDINGMODE
production-ssd   ebs.csi.aws.com      Retain          WaitForFirstConsumer
```

### 🔍 What to Verify

-   The provisioner matches the storage platform and CSI driver
    installed in the cluster.
-   The reclaim policy is appropriate for the data's importance.
-   `Retain` keeps the underlying volume after the claim is released; it
    does not automatically create a backup.
-   `WaitForFirstConsumer` delays binding until a Pod is scheduled,
    helping with topology-aware placement.
-   **Failure signals:** missing CSI driver, provisioning errors,
    unsupported parameters, or a PVC remaining `Pending`.
-   Do not copy the AWS provisioner into a non-AWS cluster. Use the
    provisioner supported by that environment.

------------------------------------------------------------------------

## 23. Create a PersistentVolumeClaim for application data

### 🎯 The Goal --- Concept

A PersistentVolumeClaim (PVC) is a request for storage. A Pod uses the
claim rather than needing to know the low-level details of the
underlying disk.

Think of a PVC as a request for a storage locker. The storage system
finds or creates a suitable locker and Kubernetes attaches it to the
workload.

### 💻 The Command

Create `postgres-pvc.yaml`:

``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: production-ssd
  resources:
    requests:
      storage: 100Gi
```

Apply and inspect:

```bash
kubectl apply -f postgres-pvc.yaml
kubectl get pvc -n production
kubectl describe pvc postgres-data -n production
kubectl get pv
```

### 🖥️ Expected Terminal Output

```text
persistentvolumeclaim/postgres-data created

NAME            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS
postgres-data   Bound    pvc-12345678-abcd-1234-abcd-123456789012   100Gi      RWO            production-ssd
```

### 🔍 What to Verify

-   PVC status should be `Bound` when provisioning and binding are
    complete.
-   Capacity, access mode, and StorageClass should match the workload
    requirement.
-   The Pod should mount the intended claim at the correct path.
-   **Failure signals:** `Pending`, `ProvisioningFailed`, insufficient
    storage, or mount/attach errors.
-   `ReadWriteOnce` is an access mode with specific volume attachment
    semantics; it does not simply mean that only one process can ever
    read the data.

------------------------------------------------------------------------

## 24. Configure replicas and Pod anti-affinity

### 🎯 The Goal --- Concept

Multiple replicas keep an application available when one Pod fails.
Anti-affinity influences where those replicas are placed.

Think of keeping spare copies of an important document in different
buildings. If all copies are in one building and that building fails,
every copy is lost at once.

### 💻 The Command

Add replica and placement settings to the Deployment:

``` yaml
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: myapp
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: myapp
                topologyKey: kubernetes.io/hostname
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: myapp
              topologyKey: topology.kubernetes.io/zone
```

Apply and inspect placement:

```bash
kubectl apply -f deployment.yaml
kubectl get pods -n production -l app=myapp -o wide
```

### 🖥️ Expected Terminal Output

```text
NAME                     READY   STATUS    NODE
myapp-7d8f9c6b5d-abcde   1/1     Running   worker-1
myapp-7d8f9c6b5d-fghij   1/1     Running   worker-2
myapp-7d8f9c6b5d-klmno   1/1     Running   worker-3
```

### 🔍 What to Verify

-   The desired replica count is realistic for the application's
    availability target.
-   Pods are spread across nodes and, where supported, availability
    zones.
-   Nodes have the required topology labels.
-   **Failure signals:** Pods stuck `Pending` because required
    anti-affinity cannot be satisfied, or all replicas concentrated on
    one failure domain.
-   The example makes zone separation required and hostname separation
    preferred. If there are not enough eligible zones, scheduling may
    fail.

------------------------------------------------------------------------

## 25. Configure graceful termination

### 🎯 The Goal --- Concept

When Kubernetes stops a Pod during an update, scale-down, or node
operation, the application needs time to finish active requests and
close connections.

Think of a shop that stops accepting new customers but gives existing
customers time to finish before closing.

### 💻 The Command

Add a termination grace period:

``` yaml
spec:
  terminationGracePeriodSeconds: 60
  containers:
    - name: myapp
      image: myregistry.io/myapp:v1.2.3
```

The source gives this Node.js shutdown pattern:

``` javascript
process.on('SIGTERM', async () => {
  console.log('SIGTERM received. Starting graceful shutdown...');

  server.close(async () => {
    await db.end();
    await redis.quit();
    console.log('Graceful shutdown complete');
    process.exit(0);
  });

  setTimeout(() => process.exit(1), 50000);
});
```

Inspect the applied setting:

```bash
kubectl get deployment myapp -n production \
  -o jsonpath='{.spec.template.spec.terminationGracePeriodSeconds}{"\n"}'
```

### 🖥️ Expected Terminal Output

```text
60
```

### 🔍 What to Verify

-   The application handles `SIGTERM`.
-   New requests stop being accepted while in-flight work is completed.
-   Database and cache connections are closed cleanly.
-   The shutdown finishes before the grace period expires.
-   **Failure signals:** forced termination, interrupted requests,
    incomplete transactions, or shutdown timeouts.
-   The application's shutdown timeout must be shorter than the Pod
    termination grace period.

------------------------------------------------------------------------

# Phase 4 --- Deploy and track the application

Now the application configuration and workload design are ready. Apply
the Deployment, then verify the rollout rather than assuming that the
API server accepting the manifest means the application is live.

## 26. Configure a safe rolling update strategy

### 🎯 The Goal --- Concept

A rolling update replaces old Pods with new Pods gradually. `maxSurge`
controls how many extra Pods can be created above the desired replica
count. `maxUnavailable` controls how many replicas may be unavailable
during the update.

Think of replacing a team member one at a time while keeping enough
people available to continue the work.

### 💻 The Command

Add this to the Deployment:

``` yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

Apply and inspect:

```bash
kubectl apply -f deployment.yaml
kubectl describe deployment myapp -n production
```

### 🖥️ Expected Terminal Output

```text
StrategyType:           RollingUpdate
RollingUpdateStrategy:  0 max unavailable, 1 max surge
```

### 🔍 What to Verify

-   The strategy is `RollingUpdate`.
-   `maxUnavailable: 0` protects availability during the rollout,
    assuming capacity and readiness permit progress.
-   `maxSurge: 1` allows one additional Pod above the desired count.
-   **Failure signals:** rollout stalled because new Pods cannot
    schedule, pass probes, or pull the image.
-   A zero-unavailable rollout needs enough cluster capacity to create
    surge Pods.

------------------------------------------------------------------------

## 27. Apply the Deployment manifest

### 🎯 The Goal --- Concept

A Deployment describes the desired state of an application: which image
to run, how many replicas to maintain, and how Pods should be
configured.

Think of it as giving Kubernetes a written target. Kubernetes
continuously works to make the actual state match that target.

### 💻 The Command

```bash
kubectl apply -f deployment.yaml
kubectl get deployments -n production
kubectl get replicasets -n production
kubectl get pods -n production -l app=myapp
```

### 🖥️ Expected Terminal Output

```text
$ kubectl apply -f deployment.yaml
deployment.apps/myapp created

$ kubectl get deployments -n production
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
myapp   3/3     3            3           45s

$ kubectl get replicasets -n production
NAME               DESIRED   CURRENT   READY   AGE
myapp-7d8f9c6b5d   3         3         3       45s

$ kubectl get pods -n production -l app=myapp
NAME                     READY   STATUS    RESTARTS
myapp-7d8f9c6b5d-abcde   1/1     Running   0
myapp-7d8f9c6b5d-fghij   1/1     Running   0
myapp-7d8f9c6b5d-klmno   1/1     Running   0
```

### 🔍 What to Verify

-   The Deployment exists in `production`.
-   `READY`, `UP-TO-DATE`, and `AVAILABLE` reach the desired replica
    count.
-   The ReplicaSet has the expected number of Pods.
-   **Failure signals:** `0/3`, unavailable replicas, old and new
    ReplicaSets behaving unexpectedly, or Pods failing to schedule.
-   `created` or `configured` from `apply` only confirms the API object
    was accepted. Continue with rollout checks.

------------------------------------------------------------------------

## 28. Track rollout status and history

### 🎯 The Goal --- Concept

A rollout is the process of moving a Deployment from one version to
another. Rollout status tells us whether the new version has become
available. Rollout history helps us understand previous revisions.

Think of watching a staged replacement process rather than assuming the
new version is ready because the instruction was submitted.

### 💻 The Command

```bash
kubectl rollout status deployment/myapp -n production
kubectl rollout history deployment/myapp -n production
kubectl rollout history deployment/myapp -n production --revision=2
```

### 🖥️ Expected Terminal Output

```text
$ kubectl rollout status deployment/myapp -n production
deployment "myapp" successfully rolled out

$ kubectl rollout history deployment/myapp -n production
deployment.apps/myapp
REVISION  CHANGE-CAUSE
1         Initial deployment
2         Update image to v1.2.3

$ kubectl rollout history deployment/myapp -n production --revision=2
deployment.apps/myapp with revision #2
Pod Template:
  Containers:
    myapp:
      Image: myregistry.io/myapp:v1.2.3
```

### 🔍 What to Verify

-   Rollout status reports successful completion.
-   The current revision corresponds to the intended release.
-   The image and configuration in the current revision are correct.
-   **Failure signals:** `ProgressDeadlineExceeded`, rollout timeout,
    unavailable replicas, or unexpected revision history.
-   `CHANGE-CAUSE` is only useful when the team records meaningful
    change information; do not assume it is automatically populated.

------------------------------------------------------------------------

## 29. Verify replica counts and Pod placement

### 🎯 The Goal --- Concept

The Deployment's desired replica count is the target. Available replicas
are the Pods that have become available according to Kubernetes.

Think of ordering three workers and checking that all three have
arrived, completed onboarding, and are ready to work.

### 💻 The Command

```bash
kubectl get deployment myapp -n production
kubectl get rs -n production
kubectl get pods -n production -l app=myapp -o wide
```

### 🖥️ Expected Terminal Output

```text
$ kubectl get deployment myapp -n production
NAME    READY   UP-TO-DATE   AVAILABLE
myapp   3/3     3            3

$ kubectl get rs -n production
NAME               DESIRED   CURRENT   READY
myapp-7d8f9c6b5d   3         3         3

$ kubectl get pods -n production -l app=myapp -o wide
NAME                     READY   STATUS    RESTARTS   NODE
myapp-7d8f9c6b5d-abcde   1/1     Running   0          worker-1
myapp-7d8f9c6b5d-fghij   1/1     Running   0          worker-2
myapp-7d8f9c6b5d-klmno   1/1     Running   0          worker-3
```

### 🔍 What to Verify

-   Desired, current, updated, and available counts converge.
-   Each Pod has all expected containers ready, such as `1/1`.
-   Restarts are zero or understood.
-   Placement is consistent with anti-affinity and topology
    requirements.
-   **Failure signals:** missing replicas, `Pending`,
    `CrashLoopBackOff`, `ImagePullBackOff`, or repeated restarts.

------------------------------------------------------------------------

## 30. Configure Horizontal Pod Autoscaling

### 🎯 The Goal --- Concept

A HorizontalPodAutoscaler (HPA) changes the number of Pod replicas based
on observed metrics such as CPU and memory utilization.

Think of a support team that adds staff when the queue becomes busy and
reduces staff when demand falls.

### 💻 The Command

Example HPA:

``` yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 25
          periodSeconds: 60
```

Apply and inspect:

```bash
kubectl apply -f hpa.yaml
kubectl get hpa -n production
kubectl describe hpa myapp-hpa -n production
```

### 🖥️ Expected Terminal Output

```text
NAME        REFERENCE          TARGETS           MINPODS   MAXPODS   REPLICAS
myapp-hpa   Deployment/myapp   35%/70%, 48%/80%  3         20        3
```

### 🔍 What to Verify

-   The HPA targets the correct Deployment.
-   Current metrics are visible and targets are sensible.
-   Replica count stays between `minReplicas` and `maxReplicas`.
-   **Failure signals:** `<unknown>` metrics, `FailedGetResourceMetric`,
    or scaling that cannot happen because of capacity limits.
-   CPU and memory utilization percentages are calculated relative to
    resource requests. Set requests before relying on utilization-based
    HPA metrics.
-   Metrics Server or another supported metrics pipeline must be
    available.
-   HPA changes replica count; it does not create additional node
    capacity by itself.

------------------------------------------------------------------------

## 31. Configure a PodDisruptionBudget

### 🎯 The Goal --- Concept

A PodDisruptionBudget (PDB) limits how many replicas may be voluntarily
disrupted at once. It helps protect availability during planned
operations such as node drains.

Think of a minimum staffing rule: even during maintenance, a certain
number of workers must remain available.

### 💻 The Command

Create `myapp-pdb.yaml`:

``` yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
  namespace: production
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: myapp
```

Apply and inspect:

```bash
kubectl apply -f myapp-pdb.yaml
kubectl get pdb -n production
kubectl describe pdb myapp-pdb -n production
```

### 🖥️ Expected Terminal Output

```text
NAME        MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS
myapp-pdb   2               N/A               1
```

### 🔍 What to Verify

-   The PDB selector matches the intended application Pods.
-   `minAvailable` is consistent with the replica count and availability
    objective.
-   `ALLOWED DISRUPTIONS` reflects the number of voluntary disruptions
    currently permitted.
-   **Failure signals:** zero allowed disruptions when maintenance is
    expected, or a PDB selector that matches no Pods.
-   A PDB does not prevent involuntary failures such as a sudden machine
    crash. It governs supported voluntary disruptions.

------------------------------------------------------------------------

## 32. Configure the Service and verify Pod endpoints

### 🎯 The Goal --- Concept

A Service gives a stable network identity to a changing set of Pods. It
selects Pods by labels and forwards traffic to their target port.

Think of a company phone number. Employees may change, but callers use
the same number.

### 💻 The Command

Create `myapp-service.yaml`:

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
  namespace: production
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 3000
  type: ClusterIP
```

Apply and inspect:

```bash
kubectl apply -f myapp-service.yaml
kubectl get service myapp-service -n production
kubectl get endpointslice -n production \
  -l kubernetes.io/service-name=myapp-service
```

### 🖥️ Expected Terminal Output

```text
service/myapp-service created

NAME            TYPE        CLUSTER-IP     PORT(S)
myapp-service   ClusterIP   10.96.120.15   80/TCP

NAME                         ADDRESSTYPE   PORTS   ENDPOINTS
myapp-service-abcde          IPv4          3000    10.244.1.10,10.244.2.11,10.244.3.12
```

### 🔍 What to Verify

-   Service selector matches the intended Pod labels.
-   `port` is the port clients use; `targetPort` is the application
    port.
-   EndpointSlices contain ready Pod addresses.
-   **Failure signals:** no endpoints, wrong target port, or connection
    refused.
-   The source's `trafficPolicy` snippet is not valid as written for a
    standard Service manifest. `internalTrafficPolicy: Local` is a valid
    Service field, but it changes endpoint selection behavior and should
    be used only when local endpoints are intentionally required.

------------------------------------------------------------------------

## 33. Configure Ingress and TLS

### 🎯 The Goal --- Concept

Ingress provides HTTP and HTTPS routing from outside the cluster to
Services. A TLS configuration allows clients to use encrypted HTTPS
connections.

Think of Ingress as the reception desk for a building. It reads the
requested hostname and path, then directs the visitor to the correct
internal service.

### 💻 The Command

Example `myapp-ingress.yaml` based on the source:

``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - app.example.com
      secretName: myapp-tls-cert
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-service
                port:
                  number: 80
```

Apply and inspect:

```bash
kubectl apply -f myapp-ingress.yaml
kubectl get ingress -n production
kubectl describe ingress myapp-ingress -n production
kubectl get secret myapp-tls-cert -n production
```

Test from an appropriate client:

```bash
curl -I https://app.example.com
```

### 🖥️ Expected Terminal Output

```text
NAME            CLASS   HOSTS             ADDRESS        PORTS
myapp-ingress   nginx   app.example.com   203.0.113.10   80, 443

HTTP/2 200
server: nginx
content-type: application/json
```

### 🔍 What to Verify

-   The correct IngressClass and hostname are configured.
-   The backend Service name and port are correct.
-   The TLS Secret exists and contains a valid certificate and key.
-   DNS points to the Ingress controller's reachable address.
-   **Failure signals:** `404`, `502`, `503`, certificate errors,
    missing Ingress address, or certificate issuance failures.
-   An Ingress object alone does not create an Ingress controller. The
    controller and its dependencies must already be installed.
-   Annotations are controller-specific. Verify that the selected
    controller supports them.

------------------------------------------------------------------------

## 34. Choose a deployment traffic strategy: Canary or Blue-Green

### 🎯 The Goal --- Concept

Blue-Green deployment keeps the current version and the new version
available separately. Traffic is switched to the new version after
validation.

Canary deployment sends a small portion of traffic to the new version
before increasing its share.

Think of opening a new service counter to a small group first. If it
works, more customers are directed to it.

### 💻 The Command

The source illustrates Blue-Green with two Deployments and a Service
selector.

Blue Deployment:

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
        - name: myapp
          image: myregistry.io/myapp:v1.2.2
```

Green Deployment:

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
        - name: myapp
          image: myregistry.io/myapp:v1.2.3
```

Switch the Service selector only after validating Green:

``` yaml
spec:
  selector:
    app: myapp
    version: green
```

Apply and verify:

```bash
kubectl apply -f blue-green.yaml
kubectl get deployments -n production
kubectl get pods -n production -l app=myapp --show-labels
kubectl get endpointslice -n production \
  -l kubernetes.io/service-name=myapp-service
```

### 🖥️ Expected Terminal Output

```text
NAME         READY   UP-TO-DATE   AVAILABLE
myapp-blue   3/3     3            3
myapp-green  3/3     3            3

Service endpoints after switch:
10.244.2.20
10.244.3.21
10.244.4.22
```

### 🔍 What to Verify

-   Both versions are healthy before switching traffic.
-   The Service selector matches only the intended active version.
-   EndpointSlices contain Green Pod addresses after the switch.
-   Application checks and metrics remain healthy after the switch.
-   **Failure signals:** no endpoints after selector changes, errors
    from the new version, or incompatible database changes.
-   Blue-Green requires capacity to run both environments at once. A
    Service selector flip is a simple traffic switch, not a
    full-featured weighted canary system.

------------------------------------------------------------------------

# Phase 5 --- Runtime diagnostics, observability, and resource optimization

A successful rollout is only the beginning. We must confirm that the
application works, inspect logs and events, monitor resource use, and
ensure that alerts can detect problems.

## 35. Check Pod and container logs

### 🎯 The Goal --- Concept

Logs show what the application reports while it runs. They are one of
the first places to investigate startup errors, failed requests,
authentication problems, and dependency failures.

Think of logs as the application's activity diary. They provide evidence
of what happened and when.

### 💻 The Command

```bash
kubectl logs -n production -l app=myapp --tail=100
kubectl logs -n production <pod-name> -c myapp --tail=100
kubectl logs -n production <pod-name> -c myapp --previous
```

Use `--previous` when a container has restarted and you need the logs
from its previous instance.

### 🖥️ Expected Terminal Output

```text
$ kubectl logs -n production -l app=myapp --tail=100
2026-11-02T10:15:02Z level=info message="Application started" port=3000
2026-11-02T10:15:04Z level=info message="Database connection established"

$ kubectl logs -n production <pod-name> -c myapp --tail=100
2026-11-02T10:15:10Z level=info message="Request processed" status=200 duration_ms=24

$ kubectl logs -n production <pod-name> -c myapp --previous
2026-11-02T10:14:58Z level=error message="Previous container stopped"
```

### 🔍 What to Verify

-   The application starts successfully.
-   Dependencies connect successfully.
-   No repeating fatal errors, stack traces, or authentication failures
    appear.
-   **Failure signals:** repeated connection errors, panic/exception
    loops, permission errors, or repeated shutdown messages.
-   Kubernetes captures container stdout and stderr. Prefer structured
    logs to stdout/stderr rather than writing only to local files.
-   Do not log passwords, tokens, private keys, or sensitive personal
    data.

------------------------------------------------------------------------

## 36. Inspect Pod details, events, and runtime state

### 🎯 The Goal --- Concept

`kubectl describe` provides a detailed view of a resource, including
conditions, configuration, and recent events. Events often explain why a
Pod cannot start or become ready.

Think of `get` as a quick status board and `describe` as opening the
maintenance record for one machine.

### 💻 The Command

```bash
kubectl get pods -n production
kubectl describe pod <pod-name> -n production
kubectl get events -n production --sort-by=.metadata.creationTimestamp
```

### 🖥️ Expected Terminal Output

```text
$ kubectl get pods -n production
NAME                     READY   STATUS    RESTARTS
myapp-7d8f9c6b5d-abcde   1/1     Running   0

$ kubectl describe pod myapp-7d8f9c6b5d-abcde -n production
Name:             myapp-7d8f9c6b5d-abcde
Namespace:        production
Node:             worker-1/10.0.0.21
Status:           Running
IP:               10.244.1.10
Containers:
  myapp:
    State:        Running
    Ready:        True
    Restart Count: 0

$ kubectl get events -n production --sort-by=.metadata.creationTimestamp
LAST SEEN   TYPE     REASON      MESSAGE
2m          Normal   Scheduled   Successfully assigned Pod to worker-1
2m          Normal   Pulled      Container image already present
2m          Normal   Started     Started container myapp
```

### 🔍 What to Verify

-   Pod phase is `Running`, and container readiness is `True`.
-   The Pod is assigned to an appropriate node.
-   Events show successful scheduling, image pull, and container start.
-   **Failure signals:** `FailedScheduling`, `FailedMount`, `BackOff`,
    `Unhealthy`, `FailedCreatePodSandBox`, or repeated warnings.
-   Events are retained for a limited period. Use centralized monitoring
    and logging for longer-term incident analysis.

------------------------------------------------------------------------

## 37. Execute a command inside a running container

### 🎯 The Goal --- Concept

`kubectl exec` opens a process inside a running container. It is useful
for targeted checks such as testing a local HTTP endpoint, inspecting a
mounted file, or checking name resolution.

Think of it as entering the application's running environment to inspect
what the process can see.

### 💻 The Command

```bash
kubectl exec -it -n production <pod-name> -c myapp -- sh
```

For a one-time command:

```bash
kubectl exec -n production <pod-name> -c myapp -- \
  sh -c 'printenv | grep LOG_LEVEL'
```

### 🖥️ Expected Terminal Output

```text
/ # printenv | grep LOG_LEVEL
LOG_LEVEL=info
```

### 🔍 What to Verify

-   The command runs in the intended Pod and container.
-   The expected configuration or mounted files are present.
-   The container has the diagnostic utility you are trying to use.
-   **Failure signals:** `container not found`, `container not running`,
    `command not found`, or permission errors.
-   Minimal production images may not include a shell, `curl`, or
    `wget`. Use an approved ephemeral debugging container or a dedicated
    diagnostic Pod when necessary.
-   Avoid changing application files or state manually through `exec`;
    such changes are usually not reproducible.

------------------------------------------------------------------------

## 38. Verify network connectivity and DNS from the workload

### 🎯 The Goal --- Concept

A Pod can be healthy but still fail to reach its Service, database,
cache, or external endpoint. Network checks help isolate DNS, routing,
policy, and application-listener problems.

Think of checking the full delivery route: can we find the destination,
reach its address, and communicate through the correct port?

### 💻 The Command

Use tools available in the application container:

```bash
kubectl exec -n production <pod-name> -- \
  nslookup myapp-service.production.svc.cluster.local
```

Test the Service endpoint:

```bash
kubectl exec -n production <pod-name> -- \
  wget -qO- http://myapp-service.production.svc.cluster.local
```

If the application image lacks diagnostic tools, use an approved
temporary diagnostic Pod:

```bash
kubectl run net-debug -n production --rm -it \
  --image=busybox:1.36 --restart=Never -- sh
```

### 🖥️ Expected Terminal Output

```text
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      myapp-service.production.svc.cluster.local
Address 1: 10.96.120.15
```

### 🔍 What to Verify

-   DNS resolves the Service name to the expected ClusterIP.
-   The Service has ready EndpointSlices.
-   The application listens on the expected target port.
-   NetworkPolicies permit the required traffic.
-   **Failure signals:** DNS lookup failure, connection timeout,
    connection refused, or an empty EndpointSlice.
-   `connection refused` often means the destination was reached but no
    process accepted the connection on that port. A timeout can indicate
    routing, firewall, policy, or an unresponsive destination.

------------------------------------------------------------------------

## 39. Monitor live CPU and memory usage

### 🎯 The Goal --- Concept

Resource usage tells us what the workload consumes while it is running.
This helps identify CPU saturation, memory growth, and whether requests
and limits are realistic.

Think of it as checking fuel and power consumption while a machine is
doing real work.

### 💻 The Command

```bash
kubectl top nodes
kubectl top pods -n production
kubectl top pods -n production --containers
```

### 🖥️ Expected Terminal Output

```text
$ kubectl top nodes
NAME       CPU(cores)   CPU(%)   MEMORY(bytes)   MEMORY(%)
worker-1   650m         16%      2400Mi          31%
worker-2   720m         18%      2600Mi          34%
worker-3   590m         15%      2200Mi          28%

$ kubectl top pods -n production
NAME                     CPU(cores)   MEMORY(bytes)
myapp-7d8f9c6b5d-abcde   120m         150Mi
myapp-7d8f9c6b5d-fghij   135m         158Mi
myapp-7d8f9c6b5d-klmno   110m         147Mi

$ kubectl top pods -n production --containers
POD                      NAME    CPU(cores)   MEMORY(bytes)
myapp-7d8f9c6b5d-abcde   myapp   120m         150Mi
myapp-7d8f9c6b5d-fghij   myapp   135m         158Mi
```

### 🔍 What to Verify

-   Node and Pod usage is within expected operating ranges.
-   Compare usage with configured requests and limits.
-   Look for unusual increases, especially memory growth over time.
-   **Failure signals:** usage near memory limits, CPU throttling, node
    pressure, or unexpected resource spikes.
-   `kubectl top` requires Metrics Server or a compatible metrics API.
-   A single snapshot is not enough to identify trends. Use Prometheus
    and Grafana for historical data and percentiles.

------------------------------------------------------------------------

## 40. Expose application metrics, alerts, and SLO dashboards

### 🎯 The Goal --- Concept

Logs describe events. Metrics show numerical behavior over time. Alerts
notify the team when important conditions become unhealthy. Dashboards
help us understand whether users are receiving the expected service.

Think of a vehicle dashboard: speed, temperature, fuel, and warning
lights provide different views of the same machine.

### 💻 The Command

The source uses Prometheus client instrumentation in a Node.js
application:

``` javascript
const client = require('prom-client');
const collectDefaultMetrics = client.collectDefaultMetrics;

collectDefaultMetrics({ prefix: 'myapp_' });

const httpRequestDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1, 5]
});

app.get('/metrics', (req, res) => {
  res.set('Content-Type', client.register.contentType);
  res.end(client.register.metrics());
});
```

Pod annotations from the source:

``` yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "3000"
    prometheus.io/path: "/metrics"
```

Inspect the application endpoint:

```bash
kubectl exec -n production <pod-name> -- \
  wget -qO- http://127.0.0.1:3000/metrics
```

The source also provides alert examples for crash looping, high error
rate, OOM kills, and Deployment replica mismatch. A simplified
Prometheus rule example is:

``` yaml
groups:
  - name: myapp-critical
    rules:
      - alert: PodCrashLooping
        expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Pod {{ $labels.pod }} is crash looping"

      - alert: DeploymentReplicasMismatch
        expr: kube_deployment_spec_replicas != kube_deployment_status_available_replicas
        for: 5m
        labels:
          severity: warning
```

Example SLO targets from the source:

``` yaml
SLOs:
  availability: 99.9%
  latency_p99: 500ms
  error_rate: 0.1%
```

### 🖥️ Expected Terminal Output

```text
# HELP process_cpu_user_seconds_total Total user CPU time spent in seconds.
# TYPE process_cpu_user_seconds_total counter
process_cpu_user_seconds_total 12.42

# HELP http_request_duration_seconds Duration of HTTP requests in seconds
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{le="0.1"} 245
http_request_duration_seconds_bucket{le="0.5"} 310
```

### 🔍 What to Verify

-   The `/metrics` endpoint returns Prometheus-formatted metrics.
-   Prometheus successfully scrapes the application.
-   Alerts are loaded and evaluated by the alerting system.
-   Dashboards include request rate, error rate, P50/P95/P99 latency,
    Pod count, CPU/memory usage, and restarts.
-   SLO targets are measurable and have clear owners.
-   **Failure signals:** missing metrics, scrape errors, alerts that
    never evaluate, high error rates, increasing latency, or replica
    mismatch.
-   Prometheus annotations are not universally auto-discovered by every
    Prometheus installation. Confirm the scrape configuration or
    ServiceMonitor/PodMonitor setup used by your environment.

------------------------------------------------------------------------

# Phase 6 --- Resilience, recovery, and launch readiness

The workload is running and observable. The final step is to prove that
it can survive expected operational events and that the team can recover
it safely.

## 41. Configure centralized logging

> This is an additional operational implementation of the source's
> centralized logging practice. The original list has 40 topics; this
> section expands that topic into a practical procedure without changing
> the original topic count.

### 🎯 The Goal --- Concept

A Pod's local filesystem is temporary. If a Pod is deleted or
rescheduled, local files may disappear. Centralized logging collects
logs from workloads and sends them to a system where they can be
searched and retained.

Think of it as sending every department's daily records to one central
archive instead of leaving them in individual rooms.

### 💻 The Command

The source describes Fluent Bit as a DaemonSet. A DaemonSet runs a Pod
on each eligible node.

A simplified inspection workflow after the logging stack is deployed:

```bash
kubectl get daemonset -n logging
kubectl get pods -n logging -o wide
kubectl logs -n logging -l app=fluent-bit --tail=100
```

The source's Fluent Bit configuration mounts `/var/log` and a ConfigMap:

``` yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
spec:
  selector:
    matchLabels:
      app: fluent-bit
  template:
    metadata:
      labels:
        app: fluent-bit
    spec:
      tolerations:
        - operator: Exists
      containers:
        - name: fluent-bit
          image: fluent/fluent-bit:2.2
          volumeMounts:
            - name: varlog
              mountPath: /var/log
            - name: config
              mountPath: /fluent-bit/etc
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: config
          configMap:
            name: fluent-bit-config
```

### 🖥️ Expected Terminal Output

```text
NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE
fluent-bit   3         3         3       3            3

NAME                READY   STATUS    RESTARTS
fluent-bit-abcde    1/1     Running   0
fluent-bit-fghij    1/1     Running   0
fluent-bit-klmno    1/1     Running   0
```

### 🔍 What to Verify

-   The DaemonSet has a ready Pod on every eligible node.
-   Fluent Bit can read the intended log paths.
-   Its output destination is reachable and receiving records.
-   **Failure signals:** missing node agents, configuration errors,
    permission errors, or output delivery failures.
-   The example is simplified. Production logging needs an explicit
    backend, retention policy, resource requests/limits, security
    review, and appropriate tolerations.

------------------------------------------------------------------------

## 42. Test backup and restore with Velero

> The original source counts backup and restore as one of its 40
> practices. This section expands that practice into separate
> operational checks.

### 🎯 The Goal --- Concept

A backup is useful only if we can restore it. Velero can back up
Kubernetes resources and, with the correct integrations, persistent
volume data.

Think of a backup as a spare key. We must test that the key actually
opens the door before an emergency.

### 💻 The Command

The source provides an AWS-oriented installation example:

```bash
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.8.0 \
  --bucket my-velero-backups \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1
```

Create a scheduled backup:

```bash
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --ttl 720h0m0s \
  --include-namespaces production
```

Inspect backups:

```bash
velero schedule get
velero backup get
velero backup describe <backup-name>
```

Restore from a completed backup in a controlled test environment:

```bash
velero restore create --from-backup <backup-name>
velero restore get
velero restore describe <restore-name>
```

Then verify Kubernetes resources:

```bash
kubectl get pods -n production
kubectl get pvc -n production
```

### 🖥️ Expected Terminal Output

```text
NAME                         STATUS      ERRORS   WARNINGS   CREATED
daily-backup-20261102-020000 Completed   0        0          2026-11-02 02:00:00

NAME                         BACKUP                      STATUS
restore-test                 daily-backup-20261102-020000 Completed
```

### 🔍 What to Verify

-   Backup storage location is available.
-   The backup reaches `Completed`.
-   Errors and warnings are reviewed, not ignored.
-   Restore reaches `Completed`.
-   Restored Pods, PVCs, and application data are actually usable.
-   **Failure signals:** backup `PartiallyFailed` or `Failed`, missing
    volumes, restore errors, or corrupted/inconsistent application data.
-   Test restores in an isolated environment. Do not restore over
    production resources without a carefully approved plan.
-   The source's example uses an AWS plugin and region. Adapt it to the
    actual object-storage provider and snapshot system.

------------------------------------------------------------------------

## 43. Test node maintenance and failure recovery

> The original source includes node drain, Pod rescheduling, service
> testing, and uncordon as one practice. This section explains the
> sequence separately because it is operationally sensitive.

### 🎯 The Goal --- Concept

A node drain marks a node unschedulable and evicts eligible Pods so they
can run elsewhere. It is commonly used before planned node maintenance.

Think of temporarily closing one work area and moving its active tasks
to other available work areas.

### 💻 The Command

**Warning:** Perform this only in a lab or under an approved maintenance
procedure. Confirm PDBs, capacity, storage behavior, and workload
ownership first.

```bash
kubectl get pdb -n production
kubectl get pods -n production -o wide

kubectl drain worker-1 \
  --ignore-daemonsets \
  --delete-emptydir-data
```

Verify rescheduling and service health:

```bash
kubectl get nodes
kubectl get pods -n production -o wide
curl -f https://app.example.com/health
```

Return the node to scheduling:

```bash
kubectl uncordon worker-1
kubectl get nodes
```

### 🖥️ Expected Terminal Output

```text
node/worker-1 cordoned
evicting pod production/myapp-...
pod/myapp-... evicted
node/worker-1 drained

NAME       STATUS
worker-1   Ready,SchedulingDisabled
worker-2   Ready
worker-3   Ready
```

After uncordon:

```text
node/worker-1 uncordoned
```

### 🔍 What to Verify

-   PDBs permit the planned disruption.
-   Remaining nodes have enough capacity for rescheduled Pods.
-   Pods become ready on other nodes.
-   Service endpoints remain available.
-   External health checks continue to succeed.
-   Node returns to `Ready` and schedulable after `uncordon`.
-   **Failure signals:** drain blocked by PDB, Pods stuck `Pending`,
    volume attachment failures, loss of service endpoints, or failed
    external health checks.
-   `--delete-emptydir-data` permits deletion of Pods using `emptyDir`;
    data in those directories is lost. Understand this before running
    the command.

------------------------------------------------------------------------

## 44. Perform a pre-launch load test

> The original source includes a k6 load test as its final readiness
> practice. This section gives the practical test flow.

### 🎯 The Goal --- Concept

A load test checks how an application behaves under expected and
higher-than-expected traffic. It can expose CPU, memory, database,
cache, network, and scaling bottlenecks before real users encounter
them.

Think of testing a bridge with controlled traffic before opening it to
the public.

### 💻 The Command

The source provides this k6 script:

``` javascript
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 100 },
    { duration: '5m', target: 100 },
    { duration: '2m', target: 500 },
    { duration: '2m', target: 0 },
  ],
  thresholds: {
    http_req_duration: ['p(99)<500'],
    http_req_failed: ['rate<0.01'],
  },
};

export default function () {
  const res = http.get('https://app.example.com/api/health');
  check(res, {
    'status is 200': (r) => r.status === 200,
  });
}
```

Run it:

```bash
k6 run loadtest.js
```

While the test runs, monitor:

```bash
kubectl get hpa -n production --watch
kubectl top pods -n production
kubectl get pods -n production -w
```

### 🖥️ Expected Terminal Output

```text
     checks.........................: 100.00% ✓ 120000  ✗ 0
     http_req_duration..............: avg=120ms p(95)=280ms p(99)=420ms
     http_req_failed................: 0.00%   ✓ 0       ✗ 120000
     http_reqs......................: 120000  181.2/s
```

### 🔍 What to Verify

-   The test reaches the planned virtual-user stages.
-   p99 latency remains below the agreed threshold.
-   Error rate remains below the agreed threshold.
-   HPA scales when demand requires it and returns gradually when demand
    falls.
-   Memory stays below limits; no OOM kills or crash loops occur.
-   Database connection pools and downstream services remain healthy.
-   **Failure signals:** p99 latency above target, increasing error
    rate, CPU throttling, memory pressure, exhausted database
    connections, or HPA unable to scale.
-   Run load tests against an approved environment and test data. Do not
    unexpectedly load-test a production service.

------------------------------------------------------------------------

## 45. Maintain an incident runbook and rollback procedure

> The source includes runbook documentation and rollback commands as one
> readiness practice. This section puts the response sequence into an
> actionable format.

### 🎯 The Goal --- Concept

A runbook is a written procedure for responding to a known operational
condition. It reduces guesswork during an incident and helps engineers
perform consistent checks.

Think of it as an emergency checklist. When pressure is high, the team
should not need to invent the troubleshooting sequence.

### 💻 The Command

Example commands for a high-error-rate incident:

```bash
# 1. Check workload state
kubectl get pods -n production

# 2. Inspect recent application logs
kubectl logs -n production -l app=myapp --tail=100

# 3. Review recent rollout history
kubectl rollout history deployment/myapp -n production

# 4. Inspect the current Deployment
kubectl describe deployment myapp -n production

# 5. Roll back only when evidence indicates the new revision caused the issue
kubectl rollout undo deployment/myapp -n production

# 6. Verify the rollback
kubectl rollout status deployment/myapp -n production
kubectl get pods -n production -l app=myapp
```

### 🖥️ Expected Terminal Output

```text
deployment.apps/myapp rolled back

Waiting for deployment "myapp" rollout to finish...
deployment "myapp" successfully rolled out

NAME                     READY   STATUS    RESTARTS
myapp-older-version      1/1     Running   0
myapp-older-version      1/1     Running   0
myapp-older-version      1/1     Running   0
```

### 🔍 What to Verify

-   The runbook identifies the alert, user impact, and severity.
-   It lists immediate checks, safe mitigation steps, and escalation
    contacts.
-   Rollback is based on evidence and a known-good revision.
-   Rollout status and application health are checked after rollback.
-   **Failure signals:** no known-good revision, rollback blocked by
    incompatible data changes, or the error continuing after rollback.
-   A Deployment rollback does not automatically reverse database schema
    changes or external side effects. Plan backward-compatible
    migrations and recovery separately.

------------------------------------------------------------------------

# Final production launch checklist

Use this checklist after completing the relevant sections. A checked
item should mean that we verified evidence---not merely that a YAML file
exists.

## Cluster and namespace

-   [ ] Correct kubeconfig context and cluster confirmed.
-   [ ] API server readiness confirmed.
-   [ ] Worker nodes are `Ready` and have sufficient capacity.
-   [ ] System components, including DNS and networking, are healthy.
-   [ ] Dedicated production namespace exists and is `Active`.
-   [ ] Labels are consistent across Deployments, Pods, and Services.
-   [ ] LimitRange and ResourceQuota match the environment's policy.

## Configuration and security

-   [ ] ConfigMaps contain the correct non-secret configuration.
-   [ ] Secrets exist, are mounted correctly, and are not exposed in
    logs.
-   [ ] Encryption at rest and access restrictions are reviewed.
-   [ ] Application uses a dedicated ServiceAccount.
-   [ ] RBAC permissions follow least privilege.
-   [ ] ClusterRoleBindings and `cluster-admin` subjects are audited.
-   [ ] Pod Security Standards are enforced at the intended level.
-   [ ] Containers run as non-root with appropriate security contexts.
-   [ ] NetworkPolicies are tested and enforced by the CNI.

## Image, resources, and availability

-   [ ] Image vulnerability scan has completed and findings are
    reviewed.
-   [ ] Image uses an immutable digest or controlled version tag.
-   [ ] Image pull policy is intentional.
-   [ ] CPU and memory requests and limits are set.
-   [ ] Startup, liveness, and readiness probes work.
-   [ ] Health endpoints report the correct application state.
-   [ ] StorageClass uses the correct provisioner and reclaim policy.
-   [ ] PVC is `Bound` and mounted correctly.
-   [ ] Replicas are spread across appropriate failure domains.
-   [ ] Graceful termination is implemented.
-   [ ] Rolling update strategy is appropriate.
-   [ ] PDB supports the intended maintenance process.

## Deployment and traffic

-   [ ] Deployment has been applied from reviewed manifests.
-   [ ] Rollout completed successfully.
-   [ ] Desired, updated, and available replica counts match.
-   [ ] HPA receives metrics and has valid resource requests to
    calculate utilization.
-   [ ] Service selector matches the intended Pods.
-   [ ] EndpointSlices contain ready endpoints.
-   [ ] Ingress routes to the correct Service.
-   [ ] TLS certificate is valid.
-   [ ] Blue-Green or Canary traffic switching has been validated where
    required.

## Runtime and observability

-   [ ] Application logs show successful startup and dependency
    connections.
-   [ ] Pod descriptions and recent events show no unresolved warnings.
-   [ ] Container-level diagnostic checks pass.
-   [ ] DNS and Service connectivity tests pass.
-   [ ] CPU and memory usage are within expected ranges.
-   [ ] Metrics endpoint is scraped successfully.
-   [ ] Alerts are configured and tested.
-   [ ] Dashboards cover request rate, errors, latency, replicas,
    restarts, and resources.
-   [ ] Centralized logs are being collected and retained.

## Recovery and launch

-   [ ] Backup completes successfully.
-   [ ] Restore has been tested in a controlled environment.
-   [ ] Node drain and rescheduling have been tested in a suitable
    environment.
-   [ ] Application remains reachable during the planned disruption
    test.
-   [ ] Load test meets latency and error-rate thresholds.
-   [ ] HPA and resource behavior have been observed under load.
-   [ ] Incident runbook and rollback steps are documented.
-   [ ] Database and external dependency rollback considerations are
    documented.
-   [ ] Monitoring and escalation ownership are clear.

------------------------------------------------------------------------

# Essential command reference

  --------------------------------------------------------------------------------------------------------------
  Purpose                             Command
  ----------------------------------- --------------------------------------------------------------------------
  Current context                     `kubectl config current-context`

  Available contexts                  `kubectl config get-contexts`

  Cluster endpoint                    `kubectl cluster-info`

  API readiness                       `kubectl get --raw='/readyz?verbose'`

  Nodes                               `kubectl get nodes -o wide`

  Namespaces                          `kubectl get namespaces`

  Apply manifests                     `kubectl apply -f <file.yaml>`

  Pods                                `kubectl get pods -n production -o wide`

  Detailed Pod inspection             `kubectl describe pod <pod> -n production`

  Deployment status                   `kubectl get deployments -n production`

  Rollout status                      `kubectl rollout status deployment/myapp -n production`

  Rollout history                     `kubectl rollout history deployment/myapp -n production`

  Rollback                            `kubectl rollout undo deployment/myapp -n production`

  Application logs                    `kubectl logs <pod> -n production --tail=100`

  Previous container logs             `kubectl logs <pod> -n production --previous`

  Execute in container                `kubectl exec -it <pod> -n production -- sh`

  Namespace events                    `kubectl get events -n production --sort-by=.metadata.creationTimestamp`

  ConfigMaps                          `kubectl get configmaps -n production`

  Secrets metadata                    `kubectl get secrets -n production`

  ServiceAccounts                     `kubectl get serviceaccounts -n production`

  RBAC permission test                `kubectl auth can-i <verb> <resource> -n production`

  NetworkPolicies                     `kubectl get networkpolicies -n production`

  StorageClasses                      `kubectl get storageclass`

  PVCs                                `kubectl get pvc -n production`

  Services                            `kubectl get services -n production`

  EndpointSlices                      `kubectl get endpointslices -n production`

  Ingress                             `kubectl get ingress -n production`

  HPA                                 `kubectl get hpa -n production`

  PDB                                 `kubectl get pdb -n production`

  Node metrics                        `kubectl top nodes`

  Pod metrics                         `kubectl top pods -n production`

  Drain node                          `kubectl drain <node> --ignore-daemonsets --delete-emptydir-data`

  Return node to scheduling           `kubectl uncordon <node>`
  --------------------------------------------------------------------------------------------------------------

------------------------------------------------------------------------

# Final engineering principles

1.  **Inspect before changing.** Confirm context, namespace, and current
    state before applying or deleting resources.
2.  **Desired state is not actual state.** A manifest can be accepted
    while Pods remain unscheduled or unready.
3.  **Running is not the same as ready.** Check readiness probes and
    Service endpoints.
4.  **Requests and limits solve different problems.** Requests influence
    scheduling; limits constrain runtime consumption.
5.  **Security is layered.** Use least-privilege RBAC, non-root
    containers, Pod Security Standards, NetworkPolicies, protected
    Secrets, and image scanning together.
6.  **Availability needs capacity.** Replicas, anti-affinity, PDBs, and
    safe rollout settings help only when the cluster has enough
    resources and failure domains.
7.  **A Service needs endpoints.** A ClusterIP alone does not prove that
    traffic can reach the application.
8.  **Observability must be tested.** A dashboard with no reliable data
    and an alert that never fires do not provide operational protection.
9.  **Backups must be restored.** A successful backup job is not proof
    of recoverability.
10. **Practice failure safely.** Test node maintenance, rollback, and
    load behavior in a controlled environment before relying on them in
    production.

**The goal is not to memorize commands. The goal is to understand what
each command proves, what it cannot prove, and what evidence we need
before moving to the next deployment step.**
