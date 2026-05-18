# 💀 CKAD Hard Mode — Days 1–14 Practice Set
> **50 Questions · Harder Than the Real Exam · Pre-Setup · Post-Cleanup**

---

## ⚠️ READ BEFORE STARTING

> These questions are **intentionally harder** than the actual CKAD exam.
> The real exam gives you one clear task. These questions give you **broken environments, hidden traps, multi-step dependencies, and ambiguous requirements** — so that the real exam feels easy by comparison.

**Rules:**
- ⏱️ Each question has a **target time**. Exceed it = fail the question (exam mode)
- 🚫 Do NOT look at the solution until you've genuinely attempted the problem
- ✅ Always run the **Post-Cleanup** before the next question
- 🔁 If a question breaks your cluster state, run the cleanup and restart

**Difficulty Legend:**
- 🟡 `HARD` — harder than typical CKAD
- 🔴 `BRUTAL` — multi-step, traps, or broken environment
- 💀 `NIGHTMARE` — combines 3+ concepts, timed, with intentional noise

---

## 🌐 Global Environment Setup

> **Run once** at the very beginning. This creates baseline cluster state.

```bash
# ============================================================
# GLOBAL SETUP — Run this ONCE before starting any questions
# ============================================================

# Create base namespaces
kubectl create namespace prod
kubectl create namespace staging
kubectl create namespace monitoring
kubectl create namespace dev

# Create some "background noise" resources (red herrings)
kubectl run noise-1 --image=nginx:1.25 --labels="app=noise,env=prod" -n prod
kubectl run noise-2 --image=redis:7 --labels="app=noise,env=staging" -n staging
kubectl run noise-3 --image=busybox:1.36 --labels="app=noise" \
  --command -- sleep 9999 -n dev

# Create a pre-existing ConfigMap with WRONG data (for trap questions)
kubectl create configmap global-config \
  --from-literal=ENV=production \
  --from-literal=REGION=us-east-1 \
  --from-literal=DEBUG=true \
  -n prod

# Create a pre-existing broken Secret
kubectl create secret generic legacy-secret \
  --from-literal=API_KEY=PLACEHOLDER \
  --from-literal=DB_PASS=changeme \
  -n prod

# Label nodes for scheduling questions
kubectl label node minikube tier=backend 2>/dev/null || \
  kubectl label node $(kubectl get nodes -o jsonpath='{.items[0].metadata.name}') tier=backend

echo "✅ Global setup complete"
kubectl get namespaces | grep -E "prod|staging|monitoring|dev"
```

---

## 💀 SECTION 1 — Pod Mastery Under Pressure (Q1–Q10)

---

### Q1 · 🔴 BRUTAL — The Disappearing Pod

**⏱️ Target: 8 minutes**

**Context:** The `prod` namespace has a pod `api-server` that keeps crashing. Your job is to create a **replacement pod** that is resilient and self-documents its own restart behavior.

**Pre-Setup:**
```bash
# Creates a deliberately crashing pod
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: api-server
  namespace: prod
  labels:
    app: api-server
    version: v1
spec:
  restartPolicy: Always
  containers:
    - name: api
      image: busybox:1.36
      command: ["sh", "-c", "echo 'Starting...' && sleep 5 && exit 1"]
EOF
echo "Setup done. Wait 15 seconds before starting."
sleep 15
```

**Your Tasks:**
1. Identify the current state and restart count of `api-server` in `prod`
2. Create a **new** Pod named `api-server-v2` in `prod` that:
   - Uses `nginx:1.25`
   - Has label `app=api-server`, `version=v2`
   - Has an **annotation** `failure-reason=exit-1-crash` (documenting why v1 was replaced)
   - Has a **liveness probe** that checks for file `/tmp/healthy` every 10 seconds, with initial delay of 5 seconds
   - The container creates `/tmp/healthy` on startup using the command: `touch /tmp/healthy && nginx -g 'daemon off;'`
3. Delete the old `api-server` pod only after `api-server-v2` is `Running`

---

<details>
<summary>💡 Hint</summary>

Liveness probes use `livenessProbe.exec.command` for file checks. The annotation must be under `metadata.annotations`. Check restarts with `kubectl get pod -n prod --watch` or check `RESTARTS` column.

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Step 1: Diagnose the crashing pod
kubectl get pod api-server -n prod
# STATUS: CrashLoopBackOff, RESTARTS: 2+ (keeps crashing)

kubectl describe pod api-server -n prod | grep -E "Restart|Exit|State"
# Shows: Reason: Error, Exit Code: 1

# Step 2: Create the resilient replacement
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: api-server-v2
  namespace: prod
  labels:
    app: api-server
    version: v2
  annotations:
    failure-reason: "exit-1-crash"
spec:
  containers:
    - name: api
      image: nginx:1.25
      command: ["sh", "-c", "touch /tmp/healthy && nginx -g 'daemon off;'"]
      livenessProbe:
        exec:
          command:
            - cat
            - /tmp/healthy
        initialDelaySeconds: 5
        periodSeconds: 10
EOF

# Step 3: Wait for v2 to be Running, then delete v1
kubectl wait pod/api-server-v2 -n prod --for=condition=Ready --timeout=60s
kubectl delete pod api-server -n prod
```

**Verify:**
```bash
kubectl get pods -n prod -l app=api-server
# NAME           READY   STATUS    RESTARTS   AGE
# api-server-v2  1/1     Running   0          30s

kubectl describe pod api-server-v2 -n prod | grep -E "Annotations|Liveness"
# Annotations: failure-reason: exit-1-crash
# Liveness: exec [cat /tmp/healthy] delay=5s period=10s
```

</details>

<details>
<summary>📖 Explanation</summary>

**Trap 1:** The question says "create a replacement" but never tells you to delete the old one immediately — you must wait for the new one to be ready first. Deleting before v2 is Running = gap in service.

**Trap 2:** The liveness probe command must be able to run inside the container. `cat /tmp/healthy` works because we create the file at startup. If the file disappears, the liveness probe fails → container gets restarted.

**`kubectl wait`** is an exam superpower — it blocks until a condition is met, so you don't have to manually `watch` and wait.

**Liveness vs Readiness:**
- `livenessProbe` — "is the container alive?" → failure = restart
- `readinessProbe` — "is the container ready for traffic?" → failure = removed from Service endpoints

</details>

**Post-Cleanup:**
```bash
kubectl delete pod api-server api-server-v2 -n prod --ignore-not-found
```

---

### Q2 · 🔴 BRUTAL — The Poisoned Environment

**⏱️ Target: 10 minutes**

**Context:** A developer deployed a pod that reads config from both a ConfigMap and a Secret. The pod is running but the application is outputting wrong values. You must find the conflict and fix it **without restarting the pod if possible**, and if a restart is unavoidable, document why.

**Pre-Setup:**
```bash
kubectl create configmap app-conf \
  --from-literal=DB_HOST=localhost \
  --from-literal=DB_PORT=5432 \
  --from-literal=LOG_LEVEL=debug \
  -n staging

kubectl create secret generic app-secret \
  --from-literal=DB_HOST=prod-db.company.com \
  --from-literal=DB_PASS=s3cr3t \
  -n staging

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: conflict-pod
  namespace: staging
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "echo DB_HOST=$DB_HOST && echo DB_PORT=$DB_PORT && echo DB_PASS=$DB_PASS && echo LOG_LEVEL=$LOG_LEVEL && sleep 3600"]
      envFrom:
        - configMapRef:
            name: app-conf
        - secretRef:
            name: app-secret
EOF

sleep 5
echo "Current output:"
kubectl logs conflict-pod -n staging
```

**Your Tasks:**
1. Examine the pod logs — what value does `DB_HOST` show? Explain WHY.
2. The application **must** use `DB_HOST=prod-db.company.com` (from the secret). Fix this **without deleting the pod**.
3. After your fix, the pod logs should show the correct `DB_HOST`. If a restart was required, add annotation `restart-reason=env-conflict-resolution` to the pod.

---

<details>
<summary>💡 Hint</summary>

When `envFrom` has multiple sources with the same key, the **last one wins**. Check the order. Can you update env vars on a running pod without restarting it? (Spoiler: No — env vars are set at container start. You MUST restart the pod to change env vars, unlike volume-mounted ConfigMaps.)

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Step 1: Check current output
kubectl logs conflict-pod -n staging
# DB_HOST=localhost       ← WRONG! Should be prod-db.company.com
# DB_PORT=5432
# DB_PASS=s3cr3t
# LOG_LEVEL=debug

# WHY: envFrom processes sources in ORDER. ConfigMap is first → sets DB_HOST=localhost.
# Secret is second → sets DB_PASS, but DB_HOST is ALREADY SET so it does NOT override.
# Wait... actually let's test this. The secret also has DB_HOST.
# In Kubernetes: when both have the same key, the FIRST envFrom source wins (not last).

# Step 2: You CANNOT change env vars on a running pod without restart.
# We must delete and recreate. First, export and fix:
kubectl get pod conflict-pod -n staging -o yaml > conflict-pod-fix.yaml

# Edit the file: swap the order of envFrom (put secret FIRST, configmap SECOND)
# OR: replace envFrom with explicit env using valueFrom to be explicit

cat <<'EOF' | kubectl apply -f - --force
apiVersion: v1
kind: Pod
metadata:
  name: conflict-pod
  namespace: staging
  annotations:
    restart-reason: "env-conflict-resolution"
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "echo DB_HOST=$DB_HOST && echo DB_PORT=$DB_PORT && echo DB_PASS=$DB_PASS && echo LOG_LEVEL=$LOG_LEVEL && sleep 3600"]
      env:
        - name: DB_HOST           # Explicit: take from secret (wins over envFrom)
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: DB_HOST
        - name: DB_PORT
          valueFrom:
            configMapKeyRef:
              name: app-conf
              key: DB_PORT
        - name: DB_PASS
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: DB_PASS
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-conf
              key: LOG_LEVEL
EOF
```

**Verify:**
```bash
kubectl logs conflict-pod -n staging
# DB_HOST=prod-db.company.com  ✅
# DB_PORT=5432
# DB_PASS=s3cr3t
# LOG_LEVEL=debug

kubectl describe pod conflict-pod -n staging | grep restart-reason
# restart-reason: env-conflict-resolution
```

</details>

<details>
<summary>📖 Explanation</summary>

