# 🎯 CKAD Practice Questions — Days 1–14

> **Coverage**: Pods · Namespaces · Labels & Selectors · kubectl · ReplicaSets · Deployments · Services · ConfigMaps · Secrets · Env Vars · Resource Limits
>
> **Format**: Each question has a real exam-style scenario, setup commands, verification steps, and collapsed hints/solutions.
>
> **How to use**: Read the scenario → try to solve it yourself → check the hint if stuck → reveal solution only after your attempt → read the explanation.

---

## 📋 Quick Reference — Exam Context

```bash
# Always check your current context before starting
kubectl config current-context
kubectl config get-contexts

# Switch namespace quickly (saves time in exam)
kubectl config set-context --current --namespace=<ns>

# Dry-run trick — generates YAML without creating resources
kubectl run mypod --image=nginx --dry-run=client -o yaml > pod.yaml

# Force delete (useful during practice)
kubectl delete pod mypod --grace-period=0 --force
```

---

## 🟢 SECTION 1 — Pods (Questions 1–12)

---

### Q1 · Basic Pod Creation

**Scenario:** You are a developer at a startup. Your team needs a simple web server running for a quick demo. Create a Pod named `demo-web` in the `default` namespace using the `nginx:1.25` image.

**Setup:** No prior setup needed.

**Your Task:**
- Create the Pod using a **YAML manifest** (not just `kubectl run`)
- The Pod must have the label `app=demo`
- Verify the Pod is running

---

<details>
<summary>💡 Hint</summary>

Use `kubectl run` with `--dry-run=client -o yaml` to generate the base YAML, then add the label under `metadata.labels`.

</details>

<details>
<summary>✅ Solution</summary>

```yaml
# pod-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-web
  namespace: default
  labels:
    app: demo
spec:
  containers:
    - name: web
      image: nginx:1.25
```

```bash
kubectl apply -f pod-demo.yaml
```

**Verify:**
```bash
kubectl get pod demo-web --show-labels
# Expected output:
# NAME       READY   STATUS    RESTARTS   AGE   LABELS
# demo-web   1/1     Running   0          10s   app=demo
```

</details>

<details>
<summary>📖 Explanation</summary>

A **Pod** is the smallest deployable unit in Kubernetes — think of it as a wrapper around one or more containers. Every Pod needs:
- `apiVersion: v1` — the API group for Pods
- `kind: Pod` — what we're creating
- `metadata.name` — the Pod's name
- `spec.containers` — at least one container definition with `name` and `image`

Labels (`app: demo`) are key-value pairs used to organize and select resources — crucial for Services and ReplicaSets.

</details>

---

### Q2 · Pod with Multiple Environment Variables

**Scenario:** Your app requires runtime configuration. Create a Pod named `config-app` using the `busybox:1.36` image. The container should set these environment variables:
- `APP_ENV=production`
- `APP_PORT=8080`
- `APP_VERSION=1.0.0`

The container command should be: `["sh", "-c", "env && sleep 3600"]`

---

<details>
<summary>💡 Hint</summary>

Environment variables go under `spec.containers[].env`. Each variable is an object with `name` and `value` fields.

</details>

<details>
<summary>✅ Solution</summary>

```yaml
# config-app-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-app
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "env && sleep 3600"]
      env:
        - name: APP_ENV
          value: "production"
        - name: APP_PORT
          value: "8080"
        - name: APP_VERSION
          value: "1.0.0"
```

```bash
kubectl apply -f config-app-pod.yaml
```

**Verify:**
```bash
kubectl logs config-app | grep -E "APP_ENV|APP_PORT|APP_VERSION"
# Expected:
# APP_ENV=production
# APP_PORT=8080
# APP_VERSION=1.0.0
```

</details>

<details>
<summary>📖 Explanation</summary>

