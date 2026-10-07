# Kubernetes Pod Specification Good Defaults

The Pod spec for your apps can be one of the more complex parts of your Kubernetes manifest design, and needs many features enabled to be a safe and reasonably secure default.

This single-file repository is meant to be a starting point for your Pod specs, to add to Deployments, DaemonSets, StatefulSets, initContainers, etc.

It's based on years of consulting, the Kubernetes courses and workshops I do, and [this tweet when I first had the idea](https://twitter.com/BretFisher/status/1550326044577730560).

## Watch me [walk through this `pod.yaml` on YouTube](https://www.youtube.com/watch?v=4CzG4Uqd9jM)

## The spec from [`./pod.yaml`](./pod.yaml)

```yaml
spec:

  containers:

      # basic container details
    - name: my-container-name
      # never use reusable tags like latest or stable
      image: my-image:tag
      # hardcode the listening port if Dockerfile isn't set with EXPOSE
      ports:
        - containerPort: 8080
          protocol: TCP

      readinessProbe:        # I always recommend using these, even if your app has no listening ports (this affects any rolling update)
        httpGet:             # Lots of timeout values with defaults, be sure they are ideal for your workload
          path: /ready
          port: 8080
      livenessProbe:         # only needed if your app tends to go unresponsive or you don't have a readinessProbe, but this is up for debate
        httpGet:             # Lots of timeout values with defaults, be sure they are ideal for your workload
          path: /alive
          port: 8080

      resources:             # memory limit = request, so the scheduler reserves what the app can actually use
        limits:
          memory: "500Mi"    # If container uses over 500MB it is killed (OOM)
          #cpu: "2"          # Not normally needed, unless you need to protect other workloads or QoS must be "Guaranteed"
        requests:
          memory: "500Mi"    # Scheduler finds a node where 500MB is available
          cpu: "1"           # Scheduler finds a node where 1 vCPU is available

      # per-container security context
      # lock down privileges inside the container
      securityContext:
        allowPrivilegeEscalation: false # prevent sudo, etc.
        privileged: false               # prevent acting like host root

  terminationGracePeriodSeconds: 600 # default is 30, but you may need more time to gracefully shutdown (HTTP long polling, user uploads, etc)

  # don't mount a Kubernetes API token into the pod unless the app calls the Kubernetes API
  automountServiceAccountToken: false # a stolen token is an API credential, so only mount it when needed

  # give the pod its own Linux user namespace (GA in Kubernetes 1.36, needs Linux 6.3+ and containerd 2.0+ or CRI-O 1.25+)
  hostUsers: false           # container UIDs (even root) map to unprivileged UIDs on the host that no other pod uses

  # per-pod security context
  # enable seccomp and force non-root user
  securityContext:

    seccompProfile:
      type: RuntimeDefault   # enable seccomp and the runtimes default profile

    runAsUser: 1001          # hardcode user to non-root if not set in Dockerfile
    runAsGroup: 1001         # hardcode group to non-root if not set in Dockerfile
    runAsNonRoot: true       # hardcode to non-root. Redundant to above if Dockerfile is set USER 1000
```

## Why each setting is in the spec

Each setting below is in [`pod.yaml`](./pod.yaml) on purpose. They're grouped by what they protect: the **security** of the node and cluster, the **availability** of your app, and the **image and networking** details that make the Pod predictable.

### Security

These settings limit what a compromised or buggy container can do to the node, to other Pods, and to the cluster.