**The Conflict Rule:** When multiple `envFrom` sources define the same key, Kubernetes uses the **first source that defines it** — subsequent sources do NOT override. This is counterintuitive (you'd expect last-wins).

**The Hard Truth:** You **cannot** change environment variables on a running container. They are baked in at container start time. To change env vars, you must:
1. Delete and recreate the Pod
2. Use a volume-mounted ConfigMap (which CAN hot-reload)

**The Right Fix:** Use explicit `env` with `valueFrom` to be deterministic about exactly where each value comes from. Never rely on `envFrom` order for critical config.

**This is a real production bug** — teams discover this when they add a Secret with overlapping keys and wonder why their app ignores it.

</details>

**Post-Cleanup:**
```bash
kubectl delete pod conflict-pod -n staging --ignore-not-found
kubectl delete configmap app-conf -n staging --ignore-not-found
kubectl delete secret app-secret -n staging --ignore-not-found
```

---

### Q3 · 💀 NIGHTMARE — The Cascade Init Failure

**⏱️ Target: 12 minutes**

**Context:** A 3-stage pipeline pod has been deployed. It uses 2 init containers and 1 main container. The pod is stuck. Find every bug, fix them all, and get the pod running.

**Pre-Setup:**
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pipeline-pod
  namespace: dev
spec:
  volumes:
    - name: stage1-out
      emptyDir: {}
    - name: stage2-out
      emptyDir: {}
  initContainers:
    - name: stage1-fetch
      image: busybox:1.36
      command: ["sh", "-c", "echo 'raw_data' > /out/data.txt && echo stage1 done"]
      volumeMounts:
        - name: stage1-out
          mountPath: /out
    - name: stage2-transform
      image: busybox:1.36
      command: ["sh", "-c", "cat /in/data.txt | tr 'a-z' 'A-Z' > /out/processed.txt && cat /out/processed.txt"]
      volumeMounts:
        - name: stage1-out
          mountPath: /wrong-path      # BUG 1
        - name: stage2-out
          mountPath: /out
  containers:
    - name: stage3-serve
      image: busybox:1.36
      command: ["sh", "-c", "cat /data/processed.txt && sleep 3600"]
      volumeMounts:
        - name: stage2-out
          mountPath: /data
        - name: stage1-out
          mountPath: /raw             # This one is fine
EOF

echo "Wait 20s then check status:"
sleep 20
kubectl get pod pipeline-pod -n dev
kubectl describe pod pipeline-pod -n dev | grep -A 5 "Events:"
```

**Your Tasks:**
1. Identify ALL bugs in this pod definition (there may be more than one that isn't obvious at first)
2. Fix the pod and make it reach `Running` state
3. Verify that `/data/processed.txt` inside the main container contains `RAW_DATA` (uppercase)

---

<details>
<summary>💡 Hint</summary>

The obvious bug is the wrong mountPath in stage2. But look carefully at what path the command actually reads from (`/in/data.txt`) vs what's mounted. There may be a second hidden issue. Use `kubectl describe pod pipeline-pod -n dev` and look at each init container's state.

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Step 1: Diagnose
kubectl describe pod pipeline-pod -n dev
# Init container stage1-fetch: Completed ✅
# Init container stage2-transform: Error/Running...

# Look at stage2's command: it reads from /in/data.txt
# But stage1-out is mounted at /wrong-path (not /in)
# BUG 1: mountPath should be /in (to match the command reading /in/data.txt)

# Step 2: Also notice - the command reads /in/data.txt but stage1 writes to /out/data.txt
# When fixed, stage1-out mounts at /in in stage2, so /in/data.txt = stage1's /out/data.txt ✅

# Fix the pod
kubectl delete pod pipeline-pod -n dev

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pipeline-pod
  namespace: dev
spec:
  volumes:
    - name: stage1-out
      emptyDir: {}
    - name: stage2-out
      emptyDir: {}
  initContainers:
    - name: stage1-fetch
      image: busybox:1.36
      command: ["sh", "-c", "echo 'raw_data' > /out/data.txt && echo stage1 done"]
      volumeMounts:
        - name: stage1-out
          mountPath: /out
    - name: stage2-transform
      image: busybox:1.36
      command: ["sh", "-c", "cat /in/data.txt | tr 'a-z' 'A-Z' > /out/processed.txt && cat /out/processed.txt"]
      volumeMounts:
        - name: stage1-out
          mountPath: /in           # FIXED: was /wrong-path
        - name: stage2-out
          mountPath: /out
  containers:
    - name: stage3-serve
      image: busybox:1.36
      command: ["sh", "-c", "cat /data/processed.txt && sleep 3600"]
      volumeMounts:
        - name: stage2-out
          mountPath: /data
        - name: stage1-out
          mountPath: /raw
EOF
```

**Verify:**
```bash
kubectl get pod pipeline-pod -n dev
# STATUS: Running

kubectl exec pipeline-pod -n dev -- cat /data/processed.txt
# RAW_DATA   ← uppercase, transformation worked ✅

kubectl logs pipeline-pod -n dev -c stage2-transform
# RAW_DATA
```

</details>

<details>
<summary>📖 Explanation</summary>

**The Bug:** `stage2-transform` mounts `stage1-out` at `/wrong-path`, but the command reads from `/in/data.txt`. The volume is in the wrong place — the cat command fails with "No such file or directory", making stage2 exit with error → Pod stays in `Init:Error`.

**Debugging init containers:**
```bash
kubectl logs pipeline-pod -n dev -c stage2-transform   # Current/last run
kubectl logs pipeline-pod -n dev -c stage2-transform --previous  # Previous crash
```

**Key concept:** Init containers run sequentially. Stage2 cannot start until Stage1 completes successfully. If ANY init container fails, the Pod restarts that init container (based on restartPolicy). The main container NEVER starts until ALL init containers succeed.

**Volume data persistence:** `emptyDir` volumes persist for the life of the Pod — not just one container. Stage1 writes to the volume, stage2 reads it, stage3 reads stage2's output. This is a data pipeline pattern.

</details>

**Post-Cleanup:**
```bash
kubectl delete pod pipeline-pod -n dev --ignore-not-found
```

---

### Q4 · 🔴 BRUTAL — Resource Archaeology

**⏱️ Target: 8 minutes**

**Context:** Something is consuming excessive resources in the `prod` namespace. You receive this alert: *"Node memory pressure detected. Identify which pod is the highest memory consumer in `prod` and add an annotation `status=memory-offender` to it. Then enforce a memory limit of `64Mi` on it."*

**Pre-Setup:**
```bash
# Creates pods with varying resource declarations
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: svc-alpha
  namespace: prod
spec:
  containers:
    - name: app
      image: nginx:1.25
      resources:
        requests:
          memory: "32Mi"
        limits:
          memory: "128Mi"
---
apiVersion: v1
kind: Pod
metadata:
  name: svc-beta
  namespace: prod
spec:
  containers:
    - name: app
      image: nginx:1.25
      resources:
        requests:
          memory: "64Mi"
        limits:
          memory: "512Mi"
---
apiVersion: v1
kind: Pod
metadata:
  name: svc-gamma
  namespace: prod
spec:
  containers:
    - name: app
      image: nginx:1.25
      resources:
        requests:
          memory: "16Mi"
        limits:
          memory: "256Mi"
EOF
sleep 10
```

**Your Tasks:**
1. Find which pod in `prod` (excluding `noise-1`) has the **highest memory limit** declared
2. Annotate that pod with `status=memory-offender`
3. You **cannot delete the pod**. Patch its memory limit to `64Mi` using `kubectl patch`
4. Verify the limit is applied

> 💡 Note: In a real cluster you'd use `kubectl top pod`. Here, inspect declared limits.

---

<details>
<summary>💡 Hint</summary>

Use `kubectl get pod -n prod -o json` and `jsonpath` to extract memory limits. Or `kubectl describe pod -n prod | grep -A2 Limits`. Then use `kubectl patch pod <name> --type='json'` with a JSON patch operation to update the resource limit.

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Step 1: Find highest memory limit
kubectl get pods -n prod -o custom-columns=\
'NAME:.metadata.name,MEM_LIMIT:.spec.containers[0].resources.limits.memory' \
--field-selector=metadata.name!=noise-1

# Output:
# NAME        MEM_LIMIT
# svc-alpha   128Mi
# svc-beta    512Mi    ← HIGHEST
# svc-gamma   256Mi

# Step 2: Annotate the offender
kubectl annotate pod svc-beta -n prod status=memory-offender

# Step 3: Patch the memory limit (cannot delete pod)
# IMPORTANT: You CANNOT patch resource limits on a running pod via kubectl patch directly
# on spec.containers[].resources — it's an immutable field for running pods!
# This is the TRAP. You must use --force to replace:

kubectl get pod svc-beta -n prod -o yaml > svc-beta-fixed.yaml
# Edit svc-beta-fixed.yaml: change limits.memory from 512Mi to 64Mi
sed -i 's/memory: 512Mi/memory: 64Mi/' svc-beta-fixed.yaml

# Apply with force replace (this will briefly restart the pod)
kubectl replace --force -f svc-beta-fixed.yaml
```

**Verify:**
```bash
kubectl describe pod svc-beta -n prod | grep -A 3 "Limits:"
# Limits:
#   memory: 64Mi ✅

kubectl describe pod svc-beta -n prod | grep "status="
# status=memory-offender ✅
```

</details>

<details>
<summary>📖 Explanation</summary>

**The Trap:** Resource limits (`spec.containers[].resources`) are **immutable** on a running Pod. You CANNOT use `kubectl patch` to change them without recreating the pod.

The only way to update immutable pod fields is `kubectl replace --force`, which deletes and recreates the Pod automatically (brief downtime).

**For Deployments**, updating resources IS safe — `kubectl set resources deployment` triggers a rolling update. But bare Pods have no such mechanism.

**`custom-columns`** is a powerful output format for quick comparisons without writing full jsonpath expressions. Pattern: `NAME:.jsonpath.expression`.

This question tests whether you know which fields are mutable vs immutable — a crucial exam trap.

</details>

**Post-Cleanup:**
```bash
kubectl delete pod svc-alpha svc-beta svc-gamma -n prod --ignore-not-found
```

---

### Q5 · 💀 NIGHTMARE — The Silent Killer (Probes)

**⏱️ Target: 15 minutes**

**Context:** A pod keeps getting restarted mysteriously. No errors in logs. Investigate and fix. Then create a **correctly probed** version.

**Pre-Setup:**
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: probe-victim
  namespace: staging
spec:
  containers:
    - name: app
      image: nginx:1.25
      livenessProbe:
        httpGet:
          path: /healthz          # BUG: nginx doesn't serve /healthz
          port: 80
        initialDelaySeconds: 3
        periodSeconds: 5
        failureThreshold: 1       # BUG: kills after just 1 failure (too aggressive)
      readinessProbe:
        httpGet:
          path: /ready            # BUG: also doesn't exist
          port: 80
        initialDelaySeconds: 2
        periodSeconds: 5
EOF
echo "Wait 30s to observe the crash loop"
sleep 30
kubectl get pod probe-victim -n staging
```

**Your Tasks:**
1. Without looking at the YAML above, diagnose why `probe-victim` is restarting using only kubectl commands
2. Create a **fixed** pod named `probe-fixed` in `staging` that:
   - Uses `nginx:1.25`
   - Has a liveness probe on `/` (nginx root) with `failureThreshold: 3` and `periodSeconds: 10`
   - Has a readiness probe on `/` with `initialDelaySeconds: 5`
   - Has a **startup probe** that gives nginx 30 seconds to start (check every 5 seconds, up to 6 times)
3. Verify `probe-fixed` never restarts and becomes `Ready`

---

<details>
<summary>💡 Hint</summary>

`kubectl describe pod probe-victim -n staging` will show probe failure events. For the startup probe, use `startupProbe` with `failureThreshold * periodSeconds` = total allowed startup time.

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Step 1: Diagnose WITHOUT reading the pre-setup YAML
kubectl get pod probe-victim -n staging
# RESTARTS: 3+ (keeps going up)

kubectl describe pod probe-victim -n staging | grep -A 5 "Events:"
# Warning  Unhealthy  Liveness probe failed: HTTP probe failed with statuscode: 404
# Warning  Killing    Container probe-victim failed liveness probe, will be restarted

kubectl describe pod probe-victim -n staging | grep -E "Liveness|Readiness"
# Liveness:   http-get http://:80/healthz delay=3s period=5s failure=1
# Readiness:  http-get http://:80/ready delay=2s period=5s failure=3
# DIAGNOSIS: /healthz doesn't exist on nginx → 404 → probe fails → restart

# Step 2: Create properly configured pod
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: probe-fixed
  namespace: staging
spec:
  containers:
    - name: app
      image: nginx:1.25
      startupProbe:
        httpGet:
          path: /
          port: 80
        failureThreshold: 6        # 6 attempts × 5s = 30s startup window
        periodSeconds: 5
      livenessProbe:
        httpGet:
          path: /
          port: 80
        failureThreshold: 3        # 3 strikes before kill
        periodSeconds: 10
        initialDelaySeconds: 0    # startupProbe covers the wait
      readinessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 10
EOF
```

**Verify:**
```bash
kubectl get pod probe-fixed -n staging -w
# Watch: 0/1 → 1/1 Running, RESTARTS stays at 0

kubectl describe pod probe-fixed -n staging | grep -E "Started|Liveness|Readiness|Startup"
# Startup probe kicks in first, then liveness + readiness
```

</details>

<details>
<summary>📖 Explanation</summary>

**The Three Probes — When Each Fires:**

```
Pod starts
    ↓
[startupProbe] ← runs until success or failureThreshold hit
    ↓ (success)
[livenessProbe] ← runs forever; failure = container restart
[readinessProbe] ← runs forever; failure = removed from Service endpoints
```

**Why startupProbe exists:** slow-starting apps (JVM, large ML models) need more time to be ready. Without startupProbe, you'd need a high `initialDelaySeconds` on liveness — but that leaves a big window where a crashed app isn't restarted. startupProbe solves this cleanly.

**The bugs analyzed:**
1. `/healthz` doesn't exist on stock nginx → 404 → probe failure
2. `failureThreshold: 1` → ONE failure kills it → even a single slow response = restart
3. `/ready` also doesn't exist → pod never becomes Ready → never gets Service traffic

**Exam rule:** Always probe endpoints that ACTUALLY EXIST on your container.

</details>

**Post-Cleanup:**
```bash
kubectl delete pod probe-victim probe-fixed -n staging --ignore-not-found
```

---

### Q6 · 🟡 HARD — Pod Security Context Puzzle

**⏱️ Target: 10 minutes**

**Context:** Security team mandates all pods in `prod` must run as non-root. A developer's pod keeps failing. Fix it.

**Pre-Setup:**
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
  namespace: prod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 0          # BUG: user 0 IS root — contradicts runAsNonRoot
  containers:
    - name: app
      image: nginx:1.25   # BUG: stock nginx requires root (port 80 + user nginx)
EOF
sleep 10
kubectl get pod secure-app -n prod
```

**Your Tasks:**
1. Diagnose why `secure-app` fails to start
2. Create a **fixed** pod named `secure-app-v2` that:
   - Runs as user `1000`, group `3000`
   - Has `allowPrivilegeEscalation: false`
   - Uses `nginx:unprivileged` (which runs as non-root on port 8080)
   - Has a readiness probe on port `8080` path `/`
3. Verify it runs without root and passes security constraints

---

<details>
<summary>💡 Hint</summary>

`runAsUser: 0` is root. Change it to `1000`. Stock nginx requires root for port 80. Use `nginxinc/nginx-unprivileged` or `nginx:unprivileged` which listens on `8080` as non-root. Add `allowPrivilegeEscalation: false` under `securityContext` at the container level.

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Step 1: Diagnose
kubectl describe pod secure-app -n prod | tail -20
# Error: container has runAsNonRoot and image will run as root
# OR the pod starts and nginx fails to bind port 80 (permission denied)

kubectl get pod secure-app -n prod
# STATUS: Error or CreateContainerConfigError

# Step 2: Create correctly secured pod
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secure-app-v2
  namespace: prod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 3000
  containers:
    - name: app
      image: nginxinc/nginx-unprivileged:latest
      securityContext:
        allowPrivilegeEscalation: false
      readinessProbe:
        httpGet:
          path: /
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 10
EOF
```

**Verify:**
```bash
kubectl get pod secure-app-v2 -n prod
# STATUS: Running, READY: 1/1

kubectl exec secure-app-v2 -n prod -- id
# uid=1000 gid=3000 groups=3000   ← NOT root ✅

kubectl exec secure-app-v2 -n prod -- cat /proc/1/status | grep CapEff
# CapEff: 0000000000000000   ← No capabilities ✅
```

</details>

<details>
<summary>📖 Explanation</summary>

**securityContext levels:**
- `spec.securityContext` → applies to ALL containers in the pod
- `spec.containers[].securityContext` → applies to ONE container (overrides pod level)

**The user 0 trap:** `runAsNonRoot: true` + `runAsUser: 0` is a contradiction. Kubernetes detects this and rejects the container with `CreateContainerConfigError`.

**Port binding:** Ports below 1024 require root privileges on Linux. Stock nginx runs on port 80 and needs root. `nginx-unprivileged` is specifically built to run on port 8080 as a normal user — always use this in security-conscious environments.

**`allowPrivilegeEscalation: false`** prevents the container from gaining more privileges than its parent process (blocks `sudo`, `setuid` binaries). This should always be `false` in production.

</details>

**Post-Cleanup:**
```bash
kubectl delete pod secure-app secure-app-v2 -n prod --ignore-not-found
```

---

### Q7 · 🔴 BRUTAL — The Missing Sidecar Data

**⏱️ Target: 12 minutes**

**Context:** A multi-container pod has a main app and a log-shipping sidecar. The sidecar reports it cannot find any logs. Debug and fix.

**Pre-Setup:**
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: log-ship
  namespace: dev
spec:
  volumes:
    - name: logs
      emptyDir: {}
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "while true; do echo \"$(date) request processed\" >> /app/logs/app.log; sleep 2; done"]
      volumeMounts:
        - name: logs
          mountPath: /app/logs
    - name: log-shipper
      image: busybox:1.36
      command: ["sh", "-c", "while true; do if [ -f /logs/app.log ]; then tail -5 /logs/app.log; else echo 'NO LOGS FOUND'; fi; sleep 5; done"]
      volumeMounts:
        - name: logs
          mountPath: /logs/app     # BUG: mounted at wrong subpath level
EOF
sleep 15
echo "Sidecar output:"
kubectl logs log-ship -n dev -c log-shipper
```

**Your Tasks:**
1. Without reading the YAML above, diagnose why the sidecar prints "NO LOGS FOUND"
2. Fix the pod (you MUST delete and recreate)
3. After fix: prove the sidecar reads real log lines by showing 3 recent log entries from the sidecar

---

<details>
<summary>💡 Hint</summary>

The app writes to `/app/logs/app.log`. The sidecar looks for `/logs/app.log`. Check what's actually in the sidecar's filesystem with `kubectl exec`.

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Step 1: Diagnose
kubectl logs log-ship -n dev -c log-shipper
# NO LOGS FOUND  ← repeated

# Check what's in the sidecar filesystem
kubectl exec log-ship -n dev -c log-shipper -- ls /logs/
# app     ← the DIRECTORY is here (it's mounted at /logs/app)

kubectl exec log-ship -n dev -c log-shipper -- ls /logs/app/
# app.log   ← the file is at /logs/app/app.log, not /logs/app.log!

# DIAGNOSIS: sidecar checks /logs/app.log but file is at /logs/app/app.log
# The volume mount at /logs/app means the shared volume root is at /logs/app/
# So app.log is at /logs/app/app.log

# Step 2: Fix — correct the sidecar mountPath to /logs
kubectl delete pod log-ship -n dev

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: log-ship
  namespace: dev
spec:
  volumes:
    - name: logs
      emptyDir: {}
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "while true; do echo \"$(date) request processed\" >> /app/logs/app.log; sleep 2; done"]
      volumeMounts:
        - name: logs
          mountPath: /app/logs
    - name: log-shipper
      image: busybox:1.36
      command: ["sh", "-c", "while true; do if [ -f /logs/app.log ]; then tail -5 /logs/app.log; else echo 'NO LOGS FOUND'; fi; sleep 5; done"]
      volumeMounts:
        - name: logs
          mountPath: /logs          # FIXED: matches where app.log is written