`env` is a list under each container spec. Every item needs `name` (the variable name) and `value` (a string — note that `8080` must be quoted as `"8080"` in YAML, otherwise it's parsed as an integer).

`kubectl logs` reads the stdout of the container. Since our command runs `env`, all environment variables are printed at startup.

</details>

---

### Q3 · Pod Inspection & Debugging

**Scenario:** A teammate says "the `broken-pod` is not working." You need to find out what's wrong.

**Setup — Run this first:**
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: broken-pod
spec:
  containers:
    - name: app
      image: nginx:INVALID_TAG
EOF
```

**Your Task:** Identify the problem and explain what you would do to fix it (without deleting the Pod).

---

<details>
<summary>💡 Hint</summary>

Use `kubectl describe pod broken-pod` and look at the **Events** section at the bottom. Also try `kubectl get pod broken-pod -o wide`.

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Step 1: Check pod status
kubectl get pod broken-pod
# STATUS will be: ErrImagePull or ImagePullBackOff

# Step 2: Describe to find the root cause
kubectl describe pod broken-pod
# Look at Events section — you'll see:
# Failed to pull image "nginx:INVALID_TAG": ... not found

# Step 3: Fix — patch the image (no delete needed)
kubectl set image pod/broken-pod app=nginx:1.25

# Step 4: Verify
kubectl get pod broken-pod
# Should transition to Running
```

**Verify:**
```bash
kubectl get pod broken-pod
# NAME         READY   STATUS    RESTARTS   AGE
# broken-pod   1/1     Running   0          30s
```

</details>

<details>
<summary>📖 Explanation</summary>

`ImagePullBackOff` means Kubernetes tried to pull the container image, failed, and is now backing off (waiting longer each retry). The two most common causes:
1. **Invalid tag** — the image tag doesn't exist in the registry
2. **Wrong image name** — typo in the image name
3. **Private registry** — missing imagePullSecrets

`kubectl describe` is your #1 debugging tool. Always scroll to the **Events** section first.

`kubectl set image` can patch a running Pod's image without deleting it — a huge time-saver in the exam.

</details>

---

### Q4 · Pod with Resource Limits

**Scenario:** Production policy requires all Pods to declare resource requests and limits. Create a Pod named `resource-pod` with:
- Image: `nginx:1.25`
- CPU request: `100m`, limit: `200m`
- Memory request: `64Mi`, limit: `128Mi`

---

<details>
<summary>💡 Hint</summary>

Resources go under `spec.containers[].resources`. There are two sub-fields: `requests` (what the scheduler reserves) and `limits` (the hard cap).

</details>

<details>
<summary>✅ Solution</summary>

```yaml
# resource-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-pod
spec:
  containers:
    - name: web
      image: nginx:1.25
      resources:
        requests:
          cpu: "100m"
          memory: "64Mi"
        limits:
          cpu: "200m"
          memory: "128Mi"
```

```bash
kubectl apply -f resource-pod.yaml
```

**Verify:**
```bash
kubectl describe pod resource-pod | grep -A 6 "Limits\|Requests"
# Limits:
#   cpu:     200m
#   memory:  128Mi
# Requests:
#   cpu:     100m
#   memory:  64Mi
```

</details>

<details>
<summary>📖 Explanation</summary>

**CPU units:** `100m` = 100 millicores = 0.1 of one CPU core. `1000m` = 1 full CPU core.

**Memory units:** `Mi` = Mebibytes (1 Mi = 1,048,576 bytes). `M` = Megabytes (slightly different). Always use `Mi` for Kubernetes.

**Requests vs Limits:**
- `requests` → what the scheduler uses to decide which node to place the Pod on
- `limits` → the hard maximum; exceeding memory limit causes the container to be **OOMKilled**; exceeding CPU limit causes **throttling** (not killed)

</details>

---

### Q5 · Pod with a Specific Node Selector

**Scenario:** Your cluster has nodes labeled by environment. Create a Pod named `prod-pod` using `redis:7` that only runs on nodes labeled `env=production`.

**Setup:**
```bash
# Label a node (use your minikube node name)
kubectl label node minikube env=production
```

---

<details>
<summary>💡 Hint</summary>

Use `spec.nodeSelector` — it's a simple key-value map. The Pod will only be scheduled on nodes with a matching label.

</details>

<details>
<summary>✅ Solution</summary>

```yaml
# prod-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: prod-pod
spec:
  nodeSelector:
    env: production
  containers:
    - name: cache
      image: redis:7
```

```bash
kubectl apply -f prod-pod.yaml
```

**Verify:**
```bash
kubectl get pod prod-pod -o wide
# NODE column should show: minikube

kubectl describe pod prod-pod | grep "Node-Selectors"
# Node-Selectors: env=production
```

</details>

<details>
<summary>📖 Explanation</summary>

`nodeSelector` is the simplest way to constrain which node a Pod runs on. It's a map of key-value pairs — the node must have ALL the labels listed.

Think of it like: "I only want to work in the Production office (`env=production`)."

If no node matches, the Pod will remain in `Pending` state — this is a very common debugging scenario in the exam.

</details>

---

### Q6 · Static Pod (Imperative + YAML)

**Scenario:** You need to create a Pod named `heartbeat` that simply echoes "alive" every 5 seconds and never stops. Use image `busybox:1.36`.

---

<details>
<summary>💡 Hint</summary>

Use `command` with a shell loop. The pattern is `["sh", "-c", "while true; do echo alive; sleep 5; done"]`.

</details>

<details>
<summary>✅ Solution</summary>

```yaml
# heartbeat-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: heartbeat
spec:
  containers:
    - name: ping
      image: busybox:1.36
      command: ["sh", "-c", "while true; do echo alive; sleep 5; done"]
```

```bash
kubectl apply -f heartbeat-pod.yaml
```

**Verify:**
```bash
kubectl logs -f heartbeat
# alive
# alive
# alive  (every 5 seconds)

# Press Ctrl+C to stop following
```

</details>

<details>
<summary>📖 Explanation</summary>

`command` overrides the container's default entrypoint (ENTRYPOINT in Dockerfile). When you pass a shell command with `sh -c`, you can use shell features like `while true`.

`kubectl logs -f` streams logs in real time (`-f` = follow), similar to `tail -f` in Linux.

This pattern (infinite loop + sleep) is very common in CKAD exam tasks to keep a container alive for testing.

</details>

---

### Q7 · Pod with Init Container

**Scenario:** Your app Pod must wait for a config file to be written before starting. Create a Pod `init-demo` where:
- An **init container** named `setup` (image: `busybox:1.36`) writes `"config ready"` to `/shared/config.txt`
- The **main container** named `app` (image: `busybox:1.36`) reads and prints that file, then sleeps

Both containers share a volume named `shared-data`.

---

<details>
<summary>💡 Hint</summary>

Init containers go under `spec.initContainers` (same structure as `spec.containers`). Volumes are defined under `spec.volumes` and mounted via `volumeMounts` in each container.

</details>

<details>
<summary>✅ Solution</summary>

```yaml
# init-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  volumes:
    - name: shared-data
      emptyDir: {}
  initContainers:
    - name: setup
      image: busybox:1.36
      command: ["sh", "-c", "echo 'config ready' > /shared/config.txt"]
      volumeMounts:
        - name: shared-data
          mountPath: /shared
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "cat /shared/config.txt && sleep 3600"]
      volumeMounts:
        - name: shared-data
          mountPath: /shared
```

```bash
kubectl apply -f init-demo.yaml
```

**Verify:**
```bash
kubectl get pod init-demo
# Watch it go: Init:0/1 → PodInitializing → Running

kubectl logs init-demo -c app
# config ready
```

</details>

<details>
<summary>📖 Explanation</summary>

**Init containers** run to completion BEFORE any main container starts. They're perfect for:
- Pre-loading configs or data
- Waiting for a dependency (e.g., a database) to be ready
- Running setup scripts

`emptyDir` is a temporary volume that lives as long as the Pod. It starts empty and is shared between all containers in the Pod. Think of it as a shared folder in RAM.

The key insight: init containers and main containers share the same volumes — that's how they pass data.

</details>

---

### Q8 · Pod Exec & Interactive Shell

**Scenario:** A running Pod `debug-target` has a mysterious file at `/tmp/secret.txt`. Without deleting or restarting the Pod, find its contents.

**Setup:**
```bash
kubectl run debug-target --image=busybox:1.36 \
  --command -- sh -c "echo 'password=k8s_r0cks' > /tmp/secret.txt && sleep 3600"
```

---

<details>
<summary>💡 Hint</summary>

Use `kubectl exec` to run a command inside a running container. The syntax is `kubectl exec <pod> -- <command>`.

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Option 1: Run a single command
kubectl exec debug-target -- cat /tmp/secret.txt
# password=k8s_r0cks

# Option 2: Get an interactive shell
kubectl exec -it debug-target -- sh
# Inside the shell:
# / # cat /tmp/secret.txt
# / # exit
```

**Verify:**
```bash
kubectl exec debug-target -- cat /tmp/secret.txt
# password=k8s_r0cks
```

</details>

<details>
<summary>📖 Explanation</summary>

`kubectl exec` runs a command inside a running container. It's like SSH-ing into the container.

- `--` separates kubectl flags from the container command
- `-it` = interactive terminal (`-i` keeps stdin open, `-t` allocates a pseudo-TTY)

In the CKAD exam, `kubectl exec -it <pod> -- sh` (or `bash` if available) is one of your most powerful debugging tools. Use it to:
- Check files and configs
- Test network connectivity with `curl` or `wget`
- Debug application issues from inside the container

</details>

---

### Q9 · Pod with Restart Policy

**Scenario:** Create a Pod named `job-runner` using `busybox:1.36` that runs `echo "Job done" && exit 0`. The Pod should **NOT restart** after completion.

---

<details>
<summary>💡 Hint</summary>

The default `restartPolicy` is `Always`. You need `Never` or `OnFailure`. For a one-time task, use `Never`.

</details>

<details>
<summary>✅ Solution</summary>

```yaml
# job-runner.yaml
apiVersion: v1
kind: Pod
metadata:
  name: job-runner
spec:
  restartPolicy: Never
  containers:
    - name: runner
      image: busybox:1.36
      command: ["sh", "-c", "echo 'Job done' && exit 0"]
```

```bash
kubectl apply -f job-runner.yaml
```

**Verify:**
```bash
kubectl get pod job-runner
# STATUS: Completed (not Restarting)

kubectl logs job-runner
# Job done
```

</details>

<details>
<summary>📖 Explanation</summary>

**Restart policies:**

| Policy | Behavior |
|---|---|
| `Always` (default) | Always restart the container, no matter the exit code |
| `OnFailure` | Restart only if exit code ≠ 0 |
| `Never` | Never restart — Pod stays in `Completed` or `Failed` state |

For batch/one-time tasks, use `Never`. For production servers (nginx, databases), `Always` is correct. In the CKAD exam, Jobs use `Never` or `OnFailure` under the hood.

</details>

---

### Q10 · Multi-Container Pod (Sidecar Pattern)

**Scenario:** You need to run an app that writes logs to a file, with a sidecar container that reads and prints those logs. Create a Pod `log-app` with:
- Container `writer` (busybox:1.36): writes a line to `/logs/app.log` every 2 seconds
- Container `reader` (busybox:1.36): tails `/logs/app.log`
- Shared volume `log-vol` mounted at `/logs` in both containers

---

<details>
<summary>💡 Hint</summary>

Both containers go under `spec.containers` (not `initContainers`). They run simultaneously. The sidecar pattern uses a shared `emptyDir` volume.

</details>

<details>
<summary>✅ Solution</summary>

```yaml
# log-app.yaml
apiVersion: v1
kind: Pod
metadata:
  name: log-app
spec:
  volumes:
    - name: log-vol
      emptyDir: {}
  containers:
    - name: writer
      image: busybox:1.36
      command: ["sh", "-c", "while true; do echo \"$(date): app running\" >> /logs/app.log; sleep 2; done"]
      volumeMounts:
        - name: log-vol
          mountPath: /logs
    - name: reader
      image: busybox:1.36
      command: ["sh", "-c", "tail -f /logs/app.log"]
      volumeMounts:
        - name: log-vol
          mountPath: /logs
```

```bash
kubectl apply -f log-app.yaml
```

**Verify:**
```bash
# Check logs from the reader sidecar
kubectl logs -f log-app -c reader
# Mon Jan 1 00:00:00 UTC 2024: app running
# (new line every 2 seconds)

# Check logs from the writer directly
kubectl logs log-app -c writer
```

</details>

<details>
<summary>📖 Explanation</summary>

The **Sidecar Pattern** is one of the most important multi-container patterns:
- Main container: does the real work
- Sidecar container: provides a supporting function (logging, proxying, config refresh)

They share the same network namespace (same IP) and can share volumes. Use `-c <container-name>` with `kubectl logs` to specify which container's logs you want in a multi-container Pod.

</details>

---

### Q11 · Pod Port Forwarding

**Scenario:** You have a Pod `web-pod` running nginx. Without creating a Service, access it from your local machine on port `9090`.

**Setup:**
```bash
kubectl run web-pod --image=nginx:1.25 --port=80
```

---

<details>
<summary>💡 Hint</summary>

`kubectl port-forward` tunnels traffic from your local machine to a Pod. Syntax: `kubectl port-forward pod/<name> <local-port>:<pod-port>`

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Forward local port 9090 to pod port 80
kubectl port-forward pod/web-pod 9090:80 &
# Output: Forwarding from 127.0.0.1:9090 -> 80

# Test it
curl http://localhost:9090
# Should return nginx HTML

# Stop port-forwarding
kill %1   # or press Ctrl+C if not backgrounded
```

**Verify:**
```bash
curl -s http://localhost:9090 | grep "<title>"
# <title>Welcome to nginx!</title>
```

</details>

<details>
<summary>📖 Explanation</summary>

`kubectl port-forward` creates an encrypted tunnel from your laptop directly to a Pod. It bypasses all networking (Services, Ingress) — great for debugging individual Pods.

The `&` at the end backgrounds the process so you can keep using the terminal.

In the exam, this is used to verify that a Pod/Service is actually serving traffic before submitting your answer.

</details>

---

### Q12 · Delete Pods Efficiently

**Scenario:** You have multiple test Pods to clean up:
- `test-1`, `test-2`, `test-3` (created individually)
- All Pods with label `env=test`

**Setup:**
```bash
kubectl run test-1 --image=nginx:1.25 --labels="env=test"
kubectl run test-2 --image=nginx:1.25 --labels="env=test"
kubectl run test-3 --image=nginx:1.25 --labels="env=test"
kubectl run keep-me --image=nginx:1.25 --labels="env=prod"
```

**Task:** Delete `test-1`, `test-2`, `test-3` using a label selector. Verify `keep-me` is untouched.

---

<details>
<summary>💡 Hint</summary>

Use `-l` (label selector) flag with `kubectl delete`. You can also delete multiple pods by name in one command.

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Option 1: Delete by label selector
kubectl delete pod -l env=test

# Option 2: Delete by name (multiple at once)
kubectl delete pod test-1 test-2 test-3

# Option 3: Force delete (faster in exam)
kubectl delete pod -l env=test --grace-period=0 --force
```

**Verify:**
```bash
kubectl get pods
# Only 'keep-me' should remain
# NAME      READY   STATUS    RESTARTS   AGE
# keep-me   1/1     Running   0          1m
```

</details>

<details>
<summary>📖 Explanation</summary>

Label selectors are the Kubernetes way to work with groups of resources. `-l key=value` applies to almost every `kubectl` command (`get`, `delete`, `describe`, `logs`).

`--grace-period=0 --force` skips the graceful shutdown period (default 30s). Use this in practice/exams to save time — but avoid in production as it can cause data loss.

</details>

---

## 🔵 SECTION 2 — Namespaces (Questions 13–17)

---

### Q13 · Create Namespace and Deploy into It

**Scenario:** Your team needs an isolated environment. Create a namespace `team-alpha` and deploy a Pod `alpha-web` (nginx:1.25) inside it with label `team=alpha`.

---

<details>
<summary>💡 Hint</summary>

You can create a namespace with `kubectl create namespace` or a YAML. Always specify `--namespace` (or `-n`) when creating resources in a non-default namespace.

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Create namespace (imperative — fastest in exam)
kubectl create namespace team-alpha
```

```yaml
# alpha-web.yaml
apiVersion: v1
kind: Pod
metadata:
  name: alpha-web
  namespace: team-alpha
  labels:
    team: alpha
spec:
  containers:
    - name: web
      image: nginx:1.25
```

```bash
kubectl apply -f alpha-web.yaml
```

**Verify:**
```bash
kubectl get pod -n team-alpha
# NAME        READY   STATUS    RESTARTS   AGE
# alpha-web   1/1     Running   0          10s

kubectl get pod -n team-alpha --show-labels
# NAME        ...   LABELS
# alpha-web   ...   team=alpha
```

</details>

<details>
<summary>📖 Explanation</summary>

**Namespaces** are virtual clusters within a physical Kubernetes cluster. Think of them as separate folders on your computer. Resources in different namespaces are isolated by default.

Without `-n <namespace>`, `kubectl` commands default to the `default` namespace. In the exam, always double-check which namespace a question asks for — it's the most common mistake.

</details>

---

### Q14 · List All Resources Across Namespaces

**Scenario:** Your manager wants a report of all running Pods in the entire cluster. Show all Pods across all namespaces.

**Setup:**
```bash
kubectl run ns-test --image=nginx:1.25 -n kube-system 2>/dev/null || true
```

---

<details>
<summary>💡 Hint</summary>

Use `--all-namespaces` or its shorthand `-A` flag.

</details>

<details>
<summary>✅ Solution</summary>

```bash
# List all pods across all namespaces
kubectl get pods --all-namespaces
# OR shorthand:
kubectl get pods -A

# Show more details
kubectl get pods -A -o wide
```

**Verify:**
```bash
kubectl get pods -A | grep Running | wc -l
# Count of all Running pods across the cluster
```

</details>

<details>
<summary>📖 Explanation</summary>

`-A` / `--all-namespaces` is a flag available on most `kubectl get` commands. It overrides whatever namespace is currently set in your context.

The output includes a `NAMESPACE` column so you know where each resource lives. This is essential for cluster-wide auditing and is frequently needed in CKAD exam tasks.

</details>

---

### Q15 · Set Default Namespace for Your Session

**Scenario:** You'll be working exclusively in the `team-alpha` namespace for the next 30 minutes. Set it as your default to avoid typing `-n team-alpha` every time.

---

<details>
<summary>💡 Hint</summary>

Use `kubectl config set-context --current --namespace=<ns>`. This is the #1 time-saving trick in the CKAD exam.

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Set default namespace for current context
kubectl config set-context --current --namespace=team-alpha

# Verify the change
kubectl config view --minify | grep namespace
# namespace: team-alpha

# Now all kubectl commands default to team-alpha
kubectl get pods
# Shows pods in team-alpha without -n flag

# Reset to default when done
kubectl config set-context --current --namespace=default
```

**Verify:**
```bash
kubectl config view --minify --output 'jsonpath={..namespace}'
# team-alpha
```

</details>

<details>
<summary>📖 Explanation</summary>

This is one of the **most important exam shortcuts**. In the CKAD exam, you'll switch between namespaces frequently. Setting the context saves you from adding `-n <namespace>` to every single command.

**Important:** Always reset back to `default` when moving to the next question, or you might accidentally create resources in the wrong namespace.

</details>

---

### Q16 · Copy Resources Between Namespaces

**Scenario:** There is a Pod `template-pod` in the `default` namespace. You need to create an identical Pod in the `team-alpha` namespace (same spec, different namespace).

**Setup:**
```bash
kubectl run template-pod --image=redis:7 --labels="app=cache,tier=backend"
```

---

<details>
<summary>💡 Hint</summary>

Export the Pod's YAML with `-o yaml`, modify the namespace, strip auto-generated fields, then apply.

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Export the YAML, strip server-added fields, change namespace
kubectl get pod template-pod -o yaml \
  | grep -v "creationTimestamp\|resourceVersion\|uid\|selfLink\|generation" \
  | sed 's/namespace: default/namespace: team-alpha/' \
  > template-pod-alpha.yaml

# Apply to team-alpha
kubectl apply -f template-pod-alpha.yaml
```

**Or the simpler exam approach:**
```bash
# Get the spec and recreate
kubectl get pod template-pod -o yaml --export 2>/dev/null \
  || kubectl run template-pod -n team-alpha --image=redis:7 --labels="app=cache,tier=backend"
```

**Verify:**
```bash
kubectl get pod template-pod -n team-alpha
# NAME           READY   STATUS    RESTARTS   AGE
# template-pod   1/1     Running   0          10s
```

</details>

<details>
<summary>📖 Explanation</summary>

Exporting and modifying YAML is a common exam pattern. When you export a live Pod's YAML, it contains runtime fields (`uid`, `resourceVersion`, `creationTimestamp`) that must be removed before applying elsewhere — Kubernetes will reject them.

In the exam, the fastest approach is often just recreating from scratch using `kubectl run` with the right flags, rather than trying to export and clean up.

</details>

---

### Q17 · Namespace Resource Quota Check

**Scenario:** Find out the resource quotas applied to the `kube-system` namespace and describe what they mean.

---

<details>
<summary>💡 Hint</summary>

Use `kubectl describe namespace kube-system` or `kubectl get resourcequota -n kube-system`.

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Check namespace details
kubectl describe namespace kube-system

# Check resource quotas specifically
kubectl get resourcequota -n kube-system
kubectl describe resourcequota -n kube-system

# Check all resource usage in a namespace
kubectl describe namespace default
```

**Verify:**
```bash
kubectl get namespace kube-system -o yaml | grep -A 5 "status"
```

</details>

<details>
<summary>📖 Explanation</summary>

**ResourceQuota** limits the total resources that can be consumed in a namespace. For example:
- `pods: 10` → max 10 Pods in this namespace
- `requests.cpu: 4` → total CPU requests across all Pods ≤ 4 cores
- `limits.memory: 8Gi` → total memory limits ≤ 8Gi

If a ResourceQuota is set and you try to create a Pod without resource requests/limits, the Pod creation will be **rejected**. This is a common surprise in exam environments.

</details>

---

## 🟡 SECTION 3 — Labels & Selectors (Questions 18–21)

---

### Q18 · Add, Modify, and Remove Labels

**Scenario:** You have a running Pod `app-pod`. Perform these label operations:
1. Add label `version=v1`
2. Change it to `version=v2`
3. Remove the label entirely

**Setup:**
```bash
kubectl run app-pod --image=nginx:1.25 --labels="app=web"
```

---

<details>
<summary>💡 Hint</summary>

Use `kubectl label pod <name> key=value` to add/change. Use `kubectl label pod <name> key-` (with a dash at the end) to remove.

</details>

<details>
<summary>✅ Solution</summary>

```bash
# 1. Add label
kubectl label pod app-pod version=v1

# Verify
kubectl get pod app-pod --show-labels

# 2. Change label (use --overwrite)
kubectl label pod app-pod version=v2 --overwrite

# Verify
kubectl get pod app-pod --show-labels

# 3. Remove label (trailing dash)
kubectl label pod app-pod version-

# Verify
kubectl get pod app-pod --show-labels
# Only 'app=web' should remain
```

**Verify:**
```bash
kubectl get pod app-pod --show-labels
# NAME      READY   STATUS    ...   LABELS
# app-pod   1/1     Running   ...   app=web
```

</details>

<details>
<summary>📖 Explanation</summary>

Labels in Kubernetes are the primary mechanism for organizing resources. The trailing `-` syntax for removal is easy to forget — it's a kubectl-specific convention.

`--overwrite` is required when changing an existing label's value (to prevent accidental overwrites in production).

Labels are critical for Services and Deployments — they use **label selectors** to know which Pods to manage/route traffic to.

</details>

---

### Q19 · Filter Resources by Label Selectors

**Scenario:** You have a mixed fleet of Pods. Filter them using different selector types.

**Setup:**
```bash
kubectl run web-v1 --image=nginx:1.25 --labels="app=web,version=v1,env=prod"
kubectl run web-v2 --image=nginx:1.25 --labels="app=web,version=v2,env=prod"
kubectl run api-v1 --image=nginx:1.25 --labels="app=api,version=v1,env=dev"
kubectl run api-v2 --image=nginx:1.25 --labels="app=api,version=v2,env=dev"
```

**Tasks:**
1. Get all `app=web` Pods
2. Get all `env=prod` Pods
3. Get Pods that are `app=web` AND `version=v1`
4. Get Pods that are NOT `env=prod`

---

<details>
<summary>💡 Hint</summary>

Multiple label conditions are comma-separated. Use `!=` for "not equal". `kubectl get pods -l key!=value`.

</details>

<details>
<summary>✅ Solution</summary>

```bash
# 1. All web app pods
kubectl get pods -l app=web
# web-v1, web-v2

# 2. All production pods
kubectl get pods -l env=prod
# web-v1, web-v2

# 3. Multiple conditions (AND)
kubectl get pods -l app=web,version=v1
# web-v1 only

# 4. Inequality selector
kubectl get pods -l env!=prod
# api-v1, api-v2
```

**Verify:**
```bash
kubectl get pods -l app=web,version=v1 --show-labels
# NAME     READY   STATUS    ...   LABELS
# web-v1   1/1     Running   ...   app=web,env=prod,version=v1
```

</details>

<details>
<summary>📖 Explanation</summary>

Label selectors come in two forms:
1. **Equality-based**: `key=value`, `key!=value` — used in `kubectl` commands
2. **Set-based**: `key in (v1,v2)`, `key notin (v1,v2)` — used in YAML manifests

Comma = AND. There is no OR operator in simple selectors.

In Services and Deployments, selectors work the same way — they match Pods by label to decide which ones to manage.

</details>

---

### Q20 · Annotate Resources

**Scenario:** Your CI/CD pipeline requires annotations on Pods to track deployment metadata. Add these annotations to `app-pod`:
- `deployed-by=ci-pipeline`
- `build-number=1042`
- `git-commit=abc1234`

**Setup:**
```bash
kubectl run app-pod --image=nginx:1.25 2>/dev/null || true
```

---

<details>
<summary>💡 Hint</summary>

Annotations are like labels but for non-identifying metadata. Use `kubectl annotate pod <name> key=value`.

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Add annotations
kubectl annotate pod app-pod \
  deployed-by=ci-pipeline \
  build-number=1042 \
  git-commit=abc1234

# View annotations
kubectl describe pod app-pod | grep -A 5 "Annotations"
```

**Or in YAML:**
```yaml
metadata:
  name: app-pod
  annotations:
    deployed-by: ci-pipeline
    build-number: "1042"
    git-commit: abc1234
```

**Verify:**
```bash
kubectl get pod app-pod -o jsonpath='{.metadata.annotations}'
# map[deployed-by:ci-pipeline build-number:1042 git-commit:abc1234 ...]
```

</details>

<details>
<summary>📖 Explanation</summary>

**Labels vs Annotations:**

| | Labels | Annotations |
|---|---|---|
| Purpose | Identify & select resources | Store metadata |
| Used in selectors | ✅ Yes | ❌ No |
| Value constraints | Alphanumeric, short | Any string, including long JSON |
| Example use | `app=web`, `env=prod` | `git-commit=abc123`, build info |

Annotations can hold arbitrary data like JSON, URLs, or long descriptions. They are NOT used for selection, but tools like Helm and monitoring systems use them extensively.

</details>

---

### Q21 · Label Nodes and Use Node Affinity

**Scenario:** Label a node with `disk=ssd` and create a Pod that prefers (but doesn't require) SSD nodes.

**Setup:**
```bash
kubectl label node minikube disk=ssd
```

---

<details>
<summary>💡 Hint</summary>

Use `spec.affinity.nodeAffinity` with `preferredDuringSchedulingIgnoredDuringExecution` for a soft preference (vs. `requiredDuring...` for a hard requirement).

</details>

<details>
<summary>✅ Solution</summary>

```yaml
# prefer-ssd.yaml
apiVersion: v1
kind: Pod
metadata:
  name: prefer-ssd
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 1
          preference:
            matchExpressions:
              - key: disk
                operator: In
                values:
                  - ssd
  containers:
    - name: app
      image: nginx:1.25
```

```bash
kubectl apply -f prefer-ssd.yaml
```

**Verify:**
```bash
kubectl get pod prefer-ssd -o wide
# NODE column shows which node it landed on

kubectl describe pod prefer-ssd | grep -A 10 "Node-Affinity"
```

</details>

<details>
<summary>📖 Explanation</summary>

**nodeAffinity** is a more expressive version of `nodeSelector`:
- `required...` → Pod will NOT schedule if no matching node (hard rule)
- `preferred...` → Pod prefers matching nodes but can go elsewhere (soft rule)

The `weight` (1-100) controls how much the scheduler favors matching nodes when multiple preferred rules exist. Higher weight = stronger preference.

This is more powerful than `nodeSelector` because it supports `In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt` operators.

</details>

---

## 🟠 SECTION 4 — Deployments & ReplicaSets (Questions 22–29)

---

### Q22 · Create a Deployment

**Scenario:** Create a Deployment named `webapp` with 3 replicas of `nginx:1.25`. Label it `app=webapp, tier=frontend`.

---

<details>
<summary>💡 Hint</summary>

Use `kubectl create deployment` or write a YAML. The minimum required fields for a Deployment are: `apiVersion`, `kind`, `metadata`, `spec.replicas`, `spec.selector`, and `spec.template`.

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Imperative (fastest in exam)
kubectl create deployment webapp --image=nginx:1.25 --replicas=3
kubectl label deployment webapp tier=frontend
```

```yaml
# Or declarative:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  labels:
    app: webapp
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
        tier: frontend
    spec:
      containers:
        - name: web
          image: nginx:1.25
```

```bash
kubectl apply -f webapp-deployment.yaml
```

**Verify:**
```bash
kubectl get deployment webapp
# NAME     READY   UP-TO-DATE   AVAILABLE   AGE
# webapp   3/3     3            3           30s

kubectl get pods -l app=webapp
# Shows 3 running pods
```

</details>

<details>
<summary>📖 Explanation</summary>

A **Deployment** manages a ReplicaSet, which manages Pods. The hierarchy is:

```
Deployment → ReplicaSet → Pod(s)
```

The `selector.matchLabels` MUST match `template.metadata.labels` — this is how the Deployment knows which Pods it owns. A mismatch causes the Deployment to create Pods it can never find, which is a common error in the exam.

</details>

---

### Q23 · Scale a Deployment

**Scenario:** Traffic has increased. Scale the `webapp` Deployment from 3 to 6 replicas. Then scale it back to 2.

**Setup:** Requires Q22's Deployment to exist.

---

<details>
<summary>💡 Hint</summary>

Use `kubectl scale deployment <name> --replicas=N`. This is faster than editing the YAML.

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Scale up to 6
kubectl scale deployment webapp --replicas=6

# Watch the scaling happen
kubectl get pods -l app=webapp -w

# Scale down to 2
kubectl scale deployment webapp --replicas=2
```

**Verify:**
```bash
kubectl get deployment webapp
# NAME     READY   UP-TO-DATE   AVAILABLE   AGE
# webapp   2/2     2            2           5m
```

</details>

<details>
<summary>📖 Explanation</summary>

`kubectl scale` directly modifies the `spec.replicas` field. Kubernetes will:
- **Scale up**: create new Pods to reach the desired count
- **Scale down**: terminate excess Pods (gracefully, with a 30s window by default)

The `kubectl get pods -w` (`-w` = watch) streams real-time updates without refreshing — great for watching scaling happen.

</details>

---

### Q24 · Update a Deployment (Rolling Update)

**Scenario:** A new version of your app is ready. Update the `webapp` Deployment's image from `nginx:1.25` to `nginx:1.26`. Watch the rolling update happen.

---

<details>
<summary>💡 Hint</summary>

Use `kubectl set image deployment/<name> <container-name>=<new-image>`. The container name is what you defined in the YAML spec (e.g., `web`).

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Update the image
kubectl set image deployment/webapp web=nginx:1.26

# Watch the rolling update
kubectl rollout status deployment/webapp

# Check what happened
kubectl describe deployment webapp | grep Image
```

**Verify:**
```bash
kubectl rollout status deployment/webapp
# Waiting for deployment "webapp" rollout to finish: 1 out of 2 new replicas have been updated...
# deployment "webapp" successfully rolled out

kubectl get pods -l app=webapp -o jsonpath='{range .items[*]}{.spec.containers[0].image}{"\n"}{end}'
# nginx:1.26
# nginx:1.26
```

</details>

<details>
<summary>📖 Explanation</summary>

**Rolling Update** is the default Deployment strategy. It:
1. Creates new Pods with the new image
2. Waits for them to become Ready
3. Terminates old Pods

This achieves **zero downtime** — there's always at least some Pods serving traffic.

The key here: the container name (`web`) must match exactly what's in the Deployment spec. Use `kubectl describe deployment webapp` to find the container name if you're unsure.

</details>

---

### Q25 · Rollback a Deployment

**Scenario:** The nginx:1.26 update introduced a bug! Roll back the `webapp` Deployment to its previous version immediately.

---

<details>
<summary>💡 Hint</summary>

Use `kubectl rollout undo deployment/<name>`. To go back to a specific revision, add `--to-revision=N`.

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Check rollout history first
kubectl rollout history deployment/webapp

# Undo the last rollout (go to previous version)
kubectl rollout undo deployment/webapp

# Watch the rollback
kubectl rollout status deployment/webapp

# Verify the image reverted
kubectl describe deployment webapp | grep Image
```

**Verify:**
```bash
kubectl get pods -l app=webapp -o jsonpath='{range .items[*]}{.spec.containers[0].image}{"\n"}{end}'
# nginx:1.25
# nginx:1.25
```

</details>

<details>
<summary>📖 Explanation</summary>

Kubernetes keeps a **revision history** of your Deployments (default: last 10 revisions). Each image change creates a new revision.

`kubectl rollout undo` goes to the previous revision. `--to-revision=2` jumps to a specific one.

To see revision details: `kubectl rollout history deployment/webapp --revision=2`

This is a critical exam skill — rollbacks are common in incident response scenarios.

</details>

---

### Q26 · Deployment Rollout Pause & Resume

**Scenario:** You're deploying an update but want to do it in a controlled, paused manner. Pause the `webapp` rollout after updating the image, then resume it.

---

<details>
<summary>💡 Hint</summary>

`kubectl rollout pause deployment/<name>` and `kubectl rollout resume deployment/<name>` — useful for canary-style deployments.

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Pause the deployment before making changes
kubectl rollout pause deployment/webapp

# Make changes (they won't apply yet)
kubectl set image deployment/webapp web=nginx:1.27

# Nothing is rolling out yet — verify
kubectl rollout status deployment/webapp
# Waiting for rollout to finish... (paused)

# Resume to apply the changes
kubectl rollout resume deployment/webapp

# Watch it roll out
kubectl rollout status deployment/webapp
```

**Verify:**
```bash
kubectl rollout history deployment/webapp
# Shows revision history with the update
```

</details>

<details>
<summary>📖 Explanation</summary>

Pausing a Deployment lets you make multiple changes (image, resources, env vars) and apply them all at once in a single rollout — preventing multiple intermediate states.

Think of it like pausing a recipe midway to add multiple ingredients before resuming cooking.

This is less commonly tested in CKAD but knowing it shows deep understanding of Deployments.

</details>

---

### Q27 · ReplicaSet Behavior

**Scenario:** Manually delete one Pod from the `webapp` Deployment. What happens? Explain the self-healing behavior.

---

<details>
<summary>💡 Hint</summary>

Get a Pod name from the Deployment, delete it, then watch what happens to the total count.

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Get a pod name from the deployment
POD_NAME=$(kubectl get pods -l app=webapp -o jsonpath='{.items[0].metadata.name}')
echo $POD_NAME

# Delete it
kubectl delete pod $POD_NAME

# Watch what happens
kubectl get pods -l app=webapp -w
# New pod appears almost immediately!
```

**Verify:**
```bash
kubectl get deployment webapp
# READY should still show 2/2 (or whatever your replica count is)
# Kubernetes recreated the pod automatically
```

</details>

<details>
<summary>📖 Explanation</summary>

This demonstrates **self-healing** — one of Kubernetes' core powers.

The flow is:
1. ReplicaSet continuously watches the cluster
2. It detects: "I should have 2 Pods, but I only see 1"
3. It immediately creates a new Pod to restore the desired state

This is the fundamental principle of Kubernetes: you declare **desired state**, and the control plane constantly works to match it. You never need to manually restart crashed Pods.

</details>

---

### Q28 · Deployment Strategy — Recreate

**Scenario:** Create a Deployment `batch-app` (image: `nginx:1.25`, 2 replicas) with the `Recreate` strategy, then update the image and observe the difference from a Rolling Update.

---

<details>
<summary>💡 Hint</summary>

Set `spec.strategy.type: Recreate` in the Deployment YAML. Note: there is NO `rollingUpdate` section when using Recreate.

</details>

<details>
<summary>✅ Solution</summary>

```yaml
# batch-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: batch-app
spec:
  replicas: 2
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: batch-app
  template:
    metadata:
      labels:
        app: batch-app
    spec:
      containers:
        - name: app
          image: nginx:1.25
```

```bash
kubectl apply -f batch-app.yaml

# Now update the image and watch
kubectl set image deployment/batch-app app=nginx:1.26

# Watch closely — ALL old pods terminate FIRST, then new ones start
kubectl get pods -l app=batch-app -w
```

**Verify:**
```bash
kubectl describe deployment batch-app | grep "Strategy"
# Strategy: Recreate
```

</details>

<details>
<summary>📖 Explanation</summary>

**Recreate Strategy:**
- Terminates ALL old Pods first
- Then creates new Pods
- Causes **downtime** during the update

**When to use Recreate:**
- Apps that can't run two versions simultaneously (e.g., database migrations, stateful apps)
- When old and new versions conflict (shared ports, exclusive file locks)

**Rolling Update (default):**
- Gradual replacement, no downtime
- Best for stateless web applications

</details>

---

### Q29 · Inspect Deployment Events and ReplicaSet

**Scenario:** For the `webapp` Deployment, find:
1. Which ReplicaSet is currently active
2. The history of ReplicaSets
3. The events that occurred during the last update

---

<details>
<summary>💡 Hint</summary>

Use `kubectl get replicaset -l app=webapp` and `kubectl describe deployment webapp`.

</details>

<details>
<summary>✅ Solution</summary>

```bash
# List all ReplicaSets for this deployment
kubectl get replicaset -l app=webapp

# The active RS has DESIRED=READY matching your replica count
# Old RSes have DESIRED=0, CURRENT=0

# Detailed deployment info
kubectl describe deployment webapp

# Check events
kubectl get events --field-selector involvedObject.name=webapp --sort-by='.lastTimestamp'
```

**Verify:**
```bash
kubectl get rs -l app=webapp
# NAME                DESIRED   CURRENT   READY   AGE
# webapp-6d4cf56db6   2         2         2       10m   ← active
# webapp-7f8b9c4d5e   0         0         0       15m   ← old (kept for rollback)
```

</details>

<details>
<summary>📖 Explanation</summary>

When you update a Deployment, it creates a **new ReplicaSet** and gradually scales it up while scaling the old one down. Old ReplicaSets are kept (with 0 replicas) to enable rollbacks.

The active ReplicaSet is the one with `DESIRED > 0`. All others are historical revisions.

`kubectl get events` is invaluable for debugging — it shows you exactly what Kubernetes has been doing with your resources.

</details>

---

## 🔴 SECTION 5 — Services (Questions 30–35)

---

### Q30 · Create a ClusterIP Service

**Scenario:** Expose the `webapp` Deployment internally within the cluster on port `80`. Other Pods should reach it using the service name `webapp-svc`.

---

<details>
<summary>💡 Hint</summary>

`kubectl expose deployment <name> --port=<port> --name=<svc-name>`. ClusterIP is the default type.

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Imperative
kubectl expose deployment webapp --port=80 --name=webapp-svc --target-port=80

# Or with YAML
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: webapp-svc
spec:
  selector:
    app: webapp
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
EOF
```

**Verify:**
```bash
kubectl get service webapp-svc
# NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
# webapp-svc   ClusterIP   10.96.xxx.xxx   <none>        80/TCP    10s

# Test connectivity from another pod
kubectl run test-curl --image=curlimages/curl --rm -it --restart=Never \
  -- curl http://webapp-svc
# Should return nginx HTML
```

</details>

<details>
<summary>📖 Explanation</summary>

A **ClusterIP Service** gives your Deployment a stable internal IP address and DNS name.

Without a Service, Pods have individual IPs that change every time a Pod restarts. The Service provides:
- **Stable IP** (ClusterIP) — doesn't change
- **DNS name** — `webapp-svc.default.svc.cluster.local` (or just `webapp-svc` within the same namespace)
- **Load balancing** — distributes traffic across all matching Pods

The `selector` (`app: webapp`) tells the Service which Pods to route traffic to — it MUST match the Pod labels.

</details>

---

### Q31 · Create a NodePort Service

**Scenario:** You need to access the `webapp` Deployment from outside the cluster (from your laptop). Create a NodePort Service named `webapp-nodeport` on port `30080`.

---

<details>
<summary>💡 Hint</summary>

NodePort exposes a port on every node. The port must be in range `30000–32767`. Use `--type=NodePort` in the imperative command.

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Imperative with specific nodePort
kubectl expose deployment webapp --type=NodePort --port=80 \
  --name=webapp-nodeport
# Note: You can't set specific nodePort imperatively — use YAML for that
```

```yaml
# Or with YAML (to control the nodePort number)
apiVersion: v1
kind: Service
metadata:
  name: webapp-nodeport
spec:
  selector:
    app: webapp
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
  type: NodePort
```

```bash
kubectl apply -f webapp-nodeport.yaml
```

**Verify:**
```bash
kubectl get service webapp-nodeport
# NAME              TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
# webapp-nodeport   NodePort   10.96.yyy.yyy  <none>        80:30080/TCP   10s

# Access via minikube
minikube service webapp-nodeport --url
curl $(minikube service webapp-nodeport --url)
```

</details>

<details>
<summary>📖 Explanation</summary>

**Service types comparison:**

| Type | Access From | Use Case |
|---|---|---|
| `ClusterIP` | Inside cluster only | Internal microservices |
| `NodePort` | Outside cluster via Node IP | Dev/testing, direct access |
| `LoadBalancer` | Outside via cloud LB | Production (cloud environments) |

`NodePort` opens port `30080` on EVERY node. Traffic flow: `Your laptop:30080 → Node:30080 → Service:80 → Pod:80`

</details>

---

### Q32 · Service Selector Debugging

**Scenario:** The Service `broken-svc` exists but no traffic reaches the Pods. Find and fix the problem.

**Setup:**
```bash
kubectl run target-pod --image=nginx:1.25 --labels="app=backend,version=v1"
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: broken-svc
spec:
  selector:
    app: frontend   # ← BUG: wrong selector
  ports:
    - port: 80
      targetPort: 80
EOF
```

---

<details>
<summary>💡 Hint</summary>

Check if the Service has any Endpoints. `kubectl get endpoints broken-svc` — if it shows `<none>`, the selector doesn't match any Pods.

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Diagnose
kubectl get endpoints broken-svc
# NAME         ENDPOINTS   AGE
# broken-svc   <none>      10s  ← Problem! No endpoints

# Check the service selector
kubectl describe service broken-svc | grep "Selector"
# Selector: app=frontend

# Check the pod labels
kubectl get pod target-pod --show-labels
# Labels: app=backend,version=v1

# Fix: update the selector
kubectl patch service broken-svc -p '{"spec":{"selector":{"app":"backend"}}}'

# Verify fix
kubectl get endpoints broken-svc
# NAME         ENDPOINTS        AGE
# broken-svc   10.244.0.5:80   10s  ← Now has endpoints!
```

</details>

<details>
<summary>📖 Explanation</summary>

**"Service has no endpoints"** is the #1 Service problem in Kubernetes.

Debugging checklist:
1. `kubectl get endpoints <svc>` → if `<none>`, selector is broken
2. `kubectl describe service <svc>` → check the Selector field
3. `kubectl get pods --show-labels` → check what labels the Pods actually have
4. Compare the two — they must match exactly

The Service uses its `selector` to dynamically build the Endpoints list. Any mismatch = no traffic routing.

</details>

---

### Q33 · DNS Resolution Between Services

**Scenario:** Prove that Pod-to-Service DNS works. From a test Pod in the `default` namespace, resolve the `webapp-svc` service's IP using DNS.

---

<details>
<summary>💡 Hint</summary>

Kubernetes uses CoreDNS. Services get DNS names in the format `<service-name>.<namespace>.svc.cluster.local`. Within the same namespace, just the service name works.

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Launch a debug pod and test DNS
kubectl run dns-test --image=busybox:1.36 --rm -it --restart=Never \
  -- sh -c "nslookup webapp-svc && wget -qO- http://webapp-svc"

# Or test with curl
kubectl run dns-test --image=curlimages/curl --rm -it --restart=Never \
  -- curl -s http://webapp-svc

# Full DNS name (same namespace)
# webapp-svc.default.svc.cluster.local
```

**Verify:**
```bash
kubectl run dns-test --image=busybox:1.36 --rm -it --restart=Never \
  -- nslookup webapp-svc
# Server:    10.96.0.10
# Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
# 
# Name:      webapp-svc
# Address 1: 10.96.xxx.xxx webapp-svc.default.svc.cluster.local
```

</details>

<details>
<summary>📖 Explanation</summary>

Kubernetes runs **CoreDNS** as a Pod in `kube-system` namespace. It provides automatic DNS for all Services.

DNS format: `<service>.<namespace>.svc.cluster.local`

Short forms that also work (within same namespace):
- `webapp-svc` ← simplest
- `webapp-svc.default`
- `webapp-svc.default.svc`
- `webapp-svc.default.svc.cluster.local`

From a different namespace, you must use at least `webapp-svc.default`.

</details>

---

### Q34 · Headless Service

**Scenario:** Create a "headless" Service for a database Pod where you need direct Pod IP addresses instead of a load-balanced VIP. Pod: `db-pod` (image: `redis:7`, label: `app=db`).

---

<details>
<summary>💡 Hint</summary>

Set `spec.clusterIP: None` to create a headless Service. DNS returns individual Pod IPs instead of a single VIP.

</details>

<details>
<summary>✅ Solution</summary>

```bash
kubectl run db-pod --image=redis:7 --labels="app=db"
```

```yaml
# headless-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: db-headless
spec:
  clusterIP: None    # ← This makes it headless
  selector:
    app: db
  ports:
    - port: 6379
      targetPort: 6379
```

```bash
kubectl apply -f headless-svc.yaml
```

**Verify:**
```bash
kubectl get service db-headless
# NAME          TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
# db-headless   ClusterIP   None         <none>        6379/TCP   10s
#                           ^^^^ No IP assigned

# DNS returns Pod IP directly
kubectl run dns-test --image=busybox:1.36 --rm -it --restart=Never \
  -- nslookup db-headless
# Returns the actual Pod IP, not a VIP
```

</details>

<details>
<summary>📖 Explanation</summary>

A **Headless Service** (`clusterIP: None`) doesn't get a virtual IP. Instead, DNS lookups return the actual Pod IPs.

This is used for:
- **StatefulSets** — each Pod gets its own DNS name (`pod-0.svc-name`)
- **Databases and clusters** — apps that need to connect to specific instances
- **Service discovery** — when your app handles its own load balancing

</details>

---

### Q35 · ExternalName Service

**Scenario:** Your app needs to connect to an external database at `db.company.com`. Create a Service that aliases this external hostname inside the cluster as `external-db`.

---

<details>
<summary>💡 Hint</summary>

Use `spec.type: ExternalName` and `spec.externalName: <hostname>`. No selector needed.

</details>

<details>
<summary>✅ Solution</summary>

```yaml
# external-db-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: db.company.com
  # No selector — this just creates a CNAME DNS alias
```

```bash
kubectl apply -f external-db-svc.yaml
```

**Verify:**
```bash
kubectl get service external-db
# NAME          TYPE           CLUSTER-IP   EXTERNAL-IP      PORT(S)   AGE
# external-db   ExternalName   <none>       db.company.com   <none>    10s

# Inside a Pod, DNS resolves 'external-db' to db.company.com
kubectl run dns-test --image=busybox:1.36 --rm -it --restart=Never \
  -- nslookup external-db
```

</details>

<details>
<summary>📖 Explanation</summary>

`ExternalName` Services create a **DNS CNAME alias** inside your cluster. No proxying happens — it's pure DNS.

Use case: Your app connects to `external-db:5432` (thinks it's in-cluster), but Kubernetes DNS resolves it to `db.company.com:5432`. Later, if you migrate the database into the cluster, you just change the Service type — no app code changes needed.

</details>

---

## 🟣 SECTION 6 — ConfigMaps & Secrets (Questions 36–44)

---

### Q36 · Create a ConfigMap — Three Ways

**Scenario:** Create a ConfigMap named `app-config` with:
- `APP_COLOR=blue`
- `APP_MODE=debug`
- `MAX_CONNECTIONS=100`

Do this using all three methods: literal, file, and YAML.

---

<details>
<summary>💡 Hint</summary>

`--from-literal`, `--from-file`, or a YAML manifest with `data:` section. Know all three — the exam can ask for any.

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Method 1: Literal values (fastest in exam)
kubectl create configmap app-config \
  --from-literal=APP_COLOR=blue \
  --from-literal=APP_MODE=debug \
  --from-literal=MAX_CONNECTIONS=100
```

```bash
# Method 2: From a file
cat > app.properties <<EOF
APP_COLOR=blue
APP_MODE=debug
MAX_CONNECTIONS=100
EOF
kubectl create configmap app-config --from-file=app.properties
```

```yaml
# Method 3: YAML manifest
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_COLOR: blue
  APP_MODE: debug
  MAX_CONNECTIONS: "100"
```

**Verify:**
```bash
kubectl get configmap app-config -o yaml
kubectl describe configmap app-config
```

</details>

<details>
<summary>📖 Explanation</summary>

ConfigMaps store **non-sensitive** configuration data as key-value pairs. Think of them as a configuration file that lives in Kubernetes.

The three creation methods suit different use cases:
- `--from-literal` → Quick, simple key-value pairs
- `--from-file` → Entire config files (nginx.conf, app.properties)
- YAML manifest → Version-controlled, declarative approach

Remember: ConfigMap values are always **strings**. Numbers like `100` need quotes in YAML: `"100"`.

</details>

---

### Q37 · Inject ConfigMap as Environment Variables

**Scenario:** Create a Pod `web-configured` that reads ALL keys from `app-config` as environment variables.

**Setup:** Requires Q36's ConfigMap.

---

<details>
<summary>💡 Hint</summary>

Use `envFrom` with `configMapRef` to inject all ConfigMap keys at once. This is different from `env` (which injects individual keys).

</details>

<details>
<summary>✅ Solution</summary>

```yaml
# web-configured.yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-configured
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "env | grep -E 'APP_|MAX_' && sleep 3600"]
      envFrom:
        - configMapRef:
            name: app-config
```

```bash
kubectl apply -f web-configured.yaml
```

**Verify:**
```bash
kubectl logs web-configured
# APP_COLOR=blue
# APP_MODE=debug
# MAX_CONNECTIONS=100
```

</details>

<details>
<summary>📖 Explanation</summary>

Two ways to inject ConfigMap data as env vars:

**`envFrom` (all keys):**
```yaml
envFrom:
  - configMapRef:
      name: app-config
# Injects ALL keys from the ConfigMap
```

**`env` with `valueFrom` (individual keys):**
```yaml
env:
  - name: MY_COLOR
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: APP_COLOR
# Injects just one key, with optional renaming
```

Use `envFrom` when you want all values. Use `valueFrom` when you need specific keys or want to rename them.

</details>

---

### Q38 · Mount ConfigMap as a Volume

**Scenario:** Your nginx needs a custom config file. Mount the `app-config` ConfigMap as files inside a Pod at path `/etc/config/`.

---

<details>
<summary>💡 Hint</summary>

Use `volumes` with `configMap` type and `volumeMounts` in the container. Each ConfigMap key becomes a separate file.

</details>

<details>
<summary>✅ Solution</summary>

```yaml
# configmap-volume.yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-mounted
spec:
  volumes:
    - name: config-vol
      configMap:
        name: app-config
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "ls /etc/config/ && cat /etc/config/APP_COLOR && sleep 3600"]
      volumeMounts:
        - name: config-vol
          mountPath: /etc/config
```

```bash
kubectl apply -f configmap-volume.yaml
```

**Verify:**
```bash
kubectl logs config-mounted
# APP_COLOR  APP_MODE  MAX_CONNECTIONS   ← files listed
# blue                                   ← content of APP_COLOR file

kubectl exec config-mounted -- ls /etc/config/
# APP_COLOR  APP_MODE  MAX_CONNECTIONS
```

</details>

<details>
<summary>📖 Explanation</summary>

When mounted as a volume, each ConfigMap key becomes a **file**, and the value becomes the file's content.

So `APP_COLOR: blue` → file `/etc/config/APP_COLOR` with content `blue`.

This is perfect for:
- nginx.conf, application.yaml, or other full config files
- Situations where your app reads config from files (not env vars)

When the ConfigMap is updated, the mounted files are **automatically updated** too (unlike env vars, which require a Pod restart).

</details>

---

### Q39 · Create a Secret — Opaque Type

**Scenario:** Store database credentials securely. Create a Secret named `db-creds` with:
- `username=admin`
- `password=Sup3rS3cr3t!`

---

<details>
<summary>💡 Hint</summary>

Secrets work like ConfigMaps but values are base64-encoded. Use `kubectl create secret generic` for opaque (generic) secrets.

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Imperative (values are auto-encoded)
kubectl create secret generic db-creds \
  --from-literal=username=admin \
  --from-literal=password='Sup3rS3cr3t!'
```

```yaml
# YAML way (must manually base64-encode)
# echo -n 'admin' | base64        → YWRtaW4=
# echo -n 'Sup3rS3cr3t!' | base64 → U3VwM3JTM2NyM3Qh

apiVersion: v1
kind: Secret
metadata:
  name: db-creds
type: Opaque
data:
  username: YWRtaW4=
  password: U3VwM3JTM2NyM3Qh
```

**Verify:**
```bash
kubectl get secret db-creds
# Values are hidden

kubectl describe secret db-creds
# Shows keys but NOT values

# Decode a value
kubectl get secret db-creds -o jsonpath='{.data.password}' | base64 -d
# Sup3rS3cr3t!
```

</details>

<details>
<summary>📖 Explanation</summary>

**ConfigMap vs Secret:**

| | ConfigMap | Secret |
|---|---|---|
| Purpose | Non-sensitive config | Sensitive data |
| Storage | Plain text in etcd | Base64 encoded in etcd |
| Show in `describe` | ✅ Values visible | ❌ Values hidden |
| Use for | Env vars, config files | Passwords, tokens, TLS certs |

**Important:** Base64 is NOT encryption — it's just encoding. Anyone with cluster access can decode it. For true security, use solutions like Vault or sealed-secrets in production.

`echo -n` is critical when encoding — without it, a newline character gets included in the encoded value.

</details>

---

### Q40 · Inject Secret as Environment Variables

**Scenario:** Create a Pod `db-client` that reads `username` and `password` from the `db-creds` Secret as environment variables `DB_USER` and `DB_PASS`.

---

<details>
<summary>💡 Hint</summary>

Use `secretKeyRef` instead of `configMapKeyRef`. The structure is identical — just a different source type.

</details>

<details>
<summary>✅ Solution</summary>

```yaml
# db-client.yaml
apiVersion: v1
kind: Pod
metadata:
  name: db-client
spec:
  containers:
    - name: client
      image: busybox:1.36
      command: ["sh", "-c", "echo User: $DB_USER && echo Pass: $DB_PASS && sleep 3600"]
      env:
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-creds
              key: username
        - name: DB_PASS
          valueFrom:
            secretKeyRef:
              name: db-creds
              key: password
```

```bash
kubectl apply -f db-client.yaml
```

**Verify:**
```bash
kubectl logs db-client
# User: admin
# Pass: Sup3rS3cr3t!
```

</details>

<details>
<summary>📖 Explanation</summary>

`secretKeyRef` pulls individual Secret keys. You can also use `envFrom` with `secretRef` to inject all Secret keys at once:

```yaml
envFrom:
  - secretRef:
      name: db-creds
```

The key difference from ConfigMaps: Kubernetes keeps Secret values hidden in `kubectl describe` output, but once injected as env vars, your app can read them as plain text. This is why you should be careful about logging env vars in containerized apps.

</details>

---

### Q41 · Mount Secret as a Volume

**Scenario:** Mount the `db-creds` Secret as files at `/etc/secrets/` in a Pod. Read and verify the contents.

---

<details>
<summary>💡 Hint</summary>

Same as ConfigMap volume mount, but use `secret` instead of `configMap` in the volume definition.

</details>

<details>
<summary>✅ Solution</summary>

```yaml
# secret-volume.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-mounted
spec:
  volumes:
    - name: secret-vol
      secret:
        secretName: db-creds
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "ls /etc/secrets && cat /etc/secrets/username && sleep 3600"]
      volumeMounts:
        - name: secret-vol
          mountPath: /etc/secrets
          readOnly: true    # Best practice — secrets should be read-only
```

```bash
kubectl apply -f secret-volume.yaml
```

**Verify:**
```bash
kubectl logs secret-mounted
# username  password   ← files
# admin               ← content of username file

kubectl exec secret-mounted -- cat /etc/secrets/password
# Sup3rS3cr3t!
```

</details>

<details>
<summary>📖 Explanation</summary>

Mounted Secrets have a security advantage over env vars:
- Values are stored in `tmpfs` (RAM-backed filesystem) — not on disk
- The `readOnly: true` flag prevents the container from modifying them
- Auto-updated when the Secret changes (without Pod restart)

Best practice: mount secrets as volumes rather than env vars when possible. Env vars can be accidentally exposed in logs, crash dumps, or debug output.

</details>

---

### Q42 · Secret for Docker Registry (imagePullSecrets)

**Scenario:** Your company's Docker registry at `registry.company.com` requires authentication. Create the necessary Secret and configure a Pod to use it.

---

<details>
<summary>💡 Hint</summary>

Use `kubectl create secret docker-registry`. Reference it in `spec.imagePullSecrets` in the Pod spec.

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Create docker-registry secret
kubectl create secret docker-registry registry-creds \
  --docker-server=registry.company.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=me@company.com
```

```yaml
# pod-with-pull-secret.yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-app
spec:
  imagePullSecrets:
    - name: registry-creds    # ← Reference the secret here
  containers:
    - name: app
      image: registry.company.com/myapp:v1
```

**Verify:**
```bash
kubectl get secret registry-creds
# TYPE: kubernetes.io/dockerconfigjson

kubectl describe pod private-app | grep "Image\|Pull"
```

</details>

<details>
<summary>📖 Explanation</summary>

`imagePullSecrets` tells Kubernetes which credentials to use when pulling an image from a private registry.

The Secret type `kubernetes.io/dockerconfigjson` is specifically designed for registry credentials — it's equivalent to your `~/.docker/config.json` file.

This appears in CKAD when dealing with private registries. The common error is forgetting `imagePullSecrets` entirely, which results in `ImagePullBackOff`.

</details>

---

### Q43 · Update a ConfigMap and Observe Hot Reload

**Scenario:** Update the `app-config` ConfigMap's `APP_COLOR` from `blue` to `green`. Observe the change reflected in a volume-mounted Pod without restarting it.

**Setup:** Use Pod from Q38 (`config-mounted`).

---

<details>
<summary>💡 Hint</summary>

Edit the ConfigMap with `kubectl edit configmap app-config`. Volume-mounted ConfigMaps update automatically (with a short delay). Env var injections do NOT auto-update.

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Edit the configmap
kubectl edit configmap app-config
# Change: APP_COLOR: blue → APP_COLOR: green
# Save and exit

# Or patch it directly
kubectl patch configmap app-config -p '{"data":{"APP_COLOR":"green"}}'

# Wait ~60 seconds for the volume to sync, then check
kubectl exec config-mounted -- cat /etc/config/APP_COLOR
# green  ← updated without pod restart!
```

**Verify:**
```bash
kubectl describe configmap app-config | grep APP_COLOR
# APP_COLOR:  green
```

</details>

<details>
<summary>📖 Explanation</summary>

**ConfigMap update behavior:**

| Method | Auto-updates in Pod? |
|---|---|
| Volume mount | ✅ Yes (~60 seconds delay) |
| `env` / `envFrom` | ❌ No — requires Pod restart |

This difference matters in production: volume-mounted configs support hot-reloading, which enables zero-downtime config changes for apps that watch their config files.

</details>

---

### Q44 · ConfigMap from a Multi-line Config File

**Scenario:** You have a custom nginx configuration. Create a ConfigMap `nginx-conf` that contains the entire nginx config as a single key `nginx.conf`, then mount it in a Pod at `/etc/nginx/nginx.conf`.

---

<details>
<summary>💡 Hint</summary>

Use a multi-line string in YAML with `|` (literal block scalar) for the value. Mount with a `subPath` to target a specific file.

</details>

<details>
<summary>✅ Solution</summary>

```yaml
# nginx-config-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-conf
data:
  nginx.conf: |
    events {}
    http {
      server {
        listen 80;
        location / {
          return 200 'Hello from custom nginx!\n';
          add_header Content-Type text/plain;
        }
      }
    }
```

```bash
kubectl apply -f nginx-config-cm.yaml
```

```yaml
# nginx-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: custom-nginx
spec:
  volumes:
    - name: nginx-config
      configMap:
        name: nginx-conf
  containers:
    - name: nginx
      image: nginx:1.25
      volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf    # ← Mount only this specific key as a file
```

```bash
kubectl apply -f nginx-pod.yaml
kubectl port-forward pod/custom-nginx 8080:80 &
curl http://localhost:8080
# Hello from custom nginx!
```

</details>

<details>
<summary>📖 Explanation</summary>

The `|` in YAML preserves newlines — perfect for multi-line configs.

`subPath` is critical here: without it, mounting at `/etc/nginx/nginx.conf` would replace the entire `/etc/nginx/` directory with the ConfigMap content (making nginx unable to find other required files). With `subPath: nginx.conf`, you inject just that specific file while leaving the rest of the directory intact.

</details>

---

## ⚫ SECTION 7 — Mixed / Advanced Day 1–14 (Questions 45–50)

---

### Q45 · Full Stack: Deployment + ConfigMap + Service

**Scenario:** Build a complete application stack:
1. ConfigMap `frontend-config` with `WELCOME_MSG=Hello Kubernetes`
2. Deployment `frontend` (2 replicas, `nginx:1.25`) injecting the ConfigMap
3. ClusterIP Service `frontend-svc` on port 80

---

<details>
<summary>💡 Hint</summary>

Create resources in order: ConfigMap → Deployment → Service. You can put all three in one YAML file separated by `---`.

</details>

<details>
<summary>✅ Solution</summary>

```yaml
# full-stack.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-config
data:
  WELCOME_MSG: "Hello Kubernetes"

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: web
          image: nginx:1.25
          envFrom:
            - configMapRef:
                name: frontend-config

---
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
spec:
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```

```bash
kubectl apply -f full-stack.yaml
```

**Verify:**
```bash
kubectl get deployment frontend
kubectl get service frontend-svc
kubectl get endpoints frontend-svc
# Should show 2 Pod IPs

kubectl exec -it $(kubectl get pod -l app=frontend -o jsonpath='{.items[0].metadata.name}') \
  -- env | grep WELCOME_MSG
# WELCOME_MSG=Hello Kubernetes
```

</details>

<details>
<summary>📖 Explanation</summary>

This is the most common CKAD exam pattern — creating interconnected resources. The `---` separator allows multiple resources in one file.

Always verify the Service has endpoints after creation. If `kubectl get endpoints frontend-svc` shows `<none>`, the selector is wrong.

Order of creation matters: ConfigMap must exist before the Pod that references it (otherwise the Pod fails with `CreateContainerConfigError`).

</details>

---

### Q46 · Timed Challenge: Pod with Secret in 5 Minutes

**Scenario — START TIMER:** ⏱️ 5 minutes

Create a namespace `challenge`, then within it:
1. Secret `api-key` with key `token=MyS3cretT0ken`
2. Pod `api-consumer` (image: `busybox:1.36`) that:
   - Reads the token from the Secret as env var `API_TOKEN`
   - Prints `Token: $API_TOKEN` and sleeps
3. Verify the logs show the token

---

<details>
<summary>💡 Hint</summary>

Speed tips: Use `kubectl create namespace`, `kubectl create secret generic --from-literal`, then write minimal YAML. Use `kubectl run` + patch if needed.

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Fast approach for exam
kubectl create namespace challenge

kubectl create secret generic api-key \
  --from-literal=token=MyS3cretT0ken \
  -n challenge
```

```yaml
# consumer.yaml
apiVersion: v1
kind: Pod
metadata:
  name: api-consumer
  namespace: challenge
spec:
  containers:
    - name: consumer
      image: busybox:1.36
      command: ["sh", "-c", "echo Token: $API_TOKEN && sleep 3600"]
      env:
        - name: API_TOKEN
          valueFrom:
            secretKeyRef:
              name: api-key
              key: token
```

```bash
kubectl apply -f consumer.yaml
kubectl logs api-consumer -n challenge
# Token: MyS3cretT0ken
```

**Verify:**
```bash
kubectl logs api-consumer -n challenge | grep "Token:"
# Token: MyS3cretT0ken ✅
```

</details>

<details>
<summary>📖 Explanation</summary>

**Exam time-saving pattern:**
1. Use imperative commands for simple resources (namespace, secret, configmap)
2. Use `--dry-run=client -o yaml` to generate Pod YAML, then edit
3. Always verify with `kubectl logs` or `kubectl exec` before moving on

The exam uses a similar format to this — you'll have ~15-20 tasks in 2 hours, so each task should take 5-8 minutes maximum.

</details>

---

### Q47 · Resource Limits + OOM Scenario

**Scenario:** Create a Pod `memory-hog` (image: `polinux/stress`) with a memory limit of `50Mi` that tries to allocate `100Mi` of RAM. Observe what happens.

---

<details>
<summary>💡 Hint</summary>

The `stress` tool can allocate memory: `stress --vm 1 --vm-bytes 100M`. Set the limit to `50Mi` — the container will be OOMKilled.

</details>

<details>
<summary>✅ Solution</summary>

```yaml
# memory-hog.yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-hog
spec:
  containers:
    - name: stress
      image: polinux/stress
      command: ["stress"]
      args: ["--vm", "1", "--vm-bytes", "100M", "--vm-hang", "1"]
      resources:
        limits:
          memory: "50Mi"
        requests:
          memory: "10Mi"
```

```bash
kubectl apply -f memory-hog.yaml

# Watch what happens
kubectl get pod memory-hog -w
```

**Verify:**
```bash
kubectl get pod memory-hog
# STATUS: OOMKilled  ← Out of Memory

kubectl describe pod memory-hog | grep -A 5 "Last State\|OOMKilled"
# Last State:  Terminated
#   Reason:    OOMKilled
#   Exit Code: 137
```

</details>

<details>
<summary>📖 Explanation</summary>

When a container exceeds its memory **limit**, Linux kills it with signal 9 (SIGKILL) — this is **OOMKilled** (Out Of Memory Killed).

Exit code `137` = 128 + 9 (128 + signal number) — the standard Linux way of indicating a process was killed by signal 9.

**CPU** is different: exceeding CPU limit causes **throttling** (slower, not killed).

**Identifying OOMKilled in the exam:**
1. `kubectl get pod` → STATUS: `OOMKilled` or `Error`
2. `kubectl describe pod` → look for `OOMKilled` in Last State
3. Fix: increase memory limit or optimize the app

</details>

---

### Q48 · Complete Debugging Scenario

**Scenario:** The following resources were deployed but the app isn't working. Find ALL the bugs and fix them.

**Setup:**
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: buggy-config
data:
  PORT: "3000"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: buggy-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: buggy
  template:
    metadata:
      labels:
        app: buggy-app   # BUG 1
    spec:
      containers:
        - name: web
          image: nginx:LATEST   # BUG 2
          envFrom:
            - configMapRef:
                name: wrong-config-name   # BUG 3
---
apiVersion: v1
kind: Service
metadata:
  name: buggy-svc
spec:
  selector:
    app: buggy-wrong   # BUG 4
  ports:
    - port: 80
      targetPort: 3000
EOF
```

---

<details>
<summary>💡 Hint</summary>

Check each resource: `kubectl describe deployment buggy-app`, `kubectl get endpoints buggy-svc`. Look for: selector mismatch, image pull errors, missing ConfigMap, service selector.

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Diagnose all bugs
kubectl describe deployment buggy-app   # Shows selector vs template label mismatch
kubectl get pods -l app=buggy           # No pods (selector mismatch)
kubectl get endpoints buggy-svc         # <none> (wrong selector)

# Fix Bug 1: Template label doesn't match selector
kubectl patch deployment buggy-app --type=json \
  -p='[{"op":"replace","path":"/spec/template/metadata/labels/app","value":"buggy"}]'

# Fix Bug 2: Invalid image tag
kubectl set image deployment/buggy-app web=nginx:1.25

# Fix Bug 3: Wrong ConfigMap name
kubectl patch deployment buggy-app --type=json \
  -p='[{"op":"replace","path":"/spec/template/spec/containers/0/envFrom/0/configMapRef/name","value":"buggy-config"}]'

# Fix Bug 4: Service selector
kubectl patch service buggy-svc -p '{"spec":{"selector":{"app":"buggy"}}}'
```

**Verify all fixes:**
```bash
kubectl get pods -l app=buggy          # 2 pods running
kubectl get endpoints buggy-svc        # Shows 2 pod IPs
kubectl rollout status deployment/buggy-app  # Successfully rolled out
```

</details>

<details>
<summary>📖 Explanation</summary>

**The 4 bugs and why they matter:**

1. **Selector/label mismatch** — The Deployment's `selector.matchLabels.app=buggy` couldn't find Pods with `app=buggy-app`. No Pods get managed. This is the most common Deployment mistake.

2. **Invalid image tag** — `nginx:LATEST` doesn't exist (it's `nginx:latest` in lowercase). Always check image tags.

3. **Wrong ConfigMap name** — `CreateContainerConfigError` — the ConfigMap reference must match an existing ConfigMap exactly.

4. **Service selector mismatch** — Service finds no Pods, resulting in `<none>` endpoints and no traffic routing.

**Debugging order for exam:** Pod status → describe Pod → check endpoints → check service selector.

</details>

---

### Q49 · Generate and Customize YAML with dry-run

**Scenario:** Your team needs a YAML template for a Deployment with specific requirements. Use `--dry-run=client -o yaml` to generate a base, then customize it to add:
- 3 replicas
- Resource limits (CPU: 250m, Memory: 256Mi)
- A ConfigMap env injection from `app-config`
- Label `version=v1` on the Pod template

---

<details>
<summary>💡 Hint</summary>

Generate the base YAML first, redirect to a file, then edit. This is the standard CKAD workflow for complex resources.

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Step 1: Generate base YAML
kubectl create deployment custom-app --image=nginx:1.25 --replicas=3 \
  --dry-run=client -o yaml > custom-app.yaml

# Step 2: Edit the file to add requirements
# (Open in vi/nano and add the missing sections)
```

**Final YAML after editing:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: custom-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: custom-app
  template:
    metadata:
      labels:
        app: custom-app
        version: v1              # ← Added
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          resources:             # ← Added
            requests:
              cpu: "125m"
              memory: "128Mi"
            limits:
              cpu: "250m"
              memory: "256Mi"
          envFrom:               # ← Added
            - configMapRef:
                name: app-config
```

```bash
kubectl apply -f custom-app.yaml
```

**Verify:**
```bash
kubectl describe deployment custom-app | grep -E "Replicas|Image|Limits|Requests"
kubectl get pods -l app=custom-app --show-labels | grep version=v1
```

</details>

<details>
<summary>📖 Explanation</summary>

The `--dry-run=client -o yaml` pattern is the most important CKAD exam technique:

1. Generate the base YAML (saves typing boilerplate)
2. Edit to add the specific requirements
3. Apply with `kubectl apply`

**Key shortcuts:**
```bash
# Generate Pod YAML
kubectl run mypod --image=nginx --dry-run=client -o yaml > pod.yaml

# Generate Deployment YAML
kubectl create deployment myapp --image=nginx --replicas=3 --dry-run=client -o yaml > deploy.yaml

# Generate Service YAML
kubectl expose deployment myapp --port=80 --dry-run=client -o yaml > svc.yaml
```

Memorize these — they save 2-3 minutes per question in the exam.

</details>

---

### Q50 · Final Boss — Complete CKAD-Style Task

**Scenario:** ⏱️ Target: 12 minutes

You are deploying a new microservice for a finance team. Complete ALL of the following:

1. Create namespace `finance`
2. Create Secret `db-secret` in `finance` with:
   - `DB_HOST=postgres.finance.svc`
   - `DB_PORT=5432`
   - `DB_PASS=F1n@nc3P@ss`
3. Create ConfigMap `app-settings` in `finance` with:
   - `LOG_LEVEL=info`
   - `MAX_RETRIES=3`
4. Create Deployment `finance-api` in `finance` namespace:
   - Image: `nginx:1.25`
   - 2 replicas
   - All Secret values injected as env vars
   - All ConfigMap values injected as env vars
   - CPU request: `100m`, limit: `200m`
   - Memory request: `128Mi`, limit: `256Mi`
   - Label: `app=finance-api, team=finance`
5. Create ClusterIP Service `finance-api-svc` in `finance` namespace on port 80
6. Verify all resources are running and the service has endpoints

---

<details>
<summary>💡 Hint</summary>

Work in order: namespace → secrets → configmaps → deployment → service. Use imperative commands for secrets/configmaps. Generate Deployment YAML with `--dry-run`. Always `kubectl config set-context --current --namespace=finance` first.

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Step 1: Namespace
kubectl create namespace finance
kubectl config set-context --current --namespace=finance

# Step 2: Secret
kubectl create secret generic db-secret \
  --from-literal=DB_HOST=postgres.finance.svc \
  --from-literal=DB_PORT=5432 \
  --from-literal=DB_PASS='F1n@nc3P@ss'

# Step 3: ConfigMap
kubectl create configmap app-settings \
  --from-literal=LOG_LEVEL=info \
  --from-literal=MAX_RETRIES=3

# Step 4: Generate Deployment base
kubectl create deployment finance-api --image=nginx:1.25 --replicas=2 \
  --dry-run=client -o yaml > finance-api.yaml
```

**Edit finance-api.yaml to add the requirements:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: finance-api
  namespace: finance
spec:
  replicas: 2
  selector:
    matchLabels:
      app: finance-api
  template:
    metadata:
      labels:
        app: finance-api
        team: finance
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "200m"
              memory: "256Mi"
          envFrom:
            - secretRef:
                name: db-secret
            - configMapRef:
                name: app-settings
```

```bash
kubectl apply -f finance-api.yaml

# Step 5: Service
kubectl expose deployment finance-api --port=80 --name=finance-api-svc

# Step 6: Verify everything
kubectl get all -n finance
kubectl get endpoints finance-api-svc -n finance

# Verify env vars are injected
POD=$(kubectl get pod -l app=finance-api -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD -- env | grep -E "DB_|LOG_|MAX_"

# Reset namespace
kubectl config set-context --current --namespace=default
```

**Final Verification:**
```bash
kubectl get deployment finance-api -n finance
# NAME          READY   UP-TO-DATE   AVAILABLE
# finance-api   2/2     2            2         ✅

kubectl get endpoints finance-api-svc -n finance
# NAME              ENDPOINTS                     AGE
# finance-api-svc   10.244.0.x:80,10.244.0.y:80  1m  ✅
```

</details>

<details>
<summary>📖 Explanation</summary>

This question covers everything from Days 1–14 in one task. The key skills demonstrated:

1. **Namespace isolation** — real apps live in dedicated namespaces
2. **Secrets** — sensitive data (passwords) stored securely
3. **ConfigMaps** — non-sensitive runtime configuration
4. **Deployments** — scalable, self-healing app deployment
5. **Resource management** — requests/limits for proper scheduling
6. **Services** — stable networking for your Pods

**Exam checklist before submitting any task:**
- [ ] Resources created in the right namespace
- [ ] Pod labels match Service/Deployment selectors
- [ ] Service has endpoints (not `<none>`)
- [ ] Pods are in `Running` state (not `Pending`/`Error`)
- [ ] Logs show expected output if applicable
- [ ] Resource requests/limits match the requirements

If you completed this in under 12 minutes — you're exam-ready for these topics! 🎉

</details>

---

## 📊 Question Coverage Summary

| Topic | Questions | CKAD Weight |
|---|---|---|
| Pods (creation, debug, lifecycle) | 1–12 | ⭐⭐⭐⭐⭐ |
| Namespaces | 13–17 | ⭐⭐⭐ |
| Labels & Selectors | 18–21 | ⭐⭐⭐⭐ |
| Deployments & ReplicaSets | 22–29 | ⭐⭐⭐⭐⭐ |
| Services | 30–35 | ⭐⭐⭐⭐⭐ |
| ConfigMaps & Secrets | 36–44 | ⭐⭐⭐⭐⭐ |
| Mixed / Debugging | 45–50 | ⭐⭐⭐⭐⭐ |

---

## ⚡ Essential Exam Shortcuts

```bash
# Generate YAML templates (memorize these)
kubectl run pod-name --image=nginx --dry-run=client -o yaml > pod.yaml
kubectl create deployment dep-name --image=nginx --replicas=3 --dry-run=client -o yaml > dep.yaml
kubectl expose deployment dep-name --port=80 --dry-run=client -o yaml > svc.yaml
kubectl create configmap cm-name --from-literal=key=val --dry-run=client -o yaml
kubectl create secret generic sec-name --from-literal=key=val --dry-run=client -o yaml

# Set context namespace (saves time every question)
kubectl config set-context --current --namespace=<ns>

# Fast resource inspection
kubectl get all -n <ns>                          # Everything in a namespace
kubectl get pods -A                              # All pods cluster-wide
kubectl describe pod <name> | tail -20           # Just the Events section
kubectl get endpoints <svc>                      # Check service routing

# Decode a secret
kubectl get secret <name> -o jsonpath='{.data.<key>}' | base64 -d

# Quick debug pod
kubectl run tmp --image=busybox:1.36 --rm -it --restart=Never -- sh
```

---

*Good luck on your CKAD journey! 🚀 Practice these 50 questions at least twice before the exam.*