| Setting | Where | Why it's a good default |
| --- | --- | --- |
| `seccompProfile.type: RuntimeDefault` | Pod `securityContext` | Kubernetes runs containers **without** a seccomp filter (`Unconfined`) unless you ask for one, or unless the cluster admin turned on the kubelet's `seccompDefault` option. `RuntimeDefault` turns on the container runtime's default filter, which blocks dozens of syscalls a normal app never calls (kernel module loading, `reboot`, `mount`, etc). It's cheap, broad attack-surface reduction, and the Pod Security Standards "Restricted" profile requires it. |
| `runAsNonRoot: true` | Pod `securityContext` | The kubelet refuses to start the container if it would run as UID 0. This catches the image that forgot a `USER` line in its Dockerfile. Root in a container is still root on the kernel, so a container escape as root is far worse than one as a regular user. |
| `runAsUser: 1001` and `runAsGroup: 1001` | Pod `securityContext` | Forces a known non-root UID/GID, even if the image doesn't set one. Some teams require these in the manifest (or enforce them with an admission controller) so the server, not the Dockerfile, decides the user. You can remove them if your image sets a numeric non-root `USER` (also true for ko and buildpacks, thanks [@e_k_anderson](https://twitter.com/e_k_anderson/status/1550485281261817856)). Make sure the UID can read the app's files in the image. |
| `allowPrivilegeEscalation: false` | container `securityContext` | Sets the Linux `no_new_privs` flag, so a process can't gain more privileges than its parent. That blocks setuid binaries like `sudo` and file capabilities from raising privileges inside the container. Also required by the "Restricted" profile. |
| `privileged: false` | container `securityContext` | This is already the default, but stating it makes the intent clear to readers and reviewers. A privileged container gets all capabilities and access to the host's devices, which is effectively host root. |
| `hostUsers: false` | Pod | Runs the Pod in its own Linux user namespace. Container UIDs, even root, map to an unprivileged UID range on the host that no other Pod on the node shares. A container escape then lands as a nobody user on the host, and it can't touch other Pods' files. See [User namespaces](#user-namespaces-hostusers-false) below. |
| `automountServiceAccountToken: false` | Pod | By default, Kubernetes mounts a token for the Pod's ServiceAccount at `/var/run/secrets/kubernetes.io/serviceaccount/`. Most apps never call the Kubernetes API, so the token is only useful to an attacker who gets into the container (remote code execution, path traversal, SSRF that reads files). Turn it off by default, and set it to `true` only for Pods that talk to the API (operators, controllers, CI runners). See [Service account tokens](#service-account-tokens-automountserviceaccounttoken-false) below. |

#### User namespaces (`hostUsers: false`)

Without a user namespace, UID 1001 in your container **is** UID 1001 on the host, and root in a container **is** root on the host, limited only by capabilities, seccomp, and AppArmor/SELinux. Many container-escape CVEs give the attacker the container's UID on the host. With `hostUsers: false`, that host UID is a high, unused number that owns nothing on the host.

What you get:

- **Escapes land unprivileged.** Container root maps to a host UID with no rights to host files or devices.
- **Pods are isolated from each other.** The kubelet gives each Pod its own UID/GID range, so two Pods that both run as UID 1001 are different users on the host.
- **Capabilities are scoped to the Pod.** Even a capability like `CAP_SYS_ADMIN` is valid only inside the Pod's user namespace, not for the host. `CAP_SYS_MODULE` can't load kernel modules.
- **Nothing changes inside the container.** `runAsUser`, `runAsGroup`, `fsGroup`, and the file owners on volumes still use the in-container IDs. The kubelet uses idmap mounts, so you don't `chown` volumes.

Status and defaults:

- `hostUsers` became GA (stable) in **Kubernetes 1.36** (April 2026). It was beta and on by default since 1.33.
- GA didn't change the default. `hostUsers` still defaults to `true` (share the host's user namespace), so you must opt in per Pod.