EOF

sleep 15
```

**Verify:**
```bash
kubectl logs log-ship -n dev -c log-shipper | head -10
# Mon Jan 1 00:00:01 UTC 2024 request processed
# Mon Jan 1 00:00:03 UTC 2024 request processed
# Mon Jan 1 00:00:05 UTC 2024 request processed  ✅
```

</details>

<details>
<summary>📖 Explanation</summary>

**The mountPath trap:** When you mount a volume at `/logs/app`, the volume's root directory appears at `/logs/app/`. If the app writes to `/app/logs/app.log` (where the volume is mounted at `/app/logs`), the file is at path `/app/logs/app.log` from the app's view — and at the **volume root** as `app.log`.

From the sidecar's view (mounted at `/logs/app`): the file is at `/logs/app/app.log`.

The sidecar was looking for `/logs/app.log` (wrong path). The fix: mount the volume at `/logs` so the file appears at `/logs/app.log`.

**Rule:** The volume is a shared filesystem. Both containers see the SAME files, but through DIFFERENT paths determined by their respective `mountPath`. Draw it out:

```
Volume root/
    app.log

App container:   mountPath=/app/logs  → sees /app/logs/app.log
Sidecar fixed:   mountPath=/logs      → sees /logs/app.log    ✅
Sidecar broken:  mountPath=/logs/app  → sees /logs/app/app.log ❌
```

</details>

**Post-Cleanup:**
```bash
kubectl delete pod log-ship -n dev --ignore-not-found
```

---

### Q8 · 🟡 HARD — Ephemeral Debug Container

**⏱️ Target: 8 minutes**

**Context:** A distroless production pod has no shell, no curl, no debugging tools. You need to debug it without restarting it.

**Pre-Setup:**
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: distroless-app
  namespace: prod
spec:
  containers:
    - name: app
      image: gcr.io/distroless/static:nonroot
      command: ["/bin/true"]
EOF
# Note: distroless has no shell. kubectl exec -it ... -- sh will FAIL.
sleep 10
kubectl get pod distroless-app -n prod
```

**Your Tasks:**
1. Attempt to exec into `distroless-app` and document the exact error you receive
2. Use an **ephemeral debug container** to attach a busybox shell to the running pod
3. From inside the debug container, check if the process `/bin/true` is still in the process list

---

<details>
<summary>💡 Hint</summary>

Use `kubectl debug -it pod/distroless-app -n prod --image=busybox:1.36 --target=app`. Ephemeral containers share the process namespace when `--target` is specified.

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Step 1: Attempt normal exec (will fail)
kubectl exec -it distroless-app -n prod -- sh
# OCI runtime exec failed: exec failed: ... no such file or directory
# OR: error: Internal error occurred: ... cannot exec in a container without a shell

# Step 2: Launch ephemeral debug container
kubectl debug -it distroless-app -n prod \
  --image=busybox:1.36 \
  --target=app \
  -- sh

# You're now inside a busybox shell attached to the pod
# From inside:
ps aux
# PID   USER     TIME  COMMAND
# 1     nonroot  0:00  /bin/true    ← The distroless app process ✅
# (other processes from the debug container itself)

ls /proc/1/exe  # Link to the main process binary
exit
```

**Verify:**
```bash
# After exiting, ephemeral containers persist (but you can't exec back in)
kubectl describe pod distroless-app -n prod | grep -A 5 "Ephemeral Containers:"
# debugger-xxx   busybox:1.36
```

</details>

<details>
<summary>📖 Explanation</summary>

**Distroless images** contain only the application and its runtime dependencies — no shell, no package manager, no debugging tools. They're more secure (smaller attack surface) but harder to debug.

**Ephemeral containers** (kubectl debug) are temporary containers added to a running Pod. They:
- Don't restart on failure
- Can share process/network namespace with the target container (`--target=app`)
- Are perfect for debugging production pods without restart

**`--target=app`** enables process namespace sharing — you can see the main app's processes with `ps`. Without it, you only see your own debug container's processes.

**Important:** Ephemeral containers cannot be removed once added (they show in `kubectl describe` permanently). They exit when you type `exit` but the container record remains.

</details>

**Post-Cleanup:**
```bash
kubectl delete pod distroless-app -n prod --ignore-not-found
```

---

### Q9 · 🔴 BRUTAL — Scheduled Pod Placement

**⏱️ Target: 10 minutes**

**Context:** Two pods must be scheduled on the SAME node (co-location) for performance. Use pod affinity to enforce this.

**Pre-Setup:**
```bash
# Deploy the "anchor" pod first
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: cache-server
  namespace: prod
  labels:
    app: cache
    colocation-group: group-a
spec:
  containers:
    - name: cache
      image: redis:7
EOF
sleep 10
NODE=$(kubectl get pod cache-server -n prod -o jsonpath='{.spec.nodeName}')
echo "cache-server is on node: $NODE"
```

**Your Tasks:**
1. Find which node `cache-server` is running on
2. Create a pod `app-server` in `prod` that uses **pod affinity** to guarantee it runs on the SAME node as `cache-server` (matched by label `colocation-group=group-a`)
3. Verify both pods are on the same node

---

<details>
<summary>💡 Hint</summary>

Use `spec.affinity.podAffinity.requiredDuringSchedulingIgnoredDuringExecution`. The `topologyKey: kubernetes.io/hostname` means "same node". The `labelSelector` matches the anchor pod's labels.

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Step 1: Find node
kubectl get pod cache-server -n prod -o wide
# Shows NODE column

# Step 2: Create co-located pod
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: app-server
  namespace: prod
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: colocation-group
                operator: In
                values:
                  - group-a
          topologyKey: kubernetes.io/hostname
  containers:
    - name: app
      image: nginx:1.25
EOF
```

**Verify:**
```bash
kubectl get pods cache-server app-server -n prod -o wide
# Both should show same NODE value ✅
```

</details>

<details>
<summary>📖 Explanation</summary>

**Pod Affinity** schedules pods NEAR other pods (vs Node Affinity which targets node labels).

`topologyKey` defines the unit of co-location:
- `kubernetes.io/hostname` → same node
- `topology.kubernetes.io/zone` → same availability zone
- `topology.kubernetes.io/region` → same region

**Pod Anti-Affinity** (the opposite) is used to spread pods across nodes for high availability — the #1 use case in production.

```yaml
podAntiAffinity:  # Spread replicas across nodes
  preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchLabels:
            app: webapp
        topologyKey: kubernetes.io/hostname
```

</details>

**Post-Cleanup:**
```bash
kubectl delete pod cache-server app-server -n prod --ignore-not-found
```

---

### Q10 · 💀 NIGHTMARE — Tainted Node Recovery

**⏱️ Target: 10 minutes**

**Pre-Setup:**
```bash
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
kubectl taint node $NODE dedicated=gpu:NoSchedule
echo "Node $NODE tainted. New pods won't schedule here."

# This pod will get stuck in Pending
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: stuck-pod
  namespace: dev
spec:
  containers:
    - name: app
      image: nginx:1.25
EOF
sleep 10
kubectl get pod stuck-pod -n dev
```

**Your Tasks:**
1. Diagnose why `stuck-pod` is in `Pending` state
2. Create a NEW pod `gpu-workload` in `dev` that **tolerates** the taint and schedules successfully
3. After `gpu-workload` is Running, remove the taint from the node so `stuck-pod` can also schedule
4. Verify both pods are Running at the end

---

<details>
<summary>💡 Hint</summary>

Taints block pods. Tolerations allow specific pods to bypass taints. Taint format: `key=value:Effect`. To tolerate: `spec.tolerations[].key`, `.operator`, `.value`, `.effect`. Remove a taint with `kubectl taint node <node> <key>:<effect>-` (note the trailing dash).

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Step 1: Diagnose
kubectl describe pod stuck-pod -n dev | grep -A 3 "Events:"
# Warning  FailedScheduling  0/1 nodes are available: 1 node(s) had untolerated taint {dedicated: gpu}

kubectl describe node minikube | grep Taints
# Taints: dedicated=gpu:NoSchedule

# Step 2: Create pod with toleration
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: gpu-workload
  namespace: dev
spec:
  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "gpu"
      effect: "NoSchedule"
  containers:
    - name: app
      image: nginx:1.25
EOF

kubectl wait pod/gpu-workload -n dev --for=condition=Ready --timeout=60s

# Step 3: Remove taint so stuck-pod can schedule
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
kubectl taint node $NODE dedicated=gpu:NoSchedule-
# The trailing dash removes the taint

# Wait for stuck-pod to schedule
kubectl wait pod/stuck-pod -n dev --for=condition=Ready --timeout=60s
```

**Verify:**
```bash
kubectl get pod stuck-pod gpu-workload -n dev
# NAME          READY   STATUS    RESTARTS   AGE
# stuck-pod     1/1     Running   0          1m  ✅
# gpu-workload  1/1     Running   0          2m  ✅

kubectl describe node minikube | grep Taints
# Taints: <none>   ← taint removed ✅
```

</details>

<details>
<summary>📖 Explanation</summary>

**Taints and Tolerations** work like a lock-and-key system:
- **Taint** on node = "I reject pods that don't have the right key"
- **Toleration** on pod = "I have the key to bypass this taint"

**Three taint effects:**
- `NoSchedule` → new pods without toleration won't schedule (existing pods unaffected)
- `PreferNoSchedule` → soft version, scheduler tries to avoid
- `NoExecute` → new pods won't schedule AND existing pods without toleration are evicted

**Remove syntax:** `kubectl taint node <name> <key>:<effect>-` — the trailing `-` means remove.

**Real use case:** Dedicated GPU nodes — taint them so only GPU workloads (with tolerations) run there, keeping them free from regular workloads.

</details>

**Post-Cleanup:**
```bash
kubectl delete pod stuck-pod gpu-workload -n dev --ignore-not-found
# Make sure taint is removed
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
kubectl taint node $NODE dedicated=gpu:NoSchedule- 2>/dev/null || true
```

---

## 🔵 SECTION 2 — Deployment Warfare (Q11–Q20)

---

### Q11 · 🔴 BRUTAL — Deployment Rollout Autopsy

**⏱️ Target: 10 minutes**

**Pre-Setup:**
```bash
kubectl create deployment ghost-app --image=nginx:1.24 --replicas=4 -n prod
kubectl rollout status deployment/ghost-app -n prod

# Update to v2
kubectl set image deployment/ghost-app nginx=nginx:1.25 -n prod
kubectl rollout status deployment/ghost-app -n prod

# Update to a BROKEN v3
kubectl set image deployment/ghost-app nginx=nginx:DOESNOTEXIST -n prod
# Don't wait - this will hang/fail
```

**Your Tasks:**
1. The v3 rollout is stuck. Prove it's stuck (don't just look — use commands)
2. Find what revision number the working version (nginx:1.25) is on
3. Roll back to specifically that revision (not just "undo" — use the revision number)
4. Verify all 4 replicas are running nginx:1.25

---

<details>
<summary>💡 Hint</summary>

`kubectl rollout history deployment/ghost-app -n prod` shows revisions. `kubectl rollout history ... --revision=N` shows what's in that revision. `kubectl rollout undo ... --to-revision=N` rolls back to a specific version.

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Step 1: Prove it's stuck
kubectl rollout status deployment/ghost-app -n prod --timeout=10s
# error: timed out waiting for the condition

kubectl get pods -n prod -l app=ghost-app
# Some pods in ImagePullBackOff, some still running nginx:1.25

kubectl get replicasets -n prod -l app=ghost-app
# New RS: DESIRED=1+ but READY=0 (stuck)
# Old RS: still has running pods

# Step 2: Find working revision
kubectl rollout history deployment/ghost-app -n prod
# REVISION  CHANGE-CAUSE
# 1         <none>   nginx:1.24
# 2         <none>   nginx:1.25
# 3         <none>   nginx:DOESNOTEXIST (current broken)

kubectl rollout history deployment/ghost-app -n prod --revision=2
# Confirms revision 2 = nginx:1.25

# Step 3: Roll back to specific revision
kubectl rollout undo deployment/ghost-app -n prod --to-revision=2

# Step 4: Verify
kubectl rollout status deployment/ghost-app -n prod
kubectl get pods -n prod -l app=ghost-app
```

