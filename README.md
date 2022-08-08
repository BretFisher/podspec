# Kubernetes Pod Specification Good Defaults

The Pod spec for your apps can be one of the more complex parts of your Kubernetes manifest design, and needs many features enabled to be a save and reasonably secure default.

This single-file repository is meant to be a starting point for your Pod specs, to add to Deployments, DaemonSets, StatefulSets, initContainers, etc.

It's based on years of consulting, the Kubernetes courses and workshops I do, and [this tweet when I first had the idea](https://twitter.com/BretFisher/status/1550326044577730560).

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

      readinessProbe:        # only needed if your pod has a service and listening port
        httpGet:             # Lots of timeout values with defaults, be sure they are ideal for your workload
          path: /ready
          port: 8080
      livenessProbe:         # only needed if your app tends to go unresponsive or you don't have a readinessProbe, but this is up for debate
        httpGet:             # Lots of timeout values with defaults, be sure they are ideal for your workload
          path: /alive
          port: 8080

      resources:             # Because if limits = requests then QoS is set to "Guaranteed"
        limits:
          memory: "500Mi"    # If container uses over 500MB it is killed (OOM)
          cpu: "2"           # If container uses over 2 vCPU it is throttled
        requests:
          memory: "500Mi"    # Scheduler finds a node where 500MB is available
          cpu: "1"           # Scheduler finds a node where 1 vCPU is available

      # per-container security context
      # lock down privileges inside the container
      securityContext:
        allowPrivilegeEscalation: false # prevent sudo, etc.
        privileged: false               # prevent acting like host root


  terminationGracePeriodSeconds: 600 # default is 30, but you may need more time to gracefully shutdown (HTTP long polling, user uploads, etc)

  # per-pod security context
  # enable seccomp and force non-root user
  securityContext:

    seccompProfile:
      type: RuntimeDefault   # enable seccomp and the runtimes default profile

    runAsUser: 1001          # hardcode user to non-root if not set in Dockerfile
    runAsGroup: 1001         # hardcode group to non-root if not set in Dockerfile
    runAsNonRoot: true       # hardcode to non-root. Redundant to above if Dockerfile is set USER 1000
```

## Additional factors and suggestions that affect pod spec

- You can remove `runAsUser/runAsGroup` if you are using a Dockerfile that sets the user/group to non-root (or ko or buildpacks, thanks [@e_k_anderson](https://twitter.com/e_k_anderson/status/1550485281261817856)), but some teams will still require these values hardcoded in the manifest (or in admission controller) to enforce at the server-side.
- If `runAsNonRoot` is true (as it should be), you may get error `CreateContainerConfigError: Error: container has runAsNonRoot and image has non-numeric user (username), cannot verify user is non-root.` if your Dockerfile `USER` isn't an ID. Kubernetes wants it as an ID (not friendly username like `node`) to ensure it's not just a user mapping to UID 0 (root). I think this can be avoided if you hardcode the user as well in the manifest (`runAsUser`), but I haven't tested that.
- If you have over ~1,000 services in a namespace, maybe set `pod.spec.enableServiceLinks: false` to avoid [minor container startup and TCP round-trip delays](https://github.com/knative/serving/issues/8498) thanks [@e_k_anderson](https://twitter.com/e_k_anderson/status/1550486493868826630).
- You can likely avoid needing `pod.spec.containers.imagePullPolicy` because the [defaults are smart and tend to do the right thing](https://kubernetes.io/docs/concepts/containers/images/#imagepullpolicy-defaulting).
- `pod.spec.containers.securityContext.readOnlyRootFilesystem` is a good idea if possible, but usually doesn't work out-of-the-box with monoliths and traditional apps. [YMMV](https://en.wiktionary.org/wiki/your_mileage_may_vary).