Requirements on each node (a Pod with `hostUsers: false` on a node that can't do it fails to start, with an error in its events):

- Linux kernel **6.3 or later** (for idmap mounts on tmpfs, which Secret and service account token volumes use).
- Container runtime: **containerd 2.0+** or **CRI-O 1.25+**, with **runc 1.2+** or **crun 1.9+**.
- The filesystems for `/var/lib/kubelet/pods/` and for every volume the Pod mounts must support idmap mounts. ext4, xfs, btrfs, tmpfs, and overlayfs do. NFS doesn't.
- Check your nodes with `kubectl get nodes -o wide`, which shows the kernel and container runtime versions. Managed Kubernetes node images vary, so check them before you roll this out.

Limits:

- A Pod with `hostUsers: false` can't also use `hostNetwork`, `hostPID`, or `hostIPC`. Those Pods (CNI agents, node monitoring DaemonSets) need the host user namespace.
- In-container UIDs/GIDs above 65535 map to the overflow ID (usually 65534), so keep `runAsUser` below 65536.

**A `runAsNonRoot` gotcha:** if the Dockerfile `USER` is a name (like `node`) and not a number, you'll get `CreateContainerConfigError: container has runAsNonRoot and image has non-numeric user (node), cannot verify user is non-root`. The kubelet can't prove that a username isn't mapped to UID 0. Two fixes:

1. Use a numeric `USER` in the Dockerfile (for example `USER 1000`), or
2. Set `runAsUser` in the manifest. The kubelet checks `runAsUser` first, and when it's set and non-zero, the kubelet skips the image's `USER` check.

#### Service account tokens (`automountServiceAccountToken: false`)

Every Pod runs as a ServiceAccount (the namespace's `default` one if you don't set `serviceAccountName`). Unless you turn it off, the kubelet mounts a short-lived, auto-rotated token for that ServiceAccount into every container.

Why turn it off:

- **Least privilege.** If the app doesn't call the Kubernetes API, it doesn't need a credential for it.
- **Less to steal.** A token in a file is easy to read after a remote code execution or a file-read bug. With it, an attacker can call the API as your ServiceAccount, and find out what the RBAC allows.
- **RBAC drift.** Today the `default` ServiceAccount may have no permissions, but someone can add a RoleBinding to it later. A Pod with no token is safe from that change.

Things to know:

- You can set it on the Pod (as this spec does) or on the ServiceAccount object. The Pod setting wins when both are set.
- If the app needs the API, set `automountServiceAccountToken: true`, set `serviceAccountName` to a dedicated ServiceAccount, and give that ServiceAccount only the RBAC it needs. Don't give permissions to the `default` ServiceAccount.
- The token is not the only way a Pod gets identity. If the app needs a token for another audience (a cloud provider, Vault), use a [projected `serviceAccountToken` volume](https://kubernetes.io/docs/concepts/storage/projected-volumes/#serviceaccounttoken) with its own `audience` and `expirationSeconds`. That works with `automountServiceAccountToken: false`.

### Availability

These settings keep your app serving traffic during deploys, node pressure, and shutdowns.

| Setting | Where | Why it's a good default |
| --- | --- | --- |
| `readinessProbe` | container | Kubernetes sends Service traffic only to Pods that pass this check, and a rolling update waits for new Pods to be ready before it removes old ones. Without it, a Pod is "ready" the moment its process starts, so users hit an app that is still booting. I recommend it even for apps with no listening port (use an `exec` probe), because it controls rolling update speed. |
| `livenessProbe` | container | The kubelet restarts the container when this check fails. Only add it if your app is known to hang (deadlock, stuck connection pool). A bad liveness probe can cause a restart loop during a slow start or a traffic spike, so this one is up for debate. Don't make it check a dependency, like the database, or a database outage restarts every Pod. |
| Probe timing values | each probe | The defaults (`periodSeconds: 10`, `timeoutSeconds: 1`, `failureThreshold: 3`) are often wrong for slow-starting apps like JVMs and Rails. Tune them for your workload, or add a `startupProbe` for apps with a long boot. |
| `resources.requests.memory` and `resources.requests.cpu` | container | The scheduler uses requests to pick a node with enough free capacity. Without requests, the scheduler can pack too many Pods on one node, and your Pod is first in line for eviction under node pressure. |
| `resources.limits.memory` equal to `requests.memory` | container | Memory can't be throttled, only reclaimed by killing a process. With limit = request, the scheduler reserves all the memory the app can use, so the node doesn't run out because of overcommit. The container is OOM-killed if it goes over the limit, which is easier to see and fix than a node-wide memory problem. |
| No CPU limit | container | CPU is compressible: when a container wants more than its request, it gets idle CPU if there is some. A CPU limit throttles the app even when the node is idle, which hurts latency. Add one only to protect other workloads, or if you need `Guaranteed` QoS (see below). |
| `terminationGracePeriodSeconds: 600` | Pod | The default is 30 seconds. After `SIGTERM`, the kubelet waits this long before it sends `SIGKILL`. Long HTTP polls, WebSockets, file uploads, and queue workers often need more time to finish. This is a maximum: an app that exits early doesn't wait the full 600 seconds. |

**About QoS classes:** Kubernetes gives each Pod a [Quality of Service (QoS) class](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/), which decides which Pods the kubelet evicts first when a node runs out of resources. A Pod is `Guaranteed` only when **every** container sets **both** CPU and memory limits equal to its requests. This spec has no CPU limit, so it gets `Burstable`. That's a deliberate trade: no CPU throttling, and memory is still safe because limit = request. If you need `Guaranteed` (for example, for CPU pinning with the static CPU manager), uncomment the CPU limit and set it equal to the CPU request.

### Image and networking

These settings make the Pod predictable: the same image every time, and a known port.

| Setting | Where | Why it's a good default |
| --- | --- | --- |
| `image: my-image:tag` with a fixed tag | container | Never use tags that move, like `latest` or `stable`. A moving tag means two Pods of the same Deployment can run different code, a rollback may not roll back, and you can't tell later which code ran during an incident. Use a semver, git SHA, or build ID tag. A digest (`my-image:tag@sha256:...`) is the strongest pin. |
| No `imagePullPolicy` | container | You can likely leave this out, because the [defaults are smart and tend to do the right thing](https://kubernetes.io/docs/concepts/containers/images/#imagepullpolicy-defaulting): `Always` for `latest` or no tag, `IfNotPresent` for everything else. |
| `ports.containerPort: 8080` | container | Hardcode the listening port, because many images don't set `EXPOSE` in the Dockerfile. It documents the port for readers, and you can give it a `name` so probes and Services refer to the port by name. |

## Additional factors and suggestions that affect pod spec

- If you have over ~1,000 services in a namespace, maybe set `pod.spec.enableServiceLinks: false` to avoid [minor container startup and TCP round-trip delays](https://github.com/knative/serving/issues/8498) thanks [@e_k_anderson](https://twitter.com/e_k_anderson/status/1550486493868826630).
- `pod.spec.containers.securityContext.readOnlyRootFilesystem` is a good idea if possible, but usually doesn't work out-of-the-box with monoliths and traditional apps. [YMMV](https://en.wiktionary.org/wiki/your_mileage_may_vary).