**Verify:**
```bash
kubectl get pods -n prod -l app=ghost-app -o jsonpath='{range .items[*]}{.spec.containers[0].image}{"\n"}{end}'
# nginx:1.25   ✅ (all 4 pods)
```

</details>

<details>
<summary>📖 Explanation</summary>

**Rollout sticks at max surge:** By default, a Deployment allows 1 extra pod during rollout (`maxSurge=1`). It creates 1 new pod with the bad image, finds it can't pull, but won't delete any old pods until the new one is Ready. Result: stuck at "1 old pod being replaced, 1 new pod failing."

**Rollout history is your audit trail.** Each `set image` or spec change creates a new revision. `--revision=N` lets you inspect exactly what's in any revision before rolling back to it.

**`--to-revision=0`** = go back to previous (same as `undo` without `--to-revision`).
**`--to-revision=N`** = go back to a specific checkpoint.

</details>

**Post-Cleanup:**
```bash
kubectl delete deployment ghost-app -n prod --ignore-not-found
```

---

### Q12 · 💀 NIGHTMARE — Blue/Green Deployment by Hand

**⏱️ Target: 15 minutes**

**Context:** You need to implement a manual blue/green deployment. No Helm, no Argo CD — just kubectl.

**Pre-Setup:**
```bash
# Blue version is live
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-blue
  namespace: staging
  labels:
    app: myapp
    version: blue
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
        - name: app
          image: nginx:1.24
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
  namespace: staging
spec:
  selector:
    app: myapp
    version: blue     # Currently pointing at blue
  ports:
    - port: 80
      targetPort: 80
EOF
kubectl rollout status deployment/app-blue -n staging
```

**Your Tasks:**
1. Deploy a GREEN version (`app-green`, image: `nginx:1.25`, 3 replicas) in `staging` — but the service should NOT route to it yet
2. Verify green is running and healthy (all 3 replicas Ready) while blue still serves traffic
3. **Cut over** the service to green with zero downtime (single kubectl command)
4. Verify the service endpoints now point ONLY to green pods
5. Scale blue down to 0 (keep the deployment for potential rollback)

---

<details>
<summary>✅ Solution</summary>

```bash
# Step 1: Deploy green (service still points to blue)
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-green
  namespace: staging
  labels:
    app: myapp
    version: green
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
        - name: app
          image: nginx:1.25
EOF

# Step 2: Wait for green to be ready WITHOUT touching the service
kubectl rollout status deployment/app-green -n staging

# Verify blue still serves (service endpoints = blue pods)
kubectl get endpoints myapp-svc -n staging

# Step 3: Atomic cutover — update service selector in ONE command
kubectl patch service myapp-svc -n staging \
  -p '{"spec":{"selector":{"app":"myapp","version":"green"}}}'

# Step 4: Verify endpoints now show green pod IPs
kubectl get endpoints myapp-svc -n staging
# Compare with green pod IPs:
kubectl get pods -n staging -l version=green -o wide

# Step 5: Scale blue to 0 (keep deployment for rollback)
kubectl scale deployment app-blue -n staging --replicas=0
```

**Verify:**
```bash
# Service points to green
kubectl describe service myapp-svc -n staging | grep Selector
# Selector: app=myapp,version=green  ✅

# Blue is scaled to 0
kubectl get deployment app-blue -n staging
# READY: 0/0  ✅

# Green is running
kubectl get deployment app-green -n staging
# READY: 3/3  ✅
```

</details>

<details>
<summary>📖 Explanation</summary>

**Blue/Green deployment pattern:**
1. Run both versions simultaneously (blue=live, green=standby)
2. Test green thoroughly before cutover
3. Atomic cutover = single `kubectl patch` changes the Service selector
4. Rollback = single `kubectl patch` back to blue selector

**Why this is better than rolling update:**
- Zero downtime (atomic switch)
- Easy rollback (just patch selector again)
- Can test green in production without serving user traffic (via direct pod access)
- No mixed versions serving traffic simultaneously

**The `patch` command for atomic selector change is the key insight.** One command = immediate cutover. No gradual transition.

</details>

**Post-Cleanup:**
```bash
kubectl delete deployment app-blue app-green -n staging --ignore-not-found
kubectl delete service myapp-svc -n staging --ignore-not-found
```

---

### Q13 · 🔴 BRUTAL — Deployment with Custom Rolling Update Strategy

**⏱️ Target: 10 minutes**

**Context:** You're deploying a critical financial service. It must never have fewer than 80% of pods available during an update. Deploy with these exact constraints and prove they're enforced.

**Your Tasks:**
1. Create deployment `fintech-app` in `prod` (image: `nginx:1.24`, 10 replicas)
2. Configure rolling update: `maxSurge=2`, `maxUnavailable=2`
3. Perform an image update to `nginx:1.25` and watch the rollout
4. Verify that during the rollout, at least 8 pods are always available (10 - maxUnavailable=2)

---

<details>
<summary>💡 Hint</summary>

Set `spec.strategy.type: RollingUpdate` and `spec.strategy.rollingUpdate.maxSurge` + `maxUnavailable`. You can use absolute numbers or percentages. Watch with `kubectl get pods -l app=fintech-app -w`.

</details>

<details>
<summary>✅ Solution</summary>

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fintech-app
  namespace: prod
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2         # Can have up to 12 pods total
      maxUnavailable: 2   # Minimum 8 pods always available
  selector:
    matchLabels:
      app: fintech-app
  template:
    metadata:
      labels:
        app: fintech-app
    spec:
      containers:
        - name: app
          image: nginx:1.24
EOF

kubectl rollout status deployment/fintech-app -n prod

# Update and watch — open two terminals
# Terminal 1: watch
kubectl get pods -n prod -l app=fintech-app -w &

# Terminal 2: trigger update
kubectl set image deployment/fintech-app app=nginx:1.25 -n prod
kubectl rollout status deployment/fintech-app -n prod
```

**Verify strategy:**
```bash
kubectl describe deployment fintech-app -n prod | grep -A 4 "StrategyType\|RollingUpdateStrategy"
# StrategyType: RollingUpdate
# RollingUpdateStrategy: 2 max unavailable, 2 max surge  ✅
```

</details>

<details>
<summary>📖 Explanation</summary>

**maxUnavailable=2 with 10 replicas:**
- Minimum pods during rollout = 10 - 2 = 8 (80% availability ✅)
- Kubernetes won't terminate more than 2 old pods at a time

**maxSurge=2 with 10 replicas:**
- Maximum pods during rollout = 10 + 2 = 12
- Can spin up 2 new pods before terminating old ones

**The math matters in exams:** A question might say "must maintain 90% availability with 20 replicas" → maxUnavailable = 2 (10% of 20). Know how to calculate both absolute numbers and percentages.

**Percentages are also valid:**
```yaml
maxSurge: 25%
maxUnavailable: 10%
```

</details>

**Post-Cleanup:**
```bash
kubectl delete deployment fintech-app -n prod --ignore-not-found
```

---

### Q14 · 🟡 HARD — The Orphaned ReplicaSet Trap

**⏱️ Target: 8 minutes**

**Pre-Setup:**
```bash
kubectl create deployment orphan-test --image=nginx:1.24 --replicas=2 -n dev
kubectl rollout status deployment/orphan-test -n dev

# Simulate a manual mistake: someone directly scaled the ReplicaSet
RS_NAME=$(kubectl get rs -n dev -l app=orphan-test -o jsonpath='{.items[0].metadata.name}')
kubectl scale rs $RS_NAME -n dev --replicas=5
echo "ReplicaSet: $RS_NAME manually scaled to 5"
sleep 5
kubectl get pods -n dev -l app=orphan-test
```

**Your Tasks:**
1. Observe the pod count. What's happening? Why?
2. Scale the **Deployment** to 4 replicas. What do you expect to happen? What actually happens?
3. Explain the behavior and fix it permanently so the deployment controls the replica count

---

<details>
<summary>💡 Hint</summary>

Deployments manage ReplicaSets. If you directly scale an RS, the Deployment immediately reconciles it back to its desired state. The Deployment is the "source of truth."

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Step 1: Observe - the RS was manually scaled to 5
kubectl get pods -n dev -l app=orphan-test
# Shows 5 pods briefly

# But then watch what happens:
kubectl get rs -n dev -l app=orphan-test -w
# The Deployment reconciles: replicas goes 5 → 2 almost immediately
# REASON: Deployment continuously enforces its spec.replicas=2

# Step 2: Scale deployment to 4
kubectl scale deployment orphan-test -n dev --replicas=4
kubectl get pods -n dev -l app=orphan-test
# Now shows 4 pods ✅

# The Deployment updated the RS to 4 — this is the RIGHT way

# Step 3: Explanation is the answer. Fix = always use kubectl scale deployment, never kubectl scale rs
# Verify the deployment is the controller:
kubectl get rs -n dev -l app=orphan-test -o jsonpath='{.items[0].metadata.ownerReferences[0].kind}'
# Deployment  ← the RS is owned by the Deployment
```

**Verify:**
```bash
kubectl get deployment orphan-test -n dev
# READY: 4/4  ✅

kubectl get rs -n dev -l app=orphan-test
# DESIRED: 4, CURRENT: 4, READY: 4  ✅ (matches deployment)
```

</details>

<details>
<summary>📖 Explanation</summary>

**The reconciliation loop:** Deployments run a control loop that constantly compares "current state" vs "desired state." If they differ, it fixes it immediately.

Manually scaling a ReplicaSet directly is immediately overridden by the Deployment controller. The RS is a child resource — the Deployment is authoritative.

**Owner References:** Every RS has `ownerReferences` pointing to its Deployment. This is how Kubernetes tracks resource ownership. If you `kubectl delete` a Deployment, its owned RSes and Pods are also deleted (garbage collected).

**Lesson:** Never manage child resources directly when a parent controller exists. Always go through the controller (Deployment, StatefulSet, etc.).

</details>

**Post-Cleanup:**
```bash
kubectl delete deployment orphan-test -n dev --ignore-not-found
```

---

### Q15 · 💀 NIGHTMARE — Cross-Namespace Dependency Chain

**⏱️ Target: 15 minutes**

**Context:** A three-tier application spans three namespaces. Set up the complete chain and verify end-to-end connectivity.

**Your Tasks:**
1. In namespace `dev`: Deploy `backend` (nginx:1.25, 2 replicas), expose as ClusterIP service `backend-svc` on port 8080 → targetPort 80
2. In namespace `staging`: Deploy `middleware` (busybox:1.36, 1 replica) that makes an HTTP GET to `backend-svc.dev.svc.cluster.local:8080` every 10 seconds and logs the response code
3. In namespace `prod`: Deploy `frontend` (nginx:1.25, 3 replicas), expose as NodePort service `frontend-svc` on port 30090
4. Prove the middleware in `staging` can reach the backend in `dev` by showing its logs

---

<details>
<summary>✅ Solution</summary>

```bash
# Step 1: Backend in dev
kubectl create deployment backend --image=nginx:1.25 --replicas=2 -n dev
kubectl expose deployment backend --port=8080 --target-port=80 --name=backend-svc -n dev

# Step 2: Middleware in staging (calls backend cross-namespace)
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: middleware
  namespace: staging
spec:
  replicas: 1
  selector:
    matchLabels:
      app: middleware
  template:
    metadata:
      labels:
        app: middleware
    spec:
      containers:
        - name: mw
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              while true; do
                CODE=$(wget -qO- --server-response http://backend-svc.dev.svc.cluster.local:8080 2>&1 | grep "HTTP/" | awk '{print $2}')
                echo "$(date) - backend response: ${CODE:-connection_failed}"
                sleep 10
              done
EOF

# Step 3: Frontend in prod
kubectl create deployment frontend --image=nginx:1.25 --replicas=3 -n prod
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
  namespace: prod
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30090
EOF

kubectl rollout status deployment/middleware -n staging
```

**Verify:**
```bash
kubectl logs -l app=middleware -n staging --tail=5
# Mon Jan 1 00:00:10 UTC 2024 - backend response: 200  ✅
# Cross-namespace DNS works!

kubectl get service frontend-svc -n prod
# TYPE: NodePort, PORT(S): 80:30090/TCP  ✅
```

</details>

<details>
<summary>📖 Explanation</summary>

**Cross-namespace DNS pattern:** `<service>.<namespace>.svc.cluster.local`

Within the same namespace: `backend-svc` works
Cross-namespace: `backend-svc.dev.svc.cluster.local` is required

**This is the most important networking concept for CKAD:** Services are namespace-scoped, but DNS allows cross-namespace access using the full FQDN.

**NetworkPolicies** (not in days 1-14) can restrict this cross-namespace access — something to look forward to in later lessons.

</details>

**Post-Cleanup:**
```bash
kubectl delete deployment backend -n dev --ignore-not-found
kubectl delete service backend-svc -n dev --ignore-not-found
kubectl delete deployment middleware -n staging --ignore-not-found
kubectl delete deployment frontend -n prod --ignore-not-found
kubectl delete service frontend-svc -n prod --ignore-not-found
```

---

## 🟣 SECTION 3 — ConfigMap & Secret Gauntlet (Q16–Q25)

---

### Q16 · 🔴 BRUTAL — The Stale ConfigMap

**⏱️ Target: 10 minutes**

**Pre-Setup:**
```bash
kubectl create configmap live-config \
  --from-literal=FEATURE_FLAG=false \
  --from-literal=TIMEOUT=30 \
  -n staging

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: reader-env
  namespace: staging
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "while true; do echo \"FEATURE_FLAG=$FEATURE_FLAG TIMEOUT=$TIMEOUT\"; sleep 5; done"]
      envFrom:
        - configMapRef:
            name: live-config
---
apiVersion: v1
kind: Pod
metadata:
  name: reader-vol
  namespace: staging
spec:
  volumes:
    - name: config
      configMap:
        name: live-config
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "while true; do echo \"FEATURE=$(cat /config/FEATURE_FLAG) TIMEOUT=$(cat /config/TIMEOUT)\"; sleep 5; done"]
      volumeMounts:
        - name: config
          mountPath: /config
EOF
sleep 10
```

**Your Tasks:**
1. Check current logs of both `reader-env` and `reader-vol`
2. Update `live-config`: set `FEATURE_FLAG=true` and `TIMEOUT=60`
3. Wait 90 seconds, then check logs of BOTH pods again
4. Which pod reflected the change WITHOUT restart? Which one didn't? Explain precisely why.
5. Make `reader-env` reflect the new values too (with minimal disruption)

---

<details>
<summary>✅ Solution</summary>

```bash
# Step 1: Check before
kubectl logs reader-env -n staging --tail=2
# FEATURE_FLAG=false TIMEOUT=30

kubectl logs reader-vol -n staging --tail=2
# FEATURE=false TIMEOUT=30

# Step 2: Update configmap
kubectl patch configmap live-config -n staging \
  -p '{"data":{"FEATURE_FLAG":"true","TIMEOUT":"60"}}'

# Step 3: Wait 90s
sleep 90

# Step 4: Check after
kubectl logs reader-env -n staging --tail=2
# FEATURE_FLAG=false TIMEOUT=30   ← DID NOT UPDATE (env vars are static)

kubectl logs reader-vol -n staging --tail=2
# FEATURE=true TIMEOUT=60   ← UPDATED automatically ✅

# Step 5: Force reader-env to pick up new values
kubectl delete pod reader-env -n staging
# (Pod is bare, not managed by Deployment — it won't come back automatically)
# Recreate it:
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: reader-env
  namespace: staging
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "while true; do echo \"FEATURE_FLAG=$FEATURE_FLAG TIMEOUT=$TIMEOUT\"; sleep 5; done"]
      envFrom:
        - configMapRef:
            name: live-config
EOF

sleep 5
kubectl logs reader-env -n staging --tail=2
# FEATURE_FLAG=true TIMEOUT=60  ✅
```

</details>

<details>
<summary>📖 Explanation</summary>

**This is one of the most critical ConfigMap concepts:**

| Injection Method | Auto-updates on CM change? | Latency |
|---|---|---|
| `envFrom` / `env` | ❌ Never | N/A |
| Volume mount | ✅ Yes | ~30-90 seconds |

**Why env vars don't update:** Kubernetes sets environment variables once when the container process starts. They're set by the container runtime and cannot be changed in a running process — this is a Linux fundamental, not a Kubernetes limitation.

**Volume mount update mechanism:** The kubelet periodically syncs ConfigMap data to the mounted files. The sync interval is controlled by `--sync-frequency` (default: 1 minute). Apps that watch their config files (like many modern frameworks) will pick up the changes automatically.

**The right architecture for hot-reload:** Use volume mounts for any configuration you need to change without a restart.

</details>

**Post-Cleanup:**
```bash
kubectl delete pod reader-env reader-vol -n staging --ignore-not-found
kubectl delete configmap live-config -n staging --ignore-not-found
```

---

### Q17 · 💀 NIGHTMARE — Secret Rotation Under Load

**⏱️ Target: 15 minutes**

**Context:** An application uses a database password stored in a Secret. The security team requires the password to be rotated. The app MUST NOT experience downtime.

**Pre-Setup:**
```bash
kubectl create secret generic db-credentials \
  --from-literal=password=OldP@ssw0rd123 \
  -n prod

cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db-client
  namespace: prod
spec:
  replicas: 3
  selector:
    matchLabels:
      app: db-client
  template:
    metadata:
      labels:
        app: db-client
    spec:
      volumes:
        - name: secret-vol
          secret:
            secretName: db-credentials
      containers:
        - name: client
          image: busybox:1.36
          command: ["sh", "-c", "while true; do echo \"Current password: $(cat /secrets/password)\"; sleep 10; done"]
          volumeMounts:
            - name: secret-vol
              mountPath: /secrets
              readOnly: true
EOF
kubectl rollout status deployment/db-client -n prod
echo "Old password is live in all pods"
```

**Your Tasks:**
1. Confirm all 3 pods currently read `OldP@ssw0rd123`
2. Rotate the secret to `N3wP@ssw0rd456` without deleting the Secret object
3. Without restarting pods, wait for all pods to pick up the new password (volume mount auto-sync)
4. Confirm all 3 pods now read the new password
5. Add annotation `last-rotated=<current-date>` to the secret

---

<details>
<summary>✅ Solution</summary>

```bash
# Step 1: Confirm old password
kubectl logs -l app=db-client -n prod --tail=1
# Current password: OldP@ssw0rd123  (×3 pods)

# Step 2: Rotate secret (patch, don't delete)
NEW_PASS=$(echo -n 'N3wP@ssw0rd456' | base64)
kubectl patch secret db-credentials -n prod \
  -p "{\"data\":{\"password\":\"${NEW_PASS}\"}}"

# Verify secret updated
kubectl get secret db-credentials -n prod -o jsonpath='{.data.password}' | base64 -d
# N3wP@ssw0rd456

# Step 3: Wait for volume mount sync (up to 90 seconds)
echo "Waiting for volume sync..."
sleep 90

# Step 4: Confirm new password in all pods
kubectl logs -l app=db-client -n prod --tail=1
# Current password: N3wP@ssw0rd456  (×3 pods, NO RESTART NEEDED) ✅

# Step 5: Annotate the secret
TODAY=$(date +%Y-%m-%d)
kubectl annotate secret db-credentials -n prod last-rotated=$TODAY
```

**Verify:**
```bash
kubectl describe secret db-credentials -n prod | grep last-rotated
# last-rotated: 2024-01-15  ✅

kubectl logs -l app=db-client -n prod --tail=1
# Current password: N3wP@ssw0rd456  ✅ (no pod restart)
```

</details>

<details>
<summary>📖 Explanation</summary>

**Zero-downtime secret rotation is only possible with volume mounts.** This is the architectural advantage of mounting secrets as files vs environment variables.

**The rotation procedure:**
1. `kubectl patch secret` → updates the Secret object in etcd
2. Kubelet detects the Secret changed
3. Kubelet syncs the new data to all mounted volume paths
4. Running pods read the new file content on their next read

**Apps must be designed for this:** The app must re-read the file on each use (not cache it at startup). Modern secrets management libraries (e.g., in Spring Boot, Django) support this pattern.

**In production:** Combine this with:
- Vault dynamic secrets (auto-rotation with short TTLs)
- Sealed Secrets (encrypted secrets in Git)
- External Secrets Operator (sync from AWS Secrets Manager, etc.)

</details>

**Post-Cleanup:**
```bash
kubectl delete deployment db-client -n prod --ignore-not-found
kubectl delete secret db-credentials -n prod --ignore-not-found
```

---

### Q18 · 🔴 BRUTAL — Multi-Source Config Injection

**⏱️ Target: 12 minutes**

**Your Tasks — all in namespace `dev`:**

Create all resources to make a pod `mega-config` that receives configuration from **4 different sources**:
1. `env` literal: `APP_NAME=megaapp`
2. ConfigMap `base-config` (keys: `LOG_LEVEL=warn`, `REGION=ap-southeast-1`), injected via `envFrom`
3. Secret `api-creds` (keys: `API_KEY=abc123`, `API_SECRET=xyz789`), injected via `envFrom`
4. ConfigMap `feature-flags` mounted as volume at `/etc/flags/` (keys: `flag_a=enabled`, `flag_b=disabled`)

The container (`busybox:1.36`) must print ALL of these at startup, then sleep.
Verify every single value is accessible.

---

<details>
<summary>✅ Solution</summary>

```bash
# Create ConfigMaps and Secrets
kubectl create configmap base-config \
  --from-literal=LOG_LEVEL=warn \
  --from-literal=REGION=ap-southeast-1 \
  -n dev

kubectl create secret generic api-creds \
  --from-literal=API_KEY=abc123 \
  --from-literal=API_SECRET=xyz789 \
  -n dev

kubectl create configmap feature-flags \
  --from-literal=flag_a=enabled \
  --from-literal=flag_b=disabled \
  -n dev

# Create the mega-config pod
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: mega-config
  namespace: dev
spec:
  volumes:
    - name: flags
      configMap:
        name: feature-flags
  containers:
    - name: app
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          echo "=== ENV LITERAL ==="
          echo "APP_NAME=$APP_NAME"
          echo "=== FROM CONFIGMAP ==="
          echo "LOG_LEVEL=$LOG_LEVEL"
          echo "REGION=$REGION"
          echo "=== FROM SECRET ==="
          echo "API_KEY=$API_KEY"
          echo "API_SECRET=$API_SECRET"
          echo "=== FROM VOLUME ==="
          echo "flag_a=$(cat /etc/flags/flag_a)"
          echo "flag_b=$(cat /etc/flags/flag_b)"
          sleep 3600
      env:
        - name: APP_NAME
          value: "megaapp"
      envFrom:
        - configMapRef:
            name: base-config
        - secretRef:
            name: api-creds
      volumeMounts:
        - name: flags
          mountPath: /etc/flags
EOF
```

**Verify:**
```bash
kubectl logs mega-config -n dev
# === ENV LITERAL ===
# APP_NAME=megaapp          ✅
# === FROM CONFIGMAP ===
# LOG_LEVEL=warn            ✅
# REGION=ap-southeast-1     ✅
# === FROM SECRET ===
# API_KEY=abc123            ✅
# API_SECRET=xyz789         ✅
# === FROM VOLUME ===
# flag_a=enabled            ✅
# flag_b=disabled           ✅
```

</details>

<details>
<summary>📖 Explanation</summary>

**Precedence when keys conflict across sources:**
1. `env` literals (highest priority — always win)
2. First `envFrom` source
3. Second `envFrom` source (cannot override keys already set)

**Volume-mounted config is separate from env vars** — it's file-based, not process environment. You access it via `cat /path/file` not `$VAR_NAME`.

**Architecture decision guide:**
- Use `env` literals → hardcoded, non-sensitive, won't change
- Use `envFrom configMapRef` → bulk non-sensitive config
- Use `envFrom secretRef` → bulk sensitive config
- Use volume mount → large files, hot-reload required, or multiple Pods sharing config

</details>

**Post-Cleanup:**
```bash
kubectl delete pod mega-config -n dev --ignore-not-found
kubectl delete configmap base-config feature-flags -n dev --ignore-not-found
kubectl delete secret api-creds -n dev --ignore-not-found
```

---

### Q19 · 🟡 HARD — Decode and Audit All Secrets

**⏱️ Target: 8 minutes**

**Pre-Setup:**
```bash
kubectl create secret generic audit-target \
  --from-literal=username=sysadmin \
  --from-literal=password='P@$$w0rd!2024' \
  --from-literal=token=eyJhbGciOiJIUzI1NiJ9.test \
  -n prod
```

**Your Tasks:**
1. Without using `kubectl describe`, retrieve and decode ALL values from `audit-target` in `prod`
2. Output them in the format: `KEY=decoded_value` (one per line)
3. Count the total number of keys in the secret
4. Find ALL secrets across ALL namespaces that have more than 2 keys

---

<details>
<summary>💡 Hint</summary>

Use `kubectl get secret -o jsonpath` or `-o json` combined with `base64 -d`. For counting keys, use `kubectl get secret -o jsonpath='{.data}' | python3 -c` or `jq`. For all namespaces with multiple keys — think about combining `kubectl get secrets -A -o json` with filtering.

</details>

<details>
<summary>✅ Solution</summary>

```bash
# Step 1: Decode all values
kubectl get secret audit-target -n prod -o json | \
  python3 -c "
import json, sys, base64
s = json.load(sys.stdin)
for k, v in s['data'].items():
    print(f'{k}={base64.b64decode(v).decode()}')
"
# username=sysadmin
# password=P@$$w0rd!2024
# token=eyJhbGciOiJIUzI1NiJ9.test

# Alternative with jsonpath (per-key):
for key in username password token; do
  val=$(kubectl get secret audit-target -n prod \
    -o jsonpath="{.data.$key}" | base64 -d)
  echo "$key=$val"
done

# Step 2: Count keys
kubectl get secret audit-target -n prod \
  -o jsonpath='{.data}' | python3 -c \
  "import json,sys; d=json.load(sys.stdin); print(f'Key count: {len(d)}')"
# Key count: 3

# Step 3: Find secrets with > 2 keys across all namespaces
kubectl get secrets -A -o json | python3 -c "
import json, sys
data = json.load(sys.stdin)
for item in data['items']:
    ns = item['metadata']['namespace']
    name = item['metadata']['name']
    keys = item.get('data', {})
    if len(keys) > 2:
        print(f'{ns}/{name}: {len(keys)} keys - {list(keys.keys())}')
"
```

**Verify:**
```bash
# Quick count verification
kubectl get secret audit-target -n prod -o jsonpath='{.data}' | \
  grep -o '"[^"]*":' | wc -l
# 3  ✅
```

</details>

<details>
<summary>📖 Explanation</summary>

**`-o jsonpath`** is your most powerful data extraction tool in the exam. Learn these patterns:

```bash
# Single field
kubectl get secret name -o jsonpath='{.data.key}' | base64 -d

# All data keys
kubectl get secret name -o jsonpath='{.data}' 

# Loop over items
kubectl get secrets -A -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}{"\n"}{end}'
```

**`-o json` + Python/jq** is more flexible for complex operations like counting keys or filtering.

**Security audit best practice:** Regularly audit which secrets exist, who owns them, and how many keys they contain. Secrets with many keys are harder to rotate and often indicate poor design (should be split into separate secrets by rotation schedule).

</details>

**Post-Cleanup:**
```bash
kubectl delete secret audit-target -n prod --ignore-not-found
```

---

### Q20 · 💀 NIGHTMARE — ConfigMap as nginx Virtual Host

**⏱️ Target: 15 minutes**

**Your Tasks — all in namespace `staging`:**

1. Create ConfigMap `vhost-config` with a complete nginx config as key `default.conf`:
   - Server listens on port 80
   - Location `/api` returns `{"status":"ok","version":"v2"}` with content-type `application/json`
   - Location `/health` returns `healthy` with status 200
   - All other locations return 404

2. Deploy `vhost-nginx` (image: `nginx:1.25`, 2 replicas) mounting the ConfigMap at `/etc/nginx/conf.d/default.conf` using `subPath`

3. Expose as ClusterIP service `vhost-svc` on port 80

4. Verify all three endpoints work by curling from a test pod:
   - `/api` → JSON response
   - `/health` → `healthy`
   - `/` → 404

---

<details>
<summary>✅ Solution</summary>

```yaml
# Step 1: ConfigMap with nginx config
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: vhost-config
  namespace: staging
data:
  default.conf: |
    server {
        listen 80;
        
        location /api {
            default_type application/json;
            return 200 '{"status":"ok","version":"v2"}';
        }
        
        location /health {
            default_type text/plain;
            return 200 'healthy';
        }
        
        location / {
            return 404;
        }
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vhost-nginx
  namespace: staging
spec:
  replicas: 2
  selector:
    matchLabels:
      app: vhost-nginx
  template:
    metadata:
      labels:
        app: vhost-nginx
    spec:
      volumes:
        - name: nginx-conf
          configMap:
            name: vhost-config
      containers:
        - name: nginx
          image: nginx:1.25
          volumeMounts:
            - name: nginx-conf
              mountPath: /etc/nginx/conf.d/default.conf
              subPath: default.conf
---
apiVersion: v1
kind: Service
metadata:
  name: vhost-svc
  namespace: staging
spec:
  selector:
    app: vhost-nginx
  ports:
    - port: 80
      targetPort: 80
EOF
```

**Verify:**
```bash
kubectl rollout status deployment/vhost-nginx -n staging

# Test from inside cluster
kubectl run curl-test --image=curlimages/curl --rm -it --restart=Never -- sh

# Inside the shell:
curl -s http://vhost-svc.staging.svc.cluster.local/api
# {"status":"ok","version":"v2"}  ✅

curl -s http://vhost-svc.staging.svc.cluster.local/health
# healthy  ✅

curl -s -o /dev/null -w "%{http_code}" http://vhost-svc.staging.svc.cluster.local/
# 404  ✅
```

</details>

<details>
<summary>📖 Explanation</summary>

**`subPath` is critical here.** Without it, mounting at `/etc/nginx/conf.d/default.conf` would:
- Replace the entire `/etc/nginx/conf.d/` directory with a directory containing only `default.conf`
- Remove all other nginx conf.d files
- nginx might still work, but you lose the default nginx configuration

With `subPath: default.conf`, only the specific file is injected — the rest of the directory is untouched.

**Nginx `return` directive** is perfect for mock APIs in testing environments — no actual application needed.

**Real-world use:** This pattern is used for nginx sidecar proxies, custom routing rules, and SSL termination configs — all managed as ConfigMaps updated without pod restarts.

</details>

**Post-Cleanup:**
```bash
kubectl delete deployment vhost-nginx -n staging --ignore-not-found
kubectl delete service vhost-svc -n staging --ignore-not-found
kubectl delete configmap vhost-config -n staging --ignore-not-found
```

---

## 🔴 SECTION 4 — Service Networking Nightmares (Q21–Q30)

---

### Q21 · 🔴 BRUTAL — Service Endpoint Debugging Chain

**⏱️ Target: 12 minutes**

**Pre-Setup:**
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: target-app
  namespace: prod
spec:
  replicas: 3
  selector:
    matchLabels:
      app: target
      env: prod
  template:
    metadata:
      labels:
        app: target         # Missing env:prod label!
    spec:
      containers:
        - name: app
          image: nginx:1.25
---
apiVersion: v1
kind: Service
metadata:
  name: target-svc
  namespace: prod
spec:
  selector:
    app: target
    env: prod
  ports:
    - port: 80
      targetPort: 80
EOF
sleep 10
```

**Your Tasks:**
1. The service `target-svc` has no endpoints. Without reading the YAML above, find ALL the reasons using only kubectl commands.
2. Fix the issue without deleting any resources. (Hint: is there more than one way to fix this? Which is safer in production?)
3. Verify the service has 3 endpoints after your fix.

---

<details>
<summary>✅ Solution</summary>

```bash
# Step 1: Diagnose
kubectl get endpoints target-svc -n prod
# ENDPOINTS: <none>

kubectl describe service target-svc -n prod | grep Selector
# Selector: app=target,env=prod

kubectl get pods -n prod -l app=target --show-labels
# Shows pods exist BUT labels don't include env=prod

# Root cause: Pod template labels: {app:target} 
# Service selector: {app:target, env:prod} — doesn't match

# Step 2: Two fix options:
# OPTION A: Add label to pods (patch deployment template) — SAFER
kubectl patch deployment target-app -n prod \
  --type='json' \
  -p='[{"op":"add","path":"/spec/template/metadata/labels/env","value":"prod"}]'

# This triggers rolling update — new pods get the label

# OPTION B: Remove env:prod from service selector (less strict)
# kubectl patch service target-svc -n prod -p '{"spec":{"selector":{"app":"target"}}}'

# Option A is safer: keeps selector strict, fixes the root cause (missing label)
kubectl rollout status deployment/target-app -n prod

# Step 3: Verify
kubectl get endpoints target-svc -n prod
# ENDPOINTS: 10.x.x.x:80,10.x.x.x:80,10.x.x.x:80  ✅ 3 endpoints
```

</details>

<details>
<summary>📖 Explanation</summary>

**The mismatch:** Deployment's `spec.selector.matchLabels` was `{app:target, env:prod}` but `spec.template.metadata.labels` only had `{app:target}`. Kubernetes REQUIRES these to match (it validates this), BUT the service selector is separate — and it required `env:prod` which the pods didn't have.

**Wait — how did the Deployment even create pods?** Kubernetes validates that `spec.selector.matchLabels` must be a subset of `spec.template.metadata.labels`. If `env:prod` was in the selector but not the template, the Deployment would be rejected. So actually the Deployment's OWN selector matches (since it only uses `app:target`), but the SERVICE's selector also requires `env:prod`.

**Two fix philosophies:**
- Fix the pods (add label) → keeps service selector strict and accurate
- Fix the service (remove requirement) → easier but loses specificity

In production, the first option is safer — it means the service still only routes to the exact pods you intended.

</details>

**Post-Cleanup:**
```bash
kubectl delete deployment target-app -n prod --ignore-not-found
kubectl delete service target-svc -n prod --ignore-not-found
```

---

### Q22 · 💀 NIGHTMARE — Service Account + RBAC for Pod (Sneak Preview)

**⏱️ Target: 10 minutes**

**Context:** A pod needs to list all pods in the `dev` namespace using the Kubernetes API from within the cluster. Wire it up.

**Your Tasks:**
1. Create a ServiceAccount `pod-reader` in `dev`
2. Create a Role `pod-list-role` in `dev` that allows `get`, `list`, `watch` on `pods`
3. Create a RoleBinding that binds `pod-reader` to `pod-list-role` in `dev`
4. Create a Pod `k8s-spy` in `dev` using the `bitnami/kubectl` image and `pod-reader` service account, that lists all pods in `dev` using the in-cluster API
5. Verify the pod successfully lists pods

---

<details>
<summary>✅ Solution</summary>

```bash
# Step 1-3: ServiceAccount, Role, RoleBinding
kubectl create serviceaccount pod-reader -n dev

kubectl create role pod-list-role \
  --verb=get,list,watch \
  --resource=pods \
  -n dev

kubectl create rolebinding pod-reader-binding \
  --role=pod-list-role \
  --serviceaccount=dev:pod-reader \
  -n dev

# Step 4: Pod using the service account
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: k8s-spy
  namespace: dev
spec:
  serviceAccountName: pod-reader
  containers:
    - name: kubectl
      image: bitnami/kubectl:latest
      command: ["kubectl", "get", "pods", "-n", "dev"]
EOF

sleep 15
```

**Verify:**
```bash
kubectl logs k8s-spy -n dev
# NAME          READY   STATUS    RESTARTS   AGE
# k8s-spy       1/1     Running   0          10s
# noise-3       1/1     Running   0          ...
# (lists pods in dev namespace) ✅

# Verify it CAN'T list pods in prod (not authorized)
kubectl auth can-i list pods -n prod --as=system:serviceaccount:dev:pod-reader
# no  ✅ (properly scoped to dev only)
```

</details>

<details>
<summary>📖 Explanation</summary>

**In-cluster authentication:** When a pod uses a ServiceAccount, Kubernetes automatically mounts a token at `/var/run/secrets/kubernetes.io/serviceaccount/token`. The `bitnami/kubectl` image uses this token automatically to authenticate against the API server.

**RBAC components:**
- **ServiceAccount** → the identity (who)
- **Role** → the permissions (what) — namespace-scoped
- **RoleBinding** → the connection (who gets what) — namespace-scoped
- **ClusterRole/ClusterRoleBinding** → same but cluster-wide

`kubectl auth can-i` is your RBAC testing tool — use it to verify permissions without actually running the operation.

</details>

**Post-Cleanup:**
```bash
kubectl delete pod k8s-spy -n dev --ignore-not-found
kubectl delete rolebinding pod-reader-binding -n dev --ignore-not-found
kubectl delete role pod-list-role -n dev --ignore-not-found
kubectl delete serviceaccount pod-reader -n dev --ignore-not-found
```

---

### Q23 · 🔴 BRUTAL — DNS Investigation

**⏱️ Target: 10 minutes**

**Pre-Setup:**
```bash
kubectl create deployment dns-backend --image=nginx:1.25 --replicas=2 -n staging
kubectl expose deployment dns-backend --port=9000 --target-port=80 -n staging
```

**Your Tasks (from a debug pod):**
1. Launch a debug pod in `dev` namespace
2. From it, resolve the DNS of `dns-backend` service in `staging` — use ALL valid FQDN forms
3. Prove that short-name DNS (`dns-backend`) does NOT work cross-namespace
4. Test actual HTTP connectivity to the service
5. Find the ClusterIP of CoreDNS and query it directly

---

<details>
<summary>✅ Solution</summary>

```bash
# Launch debug pod in dev namespace
kubectl run debug --image=busybox:1.36 --rm -it --restart=Never -n dev -- sh

# Inside the shell:

# 1. Resolve using full FQDN forms
nslookup dns-backend.staging.svc.cluster.local
# Works ✅ — full FQDN

nslookup dns-backend.staging.svc
# Works ✅ 

nslookup dns-backend.staging
# Works ✅

# 2. Short name (FAILS cross-namespace)
nslookup dns-backend
# Server: 10.96.0.10
# ** server can't find dns-backend: NXDOMAIN  ← Expected! Cross-namespace needs FQDN

# 3. Test HTTP connectivity
wget -qO- http://dns-backend.staging.svc.cluster.local:9000
# Returns nginx HTML ✅

# 4. Find CoreDNS IP
cat /etc/resolv.conf
# nameserver 10.96.0.10   ← This is kube-dns
nslookup dns-backend.staging.svc.cluster.local 10.96.0.10
# Queries CoreDNS directly ✅

exit
```

**Verify:**
```bash
# From outside: confirm CoreDNS service
kubectl get service kube-dns -n kube-system
# NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
# kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP   xxx
```

</details>

<details>
<summary>📖 Explanation</summary>

**DNS search domains:** Inside a pod, `/etc/resolv.conf` has search domains like:
```
search dev.svc.cluster.local svc.cluster.local cluster.local
```

Short name `dns-backend` gets these appended in order:
1. `dns-backend.dev.svc.cluster.local` → NXDOMAIN (service is in staging, not dev)
2. `dns-backend.svc.cluster.local` → NXDOMAIN
3. `dns-backend.cluster.local` → NXDOMAIN

All fail, so the short name doesn't resolve cross-namespace. **You must use the full namespace FQDN.**

**CoreDNS** is the DNS server for all Kubernetes clusters. Understanding that `10.96.0.10` (or whatever your cluster's kube-dns ClusterIP is) handles all internal DNS resolution is fundamental to debugging network issues.

</details>

**Post-Cleanup:**
```bash
kubectl delete deployment dns-backend -n staging --ignore-not-found
kubectl delete service dns-backend -n staging --ignore-not-found
```

---

### Q24 · 🟡 HARD — Service Without a Selector (Manual Endpoints)

**⏱️ Target: 10 minutes**

**Context:** You need to create a Service that routes to an external IP (`8.8.8.8` DNS server on port 53) — but using a custom Service without ExternalName (to maintain full control over the Endpoints object).

**Your Tasks:**
1. Create a Service `external-dns` in `dev` with no selector
2. Manually create an Endpoints object `external-dns` in `dev` pointing to `8.8.8.8:53`
3. Verify the service routes correctly by querying it from a pod

---

<details>
<summary>💡 Hint</summary>

A Service without a `selector` doesn't auto-create Endpoints — you create them manually. The Endpoints object must have the SAME NAME as the Service.

</details>

<details>
<summary>✅ Solution</summary>

```yaml
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: external-dns
  namespace: dev
spec:
  ports:
    - port: 53
      targetPort: 53
      protocol: UDP
  # No selector — we manage endpoints manually
---
apiVersion: v1
kind: Endpoints
metadata:
  name: external-dns   # MUST match Service name
  namespace: dev
subsets:
  - addresses:
      - ip: 8.8.8.8
    ports:
      - port: 53
        protocol: UDP
EOF
```

**Verify:**
```bash
kubectl get endpoints external-dns -n dev
# NAME           ENDPOINTS       AGE
# external-dns   8.8.8.8:53      10s ✅

# Test DNS query through the service
kubectl run dns-query --image=busybox:1.36 --rm -it --restart=Never -n dev \
  -- nslookup kubernetes.default external-dns.dev.svc.cluster.local
# Should resolve using 8.8.8.8 as DNS server ✅
```

</details>

<details>
<summary>📖 Explanation</summary>

**Services without selectors** give you full manual control over what IPs a Service routes to. Use cases:
- External databases with static IPs
- Services in other Kubernetes clusters
- Legacy infrastructure not in Kubernetes
- Blue/green switching by manual endpoint management

**The Endpoints object must match the Service name exactly** — this is how Kubernetes links them.

This is more flexible than ExternalName because:
- You can have multiple IPs in the Endpoints (load balancing)
- You can use port 53 and other protocols
- ExternalName is purely DNS-based (CNAME), while manual Endpoints work at the IP level

</details>

**Post-Cleanup:**
```bash
kubectl delete service external-dns -n dev --ignore-not-found
kubectl delete endpoints external-dns -n dev --ignore-not-found
```

---

### Q25 · 💀 NIGHTMARE — The Service Cascade Debug

**⏱️ Target: 20 minutes**

**Pre-Setup:**
```bash
cat <<'EOF' | kubectl apply -f -
# Tier 1: Frontend
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fe
  namespace: prod
spec:
  replicas: 2
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
        - name: fe
          image: busybox:1.36
          command: ["sh","-c","while true; do echo $(wget -qO- http://be-svc.prod.svc.cluster.local:8080 2>/dev/null || echo 'BACKEND_UNREACHABLE'); sleep 5; done"]
---
# Tier 2: Backend — BROKEN
apiVersion: apps/v1
kind: Deployment
metadata:
  name: be
  namespace: prod
spec:
  replicas: 2
  selector:
    matchLabels:
      tier: backend
  template:
    metadata:
      labels:
        tier: backend
    spec:
      containers:
        - name: be
          image: nginx:1.25
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: be-svc
  namespace: prod
spec:
  selector:
    tier: backend
  ports:
    - port: 8080      # Service port 8080
      targetPort: 8081  # BUG: nginx listens on 80, not 8081
EOF
sleep 15
```

**Your Tasks:**
1. The frontend pods report `BACKEND_UNREACHABLE`. Use a systematic debug process to find the root cause
2. Fix the service
3. Verify frontend pods now successfully reach the backend
4. Document: at what layer was the failure? (DNS / Service / Pod / Application)

---

<details>
<summary>✅ Solution</summary>

```bash
# Systematic debug process:

# Layer 1: Is the service DNS resolving?
kubectl run debug -n prod --image=busybox:1.36 --rm -it --restart=Never -- sh
# Inside:
nslookup be-svc.prod.svc.cluster.local
# Resolves! DNS is fine.

# Layer 2: Does the service have endpoints?
kubectl get endpoints be-svc -n prod
# NAME     ENDPOINTS                         AGE
# be-svc   10.244.x.x:8081,10.244.x.x:8081  ← Port 8081! 

# Layer 3: Can we reach the pod directly on the endpoint port?
# (Still inside debug pod)
wget -qO- http://10.244.x.x:8081
# wget: can't connect  ← Port 8081 not open!

# Layer 4: What port does nginx actually listen on?
wget -qO- http://10.244.x.x:80
# Returns nginx HTML! ← Port 80 is correct

# ROOT CAUSE: Service targetPort=8081 but nginx listens on 80
exit

# Fix: Correct the targetPort
kubectl patch service be-svc -n prod -p '{"spec":{"ports":[{"port":8080,"targetPort":80}]}}'

# Verify endpoints now show port 80
kubectl get endpoints be-svc -n prod
# ENDPOINTS: 10.244.x.x:80,10.244.x.x:80  ✅

# Verify frontend now reaches backend
sleep 10
kubectl logs -l tier=frontend -n prod --tail=3
# <!DOCTYPE html>... (nginx response)  ✅
```

</details>

<details>
<summary>📖 Explanation</summary>

**The Systematic Network Debug Process (memorize this):**

```
1. DNS resolution    → nslookup <service>
2. Endpoints exist   → kubectl get endpoints <svc>
3. Endpoint port     → Can I reach pod:port directly?
4. Pod port          → What port is the app actually listening on?
5. Service port      → Does targetPort match the app port?
```

**The failure layer:** Pod layer — the service was sending traffic to the right pods but the wrong port (8081 vs 80). The pods were healthy, DNS was fine, endpoints existed — but the targetPort was wrong.

**`targetPort` vs `port`:**
- `port` = the port the Service listens on (what clients use)
- `targetPort` = the port the container listens on (must match containerPort/app)

These can be different — a common pattern is `port: 80, targetPort: 8080` when your app runs on 8080 but you want to expose it as port 80.

</details>

**Post-Cleanup:**
```bash
kubectl delete deployment fe be -n prod --ignore-not-found
kubectl delete service be-svc -n prod --ignore-not-found
```

---

## ⚫ SECTION 5 — Exam Simulation Gauntlets (Q26–Q50)

---

### Q26 · 💀 NIGHTMARE — The Full Namespace Audit

**⏱️ Target: 15 minutes**

**Your Tasks:** Perform a complete audit of the `prod` namespace and generate a report. All answers via kubectl commands only.

Find and output:
1. All running pods and their images
2. All services and their selectors
3. Any service that has NO endpoints (broken)
4. All ConfigMaps (excluding `kube-root-ca.crt`)
5. All Secrets (excluding default service account tokens)
6. Any pod without resource limits defined
7. Total count of all resources in the namespace

---

<details>
<summary>✅ Solution</summary>

```bash
echo "=== PROD NAMESPACE AUDIT ==="

echo "--- 1. Running Pods and Images ---"
kubectl get pods -n prod -o custom-columns=\
'NAME:.metadata.name,IMAGE:.spec.containers[0].image,STATUS:.status.phase'

echo "--- 2. Services and Selectors ---"
kubectl get services -n prod -o custom-columns=\
'NAME:.metadata.name,TYPE:.spec.type,SELECTOR:.spec.selector'

echo "--- 3. Services with No Endpoints ---"
kubectl get endpoints -n prod | awk 'NR==1 || $2=="<none>"'

echo "--- 4. ConfigMaps ---"
kubectl get configmaps -n prod --field-selector=metadata.name!=kube-root-ca.crt

echo "--- 5. Secrets (non-default) ---"
kubectl get secrets -n prod | grep -v "kubernetes.io/service-account-token\|default-token"

echo "--- 6. Pods without Resource Limits ---"
kubectl get pods -n prod -o json | python3 -c "
import json, sys
pods = json.load(sys.stdin)
for pod in pods['items']:
    name = pod['metadata']['name']
    for c in pod['spec']['containers']:
        if 'resources' not in c or 'limits' not in c.get('resources', {}):
            print(f'NO LIMITS: {name}/{c[\"name\"]}')
"

echo "--- 7. Total Resource Count ---"
kubectl get all -n prod --no-headers | wc -l
```

</details>

<details>
<summary>📖 Explanation</summary>

This question tests your ability to use kubectl as an auditing tool — a real-world DevOps skill that also appears in CKAD troubleshooting scenarios.

**Key techniques used:**
- `custom-columns` → structured output without jq
- `--field-selector` → server-side filtering
- `awk` → quick text processing on kubectl output
- inline Python → complex JSON processing when custom-columns isn't enough

In the CKAD exam, you'll often need to find resources matching specific criteria quickly. Practice these patterns until they're muscle memory.

</details>

---

### Q27 · 🔴 BRUTAL — Deployment Canary by Hand

**⏱️ Target: 18 minutes**

**Context:** Implement a manual canary deployment: 80% traffic to stable, 20% to canary.

**Your Tasks:**
1. Create deployment `app-stable` in `staging` (image: `nginx:1.24`, 4 replicas, label `app=myapp, track=stable`)
2. Create deployment `app-canary` in `staging` (image: `nginx:1.25`, 1 replica, label `app=myapp, track=canary`)
3. Create ONE Service `myapp` in `staging` that routes to BOTH (selector: only `app=myapp`)
4. Verify the traffic split is approximately 80/20 by checking endpoints
5. If the canary is healthy, promote it: scale stable to 0 and canary to 5 replicas

---

<details>
<summary>✅ Solution</summary>

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-stable
  namespace: staging
spec:
  replicas: 4
  selector:
    matchLabels:
      app: myapp
      track: stable
  template:
    metadata:
      labels:
        app: myapp
        track: stable
    spec:
      containers:
        - name: app
          image: nginx:1.24
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-canary
  namespace: staging
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      track: canary
  template:
    metadata:
      labels:
        app: myapp
        track: canary
    spec:
      containers:
        - name: app
          image: nginx:1.25
---
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: staging
spec:
  selector:
    app: myapp          # Matches BOTH stable and canary
  ports:
    - port: 80
      targetPort: 80
EOF

kubectl rollout status deployment/app-stable -n staging
kubectl rollout status deployment/app-canary -n staging

# Verify 5 endpoints (4 stable + 1 canary = 80/20 split)
kubectl get endpoints myapp -n staging
# Should show 5 IPs

# Promote canary
kubectl scale deployment app-stable -n staging --replicas=0
kubectl scale deployment app-canary -n staging --replicas=5
kubectl rollout status deployment/app-canary -n staging
```

**Verify promotion:**
```bash
kubectl get deployments -n staging -l app=myapp
# app-stable: 0/0
# app-canary: 5/5  ✅

kubectl get endpoints myapp -n staging
# 5 endpoints, all canary pods  ✅
```

</details>

<details>
<summary>📖 Explanation</summary>

**Manual canary works because Kubernetes Services load-balance based on Pod count.** With 4 stable + 1 canary pods all matching the same selector, roughly 1/5 = 20% of requests go to canary.

**The key insight:** The Service selector uses only the common label (`app=myapp`). Both deployments add their own `track` label for differentiation — but the Service ignores it. Both deployments' pods end up in the same Endpoints list.

**Limitations of manual canary:**
- Traffic split is not exact (depends on load balancer algorithm)
- Can't do session-based routing (same user always gets same version)
- Better tools: Istio, Argo Rollouts, Flagger for precise traffic splitting

</details>

**Post-Cleanup:**
```bash
kubectl delete deployment app-stable app-canary -n staging --ignore-not-found
kubectl delete service myapp -n staging --ignore-not-found
```

---

### Q28 · 💀 NIGHTMARE — The Broken Namespace Migration

**⏱️ Target: 20 minutes**

**Pre-Setup:**
```bash
# Simulate a "legacy" application to migrate
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: legacy-app
  namespace: dev
  annotations:
    migrated-from: "bare-metal"
    version: "3.2.1"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: legacy-app
  template:
    metadata:
      labels:
        app: legacy-app
        env: dev
    spec:
      containers:
        - name: app
          image: nginx:1.25
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "200m"
              memory: "256Mi"
          env:
            - name: DB_HOST
              value: "legacy-db.internal"
            - name: APP_PORT
              value: "8080"
---
apiVersion: v1
kind: Service
metadata:
  name: legacy-svc
  namespace: dev
spec:
  selector:
    app: legacy-app
  ports:
    - port: 80
      targetPort: 80
EOF
kubectl rollout status deployment/legacy-app -n dev
```

**Your Tasks:** Migrate `legacy-app` from `dev` to `prod` namespace:
1. Export the deployment and service to YAML
2. Modify for `prod` namespace: change `env: dev` label to `env: prod`, update `DB_HOST` to `prod-db.internal`
3. Apply to `prod` — verify it runs
4. Verify the prod service has endpoints
5. Delete the dev version only after prod is verified

---

<details>
<summary>✅ Solution</summary>

```bash
# Step 1: Export both resources
kubectl get deployment legacy-app -n dev -o yaml > legacy-deploy.yaml
kubectl get service legacy-svc -n dev -o yaml > legacy-svc.yaml

# Step 2: Clean and modify deployment YAML
# Remove server-generated fields and change namespace/labels/env
cat legacy-deploy.yaml | python3 -c "
import yaml, sys

doc = yaml.safe_load(sys.stdin)

# Remove runtime fields
for field in ['creationTimestamp','resourceVersion','uid','generation']:
    doc['metadata'].pop(field, None)
    doc['spec']['template']['metadata'].pop(field, None)

doc.pop('status', None)

# Change namespace
doc['metadata']['namespace'] = 'prod'

# Change env label
doc['spec']['template']['metadata']['labels']['env'] = 'prod'

# Change DB_HOST env var
for env in doc['spec']['template']['spec']['containers'][0]['env']:
    if env['name'] == 'DB_HOST':
        env['value'] = 'prod-db.internal'

print(yaml.dump(doc, default_flow_style=False))
" > legacy-deploy-prod.yaml

# Or manually edit using sed (exam-friendly):
sed 's/namespace: dev/namespace: prod/' legacy-deploy.yaml | \
sed 's/env: dev/env: prod/' | \
sed 's/legacy-db.internal/prod-db.internal/' > legacy-deploy-prod.yaml

# Manual clean of server fields (remove these lines from the YAML):
# creationTimestamp, resourceVersion, uid, generation, status block

# Step 3: Apply to prod
kubectl apply -f legacy-deploy-prod.yaml
kubectl rollout status deployment/legacy-app -n prod

# Service migration
sed 's/namespace: dev/namespace: prod/' legacy-svc.yaml | \
grep -v "clusterIP:\|clusterIPs:\|resourceVersion:\|uid:\|creationTimestamp:\|status:" | \
kubectl apply -f -

# Step 4: Verify
kubectl get endpoints legacy-svc -n prod
# Should show pod endpoints ✅

kubectl exec -n prod deploy/legacy-app -- env | grep DB_HOST
# DB_HOST=prod-db.internal  ✅

# Step 5: Delete dev only after prod verified
kubectl delete deployment legacy-app -n dev
kubectl delete service legacy-svc -n dev
```

</details>

<details>
<summary>📖 Explanation</summary>

**Namespace migration is a real operational task.** The challenges are:
1. Exporting clean YAML (removing runtime fields)
2. Updating all namespace references consistently
3. Validating before deleting the source

**Runtime fields that MUST be removed before re-applying:**
- `metadata.resourceVersion` → version number set by API server
- `metadata.uid` → unique ID set by API server
- `metadata.creationTimestamp` → set at creation
- `spec.clusterIP` → set by API server for Services
- `status` → entire block is runtime state

**The exam rarely asks for this level of complexity, but understanding it makes you a much stronger operator.** The `kubectl replace` command can sometimes handle this more gracefully.

</details>

**Post-Cleanup:**
```bash
kubectl delete deployment legacy-app -n prod --ignore-not-found
kubectl delete service legacy-svc -n prod --ignore-not-found
```

---

### Q29 · 💀 NIGHTMARE — Timed Full Stack (20 minutes)

**⏱️ Target: 20 minutes — STRICT**

**This is a full CKAD-style exam task. Start your timer NOW.**

Deploy a complete 3-component application in namespace `prod`:

**Component 1 — Config:**
- ConfigMap `app-cm` with: `CACHE_TTL=300`, `MAX_POOL=10`, `LOG_FORMAT=json`
- Secret `app-secret` with: `DB_USER=appuser`, `DB_PASS=Pr0dP@ss!`, `JWT_SECRET=s3cr3tK3y`

**Component 2 — Deployment:**
- Name: `main-app`
- Image: `nginx:1.25`
- Replicas: 3
- All ConfigMap keys as env vars
- All Secret keys as env vars
- CPU: request=`150m`, limit=`300m`; Memory: request=`256Mi`, limit=`512Mi`
- Label: `app=main-app`, `tier=backend`, `version=v1`
- Annotation: `team=platform`, `owner=devops@company.com`
- Liveness probe: HTTP GET `/` port 80, delay=10s, period=15s
- Readiness probe: HTTP GET `/` port 80, delay=5s

**Component 3 — Services:**
- ClusterIP service `main-app-internal` on port 80, selector: `app=main-app`
- NodePort service `main-app-external` on port 80, nodePort 30099, selector: `app=main-app`

**Verification required:**
- All 3 replicas Running
- Both services have endpoints
- Env vars from both CM and Secret are present in pods

---

<details>
<summary>✅ Solution</summary>

```bash
# FAST path — imperative + targeted YAML editing

# Config (fastest: imperative)
kubectl create configmap app-cm \
  --from-literal=CACHE_TTL=300 \
  --from-literal=MAX_POOL=10 \
  --from-literal=LOG_FORMAT=json \
  -n prod

kubectl create secret generic app-secret \
  --from-literal=DB_USER=appuser \
  --from-literal=DB_PASS='Pr0dP@ss!' \
  --from-literal=JWT_SECRET=s3cr3tK3y \
  -n prod

# Deployment — generate base then edit
kubectl create deployment main-app --image=nginx:1.25 --replicas=3 \
  --dry-run=client -o yaml -n prod > main-app.yaml
```

```yaml
# main-app.yaml (after editing)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: main-app
  namespace: prod
  annotations:
    team: platform
    owner: devops@company.com
spec:
  replicas: 3
  selector:
    matchLabels:
      app: main-app
  template:
    metadata:
      labels:
        app: main-app
        tier: backend
        version: v1
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          resources:
            requests:
              cpu: "150m"
              memory: "256Mi"
            limits:
              cpu: "300m"
              memory: "512Mi"
          envFrom:
            - configMapRef:
                name: app-cm
            - secretRef:
                name: app-secret
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 15
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
```

```bash
kubectl apply -f main-app.yaml

# Services
kubectl expose deployment main-app --port=80 --name=main-app-internal -n prod

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: main-app-external
  namespace: prod
spec:
  type: NodePort
  selector:
    app: main-app
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30099
EOF

# VERIFY (do this fast!)
kubectl rollout status deployment/main-app -n prod
kubectl get endpoints main-app-internal main-app-external -n prod
POD=$(kubectl get pod -n prod -l app=main-app -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD -n prod -- env | grep -E "CACHE_TTL|DB_USER|JWT_SECRET"
```

</details>

<details>
<summary>📖 Explanation</summary>

**Exam time allocation for a complex task like this:**
- Config (CM + Secret): 2 min
- Deployment YAML: 5 min
- Services: 2 min
- Verification: 3 min
- Buffer: 8 min

**Speed tips that save the most time:**
1. `kubectl create ... --dry-run=client -o yaml` → never write boilerplate from scratch
2. `envFrom` over `env.valueFrom` → 2 lines vs 20 lines
3. Know your probe syntax by heart — it's the most common thing people get wrong
4. `kubectl rollout status` blocks until complete — use it instead of watching

If you completed this in under 20 minutes, you're well-prepared for the CKAD exam. The real exam tasks are slightly simpler, but they come in rapid succession with context switching.

</details>

**Post-Cleanup:**
```bash
kubectl delete deployment main-app -n prod --ignore-not-found
kubectl delete service main-app-internal main-app-external -n prod --ignore-not-found
kubectl delete configmap app-cm -n prod --ignore-not-found
kubectl delete secret app-secret -n prod --ignore-not-found
```

---

### Q30–Q50 · 💀 RAPID FIRE — 30 Second Diagnosis Each

**Context:** These are pure diagnostic speed drills. For each scenario, you have **the time it takes to run 3 kubectl commands** to identify the issue. No YAML writing — diagnosis only.

**Pre-Setup for all Q30–Q50:**
```bash
cat <<'BIGEOF' | kubectl apply -f -
# Q30 - wrong image
apiVersion: v1
kind: Pod
metadata: {name: q30, namespace: dev, labels: {q: "30"}}
spec:
  containers:
  - {name: app, image: "nginx:doesnotexist999"}
---
# Q31 - OOMKilled
apiVersion: v1
kind: Pod
metadata: {name: q31, namespace: dev, labels: {q: "31"}}
spec:
  containers:
  - name: app
    image: polinux/stress
    command: ["stress"]
    args: ["--vm","1","--vm-bytes","200M"]
    resources:
      limits: {memory: "50Mi"}
---
# Q32 - pending (no resources)
apiVersion: v1
kind: Pod
metadata: {name: q32, namespace: dev, labels: {q: "32"}}
spec:
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests: {memory: "99999Gi", cpu: "9999"}
---
# Q33 - crashloop
apiVersion: v1
kind: Pod
metadata: {name: q33, namespace: dev, labels: {q: "33"}}
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh","-c","exit 1"]
---
# Q34 - completed (should not restart)
apiVersion: v1
kind: Pod
metadata: {name: q34, namespace: dev, labels: {q: "34"}}
spec:
  restartPolicy: Never
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh","-c","echo done && exit 0"]
---
# Q35 - probe failing
apiVersion: v1
kind: Pod
metadata: {name: q35, namespace: dev, labels: {q: "35"}}
spec:
  containers:
  - name: app
    image: nginx:1.25
    livenessProbe:
      exec:
        command: ["cat","/nonexistent/file"]
      initialDelaySeconds: 3
      periodSeconds: 5
      failureThreshold: 2
BIGEOF
sleep 30

# Service issues for Q36-Q40
cat <<'SVCEOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata: {name: q36-app, namespace: dev}
spec:
  replicas: 2
  selector:
    matchLabels: {app: q36}
  template:
    metadata:
      labels: {app: q36}
    spec:
      containers:
      - {name: app, image: nginx:1.25}
---
apiVersion: v1
kind: Service
metadata: {name: q36-svc, namespace: dev}
spec:
  selector: {app: q36-wrong}    # Wrong selector
  ports:
  - {port: 80, targetPort: 80}
SVCEOF

echo "Q30-40 setup complete. Wait 30 more seconds."
sleep 30
```

---

For **each** of Q30–Q35, answer these 3 questions:
1. What is the Pod status?
2. What is the exact root cause?
3. What single command would fix it (if fixable without recreating)?

---

<details>
<summary>✅ Solutions for Q30–Q40</summary>

```bash
# Q30: ImagePullBackOff
kubectl get pod q30 -n dev
# STATUS: ImagePullBackOff / ErrImagePull
# CAUSE: Image nginx:doesnotexist999 doesn't exist in registry
# FIX: kubectl set image pod/q30 app=nginx:1.25

# Q31: OOMKilled
kubectl get pod q31 -n dev
# STATUS: OOMKilled (restarts > 0)
# CAUSE: Container tried to allocate 200M but limit is 50Mi → OOM
# FIX: kubectl patch pod q31 --type json -p '[{"op":"replace","path":"/spec/containers/0/resources/limits/memory","value":"256Mi"}]'
# (Won't work on running pod — must replace)
# Real fix: kubectl delete pod q31; recreate with higher memory limit

# Q32: Pending
kubectl get pod q32 -n dev
# STATUS: Pending
kubectl describe pod q32 -n dev | grep Events
# CAUSE: Insufficient memory/CPU — no node has 99999Gi RAM
# FIX: kubectl patch... (immutable) → delete and recreate with realistic resources

# Q33: CrashLoopBackOff
kubectl get pod q33 -n dev
# STATUS: CrashLoopBackOff
kubectl logs q33 -n dev
# (empty — exits immediately)
kubectl logs q33 -n dev --previous
# (nothing useful — exits with code 1)
# CAUSE: Command exits with code 1 immediately, Always restart policy → loop
# FIX: kubectl delete pod q33; fix the command or change restartPolicy

# Q34: Completed (not an error!)
kubectl get pod q34 -n dev
# STATUS: Completed
# CAUSE: Not a bug! restartPolicy:Never + exit 0 = completed successfully
# FIX: Nothing needed. This is correct behavior.

# Q35: CrashLoopBackOff (probe kills it)
kubectl get pod q35 -n dev
# STATUS: Running → then CrashLoopBackOff after ~15s
kubectl describe pod q35 -n dev | grep -A 3 "Events:"
# Warning Unhealthy: Liveness probe failed: cat /nonexistent/file
# Warning Killing: Container q35 failed liveness probe
# CAUSE: Liveness probe checks a file that doesn't exist
# FIX: kubectl patch pod q35... (immutable) → delete and fix probe

# Q36: Service has no endpoints
kubectl get endpoints q36-svc -n dev
# ENDPOINTS: <none>
kubectl describe service q36-svc -n dev | grep Selector
# Selector: app=q36-wrong
kubectl get pods -n dev -l app=q36 --show-labels
# Pods exist with app=q36
# CAUSE: Service selector app=q36-wrong doesn't match pod label app=q36
# FIX: kubectl patch service q36-svc -n dev -p '{"spec":{"selector":{"app":"q36"}}}'
```

</details>

<details>
<summary>📖 Explanation — The 6 Pod Failure Modes</summary>

| Status | Root Cause | Mutable Fix? |
|---|---|---|
| `ImagePullBackOff` | Image doesn't exist or no pull credentials | `kubectl set image` |
| `OOMKilled` | Memory limit exceeded | No — must recreate |
| `Pending` | No node can satisfy resource requests | No — must recreate |
| `CrashLoopBackOff` | Container exits repeatedly | No — must recreate |
| `Completed` | Container exited 0 with `restartPolicy: Never` | Not a bug! |
| `Running` but probes failing | Wrong probe path/port | No — must recreate |

**The most important insight:** Most pod spec fields are **immutable**. When debugging pods in the exam, your workflow is almost always: diagnose → delete → fix YAML → apply. Never waste time trying to patch immutable fields on running pods.

</details>

**Post-Cleanup (all Q30–Q40):**
```bash
kubectl delete pod q30 q31 q32 q33 q34 q35 -n dev --ignore-not-found --grace-period=0 --force
kubectl delete deployment q36-app -n dev --ignore-not-found
kubectl delete service q36-svc -n dev --ignore-not-found
```

---

## 🌐 Global Cleanup

> Run this when you're completely done with all questions.

```bash
# ============================================================
# GLOBAL CLEANUP — Run after ALL questions are done
# ============================================================

# Delete all resources in practice namespaces
kubectl delete all --all -n prod --grace-period=0 --force 2>/dev/null
kubectl delete all --all -n staging --grace-period=0 --force 2>/dev/null
kubectl delete all --all -n dev --grace-period=0 --force 2>/dev/null
kubectl delete all --all -n monitoring --grace-period=0 --force 2>/dev/null

# Delete configmaps and secrets
kubectl delete configmap --all -n prod 2>/dev/null
kubectl delete configmap --all -n staging 2>/dev/null
kubectl delete configmap --all -n dev 2>/dev/null
kubectl delete secret --all -n prod 2>/dev/null
kubectl delete secret --all -n staging 2>/dev/null
kubectl delete secret --all -n dev 2>/dev/null

# Delete the practice namespaces
kubectl delete namespace prod staging monitoring dev 2>/dev/null

# Remove node taint if it was left on
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
kubectl taint node $NODE dedicated=gpu:NoSchedule- 2>/dev/null || true
kubectl label node $NODE tier- 2>/dev/null || true

# Reset context to default
kubectl config set-context --current --namespace=default

echo "✅ Global cleanup complete"
```

---

## 📊 Scoring & Assessment

| Score | Meaning |
|---|---|
| < 50% in time | Revisit days 1-7 fundamentals |
| 50-70% in time | On track — practice more debugging |
| 70-85% in time | Strong — review edge cases |
| 85-100% in time | Ready to attempt real CKAD exam |

**Hardest questions (do these last):**
Q3, Q5, Q12, Q15, Q17, Q20, Q25, Q28, Q29

**Most exam-representative:**
Q1, Q4, Q11, Q16, Q18, Q21, Q26, Q27, Q29

---

## ⚡ Speed Reference Card

```bash
# === GENERATE YAML FAST ===
alias kdr='kubectl --dry-run=client -o yaml'
kdr run pod --image=nginx > pod.yaml
kdr create deploy app --image=nginx --replicas=3 > dep.yaml
kdr expose deploy app --port=80 > svc.yaml
kdr create cm mymap --from-literal=k=v > cm.yaml
kdr create secret generic mysec --from-literal=k=v > sec.yaml

# === SWITCH CONTEXT ===
alias kns='kubectl config set-context --current --namespace'
kns prod    # Switch to prod
kns default # Reset

# === DEBUG FAST ===
alias kdp='kubectl describe pod'
alias kgp='kubectl get pods'
alias kge='kubectl get endpoints'

# === WAIT FOR READY ===
kubectl wait pod/name --for=condition=Ready --timeout=60s
kubectl rollout status deploy/name --timeout=120s

# === DECODE SECRET ===
kubectl get secret name -o jsonpath='{.data.key}' | base64 -d

# === TEMP DEBUG POD ===
kubectl run tmp --image=busybox:1.36 --rm -it --restart=Never -- sh
kubectl run tmp --image=curlimages/curl --rm -it --restart=Never -- sh
```

---

*These questions were designed to break you before the exam does. If you can solve them, the real CKAD will feel manageable. 💪*